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

DECISIONS: none recorded yet — accrete on touch (not audited for exception paths; see CLAUDE.md INTENT_FIDELITY MECHANICS).

SEE: README.md §Reference (exhaustive series list, endpoint catalog, config/env-var enumeration, Polly backoff numbers, Quartz job-store detail, admin-GET Swagger summary vs impl mismatch, MCP port mapping) · Events/src/Events/Protos/observation_events.proto (ObservationEventStream — EXPOSES gRPC :5001) · Events/src/Events/Protos/secmaster.proto (SecMasterRegistry.RegisterSeries — consumed f-a-f on AddSeries)
