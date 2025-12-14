# ATLAS Architecture

## Overview

ATLAS (Automated Threshold Logic and Alert System) is an event-driven platform for financial data collection, pattern evaluation, and regime detection. It ingests data from multiple sources, evaluates 50+ configurable patterns, and delivers alerts when economic conditions change.

```mermaid
flowchart LR
    subgraph Collection
        FC[FredCollector<br/>FRED API]
        AV[AlphaVantageCollector<br/>Commodities/Forex]
        FH[FinnhubCollector<br/>Stocks/Calendars]
        OFR[OfrCollector<br/>FSI/STFM/HFM]
        CS[CalendarService<br/>Market Holidays]
    end

    subgraph Metadata
        SM[SecMaster<br/>Instrument Registry]
    end

    subgraph Evaluation
        TE[ThresholdEngine<br/>50+ patterns]
    end

    subgraph Alerting
        AS[AlertService<br/>ntfy + email]
    end

    subgraph MCP["MCP Servers (Claude)"]
        FM[FredCollectorMcp]
        TM[ThresholdEngineMcp]
        FHM[FinnhubMcp]
        OM[OfrCollectorMcp]
        SMM[SecMasterMcp]
    end

    FC -->|gRPC stream| TE
    AV -->|gRPC stream| TE
    FH -->|gRPC stream| TE
    OFR -->|gRPC stream| TE
    FC -->|register series| SM
    FH -->|register symbols| SM
    OFR -->|register series| SM
    TE -->|HTTP POST| AS
    AS --> N[ntfy.sh]
    AS --> E[Email]

    FC -.-> FM
    TE -.-> TM
    FH -.-> FHM
    OFR -.-> OM
    SM -.-> SMM
```

## Services

### Data Collectors

| Service | Ports (Host:Container) | Data Source | Key Data |
|---------|------------------------|-------------|----------|
| FredCollector | 5001:8080, 5002:5001 | Federal Reserve | 47 economic series |
| AlphaVantageCollector | 5010:8080, 5011:5001 | Alpha Vantage | Commodities, forex, crypto |
| FinnhubCollector | 5012:8080, 5013:5001 | Finnhub | Stock quotes, earnings, sentiment |
| OfrCollector | 5016:8080 | OFR.gov | Financial Stress Index, repo rates |
| CalendarService | (internal) | Nager.Date, Finnhub | Market holidays, economic events |

Note: All collectors use internal port 5001 for gRPC event streaming to ThresholdEngine

### Processing & Alerting

| Service | Port | Responsibility |
|---------|------|----------------|
| ThresholdEngine | 5003 | Pattern evaluation, regime detection, macro scoring |
| AlertService | 8081 | Notification routing (ntfy, email) |
| SecMaster | 5017 | Instrument metadata registry, series search |

### MCP Servers

| Service | Port | Tools | Purpose |
|---------|------|-------|---------|
| FredCollectorMcp | 3103 | 7 | FRED data query and admin |
| ThresholdEngineMcp | 3104 | 8 | Pattern evaluation and regime status |
| FinnhubMcp | 3105 | 26 | Market data, calendars, live quotes |
| OfrCollectorMcp | 3106 | 26 | FSI, funding markets, hedge fund data |
| SecMasterMcp | (internal) | 10+ | Instrument search, metadata query |

### Infrastructure

| Service | Port | Purpose |
|---------|------|---------|
| TimescaleDB | 5432 | Time-series database |
| Prometheus | 9090 | Metrics storage |
| Grafana | 3000 | Dashboards |
| Loki | 3101 | Log aggregation |
| Tempo | 3200 | Distributed tracing |
| Ollama (GPU) | 11434 | LLM inference |

## Design Principles

### Single Responsibility
- **Collectors**: Data ingestion only. No threshold logic.
- **ThresholdEngine**: Pattern evaluation only. No data collection.
- **AlertService**: Notification delivery only. No business logic.
- **MCP Servers**: API translation only. No data storage.

### Event-Driven Communication
- Collectors → ThresholdEngine: gRPC server streaming
- ThresholdEngine → AlertService: HTTP POST to `/alerts`
- All events share the `ObservationCollectedEvent` contract

### Configuration Over Code
- 50+ patterns defined in YAML with C# expressions
- Hot reload via file watcher (no restart needed)
- Admin APIs for runtime series management

## Event Flow

