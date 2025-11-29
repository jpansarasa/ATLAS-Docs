# E6: OfrCollector

## Goal

Ingest Office of Financial Research (OFR) data into ATLAS, providing authoritative government-sourced financial stress metrics, short-term funding market data, money market fund holdings, and hedge fund statistics. This collector delivers direct regime signals for NBFI/shadow banking monitoring without requiring API keys.

## Context

The OFR is the U.S. Treasury's financial stability research arm, created by Dodd-Frank to monitor systemic risk. Their data is:
- **Authoritative**: Official government source, methodologically rigorous
- **Free & Open**: No API keys, no registration, no rate limits documented
- **Daily Updates**: FSI updates daily (T+2), STFM/HFM series update daily to monthly
- **Directly Relevant**: FSI is a proven leading indicator for economic activity (Granger-causes Chicago Fed CFNAI)

**Strategic Value for ATLAS:**
- **OFR Financial Stress Index (FSI)**: Daily composite of 33 market variables across 5 categories (credit, equity valuation, funding, safe assets, volatility) and 3 regions (US, advanced economies, emerging markets). Positive = above-average stress. Direct regime signal.
- **Short-term Funding Monitor (STFM)**: Repo rates, money market fund data, primary dealer statistics, Treasury yields. Core NBFI plumbing visibility.
- **Hedge Fund Monitor (HFM)**: Quarterly hedge fund leverage, liquidity, counterparty exposure. Shadow banking stress early warning.
- **Bank Systemic Risk Monitor**: G-SIB scores, contagion index. Too-big-to-fail monitoring.

**Comparison to Existing Sources:**

| Metric | FRED | Finnhub | OFR | Winner |
|--------|------|---------|-----|--------|
| Financial Stress Index | STLFSI (weekly, St. Louis Fed) | ❌ | FSI (daily, 33 variables) | **OFR** |
| Repo Rates | SOFR, EFFR | ❌ | Granular DVP/GCF rates | **OFR** (detail) |
| MMF Holdings | ❌ | ❌ | Full N-MFP breakdown | **OFR** |
| Hedge Fund Data | ❌ | ❌ | Form PF aggregates | **OFR** |
| Primary Dealer Stats | ❌ | ❌ | NY Fed via OFR | **OFR** |

## API Overview

OFR provides **three separate APIs** with identical structure:

| API | Base URL | Datasets | Update Frequency |
|-----|----------|----------|------------------|
| **STFM** (Short-term Funding) | `https://data.financialresearch.gov/v1/` | fnyr, repo, mmf, nypd, tyld | Daily to Monthly |
| **HFM** (Hedge Fund) | `https://data.financialresearch.gov/hf/v1/` | fpf, ficc | Quarterly |
| **FSI** (Financial Stress) | Web scrape or CSV download | fsi | Daily (T+2) |

**Note**: The FSI does not have a documented JSON API like STFM/HFM. It's available via:
1. Interactive chart on website (JavaScript-rendered)
2. CSV download link (to be discovered/scraped)
3. FRED mirror: `STLFSI2` (St. Louis Fed version, slightly different methodology)

### STFM API Endpoints

```
Base: https://data.financialresearch.gov/v1/

Metadata:
  GET /metadata/mnemonics                    # List all series
  GET /metadata/mnemonics?output=by_dataset  # Group by dataset
  GET /metadata/query?mnemonic={id}          # Series metadata
  GET /metadata/search?query={term}          # Search series

Data:
  GET /series/timeseries?mnemonic={id}       # Single series data
  GET /series/full?mnemonic={id}             # Data + metadata
  GET /series/multifull?mnemonics={id1,id2}  # Multiple series
  GET /series/dataset?dataset={name}         # All series in dataset
  GET /calc/spread?x={id1}&y={id2}           # Spread calculation

Parameters:
  start_date=YYYY-MM-DD
  end_date=YYYY-MM-DD
  periodicity=D|W|M|Q|A
  how=last|mean|sum|min|max
  remove_nulls=true|false
  time_format=string|epoch
```

### HFM API Endpoints

```
Base: https://data.financialresearch.gov/hf/v1/

Same structure as STFM:
  GET /metadata/mnemonics
  GET /series/dataset?dataset=fpf
  GET /series/timeseries?mnemonic={id}
```

### Available Datasets

**STFM Datasets:**

| Dataset | Name | Key Series |
|---------|------|------------|
| `fnyr` | NY Fed Reference Rates | SOFR, EFFR, OBFR |
| `repo` | U.S. Repo Markets | DVP/GCF rates, volumes |
| `mmf` | U.S. Money Market Funds | Holdings by type, counterparty |
| `nypd` | Primary Dealer Statistics | Positions, financing |
| `tyld` | Treasury Constant Maturity | 1mo-30yr yields |

**HFM Datasets:**

| Dataset | Name | Key Series |
|---------|------|------------|
| `fpf` | Form PF Aggregates | Leverage, liquidity, stress tests |
| `ficc` | FICC Sponsored Repo | Hedge fund repo volumes |

