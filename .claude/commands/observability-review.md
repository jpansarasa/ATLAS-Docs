---
description: "Review C# code for observability compliance (metrics, tracing, logging)"
allowed-tools: ["Bash", "Glob", "Grep", "Read", "mcp__ide__getDiagnostics"]
---

# Observability Code Review

Review code for ATLAS observability compliance based on docs/OBSERVABILITY.md.

## Scope
$ARGUMENTS (defaults to modified .cs files via `git diff --name-only`)

## Review Steps

### 1. Get LSP Diagnostics
Run `mcp__ide__getDiagnostics` on the target files to catch compile errors first.

### 2. Check Metrics Boundary
Search for metric/counter usage in wrong layers:
- VIOLATION: `*Repository.cs` files with `_counter.Add` or `_meter.`
- VIOLATION: Internal service classes (non-gRPC) with metrics
- OK: `*GrpcService.cs`, `*Endpoints.cs`, `*Worker.cs` (ExecuteAsync)

Grep pattern: `_counter|_meter|\.Add\(1` in `*Repository.cs`

### 3. Check Exception Tracing
Find catch blocks missing SetStatus or AddException:
- Look for `catch.*Exception` blocks
- Check if `activity` variable in scope (look for `using var activity =` above)
- VIOLATION: Activity in scope but no `SetStatus(ActivityStatusCode.Error`
- VIOLATION: Activity in scope but no `AddException(ex)` (for full stacktrace in traces)

**Complete pattern for catch blocks with activity:**
```csharp
catch (Exception ex)
{
    activity?.SetStatus(ActivityStatusCode.Error, ex.Message);  // marks span as error
    activity?.AddException(ex);                                  // attaches full stacktrace
    // ... logging, metrics, throw/handle
}
```

**When AddException is optional:**
- If exception will be caught by a parent span that has richer context
- If this is an internal layer that always re-throws to a boundary handler
- In most cases, add it at the origin where context is richest

### 4. Check Structured Logging
Find string interpolation in log calls:
- VIOLATION: `Log(Information|Warning|Error|Debug)\(\$"`
- VIOLATION: `_logger.Log.*\$"`

Grep pattern: `Log.*\$"` in `*.cs`

### 5. Check Tag Cardinality
Find unbounded tags:
- VIOLATION: `SetTag("user_id"`, `SetTag("event_id"`, `SetTag("request_id"`, `SetTag("series_id"`
- OK: `SetTag("status"`, `SetTag("method"`, `SetTag("source"`

Grep pattern: `SetTag.*_id"` in `*.cs`

### 6. Check Streaming Metrics
For `await foreach` loops, verify counter inside loop:
- VIOLATION: Counter only after loop completion
- OK: `_counter.Add(1)` inside `await foreach` body

### 7. Check Log Level Classification
Verify log levels match the operation type:

**LogWarning misuse** (should be LogInformation):
- VIOLATION: `LogWarning.*"(Client|User|Request) (subscribed|connected|started|completed|received)` - routine ops
- VIOLATION: `LogWarning.*"(Starting|Initializing|Loading|Processing|Collected)"` - routine ops
- VIOLATION: `LogWarning.*"Successfully"` - success is routine

**LogInformation misuse** (should be LogWarning/Error):
- VIOLATION: `LogInformation.*"(failed|error|exception|timeout|retry)"` - failures need Warning+
- VIOLATION: `LogInformation.*"(degraded|unavailable|disconnected unexpectedly)"` - degraded state

**Classification rules** (from CLAUDE.md LOG_RULES):
- `LogInformation`: routine_ops | expected_retries | client_disconnect
- `LogWarning`: unexpected_but_recoverable | degraded_state
- `LogError`: failures | exceptions | requires_attention

Grep patterns:
- `LogWarning.*"(Client|User).*(subscribed|connected|started)"` → routine, should be Information
- `LogWarning.*"Successfully"` → success is routine
- `LogInformation.*"[Ff]ailed"` → failure needs Warning+

## Output Format

Produce a report with this structure:

```
# Observability Review: [scope]

## LSP Diagnostics
[Any errors from mcp__ide__getDiagnostics]

## CRITICAL (must fix)
- [file:line] Metrics in Repository layer - move to service boundary
- [file:line] Catch block missing SetStatus - add activity?.SetStatus(ActivityStatusCode.Error, ex.Message)
- [file:line] Catch block missing AddException - add activity?.AddException(ex)

## WARNING (should fix)
- [file:line] String interpolation in log - use structured template
- [file:line] Unbounded tag "series_id" - use bounded values only

## PASSED
- Metrics boundary: ✓
- Exception tracing: ✓
- Structured logging: ✓
- Tag cardinality: ✓
- Streaming metrics: ✓
- Log level classification: ✓
```
