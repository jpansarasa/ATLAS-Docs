# FinnhubCollector

Market data collector service for Finnhub API.

## Overview

FinnhubCollector ingests diverse market data including stock quotes, candles, news sentiment, analyst ratings, economic calendars, and earnings calendars from the Finnhub API. It operates under a strict rate limit (60 requests/minute), stores data in TimescaleDB, and streams observations to downstream consumers via gRPC.

## Features

- **Data Collection**: Automated collection of configured series (quotes, sentiment, calendars)
- **Live Data Endpoints**: On-demand queries directly to Finnhub API (quotes, candles, profiles, recommendations, price targets, sentiment, peers)
- **Admin API**: Series management (add, toggle, delete, trigger collection)
- **Rate Limiting**: Token bucket algorithm (60 req/min default)
- **gRPC Streaming**: Real-time observation events to downstream services
- **OpenTelemetry**: Distributed tracing and metrics with OTLP export
- **Health Checks**: Liveness and readiness probes with database validation

## Data Types

Collected and stored data:
- **Quotes**: Real-time stock prices
- **Candles**: Historical OHLCV data
- **News Sentiment**: Company sentiment scores from news articles
- **Insider Sentiment**: Insider trading activity sentiment
- **Analyst Recommendations**: Buy/Sell/Hold ratings
- **Price Targets**: Analyst price target consensus
- **Economic Calendar**: Global economic events
- **Earnings Calendar**: Upcoming earnings releases
- **IPO Calendar**: Upcoming IPOs
- **Company Profiles**: Company metadata

## Configuration

Environment variables:

| Variable | Description | Default |
|----------|-------------|---------|
| `ConnectionStrings__Atlas` | PostgreSQL connection | `Host=timescaledb;...` |
| `Finnhub__ApiKey` | API Key from finnhub.io | **Required** |
| `Finnhub__RateLimit` | Requests per minute | `60` |
| `OpenTelemetry__OtlpEndpoint` | OTLP collector endpoint | `http://otel-collector:4317` |
| `OpenTelemetry__ServiceName` | Service name for telemetry | `finnhub-collector-service` |

## API Endpoints

### REST API (Port 8080)

#### Health
- `GET /health` - Liveness probe
- `GET /health/ready` - Readiness check (DB validation)
- `GET /health/live` - Simple liveness endpoint

#### Swagger
- `GET /swagger` - Interactive API documentation

#### Data API (`/api`)
- `GET /api/series` - Get all configured series (optionally filtered by type)
- `GET /api/series/{seriesId}` - Get specific series
- `GET /api/quotes/{symbol}` - Get latest quote
- `GET /api/quotes/{symbol}/history` - Get quote history
- `GET /api/calendar/economic` - Get upcoming economic events
- `GET /api/calendar/economic/high-impact` - Get high-impact economic events
- `GET /api/calendar/earnings` - Get upcoming earnings
- `GET /api/calendar/ipo` - Get upcoming IPOs
- `GET /api/sentiment/{symbol}/news` - Get news sentiment
- `GET /api/sentiment/{symbol}/insider` - Get insider sentiment
- `GET /api/analyst/{symbol}/recommendations` - Get analyst recommendations
- `GET /api/analyst/{symbol}/price-target` - Get price target
- `GET /api/company/{symbol}` - Get company profile
- `GET /api/market/status` - Get market status
- `GET /api/symbols/search?q={query}` - Search symbols

#### Live Data API (`/api/live`)
Direct Finnhub API queries (bypasses local storage):
- `GET /api/live/quote/{symbol}` - Live quote
- `GET /api/live/candles/{symbol}` - Live candles (historical OHLCV)
- `GET /api/live/profile/{symbol}` - Live company profile
- `GET /api/live/recommendation/{symbol}` - Live recommendations
- `GET /api/live/price-target/{symbol}` - Live price target
- `GET /api/live/news-sentiment/{symbol}` - Live news sentiment
- `GET /api/live/peers/{symbol}` - Live company peers

#### Admin API (`/api/admin`)
- `POST /api/admin/series` - Add new series
- `PUT /api/admin/series/{seriesId}/toggle` - Enable/disable series
- `DELETE /api/admin/series/{seriesId}` - Delete series
- `GET /api/admin/series` - Get all series (including inactive)
- `POST /api/admin/series/{seriesId}/collect` - Trigger immediate collection

### gRPC API (Port 5001)

**Service**: `ObservationEventService`

- `SubscribeToEvents`: Server-streaming RPC that emits `SeriesCollectedEvent` messages in real-time as data is collected

## Development

### Using Dev Container

1. Open in VS Code and select "Reopen in Container"
2. Configure API key in `.env`:
   ```bash
   Finnhub__ApiKey=your_api_key_here
   ```
3. Run the service:
   ```bash
   cd /home/james/ATLAS/FinnhubCollector/src
   dotnet run
   ```

### Local Build

Build the Docker image:
```bash
cd /home/james/ATLAS
nerdctl build -f FinnhubCollector/src/Containerfile -t finnhub-collector:latest .
```

## Project Structure

```
FinnhubCollector/
├── src/
│   ├── Api/              # Finnhub API client
│   ├── Data/             # EF Core DbContext, repositories, migrations
│   ├── Endpoints/        # REST API endpoints (API, Live, Admin)
│   ├── Events/           # Observation channel
│   ├── Grpc/             # gRPC services and repositories
│   ├── HealthChecks/     # Database health check
│   ├── Interfaces/       # Service interfaces
│   ├── Models/           # Domain models
│   ├── Services/         # Application services (rate limiter, series management)
│   ├── Telemetry/        # OpenTelemetry activity source and meter
│   ├── Workers/          # Background collection worker
│   └── Program.cs        # Application entry point
├── tests/                # Unit tests
└── migrations/           # SQL migration scripts

