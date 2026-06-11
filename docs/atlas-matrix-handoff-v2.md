# ATLAS Matrix Project — Conversation Handoff (v2)

**Purpose**: Bootstrap document for a fresh Claude conversation. **Supersedes
the v1 handoff.** Captures the working state after a redesign-pass discussion
that retired several v1 assumptions, codified the architectural contract
between components, and resolved most of the high-priority open questions.
Read this before continuing the design work.

**Role discipline**: Claude is acting as a **product owner partner**, not a
coder. The user is the architect making the build calls. Goals and acceptance
criteria are in scope; implementation specifics are not. Honor this.

---

## 1. Who, what, why

The user is the head of platform services at a quantitative asset manager in
Boston. He is personally subject to **trading restrictions** that make
individual stocks and options effectively unworkable as expression vehicles.
He has built a homelab-grade financial intelligence platform (ATLAS) for
personal use and wants to evolve it into a tool that gives him daily
situational awareness of the macro environment.

**The hypothesis**:

> Free public macro data (FRED, OFR) enriched with free unstructured sources
> is rich enough to populate a multi-dimensional matrix of macro signals ×
> macro sectors × time, with leading/coincident/lagging temporal
> relationships expressed as vector relationships through that matrix — and
> that matrix, not any specific recommendation, is what helps him understand
> where the markets are.

The matrix itself is the deliverable. Investment expression is the user's
job once he has the matrix.

**The framing analogy**: *"You are the team helping me count cards at the
blackjack table. I am the human, I will play the hands."* This is the single
most important piece of context. It means:

- The output is **state of play**, never **action**.
- Edge is **probabilistic**, not deterministic. Acceptance can't be "predicts
  returns." It's "produces situational awareness that correlates with real
  shifts often enough to be useful."
- The human stays in the loop on every decision. No auto-rebalancing, no
  recommendations packaged as imperatives.

**Corollary — disagreement is itself the signal.** The matrix's value is in
*surfacing disagreement* among signals and sources, not in producing a
single aggregate score. A coherent matrix (signals aligned, sources
agreeing) reads as "regime is X." An incoherent matrix (signals diverging,
official-vs-unofficial sources at odds) reads as "something has changed
and we don't yet know what." Both are useful states to display. The
reporting layer, the dashboard, and the MCP query surface should all
preserve disagreement rather than flatten it into consensus. This is why
the disconnect-detector framing of Sentinel (§6) and the
correlation-of-vectors validation goal (§3) are load-bearing — they're the
mechanisms that make disagreement *computable* rather than just
*displayable*.

**Cadence**: Daily intelligence rolling up to weekly digest. Not day trading.

---

## 2. The existing ATLAS platform

Public docs: https://github.com/jpansarasa/ATLAS-Docs

ATLAS is event-driven, deployed, and running. The new work is a rework of
this platform, not a greenfield build.

### Architecture, very compressed

- **Five data collectors** (FredCollector, AlphaVantageCollector,
  FinnhubCollector, OfrCollector, SentinelCollector) push observations into
  TimescaleDB and stream events via gRPC.
- **SecMaster** is the centralized instrument and series catalog with
  hybrid search (SQL, fuzzy, vector via pgvector, RAG via llama.cpp runners) and
  source-resolution routing.
- **ThresholdEngine** evaluates hot-reloadable C# pattern expressions
  against streaming observations. Roslyn-compiled, hot-reloadable, with
  Context API DSL: `GetLatest`, `GetYoY`, `GetMoM`, `GetMA`, `GetSpread`,
  `GetRatio`, `IsSustained`, etc.
- **SentinelCollector** is the unstructured-data arm: pulls news from
  SearXNG, RSS feeds, Cloudflare edge workers; runs local-LLM extraction
  (Chain of Verification + Chain of Density) with tooling (Python sandbox,
  Gemini MCP for lookup, SearXNG MCP for websearch); has a human review UI.