```mermaid
sequenceDiagram
    participant APIs as External APIs
    participant COL as Collectors
    participant DB as TimescaleDB
    participant TE as ThresholdEngine
    participant AS as AlertService

    APIs->>COL: Economic/Market data
    COL->>DB: Store observations
    COL->>TE: ObservationCollectedEvent (gRPC)
    TE->>TE: Evaluate 50+ patterns
    TE->>TE: Calculate macro score
    TE->>TE: Detect regime transitions
    TE->>AS: POST /alerts (if triggered)
    AS->>AS: Route by severity
    AS-->>ntfy: Push notification
    AS-->>Email: SMTP
```

## Pattern Categories

| Category | Count | Purpose | Key Patterns |
|----------|-------|---------|--------------|
| Recession | 12 | Contraction warnings | Sahm Rule, yield curve, claims |
| Liquidity | 9 | Market stress | VIX spikes, credit spreads, TED |
| Growth | 5 | Expansion signals | GDP, employment, ISM |
| NBFI | 14 | Shadow banking | OFR FSI, repo stress, hedge fund leverage |
| Valuation | 2 | Market levels | Buffett indicator, CAPE |
| Inflation | 8 | Price pressures | CPI, breakevens, commodity prices |
| Commodity | 1 | Real assets | Copper/Gold ratio |
| Currency | 3 | Risk sentiment | DXY, EM FX |

**Total: 50+ patterns** across 8 categories

## Regime Detection

ThresholdEngine maintains a regime state based on macro score:

| Regime | Macro Score | Meaning |
|--------|-------------|---------|
| Expansion | < 30 | Normal growth conditions |
| Caution | 30-50 | Elevated risk signals |
| Warning | 50-70 | Multiple stress indicators |
| Crisis | > 70 | Systemic stress detected |

Regime transitions trigger alerts via AlertService.

## Data Flow by Source

```mermaid
flowchart LR
    subgraph FRED["FRED (47 series)"]
        FA[FRED API] --> FC[FredCollector]
    end

    subgraph OFR["OFR (FSI, STFM, HFM)"]
        OA[OFR API] --> OC[OfrCollector]
    end

    subgraph Finnhub["Finnhub (Stocks, Calendars)"]
        FHA[Finnhub API] --> FHC[FinnhubCollector]
    end

    FC --> DB[(TimescaleDB)]
    OC --> DB
    FHC --> DB

    DB --> TE[ThresholdEngine]

    FC -->|gRPC| SM[SecMaster]
    OC -->|gRPC| SM
    FHC -->|gRPC| SM
    SM --> PG[(PostgreSQL)]

    FC -.-> FM[FredCollectorMcp]
    OC -.-> OM[OfrCollectorMcp]
    FHC -.-> FHM[FinnhubMcp]
    SM -.-> SMM[SecMasterMcp]

    FM -.-> C[Claude]
    OM -.-> C
    FHM -.-> C
    SMM -.-> C
```

**SecMaster Purpose**: Centralized instrument metadata, search across sources, series discovery

## MCP Integration

MCP (Model Context Protocol) servers expose ATLAS data to Claude:

```mermaid
flowchart TB
    CD[Claude Desktop] --> FM["fredcollector-mcp<br/>SSE :3103"]
    CD --> TM["thresholdengine-mcp<br/>SSE :3104"]
    CD --> FHM["finnhub-mcp<br/>SSE :3105"]
    CD --> OM["ofrcollector-mcp<br/>SSE :3106"]
    CD --> SMM["secmaster-mcp<br/>SSE :3107"]

    FM -.- FT["get_latest, get_observations,<br/>search, health..."]
    TM -.- TT["evaluate, list_patterns,<br/>get_pattern..."]
    FHM -.- FHT["get_quote, get_earnings_calendar,<br/>get_live_*..."]
    OM -.- OT["get_fsi_latest, list_stfm_series,<br/>add_series..."]
    SMM -.- ST["search_instruments, semantic_search,<br/>resolve_source, ask_secmaster..."]
```

## Why This Architecture?

### Composability
New data sources integrate by implementing the gRPC contract. OfrCollector was added without modifying ThresholdEngine.

### Observability
Full OpenTelemetry stack: traces (Tempo), metrics (Prometheus), logs (Loki), all visualized in Grafana.

### Flexibility
Change a threshold? Edit YAML, patterns hot-reload. Add a series? Use the admin API.

### AI-Native
MCP servers let Claude directly query financial data and evaluate patterns.

## See Also

- [FredCollector](../FredCollector/README.md) - FRED data collection
- [OfrCollector](../OfrCollector/README.md) - OFR financial stress data
- [FinnhubCollector](../FinnhubCollector/README.md) - Market data
- [ThresholdEngine](../ThresholdEngine/README.md) - Pattern evaluation
- [AlertService](../AlertService/README.md) - Notifications
- [Deployment](../deployment/README.md) - Infrastructure setup
