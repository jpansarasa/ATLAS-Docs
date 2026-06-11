# OfrCollector — architecture [agent read-first]

PURPOSE: collect OFR public data (FSI · STFM · HFM) → TimescaleDB → gRPC event-stream + macro_observations dual-write. ¬identity/classification(SecMaster owns) ¬thresholding(ThresholdEngine owns) ¬price-store.

DATA MODEL + INVARIANTS (schema does NOT enforce):
  FSI ≠ STFM/HFM shape: FSI = ONE wide row (composite + 8 contribution cols), per-date. STFM/HFM = series×observation (per-mnemonic). FSI ⊥ macro-substrate ⊥ SecMaster-registration (excluded from both).
  INV register⊥collect: SecMaster register fires ONLY on series-ADD (AddStfm/HfmSeriesAsync, fire-and-forget `_ = Register…`, guarded _secMasterClient≠null). Collection cycles ¬register. Seeder-upserted rows ¬register (seed path skips it).
  INV register⊥tag: gRPC register (RegisterSeries, asset-class str) ⊥ REST signal-identity tag (by-alias, kebab id). Different transports, different SecMaster surfaces, different consumers.
  INV is_macro⊥signal_identity: is_macro = dual-write routing predicate; SignalIdentityId = matrix-join handle. Both default false/null → migrations additive. A macro row writes substrate even with null signal_identity (counted, ¬blocked).
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
  FSI(wide-row, CSV, ¬macro, ¬registered, fan-out events) ≠ STFM/HFM(series×obs, JSON, macro-eligible, registered).
  HFM = Hedge Fund Monitor (asset-class NBFI, ~Quarterly) ≠ STFM = Short-term Funding Monitor (asset-class Rate, ~Daily). Both ≠ FSI (Financial Stress Index aggregate).
  register(gRPC, on-add, asset-class) ≠ tag(REST, periodic, signal-identity). One series can be registered but never tagged (¬in macro vocab).
  is_macro=true → ADDITIVE substrate write (legacy table unchanged) ≠ migration/cutover.
  SECMASTER_GRPC_ENDPOINT(register) ⊥ SECMASTER_REST_ENDPOINT(tag): either unset → that path silently disabled.

CROSS-SERVICE: SecMaster ← register(gRPC :5001, f-a-f, add-only; STFM/HFM only, FSI excluded) + tag(REST /api/signal-identities/by-alias, periodic; OPTIONAL). ThresholdEngine → ObservationEventStream(gRPC :5001). MacroSubstrate ← dual-write(in-proc, STFM/HFM is_macro=true). OUT: OFR HTTP (3 endpoints). FEEDS: ThresholdEngine event stream; macro_observations.

GOTCHAS:
  ✗ assume-register-on-collect (add-only) ✗ assume-FSI-registered/macro (excluded) ✗ seeded-rows-auto-registered (only Add path)
  ✗ GetEventsBetween-honors-`to` (ignored) ✗ GetLatestEventTime-is-real (placeholder now())
  ✗ dual-write-failure-aborts-collection (non-fatal, legacy already saved) ✗ macro-write-needs-signal-identity (null OK, counted)
  ✗ conflate is_macro with SignalIdentityId ✗ conflate gRPC-register with REST-tag ✗ expect on-disk series config (embedded JSON only)
  ✗ both-tables-empty-and-run (fail-fast)

SEE: README.md §API/Config/Ports · Events/src/Events/Protos/observation_events.proto (ObservationEventStream) + secmaster.proto (RegisterSeries) · MacroObservationDualWriter.cs (idempotency/null-skip) · EventStreamService.cs:283-310 (FSI fan-out) · SeriesManagementService.cs:69,239,396-458 (add-only register) · OfrSeriesSignalIdentityResolver.cs (by-alias) · DependencyInjection.cs:82-127 (conditional wiring)