## Architecture

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                               OfrCollector                                       │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐  ┌────────────┐                 │
│  │ FSI        │  │ STFM       │  │ HFM        │  │ gRPC       │                 │
│  │ Worker     │  │ Worker     │  │ Worker     │  │ Server     │                 │
│  │ (daily)    │  │ (daily)    │  │ (quarterly)│  │            │                 │
│  └─────┬──────┘  └─────┬──────┘  └─────┬──────┘  └─────┬──────┘                 │
│        │               │               │               │                         │
│        ▼               ▼               ▼               │                         │
│  ┌─────────────────────────────────────────────────────────────────────────┐    │
│  │                         OFR API Client                                   │    │
│  │    STFM: data.financialresearch.gov/v1/                                  │    │
│  │    HFM:  data.financialresearch.gov/hf/v1/                               │    │
│  │    FSI:  financialresearch.gov/financial-stress-index/ (scrape/CSV)      │    │
│  └─────────────────────────────────────────────────────────────────────────┘    │
│        │               │               │               │                         │
│        ▼               ▼               ▼               │                         │
│  ┌─────────────────────────────────────────────────────────────────────────┐    │
│  │                      TimescaleDB Repository                              │    │
│  │  Tables: ofr_fsi, ofr_stfm_series, ofr_stfm_observations,                │    │
│  │          ofr_hfm_series, ofr_hfm_observations                            │    │
│  └─────────────────────────────────────────────────────────────────────────┘    │
│        │                                                             │          │
│        └─────────────────────────────────────────────────────────────┘          │
│                                   │                                              │
│  ┌─────────────────────────────────────────────────────────────────────────┐    │
│  │                    OpenTelemetry Instrumentation                         │    │
│  │         Traces → Tempo │ Metrics → Prometheus │ Logs → Loki             │    │
│  └─────────────────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────────────────┘
                              │
                              ▼ FinancialStressEvent / ObservationCollectedEvent
                       ThresholdEngine
```

## Prerequisites

- E1 complete (shared Events project)
- Understanding of FredCollector patterns
- TimescaleDB running with OpenTelemetry collector configured

## Project Structure

```
OfrCollector/
├── src/
│   ├── OfrCollector.Core/
│   │   ├── Models/
│   │   │   ├── FinancialStressIndex.cs
│   │   │   ├── FsiCategory.cs
│   │   │   ├── FsiRegion.cs
│   │   │   ├── StfmDataset.cs
│   │   │   ├── StfmSeries.cs
│   │   │   ├── StfmObservation.cs
│   │   │   ├── HfmDataset.cs
│   │   │   ├── HfmSeries.cs
│   │   │   ├── HfmObservation.cs
│   │   │   └── SeriesMetadata.cs
│   │   ├── Interfaces/
│   │   │   ├── IOfrApiClient.cs
│   │   │   ├── IFsiClient.cs
│   │   │   ├── IStfmClient.cs
│   │   │   ├── IHfmClient.cs
│   │   │   └── IOfrRepository.cs
│   │   └── OfrCollector.Core.csproj
│   ├── OfrCollector.Application/
│   │   ├── Services/
│   │   │   ├── FsiCollectionService.cs
│   │   │   ├── StfmCollectionService.cs
│   │   │   └── HfmCollectionService.cs
│   │   └── OfrCollector.Application.csproj
│   ├── OfrCollector.Infrastructure/
│   │   ├── Api/
│   │   │   ├── OfrApiClientBase.cs
│   │   │   ├── StfmApiClient.cs
│   │   │   ├── HfmApiClient.cs
│   │   │   ├── FsiScraper.cs
│   │   │   └── ResponseModels/
│   │   │       ├── MnemonicsResponse.cs
│   │   │       ├── SeriesMetadataResponse.cs
│   │   │       ├── TimeseriesResponse.cs
│   │   │       ├── DatasetResponse.cs
│   │   │       └── SearchResponse.cs
│   │   ├── Persistence/
│   │   │   ├── OfrRepository.cs
│   │   │   └── OfrDbContext.cs
│   │   ├── Observability/
│   │   │   ├── OfrActivitySource.cs
│   │   │   └── OfrMeter.cs
│   │   └── OfrCollector.Infrastructure.csproj
│   ├── OfrCollector.Grpc/
│   │   ├── Protos/
│   │   │   └── ofr_events.proto
│   │   ├── Services/
│   │   │   └── OfrEventStreamService.cs
│   │   └── OfrCollector.Grpc.csproj
│   └── OfrCollector.Service/
│       ├── Workers/
│       │   ├── FsiCollectionWorker.cs
│       │   ├── StfmCollectionWorker.cs
│       │   └── HfmCollectionWorker.cs
│       ├── Program.cs
│       ├── appsettings.json
│       └── OfrCollector.Service.csproj
├── tests/
│   ├── OfrCollector.UnitTests/
│   └── OfrCollector.IntegrationTests/
├── config/
│   ├── stfm-series.json
│   └── hfm-series.json
├── migrations/
│   └── 001_initial_schema.sql
├── dashboards/
│   └── ofr-collector.json
├── .devcontainer/
│   └── devcontainer.json
└── Containerfile
```

## Domain Models

### FinancialStressIndex.cs

```csharp
namespace OfrCollector.Core.Models;

/// <summary>
/// OFR Financial Stress Index daily observation.
/// Positive values indicate above-average stress.
/// </summary>
public sealed record FinancialStressIndex
{
    public required DateOnly Date { get; init; }
    
    /// <summary>
    /// Composite FSI value. Zero = normal stress levels.
    /// </summary>
    public required decimal Value { get; init; }
    
    // Category contributions (sum to ~Value)
    public decimal? CreditContribution { get; init; }
    public decimal? EquityValuationContribution { get; init; }
    public decimal? FundingContribution { get; init; }
    public decimal? SafeAssetsContribution { get; init; }
    public decimal? VolatilityContribution { get; init; }
    
    // Regional contributions
    public decimal? UsContribution { get; init; }
    public decimal? AdvancedEconomiesContribution { get; init; }
    public decimal? EmergingMarketsContribution { get; init; }
    
