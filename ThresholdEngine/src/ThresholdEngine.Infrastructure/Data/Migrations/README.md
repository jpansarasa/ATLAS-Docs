# Database Migrations

This directory contains EF Core migrations for managing the database schema. Migrations are automatically applied on service startup.

## Migration Management

### Creating Migrations

```bash
# From within the devcontainer or with dotnet-ef installed
dotnet ef migrations add <MigrationName> \
  --project src/ThresholdEngine.Infrastructure \
  --startup-project src/ThresholdEngine.Service \
  --context ThresholdEngineDbContext
```

### Applying Migrations

Migrations are automatically applied when the service starts. To apply manually:

```bash
# Via service container
docker exec thresholdengine-4cf3d dotnet ef database update \
  --project src/ThresholdEngine.Infrastructure \
  --startup-project src/ThresholdEngine.Service

# Or from devcontainer
dotnet ef database update \
  --project src/ThresholdEngine.Infrastructure \
  --startup-project src/ThresholdEngine.Service
```

### Generating SQL Script

To generate a SQL script from migrations:

```bash
dotnet ef migrations script \
  --project src/ThresholdEngine.Infrastructure \
  --startup-project src/ThresholdEngine.Service \
  --output migrations.sql
```

## Current Migrations

### InitialCreate (20250116000000)

Creates the `threshold_events` table and converts it to a TimescaleDB hypertable.

**What it does:**
- Creates `threshold_events` table with all required columns
- Creates indexes for efficient time-series queries
- Converts table to hypertable using `evaluated_at` as time dimension
- Sets up 2-year retention policy (730 days)
- Uses 1-month chunk interval for optimal performance

**Verification:**

```sql
-- Check if hypertable was created
SELECT * FROM timescaledb_information.hypertables WHERE hypertable_name = 'threshold_events';

-- Check retention policy
SELECT * FROM timescaledb_information.jobs WHERE hypertable_name = 'threshold_events';

-- Test insert
INSERT INTO threshold_events (pattern_id, pattern_name, category, signal, confidence, evaluated_at)
VALUES ('test-pattern', 'Test Pattern', 'Recession', 1.5, 0.85, NOW());
```

## Retention Policy

The retention policy is set to 2 years (730 days) by default. To modify:

```sql
-- Remove existing policy
SELECT remove_retention_policy('threshold_events');

-- Add new policy (e.g., 1 year)
SELECT add_retention_policy('threshold_events', INTERVAL '365 days');
```

## Chunk Interval

The chunk interval is set to 1 month. This balances:
- Query performance (smaller chunks = faster queries)
- Chunk management overhead (larger chunks = fewer chunks to manage)

To modify, update the migration SQL or create a new migration.

## Notes

- The table uses `evaluated_at` as the time dimension for the hypertable
- A unique constraint on `(pattern_id, evaluated_at)` prevents duplicate events
- All indexes are optimized for time-series queries (DESC on time columns)
- Metadata is stored as JSONB for flexible querying
- Migrations include TimescaleDB-specific SQL for hypertable conversion and retention policies
