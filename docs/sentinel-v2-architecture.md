# Sentinel v2 Architecture

## Purpose and scope

This document specifies the internal architecture of the Sentinel v2 ingestion-and-extraction pipeline. It is written for engineers implementing Phase 1.3 (schema migration) through Phase 4.5 (audit gate). It defines component boundaries, data contracts between stages, new schema columns, feature-flag precedence, source routing rules, the multi-signal gate math, and the migration path from v1.

This document is **not** a product specification. Business requirements, success metrics, audit cohorts, and rollout sequencing for stakeholders live in `docs/sentinel-product-spec-v2.md` and are not duplicated here. This document is also not an API reference: the `SeriesCollectedEvent` proto and the SecMaster gRPC surface are treated as fixed inputs and are referenced only where they constrain design.

Out of scope: ThresholdEngine internals, SecMaster candidate ranking logic, collector-level scheduling, and the `scripts/shadow_diff_report.py` diff semantics beyond its role as a gate for cutover.

## v2 pipeline diagram

```
                   ┌────────────────┐
  RawContent  ───> │  SourceRouter  │
                   └──────┬─────────┘
                          │
            ┌─────────────┴─────────────┐
            │                           │
            v                           v
   deterministic parser        ┌────────────────────┐
   (TSA, fed-sep,              │ ContentNormalizer  │
    truflation-api,            └──────────┬─────────┘
    challenger-rss,                       │
    application/json)                     v
            │                  ┌────────────────────┐
            │                  │ CandidateGenerator │<── SecMaster
            │                  └──────────┬─────────┘      (cached)
            │                             │
            │                             v
            │                  ┌────────────────────────┐
            │                  │ MergedExtractionService│<── vLLM (v7 prompt
            │                  └──────────┬─────────────┘       + source variant)
            │                             │
            │                             v
            │                  ┌────────────────────────┐
            └─────────────────>│  DeterministicResolver │<── SecMaster
                               └──────────┬─────────────┘  (hybrid_resolve)
                                          │
                                          v
                               ┌────────────────────────┐
                               │    MultiSignalGate     │ ── Q-score
                               └──────────┬─────────────┘
                                          │
                 Approved / Pending / Rejected
                                          │
                                          v
                            sentinel.extracted_observations
                                          │
                           ┌──────────────┴───────────────┐
                           v                              v
                  ┌─────────────────────┐      ┌────────────────────┐
                  │ CorroborationWorker │      │   EventPublisher   │───> ThresholdEngine
                  │   (BackgroundSvc)   │      │ (SeriesCollected)  │     (gRPC, unchanged)
                  └─────────────────────┘      └────────────────────┘
```

The router is the entry point for every `RawContent`. Deterministic sources bypass the LLM path and land directly in the resolver with known entities. LLM sources fan through normalization, candidate retrieval, the merged extract+resolve call, resolution, and the gate. All observations persist to `sentinel.extracted_observations`, regardless of state. Only rows in the `Approved` state are eligible for publication via `EventPublisher`. `CorroborationWorker` mutates `Pending` rows out-of-band.

## Component responsibilities

### SourceRouter

New. Receives `RawContent` and dispatches on `source_kind` + MIME. For structured, high-trust sources (TSA, fed-sep, truflation-api, application/json feeds, challenger-rss tabular rows) it routes to a deterministic parser that produces `ExtractionResult` directly with a pre-known `SubjectEntity` and `InstrumentId`. For unstructured sources (fed-speeches, fed-minutes, fed-press-all, rss, searxng-content) it routes to the LLM path. The router owns the mapping table in §"Source routing table" and emits a `routing_decision` log line per raw_content.

### ContentNormalizer

Existing. Unchanged. Strips boilerplate, normalizes whitespace, extracts main article text. v2 consumes its output unmodified.

### CandidateGenerator