- **MCP servers** expose every component to Claude conversations.
- **Full OpenTelemetry stack**: Prometheus, Grafana, Loki, Tempo,
  Alertmanager.
- **Local LLM stack**: vLLM server (GPU), llama.cpp runners (CPU), FinBERT embeddings,
  custom QLoRA fine-tune on Qwen 2.5 32B for financial extraction.

### What's preserved, what's changing, what's net-new

**Preserved as-is**:
- ThresholdEngine engine — Roslyn-compiled C# expressions, hot-reload,
  Context API DSL. The engine is powerful and can be enriched without
  being rewritten.
- Pattern weighting machinery: `weight`, `freshness factor`,
  `temporalType` (L/C/L), `confidence`, `temporalMultiplier`.
- Pattern signal range: ±3.

**Being retired or replaced**:
- The 8 signal categories (Recession, Liquidity, Growth, NBFI, Inflation,
  Currency, Valuation, Commodity) — **deleted entirely.** They mix
  conceptual levels (variables, regimes, institutions, asset classes) and
  don't sit on a coherent axis. No replacement category dimension up
  front; categorization may emerge later from correlation analysis.
- Category-weighted aggregation toward macro score — must be replaced.
  Macro score's role going forward is open.
- The 6-state regime classification — retained for now but its
  relationship to the new matrix is open.

**Net new**:
- Sector dimension on patterns and observations (single-valued).
- Sector-pair lead-lag relationships (expressed as cross-correlation
  through the matrix, *not* as a separate graph data structure — see §3).
- Probable new `macro_observations` table — observations without
  instrument attribution (yield curve change, FOMC pivot, fiscal
  surprise), including qualitative ones (sentiment, lead-lag estimate,
  regime hint).
- Alpha-factor mechanism — observation normalization for redundant reports
  of the same signal.

---

## 3. The matrix specification (working state)

### Shape

- **Rows**: macro signals — pattern outputs from the (reworked)
  ThresholdEngine.
- **Columns**: 11 macro sectors (Energy, Materials, Industrials, Cons.
  Disc., Cons. Staples, Health Care, Financials, Info Tech, Comm.
  Services, Utilities, Real Estate). The eleven sector *names* are
  common-usage industry terminology, not GICS-licensed. The
  classification *mapping* is owned by SecMaster as a custom NAICS
  rollup (see §6, Workstream 1) — no GICS dependency.
- **Cell**: signed float ±3, where sign = direction (defensive vs.
  offensive for that sector) and magnitude = strength.
- **Time**: every cell is a time series at daily granularity. Weekly is
  the human-consumed rollup cadence.
- **L/C/L**: a relational property at two distinct levels — see "Two L/C/L
  concepts" below.

### Cell semantics

A cell value is the projection of a pattern's signal onto a sector,
processed through the existing weighting machinery:

```
cell(pattern, sector) = pattern.signal × sector_weight(pattern, sector)
                      × freshness × temporalMultiplier × confidence × weight
```

Per-sector aggregation reuses the existing weighted-formula structure,
restricted to the sector's column. Each sector gets its own score in ±3.

### Pattern constraint: single output per pattern

Patterns can read multiple inputs but produce **a single output value**. If
a single conceptual signal needs to fan out to sectors at *different lags*
(e.g., housing starts hits Materials today, Real Estate at lag ~6 months),
the answer is **multiple patterns** — one per lag — each reading inputs at
the appropriate offset, each projecting via its own weight vector.

This means the projection mechanism is a **weight vector** (not per-sector
expressions). Lag lives in the pattern's input definition (existing DSL
handles it via `GetMA` and friends), not in the projection step.

Example decomposition of "housing starts" as a multi-sector signal:
- `housing_starts_to_materials` — current housing starts → Materials @ 1.0
- `housing_starts_to_homebuilders` — housing starts at t−60d → Cons Disc
  (homebuilders sub-industry) @ 1.0
