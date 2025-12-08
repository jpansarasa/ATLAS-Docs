# OfrCollectorMcp

MCP server providing Claude direct access to ATLAS Office of Financial Research (OFR) financial stress and funding market data.

## Overview

Wraps the OfrCollector REST API, providing:
- **Financial Stress Index (FSI)**: OFR's primary financial stress measure with category and regional breakdowns
- **Short-term Funding Monitor (STFM)**: Repo rates, SOFR, Treasury bill rates, money market data
- **Hedge Fund Monitor (HFM)**: Hedge fund leverage and risk indicators
- **Series Management**: Admin tools to configure data collection and backfill historical data

## Available Tools

### FSI (Financial Stress Index) Tools (2 tools)

- `get_fsi_latest` - Get latest FSI observation with detailed breakdowns:
  - Category contributions (credit, equity valuation, funding, safe assets, volatility)
  - Regional contributions (US, advanced economies, emerging markets)
- `get_fsi_history` - Get historical FSI observations with date filtering

### STFM (Short-term Funding Monitor) Tools (3 tools)

- `list_stfm_series` - List all configured STFM series (repo rates, SOFR, T-bills, etc.)
- `get_stfm_latest` - Get latest observation for a STFM series
- `get_stfm_observations` - Get historical observations for a STFM series with date filtering

### HFM (Hedge Fund Monitor) Tools (3 tools)

- `list_hfm_series` - List all configured HFM series
- `get_hfm_latest` - Get latest observation for a HFM series
- `get_hfm_observations` - Get historical observations for a HFM series with date filtering

### General Tools (2 tools)

- `categories` - List all available OFR data categories
- `health` - Get OfrCollector service health status

### Data Collection Admin Tools (6 tools)

- `trigger_fsi_collection` - Trigger immediate FSI data collection
- `trigger_stfm_collection` - Trigger immediate STFM data collection (optionally filtered by dataset)
- `trigger_hfm_collection` - Trigger immediate HFM data collection (optionally filtered by dataset)
- `trigger_fsi_backfill` - Trigger FSI historical data backfill (specify date range)
- `trigger_stfm_series_collection` - Trigger collection for specific STFM series
- `trigger_hfm_series_collection` - Trigger collection for specific HFM series

### STFM Series Management Tools (5 tools)

- `list_stfm_series_admin` - List all STFM series including inactive ones (admin view)
- `add_stfm_series` - Add new STFM series to track (auto-fetches metadata from OFR API)
- `toggle_stfm_series` - Enable or disable STFM series for data collection
- `delete_stfm_series` - Delete STFM series and all observations (use with caution)
- `trigger_stfm_series_backfill` - Trigger historical backfill for specific STFM series

### HFM Series Management Tools (5 tools)

- `list_hfm_series_admin` - List all HFM series including inactive ones (admin view)
- `add_hfm_series` - Add new HFM series to track (auto-fetches metadata from OFR API)
- `toggle_hfm_series` - Enable or disable HFM series for data collection
- `delete_hfm_series` - Delete HFM series and all observations (use with caution)
- `trigger_hfm_series_backfill` - Trigger historical backfill for specific HFM series

## Configuration

### Environment Variables

```bash
OFRCOLLECTOR_API_URL=http://ofr-api:8080
OFRCOLLECTOR_MCP_LOG_LEVEL=Warning
OFRCOLLECTOR_MCP_TIMEOUT_SECONDS=30
```

### Connection

SSE endpoint: `http://mercury:3106/sse`

### Claude Desktop Configuration

```json
{
  "mcpServers": {
    "ofr-collector": {
      "command": "uvx",
      "args": ["mcp-proxy", "http://mercury:3106/sse"]
    }
  }
}
```

Claude Desktop doesn't natively support SSE transport, so `mcp-proxy` bridges stdio↔SSE.

## OFR Data Categories

### Financial Stress Index (FSI)

Comprehensive measure of financial system stress with breakdowns:
- **Credit**: Credit spreads, default risk
- **Equity Valuation**: Stock market volatility and valuations
- **Funding**: Money market and funding stress
- **Safe Assets**: Flight-to-safety indicators
- **Volatility**: Market volatility measures

Regional breakdowns:
- **US**: United States contribution
- **Advanced Economies**: Developed market stress
- **Emerging Markets**: Emerging market stress

### Short-term Funding Monitor (STFM)

Short-term funding market indicators:
- Repo rates (DVP, GCF, tri-party)
- SOFR (Secured Overnight Financing Rate)
- Treasury bill rates
- Money market fund rates
- Commercial paper rates

Example mnemonics: `REPO-DVP-AR-TOT-P`, `SOFR-AVG-30`, `TBILL-3M`

### Hedge Fund Monitor (HFM)

Hedge fund leverage and risk indicators based on SEC filings.

## Technology Stack

- **.NET 9 / C# 13** - Consistent with ATLAS platform
- **MCP Transport**: SSE (Server-Sent Events over HTTP)
- **HTTP Client**: `HttpClient`
- **Port**: 3106 (mapped from internal 8080)
