# markitdownMCP

MCP server providing document-to-markdown conversion for ATLAS AI workflows.

## Overview

Wraps Microsoft's [markitdown](https://github.com/microsoft/markitdown) Python library as an MCP server, enabling AI assistants and ATLAS services to convert PDFs, Office documents, HTML, and other formats to markdown. Runs the upstream `mcp/markitdown:latest` container image unchanged -- this directory contains no source code, no `Containerfile`, and no build step. Consumed by `sentinel-collector` (HTTP/JSON-RPC) for content normalization in the extraction pipeline, and optionally by Claude Desktop/Code (SSE via `mcp-proxy`) for ad-hoc document conversion.

## Architecture

```mermaid
flowchart LR
    subgraph Consumers
        AI[Claude Desktop/Code<br/>mcp-proxy stdio→SSE]
        SC[sentinel-collector<br/>HTTP JSON-RPC]
    end

    subgraph markitdownMCP
        MCP[markitdown-mcp<br/>:3102<br/>StreamableHTTP + SSE]
    end

    subgraph Mounts[Read-only Mounts]
        Docs[/opt/ai-inference/documents<br/>→ /workdir]
        Raw[/opt/ai-inference/raw-data<br/>→ /opt/ai-inference/raw-data]
    end

    AI -->|GET /sse| MCP
    SC -->|POST /mcp| MCP
    MCP -->|file:// I/O| Docs
    MCP -->|file:// I/O| Raw
```

The upstream image exposes **both** transports on port 3102: `POST /mcp/` (StreamableHTTP, used by `sentinel-collector`) and `GET /sse` (SSE, used by Claude Desktop via `mcp-proxy`). Despite the `--sse` flag in `compose.yaml`, the running image starts a `StreamableHTTP session manager` and serves both endpoints. Document inputs must resolve under one of the two read-only mounts; cross-container `file://` URIs work because consumers (e.g. `sentinel-collector`) and `markitdown-mcp` share the same host paths.

## Features

- **Office Documents**: PDF, Word (.docx/.doc), Excel (.xlsx/.xls), PowerPoint (.pptx/.ppt)
- **Web Content**: HTML pages via `file://` or `http(s)://` URIs
- **OCR / Images**: image-to-text extraction
- **Audio**: speech-to-text transcription
- **Plain Text**: CSV, TXT, JSON

Feature surface is upstream's; see [microsoft/markitdown](https://github.com/microsoft/markitdown) for the authoritative format list.

## Configuration

This service is configured entirely via the `command` array and `volumes` in `compose.yaml` -- there are no environment variables read by the container itself. The flags below are passed by ATLAS deployment:

| Flag | Value | Purpose |
|------|-------|---------|
| `--sse` | (set) | Selects the SSE transport (image also serves StreamableHTTP at `/mcp/`) |
| `--host` | `0.0.0.0` | Bind address |
| `--port` | `3102` | Listen port |

Consumer-side configuration (set on `sentinel-collector`, not on this container):

| Variable | Description | Default |
|----------|-------------|---------|
| `Markitdown__Endpoint` | Base URL of this service | `http://markitdown-mcp:3102` |
| `Markitdown__TimeoutSeconds` | HTTP timeout for `tools/call` | `30` |
| `Markitdown__Enabled` | Master enable for normalization | `true` |
| `Markitdown__NormalizeContentTypes` | Content types routed to markitdown | `text/html`, `application/pdf` |

### Volume Mounts

| Container Path | Host Path | Mode | Purpose |
|----------------|-----------|------|---------|
| `/workdir` | `/opt/ai-inference/documents` | ro | Ad-hoc document drop zone |
| `/opt/ai-inference/raw-data` | `/opt/ai-inference/raw-data` | ro | Sentinel raw HTML / PDF staging |

Consumers wiring up `file://` URIs against this service must write inputs under one of these host paths. `sentinel-collector` stages cross-container payloads at `/opt/ai-inference/raw-data/sentinel/tmp/` (configured via `Markitdown__SharedTempDirectory`).

