# scripts

Repo-root operator scripts. Mix of Claude Code helpers, ad-hoc auditing harnesses, and per-feature subdirectories with their own READMEs. Most of these are invoked directly by the operator (you); a few are referenced by `.claude/hooks/` or systemd timers.

## Files

| File | Purpose |
|---|---|
| `claude-mark-verified` | Manual "tests passed" marker for the `git-push-guard.sh` hook when changes are not amenable to a `compile.sh` validator (YAML, Markdown, dashboard JSON, shell). Writes a `v2-manual` marker that unblocks the push and logs an audit line. See `CLAUDE_MARK_VERIFIED.md` for full usage. |
| `CLAUDE_MARK_VERIFIED.md` | Companion doc for `claude-mark-verified` — when to use, when **not** to use, audit-log format, hook behaviour. |
| `agent-stall-watchdog.sh` | Per-agent stall detector for supervisor sessions. Given a tasks dir (the `.output`-symlink directory under `/tmp/claude-<uid>/…/<uuid>/tasks`), reports each subagent transcript whose mtime is older than N minutes. Annotates `PROMPT?` when the tail contains `"stop_reason":"tool_use"` (agent proposed a tool, harness is paused waiting for approval). Suppresses false positives with `BUSY-COMPILE` when a compile/build process is found for the agent's worktree. Designed to run every supervisor poll cycle; output fits in <10 lines for a healthy session. |

## Subdirectories (each with its own README)

| Directory | Purpose |
|---|---|
| `claude-watchdog/` | Background watchdog (`scan.py` + `notify.sh`) that flags long-running Claude Code sessions sitting idle on user input. Publishes to NTFY `atlas-claude-ask`. |
| `sentinel-quality-check/` | Production weekly Sentinel qualitative-extraction quality-check harness (runs via `atlas-sentinel-quality-check.timer`). Also serves as the on-demand A/B audit harness for the F4.6.4 entity-resolution prompt-grounding feature. Renders a Markdown scorecard from a 50-row stratified sample. |

## When to use which

- **Need to push a docs/YAML/shell-only PR?** Run `scripts/claude-mark-verified "<reason>"` first; the hook will accept the manual marker.
- **Dispatching a subagent for a Matrix epic story?** Templates live in `.claude/skills/supervisor-mode/templates/` (start from `story-implementation.md`).
- **Long-running Claude session stuck on permission prompts?** `claude-watchdog/scan.py` is what fires the NTFY ping; tail its log to debug false positives.
- **Supervisor subagent stalled overnight?** Run `scripts/agent-stall-watchdog.sh <tasks-dir>` to find which agent(s) have stopped making progress; `PROMPT?` flag identifies permission-prompt stalls specifically.
- **Investigating an F4.6.4 prompt-grounding regression?** Re-run the `sentinel-quality-check/` harness against the live extraction stack.

## See Also

- [.claude/hooks/](../.claude/hooks/) — pre-push hook that reads the `mark-verified` marker file
- [deployment/artifacts/scripts](../deployment/artifacts/scripts/README.md) — host-side scripts driven by systemd timers
