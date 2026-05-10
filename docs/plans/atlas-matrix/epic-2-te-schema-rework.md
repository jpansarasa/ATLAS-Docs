# Epic 2 — ThresholdEngine Schema Rework

## Goal

Add sector projection to patterns. Remove `category` and the macro score.
Move alpha-factor math into TE. Introduce per-sector aggregation. Retire
the 6-state regime under D14 fail-forward. Preserve the engine, the DSL,
the weighting machinery, and the ±3 signal range.

## Branch

`epic/2-te-schema-rework` off `main` (forked when Phase 1 merges).

## Phase + dependencies

- Phase: **2**
- Blocked by: Epic 1 (sector taxonomy + signal identity catalog must be
  live; TE reads them).
- Blocks: Epic 5 (matrix storage consumes TE's emitted cells).

## Canonical AC

`/home/james/ATLAS/docs/atlas-matrix-mvp-plan.md` Epic 2 (lines 278–364).

## Features (with execution notes)

### 2.1 — Remove `category`
- **2.1.1** Schema migration removing `category`. Pattern schema +
  `pattern-schema.json` updated. Existing patterns updated; any pattern
  that relied on category-driven routing flagged for re-spec under
  sector projection. Migration is rollback-safe.
- **2.1.2** Aggregation code path that referenced category retired.
  Files identified by recon (Story 2.1 audit list):
  `MacroScoreCalculator.cs`, `PatternEndpoints.cs`,
  `PatternEvaluationResult.cs`, `PatternCategory.cs` (deleted),
  `ThresholdEvent.cs`, `ObservationEventSubscriber.cs:294,340`,
  `ThresholdEventConfiguration.cs:28-30,75-77`,
  `PatternConfigurationLoader.cs:244`. All 79 pattern config files in
  `ThresholdEngine/config/patterns/` lose their `category` field.

### 2.2 — Sector projection via weight vector
- **2.2.1** `sectorWeights` field. Pattern schema gains
  `sectorWeights: { ATLASSector: weight }`. D5 sparsity policy:
  unmentioned sectors store explicit zero, not null. Schema validation
  rejects multi-output forms.
- **2.2.2** Cell projection in evaluation pipeline. For each evaluated
  pattern, TE produces 11 cell values:
  `cell(pattern, sector) = signal × sectorWeights[sector] × freshness ×
   temporalMultiplier × confidence × weight`. Output ±3 per cell (D12).
  Emitted as events for Epic 5 persistence.

### 2.3 — Single output per pattern enforced
- **2.3.1** Schema validation rejects multi-output. Linter pass over the
  79 patterns surfaces violations.
- **2.3.2** Multi-output legacy patterns split or retired. Each
  decomposition documented per pattern in a dedicated commit message.

### 2.4 — Per-sector aggregation
- **2.4.1** Per-sector score computation. Existing weighted formula
  restricted to each sector's column. Real-valued ±3. Computed each
  evaluation cycle.
- **2.4.2** Sparsity handled honestly. Sectors with no contributing
  patterns produce a defined "no signal" state (zero with `coverage = 0`
  flag), distinguishable from zero-by-cancellation. Coverage queryable.

### 2.5 — Alpha-factor / observation normalization
- **2.5.1** Burst window configuration per signal. Signal-identity-keyed
  configuration sets the burst window (default 24h). Hot-reloadable
  consistent with existing TE patterns.
- **2.5.2** Dedup grouping in TE read adapter. When reading a signal,
  TE queries `signal_identity_dedup` API (Story 1.7.2) and collapses
  observations within burst window via `s = (o1 + … + on) / n`.
  Read-time dedup, not write-time.

### 2.6 — Macro score retirement (D8, D14 fail-forward)
- **2.6.1** Macro score deleted, not parallel-run. Files removed:
  `MacroScoreCalculator.cs`, `MacroScoreConfiguration` references,
  ScoreToRegime mapping. Downstream consumers (dashboards, reports, MCP)
  cut over to per-sector scores. **6-state regime classification
  (`MacroRegime` enum + `RegimeTransitionDetector`) deleted as part of
  this epic** — it depends on the macro score and is replaced by Epic 5
  Feature 5.4's per-sector regimes. No legacy parity validation per D14.

## Files to touch

- `ThresholdEngine/src/Entities/PatternConfiguration.cs` (add
  `sectorWeights`, remove `category`)
- `ThresholdEngine/src/Entities/PatternEvaluationResult.cs`
- `ThresholdEngine/src/Entities/ThresholdEvent.cs`
- `ThresholdEngine/src/Enums/PatternCategory.cs` **(delete)**
- `ThresholdEngine/src/Enums/MacroRegime.cs` **(delete)**
- `ThresholdEngine/src/Services/MacroScoreCalculator.cs` **(delete)**
- `ThresholdEngine/src/Services/RegimeTransitionDetector.cs` **(delete)**
- `ThresholdEngine/src/Services/PatternEvaluationService.cs:46,55,64,75,103,117,186,200,227`
- `ThresholdEngine/src/Endpoints/PatternEndpoints.cs:350-392`
- `ThresholdEngine/src/Workers/ObservationEventSubscriber.cs:294,340`
- `ThresholdEngine/src/Configuration/PatternConfigurationLoader.cs:244`
- `ThresholdEngine/src/Data/Configurations/ThresholdEventConfiguration.cs:28-30,75-77`
- `ThresholdEngine/src/Data/Repositories/ThresholdEventRepository.cs:36`
- `ThresholdEngine/src/Data/Migrations/` (new EF migration: drop category
  column + index, add sector_weights serialization if applicable)
- `ThresholdEngine/config/pattern-schema.json`
- `ThresholdEngine/config/patterns/**/*.json` (all 79 files —
  remove `category`, add `sectorWeights`; some patterns split per
  Story 2.3.2)
- `ThresholdEngine/src/Services/AlphaFactorService.cs` **(new)**
  consumes Story 1.7.2 dedup grouping API.

## Subagent dispatch notes

- Story 2.1.2 (delete category infra): one `general-purpose` agent.
  Mechanical, mostly compile-driven.
- Story 2.2.x (sectorWeights + projection): one `general-purpose` agent
  with TDD per `superpowers:test-driven-development`.
- Story 2.3.2 (decompose multi-output patterns): one
  `general-purpose` agent that walks the 79 patterns, identifies
  multi-output, proposes decomposition, and lands the rewrites. Each
  pattern gets its own commit.
- Story 2.6.1 (macro score + 6-state regime delete): one
  `general-purpose` agent — large blast radius; verify all callers
  updated before merging.
- PR review: `pr-review-toolkit:review-pr` per story PR.

## Verification (epic-exit AC)

- A pattern with a non-trivial `sectorWeights` produces 11 distinct cell
  values per evaluation; the values are real-valued ±3.
- Per-sector score returns from a new endpoint
  (`GET /api/scores/sectors`) for all 11 sectors with coverage flags.
- `MacroScoreCalculator` and `MacroRegime` are gone — compile failures
  pointed at every now-orphan caller; all updated.
- Burst-window dedup demo: feed 3 redundant CPI observations within 24h;
  TE produces a single normalized signal value `(o1+o2+o3)/3`.
- `bash ThresholdEngine/.devcontainer/compile.sh` clean.
- Deploy via ansible; smoke that no patterns emit `category` in any
  endpoint output.
