# FredCollector — architecture [agent read-first]

PURPOSE: owns FRED series catalog(series_configs)+raw obs(fred_observations point-in-time). Not instrument-master, not sector-owner, not macro-substrate; dual-writes INTO macro_observations(MacroSubstrate).

DATA MODEL + INVARIANTS:
  INV catalog⊥instrument: SeriesId=FRED mnemonic(ToUpperInvariant,<=50 chars), not an instr-id. SignalIdentityId(kebab,<=64)=only SecMaster xref; may be null.
  INV sector-xor-signal: ck_series_config_sector_xor_signal -> AtlasSectorCode IS NULL OR SignalIdentityId IS NULL. both-NULL=legal(newborn pre-tagger); both-set=rejected. Never read as XOR.
  INV AsOf: live+ordinary-backfill->AsOf=date(CollectedAt); ALFRED-vintage->AsOf=realtime_start. Never conflate.
  INV LastCollectedAt: BackfillService advances it; AlfredBackfillService deliberately does NOT touch it (scheduler keys start-window off it; vintage backfill must not move it; also avoids ThresholdEngine re-processing decades of historical revisions).
  INV dual-write-gate: computed each cycle from SignalIdentityId vs SecMaster per-signal `matrix_benchmark` flag; not a per-series is_macro col. Flag is per-signal, orthogonal to Category (cuts across macro/rate/commodity/fx/credit).
  INV ObservationChannel: unbounded Channel.CreateUnbounded; no in-service reader; not live fanout -> memory-growth hazard on long-running deploy.
  INV dot-sentinel: "." value->Value=null row INSERTED fred_observations; macro dual-write skips null; insert itself NOT skipped.

PATHS (distinct code — do not conflate):
  cron [Quartz/DisallowConcurrentExecution per-series]:
    does: incremental fetch -> INSERT DO NOTHING fred_observations -> publish EventPublisher(DB events) -> conditional macro dual-write.
    does NOT: SecMaster-register; re-tag.
    on-miss: null/empty obs from FRED -> still inserts null row if "." value; macro dual-write skips null value.
  AddSeries [POST /api/admin/series]:
    does: fetch FRED metadata -> derive Frequency->cron -> persist -> schedule -> fire-and-forget BackfillService(24mo,advances LastCollectedAt) -> fire-and-forget RegisterWithSecMasterAsync(gRPC; only if _secMasterClient!=null).
    does NOT: set sector/signal tags.
  gRPC-EventStream [:5001 · ThresholdEngine]: polls DB events table; does not serve fred_observations directly; not an in-process stream.
  taggers [periodic background]: resolve NAICS->AtlasSectorCode + mnemonic->SignalIdentityId -> write-back series_configs.
    on-miss: null/NotFound/transport-fail=tag left unchanged.

PROCESSING macro_observations dual-write gate ("SecMaster owns the matrix-benchmark id-set; classify signal-identity not series; fail closed, serve last-known-good"):
  1. isMatrixBenchmark=SecMasterMatrixBenchmarkClassifier.IsMatrixBenchmarkAsync(config.SignalIdentityId) once per series. null/unresolved SignalIdentityId->not a matrix benchmark.
  2. classifier: 15-min-TTL cache of /api/signal-identities/matrix-benchmarks (per-signal `matrix_benchmark` flag; cuts across categories). transport-fail+populated-cache->last-known-good(no flap); cold-start-fail OR fetch-returns-zero->empty-set fail-closed(no dual-write).
  3. per obs: always INSERT fred_observations(incl null). isMatrixBenchmark AND value!=null->MacroObservationWriter.WriteAsync(upsert last-write-wins; idempotent on triple provenance=source_collector+source_id+obs_time). null->skip substrate only.

DISTINCTIONS:
  fred_observations(verbatim level ALL series, incl null rows) ≠ macro_observations(transformed, matrix-benchmark-flagged only; additive not replacing).
  BackfillService(AddSeries+InitialWorker+manual-admin)=publishes channel+gRPC events+advances LastCollectedAt ≠ AlfredBackfillService=repo-write only, does not publish, does not touch LastCollectedAt.
  EventPublisher(->DB events table) ≠ ObservationChannel(in-mem, no reader).
  FredCollector does NOT set is_primary: only calls RegisterSeries; SecMaster decides primacy.
  SECMASTER_GRPC_ENDPOINT(registration) ≠ SECMASTER_REST_ENDPOINT(taggers+dual-write-gate): each independently optional, different silent semantics.
  GRPC unset->_secMasterClient=null->register silently skipped, no WARN. REST unset->NullMatrixBenchmarkClassifier wired->no dual-write.
  matrix-benchmark-flagged(SignalIdentityId has matrix_benchmark=true) ≠ non-benchmark(SignalIdentityId flag false/absent) ≠ sector-rolled(AtlasSectorCode set,SignalIdentityId=null) ≠ untagged(both null). matrix_benchmark is per-signal, indep of Category — a benchmark signal may be any category.

