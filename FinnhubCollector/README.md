# FinnhubCollector

Market data collector service for Finnhub API.

## Overview

FinnhubCollector ingests diverse market data including stock quotes, candles, news sentiment, analyst ratings, economic calendars, and earnings calendars from the Finnhub API. It operates under a strict rate limit (60 requests/minute), stores data in TimescaleDB, and streams observation events to downstream consumers via gRPC. The service integrates with SecMaster for instrument registration and exports telemetry to the observability stack.

## Architecture

```mermaid
flowchart LR
    FH[Finnhub API] -->|HTTP/JSON| FC[FinnhubCollector]
    FC -->|Store| DB[(TimescaleDB)]
    FC -->|gRPC Stream| TE[ThresholdEngine]
    FC -->|Register| SM[SecMaster]
    FC -->|OTLP| OC[otel-collector]
```

## Features

- **Data Collection**: Automated background collection of configured series (quotes, sentiment, calendars, analyst data)
- **Live Data API**: On-demand queries directly to Finnhub API bypassing local storage
- **Admin API**: Series management (add, toggle, delete, trigger collection, check status)
- **Rate Limiting**: Token bucket algorithm enforcing 60 requests/minute
- **gRPC Streaming**: Real-time observation events to downstream services
- **SecMaster Integration**: Automatic instrument registration via gRPC
- **Resilience**: Retry policies with exponential backoff and circuit breaker
- **Full Observability**: Distributed tracing, metrics, and structured logging via OTLP

## Configuration

| Variable | Description | Default |
|----------|-------------|---------|
| `ConnectionStrings__AtlasDb` | PostgreSQL/TimescaleDB connection string | Required |
| `Finnhub__ApiKey` | API key from finnhub.io | Required |
| `Finnhub__BaseUrl` | Finnhub API base URL | `https://finnhub.io/api/v1/` |
| `RATE_LIMITER_CAPACITY` | Requests per minute | `60` |
| `OpenTelemetry__OtlpEndpoint` | OTLP collector endpoint | `http://otel-collector:4317` |
| `OpenTelemetry__ServiceName` | Service name for telemetry | `finnhub-collector` |
| `SECMASTER_GRPC_ENDPOINT` | SecMaster gRPC endpoint (optional) | - |
| `Kestrel__HttpPort` | HTTP API port | `8080` |
| `Kestrel__GrpcPort` | gRPC streaming port | `5001` |

## API Endpoints

### Health Checks

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/health` | GET | Full health check with database validation |
| `/health/ready` | GET | Readiness probe (database) |
| `/health/live` | GET | Liveness probe |

### Data API

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/api/series` | GET | Get active series (optionally filter by type) |
| `/api/series/{seriesId}` | GET | Get specific series |
| `/api/quotes/{symbol}` | GET | Get latest quote |
| `/api/quotes/{symbol}/history` | GET | Get quote history |
| `/api/calendar/economic` | GET | Get upcoming economic events |
| `/api/calendar/economic/high-impact` | GET | Get high-impact economic events |
| `/api/calendar/earnings` | GET | Get upcoming earnings |
| `/api/calendar/ipo` | GET | Get upcoming IPOs |
| `/api/sentiment/{symbol}/news` | GET | Get news sentiment |
| `/api/sentiment/{symbol}/insider` | GET | Get insider sentiment |
| `/api/analyst/{symbol}/recommendations` | GET | Get analyst recommendations |
| `/api/analyst/{symbol}/price-target` | GET | Get price target |
| `/api/company/{symbol}` | GET | Get company profile |
| `/api/market/status` | GET | Get market status |
| `/api/symbols/search` | GET | Search symbols (query: q) |
| `/api/search` | GET | Unified search for SecMaster gateway |
| `/api/discover` | GET | Upstream discovery for SecMaster catalog |

### Live Data API

Direct Finnhub API queries (bypasses local storage):

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/api/live/quote/{symbol}` | GET | Live quote |
| `/api/live/candles/{symbol}` | GET | Live OHLCV candles |
| `/api/live/profile/{symbol}` | GET | Live company profile |
| `/api/live/recommendation/{symbol}` | GET | Live analyst recommendations |
| `/api/live/price-target/{symbol}` | GET | Live price target |
| `/api/live/news-sentiment/{symbol}` | GET | Live news sentiment |
| `/api/live/peers/{symbol}` | GET | Live company peers |

### Admin API

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/api/admin/series` | GET | Get all series (including inactive) |
| `/api/admin/series` | POST | Add new series |
| `/api/admin/series/{seriesId}/toggle` | PUT | Enable/disable series |
| `/api/admin/series/{seriesId}` | DELETE | Delete series |
| `/api/admin/series/{seriesId}/collect` | POST | Trigger immediate collection |
| `/api/admin/series/{seriesId}/collect/status` | GET | Get collection status |

### Documentation

| Endpoint | Description |
|----------|-------------|
| `/swagger` | Interactive API documentation |

### gRPC API

Service: `ObservationEventStream` (port 5001)

| Method | Type | Description |
|--------|------|-------------|
| `SubscribeToEvents` | Server streaming | Real-time event subscription from checkpoint |
| `GetEventsSince` | Server streaming | Historical events since timestamp |
| `GetEventsBetween` | Server streaming | Historical events within time range |
| `GetLatestEventTime` | Unary | Get timestamp of most recent event |
| `GetHealth` | Unary | Service health with event statistics |

## Project Structure

```
FinnhubCollector/
├── src/
│   ├── Api/              # Finnhub HTTP client
│   ├── Data/             # EF Core DbContext, repositories
│   ├── Endpoints/        # REST endpoints (API, Live, Admin)
│   ├── Events/           # Observation channel
│   ├── Grpc/             # gRPC services and repositories
│   ├── HealthChecks/     # Database health check
│   ├── Interfaces/       # Service contracts
│   ├── Models/           # Domain models
│   ├── Services/         # Application services
│   ├── Telemetry/        # OpenTelemetry instrumentation
│   ├── Workers/          # Background collection worker
│   └── Program.cs        # Application entry point
├── tests/                # Unit tests
└── .devcontainer/        # Development container
```

## Development

### Prerequisites

- .NET 9 SDK
- VS Code with Dev Containers extension
- Docker/nerdctl

### Getting Started

```bash
# Open in VS Code and select "Reopen in Container"
cd /workspace/FinnhubCollector/src
dotnet run
```

### Build Commands

```bash
# Compile
.devcontainer/compile.sh

# Build container image
.devcontainer/build.sh
```

## Deployment

```bash
ansible-playbook playbooks/deploy.yml --tags finnhub-collector
```

## Ports

| Port | Type | Description |
|------|------|-------------|
| 8080 | HTTP (internal) | REST API, health checks |
| 5001 | HTTP/2 (internal) | gRPC event stream |

## See Also

- [ThresholdEngine](../ThresholdEngine/README.md) - Consumes observation events
- [SecMaster](../SecMaster/README.md) - Instrument registration
- [Events](../Events/README.md) - Shared gRPC event contracts
- [FinnhubMcp](mcp/README.md) - MCP server for AI assistants
