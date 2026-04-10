# Sentinel: First-Class Intelligence Network

## Executive Summary

ATLAS is a **macro engine** with six equal data collectors:

| Collector | Source Type |
|-----------|-------------|
| FredCollector | Government economic (FRED API) |
| OfrCollector | Government financial (OFR API) |
| FinnhubCollector | Market data & sentiment (Finnhub API) |
| AlphaVantageCollector | Commodities (Alpha Vantage API) |
| NasdaqCollector | Precious metals (Nasdaq Data Link) |
| SentinelCollector | Alternative intelligence (RSS, scraping, search) |

**None is more important than the others.** VIX matters as much as UNRATE. Copper prices matter as much as GDP.

Sentinel completes the collector ecosystem by gathering non-API intelligence - data that can't be obtained through traditional financial APIs. It is not a validation service; it is an **independent intelligence network**.

**Why Sentinel matters**: Government data (FRED, OFR) is:
- Revised months later (often significantly)
- Subject to political interference (BLS commissioner firing, 2025)
- Unavailable during shutdowns (44-day blackout, Oct-Nov 2025)
- Published with significant lag (CPI: monthly, GDP: quarterly)

Sentinel fills these gaps with real-time, unrevised, non-governmental intelligence.

---

## Alignment with README.md Core Objectives

### Core Objective #6: Data Independence
> "Self-hosted infrastructure reducing reliance on external data providers and government sources"

This is the foundational principle. Sentinel operationalizes data independence by collecting from sources the government doesn't control.

### Section 5: Alternative Data Integration (Planned - Q2-Q3 2026)
The README already envisions this:
- Truflation (real-time inflation)
- Indeed Job Postings (labor demand)
- Mastercard SpendingPulse (consumer spending)
- FreightWaves SONAR (freight/logistics)
- RSS Feed Monitoring
- Web Scraping Infrastructure

**Sentinel is the implementation vehicle for Section 5.**

---

## The Problem: Government Data Reliability Crisis (2025)

### Payroll Revisions in 2025

| Report | Revision | Net Change |
|--------|----------|------------|
| Jan 2025 | Nov + Dec 2024 | **+100,000** (benchmark) |
| Mar 2025 | Jan: -14K, Feb: -34K | **-48,000** |
| Aug 2025 | Jun + Jul combined | **-21,000** |
| Sep 2025 | Jul: -7K, Aug: -26K | **-33,000** |
| Nov 2025 | Aug: -22K, Sep: -11K | **-33,000** |

**Pattern**: Nearly every report revised prior months *downward*. Employment was systematically weaker than initially reported.

### Political Interference
- Trump fired BLS Commissioner Erika McEntarfer (Aug 1, 2025)
- Nominee Antoni threatens to politicize nonpartisan data agency
- Data collection processes potentially compromised

### Shutdown Data Blackout
- 44-day government shutdown (Oct 1 - Nov 12, 2025)
- No October household survey data collected
- CPI data collection resumed Nov 14, 2025
- Markets flew blind for 6+ weeks

### Indicator Reliability Tiers

| Tier | Indicator | Revisions | Trust Level |
|------|-----------|-----------|-------------|
| **High** | Initial Claims (ICSA) | Minimal | Same-week reliable |
| **High** | VIX, Treasury Yields | None | Real-time market |
| **Medium** | CPI | 5-year seasonal | Initial usually close |
| **Low** | Payrolls (PAYEMS) | Monthly, significant | Wait 2 months |
| **Low** | GDP | 3 releases | Wait 3 months |
| **Political Risk** | All BLS data | Unknown | Commissioner fired |

---

## Sentinel Intelligence Sources

### Tier 1: High-Value Targets (Implement First)

| Source | Series ID | Frequency | Value Proposition |
|--------|-----------|-----------|-------------------|
| **Challenger Job Cuts** | `CHALLENGER_JOB_CUTS` | Monthly | Company announcements, no revision possible |
| **ADP Employment** | `ADP_EMPLOYMENT` | Monthly | Released 2 days before NFP, different methodology |
| **Baltic Dry Index** | `BDIY` | Daily | Actual shipping contracts, no survey bias |
| **Truflation CPI** | `TRUFLATION_CPI` | Daily | Transaction-based inflation, not surveys |

