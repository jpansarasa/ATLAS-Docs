# SentinelCollector — architecture [agent read-first]

NOTE: this card covers the news→matrix pipeline which spans SentinelCollector AND MacroSubstrate (see MacroSubstrate/README for the shared write path).

PURPOSE: news-article → `(signal×sector)` matrix tilt via GPU vLLM JSON-CoD extraction (Extraction:Backend=VllmJson, Qwen2.5-32B-AWQ) + `:sig:` rows in macro_observations. ¬gRPC cell push ¬digest ¬extraction/CoVe/CoD paths addressed here. (CPU llama-server DSL extraction = rollback path.)

DATA MODEL + INVARIANTS:
  INV `:sig:` infix: news row identified SOLELY by literal `:sig:` in source_id (`{rawContentId}:sig:{signalId}`). ¬schema-enforced — string contract; FOUR artefacts move together: producer (MacroObservationRouter) + projector const (ObservationCellProjector) + consumer (NewsMomentumQueryService) + digest reader (DigestQueryService.ArticleSignalsSql + TryParseRawContentId). change-all-or-none.
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
  matrix feed (macro_observations→projector→matrix_cells) ≠ digest consumer (matrix_cells+sector_regimes+sentinel rows, separate read; theme taxonomy retired — sector is the only digest axis) ≠ extracted_observations (legacy numeric fallback sink, ¬read by matrix or digest).
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
  ✗ assume-CandidateSurfaceFilter-catches-all-junk: STRUCTURAL junk is rejected at ingress (D-1, #824) and markup stripped at the normalizer last-resort (D-2, #825), but SEMANTICALLY-ambiguous surfaces (outlet names, no-digit slugs) Keep BY DESIGN → SecMaster resolve → its budget-gated Gemini last-resort (#823, company-name→ticker ONLY; ungated it drained $100/6d 2026-06-30). Defense-in-depth ¬redundancy: ingress cleans the source, SecMaster gates the $. [[GIGO]] [[INTENT_FIDELITY]]
  ✗ extracted_observations-fallback-for-news ✗ assume-digest-sees-only-`:sig:`-rows ✗ Shadow=dry-run
  ✗ qualitative(null-signal)-rows-reach-matrix_cells ✗ zero-matrix_cells-no-error=broken(check-Mode-first)
  ✗ change-`:sig:`-infix-in-one-artefact ✗ CollectedAt-as-wall-clock ✗ numeric-fallback-applies-to-news
  ✗ incomplete-SectorWeights-writes-partial-cells ✗ prepass-as-entry-gate

DECISIONS:
  D-1 ingress-junk-filter: INTENT reject non-entity surfaces at the ingress where garbage is BORN (#824) — junk poisoned FRED search (#818), paid Gemini (#823) AND resolution correctness for FREE (wrong-ticker → corrupt matrix); one source fix covers every consumer / PRECOND reject STRUCTURALLY-invalid only (markup-chars|dotted-id|slug+digit|money/number|metric-abbrev|country|institution|crypto); semantically-ambiguous names Keep → SecMaster decides (dropping a real equity = the expensive error); destination gates stay defense-in-depth / GUARD EntityResolutionPrepass.ApplySurfaceFilter @ src/Services/EntityResolutionPrepass.cs:300 / TEST EntityResolutionPrepassFilterTests.enforce_mode_removes_junk_candidates_before_secmaster_call
  D-2 html-strip-last-resort: INTENT the last-resort path (both normalizers failed) must NEVER emit markup as content (#825) — raw DOCTYPE/tags/script bodies reach the LLM extractor + spaCy NER as bogus entities / PRECOND strip failure → empty ¬raw (fail-closed); OptionMaxNestedChildNodes=1000 converts an UNCATCHABLE StackOverflow (deep nesting; ~4.5k tags crashed the 1MB worker stack) into a catchable load failure / GUARD ContentNormalizer.StripHtmlFallback @ src/Services/ContentNormalizer.cs:267 / TEST ContentNormalizerTests.should_return_visible_text_not_markup_when_both_normalizers_fail_on_html
  D-3 llm-signal-validation: INTENT LLM output is never trusted into macro_observations — hallucinated ids/NaN/sub-floor dropped + counted per reason (feed must never silently re-zero); tilt clamp [-1,1] = SOURCE enforcement of INV |v|≤1 (projector 1.5 gate = destination backstop) / PRECOND classifier is additive enrichment: fail-soft→Empty, never breaks extraction / GUARD NewsSignalClassifier.Validate @ src/Semantic/NewsSignalClassifier.cs:192 / TEST NewsSignalFeedWiringTests.should_drop_hallucinated_id_so_only_valid_signals_route
  D-4 gpu-prompt-budget: INTENT refuse an over-budget prompt BEFORE the GPU call (throw PromptTooLargeException, zero HTTP) — context-overflow 400s waste GPU round-trips and mis-log as extraction errors / PRECOND callers assembling prompts client-side derive budgets from PromptTokenAllowance/ExactPromptTokenAllowance (guard+caller can never disagree); structured path verifies vLLM /tokenize exact count — chars/4 under-counts ticker-dense prose (#692) / GUARD VllmClient.EnforcePromptBudget @ src/Services/VllmClient.cs:597 / TEST VllmClientPromptSizeGuardTests.should_not_send_http_request_when_prompt_too_large_for_generate

SEE: README.md §Reference · Events/src/Events/Protos/observation_events.proto (ObservationEventStream — EXPOSES to ThresholdEngine + CONSUMES from ThresholdEngine for validation) · Events/src/Events/Protos/secmaster.proto (SecMasterRegistry — registration) · src/Semantic/NewsSignalClassifier · src/Services/MacroObservationRouter+EntityResolutionPrepass · src/Workers/ExtractionProcessor · ThresholdEngine src/Workers/ObservationCellProjector · MacroSubstrate MacroObservationRepository · NewsMomentumQueryService (digest)
