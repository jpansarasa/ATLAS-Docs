# ATLAS Indicators → FRED Series Mapping

**Status**: ✅ Implemented (66 series configured)
**Last Updated**: 2026-01-19

This file maps ATLAS framework indicators to specific FRED series IDs for automated collection.

## Framework Alignment

**Pattern Categories:**
- Recession: 10 patterns
- Liquidity: 8 patterns
- Growth: 6 patterns
- NBFI/Financial Stress: 8 patterns
- Currency: 3 patterns
- Inflation: 7 patterns
- Valuation: 7 patterns
- Commodity: 5 patterns
- OFR: 3 patterns

**Total: 66 FRED series configured in FredCollector**

---

## MVP Indicators (Priority 0)

These 7 indicators form the minimum viable product for recession monitoring:

### 1. Initial Claims (Weekly)

- **ATLAS Name**: Initial Claims
- **FRED Series ID**: `ICSA`
- **Full Name**: Initial Claims (SA)
- **Frequency**: Weekly (released Thursday 8:30 AM ET)
- **Units**: Thousands of Persons
- **Alert Threshold**: >400 (400,000 claims)
- **Alert Direction**: Above
- **Category**: Recession
- **Cron**: `0 0 9 ? * THU` (Thursday 9 AM ET)

### 2. Industrial Production: Manufacturing (Monthly)

- **ATLAS Name**: Industrial Production Manufacturing
- **FRED Series ID**: `IPMAN` ✅ **IMPLEMENTED**
- **Full Name**: Industrial Production: Manufacturing (NAICS)
- **Frequency**: Monthly (mid-month, ~15th, 9:15 AM ET)
- **Units**: Index (2017=100)
- **Alert Thresholds**: <100 (contraction from baseline)
- **Alert Direction**: Below
- **Category**: Recession
- **Cron**: `0 0 10 15 * ?` (15th of month, 10 AM ET)
- **Note**: More reliable than discontinued NAPM series

### 3. Consumer Sentiment (Monthly)

- **ATLAS Name**: Consumer Sentiment
- **FRED Series ID**: `UMCSENT`
- **Full Name**: University of Michigan: Consumer Sentiment
- **Frequency**: Monthly (mid-month preliminary + end-month final)
- **Units**: Index (1966:Q1=100)
- **Alert Threshold**: <80
- **Alert Direction**: Below
- **Category**: Recession
- **Cron**: `0 0 11 15 * ?` (15th of month, 11 AM ET - catches final)

### 4. SOFR (Daily)

- **ATLAS Name**: SOFR (Repo Market Stress)
- **FRED Series ID**: `SOFR`
- **Full Name**: Secured Overnight Financing Rate
- **Frequency**: Daily (business days, published next day)
- **Units**: Percent per annum
- **Alert Threshold**: Day-over-day change >0.10% (10 bps spike)
- **Alert Direction**: Either (unusual moves)
- **Category**: Recession / NBFI
- **Cron**: `0 0 18 * * MON-FRI` (6 PM ET daily)

### 5. 10-Year Treasury Yield (Daily)

- **ATLAS Name**: 10Y Treasury
- **FRED Series ID**: `DGS10`
- **Full Name**: Market Yield on U.S. Treasury Securities at 10-Year Constant Maturity
- **Frequency**: Daily (business days)
- **Units**: Percent per annum
- **Alert Threshold**: <3.0% (extreme safety demand) OR >5.0% (stress)
- **Alert Direction**: Either
- **Category**: Liquidity
- **Cron**: `0 0 18 * * MON-FRI` (6 PM ET daily)

### 6. Unemployment Rate (Monthly)

- **ATLAS Name**: Unemployment Rate
- **FRED Series ID**: `UNRATE`
- **Full Name**: Unemployment Rate (SA)
- **Frequency**: Monthly (1st Friday, 8:30 AM ET)
- **Units**: Percent
- **Alert Threshold**: >4.5% (Sahm Rule trigger ~0.5pp rise from 12-mo low)
- **Alert Direction**: Above
- **Category**: Recession
- **Cron**: `0 0 10 1-7 * FRI` (1st Friday, 10 AM ET)

### 7. Fed Funds Rate (Daily)

