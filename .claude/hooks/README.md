# Claude Code Hooks

Context-aware hooks that inject patterns when working on specific file types.

## Active Hooks

| Hook | Trigger | Purpose |
|------|---------|---------|
| `testing-context.sh` | Edit/Write `*Tests.cs`, `*_test.*`, `test_*.*` | Inject AAA, naming, assertions patterns |
| `benchmark-context.sh` | Edit/Write `*Benchmark*.cs`, `*_bench.*` | Inject BenchmarkDotNet, Release mode |
| `git-push-guard.sh` | Bash `git push*` | **BLOCK** - requires tests pass first |
| `ef-migration-guard.sh` | Write `Migrations/*_*.cs` | **BLOCK** - prevents manual migration creation |

## How It Works

1. Claude calls a tool (Edit, Write, Bash, etc.)
2. `PreToolUse` hook runs before execution
3. Hook checks input and decides: allow/deny/modify
4. If denied: tool execution blocked with message
5. If allowed: tool executes normally

## Git Push Guard

**Purpose**: Prevent pushing code without running tests first.

**Behavior**: Blocks any `git push` command with:
```
BLOCKED: git push requires tests to pass first.

Before pushing, you MUST:
1. Run {Project}/.devcontainer/compile.sh (without --no-test)
2. Verify: 0 errors AND 0 warnings
3. All tests pass

Only then may you push.
```

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
