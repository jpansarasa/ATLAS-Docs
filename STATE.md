# STATE.md [ATLAS Infrastructure]

## CURRENT_STATUS [2025-12-14]

### All Services: Production Ready

| Service | Status | Tests | Description |
|---------|--------|-------|-------------|
| FredCollector | done | 378 | FRED economic data (47 series) |
| ThresholdEngine | done | 226 | Pattern evaluation & regime detection |
| AlertService | done | 15+ | Notification dispatch (ntfy, email) |
| CalendarService | done | 5 | Trading day validation & market status |
| FinnhubCollector | done | 5 | Stock quotes, sentiment, analyst ratings |
| AlphaVantageCollector | done | 5 | Commodity prices (WTI, Brent, NatGas) |
| OfrCollector | done | 5 | OFR FSI, STFM, HFM data |
| NasdaqCollector | done | 5 | LBMA gold prices (AM/PM fixings) |
| SecMaster | done | 5+ | Security master & instrument metadata |

**Total Tests**: 645+ passing

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

**All collectors use internal port 5001 for gRPC event streaming**

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

**Total Patterns**: 50+ configured

---

## PIPELINE [end_to_end]

```
FredCollector ──────┐
FinnhubCollector ───┼──► ThresholdEngine ──► Prometheus ──► Alertmanager ──► AlertService ──► ntfy/email
AlphaVantageCollector──┤       │                    │
OfrCollector ───────┤       │                    Grafana
NasdaqCollector ────┘       │
                            ↓
                       SecMaster (registration + resolution)
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

### SecMaster Roadmap
| PR | Description | Status |
|----|-------------|--------|
| #66 | Core SecMaster - gRPC services, FredCollector integration | Merged |
| #67 | Sector Taxonomy - `units` column, alias endpoints, search enhancement | Next |
| #68 | Collector Metadata - Pass-through from FRED/OFR/Finnhub/AV | Planned |
| Future | LLM/RSS - News analysis, instrument/sector mapping | Requires #67 |

**Key Principle**: Store raw, display normalized. No cross-collector normalization.

### Immediate
1. Deploy SecMaster to production
2. Create PR #67 branch for taxonomy work

---
**UPDATED**: 2025-12-14 | **STATUS**: production_ready
