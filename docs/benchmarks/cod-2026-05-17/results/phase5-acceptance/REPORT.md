# Phase 5.8 Acceptance Demo — Run #4 (CPU CoD cutover, PR 6/6)

**Date:** 2026-05-27 (UTC)
**Branch:** `worktree-agent-a2ee141ba5897fc09`
**Pipeline under test:** CPU CoD migration end-to-end. `Extraction__Backend=LlamaServerDsl`
routes `IMergedExtractionService` to `CpuDslExtractionService` (PR #509), which
talks llama-server (CPU GBNF-constrained) + dsl-parser-mcp (PR #507) + the v8 baseline
prompt + v2.3 GBNF grammar mounted from `/opt/ai-inference/prompts/cod`.

**Cutover commits:**
- `Extraction__Backend=VllmServer` -> `LlamaServerDsl`
- Added: `Extraction__LlamaServerEndpoint=http://llama-server:8080`,
  `CpuCod__DslParserEndpoint=http://dsl-parser-mcp:3120`,
  `CpuCod__PromptDirectory=/prompts/cod`, `CpuCod__PromptFile=cod_dsl_v8_baseline.txt`,
  `CpuCod__GrammarFile=cod-dsl-v2.3.gbnf`
- Added bind mount `/opt/ai-inference/prompts/cod:/prompts/cod` on sentinel-collector
  (fix for missing mount caught in run #4a — see below).

**ZFS snapshots:** `pre-deploy-20260527T182616`, `pre-deploy-20260527T190348`.

**Demo reset_at:** `2026-05-27T23:05:09Z`. Hard timeout at `t+1502s` (25 min).

## VERDICT: FAIL (with PROGRESS signal) — 0/10 articles produced matrix_cells

Spec coverage bands: PASS >= 8/10 - PARTIAL >= 5/10 - PROGRESS 1-4/10 - **FAIL 0/10**.

This is NOT a regression of the architectural change. The CPU CoD pipeline is wired
end-to-end and producing extractions; the new `sentinel_extraction_outcome_total{outcome="success"}=7`
series proves PR #509 is on the hot path and PR #507's dsl-parser sidecar got
`200 OK` on both `/parse` and `/verify` against live data. The 0-cell outcome is
a downstream-grounding shortfall on the v8-baseline DSL shape, not a pipeline
failure.

Comparison to PR #499 (run #3, GPU V2 path) baseline of 3/10 cells: pipeline
swap from GPU vLLM-V2 to CPU LlamaServerDsl loses the 3-cell macro_signal_identity
grounding wins until the DSL extractor learns to populate `source_entity` and
upstream resolver-bag context.

## Run #4a (cutover #1) — prompt mount missing — repaired

First deploy completed but the bind mount for `/opt/ai-inference/prompts/cod` was
absent. All 10 articles hard-failed with `processing_error = "CPU CoD prompt file
not found at /prompts/cod/cod_dsl_v8_baseline.txt. Verify the /prompts/cod host
mount is wired and PR 2/6 ran."` — a textbook example of the error-message
contract added in PR #509 paying for itself. Fix: added the mount line to
`compose.yaml.j2`, re-deployed.

## Per-article verdict (run #4b — corrected mount)

| #  | id    | source                  | observations | state    | err | cells | verdict |
|----|-------|-------------------------|--------------|----------|-----|-------|---------|
| 1  | 27560 | fed-press-monetary-full | 1            | done     | f   | 0     | miss    |
| 2  | 27702 | rss                     | 0            | done     | f   | 0     | miss (extractor empty) |
| 3  | 27772 | rss                     | 9            | done     | f   | 0     | miss    |
| 4  | 29807 | rss                     | 0            | TIMEOUT  | f   | 0     | timeout |
| 5  | 31149 | rss                     | 3            | done     | f   | 0     | miss    |
| 6  | 31430 | searxng-content         | 0            | TIMEOUT  | f   | 0     | timeout |
| 7  | 33632 | rss                     | 0            | TIMEOUT  | f   | 0     | timeout |
| 8  | 34537 | fed-speeches-full       | 0            | done     | f   | 0     | miss (extractor empty) |
| 9  | 35352 | tsa-checkpoint          | 111          | done     | f   | 0     | miss    |
| 10 | 36065 | searxng-content         | 28           | done     | f   | 0     | miss    |

- 7/10 articles processed end-to-end inside the 25-min budget; 3 still in flight when the demo's hard timeout fired.
- 0/10 `processing_error` rows — every completed extraction is structurally valid.
- 5/10 produced extracted_observations (152 total observations across the run).
- 0/10 produced matrix_cells.

## Per-source delta — which stage caught wins

Prometheus `sentinel_entity_resolver_stage_outcome_total` deltas (post-cutover totals):

| Stage                   | Outcome   | count |
|-------------------------|-----------|-------|
| `macro_signal_identity` | grounded  | **1** |
| `macro_signal_identity` | no_match  | 40    |
| `secmaster_composer`    | no_match  | 41    |
| `secmaster_cove`        | no_match  | 41    |
| `openfigi`              | no_match  | 41    |
| `gemini_fallback`       | no_match  | 40    |

`sentinel_matrix_filter_dropped_total{reason="entity_resolution_exhausted"}=30`;
`sentinel_matrix_filter_dropped_total{reason="gpu_verifier_none"}=11`.

**Interpretation:** 1 grounded resolver hit but 0 cells survived to publish. The
`gpu_verifier_none` count of 11 explains where the grounded observation died —
the GPU verifier was bypassed on the CPU CoD path (no Phase 7 GeminiFallback handler
was wired through the DSL adapter in PR #508), so even the one grounded result
got dropped before cell write.

## CPU CoD vs GPU V2 per-stage comparison

| Metric                                        | PR #499 (GPU V2)         | PR 6/6 (CPU CoD)         |
|-----------------------------------------------|--------------------------|--------------------------|
| Wall-clock for 10 articles                    | ~101 s                   | ~1502 s + 3 in flight    |
| Articles completed                            | 10/10                    | 7/10                     |
| Articles with observations                    | 10/10                    | 5/10                     |
| Total observations across set                 | 27                       | 152                      |
| `macro_signal_identity` grounded              | 3                        | 1                        |
| `entity_resolution_exhausted` drops           | 24                       | 30                       |
| `gpu_verifier_none` drops                     | n/a (V2 always verified) | 11                       |
| matrix_cells produced                         | 4 (from 3 articles)      | 0                        |

The CPU path produced ~5.6x more observations per article than the GPU path (15.2
vs 2.7 avg) — the v8 DSL grammar enumerates every measurable token in the source
much more aggressively than the v7 master prompt — but the per-observation
`source_entity` field is consistently empty in the DSL emission, blocking the
macro_signal_identity catalog lookup that drove all 3 PR #499 wins.

## sentinel_extraction_outcome_total (PR #509 new metric)

| Outcome           | count | notes                                                                 |
|-------------------|-------|-----------------------------------------------------------------------|
| `success`         | 7     | Matches DB-confirmed 7 completed extractions                          |
| `verify_failure`  | 4     | dsl-parser /verify returned !ok; observations dropped before persist  |
| `exception`       | 17    | Carryover from run #4a (mount-missing); FileNotFound exceptions       |

The metric registry is functional. The 7 `success` rows + 4 `verify_failure`
events confirm both halves of the dsl-parser sidecar (PR #507) are being
exercised against live data.

## Root-cause next steps (NOT addressed in this PR; supervisor decides)

1. **DSL adapter (PR #508) does not populate `source_entity`** on the
   `ExtractedObservation` projection. The v8 DSL emits `MetricCells` with
   `value` + `unit` + `period` but no producer entity — needed by
   `macro_signal_identity` resolver.
2. **Phase 7 GeminiFallback handler not wired to CPU CoD path**, so all 11
   resolver-exhausted-but-could-be-grounded observations dropped as
   `gpu_verifier_none` rather than being routed to the Gemini grounding pass.
3. **CPU latency is ~3 min/article**, not the ~60s/article projected in the
   plan §3.3. Three articles never finished inside the 25-min wall.
4. **v8 baseline emits ~5.6x more observations than v7 master prompt** —
   under-pruning is more expensive in resolver work than the over-pruning the
   old path showed.

## Rollback path

1. `git revert <cutover commit>`
2. `cd /home/james/ATLAS/deployment/ansible && ansible-playbook playbooks/deploy.yml --tags sentinel-collector`

ZFS snapshot rollback also available (`pre-deploy-20260527T182616`).

## Reference

Plan task: `a1d91968baa05dccc`. Comparison baseline: PR #499 (run #3). Predecessor
PRs in this migration: #505, #506, #507, #508, #509.
