# OfrMcp

MCP server providing Claude Desktop and Claude Code access to OFR financial stress and funding market data.

## Overview

Exposes OfrCollector REST API as MCP tools, enabling AI assistants to query the Financial Stress Index (FSI), Short-term Funding Monitor (STFM), and Hedge Fund Monitor (HFM) data. All data is pre-collected in TimescaleDB for sub-second response times.

## Architecture

```mermaid
flowchart LR
    AI[AI Assistant<br/>Claude Desktop/Code] -->|MCP/SSE| MCP[OfrMcp<br/>:3106]
    MCP -->|HTTP| API[ofr-collector<br/>:8080]
    API -->|SQL| DB[(TimescaleDB)]
```

## Features

- **FSI Access**: Financial Stress Index with category/regional breakdowns
- **STFM Data**: Repo rates, SOFR, T-bills, money market and commercial paper rates
- **HFM Data**: Hedge fund leverage and risk indicators from SEC filings
- **Admin Tools**: Collection triggers, series management, backfill operations
- **SSE Transport**: Server-Sent Events for Claude Desktop integration

## Configuration

| Variable | Description | Default |
|----------|-------------|---------|
| `OFRCOLLECTOR_API_URL` | Backend service URL | `http://ofr-collector:8080` |
| `OFRCOLLECTOR_MCP_TIMEOUT_SECONDS` | HTTP request timeout | `30` |

## MCP Tools

### FSI Tools (Financial Stress Index)

| Tool | Description | Parameters |
|------|-------------|------------|
| `get_fsi_latest` | Get latest FSI with category/regional breakdowns | None |
| `get_fsi_history` | Get historical FSI observations | `start_date`, `end_date`, `limit` |

FSI breakdowns: Credit, Equity Valuation, Funding, Safe Assets, Volatility (categories); US, Advanced Economies, Emerging Markets (regions).

### STFM Tools (Short-term Funding Monitor)

| Tool | Description | Parameters |
|------|-------------|------------|
| `list_stfm_series` | List all active STFM series | None |
| `get_stfm_latest` | Get latest observation for a series | `mnemonic` |
| `get_stfm_observations` | Get historical observations | `mnemonic`, `start_date`, `end_date`, `limit` |

STFM tracks: Repo rates (DVP, GCF, tri-party), SOFR, T-bill rates, money market fund rates, commercial paper rates.

### HFM Tools (Hedge Fund Monitor)

| Tool | Description | Parameters |
|------|-------------|------------|
| `list_hfm_series` | List all active HFM series | None |
| `get_hfm_latest` | Get latest observation for a series | `mnemonic` |
| `get_hfm_observations` | Get historical observations | `mnemonic`, `start_date`, `end_date`, `limit` |

HFM tracks hedge fund leverage and risk indicators from SEC filings.

### General Tools

| Tool | Description | Parameters |
|------|-------------|------------|
| `categories` | List all available OFR data categories | None |
| `health` | Get OfrCollector service health status | None |

### Admin Tools (Collection Management)

| Tool | Description | Parameters |
|------|-------------|------------|
| `trigger_fsi_collection` | Trigger immediate FSI collection | None |
| `trigger_fsi_backfill` | Trigger FSI historical backfill | `start_date`, `end_date` |
| `trigger_stfm_collection` | Trigger STFM collection | `dataset` (optional) |
| `trigger_hfm_collection` | Trigger HFM collection | `dataset` (optional) |

### Admin Tools (Series Management)

| Tool | Description | Parameters |
|------|-------------|------------|
| `add_stfm_series` | Add new STFM series | `mnemonic`, `backfill` |
| `toggle_stfm_series` | Enable/disable STFM series | `mnemonic` |
| `delete_stfm_series` | Delete STFM series and data | `mnemonic` |
| `trigger_stfm_series_collection` | Trigger collection for specific series | `mnemonic` |
| `trigger_stfm_series_backfill` | Trigger backfill for specific series | `mnemonic`, `start_date`, `end_date` |
| `list_stfm_series_admin` | List all STFM series (including inactive) | None |
| `add_hfm_series` | Add new HFM series | `mnemonic`, `backfill` |
| `toggle_hfm_series` | Enable/disable HFM series | `mnemonic` |
| `delete_hfm_series` | Delete HFM series and data | `mnemonic` |
| `trigger_hfm_series_collection` | Trigger collection for specific series | `mnemonic` |
| `trigger_hfm_series_backfill` | Trigger backfill for specific series | `mnemonic`, `start_date`, `end_date` |
| `list_hfm_series_admin` | List all HFM series (including inactive) | None |

## Project Structure

```
OfrCollector/
├── mcp/
│   ├── Client/           # HTTP client for OfrCollector API
│   ├── Tools/            # MCP tool definitions
│   ├── Program.cs        # Application entry point
│   └── Containerfile     # Container build definition
├── src/                  # Main OfrCollector service
├── tests/                # Unit tests
└── migrations/           # Database migrations
```

## Development

### Prerequisites
- VS Code with Dev Containers extension
- Access to shared infrastructure (TimescaleDB, observability stack)

### Getting Started

1. Open in VS Code: `code OfrCollector/`
2. Reopen in Container (Cmd/Ctrl+Shift+P -> "Dev Containers: Reopen in Container")
3. Build: `dotnet build mcp/OfrMcp.csproj`
4. Run: `dotnet run --project mcp/OfrMcp.csproj`

### Build Container

```bash
cd /home/james/ATLAS
nerdctl build -t ofr-mcp:latest -f OfrCollector/mcp/Containerfile .
```

## Deployment

```bash
ansible-playbook playbooks/deploy.yml --tags ofr-mcp
```

## Ports

| Port | Description |
|------|-------------|
| 8080 | REST API (internal) |
| 3106 | Host-mapped SSE endpoint |

SSE endpoint: `http://mercury:3106/sse`

## Claude Desktop Integration

Add to `~/.config/Claude/claude_desktop_config.json` (Linux) or `~/Library/Application Support/Claude/claude_desktop_config.json` (macOS):

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

Claude Desktop uses stdio transport; `mcp-proxy` bridges stdio to SSE.

## See Also

- [OfrCollector](../README.md) - Backend service documentation
- [Model Context Protocol](https://modelcontextprotocol.io/) - MCP specification
