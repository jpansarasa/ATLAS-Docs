# Reports.Core

Section builders, renderers, and the report composer for the ATLAS reporting stack. Pure library — no EF, no Npgsql.

## Overview

`Reports.Core` is the cadence-agnostic core of the Reports stack. It owns the read contract (`IObservationReader`), the section builders (word cloud, sector radar, macro signal radar, news summary), the renderers (Markdown + HTML), and the `IReportComposer` that ties them together into a report. EF/Npgsql wiring lives in the sibling `Reports.Substrate` project; this library can be consumed by tests with a stub reader, by embedded scheduled jobs, or by any host that wants to compose a report without bringing the DB stack along.

Per Epic 6 §4, `Reports.Core` depends on the `MacroSubstrate.Client` DTO (and `AtlasSectorCode` via `Events.Client` transitively) — not on the EF substrate project.

## Architecture

```
Daily/Weekly/MonthlyHost
       │
       ▼
IReportComposer.ComposeAsync(window, cadence)  ← ReportComposer (Scoped)
       │
       ├──> IObservationReader.QueryAsync(...)  ← Reports.Substrate impl
       │                                          (or test stub)
       │
       ├──> WordCloudBuilder
       ├──> SectorRadarBuilder
       ├──> MacroSignalRadarBuilder
       └──> INewsSummaryGenerator
                  ├─ HeuristicNewsSummaryGenerator  (default)
                  └─ VllmNewsSummaryGenerator       (AddVllmNewsSummary)

       ▼
ReportSections → IReportRenderer  → markdown/html bytes
                    ├─ MarkdownReportRenderer
                    └─ HtmlReportRenderer
```

`ReportComposer` is `Scoped` because it transitively depends on `IObservationReader` (Scoped) → `MacroSubstrateDbContext` (Scoped). A `Singleton` composer would capture-and-dispose the `DbContext` on first run.

## Features

- **`IReportComposer`** — produces `ReportSections` for a `ReportWindow` + `ReportCadence`. Cadence-agnostic.
- **Section builders** — `WordCloudBuilder`, `SectorRadarBuilder`, `MacroSignalRadarBuilder`. Pure functions over observation streams.
- **News summary** — pluggable `INewsSummaryGenerator` with a heuristic default and an opt-in vLLM-backed implementation (`AddVllmNewsSummary`).
- **Renderers** — Markdown (canonical) and HTML (notification-friendly).
- **`ReportWindow.ForCadence(...)`** — cadence → window calculator (refined by host-side calendar resolvers for trading-day awareness).
- **OTEL surface** — `ReportsActivitySource` + `ReportsMeter` for library-level spans/metrics; hosts must register both.

## Configuration

`Reports.Core` itself reads only the optional vLLM news-summary section. Hosts own DB / output / cadence configuration.

| Key | Description | Default |
|---|---|---|
| `Reports:NewsSummary:Vllm:Endpoint` | vLLM base URL (presence of this section opts the host into `AddVllmNewsSummary`). Falls back to `HeuristicNewsSummaryGenerator` if absent. | None |

## DI Entry Points

```csharp
// Always-safe: builders, renderers, composer, heuristic generator.
services.AddReportingCore();

// Opt-in: replace heuristic with vLLM-backed generator.
if (config.GetSection("Reports:NewsSummary:Vllm:Endpoint").Exists())
    services.AddVllmNewsSummary(config);
```

## Project Structure

```
Reports.Core/
  DependencyInjection.cs    — AddReportingCore + AddVllmNewsSummary
  ReportCadence.cs          — Daily | Weekly | Monthly
  ReportWindow.cs           — window calculator (UTC, ForCadence)
  Builders/
    IReportComposer.cs
    ReportComposer.cs
    INewsSummaryGenerator.cs
    HeuristicNewsSummaryGenerator.cs
    VllmNewsSummaryGenerator.cs
    WordCloudBuilder.cs
    SectorRadarBuilder.cs
    MacroSignalRadarBuilder.cs
  Reading/IObservationReader.cs       — read contract (Substrate impl)
  Rendering/
    IReportRenderer.cs
    MarkdownReportRenderer.cs
    HtmlReportRenderer.cs
    ReportFormat.cs
  Sections/ReportSections.cs          — composed report payload
  Telemetry/
    ReportsActivitySource.cs          — ActivitySource name (host must AddSource)
    ReportsMeter.cs                   — Meter name (host must AddMeter)
```

## Development

Library only; built and tested as part of the consumer projects (`Reports.DailyHost`, `Reports.WeeklyHost`, `Reports.MonthlyHost`).

```bash
dotnet build Reports/src/Reports.Core/Reports.Core.csproj
dotnet test  Reports/tests/Reports.UnitTests/
```

`TreatWarningsAsErrors` is enabled — fix the warning, don't suppress it.

## Deployment

No deployment artifact. Referenced as a project reference by every report host.

## See Also

- [Reports.Hosting](../Reports.Hosting/README.md) — `IReportPublisher` + telemetry adapter contract
- [Reports.Substrate](../Reports.Substrate/README.md) — EF-backed `IObservationReader`
- [Reports.DailyHost](../Reports.DailyHost/README.md) | [Reports.WeeklyHost](../Reports.WeeklyHost/README.md) | [Reports.MonthlyHost](../Reports.MonthlyHost/README.md) — cadence-specific hosts
- [MacroSubstrate.Client](../../../MacroSubstrate/src/MacroSubstrate.Client/README.md) — DTO carried through the reader
