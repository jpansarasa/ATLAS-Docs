# ATLAS Architecture — the system as built

Current-state reference for the running system on **mercury** (the single production host).
Deep-dives: [MATRIX.md](./MATRIX.md) (signal matrix), [SENTINEL-RLM.md](./SENTINEL-RLM.md)
(extraction model/backend constraints), [OBSERVABILITY.md](./OBSERVABILITY.md),
[GRPC-ARCHITECTURE.md](./GRPC-ARCHITECTURE.md), per-service `README.md` +
`AGENT_README.md` cards (read the card before reasoning about a service).

## 1. Host and resource topology

- mercury: 48 hardware threads, 125 GiB RAM, NVIDIA RTX 5090 (32,607 MiB; ~29.8 GiB held by
  vLLM at 0.92 gpu-memory-utilization).
- CPU islands via compose `cpuset` (CFS-throttle avoidance; islands sum to all 48 threads):
  llama-server 0-23, llama-cpu-rag 24-39, llama-cpu-embed 40-47.
- ZFS: `nvme-fast` (models, timeseries DB, dashboards, containers) + `sata-bulk` (logs,
  raw-data, backups, archives, agent workspace).
- UPS: APC via `apcupsd` + ups-exporter → Prometheus job `ups`.
- Host systemd (non-container) services: `sandbox-manager` (spawns sandbox-kernel containers
  for Sentinel), `gemini-resolver-mcp` (port 9300, reached from containers via `host-gateway`),
  `apcupsd`.

## 2. Two compose stacks

| Stack | File | systemd unit | Contents |
|---|---|---|---|
| Main | `/opt/ai-inference/compose.yaml` (ansible-generated, ~1,400 lines) | `atlas.service` | DB, collectors, processing, alerting, sidecars, MCP servers, GPU (vllm-server) + CPU inference |
| OTEL | `/opt/otel/compose.otel.yaml` (templated from `deployment/artifacts/compose.otel.yaml.j2`) | `otel.service` | prometheus, grafana, loki, tempo, otel-collector, node/gpu/ups exporters |

Both stacks share the external `ai-inference` network. `atlas.service` requires
`otel.service` so the network exists first. vllm-server is a main-compose service
(since 2026-06-11; previously a standalone `nerdctl run` container) — GPU passthrough
via `deploy.resources.reservations.devices`, image digest-pinned.

> **Caveat that bites**: the OTEL services are invisible to both the full and scoped
> compose restart paths of `deploy.yml`. `scoped_restart` only reaches main-compose
> services; OTEL services need `--tags otel` or a manual `nerdctl restart`.
> vllm-server is now in the main stack, so full-stack restarts cycle it (~3.5-4 min
> GPU model reload); tag-scoped redeploys still need the opt-in `--tags vllm-server`
> deploy block (recreate + `/health` wait + 1-token smoke).

### Service inventory (main stack, live)

- **timescaledb** — timescale/timescaledb 2.23.1-pg16, host port 5432.
- **migrate-macro-substrate** — one-shot EF migrator for `macro_observations`.
- **Collectors**: fred-collector, alphavantage-collector, finnhub-collector, ofr-collector,
  sentinel-collector (host 5091 review UI). **nasdaq-collector is commented out** (NDL WAF
  blocks datacenter IPs — code complete, not running).
- **Processing/metadata**: threshold-engine (12G mem), secmaster, calendar-service (own DB
  `calendar_data`).
- **Alerting**: alertmanager (internal), alert-service (ntfy topic `atlas-alert` + Gmail SMTP +
  AutoFix queue).
- **Sidecars**: trafilatura (3109), spacy-ner (internal 3110), finbert-sidecar, dsl-parser-mcp
  (3120), markitdown-mcp (3102), whisper-service (8090) + whisper-service-mcp.
- **MCP servers** (SSE, host 310x): fredcollector-mcp 3103, thresholdengine-mcp 3104,
  finnhub-mcp 3105, ofr-mcp 3106, secmaster-mcp 3107, whisper-service-mcp 3108.
