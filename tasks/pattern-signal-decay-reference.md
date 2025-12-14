# Pattern Signal Decay & Temporal Behavior

**Purpose**: Mathematical reference for signal freshness decay and temporal weighting  
**Audience**: Engineers implementing weighted pattern evaluation  
**Version**: 1.1  
**Updated**: 2025-12-14  

---

## Exponential Decay Formula

### The Problem with Naive Decay

**Issue**: Applying decay from "last observation date" penalizes indicators that are current within their normal publication cycle.

**Example**: ISM Manufacturing published 15 days ago
- Naive formula: `freshness = e^(-15/30) = 0.606` → **60% strength**
- **But it's the latest available data!** Should be **100% strength**

### Corrected Formula (Publication Frequency Aware)

```
freshness(t) = e^(-max(0, t - f) / τ)
```

Where:
- `t` = days since last observation
- `f` = `publicationFrequencyDays` (expected update interval)
- `τ` (tau) = `signalDecayDays` (decay constant)
- `e` = Euler's number (≈2.71828)

**Key Insight**: Only decay data that is **overdue**, not data that is current within its publication cycle.

### Implementation (C#)

```csharp
decimal CalculateFreshnessFactor(PatternConfiguration pattern)
{
    var lastUpdate = _observationCache.GetLatestDate(pattern.RequiredSeries[0]);
    if (!lastUpdate.HasValue)
        return 0.0m;  // No data = no signal

    var daysSincePublication = (DateTime.UtcNow - lastUpdate.Value).TotalDays;
    
    // Calculate days OVERDUE (negative means still current)
    var daysOverdue = Math.Max(0, daysSincePublication - pattern.PublicationFrequencyDays);
    
    if (daysOverdue == 0)
        return 1.0m;  // Still current within publication cycle
    
    // Only apply decay to overdue data
    var decayFactor = Math.Exp(-daysOverdue / pattern.SignalDecayDays);
    
    return (decimal)decayFactor;
}
```

### Publication Frequency Examples

| Indicator | Publication Freq | Last Published | Days Since | Days Overdue | Freshness |
|-----------|------------------|----------------|------------|--------------|-----------|
| VIX | 0 (real-time) | 1 day | 1 | 1 | e^(-1/7) = 86% |
| Initial Claims | 7 days | 3 days | 3 | 0 | **100%** ✅ |
| Initial Claims | 7 days | 15 days | 15 | 8 | e^(-8/30) = 77% |
| ISM Manufacturing | 30 days | 15 days | 15 | 0 | **100%** ✅ |
| ISM Manufacturing | 30 days | 50 days | 50 | 20 | e^(-20/30) = 51% |
| GDP | 90 days | 45 days | 45 | 0 | **100%** ✅ |
| GDP | 90 days | 120 days | 120 | 30 | e^(-30/90) = 72% |

---

## Publication Frequency by Indicator Type

### Real-Time / Intraday (publicationFrequencyDays = 0)
- **VIX**: Continuous during market hours
- **DXY**: Forex markets 24/5
- **Credit Spreads**: Calculated from bond prices (real-time)
- **Decay**: Starts immediately after last observation

### Daily (publicationFrequencyDays = 1)
- **Standing Repo Facility**: Published by Fed daily
- **Treasury Yields**: Daily market close
- **Reverse Repo**: Fed publishes daily

### Weekly (publicationFrequencyDays = 7)
- **Initial Jobless Claims**: Every Thursday
- **Continuing Claims**: Every Thursday
- **M2 Money Supply**: Weekly (Fed H.6 release)

### Monthly (publicationFrequencyDays = 30)
- **ISM Manufacturing/Services**: 1st business day of month
- **Nonfarm Payrolls**: 1st Friday of month
- **CPI / Core CPI**: Mid-month (~15th)
- **PCE / Core PCE**: End of month (~30th)
- **Industrial Production**: Mid-month (~15th)
- **Retail Sales**: Mid-month (~15th)
- **Housing Starts**: Mid-month (~17th)
- **Consumer Sentiment**: End of month (final)
- **Chicago NFCI**: Weekly but use monthly for stability
- **OFR Financial Stress Index**: Weekly but use monthly

### Quarterly (publicationFrequencyDays = 90)
- **GDP**: ~30 days after quarter end (advance estimate)
- **Corporate Earnings**: Staggered throughout earnings season

