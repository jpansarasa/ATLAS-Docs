# Task: Implement Tool-Calling Agent for Qwen 3.5 on vLLM

## Overview

Build a CLI agent loop that connects to a local vLLM instance serving Qwen 3.5 (with LoRA) and gives the model four tools: Python execution, web search, file read, and file write. The agent sends tool schemas via the OpenAI-compatible API, executes tool calls locally, feeds results back, and loops until the model produces a final text response.

## Architecture

```
User CLI input
  → OpenAI client (pointed at vLLM)
    → Model returns tool_calls or text
      → If tool_calls: execute locally, append results, loop
      → If text: print and wait for next input
```

Single-file Python script. No framework dependencies beyond `openai` and `httpx`.

## Environment

- **Host**: Mercury — AMD Threadripper 9960X, 256GB RAM, RTX 5090
- **vLLM**: Running locally, OpenAI-compatible API at `http://localhost:8000/v1`
- **SearXNG**: Running on the local network (URL configurable)
- **Python packages on host**: torch, numpy, pandas, and others available to subprocess

## Configuration

All config via environment variables with sensible defaults:

| Variable | Default | Description |
|---|---|---|
| `VLLM_BASE_URL` | `http://localhost:8000/v1` | vLLM OpenAI-compatible endpoint |
| `VLLM_MODEL` | `qwen3.5` | Model name as registered in vLLM |
| `SEARXNG_URL` | `http://localhost:8888` | SearXNG instance base URL |
| `PYTHON_BIN` | `sys.executable` | Python binary for subprocess execution |
| `PYTHON_TIMEOUT` | `120` | Default subprocess timeout in seconds |
| `MAX_TOOL_ROUNDS` | `10` | Max tool-call round-trips per user message |
| `WORKSPACE_DIR` | `~/agent_workspace` | Shared workspace for file I/O and python_exec cwd |

## Tool Definitions

Define all four tools in OpenAI function-calling format and pass them in the `tools` parameter of each chat completion request. Use `tool_choice="auto"`.

### 1. `python_exec`

**Purpose**: Execute arbitrary Python code on the workstation.

**Parameters**:
- `code` (string, required): Python source code to execute.
- `timeout` (integer, optional): Timeout in seconds, defaults to `PYTHON_TIMEOUT`.

**Implementation**:
- Write `code` to a temp file in `/tmp` (suffix `.py`).
- Run via `subprocess.run([PYTHON_BIN, "-u", tempfile], capture_output=True, text=True, timeout=timeout, cwd=WORKSPACE_DIR)`.
- Return stdout. If stderr is non-empty, append it under a `[stderr]` header. If exit code is non-zero, append `[exit code N]`.
- On timeout, return `[timed out after Ns]`.
- Always delete the temp file in a `finally` block.

### 2. `web_search`

**Purpose**: Search the web via SearXNG JSON API.

**Parameters**:
- `query` (string, required): Search query.
- `num_results` (integer, optional): Max results, default 5.

**Implementation**:
- `GET {SEARXNG_URL}/search` with params `q`, `format=json`, `number_of_results`.
- 15 second timeout via httpx.
- Return numbered list: `{i}. {title}\n   {url}\n   {snippet[:300]}` for each result.
- On any exception, return `[search error: {e}]`.

### 3. `file_read`

**Purpose**: Read files from the workspace. Also handles directory listings.

**Parameters**:
- `path` (string, required): File path, resolved relative to `WORKSPACE_DIR` if not absolute.
- `offset` (integer, optional): Byte offset to start reading, default 0.
- `limit` (integer, optional): Max bytes to read, default 65536.

**Implementation**:
- Resolve relative paths against `WORKSPACE_DIR`.
- If path is a directory, return sorted listing (first 200 entries, prefixed `d`/`f`).
- If file not found, return `[file not found: {path}]`.
- Open in text mode with `errors="replace"`. Seek to `offset`, read `limit` bytes.
- Prepend a header line: `[{path} | {size} bytes | offset {N} | showing {limit}/{remaining} remaining]` (omit offset/remaining parts when not applicable).

