# deployment/artifacts/scripts

Host-side scripts that run **on Mercury directly** (not inside containers). They power AutoFix orchestration, container target generation for Prometheus, and one-shot seeding utilities. Most are wired into `systemd` timers.

## Files

| Script | Triggered by | Purpose |
|---|---|---|
| `autofix.sh` | `autofix-runner.sh` (per alert) | AutoFix orchestrator. Reads alert JSON from stdin, invokes Claude Code to diagnose, opens a PR, posts ntfy notification. Expects `AUTOFIX_SESSION_ID` + `AUTOFIX_LOG_FILE` env from the runner. **The prompt was passed after the variadic `--allowedTools`, so the CLI ate it as tool-rule values — fixed 2026-08-07.** Measured on 2.1.224 that exits before a session starts, so it would have made every *future* run inert; it is not what broke the past. The pipeline has not invoked the CLI since 2026-06-11 and all three binaries on disk postdate that, so **no run ever executed under 2.1.224** — the symptom appears in 2 of 584 archived logs (2026-06-10/11, as `Ignoring --allowedTools rule "**Alert**:"`, the prompt's own markdown parsed as rules), and even there two of the three invocations ran full sessions, one opening PR #689. So the operator-visible sign is the **absence of autofix PRs since June**, not a failure ntfy: the 233 logs carrying "AutoFix: Failed to Create Fix" are sessions that ran and opened no PR, which is a different fault. The spawned session is **tool-scoped** by `build_tool_scopes` — deploy, host service control, container mutation, interpreters, a nested `claude` and `gh pr merge` are denied at the CLI, in every path spelling, not merely forbidden in the prompt. That scoping is a **speed bump against an accident, not a boundary**: `Write` is allowed, so a denied command can be written to a file and run as `bash <file>` (measured). The prompt's prose constraints stay load-bearing. See the comment block above `build_tool_scopes` for the matcher measurements and the full residual risk. |
| `autofix-runner.sh` | systemd timer, every 60s — **ENABLED** | Polls the AutoFix queue directory for alert JSON files written by `alert-service`, calls `autofix.sh` for each. Needs Claude CLI installed on the host. **Never invokes Ansible** and never touches a running service — the pipeline ends at the PR. |
| `autofix-watcher.sh` | systemd timer, every 5 min — **DISABLED (2026-08-07)** | Polled GitHub for merged AutoFix PRs and auto-deployed them. Disarmed: ran `deploy.yml` with no `--skip-tags` and no `scoped_restart` against the shared working tree (full-stack restart incl a ~4min vLLM reload), retrying every 5 min on failure. Its smoke test + `:autofix-prev` rollback moved to the `deploy` skill. Still runnable as a deliberate one-shot. |
| `merged-pr-watcher.sh` | systemd timer — **DISABLED** | Generalised watcher: polls GitHub for any merged PR carrying the `auto-deploy` label, rebuilds affected images, and deploys via Ansible. Generalises `autofix-watcher.sh`; disabled for the same reason. |
| `generate-container-targets.sh` | systemd timer / on demand | Emits two files for Prometheus enrichment: a JSON file_sd target list, and a `.prom` textfile for `node_exporter` carrying `container_info` metrics (image, version, build_date). |
| `seed_secmaster.py` | one-shot (operator) | Seeds SecMaster with all currently-known series by querying each collector's HTTP surface and POSTing to `/api/register`. Uses `curl` for HTTP/2 (h2c) since the Python ecosystem's plaintext HTTP/2 support is patchy. Re-runnable — registration is idempotent. |

## When to run these directly

- **AutoFix scripts** — `autofix.sh` + `autofix-runner.sh` are driven by the runner timer + queue-polling. Manual invocation is fine for debugging an individual alert; tail `AUTOFIX_LOG_FILE` to follow. The two auto-deploy watchers are disabled — running either by hand triggers a real full-stack deploy, so treat them as operator-only. Before changing `autofix.sh`'s tool scoping, run `deployment/tests/autofix/run.sh` (add `--live` to prove refusal at the real boundary); over-narrowing the allow side fails **silently** — the session cannot diagnose, opens no PR, and it looks like "no alerts fired".
- **`generate-container-targets.sh`** — invoke after a deploy if Prometheus container metadata looks stale (rare; the timer normally handles this).
- **`seed_secmaster.py`** — initial bring-up of a fresh SecMaster instance, or after a deliberate truncation of the catalog. Not part of normal ops.

## See Also

- [deployment/ansible/scripts](../../ansible/scripts/README.md) — playbook-invoked helpers
- [scripts/](../../../scripts/) — repo-root operator scripts (Claude CLI helpers)