    public DateTimeOffset CollectedAt { get; init; } = DateTimeOffset.UtcNow;
    
    // Computed properties
    public bool IsElevated => Value > 0;
    public bool IsHighStress => Value > 2.0m;  // ~2 std dev
    public bool IsCrisisLevel => Value > 5.0m; // GFC peaked ~15
    
    public FsiCategory DominantCategory => GetDominantCategory();
    public FsiRegion DominantRegion => GetDominantRegion();
    
    private FsiCategory GetDominantCategory()
    {
        var max = new[] 
        { 
            (FsiCategory.Credit, CreditContribution ?? 0),
            (FsiCategory.EquityValuation, EquityValuationContribution ?? 0),
            (FsiCategory.Funding, FundingContribution ?? 0),
            (FsiCategory.SafeAssets, SafeAssetsContribution ?? 0),
            (FsiCategory.Volatility, VolatilityContribution ?? 0)
        }.MaxBy(x => Math.Abs(x.Item2));
        
        return max.Item1;
    }
    
    private FsiRegion GetDominantRegion()
    {
        var max = new[]
        {
            (FsiRegion.UnitedStates, UsContribution ?? 0),
            (FsiRegion.AdvancedEconomies, AdvancedEconomiesContribution ?? 0),
            (FsiRegion.EmergingMarkets, EmergingMarketsContribution ?? 0)
        }.MaxBy(x => Math.Abs(x.Item2));
        
        return max.Item1;
    }
}

public enum FsiCategory
{
    Credit,
    EquityValuation,
    Funding,
    SafeAssets,
    Volatility
}

public enum FsiRegion
{
    UnitedStates,
    AdvancedEconomies,
    EmergingMarkets
}
```

### StfmSeries.cs

```csharp
namespace OfrCollector.Core.Models;

/// <summary>
/// Short-term Funding Monitor series metadata.
/// </summary>
public sealed record StfmSeries
{
    public required string Mnemonic { get; init; }
    public required StfmDataset Dataset { get; init; }
    public required string Name { get; init; }
    public string? Description { get; init; }
    public string? Unit { get; init; }
    public string? Frequency { get; init; }
    public string? Vintage { get; init; }  // Preliminary vs Final
    public DateOnly? StartDate { get; init; }
    public DateOnly? EndDate { get; init; }
    public bool IsActive { get; init; } = true;
}

public enum StfmDataset
{
    /// <summary>NY Fed Reference Rates (SOFR, EFFR, OBFR)</summary>
    Fnyr,
    
    /// <summary>U.S. Repo Markets (DVP, GCF rates and volumes)</summary>
    Repo,
    
    /// <summary>U.S. Money Market Funds (N-MFP holdings)</summary>
    Mmf,
    
    /// <summary>Primary Dealer Statistics</summary>
    Nypd,
    
    /// <summary>Treasury Constant Maturity Rates</summary>
    Tyld
}
```

### StfmObservation.cs

```csharp
namespace OfrCollector.Core.Models;

/// <summary>
/// Short-term Funding Monitor observation.
/// </summary>
public sealed record StfmObservation
{
    public long Id { get; init; }
    public required string Mnemonic { get; init; }
    public required DateOnly Date { get; init; }
    public required decimal Value { get; init; }
    public DateTimeOffset CollectedAt { get; init; } = DateTimeOffset.UtcNow;
}
```

### HfmSeries.cs / HfmObservation.cs

```csharp
namespace OfrCollector.Core.Models;

/// <summary>
/// Hedge Fund Monitor series metadata.
/// </summary>
public sealed record HfmSeries
{
    public required string Mnemonic { get; init; }
    public required HfmDataset Dataset { get; init; }
    public required string Name { get; init; }
    public string? Description { get; init; }
    public string? Unit { get; init; }
    public string? Frequency { get; init; }  // Typically quarterly
    public DateOnly? StartDate { get; init; }
    public DateOnly? EndDate { get; init; }
    public bool IsActive { get; init; } = true;
}

public enum HfmDataset
{
    /// <summary>Form PF Aggregates (leverage, liquidity, stress tests)</summary>
    Fpf,
    
    /// <summary>FICC Sponsored Repo</summary>
    Ficc
}

public sealed record HfmObservation
{
    public long Id { get; init; }
    public required string Mnemonic { get; init; }
    public required DateOnly Date { get; init; }
    public required decimal Value { get; init; }
    public DateTimeOffset CollectedAt { get; init; } = DateTimeOffset.UtcNow;
}
```

## API Client Interfaces

### IOfrApiClient.cs

```csharp
namespace OfrCollector.Core.Interfaces;

/// <summary>
/// Base interface for OFR API operations.
/// </summary>
public interface IOfrApiClient
{
    /// <summary>
    /// Get all available series mnemonics.
    /// </summary>
    Task<IReadOnlyList<string>> GetMnemonicsAsync(
        CancellationToken cancellationToken = default);
    
    /// <summary>
    /// Get mnemonics grouped by dataset.
    /// </summary>
    Task<IReadOnlyDictionary<string, IReadOnlyList<SeriesInfo>>> GetMnemonicsByDatasetAsync(
        CancellationToken cancellationToken = default);
    
    /// <summary>
    /// Get metadata for a specific series.
    /// </summary>
    Task<SeriesMetadata?> GetSeriesMetadataAsync(
        string mnemonic,
        CancellationToken cancellationToken = default);
    
    /// <summary>
    /// Search for series matching a query.
    /// </summary>
    Task<IReadOnlyList<SearchResult>> SearchAsync(
        string query,
        CancellationToken cancellationToken = default);
    
