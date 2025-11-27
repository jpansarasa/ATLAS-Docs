# E3: AlphaVantageCollector

## Goal
Ingest Alpha Vantage commodities (copper, WTI oil) and potentially equities into ATLAS, completing the Cu/Au ratio capability and enabling real-time market data patterns.

## Context

Alpha Vantage provides free access to commodities and equity data with a 25 requests/day limit on the free tier. This collector enables:
- Real-time copper prices (for Cu/Au ratio with Nasdaq gold)
- WTI crude oil prices
- Equity/ETF prices (SPY, QQQ for index tracking)

**Key Constraints:**
- Free tier: 25 API requests/day
- Must be strategic about what series to collect
- Different data shapes than FRED or Nasdaq

## Architecture

```
┌─────────────────────────────────────────────────────────┐
│                  AlphaVantageCollector                   │
│  ┌─────────────┐  ┌──────────────┐  ┌───────────────┐  │
│  │ Collection  │  │ AlphaVantage │  │    gRPC       │  │
│  │   Worker    │──│  API Client  │  │   Server      │  │
│  └──────┬──────┘  └──────────────┘  └───────┬───────┘  │
│         │                                    │          │
│         ▼                                    │          │
│  ┌─────────────────────────────────────────────────┐   │
│  │              TimescaleDB Repository              │   │
│  └─────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────┘
                           │
                           ▼ ObservationCollectedEvent
                    ThresholdEngine
```

## Prerequisites

- E1 complete (shared Events project)
- E2 complete or in progress (NasdaqCollector pattern to follow)
- Alpha Vantage API key (free registration at alphavantage.co)
- Understanding of rate limiting patterns

## Project Structure

```
AlphaVantageCollector/
├── src/
│   ├── AlphaVantageCollector.Core/
│   ├── AlphaVantageCollector.Application/
│   ├── AlphaVantageCollector.Infrastructure/
│   ├── AlphaVantageCollector.Grpc/
│   └── AlphaVantageCollector.Service/
├── tests/
│   └── AlphaVantageCollector.UnitTests/
├── config/
│   └── series.json
├── .devcontainer/
└── Containerfile
```

## Alpha Vantage API Reference

### Commodities Endpoint
```
GET https://www.alphavantage.co/query?function=COPPER&interval=monthly&apikey=YOUR_KEY
GET https://www.alphavantage.co/query?function=WTI&interval=daily&apikey=YOUR_KEY
```

Response format:
```json
{
  "name": "Global Price of Copper",
  "interval": "monthly",
  "unit": "USD per metric ton",
  "data": [
    { "date": "2025-11-01", "value": "9835.06" },
    { "date": "2025-10-01", "value": "9531.20" }
  ]
}
```

### Available Commodities
| Function | Description | Intervals |
|----------|-------------|-----------|
| `WTI` | West Texas Intermediate Crude Oil | daily, weekly, monthly |
| `BRENT` | Brent Crude Oil | daily, weekly, monthly |
| `NATURAL_GAS` | Natural Gas | daily, weekly, monthly |
| `COPPER` | Global Copper Price | monthly, quarterly, annual |
| `ALUMINUM` | Global Aluminum Price | monthly, quarterly, annual |
| `WHEAT` | Global Wheat Price | monthly, quarterly, annual |
| `CORN` | Global Corn Price | monthly, quarterly, annual |
| `COTTON` | Global Cotton Price | monthly, quarterly, annual |
| `SUGAR` | Global Sugar Price | monthly, quarterly, annual |
| `COFFEE` | Global Coffee Price | monthly, quarterly, annual |
| `ALL_COMMODITIES` | Global Commodities Index | monthly, quarterly, annual |

### Equity Endpoint (Future)
```
GET https://www.alphavantage.co/query?function=GLOBAL_QUOTE&symbol=SPY&apikey=YOUR_KEY
```

## Tasks

### T3.1: Create Solution Structure
**Estimate:** 30 min

```bash
cd ~/ATLAS
mkdir -p AlphaVantageCollector/src AlphaVantageCollector/tests AlphaVantageCollector/config

cd AlphaVantageCollector
dotnet new sln -n AlphaVantageCollector

# Create projects (same pattern as NasdaqCollector)
dotnet new classlib -n AlphaVantageCollector.Core -o src/AlphaVantageCollector.Core -f net9.0
dotnet new classlib -n AlphaVantageCollector.Application -o src/AlphaVantageCollector.Application -f net9.0
dotnet new classlib -n AlphaVantageCollector.Infrastructure -o src/AlphaVantageCollector.Infrastructure -f net9.0
dotnet new classlib -n AlphaVantageCollector.Grpc -o src/AlphaVantageCollector.Grpc -f net9.0
dotnet new worker -n AlphaVantageCollector.Service -o src/AlphaVantageCollector.Service -f net9.0
dotnet new xunit -n AlphaVantageCollector.UnitTests -o tests/AlphaVantageCollector.UnitTests -f net9.0

# Add to solution and set up references
dotnet sln add src/AlphaVantageCollector.Core
dotnet sln add src/AlphaVantageCollector.Application
dotnet sln add src/AlphaVantageCollector.Infrastructure
dotnet sln add src/AlphaVantageCollector.Grpc
dotnet sln add src/AlphaVantageCollector.Service
dotnet sln add tests/AlphaVantageCollector.UnitTests

# Project references
cd src/AlphaVantageCollector.Application
dotnet add reference ../AlphaVantageCollector.Core

cd ../AlphaVantageCollector.Infrastructure
dotnet add reference ../AlphaVantageCollector.Core
dotnet add reference ../AlphaVantageCollector.Application

cd ../AlphaVantageCollector.Grpc
dotnet add reference ../AlphaVantageCollector.Core
dotnet add reference ../../../../Events/src/Events

cd ../AlphaVantageCollector.Service
dotnet add reference ../AlphaVantageCollector.Application
dotnet add reference ../AlphaVantageCollector.Infrastructure
dotnet add reference ../AlphaVantageCollector.Grpc
```

