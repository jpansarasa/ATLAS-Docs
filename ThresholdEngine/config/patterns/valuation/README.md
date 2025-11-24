# Valuation Patterns

Patterns detecting market valuation levels for deployment opportunities.

## Patterns

| Pattern ID | Name | Trigger | Signal Range | Status |
|------------|------|---------|--------------|--------|
| buffett-indicator | Broad Market Valuation | NASDAQNQUSBLM >20% above MA | -2 to +2 | Enabled |
| cape-attractive | CAPE Below Historical Median | CAPE <20 | +1 to +2 | Disabled |
| equal-weight-indicator | Equal Weight Valuation | NASDAQNDXSE >15% above MA | -2 to +2 | Enabled |
| equity-risk-premium | High Equity Risk Premium | ERP >5% | +1 to +2 | Disabled |
| forward-pe-value | Forward P/E Value | Fwd P/E <15x | +1 to +2 | Disabled |

## Active Patterns

### Buffett Indicator (Market Cap Proxy)
- **Series**: NASDAQNQUSBLM (Large Mid Cap Index)
- **Logic**: Compares current level to 252-day MA
- **Overvalued**: >20% above MA → -1 to -2
- **Undervalued**: <20% below MA → +1 to +2

### Equal Weight Indicator
- **Series**: NASDAQNDXSE (Nasdaq-100 Equal Weight)
- **Purpose**: Less MAG7 concentration than cap-weighted
- **Logic**: Compares current level to 60-day MA

## Disabled Patterns (Data Not in FRED)

These patterns require external data sources:
- **CAPE**: Shiller data from Yale
- **ERP**: Earnings estimates from external providers
- **Forward P/E**: Analyst estimates

## FRED Series Used

- `NASDAQNQUSBLM` - NASDAQ US Large Mid Cap Index
- `NASDAQNDXSE` - Nasdaq-100 Equal Weighted Index

## Applicable Regimes

Valuation patterns apply to: Neutral, LateCycle, Growth, Recession
