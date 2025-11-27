# E2: NasdaqCollector

## Goal
Ingest LBMA gold prices (and other Nasdaq Data Link series) into ATLAS, enabling Cu/Au ratio patterns and other commodity-based indicators.

## Context

Nasdaq Data Link (formerly Quandl) provides free access to LBMA gold fixing prices with daily frequency and data back to 1968. This fills the gap left when FRED removed gold price data in January 2022.

**Key Data:**
- `LBMA/GOLD` - London Gold Fixing (USD AM/PM, GBP AM/PM, EUR AM/PM)
- Daily frequency
- Free tier: 50,000 calls/day (authenticated)

## Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    NasdaqCollector                       │
│  ┌─────────────┐  ┌──────────────┐  ┌───────────────┐  │
│  │ Collection  │  │   Nasdaq     │  │    gRPC       │  │
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

- E1 complete (shared Events project with proto and client)
- Nasdaq Data Link API key (free registration at data.nasdaq.com)
- TimescaleDB running with ATLAS schema
- Understanding of FredCollector patterns (this follows same structure)

## Project Structure

```
NasdaqCollector/
├── src/
│   ├── NasdaqCollector.Core/           # Domain models, interfaces
│   ├── NasdaqCollector.Application/    # Business logic, services
│   ├── NasdaqCollector.Infrastructure/ # Nasdaq API client, data access
│   ├── NasdaqCollector.Api/            # REST endpoints (optional)
│   ├── NasdaqCollector.Grpc/           # gRPC server
│   └── NasdaqCollector.Service/        # Worker service host
├── tests/
│   └── NasdaqCollector.UnitTests/
├── config/
│   └── series.json                     # Configured series
├── .devcontainer/
│   ├── devcontainer.json
│   └── compose.yaml
└── Containerfile
```

## Tasks

### T2.1: Create Solution Structure
**Estimate:** 30 min

```bash
cd ~/ATLAS
mkdir -p NasdaqCollector/src NasdaqCollector/tests NasdaqCollector/config

# Create solution
cd NasdaqCollector
dotnet new sln -n NasdaqCollector

# Create projects
dotnet new classlib -n NasdaqCollector.Core -o src/NasdaqCollector.Core -f net9.0
dotnet new classlib -n NasdaqCollector.Application -o src/NasdaqCollector.Application -f net9.0
dotnet new classlib -n NasdaqCollector.Infrastructure -o src/NasdaqCollector.Infrastructure -f net9.0
dotnet new classlib -n NasdaqCollector.Grpc -o src/NasdaqCollector.Grpc -f net9.0
dotnet new worker -n NasdaqCollector.Service -o src/NasdaqCollector.Service -f net9.0
dotnet new xunit -n NasdaqCollector.UnitTests -o tests/NasdaqCollector.UnitTests -f net9.0

# Add to solution
dotnet sln add src/NasdaqCollector.Core
dotnet sln add src/NasdaqCollector.Application
dotnet sln add src/NasdaqCollector.Infrastructure
dotnet sln add src/NasdaqCollector.Grpc
dotnet sln add src/NasdaqCollector.Service
dotnet sln add tests/NasdaqCollector.UnitTests

# Add project references
cd src/NasdaqCollector.Application
dotnet add reference ../NasdaqCollector.Core

cd ../NasdaqCollector.Infrastructure
dotnet add reference ../NasdaqCollector.Core
dotnet add reference ../NasdaqCollector.Application

cd ../NasdaqCollector.Grpc
dotnet add reference ../NasdaqCollector.Core
dotnet add reference ../../../../Events/src/Events

cd ../NasdaqCollector.Service
dotnet add reference ../NasdaqCollector.Application
dotnet add reference ../NasdaqCollector.Infrastructure
dotnet add reference ../NasdaqCollector.Grpc
```

**Acceptance:** Solution builds with empty projects

---

### T2.2: Implement Core Domain Models
**Estimate:** 30 min