### Special Cases

**Sahm Rule (Unemployment-based)**:
- Uses unemployment rate (monthly) + 12-month moving average
- `publicationFrequencyDays = 30` (monthly unemployment data)
- Signal persists for 90 days → `signalDecayDays = 90`

**Yield Curve Inversion**:
- Uses daily Treasury yields but signal is persistent
- `publicationFrequencyDays = 1` (daily calculation)
- Signal persists for months → `signalDecayDays = 180`

**Freight Recession**:
- Monthly publication
- `publicationFrequencyDays = 30`
- Very persistent trend → `signalDecayDays = 180`

---

## Decay Curves by Period (For Overdue Data Only)

### Fast Decay (τ = 7 days)

**Use for**: Real-time market data (VIX, DXY, Standing Repo)

| Days Overdue | Freshness | % of Original |
|--------------|-----------|---------------|
| 0 | 1.000 | 100% (still current) |
| 1 | 0.868 | 87% |
| 3 | 0.653 | 65% |
| 7 | 0.368 | 37% |
| 14 | 0.135 | 14% |
| 21 | 0.050 | 5% |
| 30 | 0.015 | 1.5% |

**Interpretation**: Once overdue, signal loses 63% of strength in 7 days, effectively dead after 21 days overdue.

### Moderate Decay (τ = 30 days)

**Use for**: Monthly indicators (ISM, Initial Claims, most coincident)

| Days Overdue | Freshness | % of Original |
|--------------|-----------|---------------|
| 0 | 1.000 | 100% (still current) |
| 7 | 0.794 | 79% |
| 15 | 0.630 | 63% |
| 30 | 0.368 | 37% |
| 60 | 0.135 | 14% |
| 90 | 0.050 | 5% |
| 120 | 0.018 | 1.8% |

**Interpretation**: Once overdue, signal loses 63% of strength in 30 days, effectively dead after 90 days overdue.

### Slow Decay (τ = 90 days)

**Use for**: Quarterly data (GDP) and persistent trends (Sahm Rule, yield curve)

| Days Overdue | Freshness | % of Original |
|--------------|-----------|---------------|
| 0 | 1.000 | 100% (still current) |
| 30 | 0.717 | 72% |
| 60 | 0.514 | 51% |
| 90 | 0.368 | 37% |
| 180 | 0.135 | 14% |
| 270 | 0.050 | 5% |
| 365 | 0.023 | 2.3% |

**Interpretation**: Once overdue, signal loses 63% of strength in 90 days, effectively dead after 270 days overdue.

### Glacial Decay (τ = 180 days)

**Use for**: Structural trends (freight recession, Buffett indicator, long-term valuations)

| Days Overdue | Freshness | % of Original |
|--------------|-----------|---------------|
| 0 | 1.000 | 100% (still current) |
| 60 | 0.717 | 72% |
| 90 | 0.606 | 61% |
| 180 | 0.368 | 37% |
| 365 | 0.135 | 14% |
| 540 | 0.050 | 5% |
| 730 | 0.018 | 1.8% |

**Interpretation**: Once overdue, signal loses 63% of strength in 180 days, effectively dead after 540 days overdue.

---

## Decay Curve Visualization (Conceptual)

```
Freshness Factor Over Time (After Data Becomes Overdue)
1.0 |━━━━━●╮                       Fast (τ=7d)
    |      ╲                         
0.8 |       ╲                        
    |        ●━━━╮                  Moderate (τ=30d)
0.6 |         ╲  ╲                   
    |          ╲  ●━━━╮             Slow (τ=90d)
0.4 |           ╲  ╲  ╲             
    |            ╲  ╲  ●━━━━╮       Glacial (τ=180d)
0.2 |             ╲  ╲  ╲   ╲        
    |              ╲  ╲  ╲   ╲       
0.0 |_______________╲__╲__╲___╲____
    0   30   60   90  120 150 180  Days Overdue
```

**Critical Note**: The x-axis represents "Days Overdue" not "Days Since Publication". Data remains at 100% freshness until it becomes overdue.

---

## Temporal Multipliers

### Regime Transition Detection

