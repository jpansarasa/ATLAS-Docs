# Reports.MonthlyHost

Monthly-cadence report host. Fires once per month (default: 1st at 04:00 UTC), composes a month-window market report, and writes Markdown + HTML to disk.

## Overview

`Reports.MonthlyHost` shares its shape with `Reports.DailyHost` and `Reports.WeeklyHost` — same `Reports.Core` composition, same `Reports.Hosting` publisher, same `Reports.Substrate` reader. The differences are confined to scheduling (`FireDayOfMonth` + `FireAtUtc`), the window resolver (`MonthlyReportWindowResolver`), and per-host telemetry adapters.

An on-demand HTTP trigger (`POST /api/reports/monthly/run`) is exposed on **localhost only** so SREs can regenerate a month after a substrate backfill.

## Architecture

```
Schedule (FireDayOfMonth + FireAtUtc)
       │
       ▼
MonthlyReportWorker  (IHostedService + singleton)
       │
       ├──> MonthlyReportWindowResolver  ──> CalendarService
       │
       ├──> IReportComposer (Reports.Core)
       │         └──> IObservationReader (Reports.Substrate → MacroSubstrate)
       │
       └──> IReportPublisher (Reports.Hosting → FileReportPublisher)
                   │
                   ▼
              /var/lib/atlas/reports/monthly/*.md + *.html
```

## Features

- **Scheduled monthly cycle** — fires on configured day of month at UTC time; `RunOnStartIfMissed` re-fires after a deploy.
- **Trading-month window resolver** — refines the previous-month window through `CalendarService`.
- **On-demand HTTP trigger** — `POST /api/reports/monthly/run`, localhost-only.
- **Markdown + HTML publish** every cycle.
- **Health endpoints** — `/health/live` + `/health/ready` (substrate DbContext check).
- **OTEL** — MonthlyReport spans + meter alongside library-level `ReportsActivitySource` and `MacroSubstrateTelemetry`.

## Configuration

| Key | Description | Default |
|---|---|---|
| `ConnectionStrings:AtlasData` | TimescaleDB connection | Required |
| `ConnectionStrings:SecMaster` | Cross-DB connection for versioned mapping lookups | Required for as-of reads |
| `Reports:Monthly:FireDayOfMonth` | Day of month (`1`–`28` — avoid 29–31 for month-length safety) | `1` |
| `Reports:Monthly:FireAtUtc` | Fire time (UTC, `HH:mm`) | `04:00` |
| `Reports:Monthly:OutputDirectory` | Host path for rendered files | `/var/lib/atlas/reports/monthly` |
| `Reports:Monthly:Formats` | Array of `ReportFormat` | `["Markdown", "Html"]` |
| `Reports:Monthly:RunOnStartIfMissed` | Re-fire on startup if this month's cycle hasn't run | `true` |
| `Reports:NewsSummary:Vllm:Endpoint` | vLLM base URL — opts into `VllmNewsSummaryGenerator` if present | Optional |
| `OpenTelemetry:OtlpEndpoint` | OTLP exporter target | `http://otel-collector:4317` |
| `OpenTelemetry:ServiceName` | OTEL `service.name` | `reports-monthly-host` |
| `OpenTelemetry:ServiceVersion` | OTEL `service.version` | `1.0.0` |

## API Endpoints

### REST API (Port 8080, container-internal)

| Method | Path | Description |
|---|---|---|
| `POST` | `/api/reports/monthly/run` | On-demand monthly cycle. Returns published file paths. **Localhost-only**. |

### Health Endpoints

- `GET /health/live` — liveness
- `GET /health/ready` — readiness (`macrosubstrate-db` check)

## Project Structure

```
Reports.MonthlyHost/
  Program.cs                                 — DI wiring + endpoints + scheduler
  Containerfile                              — multi-stage → reports-monthly:latest
  appsettings.json                           — base configuration
  Endpoints/MonthlyReportEndpoints.cs        — on-demand HTTP trigger
  Scheduling/
    MonthlyReportSchedule.cs                 — schedule calculation
    MonthlyReportWindowResolver.cs           — calendar-aware window refinement
  Workers/MonthlyReportWorker.cs             — hosted service + on-demand RunOnceAsync
  Telemetry/
    MonthlyReportActivitySource.cs           — ActivitySource
    MonthlyPublisherTelemetry.cs             — IPublisherTelemetry adapter
```

## Development

```bash
bash Reports/.devcontainer/compile.sh           # build + test (all Reports projects)
```

Container image is built by `ansible-playbook playbooks/deploy.yml --tags reports-monthly`; the shared Reports devcontainer does not ship a `build.sh`.

## Deployment

- **Image:** `reports-monthly:latest`
- **Ansible tag:** `--tags reports-monthly`

```bash
ansible-playbook playbooks/deploy.yml --tags reports-monthly
```

## Ports

| Service | Host | Container |
|---|---|---|
| HTTP (on-demand + health) | (internal only) | 8080 |

## See Also

- [Reports.Core](../Reports.Core/README.md) — composer + renderers
- [Reports.Substrate](../Reports.Substrate/README.md) — EF reader
- [Reports.Hosting](../Reports.Hosting/README.md) — publisher
- [Reports.DailyHost](../Reports.DailyHost/README.md) | [Reports.WeeklyHost](../Reports.WeeklyHost/README.md)
- [CalendarService](../../../CalendarService/README.md)
