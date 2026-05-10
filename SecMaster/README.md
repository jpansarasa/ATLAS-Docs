# SecMaster

Centralized instrument metadata and intelligent source resolution service for ATLAS.

## Overview

SecMaster provides a single source of truth for financial instrument definitions and context-aware routing to data sources. Collectors register their series capabilities via gRPC, and consumers resolve symbols to the optimal data source based on frequency, latency, and collector preferences. Includes hybrid search combining SQL, fuzzy matching, vector similarity, and RAG-powered natural language queries via Ollama.

## Architecture

```mermaid
flowchart TD
    subgraph Collectors
        FC[FredCollector]
        AV[AlphaVantageCollector]
        FH[FinnhubCollector]
        OFR[OfrCollector]
    end

    subgraph SecMaster
        API[REST :8080 + gRPC :5001]
        REG[Registration]
        RES[Resolution]
        SEM[Semantic Search]
        DB[(TimescaleDB + pgvector)]
    end

    subgraph Consumers
        TE[ThresholdEngine]
        MCP[SecMasterMcp]
    end

    FC & AV & FH & OFR -->|Register Series| API
    API --> REG --> DB
    TE -->|Resolve Symbol| API
    MCP -->|Query Catalog| API
    API --> RES & SEM --> DB
    SEM <-->|Embeddings| OLLAMA[Ollama]
```

Collectors register series at startup via gRPC streaming (fire-and-forget). Consumers resolve symbols to optimal data sources. Semantic search uses Ollama for embeddings and RAG synthesis.

## Features

- **Instrument Registry**: Central catalog of financial instruments with metadata and source mappings
- **Context-Aware Resolution**: Routes lookups to optimal source by frequency, latency, and priority
- **Hybrid Search**: SQL exact match, fuzzy text, vector similarity, and RAG synthesis pipeline
- **Collector Gateway**: Unified API for searching and managing series across all data collectors
- **Catalog Discovery**: Search upstream collectors for new instruments and promote to collection
- **Embedding Backfill**: Background service auto-generates vector embeddings for new instruments

## Configuration

| Variable | Description | Default |
|----------|-------------|---------|
| `ConnectionStrings__SecMaster` | PostgreSQL connection string | Required |
| `Ollama__Url` | Ollama API endpoint for generation | `http://ollama-gpu:11434` |
| `Ollama__EmbeddingUrl` | Ollama API endpoint for embeddings | `http://ollama-cpu:11434` |
| `Ollama__EmbeddingModel` | Model for vector embeddings | `bge-m3` |
| `Ollama__GenerationModel` | Model for RAG synthesis | `qwen2.5:32b-instruct` |
| `SemanticSearch__VectorHighConfidenceThreshold` | High confidence similarity threshold | `0.8` |
| `SemanticSearch__DefaultMinScore` | Default minimum similarity score | `0.5` |
| `Collectors__FredCollectorUrl` | FRED collector URL | `http://fred-collector:8080` |
| `Collectors__FinnhubCollectorUrl` | Finnhub collector URL | `http://finnhub-collector:8080` |
| `Collectors__OfrCollectorUrl` | OFR collector URL | `http://ofr-collector:8080` |
| `Collectors__AlphaVantageCollectorUrl` | AlphaVantage collector URL | `http://alphavantage-collector:8080` |
| `InstrumentConfiguration__ConfigDirectory` | Path to instrument config files | `/app/config` |
| `OpenTelemetry:OtlpEndpoint` | OTLP exporter target (Loki/Tempo/Prometheus via collector) | `http://otel-collector:4317` |
| `OpenTelemetry:ServiceName` | Service name tag in OTEL resource attributes | `secmaster` |
| `OpenTelemetry:ServiceVersion` | Service version tag in OTEL resource attributes | `1.0.0` |

## API Endpoints

REST endpoints are split across the following endpoint groups under `src/Endpoints/`:

