# ATLAS Releases

Tagged phase-completion snapshots. Pull the tag to recover the full set of
plans / recon docs / iteration history that was in `docs/plans/` at that
moment:

    git fetch origin --tags
    git checkout <tag> -- docs/plans/

Or inspect a single file at a tag without checkout:

    git show <tag>:docs/plans/<filename>

## Tags

### `pre-docs-consolidation-2026-05-26`
Anchor tag preserving the full `docs/plans/` tree as it existed before
consolidation. Includes every iteration-history plan, recon doc, and spike
note that's now removed from main. Use this tag to recover any of the
deleted Phase 2 / Phase 3 / recon documents.

### `dsl-poc-phase4-done` (@ 93fd94d0)
**Phase 4 GPU semantic verifier — PASS** at recalibrated ≥80% Foundry
agreement gate. 80.64% via prompt iter-2-axis-C + IsSelfReferential
short-circuit on Qwen2.5-32B-Instruct-AWQ. Recalibration rationale in
`docs/plans/atlas-dsl-poc-phase4-gpu-semantic-verifier.md §11`.

### `dsl-poc-phase5-done` (@ ff72eb8f)
**Phase 5 matrix integration — DONE.** CPU CoD cutover live
(`Extraction__Backend=LlamaServerDsl` default); §17.4 path A
`cells>0` criterion met (2 cells from 1/10 articles via FOMC 29807).
4 review findings closed (CR-IMP-1 prune blast radius narrowed,
CR-IMP-2 timeout-override coverage, OBS-1 DSL adapter dashboard
panels, OBS-2 calculator-skip alert). 1333 unit + 6 integration
tests pass; CPU throughput lifted to 6–13 t/s. Pending Phase 6:
entity_resolution_exhausted=61 / gpu_verifier_none=28 filter drops;
audit provenance `rawContentId=null`; CPU throughput borderline for
synchronous extraction (~50% timeout on long-output articles at
900s budget). Sub-plan
`atlas-dsl-poc-phase5-matrix-integration.md` pruned from main per
PHASE_TAGS; recover via `git show dsl-poc-phase5-done:docs/plans/atlas-dsl-poc-phase5-matrix-integration.md`.

### `gpu-cod-roleflip-2026-06-09` (@ 41e649b4)
**GPU-CoD role-flip — DONE.** Sentinel extraction CoD moved from CPU
llama-server (qwen3-30b-a3b) to GPU vLLM (Qwen2.5-32B-AWQ, JSON-schema
output). ~25× throughput lift (~327 art/hr vs ~20); backlog drains ~20h.
Distributed: GPU handles CoD emission + CoVe verification; CPU handles
classifier + embeddings. Loop-guard added. Recall gate 0.79 (n=80).
PRs #640 #642 #643 #644 #645 #646.

## Docs consolidation 2026-06-11 (`docs/consolidation` branch)

`docs/` re-curated to "current documentation + active plans" (index:
`docs/README.md`). Retired to git history — recover any path via
`git show <removal-commit>^:<path>` (or the tags noted):

- `docs/benchmarks/cod-2026-05-17/` — DSL PoC Phase 1–5 benchmark tree
  (~4,500 files: corpora, prompts, results, training runs, scripts).
  Contained in tags `dsl-poc-phase4-done` / `dsl-poc-phase5-done`; the
  production parser/verifier mirror is `SentinelCollector/dsl-parser-mcp/dsl/`.
- `docs/benchmarks/cod-cove-acceptance/` — CoD→CoVe acceptance harness;
  the CoVe claim verifier it validated was removed in #647.
- `docs/plans/gpu-json-cod-rollout-2026-06-09.md` — rollout DONE; tag
  `gpu-cod-roleflip-2026-06-09` contains the plan.
- `docs/plans/cpu-tuning-throughput-2026-06-08.md` +
  `docs/plans/model-task-hardware-optimization-2026-06-08.md` — CPU-era
  throughput investigations, superseded by the GPU role-flip.
- `docs/plans/cod-truncation-experiment-plan.md` +
  `docs/superpowers/specs/2026-06-07-cod-truncation-value-experiment-design.md`
  — truncation experiment mooted by the role-flip throughput win.
