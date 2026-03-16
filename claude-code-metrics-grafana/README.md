# Claude Code Metrics — VictoriaMetrics & Grafana

Alternative to the AWS CloudWatch solution.

## Architecture

```
┌─────────────────────────────────────────────────────────────────────────────────────────────┐
│ Developer Machines                                                                          │
│   Claude Code  ──OTLP/HTTP :4318──►                                                        │
└────────────────────────────────────┼────────────────────────────────────────────────────────┘
                                     │
                              ALB Ingress (AWS)
                                     │
┌────────────────────────────────────▼────────────────────────────────────────────────────────┐
│ k8s / EKS                                                                                   │
│                                                                                             │
│  OTel Collector          VictoriaMetrics          Grafana                                   │
│  (delta→cumulative)  ──► (time-series store)  ──► (real-time ops dashboard)                 │
│                                   │                                                         │
│                                   │ HTTP query API                                          │
│                                   ▼                                                         │
│                          Daily CronJob / Jenkins Job                                        │
│                          (e.g. 23:55 UTC — aggregate prior day)                                  │
└───────────────────────────────────┼─────────────────────────────────────────────────────────┘
                                    │ S3 PutObject · dated Parquet
┌───────────────────────────────────▼─────────────────────────────────────────────────────────┐
│ AWS                                                                                         │
│                                                                                             │
│  AI Usage API  ──existing ETL──►  S3  ──►  Athena / Glue  ──►  QuickSight                  │
│                                   ▲         (daily business & reconciliation view)          │
│                       ────────────┘                                                         │
└─────────────────────────────────────────────────────────────────────────────────────────────┘
```

## Architecture Alternatives

Three options were evaluated for combining real-time Claude Code operational metrics with daily business reporting alongside other AI usage data.

### Option A — Separate tools, bridged daily ✅ chosen

- **Grafana/VictoriaMetrics** handle real-time Claude Code ops monitoring (sub-minute latency, alerting)
- A **daily k8s CronJob | Jenkins Job** reads the previous day's aggregates from VictoriaMetrics and writes dated Parquet files to S3
- **QuickSight** consumes that S3 prefix via Athena alongside existing AI usage API data, enabling cross-service cost reconciliation
- Each tool does what it is best at; no platform migration required

### Option B — Consolidate in Grafana

- Remove QuickSight; add the **Grafana Athena datasource plugin** to query existing S3/Athena data directly
- Single platform for both real-time Claude Code metrics and historical AI usage
- Trade-off: Grafana is less suited for business reporting (no pivot tables, limited ad-hoc exploration); per-scan Athena query costs

### Option C — Dual-write to both systems

- Extend the OTel Collector with a second exporter (**Firehose → S3**) so raw Claude Code data lands in both VictoriaMetrics and QuickSight natively
- Most complex pipeline; two systems holding the same data with risk of metric divergence
- Not chosen due to operational overhead and pipeline complexity

## Quick Start

```bash
# 1. Start the stack
docker compose up -d

# 2. Configure Claude Code (run once)
cd client
python3 install.py user@example.com

# 3. Open Grafana
open http://localhost:3000
# Navigate to Dashboards → Agentic Tool Metrics
```

## Services

| Service         | Port | Purpose                               |
|-----------------|------|---------------------------------------|
| OTel Collector  | 4318 | Receives OTLP/HTTP from Claude Code   |
| VictoriaMetrics | 8428 | Receives remote write, stores metrics |
| Grafana         | 3000 | Dashboard (no login required)         |

## Emitted Metrics

Resource attributes are promoted to labels via `resource_to_telemetry_conversion` in the OTel Collector: `user.email`, `host.arch`, `os.type`, `os.version`,
`service.name`, `service.version`.

| Metric                          | Description         | Unit   | Data-point labels                                                                                              | Series per session |
|---------------------------------|---------------------|--------|----------------------------------------------------------------------------------------------------------------|--------------------|
| `claude_code.cost.usage`        | Cost of the session | USD    | `user.id`, `session.id`, `terminal.type`, `model`                                                              | 1                  |
| `claude_code.token.usage`       | Token consumption   | tokens | `user.id`, `session.id`, `terminal.type`, `model`, `type` (`input` / `output` / `cacheRead` / `cacheCreation`) | 4                  |
| `claude_code.active_time.total` | Active time         | s      | `user.id`, `session.id`, `terminal.type`, `type` (`user` / `cli`)                                              | 2                  |
| `claude_code.session.count`     | Sessions started    | —      | `user.id`, `session.id`, `terminal.type`                                                                       | 1                  |

