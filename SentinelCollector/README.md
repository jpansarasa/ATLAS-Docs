# SentinelCollector

Alternative data collection and CPU-based DSL extraction service for ATLAS.

## Overview

SentinelCollector collects unstructured web content from SearXNG news search, RSS feeds, and Cloudflare edge workers, then extracts economic observations via a CPU-based Chain-of-Density (CoD) DSL pipeline. Extracted observations are resolved against SecMaster (with Finnhub + AlphaVantage + Gemini fallbacks) and streamed to ThresholdEngine over gRPC. The service also runs a digest generator, a browser-based review UI, validation-trigger machinery, and a Phase 5.5 matrix-cell enrichment stream.

## Architecture

```mermaid
flowchart LR
    subgraph Sources
        SX[SearXNG]
        RSS[RSS Feeds]
        EW[Cloudflare<br/>Edge Workers]
        VT[Validation<br/>Triggers]
    end

    subgraph SentinelCollector
        CW[Collection<br/>Workers]
        NZ[Content<br/>Normalizer]
        EX[CPU CoD<br/>Extraction]
        RES[Resolution<br/>Worker]
        DG[Digest<br/>Service]
        PUB[Event<br/>Publishers]
    end

    subgraph External
        LLAMA[llama-server<br/>CPU]
        DSL[dsl-parser-mcp]
        MD[markitdown-mcp]
        TRF[trafilatura]
        SPACY[spacy-ner]
        SM[SecMaster]
        FH[FinnhubCollector]
        AV[AlphaVantageCollector]
        GEM[Gemini<br/>resolver-mcp]
    end

    subgraph Consumers
        TE[ThresholdEngine]
        NTFY[ntfy]
    end

    SX --> CW
    RSS --> CW
    EW --> CW
    VT --> CW
    CW --> DB[(TimescaleDB)]
    CW --> NZ
    NZ --> MD
    NZ --> TRF
    NZ --> EX
    EX --> LLAMA
    EX --> DSL
    EX --> SPACY
    EX --> SM
    EX --> RES
    RES --> FH
    RES --> AV
    RES --> GEM
    EX --> PUB
    PUB -->|gRPC ObservationEventStream| TE
    PUB -->|gRPC MatrixUpdateStream| TE
    DG --> NTFY
```

Collection workers ingest from multiple sources, normalize HTML/PDF via markitdown + trafilatura, extract structured DSL observations against a host-mounted GBNF grammar on a CPU llama.cpp server (parsed/verified by the `dsl-parser-mcp` sidecar), drive a separate `ResolutionWorker` through the SecMaster → Finnhub → AlphaVantage → Gemini cascade, then publish events for ThresholdEngine over gRPC.

Schema is owned by EF Core migrations under `src/Data/Migrations/`. Database migrations + seed data (search queries, RSS feeds, validation triggers) are applied on startup.

## Features

