# MacroSubstrate.Migrator

One-shot console host that applies `MacroSubstrate` EF migrations and exits. Wired into `compose.yaml` as `migrate-macro-substrate`, runs before any collector that writes to `macro_observations`.

## Overview

Migration ownership for `macro_observations` doesn't belong to a single collector — multiple collectors write to it. This host (Variant A per owner-rec §3.1.1) is the single project that owns schema deployment. It runs `MacroSubstrateDbContext.Database.MigrateAsync()` and exits with `0` on success, `1` on failure. The compose graph blocks dependent services until this completes successfully.

## Behaviour

1. Reads connection string from `ConnectionStrings__AtlasData` (falls back to `ConnectionStrings__AtlasDb`). Loud-fails if neither is set.
2. Builds a `MacroSubstrateDbContext` with Npgsql `EnableRetryOnFailure` (10x / 15s) — longer than the runtime retry to absorb cold-start TimescaleDB.
3. Logs pending migrations by name (or "schema already current" if none).
4. Calls `MigrateAsync()`.
5. Exits `0` (success) or `1` (failure with full exception).

## Configuration

| Variable | Description | Default |
|---|---|---|
| `ConnectionStrings__AtlasData` | Primary connection string. Required (or `AtlasDb` fallback). | Required |
| `ConnectionStrings__AtlasDb` | Alternate name for the same target — preserved for collectors that already use this key. | Optional |

## Project Structure

```
MacroSubstrate.Migrator/
  Program.cs           — top-level statements: build provider, migrate, exit
  Containerfile        — multi-stage build → macro-substrate-migrator:latest
```

## Development

```bash
dotnet build MacroSubstrate/src/MacroSubstrate.Migrator/MacroSubstrate.Migrator.csproj

# Run locally against a dev TimescaleDB
ConnectionStrings__AtlasData='Host=localhost;Port=5432;Database=atlas_data;Username=...;Password=...' \
  dotnet run --project MacroSubstrate/src/MacroSubstrate.Migrator
```

## Deployment

- **Image:** `macro-substrate-migrator:latest`
- **Compose service:** `migrate-macro-substrate` (restart policy: `no` — one-shot)
- **Build:** part of the standard ansible deploy; downstream services declare `depends_on: { migrate-macro-substrate: { condition: service_completed_successfully } }`.

```bash
ansible-playbook playbooks/deploy.yml --tags macro-substrate
```

Per CLAUDE.md `DEPLOYMENT [HARD_STOP]`: never edit `/opt/ai-inference/compose.yaml` directly.

## See Also

- [MacroSubstrate](../MacroSubstrate/README.md) — owns the `DbContext` + migrations applied by this host
- [MacroSubstrate.Client](../MacroSubstrate.Client/README.md) — write-side contract
