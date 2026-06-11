# docs/ — Index

`docs/` holds **current system documentation** and **active plans** — not iteration
history. Per `CLAUDE.md ## PHASE_TAGS`: git log is the archive, tags are the index
(see [RELEASES.md](./RELEASES.md) for the tag catalog and recovery commands).
Dated explorations, superseded specs, and benchmark write-ups whose conclusions are
absorbed elsewhere get `git rm`'d once their phase completes — recover them via
`git show <tag-or-commit>:<path>`.

Per-service architecture, API, and data-model documentation lives with each service
(`<Service>/README.md` + the read-first `<Service>/AGENT_README.md` card), not here.

## System reference

| Doc | What it is |
|-----|------------|
| [ARCHITECTURE.md](./ARCHITECTURE.md) | Service topology, port allocation, event flow, inference topology, regime states |
| [EXECUTIVE-SUMMARY.md](./EXECUTIVE-SUMMARY.md) | Platform purpose, service status table, pattern library summary |
| [GRPC-ARCHITECTURE.md](./GRPC-ARCHITECTURE.md) | gRPC streaming patterns and contracts (collectors → ThresholdEngine / SecMaster) |
| [OBSERVABILITY.md](./OBSERVABILITY.md) | OTEL pipeline, dashboards, tracing/metric conventions, alerting |
| [THRESHOLDENGINE-PATTERNS.md](./THRESHOLDENGINE-PATTERNS.md) | Pattern expression API, weighting, freshness decay, configuration reference |
| [SENTINEL-RLM.md](./SENTINEL-RLM.md) | Sentinel extraction model/backend/VRAM constraints + troubleshooting |
| [f4.6.4-prepass-rollout.md](./f4.6.4-prepass-rollout.md) | Entity-resolution pre-pass operations reference (spaCy NER → SecMaster composer) |
| [FRED_SERIES_REFERENCE.md](./FRED_SERIES_REFERENCE.md) | ATLAS indicator → FRED series mapping (reference data) |
| [devcontainer-db-bridge.md](./devcontainer-db-bridge.md) | Devcontainer ↔ TimescaleDB bridge runbook |
| [Guide to Grafana Dashboards.md](./Guide%20to%20Grafana%20Dashboards.md) | Dashboard JSON authoring guide (LLM-oriented) |
| [RELEASES.md](./RELEASES.md) | Phase/epic outcomes paired with git tags — the recovery index for retired docs |

## Matrix (reference set)

Each doc has a distinct role; read in this order for onboarding:

| Doc | What it is |
|-----|------------|
| [atlas-matrix-handoff-v2.md](./atlas-matrix-handoff-v2.md) | Philosophy, framing, architectural contract (bootstrap doc; referenced by STATE.md and service docs) |
| [atlas-matrix-mvp-plan.md](./atlas-matrix-mvp-plan.md) | Canonical product plan — features, stories, acceptance criteria, decisions D1–D17 |
| [atlas-matrix-realignment-brief.md](./atlas-matrix-realignment-brief.md) | Plain-English brief: matrix mechanics, cell formula (reference; superseded as execution driver by the WS3 plan) |
| [atlas-sectorweights-methodology.md](./atlas-sectorweights-methodology.md) | `sectorWeights` 7-level signed scale, sign convention, starter vectors |
| [atlas-matrix-backtesting-spec.md](./atlas-matrix-backtesting-spec.md) | Backtesting/calibration harness design (implemented under `backtest/`) |

## Sentinel (reference set)

| Doc | What it is |
|-----|------------|
| [sentinel-product-spec-v2.md](./sentinel-product-spec-v2.md) | Product-level requirements and LLM do/don't boundaries. **Vision-level; pending product-owner review (flagged 2026-06-11) — treat requirements/LLM-boundary sections as candidate-governing, not confirmed-current.** "Implementation Status" section is a December-2025 snapshot — current pipeline state lives in `SentinelCollector/README.md` + `STATE.md` |
| [grammars/](./grammars/) | CoD DSL GBNF grammars v1–v2.3 (CPU `llama-server` rollback path; production mirrors: Lark parser grammars at `SentinelCollector/dsl-parser-mcp/dsl/` AND byte-identical runtime GBNF rollback copy at `SentinelCollector/src/cod-prompts/cod-dsl-v2.3.gbnf`) |
| [llm/ACCEPTANCE_CRITERIA.md](./llm/ACCEPTANCE_CRITERIA.md) | Ratified (D15) LLM extraction-reliability acceptance thresholds |
| [research/dsl-prior-art.md](./research/dsl-prior-art.md) | Literature survey behind the DSL extraction approach (durable references) |

The current extraction pipeline (GPU vLLM JSON-CoD, `Extraction__Backend=VllmJson`)
is documented in `SentinelCollector/README.md` and `docs/ARCHITECTURE.md`.

## Active plans (`plans/`)

| Doc | Status |
|-----|--------|
| [plans/atlas-ws3-completion-plan.md](./plans/atlas-ws3-completion-plan.md) | Active execution driver — WS3 matrix realignment completion |
| [plans/atlas-dsl-poc-plan.md](./plans/atlas-dsl-poc-plan.md) | Parent DSL PoC plan — Phases 0–5 done; remains active for Phase 6 |
| [plans/atlas-dsl-poc-phase4-gpu-semantic-verifier.md](./plans/atlas-dsl-poc-phase4-gpu-semantic-verifier.md) | Canonical Phase 4 record (kept on main per RELEASES.md; the production verifier it shipped was later removed as dead code, PR #647) |
| [plans/sentinel-edge-realization.md](./plans/sentinel-edge-realization.md) | Sentinel architecture planning doc — canonical CoD-DSL spec context |
| [plans/atlas-matrix/](./plans/atlas-matrix/) | Matrix MVP plan index — Epic 6 (consumption surfaces) is the remaining live epic plan |

## README templates

[README-TEMPLATE.md](./README-TEMPLATE.md) is the entry point and routes by project
archetype: [EDGE](./README-TEMPLATE-EDGE.md), [INDEX](./README-TEMPLATE-INDEX.md),
[INFRA](./README-TEMPLATE-INFRA.md), [LIBRARY](./README-TEMPLATE-LIBRARY.md),
[MCP](./README-TEMPLATE-MCP.md), [SIDECAR](./README-TEMPLATE-SIDECAR.md).

## Retired (recover via git history)

The 2026-06-11 consolidation (`docs/consolidation` branch) retired the CoD benchmark
trees (`benchmarks/cod-2026-05-17/`, `benchmarks/cod-cove-acceptance/`), completed
rollout/investigation plans (GPU-JSON-CoD rollout, CPU tuning, model×task×hardware,
CoD truncation, symbol-identification remediation, SecMaster parallel fail-through),
shipped design specs (`superpowers/`), `FRED_DATA_RESEARCH.md`, and the superseded
`sentinel-extraction-pipeline.md`. See the RELEASES.md entry for the full list and
recovery commands.
