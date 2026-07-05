# Sentinel Trend Visualization Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add direction/momentum ("shape of the line") visualization to the Sentinel daily digest and a new live `/sentinel/matrix` explorer, both server-rendered as inline SVG over existing tables, with zero client JS.

**Architecture:** Two new SentinelCollector components — `TrendQueryService` (UTC-day-bucketed, forward-filled series over `matrix_cells`, `sector_regimes`, and sentinel rows in `macro_observations`) and `TrendSvgRenderer` (pure string-in/string-out SVG/HTML fragments) — plus a market-first restructure of the existing `DigestRenderer` and a new read-only `/sentinel/matrix?range=` endpoint that reuses the same SVG blocks. All work is read-only presentation; no schema, migration, write-path, or projector change.

**Tech Stack:** .NET 10 / C# 14, raw Npgsql over the scoped `SentinelDbContext` connection (SELECT-only, mirrors `MatrixSectorQueryService`), Markdig passthrough for embedded raw-HTML/SVG blocks, xUnit 2.9.2 + FluentAssertions 7.0.0, real-Postgres integration via the `timescaledb-test` sidecar fixture pattern.

## Global Constraints

Copied verbatim from the spec + the SentinelCollector service card (AGENT_README.md). Every task's requirements implicitly include this section.

- **Scope: SentinelCollector ONLY.** Read-only presentation over existing tables (`matrix_cells`, `sector_regimes`, `macro_observations`). No schema/migration/write-path/projector changes. No new services, no compose changes, no JSON API, no client JS, no CDN assets.
- **`:sig:` infix is a change-all-or-none string contract.** From the card: "news row identified SOLELY by literal `:sig:` in source_id (`{rawContentId}:sig:{signalId}`) ... FOUR artefacts move together: producer (MacroObservationRouter) + projector const (ObservationCellProjector) + consumer (NewsMomentumQueryService) + digest reader (DigestQueryService.ArticleSignalsSql + TryParseRawContentId). change-all-or-none." **`TrendQueryService` MUST NOT add a fifth `:sig:` filter.** Its per-signal news-tilt read uses the SAME predicate as `NewsMomentumQueryService`, NOT a `:sig:` infix filter (see next bullet).
- **The news-tilt filter is NOT a `:sig:` filter.** From the card DISTINCTIONS, verbatim: the digest news-momentum SQL filters `source_collector='sentinel' AND signal_identity_id IS NOT NULL AND value_numeric BETWEEN -1.5 AND 1.5` — "not a `:sig:` infix filter". "Legacy non-`:sig:` sentinel numeric rows WITH signal_identity_id enter digest momentum avg but ARE dropped by projector (G2). Two consumers see DIFFERENT effective inputs." The trend tilt series deliberately mirrors this momentum filter so legacy signal-bearing rows participate.
- **Projector Mode=Off → zero `matrix_cells` + no error.** From the card GOTCHAS: "`zero-matrix_cells-no-error=broken(check-Mode-first)`" and "does not run when `Matrix:ObservationProjectorMode=Off` -> zero matrix_cells + no error. CHECK this config FIRST." The empty-matrix-in-window UI notice MUST name `Matrix:ObservationProjectorMode` and MUST NOT surface as a blank 500.
- **`matrix_cells` read excludes legacy rows.** Only projector rows (`pattern_id LIKE 'obs:%'`) are valid; legacy pre-realignment rows carry NULL `source_collector` and stale values. Provenance falls back to `split_part(pattern_id, ':', 2)` (signal) / `split_part(pattern_id, ':', 3)` (collector) for `obs:` rows written before the provenance columns — mirror `MatrixSectorQueryService.LatestSectorCellsSql`.
- **`sector_regimes` may be empty / starts 2026-06-12.** An empty result is the normal state, never an error. No synthetic backfill before the first available regime day.
- **Temperature semantics (user-flipped, CVD-safe):** hot = red `#e34948`, cold = blue `#2a78d6`, neutral gray `#f0efec` (light) / `#383835` (dark). Dark steps: hot `#e66767`, cold `#3987e5`, dark surface `#1a1a19`. Dual-trend series: cell value = 1.8px **black**, news tilt = 1.4px **violet `#7a5ec4`**.
- **Color never carries meaning alone.** Numeric values always printed beside color-coded marks (heatmap row-end lean, KPI tiles, signal values). Sign-colored values carry their `+`/`−` sign as the redundant channel. Text uses ink tokens, never series colors.
- **Colors defined once** as CSS custom properties; dark variant via `@media (prefers-color-scheme: dark)`. Diverging color interpolation lives in exactly one place (`TrendSvgRenderer`).
- **Build/verify (HARD_STOP, project CLAUDE.md):** `SentinelCollector/.devcontainer/compile.sh` (WITHOUT `--no-test`) before any push; 0 errors AND 0 warnings AND all tests pass. `compile.sh` runs UnitTests then IntegrationTests then `mark-tests-passed.sh`. Invoke via `bash .devcontainer/compile.sh` (file is not +x). Filter a single test: `sudo nerdctl compose exec -T sentinel-collector-dev sh -c "cd /workspace/SentinelCollector/tests/SentinelCollector.UnitTests && dotnet test --filter 'Name~<Test>'"`.
- **Integration tests need the sidecar.** `timescaledb-test` sidecar, connection string in env `CROSSCOLLECTOR_TEST_DB`; the fixture creates minimal mirror tables. Only reachable through `compile.sh` (it starts/stops the sidecar).

---

## File Structure

New / modified files, by responsibility. Files that change together live together in `src/Services/`, mirroring the existing digest query/builder split.

**Created:**
- `src/Services/TrendQueryService.cs` — `ITrendQueryService` + `TrendQueryResult` and its record family; the four raw-SQL reads (SELECT-only) and the pure day-axis / forward-fill transforms. One responsibility: turn the three tables into UTC-day-bucketed, gap-filled series.
- `src/Services/TrendSvgRenderer.cs` — `ITrendSvgRenderer`; pure string→string SVG/HTML fragment builders (KPI tile row, heatmap strip, lean sparkline, regime strip, dual-trend chart) + the single diverging-color function + the shared `StyleBlock`. No I/O.
- `src/Endpoints/MatrixEndpoints.cs` — `MapMatrixEndpoints` minimal-API extension serving `GET /sentinel/matrix?range={7d|30d|90d|max}`.
- `tests/SentinelCollector.UnitTests/Services/TrendQueryServiceTransformTests.cs` — pure-transform unit tests (day axis, forward-fill, sparse alignment).
- `tests/SentinelCollector.UnitTests/Services/TrendSvgRendererTests.cs` — renderer business-outcome tests (color-by-sign, heatmap sort, tilt omission, regime streak, empty-matrix notice fragment).
- `tests/SentinelCollector.IntegrationTests/Infrastructure/TrendDbFixture.cs` — real-Postgres fixture; mirror tables for `matrix_cells`, `sector_regimes`, `macro_observations` (only the columns the trend SQL touches).
- `tests/SentinelCollector.IntegrationTests/Services/TrendQueryServiceIntegrationTests.cs` — SQL-level tests (net-lean latest-per-pattern-per-day not average; tilt filter excludes `|v|>1.5` and null-signal; day bucketing).

**Modified:**
- `src/DependencyInjection.cs` — register `ITrendQueryService` (scoped, reads DB) and `ITrendSvgRenderer` (singleton, pure), alongside the existing digest registrations (~lines 859-930).
- `src/Configuration/DigestOptions.cs` — add `SectorMatrixOptions.TrendWindowDays` (default 33) + validator line.
- `src/Services/DigestRenderer.cs` — market-first restructure of `BuildMarkdown`; embed KPI + heatmap + drill-down blocks + `StyleBlock`; `IDigestRenderer.Render` gains a `TrendQueryResult trends` parameter (Stage B). Stage C removes `AppendSectorHeatSection`, `AppendPerSectorBlocks`, and their now-dead helpers (`AppendCitedBullet`, `AppendSectorSignalLine`, `SignalCell`).
- `src/Services/DigestService.cs` — fetch trends via a new `BuildTrendsAsync` (same degrade-to-empty resilience contract as `BuildSectorMatrixAsync`) and pass into `Render` (Stage B).
- `src/Program.cs` — `app.MapMatrixEndpoints();` next to `app.MapDigestEndpoints();` (Stage B).
- `src/prompts/digest/narrative.md` — remove the `## Sector Watch` section instruction (Stage C). NOTE: prompts are host-mounted (`/prompts`); the repo file is the source of truth but the deploy must refresh the mount.
- `tests/SentinelCollector.UnitTests/Services/DigestRendererTests.cs` — update/remove assertions pinning the old Sector Heat table + Sector Detail lists (Stage C).
- `tests/SentinelCollector.UnitTests/Services/DigestNarrativeGeneratorTests.cs` — update the `Contain("## Sector Watch")` assertion (~line 525) (Stage C).

