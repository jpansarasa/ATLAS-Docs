# STATE.md [ATLAS Infrastructure]

## CURRENT_STATUS [2025-12-26]

### All Services: Production Ready

| Service | Status | Tests | Description |
|---------|--------|-------|-------------|
| FredCollector | done | 378 | FRED economic data (47 series) |
| ThresholdEngine | done | 226 | Pattern evaluation & regime detection (weighted scoring) |
| AlertService | done | 15+ | Notification dispatch (ntfy, email) |
| CalendarService | done | 5 | Trading day validation & market status |
| FinnhubCollector | done | 5 | Stock quotes, sentiment, analyst ratings |
| AlphaVantageCollector | done | 5 | Commodity prices (WTI, Brent, NatGas) |
| OfrCollector | done | 5 | OFR FSI, STFM, HFM data |
| NasdaqCollector | done | 5 | LBMA gold prices (AM/PM fixings) |
| SecMaster | done | 5+ | Security master & instrument metadata |
| SentinelCollector | pr | - | Alternative data + LLM extraction (PR #72) |

**Total Tests**: 645+ passing

### Recent Changes [2025-12-26]
- → PR #72 created: SentinelCollector + Edge Worker for alternative data
  - Cloudflare Workers edge component (Challenger RSS, Fed RSS, TSA)
  - LLM extraction with Chain-of-Verification (CoVe)
  - SearXNG news search integration
  - Ansible deployment ready

### Previous [2025-12-16]
- ✓ PR #70 merged: Pattern Weighting & Temporal Metadata complete
- ✓ SecMaster hybrid search instrument catalog (#69)
- ✓ Documentation consolidated: 5 ThresholdEngine docs → `THRESHOLDENGINE-PATTERNS.md`
- ✓ Removed stale STATE.md files from service directories

### Completed [2025-12-15]
- ✓ Pattern Weighting & Temporal Metadata (ThresholdEngine)
- ✓ Freshness Decay with publication frequency awareness
- ✓ Temporal multipliers during regime transitions
- ✓ Health checks for stale/missing data
- ✓ `/api/patterns/contributions` and `/api/patterns/health` endpoints
- ✓ Grafana dashboards for pattern observability

---

## SERVICES [running]

### Core Data Collectors
| Service | Ports (Host) | Internal | Interval |
|---------|--------------|----------|----------|
| fred-collector | 5001/5002 | 8080/5001 | 6 hours |
| finnhub-collector | 5012/5013 | 8080/5001 | 60 req/min |
| alphavantage-collector | 5010/5011 | 8080/5001 | 25/day limit |
| ofr-collector | 5016/5017 | 8080/5001 | daily |
| nasdaq-collector | - | 8080/5009 | 6 hours |
| sentinel-collector | 5020 | 8080/5001 | edge-triggered |

**All collectors use internal port 5001 for gRPC event streaming**

### Edge Workers (Cloudflare)
| Worker | Sources | Schedule |
|--------|---------|----------|
| sentinel-edge | Challenger RSS, Fed RSS, TSA | 6 hours |

### Processing & Alerting
| Service | Port | Purpose |
|---------|------|---------|
| secmaster | - | Security master, instrument metadata, source resolution |
| threshold-engine | 5003 | Pattern evaluation, regime detection |
| alert-service | 8081 | Notification sink (ntfy, email) |
| calendar-service | - | Market status, trading day rules |
| timescaledb | 5432 | TimescaleDB (hypertables) |

### AI/Inference
| Service | Port | Hardware |
|---------|------|----------|
| ollama-gpu | 11434 | RTX 5090 (32GB VRAM) |
| ollama-cpu | 11435 | CPU fallback |

### MCP Servers (Claude Desktop)
| Service | Port | Purpose |
|---------|------|---------|
| ollama-mcp | 3100 | Ollama inference |
| markitdown-mcp | 3102 | Document conversion |
| fredcollector-mcp | 3103 | FRED data access |
| thresholdengine-mcp | 3104 | Pattern evaluation |
| finnhub-mcp | 3105 | Finnhub data access |
| ofrcollector-mcp | 3106 | OFR data access |
| secmaster-mcp | - | Instrument metadata & source resolution |

### Observability Stack
| Service | Port | Purpose |
|---------|------|---------|
| prometheus | 9090 | Metrics collection |
| alertmanager | 9093 | Alert routing |
| grafana | 3000 | Dashboards |
| loki | 3101 | Log aggregation |
| tempo | 3200 | Distributed tracing |
| otel-collector | 4317 | OTLP receiver |

---

## THRESHOLD_ENGINE [config_driven]

### Collector Subscriptions
ThresholdEngine subscribes to all collectors via configuration:

```json
"Collectors": {
  "Items": [
    { "Name": "FredCollector", "ServiceUrl": "http://fred-collector:5001" },
    { "Name": "FinnhubCollector", "ServiceUrl": "http://finnhub-collector:5001" },
    { "Name": "AlphaVantageCollector", "ServiceUrl": "http://alphavantage-collector:5001" },
    { "Name": "OfrCollector", "ServiceUrl": "http://ofr-collector:5001" }
  ]
}
```

### Macro Scoring Weights (configurable)
| Category | Weight | Patterns |
|----------|--------|----------|
| Recession | 30% | 12 |
| Liquidity | 20% | 9 |
| Growth | 20% | 5 |
| NBFI | 10% | 14 |
| Currency | 10% | 3 |
| Inflation | 10% | 8 |
| Valuation | 0% | 2 |
| Commodity | 0% | 1 |

**Total Patterns**: 57 configured with weighted scoring

### Pattern Weighting [NEW]
Each pattern has reliability weights and temporal metadata:
- `weight`: 0.0-1.0 reliability score (0.95=proven, 0.50=supplementary)
- `temporalType`: Leading|Coincident|Lagging
- `publicationFrequencyDays`: Expected update frequency (0=real-time, 30=monthly)
- `signalDecayDays`: How long signal remains relevant after overdue

**Formula**: `weightedSignal = signal × weight × freshness × temporal × confidence`

### Health Monitoring [NEW]
- `/api/patterns/contributions`: Weighted breakdown by category
- `/api/patterns/health`: Data freshness and staleness alerts
- `PatternDataHealthCheck`: Integrated health check for stale/missing data
- Grafana dashboards: `pattern-contributions.json`, `pattern-data-health.json`

---

## PIPELINE [end_to_end]

```
FredCollector ──────┐
FinnhubCollector ───┤
AlphaVantageCollector──┼──► ThresholdEngine ──► Prometheus ──► Alertmanager ──► AlertService ──► ntfy/email
OfrCollector ───────┤       │                    │
NasdaqCollector ────┤       │                    Grafana
SentinelCollector ──┘       │
       ↑                    ↓
  Edge Workers         SecMaster (registration + resolution)
  (Cloudflare D1)
```

---

## DEPLOYMENT

```bash
cd ~/ATLAS/deployment/ansible
ansible-playbook playbooks/deploy.yml              # Full deploy
ansible-playbook playbooks/deploy.yml --tags threshold-engine  # Single service
```

---

## NEXT_ACTIONS

### SentinelCollector Deployment (PR #72)
| Step | Description | Status |
|------|-------------|--------|
| 1 | Deploy Cloudflare Worker + D1 database | ◯ |
| 2 | Add vault credentials (edge endpoint, API key) | ◯ |
| 3 | Deploy SentinelCollector: `--tags sentinel-collector` | ◯ |
| 4 | End-to-end validation with Challenger data | ◯ |
| 5 | Merge PR #72 | ◯ |

### SecMaster Roadmap
| PR | Description | Status |
|----|-------------|--------|
| #66 | Core SecMaster - gRPC services, FredCollector integration | ✓ Merged |
| #69 | Hybrid search instrument catalog | ✓ Merged |
| #70 | Pattern Weighting & Temporal Metadata | ✓ Merged |
| #72 | SentinelCollector + Edge Worker | → In Review |
| #67 | Sector Taxonomy - `units` column, alias endpoints | Next |
| #68 | Collector Metadata - Pass-through from FRED/OFR/Finnhub/AV | Planned |

**Key Principle**: Store raw, display normalized. No cross-collector normalization.

### Immediate
1. Complete SentinelCollector deployment (PR #72)
2. Create PR #67 branch for sector taxonomy work

---
**UPDATED**: 2025-12-26 | **STATUS**: production_ready
