# CalendarService

Market and economic calendar service for ATLAS.

## Overview

CalendarService provides temporal data for the financial system:
- **Market Calendar**: Trading days, holidays, and market status (Open/Closed) for NYSE/Nasdaq
- **Economic Calendar**: Scheduled economic events (CPI, FOMC, etc.) with impact ratings

Exposes REST API and shared core library (`CalendarService.Core`) for in-process use.

## Features

- Trading day calculations (accounting for weekends and NYSE holidays)
- Real-time market status checks
- Economic event tracking with impact ratings
- Holiday data integration (Nager.Date API for US public holidays)
- Background workers for data collection
- In-memory calendar rules engine for high-performance operations

## API Endpoints

### Market Calendar

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/api/market/status` | GET | Current market status (open/closed) |
| `/api/market/holidays` | GET | Market holidays for specified year |
| `/api/market/is-trading-day` | GET | Check if date is a trading day |
| `/api/market/next-trading-day` | GET | Next trading day from specified date |
| `/api/market/trading-days` | GET | Trading days in date range |
| `/api/market/holidays/external` | GET | US public holidays from Nager.Date API |

### Economic Calendar

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/api/economic/events` | GET | Economic events by date range, impact, type, country |
| `/api/economic/upcoming` | GET | Upcoming events (default 7 days ahead) |
| `/api/economic/high-impact` | GET | Upcoming high-impact events |
| `/api/economic/has-high-impact` | GET | Check if date has high-impact events |

## Configuration

Environment variables:

| Variable | Description | Required |
|----------|-------------|----------|
| `ConnectionStrings__Calendar` | PostgreSQL connection string | Yes |
| `FINNHUB_API_KEY` | Finnhub API key for economic calendar | No |
| `FRED_API_KEY` | FRED API key for economic releases | Yes |

## Background Workers

- **MarketHolidaySyncWorker**: Syncs NYSE holidays to database (runs every 24 hours)
- **FredReleasesCollectorWorker**: Collects economic release dates from FRED API
- **EconomicCalendarCollectorWorker**: Collects economic events from Finnhub (requires paid subscription, disabled by default)

## Project Structure

```
CalendarService/
├── src/
│   ├── Core/                   # Shared library (abstractions, models, providers)
│   ├── Endpoints/              # API endpoint handlers
│   ├── External/               # External API clients (Finnhub, FRED, Nager)
│   ├── Persistence/            # Database context and repositories
│   ├── Workers/                # Background workers
│   ├── Migrations/             # EF Core migrations
│   ├── Containerfile           # Container image definition
│   └── Program.cs              # Application entry point
└── tests/                      # Unit tests
```

## Development

Uses .NET 9.0 minimal APIs with OpenAPI/Scalar for documentation.

### Container Build

```bash
# Build from ATLAS root
sudo nerdctl build -f CalendarService/src/Containerfile -t calendar-service .
```

### Local Development

```bash
# Run from CalendarService/src
dotnet run
```

API documentation available at `/scalar/v1` when running.

## Library Usage

Other services reference `CalendarService.Core` for in-memory calendar operations:

```csharp
public class MyService
{
    private readonly IMarketCalendar _calendar;

    public void Process()
    {
        if (_calendar.IsTradingDay(DateOnly.FromDateTime(DateTime.UtcNow)))
        {
            // Execute trading logic
        }
    }
}
```

