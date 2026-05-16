# Reports.WeeklyHost

Weekly-cadence report host. Fires once per week (default: Monday 03:00 UTC), composes a multi-day market report, and writes Markdown + HTML to disk.

## Overview

`Reports.WeeklyHost` shares its shape with `Reports.DailyHost` and `Reports.MonthlyHost` — same `Reports.Core` composition, same `Reports.Hosting` publisher, same `Reports.Substrate` reader. The differences are confined to scheduling (`FireDayOfWeek` + `FireAtUtc`), the window resolver (`WeeklyReportWindowResolver`), and per-host telemetry adapters.

An on-demand HTTP trigger (`POST /api/reports/weekly/run`) is exposed on **localhost only** so SREs can regenerate a week after a substrate backfill.

## Architecture

```
Schedule (FireDayOfWeek + FireAtUtc)
       │
       ▼
WeeklyReportWorker  (IHostedService + singleton)
       │
       ├──> WeeklyReportWindowResolver  ──> CalendarService
       │
       ├──> IReportComposer (Reports.Core)
       │         └──> IObservationReader (Reports.Substrate → MacroSubstrate)
       │
       └──> IReportPublisher (Reports.Hosting → FileReportPublisher)
                   │
                   ▼
              /var/lib/atlas/reports/weekly/*.md + *.html
```

## Features

- **Scheduled weekly cycle** — fires on configured day of week at UTC time; `RunOnStartIfMissed` re-fires after a deploy.
- **Trading-week window resolver** — refines the previous-week window through `CalendarService`.
- **On-demand HTTP trigger** — `POST /api/reports/weekly/run`, localhost-only.
- **Markdown + HTML publish** every cycle.
- **Health endpoints** — `/health/live` + `/health/ready` (substrate DbContext check).
- **OTEL** — WeeklyReport spans + meter alongside library-level `ReportsActivitySource` and `MacroSubstrateTelemetry`.

## Configuration

| Key | Description | Default |
|---|---|---|
| `ConnectionStrings:AtlasData` | TimescaleDB connection | Required |
| `ConnectionStrings:SecMaster` | Cross-DB connection for versioned mapping lookups | Required for as-of reads |
| `Reports:Weekly:FireDayOfWeek` | Day of week (`Monday`..`Sunday`) | `Monday` |
| `Reports:Weekly:FireAtUtc` | Fire time (UTC, `HH:mm`) | `03:00` |
| `Reports:Weekly:OutputDirectory` | Host path for rendered files | `/var/lib/atlas/reports/weekly` |
| `Reports:Weekly:Formats` | Array of `ReportFormat` | `["Markdown", "Html"]` |
| `Reports:Weekly:RunOnStartIfMissed` | Re-fire on startup if this week's cycle hasn't run | `true` |
| `Reports:NewsSummary:Vllm:Endpoint` | vLLM base URL — opts into `VllmNewsSummaryGenerator` if present | Optional |
| `OpenTelemetry:OtlpEndpoint` | OTLP exporter target | `http://otel-collector:4317` |
| `OpenTelemetry:ServiceName` | OTEL `service.name` | `reports-weekly-host` |
| `OpenTelemetry:ServiceVersion` | OTEL `service.version` | `1.0.0` |

## API Endpoints

### REST API (Port 8080, container-internal)

| Method | Path | Description |
|---|---|---|
| `POST` | `/api/reports/weekly/run` | On-demand weekly cycle. Returns published file paths. **Localhost-only**. |

### Health Endpoints

- `GET /health/live` — liveness
- `GET /health/ready` — readiness (`macrosubstrate-db` check)

## Project Structure

```
Reports.WeeklyHost/
  Program.cs                                — DI wiring + endpoints + scheduler
  Containerfile                             — multi-stage → reports-weekly:latest
  appsettings.json                          — base configuration
  Endpoints/WeeklyReportEndpoints.cs        — on-demand HTTP trigger
  Scheduling/
    WeeklyReportSchedule.cs                 — schedule calculation
    WeeklyReportWindowResolver.cs           — calendar-aware window refinement
  Workers/WeeklyReportWorker.cs             — hosted service + on-demand RunOnceAsync
  Telemetry/
    WeeklyReportActivitySource.cs           — ActivitySource
    WeeklyPublisherTelemetry.cs             — IPublisherTelemetry adapter
```

## Development

```bash
bash Reports/.devcontainer/compile.sh           # build + test (all Reports projects)
```

Container image is built by `ansible-playbook playbooks/deploy.yml --tags reports-weekly`; the shared Reports devcontainer does not ship a `build.sh`.

## Deployment

- **Image:** `reports-weekly:latest`
- **Ansible tag:** `--tags reports-weekly`

```bash
ansible-playbook playbooks/deploy.yml --tags reports-weekly
```

## Ports

| Service | Host | Container |
|---|---|---|
| HTTP (on-demand + health) | (internal only) | 8080 |

## See Also

- [Reports.Core](../Reports.Core/README.md) — composer + renderers
- [Reports.Substrate](../Reports.Substrate/README.md) — EF reader
- [Reports.Hosting](../Reports.Hosting/README.md) — publisher
- [Reports.DailyHost](../Reports.DailyHost/README.md) | [Reports.MonthlyHost](../Reports.MonthlyHost/README.md)
- [CalendarService](../../../CalendarService/README.md)
