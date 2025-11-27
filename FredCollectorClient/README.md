# FredCollectorClient

gRPC client library for consuming FredCollector event streams.

## Purpose

This library provides the `EventStreamClient` for subscribing to real-time economic indicator events from FredCollector without requiring consumers to manage protobuf compilation.

## Usage

```csharp
// Add project reference
<ProjectReference Include="..\..\FredCollectorClient\FredCollectorClient.csproj" />

// Create client
var channel = GrpcChannel.ForAddress("http://fred-collector:5002");
var client = new EventStream.EventStreamClient(channel);

// Subscribe to events
var request = new SubscriptionRequest
{
    StartFrom = Timestamp.FromDateTime(DateTime.UtcNow.AddHours(-1))
};

await foreach (var evt in client.SubscribeToEvents(request).ResponseStream.ReadAllAsync())
{
    if (evt.PayloadCase == Event.PayloadOneofCase.SeriesCollected)
    {
        Console.WriteLine($"Series: {evt.SeriesCollected.SeriesId}");
    }
}
```

## Generated Types

- `EventStreamClient` - gRPC client for connecting to FredCollector
- `Event`, `SeriesCollectedEvent`, `CollectionFailedEvent` - Event message types
- `SubscriptionRequest`, `TimeRangeRequest` - Request types
- `HealthResponse` - Health check response

## Architecture

- **Proto Source**: `../FredCollector/protos/events.proto` (FredCollector owns contract)
- **Code Generation**: `GrpcServices="Client"` (client stubs only)
- **Namespace**: `ATLAS.Events.Grpc`

## Consumers

- **ThresholdEngine** - Pattern evaluation service (subscribes to SeriesCollectedEvent)
- **Future services** - Any service needing FredCollector events

## See Also

- [gRPC Architecture](../docs/GRPC-ARCHITECTURE.md)