New. For each normalized content about to hit the LLM path, queries SecMaster for a shortlist of candidate instruments (target: 5–15) based on cheap keyword and ticker-ish token retrieval over the content. The shortlist is a list of `{index, symbol, name, asset_class, aliases}` serialized into the prompt. Results are cached keyed by `(raw_content_id, content_hash)` with a TTL matching the raw_content retention window; re-extraction of the same row must produce the same shortlist. The generator is tolerant of empty results — an empty candidate list is a valid input to the LLM and signals "no known instrument matched; either emit SubjectEntity only or decline."

### MergedExtractionService

New. Performs a single vLLM call that combines extraction and candidate selection. The prompt is assembled by `FileBasedPromptProvider`: the per-source variant from `/opt/ai-inference/prompts/sentinel/sources/v7/<source>.txt` is **prepended** to the master v7 prompt at `/opt/ai-inference/prompts/sentinel/extract_and_resolve_v7.txt`. Inputs: normalized content, candidate shortlist, source metadata. Output: a structured response containing one or more observations each with `SubjectEntity`, `SelectedCandidateIndex` (nullable), `ExtractionConfidence`, `ResolutionConfidence`, `Certainty`, and the raw extraction payload (value, period, unit, text_quote). This service does **not** resolve instruments; it only produces the candidate pick.

### DeterministicResolver

New. Consumes the extraction result and resolves to an `InstrumentId` using the cascade in §"Candidate_index resolution flow." Has three outcomes: `llm_candidate_pick` (happy path), `hybrid_subject` (fallback via `SecMaster.hybrid_resolve(SubjectEntity, text_quote)`), or `NoResolution`. The resolver never performs bag-of-words matching against the free-text description; that path is explicitly retired.

### MultiSignalGate

New. Replaces the v1 single-threshold check on `Confidence`. Computes a `QualityScore` as a weighted sum of six signals and assigns one of three states: `Approved`, `Pending`, `Rejected`. Approval additionally requires a non-null `InstrumentId` and a `certainty` of `definite` or `expected`. Formula and tier thresholds are in §"Multi-signal gate formula." Writes `QualityScore` back onto the observation.

The gate computes `QuoteFidelity` in-process by locating `text_quote` inside the normalized content reached via `ExtractionResult.NormalizedTextRef`. The reference must be populated by `MergedExtractionService` (LLM path) or by the deterministic parser (structured path); a missing reference is a contract violation and the gate logs at ERROR and sets `QuoteFidelity = 0.0`.

### CorroborationWorker

New. Hosted `BackgroundService`. Every 5 minutes, scans `sentinel.extracted_observations` for rows where `CrossSourceCount IS NULL AND ExtractedAt > now() - interval '72 hours'`. For each row, counts matches on `(SubjectEntity, period_start, value ± 1%)` drawn from **distinct** `raw_content_id`s. Writes the count to `CrossSourceCount`. Recomputes `QualityScore` using the updated corroboration term. If a row in `Pending` crosses the 0.80 threshold and still satisfies the `InstrumentId` and `Certainty` predicates, the worker transitions it to `Approved` and enqueues it for `EventPublisher`. The worker never downgrades `Approved` rows.

### EventPublisher

Existing. Unchanged wire contract. Consumes `Approved` observations and emits `SeriesCollectedEvent` to ThresholdEngine over gRPC. In v2 it receives an adapter layer (see §"Migration path") that projects new fields onto the existing proto without schema change.

## Schema additions

### `ExtractionResult` (in-process type)

| Field | Type | Notes |
|---|---|---|
| `SubjectEntity` | string | LLM-emitted canonical entity string (e.g. "Dynatrace Inc"). |
| `CandidateSymbols` | list<CandidateRef> | Snapshot of shortlist passed to LLM (ordered; indexing is zero-based). |
| `SelectedCandidateIndex` | int? | LLM-chosen zero-based index into `CandidateSymbols`; null if none matched. Out-of-range values are treated as null. |
| `ExtractionConfidence` | double | LLM self-reported confidence in value extraction, [0.0, 1.0]. |
| `ResolutionConfidence` | double? | LLM self-reported confidence in candidate selection, [0.0, 1.0]. Null iff `SelectedCandidateIndex` is null. |
| `NormalizedTextRef` | string | Reference (raw_content_id or in-memory handle) to the normalized document. Required so that `MultiSignalGate` can compute `QuoteFidelity` without a second DB fetch. |
| `Confidence` | double | `[Obsolete]` for one phase; retained for reader compat. Back-projected from `QualityScore` to preserve score semantics across the transition window. |

