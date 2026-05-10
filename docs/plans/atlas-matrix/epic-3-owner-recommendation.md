# Epic 3 — `macro_observations` Owning-Project Recommendation

**Source:** Plan-agent recon dispatched 2026-05-10 (post-Epic-1 merge `a08b806`).
**Status:** DRAFT — pending user ratification.
**Decision blocks:** Story 3.1.1 (DDL).

## Recommendation

**Stand up a new top-level `MacroSubstrate` project that owns `macro_observations` inside the existing `atlas_data` database**, alongside the collectors and ThresholdEngine. Concretely, add a sibling project (`MacroSubstrate/src/`) with its own `MacroSubstrateDbContext` + EF migrations + entity + repository, plus a thin client library (`MacroSubstrate.Client`) referenced by FRED/OFR/Sentinel.

Three sentences of rationale:

1. **SecMaster is the wrong owner** because it lives in a separate Postgres database (`atlas_secmaster`); PostgreSQL cannot enforce cross-database FKs, and the substrate must physically reside in `atlas_data` where collectors write and where TE + Epic 6 surfaces will read it directly (handoff §4: "queried directly by all three [surfaces]").
2. **A pure shared-library option is simplest but offloads migration ownership** onto whichever collector happens to deploy first, creating a confused deployment story when three independent collectors all want to "run my own EF migrations" against the same shared table.
3. **A dedicated `MacroSubstrate` project keeps migration single-owner**, mirrors the established `Events` / `Events.EntityFrameworkCore` / `Events.Client` triad pattern just landed in Epic 1, and gives Epic 6 a natural home for the direct-query repository + (eventually) gRPC/MCP read surfaces without bolting them onto a collector.

## Comparison

| Dimension | Option 1: SecMaster | **Option 2: New MacroSubstrate (recommended)** | Option 3: Shared `Atlas.Data` lib |
|---|---|---|---|
| Schema owner | SecMaster (atlas_secmaster) | MacroSubstrate (atlas_data) | Shared model class; no single owner |
| Deploy unit | Add table to existing svc | New service (or migrate-only sidecar) | No new svc; first collector to deploy migrates |
| DB topology | **Cross-DB blocker** | Same DB as writers + readers ✓ | Same DB ✓ |
| FK reach to `mapping_versions` | Same-DB FK ✓ but writers can't FK across DBs | Captured as logical label (`mapping_version_label`, matches Epic 1 precedent) | Same — logical label |
| Write path change | gRPC/REST call to SecMaster API per observation | `ProjectReference` to `MacroSubstrate.Client`; inject `IMacroObservationWriter` | `ProjectReference` to shared lib; raw INSERT |
| Query path | API gate (extra hop, extra service to scale) — **violates handoff §4** | Direct SQL/EF against `macro_observations` from any service in atlas_data | Direct |
| Complexity to add | Medium — routes every obs through SecMaster's request budget | Medium — new csproj + DI module + ansible block | Low |
| Fit with handoff §5 ("SecMaster classifies, collectors gather, TE scores") | **Misfit** — turns SecMaster into write-through cache | Clean — substrate is neither classification nor scoring | Clean but ownership-diffuse |
| Single migration history | Yes (SecMaster's) | Yes (MacroSubstrate's) | **No** — three collectors each want to add columns |
| Convention fit | Adds non-classification table to classification svc | Mirrors `Events` triad pattern (lifted in Epic 1) | No precedent in repo |

**Decisive:** the cross-DB topology (`atlas_secmaster` vs `atlas_data`) eliminates Option 1; the dual-connection pattern SecMaster already uses for dedup (`SecMaster/src/Services/Dedup/FredObservationSourceProvider.cs`) confirms the boundary is real and load-bearing.

## First Migration Plan — Story 3.1.1 (DDL)

### Project structure (new)

```
MacroSubstrate/
├── README.md                                              # 10-section template (readme-consistency enforced)
├── src/
│   ├── MacroSubstrate/
│   │   ├── MacroSubstrate.csproj                          # net10.0; refs Events.Client + Events.EntityFrameworkCore + EFCore.Design + Npgsql.EFCore.PG
│   │   ├── Data/
│   │   │   ├── MacroSubstrateDbContext.cs                 # DbSet<MacroObservationEntity>
│   │   │   ├── MacroSubstrateDbContextFactory.cs          # IDesignTimeDbContextFactory
│   │   │   ├── Configurations/MacroObservationConfiguration.cs
│   │   │   ├── Migrations/                                # `dotnet ef migrations add InitialCreate`
│   │   │   └── Repositories/{IMacroObservationRepository,MacroObservationRepository}.cs
│   │   └── DependencyInjection.cs                         # AddMacroSubstrate(IConfiguration)
│   └── MacroSubstrate.Client/
│       ├── MacroSubstrate.Client.csproj                   # refs Events.Client; EF-FREE (writers can opt out of EF)
│       ├── MacroObservation.cs                            # plain record
│       └── IMacroObservationWriter.cs                     # writer abstraction
└── tests/MacroSubstrate.UnitTests/
```