- **SearXNG Collection**: Database-driven search queries with `Daily`, `Weekly`, and `MarketTimes` schedule types
- **RSS Feed Collection**: ~50 seeded financial-news / central-bank feeds with per-feed poll intervals
- **Edge Sync**: Pulls captured content from a Cloudflare worker for low-latency web capture
- **Validation Triggers**: Search-template triggers fired on data releases (e.g. GDP) or threshold crossings (e.g. Sahm-rule)
- **Content Normalization**: HTML/PDF → markdown via `markitdown-mcp` + `trafilatura` pre-wash
- **CPU CoD DSL Extraction**: Grammar-constrained CoD against `llama-server` on CPU; parsed + verified by `dsl-parser-mcp` (`Extraction__Backend=LlamaServerDsl`)
- **CoVe + Chain of Density (legacy V1/V2 paths)**: `Ollama` / `LlamaServer` / `VllmServer` backends remain in `ExtractionOptions` and still drive shadow runs; production `IMergedExtractionService` binding routes to the CPU DSL path post-PR #510
- **Tool-Augmented CoVe / Epistemic Markers** (opt-in): `Extraction__UseToolAugmented*` flags drive a sandbox-manager-backed code-execution loop
- **Entity Resolution Pre-pass**: spaCy NER sidecar → SecMaster `/api/resolve-entities` grounding for the V2 pipeline
- **SecMaster Resolution Cascade**: `ResolutionWorker` (Finnhub → SecMaster), nightly AlphaVantage sweep, optional Gemini fallback
- **Hallucinated-symbol Quarantine**: Operator endpoint walks Approved rows and quarantines `Symbol` values absent from `text_quote` + `context_summary`
- **AutoApprove Backfill**: Replays `AutoApprovePolicy` against the historical Pending backlog
- **Re-extract Background Service**: Optional one-shot recovery for legacy rows; supports `re-extract` and `resolve-only` modes with live-queue backpressure
- **Review UI**: Server-rendered HTML queue at `/ui/review` (approve/reject/skip + inline corrections)
- **Digest Service**: Daily / Weekly / Monthly digests with HTML / Markdown / JSON variants and ntfy push
- **Event Streaming (gRPC)**: `ObservationEventStream` for observation events + `MatrixUpdateStream` for Phase 5.5 matrix-cell enrichments
- **OpenTelemetry**: Traces (`SentinelCollector.*` activity sources), metrics (`SentinelCollector.*` meters incl. `EntityResolver`), and Serilog → OTLP logs

## Configuration

Selected env vars (full list in `src/Configuration/*Options.cs`). Double-underscore (`__`) maps to colon-section nesting (`Extraction:Backend`).

