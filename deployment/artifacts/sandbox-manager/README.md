# sandbox-manager

A small FastAPI service whose only job is to spawn and tear down `sandbox-kernel` containers on behalf of services that shouldn't have container-runtime access themselves.

Runs as a **systemd service on the host** (not a compose container). Containerized nerdctl can't do containerd's local snapshot-mount prep from inside a container even with the socket mounted — so the manager lives on the host, where nerdctl "just works." Privilege isolation is the same: only this one service has nerdctl access; everyone else goes through its HTTP API.

See unit file at `deployment/artifacts/sandbox-manager.service`.

## Threat model

- **Trusted caller** (sentinel-collector in a container, which has no nerdctl/socket access). Compromise of the caller must NOT let the attacker run arbitrary images or mount arbitrary host paths.
- **Hostile content**. Sentinel routinely ingests attacker-influenced web content. The manager's validation keeps that content from flowing into nerdctl arguments.
- **Not in scope**: protecting against a compromise of the manager itself. If the manager is compromised, you have host container-runtime control — that's why its code is small and its API is narrow.

## What's pinned (not accepted from callers)

| Setting | Env var | Default |
|---|---|---|
| Image | `SANDBOX_IMAGE` | `sandbox-kernel:latest` |
| Network | `SANDBOX_NETWORK` | `ai-inference` |
| Memory cap | `SANDBOX_MEMORY` | `2g` |
| CPU cap | `SANDBOX_CPUS` | `2.0` |
| Pids limit | `SANDBOX_PIDS_LIMIT` | `512` |
| Kernel idle timeout | `SANDBOX_IDLE_SECONDS` | `900` |
| Task path allowlist | `TASKS_ROOT` | `/opt/ai-inference/agent-workspace/tasks` |

Callers cannot override any of these. Callers cannot pass extra `nerdctl` flags, additional volumes, `--privileged`, etc.

## What's validated per request

- `task_id` matches `^[a-zA-Z0-9][a-zA-Z0-9._-]{0,63}$`.
- `input_dir`, `output_dir`, `scratch_dir` resolve (via `realpath`) under `TASKS_ROOT` — rejects `..` traversal and symlinks pointing outside.
- Each directory must already exist on the host.
- Container names must start with `sandbox-kernel-` (prevents `DELETE /sessions/<anything>` becoming a general "remove any container" gadget).

## HTTP API

- `GET /health` → `{ok, image, network}`.
- `POST /sessions {task_id, input_dir, output_dir, scratch_dir}` → `{container_name, image, network, health_url, exec_url}`. Starts one kernel and returns its in-network URL. Kernel is reachable at `http://{container_name}:9100/`.
- `DELETE /sessions/{container_name}` → idempotent teardown.
- `GET /sessions` → list of live sandbox containers (ops introspection).

## Deployment

```sh
ansible-playbook playbooks/deploy.yml --tags sandbox-manager,build
```

Installs:

- `/opt/ai-inference/sandbox-manager/{manager_server.py, requirements.txt, .venv/}`
- `/etc/systemd/system/sandbox-manager.service`

The unit binds to `0.0.0.0:9200`. Inside the `ai-inference` bridge, containers reach the manager via `sandbox-manager:9200` by adding `extra_hosts: ["sandbox-manager:host-gateway"]` to their compose entry (`host-gateway` resolves to the bridge gateway, which IS the host from the bridge's perspective). The sentinel-collector compose entry has this set.

## Operations

```sh
# logs
journalctl -u sandbox-manager -f

# restart
sudo systemctl restart sandbox-manager

# list live kernels
curl -s http://127.0.0.1:9200/sessions | jq .
```
