# Pattern Weight Assignment Guide

**Purpose**: Standardized criteria for assigning reliability weights to ThresholdEngine patterns  
**Audience**: Engineers, Product Owners  
**Version**: 1.0  
**Updated**: 2025-12-14  

---

## Weight Ranges

| Range | Classification | Criteria | Example Patterns |
|-------|----------------|----------|------------------|
| **0.90-1.00** | Proven Reliable | 100% historical accuracy, decades of data, minimal false positives | Sahm Rule (0.95), Yield Curve Inversion (0.90) |
| **0.75-0.89** | Strong Signal | High reliability, timely data, good track record | ISM Manufacturing (0.85), VIX L2 (0.85), Initial Claims (0.80) |
| **0.60-0.74** | Solid Contributor | Useful signal, some noise, slower timing | Fed Liquidity (0.70), DXY Risk-Off (0.65), Continuing Claims (0.60) |
| **0.40-0.59** | Supplementary | High noise, lagging, or informational | Freight Recession (0.55), Consumer Sentiment (0.50) |
| **0.00-0.39** | Experimental | Unproven, testing, or low confidence | New patterns under evaluation |

---

## Assignment Criteria

### Historical Accuracy (40% of decision)

**Questions**:
- Has this pattern triggered before recessions/events?
- What is the false positive rate?
- What is the false negative rate?

**Scoring**:
- 100% accuracy (Sahm Rule): 0.90-0.95
- <10% error rate: 0.80-0.89
- 10-20% error rate: 0.70-0.79
- 20-30% error rate: 0.60-0.69
- >30% error rate: 0.40-0.59

**Examples**:
- **Sahm Rule**: 100% accuracy since 1970 → **0.95**
- **Yield Curve**: ~90% accuracy, occasional false positives → **0.90**
- **Consumer Sentiment**: High noise, ~60% accuracy → **0.50**

### Signal Timeliness (30% of decision)

**Questions**:
- How quickly does data update? (Daily, Weekly, Monthly, Quarterly)
- What is the publication lag? (Real-time, T+1, T+30)
- Is the signal forward-looking or backward-looking?

**Scoring**:
- Daily updates, <7 day lag: 0.80-0.90
- Weekly updates, <14 day lag: 0.70-0.85
- Monthly updates, <30 day lag: 0.60-0.75
- Quarterly updates, >45 day lag: 0.40-0.60

**Examples**:
- **VIX**: Real-time, updated continuously → **0.85**
- **ISM Manufacturing**: Monthly, released 1st business day → **0.85**
- **GDP**: Quarterly, 30-day lag, revised multiple times → **0.70**
- **Freight Index**: Monthly, persistent but slow → **0.55**

### Signal Clarity (20% of decision)

**Questions**:
- Is the threshold objective and quantifiable?
- Are there subjective interpretations required?
- Does the signal have multiple conflicting components?

**Scoring**:
- Clear threshold, binary signal: 0.85-0.95
- Clear threshold, range signal: 0.70-0.84
- Fuzzy threshold, requires interpretation: 0.50-0.69
- Composite of conflicting signals: 0.40-0.59

**Examples**:
- **Sahm Rule**: Clear formula, 0.5pp threshold → **0.95**
- **Yield Curve**: Simple spread, 0bps threshold → **0.90**
- **NBFI Escalation**: Multi-indicator composite → **0.90** (aggregates clear signals)
- **Consumer Sentiment**: Survey-based, fuzzy threshold → **0.50**

### Data Quality (10% of decision)

**Questions**:
- Is the data source authoritative (Fed, BLS, Census)?
- Is the series frequently revised?
- Are there data quality issues or gaps?

**Scoring**:
- Official government source, minimal revisions: 0.80-0.95
- Official source, frequent revisions: 0.65-0.79
- Private source, good reputation: 0.60-0.74
- Private source, limited history: 0.40-0.59

**Examples**:
- **FRED series (Fed data)**: Official, reliable → **0.80-0.95**
- **ISM surveys**: Private but established → **0.85**
- **OFR FSI**: Official, new (2017+) → **0.85**
- **Sentiment surveys**: Private, volatile → **0.50-0.60**

---

## Temporal Classification

### Leading Indicators (Signal 3+ months ahead)

**Characteristics**:
- Changes precede economic events
- Forward-looking data (permits, orders, surveys of intentions)
- Market-based (yield curve, credit spreads)

**Examples**:
| Pattern | Lead Time | Weight | Rationale |
|---------|-----------|--------|-----------|
| Yield Curve Inversion | 12-18mo | 0.90 | Historically leads recessions by 1+ year |
| Building Permits | 6-9mo | 0.75 | Construction activity ahead of housing data |
| Credit Spread Widening | 3-6mo | 0.90 | Markets anticipate stress before realization |
| M2 Money Supply | 12-18mo | 0.65 | Liquidity transmission lag |