Create `NasdaqCollector.Core/Models/NasdaqSeries.cs`:
```csharp
namespace NasdaqCollector.Core.Models;

public record NasdaqSeries
{
    public required string DatabaseCode { get; init; }  // "LBMA"
    public required string DatasetCode { get; init; }   // "GOLD"
    public required string SeriesId { get; init; }      // "LBMA/GOLD" or normalized
    public required string Title { get; init; }
    public required string Category { get; init; }
    public required string Frequency { get; init; }     // "daily", "weekly", etc.
    public string? ValueColumn { get; init; }           // Which column to use (e.g., "USD (AM)")
    public bool IsActive { get; init; } = true;
    public DateTime? LastCollectedAt { get; init; }
    
    public string FullCode => $"{DatabaseCode}/{DatasetCode}";
}
```

Create `NasdaqCollector.Core/Models/NasdaqObservation.cs`:
```csharp
namespace NasdaqCollector.Core.Models;

public record NasdaqObservation
{
    public required string SeriesId { get; init; }
    public required DateOnly Date { get; init; }
    public required decimal? Value { get; init; }
    public required DateTime CollectedAt { get; init; }
}
```

Create `NasdaqCollector.Core/Interfaces/INasdaqApiClient.cs`:
```csharp
namespace NasdaqCollector.Core.Interfaces;

public interface INasdaqApiClient
{
    Task<IReadOnlyList<NasdaqObservation>> GetObservationsAsync(
        string databaseCode,
        string datasetCode,
        DateOnly? startDate = null,
        DateOnly? endDate = null,
        string? column = null,
        CancellationToken cancellationToken = default);
}
```

Create `NasdaqCollector.Core/Interfaces/INasdaqRepository.cs`:
```csharp
namespace NasdaqCollector.Core.Interfaces;

public interface INasdaqRepository
{
    Task<IReadOnlyList<NasdaqSeries>> GetActiveSeriesAsync(CancellationToken ct = default);
    Task<NasdaqSeries?> GetSeriesAsync(string seriesId, CancellationToken ct = default);
    Task UpsertObservationsAsync(IEnumerable<NasdaqObservation> observations, CancellationToken ct = default);
    Task<NasdaqObservation?> GetLatestObservationAsync(string seriesId, CancellationToken ct = default);
    Task UpdateLastCollectedAsync(string seriesId, DateTime collectedAt, CancellationToken ct = default);
}
```

**Acceptance:** Core models compile, interfaces defined

---

### T2.3: Implement Nasdaq API Client
**Estimate:** 1 hour

Create `NasdaqCollector.Infrastructure/NasdaqApiClient.cs`:

