# generic_review — reusable diff review

# Objective
Review a code change for CLAUDE.md compliance, scope discipline, test coverage, and absence of backwards-compat cruft. Return PASS/FAIL with concrete, actionable issues.

# Inputs
One of:
- `diff_command`: shell command producing the diff (e.g. `git diff main..HEAD`, `git show <sha>`). Preferred.
- `files`: list of changed file absolute paths (fallback).
- Optional `context`: one sentence on what the change is supposed to do.

Reference (do NOT modify):
- `/home/james/ATLAS/CLAUDE.md` — style, anti-patterns, priority rules
- `/home/james/.claude/plans/we-need-to-fix-twinkly-dragon.md` — phase constraints

# Commands
1. Produce diff via `diff_command`, else `for f in files; do git diff -- "$f"; done`. Empty diff → FAIL reason `empty_diff`.
2. Enumerate changed files. Apply per-language idiom rules from CLAUDE.md `## IDIOM_MAP`.
3. Run checklist; each failure is one concrete issue.

## Checklist
- **SCOPE**: every hunk serves `context`. Drive-by edits, reformatting, or files outside the task = violation.
- **COMPLETE_IMPL**: no `TODO`, `FIXME`, `XXX`, `NotImplementedException`, `// stub`, `pass  # placeholder`.
- **NO_BACKCOMPAT_CRUFT**: no `[Obsolete]` without plan rationale, no dup old+new paths, no `if (legacy)` dead code. Redesign phases replace, not accrete.
- **CLAUDE.md ANTI**: no base `Entity{id,name}`, no premature abstract/interface-soup, no `LogInformation` in hot paths, no try/catch-swallow, no sleep loops, no workaround-over-root-cause.
- **LOGGING**: Warn/Error at failure sites; structured params (`"{Field}"`, not concat); no new `LogDebug` in services.
- **DATETIME**: `DateTime.ToString("O")` (ISO 8601) per `## DATETIME_FORMAT`.
- **TESTS_UPDATED**: `src/**/*.cs` changed → matching `*Tests.cs` updated, or justification in `context`. Scripts need a smoke invocation.
- **BOUNDARIES**: public entries validate; internals trust; externals catch+log with context fields.
- **GRAFANA** (if dashboard JSON): no `${DS_NAME}`, no `rate(metric[1m])`, `fieldConfig.defaults` present, stat panels aggregate.
- **DB_MIGRATIONS** (if `Migrations/*.cs`): `{Name}.cs` + `{Name}.Designer.cs` + updated `ModelSnapshot.cs`.
- **UNINTENDED**: no edits to `CLAUDE.md`, `STATE.md`, `/opt/ai-inference/compose.yaml`, `.azure-foundry-keys`, or files outside scope.

# Acceptance
- Deterministic for a given diff.
- Every issue cites: file, line (from `+++`/`@@`), violated rule, suggested fix.

# Report shape
- `verdict`: `"PASS"` | `"FAIL"`
- `files_reviewed`: list of absolute paths
- `issues`: list of `{file, line, severity:"error"|"warn", rule, message, suggested_fix}`
- `tests_present`: bool
- `unintended_changes`: list of paths (empty on PASS)
- `notes`: one-paragraph intent-vs-reality summary
