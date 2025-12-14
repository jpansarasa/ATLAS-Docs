# Statistical Libraries for Pattern Expressions

**Status**: ✅ Core Implementation Complete
**Last Updated**: 2025-12-14

## Current Implementation

### Available Now (PatternEvaluationContext)

| Function | Signature | Description |
|----------|-----------|-------------|
| Moving Average | `GetMA(seriesId, days)` | Simple moving average over period |
| YoY Change | `GetYoY(seriesId)` | Year-over-year percentage change |
| MoM Change | `GetMoM(seriesId)` | Month-over-month percentage change |
| Minimum | `GetLowest(seriesId, days)` | Lowest value in period |
| Maximum | `GetHighest(seriesId, days)` | Highest value in period |
| Ratio | `GetRatio(num, denom)` | Ratio of two series |
| Spread | `GetSpread(s1, s2)` | Difference between two series |
| Sustained | `IsSustained(seriesId, fn, days)` | Check condition over window |

### Available via System.Math

All standard math functions are available in expressions:

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

### Available via System.Linq

Basic aggregations can be performed on collections:

```csharp
values.Average()          // Mean
values.Sum()              // Sum
values.Count()            // Count
values.Min()              // Minimum
values.Max()              // Maximum
values.Any(predicate)     // Any match
values.All(predicate)     // All match
```

## Not Yet Implemented

The following advanced statistical functions are planned for future implementation:

| Function | Signature | Description | Priority |
|----------|-----------|-------------|----------|
| Standard Deviation | `GetStdDev(seriesId, days)` | Population std dev | Medium |
| Percentile | `GetPercentile(seriesId, days, pct)` | Percentile value | Medium |
| Z-Score | `GetZScore(seriesId, days)` | Standard score | Low |
| Correlation | `GetCorrelation(s1, s2, days)` | Pearson correlation | Low |
| Rolling StdDev | `GetRollingStdDev(seriesId, window, days)` | Rolling volatility | Low |

### Why Not Implemented Yet

1. **Current patterns don't require them** - All 24 production patterns work with existing functions
2. **Complexity** - Require proper statistical calculations and testing
3. **Performance** - Need to ensure efficient time-series queries

## Workarounds for Missing Functions

### Approximating Volatility

Without `GetStdDev`, use range-based volatility:

```csharp
var high = ctx.GetHighest("VIXCLS", 30) ?? 0m;
var low = ctx.GetLowest("VIXCLS", 30) ?? 0m;
var range = high - low;  // Simple volatility proxy
```

### Approximating Z-Score

Without `GetZScore`, compare to moving average:

```csharp
var current = ctx.GetLatest("SERIES") ?? 0m;
var ma = ctx.GetMA("SERIES", 365) ?? current;
var deviation = current - ma;  // Distance from mean
```

### Approximating Percentile

Without `GetPercentile`, use high/low comparison:

```csharp
var current = ctx.GetLatest("CAPE") ?? 0m;
var high = ctx.GetHighest("CAPE", 3650) ?? current;  // 10-year high
var low = ctx.GetLowest("CAPE", 3650) ?? current;    // 10-year low
var percentile = (current - low) / (high - low);     // Approximate percentile
```

## External Library Consideration

### MathNet.Numerics

If advanced statistics become necessary, MathNet.Numerics could be added:

**Pros:**
- Comprehensive statistical library
- Well-maintained, MIT licensed
- No file/network I/O (safe for sandboxing)

**Cons:**
- Adds ~2MB dependency
- May be overkill for most patterns
- Requires assembly registration in Roslyn options

**Current Decision:** Not needed. Built-in helpers sufficient for production patterns.

## Implementation Roadmap

### Phase 1: Core Functions ✅ Complete
- `GetMA`, `GetYoY`, `GetMoM`
- `GetLowest`, `GetHighest`
- `GetRatio`, `GetSpread`
- `IsSustained`

### Phase 2: Advanced Statistics (Future)
- `GetStdDev` - Standard deviation
- `GetPercentile` - Percentile calculations
- `GetZScore` - Z-score normalization

### Phase 3: Correlation Analysis (Future)
- `GetCorrelation` - Cross-series correlation
- `GetBeta` - Beta coefficient
- `GetCovariance` - Covariance

## Pattern Examples Using Current Functions

### VIX Volatility Detection (Range-Based)

```csharp
var current = ctx.GetLatest("VIXCLS") ?? 0m;
var ma30 = ctx.GetMA("VIXCLS", 30) ?? current;
var high30 = ctx.GetHighest("VIXCLS", 30) ?? current;

// Spike detection: current > MA and approaching recent high
return current > ma30 * 1.2m && current > high30 * 0.9m;
```

### Trend Strength (MA Crossover)

```csharp
var ma50 = ctx.GetMA("INDPRO", 50) ?? 0m;
var ma200 = ctx.GetMA("INDPRO", 200) ?? 0m;

// Golden cross: short MA above long MA
return ma50 > ma200 ? 1m : -1m;
```

### Mean Reversion (Distance from MA)

```csharp
var current = ctx.GetLatest("BAMLH0A0HYM2") ?? 0m;
var ma90 = ctx.GetMA("BAMLH0A0HYM2", 90) ?? current;
var deviation = (current - ma90) / ma90 * 100m;

// Signal based on deviation from mean
if (deviation > 20m) return -2m;  // Far above mean
if (deviation < -20m) return 2m;   // Far below mean
return 0m;
```

## Summary

The current PatternEvaluationContext API provides sufficient functionality for all production patterns. Advanced statistical functions (StdDev, Percentile, ZScore, Correlation) are deferred until a concrete use case requires them. The existing functions can approximate most statistical concepts through careful expression design.
