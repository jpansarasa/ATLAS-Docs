# CLAUDE.md [FREDCOLLECTOR.MCP v1.0]

## PROJECT [identity]
TYPE: MCP_server(stdio)
WRAPS: FredCollector.Api(:5001)
PURPOSE: Claude↔ATLAS_data_access

## STACK [required]
RUNTIME: .NET9 | C#13 | net9.0
TRANSPORT: stdio(JSON-RPC_2.0)
HTTP: HttpClient + Polly(resilience)
SERIALIZE: System.Text.Json

## STRUCTURE [layout]
```
FredCollectorMcp/
├── src/FredCollectorMcp/
│   ├── Program.cs              # entry + stdio_loop
│   ├── McpServer.cs            # JSON-RPC_dispatch
│   ├── Tools/                  # IMcpTool_impls
│   │   ├── ListSeriesTool.cs
│   │   ├── GetLatestTool.cs
│   │   └── ...
│   ├── Client/
│   │   └── FredCollectorClient.cs
│   └── Models/
├── tests/
└── FredCollectorMcp.sln
```

## TOOL_PATTERN [impl_template]
```csharp
public sealed class GetLatestTool(IFredCollectorClient client) : IMcpTool
{
    public string Name => "fred_get_latest";
    public string Description => "Get most recent observation for series";
    
    public JsonElement InputSchema => JsonSerializer.SerializeToElement(new
    {
        type = "object",
        properties = new { series_id = new { type = "string" } },
        required = new[] { "series_id" }
    });
    
    public async Task<ToolResult> ExecuteAsync(JsonElement args, CancellationToken ct)
    {
        var id = args.GetProperty("series_id").GetString()!;
        var result = await client.GetLatestAsync(id, ct);
        return ToolResult.Success(Format(result));
    }
}
```

## CONVENTIONS [inherit_ATLAS]
nullable: enable
namespace: file_scoped
constructors: primary
dto: record
compose: compose.yaml ¬ docker-compose.yml
container: Containerfile ¬ Dockerfile

## ENV [config]
```
FREDCOLLECTOR_API_URL=http://mercury:5001
FREDCOLLECTOR_MCP_LOG_LEVEL=Information
FREDCOLLECTOR_MCP_TIMEOUT_SECONDS=30
```

## TEST [strategy]
unit: mock(IFredCollectorClient) → tool_logic
integration: live_api → client_behavior
mcp: protocol_compliance

## REFS [external]
MCP_SPEC: https://modelcontextprotocol.io/docs
API_LIVE: http://mercury:5001/swagger/v1/swagger.json
ATLAS_CONVENTIONS: ../CLAUDE.md
