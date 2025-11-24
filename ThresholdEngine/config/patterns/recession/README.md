# Recession Patterns

Patterns detecting economic contraction and recession signals.

## Patterns

| Pattern ID | Name | Trigger | Signal Range |
|------------|------|---------|--------------|
| consumer-confidence-collapse | Consumer Sentiment Collapse | UMCSENT <70 | -1 to -2 |
| continuing-claims-warning | Continuing Claims Escalation | CCSA >2M | -1.5 to -2 |
| freight-recession | Freight Transportation Decline | TSIFRGHT <-3% YoY | -1.5 to -2 |
| industrial-production-contraction | Industrial Production Contraction | INDPRO <-1% YoY | -1 to -2 |
| initial-claims-spike | Initial Jobless Claims Spike | ICSA >300K sustained | -1 to -2 |
| jobs-contraction | Nonfarm Payrolls Contraction | PAYEMS MoM <0 | -2 |
| sahm-rule | Sahm Rule Recession Indicator | Unemployment +0.5pp above 12mo low | -2 |
| yield-curve-inversion | Yield Curve Inversion | T10Y2Y <0 | -1.5 to -2 |

## Key Patterns

### Sahm Rule (High Confidence)
- **Historical Accuracy**: 100% recession prediction since 1970
- **Trigger**: 3-month unemployment MA exceeds 12-month low by 0.5+ percentage points

### Yield Curve Inversion
- **Leading Indicator**: Typically precedes recession by 12-18 months
- **Trigger**: 10Y-2Y Treasury spread turns negative

## FRED Series Used

- `CCSA` - Continuing Claims
- `ICSA` - Initial Claims
- `INDPRO` - Industrial Production Index
- `PAYEMS` - Nonfarm Payrolls
- `T10Y2Y` - 10Y-2Y Treasury Spread
- `TSIFRGHT` - Freight Transportation Index
- `UMCSENT` - Consumer Sentiment Index
- `UNRATE` - Unemployment Rate

## Applicable Regimes

Recession patterns apply to: Neutral, LateCycle, Recession, Crisis