- **Reports**: reports-daily/-weekly/-monthly (FireAtUtc 02:00; Markdown+HTML to
  `/var/lib/atlas/reports/`; narrative via vllm-server).
- **CPU inference**: llama-server, llama-cpu-rag, llama-cpu-embed (§3).

## 3. Inference topology (zero ollama)

All CPU inference runs **llama.cpp** (`ghcr.io/ggml-org/llama.cpp:server`); the GPU runs
**vLLM**. No ollama container or engine exists. GGUF blobs are read-only bind-mounts from the
frozen ollama-format content store — digest-pinned; re-provisioning the store without updating
digests would silently serve stale weights (deploy `/health`+`/props` checks are the guard).

| Runtime | Port | Model | Role |
|---|---|---|---|
| **vllm-server** (GPU) | 8000 | Qwen/Qwen2.5-32B-Instruct-AWQ (awq_marlin, 32K ctx, max_num_seqs 16, fp8 KV cache) | Sentinel JSON-CoD extraction, news-signal classification, reports narrative |
| llama-server (CPU 0-23) | 11437 | qwen3-30b-a3b-instruct (32K ctx) | CPU GBNF/DSL extraction backend — **rollback path only** (`Extraction__Backend=LlamaServerDsl`) |
| llama-cpu-rag (CPU 24-39) | 11438 | qwen2.5-7b-instruct q4_K_M (8K ctx, 4 slots) | SecMaster RAG generation (sole consumer) |
| llama-cpu-embed (CPU 40-47) | 11439 | bge-m3 (16K ctx, batch=ubatch=8192) | Embeddings; pgvector-compatible with all existing rows |

LoRA is fully removed from extraction; the served model name is the canonical HF id.
whisper (CPU faster-whisper) and finbert are separate model sidecars. SecMaster's `Ollama__*`
config keys are a naming seam only — the engine behind them is llama.cpp.

## 4. Data flow

### 4a. Live threshold path (event-driven)

```
Collectors ──ObservationEventStream gRPC :5001──► ThresholdEngine
              MultiCollectorEventConsumerWorker → ObservationCache → pattern eval
              → ThresholdEvents on crossings → metrics → Prometheus → Alertmanager
              → AlertService → ntfy | email | AutoFix queue
```

This path **never writes matrix cells**. gRPC is container-to-container only; HTTP is 8080
internal / 50xx on the host.

### 4b. Matrix path (DB-polled) — see [MATRIX.md](./MATRIX.md)

```
Sentinel news ──GPU classifier──► macro_observations (":sig:" rows, value = tilt × confidence)
FRED macro-classified series ──dual-write──► macro_observations
OFR (wired, 0 series flagged → dormant) ──► macro_observations
        ▼ (5-min DB poll)
ThresholdEngine ObservationCellProjector ──► matrix_cells (11 sectors × signal × source)
```

### 4c. Registration and resolution

```
Collectors ──RegisterSeries gRPC :5001 (fire-and-forget)──► SecMaster
ThresholdEngine ──ResolveBatch gRPC (pattern LOAD TIME only)──► SecMaster
Sentinel ──POST /api/resolve-entities (HTTP, prepass)──► SecMaster
FRED / OFR ──REST /api/signal-identities/* (macro gate, tagging)──► SecMaster
```

CalendarService is **HTTP-only** (no gRPC :5001 despite older diagrams); its econ source at
runtime is FRED only, via a curated allow-list of ~18 release ids, and its `event_time` is
synthesized (8:30/10:00 ET, DST-unaware).

## 5. The Sentinel pipeline (news → extraction → digest)

Detail in `SentinelCollector/README.md`; the matrix-facing half in [MATRIX.md](./MATRIX.md).