CROSS-SERVICE:
  OUT gRPC RegisterSeriesAsync->SecMaster on AddSeries(f-a-f; GRPC unset->silently skipped, no WARN; RPC throws->WARN, does not block).
  OUT sync(cached): REST sector/signal resolve+matrix-benchmark classify(per-signal `matrix_benchmark` flag)->SecMaster.
  OUT dual-write->macro_observations(ConnectionStrings:AtlasData->fallback AtlasDb).
  IN: upstream FRED HTTP (Polly: WaitAndRetry+CircuitBreaker on transient/429).
  FEEDS: ThresholdEngine<-gRPC ObservationEventStream :5001. MCP sidecar: fredcollector-mcp=separate container; ONLY host-accessible surface.

GOTCHAS:
  ✗ per-series is_macro col ✗ read constraint as XOR ✗ SeriesId=instrument-id
  ✗ SecMaster-register=synchronous/authoritative ✗ expect WARN when GRPC unset
  ✗ EventPublisher=direct gRPC emitter ✗ ObservationChannel=live fanout
  ✗ ALFRED backfill=advances LastCollectedAt ✗ ordinary backfill=true point-in-time AsOf
  ✗ env-vars from appsettings.json(runtime=env only)

DECISIONS:
  D-1 fred-frequency-normalize-at-source: INTENT FRED metadata returns *qualified* long-form frequency strings ("Weekly, Ending Saturday", "Daily, 7-Day", "Daily, Close") — normalise them to the right CollectionFrequency at the single derivation source, else the wrong value propagates to the revision-buffer window (DataCollectionService), the SecMaster registration frequency, and any future frequency-derived cron (GIGO clean-at-source; an exact-match switch silently defaulted every qualified string to Monthly, mislabelling all daily/weekly series). Extension (same defect class, F5): vocabulary with no exact enum member maps least-wrong (Semiannual/SA->Quarterly over-poll, 5-Year->Annual, Not Applicable/NA->Monthly, BW->Weekly); a genuinely UNKNOWN string still defaults Monthly but LOUDLY — Warning with raw string + series id, never silent. / PRECOND AddSeries or MCP add_series onboarding a series from FRED / GUARD SeriesManagementService.ParseFrequency @ src/Services/SeriesManagementService.cs:211 / TEST SeriesFrequencyDerivationTests.AddSeries_derives_correct_frequency_from_fred_metadata + SeriesFrequencyDerivationTests.AddSeries_unknown_frequency_defaults_monthly_and_warns
  note: Frequency does NOT drive the steady-state Quartz cron — DataCollectionScheduler schedules off the stored CronExpression column (a flat `0 0 18 * * ?` for ~76/80 rows), so Frequency's live effect is the revision/fallback window + the SecMaster/MCP frequency label, not cadence.
  D-2 runtime-log-level: INTENT a WARN-quiet-by-design collector must be raisable to Information/Debug AT RUNTIME (on-demand deep debugging) with NO restart or redeploy, while steady state stays Warning (prod-quiet). A LoggingLevelSwitch is the SOLE level authority; the operator edits Serilog:MinimumLevel:Default in the RUNNING container and reloadOnChange picks it up live. Serilog.Settings.Configuration 8.0.4 does NOT re-apply MinimumLevel on IConfiguration reload, so an explicit ChangeToken re-parse is what makes the edit take effect; the MEL floor sits at Debug so the bridge does not pre-filter below the switch. The raised level lives ONLY in the container's writable layer — revert by editing the key back to Warning (live) OR RECREATE (not restart) the container to reinstate the image's baked-in default; a restart PRESERVES the writable layer and does NOT revert. No compose/mount change (baked-in appsettings is the assumed, acceptable case). / PRECOND operator runs `nerdctl exec fred-collector` to edit /app/appsettings.json Serilog:MinimumLevel:Default — no env override for that key exists, so the JSON is authoritative and reloadable. / GUARD RuntimeLogLevel.BuildLogger (ControlledBy = sole authority) + RuntimeLogLevel.CreateSwitch (ChangeToken re-parse on reload) + RuntimeLogLevel.AddRuntimeControlledSerilog (MEL Debug floor) @ src/Telemetry/RuntimeLogLevel.cs / TEST RuntimeLogLevelTests.Runtime_appsettings_edit_raises_then_lowers_serilog_level

SEE: README.md §Reference (exhaustive series list, endpoint catalog, config/env-var enumeration, Polly backoff numbers, Quartz job-store detail, admin-GET Swagger summary vs impl mismatch, MCP port mapping) · Events/src/Events/Protos/observation_events.proto (ObservationEventStream — EXPOSES gRPC :5001) · Events/src/Events/Protos/secmaster.proto (SecMasterRegistry.RegisterSeries — consumed f-a-f on AddSeries)
