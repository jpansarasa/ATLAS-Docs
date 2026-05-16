# Reports.Substrate

EF-backed `IObservationReader` implementation. Bridges `Reports.Core` (cadence-agnostic core) to `MacroSubstrate` (EF data layer).

## Overview

`Reports.Core` declares `IObservationReader` but holds no EF dependency. `Reports.Substrate` is the EF implementation — it consumes `MacroSubstrate`'s repository + DbContext, projects `MacroObservation` DTOs through the reader contract, and registers itself via DI. Splitting this from `Reports.Core` keeps the dependency direction one-way (Reports → MacroSubstrate) and lets unit tests stub the reader without instantiating EF.

## Architecture

```
Reports.Core
    │
    ▼
IObservationReader   ← contract (declared in Reports.Core)
    │
    ▼
Reports.Substrate.EfObservationReader          ← this package
    │
    ▼
MacroSubstrate.IMacroObservationRepository
    │
    ▼
MacroSubstrateDbContext → atlas_data.macro_observations
```

This package is the only place where Reports touches EF; `Reports.Core` stays pure and unit-testable with stub readers, and the cadence hosts depend on this package solely through `AddReportingSubstrate(...)`.

## Features

- **EF-backed `IObservationReader`** — reads `macro_observations` via the upstream `IMacroObservationRepository`, projects to the `MacroObservation` DTO carried by `Reports.Core`.
- **Two DI overloads** — `AddReportingSubstrate(configuration)` wires both `MacroSubstrate` and the reader; the parameterless overload registers only the reader for callers that already own `MacroSubstrate` registration.
- **Scoped lifetime alignment** — reader is `Scoped` to match `MacroSubstrateDbContext`; prevents capture-and-dispose bugs from a singleton holding a scoped context.

## DI Entry Points

```csharp
// Convenience: wires both MacroSubstrate and EfObservationReader.
services.AddReportingSubstrate(configuration);

// Reader-only — caller is responsible for already having registered MacroSubstrate.
services.AddReportingSubstrate();
```

`EfObservationReader` is registered as `Scoped` (matches the underlying `MacroSubstrateDbContext` lifetime).

## API Endpoints

N/A — library; no HTTP/gRPC surface. The reader is invoked in-process via the `IObservationReader` contract from `Reports.Core`'s composer.

## Ports

N/A — library; no listener. The DB connection is configured upstream by `MacroSubstrate`.

## Project Structure

```
Reports.Substrate/
  DependencyInjection.cs       — AddReportingSubstrate (two overloads)
  EfObservationReader.cs       — IObservationReader → IMacroObservationRepository
```

## Configuration

Inherits all configuration from `MacroSubstrate` (specifically `ConnectionStrings:AtlasData` and the optional `ConnectionStrings:SecMaster`). Nothing of its own.

## Development

Library only; built and tested as part of the consumer projects.

```bash
dotnet build Reports/src/Reports.Substrate/Reports.Substrate.csproj
dotnet test  Reports/tests/Reports.UnitTests/   # uses a stub IObservationReader
```

## Deployment

No deployment artifact. Referenced as a project reference by every report host.

## See Also

- [Reports.Core](../Reports.Core/README.md) — declares `IObservationReader`
- [MacroSubstrate](../../../MacroSubstrate/src/MacroSubstrate/README.md) — DbContext + repository this reader wraps
