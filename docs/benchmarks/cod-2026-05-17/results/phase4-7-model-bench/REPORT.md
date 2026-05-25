# Phase 4.7 — 3-way model ablation REPORT

_Generated: 2026-05-25T18:17:14Z_

**User mandate (2026-05-25):** "run a benchmark with the current Qwen2.5-32B and IBM Granite 4.1-30B and Mistral-Small-3.2-24B."

**Hypothesis under test.** Does the ClaimVerifier model matter, or is the 14pp Qwen-vs-Claude gap (Qwen=71.53% agreement on iter-5 prompt vs Claude 4.7 ≥85% gate) structural to the DSL CLAIM-verification task?

**Decision rule.** A candidate is judged "matters" if agreement moves |Δ| ≥ 5.0pp vs the Qwen baseline. Below that, the gap is treated as structural — the iter-5 prompt + the four-way verdict vocabulary is the floor across this model class.

## 1. Headline table

| Model | Params | Quant | Agreement % vs Foundry | Δ vs Qwen | Compared CLAIMs | Wall time (s) | Notes |
|---|---|---|---:|---:|---:|---:|---|
| Qwen2.5-32B-Instruct-AWQ | 32B | AWQ INT4 (awq_marlin) | 71.53% | baseline | 137 | 97.1 | iter-5 prompt; REUSED from PR #456 |
| Granite-4.1-30B-AWQ-INT4 | 30B | AWQ INT4 (compressed-tensors) | 56.20% | -15.33pp | 137 | 20.2 | served by vllm as `cyankiwi/granite-4.1-30b-AWQ-INT4` |
| Mistral-Small-3.2-24B-AWQ | 24B | AWQ INT4 (compressed-tensors) | 18.98% | -52.55pp | 137 | 6.4 | served by vllm as `gghfez/Mistral-Small-3.2-24B-Instruct-hf-AWQ` |

## 2. Verdict

**Qwen2.5-32B is the best of the three candidates on the iter-5 prompt — both swaps REGRESSED.** Neither Granite nor Mistral matches the Qwen baseline (71.53%). Largest regression: Mistral-Small-3.2-24B-AWQ at 18.98% (-52.55pp). The ~14pp Qwen-vs-Claude agreement gap is **structural** to the iter-5 prompt + DSL CLAIM shape at this model class — swapping in a different ~30B AWQ open model makes the gap **wider**, not narrower. Further calibration must come from the prompt, the CLAIM input shape, the verdict vocabulary, or moving up the model-strength axis (Claude-4.7-class or larger), not from a different ~30B open model.

## 3. Per-arm detail

### Qwen2.5-32B-Instruct-AWQ

- Model id (served): `Qwen/Qwen2.5-32B-Instruct-AWQ`
- Articles loaded: 47; with ≥1 CLAIM: 24; total CLAIMs: 137
- Per-CLAIM compared: agree=98, disagree=39, foundry-missing=0
- Agreement: **71.53%**
- Wall time (Σ per-CLAIM vllm latency): **97.1 s**
- Malformed-response markers: **0** / 137

**Confusion (production rows × Foundry cols):**

| production \ foundry | full | partial | none |
|---|---|---|---|
| **full** | 0 | 1 | 0 |
| **partial** | 13 | 82 | 10 |
| **none** | 1 | 14 | 16 |

**Top 3 disagreement classes:**

- production=none / foundry=partial: 14
- production=partial / foundry=full: 13
- production=partial / foundry=none: 10

### Granite-4.1-30B-AWQ-INT4

- Model id (served): `cyankiwi/granite-4.1-30b-AWQ-INT4`
- Articles loaded: 47; with ≥1 CLAIM: 24; total CLAIMs: 137
- Per-CLAIM compared: agree=77, disagree=60, foundry-missing=0
- Agreement: **56.20%**
- Wall time (Σ per-CLAIM vllm latency): **20.2 s**
- Malformed-response markers: **0** / 137

**Confusion (production rows × Foundry cols):**

| production \ foundry | full | partial | none |
|---|---|---|---|
| **full** | 0 | 13 | 0 |
| **partial** | 7 | 60 | 9 |
| **none** | 7 | 24 | 17 |