1. **Ingestion**: RSS feeds (DB-managed, per-feed poll intervals) and SearXNG queries (ET
   schedule: 09:30/12:00/16:00/22:00 + 07:00 daily, Mondays weekly) submit URLs to the
   Cloudflare edge worker; `EdgeSyncWorker` polls results back every 30s into
   `sentinel.raw_content` (payloads on disk under `/opt/ai-inference/raw-data/sentinel/`).
   A 30-day age guard prunes at startup and gates per-article.
2. **Extraction** (production = **GPU JSON-CoD**, `Extraction__Backend=VllmJson`):
   prompt+schema from host mount `/prompts/cod/` (hot-reloaded) → vLLM
   `response_format=json_schema` → truncation salvage (brace-depth recovery; distinguishes
   `truncated_salvaged` from `salvage_empty`) → `dsl-parser-mcp` `/parse_json` lifts JSON to a
   `DocumentAst` → deterministic `/verify` span verifier (hard-failed blocks dropped; 12-25% of
   articles is the normal baseline — a quality gate, not an error) → adapter persists
   observations with sentence-snapped text quotes. Dispatch is continuous streaming with
   `MaxConcurrent=8`; an `ObservationValueSanitizer` nulls un-storable values (|v| ≥ 1e16,
   NaN/Inf) rather than clamping.
3. **Resolution cascade** (per observation): LLM candidate-pick (≥0.7) → SecMaster hybrid
   resolve → Gemini fallback (≥0.85, auto-registers new instruments) → Pending; an async
   `ResolutionWorker` drains Pending via Finnhub (5 attempts → NoResolution), with a nightly
   AlphaVantage sweep (25/day quota) behind it. A per-article `EntityResolutionPrepass` (spaCy
   NER → SecMaster resolve-entities) grounds prompts and sector tags — it never gates entry.
4. **Signal classification**: one structured vLLM call per article → `macro_observations`
   `:sig:` rows (the matrix feed).
5. **Digest** (Quartz, America/New_York: daily 07:00 Mon-Fri, weekly Fri 19:00, monthly
   last-day 19:00): observations → themes (signal-derived via `SignalThemeMap`) → stats →
   sector breakdown → news momentum (early/late tilt split) → cross-collector rollup →
   LLM narrative (aggregate-first; token budget derived from the model context, shed-loop
   drops lowest-ranked articles) → render (Chart.js theme radar) → ntfy push to `atlas-digest`.
   Every enrichment degrades rather than failing the run — Tier-1 always ships.

AutoApprove is on: approve at extraction ≥0.9 ∧ resolution ≥0.8 ∧ InstrumentId; reject below
0.5 extraction; else Pending.

## 6. SecMaster — identity, classification, resolution

Live `atlas_secmaster`: 16,580 instruments (10,054 Equity, 4,998 Indicator, 797 Economic, 350
quarantined LoRA-era registrations), 7,994 source_mappings, 16,034 bge-m3 embeddings, 11
atlas_sectors, 84 signal_identities in 7 categories (macro 33, rate 14, credit 13, liquidity 9,
instrument 7, commodity 4, fx 4).

Core invariants:

- **Identity ⊥ collection**: an instrument with zero source mappings still resolves with
  identity+sector+NAICS (with a Warning). `NotFound` means "exhausted all machinery", never
  "not in table".
- **Sector gate is Equity-only**: ~87% of the catalog legitimately has null sector.
- **Lazy-load**: the catalog self-seeds from entities seen in articles; bulk-preload is
  forbidden by design.

Three distinct entry paths (do not conflate): `resolve-entities` (HTTP; Sentinel NER surfaces →
instrument + NAICS + sector grounding), `ResolveBatch` (gRPC; ThresholdEngine symbol→best
active source mapping, pattern load time only), `register` (gRPC; collector fire-and-forget,
with a trusted-macro-collector guard on Economic asset classes).

**Resolution cascade** ("fuzzy proposes, authoritative confirms"): local exact (0.95) / fuzzy
(0.85) / vector (0.75) → SecmasterDirect 0.95 → EDGAR 0.90 → OpenFIGI 0.85-0.90 → signal-alias
0.90 → upstream discovery proposes → confirmation cascade OpenFIGI → Finnhub → Gemini (0.85).
Confirmed → persist + embed; unconfirmed → null + review queue. A non-overlapping article
context zeroes a candidate's score (dropped, not down-ranked, below MinConfidence 0.8).

