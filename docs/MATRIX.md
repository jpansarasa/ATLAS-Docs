# The ATLAS Signal Matrix

The matrix is ATLAS's central derived dataset: a continuously re-projected time series of
**signal x sector contributions** stored in the `matrix_cells` hypertable (`atlas_data`,
TimescaleDB). It answers "what is each macro/market signal currently saying about each of the
11 ATLAS sectors, from each data source, and how strongly?" — with full multiplicative
provenance on every row.

This document describes the matrix as implemented. System-level context is in
[ARCHITECTURE.md](./ARCHITECTURE.md); the Sentinel news pipeline that feeds it is summarized
there and detailed in `SentinelCollector/README.md`.

## 1. What a cell is

One row of `matrix_cells` is keyed **(pattern_id, sector_code, evaluated_at)** (unique index
`ux_matrix_cells_idem`). The single wired writer — ThresholdEngine's `ObservationCellProjector`
— keys rows with a synthetic `pattern_id = "obs:{signal_identity_id}:{source_collector}"`, so
the effective cell identity is **(signal x sector x source collector) at evaluation time**.

Every projection emits exactly **11 rows — one per ATLAS sector** — writing explicit zeros
rather than sparse rows. A missing sector weight raises and skips the whole group; partial
sector vectors are never written.

The 11 sectors: `ENERGY, MATERIALS, INDUSTRIALS, CONS_DISC, CONS_STAPLES, HEALTHCARE,
FINANCIALS, INFOTECH, COMM_SVC, UTILITIES, REAL_ESTATE`.

Each row denormalizes its multiplicative provenance — `signal` (magnitude), `sector_weight`,
`freshness`, `temporal_multiplier`, `confidence`, `pattern_weight` (= source trust) — so an
audit can re-derive `cell_value` without re-reading the underlying observations.

### Invariants (layered, innermost out)

| Invariant | Enforced at |
|---|---|
| tilt ∈ [-1, 1], confidence ∈ [0, 1] | classifier clamp (`SentinelCollector/src/Semantic/NewsSignalClassifier.cs`) |
| cell_value ∈ [-3, +3] | projector core clamp (`ThresholdEngine/src/Services/ObservationCellProjectorCore.cs`) |
| re-clamp on persist (logged + counted) | `MatrixCellPersistenceWorker` defence-in-depth |
| `ck_matrix_cells_value_range` (±3), `ck_matrix_cells_sector_code` (closed 11-value list) | DB CHECK constraints (migration `20260611022650_MatrixCellsInvariantChecks`) |

## 2. Data flow end to end

```
news article --> ExtractionProcessor --> NewsSignalClassifier (GPU vLLM, structured JSON)
                                              | per-signal {id, tilt, confidence, sector?}
                                              v
                 MacroObservationRouter.TryPlanNewsSignalWrite
                   value_numeric = tilt * confidence
                   source_id = "{rawContentId}:sig:{signalId}", source_collector = "sentinel"
                                              v
FRED collector --(macro-classified series)--> macro_observations (MacroSubstrate upsert, heal-on-rewrite)
OFR  collector --(is_macro flag; currently 0 flagged)--'
                                              v  (DB poll, NOT gRPC)
                 ThresholdEngine ObservationCellProjector (5-min cycle)
                   group by (signal_identity_id, source_collector) -> 11-sector projection
                                              v
                 matrix_cells (upsert ON CONFLICT DO UPDATE, IS DISTINCT FROM guarded)
```

The matrix feed is **DB-polled, not event-driven**: collectors' live gRPC streams to
ThresholdEngine produce threshold evaluations and `ThresholdEvents` only — they never write
matrix cells (see §7).

### 2a. News entry point — the GPU signal classifier

`NewsSignalClassifier` makes a single structured vLLM call per article inside
`ExtractionProcessor`. The prompt (`news_signal_classify.md`) inlines the signal-identity
catalog and the 11 sector codes; the response schema enum is built from the catalog ids, so the
model cannot emit a signal that isn't in the catalog without being dropped.

