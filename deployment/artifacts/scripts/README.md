# deployment/artifacts/scripts

Host-side scripts that run **on Mercury directly** (not inside containers). They power AutoFix orchestration, container target generation for Prometheus, and one-shot seeding utilities. Most are wired into `systemd` timers.

## Files

| Script | Triggered by | Purpose |
|---|---|---|
| `autofix.sh` | `autofix-runner.sh` (per alert) | AutoFix orchestrator. Reads alert JSON from stdin, invokes Claude Code to diagnose, opens a PR, posts ntfy notification. Expects `AUTOFIX_SESSION_ID` + `AUTOFIX_LOG_FILE` env from the runner. |
| `autofix-runner.sh` | systemd timer + queue poll | Polls the AutoFix queue directory for alert JSON files written by `alert-service`, calls `autofix.sh` for each. Needs Claude CLI installed on the host. |
| `autofix-watcher.sh` | systemd timer (every 5 min) | Polls GitHub for merged AutoFix PRs and triggers deployment. Original single-purpose watcher; `merged-pr-watcher.sh` generalises this. |
| `merged-pr-watcher.sh` | systemd timer | Generalised watcher: polls GitHub for any merged PR carrying the `auto-deploy` label, rebuilds affected images, and deploys via Ansible. Supersedes `autofix-watcher.sh` for non-AutoFix labels. |
| `generate-container-targets.sh` | systemd timer / on demand | Emits two files for Prometheus enrichment: a JSON file_sd target list, and a `.prom` textfile for `node_exporter` carrying `container_info` metrics (image, version, build_date). |
| `seed_secmaster.py` | one-shot (operator) | Seeds SecMaster with all currently-known series by querying each collector's HTTP surface and POSTing to `/api/register`. Uses `curl` for HTTP/2 (h2c) since the Python ecosystem's plaintext HTTP/2 support is patchy. Re-runnable — registration is idempotent. |

## When to run these directly

- **AutoFix scripts** — driven by systemd timers + queue-polling. Manual invocation is fine for debugging an individual alert; tail `AUTOFIX_LOG_FILE` to follow.
- **`generate-container-targets.sh`** — invoke after a deploy if Prometheus container metadata looks stale (rare; the timer normally handles this).
- **`seed_secmaster.py`** — initial bring-up of a fresh SecMaster instance, or after a deliberate truncation of the catalog. Not part of normal ops.

## See Also

- [deployment/ansible/scripts](../../ansible/scripts/README.md) — playbook-invoked helpers
- [scripts/](../../../scripts/) — repo-root operator scripts (Claude CLI helpers)
