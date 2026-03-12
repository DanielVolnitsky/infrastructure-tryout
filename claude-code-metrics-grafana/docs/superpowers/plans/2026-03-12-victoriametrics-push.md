# VictoriaMetrics Push-Based Migration Implementation Plan

> **For agentic workers:** REQUIRED: Use superpowers:subagent-driven-development (if subagents available) or superpowers:executing-plans to implement this plan. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace Prometheus (pull-based scrape) with VictoriaMetrics (push-based remote write) in the claude-code-metrics-grafana stack so that `claude_code.cost.usage` delta metrics are aggregated correctly.

**Architecture:** The OTel Collector receives delta metrics from Claude Code, converts them to cumulative counters via the `deltatocumulative` processor, then pushes each sample directly to VictoriaMetrics via `prometheusremotewrite`. Grafana queries VictoriaMetrics using the standard Prometheus datasource plugin (VictoriaMetrics is API-compatible). Dashboard queries are updated to use `last_over_time` subqueries that correctly capture stale (completed) sessions within the selected time range.

**Tech Stack:** Docker Compose, OpenTelemetry Collector Contrib (`otel/opentelemetry-collector-contrib:0.120.0`), VictoriaMetrics single-node (`victoriametrics/victoria-metrics:latest`), Grafana 11.5.2, PromQL / MetricsQL.

**Spec:** `docs/superpowers/specs/2026-03-12-victoriametrics-push-design.md`

---

## File Map

| Action | File | Change |
|---|---|---|
| Modify | `docker-compose.yml` | Replace `prometheus` service with `victoriametrics`; update volumes and Grafana `depends_on` |
| Replace | `otel-collector/config.yml` | New config with `deltatocumulative` processor and `prometheusremotewrite` exporter |
| Modify | `grafana/provisioning/datasources/datasource.yml` | Point URL at VictoriaMetrics |
| Modify | `grafana/provisioning/dashboards/cost-usage.json` | Update all 5 panel PromQL queries |
| Delete | `prometheus/prometheus.yml` | No longer used |

---

## Chunk 1: Infrastructure — docker-compose, OTel Collector, datasource

### Task 1: Replace Prometheus with VictoriaMetrics in `docker-compose.yml`

**Files:**
- Modify: `docker-compose.yml`

Current file for reference:
```yaml
services:
  otel-collector:
    image: otel/opentelemetry-collector-contrib:0.120.0
    volumes:
      - ./otel-collector/config.yml:/etc/otelcol-contrib/config.yaml:ro
    ports:
      - "4318:4318"

  prometheus:
    image: prom/prometheus:v3.2.1
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - prometheus-data:/prometheus
    ports:
      - "9090:9090"
    depends_on:
      - otel-collector

  grafana:
    image: grafana/grafana:11.5.2
    volumes:
      - ./grafana/provisioning:/etc/grafana/provisioning:ro
      - grafana-data:/var/lib/grafana
    ports:
      - "3000:3000"
    environment:
      GF_AUTH_ANONYMOUS_ENABLED: "true"
      GF_AUTH_ANONYMOUS_ORG_ROLE: Admin
      GF_AUTH_DISABLE_LOGIN_FORM: "true"
    depends_on:
      - prometheus

volumes:
  prometheus-data:
  grafana-data:
```

- [ ] **Step 1: Replace `docker-compose.yml`**

Write the file with these exact contents:

```yaml
services:
  otel-collector:
    image: otel/opentelemetry-collector-contrib:0.120.0
    volumes:
      - ./otel-collector/config.yml:/etc/otelcol-contrib/config.yaml:ro
    ports:
      - "4318:4318" # OTLP HTTP receiver (Claude Code pushes here)

  victoriametrics:
    image: victoriametrics/victoria-metrics:latest
    volumes:
      - victoriametrics-data:/storage
    ports:
      - "8428:8428"
    command:
      - --storageDataPath=/storage
    depends_on:
      - otel-collector

  grafana:
    image: grafana/grafana:11.5.2
    volumes:
      - ./grafana/provisioning:/etc/grafana/provisioning:ro
      - grafana-data:/var/lib/grafana
    ports:
      - "3000:3000"
    environment:
      GF_AUTH_ANONYMOUS_ENABLED: "true"
      GF_AUTH_ANONYMOUS_ORG_ROLE: Admin
      GF_AUTH_DISABLE_LOGIN_FORM: "true"
    depends_on:
      - victoriametrics

volumes:
  victoriametrics-data:
  grafana-data:
```

