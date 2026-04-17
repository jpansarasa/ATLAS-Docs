# Tool-calling agent

Single-file Python CLI that connects to the local vLLM instance and gives the model three tools: `python_exec` (persistent IPython kernel), `web_search`, `read_file`.

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
- `quit` — exit (tears down the sandbox container)
- `reset` — clear conversation history AND tear down the sandbox. Next `python_exec` starts a fresh kernel in a new task dir.
- `/model` — show current model
- `/model <name>` — switch model (validates against `/v1/models`)

## Environment variables

| Variable | Default | Purpose |
|---|---|---|
| `VLLM_BASE_URL` | `http://localhost:8000/v1` | vLLM OpenAI-compatible endpoint |
| `VLLM_MODEL` | `sentinel-cove` | Default model (CoD/CoVe LoRA on Qwen2.5-32B-Instruct-AWQ) |
| `SEARXNG_URL` | `https://searxng.elasticdevelopment.com` | SearXNG base URL |
| `SANDBOX_IMAGE` | `sandbox-kernel:latest` | Container image for the persistent IPython kernel |
| `SANDBOX_NETWORK` | `ai-inference` | nerdctl bridge network the sandbox joins |
| `NERDCTL_BIN` | `sudo nerdctl` | Container runtime command |
| `AGENT_USE_LOCAL_PYTHON` | unset | Set to `1` to bypass the sandbox and run each `python_exec` as a local subprocess (no kernel state, dev fallback only) |
| `PYTHON_BIN` | Sentinel venv if present, else current interpreter | Python used by the local fallback |
| `PYTHON_TIMEOUT` | `120` | Default `python_exec` timeout in seconds |
| `MAX_TOOL_ROUNDS` | `12` | Max tool-call round-trips per user message |
| `WORKSPACE_ROOT` | `/opt/ai-inference/agent-workspace` | Parent of per-task dirs |

## How the sandbox works

One **long-lived container** per REPL session, running a FastAPI server around an IPython kernel (`deployment/artifacts/sandbox-kernel/`). The container is lazy-started on the first `python_exec` call.

Per-task directories under `WORKSPACE_ROOT/tasks/{task_id}/`:

| Host path | Container mount | Mode | Purpose |
|---|---|---|---|
| `…/input/`     | `/input`     | read-only | Inputs staged by the caller before starting the task |
| `…/output/`    | `/output`    | read-write | Model writes final results here (convention: `/output/result.json`) |
| `…/scratch/`   | `/workspace` | read-write | Kernel cwd, scratch files, intermediate artefacts |

The model is told about these paths in the system prompt. Because the kernel is persistent, variables/imports/open DataFrames survive across `python_exec` calls — the model can chain work naturally (load → clean → extract → validate → write) without re-parsing each turn.

Packages beyond the pre-installed set (`numpy`, `pandas`, `httpx`, `jsonschema`) can be installed with `!pip install <pkg>` from inside `python_exec`.

Hard caps: `--memory 2g --cpus 2 --pids-limit 512`. The server caps `/exec` output at 64 KB. Idle kernels exit after `KERNEL_IDLE_SECONDS` (default 900).

## Output handoff

The convention is: the model writes its final result to `/output/result.json` and replies with the single word `DONE`. For interactive REPL use this convention is loose — agents can also just answer in text. For Sentinel extraction, `/output/result.json` is the authoritative handoff (read from the C# side via the same host path).

## vLLM prerequisites

vLLM must be launched with:

```
--enable-auto-tool-choice
--tool-call-parser hermes
```

Already set in `deployment/ansible/playbooks/deploy.yml`. `hermes` is the correct parser for Qwen2.5-Instruct's `<tool_call>...</tool_call>` format.
