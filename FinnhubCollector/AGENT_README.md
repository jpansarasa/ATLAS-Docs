# FinnhubCollector — architecture [agent read-first]

PURPOSE: poll Finnhub REST->persist subset->re-serve via gRPC. Not threshold-eval, not alert, not instrument-registry (registers TO SecMaster, f-a-f). source-of-truth=own collected snapshots, not live-market-state.

DATA MODEL + INVARIANTS:
  INV series-id: DB=`FH-{TYPE}-{SYMBOL}` (dashes) != gRPC SeriesId=`FH/{symbol}` (slash). client filter must use `FH/` prefix.
  INV candle-dual-linkage: Candle has BOTH denormalized series_id STRING col AND nullable series_config_id INT FK->finnhub_series. two separate cols for same linkage; lookup-by-string != join-by-FK. (moot: nothing writes candles.)
  INV no-FK-on-symbol-tables: finnhub_news_sentiment/insider/recs/price_targets/profiles/calendars key on symbol+time — NO FK->finnhub_series. schema does not enforce series exists.
  INV dedup-key-varies: not uniform (symbol,collected_at). quotes=(symbol,timestamp); news_sentiment=(symbol,collected_at); insider=(symbol,year,month); recs=(symbol,period); profiles=(symbol); econ-cal=(country,event,event_time). wrong-key->silent miss.
  INV BullishPercent⊥: Ignore()d derived prop in finnhub_recommendations != mapped bullish_percent COLUMN in finnhub_news_sentiment. same name, different treatment per table.
  LIVE-WRITE PATHS: QuoteCollectionWorker(loop,Quote only) + admin-collect(trigger,Quote/NewsSentiment/Recommendation/PriceTarget/CompanyProfile). Candle/social/insider/calendars=dead-schema: upsert methods exist, 0 callers -> tables permanently empty -> MCP+/api/* reads return empty.

PATHS (distinct code — do not conflate):
  QuoteCollectionWorker [BackgroundService, 1-min loop, 10s startup]
    does: GetActiveSeriesAsync(Quote)->rate-limit->GetQuoteAsync->UpsertQuoteAsync->WriteAsync(channel)->UpdateLastCollectedAt.
    does NOT: non-Quote types; SecMaster registration.
    on-miss(null quote): UpsertQuoteAsync+WriteAsync SKIPPED; UpdateLastCollectedAt fires UNCONDITIONALLY outside null-guard -> LastCollectedAt advances on empty response.
  admin-collect [HTTP POST /api/admin/series/{id}/collect · ops/MCP]
    does: type-switch->single collect(Quote/NewsSentiment/Recommendation/PriceTarget/CompanyProfile); queued via BackgroundCollectionQueue(DropWrite@100, dup-suppressed).
    does NOT: Candle; calendars; insider; social (all fall to default+warn).
    on-miss: 404(series absent); 429(queue full OR already queued).
  stored-reads [HTTP GET /api/* · ApiEndpoints->IFinnhubRepository]
    does: read latest/history from DB.
    does NOT: Finnhub call; write. /api/market/status, /api/symbols/search, /api/discover = exceptions->proxy live Finnhub.
    on-miss: 404.
  live-passthrough [HTTP GET /api/live/* · LiveDataEndpoints->IFinnhubApiClient]
    does: direct Finnhub call, tags source="finnhub-live". includes /api/live/peers/{symbol}.
    does NOT: read DB; write DB.
  gRPC-stream [ObservationEventStream :5001 · ThresholdEngine]
    does: Subscribe/GetEventsSince/Between/GetLatestEventTime/GetHealth over finnhub_quotes only.
    does NOT: candles; sentiment; calendars (never stored -> never streamed).
    on-miss(empty table): GetLatestEventTime returns DateTime.UtcNow, not null/sentinel -> consumer polling "latest time" sees forever-advancing clock against empty table. Never use as new-data signal naively.
  gRPC port: Program.cs logs the actual bound port from Kestrel:GrpcPort (default 5001). ThresholdEngine subscribes ObservationEventStream on :5001.

PROCESSING MODEL: WaitForTokenAsync(token-bucket 60/min,RATE_LIMITER_CAPACITY) -> Finnhub HTTP(Polly: 3x exp-retry, CB 5-fail/60s, 30s timeout) -> upsert(UNIQUE idempotent) -> UpdateLastCollectedAt.
MAXIM: store-then-serve; DB table=event log; channel=not.
PROFILE2 NEGATIVE-CACHE: GetCompanyProfileAsync remembers symbols that returned a permanent 4xx (403=plan-uncovered foreign listing · 404=unknown) in a static (process-wide) cache and SKIPS re-calling them for ProfileNegativeCacheTtl (default 24h, `Finnhub:ProfileNegativeCacheTtlHours`) — stops per-cycle quota burn + WARN spam on uncovered listings. 429/5xx NOT cached (transient). First 403/404/429 logs Info (not WARN); subsequent skips are Debug-only.

DISTINCTIONS:
  live(/api/live/* + /api/market/status + /api/symbols/search + /api/discover) ≠ stored(/api/quotes,/api/sentiment,/api/analyst,/api/company): live=Finnhub-each-call+no-persist; stored=DB-only.
  /api/symbols/search+/api/discover(upstream Finnhub discovery) ≠ /api/search(UnifiedSearch over LOCAL finnhub_series for SecMaster gateway).
  BackgroundCollectionQueue(admin-collect work queue,HAS reader,DropWrite) ≠ ObservationChannel(quote-write,NO reader,FullMode=Wait->BLOCKS).
  DI(namespace FinnhubCollector,src/DependencyInjection.cs) ≠ DI(namespace FinnhubCollector.Services,src/Services/ApplicationDependencyInjection.cs). same class name, two namespaces.

CROSS-SERVICE:
  OUT(sync): Finnhub API (all data). OUT(f-a-f,optional): SecMaster RegisterSeriesAsync(assetClass=Equity; symbol=ticker; metadata["alias"]=FH/{symbol} stream id -> SecMaster series_id alias so ThresholdEngine pattern symbols resolve; gated SECMASTER_GRPC_ENDPOINT; failures=WARN,non-blocking). Backfill: POST /api/admin/series/register?seriesId=... (awaited, idempotent; query param because legacy FH/{sym} ids contain '/'). OUT: OTLP->otel-collector.
  IN(gRPC :5001): ThresholdEngine subscribes ObservationEventStream.
  FEEDS: ThresholdEngine via quote events; SecMaster catalog+sector via registration+/api/search+/api/discover gateway.

GOTCHAS:
  ✗ assume non-Quote data flows anywhere (candle/social/insider/calendars never written -> tables empty)
  ✗ ObservationChannel as harmless dead-end: FullMode=Wait+no-reader->WriteAsync BLOCKS at 1000 quotes -> stalls collection loop (does not drop)
  ✗ null-quote-is-no-op: LastCollectedAt advances unconditionally
  ✗ QuoteCollectionWorker collects non-Quote: needs admin trigger; Candle/calendar/insider/social not handled even there
  ✗ uniform (symbol,collected_at) dedup key: insider=(symbol,year,month); social=(symbol,date)
  ✗ GetLatestEventTime as new-data signal: returns UtcNow on empty table
  ✗ Quartz-schedules-jobs: Quartz wired DI but schedules 0 jobs (dormant scaffolding)
  ✗ DB_HOST/DB_USER env vars: DB=ConnectionStrings__AtlasDb only
  ✗ SECMASTER_GRPC_ENDPOINT unset=nullable-singleton: absent->SecMasterRegistryClient never registered->registration silently skips

DECISIONS:
  D-1 runtime-log-level: INTENT a WARN-quiet-by-design collector must be raisable to Information/Debug AT RUNTIME (on-demand deep debugging) with NO restart or redeploy, while steady state stays Warning (prod-quiet). A LoggingLevelSwitch is the SOLE level authority; the operator edits Serilog:MinimumLevel:Default in the RUNNING container and reloadOnChange picks it up live. Serilog.Settings.Configuration 9.0.0 does NOT re-apply MinimumLevel on IConfiguration reload, so an explicit ChangeToken re-parse is what makes the edit take effect; the MEL floor sits at Debug so the bridge does not pre-filter below the switch. The raised level lives ONLY in the container's writable layer — revert by editing the key back to Warning (live) OR RECREATE (not restart) the container to reinstate the image's baked-in default; a restart PRESERVES the writable layer and does NOT revert. No compose/mount change (baked-in appsettings is the assumed, acceptable case). / PRECOND operator runs `nerdctl exec finnhub-collector` to edit /app/appsettings.json Serilog:MinimumLevel:Default — no env override for that key exists, so the JSON is authoritative and reloadable. / GUARD RuntimeLogLevel.BuildLogger (ControlledBy = sole authority) + RuntimeLogLevel.CreateSwitch (ChangeToken re-parse on reload) + RuntimeLogLevel.AddRuntimeControlledSerilog (MEL Debug floor) @ src/Telemetry/RuntimeLogLevel.cs / TEST RuntimeLogLevelTests.Runtime_appsettings_edit_raises_then_lowers_serilog_level

SEE: README.md §Reference · Events/src/Events/Protos/observation_events.proto (ObservationEventStream — EXPOSES gRPC :5001) · Events/src/Events/Protos/secmaster.proto (SecMasterRegistry.RegisterSeries — consumed f-a-f, gated SECMASTER_GRPC_ENDPOINT) · src/Grpc/Services/EventStreamService.cs(GetLatestEventTime UtcNow fallback) · src/Services/SeriesManagementService.cs:194-244(trigger switch+SecMaster reg) · src/Workers/QuoteCollectionWorker.cs:103,115,122(null-guard+channel+unconditional-update) · src/Events/ObservationChannel.cs:14,16(FullMode=Wait,SingleReader=false)
