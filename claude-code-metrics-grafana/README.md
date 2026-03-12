# Claude Code Metrics — Prometheus & Grafana

Alternative to the AWS CloudWatch solution.

## Architecture

```
Claude Code (OTLP/HTTP JSON) → OTel Collector → Prometheus (scrape) → Grafana Dashboard
```

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

| Service        | Port | Purpose                             |
|----------------|------|-------------------------------------|
| OTel Collector | 4318 | Receives OTLP/HTTP from Claude Code |
| Prometheus     | 9090 | Scrapes and stores metrics          |
| Grafana        | 3000 | Dashboard (no login required)       |

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
- The same push also contains `claude_code.token.usage` (broken down by `input`/`output`/`cacheRead`/`cacheCreation`) and `claude_code.active_time.total` (
  broken down by `user`/`cli`)
- Prometheus scrapes the collector every ~15 seconds, so the metric is served many more times than it is pushed

## Known Limitations — Grafana Visualization

Delta temporality metrics don't map cleanly to Prometheus, which is designed for **monotonically increasing cumulative counters**. This creates several
challenges when visualizing `claude_code.cost.usage` (e.g. total session cost by user):

### Over-counting due to scrape/push frequency mismatch

Pushes arrive every **~1 minute** (delta), but Prometheus scrapes every **~15 seconds**. Each delta value (e.g. `$0.03`) is scraped **~4 times** between pushes.
Naive aggregation (`sum_over_time`, `increase`) will inflate totals by roughly 4x.

### Standard PromQL functions assume cumulative counters

`rate()` and `increase()` expect values that only go up. Delta metrics break this assumption:

- Exposed as a **gauge** — `sum_over_time()` over-counts (see above)
- Exposed as a **counter** — values aren't monotonic between pushes, so `increase()` misinterprets resets

### No clean PromQL for "total session cost by user"

Conceptually you want: sum all delta pushes per `session.id` per `user.email`. But PromQL has no native "sum only distinct pushes" or "deduplicate scrapes"
operation. The options are all flawed:

- `increase()` on a counter — unreliable with delta temporality and session boundaries
- `sum_over_time()` on a gauge — over-counts by the scrape/push ratio
- Manual scaling (e.g. divide by 4) — fragile, breaks if timing drifts

### Series staleness at session end

When a Claude Code session ends, pushes stop. After ~5 minutes Prometheus marks the series stale. This can cause `increase()` to miss or double-count the final
delta near the staleness boundary, and range queries spanning session end to produce unexpected gaps or jumps.

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