---

## Pipeline / Dependency Ordering (walk backward for review)

The delivered surfaces are (1) the redesigned digest HTML and (2) the live `/sentinel/matrix` page. Walking backward from each:

```
/sentinel/matrix page (B2)  ─┐
redesigned digest HTML (B1) ─┤─ both call ─▶ TrendSvgRenderer (A3-A5)  ─▶ needs colors/StyleBlock (A3)
                             │                TrendQueryResult shape     ─▶ from TrendQueryService (A1 transforms + A2 SQL)
Stage C (cutover)  ─────────▶ deletes the sections B1 superseded; must land AFTER B1 ships and is verified.
DI registration (A6)        ─▶ prerequisite for B1/B2 resolving the two new services.
```

- **Stage A** is mergeable alone: it adds `TrendQueryService` + `TrendSvgRenderer` + tests + DI, wired into nothing live. Reviewer can approve it without any page changing.
- **Stage B** integrates: digest restructure (B1) and the new endpoint (B2) both consume Stage A. Ships the new visuals ALONGSIDE the old sections (both render) so the diff is reviewable and the old view is a fallback if a chart misbehaves.
- **Stage C** cuts over: removes the superseded Sector Watch / raw-quantity bullets / old Sector Heat table + Sector Detail lists ONLY after Stage B is deployed and smoke-verified.

Each stage = its own PR. Within a stage, commit per layer (query → renderer → wiring → tests as the task boundaries dictate).

---

# STAGE A — Additive: TrendQueryService + TrendSvgRenderer (own PR, mergeable alone)

Out-of-scope for the whole stage: no change to `DigestRenderer`, `DigestService`, `Program.cs`, or any live page. Nothing here renders to a user yet.

## Task A1: TrendQueryResult shape + pure day-axis/forward-fill transforms

**Files:**
- Create: `src/Services/TrendQueryService.cs` (types + `internal static` transforms only in this task; SQL reads land in A2)
- Test: `tests/SentinelCollector.UnitTests/Services/TrendQueryServiceTransformTests.cs`

**Interfaces:**
- Produces (consumed by A2, A3-A5, B1, B2):
  ```csharp
  namespace SentinelCollector.Services;

  // UTC-day-bucketed trend series over the matrix window. Every list is aligned
  // index-for-index to Days (ascending). Forward-fill semantics differ by series
  // (per-record docs) so both the digest drill-downs and the live matrix page
  // render from one shape.
  public sealed record TrendQueryResult(
      IReadOnlyList<DateOnly> Days,
      IReadOnlyList<SectorLeanTrend> SectorLeans,
      IReadOnlyList<SectorRegimeTrend> SectorRegimes,
      IReadOnlyList<SignalDualTrend> SignalTrends);

  // Net lean per sector per day. NetLeanByDay[i] aligns to Days[i]: null before
  // the sector's first in-window cell (no synthetic backfill); forward-filled
  // across gap days after (matrix state persists until re-projected).
  public sealed record SectorLeanTrend(string SectorCode, IReadOnlyList<decimal?> NetLeanByDay);

  // Regime per sector per day, forward-filled after the first row (a regime is
  // the sector's current classification until replaced); null before the first
  // row (history starts 2026-06-12 — no synthetic backfill).
  public sealed record SectorRegimeTrend(string SectorCode, IReadOnlyList<RegimeDay?> RegimeByDay);
  public sealed record RegimeDay(string Regime, decimal Score);

  // One (sector, signal) dual trend. CellValueByDay = daily latest-per-pattern
  // cell value averaged across patterns, forward-filled after first cell.
  // TiltByDay = daily news tilt, NOT forward-filled (news is a per-day event) —
  // null on days with no sentinel tilt rows so the renderer draws a gap.
  public sealed record SignalDualTrend(
      string SectorCode,
      string SignalIdentityId,
      IReadOnlyList<decimal?> CellValueByDay,
      IReadOnlyList<TiltDay?> TiltByDay);
  public sealed record TiltDay(double Tilt, int RowCount);

  public interface ITrendQueryService
  {
      Task<TrendQueryResult> GetTrendsAsync(
          DateTimeOffset windowStart, DateTimeOffset windowEnd, CancellationToken ct = default);
  }
  ```
- Pure transforms produced by this task (consumed by A2):
  - `internal static IReadOnlyList<DateOnly> BuildDayAxis(DateOnly firstDay, DateOnly lastDay)` — every UTC date from `firstDay` to `lastDay` inclusive, ascending.
  - `internal static List<T?> ForwardFill<T>(IReadOnlyList<DateOnly> days, IReadOnlyDictionary<DateOnly, T> observed) where T : class` — carry last observed forward; null before first observed day.
  - `internal static List<decimal?> ForwardFill(IReadOnlyList<DateOnly> days, IReadOnlyDictionary<DateOnly, decimal> observed)` — value-type overload, same semantics.
  - `internal static List<TiltDay?> AlignSparse(IReadOnlyList<DateOnly> days, IReadOnlyDictionary<DateOnly, TiltDay> observed)` — null on unobserved days, NO fill.

- [ ] **Step 1: Write the failing test** (`TrendQueryServiceTransformTests.cs`)

```csharp
using FluentAssertions;
using SentinelCollector.Services;
using Xunit;

namespace SentinelCollector.UnitTests.Services;

public class TrendQueryServiceTransformTests
{
    private static DateOnly D(int day) => new(2026, 6, day);

    [Fact]
    public void day_axis_is_every_utc_date_inclusive_ascending()
    {
        var axis = TrendQueryService.BuildDayAxis(D(1), D(4));
        axis.Should().Equal(D(1), D(2), D(3), D(4));
    }

    [Fact]
    public void forward_fill_carries_last_value_across_a_gap_day_and_is_null_before_first()
    {
        var days = TrendQueryService.BuildDayAxis(D(1), D(4));
        // Observed only on day 2 (0.5) and day 4 (0.9); day 3 is a gap.
        var observed = new Dictionary<DateOnly, decimal> { [D(2)] = 0.5m, [D(4)] = 0.9m };

        var filled = TrendQueryService.ForwardFill(days, observed);

        filled.Should().Equal(null, 0.5m, 0.5m, 0.9m); // day1 null (before first), day3 carries day2
    }

    [Fact]
    public void align_sparse_leaves_gap_days_null_and_does_not_carry_tilt_forward()
    {
        var days = TrendQueryService.BuildDayAxis(D(1), D(3));
        var observed = new Dictionary<DateOnly, TiltDay> { [D(1)] = new TiltDay(0.4, 3) };

        var aligned = TrendQueryService.AlignSparse(days, observed);

        aligned[0].Should().Be(new TiltDay(0.4, 3));
        aligned[1].Should().BeNull(); // tilt is NOT forward-filled — news is per-day
        aligned[2].Should().BeNull();
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `sudo nerdctl compose exec -T sentinel-collector-dev sh -c "cd /workspace/SentinelCollector/tests/SentinelCollector.UnitTests && dotnet test --filter 'Name~TrendQueryServiceTransformTests'"`
Expected: FAIL — `TrendQueryService` / `TiltDay` do not exist yet.

- [ ] **Step 3: Write the record family + `ITrendQueryService` + the pure transforms**

Add the record/interface block from the Interfaces section above, then the transforms:

```csharp
public sealed class TrendQueryService : ITrendQueryService
{
    // A2 adds the constructor + SQL reads + GetTrendsAsync. This task lands only
    // the pure transforms so they are unit-testable without a DB.

    internal static IReadOnlyList<DateOnly> BuildDayAxis(DateOnly firstDay, DateOnly lastDay)
    {
        var days = new List<DateOnly>();
        for (var d = firstDay; d <= lastDay; d = d.AddDays(1)) days.Add(d);
        return days;
    }