**Acceptance:** Solution builds with empty projects

---

### T3.2: Implement Core Domain Models
**Estimate:** 30 min

Create `AlphaVantageCollector.Core/Models/AlphaVantageSeries.cs`:

```csharp
namespace AlphaVantageCollector.Core.Models;

public enum SeriesType
{
    Commodity,
    Equity,
    Forex,
    Crypto
}

public record AlphaVantageSeries
{
    public required string SeriesId { get; init; }        // "AV/COPPER", "AV/WTI"
    public required string Function { get; init; }         // "COPPER", "WTI", "GLOBAL_QUOTE"
    public required string Title { get; init; }
    public required string Category { get; init; }
    public required SeriesType Type { get; init; }
    public string? Symbol { get; init; }                   // For equities: "SPY"
    public string Interval { get; init; } = "monthly";     // daily, weekly, monthly
    public string? Unit { get; init; }                     // "USD per metric ton"
    public bool IsActive { get; init; } = true;
    public DateTime? LastCollectedAt { get; init; }
    public int Priority { get; init; } = 10;               // Lower = higher priority
}
```

Create `AlphaVantageCollector.Core/Models/AlphaVantageObservation.cs`:

```csharp
namespace AlphaVantageCollector.Core.Models;

public record AlphaVantageObservation
{
    public required string SeriesId { get; init; }
    public required DateOnly Date { get; init; }
    public required decimal? Value { get; init; }
    public required DateTime CollectedAt { get; init; }
}
```

Create `AlphaVantageCollector.Core/Interfaces/IAlphaVantageApiClient.cs`:

```csharp
namespace AlphaVantageCollector.Core.Interfaces;

public interface IAlphaVantageApiClient
{
    int RemainingDailyRequests { get; }
    
    Task<IReadOnlyList<AlphaVantageObservation>> GetCommodityAsync(
        string function,
        string interval = "monthly",
        CancellationToken cancellationToken = default);
    
    Task<AlphaVantageObservation?> GetEquityQuoteAsync(
        string symbol,
        CancellationToken cancellationToken = default);
}
```

Create `AlphaVantageCollector.Core/Interfaces/IAlphaVantageRepository.cs`:

```csharp
namespace AlphaVantageCollector.Core.Interfaces;

public interface IAlphaVantageRepository
{
    Task<IReadOnlyList<AlphaVantageSeries>> GetActiveSeriesAsync(CancellationToken ct = default);
    Task<AlphaVantageSeries?> GetSeriesAsync(string seriesId, CancellationToken ct = default);
    Task UpsertObservationsAsync(IEnumerable<AlphaVantageObservation> observations, CancellationToken ct = default);
    Task<AlphaVantageObservation?> GetLatestObservationAsync(string seriesId, CancellationToken ct = default);
    Task UpdateLastCollectedAsync(string seriesId, DateTime collectedAt, CancellationToken ct = default);
}
```

**Acceptance:** Core models and interfaces compile

---

### T3.3: Implement Rate-Limited API Client
**Estimate:** 1.5 hours

This is the critical differentiator - must respect 25/day limit strictly.

Create `AlphaVantageCollector.Infrastructure/AlphaVantageApiClient.cs`:

