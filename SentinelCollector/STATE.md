# STATE.md [SentinelCollector]

## GOAL
Alternative data collection platform with LLM-based extraction using Chain-of-Verification (CoVe).

### Accept Criteria
- Edge worker collects RSS/scrape data to D1 buffer
- Home-side pulls from edge, stores to raw layer
- LLM extraction with CoVe produces structured observations
- SecMaster resolves instruments with confidence routing
- Events published to ThresholdEngine via gRPC

### Constraints
- Must use existing gRPC EventStreamService pattern
- CoVe extraction requires Ollama 70B model
- Edge sync requires Cloudflare D1 database

---

## ARCH

### Decisions
| Decision | Choice | Rationale |
|----------|--------|-----------|
| Edge collection | Cloudflare Workers + D1 | Low-latency RSS/scrape, edge buffering |
| LLM extraction | Ollama 70B + CoVe | Chain-of-Verification for accuracy |
| Instrument resolution | SecMaster semantic search | Confidence-based routing |
| Event publishing | gRPC EventStreamService | Consistent with other collectors |

### Data Flow
```
Edge Workers (D1) → EdgeSyncWorker → Raw Layer → ExtractionProcessor (CoVe) → SecMaster → Events → ThresholdEngine
```

---

## STATUS

### Phase 1 Implementation ✓ Complete
| Phase | Description | Status |
|-------|-------------|--------|
| 1.1 | SentinelCollector scaffold | ✓ |
| 1.2 | Database layer (EF Core, hypertables) | ✓ |
| 1.3 | Edge Worker - Challenger RSS | ✓ |
| 1.4 | Edge Sync Worker | ✓ |
| 1.5 | LLM Extraction (Ollama + CoVe) | ✓ |
| 1.6 | SecMaster integration | ✓ |
| 1.7 | Event Publishing | ✓ |
| 1.8 | Fed RSS + TSA collectors | ✓ |
| 1.9 | SearXNG collector | ✓ |
| 1.10 | Ansible deployment | ✓ |

**PR**: #72 (feature/sentinel-collector)

---

## NEXT [remaining_work]

### Pre-Deployment (Required)
- [ ] Add vault credentials to `vault.yml`:
  - `vault_sentinel_edge_endpoint`
  - `vault_sentinel_edge_api_key`
- [ ] Deploy Cloudflare Worker (see edge/sentinel-edge/STATE.md)
- [ ] Verify compose.yaml.j2 templating works

### Deployment
- [ ] Deploy with Ansible: `ansible-playbook playbooks/deploy.yml --tags sentinel-collector`
- [ ] Verify container starts and connects to dependencies
- [ ] Check health endpoint: `curl http://localhost:5020/health`

### Validation
- [ ] Trigger manual edge sync and verify raw content stored
- [ ] Verify LLM extraction produces observations
- [ ] Verify events published to ThresholdEngine
- [ ] Check Grafana dashboards for metrics

### Post-MVP (Phase 2)
- [ ] Add unit tests for extraction logic
- [ ] Add integration tests for edge sync
- [ ] Grafana dashboard for SentinelCollector
- [ ] Alert rules for extraction failures

---

## CONTEXT

### Key Files
| File | Purpose |
|------|---------|
| `src/Workers/EdgeSyncWorker.cs` | Polls edge D1, writes to raw layer |
| `src/Workers/ExtractionProcessor.cs` | LLM extraction with CoVe |
| `src/Workers/SearxngCollectionScheduler.cs` | SearXNG news search |
| `src/Extraction/ChainOfVerification.cs` | 4-step CoVe implementation |
| `src/Services/SecMasterClient.cs` | Instrument resolution |
| `src/Publishers/EventPublisher.cs` | gRPC event creation |

### Dependencies
| Service | Purpose |
|---------|---------|
| timescaledb | sentinel schema hypertables |
| secmaster | Instrument resolution |
| ollama-gpu | LLM extraction (70B model) |
| otel-collector | Telemetry |

### Configuration
- `EdgeSync:Endpoint` - Cloudflare Worker URL
- `EdgeSync:ApiKey` - Worker API key
- `Searxng:Endpoint` - SearXNG instance URL
- `Searxng:Queries` - Economic indicator search terms

---
**UPDATED**: 2025-12-26 | **STATUS**: pr_created
