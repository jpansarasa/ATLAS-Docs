---
name: observability-review
description: Use when reviewing C#/.NET code changes, a pull request, or a git diff for any ATLAS service (collectors, ThresholdEngine, SecMaster, AlertService, MacroSubstrate) before approving or merging — verifies OpenTelemetry + Serilog observability compliance (OTEL metrics, exception tracing in catch blocks, structured logging, log levels, tag cardinality, runtime instrumentation). Run on every code review; pr-review-toolkit does NOT cover observability.
argument-hint: optional scope (files or glob); defaults to changed *.cs from git diff
---

# OBS_REVIEW [CLAUDE.md OBS_STACK compliance]

Observability is NOT one of `pr-review-toolkit:review-pr`'s default aspects (code/tests/errors/types/comments) — run THIS pass on every C#/.NET review so OTEL/Serilog gaps don't ship.

AUTHORITATIVE STANDARD (consult, don't duplicate):
- `docs/OBSERVABILITY.md` — canonical OTel `Program.cs` config, per-service metric catalogs, `.NET Runtime Metrics` + the cgroup-quota sizing lesson, and Best Practices §1–§4 (metrics-at-boundaries, tag-cardinality, exception-tracing, streaming).
- CLAUDE.md `OBSERVABILITY` + `LOG_RULES` (this repo) and `~/.claude/CLAUDE.md` LOG_RULES.

## WHEN
- ANY review / PR / diff that touches `*.cs` in an ATLAS .NET service — run alongside `pr-review-toolkit:review-pr`.
- Skip only for docs/config/test-only diffs with no instrumentation surface.

## SCOPE
`$ARGUMENTS` | default: `git diff --name-only` → `*.cs` (whole PR: `git diff main...HEAD`).

## EXECUTION

### Step 1: LSP (blocking)
`mcp__ide__getDiagnostics` → catch compile errors before reviewing instrumentation.

### Step 2: the 8 checks
Large diff → dispatch in parallel via Task `subagent_type: "general-purpose"`; small diff → run inline. Each reports `file:line → verdict`.

**A1 — Metrics boundary** (`docs/OBSERVABILITY.md` Best Practices §1)
```
grep: `Meter\.|_counter|\.Add\(1|\.Record\(`
✓ *Endpoints.cs|*GrpcService.cs→service_edge · *ApiClient.cs→external_api · *Worker.cs:ExecuteAsync→background · BackfillService|*CollectionService→long_running · gRPC client→cross_process
✗ *Repository.cs duplicating a service metric→double_count · internal helpers→use_traces
METRIC_VALUE_TEST: actionable ∧ variable ∧ observed(dashboarded|alerted)
report: file:line → OK_boundary | VIOLATION_double_count
```

**A2 — Exception tracing** (`docs/OBSERVABILITY.md` "Recording Exceptions" + Best Practices §3)
```
grep: `catch.*Exception` -C10
RULE_1 explicit activity in scope → require BOTH: activity?.SetStatus(ActivityStatusCode.Error, ex.Message) ∧ activity?.AddException(ex)
RULE_2 swallowed catch (returns fallback 0/null/[] ¬rethrow) in a TRACED caller → require BOTH on the ambient span:
  Activity.Current?.SetStatus(ActivityStatusCode.Error, ex.Message) ∧ Activity.Current?.AddException(ex)
  DETECT: catch returns default ¬rethrows; VERIFY: a caller in-project starts an activity that would be active
report: file:line → missing SetStatus | missing AddException | missing Activity.Current
```

**A3 — Structured logging** (`docs/OBSERVABILITY.md` "Structured Logging")
```
grep: `Log(Information|Warning|Error|Debug)\(\$"`  → string interpolation in log = VIOLATION
fix: structured template Log…("…{Param}", param)
report: file:line → string_interpolation_in_log
```

**A4 — Tag cardinality** (`docs/OBSERVABILITY.md` Best Practices §2 / "Required Tags")
```
grep: `SetTag.*_id"`
BOUNDED ✓: series_id(<100) | method | status | source_collector
UNBOUNDED ✗: user_id | request_id | event_id | correlation_id
report: file:line → cardinality_assessment
```

**A5 — Streaming metrics** (`docs/OBSERVABILITY.md` Best Practices §4)
```
grep: `await foreach`
long_running_stream → counter INSIDE loop (per-event visibility) ✓ · counter after loop → acceptable only for short streams ⚠
report: file:line → assessment
```

**A6 — Log-level classification** (`docs/OBSERVABILITY.md` "Log Levels" + CLAUDE.md LOG_RULES)
```
LogInformation→LogDebug: "(Processing|Checking|Iterating)" | "item \d+ of" | "[Ss]kipping.*already exists"   # verbose/loop/dup-handling
LogInformation→LogWarning/Error: "[Ff]ailed" | "[Ee]rror"
LogWarning→LogInformation: "(Client|User).*(subscribed|connected|started)" | "Successfully"
LogError→LogWarning (system continues = degraded, not Error): "will retry" | "[Tt]ransient" | "[Cc]ontinuing" | "[Ss]kipping" | "[Ff]alling back" | "[Uu]sing default" | "[Pp]artial" | "[Dd]egraded" | "next (item|query|record)"
RULES: Debug=debug_sessions_only · Info=runtime_diagnostics|routine|expected_retries|client_disconnect · Warn=unexpected_recoverable|degraded|service_startup · Error=failures|requires_attention
EXCEPTION: service_startup banner ("Starting.*Service") → LogWarning (restart visibility)
report: file:line → suggested_level
```

**A7 — Metrics coverage**
```
PART_A unused: *Meter.cs CreateCounter|CreateHistogram|CreateGauge with no .Add(/.Record( → defined_but_unused (dead code | missing instrumentation)
PART_B missing at uncertainty_boundaries: *ApiClient.cs→duration+error_counter · BackfillService→duration+processed · *CollectionService→duration+items · *Worker.cs(long_running)→duration+batch · gRPC endpoints→duration+errors
NOT_REQUIRED (use traces): internal helpers | repository | delegation
report: UNUSED list | MISSING list → recommendation
```

**A8 — Runtime instrumentation** (`docs/OBSERVABILITY.md` "Service Configuration" + ".NET Runtime Metrics") — the check generic reviewers miss
```
scope: Program.cs + matching *.csproj
REQUIRE on every .NET service exporting metrics:
  ✓ .WithMetrics(...).AddRuntimeInstrumentation()
  ✓ csproj refs OpenTelemetry.Instrumentation.Runtime (version aligned w/ other OpenTelemetry.* pkgs; no NU1902)
  rationale: GC/heap/threadpool/exception + dotnet_process_cpu_count (cgroup cpu.max, NOT host cores) visibility
SIZING_CHECK (diff touches compose/deploy cpus|memory limits): size cpus to real parallelism (CLR ProcessorCount ← cpu.max); separate managed-heap (dotnet_gc_*) from native (container_memory_anon_bytes)
report: file → has_runtime_instrumentation | MISSING .AddRuntimeInstrumentation() | MISSING package
```

### Step 3: Aggregate → OUTPUT_FORMAT

## OUTPUT_FORMAT
```
# Observability Review: [scope]
## LSP Diagnostics
[errors from mcp__ide__getDiagnostics]
## CRITICAL (must fix)
- [file:line] description → fix
## WARNING (should fix)
- [file:line] description → fix
## PASSED
- A1 metrics boundary ✓ · A2 exception tracing ✓ · A3 structured logging ✓ · A4 tag cardinality ✓
- A5 streaming metrics ✓ · A6 log level ✓ · A7 metrics coverage ✓ · A8 runtime instrumentation ✓
```

## REFERENCE

catch (explicit activity in scope):
```csharp
catch (Exception ex)
{
    activity?.SetStatus(ActivityStatusCode.Error, ex.Message);
    activity?.AddException(ex);
    _logger.LogError(ex, "Message {Param}", param);
    throw;
}
```
catch (swallowed in a traced caller — record on the ambient span):
```csharp
catch (Exception ex)
{
    Activity.Current?.SetStatus(ActivityStatusCode.Error, ex.Message);
    Activity.Current?.AddException(ex);
    _logger.LogWarning(ex, "Message {Param}", param);
    return fallbackValue;
}
```
- metrics location ✓ service_edge|external_api|database_bulk|cross_process|long_running · ✗ internal_method|double_count
- metric_value_test: actionable ∧ variable ∧ observed
- canonical OTel `Program.cs` (with `.AddRuntimeInstrumentation()`), metric naming `{service}.{component}.{metric}`, and per-service metric catalogs live in `docs/OBSERVABILITY.md` — cite it for the standard rather than guessing.