### `CandidateRef` (embedded in `ExtractionResult`)

| Field | Type | Notes |
|---|---|---|
| `Index` | int | Zero-based position in the shortlist; is the value the LLM returns as `selected_candidate_index`. |
| `InstrumentId` | int | SecMaster primary key. |
| `Symbol` | string | Canonical SecMaster symbol (e.g. "DT", "sentinel:challenger:layoffs:monthly"). |
| `Name` | string | Human-readable name used by the LLM for disambiguation. |
| `AssetClass` | string | SecMaster asset-class tag. |
| `Aliases` | list<string> | Zero or more alternate names (empty list is valid, never null). |

The shortlist is deterministic given `(raw_content_id, content_hash, SecMaster_state)`; two extractions of the same row produce identical `CandidateRef` lists unless SecMaster state changes between runs.

### `ExtractedObservation` (persisted, `sentinel.extracted_observations`)

| Column | Type | Nullable | Rationale |
|---|---|---|---|
| `SubjectEntity` | text | YES | v1 rows predate this field. |
| `CandidateSymbolsJson` | jsonb | YES | v1 rows have no shortlist. Stores the *exact* shortlist snapshot passed into the v7 prompt (not the raw SecMaster response), so the audit path can replay prompts verbatim. |
| `SelectedCandidateIndex` | int | YES | Null for v1 and for deterministic-path rows. Zero-based. |
| `ExtractionConfidence` | double | YES | Null on v1 rows and deterministic rows. |
| `ResolutionConfidence` | double | YES | Null on v1 rows; null when resolver used `hybrid_subject`. |
| `CrossSourceCount` | int | YES | Null until `CorroborationWorker` processes the row; after processing, always an integer (possibly zero). |
| `QualityScore` | double | YES | Null for v1 rows; non-null for every v2 row after gate. Recomputed in-place by `CorroborationWorker` when `CrossSourceCount` changes. |
| `ResolutionMethod` | text | YES | Already present in v1; v2 extends enum with `llm_candidate_pick` and `hybrid_subject`. |

All new columns are nullable. v1 and v2 rows coexist in the same table during the shadow and cutover windows; every reader must tolerate `NULL` on new columns. The legacy `Confidence` column is preserved unchanged.

## Feature flags

Flags are read from configuration (env + appsettings). Precedence is evaluated per-raw_content at routing time.

| Key | Type | Effect |
|---|---|---|
| `Extraction__UseV2Pipeline` | bool | Global kill. `false` forces v1 path for every source. |
| `Extraction__V2EnabledSources__<N>` | string list | Per-source allow-list. Only sources in this list use v2 when the global flag is on. |
| `Extraction__ShadowMode` | bool | When true, v2 runs alongside v1 for enabled sources; v2 writes to `extracted_observations_shadow`, v1 continues writing to prod table and remains the source of events. |
| `Extraction__SourceTrustScores__<source>` | double | Override for `SourceTrust` coefficient in the gate. |
| `Extraction__ResolverCandidatePickThreshold` | double | Threshold for `llm_candidate_pick` resolution (default 0.7). Below this, the resolver falls through to `hybrid_subject`. |

Precedence: `UseV2Pipeline=false` short-circuits to v1. Otherwise, if the source is not in `V2EnabledSources`, v1 runs. Otherwise, if `ShadowMode=true`, v2 executes but writes to the shadow table. Otherwise v2 runs in production. The `AutoApproveEnabled` kill switch sits downstream and remains `false` until Phase 4.5.

## Source routing table