**Weight Adjustment**: +0.05 to +0.10 if proven leading with good track record

### Coincident Indicators (Real-time snapshot)

**Characteristics**:
- Reflect current economic state
- Published monthly/weekly with short lag
- Direct measures (employment, production, sales)

**Examples**:
| Pattern | Lag | Weight | Rationale |
|---------|-----|--------|-----------|
| Sahm Rule | 0mo | 0.95 | Unemployment MA is current state |
| ISM Manufacturing | 0mo | 0.85 | Survey of current business conditions |
| Initial Jobless Claims | 0mo | 0.80 | Weekly real-time labor data |
| VIX | 0mo | 0.75-0.85 | Current market volatility |

**Weight Adjustment**: Neutral (base weight on accuracy/quality)

### Lagging Indicators (Confirm past events)

**Characteristics**:
- Changes follow economic events
- Validate what already happened
- Slow to turn (unemployment rate, CPI)

**Examples**:
| Pattern | Lag | Weight | Rationale |
|---------|-----|--------|-----------|
| Continuing Claims | 1-2mo | 0.60 | Confirms labor market deterioration |
| CPI Acceleration | 2-3mo | 0.80 | Prices react to past conditions |
| Unemployment Rate | 3-6mo | 0.70 | Last to rise in recession |
| Freight Recession | 6mo+ | 0.55 | Persistent but very slow signal |

**Weight Adjustment**: -0.05 to -0.15 for lagging confirmation indicators

---

## Decay Period Assignment

The `signalDecayDays` parameter controls how quickly stale data loses influence.

### Fast Decay (7-14 days)

**Use for**:
- Market-based signals (VIX, credit spreads, DXY)
- High-frequency data (daily updates)
- Volatile signals that change rapidly

**Examples**:
- VIX: 7 days (volatility mean-reverts quickly)
- DXY Risk-Off: 14 days (FX moves are transient)
- Standing Repo Stress: 14 days (emergency facility usage)

**Formula**: `signalDecayDays = 2 × update_frequency_days`

### Moderate Decay (30-60 days)

**Use for**:
- Monthly economic data
- Business surveys (ISM, sentiment)
- Most coincident indicators

**Examples**:
- ISM Manufacturing: 30 days (monthly release)
- Initial Claims: 30 days (weekly but smoothed)
- Chicago NFCI: 30 days (monthly financial conditions)
- Continuing Claims: 60 days (slower-moving labor metric)

**Formula**: `signalDecayDays = 1 × typical_update_cycle`

### Slow Decay (90-180 days)

**Use for**:
- Quarterly data (GDP)
- Persistent trends (freight, yield curve)
- Structural indicators (M2, demographics)

**Examples**:
- Sahm Rule: 90 days (unemployment changes slowly)
- Yield Curve: 180 days (signal persists for quarters)
- Freight Recession: 180 days (industrial trends are persistent)
- Buffett Indicator: 180 days (market cap ratios change slowly)

**Formula**: `signalDecayDays = 2-3 × quarterly_period`

### Glacial Decay (365 days)

**Use for**:
- Long-term structural indicators
- Informational patterns (not used for regime detection)
- Annual data

**Examples**:
- Demographics-based patterns (if added)
- Secular trend indicators

---

## Confidence Score (Informational Only)

The `confidence` field (0.0-1.0) is **not used in calculations** but serves as documentation of historical reliability.

**Assignment**:
- 0.95-1.00: Proven track record, >20 years of data, near-perfect accuracy
- 0.85-0.94: Strong historical performance, occasional false positives
- 0.75-0.84: Good track record, some noise
- 0.60-0.74: Moderate reliability, known limitations
- 0.50-0.59: Experimental or low sample size
- <0.50: Testing, unvalidated

**Use**:
- Documentation for future weight adjustments
- Comparison across patterns in same category
- Justification for weight assignments

---

## Example: Assigning Weights to New Pattern

### Scenario: "OFR FSI Extreme Stress" pattern

**Step 1: Gather Facts**
- **Data Source**: Office of Financial Research (official US Treasury data)
- **History**: Available since 2017 (8 years)
- **Update Frequency**: Weekly
- **Threshold**: FSI > 1.5 (extreme stress, historical events: COVID, 2018 selloff)
- **Accuracy**: 2/2 major stress events detected, 1 false positive (2018 volatility spike)

**Step 2: Apply Criteria**

