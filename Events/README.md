# Events

Shared event contracts for ATLAS microservices.

## Purpose

This project provides a single source of truth for all event definitions used across ATLAS services, preventing drift and ensuring type safety.

## Events

### ObservationCollectedEvent
- **Published by**: FredCollector
- **Consumed by**: ThresholdEngine
- **Purpose**: Notifies when new observation data is collected

### ThresholdCrossedEvent
- **Published by**: ThresholdEngine
- **Consumed by**: AlertService
- **Purpose**: Notifies when a pattern threshold is crossed

### RegimeTransitionEvent
- **Published by**: ThresholdEngine
- **Consumed by**: AlertService
- **Purpose**: Notifies when macro economic regime transitions

## Usage

Reference this project in your service:

```xml
<ProjectReference Include="..\..\..\Events\src\Events\Events.csproj" />
```

Then use events:

```csharp
using ATLAS.Events;

var evt = ObservationCollectedEvent.FromObservation(
    seriesId, date, value, collectedAt);
```

## Versioning

Events are immutable records. Breaking changes require:
1. Create new event version (e.g., `ObservationCollectedEventV2`)
2. Update all publishers and consumers
3. Deprecate old event
4. Remove old event in next major version

## Future: NuGet Package

For Phase 2+, this can be packaged as a NuGet package for easier distribution across services. The PackageId in the .csproj is already configured.