| Source | Route | Rationale |
|---|---|---|
| `TSA` | deterministic | Structured daily counts; known subject. |
| `fed-sep` | deterministic | Tabular Summary of Economic Projections. |
| `fed-minutes` | LLM | Free-form narrative; requires extraction. |
| `fed-press-all` | LLM | Prepared remarks + Q&A; narrative. |
| `fed-speeches` | LLM | Narrative; source-variant prompt prepended. |
| `truflation-api` | deterministic | Structured JSON index values. |
| `challenger-rss` | deterministic | Fixed-schema monthly layoff tallies. |
| `rss` | LLM | Heterogeneous headlines + body. |
| `searxng-content` | LLM | Arbitrary third-party pages; lowest trust. |
| `application/json` (generic) | deterministic | Structured payloads with declared schema. |

Deterministic routes skip `CandidateGenerator` and `MergedExtractionService`; they still pass through `MultiSignalGate` for uniform state assignment, but with `ExtractionConfidence=1.0` and `ResolutionConfidence=1.0` by construction.

## Multi-signal gate formula

```
Q = 0.30 * ExtractionConfidence
  + 0.25 * ResolutionConfidence
  + 0.15 * SourceTrust
  + 0.15 * QuoteFidelity
  + 0.10 * CertaintyWeight
  + 0.05 * min(1.0, CrossSourceCount / 3)

Approved:  Q >= 0.80  AND  InstrumentId != NULL  AND  certainty IN {definite, expected}
Pending:   0.50 <= Q < 0.80
Rejected:  Q < 0.50
```

Certainty comparison is case-insensitive. Canonical form is lowercase, matching the v7 prompt contract (plan §1.1). An unknown certainty value is treated as `CertaintyWeight = 0.0` and logged at WARN; it does not block evaluation of the other signals.

`SourceTrust` defaults:

| Source | Default |
|---|---|
| `tsa` | 0.99 |
| `fed-*` | 0.95 |
| `truflation-api` | 0.95 |
| `challenger-rss` | 0.90 |
| `rss` | 0.75 |
| `searxng-content` | 0.60 |

`CertaintyWeight` mapping (lowercase, matches v7 prompt contract `definite | expected | speculative | conditional`):

| Certainty | Weight |
|---|---|
| `definite` | 1.0 |
| `expected` | 0.8 |
| `speculative` | 0.4 |
| `conditional` | 0.5 |

`QuoteFidelity` is a 0.0–1.0 score indicating whether `text_quote` is a verbatim substring of the normalized content (1.0 exact, fractional for normalized-whitespace match, 0.0 if not found). It is computed by the gate, not the LLM.

`CrossSourceCount` is 0 (or null, treated as 0) at initial gate evaluation; the corroboration term is near-zero on first pass and grows as corroboration arrives.

## Candidate_index resolution flow

```
IF selected_candidate_index != null AND resolution_confidence >= ResolverCandidatePickThreshold:
    lookup candidate[selected_candidate_index] -> InstrumentId, Symbol
    resolution_method = "llm_candidate_pick"

ELSE IF SubjectEntity != null:
    (InstrumentId, Symbol) = SecMaster.hybrid_resolve(SubjectEntity, text_quote)
    IF match found:
        resolution_method = "hybrid_subject"
    ELSE:
        state = NoResolution
        resolution_method = "none"
        // observation persists as Pending with InstrumentId = NULL

ELSE:
    state = NoResolution
    resolution_method = "none"
```

`ResolverCandidatePickThreshold` is configurable via `Extraction__ResolverCandidatePickThreshold` (default `0.7`). An out-of-range `selected_candidate_index` is treated as null. A candidate pick with `ResolutionConfidence` below the threshold falls through to the `hybrid_subject` branch; the LLM's own hedging is respected. `NoResolution` rows always land in `Pending` regardless of `QualityScore`, because the `InstrumentId != NULL` predicate in the approval rule blocks them; they remain candidates for later corroboration-driven resolution only if a future worker pass can supply an `InstrumentId`.

## Cross-source corroboration

