# Events

In-memory C# event records for ATLAS intra-process buses (`System.Threading.Channels`), plus the canonical Protobuf contracts shipped with the package.

## Overview

`Events` is the lightweight, EF-free, gRPC-free core of the ATLAS event taxonomy. It defines:

- The `IEvent` marker interface that all ATLAS events implement.
- Three in-memory event records — `ObservationCollectedEvent`, `ThresholdCrossedEvent`, `RegimeTransitionEvent` — used over `System.Threading.Channels<T>` for intra-process pub/sub.
- The `.proto` files (`observation_events.proto`, `secmaster.proto`) that define the wire contracts. Generated stubs are produced by `Events.Client` (which references this package's protos via build-time copy).

Use this package when a service needs to publish or consume events on its own internal channel without dragging in gRPC client dependencies.

## Architecture

This package is a leaf — it has no package references beyond the .NET 10 BCL. Its outputs are consumed two ways:

```
.cs records ─► consumed directly by services for in-process IEvent over Channels
Protos/*.proto ─► consumed by Events.Client which generates gRPC stubs
```

The protos live here (not in `Events.Client`) so a future second client wrapper can also consume them without circular references.

## Features

- **Marker interface (`IEvent`)** for typed channel buses.
- **Three event records** — observation collected, threshold crossed, regime transition — covering the ATLAS data → eval → alert path.
- **Canonical proto sources** — single home for `observation_events.proto` and `secmaster.proto`.
- **Zero runtime dependencies** — pure C# records, no DI, no IO, no gRPC tooling.

## Usage

```xml
<ProjectReference Include="../Events/src/Events/Events.csproj" />
```

```csharp
using ATLAS.Events;
using System.Threading.Channels;

// Publisher (e.g. ThresholdEngine)
var channel = Channel.CreateUnbounded<IEvent>();
await channel.Writer.WriteAsync(new ThresholdCrossedEvent(
    PatternId: "vix-deployment-l1",
    PatternName: "VIX Level 1 Deployment Trigger",
    Category: "Liquidity",
    Signal: 1.0m,
    Confidence: 0.92m,
    CurrentValue: 28.4m,
    Threshold: 22m,
    Metadata: new() { ["frequency"] = "daily" },
    EvaluatedAt: DateTimeOffset.UtcNow));

// Consumer (e.g. AlertService)
await foreach (var evt in channel.Reader.ReadAllAsync())
{
    if (evt is ThresholdCrossedEvent t) HandleThreshold(t);
}
```

### Event types

| Record | Publisher | Consumer | Purpose |
|---|---|---|---|
| `ObservationCollectedEvent` | Collectors | ThresholdEngine | New observation persisted |
| `ThresholdCrossedEvent` | ThresholdEngine | AlertService | Pattern triggered |
| `RegimeTransitionEvent` | ThresholdEngine | AlertService, AnalysisEngine | Macro regime shifted |

## Configuration

No runtime configuration — pure record types. Channel capacity, bounded vs. unbounded, single-reader/writer hints, etc. are owned by the host that creates the channel.

## API Endpoints

This is a library, not a service. It exposes no HTTP/gRPC surface. The public API is `IEvent` and the three event records.

## Project Structure

```
Events/
├── IEvent.cs                          # Marker interface
├── ObservationCollectedEvent.cs       # In-memory observation event
├── ThresholdCrossedEvent.cs           # In-memory threshold-cross event
├── RegimeTransitionEvent.cs           # In-memory regime-transition event
├── Protos/
│   ├── observation_events.proto       # gRPC contract (consumed by Events.Client)
│   └── secmaster.proto                # gRPC contract (consumed by Events.Client)
└── Events.csproj                      # NuGet: Events 1.0.0
```

## Development

### Prerequisites

- .NET 10 SDK (or via devcontainer)

### Building

```bash
dotnet build Events/src/Events/Events.csproj
```

The proto files in `Protos/` are not compiled here — they are consumed by `Events.Client` which sets `<Protobuf>` items with `GrpcServices="Both"`. This package stays free of gRPC tooling so callers that only want in-memory records pull no transitive gRPC dependencies.

## Deployment

Library only — not deployed independently. Consumers reference this project and ship it inside their own container image. There is no ansible tag for this package. Published to the local `nupkg/` feed when a downstream service needs a fresh version pin.

## Ports

None — library, no listening sockets.

## Versioning

- **Package id:** `Events`, version `1.0.0`.
- **Forward compatible:** new fields go on the end (positional records require care; prefer adding via `init` properties or new record types).
- **Backward compatible:** never remove or rename fields without a new record type.

## See Also

- [Events](../../README.md) — top-level overview, gRPC contracts, full event matrix
- [Events.Client](../Events.Client/) — gRPC client libraries built on these protos
- [Events.EntityFrameworkCore](../Events.EntityFrameworkCore/) — EF value-converters for shared identity types
- [ThresholdEngine](../../../ThresholdEngine/) — main publisher of threshold/regime events
- [AlertService](../../../AlertService/) — consumer of threshold/regime events
