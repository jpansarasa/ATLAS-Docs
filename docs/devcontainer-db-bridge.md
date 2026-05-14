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
| **A — Join `ai-inference` network** (chosen) | Devcontainer becomes a network peer of `timescaledb`. Tests connect via `Host=timescaledb` and use isolated `*_integration_test` databases. | Reuses the live DB infra (no extra container); test isolation is at the database level, not the instance level. |
| B — Dedicated test timescaledb | Add a second `timescaledb` container scoped to devcontainer. | Doubles DB infra; needs schema seed/keep-in-sync; extra ~100MB RAM per dev session. |
| C — Host port-forward | Expose timescaledb on `localhost:5432`, connect via `host.containers.internal`. | Adds a host-side attack surface; requires firewall consideration; CNI gymnastics for `host.containers.internal` on containerd. |

Option A wins because:

1. The `IntegrationTestBase` pattern (shared across 6+ test projects) already
   creates and drops dedicated `*_integration_test` databases inside the DB
   instance — instance-level isolation is unnecessary.
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
    networks:
      - default
      - ai-inference

networks:
  ai-inference:
    external: true
    name: ai-inference
```

Each integration test base class reads `DB_HOST` / `DB_PORT` / `DB_USER` /
`DB_PASSWORD` and connects to a per-service isolated test database (e.g.
`atlas_secmaster_integration_test`, `ofr_integration_test`,
`finnhub_integration_test`).

## Bridged services

| Service | Test database | Status |
|---------|---------------|--------|
| SecMaster | `atlas_secmaster_integration_test` | bridged (this PR) |
| ThresholdEngine | `atlas_integration_test` | bridged (this PR) |
| MacroSubstrate | n/a (no integration tests today) | bridged (this PR) |
| FredCollector | `atlas_integration_test` | bridged (this PR) |
| OfrCollector | `ofr_integration_test` | bridged (this PR) |
| SentinelCollector | n/a (no integration tests today) | already on `ai-inference`, env vars added |
| NasdaqCollector | `atlas_integration_test` | already bridged |
| FinnhubCollector | private `finnhub-timescaledb` container | self-contained, not bridged |
| AlphaVantageCollector | private (self-contained) | not bridged |
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
