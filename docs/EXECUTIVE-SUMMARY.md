# ATLAS — Executive Summary

**Last verified against the live system: 2026-06-11.**

ATLAS is a self-hosted financial-signal platform running on a single production server
(mercury). It continuously collects hard economic data (FRED, OFR, Finnhub, AlphaVantage),
ingests financial news (RSS + search via a Cloudflare edge worker), extracts structured
observations from that news with a local 32B LLM, and fuses both worlds into a single derived
dataset — the **signal matrix** — alongside threshold alerting, daily digests, and scheduled
reports. Everything runs in containers under nerdctl/containerd, deployed by ansible, observed
through an OTEL → Prometheus/Loki/Tempo → Grafana stack.

## The signal matrix

The matrix is the system's centerpiece: a continuously re-projected table of **signal × sector
contributions** (`matrix_cells`, TimescaleDB). Every macro/market signal — whether it arrived
as a FRED data point or as the tilt of a news article — is projected onto the 11 ATLAS sectors
(ENERGY … REAL_ESTATE) as a value in [-3, +3], every 5 minutes, with full multiplicative
provenance (magnitude × source trust × freshness × temporal × confidence × sector weight) on
every row.

Two properties define it:

- **One funnel**: all matrix input flows through the `macro_observations` substrate table.
  Sentinel news writes `tilt × confidence` per classified signal; FRED dual-writes its
  macro-classified series; OFR is wired but dormant (0 series flagged). ThresholdEngine's
  `ObservationCellProjector` polls that table — the live gRPC event path never writes cells.
- **Sources coexist, never overwrite**: cells are keyed per source collector, so FRED and news
  contributions to the same signal sit side by side. Official data carries trust 1.0, named
  news 0.7, unnamed scrapes 0.4 — official data outvotes headlines, but sustained multi-source
  news consensus remains visible against it. News magnitudes decay with a 24h half-life and
  self-heal toward zero when coverage stops.

Live scale (2026-06-11): ~24.3k cells over 30 distinct signals, fed by ~4.2k macro
observations, growing by ~700 observations/day. Consumption: ThresholdEngine HTTP + MCP APIs,
four Signal Matrix Grafana dashboards, and the `sector_phase_cells` downstream view.
Full detail: [MATRIX.md](./MATRIX.md).

## The Sentinel news pipeline

SentinelCollector turns headlines into matrix input and human-readable digests:

- **Ingestion**: DB-managed RSS feeds + scheduled SearXNG queries → Cloudflare edge worker →
  `raw_content` (30-day retention guard).
- **Extraction**: GPU vLLM JSON Chain-of-Density (`Qwen2.5-32B-Instruct-AWQ`,
  `response_format=json_schema`), continuous streaming at 8 concurrent articles; a
  deterministic sidecar parses and span-verifies every emitted block (12-25% of articles
  failing verification is the normal quality-gate baseline). The former CPU GBNF/DSL path is
  retained as rollback only.
- **Resolution**: spaCy NER prepass + SecMaster's "fuzzy proposes, authoritative confirms"
  cascade (local SQL/vector tiers → OpenFIGI/Finnhub/Gemini confirmation), with async Finnhub
  and nightly AlphaVantage sweeps behind it.
- **Signal classification**: one structured GPU call per article scores tilt and confidence
  against a 77-signal catalog — this is the matrix's news feed.
- **Digest**: daily/weekly/monthly (Quartz, ET) — themes, sector breakdown, news momentum
  (early-vs-late tilt), cross-collector rollup, and an aggregate-first LLM narrative; pushed
  via ntfy. Every enrichment degrades gracefully; the Tier-1 digest always ships.

## The inference topology

All inference is local; no ollama remains (retired 2026-06-11):

- **GPU**: `vllm-server` (main-compose service since 2026-06-11, RTX 5090) serving Qwen2.5-32B-Instruct-AWQ at
  32K context — extraction, news-signal classification, report narratives.
- **CPU** (llama.cpp on pinned core islands): `llama-server` (30B, rollback extraction
  backend), `llama-cpu-rag` (7B, SecMaster RAG generation), `llama-cpu-embed` (bge-m3
  embeddings for the pgvector instrument catalog).
- Side models: faster-whisper (transcription), FinBERT (sentiment), spaCy NER.

## The data fleet

| Source | Live scope |
|---|---|
| FRED | 80 series (79 active); the matrix's hard-data feed via macro dual-write |
| Finnhub | 18 quote series: core index ETFs + all 11 SPDR sector ETFs + GLD/TLT, ~1-min cadence |
| OFR | FSI daily + 109 STFM + 336 HFM series (financial-stress / money-market / hedge-fund) |
| AlphaVantage | 5 commodity series (25 calls/day free tier) |
| Nasdaq NDL | disabled in production (WAF) |
| CalendarService | FRED release calendar (curated ~18 releases) + NYSE market calendar |

SecMaster is the identity backbone: 16.6k instruments, 84 signal identities, embeddings-backed
resolution, lazy-loaded from what the news actually mentions (bulk preload is forbidden by
design).

## Operations

- **Deployment**: ansible-only; the live compose file is regenerated every run. ZFS snapshots
  before each deploy; scoped restarts for surgical changes; prompt/dashboard/pattern updates
  hot-reload without rebuilds.
- **Alerting**: Prometheus + Loki rules → Alertmanager → AlertService → ntfy/email, plus an
  **AutoFix** pipeline that spawns a Claude session per actionable alert and auto-deploys
  merged fixes (deferring while an interactive session is live).
- **Observability**: OTEL everywhere; per-domain dashboards (Overview, Collectors, Sentinel
  Pipeline, Signal Matrix, Calendar & Reports); dead-man alerts sized to each collector's
  natural cadence.

## Where to go deeper

- [ARCHITECTURE.md](./ARCHITECTURE.md) — the full system as built.
- [MATRIX.md](./MATRIX.md) — the signal matrix deep-dive.
- [SENTINEL-RLM.md](./SENTINEL-RLM.md) — extraction model/backend/VRAM constraints.
- `deployment/README.md` — deployment machinery reference.
- Per-service `README.md` / `AGENT_README.md` — service-level truth.