- **ATLAS Name**: Fed Funds Effective Rate
- **FRED Series ID**: `FEDFUNDS`
- **Full Name**: Federal Funds Effective Rate
- **Frequency**: Daily (business days, with monthly averages)
- **Units**: Percent per annum
- **Alert Threshold**: Change >0.25% (unexpected move)
- **Alert Direction**: Either
- **Category**: Liquidity
- **Cron**: `0 0 18 * * MON-FRI` (6 PM ET daily)

---

## RECESSION INDICATORS (Priority 0 - Category Weight: 35%)

Core employment, manufacturing, and sentiment indicators that signal recession:

### 8. Continuing Claims (Weekly)

- **ATLAS Name**: Continuing Claims
- **FRED Series ID**: `CCSA`
- **Full Name**: Continued Claims (Insured Unemployment) (SA)
- **Frequency**: Weekly (released Thursday 8:30 AM ET)
- **Units**: Thousands of Persons
- **Alert Threshold**: >2,000 (2M continuing claims)
- **Alert Direction**: Above
- **Category**: Recession
- **Cron**: `0 0 9 ? * THU` (Thursday 9 AM ET)

### 9. Nonfarm Payrolls (Monthly)

- **ATLAS Name**: Payrolls
- **FRED Series ID**: `PAYEMS`
- **Full Name**: All Employees, Total Nonfarm (SA)
- **Frequency**: Monthly (1st Friday, 8:30 AM ET)
- **Units**: Thousands of Persons
- **Alert Threshold**: Negative print or <100K
- **Alert Direction**: Below
- **Category**: Recession
- **Cron**: `0 0 10 1-7 * FRI` (1st Friday, 10 AM ET)

### 10. JOLTS Job Openings (Monthly)

- **ATLAS Name**: Job Openings
- **FRED Series ID**: `JTSJOL`
- **Full Name**: Job Openings: Total Nonfarm (SA)
- **Frequency**: Monthly (~1 month lag, ~10 AM ET)
- **Units**: Thousands
- **Alert Threshold**: <8,000 (8M openings - "job hugging" signal)
- **Alert Direction**: Below
- **Category**: Recession
- **Cron**: `0 0 11 1-7 * TUE-THU` (Monthly release, flexible day)

---

## LIQUIDITY INDICATORS (Priority 0 - Category Weight: 25%)

Money supply, credit conditions, and Fed operations:

### 11. High Yield Spreads (Daily)

- **ATLAS Name**: HY Spreads
- **FRED Series ID**: `BAMLH0A0HYM2` (BofA ML HY Option-Adjusted Spread)
- **Full Name**: ICE BofA US High Yield Index Option-Adjusted Spread
- **Frequency**: Daily (business days)
- **Units**: Percent (basis points / 100)
- **Alert Thresholds**: >3.50% (350 bps - watch), >5.50% (550 bps - bear)
- **Alert Direction**: Above
- **Category**: Liquidity / Shadow Banking
- **Cron**: `0 0 18 * * MON-FRI` (6 PM ET daily)

### 12. 10Y-2Y Treasury Spread (Daily)

- **ATLAS Name**: Yield Curve
- **FRED Series ID**: `T10Y2Y`
- **Full Name**: 10-Year Treasury Constant Maturity Minus 2-Year
- **Frequency**: Daily (business days)
- **Units**: Percent
- **Alert Threshold**: <0 (inverted), >0 (un-inversion)
- **Alert Direction**: Either (transitions matter)
- **Category**: Liquidity / Recession Signal
- **Cron**: `0 0 18 * * MON-FRI` (6 PM ET daily)

### 13. Fed Balance Sheet (Weekly)

- **ATLAS Name**: Fed Balance Sheet
- **FRED Series ID**: `WALCL`
- **Full Name**: Assets: Total Assets: Total Assets (Less Eliminations from Consolidation)
- **Frequency**: Weekly (Wednesday release)
- **Units**: Millions of Dollars
- **Alert Threshold**: Week-over-week change >$100B (QE/QT acceleration)
- **Alert Direction**: Either
- **Category**: Liquidity
- **Cron**: `0 0 18 * * WED` (Wednesday 6 PM ET)

### 14. M2 Money Supply (Weekly)