| Criterion | Assessment | Score | Rationale |
|-----------|------------|-------|-----------|
| Historical Accuracy | 66% (2 true positives, 1 false positive) | 0.70 | Limited sample size but good hit rate |
| Timeliness | Weekly, <7 day lag | 0.80 | Current data, minimal lag |
| Signal Clarity | Clear threshold (1.5), objective | 0.85 | FSI is composite but well-defined |
| Data Quality | Official source, short history | 0.75 | Authoritative but new series |

**Step 3: Calculate Weight**

Weighted average: (0.70×0.4) + (0.80×0.3) + (0.85×0.2) + (0.75×0.1) = 0.28 + 0.24 + 0.17 + 0.075 = **0.755**

**Round to**: **0.75** (Solid Contributor range)

**Step 4: Assign Temporal Classification**

- OFR FSI: Coincident (reflects current financial stress)
- Lead Time: 0 months
- Decay: 30 days (weekly data, monthly smoothing)

**Step 5: Set Confidence**

- Short history (8 years): 0.70
- Good but limited sample: 0.70

**Final Pattern Config**:

```json
{
  "patternId": "ofr-fsi-extreme",
  "name": "OFR Financial Stress Index - Extreme",
  "weight": 0.75,
  "temporalType": "Coincident",
  "leadTimeMonths": 0,
  "signalDecayDays": 30,
  "confidence": 0.70
}
```

---

## Adjustment Scenarios

### Scenario 1: Pattern proves more accurate over time

**Initial**: Experimental pattern with `weight=0.50`, `confidence=0.60`  
**After 3 years**: 95% accuracy, no false positives  
**Action**: Increase `weight=0.85`, `confidence=0.90`  
**Rationale**: Track record established, promote to Strong Signal tier

### Scenario 2: Pattern becomes noisier

**Initial**: Strong signal with `weight=0.80`, `confidence=0.85`  
**After revision**: False positive rate increases to 30%  
**Action**: Decrease `weight=0.65`, `confidence=0.70`  
**Rationale**: Data quality degradation, demote to Solid Contributor

### Scenario 3: New data extends history

**Initial**: Limited history (5 years), `weight=0.70`, `confidence=0.65`  
**After 10 more years**: Same accuracy rate, larger sample  
**Action**: Maintain `weight=0.70`, increase `confidence=0.85`  
**Rationale**: More data confirms reliability but doesn't change accuracy

---

## Category-Specific Guidelines

### Recession Patterns

**Typical Weights**: 0.75-0.95 (high-stakes, well-studied)  
**Temporal**: Mix of Leading (yield curve) and Coincident (claims, ISM)  
**Decay**: Moderate to Slow (30-90 days)  
**Notes**: Prioritize proven indicators (Sahm, yield curve) over sentiment

### Liquidity Patterns

**Typical Weights**: 0.65-0.90 (market-based, react quickly)  
**Temporal**: Mix of Leading (credit spreads) and Coincident (VIX)  
**Decay**: Fast to Moderate (7-30 days)  
**Notes**: Context-dependent signals (VIX) may have variable weights

### NBFI Patterns

**Typical Weights**: 0.75-0.90 (shadow banking is critical)  
**Temporal**: Primarily Coincident  
**Decay**: Moderate (14-30 days)  
**Notes**: Composite indicators (NBFI Escalation) can have high weights

### Growth Patterns

**Typical Weights**: 0.70-0.85 (expansion signals)  
**Temporal**: Mix of Leading (permits) and Coincident (ISM)  
**Decay**: Moderate to Slow (30-90 days)  
**Notes**: Real data (production, sales) over surveys

### Inflation Patterns

**Typical Weights**: 0.70-0.90 (Fed policy driver)  
**Temporal**: Primarily Lagging (CPI, PCE)  
**Decay**: Moderate to Slow (60-90 days)  
**Notes**: Breakevens are Leading, CPI/PCE are Lagging

### Valuation Patterns

**Typical Weights**: 0.55-0.70 (informational, not predictive)  
**Temporal**: Lagging  
**Decay**: Slow to Glacial (90-180 days)  
**Notes**: Lower weights due to poor market timing

---

## Peer Review Checklist

Before committing weight assignments:

- [ ] Weight justified using 4 criteria (accuracy, timeliness, clarity, quality)
- [ ] Weight within appropriate range for category
- [ ] Temporal classification matches signal characteristics
- [ ] Decay period matches update frequency and persistence
- [ ] Confidence score documented with rationale
- [ ] Comparison to similar patterns in category
- [ ] Historical accuracy data cited (if available)
- [ ] Adjustments documented in pattern metadata

---

## References

- [Pattern Weighting Implementation Spec](./pattern-weighting-temporal-metadata.md)
- [ThresholdEngine Documentation](../ThresholdEngine/README.md)
- [ATLAS Architecture](../docs/ARCHITECTURE.md)