```csharp
bool IsRegimeTransitioning()
{
    var macroScoreTrend = _macroScoreHistory.GetTrend(days: 7);
    return Math.Abs(macroScoreTrend) > 0.5m;
}
```

**Threshold**: 7-day macro score change > ±0.5

### Multiplier Application

```csharp
decimal GetTemporalMultiplier(PatternConfiguration pattern, bool isTransitioning)
{
    if (!isTransitioning)
        return 1.0m;  // Stable regime: no adjustment

    return pattern.TemporalType switch
    {
        TemporalType.Leading => 1.3m,      // +30% boost
        TemporalType.Coincident => 1.0m,   // Neutral
        TemporalType.Lagging => 0.7m,      // -30% penalty
        _ => 1.0m
    };
}
```

### Multiplier Matrix

| Temporal Type | Stable Regime | Transitioning Regime | Rationale |
|---------------|---------------|----------------------|-----------|
| **Leading** | 1.0× | **1.3×** | Boost early warnings during regime changes |
| **Coincident** | 1.0× | 1.0× | Real-time confirmation, always valued equally |
| **Lagging** | 1.0× | **0.7×** | Reduce false confirmations, wait for coincident |

---

## Combined Weighting Example

### Scenario: ISM Manufacturing (monthly indicator)

**Pattern Config**:
```json
{
  "patternId": "ism-contraction",
  "weight": 0.85,
  "temporalType": "Coincident",
  "publicationFrequencyDays": 30,
  "signalDecayDays": 30,
  "rawSignal": -1.0
}
```

**Case 1: Data is current (15 days old)**
```
daysSincePublication = 15
daysOverdue = max(0, 15 - 30) = 0
freshness = 1.0  (100% - still current!)
weighted = -1.0 × 0.85 × 1.0 × 1.0 = -0.85
```

**Case 2: Data is overdue (50 days old)**
```
daysSincePublication = 50
daysOverdue = max(0, 50 - 30) = 20
freshness = e^(-20/30) = 0.513  (51% - stale)
weighted = -1.0 × 0.85 × 0.513 × 1.0 = -0.436
```

**Impact**: Current data maintains full strength; only overdue data decays.

### Scenario: GDP (quarterly indicator)

**Pattern Config**:
```json
{
  "patternId": "gdp-acceleration",
  "weight": 0.85,
  "temporalType": "Coincident",
  "publicationFrequencyDays": 90,
  "signalDecayDays": 90,
  "rawSignal": +2.0
}
```

**Case 1: Data is current (45 days old)**
```
daysSincePublication = 45
daysOverdue = max(0, 45 - 90) = 0
freshness = 1.0  (100% - still current, next release in 45 days)
weighted = +2.0 × 0.85 × 1.0 × 1.0 = +1.70
```

**Case 2: Data is slightly overdue (110 days old)**
```
daysSincePublication = 110
daysOverdue = max(0, 110 - 90) = 20
freshness = e^(-20/90) = 0.801  (80% - slightly stale)
weighted = +2.0 × 0.85 × 0.801 × 1.0 = +1.36
```

---

## Schema Extension

### New Pattern Configuration Field

```json
{
  "publicationFrequencyDays": {
    "type": "integer",
    "minimum": 0,
    "maximum": 365,
    "default": 30,
    "description": "Expected days between publications. 0 = real-time, 1 = daily, 7 = weekly, 30 = monthly, 90 = quarterly. Decay only applies to overdue data."
  }
}
```

### Example Pattern Configurations

**VIX (Real-Time)**:
```json
{
  "patternId": "vix-deployment-l1",
  "publicationFrequencyDays": 0,
  "signalDecayDays": 7
}
```

**ISM Manufacturing (Monthly)**:
```json
{
  "patternId": "ism-contraction",
  "publicationFrequencyDays": 30,
  "signalDecayDays": 30
}
```

**GDP (Quarterly)**:
```json
{
  "patternId": "gdp-acceleration",
  "publicationFrequencyDays": 90,
  "signalDecayDays": 90
}
```

---

## Category Score Calculation

### Naive (Equal Weight)

```csharp
var categoryScore = patterns
    .Where(p => p.Enabled)
    .Average(p => p.RawSignal);
```

**Example**: 3 patterns firing at -2.0, -1.0, -0.5
```
score = (-2.0 + -1.0 + -0.5) / 3 = -1.17
```

