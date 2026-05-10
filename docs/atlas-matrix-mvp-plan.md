# ATLAS Matrix MVP — Implementation Plan

## Overview

This plan delivers the ATLAS Matrix MVP as defined in
`atlas-matrix-handoff-v2.md`. It covers the three rework workstreams
(SecMaster, ThresholdEngine, SentinelCollector), the matrix itself
(cells, storage, vector queries), and the three consumption surfaces
(MCP, dashboard, reports), plus the `macro_observations` substrate that
links them, and the LLM extraction reliability track that gates trusted
qualitative output from Sentinel.

The deliverable is the matrix: signals × sectors × time, with
leading/coincident/lagging relationships expressed as cross-correlation
vectors through it. Investment expression is the user's job; the matrix
is situational awareness, never recommendation.

This plan does not relitigate handoff §7. Decisions captured below extend
§7 with the architect ratifications received during planning. All
decision gates are closed.

---

## Captured Decisions

### Cross-cutting

**D1. `macro_observations` lives alongside existing collector tables.**
Non-instrument observations write into `macro_observations`;
instrument-attributed observations stay in their existing per-collector
tables. Both substrates feed TE.

**D2. Historical FRED/OFR macro-series data is not migrated at MVP.**
Going-forward writes land in `macro_observations`; historical observations
stay in collector-specific tables. Backfill is parked (§8).

**D3. Sentinel-originated qualitative observations are gated TE-side, not
Sentinel-side.** Trust attribute on every qualitative observation; TE's
read adapter filters at a configurable threshold; below-threshold
observations remain queryable from MCP and dashboard surfaces (visibility
≠ scoring).

**D4. Schema shape for the qualitative variant of `macro_observations`
and the dedup grouping key are developer calls within ACs.** Capture the
choice and rationale in the migration commit.

**D5. Sparsity policy = explicit-zero.** Pattern `sectorWeights` is
sparse-by-default; unmentioned sectors store an explicit zero, not a
null.

**D6. The cell time series is the canonical matrix view.** Sector vectors
and signal vectors are queryable but not separately stored;
cross-correlation tensor is computed on demand from cell time series.

**D7. Single output per pattern.** Multi-output cases decompose into
multiple patterns.

### Architect ratifications

**D8. Macro score retired for MVP.** No global single-number macro score.
Per-sector scores replace it. The dashboard's primary visualization is a
two-axis matrix (sectors × regime phase) with a vector score in each
cell. The Fidelity business-cycle reference grid
(`1778343398312_image.png`) is inspiration; ATLAS's matrix is more
nuanced — real-valued cells, vector contents, no symbolic
discretisation.

**D9. Per-sector regime is the regime classification.** Each of the 11
sectors carries its own regime time series, classified from its sector
score. The legacy global 6-state regime is retired (see D14); no global
aggregate is produced as a primary deliverable.

**D10. Reports use a common content structure: news summary, word cloud,
radar charts (sectors and macro signals).** Daily, weekly, and monthly
cadences share the structure; the time window varies. The news summary
reads `macro_observations` directly per §4.

**D11. Chain of Verification (CoVe) applies to both numeric and
qualitative extraction** to constrain LLM hallucinations.

**D12. Matrix cells and sector scores are real-valued.** Validation
triggers are real-valued threshold crossings (e.g., sector score `> 1.5`
fires a trigger). No discretisation in the matrix substrate; thresholds
are configurable per sector.

**D13. LLM extraction reliability is in scope for MVP.** Dedicated
reliability track; detail in Epic 4 Feature 4.6.

**D14. Fail-forward retirement of legacy macro-score and 6-state regime.**
No parallel run. The current legacy outputs do not produce a usable
signal, so there is nothing to validate parity against. Legacy code paths
are deleted as part of the rework. Downstream consumers cut over to the
new per-sector signals.

**D15. LLM acceptance criteria reuse prior LoRA thresholds.** The
quantitative thresholds defined for the previous LoRA training cycle are
inherited by Feature 4.6. Architect provides the source location
(commit, doc, or values) as a prerequisite to Story 4.6.1.

