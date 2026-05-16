# SecMaster/scripts

Operator scripts for SecMaster — schema bring-up, seed data, and one-shot data-building utilities.

## Files

| File | Purpose |
|---|---|
| `build-rollup.py` | Builds the ATLAS sector rollup CSV (`atlas-sector-rollup-v1.0.csv`) from the NAICS-2022 master file (`data/naics-2022.csv`). Output columns: `naics_code, atlas_sector_code, rationale`. Re-run when NAICS source changes or rollup heuristics are updated. |
| `run_migrations.sh` | **Legacy** raw-SQL migration runner. Invokes `psql -f migrations/001_initial_schema.sql` and (with `--seed`) `seed_data.sql`. Reads `SECMASTER_DB_HOST` / `SECMASTER_DB_PORT` / `SECMASTER_DB_NAME` / `SECMASTER_DB_USER`. **Do not use against a live database** — production schema is owned by `src/Data/Migrations/` (EF Core) and applied by the SecMaster service on startup. Kept here only because the legacy SQL is referenced for historical bring-up notes. |
| `seed_data.sql` | Raw SQL seed for well-known instruments + source mappings used by ThresholdEngine patterns. **Reference / dev-bootstrap only** — production seeding flows through EF `HasData()` + app-level startup seeding per CLAUDE.md `DATABASE [ef_core]`. |
| `migrations/001_initial_schema.sql` | Legacy SQL baseline kept for reference. Current schema is owned by `src/Data/Migrations/` (EF Core). Do not use against a fresh database — use the EF migrator. |

## When to run these

- **`build-rollup.py`** — when adding or updating the NAICS → ATLAS sector mappings. Output is checked into `SecMaster/data/`.
- **`run_migrations.sh` / `seed_data.sql`** — local dev only. Production migrations apply automatically on service startup.

## Anti-pattern reminder

Per CLAUDE.md `DATABASE [ef_core]`:
- Schema changes go through EF migrations — never raw SQL.
- Seed data goes through EF `HasData()` or app-level startup seeding — never raw SQL during deploy.

The SQL files in this directory exist as historical reference + dev convenience and should not be invoked from any deployment pipeline.

## See Also

- [SecMaster](../README.md) — service README
- [SecMaster/src/Data/Migrations](../src/Data/Migrations/) — canonical EF migrations
