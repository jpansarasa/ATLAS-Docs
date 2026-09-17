# Devcontainer ↔ TimescaleDB Bridge

## What

ATLAS service devcontainers (`<Service>/.devcontainer/compose.yaml`) are bridged
to the production `timescaledb` container via the shared external `ai-inference`
network. Each devcontainer joins that network as a peer of `timescaledb`,
resolves it by DNS, and authenticates with the standard `atlas_user` credential.

This unblocks integration tests that need a real PostgreSQL/TimescaleDB instance
(EF migrations, `EnsureCreated`, pgvector + pg_trgm extensions, FK constraints,
hypertable behaviour).

## Why this approach

Three options were considered:

| Option | Description | Tradeoff |
|--------|-------------|----------|
| **A — Join `ai-inference` network** (chosen) | Devcontainer becomes a network peer of `timescaledb`. Tests connect via `Host=timescaledb` and use isolated per-worktree test databases. | Reuses the live DB infra (no extra container); test isolation is at the database level, not the instance level. |
| B — Dedicated test timescaledb | Add a second `timescaledb` container scoped to devcontainer. | Doubles DB infra; needs schema seed/keep-in-sync; extra ~100MB RAM per dev session. |
| C — Host port-forward | Expose timescaledb on `localhost:5432`, connect via `host.containers.internal`. | Adds a host-side attack surface; requires firewall consideration; CNI gymnastics for `host.containers.internal` on containerd. |

Option A wins because:

1. The `IntegrationTestBase` pattern (shared across 6+ test projects) already
   creates and drops dedicated test databases inside the DB instance —
   instance-level isolation is unnecessary as long as every name is unique per
   worktree and per service (see below).
2. `NasdaqCollector` + `SentinelCollector` already use this pattern, so the
   precedent is established.
3. Zero new infrastructure.

## How it works

Each bridged `<Service>/.devcontainer/compose.yaml` declares:

```yaml
services:
  <service>-dev:
    environment:
      - DB_HOST=timescaledb
      - DB_PORT=5432
      - DB_USER=atlas_user
      - DB_PASSWORD=${DB_PASSWORD:-atlas_secure_password_2025}
      - ATLAS_WORKTREE_ID=${ATLAS_WORKTREE_ID:-}
    networks:
      - default
      - ai-inference

networks:
  ai-inference:
    external: true
    name: ai-inference
```

Each integration test base class reads `DB_HOST` / `DB_PORT` / `DB_USER` /
`DB_PASSWORD` and connects to an isolated test database named
`<base>_<ATLAS_WORKTREE_ID>_<service>`, built by the service's
`tests/*.IntegrationTests/Infrastructure/IntegrationDatabaseName.cs`. The instance
is shared, so the name must be unique per worktree AND per service: with a fixed
name, one run's `DROP DATABASE ... WITH (FORCE)` terminated another run's sessions
(57P01), and three services shared `atlas_integration_test`. `compile.sh` hands the
worktree key over through `scripts/devcontainer-owner.sh`. A fixture refuses to run
without it, and drops its database when it finishes.

## Bridged services

| Service | Test database base names | Status |
|---------|--------------------------|--------|
| SecMaster | `atlas_secmaster_integration_test` | bridged |
| ThresholdEngine | `atlas_matrix_idempotency_test`, `atlas_latest_cells_query_test` | bridged |
| MacroSubstrate | n/a (no integration tests today) | bridged |
| FredCollector | `atlas_integration_test`, `atlas_api_integration_test`, `atlas_grpc_integration_test`, `atlas_macro_idempotency_test` | bridged |
| OfrCollector | `ofr_integration_test`, `atlas_macro_ofr_idempotency_test` | bridged |
| SentinelCollector | n/a: its fixtures use the private `timescaledb-test` sidecar (`CROSSCOLLECTOR_TEST_DB`) | on `ai-inference` |
| NasdaqCollector | `atlas_integration_test` | bridged |
| FinnhubCollector | `finnhub_integration_test` | bridged, but the fixture's fallback host `finnhub-timescaledb` resolves nowhere (docs/BACKLOG.md) |
| AlphaVantageCollector | `atlas_integration_test` | bridged, but the compose file sets no `DB_PASSWORD` (docs/BACKLOG.md) |
| CalendarService | private `calendar-db` container | not bridged |
| Reports | n/a (unit tests only) | not bridged |

## DB ownership prerequisite (one-time, already applied)

Test databases must be owned by `atlas_user` (since `atlas_user` is the
connection identity used by `IntegrationTestBase`). The pgvector and pg_trgm
extensions require superuser to install, so they were pre-installed into
`template1` so each new test database inherits them.

If a test database gets created by a different user (e.g. `ai_inference`
running migrations), reassign ownership:

```sh
PGPASSWORD=change_me_in_production sudo nerdctl exec timescaledb \
  psql -h localhost -U ai_inference -d postgres \
  -c "ALTER DATABASE <test_db_name> OWNER TO atlas_user;"
```

## Running integration tests

From the devcontainer (`./compile.sh --integration`):

```sh
cd <Service>/.devcontainer
./compile.sh --integration
```

The compile script brings the devcontainer up, runs unit tests, then runs the
integration test project. Teardown happens via `trap`.

## Disabling the bridge

To run a devcontainer without joining `ai-inference` (e.g. on a machine where
the network doesn't exist), remove the `networks: [default, ai-inference]`
block + the top-level `networks: { ai-inference: { external: true } }` block.
Integration tests will fail to resolve `timescaledb`, but unit tests still run.

## Security note

The default `atlas_secure_password_2025` is the same credential used in
production compose. It is published in the repository for development
convenience (along with `init-db.sh`). For any environment where this is a
concern, override via `DB_PASSWORD=<secret>` in the devcontainer's
environment before starting the compose stack.