The synchronous hot path is the **hybrid cascade**: ExactSQL → FuzzySQL → pgvector cosine
(bge-m3, hard similarity floor 0.5) → RAG hypothesis tier on llama-cpu-rag (25s budget,
abstains under 12s remaining budget, 4 concurrent slots, circuit breaker, no retry on
generation). Any RAG failure degrades to deterministic-tier candidates — never a 500.

## 7. The data fleet (collectors, live scope)

| Collector | Live scope | Notable absences (by design / dead schema) |
|---|---|---|
| **FRED** | 80 series configs (79 active, 55 signal-tagged); per-frequency Quartz crons; dual-writes macro-classified series to the substrate; two backfills (BackfillService advances `LastCollectedAt`; AlfredBackfillService deliberately does not) | ObservationChannel has no reader (memory-growth hazard); 0/80 sector-tagged |
| **Finnhub** | 18 active Quote series — SPY/QQQ/IWM/VTI/DIA, all 11 SPDR GICS sector ETFs, GLD/TLT; 1-min collection loop; token bucket 60/min | candles/social/insider/calendar tables exist with zero writers — permanently empty; `GetLatestEventTime` returns UtcNow on empty |
| **AlphaVantage** | 5 Commodity (scalar) series; 4h cycle, ≤4 per cycle; in-memory quota (25/day, resets on restart) | OHLCV schema unused; TechnicalIndicator type is a scaffold (never collected); gRPC stream is scalar-only, events carry no values |
| **OFR** | FSI daily 10:00 UTC, STFM daily 14:00 UTC, HFM Fri 18:00 UTC; 109 STFM + 336 HFM series | FSI excluded from registration and the substrate; macro dual-write wired but 0 series flagged |
| **Nasdaq** | **not running** (compose block commented out; NDL WAF) | events synthesized on read (EventId = fresh Ulid per read — not a dedup key) |
| **CalendarService** | FRED releases (allow-list ~18), NYSE market calendar computed in-memory | no gRPC; Finnhub calendar worker disabled; fabricated event times (DST-unaware) |

## 8. Database layout (TimescaleDB 2.23.1 / pg16)

- **Databases**: `atlas_data` (collectors + matrix), `atlas_secmaster`, `calendar_data`
  (+ `*_integration_test`).
- **atlas_data public**: `series_configs`, `events`/`processed_events`, per-collector tables
  (`fred_observations`, `alphavantage_*`, `nasdaq_*`, `ofr_*`, `finnhub_*`), matrix layer
  (`macro_observations`, `matrix_cells`, `sector_regimes`), `threshold_alerts`,
  `threshold_events`.
- **atlas_data sentinel schema**: `raw_content`, `extracted_observations` (+ `_shadow`),
  `digest_reports`, `rss_feeds`, `search_queries`, `validation_*`, `retention_policies`
  (0 rows — unused mechanism; pruning is app-managed via the 30-day raw_content age guard +
  startup pruner, §5).
- **Hypertables** — exactly five: `nasdaq_observations`, `alphavantage_observations`,
  `macro_observations`, `matrix_cells`, `sector_regimes`. Everything else (including
  `fred_observations`, `finnhub_quotes`) is a plain table. No Timescale retention policies
  are active (by design — retention is app-managed where it exists, see above).
- **atlas_secmaster**: `instruments`, `aliases`, `instrument_embeddings` (pgvector),
  `signal_identities`, `source_mappings`, `atlas_sectors`, `naics_*`, `edgar_filers`,
  `openfigi_lookup_cache`, review queue. Extensions: pg_trgm + pgvector.
- **Migrations**: EF Core, app-managed — each service migrates its own schema on startup;
  `macro_observations` via the one-shot migrator container. Ansible only creates
  DBs/users/extensions idempotently. No raw SQL schema scripts.