**D16. LLM iteration budget: 4-week timebox plus a $500 Azure spend cap
for any Opus 4.7 relabeling.** On-prem iteration (LoRA training, prompt
engineering, evaluation) is timeboxed at 4 weeks with weekly checkpoints
— constrained only by time and electricity. If the iteration requires
expanded labelling using Opus 4.7, the Azure budget cap is $500
(approximately 1/3–1/4 of equivalent Anthropic API direct spend).
Failure mode at exhaustion of either budget: architect call on extend /
ship constrained (high trust-gate threshold) / defer.

**D17. Phase-axis size for the dashboard reference matrix is a developer
call.** ATLAS's matrix is more nuanced than the Fidelity reference; the
phase taxonomy aligns with D9's per-sector regime states, but the exact
size and visual design are not pinned by the plan.

---

## Constraints (immutable, from handoff §5 / §7)

- **Calculation = TE.** No scoring, weighting, normalization, or
  aggregation outside TE.
- **Sentinel exits math.** Sentinel persists structured observations and
  trust; nothing more.
- **SecMaster classifies.** Signal identity, sector attribution, dedup
  vocabulary live here. SecMaster owns the NAICS → 11-sector mapping;
  mapping is versioned.
- **Single-valued sector tags.** No multi-sector exposure modeling at v1.
- **No GICS dependency.** Sector names are common-usage; the mapping is
  ATLAS-maintained, free.
- **Versioned-mapping aware.** Observations and cells carry the rollup
  version under which they were tagged.
- **Direct-query substrate.** All three consumption surfaces read
  observations directly, not only via TE-computed cells.
- **DSL grammar preserved.** TE engine stays; schema changes only.
- **One output per pattern.** Multi-output decomposes into multiple
  patterns.
- **Sector range ±3, signed float, real-valued.**
- **Daily granularity, weekly rollup cadence.** Not day-trading;
  situational awareness.
- **Disagreement is the signal.** Surfaces preserve disagreement; do not
  flatten into consensus.

## Out of Scope (parked from handoff §8)