```csharp
using System.Net.Http.Json;
using System.Text.Json;
using System.Text.Json.Serialization;
using Microsoft.Extensions.Logging;
using Microsoft.Extensions.Options;
using NasdaqCollector.Core.Interfaces;
using NasdaqCollector.Core.Models;

namespace NasdaqCollector.Infrastructure;

public class NasdaqApiClientOptions
{
    public required string ApiKey { get; set; }
    public string BaseUrl { get; set; } = "https://data.nasdaq.com/api/v3";
    public int MaxRetries { get; set; } = 3;
    public TimeSpan RetryDelay { get; set; } = TimeSpan.FromSeconds(2);
}

public sealed class NasdaqApiClient : INasdaqApiClient
{
    private readonly HttpClient _httpClient;
    private readonly NasdaqApiClientOptions _options;
    private readonly ILogger<NasdaqApiClient> _logger;

    public NasdaqApiClient(
        HttpClient httpClient,
        IOptions<NasdaqApiClientOptions> options,
        ILogger<NasdaqApiClient> logger)
    {
        _httpClient = httpClient;
        _options = options.Value;
        _logger = logger;
    }

    public async Task<IReadOnlyList<NasdaqObservation>> GetObservationsAsync(
        string databaseCode,
        string datasetCode,
        DateOnly? startDate = null,
        DateOnly? endDate = null,
        string? column = null,
        CancellationToken cancellationToken = default)
    {
        var url = BuildUrl(databaseCode, datasetCode, startDate, endDate);
        
        _logger.LogDebug("Fetching {Database}/{Dataset} from Nasdaq", databaseCode, datasetCode);
        
        var response = await ExecuteWithRetryAsync(
            () => _httpClient.GetAsync(url, cancellationToken),
            cancellationToken);
        
        response.EnsureSuccessStatusCode();
        
        var data = await response.Content.ReadFromJsonAsync<NasdaqDatasetResponse>(
            cancellationToken: cancellationToken);
        
        if (data?.Dataset?.Data == null)
            return [];
        
        var seriesId = $"{databaseCode}/{datasetCode}";
        var columnIndex = GetColumnIndex(data.Dataset.ColumnNames, column);
        var collectedAt = DateTime.UtcNow;
        
        return data.Dataset.Data
            .Select(row => ParseObservation(row, seriesId, columnIndex, collectedAt))
            .Where(o => o != null)
            .Cast<NasdaqObservation>()
            .ToList();
    }

    private string BuildUrl(
        string databaseCode, 
        string datasetCode, 
        DateOnly? startDate, 
        DateOnly? endDate)
    {
        var url = $"{_options.BaseUrl}/datasets/{databaseCode}/{datasetCode}.json?api_key={_options.ApiKey}";
        
        if (startDate.HasValue)
            url += $"&start_date={startDate.Value:yyyy-MM-dd}";
        if (endDate.HasValue)
            url += $"&end_date={endDate.Value:yyyy-MM-dd}";
        
        return url;
    }

    private static int GetColumnIndex(List<string>? columnNames, string? targetColumn)
    {
        if (columnNames == null || string.IsNullOrEmpty(targetColumn))
            return 1; // Default to first data column (index 0 is usually Date)
        
        var index = columnNames.FindIndex(c => 
            c.Equals(targetColumn, StringComparison.OrdinalIgnoreCase));
        
        return index >= 0 ? index : 1;
    }

    private static NasdaqObservation? ParseObservation(
        List<JsonElement> row, 
        string seriesId, 
        int valueIndex,
        DateTime collectedAt)
    {
        if (row.Count <= valueIndex)
            return null;
        
        var dateStr = row[0].GetString();
        if (!DateOnly.TryParse(dateStr, out var date))
            return null;
        
        decimal? value = null;
        if (row[valueIndex].ValueKind == JsonValueKind.Number)
            value = row[valueIndex].GetDecimal();
        
        return new NasdaqObservation
        {
            SeriesId = seriesId,
            Date = date,
            Value = value,
            CollectedAt = collectedAt
        };
    }

    private async Task<HttpResponseMessage> ExecuteWithRetryAsync(
        Func<Task<HttpResponseMessage>> action,
        CancellationToken cancellationToken)
    {
        for (int attempt = 0; attempt < _options.MaxRetries; attempt++)
        {
            try
            {
                return await action();
            }
            catch (HttpRequestException ex) when (attempt < _options.MaxRetries - 1)
            {
                _logger.LogWarning(ex, "Request failed, attempt {Attempt}/{Max}", 
                    attempt + 1, _options.MaxRetries);
                await Task.Delay(_options.RetryDelay, cancellationToken);
            }
        }
        
        return await action(); // Final attempt, let exception propagate
    }
}

// Response DTOs
internal record NasdaqDatasetResponse
{
    [JsonPropertyName("dataset")]
    public NasdaqDataset? Dataset { get; init; }
}

internal record NasdaqDataset
{
    [JsonPropertyName("column_names")]
    public List<string>? ColumnNames { get; init; }
    
    [JsonPropertyName("data")]
    public List<List<JsonElement>>? Data { get; init; }
}
```

Add packages to `NasdaqCollector.Infrastructure.csproj`:
```xml
<ItemGroup>
  <PackageReference Include="Microsoft.Extensions.Http" Version="9.0.0" />
  <PackageReference Include="Microsoft.Extensions.Options" Version="9.0.0" />
  <PackageReference Include="Microsoft.Extensions.Logging.Abstractions" Version="9.0.0" />
</ItemGroup>
```

**Acceptance:** API client compiles, handles Nasdaq Data Link response format

---

### T2.4: Implement TimescaleDB Repository
**Estimate:** 45 min

Create `NasdaqCollector.Infrastructure/NasdaqRepository.cs`:

