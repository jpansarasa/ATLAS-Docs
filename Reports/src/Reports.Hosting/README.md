# Reports.Hosting

Shared host scaffolding for the cadence-specific report hosts (`Reports.DailyHost`, `Reports.WeeklyHost`, `Reports.MonthlyHost`). Owns the `IReportPublisher` contract + the default file publisher.

## Overview

Reports hosts share more than they differ — they all publish rendered output to a file path on a mounted volume, all wrap publish with telemetry, and all differ only in cadence + window resolver. `Reports.Hosting` factors out the shared pieces so each host's `Program.cs` stays focused on cadence-specific wiring (schedule, window resolver, workers).

## Contract

### `IReportPublisher`

```csharp
Task<PublishedReport> PublishAsync(
    ReportSections sections,
    ReportFormat format,
    ReportWindow window,
    ReportCadence cadence,
    CancellationToken ct);
```

Returns a `PublishedReport` describing the file path + size that was written. `FileReportPublisher` is the only implementation today; alternatives (e.g. S3, NTFY-attachment) plug in via the same interface.

### `IPublisherTelemetry`

Each cadence host implements this interface so its `ActivitySource` / `Meter` receive the publish-span + counters. The publisher itself is cadence-agnostic.

## DI Entry Point

```csharp
// In the host's Program.cs, after AddReportingCore + AddReportingSubstrate:
services.AddFileReportPublisher<DailyPublisherTelemetry>();
// (or WeeklyPublisherTelemetry / MonthlyPublisherTelemetry)
```

Both `IPublisherTelemetry` and `IReportPublisher` are registered as `Singleton` — the publisher has no scoped state.

## Project Structure

```
Reports.Hosting/
  DependencyInjection.cs              — AddFileReportPublisher<TTelemetry>
  Publishing/
    IReportPublisher.cs               — contract
    IPublisherTelemetry.cs            — per-cadence telemetry adapter
    FileReportPublisher.cs            — writes rendered bytes to disk
```

## Configuration

This library has no configuration of its own; hosts own the output-directory option that `FileReportPublisher` reads through the cadence-specific options class.

## Development

Library only; built and tested as part of the consumer projects.

```bash
dotnet build Reports/src/Reports.Hosting/Reports.Hosting.csproj
```

## Deployment

No deployment artifact. Referenced as a project reference by every report host.

## See Also

- [Reports.Core](../Reports.Core/README.md) — composer + renderers consumed by every host
- [Reports.Substrate](../Reports.Substrate/README.md) — EF-backed reader
- [Reports.DailyHost](../Reports.DailyHost/README.md) | [Reports.WeeklyHost](../Reports.WeeklyHost/README.md) | [Reports.MonthlyHost](../Reports.MonthlyHost/README.md)
