# FRED Data Research

Research on FRED series availability for ATLAS indicators.

## Summary

**66 FRED series** configured in FredCollector, supporting **38 patterns** (33 enabled, 5 disabled) in ThresholdEngine.

## Series by Category

### Recession Indicators
| Series | Description | Used By Pattern |
|--------|-------------|-----------------|
| ICSA | Initial Claims | initial-claims-spike |
| CCSA | Continuing Claims | continuing-claims-warning |
| UNRATE | Unemployment Rate | sahm-rule |
| UMCSENT | Consumer Sentiment | consumer-confidence-collapse |
| T10Y2Y | 10Y-2Y Spread | yield-curve-inversion |
| PAYEMS | Nonfarm Payrolls | jobs-contraction |
| TSIFRGHT | Freight Index | freight-recession |
| INDPRO | Industrial Production | industrial-production-contraction |

### Liquidity Indicators
| Series | Description | Used By Pattern |
|--------|-------------|-----------------|
| VIXCLS | VIX Index | vix-deployment-l1, vix-deployment-l2 |
| BAMLH0A0HYM2 | HY Credit Spread | credit-spread-widening, nbfi-hy-spreads |
| DTWEXBGS | Dollar Index | dxy-risk-off |
| WALCL | Fed Total Assets | fed-liquidity-contraction |

### Growth Indicators
| Series | Description | Used By Pattern |
|--------|-------------|-----------------|
| GDP | Gross Domestic Product | gdp-acceleration |
| INDPRO | Industrial Production | industrial-production, industrial-production-expansion |
| RSAFS | Retail Sales | retail-sales-surge |
| HOUST | Housing Starts | housing-starts |

### NBFI / Financial Stress
| Series | Description | Used By Pattern |
|--------|-------------|-----------------|
| NFCI | Chicago Fed Financial Conditions | chicago-conditions-index |
| STLFSI4 | St. Louis Stress Index | stlouis-stress-index |
| RRPONTSYD | Overnight Reverse Repo | reverse-repo-liquidity |
| RPONTSYD | Standing Repo Facility | standing-repo-stress |
| DCPN3M | Commercial Paper Rate | nbfi-escalation |
| WLRRAL | Fed Reverse Repos | nbfi-escalation |

### Valuation
| Series | Description | Used By Pattern |
|--------|-------------|-----------------|
| NASDAQNQUSBLM | Large Mid Cap Index | buffett-indicator |
| NASDAQNDXSE | Nasdaq-100 Equal Weight | equal-weight-indicator |

### Inflation
| Series | Description | Used By Pattern |
|--------|-------------|-----------------|
| CPIAUCSL | Consumer Price Index | cpi-acceleration |
| CPILFESL | Core CPI | core-cpi-sticky |
| PCEPI | PCE Price Index | pce-acceleration |
| PCEPILFE | Core PCE | pce-above-target |
| T5YIE | 5-Year Breakeven | breakeven-5y-elevated |
| T10YIE | 10-Year Breakeven | breakeven-10y-elevated |
| T5YIFR | 5Y5Y Forward | breakeven-5y5y-forward |

## Series NOT in FRED

These indicators require external data sources (patterns disabled):

| Indicator | Required For | Status |
|-----------|--------------|--------|
| CAPE Ratio | cape-attractive | Disabled - needs Shiller data |
| Forward P/E | forward-pe-value | Disabled - needs analyst estimates |
| Equity Risk Premium | equity-risk-premium | Disabled - needs earnings data |
| KRE vs SPX | kre-underperformance | Disabled - needs ETF data |
| Bankruptcy data | bankruptcy-clusters | Disabled - manual input |

## Pattern Coverage

| Category | Total | Enabled | Disabled |
|----------|-------|---------|----------|
| Recession | 8 | 8 | 0 |
| Liquidity | 5 | 5 | 0 |
| Growth | 5 | 5 | 0 |
| NBFI | 8 | 6 | 2 |
| Valuation | 5 | 2 | 3 |
| Inflation | 7 | 7 | 0 |
| **Total** | **38** | **33** | **5** |

## Future Data Sources

For Phase 2+ when external data is needed:

- **Shiller Data** (Yale): CAPE ratio calculation
- **Analyst Estimates**: Forward P/E, earnings yield
- **ETF Data**: KRE regional bank performance
- **News/RSS**: Sentiment analysis
- **Alternative Data**: Truflation, Indeed job postings

## Verification

Check if a series exists in FRED:
```bash
curl "https://api.stlouisfed.org/fred/series?series_id=VIXCLS&api_key=YOUR_KEY&file_type=json"
```

## See Also

- [Pattern READMEs](../ThresholdEngine/config/patterns/) - Pattern definitions
- [FredCollector](../FredCollector/) - Data collection service
- [FredCollector MCP](../FredCollector/mcp/) - MCP server for FRED data access