- `housing_starts_to_real_estate` — housing starts at t−180d → Real
  Estate @ 1.0

Each is one matrix row, one signal, one output.

### Two L/C/L concepts (do not conflate)

This is the most-important conceptual clarification:

1. **Signal-level temporal classification** — the existing
   `pattern.temporalType` field. Says how a single signal relates to the
   event it predicts (yield curve = leading; ISM = coincident; CPI =
   lagging). **Already exists.** Stays as-is.
2. **Sector-level lead-lag relationships** — expressed as cross-correlation
   between sector vectors at varying time offsets:

   > "Sector column M at time t correlates with sector column I at time
   > t+Δ with strength s"

   Each such relationship is a property of two slices of the matrix
   indexed by a time offset. The collection of these relationships *is*
   the sector-pair "graph," but the primitives stay vector and matrix
   slices — **no node/edge representation, no parallel graph data
   structure.** The graph is a sparse cross-correlation tensor (sector ×
   sector × lag), which is just another view of the matrix.

   The graph is *queried* from the matrix, not separately stored.

### Vectors through the matrix

The matrix has several vector views, all queryable from the same data:

- **Cell time series** — every (signal, sector) pair across time.
  Empirical lead-lag analysis lives here. Validates and updates the
  sector-pair "graph."
- **Sector vectors** — every column (sector) as a high-dim vector across
  (signal × time). Empirical sector clustering — which sectors actually
  move together regardless of taxonomy labels (validates or challenges
  the SecMaster rollup).
- **Signal vectors** — every row (signal) across (sector × time). Signal
  redundancy detection — which signals are saying the same thing in
  different units.

Cell time series is the highest-value query (validates the lead-lag graph
and tells you when to redraw it). The other two are diagnostics.

---

## 4. Three consumption surfaces (all in scope, all consume directly)

The user wants three surfaces of equal importance, **all of which consume
the underlying observations and matrix data directly** — none is downstream
of another:

1. **Data layer** (raw, enriched, curated) accessible via MCP for Claude
   conversations.
2. **Dashboard** — Grafana or custom web app, visualizing matrix state
   and trajectory. Implementation choice deferred.
3. **Reports** — daily, weekly, monthly, generated by system + local LLM.
   Daily ≈ what changed; weekly ≈ current state assessment; monthly ≈
   trajectory and divergences.

**Important**: `macro_observations` is queried directly by all three. The
cell value is one *view* of these observations (the scored view); the news
story is another (the narrative view). Both read the same underlying
substrate.

---

## 5. Architectural contract

This was codified during the redesign and is load-bearing for every
subsequent decision:

- **Collectors gather** — raw observations, no interpretation.
- **SecMaster classifies** — signal identity, sector attribution, dedup
  grouping, signal-family lookup.
- **ThresholdEngine scores** — all calculation, all aggregation, all
  alpha math, all weighting.

**If it is a calculation, it is in the ThresholdEngine.** Period.

This settles where alpha-factor math runs (TE-time), where deduplication
windows are configured (TE), and what Sentinel/SecMaster's contract looks
like (signal identity in, structured observation out, no math).

---

## 6. The three rework workstreams

### Recommended sequencing

SecMaster is the **spine**: its taxonomy decisions are what
SentinelCollector classifies *into* and what ThresholdEngine looks up *by*.

1. **SecMaster taxonomy** — defines the vocabulary
2. **ThresholdEngine pattern schema** — defines what cells the matrix has
3. **SentinelCollector rework** — produces observations annotated to fit
   both

Parallel work is fine after taxonomy is held stable.

### Workstream 1: SecMaster

**Goal**: Add macro-economic sector metadata to instruments, series, and
observations. Establish signal identity for dedup.

**Pillars**:
- **NAICS as classification spine.** Government-standard, granular
  (~1,000 industries at 6-digit level), free, and FRED industry-aggregate
  series are NAICS-coded. This is the load-bearing classification.
