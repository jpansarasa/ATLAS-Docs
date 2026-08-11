# scripts

Repo-root operator scripts. Mix of Claude Code helpers, ad-hoc auditing harnesses, and per-feature subdirectories with their own READMEs. Most of these are invoked directly by the operator (you); a few are referenced by `.claude/hooks/` or systemd timers.

## Files

| File | Purpose |
|---|---|
| `claude-mark-verified` | Manual "tests passed" marker for the `git-push-guard.sh` hook when changes are not amenable to a `compile.sh` validator (YAML, Markdown, dashboard JSON, shell). Writes a `v2-manual` marker that unblocks the push and logs an audit line. See `CLAUDE_MARK_VERIFIED.md` for full usage. |
| `CLAUDE_MARK_VERIFIED.md` | Companion doc for `claude-mark-verified` — when to use, when **not** to use, audit-log format, hook behaviour. |
| `devcontainer-owner.sh` | Per-agent ownership of devcontainer verification runs, sourced by every `.devcontainer/compile.sh` plus sentinel-edge's `typecheck.sh`/`dev.sh`. Each run gets its own compose project `atlas-<sha1(worktree)[0:12]>-<slug>` — the same key `mark-tests-passed.sh` uses — so containers, networks and the nuget volume are all per-owner and N agents verify at once. Previously container names were identical across worktrees (no `container_name:`, relative `/workspace` mounts), so a run could silently test another worktree's source and still write a push marker; this **supersedes** the host-wide flock that fixed that by serializing everything. Also provides `devcontainer_verify_workspace` (inode proof that `/workspace` is the caller's own tree, gating the marker) and a reaper that removes `atlas-*` state whose worktree no longer exists — which is what covers SIGKILLed runs, since no trap can. |
| `test-devcontainer-owner.sh` | Guard test for the above (~3s, no containers): the project key matches the marker key, a foreign tree at `/workspace` is refused, `mark-tests-passed.sh` refuses a missing/stale/foreign attestation, **the reaper acts only on positively-identified absence and declines when it cannot tell** (including when run from a foreign repo), teardown fires exactly once, every compose-driving script owns *before* touching containers and verifies *before* building, no verification-path compose file pins a host port, and every base compose file sets a project name. No assertion count is quoted here on purpose — several sections scale with the file set, so a fixed number is drift waiting to happen; the suite prints its own totals. Discovery is depth-agnostic (`git ls-files`) and globs `compose*.ya?ml`, backed by a count floor and a declared-exception list, so neither a too-shallow glob nor a filter that matches nothing can pass silently. `--with-containers` chains the simultaneity proof. |
| `test-devcontainer-simultaneity.sh` | The real-container proof (~2 min): two different services from two worktrees concurrently, then the *same* service from two worktrees concurrently, asserting each run execs into its own `/workspace` and gets its own marker — plus a mutation that points one run at another's container and requires the refusal to fire. Section E then proves the suite's own `ATLAS_MARKER_DIR` redirect cannot widen the real push gate, end-to-end through the hook in both directions: a marker written under the override is honoured by a guard reading that directory and refused by one running with the default environment. It commits a nonce first so the branch's tree cannot be covered by any other marker on the host — otherwise a legitimate marker for this very branch would decide the assertion. Documents in-file what it cannot cover: passing runs sample interleavings, they do not prove the race is gone; the mutation and the disjoint-names argument carry that. |
| `agent-stall-watchdog.sh` | Per-agent stall detector for supervisor sessions. Given a tasks dir (the `.output`-symlink directory under `/tmp/claude-<uid>/…/<uuid>/tasks`), reports each subagent transcript whose mtime is older than N minutes. Annotates `PROMPT?` when the tail contains `"stop_reason":"tool_use"` (agent proposed a tool, harness is paused waiting for approval). Suppresses false positives with `BUSY-COMPILE` when a compile/build process is found for the agent's worktree. Designed to run every supervisor poll cycle; output fits in <10 lines for a healthy session. |

## Subdirectories (each with its own README)

| Directory | Purpose |
|---|---|
| `claude-watchdog/` | Background watchdog (`scan.py` + `notify.sh`) that flags long-running Claude Code sessions sitting idle on user input. Publishes to NTFY `atlas-claude-ask`. |
| `sentinel-quality-check/` | Production weekly Sentinel qualitative-extraction quality-check harness (runs via `atlas-sentinel-quality-check.timer`). Also serves as the on-demand A/B audit harness for the F4.6.4 entity-resolution prompt-grounding feature. Renders a Markdown scorecard from a 50-row stratified sample. |
| `gemini-spend-calibration/` | Offline calibration harness for the surface gate in front of the paid Gemini resolver — captures a window of what reached the boundary, replays it through SearXNG and scores the issuer-probe signals. Sets no thresholds. The 11-surface reference draw is committed (`testdata/reference-cache/`), so `--offline` re-checks it byte for byte instead of re-drawing a different population. Stdlib only; `unittest discover` runs the suite (**no pytest on this host**) and `mutation-check.py` says what the suite is worth. |

## When to use which

- **Need to push a docs/YAML/shell-only PR?** Run `scripts/claude-mark-verified "<reason>"` first; the hook will accept the manual marker.
- **Dispatching a subagent for a Matrix epic story?** Templates live in `.claude/skills/supervisor-mode/templates/` (start from `story-implementation.md`).
- **Long-running Claude session stuck on permission prompts?** `claude-watchdog/scan.py` is what fires the NTFY ping; tail its log to debug false positives.
- **Supervisor subagent stalled overnight?** Run `scripts/agent-stall-watchdog.sh <tasks-dir>` to find which agent(s) have stopped making progress; `PROMPT?` flag identifies permission-prompt stalls specifically.
- **Investigating an F4.6.4 prompt-grounding regression?** Re-run the `sentinel-quality-check/` harness against the live extraction stack.

## See Also

- [.claude/hooks/](../.claude/hooks/) — pre-push hook that reads the `mark-verified` marker file
- [deployment/artifacts/scripts](../deployment/artifacts/scripts/README.md) — host-side scripts driven by systemd timers