### Weighted (Signal Quality + Freshness)

```csharp
var totalWeight = patterns.Where(p => p.Enabled).Sum(p => p.Weight);
var categoryScore = patterns
    .Where(p => p.Enabled)
    .Sum(p => GetWeightedSignal(p)) / totalWeight;
```

**Example**: Same patterns with different weights and freshness

| Pattern | Raw Signal | Weight | Publication Freq | Days Since | Freshness | Weighted Signal |
|---------|------------|--------|------------------|------------|-----------|-----------------|
| Sahm Rule | -2.0 | 0.95 | 30 | 15 | 1.0 | -1.90 |
| ISM Contraction | -1.0 | 0.85 | 30 | 15 | 1.0 | -0.85 |
| Consumer Sentiment | -0.5 | 0.50 | 30 | 45 | 0.606 | -0.15 |

```
totalWeight = 0.95 + 0.85 + 0.50 = 2.30
weightedSum = -1.90 + -0.85 + -0.15 = -2.90
score = -2.90 / 2.30 = -1.26
```

**Comparison**:
- Naive: -1.17
- Weighted (with freshness): -1.26
- **Impact**: High-quality current signals dominate, stale low-quality signals diminished

---

## Macro Score Calculation

### Formula

```
MacroScore = Σ (CategoryScore[i] × CategoryWeight[i])
             i ∈ categories
```

Where:
- `CategoryScore[i]` = weighted average of patterns in category (includes freshness)
- `CategoryWeight[i]` = strategic allocation (Recession=30%, Liquidity=20%, etc.)

### Implementation

```csharp
decimal CalculateMacroScore()
{
    var scores = new Dictionary<PatternCategory, decimal>();
    
    foreach (var category in Enum.GetValues<PatternCategory>())
    {
        var patterns = _patterns
            .Where(p => p.Category == category && p.Enabled)
            .ToList();
        
        if (patterns.Count == 0)
        {
            scores[category] = 0m;
            continue;
        }
        
        var totalWeight = patterns.Sum(p => p.Weight);
        var weightedSum = patterns.Sum(p => GetWeightedSignal(p));
        
        scores[category] = weightedSum / totalWeight;
    }
    
    return scores[PatternCategory.Recession] * 0.30m +
           scores[PatternCategory.Liquidity] * 0.20m +
           scores[PatternCategory.Growth] * 0.20m +
           scores[PatternCategory.NBFI] * 0.10m +
           scores[PatternCategory.Inflation] * 0.10m +
           scores[PatternCategory.Currency] * 0.10m;
           // Valuation and Commodity weighted at 0% (informational)
}
```

---

## Regime Transition Thresholds

### Current System (Naive)

```
Crisis:     score ≤ -20
Recession:  -20 < score ≤ -10
LateCycle:  -10 < score ≤ 0
Neutral:      0 < score ≤ 10
Recovery:    10 < score ≤ 20
Growth:      20 < score
```

### Impact of Weighting + Freshness

**Hypothesis**: Weighted scoring with freshness will produce slightly more extreme scores than naive averaging.

**Reason**: High-quality signals (0.90-0.95 weight) that are current (100% freshness) dominate, while low-quality stale signals contribute minimally.

**Example**:
- Naive: Mix of strong and weak signals averages to -1.0
- Weighted + Fresh: Strong current signals dominate, weighted average -1.4

**Implication**: May need to recalibrate regime thresholds after deployment.

**Mitigation**: Monitor both weighted and naive scores for 30 days, adjust thresholds if needed.

---

## Edge Cases

### Case 1: No Recent Data

```csharp
if (!lastUpdate.HasValue)
    return 0.0m;  // No data = no signal contribution
```

**Behavior**: Pattern effectively disabled until data arrives.

### Case 2: Very Stale Data (>3× publication frequency + decay period)

```
For monthly indicator (f=30, τ=30), after 120 days:
daysOverdue = 120 - 30 = 90
freshness = e^(-90/30) = e^-3 = 0.05 (5%)
```

**Behavior**: Signal contributes only 5%, effectively noise.

### Case 3: All Patterns Disabled in Category

```csharp
if (patterns.Count == 0)
    scores[category] = 0m;
```

**Behavior**: Category contributes 0 to macro score.

