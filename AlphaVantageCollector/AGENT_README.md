# AlphaVantageCollector — architecture [agent read-first]

PURPOSE: market-data collector (commodity/economic/equity-OHLCV/forex/crypto) -> AlphaVantage HTTP -> TimescaleDB; emits ObservationEventStream gRPC. Not identity/classification (that's SecMaster), not a price-API for callers (poll-pull via REST/repo, does not push values).

DATA MODEL + INVARIANTS:
  AlphaVantageSeries(PK series_id) 1-N obs · scheduling owned local (Priority,Interval,LastCollectedAt).
  TWO obs tables: scalar `alphavantage_observations`(Value) <- Commodity/EconomicIndicator · `alphavantage_ohlcv`(O/H/L/C/Vol) <- Equity/Forex/Crypto. PK=(series_id,date) both.
  INV catalog⊥SecMaster: AV series_id-space (`AV/{sym}`,`AV/FX/..`,`AV/CRYPTO/..`) is the local truth; SecMaster instruments separate, registration optional+best-effort. AV runs fully without SecMaster.
  INV quota in-process⊥persisted: dailyCount + token-bucket = in-memory; resets at UTC-midnight roll OR on restart. not DB-backed -> restart = quota wiped.
  INV stream-reads-scalar-only: gRPC GetLatest* hits `alphavantage_observations` ONLY -> Equity/Forex/Crypto (OHLCV) series NEVER emit Events. Stream covers Commodity/Economic only.
  INV upsert-orphan-guard: UpsertObservations silently DROPS rows whose series_id absent (AnyAsync gate), no error.
  INV TechnicalIndicator=scaffold: routable via admin/series-id but CollectionWorker has no branch -> logs "Unknown series type" each cycle, not collected.
  INV migrations=raw-SQL: hand-written `migrations/*.sql` not EF (predates ATLAS ef_core convention); no ModelSnapshot.

PATHS (distinct code — do not conflate):
  collect [BackgroundService · CollectionWorker, 4h tick]
    do: holiday-skip(CME via CalendarService) -> quota-check -> scheduler picks <=4 due (Priority asc->LastCollectedAt asc) -> fetch -> upsert -> UpdateLastCollected(only if count>0). 500ms inter-series.
    does NOT: register-SecMaster; emit-on-collect(stream polls independently); retry-cycle(MaxRetries unused).
    on-miss: count==0 -> does not advance LastCollectedAt (re-picked next cycle); quota-exhausted -> skip whole cycle; holiday -> skip.
  SubscribeToEvents / GetEventsSince [gRPC :5001 · ThresholdEngine]
    do: poll repo every 5s; per active series, latest scalar obs where CollectedAt>cursor -> Event(SeriesCollectedEvent{series_id,collected_at}). In-mem per-stream cursor.
    does NOT: values-in-payload(consumer re-fetches REST/repo); OHLCV; replay-history(cursor=in-mem); DataPoint-population(proto field empty).
    on-miss: GetHealth=STUB(Healthy=true,TotalEvents=0,LatestEventTime=now), not real counts.
  admin add/toggle/delete + discover [HTTP :8080 · operator/SecMaster-gw]
    do: AddSeries -> build series_id+Function/Category/Interval defaults -> persist -> IF SecMasterClient configured: fire-and-forget RegisterSeriesAsync (failures=WARN only). /discover proxies live SYMBOL_SEARCH (burns quota).
    does NOT: register-on-toggle/delete; register-existing/seeded series(only fresh AddSeries); block-on-SecMaster.
    on-miss: dup series_id -> 409 Conflict; toggle/delete unknown -> 404; null Symbol -> skip register(WARN).
  search [HTTP :8080 /api/search · SecMaster gateway]
    do: local-DB title/symbol LIKE -> {SeriesId,Title,Interval}. No upstream call, no quota.

PROCESSING MODEL:
  rate: token-bucket(5 cap, +1/min, queue 10) x hard daily-cap (DailyLimit=25/UTC-day). quota-exhausted -> fetch returns []/null (no throw). API "Note"/"Error Message" in body -> []  (counts as used request).
  schedule: due = LastCollectedAt==null or elapsed>interval-floor(daily 20h · weekly 6d · monthly 25d · quarterly 80d · annual 350d).

DISTINCTIONS:
  scalar-obs(Value, observations tbl) ≠ OHLCV(ohlcv tbl) — type-routed; stream+`/latest` branch on Type.
  series catalog (local AV) ≠ SecMaster instrument (registration is f-a-f advisory, not authoritative source).
  Event.collected_at(emit signal) ≠ obs.Date(market date); cursor compares CollectedAt, GetLatest orders by Date.
  Function(UPPERCASE API fn) ≠ Category(TitleCase label) ≠ Interval(lowercase cadence) — 3 derived axes per Type.
  `/api/search`(local, SecMaster-gw) ≠ `/api/discover`(upstream SYMBOL_SEARCH, quota-burning).

CROSS-SERVICE: SecMaster <- register(gRPC f-a-f, OPTIONAL, on-add-only). FEEDS ThresholdEngine via SubscribeToEvents(gRPC). CalendarService -> CME holidays. OUT: AlphaVantage HTTP, OTEL :4317.

GOTCHAS:
  ✗ assume-OHLCV-series-emit-events (stream is scalar-only)
  ✗ assume-quota-survives-restart (in-mem, resets)
  ✗ treat-SecMaster-registration-as-required/authoritative (f-a-f, optional, add-only)
  ✗ expect-values-in-Event (payload = id+timestamp; re-fetch REST)
  ✗ add-TechnicalIndicator-series (scaffold; "Unknown" log spam, not collected)
  ✗ trust-GetHealth-gRPC-counts (stub) ✗ edit-migrations-as-EF (raw SQL)
  ✗ /discover-without-quota-budget (live upstream call)
  ✗ upsert-obs-before-series-exists (silently dropped)

DECISIONS: none recorded yet — accrete on touch (not audited for exception paths; see CLAUDE.md INTENT_FIDELITY MECHANICS).

SEE: README.md §API Endpoints/Config · Events/src/Events/Protos/observation_events.proto (Event+SeriesCollectedEvent) · AlphaVantageApiClient.cs:42-49,675-700(rate+quota) · CollectionScheduler.cs:37-42(sort: Priority asc ThenBy LastCollectedAt asc),51-67(due) · ObservationEventStreamService.cs:51-62(scalar-cursor) · SeriesManagementService.cs:66-106(f-a-f register) · AlphaVantageRepository.cs:104-126(orphan-guard+latest)
