# ATLAS - Adaptive Tactical Liquidity & Asset System

## Overview

ATLAS is a systematic macro regime identification framework designed to optimize portfolio allocation across all phases of the economic cycle. The system combines quantitative indicator tracking with rules-based decision protocols to guide tactical allocation between growth and defensive positioning based on objective economic assessment.

**Framework Purpose**: Economic cycle classification, regime change detection, and systematic allocation optimization  
**Architecture**: Event-driven data collection with rules-based decision engine  
**Philosophy**: Objective, data-driven analysis equally sensitive to both risk and opportunity signals  
**Core Principle**: Let the data determine positioning - no predetermined bias toward defensive or aggressive allocation

## Table of Contents

- [Documentation Structure](#documentation-structure)
- [Core Objectives](#core-objectives)
- [Framework Architecture](#framework-architecture)
  - [Macro Scoring Methodology](#macro-scoring-methodology)
  - [Indicator Categories with Symmetric Signals](#indicator-categories-with-symmetric-signals)
  - [Decision Protocols](#decision-protocols)
  - [Rebalancing Framework](#rebalancing-framework)
- [Project Components](#project-components)
  - [FredCollector - Economic Data Collection](#1-fredcollector---economic-data-collection)
  - [AlphaVantageCollector - Commodity Data](#1b-alphavantgecollector---commodity-data-collection)
  - [NasdaqCollector - LBMA Gold Prices](#1c-nasdaqcollector---lbma-gold-price-collection)
  - [FinnhubCollector - Equity & Sentiment](#1d-finnhubcollector---equity--sentiment-collection)
  - [CalendarService - Market & Economic Schedules](#1e-calendarservice---market--economic-schedules)
  - [ThresholdEngine - Pattern Evaluation & Regime Detection](#2-thresholdengine---pattern-evaluation--regime-detection)
  - [AlertService - Notification Delivery](#3-alertservice---notification-delivery)
  - [Analysis Tools](#4-analysis-tools-planned---q1-q2-2026)
  - [Alternative Data Integration](#5-alternative-data-integration-planned---q2-q3-2026)
- [Technical Stack](#technical-stack)
- [Decision Framework](#decision-framework)
  - [Operating Modes](#operating-modes)
  - [Data Verification Protocol](#data-verification-protocol-critical)
  - [Framework Rules](#framework-rules)
  - [Execution Patterns](#execution-patterns)
- [Architecture](#architecture)
- [Repository Structure](#repository-structure)
- [Getting Started](#getting-started)
  - [Prerequisites](#prerequisites)
  - [Quick Start (FredCollector)](#quick-start-fredcollector)
- [Development Philosophy](#development-philosophy)
  - [Container-First Development](#container-first-development)
  - [Engineering Principles](#engineering-principles)
  - [Code Standards](#code-standards)
- [Project Timeline](#project-timeline)
  - [Historical Evolution](#historical-evolution)
  - [Development Roadmap](#development-roadmap)
- [Quantitative Thresholds Reference](#quantitative-thresholds-reference)
  - [Key Market Thresholds](#key-market-thresholds)
- [Contributing](#contributing)
- [License](#license)

---

## Documentation Structure

- **[docs/EXECUTIVE-SUMMARY.md](./docs/EXECUTIVE-SUMMARY.md)** - Quick reference for services, ports, patterns
- **[docs/ARCHITECTURE.md](./docs/ARCHITECTURE.md)** - Microservices design decisions
- **[STATE.md](./STATE.md)** - Current system status and epic tracking
- **[CLAUDE.md](./CLAUDE.md)** - AI assistant code generation rules
- **Service READMEs** - Each service has detailed documentation in its directory

## Core Objectives

1. **Macro Regime Identification**: Classify current economic state across six-zone spectrum (Crisis → Recession → Late Cycle → Neutral → Recovery → Growth)
2. **Cycle Transition Detection**: Identify inflection points and regime changes in real-time through leading indicator analysis
3. **Opportunity Recognition**: Signal undervaluation, recovery phases, and growth acceleration for tactical equity deployment
4. **Risk Management**: Monitor recession probability, financial stress, and systemic risk for defensive positioning when warranted
5. **Systematic Decision Protocols**: Quantitative allocation frameworks with symmetric rules for both offensive and defensive moves
6. **Data Independence**: Self-hosted infrastructure reducing reliance on external data providers and government sources

## Framework Architecture

### Macro Scoring Methodology

ATLAS aggregates indicators across six weighted categories to produce a composite macro score that ranges from -30 (severe crisis) to +30 (strong growth):

```
Composite_Score = Σ(Category_Score × Weight)

Categories:
- Recession Indicators (30% weight): -10 to +10 range
- Liquidity Metrics (20% weight): -10 to +10 range
- Growth Indicators (20% weight): -10 to +10 range
- Financial Stress/NBFI (10% weight): -10 to +10 range
- Currency/Risk Sentiment (10% weight): -5 to +5 range
- Inflation Signals (10% weight): -5 to +5 range
```

**Score Interpretation & Allocation Ranges**:

| Regime | Score Range | Defensive % | Growth % | Typical Conditions |
|--------|-------------|-------------|----------|-------------------|
| **Crisis** | < -20 | 80-90% | 10-20% | Credit freeze, systemic stress, severe recession |
| **Recession** | -20 to -10 | 70-80% | 20-30% | Contracting GDP, rising unemployment, negative earnings |
| **Late Cycle** | -10 to 0 | 55-70% | 30-45% | Slowing growth, elevated valuations, tightening credit |
| **Neutral** | 0 to +10 | 40-55% | 45-60% | Stable growth, moderate valuations, balanced risks |
| **Recovery** | +10 to +20 | 20-40% | 60-80% | Accelerating growth, improving employment, earnings recovery |
| **Growth** | > +20 | 10-20% | 80-90% | Strong GDP, full employment, expanding credit, low stress |

### Indicator Categories with Symmetric Signals

Each category contains both bearish and bullish indicators, scored on a -2 to +2 scale:

#### Recession Indicators (30% weight)

**Contractionary Signals** (-2 to 0):
- Freight volumes declining (trucking -15%+, Baltic <600, rail carloads -10%+)
- Consumer sentiment <80 (U. Michigan) or confidence <90 (Conference Board)
- Manufacturing contraction (ISM <48, PMI <47, new orders declining)
- Labor market weakening (initial claims >400K, continued claims rising, JOLTS <7M)
- Yield curve inverted 2s10s (recession within 12-18 months historically)
- Corporate earnings declining (negative YoY for 2+ quarters)

**Expansionary Signals** (0 to +2):
- Freight volumes accelerating (Baltic >1500, trucking tonnage >5% YoY, rail growth >3%)
- Consumer sentiment >100, confidence >110 (strength and improving)
- Manufacturing expansion (ISM >52, PMI >53, new orders accelerating for 3+ months)
- Labor market strength (claims <300K sustained, JOLTS >8M, quits rate >2.5%)
- Steep yield curve (2s10s >100bps, positive term premium)
- Corporate earnings growth (>10% YoY for 2+ quarters, guidance upgrades)

#### Liquidity Metrics (20% weight)

**Risk-Off Signals** (-2 to 0):
- VIX >25 sustained (3+ days), realized vol >20%
- Dollar strength (DXY >110, safe-haven flows)
- Credit spreads widening (HY >400bps, IG >150bps, trending wider)
- Fed balance sheet contracting (QT accelerating), M2 growth <0%
- SOFR spikes >50bps above target, repo market stress
- Commodity weakness (Cu/Au <0.18, broad commodity decline)

**Risk-On Signals** (0 to +2):
- VIX <15 sustained, realized vol <12%, complacency indicators present
- Dollar weakness (DXY <95, risk appetite)
- Credit spreads compressing (HY <300bps, IG <100bps, tightening trend)
- Fed balance sheet expanding (QE active), M2 growth >6%
- SOFR stable at target, abundant liquidity, reverse repo facility elevated
- Commodity strength (Cu/Au >0.25, broad-based rally, backwardation)

#### Financial Stress/NBFI (10% weight)

**Stress Signals** (-2 to 0):
- NBFI stress: HY spreads >350bps, CLO spreads widening, private credit distress
- Regional bank weakness: KRE underperforming SPX >10%, deposit flight
- Bankruptcy surge: 2+ events >$500M within 30 days, rising chapter 11 filings
- Fed emergency facilities activated: Standing Repo >$50B, discount window usage spiking
- Money market stress: Prime/government spread >50bps, redemptions accelerating

**Stability Signals** (0 to +2):
- NBFI health: HY spreads <250bps and stable, CLO issuance strong, private credit flowing
- Regional bank strength: KRE outperforming or matching SPX, deposit growth
- Bankruptcy normalcy: <1 event >$500M monthly, chapter 11 filings at historical lows
- Fed facilities dormant: Standing Repo <$10B, discount window minimal usage
- Money market calm: Prime/government spread <20bps, steady inflows

#### Growth Indicators (20% weight)

**Contractionary Signals** (-2 to 0):
- GDP growth <1% (or negative for 2 quarters), GDI diverging downward
- Industrial production declining >3% YoY, capacity utilization <75%
- Retail sales negative YoY nominal (or real declining >2%)
- Semiconductor bookings declining, chip orders/billings ratio <1.0
- Housing starts <1.2M annualized, permits declining, mortgage apps -20%+ YoY

**Expansionary Signals** (0 to +2):
- GDP growth >3%, GDI confirming or stronger, potential output expansion
- Industrial production accelerating >3% YoY, capacity utilization >80%
- Retail sales growing >5% YoY nominal (>3% real), broad-based strength
- Semiconductor bookings accelerating, chip orders/billings >1.10, backlog building
- Housing starts >1.5M annualized, permits rising, mortgage apps positive YoY

#### Valuation Metrics

**Overvaluation Signals** (caution on incremental equity exposure):
- Shiller CAPE >30 (historical 90th percentile)
- Market Cap / GDP >180% (Buffett Indicator stretched)
- Forward P/E >20x when rates >4%, earnings yield <5%
- Equity risk premium <3% (stocks expensive vs bonds)

**Undervaluation Signals** (opportunity for equity deployment):
- Shiller CAPE <20 (historical median or below)
- Market Cap / GDP <120% (reasonable relative to economy)
- Forward P/E <15x or <12x in higher rate environments
- Equity risk premium >5% (stocks attractive vs bonds)

#### Currency/Risk Sentiment (10% weight)

**Risk-Off / Defensive** (-1 to 0):
- DXY >110 (flight to safety)
- EM currencies weakening >10% vs USD
- Gold outperforming equities, hitting new highs
- Bitcoin declining >30% from peak (risk appetite waning)

**Risk-On / Growth** (0 to +1):
- DXY <95 (risk appetite, global growth)
- EM currencies strengthening vs USD
- Commodities outperforming, cyclical strength
- Bitcoin rallying (speculative appetite strong)

#### Inflation Signals (10% weight)

**Deflationary / Disinflationary** (-1 to 0):
- CPI <2% YoY, trending lower, core <2%
- PCE <2%, supercore declining
- Wage growth <3%, labor cost pressures easing
- Commodity prices declining, energy weak

**Inflationary** (0 to +1):
- CPI >4% YoY and rising, core >3%
- PCE >3%, supercore accelerating
- Wage growth >5%, labor costs rising
- Commodity prices surging, energy strong, supply shocks

### Decision Protocols

#### Symmetric Allocation Framework

**Allocation adjustments triggered by sustained macro score movements** (score stable for 30+ days or confirmed by multiple leading indicators):

```mermaid
flowchart TD
    subgraph Input
        MS[Macro Score]
    end

    subgraph Regime Classification
        MS --> |"> +20"| G[🟢 Growth]
        MS --> |"+10 to +20"| R[🔵 Recovery]
        MS --> |"0 to +10"| N[⚪ Neutral]
        MS --> |"-10 to 0"| LC[🟡 Late Cycle]
        MS --> |"-20 to -10"| RC[🟠 Recession]
        MS --> |"< -20"| CR[🔴 Crisis]
    end

    subgraph Allocation Targets
        G --> GA["Defensive: 10-20%<br/>Equity: 80-90%<br/>Focus: Cyclicals, small caps, EM"]
        R --> RA["Defensive: 20-40%<br/>Equity: 60-80%<br/>Focus: Quality growth, financials"]
        N --> NA["Defensive: 40-55%<br/>Equity: 45-60%<br/>Focus: Balanced, diversified"]
        LC --> LCA["Defensive: 55-70%<br/>Equity: 30-45%<br/>Focus: Min vol, dividends"]
        RC --> RCA["Defensive: 70-80%<br/>Equity: 20-30%<br/>Focus: Treasuries, quality bonds"]
        CR --> CRA["Defensive: 80-90%<br/>Equity: 10-20%<br/>Focus: T-bills, cash"]
    end
```

**Regime Actions Detail:**

| Regime | Score | Defensive | Equity | Strategy Focus |
|--------|-------|-----------|--------|----------------|
| **Growth** | > +20 | 10-20% | 80-90% | Cyclicals, small caps, international, EM. Deploy cash over 60-90 days |
| **Recovery** | +10 to +20 | 20-40% | 60-80% | Quality growth, financials, industrials. Scale into risk assets |
| **Neutral** | 0 to +10 | 40-55% | 45-60% | Balanced portfolio, diversified sectors, quality bias |
| **Late Cycle** | -10 to 0 | 55-70% | 30-45% | Min volatility, dividends, short duration. Build cash reserves |
| **Recession** | -20 to -10 | 70-80% | 20-30% | Treasuries, quality bonds, defensive sectors. Preserve capital |
| **Crisis** | < -20 | 80-90% | 10-20% | T-bills, short-duration treasuries, cash. Maximum preservation |

#### VIX-Based Tactical Deployment (Offensive)

Deploy idle cash into equity when volatility spikes create tactical opportunities:

**Level 1: VIX > 22** (3+ day sustained)
- Deploy: 5-15% of available cash (based on macro score context)
- Target: Quality companies, minimum volatility funds, defensive equity
- If macro score > -5: Lean toward larger deployment (10-15%)
- If macro score < -15: Conservative deployment (5-8%)

**Level 2: VIX > 30** (2+ day sustained)
- Deploy: 20-40% of available cash (systematic scale-in)
- Target: Broad market index funds, quality growth, oversold names
- If macro score > 0: Maximum deployment (35-40%)
- If macro score < -10: Moderate deployment (20-25%)

**Deployment Constraints**:
- Maximum per event: 50% of available cash
- Minimum cash retention: 25% of portfolio always held
- Scale-in over 5-10 trading days (not lump sum)
- Rebalance to target allocation within 90 days post-deployment

#### NBFI Stress Escalation (Defensive)

Move to 70-75% defensive if 2+ of the following trigger simultaneously:

**Trigger Conditions**:
1. ⚠️ HY credit spreads > 350bps (critical at >550bps)
2. ⚠️ 2+ bankruptcy events >$500M within 30 days
3. ⚠️ KRE underperforms SPX by >10% over 30 days
4. ⚠️ Fed emergency lending: Standing Repo >$50B or discount window spike
5. ⚠️ CLO AAA spreads >150bps or widening >50bps in 30 days
6. ⚠️ Money market prime/government spread >50bps

**Escalation Actions**:
- Immediate reduction of equity exposure by 10-15 percentage points
- Increase T-bill and short-duration treasury allocation
- Reduce corporate bond exposure (even IG)
- Weekly monitoring until 2+ triggers clear for 30+ days

#### Cycle Position Analysis

Replace "bubble timing" with objective cycle assessment:

**Cycle Analysis Methodology**:

The system tracks multiple dimensions to assess where the economy is within the business cycle:

- **Duration Tracking**: Months elapsed since regime transition, historical cycle length comparison
- **Policy Context**: Fed monetary policy stance (tightening vs easing), fiscal policy direction
- **Historical Analogs**: Pattern matching with past cycles (1987, 1998, 2000, 2007, 2020)
- **Valuation Context**: Multiple metrics (CAPE, Buffett Indicator, forward P/E, equity risk premium)
- **Structural Factors**: Leverage ratios, NBFI system size, reflexive product penetration

**Example Historical Analysis** (2000 Tech Bubble):
- Duration: 18 months of late-cycle signals before peak
- Policy Context: Fed tightening cycle (6.5% terminal rate)
- Valuation: CAPE 44 (extreme), forward P/E 28x
- Structural: Lower leverage (~300% GDP), less NBFI presence
- Outcome: Rapid contraction once Fed broke inflation

**Example Historical Analysis** (1998 LTCM Crisis):
- Duration: Mid-cycle signals, not late-cycle
- Policy Context: Fed easing cycle (emergency cuts)
- Valuation: CAPE 33 (elevated but not extreme)
- Structural: Hedge fund contagion, Russian default trigger
- Outcome: Cycle extension of 18-24 months post-crisis

**Monitoring for Regime Change**:
- **To Neutral/Recovery**: 3+ growth indicators turning positive, macro score sustained >0
- **To Recession**: 2+ recession indicators confirming, macro score sustained <-10
- **To Crisis**: NBFI stress triggers + credit event + Fed emergency action

### Rebalancing Framework

**Time-Based Reviews**:
- **Daily** (5 minutes): VIX, DXY, credit spreads, Cu/Au ratio
- **Weekly** (20 minutes): Full indicator scan, NBFI check, sentiment surveys
- **Monthly** (60 minutes): ISM/PMI release analysis, Fed policy assessment, portfolio rebalance

**Event-Driven Rebalancing**:
- Macro score moves by ±3 points and sustains for 30 days
- VIX triggers fire (automated alerts via Fidelity + ntfy.sh)
- 2+ NBFI stress triggers fire simultaneously
- Major Fed policy shift (unscheduled meeting, emergency action)
- Allocation drift >5% from target due to market moves

**Tax-Optimized Execution Hierarchy**:
1. **401(k) accounts**: Rebalance freely (no tax impact)
2. **IRA/SEP-IRA accounts**: Rebalance freely (tax-deferred)
3. **HSA accounts**: Rebalance freely (tax-free if used for medical)
4. **Roth accounts**: Prioritize for growth positions (tax-free gains)
5. **Brokerage accounts**: Tax-loss harvest, consider holding periods, prioritize qualified dividends
6. **Cash deployment**: Use new capital before selling existing positions

## Project Components

### 1. FredCollector - Economic Data Collection

**Purpose**: Automated collection system for Federal Reserve Economic Data (FRED) API
**Status**: ✅ Production Ready | **Tests**: 378 passing
**Technology**: .NET 9, C# 13, TimescaleDB, Linux containers
**Coverage**: 25 series configured, 800,000+ searchable FRED series

**Core Capabilities**:
- **Multi-Series Collection**: Parallel collection with configurable concurrency
- **Historical Backfill**: Arbitrary time periods for any series
- **Configurable Schedules**: Daily/weekly/monthly/quarterly per series
- **Rate Limiting**: Token bucket implementation (120 req/min FRED compliance)
- **Resilience Patterns**: Polly retry, circuit breaker, timeout for robust collection
- **Time-Series Optimization**: TimescaleDB hypertables for efficient storage and queries
- **REST API**: Downstream consumption of collected data

**Indicator Coverage**: Recession (7 series), Liquidity (6 series), Growth/Valuation (12 series)

[Technical documentation](./FredCollector/README.md)

### 1b. AlphaVantageCollector - Commodity Data Collection

**Purpose**: Automated collection of commodity prices from Alpha Vantage API
**Status**: ✅ Complete
**Technology**: .NET 9, C# 13, TimescaleDB, Linux containers

**Data Coverage**:
- WTI Crude Oil (daily prices)
- Brent Crude Oil (daily prices)
- Natural Gas (daily prices)

**Core Capabilities**:
- Daily commodity price collection via Alpha Vantage API
- gRPC event streaming (shared `ObservationEventStream` contract)
- Full OpenTelemetry observability (traces, metrics, logs)
- Rate limiting for API compliance

[Technical documentation](./AlphaVantageCollector/README.md)

### 1c. NasdaqCollector - LBMA Gold Price Collection

**Purpose**: Automated collection of LBMA gold prices from Nasdaq Data Link
**Status**: ✅ Complete
**Technology**: .NET 9, C# 13, TimescaleDB, Linux containers

**Data Coverage**:
- LBMA Gold Price AM (London AM fixing)
- LBMA Gold Price PM (London PM fixing)

**Core Capabilities**:
- Daily LBMA gold price collection
- gRPC event streaming (shared `ObservationEventStream` contract)
- REST API for data queries
- Full OpenTelemetry observability (traces, metrics, logs)
- Revision detection for price corrections

[Technical documentation](./NasdaqCollector/README.md)

### 1d. FinnhubCollector - Equity & Sentiment Collection

**Purpose**: Automated collection of equity prices, news sentiment, and economic events from Finnhub API
**Status**: ✅ Complete
**Technology**: .NET 9, C# 13, TimescaleDB, Linux containers

**Core Capabilities**:
- Real-time stock quote collection
- News sentiment scoring
- Analyst ratings and recommendations
- Economic calendar event tracking
- Rate limiting (60 req/min)

[Technical documentation](./FinnhubCollector/README.md)

### 1e. CalendarService - Market & Economic Schedules

**Purpose**: Unified source of truth for market holidays, trading hours, and economic events
**Status**: ✅ Complete
**Technology**: .NET 9, C# 13, PostgreSQL, Linux containers

**Core Capabilities**:
- Trading day validation (is market open?)
- Holiday schedule management
- Economic event aggregation and impact rating
- Shared core library for low-latency checks

[Technical documentation](./CalendarService/README.md)

### 1f. SecMaster - Security Master & Instrument Metadata

**Purpose**: Centralized instrument metadata and intelligent source resolution for the ATLAS ecosystem
**Status**: ✅ Complete
**Technology**: .NET 9, C# 13, TimescaleDB, gRPC, Linux containers

**Core Capabilities**:
- Single source of truth for financial instrument definitions
- Intelligent routing to data sources based on context (frequency, latency, preference)
- Collector registration via gRPC (fire-and-forget, idempotent)
- Multi-source resolution with priority ranking and fallbacks
- Fuzzy search with pg_trgm for instrument discovery
- REST API and gRPC services for consumers

**Data Model**:
- Instruments: Symbol, Name, AssetClass, InstrumentType, Frequency, Sector, Exchange
- SourceMappings: Collector, SourceId, Priority, PublicationLag, LastObservation
- Aliases: Alternative symbols for cross-reference

**Resolution Algorithm**:
- Frequency hierarchy (intraday > daily > weekly > monthly > quarterly > annual)
- Filter by max publication lag
- Apply collector preference
- Sort by priority, return primary or highest-ranked source
- Include alternative sources for fallback

[Technical documentation](./SecMaster/README.md)

### 2. ThresholdEngine - Pattern Evaluation & Regime Detection

**Purpose**: Evaluate configurable C# expressions against economic data to detect regime transitions and generate allocation signals
**Status**: ✅ Production Ready | **Tests**: 153 passing | **Patterns**: 40 configured
**Technology**: .NET 9, C# 13, Roslyn, TimescaleDB, Linux containers

**Core Capabilities**:
- **Pattern Evaluation**: Roslyn-based compilation of C# expressions from JSON configuration
- **Context API**: GetLatest, GetYoY, GetMA, GetSpread, GetRatio, GetLowest, GetHighest, IsSustained
- **Signal Generation**: -2 to +2 signal strength with confidence scoring
- **Regime Detection**: Six-state machine (Crisis → Growth) with hysteresis
- **Hot Reload**: Configuration changes without service restart

**Pattern Categories** (40 patterns):
- **Recession** (12): Sahm Rule, yield curve, claims, sentiment, freight
- **NBFI Stress** (9): HY spreads, KRE, bankruptcy, repo facilities, stress indices
- **Liquidity** (8): VIX L1/L2, DXY, credit spreads, Fed liquidity, M2
- **Growth** (5): GDP, industrial production, retail, housing
- **Valuation** (5): CAPE, Buffett indicator, forward P/E, equity risk premium
- **Commodity** (1): Copper/Gold ratio

[Technical documentation](./ThresholdEngine/README.md)

### 3. AlertService - Notification Delivery

**Purpose**: Receives alerts from Prometheus/Alertmanager and dispatches notifications
**Status**: ✅ Production Ready | **Tests**: 15 passing
**Technology**: .NET 9, ASP.NET Core, MailKit, OpenTelemetry

**Core Capabilities**:
- **Channels**: Ntfy push notifications, email (SMTP)
- **Alertmanager Integration**: Standard Prometheus webhook format
- **Severity Routing**: Critical → email+ntfy, Warning → ntfy only
- **State Suppression**: 24-hour repeat intervals for persistent conditions
- **Weekday-Aware**: Prevents weekend spam for business-hours alerts

[Technical documentation](./AlertService/README.md)

### 4. Analysis Tools (Planned - Q1-Q2 2026)

**C# Analysis Engine**:
- **Portfolio Allocation Calculator**: Translate macro score to target allocation
- **Risk Metrics Computation**: Volatility, beta, correlation analysis, max drawdown
- **Tax-Optimized Rebalancing**: Minimize tax impact while achieving target allocation
- **Scenario Analysis**: What-if modeling for macro score changes
- **Stress Testing**: Portfolio impact under historical crisis scenarios

**Rust Performance Tools**:
- **Historical Backtesting**: Test allocation rules against historical data (1980-present)
- **Monte Carlo Simulation**: Statistical distribution of outcomes under various regimes
- **High-Frequency Processing**: Real-time indicator updates and calculations
- **Statistical Analysis**: Correlation decay detection, regime change probability

**Visualization Dashboard**:
- **Real-Time Monitoring**: Live indicator status, macro score trending
- **Historical Comparison**: Current state vs historical analogs (2000, 2007, 2020)
- **Portfolio Allocation**: Visual representation of current vs target allocation
- **Alert Management**: VIX triggers, NBFI stress, threshold breaches
- **Regime Classification**: Visual heatmap of category scores and weights

### 5. Alternative Data Integration (Planned - Q2-Q3 2026)

**Private Sector Data Sources** (diversify beyond government data):
- **Truflation**: Real-time inflation tracking (daily vs monthly CPI lag)
- **Indeed Job Postings**: Real-time labor market proxy (weekly vs monthly BLS lag)
- **Mastercard SpendingPulse**: Consumer spending proxy (weekly vs monthly lag)
- **FreightWaves SONAR**: Real-time freight data (trucking, rail, ocean)
- **Zillow Housing Data**: Real-time housing market metrics

**Web Scraping Infrastructure**:
- **RSS Feed Monitoring**: Macro research, Fed communications, earnings calls
- **Structured Data Extraction**: Key data sources (earnings reports, Fed minutes)
- **Change Detection**: Alert on material changes to monitored sources
- **Rate-Limited Crawling**: Respectful practices, robots.txt compliance

**Machine Learning Components** (exploratory):
- **Correlation Decay Detection**: Identify when historical indicator relationships break down
- **Regime Change Prediction**: Lead indicator combinations for regime transitions
- **Anomaly Detection**: Flag unusual patterns in indicator relationships
- **Natural Language Processing**: Sentiment analysis of Fed communications, earnings calls

## Technical Stack

**Data Collection**:
- .NET 9 (C# 13) for service development
- TimescaleDB (PostgreSQL + time-series extensions)
- containerd/nerdctl for container orchestration
- Multi-stage Dockerfiles (development + runtime)

**Observability**:
- OpenTelemetry (OTLP) for traces, metrics, and logs
- Tempo for distributed tracing backend
- Prometheus for metrics storage and querying
- Loki for log aggregation with trace correlation
- Grafana for unified dashboards (9 dashboards: FredCollector, ThresholdEngine x4, system x2, infrastructure, GPU)
- Serilog for structured logging (correlated with traces)

**Deployment**:
- Linux containers (Ubuntu 24)
- Self-hosted infrastructure (AMD Threadripper 9960X, 128GB RAM)
- nerdctl compose orchestration
- Bash automation scripts

**Development**:
- VS Code Dev Containers (zero environment drift)
- Multi-stage builds (dev environment === production)
- EF Core migrations for schema management
- xUnit + integration tests

**Notifications**:
- ntfy.sh (self-hosted push notifications)
- Email via configurable SMTP (MailKit)
- Extensible notification system for custom alerts

## Decision Framework

### Operating Modes

- **Research (R)**: Investigate new strategies, data sources, and historical analogs
- **Innovate (N)**: Develop new analytical approaches and indicator combinations
- **Plan (P)**: Design implementations, architectures, and deployment strategies
- **Execute (X)**: Monitor indicators, assess regime, make allocation decisions
- **Review (V)**: Evaluate performance, backtest rules, refine framework

### Data Verification Protocol (CRITICAL)

All financial facts require tool verification before use in decision-making:

**Freshness Requirements**:
- Market data (prices, rates, spreads): <1 day
- Economic indicators (GDP, employment, ISM): <30 days
- Corporate earnings and guidance: <90 days
- Macro score components: <7 days for leading indicators

**Confidence Threshold**: 75%+ required for actionable decisions
- Below 75%: Abstain from action, gather more data
- 75-85%: Proceed with caution, use smaller position sizes
- 85-95%: Standard confidence, full position sizing
- 95%+: High confidence, can accelerate execution

**LLM Training Data Limitations**:

When using AI assistants (like Claude) with the ATLAS system, remember:
- LLM training data has a cutoff date (typically 2-12 months behind current date)
- Any market data, prices, indicators, or economic releases after cutoff are unknown to the LLM
- **All financial facts must be tool-verified** regardless of LLM statements
- LLMs may hallucinate outdated or incorrect market data
- Always verify: prices, economic releases, Fed decisions, corporate earnings

### Framework Rules

1. **VERIFY**: Tool → Fresh data → Cross-validate → 75%+ confidence threshold
2. **CITE**: Every fact needs [Source | Date | Confidence level]
3. **ABSTAIN**: If verification fails, provide decision framework without specific recommendation
4. **SYSTEMATIC**: Follow rules-based protocols, eliminate emotional decision-making
5. **REPRODUCIBLE**: All analysis must be verifiable and repeatable by third party
6. **BALANCED**: Equal scrutiny to bullish and bearish signals, no confirmation bias
7. **QUANTITATIVE**: Numerical thresholds for all decisions, no subjective interpretation

### Execution Patterns

```mermaid
flowchart LR
    A[QUERY] --> B[Parse Intent]
    B --> C[Check Freshness]
    C --> D{Data < threshold?}
    D -->|Yes| E[Retrieve Data]
    D -->|No| F[Use Cached Data]
    E --> G[Validate Cross-Sources]
    F --> G
    G --> H[Calculate Confidence]
    H --> I{Confidence >= 75%?}
    I -->|Yes| J[Response]
    I -->|No| K[Abstain]
```

**Example Decision Flow**:
```
User: "Should I deploy cash now?"

1. VERIFY: Check VIX (tool), check macro score (tools), check recent indicators (tools)
2. CROSS-VALIDATE: Multiple sources for VIX, credit spreads, market conditions
3. CALCULATE CONFIDENCE: All sources agree? 95%. Sources conflict? 60-70%.
4. APPLY RULES:
   - IF VIX >22 AND macro score >-5 AND confidence >85% → Deploy 10-15% of cash (Level 1)
   - IF VIX >30 AND macro score >0 AND confidence >85% → Deploy 35-40% of cash (Level 2)
   - IF confidence <75% → Abstain, gather more data
5. RESPOND (example):
   "VIX at 24.3 [Source: Yahoo Finance, 2025-XX-XX, 95% confidence].
    Macro score -4.4 [Last verified: recent]. Level 1 trigger activated.
    Recommendation: Deploy 10-12% of available cash given late cycle macro context.
    Allocation suggestion: 60% minimum volatility, 40% quality dividend."
```

## Architecture

**⚠️ IMPORTANT**: The ATLAS system follows a microservices architecture with clear separation of concerns. See [docs/ARCHITECTURE.md](./docs/ARCHITECTURE.md) for the comprehensive architectural decision record explaining how data collection, threshold evaluation, and alerting are separated into dedicated services.

```mermaid
flowchart TB
    subgraph Data Collection
        FC[FredCollector]
        AV[AlphaVantageCollector]
        NC[NasdaqCollector]
    end

    subgraph Processing
        TE[ThresholdEngine]
    end

    subgraph Notifications
        AS[AlertService]
        NTFY[ntfy.sh]
        EMAIL[Email]
    end

    subgraph Observability
        PROM[Prometheus]
        TEMPO[Tempo]
        LOKI[Loki]
        GRAF[Grafana]
    end

    FC -->|gRPC events| TE
    AV -->|gRPC events| TE
    NC -->|gRPC events| TE
    TE -->|threshold breaches| PROM
    PROM -->|alerts| AM[Alertmanager]
    AM -->|POST /alerts| AS
    AS --> NTFY
    AS --> EMAIL

    FC & AV & NC & TE & AS -.->|traces| TEMPO
    FC & AV & NC & TE & AS -.->|logs| LOKI
    FC & AV & NC & TE & AS -.->|metrics| PROM
    PROM & TEMPO & LOKI --> GRAF
```

**Key Architectural Principles**:
- **Data Collectors** (FredCollector, AlphaVantageCollector, NasdaqCollector, FinnhubCollector, OfrCollector): Poll sources, publish events
- **SecMaster**: Centralized metadata registry, intelligent source resolution
- **ThresholdEngine**: Subscribes to all events, evaluates rules, publishes breaches
- **AlertService**: Delivers notifications (email, push, etc.)
- **Event-Driven**: System.Threading.Channels (MVP) or message queue (production scale)

## Repository Structure

```
ATLAS/
├── docs/                   # Documentation
│   ├── EXECUTIVE-SUMMARY.md
│   ├── ARCHITECTURE.md
│   └── GRPC-ARCHITECTURE.md
├── FredCollector/          # FRED API data collection
├── AlphaVantageCollector/  # Commodity prices (WTI, Brent, NatGas)
├── NasdaqCollector/        # LBMA gold prices
├── FinnhubCollector/       # Stock quotes, sentiment
├── OfrCollector/           # OFR financial stability data
├── CalendarService/        # Market status, trading days
├── SecMaster/              # Instrument metadata & source resolution
├── ThresholdEngine/        # Pattern evaluation & regime detection
├── AlertService/           # Notification dispatch
├── Events/                 # Shared gRPC event contracts
├── OllamaMCP/              # Ollama MCP server (Claude Desktop)
├── ThresholdEngine/mcp/    # Pattern evaluation MCP server
├── FredCollector/mcp/      # FRED data MCP server
├── FinnhubCollector/mcp/   # Finnhub data MCP server
├── OfrCollector/mcp/       # OFR data MCP server
├── SecMaster/mcp/          # SecMaster data MCP server
├── deployment/             # Ansible playbooks, compose templates, configs
├── CLAUDE.md               # AI assistant rules
└── STATE.md                # Current system status
```

## Getting Started

### Prerequisites

- **Container Runtime**: containerd with nerdctl
- **IDE**: VS Code or Cursor with Dev Containers extension
- **Git**: 2.40+
- **FRED API Key**: Free registration at https://fred.stlouisfed.org/
- **TimescaleDB**: 2.14+ (or use provided docker-compose)

### Quick Start

**Production Deployment** (Ansible - Recommended):

```bash
# Clone repository
git clone https://github.com/jpansarasa/ATLAS.git
cd ATLAS/ansible

# Deploy full infrastructure stack
ansible-playbook playbooks/site.yml

# Verify services
sudo systemctl status atlas.service
cd /opt/ai-inference && sudo nerdctl compose ps
```

This deploys all ATLAS services (FredCollector, ThresholdEngine, observability stack, etc.) via Ansible. See [ansible/README.md](./ansible/README.md) for detailed deployment documentation.

**Development** (Dev Containers):

For service-specific development, each service supports VS Code Dev Containers:

```bash
# FredCollector development
cd ATLAS/FredCollector
code .
# Command Palette: "Dev Containers: Reopen in Container"

# ThresholdEngine development
cd ATLAS/ThresholdEngine
code .
# Command Palette: "Dev Containers: Reopen in Container"
```

Dev containers automatically connect to shared infrastructure (TimescaleDB, observability stack) via the default compose network.

See service-specific README files for detailed setup:
- [FredCollector documentation](./FredCollector/README.md)
- [ThresholdEngine documentation](./ThresholdEngine/README.md)
- [Infrastructure documentation](./infrastructure/README.md)
- [Ansible deployment guide](./ansible/README.md)

## Development Philosophy

### Container-First Development

- All development, debugging, and testing happens inside containers
- Dev environment === Production environment (zero configuration drift)
- Multi-stage Dockerfiles with development and runtime targets
- VS Code Dev Containers for full IDE experience in container
- No local SDK installations required
- Consistent behavior across all machines

### Engineering Principles

- **Explicit over Implicit**: No magic strings, use constants/enums
- **Repository Pattern**: Clean separation of data access from business logic
- **Event-Driven Architecture**: Native .NET channels for decoupled components
- **Observability from Day One**: OpenTelemetry, structured logging, tracing
- **Defensive Programming**: Comprehensive error handling, validation, resilience
- **Production-Grade Everywhere**: No "dev-only" shortcuts, treat dev like prod
- **Testable Architecture**: Dependency injection, interface-based design
- **Quantitative Decision-Making**: No gut feelings, all rules have numeric thresholds

### Code Standards

- **C# Conventions**: Microsoft C# coding conventions
- **Documentation**: XML documentation on all public APIs
- **Test Coverage**: Minimum 80% line coverage for business logic
- **Integration Tests**: Database-dependent tests with shared TimescaleDB
- **No Mocking Frameworks**: Simple hand-rolled test doubles for clarity
- **Immutable Data**: Prefer immutable records for domain models
- **Async/Await**: Consistent async patterns, no sync-over-async

## Project Timeline

### Current Status (2025-12-14)

**Phase 1-3: Complete** - All core services production-ready
- 5 data collectors (FRED, Alpha Vantage, Nasdaq, Finnhub, OFR)
- SecMaster security master with intelligent source resolution
- ThresholdEngine with 50+ patterns and regime detection
- AlertService with ntfy + email notifications
- CalendarService for market status and trading day validation
- Full observability stack (Prometheus, Loki, Tempo, Grafana)
- 6 MCP servers for Claude Desktop integration (FRED, ThresholdEngine, Finnhub, OFR, SecMaster, Ollama)

### Future Phases

**Phase 4: Analysis Tools**
- Portfolio allocation calculator
- Tax-optimized rebalancing
- Alternative data integration (Truflation, Indeed, Mastercard SpendingPulse)

**Phase 5: Advanced Analytics**
- Rust backtesting engine
- Monte Carlo simulation
- ML components: correlation decay, regime prediction

**Phase 6: Production Hardening**
- High availability
- Expanded data sources
- Multi-year backtest validation

## Quantitative Thresholds Reference

### Key Market Thresholds

**Copper/Gold Ratio** (economic activity proxy):
- Bear Market: <0.15 (severe contraction)
- Neutral: 0.15 - 0.22 (stable)
- Bull Market: >0.22 (expansion)

**Baltic Dry Index** (global trade):
- Bear Market: <600 (weak demand)
- Neutral: 600 - 1500 (moderate)
- Bull Market: >1500 (strong demand)

**High-Yield Credit Spread** (credit stress):
- Bull Market: <400bps (tight credit)
- Neutral: 400 - 550bps (moderate)
- Bear Market: >550bps (credit stress)

**Russell 2000 / S&P 500 Ratio** (risk appetite):
- Bear Market: <0.50 (defensive positioning)
- Neutral: 0.50 - 0.65 (balanced)
- Bull Market: >0.65 (risk-on)

**Global M2 Growth YoY** (liquidity):
- Tight: <2% (liquidity contraction)
- Neutral: 2% - 6% (moderate)
- Loose: >6% (liquidity expansion)

**Real Interest Rates** (10Y TIPS yield):
- Tight: >2% (restrictive policy)
- Neutral: 0% - 2% (moderate)
- Loose: <0% (accommodative policy)

**Dollar Index (DXY)** (risk sentiment):
- Risk-Off: >110 (flight to safety)
- Neutral: 95 - 110 (balanced)
- Risk-On: <95 (risk appetite)

## Contributing

This is a personal portfolio management system. The code is open for reference and educational purposes, but external contributions are not being accepted at this time.

## License

Proprietary - Personal use only

---

**Disclaimer**: This system is designed for personal portfolio management and educational purposes. It is not investment advice. All investment decisions involve risk. Past performance does not guarantee future results. The framework provides objective data analysis but does not guarantee profitable outcomes. Markets can remain irrational longer than you can remain solvent. Consult qualified financial professionals for investment guidance.

---

**Last Updated**: 2025-12-14 | **Status**: ✅ Production Ready | **Tests**: 640+ passing | **Patterns**: 50+ configured | **Services**: 18 total (5 collectors, SecMaster, ThresholdEngine, AlertService, CalendarService, 6 MCP servers, 4 observability)