- `docs/plans/symbol-identification-remediation.md` — absorbed: RAG
  similarity floor + `GeminiSymbolFallbackService` shipped.
- `docs/plans/secmaster-parallel-failthrough-resolver.md` — shipped as
  #619; design context lives in SecMaster's service docs.
- `docs/superpowers/` — shipped design specs/plans: digest
  article-grounded narrative #610, news-signal feed #613, news-signal
  decay model #615.
- `docs/FRED_DATA_RESEARCH.md` — Jan-2026 research; conclusions absorbed
  into `docs/FRED_SERIES_REFERENCE.md`.
- `docs/sentinel-extraction-pipeline.md` — described the pre-DSL
  six-stage pipeline as deployed 2026-05-16; superseded by
  `SentinelCollector/README.md` + `docs/ARCHITECTURE.md` after the DSL
  cutover and GPU role-flip.

## Phase summaries

### Phase 2 — llama.cpp llama-server CPU sibling (DONE 2026-05-21, PR #387 @ 25cb33fa)
llama-server on :11437 serving qwen3-30b-a3b via bind-mounted Ollama blob;
v1.1 GBNF wired through ollama's `format` parameter; runner `--backend`
flag for portability. Outcome: working CPU GBNF substrate, foundation for
Phase 3 schema iteration. Full plan at
`pre-docs-consolidation-2026-05-26:docs/plans/atlas-dsl-poc-phase2-llama-server.md`.

### Phase 3 — v2 schema + word-grounding arc (DONE 2026-05-24)
Multi-PR iteration: v2 baseline (#390) → v2.1 no-offset (#392, #393) →
ENT-ref cleanup (#394) → v6 leaner prompt (#400) → context-aware /
RAG / leaner-prompt sweep (#404–#412) → v2.2 token-grounding shelved
(#425) → v2.3 word-grounding (#426–#428) → v15 compound chunked
(#429, #431, #432). Outcome: word-grounding v2.3.1 + chunked compound
v15 are the leading production candidates; T=0 A/B verdict
DESIGN-SOUND. Deterministic Python verifier shipped as
`docs/benchmarks/cod-2026-05-17/dsl/verifier_v2_3_1.py` (benchmark tree
retired from main 2026-06-11 — production mirror lives at
`SentinelCollector/dsl-parser-mcp/dsl/`; recover the original via
`git show dsl-poc-phase5-done:<path>`). Iteration
history at `pre-docs-consolidation-2026-05-26:docs/plans/`
(`atlas-dsl-poc-phase3-v2-schema.md`, `token-grounding-recon.md`,
`three-hypotheses-scoping.md`, `hyp-b-logits-processor-spike.md`).

### Phase 4 — GPU semantic verifier (DONE 2026-05-26, see `dsl-poc-phase4-done` tag)
Tag covers the milestone above. Canonical record (kept on main):
`docs/plans/atlas-dsl-poc-phase4-gpu-semantic-verifier.md`. The 4.7s SFT
sub-arc (`atlas-dsl-poc-phase4-7-sft-cove-verifier.md`) is now subsumed
by §11 of the canonical record; the full sub-plan is preserved at
`pre-docs-consolidation-2026-05-26:docs/plans/atlas-dsl-poc-phase4-7-sft-cove-verifier.md`.

### Phase 5 — Matrix integration (DONE 2026-05-28, see `dsl-poc-phase5-done` tag)
PR #510 merged at `ff72eb8f`. CPU CoD cutover live; §17.4 path A
`cells>0` met. Iteration history at
`dsl-poc-phase5-done:docs/plans/atlas-dsl-poc-phase5-matrix-integration.md`.
Parent plan `docs/plans/atlas-dsl-poc-plan.md` remains active for
Phase 6.

## Going forward

Per `CLAUDE.md ## PHASE_TAGS`: tag at phase completion
(`<workstream>-phase<N>-done`), add an entry to this file, and delete
phase-specific iteration-history docs from `docs/plans/`. Reference
docs (architecture, specs, runbooks, schemas, grammars) stay on main.
