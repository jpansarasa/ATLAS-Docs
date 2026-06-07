# Claude Code Hooks

Context-aware hooks that inject patterns when working on specific file types.

## Active Hooks

| Hook | Trigger | Purpose |
|------|---------|---------|
| `testing-context.sh` | Edit/Write `*Tests.cs`, `*_test.*`, `test_*.*` | Inject AAA, naming, assertions patterns |
| `benchmark-context.sh` | Edit/Write `*Benchmark*.cs`, `*_bench.*` | Inject BenchmarkDotNet, Release mode |
| `git-push-guard.sh` | Bash `git push*` | **BLOCK** - requires tests pass first |
| `ef-migration-guard.sh` | Write `Migrations/*_*.cs` | **BLOCK** - prevents manual migration creation |
| `ansible-gate-guard.sh` | Edit/Write on deployment/CI gate files | **ASK** - confirm intent before editing gates |
| `deploy-smoke-reminder.sh` | Bash deploy/restart commands (PostToolUse) | **ADVISE** - inject smoke-test reminder |
| `memory-density-guard.sh` | Write/Edit `*/memory/*.md` (PostToolUse) | **ADVISE** - nudge when MEMORY.md hook line or memory-file description violates the density bar |

## How It Works

1. Claude calls a tool (Edit, Write, Bash, etc.)
2. `PreToolUse` hook runs before execution
3. Hook checks input and decides: allow/deny/modify
4. If denied: tool execution blocked with message
5. If allowed: tool executes normally

## Git Push Guard

**Purpose**: Prevent pushing code without running tests first.

**Behavior**: Blocks any `git push` command unless a tests-passed marker for
the current working tree exists. The marker is **tree-hash-keyed** (stores
the tree hash of the most recent commit — committed content only, uncommitted
edits do not change the tree hash), so it survives:

- `compile.sh` → `git commit` (commit hash changes, tree unchanged)
- `git cherry-pick` / `git rebase` of a tested commit (tree preserved)
- Unrelated commits (e.g. STATE.md edits) that don't change source content

Different tree content → different tree hash → marker mismatch → re-test
required. This is the safety property: untested source changes are blocked,
but workflow churn that doesn't change source content is not.

**Caveat (HEAD^{tree} is committed-only)**: the tree hash is computed from
`HEAD^{tree}`, which reflects the tree of the most recent commit. Uncommitted
edits in the working directory do NOT change the tree hash. If you run
`compile.sh` with a dirty working tree, the marker records HEAD's tree — not
the actually-tested content. `mark-tests-passed.sh` emits a stderr warning
when it detects a dirty working tree at marker-write time; commit before
relying on the marker.

**Marker scope (per-worktree)**: the marker file is suffixed with a short
hash of the worktree toplevel path (`sha1(git rev-parse --show-toplevel)`),
so two worktrees of the same repo on the same host do not silently share a
marker even when their trees happen to coincide. Path:
`/tmp/atlas-test-markers/tests-passed.<12-hex>`. Note: `/tmp` is cleared on
reboot — markers need to be regenerated after host reboot (this is intended;
re-running tests after a reboot is cheap insurance).

**Marker format (v2)**: the marker is a single line
`v2 tree <tree_hash> <iso8601_timestamp>`. The push guard rejects any other
format (empty file, partial write, pre-v2 commit-hash-keyed marker) with a
"regenerate" message rather than silently accepting — pre-v2 markers were
40-char hex strings indistinguishable from tree hashes by shape.

Also enforces:
- No direct push to `main` / `master` (refspec form too)
- No deletion of the `main` / `master` remote branch
- Docs/config-only pushes (CLAUDE.md, *.md, .claude/**, .gitignore,
  .editorconfig) are exempt from the marker requirement
- `gh pr merge` requires a `pr-reviewed-{N}` marker (commit-hash-keyed,
  via `headRefOid` from the GitHub API — PR review is intentionally
  re-run on every new commit). The marker is written by
  `pr-review-marker.sh` using `gh pr view <N> --json headRefOid` so that
  the recorded SHA always matches what the merge guard compares against,
  even when local HEAD is on a different PR's branch. If `gh` is
  unavailable, the marker falls back to local HEAD ONLY when the current
  branch matches the PR's headRefName; otherwise the marker is refused
  (silent wrong markers caused the cross-PR block bug that motivated
  this — see commit history on `.claude/hooks/pr-review-marker.sh`).

**Configuration**: `.claude/settings.local.json`
```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "/home/james/ATLAS/.claude/hooks/git-push-guard.sh",
            "timeout": 5
          }
        ]
      }
    ]
  }
}
```

## Adding New Hooks

1. Create script in `.claude/hooks/`
2. **Make it executable**: `chmod +x script.sh`
3. Read `tool_input` from stdin JSON
4. Output JSON with `hookSpecificOutput`:
   - `permissionDecision`: "allow" | "deny"
   - `permissionDecisionReason`: message shown to Claude
   - `additionalContext`: extra context for allowed operations
5. Add to `.claude/settings.local.json` hooks config

**Important**: Scripts must have execute permission. Without it, you'll see
`PreToolUse:Bash hook error` on every command but tools will still run.

## Hook Output Format

```json
{
  "hookSpecificOutput": {
    "hookEventName": "PreToolUse",
    "permissionDecision": "deny",
    "permissionDecisionReason": "Reason shown to Claude"
  }
}
```

## Exit Codes

- `0`: Continue (allow or with modifications)
- `2`: Block command, show stderr to Claude

## EF Migration Guard

**Purpose**: Prevent manual creation of EF migration files.

**Problem**: Creating migration `.cs` files manually (without `dotnet ef migrations add`)
produces incomplete migrations missing the required `Designer.cs` file. EF Core then:
1. Records the migration in `__EFMigrationsHistory`
2. But does NOT apply the schema changes
3. Runtime errors like "column X does not exist"

**Correct Process**:
```bash
nerdctl compose exec -T {svc}-dev dotnet ef migrations add {Name} --project src/Data
```

**Blocked Pattern**: Any Write to `Migrations/[timestamp]_[Name].cs` that isn't a Designer.cs

## Ansible Gate Guard

**Purpose**: Per `ARCH_PREF` — edits to deployment/CI gate files should prompt
for explicit intent so gates are fixed at the root, not worked around.

**Matched paths**: `deployment/inventory/*`, `playbooks/*`, any
`.devcontainer/compile.sh`, `.claude/hooks/git-push-guard.sh`.

**Decision**: `ask` (harness prompts user). Fail-open on missing jq or
unparseable input (still asks).

**Session bypass**: `touch $CLAUDE_PROJECT_DIR/.claude/.ansible-gate-confirmed`
to skip the prompt for the rest of the session; delete when done.

## Deploy Smoke Reminder

**Purpose**: Per `VERIFY_TEST` — every deploy must be followed by an
(unsolicited) smoke test. This `PostToolUse` hook injects a reminder into
Claude's context whenever a real deploy/restart command completes.

**Matched commands** (leading token after optional `sudo`):
- `ansible-playbook ...deploy.yml`
- `nerdctl|docker|podman compose up`
- `compose up`
- `systemctl restart|start`

Substring matching on the full command was deliberately avoided to prevent
false positives from `echo`, `grep`, `git log` etc.
