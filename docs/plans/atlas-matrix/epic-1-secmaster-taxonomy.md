# Epic 1 — SecMaster Sector Taxonomy & Classification

## Goal

Add a NAICS classification spine, a versioned 11-sector rollup, EDGAR-sourced
instrument classification, OpenFIGI cross-reference (already mostly built),
series classification, and a signal-identity catalog for dedup. SecMaster
becomes the spine the rest of the rework builds against.

## Branch

`epic/1-secmaster-taxonomy` off `main`. Stories merge into the epic branch
via small PRs; epic merges to `main` when all features pass AC and a
`pr-review-toolkit:review-pr` pass is clean.

## Phase + dependencies

- Phase: **1 (sequential, blocking everything else)**
- Blocks: Phase 2 (Epics 2, 3, 4.6 prerequisites)
- Blocked by: nothing — kicks off the matrix work

## Canonical AC

See `/home/james/ATLAS/docs/atlas-matrix-mvp-plan.md` Epic 1 (lines 151–276
in the canonical doc). Story-level AC is not duplicated here.

## Features (with execution notes)

### 1.1 — NAICS classification spine
- **1.1.1** NAICS taxonomy import. Source: Census Bureau NAICS 2022 bulk
  file. Land in a SecMaster reference table `naics_codes` (6-digit
  granularity, hierarchy preserved). Repeatable importer + manifest.
- **1.1.2** NAICS lookup API. New methods on the SecMaster service:
  `ResolveNaicsAsync(code)` → hierarchy; `ListNaicsByPrefixAsync(prefix)`.
  Surface via REST + gRPC + MCP.

### 1.2 — Custom 11-sector rollup
- **1.2.1** Initial mapping defined. New table
  `naics_sector_rollup (naics_code, atlas_sector, mapping_version_id, rationale)`.
  Seed v1.0 covering all NAICS 6-digit codes; non-obvious assignments get
  a `rationale` row. The 11 sectors: Energy, Materials, Industrials, Cons.
  Disc., Cons. Staples, Health Care, Financials, Info Tech, Comm. Services,
  Utilities, Real Estate.
- **1.2.2** Rollup lookup API. `ResolveAtlasSectorAsync(naics, asOf?)`.
  Default = current version; historical lookup honors version active on
  date.

### 1.3 — Mapping versioning
- **1.3.1** Versioning schema. New table `mapping_versions
  (id, version_label, effective_from, effective_to, notes)`. Active version
  at any date is unambiguous (one row covers the date).
- **1.3.2** Version-aware lookup. Synthetic-mapping-change test: insert a
  v1.1 reassigning a single NAICS code and verify Story 1.2.2 returns
  different results across the change date.

### 1.4 — Instrument-level classification (PS3 destructive migration)
- **1.4.1** EDGAR ingestion. New job pulling SIC + NAICS from EDGAR for
  filers in scope. Refresh weekly default. Provenance stored:
  `(source, fetched_at, raw_fields_json)`.
- **1.4.2** Manual override mechanism. Table `instrument_sector_overrides
  (instrument_id, atlas_sector, naics_code, reason, who, when)`. Override
  takes precedence over EDGAR-derived classification. Audit log.
- **1.4.3** **PS3 — Replace `instruments.sector` + `instruments.industry`
  with `naics_code` + `rollup_version`.** Audit completed at
  `/tmp/sentinel-remediation/sector-industry-readers.md`: 64+ readers,
  35+ files, 5 high-risk findings:
  1. **Vector embeddings** — `EmbeddingService` indexes "Sector: X" /
     "Industry: Y" in embedding text. Re-embedding every active
     instrument required after migration.
  2. **Public DTO** — `InstrumentDto.Sector` is exposed via HTTP. User
     is sole consumer; bounded blast radius. Replace cleanly.
  3. **MCP tool output** — `SecMasterTools.cs:get_instrument` returns
     Sector. Update in same PR.
  4. **Sentinel ExtractedObservation.Sector** — observations carry
     resolved sector for audit/digest. Migrate the column to NAICS-
     code-derived value; digest renderer reads through SecMaster
     resolution at render time, not the cached column.
  5. **DB indexes** — `(Sector, Country)` composite + `Industry` simple.
     Drop in the migration; replace with `(rollup_version, naics_code)`
     and `(rollup_version, atlas_sector)` if needed for query patterns.
  Migration sequence (dispatched as a single PR with multiple commits):
    a. Add `naics_code` + `rollup_version` columns (nullable initially).
    b. Backfill: walk active instruments, derive NAICS via OpenFIGI /
       EDGAR / manual override → rollup → atlas_sector.
    c. Re-run `EmbeddingService.BuildEmbeddingForInstrumentAsync` for
       all active instruments (background job; CLAUDE.md says not to
       trigger embedder churn unnecessarily — make this idempotent).
    d. Update DTOs / MCP tools / digest renderer / event slug
       generation in the same PR. Remove `Sector`/`Industry` references.
    e. Migration drops the old columns + indexes; adds new indexes.
  A reclassification report shows instruments whose tag changes when
  the mapping version advances.

### 1.5 — OpenFIGI integration (mostly already built)
- **1.5.1** OpenFIGI lookup integration. Recon found
  `OpenFigiEnrichmentBackgroundService` + `IOpenFigiClient` already live.
  This story reduces to: verify it covers (CUSIP, ISIN, ticker, FIGI)
  resolution, cache hits respect free-tier rate limit, and exposes a
  `ResolveIdentifiersAsync` API on SecMaster.