| Variable | Description | Default |
|----------|-------------|---------|
| `ConnectionStrings__AtlasDb` | PostgreSQL connection string | **Required** |
| `Kestrel__HttpPort` / `Kestrel__GrpcPort` | Bound listen ports | `8080` / `5001` |
| `Extraction__Backend` | `Ollama` \| `LlamaServer` \| `VllmServer` \| `LlamaServerDsl` | `Ollama` (code default); production sets `LlamaServerDsl` |
| `Extraction__Model` | LLM model id (passed verbatim to backend) | `llama3.1:70b-instruct-q4_K_M` (code) / `Qwen/Qwen2.5-32B-Instruct-AWQ` (prod) |
| `Extraction__OllamaEndpoint` | Ollama endpoint | `http://ollama-gpu:11434` |
| `Extraction__LlamaServerEndpoint` | llama.cpp server endpoint | `http://llama-server:8080` |
| `Extraction__VllmEndpoint` | vLLM endpoint (legacy V1/V2 path) | `http://vllm-server:8000` |
| `Extraction__ContextWindowSize` | LLM context window | `32768` |
| `Extraction__MaxConcurrentExtractions` | Must match backend `NUM_PARALLEL` / scheduler limit | `1` (prod sets `4`) |
| `Extraction__PromptDirectory` | Host-mounted prompt directory | `/prompts` |
| `Extraction__WatchPromptFiles` | Hot-reload host prompt edits | `true` |
| `Extraction__AutoApproveEnabled` | Run `AutoApprovePolicy` at extraction time | `false` (prod: `true`) |
| `Extraction__UseV2Pipeline` / `__ShadowMode` / `__V2EnabledSources__N` | V2 pipeline routing + shadow runs | see prod compose |
| `Extraction__GeminiResolverEnabled` / `__GeminiFallbackEnabled` | V2 Rule 2.5 + V1 Phase 7 Gemini fallback | `false` |
| `CpuCod__DslParserEndpoint` | `dsl-parser-mcp` sidecar URL | `http://dsl-parser-mcp:3120` |
| `CpuCod__PromptDirectory` / `__PromptFile` / `__GrammarFile` | CoD prompt + GBNF grammar (host-mounted) | `/prompts/cod` / `cod_dsl_v8_baseline.txt` / `cod-dsl-v2.3.gbnf` |
| `CpuInference__Endpoint` / `__Model` / `__Enabled` | CPU ollama for CoD summarization + epistemic markers | `http://ollama-cpu:11434` / `qwen2.5:7b-instruct` / `true` (prod: `false`) |
| `EdgeSync__Endpoint` / `__ApiKey` | Cloudflare worker sync endpoint | **Required** |
| `EdgeSync__BatchSize` / `__PollIntervalSeconds` | Edge sync batch size / poll interval | `100` / `300` |
| `Searxng__Endpoint` | SearXNG instance URL | `https://searxng.elasticdevelopment.com` |
| `Searxng__Engines` / `__TimeRange` / `__MaxResultsPerQuery` | SearXNG query knobs | `bing news,brave.news` / `Day` / `20` |
| `Searxng__ScheduleTimes` | US/Eastern HH:mm collection times | `["09:30","12:00","16:00","22:00"]` |
| `SecMaster__Endpoint` / `__TimeoutSeconds` | SecMaster REST endpoint | `http://secmaster:8080` / `30` (prod: `180`) |
| `SECMASTER_GRPC_ENDPOINT` | SecMaster gRPC endpoint (registration / resolution) | `http://secmaster:5001` |
| `Markitdown__Endpoint` / `__SharedTempDirectory` | markitdown MCP + cross-container temp dir | `http://markitdown-mcp:3102` / `/opt/ai-inference/raw-data/sentinel/tmp` |
| `Trafilatura__Endpoint` / `__Enabled` | trafilatura HTML pre-wash | `http://trafilatura:3109` / `true` |
| `SpacyNer__Endpoint` / `__Enabled` / `__MinNerConfidence` | spaCy NER sidecar for entity pre-pass | `http://spacy-ner:3110` / `true` / `0.8` |
| `ResolutionWorker__FinnhubCollectorEndpoint` / `__MaxResolutionAttempts` | Finnhub-backed resolution loop | `http://finnhub-collector:8080` / `5` |
| `AlphaVantageSweep__AlphaVantageCollectorEndpoint` / `__CronExpression` | Nightly AV sweep | `http://alphavantage-collector:8080` / `0 0 4 * * ?` |
| `ReExtract__Enabled` / `__Mode` / `__Cohort` / `__RowsPerMinute` / `__RowsPerMinuteResolveOnly` | Legacy-row re-extract job | `false` / `re-extract` / `all` / `30` / `600` |
| `RawLayer__BasePath` / `__RetentionDays` | Raw-content filesystem layer | `/opt/ai-inference/raw-data/sentinel` / `90` |
| `R2__Endpoint` / `__AccessKeyId` / `__SecretAccessKey` / `__BucketName` | Cloudflare R2 raw-content offload | empty / empty / empty / `sentinel-raw` |
| `Digest__PublicBaseUrl` / `__ConfidenceFloor` / `__Ntfy__*` | Digest URL prefix, narrative floor, ntfy push | `http://localhost:5091` / `0.5` / see options |
| `MacroSignalIdentityCatalog__Categories` | SecMaster category allowlist for alias warmup | `["macro","rate"]` |
| `OpenTelemetry__OtlpEndpoint` / `__ServiceName` / `__ServiceVersion` | OTLP collector + resource attrs | `http://otel-collector:4317` / `sentinel-collector` / `1.0.0` |

## API Endpoints

REST surface lives under `src/Endpoints/`:

- **AdminEndpoints** — `/admin/*` admin/inspection/CRUD APIs.
- **ReviewUiEndpoints** — `/ui/review` browser-based human-review UI.
- **DigestEndpoints** — `/sentinel/digests/*` digest list / view / generate / resend.