    /// <summary>
    /// Get time series data for a single mnemonic.
    /// </summary>
    Task<IReadOnlyList<(DateOnly Date, decimal Value)>> GetTimeSeriesAsync(
        string mnemonic,
        DateOnly? startDate = null,
        DateOnly? endDate = null,
        Periodicity periodicity = Periodicity.Daily,
        CancellationToken cancellationToken = default);
    
    /// <summary>
    /// Get all series data for a dataset.
    /// </summary>
    Task<DatasetResult> GetDatasetAsync(
        string dataset,
        DateOnly? startDate = null,
        DateOnly? endDate = null,
        CancellationToken cancellationToken = default);
    
    /// <summary>
    /// Get multiple series with full metadata.
    /// </summary>
    Task<IReadOnlyDictionary<string, SeriesWithData>> GetMultipleSeriesAsync(
        IEnumerable<string> mnemonics,
        DateOnly? startDate = null,
        DateOnly? endDate = null,
        CancellationToken cancellationToken = default);
    
    /// <summary>
    /// Calculate spread between two series.
    /// </summary>
    Task<IReadOnlyList<(DateOnly Date, decimal Value)>> GetSpreadAsync(
        string mnemonic1,
        string mnemonic2,
        DateOnly? startDate = null,
        DateOnly? endDate = null,
        CancellationToken cancellationToken = default);
}

public enum Periodicity
{
    Daily,
    Weekly,
    Monthly,
    Quarterly,
    Annual
}
```

### IStfmClient.cs

```csharp
namespace OfrCollector.Core.Interfaces;

/// <summary>
/// Short-term Funding Monitor API client.
/// Base URL: https://data.financialresearch.gov/v1/
/// </summary>
public interface IStfmClient : IOfrApiClient
{
    /// <summary>
    /// Get NY Fed reference rates (SOFR, EFFR, OBFR).
    /// </summary>
    Task<DatasetResult> GetReferenceRatesAsync(
        DateOnly? startDate = null,
        DateOnly? endDate = null,
        CancellationToken cancellationToken = default);
    
    /// <summary>
    /// Get repo market data (DVP, GCF rates and volumes).
    /// </summary>
    Task<DatasetResult> GetRepoDataAsync(
        DateOnly? startDate = null,
        DateOnly? endDate = null,
        CancellationToken cancellationToken = default);
    
    /// <summary>
    /// Get money market fund holdings data.
    /// </summary>
    Task<DatasetResult> GetMoneyMarketFundDataAsync(
        DateOnly? startDate = null,
        DateOnly? endDate = null,
        CancellationToken cancellationToken = default);
    
    /// <summary>
    /// Get primary dealer statistics.
    /// </summary>
    Task<DatasetResult> GetPrimaryDealerDataAsync(
        DateOnly? startDate = null,
        DateOnly? endDate = null,
        CancellationToken cancellationToken = default);
    
    /// <summary>
    /// Get Treasury constant maturity rates.
    /// </summary>
    Task<DatasetResult> GetTreasuryYieldsAsync(
        DateOnly? startDate = null,
        DateOnly? endDate = null,
        CancellationToken cancellationToken = default);
}
```

### IHfmClient.cs

```csharp
namespace OfrCollector.Core.Interfaces;

/// <summary>
/// Hedge Fund Monitor API client.
/// Base URL: https://data.financialresearch.gov/hf/v1/
/// </summary>
public interface IHfmClient : IOfrApiClient
{
    /// <summary>
    /// Get Form PF aggregate data (leverage, liquidity, stress tests).
    /// </summary>
    Task<DatasetResult> GetFormPfDataAsync(
        DateOnly? startDate = null,
        DateOnly? endDate = null,
        CancellationToken cancellationToken = default);
    
    /// <summary>
    /// Get FICC sponsored repo data.
    /// </summary>
    Task<DatasetResult> GetFiccRepoDataAsync(
        DateOnly? startDate = null,
        DateOnly? endDate = null,
        CancellationToken cancellationToken = default);
}
```

### IFsiClient.cs

```csharp
namespace OfrCollector.Core.Interfaces;

/// <summary>
/// Financial Stress Index client.
/// Note: FSI does not have a documented JSON API.
/// Uses web scraping or CSV download.
/// </summary>
public interface IFsiClient
{
    /// <summary>
    /// Get the latest FSI value.
    /// </summary>
    Task<FinancialStressIndex?> GetLatestAsync(
        CancellationToken cancellationToken = default);
    
    /// <summary>
    /// Get FSI history.
    /// </summary>
    Task<IReadOnlyList<FinancialStressIndex>> GetHistoryAsync(
        DateOnly? startDate = null,
        DateOnly? endDate = null,
        CancellationToken cancellationToken = default);
    
    /// <summary>
    /// Get FSI with category breakdown.
    /// </summary>
    Task<IReadOnlyList<FinancialStressIndex>> GetWithCategoriesAsync(
        DateOnly? startDate = null,
        DateOnly? endDate = null,
        CancellationToken cancellationToken = default);
    
    /// <summary>
    /// Get FSI with regional breakdown.
    /// </summary>
    Task<IReadOnlyList<FinancialStressIndex>> GetWithRegionsAsync(
        DateOnly? startDate = null,
        DateOnly? endDate = null,
        CancellationToken cancellationToken = default);
}
```

## API Response Models

### TimeseriesResponse.cs

```csharp
namespace OfrCollector.Infrastructure.Api.ResponseModels;

