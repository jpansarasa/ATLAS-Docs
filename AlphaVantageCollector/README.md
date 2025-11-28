# AlphaVantageCollector

Collector service for Alpha Vantage commodities (copper, oil) and equities data with strict rate limiting (25 requests/day).

## Architecture

- **Core**: Domain models (AlphaVantageSeries, AlphaVantageObservation, SeriesType enum)
- **Application**: Collection scheduler with priority-based series selection
- **Infrastructure**: Rate-limited API client with TokenBucketRateLimiter and TimescaleDB repository
- **Grpc**: Event streaming server using shared Events proto
- **Service**: Worker host with 4-hour collection cycles

## Key Features

- **Rate Limiting**: Strict 25 requests/day enforcement with daily counter reset
- **Smart Scheduling**: Priority-based collection within rate limits
- **Token Bucket**: 1 request/minute sustained, burst of 5
- **Collection Strategy**: 6 cycles/day × 4 requests = 24 requests/day (1 request buffer)

## Series Configuration

| SeriesId | Priority | Interval | Purpose |
|----------|----------|----------|---------|
| AV/COPPER | 1 | monthly | Cu/Au ratio calculation |
| AV/WTI | 2 | daily | Oil price tracking |
| AV/BRENT | 3 | daily | Oil price tracking |
| AV/NATURAL_GAS | 5 | daily | Energy markets |
| AV/ALL_COMMODITIES | 10 | monthly | Commodities index |

## Configuration

```json
{
  "ConnectionStrings": {
    "Atlas": "Host=timescaledb;Database=atlas;Username=atlas;Password=${DB_PASSWORD}"
  },
  "AlphaVantage": {
    "ApiKey": "${ALPHAVANTAGE_API_KEY}",
    "DailyLimit": 25
  }
}
```

## Environment Variables

- `ALPHAVANTAGE_API_KEY`: API key from alphavantage.co
- `DB_PASSWORD`: TimescaleDB password

## Ports

- 5006: HTTP API / Health endpoint
- 5007: gRPC event streaming

## Database Migration

```bash
psql -h timescaledb -U atlas -d atlas -f migrations/001_initial_schema.sql
```

## Development

```bash
cd .devcontainer
docker compose up -d
docker compose exec dev dotnet build
docker compose exec dev dotnet test
```

## Health Check

```bash
curl http://localhost:5006/health
# Returns: {"status":"healthy","remainingRequests":23}
```

## gRPC Integration

Uses shared `observation_events.proto` from Events project. ThresholdEngine subscribes to:
- `SeriesCollectedEvent`: New observations collected
- Filters by `SeriesId` for relevant commodities

## Rate Limit Strategy

With 25 requests/day and 6 collection cycles (every 4 hours):

- ~4 requests per cycle
- Priority 1 (COPPER): Collected every 25+ days
- Priority 2-5 (daily series): Collected daily
- Fits within 25/day limit with buffer for retries

## Testing

```bash
dotnet test
```

Tests cover:
- Rate limiting (daily counter, reset)
- Collection scheduler (priority, availability)
- API client (parsing, error handling)