### Tier 2: Labor Market Intelligence

| Source | Series ID | Frequency | Value Proposition |
|--------|-----------|-----------|-------------------|
| **Indeed Job Postings** | `INDEED_POSTINGS` | Weekly | Real-time labor demand |
| **LinkedIn Hiring Rate** | `LINKEDIN_HIRING` | Monthly | Professional labor market |
| **Glassdoor Sentiment** | `GLASSDOOR_SENTIMENT` | Monthly | Worker sentiment |

### Tier 3: Economic Activity

| Source | Series ID | Frequency | Value Proposition |
|--------|-----------|-----------|-------------------|
| **FreightWaves SONAR OTVI** | `SONAR_OTVI` | Daily | Outbound tender volume |
| **FreightWaves SONAR OTRI** | `SONAR_OTRI` | Daily | Outbound tender rejection |
| **Mastercard SpendingPulse** | `MC_SPENDING` | Weekly | Consumer spending proxy |
| **Zillow Home Value Index** | `ZHVI` | Monthly | Real-time housing |

### Tier 4: Fed & Policy Intelligence

| Source | Series ID | Frequency | Value Proposition |
|--------|-----------|-----------|-------------------|
| **Fed Speech Sentiment** | `FED_SENTIMENT` | Per-event | FOMC communication tone |
| **FOMC Minutes Hawkish Score** | `FOMC_HAWKISH` | 8x/year | Policy direction |
| **CME FedWatch** | `FEDWATCH_PROB` | Daily | Market rate expectations |

---

## Architecture: Equal Peer Model

ATLAS is a **macro engine**. All data collectors are equal peers - none privileged over others.

| Collector | Data Type | Examples | Value |
|-----------|-----------|----------|-------|
| **FredCollector** | Government economic | UNRATE, GDP, CPI, ICSA | Official stats (lagging, revised) |
| **OfrCollector** | Government financial | FSI, STFM repo rates | Systemic risk monitoring |
| **FinnhubCollector** | Market prices & sentiment | VIX, stock quotes, earnings | Real-time market signals |
| **AlphaVantageCollector** | Commodities | WTI, Brent, Natural Gas | Energy/inflation proxy |
| **NasdaqCollector** | Precious metals | LBMA Gold AM/PM | Safe haven flows |
| **SentinelCollector** | Alternative intelligence | Challenger, Baltic Dry | Non-government sources |

```
┌─────────────────────────────────────────────────────────────────┐
│                    DATA COLLECTORS (Equal Peers)                │
├─────────────────────────────────────────────────────────────────┤
│  FredCollector      │  Government economic data                 │
│  OfrCollector       │  Government financial stability           │
│  FinnhubCollector   │  Market prices, sentiment, earnings       │
│  AlphaVantage       │  Commodity prices                         │
│  NasdaqCollector    │  Precious metals prices                   │
│  SentinelCollector  │  Alternative intelligence                 │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
                    ┌─────────────────┐
                    │   SecMaster     │
                    │  (Series IDs)   │
                    └─────────────────┘
                              │
                              ▼
                    ┌─────────────────┐
                    │ ThresholdEngine │
                    │   (Patterns)    │
                    └─────────────────┘
```

**Key principle**: VIX from Finnhub is as important as UNRATE from FRED. Copper from AlphaVantage matters as much as GDP. The Cu/Au ratio pattern already demonstrates this - commodity data signals economic regime.

**Existing cross-source patterns**:
- `cu-au-ratio` - Copper/Gold from AlphaVantage + Nasdaq
- `vix-deployment-l1/l2` - VIX from Finnhub
- `equity-risk-premium` - Market data vs Treasury yields

Sentinel completes the picture by adding non-API intelligence. Divergence between any sources is a signal.

