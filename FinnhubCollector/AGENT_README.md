# FinnhubCollector — architecture [agent read-first]

PURPOSE: poll Finnhub REST→persist subset→re-serve via gRPC. ¬threshold-eval ¬alert ¬instrument-registry (registers TO SecMaster, f-a-f). source-of-truth=own collected snapshots ¬live-market-state.

DATA MODEL + INVARIANTS:
  INV series-id: DB=`FH-{TYPE}-{SYMBOL}` (dashes) ≠ gRPC SeriesId=`FH/{symbol}` (slash). client filter must use `FH/` prefix.
  INV candle-dual-linkage: Candle has BOTH denormalized series_id STRING col AND nullable series_config_id INT FK→finnhub_series. two separate cols for same linkage; lookup-by-string ≠ join-by-FK. (moot: nothing writes candles.)
  INV no-FK-on-symbol-tables: finnhub_news_sentiment/insider/recs/price_targets/profiles/calendars key on symbol+time — NO FK→finnhub_series. schema ¬enforce series exists.
  INV dedup-key-varies: ¬uniform (symbol,collected_at). quotes=(symbol,timestamp); news_sentiment=(symbol,collected_at); insider=(symbol,year,month); recs=(symbol,period); profiles=(symbol); econ-cal=(country,event,event_time). wrong-key→silent miss.
  INV BullishPercent⊥: Ignore()d derived prop in finnhub_recommendations ≠ mapped bullish_percent COLUMN in finnhub_news_sentiment. same name, different treatment per table.
  LIVE-WRITE PATHS: QuoteCollectionWorker(loop,Quote only) + admin-collect(trigger,Quote/NewsSentiment/Recommendation/PriceTarget/CompanyProfile). Candle/social/insider/calendars=dead-schema: upsert methods exist, 0 callers → tables permanently empty → MCP+/api/* reads return ∅.

PATHS (distinct code — do not conflate):
  QuoteCollectionWorker [BackgroundService, 1-min loop, 10s startup]
    does: GetActiveSeriesAsync(Quote)→rate-limit→GetQuoteAsync→UpsertQuoteAsync→WriteAsync(channel)→UpdateLastCollectedAt.
    does NOT: ¬non-Quote types ¬SecMaster registration.
    on-miss(null quote): UpsertQuoteAsync+WriteAsync SKIPPED; UpdateLastCollectedAt fires UNCONDITIONALLY outside null-guard → LastCollectedAt advances on empty response.
  admin-collect [HTTP POST /api/admin/series/{id}/collect · ops/MCP]
    does: type-switch→single collect(Quote/NewsSentiment/Recommendation/PriceTarget/CompanyProfile); queued via BackgroundCollectionQueue(DropWrite@100, dup-suppressed).
    does NOT: ¬Candle ¬calendars ¬insider ¬social (all fall to default+warn).
    on-miss: 404(series absent); 429(queue full OR already queued).
  stored-reads [HTTP GET /api/* · ApiEndpoints→IFinnhubRepository]
    does: read latest/history from DB.
    does NOT: ¬Finnhub call ¬write. /api/market/status, /api/symbols/search, /api/discover = exceptions→proxy live Finnhub.
    on-miss: 404.
  live-passthrough [HTTP GET /api/live/* · LiveDataEndpoints→IFinnhubApiClient]
    does: direct Finnhub call, tags source="finnhub-live". includes /api/live/peers/{symbol}.
    does NOT: ¬read DB ¬write DB.
  gRPC-stream [ObservationEventStream :5001 · ThresholdEngine]
    does: Subscribe/GetEventsSince/Between/GetLatestEventTime/GetHealth over finnhub_quotes only.
    does NOT: ¬candles ¬sentiment ¬calendars (never stored → never streamed).
    on-miss(empty table): GetLatestEventTime returns DateTime.UtcNow ¬null/sentinel → consumer polling "latest time" sees forever-advancing clock against empty table. ¬use as new-data signal naively.
  gRPC port: Program.cs logs the actual bound port from Kestrel:GrpcPort (default 5001). ThresholdEngine subscribes ObservationEventStream on :5001.

PROCESSING MODEL: WaitForTokenAsync(token-bucket 60/min,RATE_LIMITER_CAPACITY) → Finnhub HTTP(Polly: 3x exp-retry, CB 5-fail/60s, 30s timeout) → upsert(UNIQUE idempotent) → UpdateLastCollectedAt.
MAXIM: store-then-serve; DB table=event log; channel=not.

DISTINCTIONS:
  live(/api/live/* + /api/market/status + /api/symbols/search + /api/discover) ≠ stored(/api/quotes,/api/sentiment,/api/analyst,/api/company): live=Finnhub-each-call+¬persist; stored=DB-only.
  /api/symbols/search+/api/discover(upstream Finnhub discovery) ≠ /api/search(UnifiedSearch over LOCAL finnhub_series for SecMaster gateway).
  BackgroundCollectionQueue(admin-collect work queue,HAS reader,DropWrite) ≠ ObservationChannel(quote-write,NO reader,FullMode=Wait→BLOCKS).
  DI(namespace FinnhubCollector,src/DependencyInjection.cs) ≠ DI(namespace FinnhubCollector.Services,src/Services/ApplicationDependencyInjection.cs). same class name, two namespaces.

CROSS-SERVICE:
  OUT(sync): Finnhub API (all data). OUT(f-a-f,optional): SecMaster RegisterSeriesAsync(assetClass=Equity; symbol=ticker; metadata["alias"]=FH/{symbol} stream id → SecMaster series_id alias so ThresholdEngine pattern symbols resolve; gated SECMASTER_GRPC_ENDPOINT; failures=WARN,non-blocking). Backfill: POST /api/admin/series/register?seriesId=… (awaited, idempotent; query param because legacy FH/{sym} ids contain '/'). OUT: OTLP→otel-collector.
  IN(gRPC :5001): ThresholdEngine subscribes ObservationEventStream.
  FEEDS: ThresholdEngine via quote events; SecMaster catalog+sector via registration+/api/search+/api/discover gateway.

GOTCHAS:
  ✗ assume non-Quote data flows anywhere (candle/social/insider/calendars ¬written → tables ∅)
  ✗ ObservationChannel as harmless dead-end: FullMode=Wait+no-reader→WriteAsync BLOCKS at 1000 quotes → stalls collection loop (¬drop)
  ✗ null-quote-is-no-op: LastCollectedAt advances unconditionally
  ✗ QuoteCollectionWorker collects non-Quote: needs admin trigger; Candle/calendar/insider/social ¬handled even there
  ✗ uniform (symbol,collected_at) dedup key: insider=(symbol,year,month); social=(symbol,date)
  ✗ GetLatestEventTime as new-data signal: returns UtcNow on empty table
  ✗ Quartz-schedules-jobs: Quartz wired DI but schedules 0 jobs (dormant scaffolding)
  ✗ DB_HOST/DB_USER env vars: DB=ConnectionStrings__AtlasDb only
  ✗ SECMASTER_GRPC_ENDPOINT unset=nullable-singleton: absent→SecMasterRegistryClient never registered→registration silently skips

SEE: README.md §Reference · Events/src/Events/Protos/observation_events.proto (ObservationEventStream — EXPOSES gRPC :5001) · Events/src/Events/Protos/secmaster.proto (SecMasterRegistry.RegisterSeries — consumed f-a-f, gated SECMASTER_GRPC_ENDPOINT) · src/Grpc/Services/EventStreamService.cs(GetLatestEventTime UtcNow fallback) · src/Services/SeriesManagementService.cs:194-244(trigger switch+SecMaster reg) · src/Workers/QuoteCollectionWorker.cs:103,115,122(null-guard+channel+unconditional-update) · src/Events/ObservationChannel.cs:14,16(FullMode=Wait,SingleReader=false)