### REST API (Port 8080)

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/admin/stats` | GET | Collection + extraction counts |
| `/admin/sources` | GET | Sources with row counts and last collection time |
| `/admin/recent` | GET | Recent raw content (`?limit`) |
| `/admin/observations` | GET | Recent extracted observations (`?limit`) |
| `/admin/reprocess` | POST | Reprocess raw_content ids (deletes pending observations, clears `processed_at`) |
| `/admin/queries` | GET/POST | List / create SearXNG search queries |
| `/admin/queries/{id}` | GET/PUT/DELETE | Get / update / delete |
| `/admin/queries/{id}/toggle` | POST | Toggle `enabled` |
| `/admin/retention` | GET/POST | List / create retention policies (raw_contents, extracted_observations, events) |
| `/admin/retention/{id}` | GET/PUT/DELETE | Get / update / delete |
| `/admin/review` | GET | Review queue (filter by `status`, `resolutionState`, `maxConfidence`, paginated) |
| `/admin/review/stats` | GET | Review-status + resolution-state distributions |
| `/admin/review/{id}` | GET | Single observation with full RawContent context |
| `/admin/review/{id}/approve` | POST | Approve (optionally with corrections) |
| `/admin/review/{id}/reject` | POST | Reject |
| `/admin/review/{id}/skip` | POST | Skip (keeps row in queue) |
| `/admin/review/backfill-resolution` | POST | Backfill SecMaster resolution for unresolved rows in a date range |
| `/admin/review/backfill-source-registration` | POST | Returns `410 Gone` (retired) |
| `/admin/review/backfill-auto-approve` | POST | Replay `AutoApprovePolicy` over Pending backlog (default dry-run) |
| `/admin/review/quarantine-hallucinated` | POST | Stamp `QuarantinedAt` on Approved rows whose Symbol isn't grounded in source text (default dry-run) |
| `/admin/rss-feeds` | GET/POST | List / create RSS feed subscriptions |
| `/admin/rss-feeds/{id}` | GET/PUT/DELETE | Get / update / delete |
| `/admin/rss-feeds/{id}/toggle` | POST | Toggle `enabled` |
| `/admin/validation-triggers` | GET/POST | List / create validation triggers |
| `/admin/validation-triggers/{id}` | GET/PUT/DELETE | Get / update / delete |
| `/admin/validation-triggers/{id}/toggle` | POST | Toggle `enabled` |
| `/sentinel/digests/` | GET | List digests (optional `?period=`) |
| `/sentinel/digests/latest` | GET | 302 → latest digest for `period` |
| `/sentinel/digests/{id}` | GET | Digest as HTML |
| `/sentinel/digests/{id}.md` | GET | Digest as Markdown |
| `/sentinel/digests/{id}.json` | GET | Digest metadata + markdown body |
| `/sentinel/digests/generate` | POST | Generate a digest on demand (`?period`, `?pushNotification`, `?force`) |
| `/sentinel/digests/{id}/resend` | POST | Resend ntfy notification |
| `/ui/review` | GET | Browser review queue (paginated, `?maxConfidence` filter) |
| `/ui/review/{id}` | GET | Browser review-detail page |
| `/ui/review/{id}/action` | POST | Form-encoded approve/reject/skip submission |
| `/swagger` | GET | OpenAPI UI (development environment only) |
| `/health` | GET | Aggregate health with per-check status |
| `/health/ready` | GET | Readiness (database + ollama) |
| `/health/live` | GET | Liveness |

### gRPC API (Port 5001)

| Service | Method | Description |
|---------|--------|-------------|
| `ObservationEventStream` | `SubscribeToEvents` | Server-streaming events from a `start_from` timestamp |
| `ObservationEventStream` | `GetEventsSince` | Server-streaming replay since a timestamp |
| `ObservationEventStream` | `GetEventsBetween` | Server-streaming replay across a time range |
| `ObservationEventStream` | `GetLatestEventTime` | Unary — timestamp of the most recent event |
| `ObservationEventStream` | `GetHealth` | Unary — health + event statistics |
| `MatrixUpdateStream` | `SubscribeToMatrixCellUpdates` | Server-streaming Phase 5.5 matrix-cell enrichment updates |
| `grpc.reflection.v1alpha.ServerReflection` | * | gRPC reflection (always on) |

## Project Structure

```
SentinelCollector/
├── src/
│   ├── Configuration/    # *Options.cs (EdgeSync, Extraction, CpuCod, Searxng, ReExtract, ...)
│   ├── Data/             # SentinelDbContext, Repositories/, Configurations/, Migrations/
│   ├── Endpoints/        # AdminEndpoints, ReviewUiEndpoints, DigestEndpoints
│   ├── Entities/         # RawContent, ExtractedObservation, RssFeed, ValidationTrigger, ...
│   ├── Extraction/       # CoVe, CoD, ExtractionSchemaV2, DSL adapter, prompt providers, tool-augmented variants
│   ├── Grpc/             # EventStreamService, MatrixCellUpdateStreamService, repositories
│   ├── HealthChecks/     # DatabaseHealthCheck, OllamaHealthCheck
│   ├── Publishers/       # EventPublisher (writes to events table for gRPC stream)
│   ├── Semantic/         # EntityResolver, ClaimVerifier, SectorTagger, MatrixCellEnrichmentPublisher
│   ├── Services/         # HTTP clients (Ollama/LlamaServer/Vllm/Markitdown/Trafilatura/SpacyNer/SecMaster/Finnhub/AlphaVantage/Gemini), digest, normaliser, DSL extraction, CoVe verifier
│   ├── Telemetry/        # SentinelActivitySource, SentinelMeter
│   ├── Workers/          # EdgeSync, Extraction, Searxng + RSS schedulers, Resolution, ReExtract, Validation, Digest, AlphaVantageSweep, MacroSignalIdentityCatalogRefresh
│   ├── prompts/          # Default prompt templates baked into the source tree (production uses host mount)
│   └── Containerfile     # build / development / runtime targets
├── tests/
│   ├── SentinelCollector.UnitTests/
│   ├── SentinelCollector.IntegrationTests/
│   └── Golden/           # Golden-file fixtures
├── agent/                # Sandbox-agent Python sidecar (separate README)
├── dsl-parser-mcp/       # DSL parser + verifier sidecar (separate README)
├── config/               # Operator configuration assets (separate README)
├── scripts/              # Training-data generation, audit, eval scripts
├── training/             # LoRA training artifacts and notes
└── .devcontainer/        # build.sh, compile.sh, devcontainer.json, compose.yaml
```

## Development

### Using Dev Container

```bash
# Open in VS Code and select "Reopen in Container"
cd /workspace/SentinelCollector/src
dotnet run
```

### Compile + Test

```bash
SentinelCollector/.devcontainer/compile.sh             # build + unit + integration tests
SentinelCollector/.devcontainer/compile.sh --no-test   # build only
```

### Build Container Image

```bash
SentinelCollector/.devcontainer/build.sh               # nerdctl build (with cache)
SentinelCollector/.devcontainer/build.sh --no-cache    # clean rebuild
```

## Deployment

```bash
ansible-playbook deployment/ansible/playbooks/deploy.yml --tags sentinel-collector
```

Prompts are host-mounted: edit files under `/opt/ai-inference/prompts/sentinel/` (and `/opt/ai-inference/prompts/cod/` for the CoD DSL pipeline); the prompt providers hot-reload when `Extraction__WatchPromptFiles=true`.

## Ports

| Port | Type | Description |
|------|------|-------------|
| 8080 | HTTP (container) | REST API, health checks, review UI, digests, Swagger (dev) |
| 5001 | HTTP/2 (container) | gRPC `ObservationEventStream` + `MatrixUpdateStream` |
| 5091 | HTTP (host-mapped) | Browser access to review UI + digest pages — maps to container `8080` |

## See Also

- [ThresholdEngine](../ThresholdEngine/README.md) - Consumes observation + matrix-cell events via gRPC
- [SecMaster](../SecMaster/README.md) - Instrument resolution and signal-identity catalog
- [FinnhubCollector](../FinnhubCollector/README.md) - Symbol lookup upstream for the resolution cascade
- [AlphaVantageCollector](../AlphaVantageCollector/README.md) - Nightly sweep upstream for unresolved rows
- [Events](../Events/README.md) - Shared gRPC event + matrix-update contracts
- [MacroSubstrate](../MacroSubstrate/README.md) - Shared macro-observation write path