```csharp
using System.Net.Http.Json;
using System.Text.Json;
using System.Text.Json.Serialization;
using System.Threading.RateLimiting;
using Microsoft.Extensions.Logging;
using Microsoft.Extensions.Options;
using AlphaVantageCollector.Core.Interfaces;
using AlphaVantageCollector.Core.Models;

namespace AlphaVantageCollector.Infrastructure;

public class AlphaVantageApiClientOptions
{
    public required string ApiKey { get; set; }
    public string BaseUrl { get; set; } = "https://www.alphavantage.co/query";
    public int DailyLimit { get; set; } = 25;
    public int MaxRetries { get; set; } = 2;
}

public sealed class AlphaVantageApiClient : IAlphaVantageApiClient, IAsyncDisposable
{
    private readonly HttpClient _httpClient;
    private readonly AlphaVantageApiClientOptions _options;
    private readonly ILogger<AlphaVantageApiClient> _logger;
    private readonly RateLimiter _rateLimiter;
    private int _dailyRequestCount;
    private DateOnly _currentDay;
    private readonly SemaphoreSlim _countLock = new(1, 1);

    public AlphaVantageApiClient(
        HttpClient httpClient,
        IOptions<AlphaVantageApiClientOptions> options,
        ILogger<AlphaVantageApiClient> logger)
    {
        _httpClient = httpClient;
        _options = options.Value;
        _logger = logger;
        _currentDay = DateOnly.FromDateTime(DateTime.UtcNow);
        
        // Token bucket: conservative to avoid hitting limits
        // 1 request per minute sustained, burst of 5
        _rateLimiter = new TokenBucketRateLimiter(new TokenBucketRateLimiterOptions
        {
            TokenLimit = 5,
            ReplenishmentPeriod = TimeSpan.FromMinutes(1),
            TokensPerPeriod = 1,
            QueueProcessingOrder = QueueProcessingOrder.OldestFirst,
            QueueLimit = 10
        });
    }

    public int RemainingDailyRequests => Math.Max(0, _options.DailyLimit - _dailyRequestCount);

    public async Task<IReadOnlyList<AlphaVantageObservation>> GetCommodityAsync(
        string function,
        string interval = "monthly",
        CancellationToken cancellationToken = default)
    {
        if (!await TryAcquireRequestAsync(cancellationToken))
        {
            _logger.LogWarning("Daily limit reached for Alpha Vantage API");
            return [];
        }

        using var lease = await _rateLimiter.AcquireAsync(1, cancellationToken);
        if (!lease.IsAcquired)
        {
            _logger.LogWarning("Rate limit not acquired, skipping request");
            return [];
        }

        var url = $"{_options.BaseUrl}?function={function}&interval={interval}&apikey={_options.ApiKey}";
        
        _logger.LogDebug("Fetching {Function} from Alpha Vantage", function);
        
        try
        {
            var response = await _httpClient.GetAsync(url, cancellationToken);
            response.EnsureSuccessStatusCode();
            
            var content = await response.Content.ReadAsStringAsync(cancellationToken);
            
            // Check for API error messages
            if (content.Contains("Error Message") || content.Contains("Note"))
            {
                _logger.LogWarning("Alpha Vantage API returned error: {Content}", content);
                return [];
            }
            
            var data = JsonSerializer.Deserialize<CommodityResponse>(content);
            
            if (data?.Data == null || data.Data.Count == 0)
                return [];
            
            var seriesId = $"AV/{function}";
            var collectedAt = DateTime.UtcNow;
            
            return data.Data
                .Select(d => new AlphaVantageObservation
                {
                    SeriesId = seriesId,
                    Date = DateOnly.Parse(d.Date),
                    Value = decimal.TryParse(d.Value, out var v) ? v : null,
                    CollectedAt = collectedAt
                })
                .Where(o => o.Value.HasValue)
                .ToList();
        }
        catch (HttpRequestException ex)
        {
            _logger.LogError(ex, "Failed to fetch {Function}", function);
            throw;
        }
    }

    public async Task<AlphaVantageObservation?> GetEquityQuoteAsync(
        string symbol,
        CancellationToken cancellationToken = default)
    {
        if (!await TryAcquireRequestAsync(cancellationToken))
        {
            _logger.LogWarning("Daily limit reached for Alpha Vantage API");
            return null;
        }

        using var lease = await _rateLimiter.AcquireAsync(1, cancellationToken);
        if (!lease.IsAcquired)
            return null;

        var url = $"{_options.BaseUrl}?function=GLOBAL_QUOTE&symbol={symbol}&apikey={_options.ApiKey}";
        
        try
        {
            var response = await _httpClient.GetAsync(url, cancellationToken);
            response.EnsureSuccessStatusCode();
            
            var data = await response.Content.ReadFromJsonAsync<GlobalQuoteResponse>(
                cancellationToken: cancellationToken);
            
            if (data?.GlobalQuote == null || string.IsNullOrEmpty(data.GlobalQuote.Price))
                return null;
            
            var quote = data.GlobalQuote;
            
            return new AlphaVantageObservation
            {
                SeriesId = $"AV/{symbol}",
                Date = DateOnly.Parse(quote.LatestTradingDay ?? DateTime.UtcNow.ToString("yyyy-MM-dd")),
                Value = decimal.TryParse(quote.Price, out var p) ? p : null,
                CollectedAt = DateTime.UtcNow
            };
        }
        catch (HttpRequestException ex)
        {
            _logger.LogError(ex, "Failed to fetch quote for {Symbol}", symbol);
            throw;
        }
    }

    private async Task<bool> TryAcquireRequestAsync(CancellationToken ct)
    {
        await _countLock.WaitAsync(ct);
        try
        {
            var today = DateOnly.FromDateTime(DateTime.UtcNow);
            if (today != _currentDay)
            {
                _currentDay = today;
                _dailyRequestCount = 0;
                _logger.LogInformation("Daily request counter reset, {Limit} requests available",
                    _options.DailyLimit);
            }
            
            if (_dailyRequestCount >= _options.DailyLimit)
            {
                _logger.LogDebug("Daily limit exhausted ({Count}/{Limit})",
                    _dailyRequestCount, _options.DailyLimit);
                return false;
            }
            
            _dailyRequestCount++;
            _logger.LogDebug("Request {Count}/{Limit} for today", 
                _dailyRequestCount, _options.DailyLimit);
            
            return true;
        }
        finally
        {
            _countLock.Release();
        }
    }

    public async ValueTask DisposeAsync()
    {
        await _rateLimiter.DisposeAsync();
        _countLock.Dispose();
    }
}

// Response DTOs
internal record CommodityResponse
{
    [JsonPropertyName("name")]
    public string? Name { get; init; }
    
    [JsonPropertyName("interval")]
    public string? Interval { get; init; }
    
    [JsonPropertyName("unit")]
    public string? Unit { get; init; }
    
    [JsonPropertyName("data")]
    public List<CommodityDataPoint>? Data { get; init; }
}

internal record CommodityDataPoint
{
    [JsonPropertyName("date")]
    public required string Date { get; init; }
    
    [JsonPropertyName("value")]
    public required string Value { get; init; }
}

internal record GlobalQuoteResponse
{
    [JsonPropertyName("Global Quote")]
    public GlobalQuote? GlobalQuote { get; init; }
}

internal record GlobalQuote
{
    [JsonPropertyName("01. symbol")]
    public string? Symbol { get; init; }
    
    [JsonPropertyName("05. price")]
    public string? Price { get; init; }
    
    [JsonPropertyName("07. latest trading day")]
    public string? LatestTradingDay { get; init; }
}
```

Add packages to `AlphaVantageCollector.Infrastructure.csproj`:
```xml
<ItemGroup>
  <PackageReference Include="Microsoft.Extensions.Http" Version="9.0.0" />
  <PackageReference Include="Microsoft.Extensions.Options" Version="9.0.0" />
  <PackageReference Include="Microsoft.Extensions.Logging.Abstractions" Version="9.0.0" />
</ItemGroup>
```

**Acceptance:** Client compiles, tracks daily usage, respects rate limits

---

### T3.4: Implement Smart Collection Scheduler
**Estimate:** 45 min

Given the 25/day limit, need intelligent scheduling:

Create `AlphaVantageCollector.Application/Services/CollectionScheduler.cs`:

```csharp
using Microsoft.Extensions.Logging;
using AlphaVantageCollector.Core.Interfaces;
using AlphaVantageCollector.Core.Models;

namespace AlphaVantageCollector.Application.Services;

public interface ICollectionScheduler
{
    Task<IReadOnlyList<AlphaVantageSeries>> GetSeriesDueForCollectionAsync(
        int availableRequests,
        CancellationToken ct = default);
}

public sealed class CollectionScheduler : ICollectionScheduler
{
    private readonly IAlphaVantageRepository _repository;
    private readonly ILogger<CollectionScheduler> _logger;

    public CollectionScheduler(
        IAlphaVantageRepository repository,
        ILogger<CollectionScheduler> logger)
    {
        _repository = repository;
        _logger = logger;
    }

    public async Task<IReadOnlyList<AlphaVantageSeries>> GetSeriesDueForCollectionAsync(
        int availableRequests,
        CancellationToken ct = default)
    {
        if (availableRequests <= 0)
            return [];

        var allSeries = await _repository.GetActiveSeriesAsync(ct);
        var now = DateTime.UtcNow;
        
        var dueForCollection = allSeries
            .Where(s => IsCollectionDue(s, now))
            .OrderBy(s => s.Priority)           // Lower priority number = collect first
            .ThenBy(s => s.LastCollectedAt)     // Oldest first
            .Take(availableRequests)
            .ToList();
        
        _logger.LogDebug(
            "Scheduled {Count} series for collection from {Available} available requests",
            dueForCollection.Count, availableRequests);
        
        return dueForCollection;
    }

    private static bool IsCollectionDue(AlphaVantageSeries series, DateTime now)
    {
        if (series.LastCollectedAt == null)
            return true;
        
        var elapsed = now - series.LastCollectedAt.Value;
        
        // Collection frequency based on data update frequency
        return series.Interval switch
        {
            "daily" => elapsed > TimeSpan.FromHours(20),
            "weekly" => elapsed > TimeSpan.FromDays(6),
            "monthly" => elapsed > TimeSpan.FromDays(25),
            "quarterly" => elapsed > TimeSpan.FromDays(80),
            "annual" => elapsed > TimeSpan.FromDays(350),
            _ => elapsed > TimeSpan.FromDays(1)
        };
    }
}
```

**Acceptance:** Scheduler prioritizes series within rate limits

---

### T3.5: Implement TimescaleDB Repository
**Estimate:** 30 min

Create `AlphaVantageCollector.Infrastructure/AlphaVantageRepository.cs`:

```csharp
using Dapper;
using Microsoft.Extensions.Logging;
using AlphaVantageCollector.Core.Interfaces;
using AlphaVantageCollector.Core.Models;
using Npgsql;

namespace AlphaVantageCollector.Infrastructure;

public sealed class AlphaVantageRepository : IAlphaVantageRepository
{
    private readonly string _connectionString;
    private readonly ILogger<AlphaVantageRepository> _logger;

    public AlphaVantageRepository(string connectionString, ILogger<AlphaVantageRepository> logger)
    {
        _connectionString = connectionString;
        _logger = logger;
    }

    public async Task<IReadOnlyList<AlphaVantageSeries>> GetActiveSeriesAsync(CancellationToken ct = default)
    {
        await using var conn = new NpgsqlConnection(_connectionString);
        
        var result = await conn.QueryAsync<AlphaVantageSeries>(
            """
            SELECT series_id as SeriesId, function as Function, title as Title,
                   category as Category, series_type as Type, symbol as Symbol,
                   interval as Interval, unit as Unit, is_active as IsActive,
                   last_collected_at as LastCollectedAt, priority as Priority
            FROM alphavantage_series
            WHERE is_active = true
            ORDER BY priority, last_collected_at NULLS FIRST
            """);
        
        return result.ToList();
    }

    public async Task<AlphaVantageSeries?> GetSeriesAsync(string seriesId, CancellationToken ct = default)
    {
        await using var conn = new NpgsqlConnection(_connectionString);
        
        return await conn.QuerySingleOrDefaultAsync<AlphaVantageSeries>(
            """
            SELECT series_id as SeriesId, function as Function, title as Title,
                   category as Category, series_type as Type, symbol as Symbol,
                   interval as Interval, unit as Unit, is_active as IsActive,
                   last_collected_at as LastCollectedAt, priority as Priority
            FROM alphavantage_series
            WHERE series_id = @SeriesId
            """,
            new { SeriesId = seriesId });
    }

    public async Task UpsertObservationsAsync(
        IEnumerable<AlphaVantageObservation> observations, 
        CancellationToken ct = default)
    {
        await using var conn = new NpgsqlConnection(_connectionString);
        await conn.OpenAsync(ct);
        
        await using var transaction = await conn.BeginTransactionAsync(ct);
        
        foreach (var obs in observations)
        {
            await conn.ExecuteAsync(
                """
                INSERT INTO alphavantage_observations (series_id, date, value, collected_at)
                VALUES (@SeriesId, @Date, @Value, @CollectedAt)
                ON CONFLICT (series_id, date) 
                DO UPDATE SET value = EXCLUDED.value, collected_at = EXCLUDED.collected_at
                """,
                new 
                { 
                    obs.SeriesId, 
                    Date = obs.Date.ToDateTime(TimeOnly.MinValue), 
                    obs.Value, 
                    obs.CollectedAt 
                },
                transaction);
        }
        
        await transaction.CommitAsync(ct);
        
        _logger.LogDebug("Upserted {Count} observations", observations.Count());
    }

    public async Task<AlphaVantageObservation?> GetLatestObservationAsync(
        string seriesId, 
        CancellationToken ct = default)
    {
        await using var conn = new NpgsqlConnection(_connectionString);
        
        var row = await conn.QuerySingleOrDefaultAsync<dynamic>(
            """
            SELECT series_id, date, value, collected_at
            FROM alphavantage_observations
            WHERE series_id = @SeriesId
            ORDER BY date DESC
            LIMIT 1
            """,
            new { SeriesId = seriesId });
        
        if (row == null) return null;
        
        return new AlphaVantageObservation
        {
            SeriesId = row.series_id,
            Date = DateOnly.FromDateTime(row.date),
            Value = row.value,
            CollectedAt = row.collected_at
        };
    }

    public async Task UpdateLastCollectedAsync(
        string seriesId, 
        DateTime collectedAt, 
        CancellationToken ct = default)
    {
        await using var conn = new NpgsqlConnection(_connectionString);
        
        await conn.ExecuteAsync(
            """
            UPDATE alphavantage_series 
            SET last_collected_at = @CollectedAt
            WHERE series_id = @SeriesId
            """,
            new { SeriesId = seriesId, CollectedAt = collectedAt });
    }
}
```

