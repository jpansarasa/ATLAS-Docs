# ATLAS Task: Add FRED Series for New Patterns

**Date:** 2025-11-26  
**Priority:** High  
**Executor:** Coding Agent  
**API Target:** FredCollector REST API (http://mercury:5001)

## Overview

Six new ThresholdEngine patterns were created that require additional FRED series to be collected. Use the FredCollector Series Management API to add these series.

## API Reference

### Endpoint
```
POST http://mercury:5001/api/series
Content-Type: application/json
```

### Check Existing Series
```
GET http://mercury:5001/api/series
```

### Verify Series Added
```
GET http://mercury:5001/api/series/{seriesId}
```

---

## Series to Add (7 total)

### 1. PCOPPUSDM - Global Copper Price
**Pattern:** `commodity/cu-au-ratio.json`
```json
{
  "seriesId": "PCOPPUSDM",
  "title": "Global Price of Copper",
  "description": "LME Copper price in USD per metric ton. Used for Cu/Au ratio calculation (Dr. Copper indicator).",
  "category": "Commodity",
  "frequency": "Monthly",
  "cronExpression": "0 0 12 1 * ?",
  "isActive": true,
  "alertThreshold": null,
  "thresholdDirection": "Above"
}
```

### 2. GOLDAMGBD228NLBM - London Gold Fixing Price
**Pattern:** `commodity/cu-au-ratio.json`
```json
{
  "seriesId": "GOLDAMGBD228NLBM",
  "title": "Gold Fixing Price 10:30 A.M. (London time) in London Bullion Market",
  "description": "London AM gold fix in USD per troy ounce. Used for Cu/Au ratio calculation.",
  "category": "Commodity",
  "frequency": "Daily",
  "cronExpression": "0 0 18 * * MON-FRI",
  "isActive": true,
  "alertThreshold": null,
  "thresholdDirection": "Above"
}
```

### 3. WM2NS - M2 Money Stock (Weekly)
**Pattern:** `liquidity/m2-growth-rate.json`
```json
{
  "seriesId": "WM2NS",
  "title": "M2 Money Stock (Weekly, Seasonally Adjusted)",
  "description": "Weekly M2 money supply for YoY growth calculation. Tight <2%, neutral 2-6%, loose >6%.",
  "category": "Liquidity",
  "frequency": "Weekly",
  "cronExpression": "0 0 18 ? * TUE",
  "isActive": true,
  "alertThreshold": null,
  "thresholdDirection": "Above"
}
```

### 4. T10YIE - 10-Year Breakeven Inflation Rate
**Pattern:** `liquidity/real-rates.json`
```json
{
  "seriesId": "T10YIE",
  "title": "10-Year Breakeven Inflation Rate",
  "description": "Market-implied inflation expectations (10Y nominal - 10Y TIPS). Used for real rate calculation.",
  "category": "Inflation",
  "frequency": "Daily",
  "cronExpression": "0 0 18 * * MON-FRI",
  "isActive": true,
  "alertThreshold": null,
  "thresholdDirection": "Above"
}
```

### 5. T10Y2Y - 10-Year Treasury Minus 2-Year Treasury
**Pattern:** `recession/yield-curve-steepening.json`, `recession/yield-curve-inversion.json`
```json
{
  "seriesId": "T10Y2Y",
  "title": "10-Year Treasury Constant Maturity Minus 2-Year Treasury Constant Maturity",
  "description": "Yield curve spread. Inversion (<0) precedes recessions; steepening after inversion signals imminent recession.",
  "category": "Recession",
  "frequency": "Daily",
  "cronExpression": "0 0 18 * * MON-FRI",
  "isActive": true,
  "alertThreshold": 0,
  "thresholdDirection": "Below"
}
```

### 6. SAHMCURRENT - Real-time Sahm Rule Recession Indicator
**Pattern:** `recession/sahm-rule-official.json`
```json
{
  "seriesId": "SAHMCURRENT",
  "title": "Real-time Sahm Rule Recession Indicator",
  "description": "Official FRED Sahm Rule series. Triggers at 0.5+ (100% historical accuracy since 1970).",
  "category": "Recession",
  "frequency": "Monthly",
  "cronExpression": "0 0 10 1-7 * FRI",
  "isActive": true,
  "alertThreshold": 0.5,
  "thresholdDirection": "Above"
}
```

### 7. RECPROUSM156N - Smoothed U.S. Recession Probabilities
**Pattern:** `recession/recession-probability.json`
```json
{
  "seriesId": "RECPROUSM156N",
  "title": "Smoothed U.S. Recession Probabilities",
  "description": "Markov-switching model recession probability (Chauvet-Hamilton). Warning >20%, high >50%.",
  "category": "Recession",
  "frequency": "Monthly",
  "cronExpression": "0 0 11 1-5 * ?",
  "isActive": true,
  "alertThreshold": 20,
  "thresholdDirection": "Above"
}
```

---

## Execution Script (bash)

```bash
#!/bin/bash
# Execute from mercury or any host with API access

API_BASE="http://mercury:5001/api/series"

# 1. PCOPPUSDM
curl -X POST "$API_BASE" \
  -H "Content-Type: application/json" \
  -d '{
    "seriesId": "PCOPPUSDM",
    "title": "Global Price of Copper",
    "description": "LME Copper price in USD per metric ton. Used for Cu/Au ratio calculation.",
    "category": "Commodity",
    "frequency": "Monthly",
    "cronExpression": "0 0 12 1 * ?",
    "isActive": true,
    "alertThreshold": null,
    "thresholdDirection": "Above"
  }'

echo ""

# 2. GOLDAMGBD228NLBM
curl -X POST "$API_BASE" \
  -H "Content-Type: application/json" \
  -d '{
    "seriesId": "GOLDAMGBD228NLBM",
    "title": "Gold Fixing Price (London)",
    "description": "London AM gold fix in USD per troy ounce. Used for Cu/Au ratio calculation.",
    "category": "Commodity",
    "frequency": "Daily",
    "cronExpression": "0 0 18 * * MON-FRI",
    "isActive": true,
    "alertThreshold": null,
    "thresholdDirection": "Above"
  }'

echo ""

# 3. WM2NS
curl -X POST "$API_BASE" \
  -H "Content-Type: application/json" \
  -d '{
    "seriesId": "WM2NS",
    "title": "M2 Money Stock (Weekly, SA)",
    "description": "Weekly M2 money supply for YoY growth calculation.",
    "category": "Liquidity",
    "frequency": "Weekly",
    "cronExpression": "0 0 18 ? * TUE",
    "isActive": true,
    "alertThreshold": null,
    "thresholdDirection": "Above"
  }'

echo ""

# 4. T10YIE
curl -X POST "$API_BASE" \
  -H "Content-Type: application/json" \
  -d '{
    "seriesId": "T10YIE",
    "title": "10-Year Breakeven Inflation Rate",
    "description": "Market-implied inflation expectations. Used for real rate calculation.",
    "category": "Inflation",
    "frequency": "Daily",
    "cronExpression": "0 0 18 * * MON-FRI",
    "isActive": true,
    "alertThreshold": null,
    "thresholdDirection": "Above"
  }'

echo ""

# 5. T10Y2Y
curl -X POST "$API_BASE" \
  -H "Content-Type: application/json" \
  -d '{
    "seriesId": "T10Y2Y",
    "title": "10-Year Minus 2-Year Treasury Spread",
    "description": "Yield curve spread. Inversion precedes recessions.",
    "category": "Recession",
    "frequency": "Daily",
    "cronExpression": "0 0 18 * * MON-FRI",
    "isActive": true,
    "alertThreshold": 0,
    "thresholdDirection": "Below"
  }'

echo ""

# 6. SAHMCURRENT
curl -X POST "$API_BASE" \
  -H "Content-Type: application/json" \
  -d '{
    "seriesId": "SAHMCURRENT",
    "title": "Real-time Sahm Rule Recession Indicator",
    "description": "Official FRED Sahm Rule. Triggers at 0.5+.",
    "category": "Recession",
    "frequency": "Monthly",
    "cronExpression": "0 0 10 1-7 * FRI",
    "isActive": true,
    "alertThreshold": 0.5,
    "thresholdDirection": "Above"
  }'

echo ""

# 7. RECPROUSM156N
curl -X POST "$API_BASE" \
  -H "Content-Type: application/json" \
  -d '{
    "seriesId": "RECPROUSM156N",
    "title": "Smoothed U.S. Recession Probabilities",
    "description": "Markov-switching model recession probability.",
    "category": "Recession",
    "frequency": "Monthly",
    "cronExpression": "0 0 11 1-5 * ?",
    "isActive": true,
    "alertThreshold": 20,
    "thresholdDirection": "Above"
  }'

echo ""
echo "Done. Verify with: curl $API_BASE"
```

---

## Verification

After adding series, verify:

1. **List all series:** `GET http://mercury:5001/api/series`
2. **Trigger immediate collection:** Check if API supports manual trigger or wait for cron
3. **Check observations:** `GET http://mercury:5001/api/observations/{seriesId}/latest`

---

## Pattern Files Created (for reference)

These patterns are ready and waiting for data:

| Pattern | Location | Status |
|---------|----------|--------|
| Cu/Au Ratio | `ThresholdEngine/config/patterns/commodity/cu-au-ratio.json` | ✅ Created |
| M2 Growth Rate | `ThresholdEngine/config/patterns/liquidity/m2-growth-rate.json` | ✅ Created |
| Real Rates | `ThresholdEngine/config/patterns/liquidity/real-rates.json` | ✅ Created |
| Yield Curve Steepening | `ThresholdEngine/config/patterns/recession/yield-curve-steepening.json` | ✅ Created |
| Sahm Rule Official | `ThresholdEngine/config/patterns/recession/sahm-rule-official.json` | ✅ Created |
| Recession Probability | `ThresholdEngine/config/patterns/recession/recession-probability.json` | ✅ Created |

---

## Post-Execution

After series are collecting:
1. ThresholdEngine will auto-discover new patterns on next config reload
2. Patterns will begin evaluating once sufficient historical data exists (check `maxLookbackDays`)
3. Monitor Grafana dashboards for pattern evaluation metrics

---

## Future: FredCollector MCP

When building the FredCollector MCP to replace FRED MCP Server:

| MCP Tool | FredCollector Endpoint | Description |
|----------|------------------------|-------------|
| `list_series` | `GET /api/series` | List all configured series |
| `get_series` | `GET /api/series/{id}` | Get series config details |
| `get_latest` | `GET /api/observations/{id}/latest` | Get most recent observation |
| `get_observations` | `GET /api/observations/{id}?start=&end=` | Get observation range |
| `search_series` | `GET /api/series/search?q=` | Discovery API (E12) |
| `get_macro_score` | `GET /api/macro-score` | Current regime + score |

This would give Claude direct access to ATLAS data without going through FRED's API.
