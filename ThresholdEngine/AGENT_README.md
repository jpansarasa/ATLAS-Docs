# ThresholdEngine — architecture [agent read-first]

PURPOSE: owns matrix_cells(11-sector), sector-regimes, sector-phase-view; Roslyn-compiled C# pattern eval; projects per-(signal,sector,cycle) cells. Not a collector, not the signal-registry (SecMaster), not an alert-dispatcher.

DATA MODEL + INVARIANTS:
  INV cells: 1 row per (pattern_id,sector_code,evaluated_at); upsert ON CONFLICT DO UPDATE IS DISTINCT FROM. every projection=11 rows(explicit-zero, not sparse).
  INV clamp: [-3,+3] at projector (D-3); MatrixCellPersistenceWorker re-clamps defence-in-depth — row still persists.
  INV provenance: denormalized multiplicative-factor copies; audit re-derives, never re-reads observations.
  INV projector-key: pattern_id="obs:{signal_identity_id}:{source_collector}" (synthetic); evaluated_at=group's latest obs time.
  INV coexistence: FRED⊥Sentinel same signal -> distinct rows differ by source_collector, no overwrite.
  INV signal-id: PatternConfiguration.SignalIdentityId=computed from Metadata["signalIdentity"], not a mapped field; null if key absent.
  ⚠ PatternConfiguration.Confidence XML-doc reads "Informational only"=FALSE — IS used in wired projector formula. trust code not docstring.

PATHS (distinct code — do not conflate):
  WS3-projector [ObservationCellProjector · BackgroundService · 5-min cycle · ONLY WIRED matrix_cells writer]
    does: two windowed keyset-paged reads of macro_observations(Kind=Numeric) — hard-data 366d(SourceIdExcludes=:sig:) + news 7d(SourceIdContains=:sig:); concat->groups by (signal_identity_id,source_collector); Project->WriteBatchAsync upsert. PageSize=MaxLimit=1000/page; time-window-bounded, no per-cycle-row-cap.
    formula: magnitude*source_trust*freshness*temporal*Confidence*sector_weight. Confidence=PatternConfiguration.Confidence(STATIC,default 0.75).
    does NOT: gRPC-stream; ObservationCache; ThresholdEvent.
    coverage: HardObsRead/NewsObsRead logged + traced per cycle (thin hard-data coverage visible); no read-cap (windows bound the read, paging drains them). news window = 7x Matrix:NewsDecay.HalfLifeHours(24h).
    mode: absent Matrix:ObservationProjectorMode->Authoritative; typo->Shadow(not Authoritative, same writes)+deferred-startup-warning. Off->zero cells+no error.
  live-eval [gRPC :5001 · MultiCollectorEventConsumerWorker · SeriesCollected/CollectionFailed]
    does: updates ObservationCache; EvaluateAllEnabledAsync; persists ThresholdEvent on crossing.
    does NOT: CellProjector; matrix_cells; CellProjectedEvent.
    on-miss: exponential-backoff retry 5s->300s.
  ObservationEventSubscriber [UNWIRED / dead for cell-writes]: not registered in AddWorkers. MatrixCellPersistenceWorker subscribes but RECEIVES NOTHING at runtime. formula(CellProjector.cs)=DEAD CODE. never describe as live producer. Sentinel-gRPC-bridge: DELETED(MatrixCellSentinelWorker removed WS3-A3); README mermaid(SMW node,MatrixCellUpdate edge)=STALE.

PROCESSING MODEL (projector cycle, in order):
  read descending(freshest-first; ascending->freezes matrix on stale rows) -> drop null signal_identity_id -> build ONE per-cycle signal->pattern map(GetAllAsync; dup signal_identity_id->FIRST ENABLED wins) -> per group: unknown_signal if no pattern; pattern_disabled if Enabled=false(DISTINCT counters; neither projected) -> news/hard-data fork -> Project -> upsert.
  news-path: magnitude=K*tanh(S/K); S=sum(v_i*exp(-ln2*age_i/H)) 24h-half-life decay; freshness=1.0(no floor; perishable).
  hard-data-path: magnitude=pattern.SignalExpression evaluated vs live ObservationCache (domain-tuned normalization, e.g. unemployment=-(UNRATE-4)*2; clamped +/-3) — NOT the raw value_numeric mean (huge FRED levels saturated every cell to +/-3). Welford mean is the FALLBACK only (eval fail->WARN+signal_expression_eval_failed_total counter, or Core unit-tested directly). freshness still=step-decay(10% floor) keyed to latest obs (unchanged). SignalExpression NEVER applied to news.
  NOT the discriminator: source_collector='sentinel' — `:sig:` infix in source_id is the real discriminator.