## Metric Push Behaviour

Claude Code pushes metrics to the OTel Collector via OTLP/HTTP on a **~1-minute Delta interval**. Each push covers only the cost/tokens incurred within that
window (not cumulative).

### Example

`claude_code.cost.usage` — observed metrics:

| Push | cost (USD) | session.id | user.id    | user.email    | terminal.type | model                          | host.arch | os.type | os.version | service.name | service.version | StartTimestamp | Timestamp |
|------|------------|------------|------------|---------------|---------------|--------------------------------|-----------|---------|------------|--------------|-----------------|----------------|-----------|
| 1    | 0.030103   | e0d7c017-… | 1a4540e2-… | test@epam.com | pycharm       | us.anthropic.claude-sonnet-4-6 | arm64     | darwin  | 25.3.0     | claude-code  | 2.1.74          | 13:17:57       | 13:18:46  |
| 2    | 0.022156   | same       | same       | same          | same          | same                           | same      | same    | same       | same         | same            | 13:21:40       | 13:21:46  |
| 3    | 0.035810   | same       | same       | same          | same          | same                           | same      | same    | same       | same         | same            | 13:21:46       | 13:22:46  |

**Key observations:**

- `AggregationTemporality: Delta` — each push reports cost for that interval only; sum all pushes for session total
- `session.id` is stable for the lifetime of a Claude Code session
- `user.id` is a hashed identifier; `user.email` is set by `client/install.py`
- `terminal.type` reflects the IDE/terminal Claude Code is running in (e.g. `pycharm`, `vscode`)
- `token.usage` fans out into 4 series per session (one per token `type`); `active_time.total` into 2 (one per activity `type`); neither `active_time.total` nor
  `session.count` carry a `model` label
- The OTel Collector converts delta metrics to cumulative counters via `deltatocumulative` and pushes each sample directly to VictoriaMetrics — one write per
  push, no scrape duplication

## Production Recommendations

The default configuration is tuned for development. For production deployments (e.g., EKS), consider these additional changes:

### Health Extension

Add the `health_check` extension for Kubernetes liveness/readiness probes:

```yaml
extensions:
  health_check:
    endpoint: 0.0.0.0:13133

service:
  extensions: [ health_check ]
```

This exposes a health endpoint at `:13133/` that returns `200 OK` when the collector is ready.

### Log Verbosity

Change `service.telemetry.logs.level` from `debug` to `info` and remove the `debug` exporter to reduce log volume and CPU overhead:

```yaml
service:
  telemetry:
    logs:
      level: info
  pipelines:
    metrics:
      exporters: [ prometheusremotewrite ]   # remove 'debug'
```

### `max_stale` Tuning

The `deltatocumulative` processor evicts in-memory state for series not seen within `max_stale`. After eviction, the cumulative counter resets — `increase()`
handles this correctly, but `last_over_time()` queries will undercount.

| `max_stale`   | Memory (200 users)     | Idle tolerance | Risk                                                    |
|---------------|------------------------|----------------|---------------------------------------------------------|
| 30m           | ~Low                   | 30 min gaps    | Resets after short breaks; `last_over_time` undercounts |
| 2h            | ~Moderate              | 2h gaps        | Covers lunch breaks, most meetings                      |
| 6h            | ~Higher                | Half-day gaps  | Covers long focus blocks away from Claude Code          |
| 24h (default) | ~Highest (~tens of MB) | Full workday   | Only resets after overnight                             |

Memory impact per session is small (a few KB of counter state), so even 200 users × 24h is ~tens of MB.

## Tear Down

```bash
docker compose down -v   # -v removes persistent volumes
```

## Client Setup

```bash
# Install
python3 client/install.py your-email@example.com

# Uninstall
python3 client/uninstall.py
```