- **ATLAS Name**: M2 Money Supply (GlobalM2 component)
- **FRED Series ID**: `WM2NS`
- **Full Name**: M2 Money Stock (SA)
- **Frequency**: Weekly (Tuesday release)
- **Units**: Billions of Dollars
- **Alert Threshold**: YoY growth <2% (tight) or >6% (loose)
- **Alert Direction**: Either
- **Category**: Liquidity
- **Cron**: `0 0 18 * * TUE` (Tuesday 6 PM ET)

### 15. Overnight Reverse Repo (Daily)

- **ATLAS Name**: Reverse Repo
- **FRED Series ID**: `RRPONTSYD`
- **Full Name**: Overnight Reverse Repurchase Agreements: Treasury Securities Sold by the Federal Reserve
- **Frequency**: Daily (business days)
- **Units**: Millions of Dollars
- **Alert Threshold**: >$2,000,000 (>$2T - extreme liquidity parking)
- **Alert Direction**: Above
- **Category**: Liquidity / Money Market Stress
- **Cron**: `0 0 18 * * MON-FRI` (6 PM ET daily)

---

## GROWTH INDICATORS (Priority 0 - Category Weight: 20%)

GDP, production, consumption, and housing activity:

### 16. Industrial Production (Monthly)

- **ATLAS Name**: Industrial Production
- **FRED Series ID**: `INDPRO`
- **Full Name**: Industrial Production: Total Index (SA)
- **Frequency**: Monthly (mid-month, ~15th, 9:15 AM ET)
- **Units**: Index (2017=100)
- **Alert Threshold**: 3-month average declining
- **Alert Direction**: Below (trend)
- **Category**: Growth
- **Cron**: `0 0 11 15 * ?` (15th of month)

### 17. Retail Sales (Monthly)

- **ATLAS Name**: Retail Sales (Total)
- **FRED Series ID**: `RSAFS`
- **Full Name**: Advance Retail Sales: Retail Trade and Food Services (SA)
- **Frequency**: Monthly (mid-month, ~13th, 8:30 AM ET)
- **Units**: Millions of Dollars
- **Alert Threshold**: Negative MoM for 2 consecutive months
- **Alert Direction**: Below (trend)
- **Category**: Growth
- **Cron**: `0 0 11 13-15 * ?` (13-15th of month)
- **Note**: RSXFS (Retail Trade only, excluding Food Services) also collected

### 17b. Retail Sales Ex-Food Services (Monthly)

- **ATLAS Name**: Retail Sales (Ex-Food)
- **FRED Series ID**: `RSXFS`
- **Full Name**: Advance Retail Sales: Retail Trade (SA)
- **Frequency**: Monthly (mid-month, ~13th, 8:30 AM ET)
- **Units**: Millions of Dollars
- **Category**: Growth
- **Note**: Excludes food services, useful for core retail analysis

### 18. Personal Consumption Expenditures (Monthly)

- **ATLAS Name**: PCE
- **FRED Series ID**: `PCE`
- **Full Name**: Personal Consumption Expenditures
- **Frequency**: Monthly (~end of month, 8:30 AM ET)
- **Units**: Billions of Dollars
- **Alert Threshold**: Negative MoM
- **Alert Direction**: Below
- **Category**: Growth
- **Cron**: `0 0 10 25-31 * ?` (Last week of month)

### 19. Housing Starts (Monthly)

- **ATLAS Name**: Housing Starts
- **FRED Series ID**: `HOUST`
- **Full Name**: Housing Starts: Total: New Privately Owned Housing Units Started (SAAR)
- **Frequency**: Monthly (mid-month, ~17th, 8:30 AM ET)
- **Units**: Thousands of Units
- **Alert Threshold**: <1,300 (1.3M units - significant slowdown)
- **Alert Direction**: Below
- **Category**: Growth
- **Cron**: `0 0 10 15-20 * ?` (Mid-month)

---

## FINANCIAL STRESS INDICATORS (Priority 1 - Category Weight: 10%)

Stress indices and credit quality:

### 20. St. Louis Fed Financial Stress Index (Weekly)