```csharp
using Dapper;
using Microsoft.Extensions.Logging;
using NasdaqCollector.Core.Interfaces;
using NasdaqCollector.Core.Models;
using Npgsql;

namespace NasdaqCollector.Infrastructure;

public sealed class NasdaqRepository : INasdaqRepository
{
    private readonly string _connectionString;
    private readonly ILogger<NasdaqRepository> _logger;

    public NasdaqRepository(string connectionString, ILogger<NasdaqRepository> logger)
    {
        _connectionString = connectionString;
        _logger = logger;
    }

    public async Task<IReadOnlyList<NasdaqSeries>> GetActiveSeriesAsync(CancellationToken ct = default)
    {
        await using var conn = new NpgsqlConnection(_connectionString);
        
        var result = await conn.QueryAsync<NasdaqSeries>(
            """
            SELECT database_code as DatabaseCode, dataset_code as DatasetCode,
                   series_id as SeriesId, title as Title, category as Category,
                   frequency as Frequency, value_column as ValueColumn,
                   is_active as IsActive, last_collected_at as LastCollectedAt
            FROM nasdaq_series
            WHERE is_active = true
            """);
        
        return result.ToList();
    }

    public async Task<NasdaqSeries?> GetSeriesAsync(string seriesId, CancellationToken ct = default)
    {
        await using var conn = new NpgsqlConnection(_connectionString);
        
        return await conn.QuerySingleOrDefaultAsync<NasdaqSeries>(
            """
            SELECT database_code as DatabaseCode, dataset_code as DatasetCode,
                   series_id as SeriesId, title as Title, category as Category,
                   frequency as Frequency, value_column as ValueColumn,
                   is_active as IsActive, last_collected_at as LastCollectedAt
            FROM nasdaq_series
            WHERE series_id = @SeriesId
            """,
            new { SeriesId = seriesId });
    }

    public async Task UpsertObservationsAsync(
        IEnumerable<NasdaqObservation> observations, 
        CancellationToken ct = default)
    {
        await using var conn = new NpgsqlConnection(_connectionString);
        await conn.OpenAsync(ct);
        
        await using var transaction = await conn.BeginTransactionAsync(ct);
        
        foreach (var obs in observations)
        {
            await conn.ExecuteAsync(
                """
                INSERT INTO nasdaq_observations (series_id, date, value, collected_at)
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

    public async Task<NasdaqObservation?> GetLatestObservationAsync(
        string seriesId, 
        CancellationToken ct = default)
    {
        await using var conn = new NpgsqlConnection(_connectionString);
        
        return await conn.QuerySingleOrDefaultAsync<NasdaqObservation>(
            """
            SELECT series_id as SeriesId, date as Date, value as Value, collected_at as CollectedAt
            FROM nasdaq_observations
            WHERE series_id = @SeriesId
            ORDER BY date DESC
            LIMIT 1
            """,
            new { SeriesId = seriesId });
    }

    public async Task UpdateLastCollectedAsync(
        string seriesId, 
        DateTime collectedAt, 
        CancellationToken ct = default)
    {
        await using var conn = new NpgsqlConnection(_connectionString);
        
        await conn.ExecuteAsync(
            """
            UPDATE nasdaq_series 
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

### T2.5: Create Database Migration
**Estimate:** 20 min

Create `NasdaqCollector/migrations/001_initial_schema.sql`:

```sql
-- Nasdaq series configuration
CREATE TABLE IF NOT EXISTS nasdaq_series (
    series_id TEXT PRIMARY KEY,
    database_code TEXT NOT NULL,
    dataset_code TEXT NOT NULL,
    title TEXT NOT NULL,
    category TEXT NOT NULL,
    frequency TEXT NOT NULL,
    value_column TEXT,
    is_active BOOLEAN NOT NULL DEFAULT true,
    last_collected_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Nasdaq observations (hypertable)
CREATE TABLE IF NOT EXISTS nasdaq_observations (
    series_id TEXT NOT NULL,
    date TIMESTAMPTZ NOT NULL,
    value NUMERIC,
    collected_at TIMESTAMPTZ NOT NULL,
    PRIMARY KEY (series_id, date)
);

-- Convert to hypertable
SELECT create_hypertable('nasdaq_observations', 'date', if_not_exists => TRUE);

-- Indexes
CREATE INDEX IF NOT EXISTS idx_nasdaq_observations_series 
    ON nasdaq_observations (series_id, date DESC);

-- Initial series configuration
INSERT INTO nasdaq_series (series_id, database_code, dataset_code, title, category, frequency, value_column)
VALUES 
    ('LBMA/GOLD', 'LBMA', 'GOLD', 'Gold Price: London Fixing', 'Commodity', 'daily', 'USD (AM)')
ON CONFLICT (series_id) DO NOTHING;
```

**Acceptance:** Migration runs successfully against TimescaleDB

---

### T2.6: Implement Collection Worker
**Estimate:** 45 min

Create `NasdaqCollector.Application/Services/NasdaqCollectionService.cs`:

```csharp
using Microsoft.Extensions.Logging;
using NasdaqCollector.Core.Interfaces;
using NasdaqCollector.Core.Models;

namespace NasdaqCollector.Application.Services;

public interface INasdaqCollectionService
{
    Task<CollectionResult> CollectSeriesAsync(NasdaqSeries series, CancellationToken ct = default);
    Task<IReadOnlyList<CollectionResult>> CollectAllAsync(CancellationToken ct = default);
}

public record CollectionResult(
    string SeriesId, 
    int ObservationsCollected, 
    bool Success, 
    string? Error = null);

public sealed class NasdaqCollectionService : INasdaqCollectionService
{
    private readonly INasdaqApiClient _apiClient;
    private readonly INasdaqRepository _repository;
    private readonly ILogger<NasdaqCollectionService> _logger;

    public NasdaqCollectionService(
        INasdaqApiClient apiClient,
        INasdaqRepository repository,
        ILogger<NasdaqCollectionService> logger)
    {
        _apiClient = apiClient;
        _repository = repository;
        _logger = logger;
    }

    public async Task<CollectionResult> CollectSeriesAsync(
        NasdaqSeries series, 
        CancellationToken ct = default)
    {
        try
        {
            // Determine start date (incremental collection)
            var latest = await _repository.GetLatestObservationAsync(series.SeriesId, ct);
            var startDate = latest?.Date.AddDays(1) ?? DateOnly.FromDateTime(DateTime.UtcNow.AddYears(-2));
            
            _logger.LogInformation(
                "Collecting {SeriesId} from {StartDate}", 
                series.SeriesId, startDate);
            
            var observations = await _apiClient.GetObservationsAsync(
                series.DatabaseCode,
                series.DatasetCode,
                startDate,
                DateOnly.FromDateTime(DateTime.UtcNow),
                series.ValueColumn,
                ct);
            
            if (observations.Count > 0)
            {
                await _repository.UpsertObservationsAsync(observations, ct);
                await _repository.UpdateLastCollectedAsync(series.SeriesId, DateTime.UtcNow, ct);
            }
            
            _logger.LogInformation(
                "Collected {Count} observations for {SeriesId}", 
                observations.Count, series.SeriesId);
            
            return new CollectionResult(series.SeriesId, observations.Count, true);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Failed to collect {SeriesId}", series.SeriesId);
            return new CollectionResult(series.SeriesId, 0, false, ex.Message);
        }
    }

    public async Task<IReadOnlyList<CollectionResult>> CollectAllAsync(CancellationToken ct = default)
    {
        var series = await _repository.GetActiveSeriesAsync(ct);
        var results = new List<CollectionResult>();
        
        foreach (var s in series)
        {
            var result = await CollectSeriesAsync(s, ct);
            results.Add(result);
            
            // Small delay between series to be nice to API
            await Task.Delay(100, ct);
        }
        
        return results;
    }
}
```

Create `NasdaqCollector.Service/Workers/CollectionWorker.cs`:

```csharp
using NasdaqCollector.Application.Services;

namespace NasdaqCollector.Service.Workers;

public sealed class CollectionWorker : BackgroundService
{
    private readonly INasdaqCollectionService _collectionService;
    private readonly ILogger<CollectionWorker> _logger;
    private readonly TimeSpan _interval = TimeSpan.FromHours(6); // Gold updates twice daily

    public CollectionWorker(
        INasdaqCollectionService collectionService,
        ILogger<CollectionWorker> logger)
    {
        _collectionService = collectionService;
        _logger = logger;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        _logger.LogInformation("NasdaqCollector worker starting");
        
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
        _logger.LogInformation("Starting collection cycle");
        
        var results = await _collectionService.CollectAllAsync(ct);
        
        var success = results.Count(r => r.Success);
        var failed = results.Count(r => !r.Success);
        var total = results.Sum(r => r.ObservationsCollected);
        
        _logger.LogInformation(
            "Collection complete: {Success} succeeded, {Failed} failed, {Total} observations",
            success, failed, total);
    }
}
```

**Acceptance:** Worker compiles, collection logic implemented

---

### T2.7: Implement gRPC Server
**Estimate:** 45 min

Create `NasdaqCollector.Grpc/Services/ObservationEventStreamService.cs`:

```csharp
using ATLAS.Events.Grpc;
using Google.Protobuf.WellKnownTypes;
using Grpc.Core;
using Microsoft.Extensions.Logging;
using NasdaqCollector.Core.Interfaces;

namespace NasdaqCollector.Grpc.Services;

public sealed class ObservationEventStreamService : ObservationEventStream.ObservationEventStreamBase
{
    private readonly INasdaqRepository _repository;
    private readonly ILogger<ObservationEventStreamService> _logger;

    public ObservationEventStreamService(
        INasdaqRepository repository,
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

        // For now, implement polling-based streaming
        // Could be enhanced with LISTEN/NOTIFY later
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
                    var evt = new Event
                    {
                        EventId = Ulid.NewUlid().ToString(),
                        OccurredAt = Timestamp.FromDateTime(
                            DateTime.SpecifyKind(latest.CollectedAt, DateTimeKind.Utc)),
                        SourceService = "NasdaqCollector",
                        SeriesCollected = new SeriesCollectedEvent
                        {
                            SeriesId = latest.SeriesId,
                            Date = Timestamp.FromDateTime(
                                DateTime.SpecifyKind(
                                    latest.Date.ToDateTime(TimeOnly.MinValue), 
                                    DateTimeKind.Utc)),
                            Value = (double)(latest.Value ?? 0),
                            CollectedAt = Timestamp.FromDateTime(
                                DateTime.SpecifyKind(latest.CollectedAt, DateTimeKind.Utc))
                        }
                    };
                    
                    await responseStream.WriteAsync(evt);
                    lastEventTime = latest.CollectedAt;
                }
            }
            
            await Task.Delay(1000, context.CancellationToken);
        }
    }

    public override async Task GetEventsSince(
        TimeRangeRequest request,
        IServerStreamWriter<Event> responseStream,
        ServerCallContext context)
    {
        // Implement historical query
        // For MVP, return latest observation for each series
        var series = await _repository.GetActiveSeriesAsync(context.CancellationToken);
        
        foreach (var s in series)
        {
            var latest = await _repository.GetLatestObservationAsync(
                s.SeriesId, context.CancellationToken);
            
            if (latest != null)
            {
                var evt = new Event
                {
                    EventId = Ulid.NewUlid().ToString(),
                    OccurredAt = Timestamp.FromDateTime(
                        DateTime.SpecifyKind(latest.CollectedAt, DateTimeKind.Utc)),
                    SourceService = "NasdaqCollector",
                    SeriesCollected = new SeriesCollectedEvent
                    {
                        SeriesId = latest.SeriesId,
                        Date = Timestamp.FromDateTime(
                            DateTime.SpecifyKind(
                                latest.Date.ToDateTime(TimeOnly.MinValue), 
                                DateTimeKind.Utc)),
                        Value = (double)(latest.Value ?? 0),
                        CollectedAt = Timestamp.FromDateTime(
                            DateTime.SpecifyKind(latest.CollectedAt, DateTimeKind.Utc))
                    }
                };
                
                await responseStream.WriteAsync(evt);
            }
        }
    }

    public override Task<HealthResponse> Health(HealthRequest request, ServerCallContext context)
    {
        return Task.FromResult(new HealthResponse
        {
            Healthy = true,
            Message = "NasdaqCollector healthy"
        });
    }
}
```

Add packages to `NasdaqCollector.Grpc.csproj`:
```xml
<ItemGroup>
  <PackageReference Include="Grpc.AspNetCore" Version="2.67.0" />
  <PackageReference Include="Ulid" Version="1.3.4" />
