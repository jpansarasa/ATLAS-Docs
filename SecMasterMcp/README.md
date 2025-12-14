# SecMasterMcp

Model Context Protocol (MCP) server for the SecMaster service - provides AI assistants with tools to search, resolve, and query financial instrument metadata.

## Overview

SecMasterMcp exposes SecMaster's functionality through standardized MCP tools, enabling AI assistants to:
- Search instruments by name, symbol, or natural language
- Resolve symbols to data sources with context-aware routing
- Query instrument metadata and source mappings
- Perform semantic searches using vector similarity
- Ask natural language questions with RAG synthesis
- Search across all data collectors with smart routing
- List and manage collector series (FRED, Finnhub, AlphaVantage, OFR)

## Architecture

```mermaid
flowchart LR
    AI["AI Client<br/>(Claude)"] -->|MCP| MCP["SecMasterMcp<br/>(Port 3107)"]
    MCP -->|HTTP| SM["SecMaster<br/>(Port 5017)"]
```

## MCP Tools (25 Tools)

### Basic Search & Resolution

#### `search_instruments`
Search for instruments by name, symbol, or description.

**Parameters:**
- `query` (required): Search query (e.g., 'unemployment', 'AAPL')
- `asset_class` (optional): Filter by asset class (Economic, Equity, Commodity, Index, Currency, Rate)
- `limit` (optional): Maximum results (default: 20)

**Example:**
```json
{
  "query": "unemployment",
  "asset_class": "Economic",
  "limit": 10
}
```

#### `get_instrument`
Get detailed information about an instrument by symbol or ID.

**Parameters:**
- `identifier` (required): Symbol like 'UNRATE' or UUID

**Returns:** Full instrument details including source mappings, aliases, and external identifiers (FIGI, CUSIP, SEDOL, ISIN).

#### `resolve_source`
Resolve a symbol to the best data source based on context.

**Parameters:**
- `symbol` (required): Symbol to resolve
- `frequency` (optional): Required frequency (any, intraday, daily, weekly, monthly, quarterly, annual)
- `max_lag` (optional): Maximum publication lag in days
- `prefer_collector` (optional): Preferred collector name

**Returns:** Resolved source with alternatives.

#### `resolve_batch`
Resolve multiple symbols in a single request.

**Parameters:**
- `symbols` (required): Comma-separated list (e.g., 'UNRATE,SOFR,GDP')
- `frequency` (optional): Required frequency

#### `list_sources`
List all data sources available for an instrument.

**Parameters:**
- `symbol` (required): Symbol to look up

#### `lookup_by_collector_id`
Reverse lookup: find instrument by collector-specific ID.

**Parameters:**
- `collector` (required): Collector name (e.g., 'FredCollector')
- `source_id` (required): Collector-specific ID (e.g., 'UNRATE')

### Semantic Search (Vector-based)

#### `semantic_search`
Search instruments using semantic similarity - understands natural language queries.

**Parameters:**
- `query` (required): Natural language query (e.g., 'economic indicator for job market health', 'inflation measures')
- `min_score` (optional): Minimum similarity score 0-1 (default: 0.5, higher = more relevant)
- `limit` (optional): Maximum results (default: 10)

**Example:**
```json
{
  "query": "What measures stock market volatility?",
  "min_score": 0.6,
  "limit": 5
}
```

**How it works:** Converts query to 768-dimensional embedding using `nomic-embed-text` model, then performs cosine similarity search against instrument embeddings in pgvector.

#### `ask_secmaster`
Ask a natural language question and get a synthesized answer using RAG.

**Parameters:**
- `question` (required): Question about instruments (e.g., 'What data do you have for tracking inflation?')

**Returns:** Synthesized answer from `llama3.2:3b` model with relevant instrument matches.

**Example:**
```json
{
  "question": "Which instruments measure unemployment?"
}
```

#### `hybrid_resolve`
Advanced resolution using multi-strategy approach: SQL → Fuzzy → Vector → RAG.

**Parameters:**
- `query` (required): Can be exact symbol or natural language
- `enable_rag` (optional): Enable RAG synthesis for ambiguous queries (default: true)
- `min_score` (optional): Minimum vector similarity 0-1 (default: 0.5)
- `limit` (optional): Maximum vector results (default: 5)

**Resolution Strategy:**
1. **SQL**: Try exact symbol match
2. **Fuzzy**: Try fuzzy text search (pg_trgm)
3. **Vector**: Semantic similarity search
4. **RAG**: Natural language synthesis if no clear match

**Returns:** Resolution with method used (sql/fuzzy/vector/rag) and relevant results.

### Health

#### `health`
Get SecMaster service health status.

**Returns:** Health status and check results.

### Collector Gateway (15 Tools)

SecMasterMcp provides unified access to all data collectors through smart routing and management tools.

#### `search_collectors`
Search across all data collectors (FRED, Finnhub, OFR, AlphaVantage) using smart routing.

**Parameters:**
- `query` (required): Search query (e.g., 'unemployment', 'AAPL', 'stress')
- `limit` (optional): Maximum results (default: 20)

**Returns:** Aggregated results from relevant collectors with source attribution.

**Example:**
```json
{
  "query": "unemployment",
  "limit": 10
}
```

**Smart Routing:** The search analyzes the query and routes to appropriate collectors:
- Economic indicators → FRED
- Equity symbols → Finnhub
- Treasury/funding rates → OFR
- Commodity/currency data → AlphaVantage