- **ATLAS Name**: Financial Stress
- **FRED Series ID**: `STLFSI4`
- **Full Name**: St. Louis Fed Financial Stress Index
- **Frequency**: Weekly (Thursday release)
- **Units**: Index (0 = average conditions, positive = above-average stress)
- **Alert Threshold**: >1.0 (elevated stress)
- **Alert Direction**: Above
- **Category**: Financial Stress
- **Cron**: `0 0 12 ? * THU` (Thursday noon ET)

### 21. Chicago Fed National Financial Conditions Index (Weekly)

- **ATLAS Name**: NFCI
- **FRED Series ID**: `NFCI`
- **Full Name**: Chicago Fed National Financial Conditions Index
- **Frequency**: Weekly (Monday release for prior week)
- **Units**: Index (0 = average, positive = tighter than average)
- **Alert Threshold**: >0.5 (tight conditions)
- **Alert Direction**: Above
- **Category**: Financial Stress
- **Cron**: `0 0 12 ? * MON` (Monday noon ET)

---

## CURRENCY INDICATORS (Priority 0 - Category Weight: 10%)

Dollar strength and cross-border flows:

### 22. Trade-Weighted Dollar Index (Daily)

- **ATLAS Name**: DXY / Trade-Weighted Dollar
- **FRED Series ID**: `DTWEXBGS`
- **Full Name**: Trade Weighted U.S. Dollar Index: Broad, Goods and Services
- **Frequency**: Daily (business days)
- **Units**: Index (Jan 2006 = 100)
- **Alert Thresholds**: >110 (risk-off), <95 (risk-on)
- **Alert Direction**: Either
- **Category**: Currency
- **Cron**: `0 0 18 * * MON-FRI` (6 PM ET daily)

---

## INFLATION INDICATORS (Priority 1 - Category Weight: 10%)

Price levels and inflation expectations:

### 23. Core CPI (Monthly)

- **ATLAS Name**: Core CPI
- **FRED Series ID**: `CPILFESL`
- **Full Name**: Consumer Price Index for All Urban Consumers: All Items Less Food and Energy (SA)
- **Frequency**: Monthly (mid-month, ~13th, 8:30 AM ET)
- **Units**: Index (1982-84=100)
- **Alert Threshold**: YoY >3.0% (above Fed target)
- **Alert Direction**: Above
- **Category**: Inflation
- **Cron**: `0 0 10 10-15 * ?` (Mid-month)

### 24. 10-Year Breakeven Inflation Rate (Daily)

- **ATLAS Name**: Inflation Expectations
- **FRED Series ID**: `T10YIE`
- **Full Name**: 10-Year Breakeven Inflation Rate
- **Frequency**: Daily (business days)
- **Units**: Percent
- **Alert Thresholds**: <1.5% (deflationary), >3.0% (high inflation)
- **Alert Direction**: Either
- **Category**: Inflation
- **Cron**: `0 0 18 * * MON-FRI` (6 PM ET daily)

---

## VALUATION & MARKET CONTEXT (Priority 1)

Supporting data for Buffett Indicator and market analysis:

### 25. S&P 500 Index (Daily)

- **ATLAS Name**: S&P 500
- **FRED Series ID**: `SP500`
- **Full Name**: S&P 500
- **Frequency**: Daily (business days)
- **Units**: Index
- **Category**: Valuation (reference)
- **Cron**: `0 0 18 * * MON-FRI` (6 PM ET daily)

### 26. Wilshire 5000 Total Market Index (Daily)

- **ATLAS Name**: Wilshire 5000
- **FRED Series ID**: `WILL5000INDFC`
- **Full Name**: Wilshire 5000 Full Cap Price Index
- **Frequency**: Daily (business days)
- **Units**: Index
- **Category**: Valuation (for Buffett Indicator = Wilshire / GDP × 100)
- **Cron**: `0 0 18 * * MON-FRI` (6 PM ET daily)

### 27. Real GDP (Quarterly)

- **ATLAS Name**: Real GDP
- **FRED Series ID**: `GDPC1`
- **Full Name**: Real Gross Domestic Product (SAAR)
- **Frequency**: Quarterly (advance ~1 month after quarter end)
- **Units**: Billions of Chained 2017 Dollars
- **Alert Threshold**: Negative print (technical recession)
- **Alert Direction**: Below (negative growth)
- **Category**: Growth / Valuation (Buffett denominator)
- **Cron**: `0 0 11 25-31 1,4,7,10 ?` (End of Jan, Apr, Jul, Oct)