### 4. `file_write`

**Purpose**: Write or append content to files in the workspace.

**Parameters**:
- `path` (string, required): File path, resolved relative to `WORKSPACE_DIR` if not absolute.
- `content` (string, required): Content to write.
- `append` (boolean, optional): If true, append instead of overwrite. Default false.

**Implementation**:
- Resolve relative paths against `WORKSPACE_DIR`.
- Create parent directories with `mkdir(parents=True, exist_ok=True)`.
- Open in `"a"` or `"w"` mode based on `append`.
- Return `[wrote {path} ({N} bytes)]` or `[appended to {path} ({N} bytes)]`.

## Agent Loop

```
function run_agent(client, messages) -> str:
    for _ in range(MAX_TOOL_ROUNDS):
        response = client.chat.completions.create(
            model, messages, tools, tool_choice="auto", temperature=0.6
        )
        message = response.choices[0].message
        append message to messages

        if no tool_calls on message:
            return message.content

        for each tool_call:
            parse function name and JSON arguments
            print a dim summary line to terminal: ⚡ tool_name(summary)
            dispatch to handler, get result string
            if result > 8000 chars, truncate with "...[truncated]"
            append tool result message with matching tool_call_id

    return "(max tool rounds reached)"
```

### Summary Line Format

When printing tool calls to the terminal (for the human operator), show a concise summary:
- `python_exec`: first line of code (60 chars) + line count
- `web_search`: the query (80 chars)
- `file_read` / `file_write`: the path (80 chars)

Use dim ANSI coloring (`\033[90m`).

## System Prompt

```
You are a capable assistant running on a high-end workstation
(AMD Threadripper 9960X, 256GB RAM, RTX 5090 24GB).
You have four tools:
- python_exec: run Python code on this machine (torch, numpy, pandas, etc. available)
- web_search: search the web via SearXNG
- file_read: read files from the workspace (also lists directories)
- file_write: write/append content to files in the workspace
The workspace directory is: {WORKSPACE_DIR}
For large outputs from Python, write results to a file with file_write or
print sparingly, then use file_read to inspect specific portions.
Use tools when they'd help. Be direct and concise.
```

## CLI

- Initialize OpenAI client pointed at `VLLM_BASE_URL` with `api_key="not-needed"`.
- Print a startup banner showing the vLLM URL, model name, SearXNG URL, Python binary, and workspace path.
- Read input in a loop. Handle `quit`, `reset` (clears message history), `Ctrl+C`/`EOF`.
- On each user message, call `run_agent()` and print the result.

## Dependencies

```
openai>=1.0
httpx>=0.27
```

No other external dependencies. Standard library only for everything else.

## File Structure

```
agent/
├── agent.py          # Single-file implementation
└── requirements.txt  # openai, httpx
```

## Reference Implementation

A complete reference implementation is provided in `agent.py` alongside this document. Use it as the authoritative source for exact behavior, edge cases, and formatting.

## Testing Checklist

- [ ] Agent starts and connects to vLLM
- [ ] Simple text prompt (no tools) returns a response
- [ ] `python_exec` runs code, returns stdout/stderr correctly
- [ ] `python_exec` respects timeout
- [ ] `web_search` returns formatted results from SearXNG
- [ ] `web_search` handles connection errors gracefully
- [ ] `file_write` creates files with parent directories
- [ ] `file_read` reads back written files
- [ ] `file_read` handles directory listings
- [ ] `file_read` with offset/limit pages through large files
- [ ] Multi-tool-call round: model calls python_exec to generate data, file_write to save it, file_read to verify
- [ ] Truncation triggers at 8000 chars
- [ ] `reset` clears history
- [ ] `MAX_TOOL_ROUNDS` cap is respected