Add packages:
```xml
<ItemGroup>
  <PackageReference Include="Dapper" Version="2.1.35" />
  <PackageReference Include="Npgsql" Version="8.0.5" />
</ItemGroup>
```

**Acceptance:** Repository compiles, implements all interface methods

---

### T3.6: Create Database Migration
**Estimate:** 15 min

Create `AlphaVantageCollector/migrations/001_initial_schema.sql`:

```sql
-- AlphaVantage series configuration
CREATE TABLE IF NOT EXISTS alphavantage_series (
    series_id TEXT PRIMARY KEY,
    function TEXT NOT NULL,
    title TEXT NOT NULL,
    category TEXT NOT NULL,
    series_type TEXT NOT NULL,
    symbol TEXT,
    interval TEXT NOT NULL DEFAULT 'monthly',
    unit TEXT,
    is_active BOOLEAN NOT NULL DEFAULT true,
    last_collected_at TIMESTAMPTZ,
    priority INTEGER NOT NULL DEFAULT 10,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- AlphaVantage observations (hypertable)
CREATE TABLE IF NOT EXISTS alphavantage_observations (
    series_id TEXT NOT NULL,
    date TIMESTAMPTZ NOT NULL,
    value NUMERIC,
    collected_at TIMESTAMPTZ NOT NULL,
    PRIMARY KEY (series_id, date)
);

SELECT create_hypertable('alphavantage_observations', 'date', if_not_exists => TRUE);

CREATE INDEX IF NOT EXISTS idx_alphavantage_observations_series 
    ON alphavantage_observations (series_id, date DESC);

-- Initial series: prioritized by importance for ATLAS Cu/Au ratio
INSERT INTO alphavantage_series (series_id, function, title, category, series_type, interval, priority)
VALUES 
    ('AV/COPPER', 'COPPER', 'Global Price of Copper', 'Commodity', 'Commodity', 'monthly', 1),
    ('AV/WTI', 'WTI', 'West Texas Intermediate Crude Oil', 'Commodity', 'Commodity', 'daily', 2),
    ('AV/BRENT', 'BRENT', 'Brent Crude Oil', 'Commodity', 'Commodity', 'daily', 3),
    ('AV/NATURAL_GAS', 'NATURAL_GAS', 'Natural Gas Price', 'Commodity', 'Commodity', 'daily', 5),
    ('AV/ALL_COMMODITIES', 'ALL_COMMODITIES', 'Global Commodities Index', 'Commodity', 'Commodity', 'monthly', 10)
ON CONFLICT (series_id) DO NOTHING;
```

**Acceptance:** Migration runs successfully against TimescaleDB

---

### T3.7: Implement Collection Worker
**Estimate:** 45 min

Create `AlphaVantageCollector.Service/Workers/CollectionWorker.cs`:

```csharp
using AlphaVantageCollector.Application.Services;
using AlphaVantageCollector.Core.Interfaces;
using AlphaVantageCollector.Core.Models;

namespace AlphaVantageCollector.Service.Workers;

public sealed class CollectionWorker : BackgroundService
{
    private readonly IAlphaVantageApiClient _apiClient;
    private readonly IAlphaVantageRepository _repository;
    private readonly ICollectionScheduler _scheduler;
    private readonly ILogger<CollectionWorker> _logger;
    
    // Run every 4 hours: 25 requests / 6 cycles = ~4 requests per cycle
    private readonly TimeSpan _interval = TimeSpan.FromHours(4);
    private const int RequestsPerCycle = 4;

    public CollectionWorker(
        IAlphaVantageApiClient apiClient,
        IAlphaVantageRepository repository,
        ICollectionScheduler scheduler,
        ILogger<CollectionWorker> logger)
    {
        _apiClient = apiClient;
        _repository = repository;
        _scheduler = scheduler;
        _logger = logger;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        _logger.LogInformation("AlphaVantageCollector worker starting");
        
        // Initial collection on startup
        await CollectAsync(stoppingToken);
        
        using var timer = new PeriodicTimer(_interval);
        
        while (!stoppingToken.IsCancellationRequested)
        {
            try
            {
                await timer.WaitForNextTickAsync(stoppingToken);
                await CollectAsync(stoppingToken);
            }
            catch (OperationCanceledException) when (stoppingToken.IsCancellationRequested)
            {
                break;
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Collection cycle failed");
            }
        }
    }

    private async Task CollectAsync(CancellationToken ct)
    {
        var available = _apiClient.RemainingDailyRequests;
        
        _logger.LogInformation(
            "Starting collection cycle, {Available} requests remaining today",
            available);
        
        if (available == 0)
        {
            _logger.LogWarning("No API requests remaining for today, skipping cycle");
            return;
        }
        
        // Use minimum of available and our per-cycle budget
        var toUse = Math.Min(available, RequestsPerCycle);
        
        var series = await _scheduler.GetSeriesDueForCollectionAsync(toUse, ct);
        
        if (series.Count == 0)
        {
            _logger.LogInformation("No series due for collection");
            return;
        }
        
        var collected = 0;
        var failed = 0;
        var totalObservations = 0;
        
        foreach (var s in series)
        {
            try
            {
                IReadOnlyList<AlphaVantageObservation> observations;
                
                if (s.Type == SeriesType.Commodity)
                {
                    observations = await _apiClient.GetCommodityAsync(
                        s.Function, s.Interval, ct);
                }
                else if (s.Type == SeriesType.Equity && !string.IsNullOrEmpty(s.Symbol))
                {
                    var quote = await _apiClient.GetEquityQuoteAsync(s.Symbol, ct);
                    observations = quote != null ? new[] { quote } : Array.Empty<AlphaVantageObservation>();
                }
                else
                {
                    _logger.LogWarning("Unknown series type for {SeriesId}: {Type}", 
                        s.SeriesId, s.Type);
                    continue;
                }
                
                if (observations.Count > 0)
                {
                    await _repository.UpsertObservationsAsync(observations, ct);
                    await _repository.UpdateLastCollectedAsync(s.SeriesId, DateTime.UtcNow, ct);
                    collected++;
                    totalObservations += observations.Count;
                    
                    _logger.LogInformation(
                        "Collected {Count} observations for {SeriesId}",
                        observations.Count, s.SeriesId);
                }
                else
                {
                    _logger.LogWarning("No data returned for {SeriesId}", s.SeriesId);
                }
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Failed to collect {SeriesId}", s.SeriesId);
                failed++;
            }
            
            // Delay between requests to be nice to the API
            await Task.Delay(500, ct);
        }
        
        _logger.LogInformation(
            "Collection complete: {Collected} series, {Observations} observations, {Failed} failed, {Remaining} requests remaining",
            collected, totalObservations, failed, _apiClient.RemainingDailyRequests);
    }
}
```

