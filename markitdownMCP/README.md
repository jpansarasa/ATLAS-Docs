# markitdown MCP

Converts documents (PDF, Office) to markdown for ATLAS analysis.

**Image**: `mcp/markitdown:latest`  
**Port**: 3102  
**Storage**: `/opt/ai-inference/documents` (ZFS: `sata-bulk/ai-inference/documents`)

## Deploy

Add to `infrastructure/compose.yaml`:
```yaml
markitdown-mcp:
  image: mcp/markitdown:latest
  ports: ["3102:3102"]
  volumes: ["/opt/ai-inference/documents:/workdir:ro"]
  command: ["--sse", "--host", "0.0.0.0", "--port", "3102"]
```

## Use Cases

- Portfolio reconciliation: Brokerage PDFs → positions
- NBFI monitoring: Bankruptcy filings → stress signals
- Fed analysis: FOMC/H.4.1 PDFs → policy shifts

## MCP Integration

**ATLAS Core** (server-side):
```json
{
  "MCP": {
    "Servers": {
      "markitdown": {
        "transport": "sse",
        "url": "http://markitdown-mcp:3102"
      }
    }
  }
}
```

**Claude Desktop** (local):
```json
{
  "mcpServers": {
    "markitdown": {
      "command": "uvx",
      "args": ["markitdown-mcp"]
    }
  }
}
```
