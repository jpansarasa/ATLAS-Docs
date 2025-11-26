# CLAUDE.md [THRESHOLDENGINE.MCP v1.0]

## PROJECT [identity]
TYPE: MCP_server(stdio)
WRAPS: ThresholdEngine.Service(:5003)
PURPOSE: Claude↔ATLAS_signals_access

## STACK [required]
RUNTIME: .NET9 | C#13 | net9.0
TRANSPORT: stdio(JSON-RPC_2.0)
HTTP: HttpClient + Polly(resilience)
SERIALIZE: System.Text.Json

## STRUCTURE [layout]
```
ThresholdEngineMcp/
├── src/ThresholdEngineMcp/
│   ├── Program.cs              # entry + stdio_loop
│   ├── McpServer.cs            # JSON-RPC_dispatch
│   ├── Tools/
│   │   ├── EvaluateTool.cs     # ⭐ primary_tool
│   │   ├── EvaluatePatternTool.cs
│   │   ├── ListPatternsTool.cs
│   │   └── ...
│   ├── Client/
│   │   └── ThresholdEngineClient.cs
│   └── Models/
├── tests/
└── ThresholdEngineMcp.sln
```

## TOOL_PATTERN [impl_template]
```csharp
public sealed class EvaluateTool(IThresholdEngineClient client) : IMcpTool
{
    public string Name => "threshold_evaluate";
    public string Description => "Evaluate ALL patterns → regime + score + signals";
    
    public JsonElement InputSchema => JsonSerializer.SerializeToElement(new
    {
        type = "object",
        properties = new { },
        required = Array.Empty<string>()
    });
    
    public async Task<ToolResult> ExecuteAsync(JsonElement args, CancellationToken ct)
    {
        var result = await client.EvaluateAllAsync(ct);
        return ToolResult.Success(FormatEvaluation(result));
    }
}
```

## PRIMARY_TOOL [threshold_evaluate]
PURPOSE: complete_state(one_call)
RETURNS: regime + macroScore + triggeredPatterns[] + categoryBreakdown
ENDPOINT: POST /api/patterns/evaluate
REPLACES: multiple_FRED_lookups + manual_interpretation

## SIGNAL_SCALE [interpretation]
```
-2.0 = strongly_bearish  → increase_defensive
-1.0 = moderately_bearish → monitor_closely
 0.0 = neutral           → no_action
+1.0 = moderately_bullish → consider_risk_on
+2.0 = strongly_bullish  → deploy_capital
```

## REGIME_SCALE [interpretation]
```
Crisis:    < -20  → 80%+ defensive
Recession: -20..-10 → 70-80% defensive
LateCycle: -10..0  → 60-70% defensive  # CURRENT
Neutral:   0..+10  → 50-60% defensive
Recovery:  +10..+20 → 40-50% defensive
Growth:    > +20   → 30-40% defensive
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
THRESHOLDENGINE_API_URL=http://mercury:5003
THRESHOLDENGINE_MCP_LOG_LEVEL=Information
THRESHOLDENGINE_MCP_TIMEOUT_SECONDS=30
```

## TEST [strategy]
unit: mock(IThresholdEngineClient) → tool_logic
integration: live_api → client_behavior
mcp: protocol_compliance

## REFS [external]
MCP_SPEC: https://modelcontextprotocol.io/docs
API_LIVE: http://mercury:5003/swagger/v1/swagger.json
ATLAS_CONVENTIONS: ../CLAUDE.md
