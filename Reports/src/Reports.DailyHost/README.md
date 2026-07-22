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

## DECISIONS

D-1 runtime-log-level: INTENT a WARN-quiet-by-design report host must be raisable to Debug AT RUNTIME (on-demand deep debugging) with NO restart or redeploy, while steady state stays Warning (prod-quiet) and the worker's "next fire at..." Information line keeps showing via the appsettings `MinimumLevel:Override:ATLAS.Reports = Information` per-source override (independent of the global switch). A LoggingLevelSwitch is the SOLE level authority; the operator edits `Serilog:MinimumLevel:Default` in the RUNNING container and reloadOnChange picks it up live. Serilog.Settings.Configuration 8.0.4 does NOT re-apply MinimumLevel on IConfiguration reload, so an explicit ChangeToken re-parse is what makes the edit take effect; the MEL floor sits at Debug (replacing the prior `SetMinimumLevel(Information)`) so the bridge does not pre-filter below the switch. The raised level lives ONLY in the container's writable layer — revert by editing the key back to Warning (live) OR RECREATE (not restart) the container to reinstate the image's baked-in default; a restart PRESERVES the writable layer and does NOT revert. No compose `Serilog__*` env override for that key exists, so the JSON is authoritative and reloadable. / PRECOND operator runs `nerdctl exec reports-daily` to edit `/app/appsettings.json` `Serilog:MinimumLevel:Default`. / GUARD RuntimeLogLevel.BuildLogger (ControlledBy = sole authority) + RuntimeLogLevel.CreateSwitch (ChangeToken re-parse on reload) + RuntimeLogLevel.AddRuntimeControlledSerilog (MEL Debug floor) @ Telemetry/RuntimeLogLevel.cs / TEST RuntimeLogLevelTests.Runtime_appsettings_edit_raises_then_lowers_serilog_level

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
