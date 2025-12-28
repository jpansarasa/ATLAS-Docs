# STATE.md [Sentinel Edge Worker]

## GOAL
Cloudflare Workers edge component for RSS/scrape/binary data collection with D1/R2 buffering.

### Accept Criteria
- Collect from configured sources (RSS, scrape, binary)
- Buffer metadata to D1, binary content to R2
- Content deduplication via SHA-256 hash
- Expose /sync and /sync/ack for home-side pull
- Cron triggers for scheduled collection

### Constraints
- Must run on Cloudflare Workers (edge runtime)
- D1 database for metadata buffer
- R2 bucket for binary content storage
- API key authentication for sync endpoints

---

## ARCH

### Generic Collectors
| Type | File | Purpose |
|------|------|---------|
| RSS | `src/collectors/rss.ts` | Parse RSS/Atom feeds, extract items |
| Scraper | `src/collectors/scraper.ts` | Fetch HTML pages |
| Binary | `src/collectors/binary.ts` | Fetch binary files, store in R2 |

### Configured Sources (`src/config/sources.json`)
| Source | Type | Schedule |
|--------|------|----------|
| challenger-rss | RSS | Daily |
| fed-press-all | RSS | Hourly |
| fed-press-monetary | RSS | Hourly |
| fed-press-bcreg | RSS | Hourly |
| fed-speeches | RSS | Hourly |
| tsa-checkpoint | Scraper | Daily |

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
    raw_content TEXT,              -- NULL for binary (stored in R2)
    r2_key TEXT,                   -- R2 object key for binary content
    http_status INTEGER NOT NULL,
    headers TEXT,                  -- JSON
    synced_at TEXT DEFAULT NULL,
    UNIQUE(source, content_hash)
);
```

---

## STATUS

### Implementation ✓ Complete
| Component | Status |
|-----------|--------|
| wrangler.toml | ✓ D1 + R2 bindings, cron triggers |
| schema.sql | ✓ raw_content table with r2_key |
| src/index.ts | ✓ Main worker, all endpoints |
| src/collectors/rss.ts | ✓ Generic RSS collector |
| src/collectors/scraper.ts | ✓ Generic HTML scraper |
| src/collectors/binary.ts | ✓ Binary file collector with R2 |
| src/config/sources.json | ✓ Config-driven source definitions |
| src/lib/d1.ts | ✓ D1 helpers with R2 support |
| src/lib/ulid.ts | ✓ ULID generation |
| src/lib/hash.ts | ✓ SHA-256 hashing |
| .devcontainer | ✓ Auto-start wrangler dev |

**PRs**: #72 (initial), #74 (generic collectors + R2)

---

## NEXT [deployment]

### 1. Cloudflare Setup
```bash
cd edge/sentinel-edge
wrangler login
wrangler d1 create sentinel-edge-db
```
Note the database_id from output.

### 2. Update wrangler.toml
Replace `database_id = "<UPDATE_AFTER_CREATE>"` with actual ID.

### 3. Apply D1 Schema
```bash
wrangler d1 execute sentinel-edge-db --file=schema.sql
```

### 4. Create R2 Bucket (if using binary collector)
```bash
wrangler r2 bucket create sentinel-raw
```

### 5. Set API Key Secret
```bash
wrangler secret put API_KEY
# Enter a strong random key (save for vault.yml)
```

### 6. Deploy
```bash
wrangler deploy
```
Note the worker URL (e.g., `https://sentinel-edge.<account>.workers.dev`)

### 7. Validate
```bash
# Health check
curl https://<worker>.workers.dev/health

# Manual collection
curl -X POST https://<worker>.workers.dev/collect -H "X-API-Key: <key>"

# Check stats
curl https://<worker>.workers.dev/stats -H "X-API-Key: <key>"
```

### 8. Update Ansible Vault
```bash
ansible-vault edit deployment/inventory/group_vars/all/vault.yml
```
Add:
- `vault_sentinel_edge_endpoint: https://<worker>.workers.dev`
- `vault_sentinel_edge_api_key: <the_key_you_set>`

### 9. Enable Cron Triggers
In Cloudflare dashboard, verify cron triggers are active.

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

### Adding New Sources
Edit `src/config/sources.json`:
```json
{
  "rss": [
    { "name": "new-source", "url": "https://...", "itemFilter": "optional regex" }
  ],
  "scraper": [
    { "name": "new-scrape", "url": "https://...", "accept": "text/html" }
  ],
  "binary": [
    { "name": "new-binary", "url": "https://...", "contentType": "application/pdf" }
  ]
}
```

---
**UPDATED**: 2025-12-27 | **STATUS**: ready_for_deployment