### Schema (`macro_observations`)

Columns ratifying AC (mvp-plan lines 374–385):
- `id BIGSERIAL` — surrogate (NOT idempotency key)
- `observation_time TIMESTAMPTZ NOT NULL` — hypertable partition key
- `ingestion_time TIMESTAMPTZ NOT NULL DEFAULT NOW()`
- `source_collector TEXT NOT NULL` — `'fred' | 'ofr' | 'sentinel'` (text not enum, extensible)
- `source_id TEXT NOT NULL` — series id / mnemonic / extraction correlation id
- `signal_identity_id UUID NULL` — soft FK to `atlas_secmaster.signal_identities` (cross-DB; no enforced FK; matches Epic 1 collector precedent)
- `extraction_job_id UUID NULL` — Sentinel only
- `value_numeric NUMERIC(18,6) NULL` — numeric obs only
- `value_qualitative JSONB NULL` — qualitative obs only
- `trust REAL NULL` — non-null for qualitative, null for numeric
- `atlas_sector_code TEXT NULL` — `AtlasSectorCodeConverters.Nullable`
- `mapping_version_label TEXT NULL` — captures rollup version label at write-time
- `payload JSONB NULL` — escape hatch

D4 invariants enforced as raw-SQL CHECK constraints:

```sql
ALTER TABLE macro_observations ADD CONSTRAINT ck_macro_obs_value_xor
  CHECK ( (value_numeric IS NOT NULL) <> (value_qualitative IS NOT NULL) );
ALTER TABLE macro_observations ADD CONSTRAINT ck_macro_obs_trust_only_qualitative
  CHECK ( (value_qualitative IS NULL AND trust IS NULL)
       OR (value_qualitative IS NOT NULL AND trust IS NOT NULL) );
```

Idempotency (Story 3.2.1 AC line 400):

```sql
CREATE UNIQUE INDEX ux_macro_obs_idem ON macro_observations
  (source_collector, source_id, observation_time);
```

### Migration command

Per CLAUDE.md DATABASE HARD_STOP (never manually create migration `.cs`):

```bash
nerdctl compose exec -T macrosubstrate-dev dotnet ef migrations add InitialCreate \
  --project src/MacroSubstrate \
  --output-dir Data/Migrations
```

If `macrosubstrate-dev` devcontainer doesn't exist yet, run from `secmaster-dev` against the new csproj path during bootstrap, then add the devcontainer in a follow-up story.

### Hypertable + indexes (raw SQL inside the migration, canonical pattern from `FredCollector/src/Data/Migrations/20251118000000_AddEventsTable.cs:55-63`)

```csharp
migrationBuilder.Sql(@"
    SELECT create_hypertable('macro_observations', 'observation_time',
        chunk_time_interval => INTERVAL '1 month', if_not_exists => TRUE);
");

// Story 3.1.2 indexes:
migrationBuilder.Sql(@"CREATE INDEX idx_macro_obs_signal_time ON macro_observations
    (signal_identity_id, observation_time DESC) WHERE signal_identity_id IS NOT NULL;");
migrationBuilder.Sql(@"CREATE INDEX idx_macro_obs_sector_time ON macro_observations
    (atlas_sector_code, observation_time DESC) WHERE atlas_sector_code IS NOT NULL;");
migrationBuilder.Sql(@"CREATE INDEX idx_macro_obs_source_time ON macro_observations
    (source_collector, source_id, observation_time DESC);");

// Down() — explicit retention + hypertable teardown before DropTable
```

### Migration runner (two variants)

- **Variant A (recommended)** — Tiny migrate-only console host inside `MacroSubstrate/src/`. Runs `Database.Migrate()` on startup, exits. Wire into `compose.yaml.j2` as one-shot `migrate-macro-substrate` service that runs before any collector. Ansible playbook gets a new tag `macro_substrate`. Single-source migration ownership preserved.
- **Variant B** — `FredCollector` (first writer per Story 3.2.1) runs `MacroSubstrateDbContext.Database.Migrate()` on startup. No new container, but couples migration runtime to a specific collector — undermines the dedicated-owner argument.

### Solution wiring

Add 3 sln entries (mirrors Events triad at `ATLAS.sln:130–160`):

- `MacroSubstrate/src/MacroSubstrate/MacroSubstrate.csproj`
- `MacroSubstrate/src/MacroSubstrate.Client/MacroSubstrate.Client.csproj`
- `MacroSubstrate/tests/MacroSubstrate.UnitTests/MacroSubstrate.UnitTests.csproj`

Update `CLAUDE.md SERVICES [monorepo]` to list `substrate: MacroSubstrate` (one-line edit, supervisor lands it as part of Story 3.1.0 prep).