/// <summary>
/// Response from /series/timeseries endpoint.
/// Returns array of [date, value] pairs.
/// </summary>
public sealed class TimeseriesResponse : List<JsonElement[]>
{
    // API returns: [["2024-01-02", 1.23], ["2024-01-03", 1.24], ...]
}
```

### DatasetResponse.cs

```csharp
namespace OfrCollector.Infrastructure.Api.ResponseModels;

/// <summary>
/// Response from /series/dataset endpoint.
/// </summary>
public sealed class DatasetResponse
{
    [JsonPropertyName("short_name")]
    public string? ShortName { get; set; }
    
    [JsonPropertyName("long_name")]
    public string? LongName { get; set; }
    
    [JsonPropertyName("timeseries")]
    public Dictionary<string, SeriesDataResponse>? Timeseries { get; set; }
}

public sealed class SeriesDataResponse
{
    [JsonPropertyName("timeseries")]
    public TimeseriesDataResponse? Timeseries { get; set; }
    
    [JsonPropertyName("metadata")]
    public SeriesMetadataResponse? Metadata { get; set; }
}

public sealed class TimeseriesDataResponse
{
    [JsonPropertyName("aggregation")]
    public List<JsonElement[]>? Aggregation { get; set; }
}
```

### MnemonicsResponse.cs

```csharp
namespace OfrCollector.Infrastructure.Api.ResponseModels;

/// <summary>
/// Response from /metadata/mnemonics endpoint.
/// </summary>
public sealed class MnemonicsResponse
{
    // When called without parameters: string[]
    // When called with output=by_dataset: Dictionary<string, MnemonicInfo[]>
}

public sealed class MnemonicInfo
{
    [JsonPropertyName("mnemonic")]
    public string Mnemonic { get; set; } = string.Empty;
    
    [JsonPropertyName("series_name")]
    public string SeriesName { get; set; } = string.Empty;
}
```

### SeriesMetadataResponse.cs

```csharp
namespace OfrCollector.Infrastructure.Api.ResponseModels;

/// <summary>
/// Response from /metadata/query endpoint.
/// </summary>
public sealed class SeriesMetadataResponse
{
    [JsonPropertyName("mnemonic")]
    public string Mnemonic { get; set; } = string.Empty;
    
    [JsonPropertyName("description")]
    public DescriptionInfo? Description { get; set; }
    
    [JsonPropertyName("release")]
    public ReleaseInfo? Release { get; set; }
}

public sealed class DescriptionInfo
{
    [JsonPropertyName("name")]
    public string? Name { get; set; }
    
    [JsonPropertyName("description")]
    public string? Description { get; set; }
    
    [JsonPropertyName("vintage")]
    public string? Vintage { get; set; }
    
    [JsonPropertyName("vintage_approach")]
    public string? VintageApproach { get; set; }
    
    [JsonPropertyName("subtype")]
    public string? Subtype { get; set; }
    
    [JsonPropertyName("notes")]
    public string? Notes { get; set; }
}

public sealed class ReleaseInfo
{
    [JsonPropertyName("long_name")]
    public string? LongName { get; set; }
    