---

## Current State: What Exists

### Sentinel Infrastructure (Complete)
- Multi-source collection (RSS, SearXNG, Cloudflare edge workers)
- LLM extraction pipeline (Chain of Density + Chain of Verification)
- SecMaster resolution for observations
- gRPC streaming to ThresholdEngine
- Event publishing and storage
- Review workflow for human validation

### The Gap: Series ID Mismatch

**Sentinel currently generates dynamic series IDs:**
```csharp
// Current behavior in EventPublisher.cs
var seriesId = observation.InstrumentId?.ToString()
    ?? $"sentinel:{observation.Source}:{observation.Description...}";

// Produces: sentinel:rss:challenger-report-layoffs-continue-...
// Or: 3fa85f64-5717-4562-b3fc-2c963f66afa6 (SecMaster GUID)
```

**But patterns expect stable, known IDs:**
```json
{
  "requiredSeries": ["UNRATE", "ICSA", "GDP"]
}
```

**Zero patterns currently reference Sentinel data.**

---

## Implementation Plan

### Phase 1: Series Registry (Week 1-2)

**Objective**: Establish stable series IDs for alternative data sources.

**Option A: SecMaster Integration** (Recommended)
- Add `SeriesType` enum: `Official`, `Alternative`, `Derived`
- Register alternative series in SecMaster with stable IDs
- Sentinel resolves to SecMaster ID on extraction

**Option B: Sentinel-Local Config**
- JSON config files mapping extraction rules to target series IDs
- Simpler, faster to implement
- Less integration but works immediately

**Files to modify:**
- `SecMaster/src/Entities/Instrument.cs` - Add SeriesType
- `SecMaster/src/Data/Migrations/` - New migration
- `SentinelCollector/src/Publishers/EventPublisher.cs` - Use stable IDs

### Phase 2: Collection Configuration (Week 2-3)

**Objective**: Configure Sentinel to collect Tier 1 sources.

**Challenger Job Cuts:**
```json
{
  "name": "Challenger Job Cuts",
  "targetSeriesId": "CHALLENGER_JOB_CUTS",
  "sources": [
    {"type": "rss", "url": "https://www.challengergray.com/feed"},
    {"type": "search", "query": "Challenger Gray job cuts report site:challengergray.com"}
  ],
  "extractionPrompt": "Extract the total announced job cuts...",
  "frequency": "monthly",
  "expectedReleaseDayOfMonth": 5
}
```

**Baltic Dry Index:**
```json
{
  "name": "Baltic Dry Index",
  "targetSeriesId": "BDIY",
  "sources": [
    {"type": "search", "query": "Baltic Dry Index today"},
    {"type": "rss", "url": "https://www.hellenicshippingnews.com/feed"}
  ],
  "frequency": "daily"
}
```

**Files to create/modify:**
- `SentinelCollector/config/collections/` - New directory
- `SentinelCollector/src/Entities/CollectionConfig.cs` - New entity
- `SentinelCollector/src/Workers/` - Scheduled collection worker

### Phase 3: Pattern Category - Divergence (Week 3-4)

**Objective**: Create patterns that detect divergence between sources.

**New category**: `Divergence`

**Example: challenger-vs-payroll**
```json
{
  "patternId": "challenger-vs-payroll",
  "name": "Challenger vs BLS Payroll Divergence",
  "category": "Divergence",
  "expression": "var challenger = ctx.GetYoY(\"CHALLENGER_JOB_CUTS\"); var payroll = ctx.GetMoM(\"PAYEMS\"); return challenger > 50 && payroll > 0;",
  "signalExpression": "var challenger = ctx.GetYoY(\"CHALLENGER_JOB_CUTS\") ?? 0m; return challenger > 100 ? -2m : challenger > 50 ? -1m : 0m;",
  "requiredSeries": ["CHALLENGER_JOB_CUTS", "PAYEMS"],
  "description": "Challenger announces mass layoffs while BLS reports job gains - indicates unreported labor market stress or upcoming revision."
}
```

