# Epic 3 — `macro_observations` Substrate

## Goal

Stand up `macro_observations` as the substrate for the matrix. Bridge
between Sentinel (write) and TE (read), and direct-query substrate for
all three consumption surfaces per handoff §4.

## Branch

`epic/3-macro-observations` off `main` (forked when Phase 1 merges).

## Phase + dependencies

- Phase: **2** (parallel with Epics 2 and 4-Feature-4.6)
- Blocked by: Epic 1 (sector tags + signal identities are FK targets;
  rollup version FK).
- Blocks: Epic 4 (Sentinel writes here), Epic 5 (TE reads through here),
  Epic 6 (direct-substrate panels read here).

## Canonical AC

`/home/james/ATLAS/docs/atlas-matrix-mvp-plan.md` Epic 3 (lines 366–415).

## Open architectural choice — owning project

The substrate is shared. It can live in:
- **SecMaster** (catalog owns the schema; collectors write via API). +
  consistent with "SecMaster classifies".
- **A new `MacroSubstrate` project** (a thin EF Core data project +
  migrations + repos shared via NuGet/project ref). + cleanest
  separation; - more setup.
- **A shared `Atlas.Data` library** (each collector writes directly to
  the same TimescaleDB hypertable using a shared model). + simplest
  deploy; - no central API gate.

**Decision needed before Story 3.1.1.** I'll dispatch a Plan agent to
recommend, then put it to the user via NTFY.

## Features (with execution notes)

### 3.1 — Schema and versioning
- **3.1.1** DDL for `macro_observations`. TimescaleDB hypertable,
  time-partitioned. Schema accommodates numeric **and** qualitative
  observations without forcing irrelevant fields on either type
  (D4: developer call, documented in commit). Provenance fields:
  source collector, source identifier, extraction job id,
  ingestion timestamp. Sector tag nullable (FK to rollup version
  captured at write time). Trust attribute non-nullable for
  qualitative; absent or constant for numeric.
- **3.1.2** Indexes for primary query patterns. Indexes covering
  signal-identity + time, sector + time, source + time. EXPLAIN output
  in commit message.
- **3.1.3** Versioned-mapping query support. Given a historical date,
  query resolves under that date's SecMaster rollup version
  (consumes Feature 1.3).

### 3.2 — Collector write paths
- **3.2.1** FRED vertical slice. One named macro series end-to-end into
  `macro_observations`. Idempotent on
  `(source, source_id, observation_time)`.
- **3.2.2** FRED full coverage. All in-scope FRED macro series.
  Configuration documents macro vs. instrument-attributed series.
- **3.2.3** OFR write path. Mirrors FRED.
- **3.2.4** Sentinel qualitative write path. Trust attribute populated
  by extraction job (Epic 4). Sector tag present where source is
  sector-attributed.

## Files to touch

Depends on owning-project decision. Likely candidates:
- `SecMaster/src/Data/Entities/MacroObservationEntity.cs` **(new)** OR
  new project `MacroSubstrate/`.
- New EF migration creating the hypertable + Timescale conversion
  (raw SQL under `Data/Migrations/Sql/` or `select create_hypertable`
  inside the migration).
- `FredCollector/src/Services/DataCollectionService.cs:172-173`
  (write path adds dual-write to `macro_observations`).
- `OfrCollector/src/Data/Repositories/HfmRepository.cs`,
  `OfrCollector/src/Data/Repositories/StfmRepository.cs` (dual-write).
- `SentinelCollector/src/Data/Repositories/ObservationRepository.cs`
  (qualitative observations route to `macro_observations`).
- New repository / API surface for direct-query consumers.

## Subagent dispatch notes

- **Pre-story:** dispatch `Plan` agent to recommend owning project
  (SecMaster vs. new project vs. shared lib). NTFY user with
  recommendation.
- Story 3.1.1: `general-purpose` agent — schema design + migration.
  TimescaleDB hypertable creation needs raw SQL inside an EF migration
  (use the existing pattern from `instrument_embeddings` migration).
- Story 3.2.x: one `general-purpose` agent per collector. Idempotency
  test required.
- PR review: `pr-review-toolkit:review-pr` per story PR.

## Verification (epic-exit AC)

- A FRED macro series, an OFR HFM series, and a Sentinel qualitative
  observation all land in `macro_observations` with the right
  provenance, sector tag (where applicable), trust attribute
  (qualitative only), and `rollup_version` FK.
- Idempotency holds: re-running the same FRED collection cycle does
  not double-write.
- Versioned-mapping query: a synthetic v1.1 mapping change doesn't
  rewrite past observations; historical queries resolve under v1.0.
- Indexes from 3.1.2 are on the table; EXPLAIN confirms they're used
  for the dashboard's expected queries.
- Build + deploy clean; Loki shows no errors post-deploy.
