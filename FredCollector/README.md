# FredCollector

Automated FRED economic data collection service for ATLAS.

## Overview

FredCollector retrieves economic indicators from the Federal Reserve Economic Data (FRED) API and stores them in PostgreSQL/TimescaleDB. It handles scheduling, rate limiting (120 req/min), backfill, and exposes data via REST API and gRPC event streams.

**Scope**: Data collection, storage, and delivery. Series management via admin API. MCP integration via FredCollectorMcp.

## Features

- **Scheduled Collection**: Automates data retrieval using Quartz cron schedules with Federal Reserve holiday exclusions
- **Rate Limiting**: Token bucket rate limiter respects FRED API limits (120 requests/minute)
- **Smart Backfill**: Automatically fills gaps in historical data on startup and on-demand
- **Event Streaming**: Real-time gRPC streams for downstream consumers (ThresholdEngine integration)
- **Admin API**: Add, enable/disable, delete series; trigger manual collection/backfill
- **Series Search**: Search FRED API for new series with filtering and sorting
- **SecMaster Integration**: Automatic instrument registration via gRPC
- **Observability**: Full OpenTelemetry instrumentation (metrics, traces, logs to OTLP)
- **MCP Integration**: Standalone Model Context Protocol server (FredCollectorMcp) for AI assistants

## Supported Indicators

Configured series (config/series.yaml):
- **Recession**: ICSA, IPMAN, UMCSENT, DGS10, UNRATE, FEDFUNDS
- **Liquidity**: SOFR, VIXCLS, DTWEXBGS, BAMLH0A0HYM2, BAMLC0A0CM, WALCL, M2SL
- **Growth**: GDP, GDPC1, INDPRO, TCU, RSXFS, PI, PCE, PSAVERT, CIVPART, HOUST, PERMIT, DGORDER

Additional series can be added via admin API.

## Configuration

Environment variables:

| Variable | Description | Default |
|----------|-------------|---------|
| `ConnectionStrings__AtlasDb` | PostgreSQL connection string | **Required** |
| `FRED_API_KEY` | FRED API key | **Required** |
| `FRED_API_BASE_URL` | FRED API URL | `https://api.stlouisfed.org/fred/` |
| `RATE_LIMITER_CAPACITY` | Token bucket capacity | `120` |
| `RATE_LIMITER_REFILL_RATE` | Tokens per second | `2.0` |
| `OpenTelemetry__OtlpEndpoint` | OTLP collector endpoint | `http://otel-collector:4317` |
| `OpenTelemetry__ServiceName` | Service name | `fred-collector` |
| `SECMASTER_GRPC_ENDPOINT` | SecMaster gRPC endpoint | `http://secmaster:8080` |
| `X_API_KEY` | API key for REST endpoints | **Required for API access** |

## Getting Started

### Development (devcontainer)

1. Open in VS Code and select "Reopen in Container"
2. Configure environment variables (via .env or devcontainer)
3. Run:
   ```bash
   cd /workspace/FredCollector/src
   dotnet run
   ```

### Running with Container

Build and run:
```bash
cd /home/james/ATLAS/FredCollector/src
nerdctl build -f Containerfile -t fred-collector .
nerdctl run -p 8080:8080 -p 5001:5001 \
  -e ConnectionStrings__AtlasDb="Host=postgres;Database=atlas;Username=atlas;Password=..." \
  -e FRED_API_KEY="your_key" \
  -e X_API_KEY="your_api_key" \
  fred-collector
```

### Deployment

Use Ansible playbooks:
```bash
cd /home/james/ATLAS/deployment/ansible
ansible-playbook playbooks/deploy.yml
```

## API Endpoints

### REST API (Port 8080 container, 5001 host)

Requires `X-API-Key` header for authentication.

#### Public Endpoints
| Endpoint | Method | Description |
|----------|--------|-------------|
| `/api/series` | GET | List active series |
| `/api/series/{seriesId}/observations` | GET | Get observations (query: startDate, endDate, limit) |
| `/api/series/{seriesId}/latest` | GET | Get latest observation |
| `/api/series/search` | GET | Search FRED API (query: query, limit, frequency, minPopularity, activeOnly, orderBy) |
| `/health` | GET | Health check (anonymous) |
| `/health/ready` | GET | Readiness check (anonymous) |
| `/health/live` | GET | Liveness check (anonymous) |

#### Admin Endpoints
| Endpoint | Method | Description |
|----------|--------|-------------|
| `/api/admin/series` | POST | Add series (body: {seriesId, category, backfill}) |
| `/api/admin/series` | GET | Get all series (including inactive) |
| `/api/admin/series/{seriesId}/toggle` | PUT | Enable/disable series |
| `/api/admin/series/{seriesId}` | DELETE | Delete series |
| `/api/admin/series/{seriesId}/collect` | POST | Trigger immediate collection |
| `/api/admin/series/{seriesId}/backfill` | POST | Trigger backfill (query: months) |

### gRPC API (Port 8080 container, 5001 host)

Service: `ObservationEventStream` (observation_events.proto)
Consumed by: ThresholdEngine

| Method | Description |
|--------|-------------|
| `SubscribeToEvents` | Stream events in real-time (supports filtering by type, series) |
| `GetEventsSince` | Replay events from timestamp (supports limit) |
| `GetEventsBetween` | Get events in time range |
| `GetLatestEventTime` | Get timestamp of latest event |
| `GetHealth` | Health check with event statistics |

## Project Structure

Flat structure (single project):
```
FredCollector/
├── src/
│   ├── Api/                    # FRED API client
│   ├── Data/                   # EF Core DbContext, repositories
│   ├── Dto/                    # REST API data transfer objects
│   ├── Endpoints/              # Minimal API endpoints (ApiEndpoints, AdminEndpoints)
│   ├── Entities/               # Domain models (SeriesConfig, FredObservation, EventEntity)
│   ├── Events/                 # Event channels
│   ├── Grpc/                   # gRPC service and repositories
│   ├── HealthChecks/           # Database health check
│   ├── Middleware/             # API key authentication
│   ├── Publishers/             # Event publisher
│   ├── RateLimiting/           # Token bucket rate limiter
│   ├── Services/               # Business logic (DataCollection, Backfill, SeriesManagement, SeriesSearch)
│   ├── Telemetry/              # OpenTelemetry meters and activity sources
│   ├── Workers/                # Background workers (DataCollectionScheduler, MarketStatusWorker, InitialDataBackfillWorker)
│   ├── Program.cs              # Application entry point
│   ├── DependencyInjection.cs  # Service registration
│   └── Containerfile           # Multi-stage container build
├── config/
│   └── series.yaml             # Initial series configuration
└── .devcontainer/              # VS Code dev container
