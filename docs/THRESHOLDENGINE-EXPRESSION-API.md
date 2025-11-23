# Pattern Expression API Reference

**Status**: ✅ Implemented
**Last Updated**: 2025-11-21

Pattern expressions are C# code snippets compiled at runtime using Roslyn. They have access to a `PatternEvaluationContext` object (`ctx`) and can use standard .NET libraries.

## PatternEvaluationContext API

### Properties

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

## Available Namespaces

Expressions have access to:

| Namespace | Key Types/Functions |
|-----------|---------------------|
| `System` | `decimal`, `bool`, `DateTime`, `Math.*` |
| `System.Math` | `Abs`, `Max`, `Min`, `Round`, `Sqrt`, `Pow`, `Floor`, `Ceiling` |
| `System.Linq` | `Average`, `Sum`, `Count`, `Where`, `Select`, `Any`, `All` |
| `System.Collections.Generic` | `List<T>`, `Dictionary<K,V>` |
| `ThresholdEngine.Core.Entities` | `PatternEvaluationContext` |
| `ThresholdEngine.Core.Enums` | `MacroRegime` |

## Expression Types

### Condition Expression
Returns `bool` - determines if pattern triggers.

```csharp
// Simple threshold
ctx.GetLatest("VIXCLS") > 22m

// Null-safe with HasValue
var cape = ctx.GetLatest("CAPE");
return cape.HasValue && cape.Value < 20m;
```

### Signal Expression
Returns `decimal` in range [-2, +2] - signal strength and direction.

| Signal | Interpretation |
|--------|----------------|
| +2.0 | Strong offensive (deploy cash, increase equity) |
| +1.0 | Moderate offensive (gradual risk increase) |
| 0.0 | Neutral (maintain allocation) |
| -1.0 | Moderate defensive (gradual risk reduction) |
| -2.0 | Strong defensive (raise cash, reduce equity) |

## Real Expression Examples

### VIX Deployment (Regime-Aware)

**Condition:**
```csharp
ctx.GetLatest("VIXCLS") > 22m
```

**Signal:**
```csharp
var vix = ctx.GetLatest("VIXCLS") ?? 0m;
var score = ctx.MacroScore;
if (vix <= 22m) return 0m;
if (score < -10m) return vix > 30m ? 2m : 1m;
if (score > 10m) return vix > 30m ? -2m : -1m;
return vix > 30m ? -1m : -0.5m;
```

### Sahm Rule (Moving Average)

**Condition:**
```csharp
var current = ctx.GetMA("UNRATE", 90);
var low12mo = ctx.GetLowest("UNRATE", 365);
return (current - low12mo) >= 0.5m;
```

**Signal:**
```csharp
var current = ctx.GetMA("UNRATE", 90) ?? 0m;
var low12mo = ctx.GetLowest("UNRATE", 365) ?? 0m;
var diff = current - low12mo;
return diff >= 0.5m ? -2m : 0m;
```

### Credit Spread Widening (Tiered)

**Condition:**
```csharp
var spread = ctx.GetLatest("BAMLH0A0HYM2");
return spread.HasValue && spread.Value > 350m;
```

**Signal:**
```csharp
var spread = ctx.GetLatest("BAMLH0A0HYM2") ?? 0m;
if (spread <= 350m) return 0m;
if (spread > 500m) return -2m;
if (spread > 400m) return -1.5m;
return -1m;
```

### Fed Liquidity (Year-over-Year)

**Condition:**
```csharp
var yoy = ctx.GetYoY("WALCL");
return yoy.HasValue && yoy.Value < -5m;
```

**Signal:**
```csharp
var yoy = ctx.GetYoY("WALCL") ?? 0m;
if (yoy >= -5m) return 0m;
if (yoy < -15m) return -1.5m;
if (yoy < -10m) return -1m;
return -0.5m;
```

### Yield Curve Inversion (Spread)

**Condition:**
```csharp
var spread = ctx.GetSpread("DGS10", "DGS2");
return spread.HasValue && spread.Value < 0m;
```

### Industrial Production (Moving Average Trend)

**Condition:**
```csharp
var ma3mo = ctx.GetMA("INDPRO", 90);
var ma12mo = ctx.GetMA("INDPRO", 365);
return ma3mo.HasValue && ma12mo.HasValue && ma3mo.Value > ma12mo.Value;
```

## Using Math Functions

```csharp
// Absolute value
var change = Math.Abs(ctx.GetMoM("INDPRO") ?? 0m);

// Rounding
var normalized = Math.Round(ctx.MacroScore, 2);

// Min/Max
var floor = Math.Max(ctx.GetLatest("VIXCLS") ?? 0m, 15m);
```

## Null Safety Patterns

All `ctx.Get*` methods return nullable decimals. Handle nulls appropriately:

```csharp
// Null-coalescing operator
var vix = ctx.GetLatest("VIXCLS") ?? 0m;

// HasValue check
var cape = ctx.GetLatest("CAPE");
if (!cape.HasValue) return 0m;
return cape.Value < 20m ? 1m : 0m;

// Null-conditional with fallback
return (ctx.GetLatest("VIXCLS") ?? 0m) > 22m;
```

## Compilation Details

- **Engine**: Roslyn C# compiler
- **Timeout**: 5 seconds
- **Optimization**: Release mode
- **Caching**: Compiled delegates cached by pattern ID
- **Error Handling**: Compilation errors returned with diagnostics

## Not Yet Implemented

The following statistical functions are planned but not yet available:

| Function | Description | Status |
|----------|-------------|--------|
| `GetStdDev(seriesId, days)` | Standard deviation | ◯ Planned |
| `GetPercentile(seriesId, days, pct)` | Percentile value | ◯ Planned |
| `GetZScore(seriesId, days)` | Z-score normalization | ◯ Planned |
| `GetCorrelation(series1, series2, days)` | Correlation coefficient | ◯ Planned |

For now, basic statistics can be approximated using `GetMA`, `GetLowest`, `GetHighest`.
