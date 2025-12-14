# Pattern Weight Assignment Guide

**Purpose**: Standardized criteria for assigning reliability weights to ThresholdEngine patterns  
**Audience**: Engineers, Product Owners  
**Version**: 1.1  
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

## Publication Frequency Assignment

The `publicationFrequencyDays` parameter is an **objective fact** about data availability, not a tuning parameter. This tells the system when to expect new data.

### Real-Time / Intraday (publicationFrequencyDays = 0)

**Indicators**:
- VIX (continuous during market hours)
- DXY (forex markets 24/5)
- Credit Spreads (calculated from bond prices)
- Stock/ETF prices

**Behavior**: Decay starts immediately since data could update any moment.

### Daily (publicationFrequencyDays = 1)

**Indicators**:
- Standing Repo Facility (Fed publishes daily)
- Treasury Yields (daily market close)
- Reverse Repo (Fed daily)

**Behavior**: Data older than 1 day starts decaying.

### Weekly (publicationFrequencyDays = 7)

**Indicators**:
- Initial Jobless Claims (every Thursday)
- Continuing Claims (every Thursday)
- M2 Money Supply (Fed H.6 weekly)

**Behavior**: Data remains 100% fresh for 7 days, then decays.

### Monthly (publicationFrequencyDays = 30)

**Indicators**:
- ISM Manufacturing/Services (1st business day)
- Nonfarm Payrolls (1st Friday)
- CPI / Core CPI (mid-month ~15th)
- PCE / Core PCE (end of month ~30th)
- Industrial Production (mid-month ~15th)
- Retail Sales (mid-month ~15th)
- Housing Starts (mid-month ~17th)
- Consumer Sentiment (end of month, final)
- Chicago NFCI (weekly but use 30 for stability)
- OFR FSI (weekly but use 30 for stability)

**Behavior**: Data remains 100% fresh for 30 days, then decays.

### Quarterly (publicationFrequencyDays = 90)

**Indicators**:
- GDP (advance estimate ~30 days after quarter end)
- Corporate Earnings (staggered throughout earnings season)

**Behavior**: Data remains 100% fresh for 90 days, then decays.

### Special Cases

**Sahm Rule** (uses unemployment rate):
- Based on monthly unemployment data + 12-month MA
- `publicationFrequencyDays = 30`

**Yield Curve Inversion**:
- Uses daily Treasury yields but signal is persistent
- `publicationFrequencyDays = 1` (daily calculation)

**Freight Recession**:
- Monthly publication
- `publicationFrequencyDays = 30`

**Rule of Thumb**: Set `publicationFrequencyDays` to the **shortest normal publication interval**. If weekly data is sometimes delayed by a few days, still use 7 (not 10).

---

## Signal Decay Assignment (After Data Becomes Overdue)

The `signalDecayDays` parameter controls how quickly a signal loses influence **after it becomes overdue**. This is independent of publication frequency and reflects signal persistence.

**Critical Distinction**:
- `publicationFrequencyDays` = When we expect new data (objective)
- `signalDecayDays` = How long the signal matters after overdue (subjective)

### Fast Decay (7-14 days after overdue)

**Use for**: Transient, mean-reverting signals that lose relevance quickly

**Examples**:
- **VIX spikes**: Market panic/volatility mean-reverts quickly → `signalDecayDays = 7`
- **DXY Risk-Off**: FX moves are often transient → `signalDecayDays = 14`
- **Standing Repo Stress**: Emergency facility usage is acute → `signalDecayDays = 14`

**Rationale**: These signals reflect temporary market conditions that resolve quickly.

### Moderate Decay (30-60 days after overdue)

**Use for**: Monthly indicators with normal business cycle persistence

**Examples**:
- **ISM Manufacturing**: Business conditions evolve monthly → `signalDecayDays = 30`
- **Initial Claims**: Labor market trends persist for weeks → `signalDecayDays = 30`
- **Chicago NFCI**: Financial conditions are moderately persistent → `signalDecayDays = 30`
- **Continuing Claims**: Slower labor metric → `signalDecayDays = 60`

**Rationale**: Economic data that reflects evolving conditions over weeks/months.

### Slow Decay (90-180 days after overdue)

**Use for**: Persistent structural trends and long-cycle indicators

**Examples**:
- **Sahm Rule**: Unemployment changes very slowly → `signalDecayDays = 90`
- **GDP**: Quarterly structural data → `signalDecayDays = 90`
- **Yield Curve Inversion**: Signal persists for quarters before recession → `signalDecayDays = 180`
- **Freight Recession**: Industrial cycles are long → `signalDecayDays = 180`

**Rationale**: Structural trends that persist for quarters or years.

### Glacial Decay (365+ days after overdue)

**Use for**: Long-term valuation and structural indicators

