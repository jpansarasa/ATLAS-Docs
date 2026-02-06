# OfrCollector

Data collector service for Office of Financial Research (OFR) public datasets.

## Overview

OfrCollector retrieves financial stress and funding market data from the OFR public APIs (FSI, STFM, HFM) and stores them in TimescaleDB. It handles scheduled collection, backfill, and exposes data via REST API and gRPC event streams for downstream consumption by ThresholdEngine.

## Architecture

```mermaid
flowchart LR
    OFR[OFR Public APIs<br/>financialresearch.gov] -->|HTTP/JSON/CSV| OC[OfrCollector]
    OC -->|Store| DB[(TimescaleDB)]
    OC -->|gRPC Stream| TE[ThresholdEngine]
    OC -->|Register| SM[SecMaster]
    OC -->|Metrics/Traces| OTEL[OpenTelemetry<br/>Collector]
```

The service pulls data from three OFR sources (FSI via CSV, STFM and HFM via JSON API), persists observations in TimescaleDB, and streams events to ThresholdEngine over gRPC. New instruments are registered with SecMaster on discovery.

## Features

- **Multi-source Collection**: FSI (Financial Stress Index), STFM (Short-term Funding Monitor), HFM (Hedge Fund Monitor)
- **Scheduled Collection**: Background workers with configurable intervals per dataset
- **Smart Backfill**: Fill historical data gaps on-demand with date range control
- **Event Streaming**: Real-time gRPC streams for downstream consumers (ThresholdEngine)
- **Series Management**: Admin API to add, delete, toggle, and trigger collection per series
- **Search API**: Search across all OFR data types by keyword
- **SecMaster Integration**: Automatic instrument registration via gRPC (fire-and-forget)
- **Resilience**: Polly retry, circuit breaker, and timeout policies on all HTTP clients
- **Full Observability**: OpenTelemetry instrumentation (metrics, traces, logs via OTLP)

## Configuration

| Variable | Description | Default |
|----------|-------------|---------|
| `ConnectionStrings__AtlasDb` | PostgreSQL/TimescaleDB connection string | **Required** |
| `OpenTelemetry__OtlpEndpoint` | OTLP collector endpoint | `http://otel-collector:4317` |
| `OpenTelemetry__ServiceName` | Service name for telemetry | `ofr-collector-service` |
| `OpenTelemetry__ServiceVersion` | Service version for telemetry | `1.0.0` |
| `SECMASTER_GRPC_ENDPOINT` | SecMaster gRPC endpoint (optional) | - |
| `SeriesConfig__Directory` | Directory for series config files | `/app/config` |
| `Kestrel__HttpPort` | HTTP listener port | `8080` |
| `Kestrel__GrpcPort` | gRPC listener port | `5001` |

## API Endpoints

### REST API (Port 8080)

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/api/fsi/latest` | GET | Get latest FSI value |
| `/api/fsi/history` | GET | Get FSI history (query: startDate, endDate, limit) |
| `/api/stfm/series` | GET | List active STFM series |
| `/api/stfm/{mnemonic}/latest` | GET | Get latest STFM observation |
| `/api/stfm/{mnemonic}/observations` | GET | Get STFM observations (query: startDate, endDate, limit) |
| `/api/hfm/series` | GET | List active HFM series |
| `/api/hfm/{mnemonic}/latest` | GET | Get latest HFM observation |
| `/api/hfm/{mnemonic}/observations` | GET | Get HFM observations (query: startDate, endDate, limit) |
| `/api/categories` | GET | List available data categories |
| `/api/search` | GET | Search all OFR data (query: q, type, limit) |
| `/api/health` | GET | Service health |

### Admin API (Port 8080)

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/api/admin/fsi/collect` | POST | Trigger FSI collection |
| `/api/admin/fsi/backfill` | POST | Backfill FSI history (query: months) |
| `/api/admin/stfm/{dataset}/collect` | POST | Trigger STFM dataset collection |
| `/api/admin/stfm/priority/collect` | POST | Trigger priority STFM series collection |
| `/api/admin/stfm/series` | GET/POST | List or add STFM series |
| `/api/admin/stfm/series/{mnemonic}` | DELETE | Delete STFM series |
| `/api/admin/stfm/series/{mnemonic}/toggle` | PUT | Toggle STFM series active status |
| `/api/admin/stfm/series/{mnemonic}/collect` | POST | Collect specific STFM series |
| `/api/admin/stfm/series/{mnemonic}/backfill` | POST | Backfill STFM series history |
| `/api/admin/hfm/{dataset}/collect` | POST | Trigger HFM dataset collection |
| `/api/admin/hfm/all/collect` | POST | Trigger all HFM datasets collection |
| `/api/admin/hfm/series` | GET/POST | List or add HFM series |
| `/api/admin/hfm/series/{mnemonic}` | DELETE | Delete HFM series |
| `/api/admin/hfm/series/{mnemonic}/toggle` | PUT | Toggle HFM series active status |
| `/api/admin/hfm/series/{mnemonic}/collect` | POST | Collect specific HFM series |
| `/api/admin/hfm/series/{mnemonic}/backfill` | POST | Backfill HFM series history |

