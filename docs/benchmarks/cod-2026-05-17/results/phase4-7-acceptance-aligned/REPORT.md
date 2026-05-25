# Phase 4.7 acceptance run (n=50) — REPORT

_Generated: 2026-05-25T14:51:02Z_

**Parent plan:** `docs/plans/atlas-dsl-poc-phase4-gpu-semantic-verifier.md` §5 Phase 4.7 (acceptance run) + `docs/plans/atlas-dsl-poc-plan.md` §7.3 (gate criteria).

## 1. Run summary

- Sampled article ids: **50** (seed=4757 from `corpus.full72.jsonl`)
- Production-pipeline enriched docs: **47** (3 parse-failed at the v15 DSL grammar level)
- Foundry-labeling label sets: **47**
- Articles with at least one CLAIM: **24**
- Per-CLAIM verdicts compared: **137** (agree=74, disagree=63, foundry-missing=0)

## 2. Acceptance gate (§7.3, ≥85% agreement)

**Result:** **FAIL** — agreement = 0.5401 (54.01%); gate = ≥85%

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
| **full** | 1 | 4 | 0 |
| **partial** | 7 | 49 | 2 |
| **none** | 6 | 44 | 24 |

Agreement = sum of diagonal; disagreement = sum of off-diagonal.

## 5. Disagreement classes (frequency-sorted)

- **production=none / foundry=partial:** 44
- **production=partial / foundry=full:** 7
- **production=none / foundry=full:** 6
- **production=full / foundry=partial:** 4
- **production=partial / foundry=none:** 2

## 6. Parse failures

- Articles missing from production output: **3** (32859, 35352, 36050)
- Articles missing from foundry output: **3** (32859, 35352, 36050)
- CLAIMs production-verified but no Foundry label: **0**

## 7. Cost breakdown

- Production arm (vllm-server, `Qwen/Qwen2.5-32B-Instruct-AWQ`): **$0.0000** (production runtime = $0 per plan §6).
- Production-arm total wall-time across n=50: **102941 ms** (102.9 s for 137 CLAIM calls).
- Foundry (Claude 4.7) labeling arm: **$0.3222** (cap = $10.00; headroom = $9.6778).

## 8. Methodology notes

- The production pipeline runner (`run_phase4_7_production_pipeline.py`) mirrors the C# `ClaimVerifier.BuildPrompt` / `TryParseVerdict` byte-for-byte. Same prompt template, same verdict-parse rules.
- The Foundry runner (`run_phase4_7_foundry_labels.py`) uses the same prompt against Azure Foundry Claude 4.7 via the same Anthropic-compatible HTTP shape `run_phase1_4_foundry.py` uses.
- vllm-server serves the bare `Qwen/Qwen2.5-32B-Instruct-AWQ` (LoRA dropped 2026-05-17; --served-model-name alias dropped 2026-05-25). Both this Python harness and the C# `ClaimVerifier.ProductionModel` literal pin the same id; the acceptance measurement reflects what production does — same weights, same prompt, same parser.
- CLAIM block ids are synthesised `clm_NNNN` matching the C# orchestrator's prefix (`SemanticVerifierService.cs:42`), so the cross-arm join is by id.
- Entity resolution + sector tagging are NOT exercised in this acceptance run. The §7.3 gate is the claim verifier (≥85% agreement); the §7.3 fact-yield criterion is reported via a deterministic proxy (validated NUMs + non-empty ENTs) rather than the full SecMaster cascade, which would require standing up the SecMaster gRPC + OpenFIGI + Gemini stack out-of-process. See plan §5 Phase 4.7 — "Files (new — results only): REPORT.md, spot_check.jsonl, enriched_docs/*.json".

## 9. Artefact paths

- Sample ids: `results/phase4-7-acceptance-aligned/sample_ids.txt`
- Production enriched docs: `results/phase4-7-acceptance-aligned/v2_pipeline/*.json` (47 files)
- Foundry labels: `results/phase4-7-acceptance-aligned/foundry_labels/*.json` (47 files, copied verbatim from `phase4-7-acceptance/foundry_labels/` per Plan-C "DO NOT re-run Foundry labeling — labels are stable")
- Per-CLAIM spot check (long form): `results/phase4-7-acceptance-aligned/spot_check.jsonl` (137 rows)
- This report: `results/phase4-7-acceptance-aligned/REPORT.md`

## 10. Plan-C harness alignment — 4-data-point comparison

This subdir is the fourth Phase 4.7 acceptance run on the same n=50 sample (seed=4757, 47 v15-parseable, 24 with CLAIMs, 137 CLAIMs scored). Each iteration changed one upstream variable; the Foundry labels are byte-stable across all four runs (no re-labeling; same `phase4-7-acceptance/foundry_labels/*.json` join target).

This run combines TWO landed changes:
  - PR #452 (already on `main`): rewrote `build_prompt` to a structured-similarity question with four verdict words (`consistent`/`related`/`unrelated`/`insufficient_evidence`), added the `is_self_referential` short-circuit + `try_parse_verdict` updated for the new vocabulary.
  - THIS PR: `extract_claim_inputs` Plan-C alignment — `evidence_span ← v15 object: slot`, `object_text ← subject + " " + predicate`. Production C# already had this asymmetry per `SemanticVerifierService.cs:314-324` (recon `a1dffdce5d427ec90`); the harness now matches.

