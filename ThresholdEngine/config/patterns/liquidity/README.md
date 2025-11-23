# Liquidity Patterns

Patterns monitoring market liquidity conditions and volatility signals.

## Patterns

| Pattern ID | Name | Trigger | Signal Range |
|------------|------|---------|--------------|
| vix-deployment-l1 | VIX Level 1 Deployment | VIX >22 | Context-dependent |
| vix-deployment-l2 | VIX Level 2 Deployment | VIX >30 | Context-dependent |
| credit-spread-widening | HY Credit Spread Widening | HY spreads >350bps | -1 to -2 |
| dxy-risk-off | Dollar Strength Risk-Off | DXY >110 | -1 to -2 |
| fed-liquidity-contraction | Fed Balance Sheet Contraction | WALCL <-5% YoY | -0.5 to -1.5 |

## VIX Context-Dependent Logic

VIX patterns are unique in that their signal depends on the current macro regime:

| Regime | VIX >22 Signal | VIX >30 Signal | Interpretation |
|--------|----------------|----------------|----------------|
| Crisis/Recession | +1 to +1.5 | +1.5 to +2 | Buy opportunity |
| Growth | -1 | -1.5 to -2 | Warning signal |
| Late Cycle/Neutral | -0.5 | -1 to -1.5 | Moderate defensive |

## FRED Series Used

- `VIXCLS` - CBOE Volatility Index (VIX)
- `BAMLH0A0HYM2` - ICE BofA High Yield Master II Option-Adjusted Spread
- `DTWEXBGS` - Trade Weighted U.S. Dollar Index (Broad)
- `WALCL` - Federal Reserve Total Assets

## Deployment Amounts

- **L1 (VIX >22)**: $25-100K deployment
- **L2 (VIX >30)**: $150-500K deployment
