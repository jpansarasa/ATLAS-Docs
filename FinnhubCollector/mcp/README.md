# FinnhubMcp

MCP server providing Claude Desktop and Claude Code direct access to ATLAS stock market data and economic calendar events from Finnhub.

## Overview

Exposes FinnhubCollector REST API as MCP tools, enabling AI assistants to query real-time stock quotes, economic calendar events (FOMC, CPI, etc.), earnings calendars, news sentiment, and analyst data. Includes both tracked series and live API pass-through for ad-hoc queries on any stock symbol.

## Architecture

```mermaid
flowchart LR
    AI[AI Assistant<br/>Claude Desktop/Code] -->|MCP/SSE| MCP[FinnhubMcp<br/>:3105]
    MCP -->|HTTP| API[finnhub-collector<br/>:8080]
    API -->|SQL| DB[(TimescaleDB)]
    API -->|HTTP| Finnhub[Finnhub API]
```

## MCP Tools

### Data Query Tools (15 tools)

| Tool | Description | Parameters |
|------|-------------|------------|
| `health` | Get FinnhubCollector service health status | - |
| `get_series` | List all configured Finnhub series | `type` (optional) |
| `get_quote` | Get latest quote for a tracked symbol | `symbol` |
| `get_quote_history` | Get historical quotes for a tracked symbol | `symbol`, `from`, `to` |
| `get_economic_calendar` | Get upcoming economic calendar events | `days` |
| `get_high_impact_events` | Get high-impact economic events only | `from`, `to` |
| `get_earnings_calendar` | Get upcoming earnings announcements | `days` |
| `get_ipo_calendar` | Get upcoming IPOs | `days` |
| `get_news_sentiment` | Get news sentiment analysis | `symbol` |
| `get_insider_sentiment` | Get insider buying/selling activity | `symbol` |
| `get_recommendations` | Get analyst recommendations | `symbol` |
| `get_price_target` | Get analyst price targets | `symbol` |
| `get_company_profile` | Get company profile information | `symbol` |
| `get_market_status` | Check if market is currently open | `exchange` |
| `search_symbols` | Search for stock symbols | `query` |

### Live Data Tools (7 tools)

Query any stock symbol directly from Finnhub API (not limited to tracked series):

| Tool | Description | Parameters |
|------|-------------|------------|
| `get_live_quote` | Get live quote for any symbol | `symbol` |
| `get_live_candles` | Get historical price candles | `symbol`, `resolution`, `days` |
| `get_live_profile` | Get company profile for any symbol | `symbol` |
| `get_live_recommendation` | Get analyst recommendations | `symbol` |
| `get_live_price_target` | Get analyst price target | `symbol` |
| `get_live_news_sentiment` | Get news sentiment | `symbol` |
| `get_live_peers` | Get company peers | `symbol` |

### Admin Tools (5 tools)

| Tool | Description | Parameters |
|------|-------------|------------|
| `get_all_series_admin` | Get all series including inactive | - |
| `add_series` | Add new series to track | `symbol`, `type`, `category`, `poll_interval_seconds` |
| `toggle_series` | Enable/disable series collection | `series_id` |
| `delete_series` | Delete series (destructive) | `series_id` |
| `trigger_collection` | Trigger immediate data collection | `series_id` |

### Series Types

- `Quote` - Real-time stock quotes
- `Candle` - Historical price candles
- `NewsSentiment` - News sentiment analysis
- `Recommendation` - Analyst recommendations
- `PriceTarget` - Analyst price targets
- `CompanyProfile` - Company information
- `EconomicCalendar` - Economic calendar events
- `EarningsCalendar` - Earnings announcements
- `IpoCalendar` - IPO events

## Configuration

| Variable | Default | Description |
|----------|---------|-------------|
| `FINNHUB_API_URL` | `http://finnhub-collector:8080` | Backend service URL |
| `FINNHUB_MCP_LOG_LEVEL` | `Warning` | Logging level |
| `FINNHUB_MCP_TIMEOUT_SECONDS` | `30` | HTTP request timeout |

### Port Mapping

- Internal: 8080
- External (host): 3105
- SSE endpoint: `http://mercury:3105/sse`

## Project Structure

```
FinnhubCollector/mcp/
├── Client/
│   ├── FinnhubCollectorClient.cs   # HTTP client for collector API
│   └── IFinnhubCollectorClient.cs  # Client interface
├── Tools/
│   └── FinnhubTools.cs             # MCP tool definitions (27 tools)
├── Program.cs                       # Entry point, MCP server setup
├── Containerfile                    # Container image definition
├── FinnhubMcp.csproj               # Project file
└── README.md
```

## Development

### Prerequisites

- .NET 9 SDK
- Access to finnhub-collector service

### Build

```bash
FinnhubCollector/.devcontainer/compile.sh
```

### Build Container

```bash
FinnhubCollector/.devcontainer/build.sh
```

## Deployment

```bash
ansible-playbook playbooks/deploy.yml --tags finnhub-mcp
```

## Claude Desktop Integration

Add to `~/.config/Claude/claude_desktop_config.json` (Linux) or `~/Library/Application Support/Claude/claude_desktop_config.json` (macOS):

```json
{
  "mcpServers": {
    "finnhub": {
      "command": "uvx",
      "args": ["mcp-proxy", "http://mercury:3105/sse"]
    }
  }
}
```

Claude Desktop uses stdio transport, so `mcp-proxy` bridges stdio to SSE.

## Usage Examples

**Check stock price:**
```
User: "What's Apple trading at?"
Claude calls: get_live_quote(symbol="AAPL")
```

**Economic calendar:**
```
User: "When's the next FOMC meeting?"
Claude calls: get_high_impact_events()
```

**Earnings schedule:**
```
User: "Who reports earnings this week?"
Claude calls: get_earnings_calendar(days=7)
```

## See Also

- [FinnhubCollector](../README.md) - Backend service
- [SecMaster MCP](../../SecMaster/mcp/README.md) - Instrument metadata and search
- [Model Context Protocol](https://modelcontextprotocol.io/) - MCP specification