</ItemGroup>

<ItemGroup>
  <Protobuf Include="..\..\..\..\Events\src\Events\Protos\observation_events.proto" 
            GrpcServices="Server" />
</ItemGroup>
```

**Acceptance:** gRPC server compiles, implements streaming interface

---

### T2.8: Configure Service Host
**Estimate:** 30 min

Update `NasdaqCollector.Service/Program.cs`:

```csharp
using NasdaqCollector.Application.Services;
using NasdaqCollector.Core.Interfaces;
using NasdaqCollector.Grpc.Services;
using NasdaqCollector.Infrastructure;
using NasdaqCollector.Service.Workers;

var builder = WebApplication.CreateBuilder(args);

// Configuration
builder.Services.Configure<NasdaqApiClientOptions>(
    builder.Configuration.GetSection("Nasdaq"));

// HTTP Client for Nasdaq API
builder.Services.AddHttpClient<INasdaqApiClient, NasdaqApiClient>();

// Repository
var connectionString = builder.Configuration.GetConnectionString("Atlas") 
    ?? throw new InvalidOperationException("Connection string 'Atlas' not configured");
builder.Services.AddSingleton<INasdaqRepository>(sp => 
    new NasdaqRepository(connectionString, sp.GetRequiredService<ILogger<NasdaqRepository>>()));

