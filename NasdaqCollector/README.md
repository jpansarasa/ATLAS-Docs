# NasdaqCollector

Collector service for financial time-series data from Nasdaq Data Link API.

## Overview

NasdaqCollector ingests daily financial data from Nasdaq Data Link (formerly Quandl). Configured series are collected automatically on a 6-hour schedule, respecting NYSE/Nasdaq market holidays. Data is stored in TimescaleDB and streamed in real-time via gRPC to downstream consumers.

## Key Features

- **Configurable Series**: Dynamically add/remove datasets via admin API
- **Market-Aware Scheduling**: Skips collection on NYSE/Nasdaq holidays
- **Resilient Collection**: Automatic retries with exponential backoff
- **Event Streaming**: Real-time gRPC stream of new observations
- **Efficient Storage**: TimescaleDB hypertables for time-series data
- **Full Observability**: OpenTelemetry traces, metrics, and structured logs

## Configuration

Environment variables:

| Variable | Description | Default |
|----------|-------------|---------|
| `ConnectionStrings__AtlasDb` | PostgreSQL/TimescaleDB connection | Required |
| `Nasdaq__ApiKey` | Nasdaq Data Link API key | Required |
| `OpenTelemetry__OtlpEndpoint` | OTLP collector endpoint | `http://otel-collector:4317` |
| `OpenTelemetry__ServiceName` | Service name for telemetry | `nasdaq-collector` |

## API Endpoints

### Health Checks (HTTP Port 8080)

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/health/live` | GET | Liveness probe |
| `/health/ready` | GET | Readiness check (DB connected) |
| `/health` | GET | Full health with all checks |

### Admin API (HTTP Port 8080)

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/api/admin/search?q={query}` | GET | Search Nasdaq Data Link datasets |
| `/api/admin/series` | GET | List all configured series |
| `/api/admin/series` | POST | Add new series to collect |
| `/api/admin/series/{seriesId}/toggle` | PUT | Enable/disable a series |
| `/api/admin/series/{seriesId}` | DELETE | Delete a series |

### gRPC Event Stream (Port 5009)

Service: `ObservationEventStream`

| Method | Description |
|--------|-------------|
| `SubscribeToEvents` | Real-time stream of events (long-lived connection) |
| `GetEventsSince` | Retrieve events since a timestamp |
| `GetEventsBetween` | Retrieve events in a time range |
| `GetLatestEventTime` | Get timestamp of most recent event |
| `GetHealth` | Health check with event statistics |

## Data Collection

### Workflow

1. **CollectionWorker** runs every 6 hours
2. Checks if today is a market holiday (NYSE/Nasdaq calendar)
3. If trading day, collects all active series
4. For each series, fetches observations since last collection
5. Stores observations in TimescaleDB
6. Publishes events to gRPC stream

### Adding a Series

Search for available datasets:
```bash
curl 'http://localhost:8080/api/admin/search?q=gold+price'
```

Add a series:
```bash
curl -X POST http://localhost:8080/api/admin/series \
  -H 'Content-Type: application/json' \
  -d '{
    "databaseCode": "LBMA",
    "datasetCode": "GOLD",
    "title": "Gold Price: London Fixing",
    "category": "Commodities",
    "valueColumn": "USD (AM)",
    "priority": 10
  }'
```

## Project Structure

```
NasdaqCollector/
├── src/
│   ├── Data/                   # EF Core DbContext and configurations
│   ├── Grpc/                   # gRPC services and event repository
│   ├── Interfaces/             # Service interfaces
│   ├── Models/                 # Domain models (NasdaqSeries, NasdaqObservation)
│   ├── Services/               # Collection and management services
│   ├── Workers/                # Background collection worker
│   ├── Telemetry/              # OpenTelemetry metrics and activity sources
│   ├── NasdaqApiClient.cs      # Nasdaq Data Link API client
│   ├── NasdaqRepository.cs     # Data access layer
│   └── Program.cs              # Application startup
├── tests/
│   └── NasdaqCollector.UnitTests/
├── migrations/                 # SQL schema migrations
└── Containerfile               # Container build definition
```

## Development

### Using Dev Container

```bash
# Open in VS Code and select "Reopen in Container"
# Infrastructure starts automatically
cd src
dotnet run
```

### Local Build

```bash
cd src
dotnet build
dotnet test ../tests
```

## Deployment

Deployed via Ansible:

```bash
cd deployment/ansible
ansible-playbook playbooks/deploy.yml
```

## See Also

- [CalendarService](../CalendarService/README.md) - Market calendar provider
- [Events](../Events/README.md) - Shared gRPC event contracts
