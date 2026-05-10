# CalendarService.Core

Shared library of market/economic calendar abstractions, models, providers, and Quartz integration. Consumed in-process by both the `CalendarService` host and downstream services that need calendar awareness without an HTTP round trip.

## Overview

`CalendarService.Core` is the in-process building block behind ATLAS's calendar surface. It exposes:

- **Abstractions** — `IMarketCalendar`, `IEconomicCalendar`, `IEconomicEventSource`, `ITradingHoursProvider`.
- **Models** — `MarketHoliday`, `MarketStatus`, `TradingSession`, `EconomicEvent`, plus the `MarketHolidayType` / `EconomicEventType` / `EconomicImpact` enums.
- **Providers** — `NyseMarketCalendar` (calendar over `StaticNyseHolidays`), `CachedEconomicCalendar` (memory-cache wrapper around any `IEconomicEventSource`).
- **Quartz integration** — `MarketHolidayCalendar` and `TradingHoursCalendar` for filtering Quartz triggers by market open/close and holidays.

The host project (`CalendarService`) uses these types to answer REST requests; other services can take a project reference to skip the HTTP hop when they only need NYSE rules or to suppress Quartz jobs on holidays.

## Architecture

```
consumer code
    │ inject IMarketCalendar / IEconomicCalendar
    ▼
NyseMarketCalendar ──► StaticNyseHolidays (in-memory)
CachedEconomicCalendar ──► IEconomicEventSource
                                │
                                ▼
                    (host wires this to FRED / Nager.Date)

Quartz scheduler ──► MarketHolidayCalendar / TradingHoursCalendar
                          (filter triggers by NYSE rules)
```

The library is consumed two ways: directly by services that need NYSE/holiday awareness in-process, and by the `CalendarService` host project which wraps the same types behind a REST API for clients that prefer an HTTP boundary.

## Features

- **`IMarketCalendar` / `NyseMarketCalendar`** — NYSE trading-day rules (full list of federal/NYSE holidays + half-days).
- **`IEconomicCalendar` / `CachedEconomicCalendar`** — economic-event lookup with `IMemoryCache` wrapper around an injected source.
- **`ITradingHoursProvider`** — RTH / extended-hours session classification.
- **Quartz `Calendar` integrations** — `MarketHolidayCalendar` (skip holidays) and `TradingHoursCalendar` (RTH-only) plug into Quartz triggers for holiday-aware scheduling.
- **`AddCalendarServiceCore()`** — single DI extension wires every interface to its default implementation.

## Usage

Reference the project (it is not currently published as a NuGet package):

```xml
<ProjectReference Include="../../CalendarService/src/Core/CalendarService.Core.csproj" />
```

Register the calendar implementations via DI:

```csharp
using CalendarService.Core.Extensions;

services.AddCalendarServiceCore();   // wires NyseMarketCalendar, CachedEconomicCalendar, etc.
```

Then inject the abstractions:

```csharp
public class MyJob(IMarketCalendar market, IEconomicCalendar economic)
{
    public async Task Run(DateTimeOffset now, CancellationToken ct)
    {
        if (!market.IsTradingDay(now)) return;
        var events = await economic.GetEventsForDateAsync(now.Date, ct);
        // ...
    }
}
```

### Quartz integration

```csharp
schedulerConfig.UseCalendar("nyse-holidays", new MarketHolidayCalendar(marketCalendar));
schedulerConfig.UseCalendar("nyse-hours", new TradingHoursCalendar(tradingHours));

trigger
    .ModifiedByCalendar("nyse-holidays")
    .ModifiedByCalendar("nyse-hours");
```

## Configuration

No runtime configuration — the library reads no env vars and loads no files. NYSE holidays are baked into `StaticNyseHolidays`; economic events come from the `IEconomicEventSource` the consumer injects.

The host service (`CalendarService`) does have its own configuration (FRED API key, Nager.Date toggles, holiday-sync schedule, OpenTelemetry endpoints) — those live in `CalendarService/README.md`.

## API Endpoints

This is a library, not a service. It exposes no HTTP/gRPC surface. The consumed services wrap these abstractions:

- [CalendarService](../../README.md) host — REST endpoints over the same calendar logic.

## Project Structure

```
CalendarService.Core/
├── Abstractions/
│   ├── IEconomicCalendar.cs
│   ├── IEconomicEventSource.cs
│   ├── IMarketCalendar.cs
│   └── ITradingHoursProvider.cs
├── Models/
│   ├── EconomicEvent.cs
│   ├── EconomicEventType.cs
│   ├── EconomicImpact.cs
│   ├── MarketHoliday.cs
│   ├── MarketHolidayType.cs
│   ├── MarketStatus.cs
│   └── TradingSession.cs
├── Providers/
│   ├── CachedEconomicCalendar.cs   # IMemoryCache wrapper over IEconomicEventSource
│   ├── NyseMarketCalendar.cs       # NYSE rules (holidays + half-days)
│   └── StaticNyseHolidays.cs       # static holiday list (Federal Reserve / NYSE)
├── Quartz/
│   ├── MarketHolidayCalendar.cs    # Quartz Calendar that excludes holidays
│   └── TradingHoursCalendar.cs     # Quartz Calendar limited to trading hours
├── Extensions/
│   └── ServiceCollectionExtensions.cs   # AddCalendarServiceCore()
└── CalendarService.Core.csproj
```

## Development

### Prerequisites

- .NET 10 SDK (or via devcontainer)
- `Quartz` 3.13.1
- `Microsoft.Extensions.Caching.Memory` 10.0.0

### Building

```bash
dotnet build CalendarService/src/Core/CalendarService.Core.csproj
```

The package has no runtime dependencies on the `CalendarService` host project — it can be referenced standalone by any consumer.

## Deployment

Library only — not deployed independently. Consumers reference this project and ship it inside their own container image. There is no ansible tag for this package.

## Ports

None — library, no listening sockets. The `CalendarService` host exposes the calendar logic over its own ports — see [CalendarService Ports](../../README.md#ports).

## See Also

- [CalendarService](../../README.md) — host service exposing this library over REST and running the holiday-sync workers
- [ThresholdEngine](../../../ThresholdEngine/) — pattern evaluator that gates jobs on market hours
- [FredCollector](../../../FredCollector/) — backing source for `IEconomicEventSource` (FRED economic calendar)
