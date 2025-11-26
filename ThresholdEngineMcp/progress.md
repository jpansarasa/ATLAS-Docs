# ThresholdEngineMcp Implementation

## Status: NOT STARTED

## API Endpoints (ThresholdEngine.Service :5003)

| Endpoint | Method | MCP Tool |
|----------|--------|----------|
| `/api/patterns` | GET | `list_patterns`, `categories` |
| `/api/patterns/{id}` | GET | `get_pattern` |
| `/api/patterns/evaluate` | POST | `evaluate` |
| `/api/patterns/{id}/evaluate` | POST | `evaluate_pattern` |
| `/api/patterns/reload` | POST | `reload` |
| `/health` | GET | `health` |
| `/swagger/v1/swagger.json` | GET | `api_schema` |

**Blocked:** `regime_history` needs new API endpoint or audit log query.

## Implementation Order

### 1. Project Scaffold
```
ThresholdEngineMcp/
├── src/ThresholdEngineMcp/
│   ├── Program.cs              # stdio loop
│   ├── McpServer.cs            # JSON-RPC dispatch
│   ├── IMcpTool.cs
│   ├── ToolResult.cs
│   ├── Client/
│   │   ├── IThresholdEngineClient.cs
│   │   └── ThresholdEngineClient.cs
│   ├── Models/                 # API response DTOs
│   └── Tools/                  # One file per tool
└── .devcontainer/
```

### 2. HTTP Client
- `IThresholdEngineClient` interface
- `ThresholdEngineClient` implementation with HttpClient
- Base URL from `THRESHOLDENGINE_API_URL` env var
- Methods: `ListPatternsAsync`, `GetPatternAsync`, `EvaluateAsync`, `EvaluatePatternAsync`, `ReloadAsync`, `GetHealthAsync`

### 3. Tools (priority order)

| Order | Tool | Wraps |
|-------|------|-------|
| 1 | `evaluate` | POST /api/patterns/evaluate |
| 2 | `list_patterns` | GET /api/patterns |
| 3 | `get_pattern` | GET /api/patterns/{id} |
| 4 | `evaluate_pattern` | POST /api/patterns/{id}/evaluate |
| 5 | `categories` | Aggregation from list_patterns |
| 6 | `health` | GET /health |
| 7 | `api_schema` | GET /swagger/v1/swagger.json |
| 8 | `reload` | POST /api/patterns/reload |

### 4. Container & Deployment
- `Containerfile` (copy FredCollectorMcp pattern)
- Add to `infrastructure/compose.yaml.j2`
- Deploy via ansible

## Tool Naming

Use short names without prefix (like FredCollectorMcp):
- `evaluate` not `threshold_evaluate`
- `list_patterns` not `threshold_list_patterns`

Tools are already namespaced within the `threshold-engine` MCP server context.

## Transport

**stdio** - Claude Desktop launches process directly:
```json
{
  "mcpServers": {
    "threshold-engine": {
      "command": "dotnet",
      "args": ["run", "--project", "/path/to/ThresholdEngineMcp"]
    }
  }
}
```

## Reference

Copy patterns from FredCollectorMcp:
- `McpServer.cs` - JSON-RPC handling
- `IMcpTool.cs` / `ToolResult.cs` - Tool interface
- Tool implementations
