# OfrCollector — architecture [agent read-first]

PURPOSE: collect OFR public data (FSI · STFM · HFM) → TimescaleDB → gRPC event-stream + macro_observations dual-write (STFM/HFM per is_macro flag; FSI composite+4 patterned subindices, raw, into the matrix). ¬identity/classification(SecMaster owns) ¬thresholding(ThresholdEngine owns) ¬price-store.

DATA MODEL + INVARIANTS (schema does NOT enforce):
  FSI ≠ STFM/HFM shape: FSI = ONE wide row (composite + 8 contribution cols), per-date. STFM/HFM = series×observation (per-mnemonic). FSI macro-substrate fan-out = composite + 4 PATTERNED subindices ONLY (OFR_FSI[_CREDIT|_FUNDING|_VOLATILITY|_EM] → macro_observations, raw values, source_collector="ofr", source_id=<FsiEventSeries.SeriesId>); the other 4 contributions (_EQUITY|_SAFE_ASSETS|_US|_AE) have NO pattern/identity → NOT written (matrix noise). FSI ⊥ auto-SecMaster-registration (FSI ids registrable ONLY via admin one-shot /api/admin/fsi/register, assetClass Index).
  INV register⊥collect: SecMaster register fires ONLY on series-ADD (AddStfm/HfmSeriesAsync, fire-and-forget `_ = Register…`, guarded _secMasterClient≠null). Collection cycles ¬register. Seeder-upserted rows ¬register (seed path skips it). Backfill escape hatch: admin re-register endpoints (POST /api/admin/{stfm|hfm}/series/{mnemonic}/register, /api/admin/fsi/register) — awaited ¬f-a-f, idempotent (SecMaster already-exists), 503 when SECMASTER_GRPC_ENDPOINT unset.
  INV register-case: SecMaster uppercases Symbol + resolves case-SENSITIVELY → mixed-case mnemonic (FNYR-SOFR_99Pctl-A) ships metadata["alias"]=verbatim mnemonic → SecMaster series_id alias → ThresholdEngine pattern symbols resolve verbatim.
  INV register⊥tag: gRPC register (RegisterSeries, asset-class str) ⊥ REST signal-identity tag (by-alias, kebab id). Different transports, different SecMaster surfaces, different consumers.
  INV is_macro⊥signal_identity: is_macro = STFM/HFM dual-write routing predicate; SignalIdentityId = matrix-join handle. Both default false/null → migrations additive. A macro row writes substrate even with null signal_identity (counted, ¬blocked). FSI ¬gated by is_macro (no per-series flag — it's a wide row): the 5 patterned cells are selected by FsiEventSeries.SignalIdentityId≠null, always written.
  INV dual-write non-fatal: legacy ofr_*_observations row persists FIRST; substrate failure logged+counted ¬rethrown (except host-shutdown OCE → propagates). null value SKIPPED (substrate CHECK = exactly-one numeric|qualitative).
  INV seed: OfrSeriesDbSeeder upserts MISSING mnemonics from embedded SeriesSeed/*.json at startup; operator rows/edits survive. BOTH tables empty post-seed → fail-fast (¬start). No on-disk config dir, no SeriesConfig__Directory env.
  INV macro-vocab narrow: most mnemonics ¬in pure-macro vocab → typical tag outcome = NoMatch (¬error); resolver NotFound ⇒ "not in vocabulary" ¬transport-failure.

PATHS (distinct code — do not conflate):
  ObservationEventStream [gRPC :5001 · ThresholdEngine]
    do: stream FSI+STFM+HFM since `start_from`; Subscribe replays then polls @100ms (unbounded); FSI fans 1→N (composite + each NON-NULL contribution: OFR_FSI[_CREDIT|_FUNDING|_VOLATILITY|_EQUITY|_SAFE_ASSETS|_US|_AE|_EM]).
    does NOT: ¬filter-by-series ¬register ¬resolve ¬write. GetEventsBetween IGNORES `to` (delegates GetEventsSince). GetLatestEventTime = now() placeholder ¬real max(event_time).
    on-miss: empty stream; client disconnect (OCE|RpcException.Cancelled) = Info ¬error.
  register [gRPC :5001 · SecMaster; fire-and-forget, ADD-only]
    do: RegisterSeries(collectorSeriesId=STFM-/HFM-<mnemonic>, assetClass STFM→"Rate" HFM→"NBFI", freq default Daily/Quarterly). FSI never registered.
    does NOT: ¬await ¬block-add ¬on-collect ¬on-seed. on-miss: exception swallowed → WARN + SecMasterRegistrationFailures{type}; series add succeeds anyway.
  signal-identity tag [HTTP→SecMaster REST · OfrSeriesSignalIdentityTagBackgroundService, periodic]
    do: GET /api/signal-identities/by-alias?alias=<mnemonic> → write SignalIdentityId+ResolvedAt back; idempotent (write iff id differs); shared HFM+STFM batch cap.
    does NOT: ¬gRPC ¬register ¬NER. NOT-WIRED when SECMASTER_REST_ENDPOINT unset (worker+resolver not registered → no-op).
    on-miss: 404→NotFound→no_match (Info); transport/parse→Failed (WARN, returns Failed ¬throw); non-transport exc caught by ProcessOneAsync catch-all → LogWarning + returns error count (¬throw). Only OperationCanceledException propagates to outer cycle.
  REST read + admin [HTTP :8080]
    do: /api/{fsi,stfm,hfm}/* reads, /api/search, /api/admin/* collect|backfill|series-mgmt. Admin triggers = fire-and-forget (ThreadPool.QueueUserWorkItem) → 200 immediately; outcome in logs.
    on-miss: /health = full (database check, all checks); /health/ready = tag-filtered (ready+db tags); /health/live = echo only (Predicate=_=>false, no dep probe). /api/health = separate REST echo endpoint (ApiEndpoints.cs, ¬middleware).

PROCESSING MODEL:
  3 workers (Fsi/Stfm/HfmCollectionWorker) scheduled per-dataset → CollectionService → repo upsert (idempotent) + optional macro dual-write (is_macro). 3 OFR HTTP clients, distinct base URLs (FSI=CSV www., STFM=v1/, HFM=hf/v1/), each Polly retry 3× exp + breaker (5 consecutive failures → open; stays open 60s durationOfBreak, no sliding-window; ONE keyed-singleton breaker instance per client shared across requests — per-request construction inside the policy selector never accumulates, never opens; retry wraps breaker so retried attempts count individually) + 30s timeout.
  Dual-write provenance/idempotency key = (source_collector="ofr", source_id=<mnemonic>, observation_time). Re-run ¬double-writes.

DISTINCTIONS:
  FSI(wide-row, CSV, ¬registered, fan-out events + macro_observations for composite+4 patterned subindices) ≠ STFM/HFM(series×obs, JSON, macro-eligible per is_macro, registered).
  FSI macro-write ≠ STFM/HFM macro-write: all 3 go through MacroObservationDualWriter (shared non-fatal/null-skip/idempotency/counter contract), but FSI routes via FsiEventSeries.SignalIdentityId≠null (catalog-driven, 5 cells, raw values; the wide ofr_fsi row is the "legacy" sink, already upserted first) vs STFM/HFM route via per-series is_macro flag (legacy ofr_*_observations row first).
  HFM = Hedge Fund Monitor (asset-class NBFI, ~Quarterly) ≠ STFM = Short-term Funding Monitor (asset-class Rate, ~Daily). Both ≠ FSI (Financial Stress Index aggregate).
  register(gRPC, on-add, asset-class) ≠ tag(REST, periodic, signal-identity). One series can be registered but never tagged (¬in macro vocab).
  is_macro=true → ADDITIVE substrate write (legacy table unchanged) ≠ migration/cutover.
  SECMASTER_GRPC_ENDPOINT(register) ⊥ SECMASTER_REST_ENDPOINT(tag): either unset → that path silently disabled.

CROSS-SERVICE: SecMaster ← register(gRPC :5001, f-a-f, add-only; STFM/HFM only, FSI excluded from gRPC-register) + tag(REST /api/signal-identities/by-alias, periodic; OPTIONAL). ThresholdEngine → ObservationEventStream(gRPC :5001) + macro_observations (WS3 ObservationCellProjector groups by (signal_identity_id, source_collector) → FSI patterns OFR_FSI[_CREDIT|_FUNDING|_VOLATILITY|_EM] light up). MacroSubstrate ← dual-write(in-proc; STFM/HFM is_macro=true; FSI composite+4 patterned subindices, raw). OUT: OFR HTTP (3 endpoints). FEEDS: ThresholdEngine event stream; macro_observations.

GOTCHAS:
  ✗ assume-register-on-collect (add-only) ✗ assume-FSI-gRPC-registered (excluded; FSI registers ONLY via admin one-shot) ✗ assume-ALL-FSI-cols→matrix (only composite+4 patterned subindices write; _EQUITY/_SAFE_ASSETS/_US/_AE excluded — no pattern) ✗ seeded-rows-auto-registered (only Add path)
  ✗ GetEventsBetween-honors-`to` (ignored) ✗ GetLatestEventTime-is-real (placeholder now())
  ✗ dual-write-failure-aborts-collection (non-fatal, legacy already saved) ✗ macro-write-needs-signal-identity (null OK, counted)
  ✗ conflate is_macro with SignalIdentityId ✗ conflate gRPC-register with REST-tag ✗ expect on-disk series config (embedded JSON only)
  ✗ both-tables-empty-and-run (fail-fast)

SEE: README.md §API/Config/Ports · Events/src/Events/Protos/observation_events.proto (ObservationEventStream) + secmaster.proto (RegisterSeries) · MacroObservationDualWriter.cs (idempotency/null-skip/non-fatal) · FsiEventSeries.cs (FSI fan-out catalog — single source for stream, registration AND matrix SignalIdentityId) · FsiCollectionService.WriteSubstrateAsync (FSI→macro_observations fan-out, both collect+backfill) · SeriesManagementService.cs (add-only register + Build*Registration specs + admin re-register) · OfrSeriesSignalIdentityResolver.cs (by-alias) · DependencyInjection.cs:82-127 (conditional wiring)
