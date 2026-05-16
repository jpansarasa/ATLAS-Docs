# Reports.DailyHost

Daily-cadence report host. Runs once per UTC day, composes a market-day report from `macro_observations`, and writes Markdown + HTML to disk.

## Overview

`Reports.DailyHost` is one of three cadence-specific hosts (alongside `Reports.WeeklyHost` and `Reports.MonthlyHost`) that share `Reports.Core` (composition + rendering) and `Reports.Hosting` (publishing). It fires at `Reports:Daily:FireAtUtc` (default `02:00`), resolves the previous-trading-day window via the calendar service, composes a report through `IReportComposer`, and publishes Markdown + HTML files to a mounted host volume.

An on-demand HTTP trigger (`POST /api/reports/daily/run`) is exposed on **localhost only** so SREs can regenerate yesterday's report after a substrate backfill without waiting for the next fire.

## Architecture

```
Schedule (FireAtUtc)
       │
       ▼
DailyReportWorker  (IHostedService + singleton — also resolvable for on-demand)
       │
       ├──> TradingDayWindowResolver  ──> CalendarService
       │
       ├──> IReportComposer (Reports.Core)
       │         └──> IObservationReader (Reports.Substrate → MacroSubstrate)
       │
       └──> IReportPublisher (Reports.Hosting → FileReportPublisher)
                   │
                   ▼
              /var/lib/atlas/reports/daily/*.md + *.html
```

`ValidateScopes = true` + `ValidateOnBuild = true` is forced in every environment so a regression that registers a Scoped dependency as Singleton fails at container build instead of silently breaking on cycle 2.

## Features

- **Scheduled daily cycle** — fires at configured UTC time; `RunOnStartIfMissed` re-runs the missed window after a deploy.
- **Trading-day window resolver** — refines `ReportWindow.ForCadence`'s previous-day approximation via `CalendarService.AddMarketCalendar`.
- **On-demand HTTP trigger** — `POST /api/reports/daily/run`, localhost-only, returns the published file paths. Side-channel; scheduled cycle remains source of truth.
- **Markdown + HTML publish** — both formats written every cycle.
- **Health endpoints** — `/health/live` (worker exists) + `/health/ready` (substrate reachable).
- **OTEL** — DailyReport spans + meter, plus `ReportsActivitySource` and `MacroSubstrateTelemetry` (library spans), all flowing to OTLP.

## Configuration

| Key | Description | Default |
|---|---|---|
| `ConnectionStrings:AtlasData` | TimescaleDB connection — primary observation store | Required |
| `ConnectionStrings:SecMaster` | Cross-DB connection for versioned mapping lookups | Required for as-of reads |
| `Reports:Daily:FireAtUtc` | Daily fire time (UTC, `HH:mm`) | `02:00` |
| `Reports:Daily:OutputDirectory` | Host path for rendered files (mounted into container) | `/var/lib/atlas/reports/daily` |
| `Reports:Daily:Formats` | Array of `ReportFormat` (`Markdown`, `Html`) | `["Markdown", "Html"]` |
| `Reports:Daily:RunOnStartIfMissed` | Re-fire on startup if today's cycle hasn't run | `true` |
| `Reports:NewsSummary:Vllm:Endpoint` | vLLM base URL — presence opts the host into `VllmNewsSummaryGenerator`; absent falls back to heuristic | Optional |
| `OpenTelemetry:OtlpEndpoint` | OTLP exporter target | `http://otel-collector:4317` |
| `OpenTelemetry:ServiceName` | OTEL `service.name` | `reports-daily-host` |
| `OpenTelemetry:ServiceVersion` | OTEL `service.version` | `1.0.0` |

## API Endpoints

### REST API (Port 8080, container-internal)

| Method | Path | Description |
|---|---|---|
| `POST` | `/api/reports/daily/run` | On-demand daily cycle. Returns published file paths. **Localhost-only** via `RequireHost("localhost", "127.0.0.1", "[::1]")` — call via `nerdctl exec ... curl localhost:8080/...`. |

### Health Endpoints

- `GET /health/live` — liveness
- `GET /health/ready` — readiness (`macrosubstrate-db` DbContext check)

## Project Structure

```
Reports.DailyHost/
  Program.cs                              — DI wiring + endpoints + scheduler
  Containerfile                           — multi-stage → reports-daily:latest
  appsettings.json                        — base configuration (env overrides win)
  Endpoints/DailyReportEndpoints.cs       — on-demand HTTP trigger
  Scheduling/
    DailyReportSchedule.cs                — schedule calculation
    TradingDayWindowResolver.cs           — calendar-aware window refinement
  Workers/DailyReportWorker.cs            — hosted service + on-demand RunOnceAsync
  Telemetry/
    DailyReportActivitySource.cs          — ActivitySource for daily-host spans
    DailyPublisherTelemetry.cs            — IPublisherTelemetry adapter
    DailyReportMeter.cs                   — Meter for daily-host counters
```

## Development

```bash
bash Reports/.devcontainer/compile.sh           # build + test (all Reports projects)
```

Container image is built by `ansible-playbook playbooks/deploy.yml --tags reports-daily` (which invokes `nerdctl build` against `Containerfile`); the shared Reports devcontainer does not ship a `build.sh`.

Per CLAUDE.md `VERIFY [before_commit]`: run `compile.sh` before pushing.

## Deployment

- **Image:** `reports-daily:latest`
- **Ansible tag:** `--tags reports-daily`
- **Compose service:** `reports-daily`, depends on `timescaledb` + `migrate-macro-substrate`.

```bash
ansible-playbook playbooks/deploy.yml --tags reports-daily
```

Per CLAUDE.md `DEPLOYMENT [HARD_STOP]`: never edit `/opt/ai-inference/compose.yaml` directly.

## Ports

| Service | Host | Container |
|---|---|---|
| HTTP (on-demand + health) | (internal only) | 8080 |

## See Also

- [Reports.Core](../Reports.Core/README.md) — composer + renderers
- [Reports.Substrate](../Reports.Substrate/README.md) — EF reader
- [Reports.Hosting](../Reports.Hosting/README.md) — `FileReportPublisher`
- [Reports.WeeklyHost](../Reports.WeeklyHost/README.md) | [Reports.MonthlyHost](../Reports.MonthlyHost/README.md)
- [CalendarService](../../../CalendarService/README.md) — trading-day calendar
