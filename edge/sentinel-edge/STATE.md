# STATE.md [Sentinel Edge Worker]

## GOAL
Cloudflare Workers edge component for RSS/scrape data collection with D1 buffering.

### Accept Criteria
- Collect from Challenger RSS, Fed RSS, TSA checkpoint
- Buffer to D1 with content deduplication
- Expose /sync and /sync/ack for home-side pull
- Cron triggers for scheduled collection

### Constraints
- Must run on Cloudflare Workers (edge runtime)
- D1 database for persistent buffer
- API key authentication for sync endpoints

---

## ARCH

### Collectors
| Collector | Source | Type | Schedule |
|-----------|--------|------|----------|
| challenger-rss | challengergray.com | RSS | Daily |
| fed-rss | federalreserve.gov | RSS (4 feeds) | Hourly |
| tsa-checkpoint | tsa.gov | HTML scrape | Daily |

### Endpoints
| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/health` | GET | Health check |
| `/stats` | GET | Collection statistics |
| `/sync` | GET | Fetch unsynced content |
| `/sync/ack` | POST | Mark content as synced |
| `/collect` | POST | Manual collection trigger |

### D1 Schema
```sql
CREATE TABLE raw_content (
    id TEXT PRIMARY KEY,           -- ULID
    source TEXT NOT NULL,
    collected_at TEXT NOT NULL,    -- ISO8601
    source_url TEXT NOT NULL,
    content_hash TEXT NOT NULL,    -- SHA-256 dedup
    content_type TEXT NOT NULL,
    raw_content TEXT NOT NULL,
    http_status INTEGER NOT NULL,
    headers TEXT,
    synced_at TEXT DEFAULT NULL,
    UNIQUE(source, content_hash)
);
```

---

## STATUS

### Implementation ✓ Complete
| Component | Status |
|-----------|--------|
| wrangler.toml | ✓ D1 binding, cron triggers |
| schema.sql | ✓ raw_content table |
| src/index.ts | ✓ Main worker, all endpoints |
| src/collectors/challenger.ts | ✓ Challenger RSS |
| src/collectors/fed-rss.ts | ✓ Fed RSS (4 feeds) |
| src/collectors/tsa.ts | ✓ TSA checkpoint |
| src/lib/d1.ts | ✓ D1 helpers |
| src/lib/ulid.ts | ✓ ULID generation |
| src/lib/hash.ts | ✓ SHA-256 hashing |
| .devcontainer | ✓ Auto-start wrangler dev |

**PR**: #72 (feature/sentinel-collector)

---

## NEXT [remaining_work]

### Cloudflare Setup (Required)
- [ ] Create Cloudflare Workers project:
  ```bash
  cd edge/sentinel-edge
  wrangler login
  wrangler d1 create sentinel-edge-db
  ```
- [ ] Update wrangler.toml with D1 database ID
- [ ] Apply D1 schema:
  ```bash
  wrangler d1 execute sentinel-edge-db --file=schema.sql
  ```

### Secrets Configuration
- [ ] Set API key secret:
  ```bash
  wrangler secret put API_KEY
  ```

### Deployment
- [ ] Deploy to Cloudflare:
  ```bash
  wrangler deploy
  ```
- [ ] Note the worker URL for SentinelCollector config

### Validation
- [ ] Test health endpoint: `curl https://<worker>.workers.dev/health`
- [ ] Test manual collection: `curl -X POST https://<worker>.workers.dev/collect -H "X-API-Key: <key>"`
- [ ] Verify D1 has collected content
- [ ] Test sync endpoint from home

### Post-Deployment
- [ ] Add to Ansible vault:
  - `vault_sentinel_edge_endpoint: https://<worker>.workers.dev`
  - `vault_sentinel_edge_api_key: <generated_key>`
- [ ] Enable cron triggers in Cloudflare dashboard

---

## CONTEXT

### Development
```bash
# Start devcontainer (auto-runs wrangler dev)
# Or manually:
cd edge/sentinel-edge
npm install
npm run dev
```

### Local Testing
```bash
# Health check
curl http://localhost:8787/health

# Manual collection
curl -X POST http://localhost:8787/collect

# Check stats
curl http://localhost:8787/stats

# Sync (get unsynced content)
curl http://localhost:8787/sync

# Acknowledge sync
curl -X POST http://localhost:8787/sync/ack \
  -H "Content-Type: application/json" \
  -d '{"ids": ["01ABCD..."]}'
```

### wrangler.toml Configuration
```toml
name = "sentinel-edge"
main = "src/index.ts"
compatibility_date = "2024-01-01"

[[d1_databases]]
binding = "DB"
database_name = "sentinel-edge-db"
database_id = "<UPDATE_AFTER_CREATE>"

[triggers]
crons = ["0 */6 * * *"]  # Every 6 hours
```

---
**UPDATED**: 2025-12-26 | **STATUS**: pr_created
