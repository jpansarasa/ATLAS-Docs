# ThresholdEngine — architecture [agent read-first]

PURPOSE: owns matrix_cells(11-sector), sector-regimes, sector-phase-view; Roslyn-compiled C# pattern eval; projects per-(signal,sector,cycle) cells. ¬collector ¬signal-registry(SecMaster) ¬alert-dispatcher.

DATA MODEL + INVARIANTS:
  INV cells: 1 row per (pattern_id,sector_code,evaluated_at); upsert ON CONFLICT DO UPDATE IS DISTINCT FROM. every projection=11 rows(explicit-zero ¬sparse).
  INV clamp: [-3,+3] at projector; MatrixCellPersistenceWorker re-clamps defence-in-depth — row still persists.
  INV provenance: denormalized multiplicative-factor copies; audit re-derives ¬re-reads observations.
  INV projector-key: pattern_id="obs:{signal_identity_id}:{source_collector}" (synthetic); evaluated_at=group's latest obs time.
  INV coexistence: FRED⊥Sentinel same signal → distinct rows differ by source_collector ¬overwrite.
  INV signal-id: PatternConfiguration.SignalIdentityId=computed from Metadata["signalIdentity"] ¬mapped field; null if key absent.
  ⚠ PatternConfiguration.Confidence XML-doc reads "Informational only"=FALSE — IS used in wired projector formula. trust code ¬docstring.

PATHS (distinct code — do not conflate):
  WS3-projector [ObservationCellProjector · BackgroundService · 5-min cycle · ONLY WIRED matrix_cells writer]
    does: two windowed keyset-paged reads of macro_observations(Kind=Numeric) — hard-data 366d(SourceIdExcludes=:sig:) + news 7d(SourceIdContains=:sig:); concat→groups by (signal_identity_id,source_collector); Project→WriteBatchAsync upsert. PageSize=MaxLimit=1000/page; time-window-bounded, ¬per-cycle-row-cap.
    formula: magnitude×source_trust×freshness×temporal×Confidence×sector_weight. Confidence=PatternConfiguration.Confidence(STATIC,default 0.75).
    does NOT: ¬gRPC-stream ¬ObservationCache ¬ThresholdEvent.
    coverage: HardObsRead/NewsObsRead logged + traced per cycle (thin hard-data coverage visible); ¬read-cap (windows bound the read, paging drains them). news window = 7×Matrix:NewsDecay.HalfLifeHours(24h).
    mode: absent Matrix:ObservationProjectorMode→Authoritative; typo→Shadow(¬Authoritative,same writes)+deferred-startup-warning. Off→zero cells+no error.
  live-eval [gRPC :5001 · MultiCollectorEventConsumerWorker · SeriesCollected/CollectionFailed]
    does: updates ObservationCache; EvaluateAllEnabledAsync; persists ThresholdEvent on crossing.
    does NOT: ¬CellProjector ¬matrix_cells ¬CellProjectedEvent.
    on-miss: exponential-backoff retry 5s→300s.
  ObservationEventSubscriber [UNWIRED / dead for cell-writes]
    ¬registered in AddWorkers. MatrixCellPersistenceWorker subscribes but RECEIVES NOTHING at runtime. formula(CellProjector.cs)=DEAD CODE. ¬describe as live producer.
  Sentinel-gRPC-bridge: DELETED(MatrixCellSentinelWorker removed WS3-A3). README mermaid(SMW node,MatrixCellUpdate edge)=STALE.

PROCESSING MODEL (projector cycle, in order):
  read descending(freshest-first; ascending→freezes matrix on stale rows) → drop null signal_identity_id → build ONE per-cycle signal→pattern map(GetAllAsync; dup signal_identity_id→FIRST ENABLED wins) → per group: unknown_signal if no pattern; pattern_disabled if Enabled=false(DISTINCT counters; neither projected) → news/hard-data fork → Project → upsert.
  news-path: magnitude=K·tanh(S/K); S=Σvᵢ·exp(−ln2·ageᵢ/H) 24h-half-life decay; freshness=1.0(¬floor; perishable).
  hard-data-path: magnitude=Welford mean + step-decay freshness(10% floor).
  ¬discriminator: source_collector='sentinel' — `:sig:` infix in source_id is the real discriminator.

PATTERN AUTO-DISABLE (load-time ¬per-cycle):
  JSON Enabled:false→dropped at LoadAllAsync BEFORE SecMaster check. JSON-enabled: AllReferencedMnemonics→ResolveBatch gRPC → any series ¬Found/PrimarySource→pattern.Enabled=false+ERROR.
  PatternValidation:Strict=true→accumulated misses throw at boot(fail-fast) ¬soft-disable.
  SecMasterFrequencyResolver: ¬frequency param; NO prefix-fallback; NO exception-swallowing; outage→exception(strict) or mass-auto-disable(soft).

DISTINCTIONS:
  projector-cells(wired,synthetic key,source_trust formula) ≠ live-eval-cells(UNWIRED,real pattern_id,pattern.Weight formula).
  Confidence-STATIC(PatternConfiguration.Confidence,projector,wired) ≠ Confidence-DYNAMIC(dead live-eval path only). conflating=active error risk.
  unknown_signal(no pattern maps signal) ≠ pattern_disabled(mapped;Enabled=false) — DISTINCT counters.
  auto-disable(load-time,SecMaster-miss) ≠ JSON-Enabled:false(dropped at load,¬SecMaster-checked) ≠ pattern_disabled-skip(cycle-time).
  SecMasterFrequencyResolver(load-time,¬frequency,¬fallback) ≠ DataWarmupService(collector-routing,frequency:"any",HAS prefix-fallback).

CROSS-SERVICE:
  IN: collector gRPC :5001 → live eval(threshold events only; ¬matrix_cells).
  OUT→SecMaster gRPC ResolveBatch: SecMasterFrequencyResolver(load-time) + DataWarmupService(collector-routing).
  OUT→ThresholdEngineMcp+AlertService via gRPC ObservationEventStream. REST :8080(macro-observation mapping,dashboards).
  macro_observations: written by SentinelCollector/MacroSubstrate; ThresholdEngine reads only.
  SectorPhaseViewRefreshWorker: 7-day cadence; DB outage→silent stale dashboard.

GOTCHAS:
  ✗ live-FRED-gRPC-writes-matrix_cells(ThresholdEvent only) ✗ ObservationEventSubscriber/CellProjector-as-live-producer
  ✗ merged-cell-formula ✗ trust-Confidence-docstring("informational only"=false) ✗ conflate-static/dynamic-Confidence
  ✗ prefix-fallback-to-SecMasterFrequencyResolver ✗ typo-config→Authoritative(→Shadow) ✗ Sentinel-bridge-as-live(deleted)
  ✗ ascending-projector-read ✗ unknown_signal/pattern_disabled-shared-counter ✗ freshness-floor-on-news-path
  ✗ ResolveBatch-per-cycle(load-time only) ✗ README-mermaid-SMW/MatrixCellSentinel-as-current(stale)

SEE: README.md §Reference (REST/gRPC/health endpoint catalog; config keys; mermaid SMW+MatrixCellSentinel=stale) · Events/src/Events/Protos/observation_events.proto (ObservationEventStream gRPC contract — both consumed from collectors and exposed to downstream) · Events/src/Events/Protos/secmaster.proto (SecMasterResolver.ResolveBatch — consumed at load time) · ThresholdEngine/src/Workers/ObservationCellProjector.cs · src/Workers/ObservationEventSubscriber.cs (UNWIRED)
