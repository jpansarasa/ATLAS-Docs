# E1: Refactor gRPC to Common Events Project

## Goal
Establish a single source of truth for event contracts and gRPC streaming client, enabling multiple collectors to publish `ObservationCollectedEvent` through a shared contract.

## Context

Currently:
- Proto definition lives in `FredCollector/protos/events.proto`
- Client library is `FredCollectorClient` with Fred-specific naming
- ThresholdEngine references `FredCollectorClient` directly

After refactor:
- Proto moves to `Events/src/Events/Protos/`
- Client becomes `Events.Client` with `ObservationEventClient`
- All collectors reference shared proto
- ThresholdEngine uses generic client connecting to multiple collector endpoints

## Architecture

```
Events/
├── src/
│   ├── Events/
│   │   ├── Events.csproj
│   │   ├── IEvent.cs
│   │   ├── ObservationCollectedEvent.cs
│   │   ├── ThresholdCrossedEvent.cs
│   │   ├── RegimeTransitionEvent.cs
│   │   └── Protos/
│   │       └── observation_events.proto
│   └── Events.Client/
│       ├── Events.Client.csproj
│       ├── ObservationEventClient.cs
│       └── ServiceCollectionExtensions.cs
```

## Prerequisites
- Access to ATLAS monorepo at `~/ATLAS`
- Working FredCollector → ThresholdEngine pipeline (verify before starting)
- Understanding of gRPC proto compilation in .NET

## Tasks

### T1.1: Verify Existing Pipeline
**Estimate:** 15 min

Before any changes, verify the current system works:

```bash
cd ~/ATLAS
# Check services are running
sudo nerdctl compose ps

# Verify FredCollector gRPC is responding
grpcurl -plaintext localhost:5002 list

# Check ThresholdEngine is receiving events (check logs)
sudo nerdctl logs threshold-engine --tail 50
```

**Acceptance:** Document current state, confirm events flowing

---

### T1.2: Create Events.Client Project
**Estimate:** 30 min

Create the new client library project:

```bash
cd ~/ATLAS/Events/src
dotnet new classlib -n Events.Client -f net9.0
```

Update `Events.Client.csproj`:
```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>net9.0</TargetFramework>
    <ImplicitUsings>enable</ImplicitUsings>
    <Nullable>enable</Nullable>
    <RootNamespace>ATLAS.Events.Client</RootNamespace>
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="Google.Protobuf" Version="3.28.3" />
    <PackageReference Include="Grpc.Net.Client" Version="2.67.0" />
    <PackageReference Include="Grpc.Tools" Version="2.68.1" PrivateAssets="All" />
    <PackageReference Include="Microsoft.Extensions.DependencyInjection.Abstractions" Version="9.0.0" />
    <PackageReference Include="Microsoft.Extensions.Options" Version="9.0.0" />
  </ItemGroup>

  <ItemGroup>
    <ProjectReference Include="..\Events\Events.csproj" />
  </ItemGroup>

  <ItemGroup>
    <Protobuf Include="..\Events\Protos\observation_events.proto" GrpcServices="Client" />
  </ItemGroup>
</Project>
```

**Acceptance:** Project builds with no errors

---

### T1.3: Move Proto Definition
**Estimate:** 20 min

1. Create directory:
```bash
mkdir -p ~/ATLAS/Events/src/Events/Protos
```

2. Copy proto from FredCollector:
```bash
cp ~/ATLAS/FredCollector/protos/events.proto \
   ~/ATLAS/Events/src/Events/Protos/observation_events.proto
```

3. Update proto package and options:
```protobuf
syntax = "proto3";

package atlas.events;

option csharp_namespace = "ATLAS.Events.Grpc";

import "google/protobuf/timestamp.proto";

service ObservationEventStream {
  rpc SubscribeToEvents(SubscriptionRequest) returns (stream Event);
  rpc GetEventsSince(TimeRangeRequest) returns (stream Event);
  rpc Health(HealthRequest) returns (HealthResponse);
}

message Event {
  string event_id = 1;
  google.protobuf.Timestamp occurred_at = 2;
  string source_service = 3;
  
  oneof payload {
    SeriesCollectedEvent series_collected = 10;
    CollectionFailedEvent collection_failed = 11;
  }
}

message SeriesCollectedEvent {
  string series_id = 1;
  google.protobuf.Timestamp date = 2;
  double value = 3;
  google.protobuf.Timestamp collected_at = 4;
}

message CollectionFailedEvent {
  string series_id = 1;
  string error_message = 2;
  google.protobuf.Timestamp failed_at = 3;
}

message SubscriptionRequest {
  google.protobuf.Timestamp start_from = 1;
  repeated string series_ids = 2;
}

message TimeRangeRequest {
  google.protobuf.Timestamp from = 1;
  google.protobuf.Timestamp to = 2;
  repeated string series_ids = 3;
}

message HealthRequest {}

message HealthResponse {
  bool healthy = 1;
  string message = 2;
}
```

