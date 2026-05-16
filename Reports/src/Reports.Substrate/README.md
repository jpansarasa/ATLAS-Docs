# Reports.Substrate

EF-backed `IObservationReader` implementation. Bridges `Reports.Core` (cadence-agnostic core) to `MacroSubstrate` (EF data layer).

## Overview

`Reports.Core` declares `IObservationReader` but holds no EF dependency. `Reports.Substrate` is the EF implementation — it consumes `MacroSubstrate`'s repository + DbContext, projects `MacroObservation` DTOs through the reader contract, and registers itself via DI. Splitting this from `Reports.Core` keeps the dependency direction one-way (Reports → MacroSubstrate) and lets unit tests stub the reader without instantiating EF.

## DI Entry Points

```csharp
// Convenience: wires both MacroSubstrate and EfObservationReader.
services.AddReportingSubstrate(configuration);

// Reader-only — caller is responsible for already having registered MacroSubstrate.
services.AddReportingSubstrate();
```

`EfObservationReader` is registered as `Scoped` (matches the underlying `MacroSubstrateDbContext` lifetime).

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