**Acceptance:** Worker collects data within rate limits

---

### T3.8: Implement gRPC Server
**Estimate:** 30 min

Create `AlphaVantageCollector.Grpc/Services/ObservationEventStreamService.cs`:

```csharp
using ATLAS.Events.Grpc;
using Google.Protobuf.WellKnownTypes;
using Grpc.Core;
using Microsoft.Extensions.Logging;
using AlphaVantageCollector.Core.Interfaces;

namespace AlphaVantageCollector.Grpc.Services;

public sealed class ObservationEventStreamService : ObservationEventStream.ObservationEventStreamBase
{
    private readonly IAlphaVantageRepository _repository;
    private readonly ILogger<ObservationEventStreamService> _logger;

    public ObservationEventStreamService(
        IAlphaVantageRepository repository,
        ILogger<ObservationEventStreamService> logger)
    {
        _repository = repository;
        _logger = logger;
    }

    public override async Task SubscribeToEvents(
        SubscriptionRequest request,
        IServerStreamWriter<Event> responseStream,
        ServerCallContext context)
    {
        var startFrom = request.StartFrom.ToDateTime();
        _logger.LogInformation("Client subscribed from {StartFrom}", startFrom);

        var lastEventTime = startFrom;
        
        while (!context.CancellationToken.IsCancellationRequested)
        {
            var series = await _repository.GetActiveSeriesAsync(context.CancellationToken);
            
            foreach (var s in series)
            {
                if (request.SeriesIds.Count > 0 && !request.SeriesIds.Contains(s.SeriesId))
                    continue;
                
                var latest = await _repository.GetLatestObservationAsync(
                    s.SeriesId, context.CancellationToken);
                
                if (latest != null && latest.CollectedAt > lastEventTime)
                {
                    var evt = CreateEvent(latest);
                    await responseStream.WriteAsync(evt);
                    lastEventTime = latest.CollectedAt;
                }
            }
            
            await Task.Delay(5000, context.CancellationToken); // Poll every 5s
        }
    }

    public override async Task GetEventsSince(
        TimeRangeRequest request,
        IServerStreamWriter<Event> responseStream,
        ServerCallContext context)
    {
        var series = await _repository.GetActiveSeriesAsync(context.CancellationToken);
        
        foreach (var s in series)
        {
            var latest = await _repository.GetLatestObservationAsync(
                s.SeriesId, context.CancellationToken);
            
            if (latest != null)
            {
                var evt = CreateEvent(latest);
                await responseStream.WriteAsync(evt);
            }
        }
    }

    public override Task<HealthResponse> Health(HealthRequest request, ServerCallContext context)
    {
        return Task.FromResult(new HealthResponse
        {
            Healthy = true,
            Message = "AlphaVantageCollector healthy"
        });
    }

    private static Event CreateEvent(AlphaVantageObservation obs)
    {
        return new Event
        {
            EventId = Ulid.NewUlid().ToString(),
            OccurredAt = Timestamp.FromDateTime(
                DateTime.SpecifyKind(obs.CollectedAt, DateTimeKind.Utc)),
            SourceService = "AlphaVantageCollector",
            SeriesCollected = new SeriesCollectedEvent
            {
                SeriesId = obs.SeriesId,
                Date = Timestamp.FromDateTime(
                    DateTime.SpecifyKind(
                        obs.Date.ToDateTime(TimeOnly.MinValue), 
                        DateTimeKind.Utc)),
                Value = (double)(obs.Value ?? 0),
                CollectedAt = Timestamp.FromDateTime(
                    DateTime.SpecifyKind(obs.CollectedAt, DateTimeKind.Utc))
            }
        };
    }
}
```

Add to `AlphaVantageCollector.Grpc.csproj`:
```xml
<ItemGroup>
  <PackageReference Include="Grpc.AspNetCore" Version="2.67.0" />
  <PackageReference Include="Ulid" Version="1.3.4" />
</ItemGroup>

<ItemGroup>
  <Protobuf Include="..\..\..\..\Events\src\Events\Protos\observation_events.proto" 
            GrpcServices="Server" />
</ItemGroup>

<ItemGroup>
  <ProjectReference Include="..\..\..\..\Events\src\Events\Events.csproj" />
  <ProjectReference Include="..\AlphaVantageCollector.Core\AlphaVantageCollector.Core.csproj" />
</ItemGroup>
```

**Acceptance:** gRPC server responds to subscription requests

---

### T3.9: Configure Service Host
**Estimate:** 20 min

