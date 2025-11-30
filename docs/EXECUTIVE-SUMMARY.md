# ATLAS Platform - Executive Summary

**Status:** Production Ready
**Updated:** 2025-11-30

## Overview

ATLAS is a real-time macroeconomic monitoring platform that ingests data from multiple sources (FRED, Alpha Vantage, Nasdaq, Finnhub), evaluates 40 pattern-based signals, detects regime transitions, and delivers actionable alerts for portfolio allocation decisions.

## Architecture

```mermaid
flowchart LR
    subgraph Collectors
        FC[FredCollector]
        AV[AlphaVantage]
        NC[Nasdaq]
        FH[Finnhub]
    end

    subgraph Processing
        TE[ThresholdEngine]
        AS[AlertService]
    end

    subgraph Storage
        DB[(TimescaleDB)]
    end

    subgraph Observability
        P[Prometheus]
        G[Grafana]
    end

    FC & AV & NC & FH -->|gRPC| TE
    FC & AV & NC & FH --> DB
    TE --> P
    P --> AS
    AS --> N[ntfy.sh / Email]
    DB & P --> G
```

## Services

| Service | Status | Description |
|---------|--------|-------------|
| FredCollector | ✅ | 25 FRED economic series, 378 tests |
| AlphaVantageCollector | ✅ | Commodities (WTI, Brent, Natural Gas, Copper) |
| NasdaqCollector | ✅ | LBMA gold prices (AM/PM fixings) |
| FinnhubCollector | ✅ | Stock quotes, sentiment, analyst ratings |
| CalendarService | ✅ | Market status, trading day validation |
| ThresholdEngine | ✅ | 40 patterns, regime detection, 153 tests |
| AlertService | ✅ | ntfy.sh + email notification channels |

**Total:** 22 containers, 550+ tests passing

## Pattern Library

| Category | Count | Examples |
|----------|-------|----------|
| Recession | 12 | Sahm Rule, yield curve inversion, initial claims |
| Liquidity | 8 | VIX L1/L2, credit spreads, Fed liquidity |
| NBFI Stress | 9 | HY spreads, KRE, repo facility, Chicago NFCI |
| Growth | 5 | GDP acceleration, industrial production |
| Valuation | 5 | CAPE, Buffett indicator, forward P/E |
| Commodity | 1 | Copper/Gold ratio |

## Regime Detection

Six-state machine with hysteresis:

| Regime | Macro Score | Equity | Defensive |
|--------|-------------|--------|-----------|
| Growth | > 10 | 80-90% | 10-20% |
| Recovery | 0 to 10 | 70-80% | 20-30% |
| Neutral | -10 to 0 | 60-70% | 30-40% |
| Late Cycle | -10 to 0 | 55-70% | 30-45% |
| Recession | -20 to -10 | 70-80% | 20-30% |
| Crisis | < -20 | 80-90% | 10-20% |

## Key Endpoints

| Service | Port | Purpose |
|---------|------|---------|
| fred-collector | 5001/5002 | REST API / gRPC streaming |
| alphavantage-collector | 5003/5004 | HTTP / gRPC |
| nasdaq-collector | 5005/5006 | HTTP / gRPC |
| finnhub-collector | 5007/5008 | HTTP / gRPC |
| threshold-engine | 8080 | Pattern management API |
| alert-service | 8081 | Alertmanager webhook sink |
| grafana | 3000 | 9 dashboards |
| prometheus | 9090 | Metrics |

## MCP Servers (Claude Desktop)

| Server | Port | Purpose |
|--------|------|---------|
| ollama-mcp | 3100 | Local LLM inference (GPU/CPU) |
| fredcollector-mcp | 3103 | FRED data access |
| thresholdengine-mcp | 3104 | Pattern evaluation |

## Infrastructure

- **Runtime:** nerdctl + containerd
- **Database:** TimescaleDB (PostgreSQL + hypertables)
- **Observability:** OpenTelemetry → Prometheus, Loki, Tempo, Grafana
- **Deployment:** Ansible playbooks, systemd auto-start
- **Hardware:** AMD Threadripper 9960X, RTX 5090 (32GB), 128GB RAM

## Deployment

```bash
cd ~/ATLAS/ansible
ansible-playbook playbooks/site.yml
```

## See Also

- [STATE.md](../STATE.md) - Current system status
- [Architecture](ARCHITECTURE.md) - Design decisions
- [gRPC Architecture](GRPC-ARCHITECTURE.md) - Event streaming
- [Infrastructure](../infrastructure/README.md) - Compose and configs
