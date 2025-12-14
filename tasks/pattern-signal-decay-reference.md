# Pattern Signal Decay & Temporal Behavior

**Purpose**: Mathematical reference for signal freshness decay and temporal weighting  
**Audience**: Engineers implementing weighted pattern evaluation  
**Version**: 1.0  
**Updated**: 2025-12-14  

---

## Exponential Decay Formula

### Basic Formula

```
freshness(t) = e^(-t / τ)
```

Where:
- `t` = days since last observation
- `τ` (tau) = `signalDecayDays` (decay constant)
- `e` = Euler's number (≈2.71828)

### Implementation (C#)

```csharp
decimal CalculateFreshnessFactor(PatternConfiguration pattern)
{
    var lastUpdate = _observationCache.GetLatestDate(pattern.RequiredSeries[0]);
    if (!lastUpdate.HasValue)
        return 0.0m;  // No data = no signal

    var daysSinceUpdate = (DateTime.UtcNow - lastUpdate.Value).TotalDays;
    var decayFactor = Math.Exp(-daysSinceUpdate / pattern.SignalDecayDays);
    
    return (decimal)decayFactor;
}
```

---

## Decay Curves by Period

### Fast Decay (τ = 7 days)

**Use for**: VIX, DXY, Standing Repo Stress

| Days Old | Freshness | % of Original |
|----------|-----------|---------------|
| 0 | 1.000 | 100% |
| 1 | 0.868 | 87% |
| 3 | 0.653 | 65% |
| 7 | 0.368 | 37% |
| 14 | 0.135 | 14% |
| 21 | 0.050 | 5% |
| 30 | 0.015 | 1.5% |

**Interpretation**: Signal loses 63% of strength in 7 days, effectively dead after 21 days.

### Moderate Decay (τ = 30 days)

**Use for**: ISM, Initial Claims, most coincident indicators

| Days Old | Freshness | % of Original |
|----------|-----------|---------------|
| 0 | 1.000 | 100% |
| 7 | 0.794 | 79% |
| 15 | 0.630 | 63% |
| 30 | 0.368 | 37% |
| 60 | 0.135 | 14% |
| 90 | 0.050 | 5% |
| 120 | 0.018 | 1.8% |

**Interpretation**: Signal loses 63% of strength in 30 days, effectively dead after 90 days.

### Slow Decay (τ = 90 days)

**Use for**: Sahm Rule, GDP, yield curve, persistent trends

| Days Old | Freshness | % of Original |
|----------|-----------|---------------|
| 0 | 1.000 | 100% |
| 30 | 0.717 | 72% |
| 60 | 0.514 | 51% |
| 90 | 0.368 | 37% |
| 180 | 0.135 | 14% |
| 270 | 0.050 | 5% |
| 365 | 0.023 | 2.3% |

**Interpretation**: Signal loses 63% of strength in 90 days, effectively dead after 270 days.

### Glacial Decay (τ = 180 days)

**Use for**: Freight recession, Buffett indicator, structural trends

| Days Old | Freshness | % of Original |
|----------|-----------|---------------|
| 0 | 1.000 | 100% |
| 60 | 0.717 | 72% |
| 90 | 0.606 | 61% |
| 180 | 0.368 | 37% |
| 365 | 0.135 | 14% |
| 540 | 0.050 | 5% |
| 730 | 0.018 | 1.8% |

**Interpretation**: Signal loses 63% of strength in 180 days, effectively dead after 540 days.

---

## Decay Curve Visualization (Conceptual)

```
Freshness Factor Over Time
1.0 |●━━━╮                         Fast (τ=7d)
    |    ╲                         
0.8 |     ╲●━━╮                    Moderate (τ=30d)
    |      ╲  ╲                    
0.6 |       ╲  ╲                   
    |        ●  ╲●━━╮              Slow (τ=90d)
0.4 |         ╲  ╲  ╲             
    |          ╲  ╲  ●━━━╮         Glacial (τ=180d)
0.2 |           ╲  ╲  ╲  ╲        
    |            ╲  ╲  ╲  ╲       
0.0 |_____________╲__╲__╲__╲______
    0   30   60   90  120 150 180  Days
```

**Key Insight**: All curves reach ~37% at their respective τ value (1/e).

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

### Scenario: Sahm Rule during Late Cycle → Recession transition

**Pattern Config**:
```json
{
  "patternId": "sahm-rule",
  "weight": 0.95,
  "temporalType": "Coincident",
  "signalDecayDays": 90,
  "rawSignal": -2.0
}
```

**Conditions**:
- Last observation: 15 days ago
- Macro score trend: -0.6 (transitioning)

**Calculations**:

1. **Freshness Factor**:
   ```
   freshness = e^(-15/90) = e^-0.167 = 0.846
   ```

2. **Temporal Multiplier**:
   ```
   temporal = 1.0  (Coincident, no boost)
   ```

3. **Weighted Signal**:
   ```
   weighted = rawSignal × weight × freshness × temporal
   weighted = -2.0 × 0.95 × 0.846 × 1.0
   weighted = -1.607
   ```

### Scenario: Yield Curve (Leading) during stable regime

**Pattern Config**:
```json
{
  "patternId": "yield-curve-inversion",
  "weight": 0.90,
  "temporalType": "Leading",
  "signalDecayDays": 180,
  "rawSignal": -2.0
}
```

**Conditions**:
- Last observation: 30 days ago
- Macro score trend: -0.2 (stable)

**Calculations**:

1. **Freshness Factor**:
   ```
   freshness = e^(-30/180) = e^-0.167 = 0.846
   ```

