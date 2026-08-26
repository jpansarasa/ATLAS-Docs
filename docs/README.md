# docs/ — Index

`docs/` holds **current-state system documentation** — what is implemented and running — at the
top level, plus **live working documents** in [proposals/](./proposals/) and
[superpowers/](./superpowers/). Iteration history lives in git: completed phases are tagged per
`CLAUDE.md ## PHASE_TAGS`, and [RELEASES.md](./RELEASES.md) is the named index (each entry carries
the recovery command for the docs retired with it). See §Curation policy for what earns a place
here and what gets retired.

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
| [BACKLOG.md](./BACKLOG.md) | Known defects, measurement debt, deferred work, parked epics — what outlives the epic that found it. Exists so `STATE.md` can stay disposable working memory; close entries in the PR that fixes them |

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

## Working documents

| Dir | What lives there |
|---|---|
| [proposals/](./proposals/) | Plans and specs under active work, plus permanent negative decision records |
| [superpowers/](./superpowers/) | Design specs and plans produced by the `superpowers` skills, same lifecycle |

Current contents:

| Doc | Kind |
|---|---|
| [proposals/news-pipeline-remediation-plan.md](./proposals/news-pipeline-remediation-plan.md) | Build plan — numeric-extraction contract break, SecMaster name self-seeding, catalog repair |
| [proposals/extraction-type-classifier.md](./proposals/extraction-type-classifier.md) | **Decision record — DO-NOT-BUILD.** Permanent (see below) |
| [proposals/regime-news-staleness-redesign.md](./proposals/regime-news-staleness-redesign.md) | Design spec — regime news-as-staleness-perturbation |
| [proposals/okf-rag-feasibility-spike.md](./proposals/okf-rag-feasibility-spike.md) | Feasibility spike — OKF / LLM-wiki RAG |
| [proposals/push-gate-intent-redesign.md](./proposals/push-gate-intent-redesign.md) | Design spec — push/merge gate rebuilt around the act boundary, not parser accuracy |
| [proposals/extraction-identity-remediation.md](./proposals/extraction-identity-remediation.md) | Build plan — sector-event v1/v2 contract break, observation identity = measurement not entity, ClaimKind loss |
| [proposals/extraction-identity-implementation.md](./proposals/extraction-identity-implementation.md) | Implementation plan — dispatch-ready stories S0-S5 for the above, with acceptance queries |
| [superpowers/plans/2026-07-04-sentinel-trend-visualization.md](./superpowers/plans/2026-07-04-sentinel-trend-visualization.md) | Plan — shipped (`sentinel-trend-viz-done`) |
| [superpowers/specs/2026-07-04-sentinel-trend-visualization-design.md](./superpowers/specs/2026-07-04-sentinel-trend-visualization-design.md) | Design spec — shipped |
| [superpowers/plans/2026-07-05-sentinel-review-queue-drain.md](./superpowers/plans/2026-07-05-sentinel-review-queue-drain.md) | Plan — shipped (`sentinel-review-queue-drain-done`) |

## Curation policy

- **Current-state only, at the top level.** A doc in `docs/` itself must describe the system as it
  runs. If the implementation moves, the doc moves with it or gets retired.
- **Plans and specs merge to `main`, then retire.** They live in `proposals/` or `superpowers/`
  while the work is live, and are removed via `git rm` at completion; RELEASES.md records each
  retirement with a `git show <tag>:<path>` recovery pointer.
  `proposals/intent-fidelity-enforcement.md` is the worked example — merged at `fb86b169`, retired
  2026-07-02, recoverable per [RELEASES.md](./RELEASES.md) (`intent-fidelity-done`).
  **Merging a plan is not approving it.** Each doc's own `Status` line governs approval; review
  happens in the PR, and a plan can sit merged and unapproved indefinitely.
- **Negative decision records are permanent.** A spec whose outcome is DO-NOT-BUILD never completes
  a phase, so the retirement trigger above never fires for it. Retiring it on merge would be a
  no-op, and retiring it later would delete the only artefact the work produced: an implemented
  plan is survived by the code and by the current-state docs, but a rejected one is survived by
  nothing. Losing it means the next person with the same idea cannot find out it was already
  measured and declined — they would have to know a tag name to `git show`. So these stay,
  ADR-shaped: `Status`, recommendation, the measurements behind it, and what would have to change
  to reopen the question. They are current state — "we are not building this, and here is why" is a
  standing fact about the system, not iteration history.
  *(Considered and rejected: making "decision recorded" a retirement trigger. It satisfies the
  letter of PHASE_TAGS and defeats the document's only purpose.)*
- **Everything in these directories is indexed above.** A working doc that is not in the table is
  invisible; add the row in the PR that adds the doc.
- **Append, never rewrite, RELEASES.md.**

> Supersedes the previous rule "No plans/specs/iteration docs on main", which contradicted both
> [RELEASES.md](./RELEASES.md) §Going forward ("Reference docs (architecture, specs, runbooks,
> schemas, grammars) stay on main") and the repo's actual practice — `proposals/` and
> `superpowers/` have carried merged plans and specs for months, and RELEASES.md itself records
> two of them as "Design docs (on main)". The old wording is why merged plans were shipping with
> notes inside them claiming they were "not merged".
> `CLAUDE.md ## PHASE_TAGS` was reconciled to match in the same change; this file remains the
> authority it defers to, so keep the two consistent.