Validation drops hallucinated ids, non-finite values, and anything below the 0.5 confidence
floor — every drop is metered by reason. The classifier is fail-soft: timeout, parse failure,
empty catalog, or unenforced schema all return `Empty` and extraction proceeds.

Per-article order: normalize -> `EntityResolutionPrepass` (spaCy NER -> SecMaster
`/api/resolve-entities`, fail-soft) -> classify -> write. The prepass feeds **sector grounding
only**; it never gates matrix entry — sectorless articles still classify and write.

**The catalog is 77 signals**: `atlas_secmaster.signal_identities` holds 84 rows; the
classifier loads the six-category allowlist `macro, rate, commodity, fx, credit, liquidity`
(33+14+4+4+13+9 = 77), deliberately excluding the 7 `instrument` (company-level) identities.
Credit (hy-credit-spread, ccc-credit-spread, ofr-fsi-\*) and liquidity (mmf-\*, fed-rrp-volume,
repo-\*) surfaces are part of the allowlist.

### 2b. The router (Sentinel write)

`MacroObservationRouter.TryPlanNewsSignalWrite` plans one `macro_observations` row per
classified signal: `value_numeric = tilt * confidence` (signed, |v| <= 1), idempotency key
`(source_collector="sentinel", source_id="{rawContentId}:sig:{signalId}", observation_time)`.

Sector reconciliation: a SecMaster entity-grounded sector wins over the classifier's guess. A
sectorless row still writes — `atlas_sector_code` is metadata, never an entry gate.

News writes have **no fallback sink**: a failed write means the row is lost, by design. (The
numeric non-news Sentinel path falls back to `extracted_observations` instead.)

> The `:sig:` infix is a **string contract spanning four artefacts** — router, projector,
> digest momentum query, digest per-article reader — with no schema enforcement. Change all or
> none.

### 2c. The substrate

`MacroObservationRepository` (MacroSubstrate library — in-process DI only, no HTTP/gRPC
transport) performs a parameterized upsert `ON CONFLICT (source_collector, source_id,
observation_time) DO UPDATE` guarded `IS DISTINCT FROM`: heal-on-rewrite, last-write-wins,
byte-identical re-collects write nothing. Value XOR (numeric ⊻ qualitative) is enforced by a DB
CHECK (`ck_macro_obs_value_xor`) plus client-side `EnsureValid()`.

### 2d. Hard-data dual-writers

