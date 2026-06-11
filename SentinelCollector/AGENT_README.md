# SentinelCollector — architecture [agent read-first]

NOTE: this card covers the news→matrix pipeline which spans SentinelCollector AND MacroSubstrate (see MacroSubstrate/README for the shared write path).

PURPOSE: news-article → `(signal×sector)` matrix tilt via GPU vLLM JSON-CoD extraction (Extraction:Backend=VllmJson, Qwen2.5-32B-AWQ) + `:sig:` rows in macro_observations. ¬gRPC cell push ¬digest ¬extraction/CoVe/CoD paths addressed here. (CPU llama-server DSL extraction = rollback path; CpuInference disabled.)

DATA MODEL + INVARIANTS:
  INV `:sig:` infix: news row identified SOLELY by literal `:sig:` in source_id (`{rawContentId}:sig:{signalId}`). ¬schema-enforced — string contract; THREE artefacts move together: producer + projector const + consumer. change-all-or-none.
  INV signal⊥sector: signal_identity_id=SIGNAL dim gates projection; atlas_sector_code=SECTOR dim ¬gates. null-signal→never projects (qualitative rows skipped permanently).
  INV |v|≤1: `:sig:` rows carry |value|≲1; projector excludes |v|>1.5 (raw-value leak guard); drops non-`:sig:` rows from decay sum even if same group (G2 mixed-group drop).
  INV heal-on-rewrite: macro_observations = authoritative cell source; projector re-projects in place (DO UPDATE). ¬DO NOTHING.
  INV sector-dim⊥entry: resolve-entities sectorless/fail → row still written (sector=null or fallback); prepass feeds sector-pick inside classify ¬entry gate.
  INV incomplete-SectorWeights: group whose pattern's SectorWeights missing any sector → ArgumentException caught → group skipped, 0 cells, logged Warning ¬Error.
  INV decay: news magnitude = K·tanh(S/K) over half-life-weighted decay sum (freshness=1.0, no floor). K+H from Matrix:NewsDecay. ≠ hard-data mean formula.

PATHS (distinct code — do not conflate):
  NewsSignalClassifier [in-proc · inside ExtractionProcessor]
    does: structured vLLM call; inlines signal_identities catalog + 11 sector codes; validates ids; drops <0.5 confidence / NaN / hallucinated.
    does NOT: ¬throw (fail-soft→Empty) ¬parse free-form ¬touch DB.
    on-miss: empty catalog / empty excerpt / timeout / unstructured → NewsSignalClassification.Empty; extraction proceeds.
    ⚠ excerpt capped before call — signal in tail → Empty ¬failure. Sub-floor confidence empties feed silently (counted ¬errored).
  MacroObservationRouter writer [in-proc · MacroSubstrate]
    does: upsert value_numeric=tilt×confidence; source_collector="sentinel"; `:sig:` source_id.
    does NOT (NEWS caller): ¬fall back to extracted_observations — Failed=row LOST (by design). ¬count IdempotentSkip as progress.
    ⚠ CollectedAt stored as UTC instant of its offset ¬source wall-clock; matters for idem-key matching.
    ⚠ NUMERIC caller DOES fall back to extracted_observations two-phase. Fallback behavior diverges by CALLER ¬method.
  ObservationCellProjector [ThresholdEngine background · DB-poll]
    does: read Kind=Numeric DESC; group by (signal_identity_id,source_collector) → 11 sector rows/group; decay sum → K·tanh(S/K) × trust × temporal × confidence × sectorWeight clamped ±3; upsert matrix_cells.
    does NOT: ¬gate on sector ¬project null-signal rows ¬project unknown/disabled patterns ¬write partial cell-set for incomplete SectorWeights.
    ¬runs when Matrix:ObservationProjectorMode=Off → zero matrix_cells + no error. CHECK this config FIRST before concluding pipeline broken.
    ⚠ Shadow ¬dry-run: runs identical read+project+write as Authoritative (same cells written); only Off suppresses writes.
    ⚠ read DESC + hard cap → sustained ingest silently drops oldest (cap-induced staleness, metered).