- **Custom 11-sector rollup as presentation layer.** ATLAS defines its
  own NAICS → 11-sector mapping, owned by SecMaster, free. The eleven
  sector names (Energy, Materials, Industrials, etc.) are common-usage
  industry terminology and not GICS-licensed; only the maintained
  proprietary mapping is. SecMaster carries a free equivalent.
- **Mapping versioning.** Rollup changes (sector boundary shifts,
  reclassifications) are versioned in SecMaster so historical cells
  stay reproducible across revisions.
- **Instrument-level classification** sources: SEC EDGAR (SIC + NAICS
  codes for filers, free) plus manual overrides for the tracked-name
  universe. No paid sector-data feed required.
- **OpenFIGI for instrument identity.** Cross-referencing
  (CUSIP/ISIN/ticker/FIGI). Different job from classification.

**Tag assignment by object type** (single-valued sector tags only — no
multi-valued sector exposure modeling at v1):

| Object type | Sector tag (ATLAS 11-sector rollup) | Other |
|---|---|---|
| Instrument | Yes — primary, single-valued (manually maintained or derived from EDGAR NAICS/SIC) | — |
| Industry-aggregate series | Yes — deterministic via NAICS → ATLAS rollup | — |
| Pure-macro series | No sector | Carries signal identity only |
| Unstructured observation | Sometimes — only if news is sector-tagged | Signal identity (for dedup) always |

**Sector-pair lead-lag relationships are *not* declared in SecMaster.**
They are derived from cell time-series cross-correlation, queried from the
matrix. SecMaster owns the *taxonomy* (sectors, signal identity), not the
*relationships*.

### Workstream 2: ThresholdEngine

**Goal**: Add sector projection to patterns. Remove categories. Preserve
engine.

**Schema changes**:
- Add `sectorWeights: { ATLASSector: weight }` — sparse-by-default,
  explicit-zero for unmentioned sectors (lean: explicit-zero is more
  honest). `ATLASSector` is one of the 11 macro sectors defined by the
  SecMaster rollup.
- Remove `category` field.
- Keep `temporalType`, `weight`, `freshness`, `confidence`, all
  input-reading DSL functions.
- Single output per pattern enforced. Multi-output decomposes into
  multiple patterns (see §3).

**Aggregation**:
- Per-sector score = same weighted formula as macro score, restricted to
  the sector's column.
- Macro score's role going forward — **OPEN.** With categories deleted,
  the existing weighted-aggregation formula no longer applies and needs
  replacement: keep separate global with new weights, derive from sector
  scores, or retire?
- Per-sector regime classification (11 mini-regimes) vs. single global
  regime — **OPEN.**

**Alpha-factor / observation normalization**:
- Configured per signal (or signal family) in TE.
- Acts as a divisor to convert N redundant observations of the same
  signal into 1 logical signal: `s = (o1 + o2 + … + on) / n`, not
  `s = o × n`.
- Time window for dedup is a TE configuration parameter (lean: per-signal
  configurable burst window).

### Workstream 3: SentinelCollector

**Goal**: Make extraction macro-focused. Sentinel is reframed as a
**disconnect detector + unofficial-channel signal aggregator** — its
product job is to surface where unofficial sources diverge from official
sources, and to capture purely-qualitative-macro signals that no official
series reports.

**Three observation types**:
- **Numeric, instrument-tagged** — today's behavior. Sector tags applied
  via SecMaster lookup.
- **Numeric, macro-series-tagged** — already works (CPI, payrolls, etc.
  matched against existing FRED series).
- **Qualitative, macro-tagged — new.** Sentiment polarity, sector
  affinity, lead-lag estimate, regime hint. Needs an LLM job that's
  different from numeric extraction.

**Sentinel's flow**:
1. Pull from unstructured sources (RSS, news, Cloudflare edge workers,
   future channels).
