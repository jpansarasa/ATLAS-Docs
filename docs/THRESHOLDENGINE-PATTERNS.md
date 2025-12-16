# ThresholdEngine Pattern Reference

**Status**: Production
**Updated**: 2025-12-16

Comprehensive reference for ThresholdEngine pattern configuration, expression API, weighting, and signal freshness decay.

---

## Table of Contents

1. [Overview](#overview)
2. [Pattern Expression API](#pattern-expression-api)
3. [Pattern Weighting](#pattern-weighting)
4. [Signal Freshness & Decay](#signal-freshness--decay)
5. [Configuration Reference](#configuration-reference)
6. [Implementation Details](#implementation-details)

---

## Overview

ThresholdEngine evaluates economic patterns to produce a weighted macro score for regime detection. Each pattern:

1. **Evaluates** a condition using C# expressions against time-series data
2. **Produces** a signal (-2 to +2) indicating directional strength
3. **Applies** weighting based on reliability, freshness, and temporal relevance
4. **Contributes** to category and macro scores

**Weighted Scoring Formula**:
```
weightedSignal = signal × weight × freshnessFactor × temporalMultiplier
categoryScore = Σ(weightedSignals) / Σ(weights)
macroScore = Σ(categoryScore × categoryWeight)
```

---

## Pattern Expression API

Pattern expressions are C# code snippets compiled at runtime using Roslyn. They have access to a `PatternEvaluationContext` object (`ctx`).

### Context Properties

| Property | Type | Description |
|----------|------|-------------|
| `ctx.CurrentDate` | `DateTimeOffset` | Evaluation timestamp |
| `ctx.CurrentRegime` | `MacroRegime` | Current economic regime |
| `ctx.MacroScore` | `decimal` | Composite macro score (-100 to +100) |
| `ctx.MacroScoreTrend` | `decimal` | Rate of change in macro score |

### Data Access Methods

| Method | Return | Description |
|--------|--------|-------------|
| `ctx.GetLatest(seriesId)` | `decimal?` | Most recent observation value |
| `ctx.GetHistorical(seriesId, date)` | `decimal?` | Value at specific date |
| `ctx.GetYoY(seriesId)` | `decimal?` | Year-over-year % change |
| `ctx.GetMoM(seriesId)` | `decimal?` | Month-over-month % change |
| `ctx.GetMA(seriesId, days)` | `decimal?` | Moving average over period |
| `ctx.GetSpread(series1, series2)` | `decimal?` | Difference between two series |
| `ctx.GetRatio(numerator, denominator)` | `decimal?` | Ratio of two series |
| `ctx.GetLowest(seriesId, days)` | `decimal?` | Minimum value in period |
| `ctx.GetHighest(seriesId, days)` | `decimal?` | Maximum value in period |
| `ctx.IsSustained(seriesId, condition, days)` | `bool` | Check if condition holds over window |

### Available Functions

**System.Math**:
```csharp
Math.Abs(value)           // Absolute value
Math.Max(a, b)            // Maximum of two values
Math.Min(a, b)            // Minimum of two values
Math.Round(value, digits) // Round to decimal places
Math.Floor(value)         // Round down
Math.Ceiling(value)       // Round up
Math.Sqrt(value)          // Square root
Math.Pow(base, exp)       // Exponentiation
Math.Sign(value)          // Sign (-1, 0, +1)
```

**System.Linq** (on collections):
```csharp
values.Average()          // Mean
values.Sum()              // Sum
values.Count()            // Count
values.Min() / Max()      // Min/Max
values.Any(predicate)     // Any match
values.All(predicate)     // All match
```

### MacroRegime Enum

```csharp
public enum MacroRegime
{
    Crisis = -3,
    Recession = -2,
    LateCycle = -1,
    Neutral = 0,
    Recovery = 1,
    Growth = 2
}
```

### Expression Types

**Condition Expression** - Returns `bool`, determines if pattern triggers:
```csharp
ctx.GetLatest("VIXCLS") > 22m
```

**Signal Expression** - Returns `decimal` in [-2, +2]:

| Signal | Interpretation |
|--------|----------------|
| +2.0 | Strong offensive (deploy cash, increase equity) |
| +1.0 | Moderate offensive (gradual risk increase) |
| 0.0 | Neutral (maintain allocation) |
| -1.0 | Moderate defensive (gradual risk reduction) |
| -2.0 | Strong defensive (raise cash, reduce equity) |

### Expression Examples

**VIX Deployment (Regime-Aware)**:
```csharp
// Condition
ctx.GetLatest("VIXCLS") > 22m

// Signal
var vix = ctx.GetLatest("VIXCLS") ?? 0m;
var score = ctx.MacroScore;
if (vix <= 22m) return 0m;
if (score < -10m) return vix > 30m ? 2m : 1m;   // Recession: VIX spike = deploy
if (score > 10m) return vix > 30m ? -2m : -1m;  // Growth: VIX spike = reduce
return vix > 30m ? -1m : -0.5m;                  // Neutral: moderate defensive
```

**Sahm Rule (Moving Average)**:
```csharp
// Condition
var current = ctx.GetMA("UNRATE", 90);
var low12mo = ctx.GetLowest("UNRATE", 365);
return (current - low12mo) >= 0.5m;

// Signal
var diff = (ctx.GetMA("UNRATE", 90) ?? 0m) - (ctx.GetLowest("UNRATE", 365) ?? 0m);
return diff >= 0.5m ? -2m : 0m;
```

**Yield Curve Inversion**:
```csharp
var spread = ctx.GetSpread("DGS10", "DGS2");
return spread.HasValue && spread.Value < 0m;
```

**Credit Spread Widening (Tiered)**:
```csharp
var spread = ctx.GetLatest("BAMLH0A0HYM2") ?? 0m;
if (spread <= 350m) return 0m;
if (spread > 500m) return -2m;
if (spread > 400m) return -1.5m;
return -1m;
```

### Null Safety

All `ctx.Get*` methods return nullable decimals:
```csharp
// Null-coalescing
var vix = ctx.GetLatest("VIXCLS") ?? 0m;

// HasValue check
var cape = ctx.GetLatest("CAPE");
if (!cape.HasValue) return 0m;
return cape.Value < 20m ? 1m : 0m;
```

### Compilation Details

- **Engine**: Roslyn C# compiler
- **Timeout**: 5 seconds
- **Optimization**: Release mode
- **Caching**: Compiled delegates cached by pattern ID

---

## Pattern Weighting

### Weight Ranges

| Range | Classification | Criteria |
|-------|----------------|----------|
| **0.90-1.00** | Proven Reliable | 100% historical accuracy, decades of data |
| **0.75-0.89** | Strong Signal | High reliability, timely data |
| **0.60-0.74** | Solid Contributor | Useful signal, some noise |
| **0.40-0.59** | Supplementary | High noise, lagging, informational |
| **0.00-0.39** | Experimental | Unproven, testing |

### Weight Assignment Criteria

**Historical Accuracy (40%)**: False positive/negative rates
- 100% accuracy: 0.90-0.95
- <10% error: 0.80-0.89
- 10-20% error: 0.70-0.79
- 20-30% error: 0.60-0.69
- >30% error: 0.40-0.59

**Timeliness (30%)**: Update frequency and lag
- Daily, <7 day lag: 0.80-0.90
- Weekly, <14 day lag: 0.70-0.85
- Monthly, <30 day lag: 0.60-0.75
- Quarterly, >45 day lag: 0.40-0.60

**Signal Clarity (20%)**: Threshold objectivity
- Clear threshold, binary: 0.85-0.95
- Clear threshold, range: 0.70-0.84
- Fuzzy, requires interpretation: 0.50-0.69

**Data Quality (10%)**: Source authority
- Official government, minimal revisions: 0.80-0.95
- Private source, established: 0.60-0.74

### Temporal Classification

| Type | Description | Adjustment |
|------|-------------|------------|
| **Leading** | Signal 3+ months ahead of events | +30% during transitions |
| **Coincident** | Real-time snapshot | Neutral |
| **Lagging** | Confirms past events | -30% during transitions |

**Examples**:
- Yield Curve Inversion: Leading (12-18mo ahead)
- ISM Manufacturing: Coincident (current conditions)
- CPI: Lagging (confirms past inflation)

### Publication Frequency

Set `publicationFrequencyDays` to actual data release schedule:

| Frequency | Value | Examples |
|-----------|-------|----------|
| Real-time | 0 | VIX, DXY, credit spreads |
| Daily | 1 | Treasury yields, repo rates |
| Weekly | 7 | Jobless claims, M2 |
| Monthly | 30 | ISM, CPI, payrolls |
| Quarterly | 90 | GDP |

**Critical**: Data remains 100% fresh until `publicationFrequencyDays` passes.

### Example Pattern Weights

| Pattern | Weight | Temporal | Pub Freq | Decay | Rationale |
|---------|--------|----------|----------|-------|-----------|
| Sahm Rule | 0.95 | Coincident | 30 | 90 | 100% historical accuracy |
| Yield Curve | 0.90 | Leading | 1 | 180 | 12-18mo lead, persistent |
| ISM Contraction | 0.85 | Coincident | 30 | 30 | Timely monthly survey |
| VIX L2 | 0.85 | Coincident | 0 | 7 | Extreme volatility |
| Initial Claims | 0.80 | Coincident | 7 | 30 | Weekly labor data |
| Consumer Sentiment | 0.50 | Lagging | 30 | 60 | Survey-based, noisy |

---

## Signal Freshness & Decay

### Publication-Frequency-Aware Decay

**Problem**: Naive decay penalizes current data. ISM published 15 days ago shouldn't be considered stale when it publishes monthly.

**Solution**: Only decay data that is **overdue**.

```
freshness(t) = e^(-max(0, t - f) / τ)
```

Where:
- `t` = days since last observation
- `f` = `publicationFrequencyDays` (expected update interval)
- `τ` = `signalDecayDays` (decay constant)

### Decay Examples

| Indicator | Pub Freq | Days Since | Days Overdue | Freshness |
|-----------|----------|------------|--------------|-----------|
| VIX | 0 | 1 | 1 | 86% |
| Initial Claims | 7 | 3 | 0 | **100%** |
| Initial Claims | 7 | 15 | 8 | 77% |
| ISM | 30 | 15 | 0 | **100%** |
| ISM | 30 | 50 | 20 | 51% |
| GDP | 90 | 45 | 0 | **100%** |

### Decay Curves

**Fast Decay (τ=7 days)** - Real-time market data:
- 7 days overdue: 37%
- 14 days overdue: 14%
- 21 days overdue: 5%

**Moderate Decay (τ=30 days)** - Monthly indicators:
- 30 days overdue: 37%
- 60 days overdue: 14%
- 90 days overdue: 5%

**Slow Decay (τ=90 days)** - Quarterly/persistent signals:
- 90 days overdue: 37%
- 180 days overdue: 14%
- 270 days overdue: 5%

### Temporal Multipliers

Applied during regime transitions (MacroScoreTrend > ±0.5):

| Temporal Type | Stable | Transitioning |
|---------------|--------|---------------|
| Leading | 1.0× | **1.3×** |
| Coincident | 1.0× | 1.0× |
| Lagging | 1.0× | **0.7×** |

### Multi-Series Patterns

For patterns with multiple `requiredSeries`: use the **oldest** observation date.

```csharp
// NBFI Escalation uses HY spreads (0d), Repo (1d), NFCI (30d)
// If NFCI is 40 days old → pattern is 10 days overdue
```

**Rationale**: Pattern only "current" when ALL data is current.

---

## Configuration Reference

### Pattern Schema

```json
{
  "patternId": "string",
  "name": "string",
  "category": "Recession|Liquidity|Growth|NBFI|Inflation|Currency|Valuation|Commodity",
  "expression": "C# condition expression",
  "signalExpression": "C# signal expression",
  "requiredSeries": ["SERIES1", "SERIES2"],
  "enabled": true,

  "weight": 0.85,
  "temporalType": "Leading|Coincident|Lagging",
  "leadTimeMonths": 0,
  "publicationFrequencyDays": 30,
  "signalDecayDays": 30,
  "confidence": 0.75
}
```

### Field Descriptions

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `weight` | decimal | 1.0 | Reliability weight (0.0-1.0) |
| `temporalType` | enum | Coincident | Signal timing relative to events |
| `leadTimeMonths` | int | 0 | Months ahead (+) or behind (-) |
| `publicationFrequencyDays` | int | 30 | Expected days between publications |
| `signalDecayDays` | int | 30 | Decay constant after overdue |
| `confidence` | decimal | 0.75 | Historical accuracy (informational) |

### Example Pattern

```json
{
  "patternId": "ism-contraction",
  "name": "ISM Manufacturing Below 50",
  "category": "Recession",
  "expression": "ctx.GetLatest(\"NAPM\") < 50m",
  "signalExpression": "var ism = ctx.GetLatest(\"NAPM\") ?? 50m; return ism < 45m ? -2m : ism < 50m ? -1m : 0m;",
  "requiredSeries": ["NAPM"],
  "enabled": true,
  "weight": 0.85,
  "temporalType": "Coincident",
  "leadTimeMonths": 0,
  "publicationFrequencyDays": 30,
  "signalDecayDays": 30,
  "confidence": 0.85
}
```

### Category Weights

| Category | Weight | Purpose |
|----------|--------|---------|
| Recession | 30% | Downturn detection |
| Liquidity | 20% | Market stress |
| Growth | 20% | Expansion signals |
| NBFI | 10% | Shadow banking risk |
| Currency | 10% | FX risk-off |
| Inflation | 10% | Price pressure |
| Valuation | 0% | Informational |
| Commodity | 0% | Informational |

---

## Implementation Details

### Weighted Scoring Algorithm

```
GetWeightedSignal(pattern, rawSignal):
    freshness = CalculateFreshnessFactor(pattern)
    temporalMultiplier = GetTemporalMultiplier(pattern)
    return rawSignal × pattern.weight × freshness × temporalMultiplier

CalculateFreshnessFactor(pattern):
    // Find oldest observation across all required series
    oldestUpdate = null
    for each seriesId in pattern.requiredSeries:
        lastUpdate = cache.GetLatestDate(seriesId)
        if lastUpdate is null: return 0        // Missing data
        if oldestUpdate is null or lastUpdate < oldestUpdate:
            oldestUpdate = lastUpdate

    daysSince = (now - oldestUpdate).days
    daysOverdue = max(0, daysSince - pattern.publicationFrequencyDays)

    if daysOverdue == 0: return 1.0            // Still current

    return exp(-daysOverdue / pattern.signalDecayDays)

GetTemporalMultiplier(pattern):
    if not isRegimeTransitioning: return 1.0

    if pattern.temporalType == Leading:    return 1.3
    if pattern.temporalType == Coincident: return 1.0
    if pattern.temporalType == Lagging:    return 0.7
    return 1.0
```

### API Endpoints

**GET /api/patterns/contributions**
```json
{
  "macroScore": -4.4,
  "isTransitioning": false,
  "categories": [{
    "category": "Recession",
    "categoryWeight": 0.30,
    "patterns": [{
      "patternId": "sahm-rule",
      "weight": 0.95,
      "daysSincePublication": 15,
      "daysOverdue": 0,
      "freshnessFactor": 1.0,
      "rawSignal": -2.0,
      "weightedSignal": -1.90
    }]
  }]
}
```

**GET /api/patterns/health**
```json
{
  "healthStatus": "Healthy",
  "totalPatterns": 57,
  "summary": {
    "current": 52,
    "slightlyOverdue": 4,
    "severelyOverdue": 1,
    "noData": 0
  }
}
```

### Prometheus Metrics

| Metric | Type | Description |
|--------|------|-------------|
| `thresholdengine.pattern.weighted_signal` | Histogram | Weighted signals |
| `thresholdengine.pattern.freshness_factor` | Histogram | Freshness (0-1) |
| `thresholdengine.pattern.days_overdue` | Histogram | Days past publication |
| `thresholdengine.pattern.missing_data_total` | Counter | Missing data events |
| `thresholdengine.pattern.severely_overdue_total` | Counter | Severely overdue alerts |

---

## References

- [ARCHITECTURE.md](./ARCHITECTURE.md) - System design
- [ThresholdEngine/README.md](../ThresholdEngine/README.md) - Service documentation
- [Exponential Decay](https://en.wikipedia.org/wiki/Exponential_decay)
- [Weighted Averages](https://en.wikipedia.org/wiki/Weighted_arithmetic_mean)