## 9. Deployment machinery (`deployment/`)

See `deployment/README.md` for the full reference. The essentials:

- **Everything flows through ansible** (`ansible-playbook playbooks/deploy.yml --tags {svc}`).
  Every run deletes and re-templates `/opt/ai-inference/compose.yaml` from
  `deployment/artifacts/compose.yaml.j2` — **direct edits to the live compose file are always
  clobbered**.
- **ZFS snapshots** of timeseries+dashboard datasets precede each deploy; the rollback recipe
  is written to `/opt/ai-inference/last-snapshot.txt`.
- **Restart semantics**: FULL (default) restarts the whole stack when the template or an image
  changed — and resurrects manually-stopped services (notably alert-service). SCOPED
  (`-e "scoped_restart=true scoped_services='…'"`) recreates only the named main-compose
  services. A freshness gate asserts every running service's image == local `:latest`.
- **Hot-reload tags**: `dashboards`/`alerting` (Grafana auto-provisions, Prometheus SIGHUP),
  `patterns` (ThresholdEngine hot-reload API), `sentinel-prompts`/`cpu-cod-prompts` (rsync repo
  → `/opt/ai-inference/prompts/`, overwrites host edits; sentinel watches the mount — no
  rebuild needed).
- **AutoFix pipeline** (host-side): alert-service writes alert JSON to
  `/opt/ai-inference/autofix-queue` → `autofix-runner.timer` (60s) spawns `claude --print`
  per alert (defers while an interactive claude session exists) → `autofix-watcher.timer`
  (5 min) deploys merged AutoFix PRs (skipped while an interactive session is live).
  `merged-pr-watcher.timer` (same family, human-merged PRs) is installed but **intentionally
  not enabled** — the operator enables it post-review.
- **Git gates** (`.claude/hooks/`): pushes to main are blocked (docs-extension allowlist only);
  feature-branch pushes require a tests-passed marker keyed on the git **tree** hash; `gh pr
  merge` requires a review marker keyed on the GitHub headRefOid.

## 10. Observability

Pipeline: services emit OTLP → otel-collector:4317 → Loki (logs, 30d), Tempo (traces, 7d),
Prometheus (metrics, 15s scrape). Grafana :3000 with provisioned datasources (Prometheus, Loki,
Tempo, TimescaleDB, CalendarDB).

Alert routing: Prometheus rules + Loki ruler → Alertmanager (severity routes critical /
warning / info; critical inhibits warning per service) → alert-service webhook → ntfy
`atlas-alert`, Gmail, AutoFix queue. **AlertService is UP by design** (verified end-to-end
2026-06-10).

Alert rule files (`deployment/artifacts/monitoring/alerts/`): `service-health.yml`,
`collectors-deadman.yml` (dead-man windows sized past the longest normal idle gap),
`fredcollector.yml`, `thresholdengine.yml` (incl. the matrix projector/persistence guards),
`sentinel.yml` (extraction, digest, resolution, classifier guards), `calendarservice.yml`,
plus Loki log-error rules.

Dashboards are folder-provisioned from `/opt/ai-inference/monitoring/dashboards/`: Overview,
Collectors & Services, Sentinel Pipeline, Signal Matrix, Calendar & Reports.

## 11. Known dead paths and seams (do not "fix" casually)

- ThresholdEngine `ObservationEventSubscriber` + `CellProjector` formula: unwired dead code.
- Finnhub non-Quote schema, AlphaVantage OHLCV/TechnicalIndicator: dead schema/scaffold.
- OFR `is_macro` flags all off → no OFR substrate rows yet.
- SecMaster proto `ALIAS_MATCH` never emitted; `RagStrictMode` true in code but false in prod
  via compose env.
- `sentinel_ollama_*` metric names: legacy seam now recorded by the vLLM client.
- `financial_news` database: legacy, pending orphan-cleanup drop.
- CPU CoD path (`llama-server` + GBNF): compiled-in, inert, rollback-only.
