# E4: FinnhubCollector

## Goal
Ingest Finnhub market data, economic calendar events, sentiment data, and analyst information into ATLAS, providing high-volume free API access (60 calls/min = 86,400/day) for real-time quotes, economic indicators, and alternative data that complements the rate-limited Alpha Vantage collector.

## Context

Finnhub offers the most generous free tier among financial APIs:
- **60 API calls per minute** (vs Alpha Vantage's 25/day) - 3,456x more capacity
- Real-time US equity quotes (including VXX as VIX proxy)
- Economic calendar (Fed meetings, CPI, employment reports)
- Earnings and IPO calendars
- Triple sentiment sources (news, social, insider)
- Analyst recommendations and price targets
- OHLCV candle data for stocks, forex, crypto

**Strategic Value for ATLAS:**
- VXX real-time quotes enable VIX deployment triggers without expensive VIX data feeds
- Economic calendar provides forward-looking event dates for regime anticipation
- Triple sentiment (news + social + insider) adds multi-source alternative signals
- Earnings calendar for volatility prediction around earnings
- High rate limits allow frequent polling for time-sensitive indicators

## Architecture

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                             FinnhubCollector                                     │
│  ┌───────────┐  ┌────────────┐  ┌─────────────┐  ┌───────────┐  ┌───────────┐  │
│  │ Quote     │  │ Calendar   │  │ Sentiment   │  │ Analyst   │  │  gRPC     │  │
│  │ Worker    │  │ Worker     │  │ Worker      │  │ Worker    │  │  Server   │  │
│  └─────┬─────┘  └─────┬──────┘  └──────┬──────┘  └─────┬─────┘  └─────┬─────┘  │
│        │              │                │               │              │         │
│        ▼              ▼                ▼               ▼              │         │
│  ┌─────────────────────────────────────────────────────────────────────────┐   │
│  │                       Finnhub API Client                                 │   │
│  │               (Rate Limiter: 60/min token bucket)                        │   │
│  │      18 endpoints: quotes, candles, calendars, sentiment, analyst        │   │
│  └─────────────────────────────────────────────────────────────────────────┘   │
│        │              │                │               │              │         │
│        ▼              ▼                ▼               ▼              │         │
│  ┌─────────────────────────────────────────────────────────────────────────┐   │
│  │                      TimescaleDB Repository                              │   │
│  │  12 tables: quotes, candles, economic_calendar, earnings_calendar,       │   │
│  │  ipo_calendar, news_sentiment, social_sentiment, insider_sentiment,      │   │
│  │  recommendations, price_targets, company_profiles, series                │   │
│  └─────────────────────────────────────────────────────────────────────────┘   │
│        │                                                             │         │
│        └─────────────────────────────────────────────────────────────┘         │
│                                   │                                             │
│  ┌─────────────────────────────────────────────────────────────────────────┐   │
│  │                    OpenTelemetry Instrumentation                         │   │
│  │         Traces → Tempo │ Metrics → Prometheus │ Logs → Loki             │   │
│  └─────────────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────────┘
                              │
                              ▼ ObservationCollectedEvent / CalendarEvent
                       ThresholdEngine
```

## Prerequisites

- E1 complete (shared Events project with ObservationCollectedEvent)
- Finnhub API key (free registration at finnhub.io)
- Understanding of FredCollector/AlphaVantageCollector observability patterns
- TimescaleDB running with OpenTelemetry collector configured

## Project Structure

```
FinnhubCollector/
├── src/
│   ├── FinnhubCollector.Core/
│   │   ├── Models/
│   │   │   ├── SeriesType.cs
│   │   │   ├── Quote.cs
│   │   │   ├── Candle.cs
│   │   │   ├── EconomicCalendarEvent.cs
│   │   │   ├── EarningsCalendarEvent.cs
│   │   │   ├── IpoCalendarEvent.cs
│   │   │   ├── NewsSentiment.cs
│   │   │   ├── SocialSentiment.cs
│   │   │   ├── InsiderSentiment.cs
│   │   │   ├── Recommendation.cs
│   │   │   ├── PriceTarget.cs
│   │   │   ├── CompanyProfile.cs
│   │   │   ├── MarketStatus.cs
│   │   │   ├── SymbolSearchResult.cs
│   │   │   ├── FinnhubSeries.cs
│   │   │   └── FinnhubObservation.cs
│   │   ├── Interfaces/
│   │   │   ├── IFinnhubApiClient.cs
│   │   │   ├── IFinnhubRepository.cs
│   │   │   └── IRateLimiter.cs
│   │   └── FinnhubCollector.Core.csproj
│   ├── FinnhubCollector.Application/
│   │   ├── Services/
│   │   │   ├── QuoteCollectionService.cs
│   │   │   ├── CalendarCollectionService.cs
│   │   │   ├── SentimentCollectionService.cs
│   │   │   └── AnalystDataCollectionService.cs
│   │   └── FinnhubCollector.Application.csproj
│   ├── FinnhubCollector.Infrastructure/
│   │   ├── Api/
│   │   │   ├── FinnhubApiClient.cs
│   │   │   ├── FinnhubApiClientOptions.cs
│   │   │   └── ResponseModels/
│   │   │       ├── QuoteResponse.cs
│   │   │       ├── CandleResponse.cs
│   │   │       ├── EconomicCalendarResponse.cs
│   │   │       ├── EarningsCalendarResponse.cs
│   │   │       ├── NewsSentimentResponse.cs
│   │   │       ├── SocialSentimentResponse.cs
│   │   │       ├── InsiderSentimentResponse.cs
│   │   │       ├── RecommendationResponse.cs
│   │   │       └── PriceTargetResponse.cs
│   │   ├── Persistence/
│   │   │   └── FinnhubRepository.cs
│   │   ├── TokenBucketRateLimiter.cs
│   │   ├── Observability/
│   │   │   ├── FinnhubActivitySource.cs
│   │   │   └── FinnhubMeter.cs
│   │   └── FinnhubCollector.Infrastructure.csproj
│   ├── FinnhubCollector.Grpc/
│   │   ├── Services/
│   │   │   └── ObservationEventStreamService.cs
│   │   └── FinnhubCollector.Grpc.csproj
│   └── FinnhubCollector.Service/
│       ├── Workers/
│       │   ├── QuoteCollectionWorker.cs
│       │   ├── CalendarCollectionWorker.cs
│       │   ├── SentimentCollectionWorker.cs
│       │   └── AnalystDataCollectionWorker.cs
│       ├── Program.cs
│       ├── appsettings.json
│       └── FinnhubCollector.Service.csproj
├── tests/
│   ├── FinnhubCollector.UnitTests/
│   └── FinnhubCollector.IntegrationTests/
├── config/
│   └── series.json
├── migrations/
│   └── 001_initial_schema.sql
├── dashboards/
│   └── finnhub-collector.json
├── .devcontainer/
│   └── devcontainer.json
└── Containerfile
```

## Finnhub API Reference

### Rate Limits
- **Free tier:** 60 API calls/minute, 30 calls/second burst
- **Premium:** Up to 900 calls/minute
- Rate limit headers: `X-Ratelimit-Limit`, `X-Ratelimit-Remaining`, `X-Ratelimit-Reset`

### Endpoints Exposed (18 methods)

| Category | Endpoint | Method | Purpose |
|----------|----------|--------|---------|
| **Quotes** | `/quote` | GetQuoteAsync | Real-time stock price |
| **Candles** | `/stock/candle` | GetStockCandlesAsync | Stock OHLCV history |
| | `/forex/candle` | GetForexCandlesAsync | Forex OHLCV history |
| | `/crypto/candle` | GetCryptoCandlesAsync | Crypto OHLCV history |
| **Calendars** | `/calendar/economic` | GetEconomicCalendarAsync | Fed, CPI, Jobs events |
| | `/calendar/earnings` | GetEarningsCalendarAsync | Earnings events |
| | `/calendar/ipo` | GetIpoCalendarAsync | IPO events |
| **Sentiment** | `/news-sentiment` | GetNewsSentimentAsync | News sentiment |
| | `/stock/social-sentiment` | GetSocialSentimentAsync | Reddit/Twitter sentiment |
| | `/stock/insider-sentiment` | GetInsiderSentimentAsync | Insider MSPR |
| **Analyst** | `/stock/recommendation` | GetRecommendationTrendsAsync | Analyst ratings |
| | `/stock/price-target` | GetPriceTargetAsync | Price targets |
| **Company** | `/stock/profile2` | GetCompanyProfileAsync | Company info |
| | `/stock/peers` | GetCompanyPeersAsync | Similar companies |
| **Market** | `/stock/market-status` | GetMarketStatusAsync | Exchange status |
| | | IsMarketOpenAsync | Quick open check |
| | `/search` | SearchSymbolsAsync | Symbol search |
| | `/stock/symbol` | GetStockSymbolsAsync | Exchange symbols |

### Response Examples

**Quote Response:**
```json
{
  "c": 23.45,      // Current price
  "d": 0.23,       // Change
  "dp": 0.99,      // Percent change
  "h": 23.89,      // High
  "l": 23.01,      // Low
  "o": 23.12,      // Open
  "pc": 23.22,     // Previous close
  "t": 1732780800  // Timestamp (Unix epoch)
}
```

**Economic Calendar Response:**
```json
{
  "economicCalendar": [{
    "country": "US",
    "event": "FOMC Interest Rate Decision",
    "impact": "high",
    "time": "2025-12-18 14:00:00",
    "actual": null,
    "estimate": "4.25%",
    "prev": "4.50%",
    "unit": "%"
  }]
}
```

**Insider Sentiment Response:**
```json
{
  "data": [{
    "symbol": "AAPL",
    "year": 2025,
    "month": 11,
    "change": 15000,
    "mspr": 45.5  // Monthly Share Purchase Ratio: -100 to +100
  }]
}
```

---

## Tasks

### T4.1: Create Solution Structure
**Estimate:** 30 min

```bash
cd ~/ATLAS
mkdir -p FinnhubCollector/src FinnhubCollector/tests FinnhubCollector/config \
         FinnhubCollector/migrations FinnhubCollector/dashboards

cd FinnhubCollector
dotnet new sln -n FinnhubCollector

# Create projects following established pattern
dotnet new classlib -n FinnhubCollector.Core -o src/FinnhubCollector.Core -f net9.0
dotnet new classlib -n FinnhubCollector.Application -o src/FinnhubCollector.Application -f net9.0
dotnet new classlib -n FinnhubCollector.Infrastructure -o src/FinnhubCollector.Infrastructure -f net9.0
dotnet new classlib -n FinnhubCollector.Grpc -o src/FinnhubCollector.Grpc -f net9.0
dotnet new worker -n FinnhubCollector.Service -o src/FinnhubCollector.Service -f net9.0
dotnet new xunit -n FinnhubCollector.UnitTests -o tests/FinnhubCollector.UnitTests -f net9.0
dotnet new xunit -n FinnhubCollector.IntegrationTests -o tests/FinnhubCollector.IntegrationTests -f net9.0

# Add to solution
dotnet sln add src/FinnhubCollector.Core
dotnet sln add src/FinnhubCollector.Application
dotnet sln add src/FinnhubCollector.Infrastructure
dotnet sln add src/FinnhubCollector.Grpc
dotnet sln add src/FinnhubCollector.Service
dotnet sln add tests/FinnhubCollector.UnitTests
dotnet sln add tests/FinnhubCollector.IntegrationTests

# Project references (standard layered architecture)
cd src/FinnhubCollector.Application && dotnet add reference ../FinnhubCollector.Core
cd ../FinnhubCollector.Infrastructure && dotnet add reference ../FinnhubCollector.Core ../FinnhubCollector.Application
cd ../FinnhubCollector.Grpc && dotnet add reference ../FinnhubCollector.Core ../../../../Events/src/Events
cd ../FinnhubCollector.Service && dotnet add reference ../FinnhubCollector.Application ../FinnhubCollector.Infrastructure ../FinnhubCollector.Grpc
```

**Acceptance Criteria:**
- [ ] Solution builds with `dotnet build`
- [ ] All project references resolve correctly
- [ ] Test projects can reference main projects

---

### T4.2: Implement Core Domain Models (Full API Fidelity)
**Estimate:** 1.5 hours

Create 13 domain models + enums to expose full Finnhub API fidelity:

**Core/Models/SeriesType.cs:**
```csharp
namespace FinnhubCollector.Core.Models;

/// <summary>
/// Types of data series collected from Finnhub API.
/// </summary>
public enum SeriesType
{
    Quote,              // Real-time quotes
    Candle,             // OHLCV candle data
    EconomicCalendar,   // Fed, CPI, Jobs
    EarningsCalendar,   // Earnings events
    IpoCalendar,        // IPO events
    NewsSentiment,      // News sentiment
    SocialSentiment,    // Reddit/Twitter
    InsiderSentiment,   // Insider MSPR
    Recommendation,     // Analyst ratings
    PriceTarget,        // Price targets
    TechnicalIndicator, // Technical indicators
    CompanyProfile      // Company data
}
```

**Core/Models/Quote.cs:**
```csharp
namespace FinnhubCollector.Core.Models;

/// <summary>
/// Real-time quote from Finnhub /quote endpoint.
/// </summary>
public class Quote
{
    public long Id { get; set; }
    public required string Symbol { get; set; }
    public decimal CurrentPrice { get; set; }      // c
    public decimal Change { get; set; }            // d
    public decimal PercentChange { get; set; }     // dp
    public decimal High { get; set; }              // h
    public decimal Low { get; set; }               // l
    public decimal Open { get; set; }              // o
    public decimal PreviousClose { get; set; }     // pc
    public DateTime Timestamp { get; set; }        // t (Unix epoch converted)
    public DateTime CollectedAt { get; set; }
    
    public int? SeriesConfigId { get; set; }
    public FinnhubSeries? SeriesConfig { get; set; }
}
```

**Core/Models/Candle.cs:**
```csharp
namespace FinnhubCollector.Core.Models;

/// <summary>
/// OHLCV candle data from /stock/candle, /forex/candle, /crypto/candle.
/// </summary>
public class Candle
{
    public long Id { get; set; }
    public required string SeriesId { get; set; }
    public required string Symbol { get; set; }
    public required string Resolution { get; set; }  // 1, 5, 15, 30, 60, D, W, M
    public required DateOnly Date { get; set; }
    public required DateTime Timestamp { get; set; }
    public decimal Open { get; set; }
    public decimal High { get; set; }
    public decimal Low { get; set; }
    public decimal Close { get; set; }
    public long Volume { get; set; }
    public DateTime CollectedAt { get; set; }
    
    public int? SeriesConfigId { get; set; }
    public FinnhubSeries? SeriesConfig { get; set; }
}
```

**Core/Models/EconomicCalendarEvent.cs:**
```csharp
namespace FinnhubCollector.Core.Models;

/// <summary>
/// Economic calendar event from /calendar/economic.
/// </summary>
public class EconomicCalendarEvent
{
    public long Id { get; set; }
    public required string Country { get; set; }
    public required string Event { get; set; }
    public required string Impact { get; set; }  // high, medium, low
    public required DateTime EventTime { get; set; }
    public string? Actual { get; set; }
    public string? Estimate { get; set; }
    public string? Previous { get; set; }
    public string? Unit { get; set; }
    public DateTime CollectedAt { get; set; }
    
    // Computed
    public bool IsHighImpact => Impact.Equals("high", StringComparison.OrdinalIgnoreCase);
    public bool IsFedEvent => Event.Contains("FOMC", StringComparison.OrdinalIgnoreCase) ||
                              Event.Contains("Fed", StringComparison.OrdinalIgnoreCase);
    public bool IsCpiEvent => Event.Contains("CPI", StringComparison.OrdinalIgnoreCase);
    public bool IsEmploymentEvent => Event.Contains("Nonfarm", StringComparison.OrdinalIgnoreCase) ||
                                     Event.Contains("Unemployment", StringComparison.OrdinalIgnoreCase);
}
```

**Core/Models/EarningsCalendarEvent.cs:**
```csharp
namespace FinnhubCollector.Core.Models;

/// <summary>
/// Earnings calendar event from /calendar/earnings.
/// </summary>
public class EarningsCalendarEvent
{
    public long Id { get; set; }
    public required string Symbol { get; set; }
    public required DateOnly Date { get; set; }
    public string? Hour { get; set; }  // bmo, amc, dmh
    public int? Quarter { get; set; }
    public int? Year { get; set; }
    public decimal? EpsEstimate { get; set; }
    public decimal? EpsActual { get; set; }
    public decimal? RevenueEstimate { get; set; }
    public decimal? RevenueActual { get; set; }
    public DateTime CollectedAt { get; set; }
    
    // Computed
    public decimal? EpsSurprise => EpsActual.HasValue && EpsEstimate.HasValue 
        ? EpsActual.Value - EpsEstimate.Value : null;
    public decimal? EpsSurprisePercent => EpsActual.HasValue && EpsEstimate.HasValue && EpsEstimate.Value != 0
        ? (EpsActual.Value - EpsEstimate.Value) / Math.Abs(EpsEstimate.Value) * 100 : null;
}
```

**Core/Models/IpoCalendarEvent.cs:**
```csharp
namespace FinnhubCollector.Core.Models;

/// <summary>
/// IPO calendar event from /calendar/ipo.
/// </summary>
public class IpoCalendarEvent
{
    public long Id { get; set; }
    public required string Symbol { get; set; }
    public required string Name { get; set; }
    public required DateOnly Date { get; set; }
    public string? Exchange { get; set; }
    public decimal? Price { get; set; }
    public long? NumberOfShares { get; set; }
    public decimal? TotalSharesValue { get; set; }
    public string? Status { get; set; }  // expected, priced, withdrawn
    public DateTime CollectedAt { get; set; }
}
```

**Core/Models/NewsSentiment.cs:**
```csharp
namespace FinnhubCollector.Core.Models;

/// <summary>
/// News sentiment from /news-sentiment.
/// </summary>
public class NewsSentiment
{
    public long Id { get; set; }
    public required string Symbol { get; set; }
    public int ArticlesInLastWeek { get; set; }
    public decimal Buzz { get; set; }
    public decimal WeeklyAverage { get; set; }
    public decimal CompanyNewsScore { get; set; }
    public decimal SectorAverageBullishPercent { get; set; }
    public decimal SectorAverageNewsScore { get; set; }
    public decimal BullishPercent { get; set; }
    public decimal BearishPercent { get; set; }
    public DateTime CollectedAt { get; set; }
    
    public decimal NetSentiment => BullishPercent - BearishPercent;
}
```

**Core/Models/SocialSentiment.cs:**
```csharp
namespace FinnhubCollector.Core.Models;

/// <summary>
/// Social sentiment from /stock/social-sentiment (Reddit + Twitter).
/// </summary>
public class SocialSentiment
{
    public long Id { get; set; }
    public required string Symbol { get; set; }
    public required DateOnly Date { get; set; }
    public int RedditMentions { get; set; }
    public int RedditPositiveMentions { get; set; }
    public int RedditNegativeMentions { get; set; }
    public decimal RedditPositiveScore { get; set; }
    public decimal RedditNegativeScore { get; set; }
    public int TwitterMentions { get; set; }
    public int TwitterPositiveMentions { get; set; }
    public int TwitterNegativeMentions { get; set; }
    public decimal TwitterPositiveScore { get; set; }
    public decimal TwitterNegativeScore { get; set; }
    public DateTime CollectedAt { get; set; }
    
    public int TotalMentions => RedditMentions + TwitterMentions;
    public decimal AveragePositiveScore => (RedditPositiveScore + TwitterPositiveScore) / 2;
    public decimal AverageNegativeScore => (RedditNegativeScore + TwitterNegativeScore) / 2;
}
```

**Core/Models/InsiderSentiment.cs:**
```csharp
namespace FinnhubCollector.Core.Models;

/// <summary>
/// Insider sentiment from /stock/insider-sentiment.
/// MSPR (Monthly Share Purchase Ratio): -100 to +100.
/// </summary>
public class InsiderSentiment
{
    public long Id { get; set; }
    public required string Symbol { get; set; }
    public int Year { get; set; }
    public int Month { get; set; }
    public long Change { get; set; }
    public decimal Mspr { get; set; }  // -100 to +100
    public DateTime CollectedAt { get; set; }
    
    public bool IsNetBuying => Mspr > 0;
    public bool IsStrongBuying => Mspr > 50;
    public bool IsStrongSelling => Mspr < -50;
}
```

**Core/Models/Recommendation.cs:**
```csharp
namespace FinnhubCollector.Core.Models;

/// <summary>
/// Analyst recommendations from /stock/recommendation.
/// </summary>
public class Recommendation
{
    public long Id { get; set; }
    public required string Symbol { get; set; }
    public required DateOnly Period { get; set; }
    public int StrongBuy { get; set; }
    public int Buy { get; set; }
    public int Hold { get; set; }
    public int Sell { get; set; }
    public int StrongSell { get; set; }
    public DateTime CollectedAt { get; set; }
    
    public int TotalAnalysts => StrongBuy + Buy + Hold + Sell + StrongSell;
    public int BullishCount => StrongBuy + Buy;
    public int BearishCount => Sell + StrongSell;
    public decimal BullishPercent => TotalAnalysts > 0 ? (decimal)BullishCount / TotalAnalysts : 0;
}
```

**Core/Models/PriceTarget.cs:**
```csharp
namespace FinnhubCollector.Core.Models;

/// <summary>
/// Price target consensus from /stock/price-target.
/// </summary>
public class PriceTarget
{
    public long Id { get; set; }
    public required string Symbol { get; set; }
    public DateOnly LastUpdated { get; set; }
    public decimal TargetHigh { get; set; }
    public decimal TargetLow { get; set; }
    public decimal TargetMean { get; set; }
    public decimal TargetMedian { get; set; }
    public DateTime CollectedAt { get; set; }
    
    public decimal TargetRange => TargetHigh - TargetLow;
    public decimal GetUpsidePercent(decimal currentPrice) =>
        currentPrice > 0 ? (TargetMean - currentPrice) / currentPrice * 100 : 0;
}
```

**Core/Models/CompanyProfile.cs:**
```csharp
namespace FinnhubCollector.Core.Models;

/// <summary>
/// Company profile from /stock/profile2.
/// </summary>
public class CompanyProfile
{
    public long Id { get; set; }
    public required string Symbol { get; set; }
    public string? Name { get; set; }
    public string? Country { get; set; }
    public string? Currency { get; set; }
    public string? Exchange { get; set; }
    public DateOnly? Ipo { get; set; }
    public decimal? MarketCapitalization { get; set; }
    public string? FinnhubIndustry { get; set; }
    public string? Logo { get; set; }
    public string? Phone { get; set; }
    public decimal? ShareOutstanding { get; set; }
    public string? WebUrl { get; set; }
    public DateTime CollectedAt { get; set; }
}
```

**Core/Models/MarketStatus.cs & SymbolSearchResult.cs:**
```csharp
namespace FinnhubCollector.Core.Models;

public record MarketStatus(
    string Exchange,
    bool IsOpen,
    string? Session,
    string? Holiday,
    string? Timezone);

public record SymbolSearchResult(
    string Symbol,
    string Description,
    string? DisplaySymbol,
    string? Type);
```

**Core/Models/FinnhubSeries.cs:**
```csharp
namespace FinnhubCollector.Core.Models;

/// <summary>
/// Configuration for a Finnhub data series to collect.
/// </summary>
public class FinnhubSeries
{
    public int Id { get; set; }
    public required string SeriesId { get; set; }
    public required string Symbol { get; set; }
    public required string Title { get; set; }
    public required string Category { get; set; }
    public required SeriesType Type { get; set; }
    public string? Resolution { get; set; }
    public bool IsActive { get; set; } = true;
    public DateTime? LastCollectedAt { get; set; }
    public int PollIntervalSeconds { get; set; } = 60;
    public int Priority { get; set; } = 10;
    
    public ICollection<Quote> Quotes { get; set; } = [];
    public ICollection<Candle> Candles { get; set; } = [];
}
```

**Core/Models/FinnhubObservation.cs:**
```csharp
namespace FinnhubCollector.Core.Models;

/// <summary>
/// Standard observation format for gRPC streaming.
/// </summary>
public sealed record FinnhubObservation
{
    public required string SeriesId { get; init; }
    public required DateOnly Date { get; init; }
    public required decimal? Value { get; init; }
    public required DateTime CollectedAt { get; init; }
    public Dictionary<string, object>? Metadata { get; init; }
}
```

**Acceptance Criteria:**
- [ ] All 13 models compile without warnings
- [ ] XML documentation on all public members
- [ ] Computed properties for derived values
- [ ] Models match Finnhub API response structure

---

### T4.3: Implement Core Interfaces (Full API Fidelity)
**Estimate:** 45 min

**Core/Interfaces/IFinnhubApiClient.cs:**
```csharp
namespace FinnhubCollector.Core.Interfaces;

using FinnhubCollector.Core.Models;

/// <summary>
/// Comprehensive Finnhub API client exposing full API fidelity.
/// Rate limit: 60 calls/minute on free tier.
/// </summary>
public interface IFinnhubApiClient
{
    RateLimitStatus RateLimitStatus { get; }
    
    // Quotes & Candles
    Task<Quote?> GetQuoteAsync(string symbol, CancellationToken ct = default);
    Task<IReadOnlyList<Candle>> GetStockCandlesAsync(string symbol, string resolution, long from, long to, CancellationToken ct = default);
    Task<IReadOnlyList<Candle>> GetForexCandlesAsync(string symbol, string resolution, long from, long to, CancellationToken ct = default);
    Task<IReadOnlyList<Candle>> GetCryptoCandlesAsync(string symbol, string resolution, long from, long to, CancellationToken ct = default);
    
    // Calendars
    Task<IReadOnlyList<EconomicCalendarEvent>> GetEconomicCalendarAsync(DateOnly from, DateOnly to, CancellationToken ct = default);
    Task<IReadOnlyList<EarningsCalendarEvent>> GetEarningsCalendarAsync(DateOnly? from = null, DateOnly? to = null, string? symbol = null, bool international = false, CancellationToken ct = default);
    Task<IReadOnlyList<IpoCalendarEvent>> GetIpoCalendarAsync(DateOnly from, DateOnly to, CancellationToken ct = default);
    
    // Sentiment
    Task<NewsSentiment?> GetNewsSentimentAsync(string symbol, CancellationToken ct = default);
    Task<IReadOnlyList<SocialSentiment>> GetSocialSentimentAsync(string symbol, DateOnly from, DateOnly to, CancellationToken ct = default);
    Task<IReadOnlyList<InsiderSentiment>> GetInsiderSentimentAsync(string symbol, DateOnly from, DateOnly to, CancellationToken ct = default);
    
    // Analyst Data
    Task<IReadOnlyList<Recommendation>> GetRecommendationTrendsAsync(string symbol, CancellationToken ct = default);
    Task<PriceTarget?> GetPriceTargetAsync(string symbol, CancellationToken ct = default);
    
    // Company Data
    Task<CompanyProfile?> GetCompanyProfileAsync(string symbol, CancellationToken ct = default);
    Task<IReadOnlyList<string>> GetCompanyPeersAsync(string symbol, CancellationToken ct = default);
    
    // Market Data
    Task<MarketStatus?> GetMarketStatusAsync(string exchange = "US", CancellationToken ct = default);
    Task<bool> IsMarketOpenAsync(CancellationToken ct = default);
    Task<IReadOnlyList<SymbolSearchResult>> SearchSymbolsAsync(string query, CancellationToken ct = default);
    Task<IReadOnlyList<SymbolSearchResult>> GetStockSymbolsAsync(string exchange = "US", CancellationToken ct = default);
}

public sealed record RateLimitStatus
{
    public required int Limit { get; init; }
    public required int Remaining { get; init; }
    public required DateTime ResetAt { get; init; }
    
    public bool IsNearLimit => Remaining < 10;
    public bool IsExhausted => Remaining <= 0;
    public double UsagePercent => Limit > 0 ? (double)(Limit - Remaining) / Limit * 100 : 0;
}
```

**Core/Interfaces/IFinnhubRepository.cs:**
```csharp
namespace FinnhubCollector.Core.Interfaces;

using FinnhubCollector.Core.Models;

public interface IFinnhubRepository
{
    // Series Management
    Task<IReadOnlyList<FinnhubSeries>> GetActiveSeriesAsync(SeriesType? type = null, CancellationToken ct = default);
    Task<FinnhubSeries?> GetSeriesAsync(string seriesId, CancellationToken ct = default);
    Task UpdateLastCollectedAsync(string seriesId, DateTime collectedAt, CancellationToken ct = default);
    
    // Quotes
    Task UpsertQuoteAsync(Quote quote, CancellationToken ct = default);
    Task UpsertQuotesAsync(IEnumerable<Quote> quotes, CancellationToken ct = default);
    Task<Quote?> GetLatestQuoteAsync(string symbol, CancellationToken ct = default);
    Task<IReadOnlyList<Quote>> GetQuoteHistoryAsync(string symbol, DateOnly from, DateOnly to, CancellationToken ct = default);
    
    // Candles
    Task UpsertCandlesAsync(IEnumerable<Candle> candles, CancellationToken ct = default);
    Task<IReadOnlyList<Candle>> GetCandlesAsync(string symbol, string resolution, DateOnly from, DateOnly to, CancellationToken ct = default);
    
    // Calendars
    Task UpsertEconomicCalendarEventsAsync(IEnumerable<EconomicCalendarEvent> events, CancellationToken ct = default);
    Task<IReadOnlyList<EconomicCalendarEvent>> GetUpcomingEconomicEventsAsync(int days = 7, CancellationToken ct = default);
    Task<IReadOnlyList<EconomicCalendarEvent>> GetHighImpactEventsAsync(DateOnly from, DateOnly to, CancellationToken ct = default);
    Task UpsertEarningsCalendarEventsAsync(IEnumerable<EarningsCalendarEvent> events, CancellationToken ct = default);
    Task<IReadOnlyList<EarningsCalendarEvent>> GetUpcomingEarningsAsync(int days = 7, CancellationToken ct = default);
    Task UpsertIpoCalendarEventsAsync(IEnumerable<IpoCalendarEvent> events, CancellationToken ct = default);
    Task<IReadOnlyList<IpoCalendarEvent>> GetUpcomingIposAsync(int days = 30, CancellationToken ct = default);
    
    // Sentiment
    Task UpsertNewsSentimentAsync(NewsSentiment sentiment, CancellationToken ct = default);
    Task<NewsSentiment?> GetLatestNewsSentimentAsync(string symbol, CancellationToken ct = default);
    Task UpsertSocialSentimentAsync(IEnumerable<SocialSentiment> sentiments, CancellationToken ct = default);
    Task<IReadOnlyList<SocialSentiment>> GetSocialSentimentHistoryAsync(string symbol, DateOnly from, DateOnly to, CancellationToken ct = default);
    Task UpsertInsiderSentimentAsync(IEnumerable<InsiderSentiment> sentiments, CancellationToken ct = default);
    Task<InsiderSentiment?> GetLatestInsiderSentimentAsync(string symbol, CancellationToken ct = default);
    
    // Analyst Data
    Task UpsertRecommendationsAsync(IEnumerable<Recommendation> recs, CancellationToken ct = default);
    Task<Recommendation?> GetLatestRecommendationAsync(string symbol, CancellationToken ct = default);
    Task UpsertPriceTargetAsync(PriceTarget target, CancellationToken ct = default);
    Task<PriceTarget?> GetLatestPriceTargetAsync(string symbol, CancellationToken ct = default);
    
    // Company Data
    Task UpsertCompanyProfileAsync(CompanyProfile profile, CancellationToken ct = default);
    Task<CompanyProfile?> GetCompanyProfileAsync(string symbol, CancellationToken ct = default);
    
    // gRPC Streaming
    Task<IReadOnlyList<FinnhubObservation>> GetObservationsSinceAsync(DateTime since, CancellationToken ct = default);
}
```

**Acceptance Criteria:**
- [ ] IFinnhubApiClient exposes all 18 API methods
- [ ] IFinnhubRepository has methods for all 12 data types
- [ ] RateLimitStatus includes computed properties

---

### T4.4: Implement Rate-Limited API Client with Full Observability
**Estimate:** 2.5 hours

This is the critical component implementing all 18 API methods with:
- Token bucket rate limiter (60/min)
- OpenTelemetry tracing and metrics
- Structured logging
- Retry logic with exponential backoff
- Rate limit header synchronization

**Key implementation patterns:**

```csharp
// Infrastructure/TokenBucketRateLimiter.cs
public sealed class TokenBucketRateLimiter : IAsyncDisposable
{
    private readonly int _maxTokens = 60;
    private readonly TimeSpan _replenishPeriod = TimeSpan.FromMinutes(1);
    private double _currentTokens;
    
    // Metrics
    private static readonly Meter s_meter = new("FinnhubCollector.RateLimiter");
    private static readonly Counter<long> s_requestsAllowed;
    private static readonly Counter<long> s_requestsThrottled;
    private static readonly ObservableGauge<double> s_tokensAvailable;
    
    public Task<bool> TryAcquireAsync(CancellationToken ct);
    public void UpdateFromHeaders(int remaining, DateTime resetAt);
}

// Infrastructure/FinnhubApiClient.cs - implements all 18 methods
public sealed class FinnhubApiClient : IFinnhubApiClient
{
    private static readonly ActivitySource s_activitySource = new("FinnhubCollector.ApiClient");
    private static readonly Meter s_meter = new("FinnhubCollector.ApiClient");
    
    // Each method follows this pattern:
    public async Task<Quote?> GetQuoteAsync(string symbol, CancellationToken ct)
    {
        using var activity = s_activitySource.StartActivity("GetQuote");
        activity?.SetTag("finnhub.symbol", symbol);
        
        var response = await ExecuteWithRetryAsync<QuoteResponse>($"/quote?symbol={symbol}", ct);
        // Map response to domain model
        // Record metrics
        return quote;
    }
}
```

**Acceptance Criteria:**
- [ ] All 18 API methods implemented
- [ ] Rate limiter respects 60/min limit
- [ ] ActivitySource traces for each API call
- [ ] Metrics: requests_total, requests_succeeded, requests_failed, request_duration
- [ ] Rate limit headers update internal state
- [ ] Exponential backoff retry on 429 responses

---

### T4.5: Implement TimescaleDB Repository with Observability
**Estimate:** 1.5 hours

Implement comprehensive repository with methods for all 12 data types:

```csharp
public sealed class FinnhubRepository : IFinnhubRepository
{
    private static readonly ActivitySource s_activitySource = new("FinnhubCollector.Repository");
    private static readonly Meter s_meter = new("FinnhubCollector.Repository");
    
    // ~30 repository methods covering all data types
    // Each with Activity spans and metrics
}
```

**Acceptance Criteria:**
- [ ] All repository methods implemented
- [ ] OpenTelemetry Activity spans on each operation
- [ ] Metrics: operations_total, errors_total, operation_duration_seconds
- [ ] Upsert patterns prevent duplicates
- [ ] Proper hypertable queries for time-series data

---

### T4.6: Create Database Migration (12 Tables)
**Estimate:** 30 min

**migrations/001_initial_schema.sql:**
```sql
-- FinnhubCollector Database Schema
-- 12 tables supporting full API fidelity

-- Series configuration
CREATE TABLE IF NOT EXISTS finnhub_series (
    id SERIAL PRIMARY KEY,
    series_id TEXT UNIQUE NOT NULL,
    symbol TEXT NOT NULL,
    title TEXT NOT NULL,
    category TEXT NOT NULL,
    series_type TEXT NOT NULL,
    resolution TEXT,
    is_active BOOLEAN DEFAULT true,
    last_collected_at TIMESTAMPTZ,
    poll_interval_seconds INTEGER DEFAULT 60,
    priority INTEGER DEFAULT 10,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Real-time quotes (hypertable)
CREATE TABLE IF NOT EXISTS finnhub_quotes (
    id BIGSERIAL,
    symbol TEXT NOT NULL,
    current_price NUMERIC(18,4) NOT NULL,
    change NUMERIC(18,4),
    percent_change NUMERIC(10,4),
    high NUMERIC(18,4),
    low NUMERIC(18,4),
    open NUMERIC(18,4),
    previous_close NUMERIC(18,4),
    timestamp TIMESTAMPTZ NOT NULL,
    collected_at TIMESTAMPTZ NOT NULL,
    series_config_id INTEGER REFERENCES finnhub_series(id),
    PRIMARY KEY (symbol, timestamp)
);
SELECT create_hypertable('finnhub_quotes', 'timestamp', if_not_exists => TRUE);

-- OHLCV Candles (hypertable)
CREATE TABLE IF NOT EXISTS finnhub_candles (
    id BIGSERIAL,
    series_id TEXT NOT NULL,
    symbol TEXT NOT NULL,
    resolution TEXT NOT NULL,
    date DATE NOT NULL,
    timestamp TIMESTAMPTZ NOT NULL,
    open NUMERIC(18,4),
    high NUMERIC(18,4),
    low NUMERIC(18,4),
    close NUMERIC(18,4),
    volume BIGINT,
    collected_at TIMESTAMPTZ NOT NULL,
    series_config_id INTEGER REFERENCES finnhub_series(id),
    PRIMARY KEY (symbol, resolution, timestamp)
);
SELECT create_hypertable('finnhub_candles', 'timestamp', if_not_exists => TRUE);

-- Economic Calendar Events
CREATE TABLE IF NOT EXISTS finnhub_economic_calendar (
    id BIGSERIAL PRIMARY KEY,
    country TEXT NOT NULL,
    event TEXT NOT NULL,
    impact TEXT NOT NULL,
    event_time TIMESTAMPTZ NOT NULL,
    actual TEXT,
    estimate TEXT,
    previous TEXT,
    unit TEXT,
    collected_at TIMESTAMPTZ NOT NULL,
    UNIQUE(country, event, event_time)
);

-- Earnings Calendar Events
CREATE TABLE IF NOT EXISTS finnhub_earnings_calendar (
    id BIGSERIAL PRIMARY KEY,
    symbol TEXT NOT NULL,
    date DATE NOT NULL,
    hour TEXT,
    quarter INTEGER,
    year INTEGER,
    eps_estimate NUMERIC(10,4),
    eps_actual NUMERIC(10,4),
    revenue_estimate NUMERIC(18,2),
    revenue_actual NUMERIC(18,2),
    collected_at TIMESTAMPTZ NOT NULL,
    UNIQUE(symbol, date, year, quarter)
);

-- IPO Calendar Events
CREATE TABLE IF NOT EXISTS finnhub_ipo_calendar (
    id BIGSERIAL PRIMARY KEY,
    symbol TEXT NOT NULL,
    name TEXT NOT NULL,
    date DATE NOT NULL,
    exchange TEXT,
    price NUMERIC(18,4),
    number_of_shares BIGINT,
    total_shares_value NUMERIC(18,2),
    status TEXT,
    collected_at TIMESTAMPTZ NOT NULL,
    UNIQUE(symbol, date)
);

-- News Sentiment (hypertable)
CREATE TABLE IF NOT EXISTS finnhub_news_sentiment (
    id BIGSERIAL,
    symbol TEXT NOT NULL,
    articles_in_last_week INTEGER,
    buzz NUMERIC(10,4),
    weekly_average NUMERIC(10,4),
    company_news_score NUMERIC(10,4),
    sector_average_bullish_percent NUMERIC(10,4),
    sector_average_news_score NUMERIC(10,4),
    bullish_percent NUMERIC(10,4),
    bearish_percent NUMERIC(10,4),
    collected_at TIMESTAMPTZ NOT NULL,
    PRIMARY KEY (symbol, collected_at)
);
SELECT create_hypertable('finnhub_news_sentiment', 'collected_at', if_not_exists => TRUE);

-- Social Sentiment (hypertable)
CREATE TABLE IF NOT EXISTS finnhub_social_sentiment (
    id BIGSERIAL,
    symbol TEXT NOT NULL,
    date DATE NOT NULL,
    reddit_mentions INTEGER,
    reddit_positive_mentions INTEGER,
    reddit_negative_mentions INTEGER,
    reddit_positive_score NUMERIC(10,4),
    reddit_negative_score NUMERIC(10,4),
    twitter_mentions INTEGER,
    twitter_positive_mentions INTEGER,
    twitter_negative_mentions INTEGER,
    twitter_positive_score NUMERIC(10,4),
    twitter_negative_score NUMERIC(10,4),
    collected_at TIMESTAMPTZ NOT NULL,
    PRIMARY KEY (symbol, date)
);
SELECT create_hypertable('finnhub_social_sentiment', 'date', if_not_exists => TRUE);

-- Insider Sentiment (MSPR)
CREATE TABLE IF NOT EXISTS finnhub_insider_sentiment (
    id BIGSERIAL PRIMARY KEY,
    symbol TEXT NOT NULL,
    year INTEGER NOT NULL,
    month INTEGER NOT NULL,
    change BIGINT,
    mspr NUMERIC(10,4),
    collected_at TIMESTAMPTZ NOT NULL,
    UNIQUE(symbol, year, month)
);

-- Analyst Recommendations
CREATE TABLE IF NOT EXISTS finnhub_recommendations (
    id BIGSERIAL PRIMARY KEY,
    symbol TEXT NOT NULL,
    period DATE NOT NULL,
    strong_buy INTEGER,
    buy INTEGER,
    hold INTEGER,
    sell INTEGER,
    strong_sell INTEGER,
    collected_at TIMESTAMPTZ NOT NULL,
    UNIQUE(symbol, period)
);

-- Price Targets (hypertable)
CREATE TABLE IF NOT EXISTS finnhub_price_targets (
    id BIGSERIAL,
    symbol TEXT NOT NULL,
    last_updated DATE,
    target_high NUMERIC(18,4),
    target_low NUMERIC(18,4),
    target_mean NUMERIC(18,4),
    target_median NUMERIC(18,4),
    collected_at TIMESTAMPTZ NOT NULL,
    PRIMARY KEY (symbol, collected_at)
);
SELECT create_hypertable('finnhub_price_targets', 'collected_at', if_not_exists => TRUE);

-- Company Profiles
CREATE TABLE IF NOT EXISTS finnhub_company_profiles (
    id BIGSERIAL PRIMARY KEY,
    symbol TEXT UNIQUE NOT NULL,
    name TEXT,
    country TEXT,
    currency TEXT,
    exchange TEXT,
    ipo DATE,
    market_capitalization NUMERIC(20,2),
    finnhub_industry TEXT,
    logo TEXT,
    phone TEXT,
    share_outstanding NUMERIC(18,4),
    web_url TEXT,
    collected_at TIMESTAMPTZ NOT NULL
);

-- Indexes
CREATE INDEX IF NOT EXISTS idx_finnhub_series_active ON finnhub_series(is_active, priority);
CREATE INDEX IF NOT EXISTS idx_finnhub_quotes_symbol ON finnhub_quotes(symbol, timestamp DESC);
CREATE INDEX IF NOT EXISTS idx_finnhub_candles_lookup ON finnhub_candles(symbol, resolution, date DESC);
CREATE INDEX IF NOT EXISTS idx_finnhub_economic_time ON finnhub_economic_calendar(event_time DESC);
CREATE INDEX IF NOT EXISTS idx_finnhub_economic_impact ON finnhub_economic_calendar(impact, event_time);
CREATE INDEX IF NOT EXISTS idx_finnhub_earnings_date ON finnhub_earnings_calendar(date DESC);
CREATE INDEX IF NOT EXISTS idx_finnhub_ipo_date ON finnhub_ipo_calendar(date DESC);
CREATE INDEX IF NOT EXISTS idx_finnhub_insider_symbol ON finnhub_insider_sentiment(symbol, year DESC, month DESC);
CREATE INDEX IF NOT EXISTS idx_finnhub_recommendations_symbol ON finnhub_recommendations(symbol, period DESC);

-- Initial Series Data (23 symbols)
INSERT INTO finnhub_series (series_id, symbol, title, category, series_type, poll_interval_seconds, priority)
VALUES
    -- VIX Proxy (critical for deployment triggers)
    ('FH/VXX', 'VXX', 'iPath Series B S&P 500 VIX Short-Term', 'Volatility', 'Quote', 60, 1),
    
    -- Major Indices
    ('FH/SPY', 'SPY', 'S&P 500 ETF', 'Index', 'Quote', 60, 2),
    ('FH/QQQ', 'QQQ', 'Nasdaq 100 ETF', 'Index', 'Quote', 60, 3),
    ('FH/IWM', 'IWM', 'Russell 2000 ETF', 'Index', 'Quote', 60, 4),
    ('FH/DIA', 'DIA', 'Dow Jones ETF', 'Index', 'Quote', 120, 5),
    
    -- All 8 Sector ETFs
    ('FH/XLF', 'XLF', 'Financial Sector ETF', 'Sector', 'Quote', 300, 10),
    ('FH/XLE', 'XLE', 'Energy Sector ETF', 'Sector', 'Quote', 300, 11),
    ('FH/XLK', 'XLK', 'Technology Sector ETF', 'Sector', 'Quote', 300, 12),
    ('FH/XLV', 'XLV', 'Healthcare Sector ETF', 'Sector', 'Quote', 300, 13),
    ('FH/XLI', 'XLI', 'Industrial Sector ETF', 'Sector', 'Quote', 300, 14),
    ('FH/XLU', 'XLU', 'Utilities Sector ETF', 'Sector', 'Quote', 300, 15),
    ('FH/XLP', 'XLP', 'Consumer Staples ETF', 'Sector', 'Quote', 300, 16),
    ('FH/XLY', 'XLY', 'Consumer Discretionary ETF', 'Sector', 'Quote', 300, 17),
    
    -- Credit/Rates
    ('FH/HYG', 'HYG', 'High Yield Corporate Bond ETF', 'Credit', 'Quote', 300, 20),
    ('FH/LQD', 'LQD', 'Investment Grade Corporate Bond ETF', 'Credit', 'Quote', 300, 21),
    ('FH/TLT', 'TLT', 'Long-Term Treasury ETF', 'Rates', 'Quote', 300, 22),
    ('FH/SHY', 'SHY', 'Short-Term Treasury ETF', 'Rates', 'Quote', 300, 23),
    ('FH/IEF', 'IEF', 'Intermediate Treasury ETF', 'Rates', 'Quote', 300, 24),
    
    -- Commodities
    ('FH/GLD', 'GLD', 'Gold ETF', 'Commodity', 'Quote', 300, 30),
    ('FH/SLV', 'SLV', 'Silver ETF', 'Commodity', 'Quote', 300, 31),
    ('FH/USO', 'USO', 'Oil ETF', 'Commodity', 'Quote', 300, 32),
    
    -- International
    ('FH/EFA', 'EFA', 'EAFE ETF (Developed Markets)', 'International', 'Quote', 600, 40),
    ('FH/EEM', 'EEM', 'Emerging Markets ETF', 'International', 'Quote', 600, 41)
ON CONFLICT (series_id) DO NOTHING;
```

**Acceptance Criteria:**
- [ ] Migration creates all 12 tables
- [ ] Hypertables created for time-series data
- [ ] Initial 23 series configured
- [ ] Indexes support common query patterns

---

### T4.7: Implement Collection Workers with Observability
**Estimate:** 2 hours

Four background workers with comprehensive observability:

1. **QuoteCollectionWorker** - Real-time quotes (60s during market hours)
2. **CalendarCollectionWorker** - Economic/Earnings/IPO calendars (every 6 hours)
3. **SentimentCollectionWorker** - News/Social/Insider sentiment (every 4 hours)
4. **AnalystDataCollectionWorker** - Recommendations/Price targets (daily)

Key metrics:
- VXX price as observable gauge for VIX proxy monitoring
- Collection cycle durations
- Error rates per worker

**Acceptance Criteria:**
- [ ] All 4 workers implemented with ActivitySource tracing
- [ ] VXX price gauge exposed (finnhub_vxx_current_price)
- [ ] Workers respect rate limits and poll intervals
- [ ] Graceful handling of rate limit exhaustion

---

### T4.8: Implement gRPC Server
**Estimate:** 45 min

Implement ObservationEventStreamService for ThresholdEngine integration.

**Acceptance Criteria:**
- [ ] SubscribeToEvents streams new observations
- [ ] GetEventsSince returns historical observations
- [ ] Active subscription count gauge
- [ ] Events streamed counter

---

### T4.9: Configure Service Host with OpenTelemetry
**Estimate:** 45 min

Full OpenTelemetry configuration with:
- 7 ActivitySources registered
- 7 Meters registered
- OTLP export to collector
- Health checks for database and Finnhub API
- /status endpoint with rate limit info

**Acceptance Criteria:**
- [ ] All traces exported to Tempo
- [ ] All metrics exported to Prometheus
- [ ] Health checks pass
- [ ] Rate limit status visible at /status

---

### T4.10: Create Containerfile and DevContainer
**Estimate:** 30 min

**Acceptance Criteria:**
- [ ] Container builds successfully
- [ ] DevContainer works with VS Code
- [ ] Health check endpoint functional
- [ ] Environment variables properly passed

---

### T4.11: Write Comprehensive Unit Tests
**Estimate:** 1.5 hours

Test coverage for:
- Rate limiter token acquisition and replenishment
- API client response parsing for all data types
- Repository methods
- Mock HTTP handlers for Finnhub responses

**Acceptance Criteria:**
- [ ] >80% coverage on critical paths
- [ ] All tests pass with `dotnet test`

---

### T4.12: Create Grafana Dashboard
**Estimate:** 45 min

Dashboard panels:
- Service health status
- VXX price gauge with VIX thresholds (22/30)
- Rate limit remaining gauge
- API request rates and durations
- Collection worker metrics
- Error rates
- Active gRPC subscriptions

**Acceptance Criteria:**
- [ ] Dashboard imports successfully
- [ ] VXX gauge shows deployment trigger thresholds
- [ ] All key metrics visible

---

### T4.13: Integration Test with ThresholdEngine
**Estimate:** 45 min

1. Add to compose.yaml
2. Create VXX deployment trigger pattern
3. Verify data flows to ThresholdEngine

**Acceptance Criteria:**
- [ ] FinnhubCollector starts and collects data
- [ ] VXX quotes stored in TimescaleDB
- [ ] ThresholdEngine receives gRPC events
- [ ] VXX deployment trigger pattern evaluates

---

### T4.14: Add to Ansible Deployment
**Estimate:** 30 min

**Acceptance Criteria:**
- [ ] Ansible playbook deploys FinnhubCollector
- [ ] Environment variables properly set
- [ ] Service starts on deployment

---

## Summary

| Metric | Value |
|--------|-------|
| **Total Tasks** | 14 |
| **Estimated Hours** | ~14 hours |
| **Domain Models** | 13 |
| **API Methods** | 18 |
| **Database Tables** | 12 |
| **Initial Series** | 23 symbols |
| **ActivitySources** | 7 |
| **Meters** | 7 |

## Rate Limit Budget

With 60 calls/minute (86,400/day):

| Worker | Symbols | Frequency | Calls/Min |
|--------|---------|-----------|-----------|
| Quotes | 23 | 60-600s | ~15 |
| Calendar | 3 types | Every 6h | ~0.01 |
| Sentiment | 4 symbols | Every 4h | ~0.02 |
| Analyst | 4 symbols | Daily | ~0.01 |
| Market Status | 1 | Every min | 1 |

**Total**: ~16 calls/min - well within 60/min limit

## ATLAS Value Add

1. **VXX Deployment Triggers** - Level 1 (≥22): $100K, Level 2 (≥30): $500K
2. **Economic Calendar** - Fed, CPI, Employment events for regime anticipation
3. **Triple Sentiment** - News + Social + Insider for multi-source confirmation
4. **Earnings Calendar** - Volatility prediction around earnings
5. **Sector Rotation** - All 8 SPDR sector ETFs tracked
6. **Credit Spreads** - HYG/LQD for credit stress monitoring
7. **Analyst Consensus** - Recommendations and price targets

## Environment Variables

```bash
FINNHUB_API_KEY=your_finnhub_api_key
DB_PASSWORD=your_db_password
ConnectionStrings__Atlas=Host=timescaledb;Database=atlas;Username=atlas;Password=${DB_PASSWORD}
OpenTelemetry__Endpoint=http://otel-collector:4317
```

## Ports

- 5008: REST API / Health / Metrics
- 5009: gRPC

## Observability Checklist

### ActivitySources (7)
- [x] FinnhubCollector.ApiClient
- [x] FinnhubCollector.Repository
- [x] FinnhubCollector.QuoteWorker
- [x] FinnhubCollector.CalendarWorker
- [x] FinnhubCollector.SentimentWorker
- [x] FinnhubCollector.AnalystWorker
- [x] FinnhubCollector.GrpcServer

### Meters (7)
- [x] FinnhubCollector.ApiClient
- [x] FinnhubCollector.Repository
- [x] FinnhubCollector.QuoteWorker
- [x] FinnhubCollector.CalendarWorker
- [x] FinnhubCollector.SentimentWorker
- [x] FinnhubCollector.AnalystWorker
- [x] FinnhubCollector.GrpcServer

### Key Metrics
- `finnhub_vxx_current_price` - VIX proxy gauge (critical)
- `finnhub_api_requests_total` - API request counter
- `finnhub_api_request_duration_seconds` - Latency histogram
- `finnhub_ratelimiter_tokens_available` - Rate limit gauge
- `finnhub_worker_*_collected_total` - Collection counters
- `finnhub_grpc_active_subscriptions` - Subscription gauge
