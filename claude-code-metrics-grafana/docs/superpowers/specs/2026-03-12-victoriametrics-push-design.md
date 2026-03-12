# Design: Replace Prometheus with VictoriaMetrics (Push-Based)

**Date:** 2026-03-12
**Project:** claude-code-metrics-grafana
**Status:** Approved

---

## Problem

`claude_code.cost.usage` is a delta metric pushed by Claude Code every ~1 minute via OTLP/HTTP. The current stack uses Prometheus in pull (scrape) mode at a 15-second interval, causing each delta value to be scraped ~4 times before the next push arrives. This inflates all aggregations by ~4x and makes correct session/user cost totals impossible with standard PromQL.

---

## Solution

Replace Prometheus with VictoriaMetrics (single-node). Change the OTel Collector from a pull-based Prometheus exporter to a push-based `prometheusremotewrite` exporter. Each delta received by the OTel Collector is pushed to VictoriaMetrics exactly once — eliminating the scrape/push mismatch entirely.

---

## Architecture

```
Before: Claude Code → OTel Collector (expose :8889) ← Prometheus scrapes → Grafana
After:  Claude Code → OTel Collector → prometheusremotewrite → VictoriaMetrics → Grafana
```

The OTel Collector accumulates delta values into a monotonically increasing counter internally and writes one sample per push received. VictoriaMetrics stores these samples and exposes a Prometheus-compatible query API that Grafana uses unchanged (same Prometheus datasource plugin).

---

## Components Changed

### 1. `docker-compose.yml`

- Remove the `prometheus` service.
- Add `victoriametrics/victoria-metrics:latest` single-node on port `8428`.
- Update OTel Collector dependency from `prometheus` to `victoriametrics`.
- Remove the `prometheus/prometheus.yml` volume mount.

### 2. `otel-collector/config.yml`

- Remove the `prometheus` exporter (pull endpoint on `:8889`).
- Add `prometheusremotewrite` exporter:
  - `endpoint: http://victoriametrics:8428/api/v1/write`
  - `resource_to_telemetry_conversion.enabled: true` (preserves `user_email`, `session_id`, etc. as labels)
- Update the metrics pipeline to use `prometheusremotewrite` instead of `prometheus`.

### 3. `grafana/provisioning/datasources/datasource.yml`

- Change datasource URL from `http://prometheus:9090` to `http://victoriametrics:8428`.
- No other changes — VictoriaMetrics exposes a Prometheus-compatible HTTP API.

### 4. `grafana/provisioning/dashboards/cost-usage.json`

Update all four dashboard panels with correct PromQL queries:

| Panel | Query |
|---|---|
| Cost Over Time (bar chart, range) | `sum by (session_id, user_email) (increase(claude_code_cost_usage_USD_total[$__rate_interval]))` |
| Top 5 Sessions by Cost | `topk(5, sum by (session_id, user_email) (last_over_time(claude_code_cost_usage_USD_total[$__range])))` |
| Top 5 Users by Cost | `topk(5, sum by (user_email) (last_over_time((sum by (user_email, session_id) (claude_code_cost_usage_USD_total))[$__range:])))` |
| Total Cost per User | `sum by (user_email) (last_over_time((sum by (user_email, session_id) (claude_code_cost_usage_USD_total))[$__range:]))` |

`last_over_time` captures the final cumulative value for each session within the dashboard time range, including sessions that have gone stale (ended >5 min ago).

---

## Data Flow Detail

1. Claude Code pushes a delta sample (e.g. `$0.030`) via OTLP/HTTP to OTel Collector every ~1 min.
2. OTel Collector accumulates the delta onto an in-memory running total per label set.
3. On each delta receipt, OTel Collector immediately writes one sample to VictoriaMetrics via remote write.
4. VictoriaMetrics stores the monotonically increasing counter value with the original timestamp.
5. Grafana queries VictoriaMetrics; `last_over_time` returns the final counter value per session.

**Note:** OTel Collector state is in-memory. A collector restart resets the running total for any in-progress session to zero. This is a one-time blip per restart, not a systematic error.

---

## Files Not Changed

- `prometheus/prometheus.yml` — no longer used (can be deleted or left as dead config)
- `client/install.py`, `client/uninstall.py`, `client/settings.json` — no changes needed
- `grafana/provisioning/dashboards/dashboard.yml` — no changes needed

---

## Success Criteria

- `docker compose up -d` starts cleanly with VictoriaMetrics replacing Prometheus.
- OTel Collector logs show successful remote write to VictoriaMetrics.
- All four Grafana panels display correct, non-inflated values.
- Top 5 sessions and top 5 users panels correctly include completed (stale) sessions within the selected time range.