- [ ] **Step 2: Verify compose file is valid**

```bash
cd claude-code-metrics-grafana
docker compose config --quiet
```

Expected: no output, exit code 0.

- [ ] **Step 3: Commit**

```bash
git add claude-code-metrics-grafana/docker-compose.yml
git commit -m "[claude-code-metrics-grafana] replace Prometheus with VictoriaMetrics in docker-compose (Step 1)"
```

---

### Task 2: Replace `otel-collector/config.yml`

**Files:**
- Replace: `claude-code-metrics-grafana/otel-collector/config.yml`

- [ ] **Step 1: Write new OTel Collector config**

Replace the entire file with:

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

- [ ] **Step 2: Commit**

```bash
git add claude-code-metrics-grafana/otel-collector/config.yml
git commit -m "[claude-code-metrics-grafana] add deltatocumulative + prometheusremotewrite to OTel Collector (Step 2)"
```

---

### Task 3: Update Grafana datasource to point at VictoriaMetrics

**Files:**
- Modify: `claude-code-metrics-grafana/grafana/provisioning/datasources/datasource.yml`

- [ ] **Step 1: Update the datasource URL**

Change `url: http://prometheus:9090` to `url: http://victoriametrics:8428`. Full file after change:

```yaml
apiVersion: 1

datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    url: http://victoriametrics:8428
    isDefault: true
    editable: false
```

- [ ] **Step 2: Commit**

```bash
git add claude-code-metrics-grafana/grafana/provisioning/datasources/datasource.yml
git commit -m "[claude-code-metrics-grafana] point Grafana datasource at VictoriaMetrics (Step 3)"
```

---

### Task 4: Delete unused `prometheus/prometheus.yml`

**Files:**
- Delete: `claude-code-metrics-grafana/prometheus/prometheus.yml`

- [ ] **Step 1: Delete the file and commit**

```bash
git rm claude-code-metrics-grafana/prometheus/prometheus.yml
git commit -m "[claude-code-metrics-grafana] remove unused prometheus.yml (Step 4)"
```

---

### Task 5: Smoke-test the stack starts cleanly

- [ ] **Step 1: Start the stack**

```bash
cd claude-code-metrics-grafana
docker compose up -d
```

Expected: three containers start — `otel-collector`, `victoriametrics`, `grafana`.

- [ ] **Step 2: Verify all containers are healthy**

```bash
docker compose ps
```

Expected: all three services in `running` state, no `Exit` or `Restarting`.

- [ ] **Step 3: Verify VictoriaMetrics is reachable**

```bash
curl -s http://localhost:8428/health
```

Expected: `Single-node VictoriaMetrics is healthy and ready for queries.` (or similar 200 response).

- [ ] **Step 4: Verify OTel Collector is ready**

```bash
docker compose logs otel-collector | tail -20
```

Expected: logs show the collector started with the `deltatocumulative` processor and `prometheusremotewrite` exporter in the pipeline. No `Error` lines on startup.

- [ ] **Step 5: Verify Grafana datasource connects**

Open `http://localhost:3000`, navigate to **Connections → Data Sources → Prometheus**, click **Save & Test**.

Expected: "Data source is working" (green).

---

## Chunk 2: Grafana Dashboard Queries

### Task 6: Update all five dashboard panel queries in `cost-usage.json`

**Files:**
- Modify: `claude-code-metrics-grafana/grafana/provisioning/dashboards/cost-usage.json`

The dashboard JSON has five panels identified by `id`. Make the following `expr` (and one `legendFormat`) changes:

| Panel id | Panel title | Field | Old value | New value |
|---|---|---|---|---|
| 4 | `Cost Usage Over Time` | `expr` | `sum by (user_email, session_id) (claude_code_cost_usage_USD_total)` | `sum by (session_id, user_email) (increase(claude_code_cost_usage_USD_total[$__rate_interval]))` |
| 1 | `Cost per User Comparison` | `expr` | `sum by (user_email) (claude_code_cost_usage_USD_total)` | `sum by (user_email) (last_over_time((sum by (user_email, session_id) (claude_code_cost_usage_USD_total))[$__range:1m]))` |
| 6 | `Bottom 5 Users by Cost` | `expr` | `bottomk(5, sum by (user_email) (claude_code_cost_usage_USD_total))` | `bottomk(5, sum by (user_email) (last_over_time((sum by (user_email, session_id) (claude_code_cost_usage_USD_total))[$__range:1m])))` |
| 2 | `Top 5 Users by Cost` | `expr` | `topk(5, sum by (user_email) (claude_code_cost_usage_USD_total))` | `topk(5, sum by (user_email) (last_over_time((sum by (user_email, session_id) (claude_code_cost_usage_USD_total))[$__range:1m])))` |
| 8 | `Top 5 Session by Cost` | `expr` | `sum by (session_id) (claude_code_cost_usage_USD_total)` | `sum by (session_id, user_email) (last_over_time(claude_code_cost_usage_USD_total[$__range:1m]))` |
| 8 | `Top 5 Session by Cost` | `legendFormat` | `{{session_id}}` | `{{session_id}} / {{user_email}}` |

**Why these queries:**
- `last_over_time(...[$__range:1m])` — scans the full dashboard time window at 1-minute resolution to find the last (= cumulative total) counter value per session, including sessions that ended and went stale. Without this, VictoriaMetrics instant queries only look back 5 minutes by default.
- `increase(...)` on `Cost Usage Over Time` — approximates cost incurred per time bucket from the cumulative counter, producing per-interval bars instead of a flat running total line.
- Panels 1, 2, 6, 8 keep `"instant": true` — they are evaluated at `t=now` with `$__range` covering the full window, which is what makes the subquery capture historical sessions correctly.

- [ ] **Step 1: Update panel 4 (`Cost Usage Over Time`) expr**

In `cost-usage.json`, find the `targets` array inside the panel with `"id": 4` and update its `expr`:

```json
"expr": "sum by (session_id, user_email) (increase(claude_code_cost_usage_USD_total[$__rate_interval]))"
```

- [ ] **Step 2: Update panel 1 (`Cost per User Comparison`) expr**

Find panel `"id": 1` (inside the collapsed `User` row) and update its `expr`:

```json
"expr": "sum by (user_email) (last_over_time((sum by (user_email, session_id) (claude_code_cost_usage_USD_total))[$__range:1m]))"
```

- [ ] **Step 3: Update panel 6 (`Bottom 5 Users by Cost`) expr**

Find panel `"id": 6` and update its `expr`:

```json
"expr": "bottomk(5, sum by (user_email) (last_over_time((sum by (user_email, session_id) (claude_code_cost_usage_USD_total))[$__range:1m])))"
```

- [ ] **Step 4: Update panel 2 (`Top 5 Users by Cost`) expr**

Find panel `"id": 2` and update its `expr`:

```json
"expr": "topk(5, sum by (user_email) (last_over_time((sum by (user_email, session_id) (claude_code_cost_usage_USD_total))[$__range:1m])))"
```

- [ ] **Step 5: Update panel 8 (`Top 5 Session by Cost`) expr and legendFormat**

Find panel `"id": 8` and update both fields:

```json
"expr": "sum by (session_id, user_email) (last_over_time(claude_code_cost_usage_USD_total[$__range:1m]))",
"legendFormat": "{{session_id}} / {{user_email}}"
```

- [ ] **Step 6: Verify the JSON is valid**

```bash
python3 -c "import json; json.load(open('claude-code-metrics-grafana/grafana/provisioning/dashboards/cost-usage.json')); print('valid')"
```

Expected: `valid`

- [ ] **Step 7: Commit**

```bash
git add claude-code-metrics-grafana/grafana/provisioning/dashboards/cost-usage.json
git commit -m "[claude-code-metrics-grafana] update dashboard queries for VictoriaMetrics push-based metrics (Step 5)"
```

---

### Task 7: End-to-end verification

- [ ] **Step 1: Restart the stack to pick up the dashboard changes**

```bash
cd claude-code-metrics-grafana
docker compose down
docker compose up -d
```

- [ ] **Step 2: Send a test OTLP metric push**