    [JsonPropertyName("short_name")]
    public string? ShortName { get; set; }
}
```

## Initial Series Configuration

### stfm-series.json

```json
{
  "version": "1.0",
  "description": "ATLAS Short-term Funding Monitor series for NBFI monitoring",
  "datasets": {
    "fnyr": {
      "name": "NY Fed Reference Rates",
      "frequency": "daily",
      "priority": "high",
      "series": [
        {
          "mnemonic": "FNYR-SOFR-A",
          "name": "SOFR (Actual)",
          "atlas_use": "Primary overnight rate, replaces LIBOR"
        },
        {
          "mnemonic": "FNYR-EFFR-A",
          "name": "EFFR (Actual)",
          "atlas_use": "Fed funds target gauge"
        },
        {
          "mnemonic": "FNYR-OBFR-A",
          "name": "OBFR (Actual)",
          "atlas_use": "Overnight bank funding rate"
        }
      ]
    },
    "repo": {
      "name": "U.S. Repo Markets",
      "frequency": "daily",
      "priority": "high",
      "series": [
        {
          "mnemonic": "REPO-DVP_AR_OO-P",
          "name": "DVP Overnight Rate (Preliminary)",
          "atlas_use": "Core repo rate, stress indicator"
        },
        {
          "mnemonic": "REPO-DVP_OV_OO-P",
          "name": "DVP Overnight Volume (Preliminary)",
          "atlas_use": "Repo market liquidity"
        },
        {
          "mnemonic": "REPO-GCF_AR_TRS-P",
          "name": "GCF Treasury Rate (Preliminary)",
          "atlas_use": "GCF collateralized rate"
        }
      ]
    },
    "mmf": {
      "name": "Money Market Funds",
      "frequency": "monthly",
      "priority": "medium",
      "series": [
        {
          "mnemonic": "MMF-GOVT_ASSETS_TOT",
          "name": "Government MMF Total Assets",
          "atlas_use": "Flight-to-quality indicator"
        },
        {
          "mnemonic": "MMF-PRIME_ASSETS_TOT",
          "name": "Prime MMF Total Assets",
          "atlas_use": "Credit risk appetite"
        },
        {
          "mnemonic": "MMF-REPO_ASSETS_TOT",
          "name": "MMF Repo Holdings",
          "atlas_use": "MMF-repo interconnection"
        }
      ]
    },
    "nypd": {
      "name": "Primary Dealer Statistics",
      "frequency": "weekly",
      "priority": "medium",
      "series": [
        {
          "mnemonic": "NYPD-PD_RP_T_TSY_TOT-A",
          "name": "Primary Dealer Treasury Repo",
          "atlas_use": "Dealer funding stress"
        },
        {
          "mnemonic": "NYPD-PD_RRP_T_TSY_TOT-A",
          "name": "Primary Dealer Treasury Reverse Repo",
          "atlas_use": "Dealer lending capacity"
        }
      ]
    },
    "tyld": {
      "name": "Treasury Yields",
      "frequency": "daily",
      "priority": "low",
      "notes": "Available via FRED, included for completeness",
      "series": [
        {
          "mnemonic": "TYLDR-TCMR-3Mo-A",
          "name": "3-Month Treasury CMR"
        },
        {
          "mnemonic": "TYLDR-TCMR-2Yr-A",
          "name": "2-Year Treasury CMR"
        },
        {
          "mnemonic": "TYLDR-TCMR-10Yr-A",
          "name": "10-Year Treasury CMR"
        }
      ]
    }
  }
}
```

### hfm-series.json

```json
{
  "version": "1.0",
  "description": "ATLAS Hedge Fund Monitor series for shadow banking monitoring",
  "datasets": {
    "fpf": {
      "name": "Form PF Aggregates",
      "frequency": "quarterly",
      "priority": "high",
      "series": [
        {
          "mnemonic": "FPF-ALLQHF_LEVERAGE_GAV_P50",
          "name": "Median Gross Leverage (GAV)",
          "atlas_use": "Hedge fund leverage, deleveraging risk"
        },
        {
          "mnemonic": "FPF-ALLQHF_LEVERAGE_NAV_P50",
          "name": "Median Net Leverage (NAV)",
          "atlas_use": "Net exposure indicator"
        },
        {
          "mnemonic": "FPF-ALLQHF_CASHRATIO_P50",
          "name": "Median Unencumbered Cash Ratio",
          "atlas_use": "Hedge fund liquidity buffer"
        },
        {
          "mnemonic": "FPF-ALLQHF_CDSUP250BPS_P5",
          "name": "CDS +250bps Stress Test (5th percentile)",
          "atlas_use": "Tail risk under credit stress"
        },
        {
          "mnemonic": "FPF-ALLQHF_EQDOWN10PCT_P5",
          "name": "Equity -10% Stress Test (5th percentile)",
          "atlas_use": "Tail risk under equity stress"
        },
        {
          "mnemonic": "FPF-STRATEGY_CREDIT_LEVERAGE_GAV_P50",
          "name": "Credit Strategy Median Leverage",
          "atlas_use": "Credit hedge fund leverage"
        }
      ]
    },
    "ficc": {
      "name": "FICC Sponsored Repo",
      "frequency": "daily",
      "priority": "medium",
      "series": [
        {
          "mnemonic": "FICC-SPONSORED_REPO_VOL",
          "name": "Sponsored Repo Volume",
          "atlas_use": "Hedge fund repo financing"
        },
        {
          "mnemonic": "FICC-SPONSORED_REVREPO_VOL",
          "name": "Sponsored Reverse Repo Volume",
          "atlas_use": "Hedge fund cash lending"
        }
      ]
    }
  }
}
```

## Database Schema

### 001_initial_schema.sql

```sql
-- OFR Collector Schema
-- TimescaleDB hypertables for time-series data

-- Financial Stress Index
CREATE TABLE IF NOT EXISTS ofr_fsi (
    date DATE NOT NULL,
    value DECIMAL(10, 6) NOT NULL,
    credit_contribution DECIMAL(10, 6),
    equity_valuation_contribution DECIMAL(10, 6),
    funding_contribution DECIMAL(10, 6),
    safe_assets_contribution DECIMAL(10, 6),
    volatility_contribution DECIMAL(10, 6),
    us_contribution DECIMAL(10, 6),
    advanced_economies_contribution DECIMAL(10, 6),
    emerging_markets_contribution DECIMAL(10, 6),
    collected_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    PRIMARY KEY (date)
);