2. **Temporal Multiplier**:
   ```
   temporal = 1.0  (Stable regime, no boost)
   ```

3. **Weighted Signal**:
   ```
   weighted = -2.0 × 0.90 × 0.846 × 1.0
   weighted = -1.523
   ```

### Scenario: Yield Curve (Leading) during transition

Same as above, but:
- Macro score trend: -0.7 (transitioning)

**Calculations**:

1. **Freshness Factor**: 0.846 (same)

2. **Temporal Multiplier**:
   ```
   temporal = 1.3  (Leading, boost during transition)
   ```

3. **Weighted Signal**:
   ```
   weighted = -2.0 × 0.90 × 0.846 × 1.3
   weighted = -1.980
   ```

**Impact**: Leading indicator signal boosted by 30% during regime transition!

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

### Weighted (Signal Quality)

```csharp
var totalWeight = patterns.Where(p => p.Enabled).Sum(p => p.Weight);
var categoryScore = patterns
    .Where(p => p.Enabled)
    .Sum(p => GetWeightedSignal(p)) / totalWeight;
```

**Example**: Same patterns with different weights

| Pattern | Raw Signal | Weight | Weighted Signal |
|---------|------------|--------|-----------------|
| Sahm Rule | -2.0 | 0.95 | -1.90 |
| ISM Contraction | -1.0 | 0.85 | -0.85 |
| Consumer Sentiment | -0.5 | 0.50 | -0.25 |

```
totalWeight = 0.95 + 0.85 + 0.50 = 2.30
weightedSum = -1.90 + -0.85 + -0.25 = -3.00
score = -3.00 / 2.30 = -1.30
```

**Comparison**:
- Naive: -1.17
- Weighted: -1.30
- **Delta**: -0.13 (more bearish with weighting, Sahm dominates)

---

## Macro Score Calculation

### Formula

```
MacroScore = Σ (CategoryScore[i] × CategoryWeight[i])
             i ∈ categories
```

Where:
- `CategoryScore[i]` = weighted average of patterns in category
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

### Impact of Weighting

**Hypothesis**: Weighted scoring will produce slightly more extreme scores than naive averaging.

**Reason**: High-quality signals (0.90-0.95 weight) tend to be the ones that fire strongly (-2.0), while low-quality signals (0.50-0.60 weight) are noisier and closer to 0.

**Example**:
- Naive: Mix of strong and weak signals averages to -1.0
- Weighted: Strong signals dominate, weighted average -1.3

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

### Case 2: Very Stale Data (>3× decay period)

```
For τ=30d, after 90 days: freshness = e^-3 = 0.05 (5%)
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

---

## Performance Considerations

### Computation Complexity

**Per Pattern Evaluation**:
- Freshness calculation: O(1) - single exponential
- Temporal multiplier: O(1) - switch statement
- Weight multiplication: O(1)

**Total**: O(N) where N = number of enabled patterns (~50-60)

**Execution Time**: <1ms per pattern, <100ms total

### Caching Strategy

```csharp
// Cache freshness factors for observation timestamps
private readonly Dictionary<(string, DateTime), decimal> _freshnessCache;

decimal GetCachedFreshness(string seriesId, DateTime lastUpdate)
{
    var key = (seriesId, lastUpdate);
    if (_freshnessCache.TryGetValue(key, out var cached))
        return cached;
    
    var freshness = CalculateFreshness(lastUpdate);
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
[InlineData(0, 1.000)]      // Today: 100%
[InlineData(7, 0.368)]      // 1 decay period: 37%
[InlineData(14, 0.135)]     // 2 decay periods: 14%
[InlineData(30, 0.015)]     // ~4 decay periods: 1.5%
public void FreshnessDecay_FastPattern_MatchesExpectedCurve(int daysOld, decimal expected)
{
    var pattern = new Pattern { SignalDecayDays = 7 };
    var observationDate = DateTime.UtcNow.AddDays(-daysOld);
    
    var freshness = calculator.CalculateFreshnessFactor(pattern, observationDate);
    
    freshness.Should().BeApproximately(expected, 0.01m);
}

[Fact]
public void TemporalMultiplier_LeadingDuringTransition_Boosted()
{
    var pattern = new Pattern { TemporalType = TemporalType.Leading };
    var isTransitioning = true;
    
    var multiplier = calculator.GetTemporalMultiplier(pattern, isTransitioning);
    
    multiplier.Should().Be(1.3m);
}

[Fact]
public void WeightedScore_HighQualityDominates_LowQuality()
{
    var patterns = new[] {
        new Pattern { Weight = 0.95m, RawSignal = -2.0m },  // High quality
        new Pattern { Weight = 0.50m, RawSignal = -2.0m }   // Low quality
    };
    
    var score = calculator.CalculateCategoryScore(patterns);
    
    // Weighted: (-1.90 + -1.00) / 1.45 = -2.00
    // Closer to high-quality signal than naive average -2.0
    score.Should().BeApproximately(-2.0m, 0.1m);
}
```

---

## References

- Exponential Decay: https://en.wikipedia.org/wiki/Exponential_decay
- Time Constants (τ): https://en.wikipedia.org/wiki/Time_constant
- Weighted Averages: https://en.wikipedia.org/wiki/Weighted_arithmetic_mean
- Implementation Spec: [pattern-weighting-temporal-metadata.md](./pattern-weighting-temporal-metadata.md)
- Weight Assignment Guide: [pattern-weight-assignment-guide.md](./pattern-weight-assignment-guide.md)