**Example: truflation-vs-cpi**
```json
{
  "patternId": "truflation-vs-cpi",
  "name": "Truflation vs Official CPI Divergence",
  "category": "Divergence",
  "expression": "var truflation = ctx.GetLatest(\"TRUFLATION_CPI\"); var cpi = ctx.GetLatest(\"CPIAUCSL\"); var divergence = Math.Abs((truflation ?? 0) - (cpi ?? 0)); return divergence > 1.0m;",
  "requiredSeries": ["TRUFLATION_CPI", "CPIAUCSL"],
  "description": "Real-time inflation estimate diverges significantly from official CPI."
}
```

**Files to modify:**
- `ThresholdEngine/src/Enums/PatternCategory.cs` - Add Divergence
- `ThresholdEngine/config/patterns/divergence/` - New directory
- Pattern JSON files for each divergence signal

### Phase 4: Standalone Alternative Patterns (Week 4-5)

**Objective**: Patterns using only alternative data (no government dependency).

**Example: baltic-freight-recession**
```json
{
  "patternId": "baltic-freight-recession",
  "name": "Baltic Dry Index Collapse",
  "category": "Recession",
  "expression": "var bdiy = ctx.GetLatest(\"BDIY\"); return bdiy < 600;",
  "requiredSeries": ["BDIY"],
  "description": "Baltic Dry Index below 600 indicates severe global trade contraction."
}
```

**Example: challenger-layoff-surge**
```json
{
  "patternId": "challenger-layoff-surge",
  "name": "Challenger Layoff Surge",
  "category": "Recession",
  "expression": "var cuts = ctx.GetLatest(\"CHALLENGER_JOB_CUTS\"); return cuts > 100000;",
  "requiredSeries": ["CHALLENGER_JOB_CUTS"],
  "description": "Monthly job cut announcements exceed 100,000 - recession-level layoffs."
}
```

### Phase 5: Dashboard & Alerting (Week 5-6)

**Objective**: Visualize and alert on divergence signals.

**Grafana Dashboard: Alternative Data**
- Panel 1: Official vs Alternative comparison (dual-axis)
- Panel 2: Divergence magnitude over time
- Panel 3: Data freshness (government vs alternative)
- Panel 4: Revision tracking (original vs revised)

**Prometheus Alerts:**
```yaml
groups:
  - name: divergence
    rules:
      - alert: ChallengerPayrollDivergence
        expr: thresholdengine_pattern_triggered{pattern_id="challenger-vs-payroll"} == 1
        for: 1h
        labels:
          severity: warning
        annotations:
          summary: "Challenger job cuts diverge from BLS payroll data"
```

---

## Success Metrics

### Data Coverage
- [ ] 4+ Tier 1 alternative series collecting daily/monthly
- [ ] Stable series IDs registered in SecMaster or config
- [ ] Historical backfill for pattern calibration

### Pattern Coverage
- [ ] 5+ divergence patterns operational
- [ ] 3+ standalone alternative data patterns
- [ ] All patterns have unit tests

### Operational
- [ ] Grafana dashboard showing official vs alternative
- [ ] Alerts configured for significant divergences
- [ ] < 24 hour lag on daily alternative data
- [ ] < 48 hour lag on monthly releases (Challenger, ADP)

---

## Risk Considerations

### Data Quality
- Alternative sources may have their own biases
- Extraction accuracy depends on LLM quality
- Human review queue for validation

### Legal/Terms of Service
- Respect robots.txt and rate limits
- Some sources may require API agreements
- Document data provenance

### False Positives
- Divergence doesn't always mean government data is wrong
- Methodological differences explain some gaps
- Require sustained divergence (not one-off)

---

## References

- README.md Section 5: Alternative Data Integration
- README.md Core Objective #6: Data Independence
- `tasks/sentinel-product-spec-v2.md` - Original Sentinel spec
- `SentinelCollector/src/` - Current implementation
- `ThresholdEngine/config/patterns/` - Existing pattern structure