- **InstrumentEndpoints** (`/api/instruments`) — CRUD over instruments + source mappings.
- **ResolutionEndpoints** (`/api/resolve`) — symbol → source resolution (single, batch, reverse).
- **RegistrationEndpoints** (`/api/register`) — collector-side series registration.
- **SearchEndpoints** (`/api/search`) — fuzzy text search.
- **SemanticSearchEndpoints** (`/api/semantic`) — vector / hybrid / RAG.
- **CatalogEndpoints** (`/api/catalog`) — upstream discovery + promotion.
- **CollectorEndpoints** (`/api/collectors`) — gateway for collector series management.
- **AtlasSectorEndpoints** (`/api/atlas-sectors`) — 11-sector ATLAS taxonomy lookups + NAICS rollups.
- **NaicsEndpoints** (`/api/naics`) — NAICS code hierarchy (browse + resolve).
- **SignalIdentitiesEndpoints** (`/api/signal-identities`) — signal-identity dedup grouping for cross-source equivalents.
- **EdgarEndpoints** (`/api/edgar`) — EDGAR filer registry (CIK / ticker lookup, refresh).
- **IdentifiersEndpoints** (`/api/identifiers`) — generic identifier-resolution surface.
- **InstrumentOverrideEndpoints** (`/api/instruments/{id}/sector-override`, `/api/admin/sector-overrides`) — manual sector override lifecycle.
- **AdminEndpoints** (`/api/admin`) — catalog coverage report + dedup maintenance ops.

