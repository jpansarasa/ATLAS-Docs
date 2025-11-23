# Recession Patterns

Patterns detecting economic contraction and recession signals.

## Patterns

| Pattern ID | Name | Trigger | Signal Range |
|------------|------|---------|--------------|
| sahm-rule | Sahm Rule Recession Indicator | Unemployment MA rises 0.5pp above 12mo low | -2 |
| ism-contraction | ISM Manufacturing Contraction | ISM <48 sustained 60 days | -1 to -2 |
| initial-claims-spike | Initial Jobless Claims Spike | Claims >300K sustained 30 days | -1 to -2 |
| freight-recession | Freight Transportation Decline | Freight <-3% YoY | -1 to -2 |
| consumer-confidence-collapse | Consumer Sentiment Collapse | UMCSENT <70 | -1 to -2 |

## Pattern Details

### Sahm Rule (High Confidence)
- **Source**: Claudia Sahm, 2019
- **Historical Accuracy**: 100% recession prediction since 1970
- **Trigger**: 3-month unemployment MA exceeds 12-month low by 0.5+ percentage points
- **Signal**: Strong defensive (-2)

### ISM Contraction
- **Source**: Institute for Supply Management
- **Trigger**: ISM Manufacturing <48 for 60+ days
- **Levels**: <48 contraction, <45 severe contraction

### Initial Claims Spike
- **Source**: Department of Labor
- **Trigger**: Weekly claims >300K sustained
- **Levels**: >300K concern, >400K crisis

### Freight Recession
- **Source**: Bureau of Transportation Statistics
- **Leading Indicator**: Freight declines often precede broader recession
- **Trigger**: Transportation Services Index <-3% YoY

### Consumer Confidence
- **Source**: University of Michigan
- **Historical Median**: ~85
- **Trigger**: Sentiment <70

## FRED Series Used

- `UNRATE` - Unemployment Rate
- `NAPM` - ISM Manufacturing PMI
- `ICSA` - Initial Claims
- `TSIFRGHT` - Freight Transportation Index
- `UMCSENT` - Consumer Sentiment Index

## Applicable Regimes

All recession patterns apply to: LateCycle, Recession, Crisis
