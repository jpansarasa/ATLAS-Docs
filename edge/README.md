# Edge

Cloudflare Workers edge infrastructure for alternative data collection.

## Overview

Edge compute layer for ATLAS, running on Cloudflare Workers. The `sentinel-edge` worker collects alternative financial data from RSS feeds, web pages, and binary files at the edge, buffering content to D1 (metadata/text) and R2 (binary). A home-side sync client pulls unsynced content for downstream processing.

## Architecture

```mermaid
flowchart LR
    subgraph External Sources
        RSS[RSS Feeds]
        HTML[Web Pages]
        BIN[Binary Files]
    end

    subgraph Cloudflare Edge
        W[sentinel-edge Worker]
        D1[(D1 Database)]
        R2[(R2 Bucket)]
    end

    subgraph Home Server
        Sync[Sync Client]
        Pipeline[Processing Pipeline]
    end

    RSS --> W
    HTML --> W
    BIN --> W
    W --> D1
    W --> R2
    Sync -->|GET /sync| W
    Sync -->|POST /sync/ack| W
    Sync --> Pipeline
```

Content flows from external sources into the worker on cron schedules, gets deduplicated via SHA-256 hashing, and is stored in D1/R2. The home server pulls unsynced content via the sync API and acknowledges receipt.

## Features

- **RSS collection**: Parses RSS/Atom feeds, stores individual items with metadata, supports regex filtering
- **HTML scraping**: Fetches web pages and stores raw content with content-hash deduplication
- **Binary collection**: Downloads files (PDF, Office docs) to R2 with D1 metadata references
- **URL fetching**: On-demand batch URL fetching with seen-URL deduplication tracking
- **Cron scheduling**: Automated collection on configurable schedules per source type
- **Sync API**: Pull-based sync with acknowledgment for reliable home-side ingestion
- **Admin API**: CRUD management of source configurations stored in D1
- **Auto-purge**: Daily maintenance cleans synced content (>7 days) and stale seen URLs (>30 days)
- **Workers Observability**: Invocation logs ingested to Cloudflare Observability (100% head sampling)

## Configuration

### Environment Variables

| Variable | Description | Default |
|----------|-------------|---------|
| `API_KEY` | Secret for authenticated endpoints (set via `wrangler secret`) | Required |
| `ENVIRONMENT` | Runtime environment label | `"production"` |

### Cloudflare Bindings

| Binding | Type | Resource |
|---------|------|----------|
| `DB` | D1 Database | `sentinel-edge-db` |
| `R2` | R2 Bucket | `sentinel-raw` |

### Worker Limits

| Setting | Value | Rationale |
|---------|-------|-----------|
| `cpu_ms` | `300_000` (5 min) | Raised from default 30s to accommodate large D1 queries during sync/purge |

### Cron Triggers

| Schedule | Description | Cron |
|----------|-------------|------|
| Daily purge | Clean synced content and stale URLs | `0 3 * * *` |
| Challenger RSS | First Thursday of month, 7:30 AM ET | `30 12 1-7 * 4` |
| Fed RSS | Every 6 hours | `0 */6 * * *` |
| TSA scraper | Daily at 6 AM ET | `0 11 * * *` |

### Configured Sources

Defined in `src/config/sources.json` (static, baked into the worker bundle) and mirrored into the `source_configs` D1 table for admin-API editing via migration `0001_add_source_configs.sql`.

| Source | Type | Description |
|--------|------|-------------|
| `challenger-rss` | RSS | Challenger Gray layoff announcements (filtered) |
| `fed-press-all` | RSS | Federal Reserve all press releases |
| `fed-press-monetary` | RSS | Federal Reserve monetary policy |
| `fed-press-bcreg` | RSS | Federal Reserve bank regulation |
| `fed-speeches` | RSS | Federal Reserve speeches |
| `tsa-checkpoint` | Scraper | TSA passenger volume data |

The `binary` collector code is implemented but no binary sources are currently configured.

## API Endpoints

### Public

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/health` | GET | Health check with D1 connectivity and last collection time |
| `/stats` | GET | Collection statistics (total, unsynced, by source) |
| `/sources` | GET | List statically configured sources from `sources.json` |

### Authenticated (X-API-Key header)

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/sync` | GET | Fetch unsynced content (query: `?limit=N`, max 500) |
| `/sync/ack` | POST | Mark content as synced (body: `{"ids": [...]}`) |
| `/collect` | POST | Manual collection trigger (body: `{"source?", "type?"}`) |
| `/fetch` | POST | Batch URL fetch and store (body: `{"urls": [...]}`, max 50) |
| `/purge` | POST | Manual purge of old synced content and seen URLs |