// Services
builder.Services.AddSingleton<INasdaqCollectionService, NasdaqCollectionService>();

// Workers
builder.Services.AddHostedService<CollectionWorker>();

// gRPC
builder.Services.AddGrpc();

var app = builder.Build();

// gRPC endpoint
app.MapGrpcService<ObservationEventStreamService>();

// Health check
app.MapGet("/health", () => Results.Ok(new { status = "healthy" }));

app.Run();
```

Create `NasdaqCollector.Service/appsettings.json`:
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
  "Nasdaq": {
    "ApiKey": "${NASDAQ_API_KEY}",
    "BaseUrl": "https://data.nasdaq.com/api/v3"
  },
  "Kestrel": {
    "Endpoints": {
      "Http": {
        "Url": "http://0.0.0.0:5004"
      },
      "Grpc": {
        "Url": "http://0.0.0.0:5005",
        "Protocols": "Http2"
      }
    }
  }
}
```

**Acceptance:** Service starts, workers run, gRPC endpoint responds

---

### T2.9: Create Containerfile and DevContainer
**Estimate:** 30 min

Create `NasdaqCollector/Containerfile`:

```dockerfile
FROM mcr.microsoft.com/dotnet/sdk:9.0 AS build
WORKDIR /src

# Copy project files
COPY src/NasdaqCollector.Core/*.csproj src/NasdaqCollector.Core/
COPY src/NasdaqCollector.Application/*.csproj src/NasdaqCollector.Application/
COPY src/NasdaqCollector.Infrastructure/*.csproj src/NasdaqCollector.Infrastructure/
COPY src/NasdaqCollector.Grpc/*.csproj src/NasdaqCollector.Grpc/
COPY src/NasdaqCollector.Service/*.csproj src/NasdaqCollector.Service/

# Copy Events project reference
COPY ../Events/src/Events/*.csproj ../Events/src/Events/

# Restore
RUN dotnet restore src/NasdaqCollector.Service/NasdaqCollector.Service.csproj

# Copy source
COPY . .
COPY ../Events/src/Events ../Events/src/Events

# Build
RUN dotnet publish src/NasdaqCollector.Service/NasdaqCollector.Service.csproj \
    -c Release -o /app --no-restore

# Runtime image
FROM mcr.microsoft.com/dotnet/aspnet:9.0
WORKDIR /app
COPY --from=build /app .

ENV ASPNETCORE_URLS=http://+:5004;http://+:5005
EXPOSE 5004 5005

ENTRYPOINT ["dotnet", "NasdaqCollector.Service.dll"]
```

