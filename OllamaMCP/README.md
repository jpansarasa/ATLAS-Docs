# OllamaMCP

MCP server providing Claude Desktop and Claude Code access to local Ollama LLM instances for text generation, chat, and model management.

## Overview

OllamaMCP bridges AI assistants to self-hosted Ollama instances via the Model Context Protocol (MCP) over SSE transport. It enables local LLM inference without external API dependencies, supporting both GPU-accelerated and CPU-only instances with dynamic per-request routing via the `use_gpu` parameter.

## Architecture

```mermaid
flowchart LR
    AI[AI Assistant<br/>Claude Desktop/Code] -->|MCP/SSE| MCP[OllamaMCP<br/>:3100]
    MCP -->|use_gpu=true| GPU[ollama-gpu<br/>GPU Instance]
    MCP -->|use_gpu=false| CPU[ollama-cpu<br/>CPU Instance]
```

Requests arrive over SSE, are parsed as JSON-RPC MCP messages, and forwarded to the appropriate Ollama instance based on the `use_gpu` parameter (defaults to GPU).

## Features

- **Dual Instance Routing**: Dynamic GPU/CPU selection per request via `use_gpu` parameter
- **Text Generation**: Single-prompt completions via Ollama's `/api/generate`
- **Multi-turn Chat**: Conversational context with message history via `/api/chat`
- **Model Management**: List, pull, and inspect models on either instance
- **SSE Transport**: Server-Sent Events with 30-second heartbeat keepalive

## Configuration

| Variable | Default | Description |
|----------|---------|-------------|
| `OLLAMA_GPU_URL` | `http://ollama-gpu:11434` | GPU Ollama instance URL |
| `OLLAMA_CPU_URL` | `http://ollama-cpu:11435` | CPU Ollama instance URL |
| `MCP_PORT` | `3100` | SSE server port |
| `MCP_HOST` | `0.0.0.0` | Bind address |

## API (MCP Tools)

All tools accept an optional `use_gpu` boolean parameter (default: `true`) for instance routing.

### Generation Tools

| Tool | Description | Required Parameters |
|------|-------------|---------------------|
| `ollama_generate` | Text completion | `model`, `prompt` |
| `ollama_chat` | Multi-turn chat | `model`, `messages` |

Optional parameters for generation tools: `temperature` (0.0-2.0, default 0.7), `system` (generate only).

### Model Management Tools

| Tool | Description | Required Parameters |
|------|-------------|---------------------|
| `ollama_list_models` | List available models | none |
| `ollama_pull_model` | Download a model | `model` |
| `ollama_model_info` | Model metadata | `model` |

### Chat Message Format

```json
{
  "messages": [
    { "role": "system", "content": "You are a helpful assistant." },
    { "role": "user", "content": "Hello!" }
  ]
}
```

## Project Structure

```
OllamaMCP/
├── Program.cs           # MCP server, tool definitions, Ollama client
├── OllamaMcp.csproj     # .NET 10 project file
├── Containerfile        # Container build definition
└── .devcontainer/
    ├── compile.sh       # Build script
    ├── build.sh         # Container image build script
    └── devcontainer.json
```

## Development

### Prerequisites

- VS Code with Dev Containers extension
- Access to Ollama instances (GPU and/or CPU)

### Getting Started

1. Open in VS Code: `code OllamaMCP/`
2. Reopen in Container (Cmd/Ctrl+Shift+P -> "Dev Containers: Reopen in Container")
3. Build: `.devcontainer/compile.sh`

### Build Container

```bash
.devcontainer/build.sh
```

## Deployment

```bash
ansible-playbook playbooks/deploy.yml --tags ollama-mcp
```

## Ports

| Port | Description |
|------|-------------|
| 3100 | SSE server (internal and host-mapped) |

SSE endpoint: `http://mercury:3100/sse`

## Claude Desktop Integration

Add to `~/.config/Claude/claude_desktop_config.json`:

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

Claude Desktop uses stdio transport; `mcp-proxy` bridges stdio to SSE.

## See Also

- [Ollama Documentation](https://github.com/ollama/ollama/tree/main/docs)
- [MCP Specification](https://modelcontextprotocol.io/)
- [SecMaster MCP](../SecMaster/mcp/README.md) - Instrument search and metadata
- [FredCollector MCP](../FredCollector/mcp/README.md) - FRED data access
