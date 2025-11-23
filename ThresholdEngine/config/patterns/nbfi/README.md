# NBFI (Non-Bank Financial Institution) Patterns

Patterns monitoring Non-Bank Financial Institution stress and systemic risk escalation.

## Patterns

| Pattern ID | Name | Trigger | Signal Range | Status |
|------------|------|---------|--------------|--------|
| shadow-banking-hy-spreads | HY Spread Stress | HY >350bps | -1 to -2 | Enabled |
| standing-repo-stress | Repo Facility Stress | Repo >$50B | -1 to -2 | Enabled |
| kre-underperformance | Regional Bank Underperformance | KRE -10% vs SPX | -1 to -2 | Disabled |
| bankruptcy-clusters | Large Bankruptcy Clusters | 2+ >$500M | -1 to -2 | Disabled |
| shadow-banking-escalation | Multi-Indicator Escalation | 2+ triggers | -1.5 to -2 | Enabled |

## Escalation Framework

The NBFI escalation system uses 6 key indicators:

1. **HY Spreads >350bps** - Credit market stress
2. **Standing Repo >$50B** - Funding market stress
3. **KRE Underperformance** - Regional banking stress (manual)
4. **Bankruptcy Clusters** - Corporate credit stress (manual)
5. **Commercial Paper Stress** - Short-term funding stress
6. **Additional indicators** - Phase 2

When **2+ indicators** fire simultaneously, the escalation pattern triggers defensive positioning.

## FRED Series Used

- `BAMLH0A0HYM2` - ICE BofA High Yield Spread
- `WLRRAL` - Federal Reserve Reverse Repos (Repo facility proxy)
- `DCPN3M` - 3-Month Commercial Paper Rate

## Manual Input Patterns

Two patterns require manual input as data is not available in FRED:
- **KRE Underperformance**: Monitor SPDR Regional Bank ETF vs S&P 500
- **Bankruptcy Clusters**: Monitor Bloomberg/Reuters for large bankruptcy filings

## Allocation Action

When escalation triggers: Move to 70-75% defensive allocation.
