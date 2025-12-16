# Claude Code Hooks

Context-aware hooks that inject patterns when working on specific file types.

## Active Hooks

| Hook | Trigger | Pattern Injected |
|------|---------|------------------|
| `testing-context.sh` | Edit/Write `*Tests.cs`, `*_test.*`, `test_*.*` | AAA, naming, assertions |
| `benchmark-context.sh` | Edit/Write `*Benchmark*.cs`, `*_bench.*` | BenchmarkDotNet, Release mode |

## How It Works

1. Claude calls `Edit` or `Write` tool
2. `PreToolUse` hook runs before execution
3. Hook checks file path pattern
4. If match: injects compact context as `additionalContext`
5. Tool executes with context available

## Benefits

- **Token efficient**: Context only loaded when relevant
- **Non-blocking**: Hooks allow tool execution to proceed
- **Compact**: Single-line pattern summaries vs full documentation

## Adding New Hooks

1. Create script in `.claude/hooks/`
2. Read `tool_input.file_path` from stdin JSON
3. Output JSON with `hookSpecificOutput.additionalContext`
4. Add to `.claude/settings.json` hooks config
