# OfrCollector

Data collector service for Office of Financial Research (OFR) public datasets. Collects FSI, STFM, and HFM financial data on scheduled intervals and provides REST and gRPC APIs for access.

## Data Sources

### FSI - Financial Stress Index
Composite measure of stress in global financial markets. Single daily value with regional and category breakdowns.

**Collection**: Daily at 10:00 UTC
**Frequency**: Daily
**API**: CSV-based

### STFM - Short-term Funding Monitor
Money market and repo rate data across multiple datasets:
- **FNYR**: NY Fed Reference Rates (SOFR, EFFR, OBFR)
- **Repo**: U.S. Repo Markets (DVP, GCF rates and volumes)
- **MMF**: U.S. Money Market Funds (N-MFP holdings)
- **NYPD**: Primary Dealer Statistics
- **TYLD**: Treasury Constant Maturity Rates

**Collection**: Varies by dataset (8h-24h intervals)
**Frequency**: Daily
**API**: JSON REST API

### HFM - Hedge Fund Monitor
Hedge fund leverage and risk metrics:
- **FPF**: Form PF Aggregates (leverage, liquidity, stress tests)
- **FICC**: FICC Sponsored Repo

**Collection**: Daily at 17:00 UTC
**Frequency**: Quarterly
**API**: JSON REST API

## API Endpoints

### Public API (`/api`)

**FSI**
- `GET /api/fsi/latest` - Latest FSI value
- `GET /api/fsi/history?startDate=&endDate=&limit=` - Historical FSI data

**STFM**
- `GET /api/stfm/series` - List active STFM series
- `GET /api/stfm/{mnemonic}/latest` - Latest observation for series
- `GET /api/stfm/{mnemonic}/observations?startDate=&endDate=&limit=` - Series observations

**HFM**
- `GET /api/hfm/series` - List active HFM series
- `GET /api/hfm/{mnemonic}/latest` - Latest observation for series
- `GET /api/hfm/{mnemonic}/observations?startDate=&endDate=&limit=` - Series observations

**Metadata**
- `GET /api/categories` - List available data categories
- `GET /api/health` - Service health

### Admin API (`/api/admin`)

**Collection Triggers**
- `POST /api/admin/fsi/collect` - Trigger FSI collection
- `POST /api/admin/fsi/backfill?months=` - Backfill FSI history
- `POST /api/admin/stfm/{dataset}/collect` - Trigger STFM dataset collection
- `POST /api/admin/stfm/priority/collect` - Collect priority STFM series
- `POST /api/admin/hfm/{dataset}/collect` - Trigger HFM dataset collection
- `POST /api/admin/hfm/all/collect` - Collect all HFM datasets

**STFM Series Management**
- `GET /api/admin/stfm/series` - List all STFM series
- `POST /api/admin/stfm/series` - Add new STFM series
- `PUT /api/admin/stfm/series/{mnemonic}/toggle` - Toggle series active status
- `DELETE /api/admin/stfm/series/{mnemonic}` - Delete series
- `POST /api/admin/stfm/series/{mnemonic}/collect` - Collect specific series
- `POST /api/admin/stfm/series/{mnemonic}/backfill` - Backfill series history

**HFM Series Management**
- `GET /api/admin/hfm/series` - List all HFM series
- `POST /api/admin/hfm/series` - Add new HFM series
- `PUT /api/admin/hfm/series/{mnemonic}/toggle` - Toggle series active status
- `DELETE /api/admin/hfm/series/{mnemonic}` - Delete series
- `POST /api/admin/hfm/series/{mnemonic}/collect` - Collect specific series
- `POST /api/admin/hfm/series/{mnemonic}/backfill` - Backfill series history

### gRPC API

**EventStreamService** (port 5001)
- Streams observation events to subscribers
- Used for real-time data distribution

### Health Checks

- `GET /health` - Full health check with database status
- `GET /health/ready` - Readiness probe (includes database check)
- `GET /health/live` - Liveness probe (always healthy)

## Configuration

### Environment Variables

```
DATABASE_CONNECTION_STRING - PostgreSQL connection string
OPENTELEMETRY__OTLPENDPOINT - OTLP collector endpoint (default: http://otel-collector:4317)
OPENTELEMETRY__SERVICENAME - Service name (default: ofr-collector-service)
OPENTELEMETRY__SERVICEVERSION - Service version (default: 1.0.0)
```

### Series Configuration

Series tracking is stored in PostgreSQL. Use admin endpoints to manage which STFM/HFM series are actively collected.

Priority series for STFM can be configured via `/config/stfm_series.json` (checked into repo).

## Observability

**Ports**
- 8080: HTTP/1.1 (REST API, health checks)
- 5001: HTTP/2 (gRPC)

**Telemetry**
- Logs: Structured JSON via Serilog (OTLP export)
- Traces: OpenTelemetry (ASP.NET Core, HttpClient)
- Metrics: OpenTelemetry (collection counts, durations, errors)

**Key Metrics**
- `ofr_observations_collected` - Observations collected by source
- `ofr_series_collected` - Series collected by source/dataset
- `ofr_collection_duration` - Collection duration by source
- `ofr_collection_errors` - Collection errors by source
- `ofr_fsi_value` - Current FSI value

## Development

Use devcontainer for development environment. Migrations applied automatically on startup.

```bash
# Build
cd /workspace/OfrCollector/src
dotnet build

# Run migrations
dotnet ef database update

# Run locally
dotnet run
```

## Deployment

Container-based deployment using Containerfile. Build context is ATLAS root.

```bash
# Build
nerdctl build -f OfrCollector/src/Containerfile -t ofr-collector:latest .

# Deploy
ansible-playbook -i deployment/ansible/inventory deployment/ansible/playbooks/deploy.yml
```
