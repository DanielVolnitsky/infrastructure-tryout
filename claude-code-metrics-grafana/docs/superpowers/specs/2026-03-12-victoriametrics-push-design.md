# Design: Replace Prometheus with VictoriaMetrics (Push-Based)

**Date:** 2026-03-12
**Project:** claude-code-metrics-grafana
**Status:** Approved

---

## Problem

`claude_code.cost.usage` is a delta metric pushed by Claude Code every ~1 minute via OTLP/HTTP. The current stack uses Prometheus in pull (scrape) mode at a 15-second interval, causing each delta value to be scraped multiple times before the next push arrives. This inflates aggregations and makes correct session/user cost totals impossible with standard PromQL.

---

## Solution

Replace Prometheus with VictoriaMetrics (single-node). Change the OTel Collector pipeline to:
1. Convert delta metrics to cumulative via the `deltatocumulative` processor.
2. Push to VictoriaMetrics via the `prometheusremotewrite` exporter — exactly once per delta received.

This eliminates the scrape/push mismatch and produces proper monotonically-increasing counters that standard PromQL aggregations can work with correctly.

---

## Architecture

```
Before: Claude Code → OTel Collector (expose :8889) ← Prometheus scrapes → Grafana
After:  Claude Code → OTel Collector (deltatocumulative) → prometheusremotewrite → VictoriaMetrics → Grafana
```

The `deltatocumulative` processor accumulates each incoming delta onto an in-memory running total per label set. On each delta receipt, the OTel Collector pushes one cumulative counter sample to VictoriaMetrics. VictoriaMetrics stores these samples and exposes a Prometheus-compatible query API that Grafana uses via its standard Prometheus datasource plugin.

---

## Components Changed

### 1. `docker-compose.yml`

- Remove the `prometheus` service entirely.
- Add `victoriametrics/victoria-metrics:latest` single-node on port `8428`, mounting a `victoriametrics-data` named volume.
- Update Grafana's `depends_on` from `prometheus` to `victoriametrics`.
- Update the top-level `volumes:` block — remove `prometheus-data:`, add `victoriametrics-data:`:

```yaml
volumes:
  victoriametrics-data:
  grafana-data:
```

Retention period: accept VictoriaMetrics default (1 month).

### 2. `otel-collector/config.yml`

Replace the entire file with the following structure (showing full YAML nesting to avoid placement ambiguity):

```yaml
receivers:
  otlp:
    protocols:
      http:
        endpoint: 0.0.0.0:4318

processors:
  deltatocumulative:
    max_stale: 30m

exporters:
  prometheusremotewrite:
    endpoint: http://victoriametrics:8428/api/v1/write
    send_metadata: true
    resource_to_telemetry_conversion:
      enabled: true
  debug:
    verbosity: detailed
    sampling_initial: 5
    sampling_thereafter: 200

service:
  telemetry:
    logs:
      level: debug
  pipelines:
    metrics:
      receivers: [otlp]
      processors: [deltatocumulative]
      exporters: [prometheusremotewrite, debug]
```

Key changes from current:
- New top-level `processors:` block with `deltatocumulative`. `max_stale: 30m` evicts in-memory state for series not seen in 30 minutes, preventing unbounded memory growth from high-cardinality `session_id` labels.
- `prometheusremotewrite` replaces `prometheus` exporter. `send_metadata: true` ensures VictoriaMetrics receives metric type (counter vs gauge), required for `increase()` to behave correctly.
- Pipeline updated to include the `deltatocumulative` processor.

### 3. `grafana/provisioning/datasources/datasource.yml`

Change datasource URL from `http://prometheus:9090` to `http://victoriametrics:8428`. No other changes — VictoriaMetrics exposes a Prometheus-compatible HTTP API.

### 4. `grafana/provisioning/dashboards/cost-usage.json`

The dashboard has five panels. Required changes per panel:

**`Cost Usage Over Time`** (bar chart, range query — keep `"range": true`)
- Update `expr` to: `sum by (session_id, user_email) (increase(claude_code_cost_usage_USD_total[$__rate_interval]))`

**`Cost per User Comparison`** (bar chart, instant query — keep `"instant": true`)
- Update `expr` to: `sum by (user_email) (last_over_time((sum by (user_email, session_id) (claude_code_cost_usage_USD_total))[$__range:1m]))`

**`Bottom 5 Users by Cost`** (bar chart, instant query — keep `"instant": true`)
- Update `expr` to: `bottomk(5, sum by (user_email) (last_over_time((sum by (user_email, session_id) (claude_code_cost_usage_USD_total))[$__range:1m])))`

**`Top 5 Users by Cost`** (bar chart, instant query — keep `"instant": true`)
- Update `expr` to: `topk(5, sum by (user_email) (last_over_time((sum by (user_email, session_id) (claude_code_cost_usage_USD_total))[$__range:1m])))`

**`Top 5 Session by Cost`** (bargauge, instant query — keep `"instant": true`)
- Update `expr` to: `sum by (session_id, user_email) (last_over_time(claude_code_cost_usage_USD_total[$__range:1m]))`
- Update `legendFormat` from `{{session_id}}` to `{{session_id}} / {{user_email}}`

Notes on query design:
- `instant: true` is essential for the user/session panels: evaluated at `now` with `$__range` covering the full dashboard window, `last_over_time` returns the final counter value for every session that had at least one push within that window — including sessions that ended hours ago.
- The `[$__range:1m]` subquery step evaluates at 1-minute resolution to match the push cadence, preventing short sessions' final samples from being skipped.
- The `Top 5 Session by Cost` bargauge returns all sessions; Grafana's `lastNotNull` reduce option ranks them visually. There is no query-level `topk` filter.
- `Cost Usage Over Time` uses `increase()` to approximate cost per time bucket. At step boundaries VictoriaMetrics may interpolate — this is a minor visual artefact, not a correctness issue for total cost.

### 5. `prometheus/prometheus.yml`

Delete this file — it is no longer referenced by any service.

---

## Data Flow Detail

1. Claude Code pushes a delta sample (e.g. `$0.030`) via OTLP/HTTP to OTel Collector every ~1 min.
2. The `deltatocumulative` processor adds the delta to an in-memory running total per label set (running total becomes `$0.030`, then `$0.052`, then `$0.088`).
3. The `prometheusremotewrite` exporter sends the updated cumulative value and metric type metadata to VictoriaMetrics.
4. VictoriaMetrics stores the monotonically increasing counter value with the received timestamp.
5. Grafana queries VictoriaMetrics; `last_over_time` returns the final counter value per session for the chosen time range.

**Known limitation — collector restart:** The `deltatocumulative` processor holds running totals in memory. A restart resets the in-memory total to zero. The next push after restart writes a value lower than the previously stored counter, which `increase()` will interpret as a reset — producing a zero or negative spike in the "Cost Usage Over Time" panel at the restart boundary. Data written before the restart is unaffected. This is accepted as a known trade-off.

---

## Success Criteria

- `docker compose up -d` starts all three services (otel-collector, victoriametrics, grafana) with no errors.
- OTel Collector logs confirm successful remote write delivery after metrics arrive.
- After N successive pushes for one session, the `Top 5 Session by Cost` bargauge shows a value equal to the sum of all N delta values for that session.
- `Top 5 Users by Cost` and `Cost per User Comparison` include sessions that have had no pushes in the last 5+ minutes but pushed at least once within the selected dashboard time range.
- `Cost Usage Over Time` shows non-zero bars for time buckets in which pushes occurred (not a flat line of the last delta value).
