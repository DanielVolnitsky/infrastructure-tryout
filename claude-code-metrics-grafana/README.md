# Claude Code Metrics — VictoriaMetrics & Grafana

Alternative to the AWS CloudWatch solution.

## Architecture

```
Claude Code (OTLP/HTTP JSON) → OTel Collector → VictoriaMetrics (remote write) → Grafana Dashboard
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
- The OTel Collector converts delta metrics to cumulative counters via `deltatocumulative` and pushes each sample directly to VictoriaMetrics — one write per push, no scrape duplication

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