### REST API (Port 8080)

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/api/instruments` | GET | List all instruments |
| `/api/instruments` | POST | Create instrument |
| `/api/instruments/{id}` | GET/PUT/DELETE | CRUD by ID |
| `/api/instruments/{id}/sources` | GET | Get source mappings for instrument |
| `/api/instruments/by-symbol/{symbol}` | GET | Get by symbol |
| `/api/instruments/{instrumentId}/sector-override` | GET/POST | Get active sector override / set new override |
| `/api/instruments/{instrumentId}/sector-overrides/history` | GET | Override history for an instrument |
| `/api/instruments/sector-overrides/{overrideId}` | DELETE | Revoke an active sector override |
| `/api/resolve/{symbol}` | GET | Resolve symbol to best data source |
| `/api/resolve` | POST | Resolve with context (frequency, lag, preference) |
| `/api/resolve/batch` | GET | Batch resolve multiple symbols |
| `/api/resolve/lookup/{collector}/{sourceId}` | GET | Reverse lookup by collector and source ID |
| `/api/register` | POST | Register a data source from a collector |
| `/api/search` | GET | Fuzzy text search |
| `/api/semantic/search` | GET | Vector similarity search |
| `/api/semantic/resolve` | GET | Hybrid resolution (SQL, fuzzy, vector, RAG) |
| `/api/semantic/resolve-local` | GET | Hybrid resolution restricted to local catalog |
| `/api/semantic/ask` | POST | Natural language Q&A with RAG |
| `/api/semantic/embed/{instrumentId}` | POST | Force-embed a single instrument |
| `/api/semantic/embed/backfill` | POST | Backfill embeddings for missing/stale rows |
| `/api/catalog/search` | GET | Search catalog with upstream discovery |
| `/api/catalog/{id}/promote` | POST | Promote discovered instrument to collection |
| `/api/collectors/search` | GET | Smart search across all collectors |
| `/api/collectors/{collector}/series` | GET | List series for a collector |
| `/api/collectors/{collector}/series` | POST | Add series to a collector |
| `/api/collectors/{collector}/series/{id}/toggle` | PUT | Toggle series active status |
| `/api/collectors/{collector}/series/{id}` | DELETE | Remove series from collector |
| `/api/atlas-sectors/` | GET | List the 11 ATLAS sectors |
| `/api/atlas-sectors/resolve` | GET | Resolve a symbol/identifier to its ATLAS sector |
| `/api/atlas-sectors/{sectorCode}/naics` | GET | List NAICS codes that roll up to a sector |
| `/api/naics/` | GET | Browse NAICS codes by prefix |
| `/api/naics/{parentCode}/children` | GET | List child NAICS codes under a parent |
| `/api/naics/{code}` | GET | Resolve a NAICS code (with rollup target) |
| `/api/signal-identities/` | GET | List signal identities |
| `/api/signal-identities/by-alias` | GET | Lookup signal identity by alias |
| `/api/signal-identities/by-category/{category}` | GET | List by signal category |
| `/api/signal-identities/{id}` | GET | Signal-identity detail |
| `/api/signal-identities/{id}/dedup` | GET | Dedup grouping (cross-source equivalents) |
| `/api/edgar/filers/{cik}` | GET | EDGAR filer by CIK |
| `/api/edgar/filers/by-ticker/{ticker}` | GET | EDGAR filer by ticker |
| `/api/edgar/refresh` | POST | Refresh EDGAR filer registry from upstream |
| `/api/identifiers/resolve` | GET | Resolve generic identifier (CUSIP, ISIN, etc.) |
| `/api/admin/catalog/coverage` | GET | Catalog coverage report |
| `/api/admin/catalog/dedupe-fred-hijacks` | POST | Dedup catalog FRED hijack rows |
| `/api/admin/sector-overrides` | GET | List all active sector overrides |

### gRPC Services (Port 5001)

| Service | Method | Description |
|---------|--------|-------------|
| `SecMasterRegistry` | `RegisterSeries` | Register single series (fire-and-forget) |
| `SecMasterRegistry` | `RegisterSeriesBatch` | Batch registration via client streaming |
| `SecMasterResolver` | `ResolveSymbol` | Resolve symbol to best data source |
| `SecMasterResolver` | `ResolveBatch` | Batch resolution via server streaming |
| `SecMasterResolver` | `LookupSource` | Reverse lookup by collector and source ID |

## Project Structure

```
SecMaster/
├── src/
│   ├── Configuration/    # Options classes (collectors, semantic search)
│   ├── Data/
│   │   ├── Entities/     # 11 entities — see list below
│   │   ├── Migrations/   # EF migrations (baseline + 13 since)
│   │   └── Repositories/ # query/persistence helpers
│   ├── Endpoints/        # REST API handlers (see API Endpoints section for the full list)
│   ├── Grpc/             # gRPC service implementations
│   ├── HealthChecks/     # Database health check
│   ├── Models/           # Resolution models
│   ├── Services/         # Registration, resolution, semantic search, collector clients
│   └── Telemetry/        # OpenTelemetry metrics and tracing
├── tests/                # Unit and integration tests
├── mcp/                  # MCP server for Claude Code integration
├── config/               # Instrument configuration files
└── .devcontainer/        # Dev container config
```

### Entities (`src/Data/Entities/`)

- `InstrumentEntity` — primary instrument row (symbol, name, asset class, NAICS + ATLAS sector tags).
- `InstrumentEmbeddingEntity` — pgvector embedding (bge-m3, 1024-dim) for semantic search.
- `InstrumentSectorOverrideEntity` — manual ATLAS-sector override (lifecycle-tracked, FK to instrument).
- `SourceMappingEntity` — instrument ↔ collector source-id mapping (with frequency/lag metadata).
- `AliasEntity` — alternate symbols / tickers for an instrument.
- `AtlasSectorEntity` — 11-row ATLAS sector taxonomy (FK target).
- `NaicsCodeEntity` — full NAICS 2022 hierarchy.
- `NaicsSectorRollupEntity` — NAICS → ATLAS sector rollup mappings.
- `SignalIdentityEntity` — cross-source signal-identity grouping (dedup target).
- `EdgarFilerEntity` — EDGAR filer registry (CIK + tickers).
- `MappingVersionEntity` — version-pinned mappings for reproducible resolution.

## Development

### Prerequisites
- VS Code with Dev Containers extension
- Access to shared infrastructure (PostgreSQL, observability stack)

### Getting Started

1. Open in VS Code: `code SecMaster/`
2. Reopen in Container (Cmd/Ctrl+Shift+P -> "Dev Containers: Reopen in Container")
3. Build and test: `.devcontainer/compile.sh`
4. Build without tests: `.devcontainer/compile.sh --no-test`

### Build Container

```bash
.devcontainer/build.sh
```

## Deployment

```bash
ansible-playbook playbooks/deploy.yml --tags secmaster
```

## Ports

| Port | Type | Description |
|------|------|-------------|
| 8080 | Internal | REST API (HTTP/1.1 + HTTP/2) |
| 5001 | Internal | gRPC (HTTP/2 only) |

No host-mapped ports. All access is container-to-container only. The MCP server (`secmaster-mcp`) exposes port 3107 for external access.

## See Also

- [SecMaster MCP](mcp/README.md) - MCP server for Claude Code integration
- [ThresholdEngine](../ThresholdEngine/README.md) - Primary consumer of resolution services
- [FredCollector](../FredCollector/README.md) - Economic data collector
- [FinnhubCollector](../FinnhubCollector/README.md) - Stock quotes and sentiment
- [OfrCollector](../OfrCollector/README.md) - Financial stability data
- [AlphaVantageCollector](../AlphaVantageCollector/README.md) - Market data collector
