# Events.Client

gRPC client libraries and shared identity types for ATLAS event streaming and SecMaster integration.

## Overview

`Events.Client` carries the auto-generated gRPC stubs (from `../Events/Protos/*.proto`, with `GrpcServices="Both"` so server-side base classes are also generated) plus thin, opinionated client wrappers and a small set of cross-cutting types (`AtlasSectorCode`, `Observation`) that callers exchange over both gRPC and EF persistence.

Use this package whenever a service needs to:

- Subscribe to / publish observation event streams (`ObservationEventClient`).
- Register series with SecMaster (`SecMasterRegistryClient`).
- Resolve symbols against SecMaster (`SecMasterResolverClient`).
- Persist or transmit an `AtlasSectorCode` and round-trip it through the SecMaster `atlas_sectors` table.

## Architecture

```
Events/Protos/*.proto ──► Grpc.Tools (build-time)
                              │
                              ▼
              generated C# stubs (client + server)
                              │
                              ▼
       hand-rolled wrappers (ObservationEventClient,
       SecMasterRegistryClient, SecMasterResolverClient)
                              │
                              ▼
           DI registration (ServiceCollectionExtensions)
                              │
                              ▼
                  consumed by services + tests
```

`Events.Client` references `Events/Protos/*.proto` via `<Protobuf>` items with `GrpcServices="Both"`, so consumers can implement either side of the wire from this same package. Identity types (`AtlasSectorCode`, `Observation`) live here so they can be exchanged on both gRPC and downstream persistence (EF) without an extra dependency.

## Features

- **Auto-generated client + server stubs** for `ObservationEventStream`, `SecMasterRegistry`, `SecMasterResolver`.
- **Opinionated client wrappers** with sensible defaults (keepalive, retry, deadline propagation).
- **Cross-cutting identity types** — `AtlasSectorCode` (11-sector ATLAS taxonomy) and `Observation` (flattened observation DTO).
- **DI helpers** — `services.AddObservationEventClient(...)`, `AddSecMasterRegistryClient(...)`, `AddSecMasterResolverClient(...)`.
- **Server-side compatibility** — `GrpcServices="Both"` means SecMaster and collectors implement their wire contracts from the same generated base classes.

## Usage

```xml
<ProjectReference Include="../Events/src/Events.Client/Events.Client.csproj" />
```

Register clients via DI:

```csharp
services.AddObservationEventClient(opt => opt.Endpoint = "http://fred-collector:5001");
services.AddSecMasterRegistryClient(opt => opt.Endpoint = "http://secmaster:8080");
services.AddSecMasterResolverClient(opt => opt.Endpoint = "http://secmaster:8080");
```

Then inject and use:

```csharp
public class MyConsumer(ObservationEventClient events, SecMasterResolverClient resolver)
{
    public async Task Run(CancellationToken ct)
    {
        var resolved = await resolver.ResolveSymbolAsync("VIXCLS", ct);
        await foreach (var obs in events.SubscribeAsync(resolved.SourceId, ct))
        {
            // ...
        }
    }
}
```

### Identity types

| Type | Purpose |
|---|---|
| `AtlasSectorCode` | 11-sector ATLAS taxonomy enum. DB form via `AtlasSectorCodes.ToDbString` / parse via `AtlasSectorCodes.Parse` / `TryParse`. Persisted by SecMaster (`InstrumentEntity`, overrides, NAICS rollups), FredCollector (`SeriesConfig`), and SentinelCollector (`ExtractedObservation`). |
| `Observation` | Flattened observation record (from gRPC `SeriesCollectedEvent` / `OhlcvCollectedEvent`) for downstream consumers that don't want to depend on the raw protobuf message types. |

### Generated gRPC stubs

| Proto | Services (client + server stubs) |
|---|---|
| `observation_events.proto` | `ObservationEventStream` |
| `secmaster.proto` | `SecMasterRegistry`, `SecMasterResolver` |

`Events.Client` ships both client and server stubs (`GrpcServices="Both"`), so collectors implement `ObservationEventStreamBase` and SecMaster implements `SecMasterRegistryBase` / `SecMasterResolverBase` from this same package.

## Configuration

Configuration is per-DI-registration via the options callback passed to `services.Add*Client(...)`. Each client wrapper exposes:

| Option | Description | Default |
|---|---|---|
| `Endpoint` | Target gRPC URL (e.g. `http://secmaster:8080`) | required |
| `KeepAlive` | Keepalive ping interval / timeout | sensible defaults |
| `Retry` | Whether to install the default retry policy | `true` |

This is a library — no env vars, no config files. The host service's environment-bound options (e.g. `SECMASTER_GRPC_ENDPOINT`) are passed in by the consumer's `Startup` / `Program.cs`.

## API Endpoints

Library, not a service. The package's public API is the three client wrappers (`ObservationEventClient`, `SecMasterRegistryClient`, `SecMasterResolverClient`), the generated server-base classes, and the identity types. The HTTP/gRPC endpoints those clients hit live on the consumed services (SecMaster, collectors) — see those services' READMEs for the wire surface.

## Project Structure

```
Events.Client/
├── Observation.cs                     # Flattened observation DTO
├── ObservationEventClient.cs          # gRPC client wrapper (subscribe/query)
├── SecMasterRegistryClient.cs         # gRPC client wrapper (register series)
├── SecMasterResolverClient.cs         # gRPC client wrapper (resolve symbol/source)
├── ServiceCollectionExtensions.cs     # DI registration helpers
├── AtlasSectorCode.cs                 # 11-sector enum + DB string converters
└── Events.Client.csproj               # consumes ../Events/Protos/*.proto
```

## Development

### Prerequisites

- .NET 10 SDK (or via devcontainer)
- `Grpc.Tools` (already a `PrivateAssets="All"` package reference)

### Building

gRPC stubs are auto-generated at build:

```bash
dotnet build Events/src/Events.Client/Events.Client.csproj
```

### Implementing a server

```csharp
public class MyEventStream : ObservationEventStream.ObservationEventStreamBase
{
    public override async Task Subscribe(
        SubscribeRequest request,
        IServerStreamWriter<SeriesCollectedEvent> stream,
        ServerCallContext context)
    {
        // ...
    }
}
```

## Deployment

Library only — not deployed independently. Consumers reference this project and ship it inside their own container image. There is no ansible tag for this package. Published to the local `nupkg/` feed when a downstream service needs a fresh version pin.

## Ports

None — library, no listening sockets. Consumed services bind their own ports:

- SecMaster: `8080` (HTTP/gRPC) — see [SecMaster](../../../SecMaster/README.md#ports).
- Collectors: `5001` (gRPC event stream) — see each collector's README.

## See Also

- [Events](../../README.md) — top-level overview and event matrix
- [Events](../Events/) — in-memory `IEvent` records (no gRPC dependency)
- [Events.EntityFrameworkCore](../Events.EntityFrameworkCore/) — EF value-converters for `AtlasSectorCode`
- [SecMaster](../../../SecMaster/) — owner of `secmaster.proto` server side
- [FredCollector](../../../FredCollector/) — observation event publisher and `AtlasSectorCode` consumer
- [ThresholdEngine](../../../ThresholdEngine/) — primary subscriber to observation streams
