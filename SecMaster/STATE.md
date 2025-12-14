# SecMaster Implementation State

## GOAL
Create a Security Master (SecMaster) microservice for ATLAS that provides centralized instrument metadata management, enabling resolution of symbolic pattern references to the best available data source based on context.

### Accept Criteria
- Standalone service with PostgreSQL database, REST API, MCP server
- Metadata only (no observation data storage)
- Well-known columns + JSONB for extensibility
- Collectors auto-register series when adding new ones
- Context-dependent source selection (frequency, lag requirements)
- Fuzzy search by name, symbol, or description

### Constraints
- Keep symbolic pattern references in ThresholdEngine
- Future-proof for OpenFIGI, CUSIP, SEDOL metadata
- Fire-and-forget registration (resilient to SecMaster unavailability)

---

## ARCH

### Database Schema
```
instruments (id, symbol, name, asset_class, frequency, metadata JSONB, ...)
source_mappings (instrument_id, collector, source_id, frequency, priority, lag_days, is_primary)
aliases (instrument_id, alias, alias_type)
```

### Resolution Algorithm
- Find instrument by symbol or alias
- Filter sources by frequency requirement (daily satisfies monthly, not vice versa)
- Filter by max lag if specified
- Return by priority (primary first, then priority number, then lag)

### Key Decisions
- Using EF Core with PostgreSQL/Npgsql
- Trigram indexes (pg_trgm) for fuzzy search
- JSONB metadata column for extensibility without schema changes
- REST API for simplicity (gRPC optional later)

---

## STATUS

### ✓ Completed
- [x] SecMaster project structure (.devcontainer, src/, Containerfile)
- [x] Database schema and EF Core entities (InstrumentEntity, SourceMappingEntity, AliasEntity)
- [x] Entity configurations with indexes
- [x] InstrumentRepository with CRUD, search, FindBySymbolOrAlias
- [x] ResolutionService with frequency hierarchy logic
- [x] RegistrationService for collector auto-registration
- [x] REST API endpoints (instruments, resolve, register, search)
- [x] OpenTelemetry tracing and metrics
- [x] Health checks
- [x] SecMasterMcp project with all tools
- [x] Both projects compile successfully

### ◯ Pending
- [ ] Add secmaster_registration.proto to Events package
- [ ] Integrate SecMaster registration in collectors (5 services)
- [ ] Integrate SecMaster client in ThresholdEngine
- [ ] Create seed data and migration scripts
- [ ] Deploy and test end-to-end

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

### MCP Tools
- search_instruments, get_instrument, resolve_source, resolve_batch
- list_sources, lookup_by_collector_id, health

### Dependencies
- Requires: timescaledb (PostgreSQL), otel-collector
- Depended on by: ThresholdEngine, all collectors (optional)

---

## NEXT STEPS

1. **Proto definition** - Add `secmaster_registration.proto` to Events package for gRPC registration
2. **Collector integration** - Add `ISecMasterRegistrationService` to each collector's `SeriesManagementService.AddSeriesAsync()`
3. **ThresholdEngine integration** - Replace prefix-based routing in `DataWarmupService.RouteSeriestoCollectors()` with SecMaster lookup
4. **Seed data** - Create SQL script for well-known instruments (SOFR, major economic indicators)
5. **Deploy** - Add to deployment/compose.yaml, create Grafana dashboard