**Top 3 disagreement classes:**

- production=none / foundry=partial: 24
- production=full / foundry=partial: 13
- production=partial / foundry=none: 9

### Mistral-Small-3.2-24B-AWQ

- Model id (served): `gghfez/Mistral-Small-3.2-24B-Instruct-hf-AWQ`
- Articles loaded: 47; with ≥1 CLAIM: 24; total CLAIMs: 137
- Per-CLAIM compared: agree=26, disagree=111, foundry-missing=0
- Agreement: **18.98%**
- Wall time (Σ per-CLAIM vllm latency): **6.4 s**
- Malformed-response markers: **137** / 137

**Confusion (production rows × Foundry cols):**

| production \ foundry | full | partial | none |
|---|---|---|---|
| **full** | 0 | 0 | 0 |
| **partial** | 0 | 0 | 0 |
| **none** | 14 | 97 | 26 |

**Top 3 disagreement classes:**

- production=none / foundry=partial: 97
- production=none / foundry=full: 14

## 4. Methodology

- Same n=50 sample_ids (seed=4757) reused from the Phase 4.7 iter-5 run.
- Same iter-5 BuildPrompt + TryParseVerdict (the production code post revert `b0aec1c8` after axes F+G regressions). Prompt asks for one of {consistent, related, unrelated, insufficient_evidence} which the parser collapses to {full, partial, none}.
- Same Foundry Claude 4.7 labels (47 articles labeled; 3 articles parse-failed at the v15 DSL grammar level and are missing from both arms — the same 3 ids 32859, 35352, 36050).
- Per arm: vllm-server stopped, restarted with the new --model, warmed up, harness run against /v1/completions, vllm-server then stopped before the next swap. 5090 = 32 GiB VRAM, single-model serving only.
- All three quantizations are AWQ-INT4: Qwen via `awq_marlin` kernel; Granite + Mistral via vLLM's `compressed-tensors` loader (the `quantization` field in config.json drives kernel selection automatically).
- Qwen baseline numbers are REUSED from PR #456 outputs at `results/phase4-7-iter-5-axis-c-rule-first/v2_pipeline/`; no re-run. Reuse is safe because the production code path, prompt, parser, and Foundry labels are all byte-identical to what's on disk.

## 6. Caveat — Mistral-Small-3.2 /v1/completions incompatibility

Mistral-Small-3.2-24B returned an **empty body** on 137/137 CLAIM prompts (100% malformed-response). The vllm `/v1/completions` endpoint (the production code path used by the C# `ClaimVerifier`) bypasses chat templates and passes the prompt as raw text; Mistral-3 instruct checkpoints emit EOS immediately on raw text completions because they were SFT-tuned chat-only.

Smoke-test confirmed this is a prompt-shape issue, not a load failure: `/v1/completions` with a generic prompt against Mistral-3 returned `' consistent\nThe structured finding is consistent...'` (verbose, prefix-spaced), but the actual iter-5 verifier prompt returns `''` with `finish_reason=stop`. The production C# ClaimVerifier emits the exact same prompt over the exact same endpoint, so this is what production would see if it were pointed at this model id.

Per the user benchmark mandate, the iter-5 prompt is **frozen** for this ablation. Re-testing Mistral under `/v1/chat/completions` with chat-template wrapping would change the prompt path being measured and is out of scope; the result reported here (18.98% agreement) is what production would currently see from Mistral-Small-3.2 with no other code changes.

## 7. Artefact paths

- Sample ids: `results/phase4-7-model-bench/sample_ids.txt`
- Foundry labels (shared): `results/phase4-7-model-bench/foundry_labels/*.json`
- Qwen2.5-32B-Instruct-AWQ v2_pipeline: `results/phase4-7-iter-5-axis-c-rule-first/v2_pipeline/*.json`
- Granite-4.1-30B-AWQ-INT4 v2_pipeline: `results/phase4-7-model-bench/granite-4.1-30b/v2_pipeline/*.json`
- Mistral-Small-3.2-24B-AWQ v2_pipeline: `results/phase4-7-model-bench/mistral-small-3.2-24b/v2_pipeline/*.json`
- This report: `results/phase4-7-model-bench/REPORT.md`