PATTERN AUTO-DISABLE (load-time, not per-cycle):
  JSON Enabled:false->dropped at LoadAllAsync BEFORE SecMaster check. JSON-enabled: AllReferencedMnemonics->ResolveBatch gRPC -> any series not Found/PrimarySource->pattern.Enabled=false+ERROR. PatternValidation:Strict=true->accumulated misses throw at boot(fail-fast), not soft-disable.
  SecMasterFrequencyResolver: no frequency param; NO prefix-fallback; NO exception-swallowing; outage->exception(strict) or mass-auto-disable(soft).

DISTINCTIONS:
  projector-cells(wired,synthetic key,source_trust formula) ≠ live-eval-cells(UNWIRED,real pattern_id,pattern.Weight formula).
  Confidence-STATIC(PatternConfiguration.Confidence,projector,wired) ≠ Confidence-DYNAMIC(dead live-eval path only). conflating=active error risk.
  unknown_signal(no pattern maps signal) ≠ pattern_disabled(mapped;Enabled=false) — DISTINCT counters.
  auto-disable(load-time,SecMaster-miss) ≠ JSON-Enabled:false(dropped at load, not SecMaster-checked) ≠ pattern_disabled-skip(cycle-time).
  SecMasterFrequencyResolver(load-time, no frequency, no fallback) ≠ DataWarmupService(collector-routing,frequency:"any",HAS prefix-fallback).

CROSS-SERVICE:
  IN: collector gRPC :5001 -> live eval(threshold events only; not matrix_cells).
  OUT->SecMaster gRPC ResolveBatch: SecMasterFrequencyResolver(load-time) + DataWarmupService(collector-routing).
  FEEDS ThresholdEngineMcp+AlertService via gRPC ObservationEventStream. REST :8080(macro-observation mapping,dashboards).
  macro_observations: written by SentinelCollector/MacroSubstrate; ThresholdEngine reads only. SectorPhaseViewRefreshWorker: 7-day cadence; DB outage->silent stale dashboard.

GOTCHAS:
  ✗ live-FRED-gRPC-writes-matrix_cells(ThresholdEvent only) ✗ ObservationEventSubscriber/CellProjector-as-live-producer
  ✗ merged-cell-formula ✗ trust-Confidence-docstring("informational only"=false) ✗ conflate-static/dynamic-Confidence
  ✗ prefix-fallback-to-SecMasterFrequencyResolver ✗ typo-config->Authoritative(->Shadow) ✗ Sentinel-bridge-as-live(deleted)
  ✗ ascending-projector-read ✗ unknown_signal/pattern_disabled-shared-counter ✗ freshness-floor-on-news-path
  ✗ ResolveBatch-per-cycle(load-time only) ✗ README-mermaid-SMW/MatrixCellSentinel-as-current(stale)