PROCESSING ORDER (per article): normalize → EntityResolutionPrepass(spaCy NER→SecMaster,fail-soft) → V2 pipeline → TryClassifyNewsSignalsAsync → per-signal preferredSector=grounded??classifier.Sector → plan → write → addRange.
MAXIM: prepass output FEEDS classify as parameter → router derives grounded sector. Prepass=sector input ¬entry gate.

DISTINCTIONS:
  `:sig:` news row ≠ raw-legacy sentinel row — same table/group; only `:sig:` feeds projector decay sum; legacy raw values dropped (G2).
  signal-dim (signal_identity_id, gates projection) ≠ sector-dim (atlas_sector_code, ¬gates).
  matrix feed (macro_observations→projector→matrix_cells) ≠ digest consumer (matrix_cells+sentinel rows, separate read) ≠ extracted_observations (legacy numeric fallback sink, ¬read by matrix or digest).
  digest news-momentum SQL filters source_collector='sentinel' AND signal_identity_id IS NOT NULL AND value_numeric BETWEEN -1.5 AND 1.5 — ¬`:sig:` infix filter; splits window at midpoint for early→late per-signal tilt trend (replaces former bias-vs-FRED-matrix view, removed: FRED matrix too stale/sparse → fake divergence). Legacy non-`:sig:` sentinel numeric rows WITH signal_identity_id enter digest momentum avg but ARE dropped by projector (G2). Two consumers see DIFFERENT effective inputs.
  news magnitude=decay-weighted SUM(K·tanh(S/K)) ≠ hard-data mean(Σoᵢ/n × freshness scalar). Unexpected news cell magnitude → CHECK Matrix:NewsDecay K/H.
  ThresholdEngine projector(DB-polled) ≠ gRPC MatrixUpdateStream/ObservationEventStream (separate event surface; matrix feed is DB-polled ¬gRPC).

CROSS-SERVICE:
  IN: SecMaster HTTP :8080 /api/signal-identities (catalog warmup) + /api/resolve-entities (sector grounding, fail-soft). vLLM GPU (classify).
  IN: ThresholdEngine gRPC-stream :5001 ObservationEventStream (ValidationEventConsumerWorker — sector-crossing events → validation triggers).
  OUT: macro_observations via in-proc DI ¬network. FEEDS: ThresholdEngine projector (DB-poll) → matrix_cells; digest reads matrix_cells separately.
  EXPOSES gRPC :5001: ObservationEventStream (ThresholdEngine consumes) + SecMaster gRPC :5001 RegisterSeries (f-a-f on extracted observations).

GOTCHAS:
  ✗ gate-news-entry-on-resolve-entities/sector ✗ route-news-via-gRPC-MatrixUpdateStream
  ✗ extracted_observations-fallback-for-news ✗ assume-digest-sees-only-`:sig:`-rows
  ✗ qualitative(null-signal)-rows-reach-matrix_cells ✗ Shadow=dry-run
  ✗ zero-matrix_cells-no-error=broken(check-Mode-first) ✗ change-`:sig:`-infix-in-one-artefact
  ✗ CollectedAt-as-wall-clock ✗ numeric-fallback-applies-to-news
  ✗ incomplete-SectorWeights-writes-partial-cells ✗ prepass-as-entry-gate

SEE: README.md §Reference · Events/src/Events/Protos/observation_events.proto (ObservationEventStream — EXPOSES to ThresholdEngine + CONSUMES from ThresholdEngine for validation) · Events/src/Events/Protos/secmaster.proto (SecMasterRegistry — registration) · src/Semantic/NewsSignalClassifier · src/Services/MacroObservationRouter+EntityResolutionPrepass · src/Workers/ExtractionProcessor · ThresholdEngine src/Workers/ObservationCellProjector · MacroSubstrate MacroObservationRepository · NewsMomentumQueryService (digest)