Create `AlphaVantageCollector.Service/Program.cs`:

```csharp
using AlphaVantageCollector.Application.Services;
using AlphaVantageCollector.Core.Interfaces;
using AlphaVantageCollector.Grpc.Services;
using AlphaVantageCollector.Infrastructure;
using AlphaVantageCollector.Service.Workers;

var builder = WebApplication.CreateBuilder(args);

// Configuration
builder.Services.Configure<AlphaVantageApiClientOptions>(
    builder.Configuration.GetSection("AlphaVantage"));

// HTTP Client for Alpha Vantage API
builder.Services.AddHttpClient<IAlphaVantageApiClient, AlphaVantageApiClient>();

// Repository
var connectionString = builder.Configuration.GetConnectionString("Atlas") 
    ?? throw new InvalidOperationException("Connection string 'Atlas' not configured");
builder.Services.AddSingleton<IAlphaVantageRepository>(sp => 
    new AlphaVantageRepository(connectionString, sp.GetRequiredService<ILogger<AlphaVantageRepository>>()));

// Services
builder.Services.AddSingleton<ICollectionScheduler, CollectionScheduler>();

// Workers
builder.Services.AddHostedService<CollectionWorker>();

// gRPC
builder.Services.AddGrpc();

var app = builder.Build();

// gRPC endpoint
app.MapGrpcService<ObservationEventStreamService>();

// Health check with API status
app.MapGet("/health", (IAlphaVantageApiClient client) => Results.Ok(new 
{ 
    status = "healthy",
    remainingRequests = client.RemainingDailyRequests
}));

app.Run();
```

Create `AlphaVantageCollector.Service/appsettings.json`:
```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning"
    }
  },
  "ConnectionStrings": {
    "Atlas": "Host=timescaledb;Database=atlas;Username=atlas;Password=${DB_PASSWORD}"
  },
  "AlphaVantage": {
    "ApiKey": "${ALPHAVANTAGE_API_KEY}",
    "DailyLimit": 25
  },
  "Kestrel": {
    "Endpoints": {
      "Http": {
        "Url": "http://0.0.0.0:5006"
      },
      "Grpc": {
        "Url": "http://0.0.0.0:5007",
        "Protocols": "Http2"
      }
    }
  }
}
```

**Acceptance:** Service starts, workers run, gRPC endpoint responds

---

### T3.10: Create Containerfile and DevContainer
**Estimate:** 20 min

Create `AlphaVantageCollector/Containerfile`:

```dockerfile
FROM mcr.microsoft.com/dotnet/sdk:9.0 AS build
WORKDIR /src

COPY src/AlphaVantageCollector.Core/*.csproj src/AlphaVantageCollector.Core/
COPY src/AlphaVantageCollector.Application/*.csproj src/AlphaVantageCollector.Application/
COPY src/AlphaVantageCollector.Infrastructure/*.csproj src/AlphaVantageCollector.Infrastructure/
COPY src/AlphaVantageCollector.Grpc/*.csproj src/AlphaVantageCollector.Grpc/
COPY src/AlphaVantageCollector.Service/*.csproj src/AlphaVantageCollector.Service/
COPY ../Events/src/Events/*.csproj ../Events/src/Events/

RUN dotnet restore src/AlphaVantageCollector.Service/AlphaVantageCollector.Service.csproj

COPY . .
COPY ../Events/src/Events ../Events/src/Events

RUN dotnet publish src/AlphaVantageCollector.Service/AlphaVantageCollector.Service.csproj \
    -c Release -o /app --no-restore

FROM mcr.microsoft.com/dotnet/aspnet:9.0
WORKDIR /app
COPY --from=build /app .

ENV ASPNETCORE_URLS=http://+:5006;http://+:5007
EXPOSE 5006 5007

ENTRYPOINT ["dotnet", "AlphaVantageCollector.Service.dll"]
```

Create `.devcontainer/` following NasdaqCollector pattern.

**Acceptance:** Container builds, devcontainer starts

---

### T3.11: Write Unit Tests
**Estimate:** 1 hour

Create `tests/AlphaVantageCollector.UnitTests/AlphaVantageApiClientTests.cs`:

```csharp
using System.Net;
using Microsoft.Extensions.Logging.Abstractions;
using Microsoft.Extensions.Options;
using Moq;
using Moq.Protected;
using AlphaVantageCollector.Infrastructure;

namespace AlphaVantageCollector.UnitTests;

public class AlphaVantageApiClientTests
{
    [Fact]
    public async Task DailyLimit_StopsRequestsAfterLimitReached()
    {
        // Arrange
        var handler = CreateMockHandler(CreateCopperResponse());
        var client = CreateClient(handler, dailyLimit: 2);
        
        // Act
        var result1 = await client.GetCommodityAsync("COPPER");
        var result2 = await client.GetCommodityAsync("WTI");
        var result3 = await client.GetCommodityAsync("BRENT"); // Should be blocked
        
        // Assert
        Assert.NotEmpty(result1);
        Assert.NotEmpty(result2);
        Assert.Empty(result3); // Blocked by limit
        Assert.Equal(0, client.RemainingDailyRequests);
    }

    [Fact]
    public async Task DailyLimit_ResetsAtMidnight()
    {
        // This would require time manipulation - consider using TimeProvider
    }

    [Fact]
    public async Task GetCommodityAsync_ParsesCopperResponse()
    {
        // Arrange
        var handler = CreateMockHandler(CreateCopperResponse());
        var client = CreateClient(handler);
        
        // Act
        var observations = await client.GetCommodityAsync("COPPER", "monthly");
        
        // Assert
        Assert.Equal(2, observations.Count);
        Assert.Equal("AV/COPPER", observations[0].SeriesId);
        Assert.Equal(9835.06m, observations[0].Value);
    }

    [Fact]
    public async Task GetCommodityAsync_HandlesApiError()
    {
        // Arrange
        var errorResponse = """{"Error Message": "Invalid API call"}""";
        var handler = CreateMockHandler(errorResponse);
        var client = CreateClient(handler);
        
        // Act
        var result = await client.GetCommodityAsync("INVALID");
        
        // Assert
        Assert.Empty(result);
    }

    private static string CreateCopperResponse() => """
        {
          "name": "Global Price of Copper",
          "interval": "monthly",
          "unit": "USD per metric ton",
          "data": [
            { "date": "2025-11-01", "value": "9835.06" },
            { "date": "2025-10-01", "value": "9531.20" }
          ]
        }
        """;

    private static HttpMessageHandler CreateMockHandler(string content)
    {
        var mock = new Mock<HttpMessageHandler>();
        mock.Protected()
            .Setup<Task<HttpResponseMessage>>(
                "SendAsync",
                ItExpr.IsAny<HttpRequestMessage>(),
                ItExpr.IsAny<CancellationToken>())
            .ReturnsAsync(new HttpResponseMessage
            {
                StatusCode = HttpStatusCode.OK,
                Content = new StringContent(content)
            });
        return mock.Object;
    }

    private static AlphaVantageApiClient CreateClient(HttpMessageHandler handler, int dailyLimit = 25)
    {
        var httpClient = new HttpClient(handler);
        var options = Options.Create(new AlphaVantageApiClientOptions
        {
            ApiKey = "test-key",
            DailyLimit = dailyLimit
        });
        return new AlphaVantageApiClient(httpClient, options, NullLogger<AlphaVantageApiClient>.Instance);
    }
}
```

