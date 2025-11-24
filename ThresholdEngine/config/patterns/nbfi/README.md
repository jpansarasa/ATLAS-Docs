# NBFI (Non-Bank Financial Institution) Patterns

Patterns monitoring financial conditions, systemic stress, and NBFI risk escalation.

## Patterns

| Pattern ID | Name | Trigger | Signal Range | Status |
|------------|------|---------|--------------|--------|
| chicago-conditions-index | Chicago Fed NFCI | NFCI >0.5 | -0.5 to -2 | Enabled |
| nbfi-escalation | Multi-Indicator Escalation | 2+ triggers | -1.5 to -2 | Enabled |
| nbfi-hy-spreads | HY Spread Stress | HY >350bps | -1 to -2 | Enabled |
| reverse-repo-liquidity | Reverse Repo Utilization | RRP >$2T | -1 to +1.5 | Enabled |
| standing-repo-stress | Standing Repo Facility Stress | SRF >$5B | -1.5 to -2 | Enabled |
| stlouis-stress-index | St. Louis Financial Stress | STLFSI >1.0 | -1 to -2 | Enabled |
| bankruptcy-clusters | Large Bankruptcy Clusters | 2+ >$500M | -1 to -2 | Disabled (manual) |
| kre-underperformance | Regional Bank Underperformance | KRE -10% vs SPX | -1 to -2 | Disabled (manual) |

## FRED Series Used

- `NFCI` - Chicago Fed National Financial Conditions Index
- `BAMLH0A0HYM2` - ICE BofA High Yield Spread
- `RRPONTSYD` - Overnight Reverse Repo (ON RRP)
- `RPONTSYD` - Standing Repo Facility
- `STLFSI4` - St. Louis Financial Stress Index
- `WLRRAL` - Fed Reverse Repos
- `DCPN3M` - 3-Month Commercial Paper Rate

## Escalation Framework

The NBFI escalation pattern triggers when 2+ indicators fire simultaneously:
1. HY Spreads >350bps
2. Standing Repo >$50B
3. Commercial Paper stress

## Applicable Regimes

All NBFI patterns apply to: Neutral, LateCycle, Recession, Crisis
