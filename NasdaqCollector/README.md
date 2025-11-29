# NasdaqCollector

LBMA gold price data collector for ATLAS using Nasdaq Data Link API (formerly Quandl).

## Overview

Collects daily gold fixing prices from the London Bullion Market Association (LBMA) via Nasdaq Data Link. Stores data in TimescaleDB and streams observation events via gRPC.

**Data Source**: LBMA/GOLD dataset from Nasdaq Data Link
**Update Frequency**: Every 6 hours (gold fixes at 10:30 AM and 3:00 PM London time)
**API Rate Limit**: 50,000 calls/day (free tier with API key)

## Architecture

```
NasdaqCollector.Core         - Domain models (NasdaqSeries, NasdaqObservation)
NasdaqCollector.Application  - Business logic (NasdaqCollectionService)
NasdaqCollector.Infrastructure - Nasdaq API client, repository (Dapper)
NasdaqCollector.Grpc          - gRPC event streaming
NasdaqCollector.Service       - ASP.NET Core host with collection worker
```

## Configuration

### Environment Variables

```bash
NASDAQ_API_KEY=your_api_key_here
ConnectionStrings__Atlas=Host=timescaledb;Database=atlas;Username=atlas;Password=yourpassword
```

### appsettings.json

```json
{
  "Nasdaq": {
    "ApiKey": "${NASDAQ_API_KEY}",
    "BaseUrl": "https://data.nasdaq.com/api/v3",
    "MaxRetries": 3,
    "RetryDelay": "00:00:02"
  },
  "Kestrel": {
    "Endpoints": {
      "Http": { "Url": "http://0.0.0.0:5004" },
      "Grpc": { "Url": "http://0.0.0.0:5005", "Protocols": "Http2" }
    }
  }
}
```

## Database Setup

Run the migration script to create tables:

```bash
psql -h timescaledb -U atlas -d atlas -f migrations/001_initial_schema.sql
```

Tables created:
- `nasdaq_series` - Series configuration
- `nasdaq_observations` - Hypertable for time-series observations

## Running Locally

### Using Docker

```bash
docker build -f Containerfile -t nasdaqcollector:latest .
docker run -e NASDAQ_API_KEY=your_key nasdaqcollector:latest
```

### Using DevContainer

```bash
cd .devcontainer
docker compose up -d
```

Then attach to the `dev` container and run:

```bash
dotnet run --project src/NasdaqCollector.Service
```

## Endpoints

- **HTTP Health**: `GET http://localhost:5004/health`
- **gRPC**: `grpc://localhost:5005/atlas.events.ObservationEventStream`

## gRPC Service

Implements `ObservationEventStream` from Events project:

- `SubscribeToEvents` - Real-time event streaming
- `GetEventsSince` - Historical query from timestamp
- `GetEventsBetween` - Historical query with time range
- `GetLatestEventTime` - Get latest event timestamp
- `GetHealth` - Service health check

Example using grpcurl:

```bash
grpcurl -plaintext localhost:5005 atlas.events.ObservationEventStream/GetHealth
```

## Testing

```bash
dotnet test
```

## Series Configuration

Default series configured in migration:
- **LBMA/GOLD** - Gold Price: London Fixing (USD AM)

Additional series can be added to `nasdaq_series` table:

```sql
INSERT INTO nasdaq_series (series_id, database_code, dataset_code, title, category, frequency, value_column)
VALUES (LBMA/SILVER, LBMA, SILVER, Silver
