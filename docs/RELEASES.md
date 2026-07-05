# ATLAS Releases

Tagged phase-completion snapshots. Pull the tag to recover the full set of
plans / recon docs / iteration history that was in `docs/plans/` at that
moment:

    git fetch origin --tags
    git checkout <tag> -- docs/plans/

Or inspect a single file at a tag without checkout:

    git show <tag>:docs/plans/<filename>

## Tags

### `sentinel-trend-viz-done` (@ a2141c54)
**Sentinel digest trend-visualization redesign — SHIPPED** (2026-07-04). Replaced the points-in-time digest with a market-first, trend-over-time design (user ask: the reports were "always points in time... hard to see trend"). Delivered: (1) redesigned digest — KPI tiles → sector×day heatmap index → per-sector `<details>` drill-downs (net-lean sparkline + regime-timeline strip + top-5 per-signal dual-trend [matrix cell vs daily news tilt]) → Executive Take; (2) live `GET /sentinel/matrix?range={7d|30d|90d|max}` explorer reusing the same blocks; (3) "News Signal Momentum" rebuilt as server-rendered SVG (bar color = tilt SIGN matching the heatmap, momentum direction moved to a separate ↗/↘/→ arrow, Chart.js/CDN removed). Server-rendered inline SVG, zero client JS, native `<details>` drill-down; hot=red / cold=blue, CVD-safe. Cutover removed the old Sector Heat markdown table + Sector Detail lists + raw extracted-quantity bullets + the "Sector Watch" narrative (host-mounted `/prompts`). SentinelCollector-only, read-only over existing tables (matrix_cells / sector_regimes / macro_observations); no schema/projector/write-path change.
5 PRs: **#841** Stage A (TrendQueryService + TrendSvgRenderer + tests), **#842** Stage B (market-first DigestRenderer + `/sentinel/matrix` endpoint), **#843** drill-down nesting fix (user-caught empty-body bug — rows opened to nothing), **#844** momentum SVG redesign (user-caught confusing green/red dual-encoding), **#845** Stage C cutover. All merged + deployed + verified; live at digest report 185. Design docs (on main): `docs/superpowers/specs/2026-07-04-sentinel-trend-visualization-design.md` + `docs/superpowers/plans/2026-07-04-sentinel-trend-visualization.md`. Recover via `git show sentinel-trend-viz-done:<path>`. Notable: the two user live-review catches (empty drill-downs, confusing momentum) both passed automated tests+smoke — the fix PRs added behavior-level guard tests (nesting-inside-details, glyph-tracks-sign) that go RED on the actual regression.

### `okf-rag-spike-done` (@ d519fcf1, NEGATIVE RESULT — not merged)
**OKF / LLM-wiki RAG feasibility spike — NO-GO as a subsystem** (2026-07-02).
Investigated whether the Karpathy LLM-wiki / Google Open-Knowledge-Format pattern
(LLM curates a compounding folder of cross-linked markdown) should augment or replace
ATLAS RAG. Verdict: no. The wiki is squeezed out from both sides — grounded/structural
retrieval is done better, cheaper, and more context-efficiently by the existing
bge-m3 + pgvector RAG (the 7b arm truncated 8/14 ingests because whole-page YAML reads
exhaust its real 2048-token slot under `--parallel 4` — the context cost a targeted
vector retrieval avoids); and news alpha is fast-decaying and must NOT accumulate into a
compounding store (reinforcement bias) — it belongs in the numeric benchmark-staleness
pipeline (#729). The 32b "GO" only cleared quality bars by burning 32K context, which is
orthogonal to the small-model/context-economy goal. Structural finding worth keeping: the
video's "LLMs mangle markdown at scale" skepticism was refuted for both arms (0 invented
links / 28 ingests) — format was never the problem; fit was. Keepers: the adversarial
eval-gate harness (caught a rigged eval-set pre-inference) + the measured local 2048-token
slot reality. Baseline `pre-okf-rag-investigation` @ c558bb52 was the revert point; main
was never touched, so no revert was needed. Full artifacts recoverable:
`git checkout okf-rag-spike-done`.

### `intent-fidelity-done` (@ 84569aac)
**Intent-fidelity enforcement epic — COMPLETE** (shipped 2026-07-02).
D-entry decision records (AGENT_README `DECISIONS` blocks, atomic with
INTENT(D-n) comments + guards + RED-capable guard tests) for
SentinelCollector, ThresholdEngine, and SecMaster; enforcement hooks
(dispatch-guard BLOCK / decisions-injector ADVISE / plan-retirement ASK)
plus the testing-context outbound-boundary line; `intent-review` skill
wired into REVIEW_FIX_LOOP and the deploy skill; CLAUDE.md MECHANICS.
PRs #826–#830. Spec doc `docs/proposals/intent-fidelity-enforcement.md`
retired 2026-07-02 (human-confirmed through the plan-retirement-guard's
first ASK, after verifying all decisions migrated to the 13 D-entries +
CLAUDE.md MECHANICS + intent-review GUARD_TEST_CONTRACT). Recover the
spec via `git show intent-fidelity-done:docs/proposals/intent-fidelity-enforcement.md`.