-- STFM Series Metadata
CREATE TABLE IF NOT EXISTS ofr_stfm_series (
    mnemonic VARCHAR(100) PRIMARY KEY,
    dataset VARCHAR(20) NOT NULL,
    name VARCHAR(500) NOT NULL,
    description TEXT,
    unit VARCHAR(50),
    frequency VARCHAR(20),
    vintage VARCHAR(50),
    start_date DATE,
    end_date DATE,
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_stfm_series_dataset ON ofr_stfm_series(dataset);

-- STFM Observations
CREATE TABLE IF NOT EXISTS ofr_stfm_observations (
    id BIGSERIAL,
    mnemonic VARCHAR(100) NOT NULL REFERENCES ofr_stfm_series(mnemonic),
    date DATE NOT NULL,
    value DECIMAL(20, 8) NOT NULL,
    collected_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    PRIMARY KEY (mnemonic, date)
);

SELECT create_hypertable('ofr_stfm_observations', 'date', 
    chunk_time_interval => INTERVAL '1 month',
    if_not_exists => TRUE);

CREATE INDEX idx_stfm_obs_mnemonic ON ofr_stfm_observations(mnemonic, date DESC);

-- HFM Series Metadata
CREATE TABLE IF NOT EXISTS ofr_hfm_series (
    mnemonic VARCHAR(100) PRIMARY KEY,
    dataset VARCHAR(20) NOT NULL,
    name VARCHAR(500) NOT NULL,
    description TEXT,
    unit VARCHAR(50),
    frequency VARCHAR(20),
    start_date DATE,
    end_date DATE,
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_hfm_series_dataset ON ofr_hfm_series(dataset);

-- HFM Observations
CREATE TABLE IF NOT EXISTS ofr_hfm_observations (
    id BIGSERIAL,
    mnemonic VARCHAR(100) NOT NULL REFERENCES ofr_hfm_series(mnemonic),
    date DATE NOT NULL,
    value DECIMAL(20, 8) NOT NULL,
    collected_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    PRIMARY KEY (mnemonic, date)
);

SELECT create_hypertable('ofr_hfm_observations', 'date', 
    chunk_time_interval => INTERVAL '3 months',
    if_not_exists => TRUE);

CREATE INDEX idx_hfm_obs_mnemonic ON ofr_hfm_observations(mnemonic, date DESC);

-- Collection tracking
CREATE TABLE IF NOT EXISTS ofr_collection_log (
    id BIGSERIAL PRIMARY KEY,
    source VARCHAR(20) NOT NULL,  -- 'fsi', 'stfm', 'hfm'
    dataset VARCHAR(50),
    series_count INT,
    observation_count INT,
    started_at TIMESTAMPTZ NOT NULL,
    completed_at TIMESTAMPTZ,
    status VARCHAR(20) NOT NULL,  -- 'success', 'partial', 'failed'
    error_message TEXT
);

-- Compression policies for older data
SELECT add_compression_policy('ofr_stfm_observations', INTERVAL '6 months');
SELECT add_compression_policy('ofr_hfm_observations', INTERVAL '1 year');

-- Continuous aggregates for dashboards
CREATE MATERIALIZED VIEW ofr_fsi_weekly
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 week', date) AS week,
    AVG(value) AS avg_fsi,
    MAX(value) AS max_fsi,
    MIN(value) AS min_fsi,
    LAST(value, date) AS latest_fsi
FROM ofr_fsi
GROUP BY time_bucket('1 week', date);

SELECT add_continuous_aggregate_policy('ofr_fsi_weekly',
    start_offset => INTERVAL '1 month',
    end_offset => INTERVAL '1 day',
    schedule_interval => INTERVAL '1 day');
```

## Collection Schedule

| Worker | Schedule | Data Lag | Notes |
|--------|----------|----------|-------|
| FSI | Daily 10:00 UTC | T+2 | After OFR publishes (~08:00 ET) |
| STFM (fnyr, repo) | Daily 14:00 UTC | T+1 | Reference rates published ~09:00 ET |
| STFM (mmf) | Monthly, 15th | ~45 days | N-MFP filing lag |
| STFM (nypd) | Weekly, Friday 18:00 UTC | 1 week | Published Thursdays |
| HFM (fpf) | Quarterly, month+45 days | ~45 days | Form PF filing deadline |
| HFM (ficc) | Daily 14:00 UTC | T+1 | Published with repo data |

## Events

### FinancialStressEvent.cs

```csharp
namespace Atlas.Events;

/// <summary>
/// Published when FSI crosses significance thresholds.
/// </summary>
public sealed record FinancialStressEvent
{
    public required DateOnly Date { get; init; }
    public required decimal FsiValue { get; init; }
    public required decimal PreviousFsiValue { get; init; }
    public required FsiAlertLevel AlertLevel { get; init; }
    public required FsiCategory? DominantCategory { get; init; }
    public required FsiRegion? DominantRegion { get; init; }
    public DateTimeOffset Timestamp { get; init; } = DateTimeOffset.UtcNow;
}

public enum FsiAlertLevel
{
    Normal,       // FSI < 0
    Elevated,     // FSI 0-2
    High,         // FSI 2-5
    Crisis        // FSI > 5
}
```

## Observability

### Metrics

```csharp
// OfrMeter.cs
public static class OfrMeter
{
    public static readonly Meter Meter = new("OfrCollector", "1.0.0");
    
    // FSI metrics
    public static readonly Gauge<double> FsiValue = Meter.CreateGauge<double>(
        "ofr.fsi.value", description: "Current OFR Financial Stress Index value");
    
    public static readonly Gauge<double> FsiCategoryContribution = Meter.CreateGauge<double>(
        "ofr.fsi.category_contribution", description: "FSI contribution by category");
    
    // Collection metrics
    public static readonly Counter<long> SeriesCollected = Meter.CreateCounter<long>(
        "ofr.collector.series_collected_total", description: "Series collected");
    
    public static readonly Counter<long> ObservationsCollected = Meter.CreateCounter<long>(
        "ofr.collector.observations_collected_total", description: "Observations collected");
    
    public static readonly Counter<long> CollectionErrors = Meter.CreateCounter<long>(
        "ofr.collector.errors_total", description: "Collection errors");
    
    public static readonly Histogram<double> ApiLatency = Meter.CreateHistogram<double>(
        "ofr.api.latency_seconds", description: "OFR API request latency");
}
```

### Grafana Dashboard Panels

1. **FSI Overview**: Current value, 7-day trend, category breakdown
2. **Repo Market Health**: DVP/GCF rates vs SOFR, volume trends
3. **MMF Flows**: Government vs Prime assets, repo holdings
4. **Hedge Fund Leverage**: Median GAV/NAV leverage, stress test results
5. **Collection Health**: Success rates, latency, error counts

## ThresholdEngine Integration

### FSI Patterns

```json
{
  "patterns": [
    {
      "id": "ofr-fsi-elevated",
      "name": "OFR FSI Elevated Stress",
      "description": "Financial stress above historical average",
      "category": "NBFI",
      "expression": "GetLatest('OFR_FSI') > 0",
      "signal_strength": "Math.Min(GetLatest('OFR_FSI') / 5.0, 1.0)",
      "weight": 1.0
    },
    {
      "id": "ofr-fsi-high-stress",
      "name": "OFR FSI High Stress",
      "description": "Financial stress exceeds 2 standard deviations",
      "category": "NBFI",
      "expression": "GetLatest('OFR_FSI') > 2.0",
      "signal_strength": 1.0,
      "weight": 1.5
    },
    {
      "id": "ofr-fsi-crisis",
      "name": "OFR FSI Crisis Level",
      "description": "Financial stress at crisis levels (GFC peaked ~15)",
      "category": "NBFI",
      "expression": "GetLatest('OFR_FSI') > 5.0",
      "signal_strength": 1.0,
      "weight": 2.0
    },
    {
      "id": "ofr-fsi-funding-stress",
      "name": "OFR FSI Funding Category Stress",
      "description": "Funding component driving FSI elevation",
      "category": "Liquidity",
      "expression": "GetLatest('OFR_FSI') > 0 && GetLatest('OFR_FSI_FUNDING') > GetLatest('OFR_FSI') * 0.3",
      "signal_strength": 1.0,
      "weight": 1.2
    },
    {
      "id": "ofr-hf-leverage-elevated",
      "name": "Hedge Fund Leverage Elevated",
      "description": "Median hedge fund gross leverage above historical norms",
      "category": "NBFI",
      "expression": "GetLatest('FPF-ALLQHF_LEVERAGE_GAV_P50') > GetMA('FPF-ALLQHF_LEVERAGE_GAV_P50', 8) * 1.1",
      "signal_strength": 0.7,
      "weight": 0.8
    },
    {
      "id": "ofr-repo-stress",
      "name": "Repo Rate Stress",
      "description": "DVP repo rate elevated vs SOFR",
      "category": "Liquidity",
      "expression": "GetLatest('REPO-DVP_AR_OO-P') - GetLatest('FNYR-SOFR-A') > 0.10",
      "signal_strength": "Math.Min((GetLatest('REPO-DVP_AR_OO-P') - GetLatest('FNYR-SOFR-A')) / 0.50, 1.0)",
      "weight": 1.0
    }
  ]
}
```

## Tasks

| # | Task | Est | Status |
|---|------|-----|--------|
| 1 | Create solution structure (4 projects) | 30m | ⬜ |
| 2 | Implement Core models (FSI, STFM, HFM) | 1h | ⬜ |
| 3 | Implement API response models | 45m | ⬜ |
| 4 | Implement OfrApiClientBase with HttpClient | 1h | ⬜ |
| 5 | Implement StfmApiClient | 1h | ⬜ |
| 6 | Implement HfmApiClient | 45m | ⬜ |
| 7 | Implement FsiScraper (CSV download or web scrape) | 1.5h | ⬜ |
| 8 | Implement OfrRepository (EF Core + TimescaleDB) | 1.5h | ⬜ |
| 9 | Implement FsiCollectionWorker | 1h | ⬜ |
| 10 | Implement StfmCollectionWorker | 1h | ⬜ |
| 11 | Implement HfmCollectionWorker | 45m | ⬜ |
| 12 | Add OpenTelemetry instrumentation | 1h | ⬜ |
| 13 | Create database migrations | 30m | ⬜ |
| 14 | Create series configuration files | 30m | ⬜ |
| 15 | Implement gRPC event streaming | 1h | ⬜ |
| 16 | Create Grafana dashboard | 1h | ⬜ |
| 17 | Unit tests: API clients | 1.5h | ⬜ |
| 18 | Unit tests: Repository | 1h | ⬜ |
| 19 | Integration tests: Full collection flow | 1.5h | ⬜ |
| 20 | Create Containerfile and compose.yaml | 30m | ⬜ |
| 21 | Add ThresholdEngine patterns | 30m | ⬜ |
| 22 | Documentation: API notes and edge cases | 30m | ⬜ |

**Total Estimate**: ~18-20 hours

## Success Criteria

- [ ] FSI collected daily with category/region breakdown
- [ ] STFM series collected per schedule (daily/weekly/monthly)
- [ ] HFM series collected quarterly
- [ ] All observations stored in TimescaleDB with proper hypertables
- [ ] FinancialStressEvent published when FSI crosses thresholds
- [ ] Grafana dashboard shows FSI trends and repo market health
- [ ] ThresholdEngine patterns evaluate against OFR data
- [ ] 95%+ collection success rate over 30 days
- [ ] API latency <2s p99

## Risks & Mitigations

| Risk | Probability | Impact | Mitigation |
|------|-------------|--------|------------|
| FSI has no JSON API | **Confirmed** | Medium | CSV download or FRED STLFSI2 fallback |
| API structure changes | Low | Medium | Versioned clients, good error handling |
| Rate limits undocumented | Low | Low | Conservative polling, exponential backoff |
| Data lag varies by series | Medium | Low | Track publication schedules, alert on staleness |
| Form PF quarterly lag | **Confirmed** | Low | Accept 45-day lag, use for trend analysis not real-time |

## Future Enhancements

- **Phase 2**: Bank Systemic Risk Monitor (G-SIB scores, Contagion Index)
- **Phase 3**: Financial System Vulnerabilities Monitor (58 indicators, currently archived)
- **Phase 4**: Real-time FSI alerts via ntfy.sh when crossing thresholds
- **Phase 5**: Cross-correlation analysis: FSI vs VIX, credit spreads, FRED indicators