- Backfill of historical observations into the new substrate (#12).
- Point-in-time / vintage data infrastructure (#13).
- Validation / acceptance criteria for the matrix as a predictive
  instrument (#10) — parked until data flow is in place.
- ETF / expression layer; matrix is the deliverable (§7).

---

# Epic 1: SecMaster Sector Taxonomy & Classification

**Goal.** Add macro-economic sector metadata to instruments, series, and
observations. Establish signal identity for dedup. SecMaster becomes the
spine the rest of the rework builds against.

## Feature 1.1: NAICS classification spine

**Description.** Import and store NAICS at 6-digit granularity. NAICS is
the load-bearing classification; the 11-sector rollup is a presentation
layer over it.

### Story 1.1.1: NAICS taxonomy import
- NAICS 6-digit codes loaded into a SecMaster reference table.
- Hierarchy preserved (sector → subsector → industry group → industry
  → national industry).
- A documented data source (Census Bureau bulk file) is the import path;
  the import is repeatable.

### Story 1.1.2: NAICS lookup API
- A SecMaster API method resolves a NAICS code to its hierarchy.
- A SecMaster API method lists all codes under a given prefix.
- Used by Story 1.4.x for instrument classification.

## Feature 1.2: Custom 11-sector rollup

**Description.** ATLAS-owned NAICS → 11-sector mapping. Names are
common-usage industry terminology; mapping is the proprietary asset and
is free (no GICS).

### Story 1.2.1: Initial rollup mapping defined
- Every NAICS 6-digit code maps to exactly one of the 11 sectors
  (Energy, Materials, Industrials, Cons. Disc., Cons. Staples,
  Health Care, Financials, Info Tech, Comm. Services, Utilities,
  Real Estate).
- Mapping stored in a versioned table (see Feature 1.3).
- Initial version published as v1.0.
- Rationale for non-obvious assignments (e.g., REITs, utilities edge
  cases) is captured alongside the mapping.

### Story 1.2.2: Rollup lookup API
- Given a NAICS code, resolve to its rollup sector under a given mapping
  version.
- Default lookup uses current version; historical lookup takes a date
  or version id.

## Feature 1.3: Mapping versioning

**Description.** Rollup boundaries shift over time. Versioning preserves
historical reproducibility.

### Story 1.3.1: Versioning schema
- Mapping versions stored with effective-from / effective-to dates.
- A version is immutable once effective; changes produce new versions.
- Active version at any historical date is unambiguous.

### Story 1.3.2: Version-aware lookup
- The Story 1.2.2 API accepts a date and resolves under the version
  active on that date.
- Test case: a synthetic mapping change produces different rollup
  results across the change date.

## Feature 1.4: Instrument-level classification

**Description.** Instruments in the tracked-name universe carry a
single-valued sector tag. Source: SEC EDGAR (SIC + NAICS for filers,
free), with manual override for edge cases.

### Story 1.4.1: EDGAR ingestion
- A SecMaster job pulls SIC + NAICS codes from EDGAR for filers in
  scope.
- Refresh cadence configurable (default: weekly).
- Provenance stored: source = EDGAR, fetched-at, raw fields.

### Story 1.4.2: Manual override mechanism
- A SecMaster table or CLI accepts manual sector overrides per
  instrument identifier.
- Override takes precedence over EDGAR-derived classification.
- Audit log captures who/when/old/new.

### Story 1.4.3: Instrument sector tag attachment
- Each instrument in the tracked universe carries a single sector tag
  resolved through (override → EDGAR NAICS → rollup).
- Tag references the rollup mapping version that produced it.
- A reclassification report shows instruments whose tag changes when
  the mapping version advances.

## Feature 1.5: OpenFIGI integration for instrument identity

### Story 1.5.1: OpenFIGI lookup integration
- A SecMaster API method resolves any of (CUSIP, ISIN, ticker, FIGI) to
  the canonical identifier set.
- Results cached locally to respect the OpenFIGI free-tier rate limit.
- Used by collector code paths receiving heterogeneous instrument
  identifiers.

## Feature 1.6: Series classification

### Story 1.6.1: Industry-aggregate series tagging
- FRED industry-aggregate series with NAICS codes auto-resolve to
  rollup sector under the active mapping version.
- Tag stored alongside the series metadata.
- Versioning is honored — historical observations resolve under their
  write-time version.

### Story 1.6.2: Pure-macro series — signal identity only
- Pure-macro series carry signal identity (e.g., "CPI-headline-yoy")
  but no sector tag.
- Signal identity is the dedup grouping vocabulary used by Epic 2's
  alpha-factor math.

## Feature 1.7: Signal identity & dedup grouping

### Story 1.7.1: Signal identity catalog
- A SecMaster table holds signal identities with description and any
  alias lists.
- Catalog queryable by Sentinel, FRED collector, OFR collector, and TE.
- Initial catalog seeded with the signals already tracked across
  collectors.

### Story 1.7.2: Dedup grouping API
- Given a signal identity and a time window, return all observations
  across substrates that share the identity within the window.
- Used by TE alpha-factor math (Epic 2 Feature 2.5).

---

# Epic 2: ThresholdEngine Schema Rework

**Goal.** Add sector projection to patterns. Remove categories. Move
alpha-factor math into TE. Introduce per-sector aggregation. Retire the
macro score (D8, D14). Preserve the engine, the DSL, the weighting
machinery, and the ±3 signal range.

## Feature 2.1: Remove `category` field

### Story 2.1.1: Schema migration removing `category`
- `category` removed from pattern schema.
- Existing patterns updated; any pattern that relied on category-driven
  routing is flagged for re-spec under sector projection.
- Migration is rollback-safe.

### Story 2.1.2: Aggregation code path that referenced category retired
- All code that summed by category is removed.
- Tests that asserted category-based aggregation are removed or
  refactored.

## Feature 2.2: Sector projection via weight vector

### Story 2.2.1: `sectorWeights` field in pattern schema
- Each pattern can define `sectorWeights: { ATLASSector: weight }`.
- Schema validation rejects multi-output forms.
- Existing patterns receive a default `sectorWeights` (zeroed across
  all 11 sectors per D5) and are flagged for re-spec.

### Story 2.2.2: Cell projection in evaluation pipeline
- For each evaluated pattern, TE produces 11 cell values (one per
  sector) using:
  `cell(pattern, sector) = pattern.signal × sectorWeights[sector] ×
  freshness × temporalMultiplier × confidence × weight`
- Output is real-valued, range ±3 per cell (D12).
- Cell values emitted as events for downstream persistence (Epic 5).

## Feature 2.3: Single output per pattern enforced

### Story 2.3.1: Validation rejects multi-output patterns
- Pattern schema validation rejects any pattern emitting more than one
  signal value per evaluation.
- Linter flag for the existing pattern set surfaces violations during
  the rework.

### Story 2.3.2: Multi-output legacy patterns split or retired
- Each existing multi-output pattern is decomposed (each new pattern
  gets its own `sectorWeights` and lag) or retired.
- Decomposition is documented per pattern.

## Feature 2.4: Per-sector aggregation

### Story 2.4.1: Per-sector score computation
- For each sector, TE computes a sector score using the existing
  weighted formula restricted to cells in that sector's column.
- Result is real-valued, range ±3 (D12).
- Computed at each evaluation cycle and emitted.

### Story 2.4.2: Sparsity handled honestly
- Sectors with no contributing patterns produce a defined "no signal"
  state (e.g., zero with `coverage = 0` flag), distinguishable from
  zero-by-cancellation.
- Coverage queryable from MCP / dashboard.

## Feature 2.5: Alpha-factor / observation normalization

### Story 2.5.1: Burst window configuration per signal
- Signal-identity-keyed configuration sets the burst window per signal.
- Default 24h documented; overridable per signal.
- Configuration hot-reloadable consistent with existing TE patterns.

### Story 2.5.2: Dedup grouping in TE read adapter
- When reading a signal, TE queries the dedup grouping API
  (Story 1.7.2) and collapses observations within the burst window via
  `s = (o1 + … + on) / n`.
- Observations outside the window remain distinct.
- Dedup is read-time, not write-time.

## Feature 2.6: Macro score retirement (per D8, D14)

### Story 2.6.1: Macro score deleted
- Macro-score code paths deleted, not parallel-run.
- Downstream consumers (dashboards, reports, MCP) updated to read
  per-sector scores.
- No validation against legacy values; legacy outputs were not usable
  signal.

---

# Epic 3: `macro_observations` Substrate

**Goal.** Stand up `macro_observations` as the substrate for the matrix.
Bridge between Sentinel (write) and TE (read), and direct-query substrate
for all three consumption surfaces per §4.

## Feature 3.1: Schema and versioning

### Story 3.1.1: DDL for `macro_observations`
- TimescaleDB hypertable, time-partitioned consistent with existing
  collector tables.
- Schema accommodates numeric and qualitative observations without
  forcing irrelevant fields on either type (D4: developer call,
  documented in commit).
- Provenance fields: source collector, source identifier, extraction
  job id, ingestion timestamp.
- Sector tag nullable; references SecMaster rollup version captured at
  write time.
- Trust attribute non-nullable for qualitative; absent or constant for
  numeric.

### Story 3.1.2: Indexes for primary query patterns
- Indexes covering: signal-identity + time, sector + time, source +
  time for dedup.
- EXPLAIN output for each pattern in the commit message.

### Story 3.1.3: Versioned-mapping query support
- Given a historical date, query resolves under that date's SecMaster
  rollup version (consumes Feature 1.3).

## Feature 3.2: Collector write paths

### Story 3.2.1: FRED vertical slice
- One named macro series flows end-to-end into `macro_observations`.
- Idempotent against `(source, source_id, observation_time)`.

### Story 3.2.2: FRED full coverage
- All in-scope FRED macro series write into `macro_observations`.
- Configuration documents macro vs. instrument-attributed series.

### Story 3.2.3: OFR write path
- OFR collector writes non-instrument observations into
  `macro_observations` mirroring FRED.

### Story 3.2.4: Sentinel qualitative write path
- Sentinel writes qualitative observations into `macro_observations`
  with trust attribute populated by extraction job (Epic 4).
- Sector tag present where source is sector-attributed; absent where
  not.

---

# Epic 4: SentinelCollector Rework

**Goal.** Reframe Sentinel as disconnect detector and unofficial-channel
signal aggregator. Three observation types extracted. Sentinel exits
math entirely. LLM extraction reliability brought to a measurable
acceptance bar (D13).

## Feature 4.1: Sentinel exits math

### Story 4.1.1: Math-path audit and removal
- Every Sentinel code path computing a derived value beyond extraction
  is identified.
- Each such path is removed or moved to TE.
- Post-audit: Sentinel persists structured observations and a trust
  attribute; nothing more.

## Feature 4.2: Numeric instrument-tagged extraction (verify existing)

### Story 4.2.1: Verify and document existing behavior
- Existing extraction code producing numeric instrument-tagged
  observations is documented.
- Sector tag attachment via SecMaster routes through Feature 1.4.
- Gaps with the contract filed as defects.

## Feature 4.3: Numeric macro-series-tagged extraction

### Story 4.3.1: Routing to `macro_observations`
- When Sentinel extraction matches a known macro series, the resulting
  observation writes into `macro_observations`.
- Signal identity from SecMaster (Feature 1.7) tags the observation.

## Feature 4.4: Qualitative macro-tagged extraction (NEW)

**Description.** Separate prompt and adapter from numeric extraction.
Outputs sentiment polarity, sector affinity, lead-lag estimate, regime
hint. CoVe applies (D11).

### Story 4.4.1: Qualitative extraction prompt and schema
- Prompt and structured-output schema defined.
- Outputs validated against schema; malformed outputs rejected with
  diagnostic logging.
- LoRA adapter applied per spec (informed by Feature 4.6 reliability
  track outputs).

### Story 4.4.2: Qualitative observation persistence
- Qualitative observations write into `macro_observations` (Epic 3).
- Trust attribute populated by extraction job, clamped to defined
  range.
- Sector tag attached when output specifies one of the 11 sectors;
  absent for sector-agnostic outputs.

### Story 4.4.3: CoVe applied to qualitative extraction (per D11)
- Chain of Verification runs over qualitative outputs to constrain
  hallucination.
- Verification step documented; failures reduce trust attribute rather
  than reject the observation.
- CoVe overhead and effect on extraction latency measured.

## Feature 4.5: Validation trigger subscription (per D12)

### Story 4.5.1: Event subscription and narrative search
- Sentinel subscribes to TE's "sector threshold crossed" event
  (Epic 5 Feature 5.5).
- On event (e.g., sector score `> 1.5`), Sentinel queries SearXNG / RSS
  / Cloudflare for narratives on the named sector and time window.
- Resulting observations write through the standard qualitative path
  (Feature 4.4) with trigger source logged in provenance.

## Feature 4.6: LLM extraction reliability track (per D13, D15, D16)

**Description.** Bring the LoRA + qualitative-extraction prompt to a
measurable acceptance bar. The labelled dataset (~5,500 examples, ~550
negative) is the evaluation substrate. Iteration is timeboxed (4 weeks)
plus a $500 Azure cap for any Opus 4.7 relabeling. Acceptance thresholds
inherited from prior LoRA training (D15).

### Story 4.6.1: Import inherited acceptance criteria
- Architect provides the location/values of the previous LoRA training
  cycle's acceptance thresholds (precision, recall, F1 per output type;
  negative-rejection metric).
- Thresholds documented in this Epic and committed.
- *Prerequisite for downstream stories*: source location supplied by
  architect (commit hash, doc link, or pasted values).

### Story 4.6.2: Evaluation harness against labelled dataset
- Labelled dataset (~5,500 examples, ~550 negative) loaded into the
  evaluation framework.
- Harness runs the qualitative extraction prompt + current LoRA against
  the dataset.
- Output: scorecard with per-output-type metrics and negative-handling
  metric, plus delta from inherited acceptance criteria.
- Reproducible run; baseline scorecard captured before iteration begins.

### Story 4.6.3: Iteration cycles
- Each cycle: hypothesis → modification (LoRA training adjustment,
  prompt change, data augmentation, CoVe tuning) → re-evaluate →
  document delta.
- Iteration budget per D16: 4 weeks of on-prem effort with weekly
  checkpoints, plus a $500 Azure spend cap for any Opus 4.7 relabeling.
- Spend tracking visible at weekly checkpoints.
- All cycle artefacts (configs, prompts, training runs, scorecards)
  committed for traceability.

### Story 4.6.4: Success / failure decision
- At budget exhaustion (time or money), scorecard compared to inherited
  acceptance criteria.
- Success: criteria met → LoRA + prompt deployed → Feature 4.4 enables
  full-trust qualitative writes.
- Failure: architect call on extend / ship constrained (high trust-gate
  threshold, limited Sentinel coverage) / defer.
- Decision recorded; if success, the ratified threshold becomes the
  trust-gate default in Epic 2 Feature 2.5 / Epic 3 Story 3.1.1 / Epic
  5 Story 5.5.x.

---

# Epic 5: Matrix Computation, Storage, and Vector Queries

**Goal.** Persist the matrix produced by TE. Make cell, sector, and
signal vectors queryable. Compute per-sector regimes (D9). Emit the
events Sentinel subscribes to.

## Feature 5.1: Cell persistence

### Story 5.1.1: `matrix_cells` schema and write path
- TimescaleDB hypertable with columns for pattern id, sector, time,
  cell value (real, ±3), mapping version, contributing-observation
  references.
- Subscriber consumes TE's cell-emit stream and writes idempotently.
- Idempotency keyed on `(pattern_id, sector, evaluation_time)`.

### Story 5.1.2: Cell time-series query path
- Query: given (pattern, sector), return time series.
- Query honors versioned mapping for historical reads.
- Indexes support dashboard time-window panels.

## Feature 5.2: Sector vector and signal vector queries

### Story 5.2.1: Sector vector query
- Given (sector, time window), return the matrix of (pattern × time)
  cells.
- Used by sector-clustering analysis — diagnostic.

### Story 5.2.2: Signal vector query
- Given (pattern, time window), return the matrix of (sector × time)
  cells.
- Used by signal-redundancy detection — diagnostic.

## Feature 5.3: Cross-correlation tensor query

### Story 5.3.1: Pairwise cross-correlation API
- Given (sector_A, sector_B, lag_range, time_window), return correlation
  series across the lag range.
- Implementation reads cell time series; no separate storage.
- Performance target: single pair query returns in <2s for the
  dashboard's interactive latency budget.

### Story 5.3.2: All-pairs lead-lag scan
- Compute the full sector × sector × lag tensor for a configured time
  window.
- Result cacheable; cache invalidates on new cell writes within the
  window.
- Used by monthly trajectory reports.

## Feature 5.4: Per-sector regime classification (per D9, D14)

### Story 5.4.1: Per-sector regime computation
- For each sector, classify the sector score into a regime state at
  each evaluation cycle.
- Regime taxonomy (state set) is configurable.
- Result persisted as a time series alongside `matrix_cells`.
- 11 regime time series, one per sector.

### Story 5.4.2: Legacy global 6-state regime deleted
- Global 6-state regime code paths deleted, not parallel-run (per D14).
- Downstream consumers updated to read per-sector regimes.

## Feature 5.5: TE event emission for validation triggers (per D12)

### Story 5.5.1: Event contract definition
- Event schema: sector, score (real), threshold-crossed (real),
  direction, evaluation-time, contributing-pattern ids.
- Threshold semantics: crossing a real-valued bound (e.g., score
  passing `> 1.5`) fires the event.

### Story 5.5.2: Event emission in TE evaluation loop
- TE emits the event whenever a sector score crosses a configured
  threshold.
- Per-sector thresholds configurable; default documented.
- Hot-reloadable.

### Story 5.5.3: Sector × phase reference matrix derived view
**Description.** The dashboard primary visualization (per D8) needs a
derived view: for each sector × regime-phase combination, an aggregate
of historical cell values during periods when the sector was in that
phase. This story produces and persists that derived view. Phase
taxonomy size is a developer call (D17), aligning with D9.

- Given the cell time series and per-sector regime time series, compute
  for each (sector, phase) bucket the vector of contributing signal
  scores aggregated over the bucket's historical periods.
- Result persisted as a derived view (refreshed on a defined cadence;
  weekly initial proposal).
- Query path: given (sector, phase), return the vector with metadata on
  contributing signals and the time periods aggregated.

---

# Epic 6: Three Consumption Surfaces

**Goal.** All three surfaces consume `macro_observations` and matrix
data per §4. None is downstream of another. Disagreement is preserved
across all three.

## Feature 6.1: MCP query surface

### Story 6.1.1: `macro_observations` query tool
- Query by signal, sector, time range, observation type.
- Below-threshold qualitative observations returned with trust attribute
  visible (gate is TE-side only).
- Versioned-mapping query path honored.

### Story 6.1.2: Matrix query tools
- Cell time series query.
- Sector vector query.
- Signal vector query.
- Cross-correlation tensor query (single pair and all-pairs scan).

### Story 6.1.3: Regime query tools
- Per-sector regime time series (D9).
- Sector × phase reference matrix query (Story 5.5.3).

## Feature 6.2: Dashboard

**Description.** Visualizes matrix state and trajectory. Implementation
choice (Grafana vs. custom web app) deferred to architect; this Feature
specifies what it must show, not how. Primary visualization is the
sector × phase matrix per D8; ATLAS's matrix is more nuanced than the
Fidelity reference (D17).

### Story 6.2.1: Sector × phase matrix panel (primary view)
- Renders the derived view from Story 5.5.3 as a two-axis matrix.
- Rows: 11 sectors. Columns: regime phases per D9 / D17.
- Each cell contains a vector of contributing signal scores
  (real-valued, not symbolic). Visualization of the vector is a
  developer call (e.g., sparkline, mini-histogram, signed magnitude
  bar group); must preserve the multi-signal information rather than
  flatten to a single aggregate.
- Drilldown affordance: clicking a cell reveals current observations
  contributing to that cell.
- Reference image (`1778343398312_image.png`) is layout inspiration
  only; ATLAS's matrix is more nuanced in cell content and value type.

### Story 6.2.2: Sector score and regime trajectory panels
- Per sector: score time series with regime overlay.
- Aggregate: 11-sector regime panel showing current state and recent
  transitions.

### Story 6.2.3: Direct-substrate panel
- At least one panel reads `macro_observations` directly (not via TE
  cells) per §4.
- Renders qualitative observations with provenance and trust attribute.
- Below-threshold observations visible with trust indicator (visibility
  ≠ scoring).

### Story 6.2.4: Disagreement panel
- Surfaces signal-vs-signal disagreement and
  official-vs-unofficial-source divergence.
- Per §1 corollary: load-bearing matrix value, not a side-panel.

## Feature 6.3: Reports (daily / weekly / monthly per D10)

**Description.** Common content structure across all three cadences:
news summary, word cloud, radar charts (sectors and macro signals). The
time window varies by cadence.

### Story 6.3.1: Common report template
- Sections:
  - **News summary.** LLM-rendered narrative grounded in observation
    references over the window. Reads `macro_observations` directly per
    §4.
  - **Word cloud.** Dominant terms / themes from observations and
    extracted text over the window.
  - **Sector radar chart.** 11-axis radar visualizing sector scores at
    end of window (with optional overlay of start-of-window scores).
  - **Macro signal radar chart.** N-axis radar of dominant macro
    signals over the window (signal selection rule defined in story).
- Output formats configurable (markdown, HTML, PDF).

### Story 6.3.2: Daily report
- Time window: previous trading day.
- Generated on a daily schedule.
- Delivery mechanism per architect preference (email, file drop, MCP
  resource).

### Story 6.3.3: Weekly report
- Time window: previous week.
- Generated on a weekly schedule (Mondays proposed).
- Delivery as Story 6.3.2.

### Story 6.3.4: Monthly report
- Time window: previous calendar month.
- Generated on a monthly schedule.
- Delivery as Story 6.3.2.
- Additionally surfaces the all-pairs lead-lag scan output (Story
  5.3.2) where relevant for trajectory analysis.

---

## Sequencing

**Phase 1 — Spine (sequential).**
1. Epic 1 Features 1.1–1.3 — taxonomy, rollup, versioning.
2. Epic 1 Features 1.4–1.7 — classification and signal identity.

**Phase 2 — Substrate, engine, LLM track (parallel after Phase 1).**
- Epic 2 (TE schema rework).
- Epic 3 (`macro_observations` substrate).
- Epic 4 Feature 4.6 (LLM reliability track) starts in parallel.
  Story 4.6.1 prerequisite: architect supplies the source for inherited
  thresholds (D15).

**Phase 3 — Sentinel rework and matrix (parallel after Phase 2 has
Epic 3 substrate live and Epic 2 emitting cells).**
- Epic 4 Features 4.1–4.5 (rework proper). Feature 4.4's full-trust
  output gated by Feature 4.6 outcome.
- Epic 5 (matrix computation, storage, vectors).

**Phase 4 — Surfaces (after Phase 3 has cells flowing and trust gate
enforced).**
- Epic 6 (consumption surfaces).

**Out-of-band.**
- Backfill, vintage data, predictive validation criteria — parked.

All decision gates and sub-gates are closed. The only remaining
prerequisite for kickoff is the architect supplying the source location
for the inherited LoRA acceptance thresholds (D15 / Story 4.6.1).
