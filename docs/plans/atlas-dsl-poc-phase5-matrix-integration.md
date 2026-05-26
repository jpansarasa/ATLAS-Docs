# DSL PoC — Phase 5 — Integration with ATLAS matrix (plan §8)

**Status:** scoping. No code, no deploy. Single source of truth for Phase 5
work until split into stories and merged.
**Parent plan:** `docs/plans/atlas-dsl-poc-plan.md` §8 (lines 358–406).
**Predecessor:** Phase 4 — `docs/plans/atlas-dsl-poc-phase4-gpu-semantic-verifier.md`.
4.7 PASS milestone on n=563 holdout (Foundry agreement 80.64%) landed
in PR #476; the production runtime path now emits `EnrichedDocument`
per article inside `SentinelCollector.Workers.ExtractionProcessor`
(see L1089–1207).
**Topology rule (immovable, PR #386):** CPU = extraction (qwen3:30b-a3b
on `llama-server` :11437); GPU = verification (Qwen2.5-32B-AWQ on
`vllm-server` :8000). Phase 5 introduces **no new model inference** —
it is data routing + storage from validated DSL → matrix cells.
**Architectural rule (memory `project_sentinel_llm_strength_layering`):**
small CPU does pattern-recognition + verbatim copy + classification;
large GPU does structured emission and judgment. Phase 5 sits BELOW
both layers — it consumes their joint product.

---

## 1. Executive summary