#### List Series Tools (5 tools - readonly)

##### `list_fred_series`
List all active FRED series being collected.

**Returns:** Array of FRED series with ID, title, frequency, and active status.

##### `list_finnhub_series`
List all active Finnhub series being collected.

**Returns:** Array of Finnhub series with ID, title, frequency, and active status.

##### `list_ofr_stfm_series`
List all OFR Short-term Funding Monitor series.

**Returns:** Array of OFR STFM series (read-only, managed via config).

##### `list_ofr_hfm_series`
List all OFR Hedge Fund Monitor series.

**Returns:** Array of OFR HFM series (read-only, managed via config).

##### `list_alphavantage_series`
List all active AlphaVantage series being collected.

**Returns:** Array of AlphaVantage series with ID, title, frequency, and active status.

#### Add Series Tools (3 tools)

##### `add_fred_series`
Add a new series to FRED collector.

**Parameters:**
- `series_id` (required): FRED series ID (e.g., 'UNRATE', 'GDP')
- `priority` (optional): Collection priority (default: 10)

**Returns:** Created series details or error if already exists.

##### `add_finnhub_series`
Add a new series to Finnhub collector.

**Parameters:**
- `symbol` (required): Stock symbol (e.g., 'AAPL', 'MSFT')
- `priority` (optional): Collection priority (default: 10)

**Returns:** Created series details or error if already exists.

##### `add_alphavantage_series`
Add a new series to AlphaVantage collector.

**Parameters:**
- `symbol` (required): Symbol (e.g., 'GOLD', 'EUR/USD')
- `type` (required): Data type (e.g., 'commodity', 'forex')
- `title` (optional): Custom title
- `priority` (optional): Collection priority (default: 10)

**Returns:** Created series details or error if already exists.

#### Toggle Series Tools (3 tools)

##### `toggle_fred_series`
Toggle FRED series active/inactive status.

**Parameters:**
- `series_id` (required): FRED series ID

**Returns:** Updated series with new active status.

##### `toggle_finnhub_series`
Toggle Finnhub series active/inactive status.

**Parameters:**
- `series_id` (required): Finnhub series ID

**Returns:** Updated series with new active status.

##### `toggle_alphavantage_series`
Toggle AlphaVantage series active/inactive status.

**Parameters:**
- `series_id` (required): AlphaVantage series ID

**Returns:** Updated series with new active status.

#### Remove Series Tools (3 tools)

##### `remove_fred_series`
Remove series from FRED collector.

**Parameters:**
- `series_id` (required): FRED series ID

**Returns:** Success status.

##### `remove_finnhub_series`
Remove series from Finnhub collector.

**Parameters:**
- `series_id` (required): Finnhub series ID

**Returns:** Success status.

##### `remove_alphavantage_series`
Remove series from AlphaVantage collector.

**Parameters:**
- `series_id` (required): AlphaVantage series ID

**Returns:** Success status.

**Note:** OFR series are read-only and managed through configuration files, so there are no add/toggle/remove tools for OFR.

## Port Mapping

- Internal: 8080
- External (host): 3107
- SSE endpoint: http://mercury:3107/sse

## Configuration

### Environment Variables

```bash
SECMASTER_API_URL=http://secmaster:8080
SECMASTER_MCP_LOG_LEVEL=Warning
SECMASTER_MCP_TIMEOUT_SECONDS=30
```

### Connection

SSE endpoint: `http://mercury:3107/sse`

## Development

### Build
```bash
SecMasterMcp/.devcontainer/compile.sh
```

### Build Container
```bash
SecMasterMcp/.devcontainer/build.sh
```

### Deploy
```bash
ansible-playbook playbooks/deploy.yml --tags secmaster-mcp -i inventory/hosts.yml
```

## Usage with Claude Desktop

Add to Claude Desktop config (`~/Library/Application Support/Claude/claude_desktop_config.json` on macOS):

```json
{
  "mcpServers": {
    "secmaster": {
      "command": "uvx",
      "args": ["mcp-proxy", "http://mercury:3107/sse"]
    }
  }
}
```

Claude Desktop doesn't natively support SSE transport, so `mcp-proxy` bridges stdio↔SSE.

## Example Interactions

### Basic Search
```
User: Search for unemployment data
Tool: search_instruments(query="unemployment", asset_class="Economic")
```

### Semantic Search
```
User: What instruments measure job market health?
Tool: semantic_search(query="job market health", min_score=0.6)
```

### Natural Language Query
```
User: What inflation data is available?
Tool: ask_secmaster(question="What inflation data is available?")
```

### Hybrid Resolution
```
User: Find data for treasury yields
Tool: hybrid_resolve(query="treasury yields", enable_rag=true)
```

## Integration with SecMaster

SecMasterMcp is a thin wrapper that:
1. Exposes SecMaster REST API as MCP tools
2. Handles parameter validation and error handling
3. Formats responses for AI consumption
4. Maps HTTP status codes to appropriate MCP responses

All data comes directly from SecMaster - SecMasterMcp maintains no state.

## Observability

SecMasterMcp emits OpenTelemetry traces for all tool invocations, forwarded to the ATLAS observability stack (Tempo/Loki/Prometheus/Grafana).

## See Also

- [SecMaster README](../SecMaster/README.md) - Core service documentation
- [MCP Specification](https://modelcontextprotocol.io) - Model Context Protocol