`CorroborationWorker` is a `BackgroundService` registered in the host. Cadence: every 5 minutes. Scan predicate:

```
CrossSourceCount IS NULL
  AND ExtractedAt > now() - interval '72 hours'
```

For each row, it counts distinct `raw_content_id` values whose observations match on:

- identical `SubjectEntity` (case-insensitive exact),
- identical `period_start`,
- `value` within ±1% of the candidate row's value.

The count is written to `CrossSourceCount`. The gate formula is recomputed in-place. If the recomputed `QualityScore >= 0.80`, the row satisfies `certainty IN {definite, expected}`, and `InstrumentId IS NOT NULL`, the worker transitions it from `Pending` to `Approved` and emits the corresponding `SeriesCollectedEvent` via `EventPublisher`. The worker is idempotent: re-running on an already-counted row is a no-op by the scan predicate.

Shadow isolation: the worker only scans `sentinel.extracted_observations`. It never reads the shadow table and never promotes shadow rows; shadow rows are not migrated to prod on cutover (a fresh v2 re-extraction occurs against the prod table instead). This prevents a `ShadowMode=false` flip mid-window from publishing rows that were never visible to v1 consumers.

## Migration path

The `SeriesCollectedEvent` proto is frozen (decision D8). v2's additional fields do not propagate to ThresholdEngine. `EventPublisher` contains an adapter that reads v2 columns (`InstrumentId`, `SubjectEntity`, `QualityScore`) and projects them into the existing proto shape — `InstrumentId` continues to occupy the same field, v2-only metadata is dropped at the boundary.

Shadow period:

1. `UseV2Pipeline=true`, `ShadowMode=true`, `V2EnabledSources` seeded with one low-risk source (e.g., `fed-speeches`).
2. v1 runs production, writes to `sentinel.extracted_observations`, publishes events.
3. v2 runs in parallel for enabled sources, writes to `sentinel.extracted_observations_shadow`, publishes nothing.
4. `scripts/shadow_diff_report.py` diffs v1 vs shadow rows on `(raw_content_id, period_start)` and reports Symbol disagreement rate, state disagreement rate, and lift on the known-broken "Dynatrace/RBC" cohort.

`sentinel.extracted_observations_shadow` must be created in the *same* EF migration as the v2 column additions, and must be declared with `CREATE TABLE LIKE sentinel.extracted_observations INCLUDING ALL` (or the EF equivalent) so schema drift is structurally impossible. Future column additions must modify both tables in the same migration transaction.

Cutover (per source):

1. Parity review on shadow diff + demonstrated lift on broken cohort.
2. Toggle `ShadowMode=false` for that source; v2 writes to the production table and publishes.
3. v1 path remains compiled and reachable for sources still absent from `V2EnabledSources`.

v1 retirement is a separate, user-approved step in Phase 5.1 — not automatic on cutover.

## Backward compatibility strategy

v1 and v2 share `sentinel.extracted_observations`. v1 rows carry `NULL` for every new column; v2 rows carry non-null values except where documented (e.g., `SelectedCandidateIndex` on deterministic rows). Readers — including the audit tooling and any query that feeds the dashboards — must treat every new column as nullable and must not assume `QualityScore` is populated.

`Confidence` is preserved and marked `[Obsolete]` on the in-process type for one phase. v2 writes `Confidence = QualityScore` so that the preserved column still carries a single holistic score comparable to v1 semantics; projecting `ExtractionConfidence` alone would drop the resolution-quality and source-trust components and produce a distributional shift on dashboards. Dashboard owners are notified of the change; the new column `QualityScore` is the canonical forward signal. After cutover, the obsolete attribute escalates to removal in a subsequent phase, gated separately.

The `SecMaster` gRPC client surface is unchanged; v2 adds new call sites (shortlist retrieval, `hybrid_resolve`) but introduces no new methods.

## NEW vs RENAMED vs RETIRED