Create `NasdaqCollector/.devcontainer/devcontainer.json`:
```json
{
  "name": "NasdaqCollector Dev",
  "dockerComposeFile": "compose.yaml",
  "service": "dev",
  "workspaceFolder": "/workspace",
  "customizations": {
    "vscode": {
      "extensions": [
        "ms-dotnettools.csharp",
        "ms-dotnettools.csdevkit"
      ]
    }
  },
  "forwardPorts": [5004, 5005]
}
```

Create `NasdaqCollector/.devcontainer/compose.yaml`:
```yaml
services:
  dev:
    image: mcr.microsoft.com/dotnet/sdk:9.0
    volumes:
      - ..:/workspace:cached
      - ../../Events:/Events:cached
    working_dir: /workspace
    command: sleep infinity
    environment:
      - NASDAQ_API_KEY=${NASDAQ_API_KEY}
      - ConnectionStrings__Atlas=Host=timescaledb;Database=atlas;Username=atlas;Password=atlas
    depends_on:
      - timescaledb
  
  timescaledb:
    image: timescale/timescaledb:latest-pg16
    environment:
      POSTGRES_USER: atlas
      POSTGRES_PASSWORD: atlas
      POSTGRES_DB: atlas
    ports:
      - "5432:5432"
    volumes:
      - timescale_data:/var/lib/postgresql/data

volumes:
  timescale_data:
```

