# OfrCollector — Single Source of Truth Migration

## GOAL

Eliminate the dual-source-of-truth bug where `SeriesManagementService` (the MCP/admin path) writes series to the DB while `StfmCollectionService` and `HfmCollectionService` read series from a baked JSON file via `SeriesConfigurationProvider`. End state: DB is authoritative for active series; JSON files, the file watcher, and the Containerfile bake are all gone.

**Accept criteria**

- `HfmCollectionService` and `StfmCollectionService` source their series list from the DB (same repository the MCP/admin layer writes to). No reference to `ISeriesConfigurationProvider`.
- `SeriesConfigurationProvider`, `ISeriesConfigurationProvider`, and `SeriesConfigurationProviderTests` deleted.
- `OfrCollector/config/{hfm,stfm}-series.json` deleted.
- `OfrCollector/src/Containerfile:94` `COPY ... /src/OfrCollector/config ./config` removed.
- Adding a series via MCP `add_hfm_series` results in that series being collected on the next worker tick (smoke-tested end-to-end after deploy).
- Startup refuses to run if both `ofr_hfm_series` and `ofr_stfm_series` are empty (currently the JSON masks empty-DB).
- A first-boot seeder populates the DB from the JSON's contents *only if* the tables are empty — preserves historical defaults for fresh installs without making the JSON load-bearing.
- All existing OFR tests still pass; add tests for the seeder's empty-table-only behavior and the empty-table startup guard.

## ARCH

Today:
```
[MCP add_hfm_series]
   ↓
SeriesManagementService → IHfmSeriesRepository → ofr_hfm_series (DB) [331 rows]
                                                       ↑ DECORATIVE
[Worker tick]
   ↓
HfmCollectionService → SeriesConfigurationProvider → /app/config/hfm-series.json [baked]
                              ↑ FileSystemWatcher hot-reload
```

Two separate paths; the worker never reads what the admin API writes. Operator-added series silently never get polled.

Target:
```
[MCP add_hfm_series]
   ↓
SeriesManagementService → IHfmSeriesRepository → ofr_hfm_series (DB) [authoritative]
                                                       ↑ READ
[Worker tick]
   ↓
HfmCollectionService → IHfmSeriesRepository.GetActiveAsync()
```

Single path. No JSON, no watcher, no provider, no bake. First-boot DB seeding handled by a one-shot startup seeder (per CLAUDE.md: `SEED_DATA: EF HasData() | app-level seeding on startup`).

## STATUS