**Examples**:
- **Buffett Indicator**: Market cap ratios change very slowly → `signalDecayDays = 180-365`
- **Demographics-based patterns**: Secular trends

**Rationale**: Informational patterns that don't require frequent updates.

---

## Publication Frequency vs Signal Decay: Decision Matrix

| Indicator | Pub Freq | Why | Decay | Why |
|-----------|----------|-----|-------|-----|
| **VIX** | 0 (real-time) | Continuous updates | 7 | Volatility mean-reverts fast |
| **Standing Repo** | 1 (daily) | Fed publishes daily | 14 | Emergency stress is acute |
| **Initial Claims** | 7 (weekly) | Every Thursday | 30 | Labor trends persist weeks |
| **ISM Manufacturing** | 30 (monthly) | 1st business day | 30 | Business conditions monthly |
| **CPI** | 30 (monthly) | Mid-month release | 60 | Inflation is persistent |
| **Sahm Rule** | 30 (monthly) | Monthly unemployment | 90 | Unemployment slow-moving |
| **GDP** | 90 (quarterly) | ~30d after quarter | 90 | Structural, slow-moving |
| **Yield Curve** | 1 (daily) | Daily yield calculation | 180 | Signal persists for quarters |
| **Freight Recession** | 30 (monthly) | Monthly freight data | 180 | Industrial cycles are long |
| **Buffett Indicator** | 90 (quarterly) | Market cap data | 180 | Valuation ratios very slow |

**Key Pattern**: Decay period often equals or exceeds publication frequency, but they measure different things:
- VIX: Pub=0, Decay=7 (fast reaction, fast decay)
- ISM: Pub=30, Decay=30 (monthly update, monthly persistence)
- Yield Curve: Pub=1, Decay=180 (daily update, long persistence)

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
- **Update Frequency**: Weekly (every Wednesday)
- **Threshold**: FSI > 1.5 (extreme stress, historical events: COVID, 2018 selloff)
- **Accuracy**: 2/2 major stress events detected, 1 false positive (2018 volatility spike)

**Step 2: Apply Weight Criteria**

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

- OFR FSI: **Coincident** (reflects current financial stress)
- Lead Time: **0 months**

**Step 5: Set Publication Frequency**

- Published weekly (every Wednesday)
- `publicationFrequencyDays = 7`

**Step 6: Set Decay Period**

- Financial stress is moderately persistent (not as fast as VIX, not as slow as GDP)
- `signalDecayDays = 30` (decays over 30 days once overdue)

