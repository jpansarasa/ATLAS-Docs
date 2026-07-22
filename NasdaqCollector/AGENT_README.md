# NasdaqCollector — architecture [agent read-first]

PURPOSE: daily time-series collector for Nasdaq Data Link (ex-Quandl) -> TimescaleDB; serves ObservationEventStream gRPC. Not a live-price-feed, not tick-data, not a resolver. DISABLED in prod (NDL WAF blocks datacenter IPs; compose block commented; no prod gRPC consumer wired).

DATA MODEL + INVARIANTS:
  series_id = "{DatabaseCode}/{DatasetCode}" (derived, unique). obs unique(series_id,date). Indep of Id(int PK): FK obs.series_config_id->series.Id != string series_id col.
  INV events-are-synthetic: NO event table. Events SYNTHESIZED on-read from nasdaq_observations rows (EventRepository.CreateEvent). EventId=fresh Ulid PER READ -> not stable, not an idempotent-key; same row re-streamed = new EventId. OccurredAt=CollectedAt always.
  INV stream-by-collected_at: all SubscribeToEvents/GetEventsSince/Between filter+order by CollectedAt, not observation Date. SubscribeToEvents StartFrom default=now -> no history. 1 DataPoint per Event (not batched). ApiVersion="nasdaq-v3" hardcoded.
  INV eventTypes-ignored: SubscriptionRequest.EventTypes accepted on-wire but EventRepository never filters on it (dead param). SeriesIds filter IS honored.
  INV counts-by-series: GetHealth.EventsByType is GROUPED BY SeriesId, not event-type (single SeriesCollected type exists). map key = series_id.
  INV incremental: CollectSeries start = latestObs.Date+1 ?? now-2yr backfill. empty NDL response -> no UpdateLastCollected. 100ms delay between series.
  INV secmaster-register⊥collection: register fire-&-forget on series-ADD only (admin POST), not per-obs, not on-toggle; only if SECMASTER_GRPC_ENDPOINT set. failure=WARN+swallow (incl Success=false rejects + null retry-exhaustion — never a false "Registered" info), does not block add. instrumentType="Economic", symbol=DatasetCode, assetClass=Category (default "General"). ⚠ SecMaster D-4 register guard = equity-shaped ALLOWLIST {Equity,ETF,Index,Currency,Crypto,Commodity} for untrusted collectors — "General" is not in it, so EVERY Nasdaq registration is rejected today; a Category->allowlisted-assetClass mapping is needed before prod re-enable (unresolved, deliberately not shipped with the truthful-logging fix).
  INV holiday-skip computational: MarketCalendarAdapter (CalendarService.Core, no service call); weekend OR IsMarketClosed -> whole cycle skipped, not per-series.

PATHS (distinct code — do not conflate):
  ObservationEventStream [gRPC :5009(prod)/:5005(dev) · downstream(none-wired); proto ATLAS.Events.Grpc]
    do: stream synthesized Events (Since/Between/Subscribe-poll@1s); GetLatestEventTime; GetHealth(totals+per-series counts).
    does NOT: persist-events; eventType-filter; register; resolve; price.
    on-miss: empty range->0 events (no error); GetLatestEventTime(no rows)->now(); GetHealth-fail->Healthy=false.
  admin series CRUD [HTTP :8080(prod)/:5004(dev) · operator]
    do: POST add (->secmaster register f-&-f) | PUT toggle IsActive | DELETE | GET all.
    does NOT: backfill-on-add; trigger-collection (next 6h tick picks up active).
    on-miss: add-dup->409 Conflict; toggle/delete-absent->404.
  read+discover [HTTP :8080 · SecMaster-gateway/operator]
    do: /api/search (LOCAL catalog) | /api/discover (PROXIES upstream NDL /datasets.json) | series/obs/latest reads.
    on-miss: empty q->400; series-absent->404; latest-no-obs->404.

