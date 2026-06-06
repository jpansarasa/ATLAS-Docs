# CalendarService — architecture [agent read-first]

PURPOSE: temporal truth for ATLAS — NYSE trading-days/holidays + scheduled high-impact econ-release dates. ¬price/obs ¬gRPC ¬a-generic-econ-calendar(curated allow-list only).

DATA MODEL + INVARIANTS (schema does NOT enforce):
  TRANSPORT: HTTP :8080 ONLY (internal, no host port). ¬gRPC :5001 despite monorepo DATA_FLOW — consumers reach via HTTP. Containerfile EXPOSE 8080.
  econ-source = FRED ONLY at runtime. Finnhub wired (client+opts+worker) but worker COMMENTED-OUT in DI (paid sub). Nager = on-demand passthrough, never collected.
  INV curated-allow-list: FredReleasesClient.HighImpactReleases = ~18 hardcoded release_ids → unknown release_id SILENTLY SKIPPED (continue), ¬persisted. econ_events ≠ "all FRED releases".
  INV synthetic-times: FRED gives DATE only; event_time FABRICATED = 8:30 ET(High)|10:00 ET(Med), hardcoded etOffset=-5h → DST-UNAWARE (off by 1h during EDT). ¬actual release timestamp.
  INV provider⊥db: market endpoints read in-memory StaticNyseHolidays (computed: Easter/nth-weekday algos), ¬market_holidays table. DB = mirror for external consumers; API answers don't touch it.
  INV upsert-key = event_id(unique idx "FRED-{releaseId}-{yyyyMMdd}"), ¬PK id(long). Re-collect = UPDATE-by-event_id.
  INV us-hardcoded: /upcoming /high-impact /has-high-impact force Country=="US" in repo. ONLY /events honors ?country filter. impact/eventType compared as STRINGS ("High"), parse-fail→Low/Other enum fallback.
  INV otel-IS-wired: Program.cs exports traces+metrics OTLP→:4317. (README "OTEL unused" = DRIFT, stale.)

PATHS (distinct entry — do not conflate):
  market-calendar [HTTP :8080 /api/market/* · ThresholdEngine+services]
    do: status/is-trading-day/next-trading-day/trading-days/holidays from computed NYSE rules (weekend+FullClosure+Observed; EarlyClose=trading). status in ET, DST-aware (America/New_York).
    does NOT: ¬read DB ¬gRPC ¬write. trading-days loop unbounded by [from,to] caller.
    on-miss: GetNextTradingDay throws after 14d scan (not null).
  econ-calendar [HTTP :8080 /api/economic/* · services]
    do: query persisted econ_events (range/impact/eventType/country); upcoming/high-impact US-only.
    does NOT: ¬live-fetch (serves last collected). ¬non-allow-list releases ever appear.
    on-miss: empty list (no event ≠ error). high-impact uses partial idx (impact='High').
  external-holidays [HTTP :8080 /api/market/holidays/external · on-demand]
    do: Nager.Date US public holidays, filtered to 9 NYSE-relevant names (British-spelling map "Labour Day").
    on-miss: Nager HTTP fail → LogWarning + return [] (swallowed; passthrough, not persisted).

PROCESSING MODEL (2 hosted workers; PeriodicTimer + initial run on startup):
  MarketHolidaySyncWorker 24h → upsert curr+next year into market_holidays (from computed provider).
  FredReleasesCollectorWorker 6h → next 60d FRED releases → upsert econ_events.
  Polly retry 3×exp(5s); consecutive-fail escalates LogWarning→LogError @3; per-source success/error counters + last_success dead-man gauge (source∈{fred,holiday}).

DISTINCTIONS:
  FRED HTTP-error THROWS (¬swallow-to-[]) — load-bearing: [] would fresh the dead-man gauge + make RecordError unreachable. Empty [] = genuinely no high-impact release in window = the ONLY success-with-no-rows.
  Nager-fail swallows-to-[] ≠ FRED-fail throws — passthrough is best-effort, collection is monitored.
  Finnhub worker DISABLED ≠ removed (client+DI HttpClient+meter "finnhub" source all live, just no AddHostedService).
  computed-holidays(provider, authoritative) ≠ Nager-holidays(external, validation/supplement) ≠ DB-holidays(mirror).
  EconomicImpact/EventType = enum in Core but stored+queried as STRING; filter is exact-string ¬enum.

CROSS-SERVICE: ThresholdEngine → /api/market/* (HTTP, trading-day checks). AlphaVantageCollector/NasdaqCollector → CalendarService.Core lib (in-proc, ¬HTTP). OUT: FRED HTTP (econ calendar); Nager.Date HTTP (on-demand external holidays). FEEDS: econ_events table + market_holidays mirror.

GOTCHAS:
  ✗ assume-gRPC :5001 (HTTP-only) ✗ expect-all-FRED-releases (curated ~18) ✗ trust FRED event_time as real (synthetic, DST-off)
  ✗ expect non-US in /upcoming|/high-impact (US-hardcoded) ✗ read holidays from DB (API uses provider)
  ✗ patch FRED client to return [] on error (re-introduces dead-man-gauge masking) ✗ believe README "OTEL unused"
  ✗ enable-Finnhub-worker without paid sub ✗ rely on Nager for closures (validation-only, swallows errors)

SEE: README.md §API+Config (drift: OTEL/gRPC) · src/External/FredReleasesClient.cs:17-52(allow-list)+93-100(synthetic ET time)+122-131(throw-not-swallow) · src/Workers/*(Polly+escalation) · src/DependencyInjection.cs:52-55(Finnhub disabled) · src/Persistence/CalendarDbContext.cs(idx+event_id unique) · src/Core/README.md(shared lib)
