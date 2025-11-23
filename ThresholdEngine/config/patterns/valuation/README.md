# Valuation Patterns

Patterns detecting attractive market valuations for deployment opportunities.

**Status**: All patterns DISABLED - data not available in FRED. Scheduled for Phase 2 with calculated indicators.

## Patterns

| Pattern ID | Name | Trigger | Signal Range | Status |
|------------|------|---------|--------------|--------|
| cape-attractive | CAPE Below Historical Median | CAPE <20 | +1 to +2 | Disabled |
| buffett-indicator | Buffett Indicator Attractive | MC/GDP <120% | +1 to +2 | Disabled |
| equity-risk-premium | High Equity Risk Premium | ERP >5% | +1 to +2 | Disabled |
| forward-pe-value | Forward P/E Value | Fwd P/E <15x | +1 to +2 | Disabled |

## Pattern Details

### CAPE Ratio (Shiller P/E)
- **Source**: Robert Shiller, Yale University
- **Historical Median**: ~16.8
- **Trigger**: CAPE <20 (below median)
- **Strong Signal**: CAPE <15

### Buffett Indicator
- **Formula**: Total Market Cap / GDP
- **Historical Median**: ~100%
- **Trigger**: Ratio <120%
- **Strong Signal**: Ratio <80%

### Equity Risk Premium
- **Formula**: Earnings Yield - 10Y Treasury Yield
- **Historical Average**: ~4%
- **Trigger**: ERP >5%
- **Strong Signal**: ERP >7%

### Forward P/E
- **Description**: S&P 500 Forward Price-to-Earnings
- **Historical Average**: ~16x
- **Trigger**: Fwd P/E <15x
- **Strong Signal**: Fwd P/E <12x

## Data Sources (Phase 2)

These patterns require data not available in FRED:
- **CAPE**: Shiller data from Yale website or calculated
- **Market Cap/GDP**: Wilshire 5000 / GDP calculation
- **ERP**: Earnings estimates from external providers
- **Forward P/E**: Analyst estimates from external providers

## Applicable Regimes

All valuation patterns apply to: Crisis, Recession (buy signals)