| Category | Items |
|---|---|
| **NEW components** | `SourceRouter`, `CandidateGenerator`, `MergedExtractionService`, `DeterministicResolver`, `MultiSignalGate`, `CorroborationWorker` |
| **NEW columns** | `SubjectEntity`, `CandidateSymbolsJson`, `SelectedCandidateIndex`, `ExtractionConfidence`, `ResolutionConfidence`, `CrossSourceCount`, `QualityScore` |
| **NEW config** | `Extraction__UseV2Pipeline`, `Extraction__V2EnabledSources`, `Extraction__ShadowMode`, `Extraction__SourceTrustScores`, `Extraction__ResolverCandidatePickThreshold` |
| **RENAMED** | None. Existing fields and types preserved for contract stability. |
| **RETIRED (Phase 5.1, user-gated)** | Post-hoc bag-of-words Symbol resolver in `ExtractionProcessor`; single-threshold gate on `Confidence`; `Confidence` as primary ranking signal |
| **PRESERVED (unchanged)** | `ContentNormalizer`, `EventPublisher`, `SeriesCollectedEvent` proto, `SecMaster` gRPC client surface, `ExtractionProcessor` shell (branches on `UseV2Pipeline`), `RawContent` ingestion path |

## Risks and open items

Items below are advisory for Phase 1.3 (schema migration) and Phase 2 (component implementation).

### Phase 1.3 (schema migration)

1. **Partial index for `CorroborationWorker` scan.** Ship `CREATE INDEX CONCURRENTLY ix_eo_corrob_pending ON sentinel.extracted_observations (extracted_at) WHERE cross_source_count IS NULL` in the same migration that adds the column. Without it, worker cost grows linearly with table size. Expected cardinality: at most a few thousand rows (72h window of unprocessed v2 rows).

2. **Corroboration-match composite index.** Matching on `(SubjectEntity, period_start, value)` across distinct raw_contents requires a supporting index. Recommend `(subject_entity, period_start)` with the `value` ±1% match done in-process after the lookup; avoids a range-over-numeric index. Confirm row-volume estimates before settling on the composite.

3. **Shadow table creation.** `sentinel.extracted_observations_shadow` is created in the same migration as the new columns, with `LIKE … INCLUDING ALL`. No separate ticket.

4. **No defaults / no future NOT NULL.** All new columns are nullable with no default. Readers already tolerate null; no backfill of `QualityScore` for v1 rows (explicitly recommended null, not synthetic). Tightening to NOT NULL is explicitly out of scope for this redesign.

5. **Back-projection of `Confidence`.** Migration docs must note that v2 writes `Confidence = QualityScore` going forward; dashboard owners are notified before cutover.

### Phase 2 (component implementation)

6. **Candidate cache invalidation.** `CandidateGenerator` caches on `(raw_content_id, content_hash)`. SecMaster listing changes mid-TTL produce stale shortlists. Acceptable for Phase 1–4 given re-extraction latency; flag for revisit in Phase 5.

7. **`SubjectEntity` normalization.** Corroboration matches are case-insensitive exact. Alias variation ("Dynatrace", "Dynatrace Inc", "DT") will under-count until a normalization pass (lowercased, punctuation-stripped, alias-mapped via SecMaster) is implemented as a separate Phase 2 ticket.

8. **Re-extraction row management.** A retry or reprocessing of the same `raw_content_id` inserts new observation rows. Corroboration on `distinct raw_content_id` still treats them as one source, but duplicate rows accumulate. A dedup or superseding policy is required before Phase 4.5 audit; tracked as a Phase 2 ticket.

9. **`application/json` generic route.** The routing table assigns `application/json` feeds to a deterministic parser. The parser must enforce a declared schema per feed; undeclared payloads fall back to the LLM path. Detailed parser-per-feed mapping lives in a Phase 2 ticket.

10. **Deterministic-path `QuoteFidelity`.** Deterministic parsers do not emit a literal `text_quote` (the whole row is the quote). The gate should short-circuit to `QuoteFidelity = 1.0` for deterministic-path rows; this is a small Phase 2 implementation detail, not a design question.

The design above is considered complete for Phase 1.3 ticketing.