### Admin (X-API-Key header, `/admin/` prefix)

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/admin/sources` | GET | List all D1-managed source configs |
| `/admin/sources` | POST | Create a new source config |
| `/admin/sources/:id` | GET | Get a single source config |
| `/admin/sources/:id` | PUT | Update a source config |
| `/admin/sources/:id` | DELETE | Delete a source config |
| `/admin/sources/:id/test` | POST | Test-collect from a source config |

## Project Structure

```
edge/
├── sentinel-edge/
│   ├── src/
│   │   ├── index.ts             # Worker entry point and route handler
│   │   ├── admin.ts             # Admin CRUD for source configs
│   │   ├── collectors/
│   │   │   ├── rss.ts           # RSS/Atom feed collector
│   │   │   ├── scraper.ts       # HTML page collector
│   │   │   ├── binary.ts        # Binary file collector (R2)
│   │   │   └── url-fetcher.ts   # On-demand URL batch fetcher
│   │   ├── config/
│   │   │   └── sources.json     # Static source definitions
│   │   └── lib/
│   │       ├── d1.ts            # D1 query helpers
│   │       ├── hash.ts          # SHA-256 hashing
│   │       └── ulid.ts          # ULID generation
│   ├── migrations/              # D1 schema migrations (0001_add_source_configs, 0002_add_seen_urls)
│   ├── schema.sql               # Base D1 schema (raw_content, seen_urls, source_configs)
│   ├── wrangler.toml            # Worker config: bindings, crons, limits, observability
│   ├── tsconfig.json            # TypeScript config
│   ├── package.json             # Node dependencies and scripts
│   └── .devcontainer/
│       ├── Dockerfile           # node:22-bookworm-slim + wrangler + typescript
│       ├── compose.yaml         # nerdctl/docker compose for dev container
│       ├── devcontainer.json    # VS Code Dev Containers config
│       ├── build.sh             # Build dev image (nerdctl/docker)
│       ├── dev.sh               # Start wrangler dev server in container
│       └── typecheck.sh         # Run `tsc --noEmit` in container
└── README.md
```

## Development

### Prerequisites

- VS Code with Dev Containers extension
- Node.js 22 (provided by devcontainer via `node:22-bookworm-slim`)
- Cloudflare account with Wrangler CLI access

### Getting Started

1. Open in VS Code: `code edge/sentinel-edge/`
2. Reopen in Container (Cmd/Ctrl+Shift+P -> "Dev Containers: Reopen in Container")
3. Dev server starts automatically on port 8787

### Manual Development

Container-based helpers (recommended, match other ATLAS services):

```bash
cd edge/sentinel-edge/.devcontainer
./build.sh           # Build dev image (nerdctl or docker)
./dev.sh             # Start wrangler dev server inside container
./typecheck.sh       # Run TypeScript typecheck inside container
```

Host-side (requires local Node 22+ and wrangler):

```bash
cd edge/sentinel-edge
npm install
npm run dev          # wrangler dev (port 8787)
npm run typecheck    # tsc --noEmit
```

> Note: `package.json` also declares a `lint` script (`eslint src/`), but `eslint` is not in `devDependencies` — running it would fail without a separate install. Use the VS Code ESLint extension (auto-installed by the devcontainer) instead.

## Deployment

### Initial Setup

```bash
cd edge/sentinel-edge
wrangler login
wrangler d1 create sentinel-edge-db    # Update database_id in wrangler.toml
wrangler d1 execute sentinel-edge-db --file=schema.sql --remote
wrangler d1 execute sentinel-edge-db --file=migrations/0001_add_source_configs.sql --remote
wrangler d1 execute sentinel-edge-db --file=migrations/0002_add_seen_urls.sql --remote
wrangler r2 bucket create sentinel-raw
wrangler secret put API_KEY
```

### Deploy

```bash
cd edge/sentinel-edge
wrangler deploy            # production
wrangler deploy --env dev  # dev environment (sentinel-edge-dev worker)
```

Worker runs on Cloudflare's edge network; no ATLAS ansible playbook applies (sentinel-edge is not in `/opt/ai-inference/compose.yaml`). Home-side consumers (e.g. the SentinelCollector `EdgeSyncWorker`) reach the worker via its public Cloudflare URL using `EdgeSync__*` env vars.

## Ports

| Port | Description |
|------|-------------|
| 8787 | Wrangler dev server (local development only) |

Production traffic is routed by Cloudflare's edge network; no host port mapping is needed.

## See Also

- [sentinel-edge/schema.sql](sentinel-edge/schema.sql) - D1 database schema
- [sentinel-edge/migrations/](sentinel-edge/migrations/) - D1 schema migrations
- [sentinel-edge/wrangler.toml](sentinel-edge/wrangler.toml) - Worker, bindings, crons, limits, observability
- [sentinel-edge/src/config/sources.json](sentinel-edge/src/config/sources.json) - Static source definitions
- [docs/](../docs/) - System documentation