### `digest-matrix-redesign-done` (@ b8dcd2a2)
**Matrix-first digest redesign — COMPLETE.** The Sentinel digest is now
organized by the signal x sector matrix instead of the 7-theme taxonomy:
Sector Heat strip + per-sector Detail blocks (net tilt, regime, top signals
cross-referenced with news momentum, cited articles), articles grounded to
sectors via their `:sig:` rows' `atlas_sector_code`, narrative prompt
rewritten (Executive Take / Sector Watch / Cross-Sector Signals /
Noteworthy One-Offs). ThresholdEngine `SectorRegimeProjectionWorker` now
publishes SectorScoreEvents so `sector_regimes` populates (first rows ever,
2026-06-12). ThemeClassifier/DigestTheme deleted; Grafana theme panel ->
sector panel. PRs #693 (matrix read + heat), #695 (sector grounding),
#697 (sector skeleton), #698 (deletion sweep), #694 (regime publisher).
Plan retired from `docs/plans/digest-matrix-redesign.md` — recover via
`git show digest-matrix-redesign-done:docs/plans/digest-matrix-redesign.md`.


### `pre-docs-consolidation-2026-05-26`
Anchor tag preserving the full `docs/plans/` tree as it existed before
consolidation. Includes every iteration-history plan, recon doc, and spike
note that's now removed from main. Use this tag to recover any of the
deleted Phase 2 / Phase 3 / recon documents.

### `dsl-poc-phase4-done` (@ 93fd94d0)
**Phase 4 GPU semantic verifier — PASS** at recalibrated >=80% Foundry
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
output). ~25x throughput lift (~327 art/hr vs ~20); backlog drains ~20h.
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
- `docs/benchmarks/cod-cove-acceptance/` — CoD->CoVe acceptance harness;
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
Multi-PR iteration: v2 baseline (#390) -> v2.1 no-offset (#392, #393) ->
ENT-ref cleanup (#394) -> v6 leaner prompt (#400) -> context-aware /
RAG / leaner-prompt sweep (#404–#412) -> v2.2 token-grounding shelved
(#425) -> v2.3 word-grounding (#426–#428) -> v15 compound chunked
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

## Docs current-state rebuild (2026-06-11)

The documentation set was rewritten to describe the system **as implemented**
(code-grounded briefs, live config, live DB — see `docs/README.md` for the
curation policy). The following were retired from main in the same change;
recover any of them via `git show 5cf265f7:<path>` (the last commit that
contains them all):

- `docs/atlas-matrix-mvp-plan.md` — matrix MVP plan (superseded by `docs/MATRIX.md`)
- `docs/atlas-matrix-handoff-v2.md` — matrix handoff brief (superseded by `docs/MATRIX.md`)
- `docs/atlas-matrix-realignment-brief.md` — realignment brief (superseded by `docs/MATRIX.md`)
- `docs/atlas-matrix-backtesting-spec.md` — backtesting spec (the `backtest/` tooling it
  specified remains in-repo; its READMEs carry self-contained summaries)
- `docs/f4.6.4-prepass-rollout.md` — prepass rollout working doc (prepass is live;
  current behavior documented in `docs/ARCHITECTURE.md` §5 and `SentinelCollector/README.md`)
- `docs/sentinel-product-spec-v2.md` — vision-level product spec (current behavior in
  `docs/ARCHITECTURE.md` + `SentinelCollector/README.md`)
- `docs/plans/` (entire tree): `atlas-dsl-poc-plan.md`,
  `atlas-dsl-poc-phase4-gpu-semantic-verifier.md`, `atlas-ws3-completion-plan.md`,
  `sentinel-edge-realization.md`, `atlas-matrix/README.md`,
  `atlas-matrix/epic-6-consumption-surfaces.md` — all phases complete; outcomes
  recorded above and tagged per PHASE_TAGS
- `docs/research/dsl-prior-art.md` — research notes (DSL era closed)
- `docs/llm/ACCEPTANCE_CRITERIA.md` — LoRA-era ratified criteria; the pinned copy
  (`SentinelCollector/training/acceptance_criteria.json`, sha-locked) remains the
  machine-readable record
- `docs/grammars/` (cod-dsl v1–v2.3 GBNF): the production grammar mirror lives at
  `SentinelCollector/src/cod-prompts/cod-dsl-v2.3.gbnf` (the CPU rollback backend);
  earlier grammar versions are history
- `docs/README-TEMPLATE*.md` (7 files) — README templates; the canonical copies used by
  tooling live in `.claude/skills/readme-consistency/` (`TEMPLATE.md`,
  `TEMPLATE_CONFIG.md`, `TEMPLATE_SCRIPTS.md`)
