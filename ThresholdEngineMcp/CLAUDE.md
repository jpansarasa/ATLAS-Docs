# CLAUDE.md [THRESHOLDENGINE.MCP v1.0]

## PROJECT [identity]
TYPE: MCP_server(SSE)
WRAPS: ThresholdEngine.Service(:8080)
PURPOSE: Claude↔ATLAS_signals_access

## STACK [required]
RUNTIME: .NET9 | C#13 | net9.0
TRANSPORT: SSE(JSON-RPC_2.0) | /sse + /message
HTTP: HttpClient
SERIALIZE: System.Text.Json
LOGGING: Serilog(Warning)

## STRUCTURE [layout]
```
ThresholdEngineMcp/
├── src/
│   ├── Program.cs
│   ├── McpServer.cs
│   ├── SseConnectionManager.cs
│   ├── Tools/
│   ├── Client/
│   ├── Models/
│   ├── Containerfile
│   └── ThresholdEngineMcp.csproj
└── .devcontainer/
```

## TOOLS [8 total]
```
evaluate           # ⭐ primary - regime + score + triggered patterns
list_patterns      # pattern configurations
get_pattern        # single pattern detail
evaluate_pattern   # on-demand single evaluation
categories         # pattern category breakdown
health             # service health
api_schema         # OpenAPI spec
reload             # hot-reload configs
```

## CONVENTIONS [inherit_ATLAS]
nullable: enable
namespace: file_scoped
constructors: primary
compose: compose.yaml ¬ docker-compose.yml
container: Containerfile ¬ Dockerfile

## ENV [config]
```
THRESHOLDENGINE_API_URL=http://threshold-engine:8080
THRESHOLDENGINE_MCP_LOG_LEVEL=Warning
THRESHOLDENGINE_MCP_TIMEOUT_SECONDS=30
```

## REFS [external]
MCP_SPEC: https://modelcontextprotocol.io/docs
API_LIVE: http://mercury:5003/swagger/v1/swagger.json
ATLAS_CONVENTIONS: ../CLAUDE.md
