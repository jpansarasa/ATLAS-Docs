# Events

Shared event contracts and gRPC definitions for ATLAS microservices.

## Overview

This library serves as the "Schema Registry" for the ATLAS system. It ensures that all services speak the same language when exchanging data, whether in-process or over the network.

## Structure

```
Events/
├── src/
│   ├── Events/                 # Core definitions
│   │   ├── Protos/             # gRPC Protobuf contracts (.proto)
│   │   └── *.cs                # In-memory C# event records
│   └── Events.Client/          # gRPC Client Library
```

## Protobuf Contracts

The source of truth for inter-service communication.

### `events.proto`

Defines the data exchange format between Collectors and ThresholdEngine.

```protobuf
message Event {
    string event_id = 1;
    google.protobuf.Timestamp occurred_at = 2;
    string source_service = 3;
    
    oneof payload {
        SeriesCollectedEvent series_collected = 10;
        CollectionFailedEvent collection_failed = 11;
        // ...
    }
}

message SeriesCollectedEvent {
    string series_id = 1;
    google.protobuf.Timestamp collected_at = 2;
    repeated DataPoint data_points = 3;
}
```

## C# Event Records

Used for internal event buses (e.g., `ChannelEventBus` inside a service).

```csharp
public record ObservationCollectedEvent(
    string SeriesId,
    DateTimeOffset Date,
    decimal Value,
    DateTimeOffset CollectedAt
) : IEvent;
```

## Events.Client

A standard gRPC client wrapper that simplifies consuming events from any Collector service.

### Usage

```csharp
// In Consumer Service (e.g., ThresholdEngine)
services.AddCollectorClient("FredCollector", "http://fred-collector:5001");

// In Code
await foreach (var evt in client.SubscribeToEventsAsync(cancellationToken))
{
    // Handle event
}
```

## Versioning Strategy

1. **Forward Compatibility**: New fields in Protobuf are always optional.
2. **Backward Compatibility**: Never remove or renumber existing fields.
3. **Breaking Changes**: Require a new message type (e.g., `SeriesCollectedEventV2`).
