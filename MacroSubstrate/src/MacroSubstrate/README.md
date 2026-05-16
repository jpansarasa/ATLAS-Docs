# MacroSubstrate

EF-backed substrate library for the `macro_observations` hypertable — the canonical write target for every collector that emits macro data.

## Overview

`MacroSubstrate` is the EF Core implementation of the `IMacroObservationWriter` contract (defined in the sibling `MacroSubstrate.Client` package). Collectors take the writer via DI; this library wires up the `DbContext`, the repository, and the `IMappingVersionLookup` against the shared `atlas_data` connection so the writer is fully resolved without any per-collector EF wiring.

The library is intentionally optional: collectors that want to stay EF-free can implement `IMacroObservationWriter` themselves against a raw-INSERT path and skip this package entirely. The `MacroSubstrate.Migrator` console host owns schema deployment (Variant A per owner-rec §3.1.1) so no single collector owns the table.

## Architecture

```
Collector (DI)
    │
    ▼
IMacroObservationWriter   ← contract from MacroSubstrate.Client
    │
    ▼
MacroObservationRepository
    │
    ▼
MacroSubstrateDbContext   ← Npgsql + EnableRetryOnFailure
    │
    ▼
atlas_data.macro_observations (TimescaleDB hypertable)

         + (read paths only)
         ▼
IMappingVersionLookup     ← reads atlas_secmaster.mapping_versions
                            via NpgsqlMappingVersionLookup
```

Telemetry contract: spans + a failure counter under names exposed at `MacroSubstrateTelemetry.ActivitySourceName` and `MacroSubstrateTelemetry.MeterName`. Consumers MUST register both in their OpenTelemetry pipeline or the telemetry is silently dropped.

## Features

- **EF writer for `macro_observations`** — implements `IMacroObservationWriter` with single + bulk write paths and idempotency-key dedup (`source_collector`, `source_id`, `observation_time`).
- **Versioned mapping-lookup** — `IMappingVersionLookup` reads `atlas_secmaster.mapping_versions` so historical reads can resolve under the original rollup label. Cross-DB by design.
- **Retry-on-failure** — Npgsql transient-error retry baked into the `DbContextOptionsBuilder` (5x, 10s backoff).
- **Bootstrap warning if SecMaster connection absent** — registration succeeds but logs a structured warning; write paths still work, read paths throw `MappingVersionLookupUnavailableException` if invoked.

## Configuration

| Key | Description | Default |
|---|---|---|
| `ConnectionStrings:AtlasData` | Primary connection string for `macro_observations`. Falls back to `AtlasDb` for collectors that already use that name. | Required |
| `ConnectionStrings:SecMaster` | Cross-DB connection to `atlas_secmaster` for versioned mapping lookups. Falls back to `AtlasSecMaster`. | Optional (warns at startup if missing) |

## API Endpoints

N/A — library; no HTTP/gRPC surface of its own. The repository is invoked in-process via the `IMacroObservationWriter` contract (see [`MacroSubstrate.Client`](../MacroSubstrate.Client/README.md)). Schema deployment runs through the sibling [`MacroSubstrate.Migrator`](../MacroSubstrate.Migrator/README.md) console host.

## Ports

N/A — library; no listener. Consumers own their HTTP/gRPC surface.

## Project Structure

```
MacroSubstrate/
  src/MacroSubstrate/
    DependencyInjection.cs                  — AddMacroSubstrate() extension
    Data/
      MacroSubstrateDbContext.cs            — single DbSet<MacroObservationEntity>
      MacroSubstrateDbContextFactory.cs     — design-time factory for dotnet-ef
      Entities/MacroObservationEntity.cs    — EF row shape
      Configurations/                       — EF fluent-API mapping
      Repositories/MacroObservationRepository.cs — IMacroObservationWriter impl
      Migrations/                           — EF migrations (baseline + future)
    Versioning/
      IMappingVersionLookup.cs              — cross-DB version-label resolver
      NpgsqlMappingVersionLookup.cs         — raw Npgsql impl (avoids second DbContext)
    Telemetry/MacroSubstrateTelemetry.cs    — ActivitySource + Meter names
```

## Development

### Prerequisites

- .NET 10 SDK (or via the parent project's devcontainer)
- Connection to TimescaleDB (`atlas_data`) for runtime; SecMaster (`atlas_secmaster`) optional for versioned reads

### Building

This project is a library — built and tested as part of the consumer projects (`MacroSubstrate.Migrator`, `Reports.*`, etc.). There is no standalone devcontainer.

```bash
dotnet build MacroSubstrate/src/MacroSubstrate/MacroSubstrate.csproj
dotnet test MacroSubstrate/tests/
```

### Adding Migrations

Per CLAUDE.md `MIGRATIONS [HARD_STOP]` — always use the EF CLI; never hand-write `.cs` migration files.

```bash
dotnet ef migrations add {Name} \
  --project MacroSubstrate/src/MacroSubstrate \
  --startup-project MacroSubstrate/src/MacroSubstrate.Migrator
```

The migrator host (`MacroSubstrate.Migrator`) applies them at deploy time.

## Deployment

This is a library — it has no deployment of its own. Schema deployment is owned by the sibling `MacroSubstrate.Migrator` console host, which runs as a one-shot `migrate-macro-substrate` compose service ahead of any dependent collector.

## See Also

- [MacroSubstrate.Client](../MacroSubstrate.Client/README.md) — contract + DTO (depend on this from EF-free collectors)
- [MacroSubstrate.Migrator](../MacroSubstrate.Migrator/README.md) — one-shot migration runner
- [SecMaster](../../../SecMaster/README.md) — owner of `mapping_versions` (cross-DB read target)
- [Reports.Substrate](../../../Reports/src/Reports.Substrate/README.md) — primary read-side consumer
