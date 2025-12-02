# Inflation Patterns

Patterns monitoring inflation indicators and expectations for defensive positioning.

## Patterns

| Pattern ID | Name | Trigger | Signal Range |
|------------|------|---------|--------------|
| cpi-acceleration | CPI Inflation Acceleration | YoY CPI >4% | -1 to -2 |
| core-cpi-sticky | Core CPI Sticky Inflation | Core CPI >3.5% | -1 to -2 |
| pce-above-target | PCE Above Fed Target | Core PCE >2.5% | -0.5 to -2 |
| pce-acceleration | PCE Inflation Acceleration | Headline PCE >3.5% | -1 to -2 |
| breakeven-5y-elevated | 5-Year Breakeven Elevated | T5YIE >2.8% | -1 to -2 |
| breakeven-10y-elevated | 10-Year Breakeven Elevated | T10YIE >2.6% | -1 to -2 |
| breakeven-5y5y-forward | 5Y5Y Forward Breakeven | T5YIFR >2.5% | -1 to -2 |

## Signal Interpretation

All inflation patterns generate **negative signals** as elevated inflation typically:
- Erodes purchasing power
- Leads to Fed rate hikes
- Compresses equity valuations
- Increases volatility

## FRED Series Used

### CPI-Based
- `CPIAUCSL_PC1` - Consumer Price Index YoY % Change
- `CPILFESL_PC1` - Core CPI (Less Food and Energy) YoY % Change

### PCE-Based
- `PCEPI_PC1` - Personal Consumption Expenditures Price Index YoY % Change
- `PCEPILFE_PC1` - Core PCE (Less Food and Energy) YoY % Change

### Breakeven Inflation (TIPS-derived)
- `T5YIE` - 5-Year Breakeven Inflation Rate
- `T10YIE` - 10-Year Breakeven Inflation Rate
- `T5YIFR` - 5-Year, 5-Year Forward Inflation Expectation Rate

## Threshold Logic

| Severity | CPI/PCE | Breakevens | Interpretation |
|----------|---------|------------|----------------|
| Warning | >4% / >3.5% | >2.6-2.8% | Moderately defensive |
| Elevated | >5% / >4% | >2.8-3% | Strongly defensive |
| Severe | >6% / >5% | >3-3.5% | Maximum defensive |
