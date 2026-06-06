# NasdaqCollector — architecture [agent read-first]

PURPOSE: daily time-series collector for Nasdaq Data Link (ex-Quandl) → TimescaleDB; serves ObservationEventStream gRPC. ¬live-price-feed ¬tick-data ¬resolver. DISABLED in prod (NDL WAF blocks datacenter IPs; compose block commented; no prod gRPC consumer wired).

DATA MODEL + INVARIANTS:
  series_id = "{DatabaseCode}/{DatasetCode}" (derived, unique). obs unique(series_id,date). ⊥ Id(int PK): FK obs.series_config_id→series.Id ≠ string series_id col.
  INV events-are-synthetic: NO event table. Events SYNTHESIZED on-read from nasdaq_observations rows (EventRepository.CreateEvent). EventId=fresh Ulid PER READ → ¬stable ¬idempotent-key; same row re-streamed = new EventId. OccurredAt=CollectedAt always.
  INV stream-by-collected_at: all SubscribeToEvents/GetEventsSince/Between filter+order by CollectedAt ¬ observation Date. SubscribeToEvents StartFrom default=now → ¬history. 1 DataPoint per Event (¬batched). ApiVersion="nasdaq-v3" hardcoded.
  INV eventTypes-ignored: SubscriptionRequest.EventTypes accepted on-wire but EventRepository ¬filters on it (dead param). SeriesIds filter IS honored.
  INV counts-by-series: GetHealth.EventsByType is GROUPED BY SeriesId ¬ event-type (single SeriesCollected type exists). map key = series_id.
  INV incremental: CollectSeries start = latestObs.Date+1 ?? now−2yr backfill. empty NDL response → ¬UpdateLastCollected. 100ms delay between series.
  INV secmaster-register⊥collection: register fire-&-forget on series-ADD only (admin POST), ¬per-obs ¬on-toggle; only if SECMASTER_GRPC_ENDPOINT set. failure=WARN+swallow ¬block add. instrumentType="Economic", symbol=DatasetCode, assetClass=Category. ⚠ SecMaster GUARD gates Economic to TrustedMacroCollectors — register may silently reject.
  INV holiday-skip computational: MarketCalendarAdapter (CalendarService.Core, no service call); weekend OR IsMarketClosed → whole cycle skipped, ¬per-series.

PATHS (distinct code — do not conflate):
  ObservationEventStream [gRPC :5009(prod)/:5005(dev) · downstream(none-wired); proto ATLAS.Events.Grpc]
    do: stream synthesized Events (Since/Between/Subscribe-poll@1s); GetLatestEventTime; GetHealth(totals+per-series counts).
    does NOT: ¬persist-events ¬eventType-filter ¬register ¬resolve ¬price.
    on-miss: empty range→0 events (¬error); GetLatestEventTime(no rows)→now(); GetHealth-fail→Healthy=false.
  admin series CRUD [HTTP :8080(prod)/:5004(dev) · operator]
    do: POST add (→secmaster register f-&-f) | PUT toggle IsActive | DELETE | GET all.
    does NOT: add ¬backfill-on-add ¬trigger-collection (next 6h tick picks up active).
    on-miss: add-dup→409 Conflict; toggle/delete-absent→404.
  read+discover [HTTP :8080 · SecMaster-gateway/operator]
    do: /api/search (LOCAL catalog) | /api/discover (PROXIES upstream NDL /datasets.json) | series/obs/latest reads.
    on-miss: empty q→400; series-absent→404; latest-no-obs→404.

PROCESSING MODEL:
  CollectionWorker(BackgroundService): run-once-on-start → PeriodicTimer 6h. cycle = holiday-gate → CollectAll(active series, sequential, 100ms gap) → per-series GetObservations(NDL HTTP, retry HttpRequestException ×MaxRetries=3 @2s) → UpsertObservations + UpdateLastCollected. per-series failure isolated (CollectionResult{Success=false}, cycle continues).

DISTINCTIONS:
  Event.OccurredAt/CollectedAt (collected_at) ≠ DataPoint.Date (observation date) — stream filters on FORMER.
  series_config_id (FK→Id int) ≠ series_id (string "{db}/{ds}" col).
  /api/search (local DB) ≠ /api/discover (live upstream NDL proxy).
  IsRevised/PreviousValue = revision tracking on obs row, surfaced into DataPoint ≠ separate event type.
  Worker.cs = unused legacy stub ≠ Workers/CollectionWorker.cs (the real hosted worker).

CROSS-SERVICE: SecMaster ← register(gRPC f-a-f, add-only, OPTIONAL; Economic GUARD may silently reject). CalendarService.Core → holiday-check (in-proc lib, ¬HTTP). OUT: Nasdaq Data Link HTTP. ¬active gRPC consumer wired in prod.

GOTCHAS:
  ✗ treat-EventId-as-stable/dedup-key (regenerated per read) ✗ expect-event-persistence (synthesized) ✗ filter-stream-by-EventTypes (dead param) ✗ read GetHealth.EventsByType as event-type (it's series_id) ✗ Subscribe-expects-history (StartFrom=now default) ✗ assume-prod-running (disabled, WAF) ✗ assume register-always-lands (SecMaster Economic-collector GUARD) ✗ trust compose port-map (5008→5004/5009→5005 ≠ image URLs 8080/5009 — unreconciled).

SEE: README.md (endpoints, ports, deploy-status) · Events/src/Events/Protos/observation_events.proto (ObservationEventStream contract) · EventRepository.cs (synth + by-series counts + eventTypes-ignore) · SeriesManagementService.cs:90-116 (secmaster register) · NasdaqCollectionService.cs:50-67 (incremental window) · Workers/CollectionWorker.cs (6h cycle + holiday gate).