### Health Checks

| Endpoint | Description |
|----------|-------------|
| `/health` | Full health check with database status |
| `/health/ready` | Readiness probe (includes database check) |
| `/health/live` | Liveness probe (always healthy) |

### gRPC Services (Port 5001)

| Service | Method | Description |
|---------|--------|-------------|
| `ObservationEventStream` | `SubscribeToEvents` | Stream events in real-time (long-running) |
| `ObservationEventStream` | `GetEventsSince` | Replay events from timestamp |
| `ObservationEventStream` | `GetEventsBetween` | Get events in time range |
| `ObservationEventStream` | `GetLatestEventTime` | Get timestamp of latest event |
| `ObservationEventStream` | `GetHealth` | Health check with event statistics |

## Project Structure

```
OfrCollector/
├── src/
│   ├── Api/                    # OFR API clients (FSI CSV, STFM JSON, HFM JSON)
│   ├── Configuration/          # Series configuration provider
│   ├── Data/                   # EF Core DbContext, repositories
│   ├── Endpoints/              # REST API and Admin endpoints
│   ├── Enums/                  # Dataset and category enumerations
│   ├── Grpc/                   # gRPC EventStreamService and repositories
│   ├── HealthChecks/           # Database health check
│   ├── Models/                 # Domain models (FSI, STFM, HFM)
│   ├── Services/               # Collection and series management services
│   ├── Telemetry/              # OpenTelemetry meters and activity sources
│   └── Workers/                # Background collection workers (FSI, STFM, HFM)
├── tests/                      # Unit and integration tests
├── config/                     # Series configuration (stfm-series.json, hfm-series.json)
├── mcp/                        # MCP server for AI assistants
└── .devcontainer/              # Dev container config
```

## Development

### Prerequisites

- VS Code with Dev Containers extension
- Access to shared infrastructure (TimescaleDB, observability stack)

### Getting Started

1. Open in VS Code: `code OfrCollector/`
2. Reopen in Container (Cmd/Ctrl+Shift+P -> "Dev Containers: Reopen in Container")
3. Build: `dotnet build`
4. Run: `cd /workspace/OfrCollector/src && dotnet run`

### Compile

```bash
.devcontainer/compile.sh
```

### Build Container Image

```bash
.devcontainer/build.sh
```

## Deployment

```bash
ansible-playbook playbooks/deploy.yml --tags ofr-collector
```

## Ports

| Port | Type | Description |
|------|------|-------------|
| 8080 | HTTP (container) | REST API, health checks |
| 5001 | HTTP/2 (container) | gRPC event stream |

No external host port mapping -- internal API only.

## See Also

- [ThresholdEngine](../ThresholdEngine/README.md) - Consumes observation events via gRPC
- [SecMaster](../SecMaster/README.md) - Instrument registration
- [Events](../Events/README.md) - Shared gRPC event contracts
- [OfrMcp](./mcp/README.md) - MCP server for AI assistants