**Step 7: Set Confidence**

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
  "publicationFrequencyDays": 7,
  "signalDecayDays": 30,
  "confidence": 0.70
}
```

**Interpretation**:
- Weekly data remains 100% fresh for 7 days
- After 7 days (overdue), signal decays with 30-day half-life
- At 37 days old (30 days overdue), signal is at ~37% strength
- At 67 days old (60 days overdue), signal is at ~14% strength

---

## More Complete Examples

### VIX Deployment L1 (Real-Time Market Data)

```json
{
  "patternId": "vix-deployment-l1",
  "name": "VIX Level 1 Deployment Trigger",
  "weight": 0.75,
  "temporalType": "Coincident",
  "leadTimeMonths": 0,
  "publicationFrequencyDays": 0,   // Real-time
  "signalDecayDays": 7,             // Volatility mean-reverts quickly
  "confidence": 0.80
}
```

**Behavior**: VIX from 1 day ago is already decaying (e^(-1/7) = 86% fresh).

### ISM Manufacturing Contraction (Monthly Survey)

```json
{
  "patternId": "ism-contraction",
  "name": "ISM Manufacturing Below 50",
  "weight": 0.85,
  "temporalType": "Coincident",
  "leadTimeMonths": 0,
  "publicationFrequencyDays": 30,  // Monthly release
  "signalDecayDays": 30,           // Business conditions evolve monthly
  "confidence": 0.85
}
```

**Behavior**: ISM published 15 days ago is 100% fresh. ISM published 50 days ago (20 days overdue) is 51% fresh.

### GDP Growth Acceleration (Quarterly Data)

```json
{
  "patternId": "gdp-acceleration",
  "name": "GDP Growth Above 3% YoY",
  "weight": 0.85,
  "temporalType": "Coincident",
  "leadTimeMonths": 0,
  "publicationFrequencyDays": 90,  // Quarterly release
  "signalDecayDays": 90,           // Structural economic data
  "confidence": 0.80
}
```

**Behavior**: GDP published 45 days ago is 100% fresh (next release in 45 days). GDP published 120 days ago (30 days overdue) is 72% fresh.

### Yield Curve Inversion (Leading, Persistent Signal)

```json
{
  "patternId": "yield-curve-inversion",
  "name": "10Y-2Y Treasury Spread Negative",
  "weight": 0.90,
  "temporalType": "Leading",
  "leadTimeMonths": 12,
  "publicationFrequencyDays": 1,   // Daily calculation
  "signalDecayDays": 180,          // Signal persists for quarters
  "confidence": 0.90
}
```

**Behavior**: Yield curve from yesterday is 100% fresh. Yield curve from 30 days ago (29 days overdue) is 85% fresh (signal is very persistent).

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

### Scenario 4: Publication frequency changes

**Initial**: Weekly data, `publicationFrequencyDays=7`  
**After change**: Monthly data, `publicationFrequencyDays=30`  
**Action**: Update `publicationFrequencyDays=30`, possibly adjust `signalDecayDays` if signal persistence changes  
**Rationale**: Reflects new data availability schedule

---

## Category-Specific Guidelines

### Recession Patterns

**Typical Weights**: 0.75-0.95 (high-stakes, well-studied)  
**Temporal**: Mix of Leading (yield curve) and Coincident (claims, ISM)  
**Publication Freq**: 1-30 days (mostly monthly)  
**Decay**: Moderate to Slow (30-90 days)  
**Notes**: Prioritize proven indicators (Sahm, yield curve) over sentiment

### Liquidity Patterns

**Typical Weights**: 0.65-0.90 (market-based, react quickly)  
**Temporal**: Mix of Leading (credit spreads) and Coincident (VIX)  
**Publication Freq**: 0-7 days (real-time to weekly)  
**Decay**: Fast to Moderate (7-30 days)  
**Notes**: Context-dependent signals (VIX) may have variable weights

### NBFI Patterns

**Typical Weights**: 0.75-0.90 (shadow banking is critical)  
**Temporal**: Primarily Coincident  
**Publication Freq**: 1-30 days (daily to monthly)  
**Decay**: Moderate (14-30 days)  
**Notes**: Composite indicators (NBFI Escalation) can have high weights

### Growth Patterns

**Typical Weights**: 0.70-0.85 (expansion signals)  
**Temporal**: Mix of Leading (permits) and Coincident (ISM)  
**Publication Freq**: 30 days (mostly monthly)  
**Decay**: Moderate to Slow (30-90 days)  
**Notes**: Real data (production, sales) over surveys

### Inflation Patterns

**Typical Weights**: 0.70-0.90 (Fed policy driver)  
**Temporal**: Primarily Lagging (CPI, PCE)  
**Publication Freq**: 30 days (monthly)  
**Decay**: Moderate to Slow (60-90 days)  
**Notes**: Breakevens are Leading, CPI/PCE are Lagging

### Valuation Patterns

**Typical Weights**: 0.55-0.70 (informational, not predictive)  
**Temporal**: Lagging  
**Publication Freq**: 30-90 days  
**Decay**: Slow to Glacial (90-180 days)  
**Notes**: Lower weights due to poor market timing

---

## Peer Review Checklist

Before committing weight assignments:

- [ ] Weight justified using 4 criteria (accuracy, timeliness, clarity, quality)
- [ ] Weight within appropriate range for category
- [ ] Temporal classification matches signal characteristics
- [ ] **Publication frequency set to actual data release schedule**
- [ ] **Decay period reflects signal persistence (not publication frequency)**
- [ ] **Both parameters documented with rationale**
- [ ] Confidence score documented with rationale
- [ ] Comparison to similar patterns in category
- [ ] Historical accuracy data cited (if available)
- [ ] Adjustments documented in pattern metadata

---

## Common Mistakes to Avoid

### ❌ Mistake 1: Conflating Publication Frequency with Decay

**Wrong**: "ISM is monthly, so `signalDecayDays = 30`"  
**Right**: "ISM publishes monthly (`publicationFrequencyDays = 30`) AND business conditions persist monthly (`signalDecayDays = 30`)"

These happen to be equal for ISM, but they're conceptually different!

### ❌ Mistake 2: Real-Time Data Without Immediate Decay

**Wrong**: VIX with `publicationFrequencyDays = 1` (treating it as daily)  
**Right**: VIX with `publicationFrequencyDays = 0` (it's continuous, decays immediately)

### ❌ Mistake 3: Ignoring Signal Persistence

**Wrong**: Yield curve with `signalDecayDays = 1` (just because it updates daily)  
**Right**: Yield curve with `signalDecayDays = 180` (signal persists for quarters)

### ❌ Mistake 4: Over-Decaying Quarterly Data

**Wrong**: GDP published 45 days ago treated as stale  
**Right**: GDP published 45 days ago is 100% fresh (next release in 45 days)

---

## References

- [Pattern Weighting Implementation Spec](./pattern-weighting-temporal-metadata.md)
- [Signal Decay Reference](./pattern-signal-decay-reference.md)
- [ThresholdEngine Documentation](../ThresholdEngine/README.md)
- [ATLAS Architecture](../docs/ARCHITECTURE.md)