4. Update Events.csproj to include proto for server generation (if needed by collectors):
```xml
<ItemGroup>
  <Protobuf Include="Protos\observation_events.proto" GrpcServices="Server" />
</ItemGroup>
```

**Acceptance:** Proto compiles in Events project, generates C# types

---

### T1.4: Implement ObservationEventClient
**Estimate:** 45 min

Create `Events.Client/ObservationEventClient.cs`:

```csharp
using System.Runtime.CompilerServices;
using ATLAS.Events.Grpc;
using Grpc.Core;
using Grpc.Net.Client;
using Microsoft.Extensions.Logging;
using Microsoft.Extensions.Options;

namespace ATLAS.Events.Client;

public sealed class ObservationEventClientOptions
{
    public required string Endpoint { get; set; }
    public string? Name { get; set; }
    public TimeSpan ConnectionTimeout { get; set; } = TimeSpan.FromSeconds(30);
    public TimeSpan RetryDelay { get; set; } = TimeSpan.FromSeconds(5);
}

public sealed class ObservationEventClient : IAsyncDisposable
{
    private readonly GrpcChannel _channel;
    private readonly ObservationEventStream.ObservationEventStreamClient _client;
    private readonly ILogger<ObservationEventClient> _logger;
    private readonly string _name;

    public ObservationEventClient(
        IOptions<ObservationEventClientOptions> options,
        ILogger<ObservationEventClient> logger)
    {
        var opts = options.Value;
        _name = opts.Name ?? opts.Endpoint;
        _logger = logger;
        
        _channel = GrpcChannel.ForAddress(opts.Endpoint, new GrpcChannelOptions
        {
            HttpHandler = new SocketsHttpHandler
            {
                PooledConnectionIdleTimeout = Timeout.InfiniteTimeSpan,
                KeepAlivePingDelay = TimeSpan.FromSeconds(60),
                KeepAlivePingTimeout = TimeSpan.FromSeconds(30),
                EnableMultipleHttp2Connections = true
            }
        });
        
        _client = new ObservationEventStream.ObservationEventStreamClient(_channel);
    }

    public async IAsyncEnumerable<Event> SubscribeAsync(
        DateTime startFrom,
        IEnumerable<string>? seriesIds = null,
        [EnumeratorCancellation] CancellationToken cancellationToken = default)
    {
        var request = new SubscriptionRequest
        {
            StartFrom = Google.Protobuf.WellKnownTypes.Timestamp.FromDateTime(
                DateTime.SpecifyKind(startFrom, DateTimeKind.Utc))
        };
        
        if (seriesIds != null)
            request.SeriesIds.AddRange(seriesIds);

        using var call = _client.SubscribeToEvents(request, cancellationToken: cancellationToken);
        
        _logger.LogInformation("Subscribed to {Name} from {StartFrom}", _name, startFrom);

        await foreach (var evt in call.ResponseStream.ReadAllAsync(cancellationToken))
        {
            yield return evt;
        }
    }

    public async IAsyncEnumerable<Event> GetEventsSinceAsync(
        DateTime from,
        DateTime? to = null,
        IEnumerable<string>? seriesIds = null,
        [EnumeratorCancellation] CancellationToken cancellationToken = default)
    {
        var request = new TimeRangeRequest
        {
            From = Google.Protobuf.WellKnownTypes.Timestamp.FromDateTime(
                DateTime.SpecifyKind(from, DateTimeKind.Utc)),
            To = Google.Protobuf.WellKnownTypes.Timestamp.FromDateTime(
                DateTime.SpecifyKind(to ?? DateTime.UtcNow, DateTimeKind.Utc))
        };
        
        if (seriesIds != null)
            request.SeriesIds.AddRange(seriesIds);

        using var call = _client.GetEventsSince(request, cancellationToken: cancellationToken);

        await foreach (var evt in call.ResponseStream.ReadAllAsync(cancellationToken))
        {
            yield return evt;
        }
    }

    public async Task<bool> HealthCheckAsync(CancellationToken cancellationToken = default)
    {
        try
        {
            var response = await _client.HealthAsync(new HealthRequest(), cancellationToken: cancellationToken);
            return response.Healthy;
        }
        catch (RpcException ex)
        {
            _logger.LogWarning(ex, "Health check failed for {Name}", _name);
            return false;
        }
    }

    public async ValueTask DisposeAsync()
    {
        await _channel.ShutdownAsync();
        _channel.Dispose();
    }
}
```

Create `Events.Client/ServiceCollectionExtensions.cs`:

```csharp
using Microsoft.Extensions.DependencyInjection;

namespace ATLAS.Events.Client;

public static class ServiceCollectionExtensions
{
    public static IServiceCollection AddObservationEventClient(
        this IServiceCollection services,
        Action<ObservationEventClientOptions> configure)
    {
        services.Configure(configure);
        services.AddSingleton<ObservationEventClient>();
        return services;
    }
    
    public static IServiceCollection AddObservationEventClient(
        this IServiceCollection services,
        string name,
        Action<ObservationEventClientOptions> configure)
    {
        services.Configure<ObservationEventClientOptions>(name, configure);
        // For named clients, use IOptionsSnapshot or keyed services
        return services;
    }
}
```

