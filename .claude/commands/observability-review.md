---
description: "Review C# code for observability compliance (metrics, tracing, logging) (project)"
allowed-tools: ["Bash", "Glob", "Grep", "Read", "Task", "mcp__ide__getDiagnostics"]
---

# OBS_REVIEW [CLAUDE.md:OBS_STACK compliance]

## SCOPE
$ARGUMENTS | default: `git diff --name-only` → *.cs

## EXECUTION

### Step 1: LSP (blocking)
`mcp__ide__getDiagnostics` → catch compile errors first

### Step 2: Parallel Agents
Launch via Task tool `subagent_type: "general-purpose"`:

**A1: Metrics Boundary**
```
scope: {scope}
grep: `Meter\.|_counter|\.Add\(1`

LOCATION_CHECK:
  ✓ *Endpoints.cs | *GrpcService.cs → service_edge
  ✓ *ApiClient.cs → external_api
  ✓ *Worker.cs:ExecuteAsync → background_ops
  ✓ BackfillService | *CollectionService → long_running
  ✓ gRPC client calls → cross_process
  ✗ *Repository.cs duplicating service metrics → double_count
  ✗ internal helpers → use_traces

METRIC_VALUE_TEST: actionable ∧ variable ∧ observed
  actionable: degradation → clear_investigation_path
  variable: changes_over_time ¬ always_same
  observed: dashboarded | alerted

report: file:line → OK_boundary | VIOLATION_double_count
```

**A2: Exception Tracing**
```
scope: {scope}
grep: `catch.*Exception` -C10

RULE: activity_in_scope → require_both:
  ✓ SetStatus(ActivityStatusCode.Error, ex.Message)
  ✓ AddException(ex)

report: file:line → missing SetStatus | missing AddException
```

**A3: Structured Logging**
```
scope: {scope}
grep: `Log.*\$"` in *.cs

VIOLATION: Log(Information|Warning|Error|Debug)\(\$"
  fix: use structured template {Param}

report: file:line → string_interpolation_in_log
```

**A4: Tag Cardinality**
```
scope: {scope}
grep: `SetTag.*_id"`

BOUNDED (OK): series_id(<100) | method | status | source
UNBOUNDED (✗): user_id | request_id | event_id | correlation_id

report: file:line → cardinality_assessment
```

**A5: Streaming Metrics**
```
scope: {scope}
grep: `await foreach`

CHECK: long_running_stream → counter_inside_loop
  ⚠ counter_after_loop → acceptable_for_short_streams
  ✓ counter_in_loop_body → proper_visibility

report: file:line → assessment
```

**A6: Log Level Classification**
```
scope: {scope}

MISCLASSIFIED:
  LogWarning → LogInformation:
    grep: `LogWarning.*"(Client|User).*(subscribed|connected|started)"`
    grep: `LogWarning.*"Successfully"`

  LogInformation → LogWarning:
    grep: `LogInformation.*"[Ff]ailed"`
    grep: `LogInformation.*"[Ee]rror"`

  LogError → LogWarning:
    grep: `LogError.*"will retry"` → recoverable
    grep: `LogError.*"[Tt]ransient"` → recoverable

RULES (CLAUDE.md:LOG_RULES):
  LogInformation: routine_ops | expected_retries | client_disconnect
  LogWarning: unexpected_but_recoverable | degraded_state
  LogError: failures | exceptions | requires_attention

report: file:line → suggested_level
```

**A7: Metrics Coverage**
```
scope: {scope}

PART_A [unused_metrics]:
  find: *Meter.cs → CreateCounter | CreateHistogram | CreateGauge
  check: usage exists → .Add( | .Record(
  ✗ defined_but_unused → missing_instrumentation | dead_code

PART_B [missing_metrics]:
  REQUIRED at uncertainty_boundaries:
    *ApiClient.cs → duration_histogram + error_counter
    BackfillService → duration + observations_processed
    *CollectionService → duration + items_collected
    *Worker.cs (long_running) → duration + batch_counts
    gRPC endpoints → duration + error_counts

  NOT_REQUIRED (use_traces):
    internal_helpers | repository_methods | delegation_methods

UNCERTAINTY_BOUNDARY: ¬control_other_side
  ✓ external_api | db_bulk_ops | long_running_background
  ✗ internal_method_chains

report: UNUSED list | MISSING list → recommendation
```

### Step 3: Aggregate
Collect agent outputs → final report

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
- Metrics boundary: ✓
- Exception tracing: ✓
- Structured logging: ✓
- Tag cardinality: ✓
- Streaming metrics: ✓
- Log level: ✓
- Metrics coverage: ✓
```

## REFERENCE

**catch_block_pattern:**
```csharp
catch (Exception ex)
{
    activity?.SetStatus(ActivityStatusCode.Error, ex.Message);
    activity?.AddException(ex);
    _logger.LogError(ex, "Message {Param}", param);
    throw;
}
```

**metrics_location:**
  ✓ service_edge | external_api | database_bulk | cross_process | long_running
  ✗ internal_method | double_count

**metric_value_test:** actionable ∧ variable ∧ observed