PROCESSING MODEL:
  CollectionWorker(BackgroundService): run-once-on-start -> PeriodicTimer 6h. cycle = holiday-gate -> CollectAll(active series, sequential, 100ms gap) -> per-series GetObservations(NDL HTTP, retry HttpRequestException x MaxRetries=3 @2s) -> UpsertObservations + UpdateLastCollected. per-series failure isolated (CollectionResult{Success=false}, cycle continues).

DISTINCTIONS:
  Event.OccurredAt/CollectedAt (collected_at) ≠ DataPoint.Date (observation date) — stream filters on FORMER.
  series_config_id (FK->Id int) ≠ series_id (string "{db}/{ds}" col).
  /api/search (local DB) ≠ /api/discover (live upstream NDL proxy).
  IsRevised/PreviousValue = revision tracking on obs row, surfaced into DataPoint ≠ separate event type.
  Worker.cs = unused legacy stub ≠ Workers/CollectionWorker.cs (the real hosted worker).

CROSS-SERVICE: SecMaster <- register(gRPC f-a-f, add-only, OPTIONAL; D-4 allowlist rejects "General" -> WARN, see INV secmaster-register). CalendarService.Core -> holiday-check (in-proc lib, not HTTP). OUT: Nasdaq Data Link HTTP. FEEDS: none wired in prod (no active gRPC consumer).

GOTCHAS:
  ✗ treat-EventId-as-stable/dedup-key (regenerated per read) ✗ expect-event-persistence (synthesized) ✗ filter-stream-by-EventTypes (dead param) ✗ read GetHealth.EventsByType as event-type (it's series_id) ✗ Subscribe-expects-history (StartFrom=now default) ✗ assume-prod-running (disabled, WAF) ✗ assume register-lands (SecMaster D-4 allowlist rejects ALL "General"-classed registrations — rejection is WARN-logged, not silent; map Category to an allowlisted assetClass before re-enable) ✗ trust compose port-map (5008->5004/5009->5005 != image URLs 8080/5009 — unreconciled).

DECISIONS:
  D-1 runtime-log-level: INTENT a WARN-quiet-by-design collector must be raisable to Information/Debug AT RUNTIME (on-demand deep debugging) with NO restart or redeploy, while steady state stays Warning (prod-quiet). A LoggingLevelSwitch is the SOLE level authority; the operator edits Serilog:MinimumLevel:Default in the RUNNING container and reloadOnChange picks it up live. Serilog.Settings.Configuration 9.0.0 does NOT re-apply MinimumLevel on IConfiguration reload, so an explicit ChangeToken re-parse is what makes the edit take effect; the MEL floor sits at Debug so the bridge does not pre-filter below the switch. The raised level lives ONLY in the container's writable layer — revert by editing the key back to Warning (live) OR RECREATE (not restart) the container to reinstate the image's baked-in default; a restart PRESERVES the writable layer and does NOT revert. No compose/mount change (baked-in appsettings is the assumed, acceptable case). / PRECOND operator runs `nerdctl exec nasdaq-collector` to edit /app/appsettings.json Serilog:MinimumLevel:Default — no env override for that key exists, so the JSON is authoritative and reloadable (applies when the service is re-enabled; disabled in prod per the WAF note above). / GUARD RuntimeLogLevel.BuildLogger (ControlledBy = sole authority) + RuntimeLogLevel.CreateSwitch (ChangeToken re-parse on reload) + RuntimeLogLevel.AddRuntimeControlledSerilog (MEL Debug floor) @ src/Telemetry/RuntimeLogLevel.cs / TEST RuntimeLogLevelTests.Runtime_appsettings_edit_raises_then_lowers_serilog_level

SEE: README.md (endpoints, ports, deploy-status) · Events/src/Events/Protos/observation_events.proto (ObservationEventStream contract) · EventRepository.cs (synth + by-series counts + eventTypes-ignore) · SeriesManagementService.cs:90-127 (secmaster register + truthful result handling) · NasdaqCollectionService.cs:50-67 (incremental window) · Workers/CollectionWorker.cs (6h cycle + holiday gate).
