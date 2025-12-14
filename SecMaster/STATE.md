# SecMaster Implementation State

## GOAL
Centralized instrument metadata and intelligent source resolution service with semantic search capabilities.

### Accept Criteria
- ✓ Standalone service with PostgreSQL/TimescaleDB
- ✓ REST API + gRPC services for registration/resolution
- ✓ MCP server for AI assistants
- ✓ EF Core migrations for schema management
- ✓ Semantic vector search with pgvector + Ollama embeddings
- ✓ Hybrid resolution (SQL → Fuzzy → Vector → RAG)
- ✓ Fire-and-forget collector registration
- ✓ Context-dependent source selection (frequency, lag, preference)

### Current Status
SecMaster is **deployed and operational** with:
- REST API on port 5017 (host) / 8080 (internal)
- gRPC services (RegistryGrpcService, ResolverGrpcService)
- SecMasterMcp on port 3107
- Semantic search using nomic-embed-text (768d vectors)
- RAG synthesis using llama3.2:3b
- pgvector HNSW indexing for fast similarity search
- Collectors integrated and registering series
- Background embedding service for new instruments

---

## ARCH

### Database Schema
```
instruments (id, symbol, name, description, asset_class, frequency, metadata JSONB, external IDs, ...)
source_mappings (instrument_id, collector, source_id, frequency, priority, lag_days, is_primary)
aliases (instrument_id, alias, alias_type)
instrument_embeddings (id, instrument_id, model, embedding vector(768), embedded_text, created_at)
```

### Resolution Strategies

**Standard Resolution:**
1. Find instrument by symbol or alias
2. Filter sources by frequency requirement (hierarchy: intraday > daily > weekly > monthly > quarterly > annual)
3. Filter by max lag if specified
4. Return by priority (primary first, then priority number, then lag)

**Hybrid Resolution (SQL → Fuzzy → Vector → RAG):**
1. Try exact SQL match on symbol/alias
2. Fall back to fuzzy trigram search (pg_trgm)
3. Fall back to vector similarity (pgvector cosine distance)
4. Fall back to RAG synthesis with LLM if no clear match

### Key Decisions
- EF Core migrations for schema management (applied on startup)
- PostgreSQL with pgvector extension for vector search
- Trigram indexes (pg_trgm) for fuzzy text search
- HNSW indexes for fast approximate nearest neighbor search
- Ollama for local embeddings/generation (nomic-embed-text, llama3.2:3b)
- Kestrel configured for HTTP/1.1 (REST) + HTTP/2 (gRPC) on same port
- Background service for automatic embedding generation

---

## STATUS

### ✓ Core Features (Complete)
- [x] SecMaster service structure and Containerfile
- [x] EF Core entities (Instrument, SourceMapping, Alias, InstrumentEmbedding)
- [x] Database migrations with pgvector extension setup
- [x] InstrumentRepository with CRUD, search, fuzzy text search
- [x] ResolutionService with frequency hierarchy
- [x] RegistrationService for collector auto-registration
- [x] REST API endpoints (instruments, resolve, register, search, health)
- [x] gRPC services (RegistryGrpcService, ResolverGrpcService)
- [x] OpenTelemetry tracing and metrics
- [x] Health checks endpoint
- [x] Deployed to production (port 5017)

### ✓ Semantic Search (Complete)
- [x] OllamaClient for embedding generation
- [x] EmbeddingService with vector search
- [x] InstrumentEmbedding entity with pgvector support
- [x] HNSW vector index for fast similarity search
- [x] RagService for natural language synthesis
- [x] HybridResolutionService (SQL→Fuzzy→Vector→RAG)
- [x] Semantic endpoints (/api/semantic/search, /ask, /resolve, /embed)
- [x] EmbeddingBackgroundService for automatic embedding generation
- [x] Integration with Ollama GPU service (nomic-embed-text, llama3.2:3b)

### ✓ MCP Server (Complete)
- [x] SecMasterMcp service structure
- [x] SecMasterClient HTTP client
- [x] All standard MCP tools (search, get, resolve, list, lookup, health)
- [x] Semantic MCP tools (semantic_search, ask_secmaster, hybrid_resolve)
- [x] Deployed to production (port 3107)

