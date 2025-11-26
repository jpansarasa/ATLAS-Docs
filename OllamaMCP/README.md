# Ollama MCP Server

MCP server exposing Ollama (GPU/CPU) as tools for Claude Desktop.

**Port**: 3100 (SSE)  
**Tools**: `ollama_generate`, `ollama_chat`, `ollama_list_models`, `ollama_pull_model`

## Deploy

Via `infrastructure/compose.yaml`:
```yaml
ollama-mcp:
  image: ollama-mcp:latest
  ports: ["3100:3100"]
  environment:
    OLLAMA_GPU_URL: http://ollama-gpu:11434
    OLLAMA_CPU_URL: http://ollama-cpu:11434
```

## Configure Claude Desktop

`~/.config/Claude/claude_desktop_config.json`:
```json
{
  "mcpServers": {
    "ollama-local": {
      "command": "uvx",
      "args": ["mcp-proxy", "http://yourserver:3100/sse"]
    }
  }
}
```

## Test

```bash
curl -X POST http://localhost:3100/messages/ \
  -H "Content-Type: application/json" \
  -d '{"method":"tools/call","params":{"name":"ollama_list_models"}}'
```