### 28. Smoothed US Recession Probabilities (Monthly)

- **ATLAS Name**: Recession Probability
- **FRED Series ID**: `RECPROUSM156N`
- **Full Name**: Smoothed U.S. Recession Probabilities
- **Frequency**: Monthly
- **Units**: Percent
- **Alert Threshold**: >20% (elevated recession risk)
- **Alert Direction**: Above
- **Category**: Recession (model-based)
- **Cron**: `0 0 11 1-5 * ?` (Early month)

---

## Phase 2 Expansion - Commodity Indicators

Commodity prices for Cu/Au ratio calculation:

### 29. Copper Price (Daily)

- **ATLAS Name**: Copper
- **FRED Series ID**: `PCOPPUSDM` (LME Copper)
- **Full Name**: Global Price of Copper
- **Frequency**: Daily (business days)
- **Units**: U.S. Dollars per Metric Ton
- **Category**: Commodity (for Cu/Au ratio calculation)
- **Cron**: `0 0 18 * * MON-FRI`

### 30. Gold Price (Daily)

- **ATLAS Name**: Gold
- **FRED Series ID**: `GOLDAMGBD228NLBM`
- **Full Name**: Gold Fixing Price (London, USD)
- **Frequency**: Daily (business days)
- **Units**: U.S. Dollars per Troy Ounce
- **Category**: Commodity (for Cu/Au ratio calculation)
- **Cron**: `0 0 18 * * MON-FRI`

**Cu/Au Ratio Calculation**: Copper Price ÷ Gold Price

- Thresholds: <0.15 (bear), 0.15-0.22 (neutral), >0.22 (bull)

---

## FRED COLLECTION SUMMARY

**Priority 0 (MVP - 20 Core Series):**
1-10: Recession indicators (Claims, ISM, Sentiment, SOFR, 10Y, Unemployment, Fed Funds, Continuing Claims, Payrolls, JOLTS)
11-15: Liquidity indicators (HY Spreads, Yield Curve, Fed Balance, M2, Reverse Repo)
16-19: Growth indicators (Industrial Production, Retail Sales, PCE, Housing Starts)
22: Currency (DXY)

**Priority 1 (Expansion - 10 Additional Series):**
20-21: Financial Stress (STLFSI4, NFCI)
23-24: Inflation (Core CPI, Breakeven Inflation)
25-28: Valuation & Context (S&P 500, Wilshire 5000, GDP, Recession Probability)
29-30: Commodities (Copper, Gold)

**Total: 66 series from FRED for complete ATLAS coverage**

**Additional Series Collected (Phase 2 Expansion):**
- `BAMLC0A0CM` - Investment Grade Corporate Bond Spread
- `BOGZ1LM193064005Q` - Household Net Worth
- `CBBTCUSD` - Bitcoin Price
- `CIVPART` - Labor Force Participation Rate
- `CPIAUCSL` - All Items CPI
- `DCOILBRENTEU` - Brent Crude Oil
- `DCOILWTICO` - WTI Crude Oil
- `DEXCHUS` - Chinese Yuan Exchange Rate
- `DEXJPUS` - Japanese Yen Exchange Rate
- `DEXUSEU` - Euro Exchange Rate
- `DFII10` - 10-Year TIPS Rate
- `DGORDER` - Durable Goods Orders
- `DGS2` - 2-Year Treasury Yield
- `DPRIME` - Prime Rate
- `M2SL` - M2 Money Stock (monthly)
- `MORTGAGE30US` - 30-Year Mortgage Rate
- `MSPUS` - Median Home Sale Price
- `NCBEILQ027S` - Corporate Profits
- `PCEDG` - PCE Durable Goods
- `PERMIT` - Building Permits
- `PI` - Personal Income
- `PRFI` - Private Residential Fixed Investment
- `PSAVERT` - Personal Savings Rate
- `SAHMCURRENT` - Sahm Rule Recession Indicator
- `TCU` - Total Capacity Utilization
- `Y033RC1Q027SBEA` - Private Nonresidential Fixed Investment