2. Run LLM extraction with tooling (Python sandbox, Gemini MCP, SearXNG
   MCP).
3. Query SecMaster to resolve signal identity, sector attribution, dedup
   grouping.
4. Persist structured observations into `macro_observations` (proposed
   new table) with that attribution.

**Sentinel exits the math business entirely.** No scoring, no weighting,
no aggregation. Those happen in TE.

**LLM reliability concern (open issue, hard acceptance criterion)**: The
local LLM has not produced reliably-usable signal yet, even with the
LoRA. Possibilities: LoRA training is wrong, RAG is insufficient, prompts
need work, or all three. **Until reliability is established, observations
from Sentinel should not be unconditionally fed to TE.** A separate
validation service or TE-side gating layer is acceptable. The Sentinel
workstream's acceptance criterion is not just "extract and tag," but
"extract reliably enough to score."

---

## 7. Decisions made (constraints; do not relitigate without reason)

| Decision | Resolution |
|---|---|
| ETF / expression layer | Out of scope. Matrix is the deliverable. |
| Output type | Situational awareness, never action. |
| Cadence | Daily intelligence, weekly rollup. |
| Cell range | ±3, signed float (sign = direction, magnitude = strength). |
| Sector taxonomy basis | NAICS spine + custom 11-sector rollup owned by SecMaster. **No GICS dependency** (GICS is licensed/paid). Sector names are common-usage; the mapping is ATLAS-maintained. |
| Sector dimension in matrix | 11 macro sectors as columns (Energy, Materials, Industrials, Cons. Disc., Cons. Staples, Health Care, Financials, Info Tech, Comm. Services, Utilities, Real Estate). |
| Instrument-level sector data | SEC EDGAR (SIC + NAICS for filers, free) plus manual overrides. No paid sector feed. |
| Rollup versioning | SecMaster versions the NAICS → 11-sector mapping so historical cells stay reproducible across boundary changes. |
| Sector tags on instruments | **Single-valued only.** No multi-sector exposure modeling at v1. |
| 8 signal categories | **Deleted entirely.** No replacement category dimension up front. |
| Two L/C/L concepts | Distinct. Signal-level (existing); sector-level expressed as cross-correlation through matrix. |
| Sector-pair "graph" representation | Cross-correlation tensor (sector × sector × lag), *queried* from matrix. No separate graph data structure. |
| Pattern output cardinality | **One output per pattern.** Multi-output cases handled by splitting into multiple patterns. |
| Projection mechanism | Weight vector per pattern. Lag handled by reading lagged inputs in separate patterns, not by per-sector expressions. |
| Alpha factor purpose | Observation normalization for redundant reports of the same signal. Math is `s = (o1 + … + on) / n`, not `s = o × n`. |
| Where alpha math runs | ThresholdEngine (calculation = TE). |
| Architectural contract | Collectors gather. SecMaster classifies. TE scores. **If it is a calculation, it is in TE.** |
| Macro classification at Sentinel | Additive to instrument classification, not replacement. |
| Sentinel role | **Disconnect detector + unofficial-channel signal aggregator** (sharper than v1 "unstructured-data arm"). |
| Sentinel exits math | No scoring, weighting, or aggregation in Sentinel. |
| `macro_observations` table | Probable new table — separate from instrument-tagged observations. |
| Consumption surfaces | Three: MCP/data, dashboard, reports. **All consume `macro_observations` directly**, not only via TE-computed cells. |
| ThresholdEngine engine | Preserved. Schema changes only; no DSL grammar rewrite. |
| Sequencing | SecMaster taxonomy → TE schema → Sentinel rework. Parallel after taxonomy stable. |
| "Sacred cows" | None. Every prior decision is in play during the redesign. |

---

## 8. Open questions, ranked

### High priority (unlock other decisions)

