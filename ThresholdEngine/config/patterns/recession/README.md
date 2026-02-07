# Recession Patterns

Patterns detecting economic contraction and recession signals.

## Patterns

| Pattern ID | Name | Trigger | Signal Range |
|------------|------|---------|--------------|
| beveridge-curve-divergence | Beveridge Curve Divergence (Flat Beveridge) | JTSJOL <7.5M + UNRATE <4.5% + V/U <1.2 | -1.5 to -3 |
| stagflation-warning | Stagflation Warning | UNRATE ≥4.5% + Core PCE YoY ≥2.8% | -1.5 to -3 |
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

### Beveridge Curve Divergence (NEW)
- **Leading Indicator**: 6+ months ahead of unemployment spike
- **Trigger**: Job openings <7.5M AND unemployment <4.5% AND V/U ratio <1.2
- **Framework**: Snider/Eurodollar University analysis of "flat Beveridge" phenomenon
- **Interpretation**: When openings decline but unemployment doesn't rise proportionally, it indicates unstable equilibrium. Unemployment typically snaps higher to restore historical Beveridge relationship.

### Stagflation Warning (NEW)
- **Coincident Indicator**: Detects active stagflation conditions
- **Trigger**: Unemployment ≥4.5% AND Core PCE YoY ≥2.8%
- **Interpretation**: Detects simultaneous elevation of BOTH unemployment AND inflation. Fed faces impossible tradeoff — cutting rates risks re-igniting inflation, holding rates risks deeper economic damage.
- **Note**: This is NOT a Phillips Curve model (which failed in the 1970s). Simply detects when stagflationary conditions exist.

### Yield Curve Inversion
- **Leading Indicator**: Typically precedes recession by 12-18 months
- **Trigger**: 10Y-2Y Treasury spread turns negative

## FRED Series Used

- `CCSA` - Continuing Claims
- `JTSJOL` - Job Openings: Total Nonfarm
- `PCEPILFE` - Core PCE Price Index (Fed's preferred inflation measure)
- `ICSA` - Initial Claims
- `INDPRO` - Industrial Production Index
- `PAYEMS` - Nonfarm Payrolls
- `T10Y2Y` - 10Y-2Y Treasury Spread
- `TSIFRGHT` - Freight Transportation Index
- `UMCSENT` - Consumer Sentiment Index
- `UNRATE` - Unemployment Rate

## Applicable Regimes

Recession patterns apply to: Neutral, LateCycle, Recession, Crisis
