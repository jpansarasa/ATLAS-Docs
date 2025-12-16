# Events

Shared gRPC event contracts and in-memory event definitions for ATLAS microservices.

## Overview

This library is the schema registry for ATLAS. It defines two types of contracts:

1. **gRPC Protobuf contracts** - Wire format for inter-service streaming (collectors to consumers)
2. **C# event records** - In-memory events for internal event buses (System.Threading.Channels)

All collectors stream events using the same gRPC interface. All consumers subscribe using the same client.

## Proto Files

| File | Purpose |
|------|---------|
| `observation_events.proto` | Data collection events (SeriesCollected, OhlcvCollected, CollectionFailed) |
| `secmaster.proto` | SecMaster registration and resolution services |

## Event Types

### Data Collection Events (observation_events.proto)

| Event Type | Description | Used By |
|------------|-------------|---------|
| `SeriesCollectedEvent` | Scalar data points collected (economic indicators, commodities) | FredCollector, AlphaVantageCollector, OfrCollector |
| `OhlcvCollectedEvent` | OHLCV candles collected (equities, forex, crypto) | FinnhubCollector |
| `CollectionFailedEvent` | Collection attempt failed (rate limit, API error, network) | All collectors |

### SecMaster Services (secmaster.proto)

| Service | Purpose | Used By |
|---------|---------|---------|
| `SecMasterRegistry` | Register series with SecMaster (fire-and-forget) | All collectors |
| `SecMasterResolver` | Resolve symbols to data sources | ThresholdEngine |

## In-Memory C# Events

Used for internal event buses within a service:

- `ObservationCollectedEvent` - Single observation collected
- `ThresholdCrossedEvent` - Threshold evaluation triggered
- `RegimeTransitionEvent` - Market regime change detected
- `IEvent` - Base interface for all events

## Usage

### Collectors (Publishing Events)

Collectors implement `ObservationEventStream` service:

```csharp
public class EventStreamService : ObservationEventStream.ObservationEventStreamBase
{
    public override async Task SubscribeToEvents(
        SubscriptionRequest request,
        IServerStreamWriter<Event> responseStream,
        ServerCallContext context)
    {
        // Stream events to consumer
    }
}
```

### Consumers (Subscribing to Events)

Consumers use `Events.Client` wrapper:

```csharp
services.AddCollectorClient("FredCollector", "http://fred-collector:5002");

// Subscribe to events
await foreach (var evt in client.SubscribeToEventsAsync(cancellationToken))
{
    switch (evt.PayloadCase)
    {
        case Event.PayloadOneofCase.SeriesCollected:
            // Handle scalar data
            break;
        case Event.PayloadOneofCase.OhlcvCollected:
            // Handle OHLCV data
            break;
    }
}
```

## Building

gRPC code is auto-generated during build via `Grpc.Tools` package.

Manual regeneration (if needed):

```bash
cd Events/src/Events
dotnet build Events/src/Events/Events.csproj
```

Generated files appear in `obj/Debug/net9.0/`:
- `Events.cs` - Message classes
- `EventsGrpc.cs` - Service stubs

## Versioning

- Forward compatible: New fields are always optional
- Backward compatible: Never remove or renumber fields
- Breaking changes: Create new message types (e.g., `SeriesCollectedEventV2`)