1. **Macro score's role going forward.** Keep as separate global with new
   weights, derive from sector scores, or retire? With categories deleted,
   the v1 weighted-aggregation formula is gone and needs successor.
2. **Per-sector regime (11 mini-regimes) vs. single global regime?** And
   what's the relationship between either of those and the existing
   6-state regime?
3. **`macro_observations` table shape.** Schema for non-instrument
   observations including qualitative ones (sentiment, lead-lag estimate,
   regime hint). Likely qualitative observations need their own schema
   variant within the table.
4. **LLM/LoRA reliability path.** Validation gate, retraining, RAG
   improvements, prompt engineering, or separate validation service.
   Without this, Sentinel observations aren't trusted enough to drive TE.
   Hard acceptance criterion for the Sentinel workstream.

### Medium priority

5. Sparsity policy for sector projections — explicit-zero for unmentioned
   sectors, or assumed-zero? (Lean: explicit-zero is more honest.)
6. Daily / weekly / monthly report structure — each needs a content
   sketch.
7. Qualitative observation type in Sentinel — what does the LLM job look
   like (sector affinity classifier, sentiment polarity, lead-lag
   estimator)? Likely needs its own prompt/adapter separate from numeric
   extraction.
8. CoVe (Chain of Verification) for qualitative classification — does it
   apply where there's no numeric ground truth, or is it numeric-only?
9. Validation triggers — when sector score crosses threshold, trigger
   Sentinel to search for narratives explaining it. Spec this loop.

### Cross-cutting / parked

10. **Validation / acceptance criteria for the matrix** — *parked until
    data is collected, labeled, and queryable.* Three candidate criteria
    once data flow is in place: coherence (cells move together when
    economic priors say they should), disagreement detection
    (official-vs-unofficial divergence flagged before headlines), drift
    detection (pattern predictive value decays visibly over time). These
    can't be defined in advance of data flow.
11. Migration plan — existing macro-score and 6-state regime have
    downstream consumers. Big bang vs. parallel run vs. phased.
12. Backfill — TE has `/api/patterns/backfill` today; the new matrix
    needs the equivalent so dashboards aren't blank for 18 months.
13. Vintage / point-in-time data — not confirmed to exist in ATLAS.
    Important if any backtest is in scope.

---

## 9. Reference links

- Repo: https://github.com/jpansarasa/ATLAS-Docs
- Architecture: https://github.com/jpansarasa/ATLAS-Docs/blob/main/docs/ARCHITECTURE.md
- Pattern reference: https://github.com/jpansarasa/ATLAS-Docs/blob/main/docs/THRESHOLDENGINE-PATTERNS.md
- ThresholdEngine: https://github.com/jpansarasa/ATLAS-Docs/blob/main/ThresholdEngine/README.md
- SecMaster: https://github.com/jpansarasa/ATLAS-Docs/blob/main/SecMaster/README.md
- SentinelCollector: https://github.com/jpansarasa/ATLAS-Docs/blob/main/SentinelCollector/README.md

---

## 10. Suggested next move

The taxonomy work is now mostly settled — single-valued sector tags, no
categories, NAICS spine with a SecMaster-owned 11-sector rollup (no GICS
dependency), sector-pair relationships derived rather than declared.
What's left in SecMaster is mostly ratification and edge cases.

Two highest-leverage threads to pick up next:

1. **`macro_observations` table shape (open #3)** — concrete spec for the
   new observation type, including how qualitative-only observations are
   represented. This is the bridge between Sentinel's reframed role and
   TE's input contract, and it's what the report generator and dashboard
   query directly.
2. **Macro score's role going forward (open #1)** — with categories
   deleted, the v1 weighted aggregation is gone. Pick a successor:
   derived-from-sector-scores, kept-separate-with-new-weights, or retired.

Either is ready for goal/acceptance-criteria definition. Pick one.

---

**End of handoff. Pick up at §10 or wherever the user directs.**
