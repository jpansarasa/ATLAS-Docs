# Tool-calling agent

Single-file Python CLI that connects to the local vLLM instance and gives the model four tools: `python_exec`, `web_search`, `file_read`, `file_write`.

## Setup

```bash
python -m venv agent/.venv
agent/.venv/bin/pip install -r agent/requirements.txt
```

## Run

```bash
agent/.venv/bin/python agent/agent.py
```

Commands in the REPL:
- `quit` — exit
- `reset` — clear conversation history
- `/model` — show current model
- `/model <name>` — switch model (validates against `/v1/models`)

## Environment variables

| Variable | Default | Purpose |
|---|---|---|
| `VLLM_BASE_URL` | `http://localhost:8000/v1` | vLLM OpenAI-compatible endpoint |
| `VLLM_MODEL` | `sentinel-cove` | Default model (the CoD/CoVe LoRA on Qwen2.5-32B-Instruct-AWQ). Use `sentinel-cove-v6.2` for the plain base. |
| `SEARXNG_URL` | `https://searxng.elasticdevelopment.com` | SearXNG instance base URL |
| `SANDBOX_IMAGE` | `python-sandbox:latest` | Container image for sandboxed `python_exec`. Set to empty string to fall back to local subprocess. |
| `NERDCTL_BIN` | `sudo nerdctl` | Container runtime command. |
| `PIP_CACHE_VOLUME` | `python-sandbox-pip-cache` | Named volume for pip download cache across ephemeral containers. |
| `PYTHON_BIN` | `SentinelCollector/scripts/.venv/bin/python` if present, else current interpreter | Python binary for local fallback (when `SANDBOX_IMAGE` is empty). |
| `PYTHON_TIMEOUT` | `120` | Default subprocess timeout in seconds |
| `MAX_TOOL_ROUNDS` | `10` | Max tool-call round-trips per user message |
| `WORKSPACE_DIR` | `/opt/ai-inference/agent-workspace` | Shared workspace for file I/O and `python_exec` cwd |

## Sandboxing

`python_exec` runs code in **ephemeral containers** via `nerdctl run --rm`. Each call gets a fresh container — no state leaks between calls. The base image (`deployment/artifacts/python-sandbox/`) has numpy, pandas, and httpx pre-installed. Additional packages can be requested per-call via the `dependencies` parameter; a named pip cache volume avoids repeated downloads.

The workspace ZFS dataset (`sata-bulk/agent-workspace`) is mounted at `/workspace/` inside the container. The host filesystem is not accessible — only the workspace mount.

Set `SANDBOX_IMAGE=""` to fall back to local subprocess execution (for development/testing).

`file_read`/`file_write` still run on the host but resolve relative paths to `WORKSPACE_DIR`.

## vLLM prerequisites

vLLM must be launched with tool-calling support:

```
--enable-auto-tool-choice
--tool-call-parser hermes
```

These are already set in `deployment/ansible/playbooks/deploy.yml` for the vllm-server task. `hermes` is the correct parser for Qwen2.5-Instruct's `<tool_call>...</tool_call>` format.

## Reference implementation

`tasks/agent.py` holds the original reference implementation from the design doc. The production version in this directory differs in: config defaults, exception handling in the agent loop, Ctrl+C handling in the REPL, binary-file detection in `file_read`, correct byte counts in `file_write`, directory-listing truncation indicator, message history cap, and the `/model` command.
