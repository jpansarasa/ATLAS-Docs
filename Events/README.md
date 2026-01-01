# Events

Shared gRPC event contracts and in-memory event definitions for ATLAS microservices.

## Overview

Events is the schema registry for ATLAS. It defines gRPC Protobuf contracts for inter-service streaming and C# event records for internal event buses. All collectors stream events using the same gRPC interface, and all consumers subscribe using the same client libraries.

## Features

- **gRPC Event Streaming** - Protobuf-based contracts for real-time data collection events between services
- **SecMaster Integration** - Registration and resolution services for symbol/instrument management
- **In-Memory Events** - C# record types for internal event buses using System.Threading.Channels
- **Client Libraries** - Ready-to-use gRPC clients with retry logic and DI integration
- **Auto-Generated Code** - Grpc.Tools generates C# stubs from proto files during build

## Project Structure

```
Events/
├── src/
│   ├── Events/                     # Core contracts
│   │   ├── Protos/
│   │   │   ├── observation_events.proto   # Data collection event contracts
│   │   │   └── secmaster.proto            # SecMaster registration/resolution
│   │   ├── IEvent.cs                      # Base event interface
│   │   ├── ObservationCollectedEvent.cs   # In-memory observation event
│   │   ├── ThresholdCrossedEvent.cs       # In-memory threshold event
│   │   └── RegimeTransitionEvent.cs       # In-memory regime event
│   └── Events.Client/              # gRPC client libraries
│       ├── ObservationEventClient.cs      # Event stream subscription client
│       ├── SecMasterRegistryClient.cs     # Series registration client
│       ├── SecMasterResolverClient.cs     # Symbol resolution client
│       └── ServiceCollectionExtensions.cs # DI registration helpers
└── README.md
```

## Development

### Prerequisites

- .NET 9.0 SDK
- Grpc.Tools NuGet package (included in project)

### Building

gRPC code is auto-generated during build:

```bash
cd Events/src/Events.Client
dotnet build
```

Generated files appear in `obj/Debug/net9.0/`:
- `ObservationEvents.cs` - Message classes
- `ObservationEventsGrpc.cs` - Service stubs
- `Secmaster.cs` - SecMaster message classes
- `SecmasterGrpc.cs` - SecMaster service stubs

## Usage

### Adding as a Project Reference

```xml
<ProjectReference Include="../Events/src/Events.Client/Events.Client.csproj" />
```

### Subscribing to Events (Consumer)

```csharp
// Register client in DI
services.AddObservationEventClient(options =>
{
    options.Endpoint = "http://fred-collector:5002";
    options.Name = "FredCollector";
});

// Subscribe to event stream
await foreach (var evt in client.SubscribeAsync(DateTime.UtcNow.AddHours(-1), cancellationToken: ct))
{
    switch (evt.PayloadCase)
    {
        case Event.PayloadOneofCase.SeriesCollected:
            HandleSeriesCollected(evt.SeriesCollected);
            break;
        case Event.PayloadOneofCase.OhlcvCollected:
            HandleOhlcvCollected(evt.OhlcvCollected);
            break;
        case Event.PayloadOneofCase.CollectionFailed:
            HandleCollectionFailed(evt.CollectionFailed);
            break;
    }
}
```

### Registering Series with SecMaster (Collector)

```csharp
// Register client in DI
services.AddSecMasterRegistryClient(options =>
{
    options.Endpoint = "http://secmaster:8080";
});

// Register a series (fire-and-forget with retry)
await registryClient.RegisterSeriesAsync(
    collectorName: "FredCollector",
    collectorSeriesId: "UNRATE",
    symbol: "UNRATE",
    title: "Unemployment Rate",
    assetClass: "Economic",
    frequency: "Monthly");
```

### Resolving Symbols (ThresholdEngine)

```csharp
// Register client in DI
services.AddSecMasterResolverClient(options =>
{
    options.Endpoint = "http://secmaster:8080";
});

// Resolve symbol to data source
var resolution = await resolverClient.ResolveSymbolAsync("UNRATE");
if (resolution?.Found == true)
{
    var collector = resolution.PrimarySource.Collector;
    var seriesId = resolution.PrimarySource.CollectorSeriesId;
}
```

## Contract Reference

### Proto Files

| File | Services | Purpose |
|------|----------|---------|
| `observation_events.proto` | ObservationEventStream | Data collection events streaming |
| `secmaster.proto` | SecMasterRegistry, SecMasterResolver | Symbol registration and resolution |

### Event Types (observation_events.proto)

| Event | Description | Publishers |
|-------|-------------|------------|
| SeriesCollectedEvent | Scalar data points (economic indicators) | FredCollector, AlphaVantageCollector, OfrCollector |
| OhlcvCollectedEvent | OHLCV candles (equities, forex) | FinnhubCollector |
| CollectionFailedEvent | Collection failures | All collectors |

### In-Memory Events (C# Records)

| Event | Publisher | Consumer |
|-------|-----------|----------|
| ObservationCollectedEvent | Collectors | ThresholdEngine |
| ThresholdCrossedEvent | ThresholdEngine | AlertService |
| RegimeTransitionEvent | ThresholdEngine | AlertService |

## Versioning

- **Forward compatible**: New fields are always optional
- **Backward compatible**: Never remove or renumber fields
- **Breaking changes**: Create new message types (e.g., `SeriesCollectedEventV2`)

## See Also

- [FredCollector](../FredCollector/) - FRED economic data collector
- [FinnhubCollector](../FinnhubCollector/) - Finnhub market data collector
- [ThresholdEngine](../ThresholdEngine/) - Event consumer and threshold evaluation
- [SecMaster](../SecMaster/) - Instrument and source registry
