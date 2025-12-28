# STATE.md [SentinelCollector]

## GOAL
Alternative data collection platform with dual LLM-based extraction:
- **Chain of Verification (CoVe)**: Verified structured data extraction
- **Chain of Density (CoD)**: Context-preserving summaries for RAG

### Accept Criteria
- Edge worker collects RSS/scrape data to D1 buffer
- Home-side pulls from edge, stores to raw layer
- Dual extraction: CoVe (structured) + CoD (context summaries)
- Epistemic certainty classification (definite/expected/speculative/conditional)
- SecMaster resolves instruments with confidence routing
- Events published to ThresholdEngine via gRPC (definite/expected only)
- Context summaries stored for RAG consumption

### Constraints
- Must use existing gRPC EventStreamService pattern
- CoVe/CoD extraction requires Ollama 70B model
- Edge sync requires Cloudflare D1 database

---

## ARCH

### Decisions
| Decision | Choice | Rationale |
|----------|--------|-----------|
| Edge collection | Cloudflare Workers + D1 + R2 | Low-latency RSS/scrape, edge buffering, binary storage |
| Structured extraction | Ollama 70B + CoVe | Chain-of-Verification for accuracy |
| Context summaries | Ollama 70B + CoD | Chain-of-Density preserves epistemic nuance |
| Certainty classification | 4-level enum | Prevents speculative data triggering alerts |
| Instrument resolution | SecMaster semantic search | Confidence-based routing |
| Event publishing | gRPC EventStreamService | Consistent with other collectors |
| Content normalization | Markitdown MCP | Converts HTML/PDF to markdown for extraction |

### Data Flow
```
Edge Workers (D1/R2) → EdgeSyncWorker → Raw Layer
                                            │
                       ┌────────────────────┴────────────────────┐
                       ▼                                          ▼
                 CoD (5 iterations)                    CoVe (4 steps)
                 Context Summary                       Structured Data
                 Epistemic Markers                     + Certainty
                       │                                          │
                       ▼                                          ▼
                 RawContent.ContextSummary            ExtractedObservation
                 (for RAG)                            + Certainty + Condition
                                                                  │
                                                      ┌───────────┴───────────┐
                                                      ▼                       ▼
                                               definite/expected      speculative/conditional
                                                      ▼                       ▼
                                               SecMaster → Events      Stored only (no alerts)
```

---

## STATUS

### Phase 1 Implementation ✓ Complete
| Phase | Description | Status |
|-------|-------------|--------|
| 1.1 | SentinelCollector scaffold | ✓ |
| 1.2 | Database layer (EF Core) | ✓ |
| 1.3 | Edge Worker - Generic collectors | ✓ |
| 1.4 | Edge Sync Worker + R2 binary support | ✓ |
| 1.5 | LLM Extraction (Ollama + CoVe + CoD) | ✓ |
| 1.6 | SecMaster integration | ✓ |
| 1.7 | Event Publishing | ✓ |
| 1.8 | Markitdown content normalization | ✓ |
| 1.9 | SearXNG collector | ✓ |
| 1.10 | Ansible deployment config | ✓ |
| 1.11 | EF migration fix (context_summary, certainty) | ✓ |

### Sources (4 ready for MVP)
| Source | Type | Status |
|--------|------|--------|
| Challenger RSS | RSS | ✓ Ready |
| Fed RSS (4 feeds) | RSS | ✓ Ready |
| TSA Checkpoint | Scrape | ✓ Ready |
| SearXNG | Search API | ✓ Ready |
| Truflation | API | ⊘ Skipped (no free API key) |

**PRs**: #72 (initial), #73 (markitdown), #74 (edge refactor + R2)

---

## NEXT [deployment]

### 1. Deploy Cloudflare Worker (FIRST)
See `edge/sentinel-edge/STATE.md` for detailed steps:
```bash
cd edge/sentinel-edge
wrangler login
wrangler d1 create sentinel-edge-db
# Update wrangler.toml with database_id
wrangler d1 execute sentinel-edge-db --file=schema.sql
wrangler secret put API_KEY
wrangler deploy
```

### 2. Add Vault Credentials
```bash
ansible-vault edit deployment/inventory/group_vars/all/vault.yml
```
Add:
- `vault_sentinel_edge_endpoint: https://<worker>.workers.dev`
- `vault_sentinel_edge_api_key: <generated_key>`

### 3. Deploy SentinelCollector
```bash
ansible-playbook playbooks/deploy.yml --tags sentinel-collector
```

### 4. Validate
- [ ] Health check: `curl http://localhost:5020/health`
- [ ] Trigger edge collection: `curl -X POST https://<worker>/collect -H "X-API-Key: <key>"`
- [ ] Verify edge sync pulls content
- [ ] Verify LLM extraction produces observations
- [ ] Verify events published to ThresholdEngine

---

## CONTEXT

### Key Files
| File | Purpose |
|------|---------|
| `src/Workers/EdgeSyncWorker.cs` | Polls edge D1, writes to raw layer |
| `src/Workers/ExtractionProcessor.cs` | Dual extraction (CoD + CoVe) |
| `src/Workers/SearxngCollectionScheduler.cs` | SearXNG news search |
| `src/Extraction/ChainOfVerification.cs` | 4-step CoVe for structured extraction |
| `src/Extraction/ChainOfDensity.cs` | 5-iteration CoD for context summaries |
| `src/Extraction/Certainty.cs` | Epistemic certainty enum |
| `src/Services/SecMasterClient.cs` | Instrument resolution |
| `src/Services/R2Client.cs` | Cloudflare R2 binary downloads |
| `src/Services/ContentNormalizer.cs` | Markitdown integration |
| `src/Publishers/EventPublisher.cs` | gRPC event creation |

### Dependencies
| Service | Purpose |
|---------|---------|
| timescaledb | sentinel schema tables |
| secmaster | Instrument resolution |
| ollama-gpu | LLM extraction (70B model) |
| markitdown-mcp | Content normalization |
| otel-collector | Telemetry |

### Configuration
- `EdgeSync:Endpoint` - Cloudflare Worker URL
- `EdgeSync:ApiKey` - Worker API key
- `R2:Endpoint` - Cloudflare R2 endpoint
- `R2:AccessKeyId` / `R2:SecretAccessKey` - R2 credentials
- `Searxng:Endpoint` - SearXNG instance URL
- `Searxng:Queries` - Economic indicator search terms

---
**UPDATED**: 2025-12-27 | **STATUS**: ready_for_deployment
