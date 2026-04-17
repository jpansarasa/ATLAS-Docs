# sandbox-kernel

Persistent IPython kernel in a container, addressed over HTTP via a thin FastAPI wrapper around `jupyter_client.BlockingKernelClient`.

Used by:

- `agent/agent.py` — interactive REPL agent. Spawns one container per REPL session via `sudo nerdctl` (agent runs on the host).
- `SentinelCollector` (flag-gated via `Extraction__UseToolAugmentedCove` / `Extraction__UseToolAugmentedEpistemicMarkers`) — one container per extraction task, provisioned through the `sandbox-manager` sidecar.

## Build

```sh
ansible-playbook playbooks/deploy.yml --tags sandbox-kernel,build
```

Or direct:

```sh
sudo nerdctl build -t sandbox-kernel:latest deployment/artifacts/sandbox-kernel
```

## HTTP API

- `GET /health` → `{alive: bool, idle_seconds: float}`
- `POST /exec {code, timeout_s}` → `{status: "ok"|"error"|"timeout", stdout, stderr, result, error, truncated}`
- `POST /interrupt` → sends SIGINT to the kernel (use while `/exec` is running).
- `POST /restart` → restart the kernel, clearing all state.

Output is capped at 64 KB per call; `truncated: true` indicates the cap fired. The idle reaper (default 15 min via `KERNEL_IDLE_SECONDS`) exits the process so orphan containers tear down on their own.

## Runtime expectations

- Volumes: `/input` (read-only inputs), `/output` (writable — the convention is the caller reads `/output/result.json` back), `/workspace` (scratch, kernel cwd).
- Network: joins the `ai-inference` nerdctl bridge. Outbound internet works (for `!pip install`, SearXNG, etc.).
- Resource caps in production: `--memory 2g --cpus 2 --pids-limit 512` (set by `sandbox-manager`, not by callers).

## Who spawns these?

Short answer: never the service that will talk to the kernel.

- **agent.py** runs on the host and already has `nerdctl` access, so it shells out directly.
- **Sentinel** runs in a container with no container-runtime access. It calls `sandbox-manager` to spawn/tear down kernels; the manager is the only service with `/run/containerd/containerd.sock` mounted. See `deployment/artifacts/sandbox-manager/README.md`.

This split keeps the "LLM-driven python, possibly influenced by scraped content" concern contained to the kernel container, and keeps the "spawn arbitrary containers on the host" privilege inside a small, audited sidecar with a narrow API.
