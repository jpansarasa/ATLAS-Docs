# docs/ — Index

`docs/` holds **current-state system documentation only** — what is implemented and running,
not plans, iteration history, or roadmaps. History lives in git: completed phases are tagged
per `CLAUDE.md ## PHASE_TAGS`, and [RELEASES.md](./RELEASES.md) is the named index (each entry
carries the recovery command for the docs retired with it).

## Core reference set

| Doc | What it covers |
|---|---|
| [EXECUTIVE-SUMMARY.md](./EXECUTIVE-SUMMARY.md) | What ATLAS is today: the matrix, the Sentinel pipeline, inference topology, data fleet, operations |
| [ARCHITECTURE.md](./ARCHITECTURE.md) | The system as built: stacks, services, data flows, resolution cascade, DB layout, deployment, observability |
| [MATRIX.md](./MATRIX.md) | Signal-matrix deep-dive: cell identity, invariants, feed flow, formula/decay/source-trust, consumption, live numbers |
| [SENTINEL-RLM.md](./SENTINEL-RLM.md) | Sentinel extraction model/backend/VRAM constraints and troubleshooting |
| [GRPC-ARCHITECTURE.md](./GRPC-ARCHITECTURE.md) | gRPC streaming patterns and contracts |
| [OBSERVABILITY.md](./OBSERVABILITY.md) | OTEL pipeline, dashboards, tracing conventions |
| [THRESHOLDENGINE-PATTERNS.md](./THRESHOLDENGINE-PATTERNS.md) | Pattern catalog and category weights |
| [RELEASES.md](./RELEASES.md) | Phase/epic outcomes, tags, and doc-retirement recovery pointers (append-only) |

## Operational reference

| Doc | What it covers |
|---|---|
| [FRED_SERIES_REFERENCE.md](./FRED_SERIES_REFERENCE.md) | FRED series catalog mapping (live catalog: fred-collector MCP) |
| [atlas-sectorweights-methodology.md](./atlas-sectorweights-methodology.md) | How pattern sector weights are derived |
| [Guide to Grafana Dashboards.md](./Guide%20to%20Grafana%20Dashboards.md) | Dashboard authoring guide |
| [devcontainer-db-bridge.md](./devcontainer-db-bridge.md) | Devcontainer ↔ TimescaleDB bridge |
| `../deployment/README.md` | Deployment machinery (ansible, tags, restart semantics, AutoFix) |

Service-level truth lives with each service: `README.md` (humans) and `AGENT_README.md`
(read-first architecture card) in every service directory. README templates used by the
consistency tooling live in `.claude/skills/readme-consistency/`.

## Curation policy

- **Current-state only.** A doc in this directory must describe the system as it runs. If the
  implementation moves, the doc moves with it or gets retired.
- **No plans/specs/iteration docs on main.** Working docs are retired via `git rm` at phase
  completion; RELEASES.md records each retirement with a `git show <sha>:<path>` recovery
  pointer.
- **Append, never rewrite, RELEASES.md.**