- **FRED** (`FredCollector/src/Services/DataCollectionService.cs`): a per-series gate consults
  SecMaster REST `/api/signal-identities/by-category/macro` (cached). Macro-classified series
  dual-write `source_collector="fred"`, `source_id={SeriesId}`, after a value transform that
  maps index levels -> YoY for the inflation family (a null transform result means skip, never a
  fabricated value). 27 distinct signal ids have flowed from FRED (24 within
  the projector's 120-day lookback; DB-verified 2026-06-11).
- **OFR** (`OfrCollector/src/Services/MacroObservationDualWriter.cs`): shared HFM/STFM
  dual-write, `source_collector="ofr"`, gated on a per-series `is_macro` flag. **Dormant in
  production**: 0 of 336 HFM and 0 of 109 STFM series are flagged, so `macro_observations`
  contains no OFR rows. The path is wired; the flags are off.

No other collector writes the substrate (§7).

### 2e. The projector — the only wired matrix writer

`ThresholdEngine/src/Workers/ObservationCellProjector.cs` is a `BackgroundService` on a
**5-minute cycle** with a **120-day lookback**. It reads `macro_observations` Kind=Numeric
**descending** (newest-first; an ascending read would freeze the matrix on stale rows), capped
at 1000 rows per cycle (hitting the cap WARNs and meters).

Mode comes from `Matrix:ObservationProjectorMode`; **live = `authoritative`**. `Shadow` runs
the identical write path — it is *not* a dry-run; only `Off` suppresses writes. An unrecognized
value falls back with a startup WARN.

Per cycle:

1. Drop null-`signal_identity_id` rows (counted).
2. Build one signal->pattern map from the full pattern catalog, first-enabled-wins on
   duplicates.
3. Per (signal, collector) group, skip `unknown_signal` / `pattern_disabled` (distinct
   counters).
4. **News fork is row-level**: a group is news iff it contains `:sig:` rows —
   `source_collector='sentinel'` is *not* the discriminator. Guards:
   - G2: legacy rows in a mixed group are dropped from the magnitude;
   - G1: rows with `|v| > 1.5` are excluded as raw-value leaks;
   - a news group with all rows excluded skips entirely — it never falls back to its raw
     legacy rows;
   - future-dated rows get decay weight clamped to 1.0 (counted, warned once per cycle).
5. Project 11 sector cells per group; upsert with `IS DISTINCT FROM` guard.

## 3. Cell formula, decay, counterweight

Per sector:

```
cell = magnitude * sourceTrust * freshness * temporal * confidence * sectorWeight   (clamped ±3)
```

- **News magnitude (decay-weighted sum + saturation)**:
  `S = sum_i v_i*exp(-ln2*age_i/H)`, then `magnitude = K*tanh(S/K)`. Sustained same-direction
  coverage accumulates; tanh saturates a burst gracefully before the ±3 clamp. News uses
  `freshness = 1.0` and **no floor**: an idle signal's cell self-heals toward 0 as ages advance
  each cycle and the DO-UPDATE re-projection overwrites in place. Live tunables
  (`Matrix:NewsDecay`): `HalfLifeHours=24.0`, `SaturationScale=2.0` (bind-time validated,
  `NewsDecayOptions`).
- **Hard-data magnitude**: Welford running mean (sum(o_i)/n) x step-decay freshness derived from
  publication frequency, with a 10% freshness floor.
- **Confidence is STATIC**: `PatternConfiguration.Confidence` (default 0.75). Its XML-doc
  claim of "informational only" is **false** for the wired path — it multiplies into every
  cell.
- **Source trust (the counterweight)** (`SourceTrustConfig`, tunable via `Matrix:SourceTrust`):

  | source | trust |
  |---|---|
  | fred | 1.0 |
  | nasdaq / alphavantage / finnhub / ofr | 0.9 |
  | sentinel, sentinel-rss-named | 0.7 |
  | sentinel-rss-unnamed, searxng | 0.4 |
  | unknown | 0.5 |

  Because the cell key folds in `source_collector`, a FRED and a Sentinel contribution to the
  **same signal coexist as distinct rows in the same cell**. Official data outvotes scraped
  headlines by trust weight, but multi-source news consensus can still tip against FRED —
  disagreement stays magnitude-comparable and is never overwritten.

## 4. Consumption surfaces

- **HTTP** (ThresholdEngine :8080): `/api/matrix-cells` (cell time series with provenance
  factors), `/api/matrix-cells/sector-vector`; audit reads `/api/matrix-cells/audit/{id}`,
  `/by-article`, `/by-created`.
- **MCP** (threshold-engine server): `get_matrix_cells`, `get_matrix_sector_vector`,
  `get_sector_regimes`, plus the `macro_observations` query tool.
- **Dashboards** (`deployment/artifacts/monitoring/dashboards/`): Signal Matrix folder —
  `atlas-matrix-primary`, `atlas-disagreement` (cross-source disagreement),
  `atlas-direct-substrate`, `atlas-matrix-trajectory` — plus Sentinel Pipeline
  `sentinel-matrix-pipeline` and `thresholdengine` ops panels.
- **Digest sector heat** — the Sentinel digest reads `matrix_cells` directly:
  `MatrixSectorQueryService` (raw SELECT-only Npgsql) takes the latest cell per
  (pattern, sector) over a lookback window (`obs:` rows only) plus the latest
  `sector_regimes` per sector; `SectorMatrixBuilder` rolls these into the 11-sector
  heat table and per-sector detail blocks. Regimes attach only when fresh and backed
  by contributing patterns. (The theme taxonomy is retired — sector is the only
  digest axis.)
- **Digest momentum** — a consumer of the *feed*, not of `matrix_cells`:
  `NewsMomentumQueryService` reads `macro_observations` directly
  (`source_collector='sentinel' AND signal_identity_id IS NOT NULL AND value_numeric BETWEEN
  -1.5 AND 1.5`), splitting its window at the midpoint for an early->late per-signal tilt trend.
  Note this filter admits legacy non-`:sig:` sentinel rows that the projector drops — the two
  consumers see different effective inputs. The digest JSON key `bias_matrix` survives for
  schema compatibility but carries the momentum payload.
- **Downstream aggregate**: the `sector_phase_cells` materialized view aggregates matrix cells
  per (sector, phase), refreshed by `SectorPhaseViewRefreshWorker` on a 7-day cadence (a DB
  outage leaves the view silently stale).

## 5. Operational reality (live DB, 2026-06-11)

| measure | value |
|---|---|
| matrix_cells total | 24,324 (24,101 projector `obs:` rows; 223 legacy NULL-collector) |
| cells by collector | sentinel 23,936; fred 165 |
| distinct signals in cells | 30 |
| macro_observations total | 4,191 (sentinel 4,047; fred 144; ofr 0) |
| `:sig:` news rows | 3,968 |
| last 24 h | 717 obs ingested (715 `:sig:`, 6 fred) -> 4,785 cell writes |
| max \|cell_value\| | 3.0000 (clamp + CHECK holding) |

**Patterns**: 72 JSON files under `ThresholdEngine/config/patterns/`, 44 carrying
`metadata.signalIdentity` — only those 44 signals can project; the rest of the 77-signal
catalog skips as `unknown_signal` (counted per cycle). Pattern auto-disable happens at load
time via SecMaster `ResolveBatch`: any referenced mnemonic not Found/PrimarySource disables the
pattern with an ERROR (`PatternValidation:Strict` would fail boot instead).

## 6. Alerts guarding the pipeline

`thresholdengine.yml`: `ThresholdEngineCellPersistFailing`, `ThresholdEngineProjectorErroring`,
`ThresholdEngineProjectorHealsFlatlined`, `ThresholdEngineNewsContractViolations`, plus
pattern-data health alerts. `sentinel.yml`:
`NewsSignalClassifierFailingHard`, `NewsSignalSubFloorDropHigh` (the silent re-zero-vector
guard), and the extraction dead-man chain upstream of the feed.

## 7. What does NOT feed the matrix (by design)

- **The live gRPC path**: collectors -> ThresholdEngine `MultiCollectorEventConsumerWorker`
  (:5001) updates the ObservationCache and persists ThresholdEvents on crossings — it never
  writes `matrix_cells`.
- **`ObservationEventSubscriber` is unwired** dead code (absent from hosted-service
  registrations). `MatrixCellPersistenceWorker` is registered and subscribes to the projection
  event bus but receives nothing at runtime; the `CellProjector.cs` formula is dead code for
  cell writes.
- **No SemanticVerifier exists** — zero references in source.
- **No Sentinel gRPC matrix bridge** — Sentinel reaches the matrix only via
  `macro_observations` rows.
- **No bias-vs-matrix digest view** — the digest compares early-vs-late news tilt within its
  own window; it does not diff news against FRED matrix cells. Only the `bias_matrix` JSON key
  name remains.
- **Qualitative rows never project**: `signal_identity_id` is NULL by construction and the
  projector skips null-signal rows.
- **AlphaVantage, Finnhub, Nasdaq, CalendarService**: no `IMacroObservationWriter` reference in
  any of them — their data never enters `macro_observations` or the matrix.
- **`extracted_observations`**: legacy numeric-fallback sink only; read by neither the
  projector nor the digest.
- **OFR in practice**: writer wired, but with 0/445 series macro-flagged it currently
  contributes nothing (§2d).
