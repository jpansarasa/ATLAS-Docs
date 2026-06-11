# backtest/SignalReplay — C# signal-replay (WS3 B2, Component 1)

Offline developer tool. **Not** a deployed service: absent from `compose.yaml` and
ansible. It replays each production pattern's `signalExpression` over historical
monthly eval-dates and emits the **signal-value dataset** that the Python harness
(Component 3) consumes. Spec §"Component 1" (retired to git history:
`git show 5cf265f7:docs/atlas-matrix-backtesting-spec.md`).

## What it does

For every pattern under `ThresholdEngine/config/patterns/**` that carries a
`signalExpression` — **including `enabled: false` patterns** (spec §"Bonus coverage") —
and for each month-end eval-date from the earliest `fred_observations` date through
today, it:

1. builds a `HistoricalPatternEvaluationContext` with `asOfDate = eval-date`,
2. runs the pattern's Roslyn-compiled `signalExpression` against it (the identical
   production code path), and
3. clamps to `[-3, +3]` and emits one row.

When a required series has no value as of the eval-date the row carries a null
`signal_value` and a `missing_data_reason` — the run is never aborted on data gaps.

**Reuse, not reimplement.** The signal math is the production
`RoslynExpressionCompiler` + `HistoricalPatternEvaluationContext` +
`SignalUtilities.ClampSignal`, reached by project-referencing
`ThresholdEngine.csproj`. Only the orchestration (eval-date walk, dataset emission,
a SELECT-only point-in-time `fred_observations` repository) is new.

## Output (`backtest/data/`, git-ignored)

`signal_values.csv`, one row per `(pattern_id, eval_date)`:

| column | meaning |
|---|---|
| `pattern_id` | the pattern file's `patternId` |
| `signal_identity` | `metadata.signalIdentity`, or `pattern_id` fallback |
| `eval_date` | as-of evaluation date (ISO-8601) |
| `signal_value` | replayed production signal in `[-3,+3]`; empty when data missing as-of |
| `missing_data_reason` | populated only when `signal_value` is empty |

## Run (devcontainer — has DB access + net10 SDK)

```bash
# from the ThresholdEngine devcontainer (DB_* env preset):
dotnet run --project backtest/SignalReplay -- \
    --config ThresholdEngine/config --out backtest/data/signal_values.csv
# optional: --start yyyy-MM-dd  --end yyyy-MM-dd  (default: earliest FRED date → today, month-end)
```

DB connection comes from `DB_HOST/DB_PORT/DB_NAME/DB_USER/DB_PASSWORD` (same env as
`ThresholdEngineDbContextFactory`). The tool only ever SELECTs from `fred_observations`.

## Tests

```bash
dotnet test backtest/SignalReplay.Tests
```

- **Synthetic (no DB):** eval-date walk, CSV emission, replay orchestration, point-in-time
  cutoff, missing-data skip-and-record.
- **Drift-check (the gate, DB-backed):** replays real FRED patterns as-of today and asserts
  each replayed signal equals the live ThresholdEngine's reported signal within float
  tolerance — proving replay == production (spec §Testing). Requires `DB_PASSWORD`; run in
  the devcontainer.

## Limitations (v1)

- **Latest-vintage `AsOf`.** For the pre-ATLAS-collection span the `fred_observations.AsOf`
  stamp is the ingestion time, not the true FRED release vintage, so point-in-time reads use
  revised values stamped as if known at ingestion. Flagged in the spec §Limitations; ALFRED
  vintage replay is the named hardening step.
- **Coverage is bounded by the DB.** This DB holds FRED data from ~2016 onward and lacks some
  required series (e.g. `GOLDAMGBD228NLBM`, OFR, Sentinel prefix series), so patterns needing
  those emit `no_data` rows for the affected dates. That is auditable coverage, not failure.