### Case 4: Division by Zero (no enabled patterns)

```csharp
var totalWeight = patterns.Sum(p => p.Weight);
if (totalWeight == 0m)
    return 0m;
```

**Behavior**: Safe fallback, category score = 0.

### Case 5: Real-Time Data (publicationFrequencyDays = 0)

```csharp
daysOverdue = max(0, daysSince - 0) = daysSince
// Decay starts immediately
```

**Behavior**: VIX data from yesterday starts decaying immediately, which is correct for real-time indicators.

---

## Performance Considerations

### Computation Complexity

**Per Pattern Evaluation**:
- Freshness calculation: O(1) - single exponential + max()
- Temporal multiplier: O(1) - switch statement
- Weight multiplication: O(1)

**Total**: O(N) where N = number of enabled patterns (~50-60)

**Execution Time**: <1ms per pattern, <100ms total

### Caching Strategy

```csharp
// Cache freshness factors for observation timestamps + publication frequency
private readonly Dictionary<(string, DateTime, int), decimal> _freshnessCache;

decimal GetCachedFreshness(string seriesId, DateTime lastUpdate, int pubFreq)
{
    var key = (seriesId, lastUpdate, pubFreq);
    if (_freshnessCache.TryGetValue(key, out var cached))
        return cached;
    
    var freshness = CalculateFreshness(lastUpdate, pubFreq);
    _freshnessCache[key] = freshness;
    return freshness;
}
```

**Cache Invalidation**: Clear cache daily or when observation updates.

---

## Validation & Testing

### Unit Test Examples

```csharp
[Theory]
[InlineData(15, 30, 1.000)]     // 15 days old, 30-day cycle: still current
[InlineData(35, 30, 0.856)]     // 5 days overdue: slight decay
[InlineData(60, 30, 0.368)]     // 30 days overdue: significant decay
public void FreshnessDecay_MonthlyPattern_AccountsForPublicationCycle(
    int daysSince, int pubFreq, decimal expected)
{
    var pattern = new Pattern { 
        PublicationFrequencyDays = pubFreq,
        SignalDecayDays = 30 
    };
    var observationDate = DateTime.UtcNow.AddDays(-daysSince);
    
    var freshness = calculator.CalculateFreshnessFactor(pattern, observationDate);
    
    freshness.Should().BeApproximately(expected, 0.01m);
}

[Fact]
public void FreshnessDecay_CurrentMonthlyData_NoDecay()
{
    // ISM published 15 days ago (monthly cycle = 30 days)
    var pattern = new Pattern { 
        PublicationFrequencyDays = 30,
        SignalDecayDays = 30 
    };
    var observationDate = DateTime.UtcNow.AddDays(-15);
    
    var freshness = calculator.CalculateFreshnessFactor(pattern, observationDate);
    
    freshness.Should().Be(1.0m);  // Full strength, not overdue
}

[Fact]
public void FreshnessDecay_RealTimeData_DecaysImmediately()
{
    // VIX from 1 day ago (real-time = 0 publication frequency)
    var pattern = new Pattern { 
        PublicationFrequencyDays = 0,
        SignalDecayDays = 7 
    };
    var observationDate = DateTime.UtcNow.AddDays(-1);
    
    var freshness = calculator.CalculateFreshnessFactor(pattern, observationDate);
    
    // e^(-1/7) = 0.868
    freshness.Should().BeApproximately(0.868m, 0.01m);
}

[Fact]
public void TemporalMultiplier_LeadingDuringTransition_Boosted()
{
    var pattern = new Pattern { TemporalType = TemporalType.Leading };
    var isTransitioning = true;
    
    var multiplier = calculator.GetTemporalMultiplier(pattern, isTransitioning);
    
    multiplier.Should().Be(1.3m);
}
```

---

## References

- Exponential Decay: https://en.wikipedia.org/wiki/Exponential_decay
- Time Constants (τ): https://en.wikipedia.org/wiki/Time_constant
- Weighted Averages: https://en.wikipedia.org/wiki/Weighted_arithmetic_mean
- Implementation Spec: [pattern-weighting-temporal-metadata.md](./pattern-weighting-temporal-metadata.md)
- Weight Assignment Guide: [pattern-weight-assignment-guide.md](./pattern-weight-assignment-guide.md)