Phase 5 wires the validated Phase 4 `EnrichedDocument` envelope —
emitted today inside `SentinelCollector` (`SemanticVerifierService`,
PR #476 production-locked) — into ThresholdEngine's `matrix_cells`
hypertable as a second producer alongside the existing FRED-pattern
projector. The pipeline is pure plumbing + a deterministic
signal-magnitude calculator: no new model inference, no new DB, no new
LLM. Estimated ~1 week (parent plan §11), 8 phased PRs each ≤500 LOC,
direct cutover at Phase 5.5 per `[[feedback-drop-feature-flags]]`.

---

## 2. Architectural reframe (per layering memory)

| Layer | Status | Notes |
|---|---|---|
| CPU extraction (v15 DSL on `llama-server`) | DONE | Phase 3; production-locked T=0 A/B (PRs #431/#432/#433). |
| Deterministic Python verifier (`verifier_v2_3_1.py`) | DONE | Phase 3 / §7.1; word-grounded. |
| GPU semantic verifier (`SemanticVerifierService`) | DONE | Phase 4.5 cutover; Phase 4.7 n=563 PASS (PR #476). |
| **Matrix integration** | **THIS PHASE** | Data routing + storage. Zero new inference. |

Phase 5 is the consumer wiring the parent plan §8 calls for. The
`EnrichedDocument` envelope is already being computed per article
(see `SentinelCollector/src/Workers/ExtractionProcessor.cs:1089–1207`);
Phase 5 takes that envelope, runs a deterministic signal-magnitude
calculator over it, and writes the resulting `(sector_code,
industry_code, time_bucket, signal_magnitude, polarity, confidence,
source_provenance)` rows into ThresholdEngine's matrix store.

The existing FRED-driven projector
(`ThresholdEngine.Workers.ObservationEventSubscriber` →
`CellProjector.ProjectBatch` → `CellProjectedEvent` →
`MatrixCellPersistenceWorker` →
`matrix_cells`) stays untouched. Phase 5 adds a **second producer**
into the same hypertable (parallel path, idempotent on a different
key tuple — see §7.1).

---

## 3. Input / output contract

### 3.1 Input — `EnrichedDocument` (Phase 4)

Source: `SentinelCollector/src/Semantic/EnrichedDocument.cs`. Per-block
enrichments keyed by block id:

- `EntityResolutions[ent_local_id]` → `EntityResolution { InstrumentId:
  Guid?, Symbol, SectorCode (NAICS 2), IndustryCode (NAICS 3-/4-/6),
  Confidence, GroundingSource, CascadeStagesAttempted, ReviewNotes }`
  (`EntityResolution.cs:59`).
- `ClaimVerifications[claim_block_id]` → `ClaimVerification { Support:
  Full|Partial|None, Confidence ∈ [0,1], Arm (VllmQwen2_5_32b
  production-locked), Rationale, LatencyMs, CostUsd=0 in prod }`
  (`ClaimVerification.cs:73`).
- `EventSectorTaggings[evt_block_id]` → `EventSectorTagging { Tags:
  list<SectorTag {Rank, SectorCode, IndustryCode?, Confidence}>, Arm,
  LatencyMs, CostUsd }` (`EventSectorTagging.cs:76`).
- `SourceDocument: SourceDocumentRef { Source, Timestamp (ISO-8601),
  DslVersion }` — audit anchor.
- `Metadata: EnrichmentMetadata { TotalCostUsd=0 prod, TotalLatencyMs,
  ProducerVersion }`.

Raw block payloads (NUM `value`, EVT `polarity`, CLAIM
`subject/predicate/object`) live on `MergedExtractionResult.Extractions`
beside the envelope; 5.1 captures what it needs at DTO-construct time.
Per EnrichedDocument design note (L15–24), the full Python DSL AST is
**not** marshalled into C# — keys on block ids only.

### 3.2 Output — `MatrixCellUpdate` (new)

```csharp
public sealed record MatrixCellUpdate(
    string SectorCode,            // NAICS 2-digit (entity OR tagger source)
    string? IndustryCode,         // NAICS 3-/4-/6-digit; null when only sector confident
    DateTimeOffset TimeBucket,    // see §6 — bucket granularity per open Q1
    decimal SignalMagnitude,      // computed per §6 formula; clamped to ±3 (matches MatrixCell)
    short Polarity,               // -1 | 0 | +1 from EVT polarity slot
    double Confidence,            // aggregate confidence ∈ [0, 1]
    SourceProvenance Provenance,  // audit trail back to source article
    string DslBlockId,            // EVT or NUM block id this update was derived from
    DateTimeOffset GeneratedAt);

public sealed record SourceProvenance(
    string SourceDocumentRef,     // SourceDocument.Source
    string SourceTimestamp,       // SourceDocument.Timestamp (ISO-8601)
    string DslVersion,            // "v15"
    string ProducerVersion,       // EnrichmentMetadata.ProducerVersion
    Guid? RawContentId);          // SentinelCollector RawContent row id, when available
```

`MatrixCellUpdate` is a write-once transport contract — it carries
enough to (a) idempotently persist into `matrix_cells` and (b) reverse
the audit (§7) back to the source article. Idempotency key tuple is
`(sector_code, industry_code, time_bucket, source_document_ref,
dsl_block_id)` — the same article re-extracted produces the same
update (no double-count); a different article on the same bucket sums
into the same cell per §6 aggregation rule.

---

## 4. §8.1 — What feeds the matrix (parent plan)

Per parent §8.1 (L386–391):
1. **Validated NUM blocks** in EVTs whose subject resolves via
   `EntityResolutions` to a sector OR whose wrapping EVT has a
   `EventSectorTaggings` tag → magnitude contribution.
2. **Validated EVT blocks with non-zero polarity** → direction (DSL
   slot `polarity: + | - | 0`, parent §6.2 L298).
3. **CLAIM confidence < threshold** → downweight/discard. Phase 4.7
   calibrated `claim_pass_rate ≥ 0.70`; 5.2 reads
   `ClaimVerification.Confidence` directly.

**Direct cutover** at 5.5 per `[[feedback-drop-feature-flags]]`;
rollback = git revert + ansible (§10).

---

## 5. §8.2 — What does NOT go in the matrix (safety property)

Per parent §8.2 (L393–399). Matrix never receives a value not anchored
to a source span. 5.3 filter drops:

- **Unvalidated claims** — `hard_fail` blocks already absent from
  `EnrichedDocument` (Phase 4.5 contract); 5.3 re-checks defensively
  at the bridge boundary.
- **Numerics failing copy-slot byte-check** — `verifier_v2_3_1` gates
  upstream; absent from envelope.
- **Entities not in catalog** — `GroundingSource == NoResolution` →
  drop the entity-derived sector contribution. EVT sector tagger may
  still tag the EVT independently; tagger contribution flows.
- **Verifier `unrelated` / `insufficient_evidence`** — both map to
  `ClaimSupport.None` (`ClaimVerification.cs:42`). Drop CLAIMs with
  `Support=None AND Confidence < 0.70`. `Support=Partial` downweights
  ×0.5 per §6.3, not dropped.
- **Sector tag below floor** — `Tags` empty → EVT contributes no
  sector signal (Phase 4 §9 Q2 decision). "No signal", not "drop EVT"
  — other EVTs in the doc still flow.

Audit invariant: every `MatrixCellUpdate` written carries a non-null
`SourceProvenance.SourceDocumentRef` + `DslBlockId`. The Phase 5.5
bridge fails closed if either is missing (DI bug, not runtime
transient — throw at boundary per `BOUNDARY_HANDLING`).

---

## 6. Signal magnitude formula

Deterministic, no LLM. Mirrors the FRED `CellProjector` shape (`signal
× sectorWeight × freshness × temporal × confidence × patternWeight`,
clamp ±3) so outputs populate the same `matrix_cells` columns the
dashboard reads.

**6.1 Per-EVT magnitude.** For each EVT with ≥1 sector source
(`Tags` rank=1 OR resolved entity sector):
```
sector_code      = tag.SectorCode (rank=1) OR entity.SectorCode
industry_code    = tag.IndustryCode OR entity.IndustryCode
polarity         = evt.polarity                 (-1|0|+1; DSL slot)
raw_magnitude    = num_contribution_sum(evt)    # §6.2
confidence       = aggregate_confidence(evt)    # §6.3
freshness        = freshness_decay(...)         # §6.4
signal_magnitude = clamp(polarity × raw_magnitude × confidence × freshness, -3, +3)
```
±3 clamp matches `MatrixCellEntity.CellValue` (`numeric(6,4)`).

**6.2 NUM-to-magnitude.** NUMs ref their EVT via `numerics: [<num_ref>]`
(parent §6.2 L282). Per NUM:
`num_signal = sign(value) × log10(1+|value|) / log10(1+anchor)`,
`anchor=1e9` USD (override `data/num_scale_anchors.jsonl`).
$1M ≈ 0.5, $1B ≈ 1.0, $1T ≈ 1.3. Sum across EVT's NUMs → `raw_magnitude`
pre-clamp.

**6.3 Aggregate confidence.** `confidence = ent_weight × claim_weight
× tag_weight`:
- `ent_weight = EntityResolutions[primary_subject].Confidence` if
  grounded; 0.5 if sector tag exists but entity unresolved.
- `claim_weight = 1.0` (any Full) / `0.5` (any Partial) / `1.0` (no
  CLAIM references EVT — NUM-only). Support=None already dropped by §5.
- `tag_weight = EventSectorTaggings[evt_id].Tags[0].Confidence`.

**6.4 Freshness.** `freshness = exp(-ln(2) × age_days / halflife)`,
default halflife=7d (override `data/freshness_halflife_days.jsonl`).
0d → 1.0; 7d → 0.5; 30d → ~0.05. Reuses the FRED-populated `freshness`
column; `atlas-matrix-trajectory.json` renders unchanged.

**6.5 Bucket + aggregation.** Bucket (Q1) default **daily**
UTC-midnight (matches `SectorPhaseAggregateEntity.WindowStart`).
Aggregation (Q2) default **weighted-mean**, weights = `confidence
× freshness`. Sum double-counts re-extraction; max lets outliers
dominate. Mirrors `SectorAggregator.Aggregate` contract.

**6.6 Implementation.** Pure function `SignalCalculator.Calculate(
enrichedDoc, mergedExtraction) -> IReadOnlyList<MatrixCellUpdate>`.
C# `ThresholdEngine.Matrix.Sentinel` (production); Python reference
`dsl/matrix_calculator.py` (PoC corpus). Parity tested in 5.2.

---

## 7. Component inventory

### 7.1 SentinelCollector → ThresholdEngine bridge (NEW)

`ExtractionProcessor.TryRunSemanticVerifierAsync` (L1158–1207)
already computes `EnrichedDocument`; today it's observed-only
(L1184–1186 set Activity tags). 5.4 adds `MatrixCellEnrichmentPublisher`
in `SentinelCollector.Semantic` that runs the §6 calculator + §5
filter and publishes `SentinelMatrixUpdateEvent` via the existing
`IEventBus` (in-process). For cross-container delivery: add
`sentinel_matrix_update = 15` to the oneof in
`observation_events.proto` (backward-compatible — existing consumers
ignore the new variant). Idempotency key lives on the update (§3.2),
not the event ID.

### 7.2 ThresholdEngine `MatrixCellSentinelWorker` (NEW)

Today `matrix_cells` has one writer: FRED via
`ObservationEventSubscriber` → `CellProjector` →
`MatrixCellPersistenceWorker`. 5.5 adds `MatrixCellSentinelWorker`
(clone of `MatrixCellPersistenceWorker`: channel / coalesce / clamp /
drop-write, `MatrixCellPersistenceWorker.cs:104–127`) subscribing to
`SentinelMatrixUpdateEvent`, writing via existing
`IMatrixCellRepository.WriteBatchAsync`. Schema overlap → §7.3.

### 7.3 `matrix_cells` schema (KEEP / extend)

**Current.** `MatrixCellEntity` (`matrix_cells` hypertable,
`MatrixCellEntity.cs:28`) — columns Id, EvaluatedAt, PatternId,
SectorCode, CellValue, Signal, SectorWeight, Freshness,
TemporalMultiplier, Confidence, PatternWeight, RollupVersion (null),
ContributingObservationRefs jsonb (null), CreatedAt. Idempotency
index `(pattern_id, sector_code, evaluated_at)` (EF migration
`20260512023230_CreateMatrixCells`).

**5.1 migration (EF per CLAUDE.md DATABASE_MIGRATIONS):**
1. Re-purpose `PatternId` to carry either pattern id OR Sentinel block
   ref `sentinel:{source_document_ref}:{dsl_block_id}` (deterministic
   from `SourceProvenance` + `DslBlockId`) — index tuple doubles as
   Sentinel idempotency key.
2. Add nullable `industry_code text` — NULL for FRED rows (sector-only),
   populated for Sentinel when tagger emits industry drill-down. Index
   `(sector_code, industry_code, evaluated_at)`.
3. Add nullable `polarity smallint` — FRED NULL (signed `Signal` carries
   direction), Sentinel ∈ {-1, 0, +1}.
4. Add nullable `source_provenance jsonb` — populated by Sentinel,
   NULL on FRED. Holds §3.2 `SourceProvenance` for §7.5 audit JOIN.

Migration name `Phase5SentinelMatrixColumns`. Command:
```
nerdctl compose exec -T threshold-engine-dev \
    dotnet ef migrations add Phase5SentinelMatrixColumns --project src
```
No data migration: nullable + new index, default NULL on existing rows.

### 7.4 MacroSubstrate (KEEP — no changes)

`macro_observations` (Epic 4 4.4.2, PR #246) is the Sentinel qualitative
sink — raw per-observation rows, no aggregation. Phase 5 routes directly
to `matrix_cells` (the projected signal); `macro_observations` is the
raw observation. Different purposes; untouched.

### 7.5 Audit trail (NEW — JOIN-on-demand)

**Decision (per Q5):** JOIN-on-demand, not separate table.
`source_provenance` JSONB on `matrix_cells` carries `(SourceDocumentRef,
SourceTimestamp, DslVersion, ProducerVersion, RawContentId)`. Query:
```sql
SELECT raw_content.*, mc.*
FROM threshold_engine.matrix_cells mc
JOIN sentinel.raw_content ON raw_content.id =
    (mc.source_provenance->>'rawContentId')::uuid
WHERE mc.id = $1;
```
PK + JSONB extract → <1s per row. Rejected alternative: separate
`matrix_cell_audit` table — extra write + duplicated fields without
read-path win. Q5 surfaces the alternative for user.

---

## 8. Phased PR sequence

Eight PRs. Each ≤500 LOC, independently mergeable, no PR depends on a
future PR. Per `[[feedback-drop-feature-flags]]`: no default-OFF flag;
integration glue at Phase 5.5 is the cutover. Per
`[[feedback-no-placeholder-prs]]`: each PR ships substantive impl.

### Phase 5.1 — schema migration + `MatrixCellUpdate` DTO

**Scope.** EF migration `Phase5SentinelMatrixColumns` adding
`industry_code text`, `polarity smallint`, `source_provenance jsonb`
to `matrix_cells` + index `(sector_code, industry_code, evaluated_at)`.
Shared `MatrixCellUpdate` + `SourceProvenance` records in `Events`
project so producer (Sentinel) + consumer (TE) read one shape.

**Files (new):** `ThresholdEngine/src/Data/Migrations/{ts}_Phase5SentinelMatrixColumns.cs`
(+ Designer + ModelSnapshot update); `Events/src/Events/{SentinelMatrixUpdateEvent,
MatrixCellUpdate, SourceProvenance}.cs`.
**Files (modified):** `MatrixCellEntity.cs` + `MatrixCellConfiguration.cs`
(new properties + JSONB conversion).
**Acceptance.** Migration applies + reverts on throwaway TimescaleDB;
existing rows readable; DTO ctor invariants tested (sector_code non-empty,
polarity ∈ {-1,0,+1}, confidence ∈ [0,1], provenance non-null).

### Phase 5.2 — `SignalCalculator` (pure library)

**Scope.** Pure function per §6. C#: `ThresholdEngine.Matrix.Sentinel.SignalCalculator`.
Python reference: `dsl/matrix_calculator.py` for PoC corpus runs.

**Files (new):** `ThresholdEngine/src/Matrix/Sentinel/SignalCalculator.cs`
+ `SignalCalculatorOptions.cs`; data files
`data/num_scale_anchors.jsonl`, `data/freshness_halflife_days.jsonl`;
unit tests `ThresholdEngine/tests/Unit/Matrix/Sentinel/SignalCalculatorTests.cs`;
Python `dsl/matrix_calculator.py` + `dsl/tests/test_matrix_calculator.py`.
**Acceptance.** Unit tests: (a) NUM log-scale on anchors (1M=0.5, 1B=1.0,
1T=1.3); (b) polarity propagation; (c) confidence weighting per §6.3;
(d) 7d half-life decay; (e) ±3 clamp; (f) deterministic byte-identical
runs. PoC parity: Python + C# produce matching updates on fixture.

### Phase 5.3 — `EnrichmentFilter` (the §5 safety property)

**Scope.** Pure function enforcing §5 drops. Emits per-reason drop-count
+ surviving subset so Phase 5.7 can dashboard.

**Files (new):** `ThresholdEngine/src/Matrix/Sentinel/{EnrichmentFilter,
FilteredEnrichment, DropReason}.cs` (DropReason enum =
`EntityNoResolution | ClaimSupportNoneLowConfidence | SectorTagBelowFloor
| EvtHardFail | MissingProvenance`); unit tests under
`ThresholdEngine/tests/Unit/Matrix/Sentinel/`.
**Acceptance.** Per-reason fixture triggers exactly one drop; counts
surface per reason; all-pass fixture has zero drops; empty
EnrichedDocument → empty output no-throw. Every drop reason exercised
per `[[feedback-no-placeholder-prs]]`.

### Phase 5.4 — `MatrixCellEnrichmentPublisher` (Sentinel bridge)

New publisher in `SentinelCollector.Semantic` consuming
`EnrichedDocument` + `MergedExtractionResult`, running `EnrichmentFilter`
+ `SignalCalculator` (5.2/5.3 live in shared `Events` so SentinelCollector
refs without a TE project ref), publishing `SentinelMatrixUpdateEvent`
via existing `IEventBus`. Wired into `ExtractionProcessor.TryRunSemanticVerifierAsync`
(`ExtractionProcessor.cs:1158`) replacing today's observed-only path.
gRPC oneof gains `sentinel_matrix_update = 15` in `observation_events.proto`
(backward-compatible).
**Files**: new `{MatrixCellEnrichmentPublisher,IMatrixCellEnrichmentPublisher}.cs`
+ tests; mod `ExtractionProcessor.cs`, `observation_events.proto`,
`DependencyInjection.cs`.
**Acceptance**: (a) calls filter→calculator→bus in order; (b)
zero-update → empty publish (heartbeat); (c) bus throw fail-soft with
counter `atlas_sentinel_matrix_publish_failures_total`; (d) in-proc
event-bus integration fixture → captured consumer; protobuf regen clean.

### Phase 5.5 — `MatrixCellSentinelWorker` (TE CUTOVER)

New `BackgroundService` cloning `MatrixCellPersistenceWorker`
(channel + coalesce + clamp + drop-write,
`MatrixCellPersistenceWorker.cs:104–127`); subscribes to
`SentinelMatrixUpdateEvent`; writes via existing
`IMatrixCellRepository.WriteBatchAsync`. **CUTOVER**: live consumer for
5.4's publisher. No flag per `[[feedback-drop-feature-flags]]`.
**Files**: new `{MatrixCellSentinelWorker,...Options}.cs` + tests;
mod `DependencyInjection.cs`, `Program.cs`, `MatrixCellRepository.cs`
(extend WriteBatchAsync for new columns).
**Acceptance**: integration test (in-proc bus + EF in-memory) publishes
3 updates → 3 rows with `pattern_id="sentinel:..."` +
industry_code/polarity/source_provenance populated; republish →
0 new (idempotency); 50k saturation → WriteFailures ticks,
rate-limited warning, FRED writes uninterrupted.

### Phase 5.6 — Audit trail (JOIN-on-demand endpoint + MCP)

REST `GET /matrix-cells/{id}/source-article` returning §7.5
JOIN-on-demand: reads `source_provenance` JSONB, dereferences
`rawContentId` against `sentinel.raw_content` (cross-DB per Epic
1/2/3 conn-string pattern). MCP tool wraps the same query (matches
Epic 6 matrix-cells MCP).
**Files**: new `MatrixCellAuditEndpoints.cs` (extends
`MatrixCellEndpoints.cs`); `MatrixCellAuditService.cs` +
`IMatrixCellAuditService.cs`; tests; MCP tool registration.
**Acceptance**: insert row with provenance → fixture `raw_content`
row → endpoint returns source article p95 < 1s per §14. Latency on
`atlas_matrix_audit_lookup_duration_ms`.

### Phase 5.7 — Observability

Pairs with `v2-pipeline-health.json` + `atlas-matrix-primary.json`
/ `atlas-matrix-trajectory.json`.

**Metrics** (bounded cardinality — sector=11, reason=5, outcome=2;
no article/block ids in tags):
`atlas_sentinel_matrix_updates_published_total{outcome}`,
`..._updates_persisted_total{sector}`,
`..._drops_total{reason}`,
`..._cell_freshness_seconds` (histogram, ingestion lag),
`atlas_matrix_audit_lookup_duration_ms` (histogram),
`..._queue_drop_total` (mirrors FRED `MatrixCellQueueDrop`).

**Panels** (new "Sentinel matrix integration" row):
(1) published vs persisted rate; (2) drop rate by reason stacked
(5 series); (3) per-sector fill rate 24h (11 series); (4) cell
freshness p50/p95.

**Alerts**: P2_URGENT queue-drop>0 for 15 min (backpressure);
P2_URGENT persisted_rate==0 while published_rate>0 for 10 min (bridge
dropped); P3_NOTIFY any single drop reason >50% total for 30 min
(producer regression).

**Files**: `ThresholdEngineMeter.cs` + `SentinelMeter.cs` (counters/histograms);
`v2-pipeline-health.json` (4 panels); `deployment/artifacts/monitoring/alerts/*.yaml`
(3 alerts).
**Acceptance**: metrics in Prometheus after n=10 smoke; 4 panels
render; `promtool check` clean.

### Phase 5.8 — n=10 acceptance + production deploy

**Scope.** Parent plan §8.3: end-to-end demo on n=10 R2/R3 articles;
per-article wall time within Q3 budget; per-cell auditable back to
source.

**Files (results only):**
`docs/benchmarks/cod-2026-05-17/results/phase5-acceptance-n10/{REPORT.md,
source_articles/*.json, matrix_cells.csv, audit_lookups.jsonl}`.
**Gates.** (1) n=10 within Q3 wall-time budget; (2) every cell
audit-resolves in <1s per §14; (3) cell freshness p95 < 5 min;
(4) `ClaimSupport.None` low-conf never appears in matrix (spot-check
10 dropped CLAIMs from drops counter, confirm zero rows reference
their `dsl_block_id`).
**Deploy.** `ansible-playbook playbooks/deploy.yml --tags
sentinel-collector,threshold-engine` per CLAUDE.md DEPLOYMENT.
Smoke test post-deploy per CLAUDE.md VERIFY_TEST COMPLETION_GATE.
NTFY milestone per `[[feedback-ntfy-not-inline]]`.

---

## 9. §8.3 — Acceptance criteria (parent plan)

Parent §8.3 (L402–406): (1) n=10 R2/R3 demo end-to-end (5.8 gate 1);
(2) per-article wall time within production budget — TBD, §12 Q3;
(3) per-cell auditable to source (5.6 + 5.8 gate 2).

---

## 10. Rollback strategy

Per `[[feedback-drop-feature-flags]]`: no default-OFF flag. Additive
PRs; cutover at 5.5.

- **5.1 (schema)**: EF rollback removes nullable columns + index
  cleanly. Sentinel rows wipe via `DELETE FROM matrix_cells WHERE
  pattern_id LIKE 'sentinel:%'` (deterministic prefix from §7.3).
- **5.2 / 5.3 (libraries)**: code revert, no runtime state.
- **5.4 (publisher)**: `git revert` removes the publish call; pre-5.4
  observed-only behaviour restores. Protobuf addition is
  backward-compatible — orphan oneof variant cleans up in a follow-up.
- **5.5 (TE worker, CUTOVER)**: `git revert` + ansible redeploy. The
  publisher keeps publishing; events drop at gRPC (no subscriber); no
  DB write. Wipe pre-revert rows via the §5.1 DELETE if needed.
- **5.6 (audit)** / **5.7 (obs)**: `git revert` removes endpoint /
  panels / alerts. No state.
- **5.8 (acceptance + deploy)**: results-only.

**Hard constraint** (mirrors Phase 4 §7): no partial rollback. Broken
component → follow-on fix PR. ZFS snapshot `/var/lib/atlas` before
5.5 + 5.8 deploys as catch-all per `[[feedback-drop-feature-flags]]`.

---

## 11. Observability deliverables

Detail in §8 Phase 5.7. New "Sentinel matrix integration" row on
`v2-pipeline-health.json`: throughput (published vs persisted), fill
rate per sector (11 series), drop rate by reason (5 series), audit
lookup p50/p95, cell freshness p50/p95. Alerts: 2 P2_URGENT
(backpressure, publish↔persist gap) + 1 P3_NOTIFY (drop-reason spike).

---

## 12. Open questions — for user

Defaults proposed in §6; need user sign-off before 5.2 + 5.5 ship.
NTFY per `[[feedback-ntfy-not-inline]]`.

1. **Time bucket granularity.** Default **daily** UTC-midnight
   (matches `SectorPhaseAggregateEntity` window). Hourly = ~24× row
   volume; per-source-publish = irregular but most reactive.
2. **Aggregation rule** (multiple updates → same cell). Default
   **weighted-mean** (weights = `confidence × freshness`). Sum
   double-counts re-extraction; max lets outliers dominate.
3. **Production wall-time budget per article.** Parent §8.3 left TBD.
   Phase 5 adds calculator (<10 ms) + publish/persist (<50 ms in-proc) —
   negligible additive. Real question: end-to-end article → matrix-cell
   SLO. Phase 4.7 p95 enrichment latency captured in 5.8 REPORT.
4. **Freshness half-life.** Default 7 days. Per-source override table
   proposed (24h earnings vs 30d macro commentary).
5. **Audit trail: JOIN-on-demand vs separate table.** JOIN-on-demand
   (§7.5, proposed) is simpler + same audit power. Separate table =
   faster bulk audit but 2× write + duplicated source of truth.

---

## 13. Risks + mitigations

**Risk 1 — Cell-throughput volume unknown.** ~5 updates/article ×
~1k articles/day ≈ 5k Sentinel writes/day vs ~100k FRED writes/day
(rough). Could overshoot if articles average 20+ updates.
*Mitigation:* 5.5 worker reuses the FRED `MatrixCellPersistenceWorker`
bounded-channel/drop-write pattern; 5.7
`atlas_sentinel_matrix_queue_drop_total` alerts on saturation.

**Risk 2 — Sparse sectors look like "broken pipeline".** NAICS 11
(agriculture) rarely hits financial-news vs 52 (finance).
*Mitigation:* 5.7 fill-rate panel surfaces per-sector fill rate
(11 series); empty sectors render as flat zero with description noting
expected sparsity.

**Risk 3 — Audit-trail storage growth.** `source_provenance` JSONB
on every Sentinel row. 5k/day × 365 × ~200B ≈ 365 MB/year (50k/day →
3.6 GB/year).
*Mitigation:* TimescaleDB JSONB chunk compression already enabled. No
retention at PoC scale; revisit Phase 6+ if volume overshoots. Growth
visible on existing TimescaleDB chunk-size panels.

---

## 14. Success criteria (consolidated, per parent §8.3)

Concrete + measurable, all reported in 5.8 REPORT.md:

1. **n=10 end-to-end within Q3 wall-time budget** — histogram + p50/p95
   in REPORT.
2. **Cell update latency p95 < 5 min** from extraction →
   matrix-cell-available (`atlas_sentinel_matrix_cell_freshness_seconds`).
3. **Audit lookup p95 < 1 s** for `/matrix-cells/{id}/source-article`
   incl. cross-DB JOIN (`atlas_matrix_audit_lookup_duration_ms`).
4. **Drop rate populated for all 5 reasons** across the corpus
   (`atlas_sentinel_matrix_drops_total{reason}`).
5. **Hard safety: `ClaimSupport.None` low-conf never reaches matrix.**
   Spot-check 10 dropped CLAIMs → confirm zero rows reference their
   `dsl_block_id`.

All five clear → Phase 5 closes; DSL PoC end-to-end loop demonstrated.
Any miss → diagnose via 5.2/5.3 unit tests + 5.7 panels before
declaring done.

---

## 15. Phase ordering

5.1 first (schema + DTOs); 5.2 + 5.3 parallel after 5.1 (pure libs);
5.4 after 5.2 + 5.3; 5.5 after 5.4 (CUTOVER); 5.6 + 5.7 parallel
after 5.5; 5.8 last. Parent §11 budgets ~3 days; 8-PR slice runs ~1
week with 5.2/5.3 + 5.6/5.7 parallelised per
`[[feedback-parallel-agents-use-worktrees]]`.

---

## 16. References

- Parent: `docs/plans/atlas-dsl-poc-plan.md` §8 (L358–406), §10 (L421–435).
- Phase 4: `docs/plans/atlas-dsl-poc-phase4-gpu-semantic-verifier.md`.
- Phase 4.1 contract + 4.5 orchestrator:
  `SentinelCollector/src/Semantic/{EnrichedDocument,EntityResolution,
  ClaimVerification,EventSectorTagging,SemanticVerifierService}.cs`.
- Producer wiring: `SentinelCollector/src/Workers/ExtractionProcessor.cs:1089–1207`.
- Matrix store: `ThresholdEngine/src/Data/Entities/MatrixCellEntity.cs`,
  `ThresholdEngine/src/Workers/MatrixCellPersistenceWorker.cs`,
  `ThresholdEngine/src/Services/CellProjector.cs`,
  `Events/src/Events/CellProjectedEvent.cs`.
- Event bus + gRPC: `Events/src/Events/Protos/observation_events.proto`,
  `Events/src/Events/ObservationCollectedEvent.cs`.
- Derived view: `ThresholdEngine/src/Data/Entities/SectorPhaseAggregateEntity.cs`.
- Dashboards: `deployment/artifacts/monitoring/dashboards/{v2-pipeline-health,
  atlas-matrix-primary,atlas-matrix-trajectory}.json`.
- Memory rules: `[[sentinel-llm-strength-layering]]`,
  `[[atlas-inference-topology]]`, `[[feedback-drop-feature-flags]]`,
  `[[feedback-no-placeholder-prs]]`, `[[feedback-parallel-agents-use-worktrees]]`,
  `[[feedback-ntfy-not-inline]]`.