> **Correction (post-Story 3.1.0, 2026-05-10):** ATLAS.sln is **gitignored** per commit `6d30136` ("chore: Remove .sln files and fix devcontainer build context"); the sln is dev-local only. `compile.sh` builds csprojs directly without it. Story 3.1.0 landed without sln changes; this Solution-wiring step is dev-local convenience only and **must not be committed**. Future stories should not include sln updates as a deliverable.

## Open risks (NTFY-worthy before ratifying)

1. **New top-level project = new build/deploy artifact.** Variant A adds a 4th container topology pattern (migrate-only). Default to A; alternative is B (piggyback on FredCollector startup). One-line user ratification needed.
2. **`MacroSubstrate.Client` vs. shared model duplication.** Each collector currently has its own private `Observation` shape. Introducing `MacroObservation` as a separate canonical record means three collector PRs in Stories 3.2.1–3.2.4 each take a `ProjectReference` to `MacroSubstrate.Client`. Worth saying out loud so reviewers don't flag YAGNI.
3. **Soft FK to `signal_identities` and `mapping_versions`** (both in `atlas_secmaster`). No PG enforcement; data integrity falls on the writer. Story 3.1.3 versioned-mapping query becomes a JOIN across two databases — same dual-connection pattern SecMaster already uses for dedup. Recommend Story 3.1.3 spec the cross-DB query path explicitly.
4. **D4 schema decision (numeric vs. qualitative XOR via CHECK constraints).** Proposed CHECK constraints above; AC says "developer call, documented in commit". Worth pre-ratifying the CHECK shape now so Story 3.1.1 doesn't get review-bounced.
5. **Idempotency-key choice.** `(source_collector, source_id, observation_time)` matches Story 3.2.1 AC verbatim, but Sentinel qualitative observations don't always have a stable `source_id`. May need `(source_collector, source_id, observation_time, extraction_job_id)` for Sentinel rows. Worth raising before Story 3.2.4.
6. **Hypertable retention default.** FRED `events` table uses 2-year retention; macro series (CPI, GDP) often want 30+ years. Recommend `INTERVAL '50 years'` or **no retention policy** on this hypertable. Confirm before merging migration.
7. **Devcontainer bootstrap.** New project needs `.devcontainer/` (compile.sh, build.sh, compose.yaml, devcontainer.json) per CLAUDE.md `BUILD [devcontainer]`. Adds ~30min mechanical setup. Consider splitting devcontainer setup into a Story 3.1.0 prep story so 3.1.1 stays focused on DDL.

## Files inspected (Plan-agent provenance)

- `docs/plans/atlas-matrix/epic-3-macro-observations.md`
- `docs/atlas-matrix-mvp-plan.md` (lines 366–415)
- `docs/atlas-matrix-handoff-v2.md` §4 + §5
- `CLAUDE.md` (DATABASE, SERVICES, DATA_FLOW)
- `STATE.md`
- `SecMaster/src/Data/SecMasterDbContext.cs`
- `SecMaster/src/Data/Migrations/20241214160001_AddInstrumentEmbeddings.cs` (raw-SQL-in-EF reference)
- `SecMaster/src/Services/Dedup/FredObservationSourceProvider.cs` (cross-DB precedent)
- `FredCollector/src/Data/Migrations/20251118000000_AddEventsTable.cs` (canonical `create_hypertable` + retention pattern)
- `FredCollector/src/Data/FredCollectorDbContext.cs`
- `FredCollector/src/Services/DataCollectionService.cs` (lines 160–200)
- `FredCollector/src/Entities/SeriesConfig.cs` (sector tag + version label as plain columns)
- `OfrCollector/src/Data/Repositories/IOfrRepository.cs`
- `SentinelCollector/src/Entities/ExtractedObservation.cs`
- `SentinelCollector/src/Data/IShadowObservationWriter.cs` (raw-INSERT shadow-write precedent)
- `Events/src/Events.EntityFrameworkCore/README.md` + `Events.EntityFrameworkCore.csproj` (triad pattern)
- `ThresholdEngine/src/Data/ThresholdEngineDbContext.cs`
- `deployment/artifacts/compose.yaml.j2` (lines 587–630)
- `ATLAS.sln`

## Supervisor verification (post-recon spot-check, 2026-05-10)

- ✓ `Events / Events.Client / Events.EntityFrameworkCore` triad confirmed at `Events/src/`.
- ✓ `FredObservationSourceProvider.cs` exists in `SecMaster/src/Services/Dedup/` — cross-DB pattern is real.
- ✓ `AddInstrumentEmbeddings.cs` + `.Designer.cs` companion exist (CLAUDE.md DATABASE HARD_STOP precedent).
- ✓ `AddEventsTable.cs` exists in `FredCollector/src/Data/Migrations/` — `create_hypertable` precedent confirmed.
