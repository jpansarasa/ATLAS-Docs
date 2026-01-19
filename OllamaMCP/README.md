# OllamaMCP

MCP server providing Claude Desktop and Claude Code access to local Ollama LLM instances for text generation, chat, and model management.

## Overview

Bridges AI assistants to self-hosted Ollama instances, enabling local LLM inference without external API dependencies. Supports both GPU-accelerated and CPU fallback instances with dynamic routing via the `use_gpu` parameter on each tool call.

## Architecture

```mermaid
flowchart LR
    AI[AI Assistant<br/>Claude Desktop/Code] -->|MCP/SSE| MCP[OllamaMCP<br/>:3100]
    MCP -->|use_gpu=true| GPU[ollama-gpu<br/>:11434<br/>RTX 5090]
    MCP -->|use_gpu=false| CPU[ollama-cpu<br/>:11435<br/>CPU Only]
```

## MCP Tools

### Generation Tools

| Tool Name | Description | Key Parameters |
|-----------|-------------|----------------|
| `ollama_generate` | Generate text completion | `model`, `prompt`, `system`, `temperature`, `use_gpu` |
| `ollama_chat` | Multi-turn chat with context | `model`, `messages`, `temperature`, `use_gpu` |

### Model Management Tools

| Tool Name | Description | Key Parameters |
|-----------|-------------|----------------|
| `ollama_list_models` | List available models on instance | `use_gpu` |
| `ollama_pull_model` | Download model from Ollama library | `model`, `use_gpu` |
| `ollama_model_info` | Get detailed model metadata | `model`, `use_gpu` |

### Chat Message Format

```json
{
  "messages": [
    { "role": "system", "content": "You are a helpful assistant." },
    { "role": "user", "content": "Hello!" },
    { "role": "assistant", "content": "Hi there!" },
    { "role": "user", "content": "What can you do?" }
  ]
}
```

## Configuration

### Environment Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `OLLAMA_GPU_URL` | `http://ollama-gpu:11434` | GPU instance URL |
| `OLLAMA_CPU_URL` | `http://ollama-cpu:11435` | CPU instance URL |
| `MCP_PORT` | `3100` | SSE server port |
| `MCP_HOST` | `0.0.0.0` | Bind address |

### Port Mapping

- Internal: 3100
- External (host): 3100
- SSE endpoint: `http://mercury:3100/sse`

## Project Structure

```
OllamaMCP/
  Program.cs           # MCP server, tool definitions, Ollama client
  OllamaMcp.csproj     # .NET 9 project file
  Containerfile        # Container build definition
  .devcontainer/
    compile.sh         # Build script
    build.sh           # Container build script
```

## Development

### Prerequisites

- .NET 9 SDK
- nerdctl/containerd

### Build

```bash
.devcontainer/compile.sh
```

### Build Container

```bash
.devcontainer/build.sh
```

## Deployment

```bash
ansible-playbook playbooks/deploy.yml --tags ollama-mcp
```

## Claude Desktop Integration

Add to `~/.config/Claude/claude_desktop_config.json` (Linux) or `~/Library/Application Support/Claude/claude_desktop_config.json` (macOS):

```json
{
  "mcpServers": {
    "ollama": {
      "command": "uvx",
      "args": ["mcp-proxy", "http://mercury:3100/sse"]
    }
  }
}
```

Claude Desktop uses stdio transport, so `mcp-proxy` bridges stdio to SSE.

## Usage Examples

**Quick generation:**
```
User: "Generate a haiku about coding"
Claude calls: ollama_generate(model="llama3.2:3b", prompt="Write a haiku about coding")
```

**Multi-turn chat:**
```
User: "Review this function using the local model"
Claude calls: ollama_chat(
  model="codellama:13b",
  messages=[
    {"role": "system", "content": "You are a senior code reviewer."},
    {"role": "user", "content": "Review: function add(a,b){return a+b}"}
  ],
  use_gpu=true
)
```

**List available models:**
```
User: "What models do I have locally?"
Claude calls: ollama_list_models(use_gpu=true)
```

**Download new model:**
```
User: "Pull the Mistral model"
Claude calls: ollama_pull_model(model="mistral:7b", use_gpu=true)
```

## See Also

- [Ollama Documentation](https://github.com/ollama/ollama/tree/main/docs)
- [MCP Specification](https://modelcontextprotocol.io/)
- [SecMaster MCP](../SecMaster/mcp/README.md) - Instrument search and metadata
- [FredCollector MCP](../FredCollector/mcp/README.md) - FRED data access
