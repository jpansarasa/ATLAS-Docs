# ATLAS

Self-hosted platform for financial data collection, pattern evaluation, regime detection, and LLM-based document extraction.

This file is the project **index of truth**: where each service lives, what it does, and which document to read for deeper detail. It deliberately avoids duplicating engineering rules (see [CLAUDE.md](./CLAUDE.md)), architectural decisions (see [docs/ARCHITECTURE.md](./docs/ARCHITECTURE.md)), or framework methodology (see [docs/EXECUTIVE-SUMMARY.md](./docs/EXECUTIVE-SUMMARY.md)).

## What ATLAS Does

- Ingests economic, market, and sentiment data from FRED, Alpha Vantage, Nasdaq, Finnhub, OFR, and unstructured-document sources.
- Evaluates configurable patterns (Roslyn-compiled C# expressions) against incoming observations.
- Detects regime transitions across a multi-state machine (Crisis → Growth) and emits structured signals.
- Delivers alerts via ntfy + email, exposes data via REST/gRPC, and exposes MCP servers for Claude Code integration.
- Runs an LLM-based extraction pipeline (SentinelCollector) for unstructured financial documents.

Stack: .NET 10 / C# 14, TimescaleDB, containerd + nerdctl compose, OpenTelemetry → Loki / Prometheus / Tempo → Grafana, Serilog, Polly.

## Service Inventory

All services run on a single host (`mercury`) via `nerdctl compose`, deployed by Ansible from this monorepo. See [deployment/README.md](./deployment/README.md) for how they are built and shipped.

### Data Collectors

| Service | Role | Docs |
|---|---|---|
| FredCollector | FRED economic series (rates, employment, GDP, etc.) | [README](./FredCollector/README.md) |
| AlphaVantageCollector | WTI / Brent / Natural Gas prices | [README](./AlphaVantageCollector/README.md) |
| NasdaqCollector | LBMA gold price (AM / PM fixing) | [README](./NasdaqCollector/README.md) |
| FinnhubCollector | Equity quotes, news sentiment, economic calendar | [README](./FinnhubCollector/README.md) |
| OfrCollector | OFR financial stability data (FSI, STFM, HFM) | [README](./OfrCollector/README.md) |
| SentinelCollector | LLM-based extraction from unstructured documents | [README](./SentinelCollector/README.md) |
| CalendarService | Market holidays, trading-day validation, event schedules | [README](./CalendarService/README.md) |

### Core Services

| Service | Role | Docs |
|---|---|---|
| SecMaster | Centralized instrument metadata, source resolution, fuzzy + semantic search | [README](./SecMaster/README.md) |
| ThresholdEngine | Pattern evaluation, regime detection, macro scoring, signal emission | [README](./ThresholdEngine/README.md) |
| AlertService | Severity-routed notification dispatch (ntfy + SMTP) | [README](./AlertService/README.md) |
| MacroSubstrate | Shared macro context substrate (library + migrator) | [src](./MacroSubstrate/) |
| Reports | Daily / weekly / monthly report hosts built on the substrate | [src](./Reports/) |

### AI / Auxiliary Services

| Service | Role | Docs |
|---|---|---|
| WhisperService | Audio transcription via faster-whisper (YouTube ingestion) | [README](./WhisperService/README.md) |
| FinBertSidecar | Financial-text embeddings consumed by SecMaster | [README](./FinBertSidecar/README.md) |
| LlmBenchmark | Extraction-pipeline benchmarking harness | [BENCHMARKS](./LlmBenchmark/BENCHMARKS.md) |

### MCP Servers (Claude Code integration)

MCP servers are SSE endpoints in the 3100-range; each lives next to (or inside) its parent service.

| Server | Parent | Path |
|---|---|---|
| FredCollectorMcp | FredCollector | [FredCollector/mcp](./FredCollector/) |
| ThresholdEngineMcp | ThresholdEngine | [ThresholdEngine/mcp](./ThresholdEngine/) |
| FinnhubMcp | FinnhubCollector | [FinnhubCollector/mcp](./FinnhubCollector/) |
| OfrMcp | OfrCollector | [OfrCollector/mcp](./OfrCollector/) |
| SecMasterMcp | SecMaster | [SecMaster/mcp](./SecMaster/) |
| WhisperServiceMcp | WhisperService | [WhisperService/mcp](./WhisperService/) |
| markitdownMCP | standalone | [README](./markitdownMCP/README.md) |
| ntfy-mcp | standalone | [README](./ntfy-mcp/README.md) |
| gemini-resolver-mcp | standalone | [src](./gemini-resolver-mcp/) |
| dsl-parser-mcp | inside SentinelCollector | [SentinelCollector](./SentinelCollector/) |

### Shared Code & Infrastructure

| Path | Role | Docs |
|---|---|---|
| [Events/](./Events/) | gRPC contracts (`.proto`) + typed clients + in-memory event records | [README](./Events/README.md) |
| [deployment/](./deployment/) | Ansible playbooks, compose templates, monitoring configs, ZFS snapshots | [README](./deployment/README.md) |
| [edge/](./edge/) | Cloudflare Workers (edge-deployed components) | [README](./edge/README.md) |
| [scripts/](./scripts/) | Repo-level utility scripts | [README](./scripts/README.md) |
| [docs/](./docs/) | Architecture, methodology, plans, research notes | see below |

## Documentation Map

| Document | What it covers |
|---|---|
| [CLAUDE.md](./CLAUDE.md) | Project conventions (compose vs. docker-compose, deploy rules, EF migration rules, Sentinel sizing, git-push gate) — read this before generating code |
| [STATE.md](./STATE.md) | Current epic / phase status |
| [docs/README.md](./docs/README.md) | **Curated index of `docs/`** — what each doc is for, what was retired to git history |
| [docs/EXECUTIVE-SUMMARY.md](./docs/EXECUTIVE-SUMMARY.md) | What ATLAS is today: the signal matrix, Sentinel pipeline, inference topology, data fleet |
| [docs/ARCHITECTURE.md](./docs/ARCHITECTURE.md) | The system as built: stacks, services, data flows, DB layout, deployment, observability |
| [docs/MATRIX.md](./docs/MATRIX.md) | Signal-matrix deep-dive: cells, invariants, feeds, formula/decay/source-trust, consumption |
| [docs/GRPC-ARCHITECTURE.md](./docs/GRPC-ARCHITECTURE.md) | gRPC streaming patterns and contracts |
| [docs/OBSERVABILITY.md](./docs/OBSERVABILITY.md) | OTEL pipeline, dashboards, tracing conventions |
| [docs/THRESHOLDENGINE-PATTERNS.md](./docs/THRESHOLDENGINE-PATTERNS.md) | Pattern catalog and category weights |
| [docs/SENTINEL-RLM.md](./docs/SENTINEL-RLM.md) | Sentinel extraction model/backend/VRAM constraints (pipeline detail: `SentinelCollector/README.md`) |
| [docs/RELEASES.md](./docs/RELEASES.md) | Phase / epic outcomes (paired with git tags) + doc-retirement recovery pointers |
| [docs/FRED_SERIES_REFERENCE.md](./docs/FRED_SERIES_REFERENCE.md) | FRED series catalog |
| [docs/Guide to Grafana Dashboards.md](./docs/Guide%20to%20Grafana%20Dashboards.md) | Dashboard authoring guide |

## Conventions (Index)

Authoritative rules live in [CLAUDE.md](./CLAUDE.md). Quick pointers:

- **Compose / container files**: `compose.yaml` and `Containerfile` (not `docker-compose.yml` / `Dockerfile`). [CLAUDE.md → PROJECT_CONVENTIONS](./CLAUDE.md)
- **Deployments**: always via `ansible-playbook playbooks/deploy.yml --tags {service}`; never edit `/opt/ai-inference/compose.yaml` directly. [CLAUDE.md → DEPLOYMENT](./CLAUDE.md), [deployment/README.md](./deployment/README.md)
- **Build & test before push**: `{Project}/.devcontainer/compile.sh` must pass (0 errors, 0 warnings, all tests) before `git push`. Enforced by `.claude/hooks/git-push-guard.sh`. [CLAUDE.md → GIT_PUSH](./CLAUDE.md)
- **Database schema**: EF Core migrations only; no raw SQL scripts; seed via `HasData()` or app startup. [CLAUDE.md → DATABASE](./CLAUDE.md)
- **Sentinel extraction**: ≥30B-parameter models, 32K context; GPU inference via vLLM, CPU via llama.cpp (no ollama in the topology). [CLAUDE.md → SENTINEL](./CLAUDE.md)
- **Container image naming**: kebab-case `{service-name}:latest` (e.g. `fred-collector`, not `fredcollector`). [CLAUDE.md → CONTAINER_BUILD](./CLAUDE.md)
- **Service README template**: [.claude/skills/readme-consistency/TEMPLATE.md](./.claude/skills/readme-consistency/TEMPLATE.md).

## Getting Started

This repo deploys to a single host (`mercury`). The supported workflow is:

```bash
# Full deploy (templates, builds, systemd, db init)
ansible-playbook playbooks/deploy.yml

# Single service
ansible-playbook playbooks/deploy.yml --tags fred-collector

# Smoke tests
ansible-playbook playbooks/smoke-test.yml
```

Run from `deployment/ansible/`. Full tag list, vault usage, and ZFS snapshot procedures: [deployment/README.md](./deployment/README.md).

For per-service local development (Dev Containers), see each service's README.

## Where to Look for What

| Question | Start here |
|---|---|
| "What does this service do?" | Its README in the service directory |
| "How is the system wired together?" | [docs/ARCHITECTURE.md](./docs/ARCHITECTURE.md) |
| "What event contract should I publish/consume?" | [Events/README.md](./Events/README.md) |
| "How do I deploy / roll back?" | [deployment/README.md](./deployment/README.md) |
| "What rules must my code change follow?" | [CLAUDE.md](./CLAUDE.md) |
| "What's the current epic state?" | [STATE.md](./STATE.md) |
| "Which patterns are configured?" | [docs/THRESHOLDENGINE-PATTERNS.md](./docs/THRESHOLDENGINE-PATTERNS.md) |
| "What's the macro framework / scoring methodology?" | [docs/EXECUTIVE-SUMMARY.md](./docs/EXECUTIVE-SUMMARY.md) |
| "How is observability wired?" | [docs/OBSERVABILITY.md](./docs/OBSERVABILITY.md) |

---

**Status**: Production. Personal portfolio-management system; not investment advice. Proprietary, personal use only — external contributions are not accepted at this time.
