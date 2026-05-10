# Epic 5 — Matrix Computation, Storage, and Vector Queries

## Goal

Persist the matrix produced by TE. Make cell, sector, and signal vectors
queryable. Compute per-sector regimes (D9). Emit the events Sentinel
subscribes to (D12).

## Branch

`epic/5-matrix-storage` off `main` (forked in Phase 3 once Epic 3 is
merged + Epic 2 is emitting cells).

## Phase + dependencies

- Phase: **3**
- Blocked by: Epic 2 (cell projection emits per-pattern × per-sector
  events) + Epic 3 (`macro_observations` substrate provides input
  context for cell provenance) + Epic 1 (rollup_version FK).
- Blocks: Epic 6 (consumption surfaces query the matrix here).

## Canonical AC

`/home/james/ATLAS/docs/atlas-matrix-mvp-plan.md` Epic 5 (lines 533–624).

## Features (with execution notes)

### 5.1 — Cell persistence
- **5.1.1** `matrix_cells` schema and write path. TimescaleDB
  hypertable: `(pattern_id, sector, evaluation_time, cell_value,
  rollup_version, contributing_observation_refs)`. Subscriber
  consumes TE's cell-emit stream (Epic 2 Story 2.2.2). Idempotent on
  `(pattern_id, sector, evaluation_time)`.
- **5.1.2** Cell time-series query path. `(pattern, sector)` → time
  series. Honors versioned mapping. Indexes support dashboard
  time-window panels.

### 5.2 — Sector / signal vector queries
- **5.2.1** Sector vector query. `(sector, time window)` → matrix of
  (pattern × time) cells. Sector-clustering diagnostic.
- **5.2.2** Signal vector query. `(pattern, time window)` → matrix of
  (sector × time) cells. Signal-redundancy diagnostic.

### 5.3 — Cross-correlation tensor
- **5.3.1** Pairwise cross-correlation API.
  `(sector_A, sector_B, lag_range, time_window)` → correlation
  series across the lag range. Reads cell time series; no separate
  storage. Performance target: single pair < 2s.
- **5.3.2** All-pairs lead-lag scan. Full sector × sector × lag tensor
  for a configured time window. Cacheable; cache invalidates on new
  cell writes within the window. Used by monthly trajectory reports.

### 5.4 — Per-sector regime classification (D9, D14)
- **5.4.1** Per-sector regime computation. Each sector classified
  from its sector score per evaluation cycle. Regime taxonomy
  configurable. Persisted as time series alongside `matrix_cells`.
  11 regime time series, one per sector.
- **5.4.2** Legacy global 6-state regime deleted. Already deleted in
  Epic 2 Feature 2.6 (`MacroRegime` + `RegimeTransitionDetector`);
  this story is bookkeeping — confirm no residual references.

### 5.5 — TE event emission for validation triggers (D12)
- **5.5.1** Event contract definition. Schema: sector, score (real),
  threshold-crossed (real), direction, evaluation-time,
  contributing-pattern-ids. Threshold semantics: real-valued bound
  (e.g., score passing `> 1.5` fires).
- **5.5.2** Event emission in TE evaluation loop. Per-sector
  thresholds configurable, default documented, hot-reloadable.
  Reuses `IEventBus` + extends `ThresholdEventStreamService`.
- **5.5.3** Sector × phase reference matrix derived view. Feeds
  Epic 6 dashboard primary panel. Computed from cell time series +
  per-sector regime time series. Persisted; refreshed weekly
  (initial proposal). Phase taxonomy size = developer call (D17).

## Files to touch

- `ThresholdEngine/src/Data/Entities/MatrixCellEntity.cs` **(new)**
- `ThresholdEngine/src/Data/Entities/SectorRegimeEntity.cs` **(new)**
- `ThresholdEngine/src/Data/Entities/SectorPhaseDerivedView.cs` **(new)**
- `ThresholdEngine/src/Data/Migrations/` (new EF migrations: hypertable
  for `matrix_cells`, regime time series, derived view)
- `ThresholdEngine/src/Data/Repositories/MatrixCellRepository.cs` **(new)**
- `ThresholdEngine/src/Services/CrossCorrelationService.cs` **(new)**
- `ThresholdEngine/src/Services/SectorRegimeClassifier.cs` **(new)**
- `ThresholdEngine/src/Services/SectorPhaseDerivedViewBuilder.cs` **(new)**
- `ThresholdEngine/src/Workers/MatrixCellPersistenceWorker.cs` **(new)**
- `ThresholdEngine/src/Workers/ThresholdCrossEventEmitter.cs` **(new
  or extend existing)**
- `ThresholdEngine/src/Grpc/Services/ThresholdEventStreamService.cs`
  (extend with new event payloads)
- `ThresholdEngine/src/Endpoints/MatrixEndpoints.cs` **(new)**
- `Events/` protobuf updates (new event payload types)

## Subagent dispatch notes

- Story 5.1.x (persistence): one `general-purpose` agent with TDD.
  Idempotency test required.
- Story 5.3.x (cross-correlation): one `general-purpose` agent;
  performance target is the AC — agent must benchmark.
- Story 5.4.x (per-sector regime): can parallel with 5.3 once 5.1 lands.
- Story 5.5.x (event emission): coordinate carefully with Epic 4
  Feature 4.5 — they're the consumer.
- PR review: `pr-review-toolkit:review-pr` per story PR.

## Verification (epic-exit AC)

- `matrix_cells` hypertable populated from a TE evaluation; row count
  matches expected (#patterns × 11 sectors × evaluations).
- Cell time-series query returns continuous data for an
  (active-pattern, sector) pair.
- Cross-correlation single-pair query returns under 2s for the
  dashboard's typical time window.
- Per-sector regime time series visible for all 11 sectors.
- TE emits a `SectorThresholdCrossedEvent`; `ValidationEventConsumerWorker`
  receives it (verified via OTEL trace).
- Story 5.5.3 derived view populated; query for (sector, phase) returns
  the expected vector.
- Build + deploy clean; Loki silent post-deploy.
