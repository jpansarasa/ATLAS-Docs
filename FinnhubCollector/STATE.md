# FinnhubCollector STATE

## GOAL
Implement FinnhubCollector service to ingest market data, economic calendar events, sentiment data, and analyst information from Finnhub API.

### Accept Criteria
- 60 calls/min rate-limited API client with token bucket
- 12 tables for all data types
- gRPC streaming to ThresholdEngine
- OpenTelemetry observability (1 ActivitySource, 1 Meter)
- Own TimescaleDB instance

### Constraints
- VIX data comes from FRED at VIXCLS (not VXX proxy)
- Each service owns its database (not shared)
- Start with 1 ActivitySource + 1 Meter (consolidate later if needed)
- Sentiment saved and sent as events for RSS collectors

## ARCH

### Decisions
| Decision | Rationale |
|----------|-----------|
| Own TimescaleDB | Service database ownership pattern |
| Single ActivitySource/Meter | Start simple, consolidate per user guidance |
| No VXX as VIX proxy | VIX available from FRED VIXCLS |
| gRPC ObservationEventStream | Consistent with FredCollector/NasdaqCollector |
| Port 5008 for gRPC | Different from FredCollector (5001) to avoid conflicts |

### Project Structure
```
FinnhubCollector/
├── src/
│   ├── FinnhubCollector.Core/          # Domain models, interfaces
│   ├── FinnhubCollector.Application/   # Services, workers
│   ├── FinnhubCollector.Infrastructure/# API client, repository
│   ├── FinnhubCollector.Grpc/          # gRPC services
│   └── FinnhubCollector.Service/       # Host with OpenTelemetry
├── tests/
│   └── FinnhubCollector.UnitTests/
├── migrations/
│   └── 001_initial_schema.sql
└── .devcontainer/
```

## ATTEMPTED
(none - clean implementation)

## STATUS
- ✓T4.1: Create Solution Structure
- ✓T4.2: Implement Core Domain Models (13 models)
- ✓T4.3: Implement Core Interfaces (IFinnhubApiClient, IFinnhubRepository, IRateLimiter)
- ✓T4.4: Implement Rate-Limited API Client (18 API methods)
- ✓T4.5: Implement TimescaleDB Repository (full EF Core implementation)
- ✓T4.6: Create Database Migration (12 tables, 22 series)
- ✓T4.7: Implement Collection Workers (QuoteCollectionWorker)
- ✓T4.8: Implement gRPC Server (EventStreamService, EventRepository)
- ✓T4.9: Configure Service Host with OpenTelemetry
- ✓T4.10: Create Containerfile
- ✓T4.11: Write Unit Tests (17 tests - TokenBucketRateLimiter, EventStreamService)
- ◯T4.12: Integration Test with ThresholdEngine
- ◯T4.13: Add to Ansible Deployment

## CONTEXT

### Files Created
- src/FinnhubCollector.Core/Models/* (13 domain models)
- src/FinnhubCollector.Core/Interfaces/* (3 interfaces)
- src/FinnhubCollector.Core/Telemetry/* (ActivitySource, Meter)
- src/FinnhubCollector.Infrastructure/* (API client, repository, rate limiter)
- src/FinnhubCollector.Infrastructure/Data/* (DbContext, configurations)
- src/FinnhubCollector.Application/* (DI, workers, channels)
- src/FinnhubCollector.Grpc/* (EventStreamService, EventRepository)
- src/FinnhubCollector.Service/Program.cs (host with OpenTelemetry)
- migrations/001_initial_schema.sql (12 tables)
- .devcontainer/compose.yaml (own TimescaleDB at port 5434)
- tests/FinnhubCollector.UnitTests/* (17 unit tests)

### Dependencies
- ATLAS.Events.Grpc (proto compiled via Grpc.Tools)
- EF Core 9.0 + Npgsql
- OpenTelemetry 1.10
- Serilog with OpenTelemetry sink
- Quartz.NET for scheduling
- Polly for HTTP resilience

### External Systems
- Finnhub API (https://finnhub.io)
- TimescaleDB (own instance at port 5434)
- OTel Collector