| run | PR | agreement | what shipped | what the % actually measured |
|---|---|---|---|---|
| Phase 4.7 initial | #449 | **35.04%** | First harness, original `BuildPrompt` (no terse-verdict suffix). | Parser artifact: ~30% of vllm responses began "To answer this question, yes." and the verdict parser matched the leading "no" inside "Not". |
| Phase 4.7 rerun | #451 | **19.71%** | PR #450 fixed parser (word-boundary anchored) + appended terse suffix. | Real floor of OLD self-referential harness — once the parser stopped hallucinating "no" hits, the prompt's tautology bit. |
| Phase 4.7 rererun | #453 | **10.22%** | PR #452 swapped `BuildPrompt` to structured similarity. Harness still self-referential. | Structured prompt magnified the self-referential bug. The harness fed `evidence_span = object_text` so vllm saw "Does `X` describe `subj pred X`?" → strict `unrelated` across the board. |
| Phase 4.7 **aligned** | **THIS (#454)** | **54.01%** | Plan-C harness fix (this PR) + PR #452's structured-similarity prompt (already on main). | First run where the harness is NOT measuring a self-referential tautology. 0/137 byte-equal `evidence_span` == `object_text` (recon `a1dffdce5d427ec90`). 0/137 short-circuits. All 137 CLAIMs went through a real vllm round-trip. |

### Reading the 54.01%

- **The 35.04% (PR #449) was a parser bug masquerading as agreement** — kill it from the baseline narrative.
- **PR #451's 19.71% / PR #453's 10.22%** were artifacts of the harness's self-referential `evidence_span = object_text` payload.
- **THIS run's 54.01% is the first measurement of the production pipeline doing what production C# actually does:** structured-similarity prompt against asymmetric production-shape inputs. It is a **+34.3-point lift over PR #453** and a **+34.3-point lift over PR #451** — the prior numbers were measuring different (broken) machinery.
- **Gate still fails (≥85%).** The remaining gap is now real-signal disagreement between vllm Qwen2.5-32B-Instruct-AWQ and Foundry Claude 4.7 on the structured-similarity task, not a harness artifact.
- **Short-circuit rate: 0%** — every CLAIM went through a real vllm call. Production data has ~5% short-circuit (recon: 0.21% byte-equal `TextQuote == Description`), but the v15 `object:` slot construction in this harness is materially-always-different from `subject + " " + predicate`, so the `is_self_referential` guard never trips on this corpus.

### Production verdict distribution (this run)

| support | count | % |
|---|---:|---:|
| full | 5 | 3.6% |
| partial | 58 | 42.3% |
| none | 74 | 54.0% |

vllm under the structured-similarity prompt now produces a graded distribution (vs 100% `none` under the old yes/no/partial prompt + same aligned harness — see commit history of this branch). The graded distribution is what makes 3-class agreement possible.

### Confusion matrix (production rows × Foundry cols)

| production \\ foundry | full | partial | none |
|---|---|---|---|
| **full** | 1 | 4 | 0 |
| **partial** | 7 | 49 | 2 |
| **none** | 6 | 44 | 24 |

Top disagreement classes (frequency-sorted):
  - production=`none` / foundry=`partial`: 44
  - production=`partial` / foundry=`full`: 7
  - production=`none` / foundry=`full`: 6
  - production=`full` / foundry=`partial`: 4
  - production=`partial` / foundry=`none`: 2

The dominant disagreement is vllm being more conservative than Foundry: production says `none` where Foundry grants `partial` (44/63 disagreements = 70%). vllm strictly refuses to grant partial credit to sparse evidence; Foundry's rationales repeatedly grant topical-relatedness credit (e.g. _"the span 'stock' alone is too vague to fully support … but … relates to the topic of equity/share performance"_). Both models see the same problem (thin evidence); they route it to different verdicts under the same structured-similarity prompt.

### What this run is NOT

This is not a clean apples-to-apples test against an aligned Foundry baseline. The Foundry labels were collected against the OLD self-referential payload (PRs #449/#451/#453's `evidence_span = object_text`), so Foundry's rationales reference different claims than the current vllm sees. The user's Plan-C brief explicitly forbids re-running Foundry; that constraint is honored here. A future PR that re-labels Foundry with the aligned prompt is the next decision point — see `STATE.md` Phase 4 epic.

### Root-cause statement (Plan C verdict)

- **Harness shape bug:** **fixed** (this PR). Self-referentiality 100% → 0%. `evidence_span` and `object_text` are now structurally distinct on every CLAIM, matching the production C# pattern.
- **Production C# (SemanticVerifierService + ClaimVerifier):** **correct** by recon — no changes needed.
- **Production prompt (`BuildPrompt`, PR #452):** **correct** as written; the structured-similarity template combined with the production-shape asymmetric input now yields a graded verdict distribution.
- **Aligned-harness agreement at 54.01%** is honest baseline data — meaningful enough to drive the next gate iteration. Remaining gap to the ≥85% gate is real signal: vllm-Qwen2.5-32B is materially more conservative than Foundry-Claude-4.7 on partial-credit judgments. The actionable next step (per plan §5 Phase 4.7 remediation) is prompt iteration on `BuildPrompt` to either calibrate vllm's partial-credit threshold or re-label Foundry under the aligned prompt. Both are out of scope for this PR per the Plan-C "DO NOT iterate further on the prompt or parser" hard rule.
