# FinnhubMcp

MCP server providing Claude direct access to ATLAS stock market data and economic calendar events.

## Overview

Wraps the FinnhubCollector REST API, providing:
- **Real-time Stock Quotes**: Current and historical price data
- **Economic Calendar**: FOMC meetings, CPI releases, employment reports
- **Earnings & IPO Calendars**: Corporate event tracking
- **Market Sentiment**: News sentiment, insider activity, analyst recommendations
- **Live Data Access**: Query any symbol via Finnhub API (not just tracked series)
- **Series Management**: Admin tools to configure data collection

## Available Tools (26 Tools)

### Data Query Tools (14 tools)

- `health` - Get FinnhubCollector service health status
- `get_series` - List all configured Finnhub series (filter by type)
- `get_quote` - Get latest quote for a tracked stock symbol
- `get_quote_history` - Get historical quotes for a tracked symbol
- `get_economic_calendar` - Get upcoming economic calendar events (FOMC, CPI, etc.)
- `get_high_impact_events` - Get high-impact economic events (Fed decisions, major reports)
- `get_earnings_calendar` - Get upcoming earnings announcements
- `get_ipo_calendar` - Get upcoming IPOs
- `get_news_sentiment` - Get news sentiment analysis for a stock
- `get_insider_sentiment` - Get insider buying/selling activity
- `get_recommendations` - Get analyst recommendations (buy/hold/sell)
- `get_price_target` - Get analyst price targets
- `get_company_profile` - Get company profile information
- `get_market_status` - Check if stock market is currently open
- `search_symbols` - Search for stock symbols by company name

### Live Data Tools (7 tools)

Query any stock symbol directly from Finnhub API (not limited to tracked series):

- `get_live_quote` - Get live quote for any symbol
- `get_live_candles` - Get historical price candles (configurable resolution: 1m, 5m, D, W, M)
- `get_live_profile` - Get company profile for any symbol
- `get_live_recommendation` - Get analyst recommendations for any symbol
- `get_live_price_target` - Get analyst price target for any symbol
- `get_live_news_sentiment` - Get news sentiment for any symbol
- `get_live_peers` - Get company peers for any symbol

### Admin Tools (5 tools)

- `get_all_series_admin` - Get all configured series including inactive ones
- `add_series` - Add new series to track (Quote, Candle, NewsSentiment, Recommendation, etc.)
- `toggle_series` - Enable or disable a series for data collection
- `delete_series` - Delete a series (use with caution)
- `trigger_collection` - Trigger immediate data collection for a series

## Configuration

### Environment Variables

```bash
FINNHUB_API_URL=http://finnhub-collector:8080
FINNHUB_MCP_LOG_LEVEL=Warning
FINNHUB_MCP_TIMEOUT_SECONDS=30
```

### Connection

SSE endpoint: `http://mercury:3105/sse`

### Claude Desktop Configuration

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

Claude Desktop doesn't natively support SSE transport, so `mcp-proxy` bridges stdio↔SSE.

## Port Mapping

- Internal: 8080
- External (host): 3105
- SSE endpoint: http://mercury:3105/sse

## Series Types

Tracked series can be of type:
- `Quote` - Real-time stock quotes
- `Candle` - Historical price candles
- `NewsSentiment` - News sentiment analysis
- `Recommendation` - Analyst recommendations
- `PriceTarget` - Analyst price targets
- `CompanyProfile` - Company profile information
- `EconomicCalendar` - Economic calendar events
- `EarningsCalendar` - Earnings announcements
- `IpoCalendar` - IPO events

## Technology Stack

- **.NET 9 / C# 13** - Consistent with ATLAS platform
- **MCP Transport**: SSE (Server-Sent Events over HTTP)
- **HTTP Client**: `HttpClient`
- **Port**: 3105 (mapped from internal 8080)