Use `curl` to simulate a Claude Code push with a known cost value. Send two pushes for the same session to verify accumulation:

```bash
# Push 1 — session abc123, $0.030
curl -s -X POST http://localhost:4318/v1/metrics \
  -H "Content-Type: application/json" \
  -d '{
    "resourceMetrics": [{
      "resource": {
        "attributes": [
          {"key": "user.email", "value": {"stringValue": "test@example.com"}},
          {"key": "service.name", "value": {"stringValue": "claude-code"}}
        ]
      },
      "scopeMetrics": [{
        "metrics": [{
          "name": "claude_code.cost.usage",
          "unit": "USD",
          "sum": {
            "aggregationTemporality": 1,
            "isMonotonic": true,
            "dataPoints": [{
              "attributes": [
                {"key": "session.id", "value": {"stringValue": "abc123"}},
                {"key": "user.email", "value": {"stringValue": "test@example.com"}}
              ],
              "asDouble": 0.030,
              "startTimeUnixNano": "1700000000000000000",
              "timeUnixNano": "1700000060000000000"
            }]
          }
        }]
      }]
    }]
  }'
```

Wait 2 seconds, then send push 2 (same session, different delta):

```bash
# Push 2 — same session abc123, $0.020
curl -s -X POST http://localhost:4318/v1/metrics \
  -H "Content-Type: application/json" \
  -d '{
    "resourceMetrics": [{
      "resource": {
        "attributes": [
          {"key": "user.email", "value": {"stringValue": "test@example.com"}},
          {"key": "service.name", "value": {"stringValue": "claude-code"}}
        ]
      },
      "scopeMetrics": [{
        "metrics": [{
          "name": "claude_code.cost.usage",
          "unit": "USD",
          "sum": {
            "aggregationTemporality": 1,
            "isMonotonic": true,
            "dataPoints": [{
              "attributes": [
                {"key": "session.id", "value": {"stringValue": "abc123"}},
                {"key": "user.email", "value": {"stringValue": "test@example.com"}}
              ],
              "asDouble": 0.020,
              "startTimeUnixNano": "1700000060000000000",
              "timeUnixNano": "1700000120000000000"
            }]
          }
        }]
      }]
    }]
  }'
```

- [ ] **Step 3: Verify OTel Collector received and forwarded the pushes**

```bash
docker compose logs otel-collector | grep -E "(cost|remote|write|error)" | tail -20
```

Expected: log lines showing the metric was received and remote write succeeded. No `error` lines.

- [ ] **Step 4: Query VictoriaMetrics directly to verify accumulation**

```bash
curl -s "http://localhost:8428/api/v1/query?query=claude_code_cost_usage_USD_total" | python3 -m json.tool
```

Expected: a JSON response containing a result with value `0.05` (sum of the two pushes: $0.030 + $0.020), for labels `{session_id="abc123", user_email="test@example.com"}`.

- [ ] **Step 5: Verify the Grafana dashboard shows correct data**

Open `http://localhost:3000`, navigate to **Dashboards → Agentic Tool Metrics**. Set the time range to **Last 15 minutes**.

- **`Top 5 Session by Cost`** bargauge: should show `$0.05` for session `abc123 / test@example.com`.
- **`Cost per User Comparison`**: should show `$0.05` for `test@example.com`.
- **`Top 5 Users by Cost`**: should show `test@example.com` with `$0.05`.

- [ ] **Step 6: Update README to reflect the new architecture**

In `claude-code-metrics-grafana/README.md`, update the Architecture section from:

```
Claude Code (OTLP/HTTP JSON) → OTel Collector → Prometheus (scrape) → Grafana Dashboard
```

to:

```
Claude Code (OTLP/HTTP JSON) → OTel Collector → VictoriaMetrics (remote write) → Grafana Dashboard
```

Also update the Services table: replace the `Prometheus | 9090` row with `VictoriaMetrics | 8428 | Stores metrics (remote write endpoint)`.

Remove or update the **Known Limitations** section — the scrape/push mismatch issues described there are resolved by this migration.

- [ ] **Step 7: Final commit**

```bash
git add claude-code-metrics-grafana/README.md
git commit -m "[claude-code-metrics-grafana] update README for VictoriaMetrics architecture (Step 6)"
```
