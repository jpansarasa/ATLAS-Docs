# CLAUDE.md [FREDCOLLECTOR.MCP v1.0]

## PROJECT [identity]
TYPE: MCP_server(SSE)
WRAPS: FredCollector.Api(:8080)
PURPOSE: Claude_Desktop↔ATLAS_data_access

## STACK [required]
RUNTIME: .NET9 | C#13 | net9.0
TRANSPORT: SSE(JSON-RPC_2.0) | /sse + /message
HTTP: HttpClient
SERIALIZE: System.Text.Json
LOGGING: Serilog(Warning)

## STRUCTURE [layout]
```
FredCollectorMcp/
├── src/
│   ├── Program.cs
│   ├── McpServer.cs
│   ├── SseConnectionManager.cs
│   ├── Tools/
│   ├── Client/
│   ├── Models/
│   ├── Containerfile
│   └── FredCollectorMcp.csproj
└── .devcontainer/
```

## CONVENTIONS [inherit_ATLAS]
nullable: enable
namespace: file_scoped
constructors: primary
compose: compose.yaml ¬ docker-compose.yml
container: Containerfile ¬ Dockerfile

## ENV [config]
```
FREDCOLLECTOR_API_URL=http://fred-api:8080
FREDCOLLECTOR_MCP_LOG_LEVEL=Warning
FREDCOLLECTOR_MCP_TIMEOUT_SECONDS=30
```

## REFS [external]
MCP_SPEC: https://modelcontextprotocol.io/docs
API_LIVE: http://mercury:5001/swagger/v1/swagger.json (fred-api:8080 internally)
ATLAS_CONVENTIONS: ../CLAUDE.md