- ✓ 0. PR D (#188): drop FredCollector vestigial empty `config/` bake
- ✓ 1. Audit: JSON has 23 canonical defaults; DB has 431 operator-added rows. 9 overlap. 14 JSON-only mnemonics are *currently being collected via the JSON path* and must be imported to DB or they'll stop being polled at cutover. 417 DB-only rows are ghosts the worker never sees today.
- → 2. Build idempotent seeder: embed both JSON files as resources, upsert-if-missing on every boot (semantics revised from "seed only if empty" — see **AUDIT FINDINGS** below).
- ◯ 3. Switch readers: `HfmCollectionService` + `StfmCollectionService` → `_repository.GetActive*Async()`; preserve priority ordering.
- ◯ 4. Add startup guard: refuse to start if both series tables are empty *after* seeder runs.
- ◯ 5. Delete legacy: `SeriesConfigurationProvider`, `ISeriesConfigurationProvider`, JSON files, FileSystemWatcher, DI registration, Containerfile `COPY config` line, related tests.
- ◯ 6. Smoke: add a series via MCP, confirm it gets collected on next tick (Loki + DB row check).

## AUDIT FINDINGS (2026-04-20)

|            | JSON | DB  | Both | JSON-only | DB-only |
|------------|-----:|----:|-----:|----------:|--------:|
| HFM        |   9  | 331 |   4  |     **5** |   **327** |
| STFM       |  14  | 100 |   5  |     **9** |    **95** |

**Interpretation**: the JSON was the collector's runtime config (23 series being polled). MCP `add_*_series` writes lands in DB but never made it into JSON, so those 417 rows are "ghost series" — added successfully, one-shot backfilled, then never collected on schedule because the worker reads JSON only.

**Cutover requirement**: the 14 JSON-only mnemonics need to exist in the DB *before* the reader switch, or we stop collecting them. Can't use "seed only when empty" semantics since the tables are already populated.

**Revised seeder contract**: run on every boot, upsert-if-missing per mnemonic. DB-authoritative rows are untouched; any JSON defaults that aren't in the DB get inserted. Idempotent, self-healing if someone accidentally deletes a default row, and safe to re-run.

## TASKS

### 0. PR D — FredCollector vestigial bake removal (separate, trivial, do first)

Same shape as Sentinel #187 but smaller: the `config/` dir in `FredCollector/` is empty (just `.gitkeep`); nothing reads it; the `COPY` line in Containerfile is dead weight that ships with every build. Drop it. No code or tests touched.

**Deliverables**:
- `FredCollector/src/Containerfile:39` line removed
- Verify image: `ls /app/config` returns "No such file or directory"
- One commit, one PR, no behavioral change

### 1. Audit JSON vs DB

- Walk `OfrCollector/config/{hfm,stfm}-series.json` — they nest series under `datasets[<dataset_name>]`. Build a flat list of `{mnemonic, dataset, dataset-specific fields, priority, enabled, …}`.
- Compare against `SELECT mnemonic, dataset FROM ofr_hfm_series` and same for stfm.
- Report:
  - JSON-only mnemonics (need import)
  - DB-only mnemonics (operator-added, must not be lost)
  - Mismatches in fields between JSON and DB rows for the same mnemonic (decide policy: DB wins, since that's where MCP writes land)
- Output: a small markdown summary in this STATE for the import step to follow.

### 2. Idempotent seeder (revised)

- New class `OfrSeriesDbSeeder` in `OfrCollector/src/Data/`. Runs at startup before `IHostedService`s.
- Behavior (per table, independently):
  - Parse embedded-resource JSON for the table (`hfm-series.json` / `stfm-series.json`).
  - For each mnemonic in the JSON: check DB; if missing, insert. If present, skip (do NOT overwrite — DB may have operator edits).
  - Log counts: `Seeded {N} {table} series, skipped {M} already present`.
- Embed the seed JSON as `<EmbeddedResource>` in `OfrCollector.csproj`, not `/app/config/`. Makes the seed code-versioned (defaults that ship with the binary), not config-in-disguise.
- Tests:
  - `OfrSeriesDbSeederTests`: `should_insert_mnemonic_when_missing`, `should_skip_mnemonic_when_present`, `should_preserve_existing_row_fields_when_mnemonic_present`, `should_seed_each_table_independently`.
- Tests:
  - `OfrSeriesDbSeederTests`: `should_seed_when_table_empty`, `should_skip_when_table_has_rows`, `should_seed_each_table_independently`.

### 3. Switch readers to DB

- `HfmCollectionService:?` — find every `_configProvider.GetHfmConfig().*` call. Replace with `_hfmRepository.GetActiveAsync(ct)` (or whichever repo method already serves the MCP read path; check `SeriesManagementService` for the read shape).
- Preserve ordering: if the JSON config exposes `priority` for scheduling, ensure the DB column / repository sort matches.
- Same surgery on `StfmCollectionService`. Confirmed call site: `StfmCollectionService.cs:144`.
- Drop the `_configProvider` constructor param and DI binding from these two classes.
- Update unit tests to mock `IHfmSeriesRepository` / `IStfmSeriesRepository` instead of `ISeriesConfigurationProvider`. Existing tests that exercised the file-watcher behavior get deleted (no longer applicable).

### 4. Empty-table startup guard

- After the seeder runs, check counts. If both tables are still empty (e.g. seeder failed to populate, or someone deployed with empty seed JSON), throw `InvalidOperationException` with a readable message: `"OFR series tables empty after seed pass. Either the seed JSON is empty or the seeder failed. Check logs for OfrSeriesDbSeeder."` Container exits non-zero.
- Distinguish from "operator deliberately disabled all series" — that's `enabled=false` rows, still present. The guard is *no rows at all*, which is unambiguously misconfig.

### 5. Delete legacy

- Files to delete:
  - `OfrCollector/src/Configuration/SeriesConfigurationProvider.cs`
  - `OfrCollector/src/Configuration/ISeriesConfigurationProvider.cs` (if it exists; may be inline)
  - `OfrCollector/config/hfm-series.json`
  - `OfrCollector/config/stfm-series.json` (after their content has been embedded as seed)
  - `OfrCollector/config/` directory entirely
  - any `SeriesConfigurationProviderTests*.cs`
- Containerfile: drop `COPY --from=build --chown=appuser:appuser /src/OfrCollector/config ./config` (line 94).
- DI registration: drop `services.AddSingleton<ISeriesConfigurationProvider, SeriesConfigurationProvider>()` (or however it's registered).
- Verify with grep: zero references to `SeriesConfigurationProvider`, `ISeriesConfigurationProvider`, `hfm-series.json`, `stfm-series.json` across `OfrCollector/`.

### 6. End-to-end smoke

- Deploy.
- Pick a mnemonic NOT in the seeded set. `mcp__ofr-collector__add_hfm_series mnemonic=<X> backfill=false`.
- Wait one collection tick (check service config for cadence).
- Verify: `ofr_hfm_observations` has new rows tagged with mnemonic `<X>` AND Loki shows the worker mentioning `<X>` in its collection log.
- Then `mcp__ofr-collector__delete_hfm_series mnemonic=<X>`. Confirm the next tick stops collecting it.

## CONTEXT

- Branch (suggested): `fix/ofr-single-source-of-truth` for the migration; `fix/fred-vestigial-bake` for PR D
- Per CLAUDE.md DATABASE rules: `SEED_DATA: EF HasData() | app-level seeding on startup`. The seeder approach is the app-level option; EF `HasData` would also work but ties the seed data to the migration, which isn't ideal here since the seed is a one-time historical import, not part of the schema's lifecycle.
- DB connection for inspection: `sudo nerdctl exec timescaledb psql -U ai_inference -d atlas_data` per CLAUDE.md DATABASE.
- Counts at start of work (2026-04-20): `ofr_hfm_series`=331, `ofr_stfm_series`=100. JSON datasets: HFM `{fpf, ficc}`, STFM `{fnyr, repo, mmf, nypd, tyld}`.
- Don't try to split this into many small PRs in the middle. Steps 1-5 are tightly coupled — half-deleting the legacy path while collectors still call it would crash. One worktree, one PR, ordered commits inside it.

## ATTEMPTED

- (none yet)