### Resource Limits

| Resource | Limit |
|----------|-------|
| Memory | 2G |
| CPUs | 2.0 |

## API (MCP Tools)

| Tool | Description | Parameters |
|------|-------------|------------|
| `convert_to_markdown` | Convert a document to markdown | `uri` (string): `file:`, `http:`, `https:`, or `data:` URI |

### Usage (StreamableHTTP, container-to-container)

`sentinel-collector` issues:

```http
POST /mcp HTTP/1.1
Host: markitdown-mcp:3102
Accept: application/json, text/event-stream
Content-Type: application/json

{"jsonrpc":"2.0","id":"<guid>","method":"tools/call",
 "params":{"name":"convert_to_markdown",
           "arguments":{"uri":"file:///opt/ai-inference/raw-data/.../page.html"}}}
```

The server 307-redirects `/mcp` → `/mcp/` and returns a JSON-RPC response whose `result.content[0].text` is the markdown.

### Usage (CLI, ad-hoc)

```
# Local PDF under /workdir
convert_to_markdown(uri="file:///workdir/statements/2024-Q4.pdf")

# Web page
convert_to_markdown(uri="https://federalreserve.gov/monetarypolicy/fomcminutes20241218.htm")

# Sentinel raw-data path
convert_to_markdown(uri="file:///opt/ai-inference/raw-data/filings/10-K.pdf")
```

**Typical use cases**: brokerage PDFs, FOMC minutes, research reports, 10-K/10-Q filings, SentinelCollector HTML fallback when trafilatura yields too little body text.

## Project Structure

```
markitdownMCP/
└── README.md    # this file -- no source, no Containerfile, no build
```

The service is operated entirely from `/opt/ai-inference/compose.yaml` (rendered from `deployment/artifacts/compose.yaml.j2`). There is no per-service `.devcontainer/`, `src/`, or `tests/` tree.

## Development

There is no local development loop for this service -- it is the upstream image, unmodified. Useful one-offs:

```bash
# Convert a file with the standalone CLI (no server)
uvx markitdown /path/to/document.pdf

# Run the MCP server locally (matches production command)
uvx markitdown-mcp --sse --host 0.0.0.0 --port 3102
```

To change behavior, either pin a different `mcp/markitdown` image tag in `deployment/artifacts/compose.yaml.j2` or replace the upstream image with a local fork built outside this directory.

## Deployment

No build step. The image is pulled by `nerdctl` on container start.

```bash
# Full stack (includes markitdown-mcp)
ansible-playbook playbooks/deploy.yml
```

This directory has no individual deploy tag because there is nothing to build; restarting `markitdown-mcp` is handled by the broader deploy.

### Claude Desktop / Code Integration

Add to `~/.config/Claude/claude_desktop_config.json`:

```json
{
  "mcpServers": {
    "markitdown": {
      "command": "uvx",
      "args": ["mcp-proxy", "http://mercury:3102/sse"]
    }
  }
}
```

Claude Desktop speaks stdio; `mcp-proxy` bridges stdio to the SSE endpoint.

## Ports

| Port | Protocol | Description |
|------|----------|-------------|
| 3102 | HTTP | StreamableHTTP MCP at `/mcp/` (consumed by `sentinel-collector`) and SSE at `/sse` (consumed by Claude Desktop via `mcp-proxy`). Same port, both transports. |

Host-mapped 1:1 (`3102:3102`); accessible inside the compose network as `http://markitdown-mcp:3102` and from the host as `http://mercury:3102`.

## See Also

- [markitdown GitHub](https://github.com/microsoft/markitdown) -- upstream Python library and CLI
- [MCP Protocol](https://modelcontextprotocol.io/) -- Model Context Protocol specification
- [SentinelCollector](../SentinelCollector/README.md) -- primary consumer; see `Services/MarkitdownClient.cs`
- [Trafilatura](https://github.com/adbar/trafilatura) -- sibling extraction service used alongside markitdown for HTML normalization