### ✓ Integration (Complete)
- [x] gRPC proto definitions in Events package
- [x] All collectors integrated and registering series
- [x] ThresholdEngine using SecMaster for resolution
- [x] Ansible deployment configured
- [x] Smoke tests for health endpoints
- [x] Seeding script for collector data

### ◯ Future Enhancements
- [ ] External identifier enrichment (OpenFIGI API integration)
- [ ] Instrument versioning/history tracking
- [ ] Advanced alias management UI
- [ ] Performance tuning for large-scale vector searches
- [ ] Multi-tenancy support

---

## CONTEXT

### Files Created
```
SecMaster/
  .devcontainer/devcontainer.json, compose.yaml
  src/
    SecMaster.csproj
    Program.cs
    DependencyInjection.cs
    Containerfile
    appsettings.json
    Data/
      SecMasterDbContext.cs
      Entities/InstrumentEntity.cs, SourceMappingEntity.cs, AliasEntity.cs
      Configurations/InstrumentConfiguration.cs, SourceMappingConfiguration.cs, AliasConfiguration.cs
      Repositories/IInstrumentRepository.cs, InstrumentRepository.cs
    Endpoints/
      InstrumentEndpoints.cs, ResolutionEndpoints.cs, RegistrationEndpoints.cs, SearchEndpoints.cs
    Services/
      IResolutionService.cs, ResolutionService.cs
      IRegistrationService.cs, RegistrationService.cs
    Models/ResolutionModels.cs
    Telemetry/SecMasterActivitySource.cs, SecMasterMeter.cs
    HealthChecks/DatabaseHealthCheck.cs

SecMasterMcp/
  .devcontainer/devcontainer.json, compose.yaml
  src/
    SecMasterMcp.csproj
    Program.cs
    Containerfile
    Client/
      ISecMasterClient.cs, SecMasterClient.cs
      Models/SecMasterModels.cs
    Tools/SecMasterTools.cs
```

### API Endpoints
- `GET /api/instruments` - list all
- `GET /api/instruments/{id}` - get by ID
- `GET /api/instruments/by-symbol/{symbol}` - get by symbol
- `POST /api/instruments` - create
- `PUT /api/instruments/{id}` - update
- `DELETE /api/instruments/{id}` - delete
- `GET /api/instruments/{id}/sources` - list sources
- `POST /api/instruments/{id}/sources` - add source
- `GET /api/resolve/{symbol}` - resolve with optional frequency/maxLag/preferCollector
- `POST /api/resolve` - resolve with full context
- `GET /api/resolve/batch?symbols=A,B,C` - batch resolve
- `GET /api/resolve/lookup/{collector}/{sourceId}` - reverse lookup
- `POST /api/register` - collector registration
- `GET /api/search?q=...&assetClass=...&limit=...` - fuzzy search

### MCP Tools (Standard)
- search_instruments, get_instrument, resolve_source, resolve_batch
- list_sources, lookup_by_collector_id, health

### MCP Tools (Semantic)
- semantic_search (vector similarity)
- ask_secmaster (RAG synthesis)
- hybrid_resolve (SQL→Fuzzy→Vector→RAG)

### Dependencies

**Runtime:**
- timescaledb (PostgreSQL with pgvector + pg_trgm)
- ollama-gpu (nomic-embed-text, llama3.2:3b)
- otel-collector (traces/metrics)

**Consumers:**
- ThresholdEngine (resolution client via gRPC)
- All collectors (registration clients via gRPC)
- AI assistants (via SecMasterMcp on port 3107)

---

## RECENT CHANGES

- **2024-12-14**: Added pgvector semantic search with Ollama embeddings
- **2024-12-14**: Implemented hybrid resolution (SQL→Fuzzy→Vector→RAG)
- **2024-12-14**: Added RAG synthesis with llama3.2:3b
- **2024-12-14**: Created background embedding service
- **2024-12-14**: Extended SecMasterMcp with semantic tools (semantic_search, ask_secmaster, hybrid_resolve)
- **2024-12-14**: Updated documentation to reflect semantic search features

See [README.md](README.md) for full documentation.