    internal static List<T?> ForwardFill<T>(
        IReadOnlyList<DateOnly> days, IReadOnlyDictionary<DateOnly, T> observed) where T : class
    {
        var result = new List<T?>(days.Count);
        T? carried = null;
        foreach (var day in days)
        {
            if (observed.TryGetValue(day, out var v)) carried = v;
            result.Add(carried);
        }
        return result;
    }

    internal static List<decimal?> ForwardFill(
        IReadOnlyList<DateOnly> days, IReadOnlyDictionary<DateOnly, decimal> observed)
    {
        var result = new List<decimal?>(days.Count);
        decimal? carried = null;
        foreach (var day in days)
        {
            if (observed.TryGetValue(day, out var v)) carried = v;
            result.Add(carried);
        }
        return result;
    }

    internal static List<TiltDay?> AlignSparse(
        IReadOnlyList<DateOnly> days, IReadOnlyDictionary<DateOnly, TiltDay> observed)
    {
        var result = new List<TiltDay?>(days.Count);
        foreach (var day in days)
            result.Add(observed.TryGetValue(day, out var v) ? v : null);
        return result;
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: same filter as Step 2. Expected: PASS (3 tests).

- [ ] **Step 5: Commit**

```bash
git add src/Services/TrendQueryService.cs tests/SentinelCollector.UnitTests/Services/TrendQueryServiceTransformTests.cs
git commit -m "feat(sentinel): TrendQueryResult shape + pure day-axis/forward-fill transforms (Stage A)"
```

**Out of scope for A1:** no SQL, no DB connection, no DI. The class has no constructor yet — A2 adds it.

---

## Task A2: TrendQueryService SQL reads + GetTrendsAsync

**Files:**
- Modify: `src/Services/TrendQueryService.cs` (add constructor, four SQL consts, `GetTrendsAsync`)
- Create: `tests/SentinelCollector.IntegrationTests/Infrastructure/TrendDbFixture.cs`
- Create: `tests/SentinelCollector.IntegrationTests/Services/TrendQueryServiceIntegrationTests.cs`

**Interfaces:**
- Consumes: the A1 record family + transforms; `SentinelDbContext` (scoped); `SentinelActivitySource.Source` (from `src/Telemetry/SentinelActivitySource.cs`, `new("SentinelCollector", "1.0.0")`).
- Produces: `GetTrendsAsync` returning a populated `TrendQueryResult`.

**SQL (grounded in the verified prod columns — mirror `MatrixSectorQueryService` idioms). Window is `[@from, @to]` half-open at neither end for the day-bucket edges; `@from` = start-of-first-UTC-day midnight, `@to` = `windowEnd` instant.**

Sector net lean (latest cell per `(pattern_id, sector_code)` within each UTC day, summed per sector):
```sql
WITH latest_per_day AS (
    SELECT DISTINCT ON ((evaluated_at AT TIME ZONE 'UTC')::date, pattern_id, sector_code)
           (evaluated_at AT TIME ZONE 'UTC')::date AS day,
           sector_code,
           cell_value
    FROM public.matrix_cells
    WHERE evaluated_at >= @from AND evaluated_at <= @to
      AND pattern_id LIKE 'obs:%'
    ORDER BY (evaluated_at AT TIME ZONE 'UTC')::date, pattern_id, sector_code, evaluated_at DESC
)
SELECT day, sector_code, sum(cell_value) AS net_lean
FROM latest_per_day
GROUP BY day, sector_code;
```

Regime history (last `(regime, score)` per sector per day):
```sql
SELECT DISTINCT ON ((evaluated_at AT TIME ZONE 'UTC')::date, sector_code)
       (evaluated_at AT TIME ZONE 'UTC')::date AS day,
       sector_code, regime, score
FROM public.sector_regimes
WHERE evaluated_at >= @from AND evaluated_at <= @to
ORDER BY (evaluated_at AT TIME ZONE 'UTC')::date, sector_code, evaluated_at DESC;
```

Per-signal daily cell value (latest cell per pattern for `(signal, sector)`, averaged across patterns per day):
```sql
WITH latest_per_day AS (
    SELECT DISTINCT ON ((evaluated_at AT TIME ZONE 'UTC')::date, pattern_id, sector_code)
           (evaluated_at AT TIME ZONE 'UTC')::date AS day,
           coalesce(signal_identity_id, split_part(pattern_id, ':', 2)) AS signal_identity_id,
           sector_code,
           cell_value
    FROM public.matrix_cells
    WHERE evaluated_at >= @from AND evaluated_at <= @to
      AND pattern_id LIKE 'obs:%'
    ORDER BY (evaluated_at AT TIME ZONE 'UTC')::date, pattern_id, sector_code, evaluated_at DESC
)
SELECT day, signal_identity_id, sector_code, avg(cell_value) AS cell_value
FROM latest_per_day
GROUP BY day, signal_identity_id, sector_code;
```

Per-signal daily news tilt — **SAME filter as `NewsMomentumQueryService`, NOT a `:sig:` filter** (Global Constraints):
```sql
SELECT (observation_time AT TIME ZONE 'UTC')::date AS day,
       signal_identity_id,
       avg(value_numeric)::double precision AS tilt,
       count(*)::int AS row_count
FROM public.macro_observations
WHERE source_collector = 'sentinel'
  AND signal_identity_id IS NOT NULL
  AND value_numeric IS NOT NULL
  AND value_numeric BETWEEN -1.5 AND 1.5
  AND observation_time >= @from AND observation_time <= @to
GROUP BY (observation_time AT TIME ZONE 'UTC')::date, signal_identity_id;
```

- [ ] **Step 1: Write the failing integration fixture + tests**

`TrendDbFixture.cs` — copy the `MatrixSectorDbFixture` shape (`CROSSCOLLECTOR_TEST_DB` env, `WaitUntilReadyAsync`, `ExecuteAsync`, `CreateContext`), with mirror tables for all three read targets and insert helpers:

```csharp
using Microsoft.EntityFrameworkCore;
using Npgsql;
using NpgsqlTypes;
using SentinelCollector.Data;
using Xunit;

namespace SentinelCollector.IntegrationTests.Infrastructure;

[CollectionDefinition("Trend_Db")]
public sealed class TrendDbCollection : ICollectionFixture<TrendDbFixture> { }

public sealed class TrendDbFixture : IAsyncLifetime
{
    private const string EnvVar = "CROSSCOLLECTOR_TEST_DB";

    // Mirror only the columns the trend SQL touches (prod names/types from
    // ThresholdEngine MatrixCellConfiguration / SectorRegimeConfiguration and
    // MacroSubstrate MacroObservationConfiguration).
    private const string CreateSchemaSql = """
        CREATE TABLE IF NOT EXISTS matrix_cells (
            pattern_id         varchar(100) NOT NULL,
            sector_code        varchar(16)  NOT NULL,
            signal_identity_id varchar(100),
            source_collector   varchar(32),
            cell_value         numeric(6,4) NOT NULL,
            evaluated_at       timestamptz  NOT NULL);
        CREATE TABLE IF NOT EXISTS sector_regimes (
            sector_code                varchar(16)  NOT NULL,
            regime                     varchar(32)  NOT NULL,
            score                      numeric(6,4) NOT NULL,
            contributing_pattern_count int          NOT NULL,
            evaluated_at               timestamptz  NOT NULL);
        CREATE TABLE IF NOT EXISTS macro_observations (
            source_collector   varchar(32),
            signal_identity_id varchar(100),
            value_numeric      numeric,
            observation_time   timestamptz);
        """;

    public string ConnectionString { get; private set; } = string.Empty;

    public async Task InitializeAsync()
    {
        ConnectionString = Environment.GetEnvironmentVariable(EnvVar)
            ?? throw new InvalidOperationException(
                $"{EnvVar} is not set. Run via SentinelCollector/.devcontainer/compile.sh.");
        await WaitUntilReadyAsync();
        await ExecuteAsync(CreateSchemaSql);
    }

    public Task DisposeAsync() => Task.CompletedTask;

    public Task ResetAsync() =>
        ExecuteAsync("TRUNCATE matrix_cells, sector_regimes, macro_observations;");

    public SentinelDbContext CreateContext() => new(
        new DbContextOptionsBuilder<SentinelDbContext>().UseNpgsql(ConnectionString).Options);

    public Task InsertCellAsync(
        string signal, string sector, decimal value, DateTimeOffset evaluatedAt) =>
        ExecuteAsync(
            "INSERT INTO matrix_cells (pattern_id, sector_code, signal_identity_id, source_collector, cell_value, evaluated_at) VALUES (@p0,@p1,@p2,@p3,@p4,@p5);",
            ("p0", NpgsqlDbType.Varchar, $"obs:{signal}:sentinel"),
            ("p1", NpgsqlDbType.Varchar, sector),
            ("p2", NpgsqlDbType.Varchar, signal),
            ("p3", NpgsqlDbType.Varchar, "sentinel"),
            ("p4", NpgsqlDbType.Numeric, value),
            ("p5", NpgsqlDbType.TimestampTz, evaluatedAt));

    public Task InsertTiltAsync(
        string? signal, decimal? value, DateTimeOffset observationTime, string collector = "sentinel") =>
        ExecuteAsync(
            "INSERT INTO macro_observations (source_collector, signal_identity_id, value_numeric, observation_time) VALUES (@p0,@p1,@p2,@p3);",
            ("p0", NpgsqlDbType.Varchar, (object?)collector ?? DBNull.Value),
            ("p1", NpgsqlDbType.Varchar, (object?)signal ?? DBNull.Value),
            ("p2", NpgsqlDbType.Numeric, (object?)value ?? DBNull.Value),
            ("p3", NpgsqlDbType.TimestampTz, observationTime));

    private async Task WaitUntilReadyAsync()
    {
        var deadline = DateTimeOffset.UtcNow.AddSeconds(60);
        while (true)
        {
            try { await using var c = new NpgsqlConnection(ConnectionString); await c.OpenAsync(); return; }
            catch (Exception) when (DateTimeOffset.UtcNow < deadline) { await Task.Delay(TimeSpan.FromSeconds(1)); }
        }
    }

    private async Task ExecuteAsync(string sql, params (string Name, NpgsqlDbType Type, object Value)[] ps)
    {
        await using var conn = new NpgsqlConnection(ConnectionString);
        await conn.OpenAsync();
        await using var cmd = new NpgsqlCommand(sql, conn);
        foreach (var (n, t, v) in ps) cmd.Parameters.Add(new NpgsqlParameter(n, t) { Value = v });
        await cmd.ExecuteNonQueryAsync();
    }
}
```

`TrendQueryServiceIntegrationTests.cs` — the SQL-level business outcomes the spec's Testing section names:

```csharp
using FluentAssertions;
using SentinelCollector.IntegrationTests.Infrastructure;
using SentinelCollector.Services;
using Xunit;

namespace SentinelCollector.IntegrationTests.Services;

[Collection("Trend_Db")]
public sealed class TrendQueryServiceIntegrationTests : IAsyncLifetime
{
    private static readonly DateTimeOffset WindowStart = new(2026, 6, 1, 0, 0, 0, TimeSpan.Zero);
    private static readonly DateTimeOffset WindowEnd = new(2026, 6, 4, 23, 59, 0, TimeSpan.Zero);
    private readonly TrendDbFixture _fixture;

    public TrendQueryServiceIntegrationTests(TrendDbFixture fixture) => _fixture = fixture;
    public Task InitializeAsync() => _fixture.ResetAsync();
    public Task DisposeAsync() => Task.CompletedTask;

    private async Task<TrendQueryResult> RunAsync()
    {
        await using var db = _fixture.CreateContext();
        var svc = new TrendQueryService(db);
        return await svc.GetTrendsAsync(WindowStart, WindowEnd);
    }

    private static DateOnly D(int day) => new(2026, 6, day);

    [Fact]
    public async Task net_lean_uses_latest_cell_per_pattern_per_day_then_sums_not_averages()
    {
        // ENERGY, same day, two patterns: oil (latest 0.6 after 0.2) + gas (0.3).
        // Net lean must be 0.6 + 0.3 = 0.9 (sum of latest-per-pattern), NOT an average.
        await _fixture.InsertCellAsync("oil-price", "ENERGY", 0.2m, WindowStart.AddDays(1).AddHours(1));
        await _fixture.InsertCellAsync("oil-price", "ENERGY", 0.6m, WindowStart.AddDays(1).AddHours(5));
        await _fixture.InsertCellAsync("nat-gas-price", "ENERGY", 0.3m, WindowStart.AddDays(1).AddHours(2));

        var result = await RunAsync();

        var energy = result.SectorLeans.Single(s => s.SectorCode == "ENERGY");
        var dayIndex = result.Days.ToList().IndexOf(D(2));
        energy.NetLeanByDay[dayIndex].Should().Be(0.9m);
    }

    [Fact]
    public async Task tilt_query_excludes_out_of_band_and_null_signal_rows()
    {
        var t = WindowStart.AddDays(1).AddHours(3);
        await _fixture.InsertTiltAsync("oil-price", 0.4m, t);   // in band, kept
        await _fixture.InsertTiltAsync("oil-price", 1.9m, t);   // |v|>1.5 raw leak, dropped
        await _fixture.InsertTiltAsync(null, 0.5m, t);          // null signal, dropped
        await _fixture.InsertTiltAsync("oil-price", 0.2m, t, collector: "fred"); // non-sentinel, dropped
        // Give oil-price a cell so it appears as a SignalDualTrend row.
        await _fixture.InsertCellAsync("oil-price", "ENERGY", 0.5m, t);

        var result = await RunAsync();

        var oil = result.SignalTrends.Single(s => s.SignalIdentityId == "oil-price");
        var dayIndex = result.Days.ToList().IndexOf(D(2));
        oil.TiltByDay[dayIndex]!.Tilt.Should().BeApproximately(0.4, 1e-9); // only the one in-band sentinel row
        oil.TiltByDay[dayIndex]!.RowCount.Should().Be(1);
    }

    [Fact]
    public async Task obs_prefix_filter_excludes_legacy_cells_and_regimes_may_be_empty()
    {
        var result = await RunAsync();
        result.SectorRegimes.Should().OnlyContain(r => r.RegimeByDay.All(x => x == null))
            .And.Subject.Should().NotBeNull(); // empty sector_regimes is normal, never an error
    }
}
```

- [ ] **Step 2: Run to verify it fails**

Run: `bash .devcontainer/compile.sh` (runs unit then integration). Expected: FAIL — `TrendQueryService` has no constructor / `GetTrendsAsync`.

- [ ] **Step 3: Implement the constructor, SQL consts, and `GetTrendsAsync`**

Add to `TrendQueryService`:
```csharp
private readonly SentinelDbContext _db;
public TrendQueryService(SentinelDbContext db) => _db = db;
```
Add the four `private const string` SQL blocks above. Implement `GetTrendsAsync`:
1. `windowStart` → `firstDay = DateOnly.FromDateTime(windowStart.UtcDateTime)`; `lastDay = DateOnly.FromDateTime(windowEnd.UtcDateTime)`; `@from` param = `new DateTimeOffset(firstDay.ToDateTime(TimeOnly.MinValue), TimeSpan.Zero)`, `@to` = `windowEnd`.
2. Open the DbContext connection (the open-if-closed / close-if-opened pattern from `NewsMomentumQueryService`), run each SQL into flat lists keyed by day.
3. Build `days = BuildDayAxis(firstDay, lastDay)`.
4. Sector leans: group net-lean rows by sector → `observed[day]=netLean` → `ForwardFill(days, observed)` → `SectorLeanTrend`.
5. Regimes: group by sector → `observed[day]=new RegimeDay(regime, score)` → `ForwardFill<RegimeDay>` → `SectorRegimeTrend`.
6. Signal trends: for each `(sector, signal)` present in the cell rows → `ForwardFill` the cell values; align tilt by signal via `AlignSparse` (tilt keyed by signal only — the same tilt series attaches to every sector that signal appears in) → `SignalDualTrend`.
7. Wrap in `using var activity = SentinelActivitySource.Source.StartActivity("Trend.GetTrends");` and tag `days.Count` + row counts.

Use one `NpgsqlDbType.TimestampTz` `@from`/`@to` param pattern (copy `AddTime` from `NewsMomentumQueryService`). All reads SELECT-only.

- [ ] **Step 4: Run to verify it passes**

Run: `bash .devcontainer/compile.sh`. Expected: unit + integration PASS (Trend integration tests green).

- [ ] **Step 5: Commit**

```bash
git add src/Services/TrendQueryService.cs tests/SentinelCollector.IntegrationTests/Infrastructure/TrendDbFixture.cs tests/SentinelCollector.IntegrationTests/Services/TrendQueryServiceIntegrationTests.cs
git commit -m "feat(sentinel): TrendQueryService raw-SQL reads over matrix/regime/macro tables (Stage A)"
```

**Out of scope for A2:** no renderer, no DI registration (A6), no live wiring. The tilt filter is the momentum filter verbatim — DO NOT add a `:sig:` predicate.

---

## Task A3: TrendSvgRenderer color model + StyleBlock

**Files:**
- Create: `src/Services/TrendSvgRenderer.cs` (interface, `StyleBlock`, `LeanColor`)
- Test: `tests/SentinelCollector.UnitTests/Services/TrendSvgRendererTests.cs`

**Interfaces:**
- Produces (consumed by A4, A5, B1, B2):
  ```csharp
  public interface ITrendSvgRenderer
  {
      string StyleBlock { get; }  // CSS custom props (light + dark) — emit once per page
      // heatmap / sparkline / regime / dual-trend / KPI methods land in A4 + A5
  }
  public sealed class TrendSvgRenderer : ITrendSvgRenderer { ... }
  ```
- `internal static string LeanColor(decimal value, decimal cap)` — diverging interpolation cold→neutral→hot, the ONE place color meaning is computed. `value <= -cap` → cold token; `>= cap` → hot token; interpolate between. Returns a CSS color string referencing the custom props / a computed hex.

Color decisions verbatim (Global Constraints): hot `#e34948`, cold `#2a78d6`, neutral `#f0efec`. `cap` clamps the ±3 cell range to a readable spread (default `3m`, matching the projector's ±3 clamp).

- [ ] **Step 1: Write the failing test**

```csharp
using FluentAssertions;
using SentinelCollector.Services;
using Xunit;

namespace SentinelCollector.UnitTests.Services;

public class TrendSvgRendererTests
{
    private readonly TrendSvgRenderer _r = new();

    [Fact]
    public void style_block_defines_hot_and_cold_custom_properties_and_a_dark_media_query()
    {
        _r.StyleBlock.Should().Contain("#e34948")   // hot (light)
            .And.Contain("#2a78d6")                 // cold (light)
            .And.Contain("prefers-color-scheme: dark")
            .And.Contain("#e66767");                // hot (dark step)
    }

    [Fact]
    public void lean_color_is_cold_for_negative_hot_for_positive_neutral_at_zero()
    {
        TrendSvgRenderer.LeanColor(-3m, 3m).Should().Be(TrendSvgRenderer.ColdHex);
        TrendSvgRenderer.LeanColor(3m, 3m).Should().Be(TrendSvgRenderer.HotHex);
        // Zero interpolates to the neutral midpoint, not hot or cold.
        var mid = TrendSvgRenderer.LeanColor(0m, 3m);
        mid.Should().NotBe(TrendSvgRenderer.HotHex).And.NotBe(TrendSvgRenderer.ColdHex);
    }
}
```

- [ ] **Step 2: Run to verify it fails**

Run: `... dotnet test --filter 'Name~TrendSvgRendererTests'`. Expected: FAIL — type does not exist.

- [ ] **Step 3: Implement the interface, `StyleBlock`, `LeanColor` + the hex constants**

- Public consts `HotHex = "#e34948"`, `ColdHex = "#2a78d6"`, `NeutralHex = "#f0efec"`.
- `StyleBlock` = a `<style>` string declaring `:root { --trend-hot:#e34948; --trend-cold:#2a78d6; --trend-neutral:#f0efec; --trend-tilt:#7a5ec4; --trend-surface:#ffffff; }` and `@media (prefers-color-scheme: dark) { :root { --trend-hot:#e66767; --trend-cold:#3987e5; --trend-neutral:#383835; --trend-surface:#1a1a19; } }`.
- `LeanColor(value, cap)`: clamp `t = clamp(value / cap, -1, 1)`; for `t <= 0` interpolate NeutralHex→ColdHex by `-t`; for `t >= 0` NeutralHex→HotHex by `t`; return `#rrggbb`. Endpoints return exactly `ColdHex`/`HotHex` at `±cap` (test pins this).

- [ ] **Step 4: Run to verify it passes** — same filter. Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add src/Services/TrendSvgRenderer.cs tests/SentinelCollector.UnitTests/Services/TrendSvgRendererTests.cs
git commit -m "feat(sentinel): TrendSvgRenderer color model + shared StyleBlock (Stage A)"
```

**Out of scope for A3:** no fragment builders yet (A4/A5). No dependency on `TrendQueryResult`.

---

## Task A4: Heatmap strip + KPI tile row

**Files:**
- Modify: `src/Services/TrendSvgRenderer.cs`
- Modify: `tests/SentinelCollector.UnitTests/Services/TrendSvgRendererTests.cs`

**Interfaces:**
- Consumes: `TrendQueryResult`, `LeanColor`, `HotHex`/`ColdHex`.
- Produces (consumed by B1, B2):
  - `string RenderHeatmapStrip(IReadOnlyList<DateOnly> days, IReadOnlyList<SectorLeanTrend> sectorLeans, IReadOnlyDictionary<string,string> sectorNames)` — one `<summary>` row per sector, cells colored by `LeanColor(netLean, 3m)`, rows sorted by LATEST non-null net lean descending (hot first), net lean value printed at row end. A sector whose `NetLeanByDay` is all-null renders a gray "no data" row.
  - `string RenderKpiTileRow(IReadOnlyList<DateOnly> days, IReadOnlyList<SectorLeanTrend> sectorLeans, IReadOnlyList<SectorRegimeTrend> regimes, IReadOnlyList<SignalDualTrend> signalTrends, IReadOnlyDictionary<string,string> sectorNames)` — three tiles: hottest sector (max latest net lean), coldest sector (min), top news mover (signal with the largest `|latest non-null tilt|`), each with its numeric value + one-word regime.

- [ ] **Step 1: Write the failing tests**

```csharp
[Fact]
public void heatmap_sorts_rows_by_latest_net_lean_hot_first()
{
    var days = TrendQueryService.BuildDayAxis(new(2026,6,1), new(2026,6,2));
    var leans = new[]
    {
        new SectorLeanTrend("FINANCIALS", new decimal?[] { -0.2m, -1.5m }), // coldest latest
        new SectorLeanTrend("ENERGY",     new decimal?[] {  0.1m,  2.0m }), // hottest latest
    };
    var names = new Dictionary<string,string> { ["ENERGY"]="Energy", ["FINANCIALS"]="Financials" };

    var svg = _r.RenderHeatmapStrip(days, leans, names);

    svg.IndexOf("Energy").Should().BeLessThan(svg.IndexOf("Financials")); // hot row first
}

[Fact]
public void heatmap_renders_a_no_data_row_when_a_sector_has_no_cells_in_window()
{
    var days = TrendQueryService.BuildDayAxis(new(2026,6,1), new(2026,6,2));
    var leans = new[] { new SectorLeanTrend("UTILITIES", new decimal?[] { null, null }) };
    var names = new Dictionary<string,string> { ["UTILITIES"]="Utilities" };

    var svg = _r.RenderHeatmapStrip(days, leans, names);

    svg.Should().Contain("Utilities").And.Contain("no data");
}

[Fact]
public void heatmap_prints_the_numeric_lean_beside_the_colored_cells()
{
    var days = TrendQueryService.BuildDayAxis(new(2026,6,1), new(2026,6,1));
    var leans = new[] { new SectorLeanTrend("ENERGY", new decimal?[] { 1.23m }) };
    var names = new Dictionary<string,string> { ["ENERGY"]="Energy" };

    _r.RenderHeatmapStrip(days, leans, names).Should().Contain("+1.23"); // color never alone
}

[Fact]
public void kpi_tiles_name_the_hottest_and_coldest_sector_by_latest_lean()
{
    var days = TrendQueryService.BuildDayAxis(new(2026,6,1), new(2026,6,1));
    var leans = new[]
    {
        new SectorLeanTrend("ENERGY",     new decimal?[] {  2.0m }),
        new SectorLeanTrend("FINANCIALS", new decimal?[] { -1.5m }),
    };
    var names = new Dictionary<string,string> { ["ENERGY"]="Energy", ["FINANCIALS"]="Financials" };

    var html = _r.RenderKpiTileRow(days, leans, Array.Empty<SectorRegimeTrend>(), Array.Empty<SignalDualTrend>(), names);

    html.Should().Contain("Energy").And.Contain("Financials").And.Contain("+2.00").And.Contain("-1.50");
}
```

- [ ] **Step 2: Run to verify it fails** — Expected: FAIL (methods missing).

- [ ] **Step 3: Implement `RenderHeatmapStrip` + `RenderKpiTileRow`**

- Latest net lean helper: `static decimal? Latest(IReadOnlyList<decimal?> series) => series.LastOrDefault(v => v != null);`
- Sort sectors by `Latest(...)` descending, nulls last; each row = `<details><summary>` with `days.Count` `<rect>`/`<td>` cells filled `LeanColor(v ?? 0, 3m)` (null cell → neutral/gray class), plus the sector display name and the latest lean formatted `+0.00;-0.00;0.00`. All-null series → a row containing the literal `no data`.
- KPI: hottest = max `Latest`, coldest = min `Latest`; top mover = signal maximizing `|Latest(TiltByDay-as-decimal)|`; look up the sector's latest regime word from `regimes` (`RegimeByDay` last non-null), default `n/a`. Every tile prints its numeric value.

- [ ] **Step 4: Run to verify it passes** — Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add src/Services/TrendSvgRenderer.cs tests/SentinelCollector.UnitTests/Services/TrendSvgRendererTests.cs
git commit -m "feat(sentinel): TrendSvgRenderer heatmap strip + KPI tile row (Stage A)"
```

**Out of scope for A4:** sparkline / regime strip / dual-trend (A5); no `<style>` duplication (reuse A3 `StyleBlock`).

---

## Task A5: Lean sparkline + regime strip + dual-trend chart

**Files:**
- Modify: `src/Services/TrendSvgRenderer.cs`
- Modify: `tests/SentinelCollector.UnitTests/Services/TrendSvgRendererTests.cs`

**Interfaces:**
- Consumes: `TrendQueryResult` slices, `LeanColor`.
- Produces (consumed by B1, B2):
  - `string RenderLeanSparkline(IReadOnlyList<decimal?> netLeanByDay)` — diverging fill above/below a zero baseline.
  - `string RenderRegimeStrip(IReadOnlyList<RegimeDay?> regimeByDay)` — score-colored cells, label `"{regime} · day {N}"` where N = count of consecutive most-recent days whose forward-filled regime equals the latest regime.
  - `string RenderDualTrendChart(string signalDisplayName, IReadOnlyList<decimal?> cellValueByDay, IReadOnlyList<TiltDay?> tiltByDay)` — cell polyline (black, 1.8px) vs tilt polyline (violet `#7a5ec4`, 1.4px); the tilt polyline + its legend entry are OMITTED entirely when every `tiltByDay` entry is null; article count shown beside the tilt value.

- [ ] **Step 1: Write the failing tests**

```csharp
[Fact]
public void regime_streak_label_counts_only_consecutive_days_of_the_current_regime()
{
    // expansion, expansion, contraction, contraction, contraction → "contraction · day 3"
    var series = new RegimeDay?[]
    {
        new("expansion", 1.0m), new("expansion", 1.1m),
        new("contraction", -1.0m), new("contraction", -1.2m), new("contraction", -1.3m),
    };

    _r.RenderRegimeStrip(series).Should().Contain("contraction · day 3");
}

[Fact]
public void dual_trend_omits_the_tilt_polyline_and_legend_when_no_tilt_rows_exist()
{
    var cells = new decimal?[] { 0.4m, 0.5m, 0.6m };
    var noTilt = new TiltDay?[] { null, null, null };

    var svg = _r.RenderDualTrendChart("Oil Price", cells, noTilt);

    svg.Should().NotContain("#7a5ec4"); // violet tilt series absent
    svg.Should().Contain("Oil Price");  // cell line still drawn
}

[Fact]
public void dual_trend_draws_the_violet_tilt_series_with_article_count_when_tilt_exists()
{
    var cells = new decimal?[] { 0.4m, 0.5m };
    var tilt = new TiltDay?[] { new TiltDay(0.3, 7), null };

    var svg = _r.RenderDualTrendChart("Oil Price", cells, tilt);

    svg.Should().Contain("#7a5ec4").And.Contain("7"); // violet series + article count beside tilt
}

[Fact]
public void lean_sparkline_fills_above_baseline_hot_and_below_baseline_cold_in_day_order()
{
    // Lean flips sign mid-window: negative then positive → cold cell precedes hot cell.
    var series = new decimal?[] { -1.0m, 1.0m };

    var svg = _r.RenderLeanSparkline(series);

    svg.IndexOf(TrendSvgRenderer.ColdHex).Should().BeLessThan(svg.IndexOf(TrendSvgRenderer.HotHex));
}
```

- [ ] **Step 2: Run to verify it fails** — Expected: FAIL.

- [ ] **Step 3: Implement the three fragment builders**

- Sparkline: map each day to an `x`; bar/area from baseline to the value, `fill=LeanColor(v,3m)`; skip null days (gap). Emit cold-then-hot in day order so the flip test passes.
- Regime strip: color each cell by `LeanColor(score, ?)` or a regime→hue map; streak = walk from the last non-null regime backward while `regime == latest.Regime`; label `"{regime} · day {N}"`.
- Dual-trend: build the cell polyline points (skip null); if `tiltByDay.All(t => t is null)` → render cell line + legend for cell only (NO violet, NO tilt legend); else render both, with the tilt legend entry showing the latest tilt value and its `RowCount` (article count).

- [ ] **Step 4: Run to verify it passes** — Expected: PASS (all A5 tests + the earlier suites).

- [ ] **Step 5: Commit**

```bash
git add src/Services/TrendSvgRenderer.cs tests/SentinelCollector.UnitTests/Services/TrendSvgRendererTests.cs
git commit -m "feat(sentinel): lean sparkline + regime strip + dual-trend chart (Stage A)"
```

**Out of scope for A5:** no page assembly, no DI. Fragment order/layout on a page is B1/B2's job.

---

## Task A6: DI registration for the two new services

**Files:**
- Modify: `src/DependencyInjection.cs` (add to `AddApplication`, beside the matrix-sector registrations ~lines 892-897)

**Interfaces:**
- Consumes: `ITrendQueryService`, `ITrendSvgRenderer`.
- Produces: resolvable services for B1/B2.

- [ ] **Step 1: Add the registrations**

```csharp
// Matrix trend visualization (spec 2026-07-04). Query service reads the
// ThresholdEngine-owned matrix_cells / sector_regimes and the sentinel
// macro_observations rows raw via the scoped SentinelDbContext connection, so
// it is scoped; the SVG renderer is pure (no I/O), so it is a singleton.
services.AddScoped<ITrendQueryService, TrendQueryService>();
services.AddSingleton<ITrendSvgRenderer, TrendSvgRenderer>();
```

- [ ] **Step 2: Verify it compiles + the host boots**

Run: `bash .devcontainer/compile.sh --no-test` then a boot smoke (host `/health/ready` returns healthy under the dev container). Expected: clean build, no DI resolution error at startup (both services construct).

- [ ] **Step 3: Commit**

```bash
git add src/DependencyInjection.cs
git commit -m "feat(sentinel): register TrendQueryService + TrendSvgRenderer in DI (Stage A)"
```

**Out of scope for A6:** no consumer wiring — nothing resolves these yet. That is Stage B.

**Stage A PR gate:** `bash .devcontainer/compile.sh` green (unit + integration), 0 warnings. PR body notes: additive only, no live surface changed.

---

# STAGE B — Integrate: market-first digest + `/sentinel/matrix` endpoint (own PR)

Depends on Stage A merged. Both new visuals render ALONGSIDE the old digest sections (no deletions yet) so the diff is reviewable and the old view remains a fallback.

## Task B1: Restructure DigestRenderer to market-first + embed trend blocks

**Files:**
- Modify: `src/Configuration/DigestOptions.cs` (add `TrendWindowDays`)
- Modify: `src/Services/DigestRenderer.cs` (signature + market-first order + embed blocks + empty-matrix notice)
- Modify: `src/Services/DigestService.cs` (fetch trends, pass to `Render`)
- Modify: `tests/SentinelCollector.UnitTests/Services/DigestRendererTests.cs`
- Modify: `tests/SentinelCollector.UnitTests/Services/DigestServiceTests.cs` (constructor gains a dep — update the test's service construction)

**Interfaces:**
- Consumes: `ITrendQueryService`, `ITrendSvgRenderer`, `TrendQueryResult`, `IAtlasSectorDisplayResolver`.
- Produces: `IDigestRenderer.Render(DigestPeriodType periodType, DigestStats stats, TrendQueryResult trends, string narrativeMarkdown, DigestLlmStatus llmStatus)` — new `trends` parameter inserted before `narrativeMarkdown`.

- [ ] **Step 1: Add `TrendWindowDays` config (failing validator test first)**

In `DigestOptionsValidatorTests` add a case asserting `TrendWindowDays <= 0` fails; run → FAIL. Then in `SectorMatrixOptions` add `public int TrendWindowDays { get; set; } = 33;` and in `DigestOptionsValidator.Validate` add `if (sm.TrendWindowDays <= 0) errors.Add(...)`. Run → PASS.

Commit: `feat(sentinel): add SectorMatrix.TrendWindowDays (default 33) (Stage B)`

- [ ] **Step 2: Write the failing renderer tests (market-first order + empty-matrix notice)**

```csharp
[Fact]
public void market_first_puts_kpi_and_heatmap_before_the_narrative()
{
    var stats = BuildStats();
    var trends = BuildTrends(); // helper: non-empty TrendQueryResult
    var result = _renderer.Render(DigestPeriodType.Daily, stats, trends, "## Executive Take\nbody", DigestLlmStatus.Ok);

    var kpiPos = result.Html.IndexOf("kpi", StringComparison.OrdinalIgnoreCase);
    var narrativePos = result.Html.IndexOf("Executive Take", StringComparison.Ordinal);
    kpiPos.Should().BeGreaterThan(0);
    kpiPos.Should().BeLessThan(narrativePos);
}

[Fact]
public void empty_matrix_window_renders_the_explicit_notice_naming_the_projector_mode_config()
{
    var stats = BuildStats();
    var emptyTrends = new TrendQueryResult(
        TrendQueryService.BuildDayAxis(new(2026,6,1), new(2026,6,2)),
        Array.Empty<SectorLeanTrend>(), Array.Empty<SectorRegimeTrend>(), Array.Empty<SignalDualTrend>());

    var result = _renderer.Render(DigestPeriodType.Daily, stats, emptyTrends, string.Empty, DigestLlmStatus.Ok);

    result.Html.Should().Contain("no matrix data in window")
        .And.Contain("Matrix:ObservationProjectorMode"); // never a blank 500
}
```

Run → FAIL (signature mismatch / behavior missing).

- [ ] **Step 3: Implement the renderer change**

- Change `IDigestRenderer.Render` + `DigestRenderer.Render` + `BuildMarkdown` to take `TrendQueryResult trends`.
- New `BuildMarkdown` order (spec §"Daily digest — page order"): header + window stats → `_svg.StyleBlock` (once) → KPI tile row → heatmap strip (each row a `<summary>` wrapping the sector's drill-down `<details>` body: lean sparkline + regime strip + top-5 dual-trend charts) → narrative ("Executive Take") → momentum ("Cross-sector signals") → cross-collector ("Noteworthy") → keep word cloud / certainty / publishers at the tail.
- Inject `ITrendSvgRenderer _svg` + `IAtlasSectorDisplayResolver _sectorDisplay` into `DigestRenderer` (it becomes non-static-only; it is a singleton, both deps are singletons — safe). Build the `sectorNames` map from `AtlasSectorCodes.All` + `_sectorDisplay.TryGetDisplayName`.
- Empty-matrix guard: when `trends.SectorLeans` is empty (or all sectors all-null), emit the literal notice `"no matrix data in window"` naming `Matrix:ObservationProjectorMode` instead of an empty heatmap. Never throw.
- The old `AppendSectorHeatSection` / `AppendPerSectorBlocks` STAY (both views render this stage) — do not delete here.

- [ ] **Step 4: Wire DigestService to fetch trends**

- Inject `ITrendQueryService _trendQuery` into `DigestService`.
- Add `BuildTrendsAsync(periodType, end, activity, ct)` mirroring `BuildSectorMatrixAsync`'s resilience: `trendStart = end - TimeSpan.FromDays(_options.SectorMatrix.TrendWindowDays)`; on ANY DB failure log Warning + span event and return an EMPTY `TrendQueryResult` (day axis only) so the digest still ships (the renderer shows the notice).
- Call it after `BuildSectorMatrixAsync`; pass the result into `_renderer.Render(periodType, stats, trends, narrative.Markdown, narrative.LlmStatus)`.

- [ ] **Step 5: Run to verify it passes**

Run: `bash .devcontainer/compile.sh`. Expected: unit + integration PASS. Update `DigestServiceTests` / `DigestRendererTests` construction sites for the new deps/signature until green.

- [ ] **Step 6: Commit**

```bash
git add src/Configuration/DigestOptions.cs src/Services/DigestRenderer.cs src/Services/DigestService.cs tests/SentinelCollector.UnitTests/Services/DigestRendererTests.cs tests/SentinelCollector.UnitTests/Services/DigestServiceTests.cs tests/SentinelCollector.UnitTests/Services/DigestOptionsValidatorTests.cs
git commit -m "feat(sentinel): market-first digest with embedded trend SVG blocks (Stage B)"
```

**Out of scope for B1:** no deletion of old sections (Stage C). No change to the `.md` digest variant (charts are HTML-only — the markdown body still carries the old text sections this stage; Stage C prunes them).

---

## Task B2: New `GET /sentinel/matrix?range={7d|30d|90d|max}` endpoint

**Files:**
- Create: `src/Endpoints/MatrixEndpoints.cs`
- Modify: `src/Program.cs` (`app.MapMatrixEndpoints();`)
- Create: `tests/SentinelCollector.IntegrationTests/Services/MatrixEndpointTests.cs` (or a WebApplicationFactory-based endpoint test if that harness already exists; otherwise a thin handler unit test over the range parser)

**Interfaces:**
- Consumes: `ITrendQueryService` (scoped), `ITrendSvgRenderer` (singleton), `IAtlasSectorDisplayResolver`.
- Produces: an HTML page (KPI + heatmap + drill-downs, NO narrative), reusing the same `TrendSvgRenderer` blocks and `StyleBlock`.

- [ ] **Step 1: Write the failing range-parser test**

```csharp
[Theory]
[InlineData("7d", 7)]
[InlineData("30d", 30)]
[InlineData("90d", 90)]
[InlineData(null, 30)]      // absent → default 30d
[InlineData("garbage", 30)] // malformed → default 30d, no error
public void range_param_maps_to_window_days_defaulting_to_30(string? range, int expectedDays)
{
    MatrixEndpoints.ResolveWindowDays(range).Should().Be(expectedDays);
}

[Fact]
public void max_range_maps_to_the_matrix_hypertable_floor()
{
    // matrix_cells hypertable since 2025-12-31 (spec). "max" resolves to that floor.
    MatrixEndpoints.ResolveWindowStart("max", asOf: new(2026,7,4,0,0,0,TimeSpan.Zero))
        .Should().Be(new DateTimeOffset(2025,12,31,0,0,0,TimeSpan.Zero));
}
```

Run → FAIL.

- [ ] **Step 2: Implement `MatrixEndpoints`**

- `internal static int ResolveWindowDays(string? range)` → `7|30|90` from `"7d"|"30d"|"90d"`, else 30. `"max"` handled by `ResolveWindowStart`.
- `internal static DateTimeOffset ResolveWindowStart(string? range, DateTimeOffset asOf)` → `"max"` → `2025-12-31Z`; else `asOf - ResolveWindowDays(range) days`.
- `MapMatrixEndpoints`: `app.MapGet("/sentinel/matrix", async (string? range, ITrendQueryService q, ITrendSvgRenderer svg, IAtlasSectorDisplayResolver disp, CancellationToken ct) => { ... })`:
  1. `asOf = DateTimeOffset.UtcNow`; `start = ResolveWindowStart(range, asOf)`.
  2. `await disp.EnsureLoadedAsync(ct)`; build `sectorNames`.
  3. `var trends = await q.GetTrendsAsync(start, asOf, ct);`
  4. Compose the page: an HTML shell (a `WrapMatrixHtml` mirroring `DigestEndpoints.WrapHtml`'s inline-style shell) including `svg.StyleBlock`, `svg.RenderKpiTileRow(...)`, `svg.RenderHeatmapStrip(...)` with per-sector `<details>` drill-downs, range switch links (`?range=7d|30d|90d|max` — plain `<a>` links, page reload, zero JS). NO narrative.
  5. Empty-matrix: when `trends.SectorLeans` is empty, render the same `"no matrix data in window" / Matrix:ObservationProjectorMode` notice, HTTP 200 (never blank 500).
  6. Return `Results.Content(html, "text/html; charset=utf-8")`.
- In `Program.cs`, add `app.MapMatrixEndpoints();` immediately after `app.MapDigestEndpoints();` (line ~525).

- [ ] **Step 3: Run to verify it passes** — parser tests PASS; `compile.sh` green; manual smoke `curl -s localhost:5091/sentinel/matrix?range=30d | head` returns the KPI+heatmap HTML.

- [ ] **Step 4: Commit**

```bash
git add src/Endpoints/MatrixEndpoints.cs src/Program.cs tests/SentinelCollector.IntegrationTests/Services/MatrixEndpointTests.cs
git commit -m "feat(sentinel): live /sentinel/matrix trend explorer endpoint (Stage B)"
```

**Out of scope for B2:** no JSON API, no client JS, no CDN, no compose/port change (reuses the existing :5091 host mapping). No narrative on this page.

**Stage B PR gate:** `bash .devcontainer/compile.sh` green; smoke both surfaces (`POST /sentinel/digests/generate?period=Daily&force=true` then GET the HTML; `GET /sentinel/matrix?range=30d`). Deploy + verify before starting Stage C.

---

# STAGE C — Cutover: remove superseded sections (own PR, only after Stage B verified)

Depends on Stage B deployed + smoke-verified. This deletes the sections the trend blocks replaced.

## Task C1: Remove old Sector Heat table + Sector Detail lists + raw extracted-quantity bullets

**Files:**
- Modify: `src/Services/DigestRenderer.cs`
- Modify: `tests/SentinelCollector.UnitTests/Services/DigestRendererTests.cs`

**Interfaces:**
- Consumes: nothing new.
- Produces: a slimmer `BuildMarkdown` (no old Sector Heat / Sector Detail).

- [ ] **Step 1: Flip the failing tests to assert ABSENCE**

Update `DigestRendererTests`: any test asserting the old `## Sector Heat` markdown table or the `## Sector Detail` blocks (including the `700000000000.0000 USD` raw-quantity bullet) now asserts those strings are ABSENT; add a test that the drill-down heatmap (Stage B) is still present. Run → FAIL (old sections still render).

- [ ] **Step 2: Delete the superseded methods + their calls**

- Remove the `AppendSectorHeatSection(sb, stats);` and `AppendPerSectorBlocks(sb, stats);` calls from `BuildMarkdown`.
- Delete the now-dead private methods: `AppendSectorHeatSection`, `AppendPerSectorBlocks`, `AppendSectorSignalLine`, `SignalCell`, and `AppendCitedBullet` (grep-verified: `AppendCitedBullet` is called ONLY inside `AppendPerSectorBlocks` at lines 414/426; `AppendSectorSignalLine`/`SignalCell` only inside those two sections). Remove any now-unused `using`/helpers left dangling.
- Keep the word cloud / certainty / publisher sections (not superseded).

- [ ] **Step 3: Run to verify it passes** — `bash .devcontainer/compile.sh` green; the old strings are gone, the heatmap remains.

- [ ] **Step 4: Commit**

```bash
git add src/Services/DigestRenderer.cs tests/SentinelCollector.UnitTests/Services/DigestRendererTests.cs
git commit -m "refactor(sentinel): drop old Sector Heat table + Sector Detail lists (Stage C)"
```

**Out of scope for C1:** the narrative "Sector Watch" section lives in the LLM prompt, not the renderer — that is C2.

---

## Task C2: Remove the narrative "Sector Watch" section

**Files:**
- Modify: `src/prompts/digest/narrative.md` (remove the `## Sector Watch` section instruction, line ~36)
- Modify: `tests/SentinelCollector.UnitTests/Services/DigestNarrativeGeneratorTests.cs` (the `Contain("## Sector Watch")` assertion, ~line 525)
- Possibly modify: `src/Services/DigestAggregateSerializer.cs` (a `// Sector Watch` comment at ~line 74 marks the sector list fed to the LLM — verify whether the section header is emitted here or purely in the prompt)

**Interfaces:** none (prompt + test only).

- [ ] **Step 1: Flip the failing narrative test**

Change the `DigestNarrativeGeneratorTests` assertion (~line 525) from `Contain("## Sector Watch")` to `NotContain("## Sector Watch")` (the redesign drops it — its prose duplicated the Executive Take). Run → FAIL.

- [ ] **Step 2: Edit the narrative prompt**

Remove the `## Sector Watch` section block from `src/prompts/digest/narrative.md`. Verify `DigestAggregateSerializer` does not itself hard-code the header (the comment at ~line 74 describes the sector aggregate, not necessarily the header); if it emits a `Sector Watch` label, remove it too.

- [ ] **Step 3: Run to verify it passes** — `bash .devcontainer/compile.sh` green.

- [ ] **Step 4: Commit**

```bash
git add src/prompts/digest/narrative.md tests/SentinelCollector.UnitTests/Services/DigestNarrativeGeneratorTests.cs src/Services/DigestAggregateSerializer.cs
git commit -m "refactor(sentinel): drop narrative Sector Watch section (Stage C)"
```

**Out of scope for C2:** no other prompt change (Non-goals: "No change to ... LLM narrative prompts ... except sections deleted above"). **DEPLOY NOTE:** prompts are host-mounted at `/prompts` (project CLAUDE.md DATA_ML_CONTEXT + memory `project_sentinel...`). The repo file is the source of truth, but the deploy must refresh the mount (`ansible ... --tags patterns` or the prompts mount sync) — a container-only or repo-only change will not take effect. Flag to the deployer.

**Stage C PR gate:** `bash .devcontainer/compile.sh` green; smoke a freshly generated digest confirms the old sections are gone and the trend blocks + Executive Take remain; confirm the `/prompts` mount is refreshed post-deploy so the narrative actually drops Sector Watch.

---

## Self-Review (against the spec)

- **Spec Components → tasks:** `TrendQueryService` (A1+A2), `TrendSvgRenderer` (A3-A5), `DigestRenderer` restructure (B1), `/sentinel/matrix` (B2). ✓
- **Spec Surfaces → tasks:** redesigned digest page order (B1), live explorer range switch (B2). ✓
- **Spec Error handling → tasks:** no-data sector row (A4), regime-from-first-day/no-backfill (A1 forward-fill + A2), tilt-omission for hard-data-only signals (A5), empty-matrix notice naming the projector mode (B1 + B2), malformed/absent range → 30d (B2). ✓
- **Spec Testing list → tasks:** sign-flip blue→red in day order (A5 sparkline + A4 heatmap), regime streak consecutive-only (A5), dual-trend tilt omission (A5), heatmap sort hot-first (A4), forward-fill across a gap (A1), tilt excludes `|v|>1.5` + null-signal (A2), net-lean latest-per-pattern-per-day not average (A2), empty-matrix notice (B1). ✓
- **Spec Non-goals honored:** no JSON API / client JS / CDN / Grafana / new services / compose / write-path change — enforced per-task out-of-scope notes + Global Constraints. ✓
- **Card invariants carried verbatim:** `:sig:` change-all-or-none + the momentum-filter-not-`:sig:` distinction (Global Constraints + A2), projector Mode=Off notice (Global Constraints + B1/B2), `obs:%` legacy exclusion (A2). ✓
- **Type consistency:** `TrendQueryResult` / `SectorLeanTrend` / `SectorRegimeTrend` / `RegimeDay` / `SignalDualTrend` / `TiltDay` / `ITrendQueryService.GetTrendsAsync` / `ITrendSvgRenderer` method names are used identically across A1→B2. ✓
