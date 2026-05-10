# Events.EntityFrameworkCore

EF Core value-converters and persistence helpers for ATLAS event/identity types defined in `Events.Client`.

## Overview

`Events.EntityFrameworkCore` is a thin sibling package to `Events.Client`. It carries reusable EF Core wiring — currently the value-converters that round-trip the `AtlasSectorCode` enum through database string columns — so every writer of those types (SecMaster's `InstrumentEntity`, `InstrumentSectorOverrideEntity`, `NaicsSectorRollupEntity`; FredCollector's `SeriesConfig`; SentinelCollector's `ExtractedObservation`) shares a single converter implementation.

It lives in a separate package so `Events.Client` (the gRPC contracts and lightweight clients) stays EF-free for callers that only consume the wire types.

## Architecture

This package is a leaf in the dependency graph: it depends only on `Events.Client` (for the `AtlasSectorCode` enum) and `Microsoft.EntityFrameworkCore`. There is no runtime, no host, no IO — just static `ValueConverter<,>` instances consumed by downstream `DbContext` configurations.

```
SecMaster/FredCollector/SentinelCollector DbContext
    └─► Events.EntityFrameworkCore (value-converters)
            └─► Events.Client (AtlasSectorCode enum + Parse/ToDbString)
```

## Features

- **Single source of truth** for `AtlasSectorCode` ↔ DB-string conversion across every writer.
- **Nullable + non-nullable variants** matching real-world column nullability (instruments awaiting classification vs. NOT NULL overrides/rollups).
- **Loud-failure on schema drift** — unknown DB strings throw at the converter boundary instead of silently returning a default.
- **EF-free downstream** — `Events.Client` callers that don't touch EF take no transitive EF dependency.

## Usage

Reference the package alongside `Events.Client`:

```xml
<ProjectReference Include="../Events/src/Events.Client/Events.Client.csproj" />
<ProjectReference Include="../Events/src/Events.EntityFrameworkCore/Events.EntityFrameworkCore.csproj" />
```

Apply the converters in your `DbContext.OnModelCreating`:

```csharp
using ATLAS.Events.EntityFrameworkCore;

protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.Entity<InstrumentEntity>()
        .Property(e => e.AtlasSectorCode)
        .HasConversion(AtlasSectorCodeConverters.Nullable)
        .HasMaxLength(64);

    modelBuilder.Entity<InstrumentSectorOverrideEntity>()
        .Property(e => e.AtlasSectorCode)
        .HasConversion(AtlasSectorCodeConverters.NonNullable)
        .HasMaxLength(64);
}
```

Both converters call into `AtlasSectorCodes.Parse` on read, which throws `InvalidOperationException` on an unknown DB value. That loud-failure is intentional — the column is FK-bounded by `atlas_sectors`, so an unknown code means schema drift, not a normal value.

### Choosing a converter

| Converter | Use when |
|---|---|
| `AtlasSectorCodeConverters.Nullable` | column is nullable (instruments awaiting classification, FRED series with no sector tag, Sentinel observations resolved before classification cutover) |
| `AtlasSectorCodeConverters.NonNullable` | column is NOT NULL (overrides, NAICS rollups) |

## Configuration

This package has no runtime configuration — it ships static converters consumed at `OnModelCreating` time. Column max-length, nullability, and FK-binding are owned by the consumer's entity configuration; this package only deals with the read/write transform.

## API Endpoints

This is a library, not a service. It exposes no HTTP/gRPC surface. The public API is the static `AtlasSectorCodeConverters` class (`Nullable` and `NonNullable` `ValueConverter<,>` fields).

## Project Structure

```
Events.EntityFrameworkCore/
├── AtlasSectorCodeConverters.cs   # ValueConverter<AtlasSectorCode, string> (nullable + non-nullable)
└── Events.EntityFrameworkCore.csproj
```

## Development

### Prerequisites

- .NET 10 SDK (or via devcontainer)
- `Microsoft.EntityFrameworkCore` 10.0.0 (referenced by package)
- `Events.Client` (referenced by package)

### Building

The package builds as part of the `Events` solution and is republished into `nupkg/` when downstream services need a fresh local feed entry. There is no standalone `compile.sh` — building any consumer (e.g. `SecMaster/.devcontainer/compile.sh`) is sufficient.

## Deployment

Library only — not deployed independently. Consumers (SecMaster, FredCollector, SentinelCollector) reference this project and ship it inside their own container image. There is no ansible tag for this package.

## Ports

None — library, no listening sockets.

## See Also

- [Events](../Events/) — in-memory event records (`IEvent`, `ObservationCollectedEvent`, `ThresholdCrossedEvent`, `RegimeTransitionEvent`)
- [Events.Client](../Events.Client/) — gRPC client libraries and wire types (including `AtlasSectorCode`)
- [Events](../../README.md) — top-level package overview, gRPC contracts, versioning policy
- [SecMaster](../../../SecMaster/) — primary consumer (instruments, overrides, NAICS rollups)
- [FredCollector](../../../FredCollector/) — series-level sector tagging
- [SentinelCollector](../../../SentinelCollector/) — extracted-observation sector tagging