**Acceptance:** Container builds, devcontainer starts

---

### T2.10: Write Unit Tests
**Estimate:** 1 hour

Create tests for:
- `NasdaqApiClient` - response parsing, retry logic
- `NasdaqCollectionService` - incremental collection logic
- `NasdaqRepository` - database operations (integration)

Example test `tests/NasdaqCollector.UnitTests/NasdaqApiClientTests.cs`:

```csharp
using System.Net;
using System.Text.Json;
using Microsoft.Extensions.Logging.Abstractions;
using Microsoft.Extensions.Options;
using Moq;
using Moq.Protected;
using NasdaqCollector.Infrastructure;

namespace NasdaqCollector.UnitTests;

public class NasdaqApiClientTests
{
    [Fact]
    public async Task GetObservationsAsync_ParsesLbmaGoldResponse()
    {
        // Arrange
        var response = new
        {
            dataset = new
            {
                column_names = new[] { "Date", "USD (AM)", "USD (PM)", "GBP (AM)" },
                data = new[]
                {
                    new object[] { "2025-11-26", 2650.50, 2651.25, 2080.00 },
                    new object[] { "2025-11-25", 2645.00, 2648.75, 2076.50 }
                }
            }
        };
        
        var handler = CreateMockHandler(JsonSerializer.Serialize(response));
        var client = CreateClient(handler);
        
        // Act
        var observations = await client.GetObservationsAsync("LBMA", "GOLD", column: "USD (AM)");
        
        // Assert
        Assert.Equal(2, observations.Count);
        Assert.Equal("LBMA/GOLD", observations[0].SeriesId);
        Assert.Equal(2650.50m, observations[0].Value);
    }

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

    private static NasdaqApiClient CreateClient(HttpMessageHandler handler)
    {
        var httpClient = new HttpClient(handler);
        var options = Options.Create(new NasdaqApiClientOptions
        {
            ApiKey = "test-key",
            BaseUrl = "https://data.nasdaq.com/api/v3"
        });
        return new NasdaqApiClient(httpClient, options, NullLogger<NasdaqApiClient>.Instance);
    }
}
```

**Acceptance:** Tests pass, >80% coverage on critical paths

---

### T2.11: Integration Test with ThresholdEngine
**Estimate:** 30 min

1. Add NasdaqCollector to compose.yaml
2. Configure ThresholdEngine to connect to both collectors
3. Create Cu/Au pattern that uses gold data
4. Verify pattern evaluation works

**Acceptance:** Gold prices flow to ThresholdEngine, patterns evaluate

---

### T2.12: Add to Ansible Deployment
**Estimate:** 20 min

Update `ansible/playbooks/site.yml` and related files to deploy NasdaqCollector alongside existing services.

**Acceptance:** `ansible-playbook site.yml` deploys NasdaqCollector

---

## Acceptance Criteria

- [ ] NasdaqCollector service runs and collects LBMA/GOLD data
- [ ] Data stored in TimescaleDB nasdaq_observations table
- [ ] gRPC streaming works (verified with grpcurl)
- [ ] ThresholdEngine receives ObservationCollectedEvent from NasdaqCollector
- [ ] All tests pass
- [ ] Container builds and runs
- [ ] Ansible deployment works
- [ ] Documentation complete

## Environment Variables

```bash
NASDAQ_API_KEY=your_api_key_here
DB_PASSWORD=your_db_password
ConnectionStrings__Atlas=Host=timescaledb;Database=atlas;Username=atlas;Password=${DB_PASSWORD}
```

## Ports

- 5004: REST API / Health
- 5005: gRPC

## Notes

- Gold fixing happens twice daily (10:30 AM and 3:00 PM London time)
- Collection interval of 6 hours is sufficient
- Consider adding more LBMA series (silver, platinum) in future
- Rate limit is generous (50K/day) - no aggressive throttling needed