### 1.6 — Series classification
- **1.6.1** Industry-aggregate series tagging. FRED industry-aggregate
  series with NAICS codes auto-resolve to rollup sector under the active
  mapping version. Tag stored alongside series metadata. Versioning is
  honored for historical observations.
- **1.6.2** Pure-macro series — signal identity only. Pure-macro series
  carry signal identity (e.g., "CPI-headline-yoy") but no sector tag.
  Signal identity is the dedup grouping vocabulary used by Epic 2's
  alpha-factor math (Feature 2.5).

### 1.7 — Signal identity & dedup grouping
- **1.7.1** Signal identity catalog. New table `signal_identities
  (id, label, description, aliases_json, default_burst_window_seconds)`.
  Initial catalog seeded with signals already tracked across collectors.
  Queryable from Sentinel, FRED, OFR, TE.
- **1.7.2** Dedup grouping API. Given (signal_identity, time window),
  return all observations across substrates that share the identity in
  the window. Used by TE alpha-factor math (Story 2.5.2).

## Files to touch

- `SecMaster/src/Data/Entities/InstrumentEntity.cs`
- `SecMaster/src/Data/Entities/` (new entities: `NaicsCodeEntity`,
  `NaicsSectorRollupEntity`, `MappingVersionEntity`,
  `InstrumentSectorOverrideEntity`, `SignalIdentityEntity`)
- `SecMaster/src/Data/Migrations/` (new migrations — must use
  `dotnet ef migrations add` per CLAUDE.md HARD_STOP)
- `SecMaster/src/Data/Repositories/` (new repos for the above)
- `SecMaster/src/Endpoints/` (new endpoints; existing are listed below)
  - `AdminEndpoints.cs`, `CatalogEndpoints.cs`, `CollectorEndpoints.cs`,
    `InstrumentEndpoints.cs`, `RegistrationEndpoints.cs`,
    `ResolutionEndpoints.cs`, `SearchEndpoints.cs`,
    `SemanticSearchEndpoints.cs`
  - new: `NaicsEndpoints.cs`, `MappingVersionEndpoints.cs`,
    `SignalIdentityEndpoints.cs`
- `SecMaster/src/Grpc/RegistryGrpcService.cs`,
  `SecMaster/src/Grpc/ResolverGrpcService.cs` (extend)
- `SecMaster/mcp/Tools/` (new MCP tools: `resolve_naics`, `list_naics`,
  `resolve_atlas_sector`, `lookup_signal_identity`,
  `dedup_grouping_query`)
- `SecMaster/src/Services/HybridResolutionService.cs` (signal-identity
  awareness in resolution path; PR #214 deterministic-gate revert
  is out-of-scope for this epic)
- net-new EDGAR client + worker
- existing `OpenFigiEnrichmentBackgroundService` — verify, do not rewrite

## Subagent dispatch notes

- Story 1.1.1 (NAICS import): `general-purpose` agent. Bounded
  deliverable: importer + reference table + repeatable manifest.
- Story 1.4.3 (destructive migration): **must wait** for the audit at
  `/tmp/sentinel-remediation/sector-industry-readers.md`. Two-agent
  approach — one updates SecMaster (entity, migration, repo, endpoints),
  another updates downstream readers across the monorepo. Coordinate
  via a single PR.
- Stories with EF migrations: per CLAUDE.md HARD_STOP, **never**
  hand-author migration `.cs` files — agents must run
  `dotnet ef migrations add` inside the SecMaster devcontainer.
- Story 1.5.1: lightweight `Explore` then `code-simplifier` — verify +
  document, don't reimplement.
- PR review: `pr-review-toolkit:review-pr` on every story PR; epic-level
  `code-review:code-review` before merge to `main`.

## Verification (epic-exit AC)

- `mcp__secmaster-mcp__resolve_atlas_sector` returns the right rollup for
  AAPL, JPM, JNJ, XOM (one per spectrum-anchor sector).
- A synthetic v1.1 of the rollup mapping changes the result for one of
  the above, and historical-as-of queries return v1.0 results.
- `instruments` table has `naics_code` + `rollup_version` columns; old
  `sector` + `industry` are gone; every reader from the audit report is
  updated and tests pass.
- `signal_identities` table is populated; the dedup-grouping API returns
  observations across substrates for a known signal in a known window.
- OpenFIGI cross-reference resolves AAPL → (FIGI, CUSIP, ISIN, ticker)
  consistently.
- `bash SecMaster/.devcontainer/compile.sh` runs clean (0 errors, 0
  warnings, all tests pass).
- `ansible-playbook playbooks/deploy.yml --tags secmaster` deploys clean;
  Loki shows no errors in the 10-minute window post-deploy
  (`feedback_completion_gate`).

## Smoke test for the epic

After deploy, dispatch a `general-purpose` smoke agent with this script:
1. Hit MCP `resolve_atlas_sector` for the four anchor instruments.
2. Hit MCP `lookup_signal_identity` for "CPI-headline-yoy" and an
   instrument-tagged signal.
3. Query SecMaster directly:
   `SELECT instrument_id, naics_code, rollup_version FROM instruments
    WHERE symbol IN ('AAPL','JPM','JNJ','XOM');`
4. Run a versioned-mapping query for a synthetic past date and verify
   the result reflects the active version.