DECISIONS:
  D-1 news-contract-exclusion: INTENT `:sig:` news rows carry tilt*confidence |v|~<=1; a raw-value leak entering the decay sum slams the cell to a fake saturated burst masquerading as real coverage / PRECOND |v|<=1.5 to enter the news sum, violating rows EXCLUDED (WARN+counted); a group whose :sig: rows are ALL excluded skips — NEVER falls back to its raw legacy rows / GUARD ObservationCellProjector.BuildCellsAsync @ src/Workers/ObservationCellProjector.cs:550 / TEST ObservationCellProjectorTests.BuildCellsAsync_SigRow_ContractBreach_ExcludedFromCell_StaysBounded
  D-2 mixed-group-legacy-drop: INTENT legacy non-:sig: sentinel rows carry RAW economic values (cpi 332, jobless 1.82e6) that must never blend into the news tilt / PRECOND in a group with >=1 :sig: row ONLY :sig: rows feed the magnitude; the drop is counted+logged / GUARD ObservationCellProjector.BuildCellsAsync @ src/Workers/ObservationCellProjector.cs:576 / TEST ObservationCellProjectorTests.BuildCellsAsync_MixedGroup_RawValuedNonSigRow_ExcludedFromDecaySum
  D-3 cell-clamp-pm3: INTENT matrix consumers assume cell in [-3,+3] (legacy ref D12) regardless of upstream magnitude/trust blowups / PRECOND every projected cell hard-clamped in the Core — the ONLY wired write-path enforcement (persistence re-clamp is on the dead path); AnyClamped flags saturation / GUARD ObservationCellProjectorCore.Project @ src/Services/ObservationCellProjectorCore.cs:165 / TEST ObservationCellProjectorCoreTests.Project_ClampsCellToPlusMinusThree
  D-4 all-11-sectors-or-skip: INTENT explicit zero != omission (legacy ref D5): a partial or silently-zero-defaulted cell-set corrupts the sector vector / PRECOND SectorWeights missing any of the 11 -> Core THROWS, worker skips the WHOLE group (0 cells, WARN+counted, siblings unaffected) / GUARD ObservationCellProjectorCore.Project @ src/Services/ObservationCellProjectorCore.cs:151 / TEST ObservationCellProjectorTests.BuildCellsAsync_InvalidSectorWeights_SkipsWholeGroup_SiblingsUnaffected
  D-5 mode-off-only-suppressor: INTENT Off is the projector kill-switch; Shadow != dry-run — it writes the SAME cells as Authoritative (flag-reversibility, telemetry-only tag) / PRECOND write suppression ONLY via the Off short-circuit before any scope/read; gating writes on Shadow silently zeroes the matrix feed / GUARD ObservationCellProjector.ExecuteAsync @ src/Workers/ObservationCellProjector.cs:253 / TEST ObservationCellProjectorTests.ExecuteAsync_Off_WritesNoCells_EvenWithObservationsPresent
  D-6 runtime-log-level: INTENT a WARN-quiet-by-design service must be raisable to Information/Debug AT RUNTIME (on-demand deep debugging) with NO restart or redeploy, while steady state stays Warning (prod-quiet). A LoggingLevelSwitch is the SOLE level authority; the operator edits Serilog:MinimumLevel:Default in the RUNNING container and reloadOnChange picks it up live. Serilog.Settings.Configuration 9.0.0 (bumped from 8.0.4 with Serilog.Extensions.Hosting so this contract is truthful) does NOT re-apply MinimumLevel on IConfiguration reload, so an explicit ChangeToken re-parse is what makes the edit take effect; the MEL floor sits at Debug so the bridge does not pre-filter below the switch. The service's file-config watchers (pattern/burst-window/sector-threshold) are SEPARATE files and independent of this appsettings switch. The raised level lives ONLY in the container's writable layer — revert by editing the key back to Warning (live) OR RECREATE (not restart) the container to reinstate the image's baked-in default; a restart PRESERVES the writable layer and does NOT revert. No compose/mount change (baked-in appsettings is the assumed, acceptable case). / PRECOND operator runs `nerdctl exec threshold-engine` to edit /app/appsettings.json Serilog:MinimumLevel:Default — no env override for that key exists, so the JSON is authoritative and reloadable. / GUARD RuntimeLogLevel.BuildLogger (ControlledBy = sole authority) + RuntimeLogLevel.CreateSwitch (ChangeToken re-parse on reload) + RuntimeLogLevel.AddRuntimeControlledSerilog (MEL Debug floor) @ src/Telemetry/RuntimeLogLevel.cs / TEST RuntimeLogLevelTests.Runtime_appsettings_edit_raises_then_lowers_serilog_level

SEE: README.md §Reference (REST/gRPC/health endpoint catalog; config keys; mermaid SMW+MatrixCellSentinel=stale) · Events/src/Events/Protos/observation_events.proto (ObservationEventStream gRPC contract — both consumed from collectors and exposed to downstream) · Events/src/Events/Protos/secmaster.proto (SecMasterResolver.ResolveBatch — consumed at load time) · ThresholdEngine/src/Workers/ObservationCellProjector.cs · src/Workers/ObservationEventSubscriber.cs (UNWIRED)
