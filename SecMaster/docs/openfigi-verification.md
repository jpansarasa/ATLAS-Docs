# OpenFIGI integration — verification (Story 1.5.1)

This is a one-pager verifying what the existing OpenFIGI integration covers
and what was added in Story 1.5.1 to expose it as a deliberate public
identifier-resolution surface.

## What's already in place

### `IOpenFigiClient` / `OpenFigiClient` (`src/Services/`)

- Wraps the OpenFIGI **`POST /v3/mapping`** batch API.
- **Input:** ticker symbol with optional exchange suffix (`AAPL`, `600809.SS`,
  `DXR.TO`, share-class tickers like `BRK.A`). The client normalises suffix
  to OpenFIGI `exchCode` (`SS`, `SZ`, `HK`, `CT`, `LN`, …) — see
  `SuffixToExchCode` in `OpenFigiClient.cs`.
- **Output (`OpenFigiMapping`):** FIGI, compositeFIGI, shareClassFIGI, name,
  ticker, exchCode, country, marketSector, securityType. **CUSIP / ISIN /
  SEDOL are explicitly nulled** in `MapResponse` (lines 254-256) — the
  default `/v3/mapping` response does **not** carry those fields. Surfacing
  them requires `/v3/search` (different endpoint, different auth budget).
- API key is sent on the request body as `X-OPENFIGI-APIKEY` only when
  `OpenFigiOptions.ApiKey` is non-empty; when null the client gracefully
  falls back to anonymous quota.
- HTTP error handling: 4xx/5xx propagate as `HttpRequestException` so the
  caller's retry / no-data classification logic decides what to do.
  Validated by `OpenFigiClientTests.should_throw_on_500_so_caller_retries`
  and `should_throw_on_401_with_bad_key`.

### `OpenFigiEnrichmentBackgroundService`

- Runs on a polling interval (`EnrichmentOptions.IntervalSeconds`) and pulls
  active equity / ETF / stock instruments missing **any** of FIGI / CUSIP /
  ISIN / SEDOL.
- Honours an in-row `openfigi_enrichment_attempted_at` cooldown
  (`RetryAfterDays`) so a row that came back `no_data` is not hammered.
- Bumps `instruments.UpdatedAt` on a successful enrichment so the embedding
  background service re-embeds with the populated cross-reference fields.
- Telemetry: `secmaster_openfigi_enrichment_total{result=enriched|no_data|error}`,
  duration histogram, pending gauge — all already emitted to OTel.

### Wiring (`DependencyInjection.cs`)

- `services.Configure<OpenFigiOptions>(...)` — section name `"OpenFigi"`.
- `services.AddHttpClient<IOpenFigiClient, OpenFigiClient>(...)` — named
  HTTP client with `BaseUrl` and `RequestTimeout` from options.
- `services.AddHostedService<OpenFigiEnrichmentBackgroundService>()`.

## What was observed about rate limiting & caching

OpenFIGI free tier limits (no key): **25 req/min, 250 req/6h**. With key:
**25 req/min, 60K/day** (each request can carry up to 100 jobs).

- **No explicit rate limiter** exists in the client. Throttling is
  achieved by:
  1. The background service's `EnrichmentOptions.IntervalSeconds` cadence —
     each cycle issues at most `ceil(BatchSize / MaxJobsPerRequest)` POSTs.
  2. `OpenFigiOptions.IntervalBetweenBatchesMs` (default 250ms) inserted
     between sub-batches inside a single cycle.
  3. `RetryAfterDays` cooldown which prevents re-attempting `no_data` rows.
- **No 429-specific backoff.** A 429 today is propagated as
  `HttpRequestException` and the entire batch is dropped on the floor for
  the next cycle (no `attempted_at` is written, so it retries next cycle).
  This is acceptable for the background-enrichment use case where cycle
  cadence is on the order of minutes; it is **not ideal for a
  synchronous on-demand resolve API**, and the new
  `IdentifierResolutionService` therefore returns the OpenFIGI failure as
  null (and logs a warning) rather than retrying inline.
- **Cache layer:** there is no in-process cache. The de-facto cache is the
  `instruments` table itself — once a row is enriched, subsequent requests
  resolved through the new `IdentifierResolutionService` hit the DB and
  never call OpenFIGI. `aliases` table is a parallel lookup path for
  ticker / FIGI strings.

## What Story 1.5.1 added

A thin domain service that **exposes the existing data** as a deliberate
identifier-resolution surface, **DB-first** so the OpenFIGI free-tier
budget is preserved.

- `IIdentifierResolutionService.ResolveIdentifiersAsync(IdentifierQuery)`
  returns `IdentifierResolutionResult?` populated from the `instruments`
  table when a row already carries the requested identifier.
- Cache miss + ticker query → falls through to `IOpenFigiClient` (the only
  upstream lookup the existing client supports for input).
- Cache miss + CUSIP / ISIN / FIGI query → returns null. `/v3/mapping`
  cannot resolve those input types, and re-implementing `/v3/search` is
  out of scope for this story. Documented in the
  `cusip_isin_figi_lookup_requires_local_cache` test.
- REST: `GET /api/identifiers/resolve?{cusip|isin|figi|ticker}=…`.
- MCP: `resolve_identifiers(input_kind, value, exchange_hint?)`.
- gRPC: deferred (matches the prior epic precedent — REST + MCP only).

## What was found broken

**Nothing was rewritten in the existing service.** The deliberate scoping
issues above (CUSIP/ISIN/SEDOL not returned by `/v3/mapping`, no inline
429 backoff, no in-process cache) are documented as known limitations
rather than fixed in place — they are non-blocking for the verify-and-expose
mission and the supervisor's mandate explicitly says "Do not rewrite the
existing service. Do not change its config shape unless something is
genuinely broken."

The new `IdentifierResolutionService` works around the input-type
limitation by relying on the local cache (`instruments` columns +
`aliases`) for CUSIP / ISIN / FIGI inputs — when those are present in the
DB (populated by the background enrichment cycle), they are returned
without an upstream call. Only ticker queries can fall through to live
OpenFIGI.