---

## Indicators NOT Available in FRED

These indicators may require alternative data sources:

### VIX (CBOE Volatility Index) ✅ CONFIRMED IN FRED

- **Status**: ✅ Available as `VIXCLS` - Daily updates
- **FRED Series**: VIXCLS (CBOE Volatility Index: VIX)
- **Alert Thresholds**: >22 (L1: deploy $25-100K), >30 (L2: deploy $150-500K)
- **Category**: Liquidity / VIX Deployment System (CRITICAL for ATLAS)
- **Implementation**: ✅ Configured in FredCollector, actively collecting

### Baltic Dry Index

- **Status**: ❌ Not in FRED
- **Alternative**: Bloomberg, MarketWatch scraping, or paid API
- **Thresholds**: <600 (bear), 600-1500 (neutral), >1500 (bull)
- **Phase**: Defer to Phase 2 (NewsCollector or web scraping)

### R2K/SPX Ratio

- **Status**: ⚠️ Components may be in FRED, ratio needs calculation
- **FRED Series**: Check for `RU2000PR` (Russell 2000) and `SP500`
- **Calculation**: Russell 2000 ÷ S&P 500
- **Thresholds**: <0.50 (bear), 0.50-0.65 (neutral), >0.65 (bull)
- **Phase**: Phase 2 (calculated indicators)

### Shiller CAPE

- **Status**: ⚠️ May be in FRED, verify
- **Potential Series**: Check for Robert Shiller data
- **Alternative**: Calculate from S&P 500 + 10Y avg earnings
- **Phase**: Phase 2 (valuation metrics)

### Buffett Indicator (Market Cap / GDP)

- **Status**: ⚠️ Components in FRED, needs calculation
- **FRED Series**: `WILSHIRE5000PRFC` (market cap) ÷ `GDP`
- **Calculation**: Wilshire 5000 ÷ GDP × 100
- **Threshold**: >217% (extreme bubble, per sigma.md)
- **Phase**: Phase 2 (calculated indicators)

---

## Configuration File Template

Use this template for `config/atlas-indicators.json`:

```json
{
  "seriesConfigurations": [
    {
      "seriesId": "ICSA",
      "title": "Initial Claims",
      "description": "Initial unemployment claims, seasonally adjusted",
      "category": "Recession",
      "frequency": "Weekly",
      "cronExpression": "0 0 9 ? * THU",
      "alertThreshold": 400000,
      "thresholdDirection": "Above",
      "isActive": true,
      "notes": "Key recession indicator, watch for >400K"
    },
    {
      "seriesId": "UMCSENT",
      "title": "Consumer Sentiment",
      "description": "University of Michigan Consumer Sentiment Index",
      "category": "Recession",
      "frequency": "Monthly",
      "cronExpression": "0 0 11 15 * ?",
      "alertThreshold": 80,
      "thresholdDirection": "Below",
      "isActive": true,
      "notes": "Recession confirmed when <80"
    }
    // ... repeat for all 7 MVP series
  ]
}
```

---

## Verification Checklist ✅ COMPLETE

All items verified during implementation:

- [x] Verify `NAPM` is correct series for ISM Manufacturing - Using IPMAN instead (more reliable)
- [x] Verify `VIXCLS` exists in FRED - ✅ Confirmed, daily updates available
- [x] Confirm FRED publication times match cron schedules - ✅ Verified
- [x] Test FRED API with each series ID to ensure they return data - ✅ All 66 series tested
- [x] Document any series that need alternative sources - Baltic Dry, Shiller CAPE deferred

**API Test Command**:

```bash
curl "https://api.stlouisfed.org/fred/series/observations?series_id=ICSA&api_key=YOUR_KEY&file_type=json"
```

---

**Last Updated**: 2026-01-19
**Status**: ✅ Implemented - 66 series configured and collecting
**Owner**: James
**Implementation**: FredCollector service + FredCollector MCP server

## FredCollector Architecture

The FRED data collection system consists of two components:
- **FredCollector** (`FredCollector/src/`) - Core data collection service
- **FredCollector MCP** (`FredCollector/mcp/`) - MCP server for AI-assisted data access
