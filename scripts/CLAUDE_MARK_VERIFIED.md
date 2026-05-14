# claude-mark-verified

Manual "tests passed" marker for cases the automated validators don't cover.

## What it does

The `.claude/hooks/git-push-guard.sh` hook blocks `git push` unless a marker
file says the current tree has been verified. `compile.sh` writes that marker
automatically after .NET tests pass — but only for .NET projects. Branches
that touch only YAML / Markdown / shell / dashboards / etc. can't earn a
marker that way and get stuck.

`scripts/claude-mark-verified` writes the same marker, but tagged as a
manual override (`v2-manual` instead of `v2`). The hook still unblocks the
push; it just logs a warning on every push and records the override in an
audit log.

## When to use it

- YAML / Markdown / dashboard JSON changes that have no compile step.
- Shell or Python scripts you've reviewed and smoke-tested by hand.
- Generated artifacts (e.g. Grafana dashboard exports) where the validator
  is your own eyes.

## When NOT to use it

- **.NET source changes.** Run `{Project}/.devcontainer/compile.sh` — that's
  what it's for. Manual marking bypasses tests and is how bugs ship.
- **A working tree with STAGED changes.** The tool refuses when staged
  changes are present (since those would change `HEAD^{tree}` on the next
  commit). Unstaged tracked edits are ignored — they don't affect the marker.

## Examples

```bash
# Dashboards-only PR
scripts/claude-mark-verified "Grafana dashboard JSON — diff reviewed, no query changes"

# Docs/Markdown
scripts/claude-mark-verified "README + plan docs only, no executable code"

# Shell script change with manual smoke test
scripts/claude-mark-verified "ran scripts/foo.sh against staging, output matches expected"
```

## Audit log

Every invocation appends to `.claude/hooks/mark-verified.log`:

```
<iso8601> <tree_hash> <git-user.email-or-$USER> <branch> <reason>
```

The log is intentionally NOT gitignored — committers should
`git add .claude/hooks/mark-verified.log` periodically when reviewing
audit entries, so the pattern of manual overrides becomes part of repo
history over time. Nothing auto-commits it; this is a manual workflow.

## Hook behavior

The push guard accepts both marker formats:

- `v2 tree <hash> <ts>` — written by `compile.sh`, no warning.
- `v2-manual tree <hash> <ts> <reason>` — written by this tool, warns on
  stderr (`tree <hash> verified manually via claude-mark-verified —
  reason: ...`) on every push.
