# AlphaVantageCollector

Collector service for Alpha Vantage market data with strict rate limiting and priority-based scheduling.

## Overview

AlphaVantageCollector fetches financial and economic data from Alpha Vantage, including commodities, economic indicators, equities, forex, and cryptocurrencies. It implements priority-based scheduling to efficiently work within the free tier limit (25 requests/day).

## Features

- **Multi-Asset Support**: Commodities, economic indicators, equities (OHLCV), forex, cryptocurrencies
- **Priority Scheduling**: High-priority series collected more frequently based on urgency and interval
- **Rate Limiting**: Token bucket algorithm enforces 25 requests/day with burst control
- **Market Calendar Integration**: Skips collection on CME market holidays
- **Real-time Streaming**: gRPC event stream for downstream consumers
- **Admin API**: Dynamic series management without service restart
- **TimescaleDB Storage**: Efficient time-series storage with hypertables

## Data Sources

### Commodities
- Copper, Aluminum, WTI, Brent, Natural Gas, Wheat, Corn, Cotton, Sugar, Coffee

### Economic Indicators
- Real GDP, Treasury Yield, Federal Funds Rate, CPI, Inflation, Retail Sales, Durables, Unemployment, Nonfarm Payroll

### Equities (OHLCV)
- Major ETFs: SPY, QQQ, DIA, IWM, GLD, SLV, TLT, VIXY

### Forex
- Major pairs: EUR/USD, GBP/USD, USD/JPY, USD/CHF, AUD/USD, USD/CAD

### Cryptocurrencies
- BTC, ETH, SOL, XRP

## Configuration

Environment variables:

| Variable | Description | Default |
|----------|-------------|---------|
| `ConnectionStrings__AtlasDb` | PostgreSQL connection | Required |
| `AlphaVantage__ApiKey` | API key from alphavantage.co | Required |
| `AlphaVantage__DailyLimit` | Max requests per day | `25` |
| `OpenTelemetry__OtlpEndpoint` | OTLP collector endpoint | `http://otel-collector:4317` |

## API Endpoints

### REST API (Port 5006)

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/api/admin/series` | GET | List all configured series |
| `/api/admin/series` | POST | Add new series |
| `/api/admin/series/{id}/toggle` | PUT | Enable/disable series |
| `/api/admin/series/{id}` | DELETE | Delete series |
| `/api/admin/search?q={query}` | GET | Search available symbols |
| `/health` | GET | Health check with database status |
| `/health/ready` | GET | Readiness probe |
| `/health/live` | GET | Liveness probe |

### gRPC API (Port 5007)

Service: `ObservationEventStream`

- `SubscribeToEvents(SubscriptionRequest)`: Stream observation events in real-time

## Collection Behavior

- **Collection Interval**: Runs every 4 hours
- **Requests per Cycle**: Maximum 4 requests per cycle
- **Priority Ordering**: Lower priority number = higher importance
- **Interval-based Scheduling**: Daily series collected every 20+ hours, monthly every 25+ days, etc.
- **Holiday Awareness**: Skips collection on CME market holidays

## Project Structure

```
AlphaVantageCollector/
├── src/
│   ├── Api/                    # Alpha Vantage API client
│   ├── Data/                   # EF Core DbContext and repository
│   ├── Grpc/                   # gRPC event stream service
│   ├── HealthChecks/           # Database health check
│   ├── Interfaces/             # Service contracts
│   ├── Models/                 # Domain models
│   ├── Services/               # Scheduler, series management
│   ├── Telemetry/              # OpenTelemetry metrics and traces
│   ├── Workers/                # Background collection worker
│   └── Program.cs              # Service host
├── migrations/                 # SQL schema migrations
└── tests/                      # Unit tests
```

## Development

### Dev Container

1. Open in VS Code and select "Reopen in Container"
2. Create `.env` file with `AlphaVantage__ApiKey=your_key`
3. Run: `cd src && dotnet run`

### Local Build

```bash
cd src
dotnet build
dotnet run
```
