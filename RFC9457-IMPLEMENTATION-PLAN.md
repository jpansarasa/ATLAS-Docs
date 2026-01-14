# RFC 9457 Implementation Plan

## Overview

Adopt RFC 9457 (Problem Details for HTTP APIs) consistently across all ATLAS REST APIs.

### What is RFC 9457?

RFC 9457 (obsoletes RFC 7807) standardizes HTTP API error responses with:
- `type` - URI identifying the problem category
- `status` - HTTP status code
- `title` - Human-readable summary
- `detail` - Specific explanation for this occurrence
- `instance` - URI identifying this error instance (for debugging)
- Extension fields - Custom context (traceId, resource IDs, etc.)

### Why Adopt It?

| Benefit | Description |
|---------|-------------|
| Consistency | All services return same error structure |
| Machine-readable | Clients can handle errors programmatically via `type` |
| Debugging | `instance` links directly to Tempo traces |
| Extensibility | Add domain-specific context without breaking clients |
| Standards compliance | Industry standard, built into ASP.NET Core |

---

## Current State

| Service | Current Error Format | RFC 9457 Status |
|---------|---------------------|-----------------|
| FredCollector | `ProblemDetails` | Partial (missing `type`, `instance`) |
| ThresholdEngine | `ProblemDetails` | Partial (missing `type`, `instance`) |
| SecMaster | `BadRequest<string>` | None |
| CalendarService | Simple results | None |
| AlertService | Plain responses | None |
| SentinelCollector | HTML pages | N/A (UI, not REST) |

---

## Implementation Approach

### Option A: Shared Library
Create `ATLAS.Common` NuGet package with shared ProblemDetails infrastructure.
- Pro: Single source of truth
- Con: New project, versioning complexity, deployment overhead

### Option B: Documented Pattern (Recommended)
Document a consistent pattern, implement inline per-service.
- Pro: Simple, no new dependencies, services remain independent
- Con: Minor code duplication (~50 lines per service)

**Decision**: Option B - services are already independent; shared library adds complexity for minimal code.

---

## Problem Type URI Scheme

### Format
```
/problems/{category}/{specific-error}
```

### Standard Categories

| URI | HTTP Status | Description |
|-----|-------------|-------------|
| `/problems/not-found/series` | 404 | FRED series not found |
| `/problems/not-found/instrument` | 404 | SecMaster instrument not found |
| `/problems/not-found/pattern` | 404 | ThresholdEngine pattern not found |
| `/problems/validation/query-required` | 400 | Missing required query param |
| `/problems/validation/query-too-long` | 400 | Query exceeds max length |
| `/problems/validation/invalid-range` | 400 | Numeric value out of bounds |
| `/problems/conflict/already-exists` | 409 | Resource already exists |
| `/problems/upstream/unavailable` | 502 | External API unavailable |
| `/problems/upstream/rate-limited` | 429 | External API rate limit |
| `/problems/internal/unexpected` | 500 | Unhandled exception |

---

## Implementation Pattern

### 1. Program.cs Configuration

Add to each service:

```csharp
builder.Services.AddProblemDetails(options =>
{
    options.CustomizeProblemDetails = ctx =>
    {
        var activity = Activity.Current;
        if (activity?.TraceId.ToString() is { } traceId)
        {
            ctx.ProblemDetails.Instance = $"/traces/{traceId}";
            ctx.ProblemDetails.Extensions["traceId"] = traceId;
        }
        ctx.ProblemDetails.Extensions["timestamp"] = DateTime.UtcNow.ToString("O");
        ctx.ProblemDetails.Extensions["service"] = "service-name";
    };
});

// Add exception handler
app.UseExceptionHandler();
app.UseStatusCodePages();
```

### 2. Problem Types Constants

Per-service static class:

```csharp
public static class ProblemTypes
{
    private const string Base = "/problems";

    // Not Found
    public const string SeriesNotFound = $"{Base}/not-found/series";
    public const string InstrumentNotFound = $"{Base}/not-found/instrument";

    // Validation
    public const string QueryRequired = $"{Base}/validation/query-required";
    public const string QueryTooLong = $"{Base}/validation/query-too-long";
    public const string InvalidRange = $"{Base}/validation/invalid-range";

    // Conflict
    public const string AlreadyExists = $"{Base}/conflict/already-exists";

    // Upstream
    public const string UpstreamUnavailable = $"{Base}/upstream/unavailable";
}
```

### 3. Endpoint Error Responses

**Before** (current):
```csharp
return TypedResults.BadRequest("Query parameter is required");
return TypedResults.NotFound();
```

**After** (RFC 9457):
```csharp
return TypedResults.Problem(
    type: ProblemTypes.QueryRequired,
    title: "Validation Error",
    detail: "Query parameter 'q' is required",
    statusCode: 400);

return TypedResults.Problem(
    type: ProblemTypes.SeriesNotFound,
    title: "Series Not Found",
    detail: $"Series '{seriesId}' does not exist",
    statusCode: 404);
```

### 4. Global Exception Handler (Optional)

For unhandled exceptions:

```csharp
public class GlobalExceptionHandler : IExceptionHandler
{
    private readonly ILogger<GlobalExceptionHandler> _logger;

    public async ValueTask<bool> TryHandleAsync(
        HttpContext context, Exception exception, CancellationToken ct)
    {
        _logger.LogError(exception, "Unhandled exception");

        var problem = new ProblemDetails
        {
            Type = "/problems/internal/unexpected",
            Title = "Internal Server Error",
            Status = 500,
            Detail = "An unexpected error occurred"
        };

        if (Activity.Current?.TraceId.ToString() is { } traceId)
        {
            problem.Instance = $"/traces/{traceId}";
            problem.Extensions["traceId"] = traceId;
        }

        context.Response.StatusCode = 500;
        await context.Response.WriteAsJsonAsync(problem, ct);
        return true;
    }
}
```

---

## Implementation Phases

### Phase 1: SecMaster (Highest Impact)

**Files to modify:**
- `SecMaster/src/Program.cs` - Add ProblemDetails configuration
- `SecMaster/src/Endpoints/SearchEndpoints.cs`
- `SecMaster/src/Endpoints/RegistrationEndpoints.cs`
- `SecMaster/src/Endpoints/InstrumentEndpoints.cs`
- `SecMaster/src/Endpoints/ResolutionEndpoints.cs`
- `SecMaster/src/Endpoints/CatalogEndpoints.cs`
- `SecMaster/src/Endpoints/SemanticSearchEndpoints.cs`
- `SecMaster/src/Endpoints/CollectorEndpoints.cs`

**Effort**: Medium (~50 error response changes)

### Phase 2: CalendarService (Simple)

**Files to modify:**
- `CalendarService/src/Program.cs`
- `CalendarService/src/Endpoints/MarketCalendarEndpoints.cs`

**Effort**: Low (~5 error responses)

### Phase 3: FredCollector (Enhance Existing)

Already uses ProblemDetails, just add `type` and `instance`.

**Files to modify:**
- `FredCollector/src/Program.cs` - Add customization
- `FredCollector/src/Endpoints/ApiEndpoints.cs`
- `FredCollector/src/Endpoints/AdminEndpoints.cs`

**Effort**: Low (add fields to existing ProblemDetails)

### Phase 4: ThresholdEngine (Enhance Existing)

**Files to modify:**
- `ThresholdEngine/src/Program.cs`
- `ThresholdEngine/src/Endpoints/PatternEndpoints.cs`

**Effort**: Low

### Phase 5: AlertService (Minimal)

**Files to modify:**
- `AlertService/src/Program.cs`
- `AlertService/src/Endpoints/AlertEndpoints.cs`

**Effort**: Low (few error cases)

### Phase 6: Validation

- Run `compile.sh` for each modified service
- Run all tests
- Verify error responses return correct format

---

## Example: Before/After

### SecMaster SearchEndpoints.cs

**Before:**
```csharp
if (string.IsNullOrWhiteSpace(query))
    return TypedResults.BadRequest("Query parameter 'q' is required");

if (query.Length > 200)
    return TypedResults.BadRequest("Query too long (max 200 characters)");
```

**After:**
```csharp
if (string.IsNullOrWhiteSpace(query))
    return TypedResults.Problem(
        type: ProblemTypes.QueryRequired,
        title: "Validation Error",
        detail: "Query parameter 'q' is required",
        statusCode: 400);

if (query.Length > 200)
    return TypedResults.Problem(
        type: ProblemTypes.QueryTooLong,
        title: "Validation Error",
        detail: $"Query exceeds maximum length of 200 characters (was {query.Length})",
        statusCode: 400,
        extensions: new Dictionary<string, object?> { ["maxLength"] = 200, ["actualLength"] = query.Length });
```

### API Response Example

```json
{
  "type": "/problems/validation/query-too-long",
  "title": "Validation Error",
  "status": 400,
  "detail": "Query exceeds maximum length of 200 characters (was 250)",
  "instance": "/traces/abc123def456",
  "traceId": "abc123def456",
  "timestamp": "2026-01-13T10:30:00.0000000Z",
  "service": "secmaster",
  "maxLength": 200,
  "actualLength": 250
}
```

---

## Effort Summary

| Phase | Service | Files | Complexity | Time |
|-------|---------|-------|------------|------|
| 1 | SecMaster | 8 | Medium | ~1 hr |
| 2 | CalendarService | 2 | Low | ~15 min |
| 3 | FredCollector | 3 | Low | ~20 min |
| 4 | ThresholdEngine | 2 | Low | ~15 min |
| 5 | AlertService | 2 | Low | ~10 min |
| 6 | Validation | - | - | ~30 min |

**Total**: ~17 files, ~2.5 hours

---

## Risks & Mitigations

| Risk | Likelihood | Mitigation |
|------|------------|------------|
| Breaking existing clients | Low | ProblemDetails adds fields; status codes unchanged |
| Inconsistent implementation | Medium | Follow this documented pattern exactly |
| Missing error cases | Medium | Search for all `BadRequest`, `NotFound` calls |

---

## Decision Points for Review

1. **Problem type URI scheme** - Is `/problems/{category}/{specific}` acceptable?
2. **Extension fields** - Should we add more standard fields (requestId, userId)?
3. **Global exception handler** - Should we add this for unhandled exceptions?
4. **Phased rollout** - Implement all at once or service-by-service?

---

## Next Steps

Once approved:
1. Start with Phase 1 (SecMaster) as it has the most endpoints
2. Establish the pattern, then apply to remaining services
3. Run full test suite after each phase
4. Commit and push when all phases complete