Create scheduler tests:

```csharp
public class CollectionSchedulerTests
{
    [Fact]
    public async Task GetSeriesDueForCollection_PrioritizesByPriority()
    {
        // Series with lower priority number should be collected first
    }

    [Fact]
    public async Task GetSeriesDueForCollection_RespectsAvailableRequests()
    {
        // Should not return more series than available requests
    }

    [Fact]
    public async Task IsCollectionDue_ReturnsTrueForNeverCollected()
    {
        // Series with null LastCollectedAt should always be due
    }
}
```

**Acceptance:** Tests pass, rate limiting verified

---

### T3.12: Integration Test with ThresholdEngine
**Estimate:** 30 min

1. Add AlphaVantageCollector to infrastructure compose.yaml
2. Configure ThresholdEngine to connect to all three collectors
3. Create Cu/Au pattern combining Nasdaq gold + Alpha Vantage copper
4. Verify pattern evaluation works

Create pattern `ThresholdEngine/config/patterns/commodity/copper-gold-ratio.json`:
```json
{
  "patternId": "copper-gold-ratio",
  "name": "Copper/Gold Ratio (Risk Sentiment)",
  "description": "Cu/Au ratio below 4 suggests risk-off, above 5 suggests risk-on",
  "category": "Commodity",
  "enabled": true,
  "expression": "ctx.GetLatest(\"AV/COPPER\") != null && ctx.GetLatest(\"LBMA/GOLD\") != null",
  "signalExpression": "var cu = ctx.GetLatest(\"AV/COPPER\") ?? 0m; var au = ctx.GetLatest(\"LBMA/GOLD\") ?? 2000m; var ratio = cu / au; return ratio < 4m ? -1m : ratio > 5m ? 1m : 0m;",
  "requiredSeries": ["AV/COPPER", "LBMA/GOLD"],
  "applicableRegimes": ["Crisis", "Recession", "LateCycle", "Neutral", "Recovery", "Growth"]
}
```

**Acceptance:** Cu/Au pattern evaluates successfully

---

### T3.13: Add to Ansible Deployment
**Estimate:** 20 min

Update `ansible/playbooks/site.yml` and compose.yaml to include AlphaVantageCollector.

**Acceptance:** Full deployment works

---

### T3.14: Add Monitoring Dashboard
**Estimate:** 30 min

Create Grafana dashboard `AlphaVantage API Usage`:

Panels:
- Daily requests used (gauge: 0-25)
- Requests remaining (single stat)
- Collection success rate (time series)
- Data freshness by series (table)
- API errors (logs panel)

Add alerts:
- Warning when remaining < 5
- Critical when remaining = 0

**Acceptance:** Dashboard deployed, metrics visible

---

## Acceptance Criteria

- [ ] AlphaVantageCollector service runs and collects copper/WTI data
- [ ] Rate limiting strictly enforced (never exceeds 25/day)
- [ ] Smart scheduler prioritizes series appropriately
- [ ] Daily counter resets at midnight UTC
- [ ] gRPC streaming works
- [ ] ThresholdEngine receives events
- [ ] Cu/Au ratio pattern evaluates with combined data
- [ ] All tests pass
- [ ] Monitoring dashboard shows API usage
- [ ] Ansible deployment works
- [ ] Documentation complete

## Environment Variables

```bash
ALPHAVANTAGE_API_KEY=your_api_key_here
DB_PASSWORD=your_db_password
ConnectionStrings__Atlas=Host=timescaledb;Database=atlas;Username=atlas;Password=${DB_PASSWORD}
```

## Ports

- 5006: REST API / Health
- 5007: gRPC

## Rate Limit Strategy

With 25 requests/day and 6 collection cycles (every 4 hours):
- ~4 requests per cycle
- Priority 1: COPPER (essential for Cu/Au)
- Priority 2-3: WTI, BRENT (daily oil prices)
- Priority 5: NATURAL_GAS
- Priority 10: ALL_COMMODITIES (monthly index)

Monthly series (COPPER) collected once per month = 1 request/month
Daily series (WTI, BRENT, NATURAL_GAS) = ~21 requests/day

This fits within the 25/day limit with buffer for retries.

## Notes

- Copper is monthly on Alpha Vantage - may want to explore other sources for daily copper
- Consider premium tier ($50/month) if daily copper needed
- Health endpoint exposes remaining requests for monitoring
- Scheduler ensures high-priority series always collected first