**Acceptance:** Client compiles, exposes Subscribe/GetEventsSince/HealthCheck methods

---

### T1.5: Update FredCollector to Reference Shared Proto
**Estimate:** 30 min

1. Update `FredCollector.Grpc/FredCollector.Grpc.csproj`:
```xml
<ItemGroup>
  <!-- Remove local proto, reference shared -->
  <Protobuf Include="..\..\..\..\Events\src\Events\Protos\observation_events.proto" 
            GrpcServices="Server" />
</ItemGroup>

<ItemGroup>
  <ProjectReference Include="..\..\..\..\Events\src\Events\Events.csproj" />
</ItemGroup>
```

2. Update gRPC service implementation to use new namespace:
```csharp
using ATLAS.Events.Grpc;
// Update service class to implement ObservationEventStream.ObservationEventStreamBase
```

3. Update any code referencing old proto-generated types

4. Build and verify:
```bash
cd ~/ATLAS/FredCollector
dotnet build
dotnet test
```

**Acceptance:** FredCollector builds, all tests pass

---

### T1.6: Update ThresholdEngine to Use Events.Client
**Estimate:** 30 min

1. Remove `FredCollectorClient` project reference from ThresholdEngine

2. Add `Events.Client` reference to `ThresholdEngine.Infrastructure.csproj`:
```xml
<ItemGroup>
  <ProjectReference Include="..\..\..\..\Events\src\Events.Client\Events.Client.csproj" />
</ItemGroup>
```

3. Update DI registration in `Program.cs`:
```csharp
builder.Services.AddObservationEventClient(options =>
{
    options.Endpoint = builder.Configuration["Collectors:FredCollector:Endpoint"] 
        ?? "http://fred-collector:5002";
    options.Name = "FredCollector";
});
```

4. Update `EventConsumerWorker` to use `ObservationEventClient`:
```csharp
public class EventConsumerWorker : BackgroundService
{
    private readonly ObservationEventClient _client;
    // ... update constructor and usage
}
```

5. Build and verify:
```bash
cd ~/ATLAS/ThresholdEngine
dotnet build
dotnet test
```

**Acceptance:** ThresholdEngine builds, all tests pass

---

### T1.7: Delete FredCollectorClient Project
**Estimate:** 10 min

1. Verify no other projects reference it:
```bash
grep -r "FredCollectorClient" ~/ATLAS --include="*.csproj"
```

2. Remove from solution:
```bash
cd ~/ATLAS
dotnet sln remove FredCollectorClient/FredCollectorClient.csproj
```

3. Delete directory:
```bash
rm -rf ~/ATLAS/FredCollectorClient
```

4. Update any documentation references

**Acceptance:** FredCollectorClient removed, no dangling references

---

### T1.8: Integration Test
**Estimate:** 30 min

1. Rebuild all projects:
```bash
cd ~/ATLAS
dotnet build
```

2. Run all tests:
```bash
dotnet test
```

3. Deploy to local environment:
```bash
cd ~/ATLAS/ansible
ansible-playbook playbooks/site.yml
```

4. Verify end-to-end flow:
```bash
# Check FredCollector gRPC
grpcurl -plaintext localhost:5002 list
grpcurl -plaintext localhost:5002 atlas.events.ObservationEventStream/Health

# Check ThresholdEngine logs for event consumption
sudo nerdctl logs threshold-engine --tail 100 | grep -i event

# Verify pattern evaluation still works
curl http://localhost:5003/api/patterns | jq '.[] | select(.enabled==true) | .patternId'
```

5. Monitor Grafana dashboards for any anomalies

**Acceptance:** Full pipeline operational, events flowing, patterns evaluating

---

### T1.9: Update Documentation
**Estimate:** 20 min

1. Update `Events/README.md` with new structure
2. Update `FredCollector/README.md` to reference shared proto
3. Update `ThresholdEngine/README.md` to reference Events.Client
4. Update `docs/GRPC-ARCHITECTURE.md` with new project locations
5. Delete `FredCollectorClient/README.md` (already removed with project)

**Acceptance:** All documentation reflects new architecture

---

## Acceptance Criteria

- [ ] Proto definition lives in `Events/src/Events/Protos/`
- [ ] `Events.Client` project exists with `ObservationEventClient`
- [ ] FredCollector references shared proto, builds successfully
- [ ] ThresholdEngine uses `Events.Client`, builds successfully
- [ ] `FredCollectorClient` project deleted
- [ ] All existing tests pass
- [ ] End-to-end pipeline works (FredCollector → ThresholdEngine → AlertService)
- [ ] Documentation updated

## Rollback Plan

If issues arise:
1. Git revert to pre-refactor commit
2. Redeploy previous containers
3. Verify pipeline restored

## Notes

- Keep proto backward compatible (no field number changes)
- Service name change (`EventStream` → `ObservationEventStream`) requires coordinated deploy
- Consider feature flag for gradual rollout if concerned
