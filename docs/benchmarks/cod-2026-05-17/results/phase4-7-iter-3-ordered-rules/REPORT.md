# Phase 4.7 acceptance run (n=50) — REPORT

_Generated: 2026-05-25T15:38:54Z_

**Parent plan:** `docs/plans/atlas-dsl-poc-phase4-gpu-semantic-verifier.md` §5 Phase 4.7 (acceptance run) + `docs/plans/atlas-dsl-poc-plan.md` §7.3 (gate criteria).

## 1. Run summary

- Sampled article ids: **50** (seed=4757 from `corpus.full72.jsonl`)
- Production-pipeline enriched docs: **47** (3 parse-failed at the v15 DSL grammar level)
- Foundry-labeling label sets: **47**
- Articles with at least one CLAIM: **24**
- Per-CLAIM verdicts compared: **137** (agree=83, disagree=54, foundry-missing=0)

## 2. Acceptance gate (§7.3, ≥85% agreement)

**Result:** **FAIL** — agreement = 0.6058 (60.58%); gate = ≥85%

Gate FAILED. Per plan §5 Phase 4.7, remediation = prompt iteration on the production claim-verifier (the `BuildPrompt` template). The top disagreement classes are listed in §5 below.

## 3. Per-article fact yield (validated NUMs + catalog-grounded ENTs)

- Mean: **17.57**
- p50: **11.0**
- p95: **54.0**
- min / max: **0 / 114**
- Articles measured: **47**

**Caveat.** Per `run_phase4_7_production_pipeline.compute_fact_yield`, this is a deterministic proxy: validated NUMs = `pass` verdicts from `verifier.verify_document`; grounded ENTs = ENTs with `verbatim_name` (catalog grounding via SecMaster gRPC is out of scope for the §7.3 gate per plan §5 Phase 4.7 — see runner docstring).

Histogram (fact-yield total → article count):

| fact_yield | articles |
|---:|---:|
| 0 | 6 |
| 2 | 3 |
| 3 | 1 |
| 4 | 1 |
| 5 | 2 |
| 6 | 5 |
| 8 | 2 |
| 11 | 4 |
| 12 | 2 |
| 13 | 2 |
| 15 | 2 |
| 17 | 1 |
| 18 | 1 |
| 20 | 2 |
| 23 | 2 |
| 24 | 2 |
| 28 | 1 |
| 30 | 1 |
| 35 | 1 |
| 42 | 1 |
| 46 | 1 |
| 47 | 1 |
| 57 | 1 |
| 65 | 1 |
| 114 | 1 |

Aggregated: validated NUMs total = **480**, grounded ENTs total = **346**.

## 4. Per-CLAIM confusion matrix (production rows × Foundry cols)

| production \ foundry | full | partial | none |
|---|---|---|---|
| **full** | 0 | 2 | 0 |
| **partial** | 10 | 66 | 9 |
| **none** | 4 | 29 | 17 |

Agreement = sum of diagonal; disagreement = sum of off-diagonal.

## 5. Disagreement classes (frequency-sorted)

- **production=none / foundry=partial:** 29
- **production=partial / foundry=full:** 10
- **production=partial / foundry=none:** 9
- **production=none / foundry=full:** 4
- **production=full / foundry=partial:** 2

## 6. Parse failures

- Articles missing from production output: **3** (32859, 35352, 36050)
- Articles missing from foundry output: **3** (32859, 35352, 36050)
- CLAIMs production-verified but no Foundry label: **0**

## 7. Cost breakdown

- Production arm (vllm-server, `Qwen/Qwen2.5-32B-Instruct-AWQ`): **$0.0000** (production runtime = $0 per plan §6).
- Production-arm total wall-time across n=50: **96516 ms** (96.5 s for 137 CLAIM calls).
- Foundry (Claude 4.7) labeling arm: **$0.3222** (cap = $10.00; headroom = $9.6778).

## 8. Methodology notes

- The production pipeline runner (`run_phase4_7_production_pipeline.py`) mirrors the C# `ClaimVerifier.BuildPrompt` / `TryParseVerdict` byte-for-byte. Same prompt template, same verdict-parse rules.
- The Foundry runner (`run_phase4_7_foundry_labels.py`) uses the same prompt against Azure Foundry Claude 4.7 via the same Anthropic-compatible HTTP shape `run_phase1_4_foundry.py` uses.
- vllm-server serves the bare `Qwen/Qwen2.5-32B-Instruct-AWQ` (LoRA dropped 2026-05-17; --served-model-name alias dropped 2026-05-25). Both this Python harness and the C# `ClaimVerifier.ProductionModel` literal pin the same id; the acceptance measurement reflects what production does — same weights, same prompt, same parser.
- CLAIM block ids are synthesised `clm_NNNN` matching the C# orchestrator's prefix (`SemanticVerifierService.cs:42`), so the cross-arm join is by id.
- Entity resolution + sector tagging are NOT exercised in this acceptance run. The §7.3 gate is the claim verifier (≥85% agreement); the §7.3 fact-yield criterion is reported via a deterministic proxy (validated NUMs + non-empty ENTs) rather than the full SecMaster cascade, which would require standing up the SecMaster gRPC + OpenFIGI + Gemini stack out-of-process. See plan §5 Phase 4.7 — "Files (new — results only): REPORT.md, spot_check.jsonl, enriched_docs/*.json".

## 9. Artefact paths

- Sample ids: `results/phase4-7-acceptance/sample_ids.txt`
- Production enriched docs: `results/phase4-7-acceptance/v2_pipeline/*.json` (47 files)
- Foundry labels: `results/phase4-7-acceptance/foundry_labels/*.json` (47 files)
- Per-CLAIM spot check (long form): `results/phase4-7-acceptance/spot_check.jsonl` (137 rows)
- This report: `results/phase4-7-acceptance/REPORT.md`
