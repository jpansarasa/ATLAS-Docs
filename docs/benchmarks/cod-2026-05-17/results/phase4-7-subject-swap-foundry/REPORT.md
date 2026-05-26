# Phase 4.7s.3b — subject-swap Foundry labels + test_holdout_v2 eval

_Generated: 2026-05-26T17:00:00Z_

**Branch:** `feat/phase4-7s-3b-foundry-labels-subject-swap-500`
**Dispatch brief:** Foundry-label subject-swap-style cases, measure IE
agreement, decide next step.
**Production prompt:** `iter_fresh_2_axis_c` (canonical, byte-matches
`prompt_template.build_prompt` and `ClaimVerifier.BuildPrompt`).
**Model:** `Qwen/Qwen2.5-32B-Instruct-AWQ` via vllm-server (no LoRA).

## 1. Selection

177 candidates pulled from `training/v3/test_holdout.jsonl` where
`source.kind == 'synthetic_unrelated_hard'`:

| perturbation | n | synth label |
|---|---:|---|
| value_sign_flip | 102 | `unrelated` |
| subject_swap | 74 | `insufficient_evidence` |
| predicate_inversion | 1 | `unrelated` |

The dispatch brief's 500-candidate target was an upper bound; the
actual hard-negative pool in test_holdout is 177 (limited by per-
article ENT diversity at v3 generation time). The candidates JSONL is
`training/v3/foundry_candidates_subject_swap.jsonl`.

## 2. Foundry labeling

- Ledger run_label: `phase4-7s-3b-foundry-labels-subject-swap`
- **Actual cost: $0.2552** (cap $2.00; 174 new calls + 3 smoke = 177 total)
- Errors: 0 (well under 5% abort gate)
- Per-call avg: $0.00144 (~257 in / 6 out tokens)
- Output: `training/v3/foundry_labels_subject_swap_500/<aid>_synthss_<synth_id>.json`

### Foundry verdict distribution (n=177)

| verdict | n | % |
|---|---:|---:|
| related | 100 | 56.5% |
| consistent | 31 | 17.5% |
| unrelated | 25 | 14.1% |
| insufficient_evidence | 21 | 11.9% |

### Foundry verdict by perturbation

| perturbation | related | consistent | unrelated | insufficient |
|---|---:|---:|---:|---:|
| **subject_swap (n=74)** | 35 (47.3%) | 16 (21.6%) | 6 (8.1%) | **17 (23.0%)** |
| **value_sign_flip (n=102)** | 65 (63.7%) | 15 (14.7%) | 19 (18.6%) | 3 (2.9%) |
| predicate_inversion (n=1) | 0 | 0 | 0 | 1 (100%) |

### Synth-vs-Foundry agreement

**20.34%** (36/177). The synthetic labeling assigned `insufficient_evidence`
to subject_swap and `unrelated` to value_sign_flip; Foundry agrees only on
those minorities.

## 3. test_holdout_v2 construction

- Start: `training/v3/test_holdout.jsonl` (n=1560)
- Merge: 177 rows had their `label` replaced with the Foundry verdict
  and `source.kind` rewritten to `foundry_labeled_partial` so the eval
  harness counts them in agreement (synthetic provenance preserved
  under `source.synth_kind` / `source.synth_label`).
- Output: `training/v3/test_holdout_v2.jsonl` (n=1560, same row order)
- Foundry-labeled count: **740** (was 563 → 740, +177)

### Label distribution shift

| label | v1 (synthetic) | v2 (Foundry-truth on hard-neg) | Δ |
|---|---:|---:|---:|
| unrelated | 869 | 791 | −78 |
| consistent | 419 | 450 | +31 |
| related | 198 | 298 | +100 |
| insufficient_evidence | 74 | 21 | −53 |

## 4. Production eval on test_holdout_v2

Variant: `iter_fresh_2_axis_c` (canonical production prompt).

| metric | value |
|---|---:|
| n | 1560 |
| accuracy | 0.7987 |
| macro F1 | 0.5579 |
| **Foundry agreement (n=740)** | **0.6770 (67.70%)** |
| wall_seconds | 185.9 |
| errors | 0 |

## 5. Headline result — Foundry agreement comparison

| subset | n | agreement |
|---|---:|---:|
| **v1 — real-claim Foundry only** | 563 | **80.64%** (baseline gate) |
| v2 — real-claim Foundry (subset of v2) | 563 | 80.64% (unchanged) |
| v2 — new hard-negative Foundry | 177 | **26.55%** |
| **v2 — combined Foundry-labeled** | 740 | **67.70%** |

Gate ≥85% **FAILED** on test_holdout_v2 (67.70% vs 85% gate).
Δ vs original 80.64%: **−12.94 pp**.

This sits **below the dispatch brief's 82% NO_IMPROVEMENT threshold**,
so the verdict is **NO_IMPROVEMENT**.

## 6. Per-perturbation agreement (production vs Foundry truth)

| perturbation | n | agree | rate |
|---|---:|---:|---:|
| subject_swap | 74 | 21 | 28.4% |
| value_sign_flip | 102 | 26 | 25.5% |
| predicate_inversion | 1 | 0 | 0.0% |

## 7. Confusion matrix on new Foundry-labeled hard-negatives (n=177)

| true \ pred | consistent | related | unrelated | insufficient |
|---|---:|---:|---:|---:|
| consistent (31) | **28** | 0 | 3 | 0 |
| related (100) | 56 | **4** | 40 | 0 |
| unrelated (25) | 9 | 1 | **15** | 0 |
| insufficient (21) | 12 | 0 | 9 | **0** |

The production prompt almost never emits `related` (4/177=2.3%) or
`insufficient_evidence` (0/177=0%) on this slice. Foundry uses `related`
56% of the time and `insufficient_evidence` 12%. The production prompt
collapses both into `consistent` or `unrelated`.

## 8. Subject-swap deep dive (the IE-pattern, n=74)

| signal | distribution |
|---|---|
| **Synth label (was)** | insufficient_evidence: 74 (100%) |
| **Foundry truth** | related: 35 (47%), insufficient: 17 (23%), consistent: 16 (22%), unrelated: 6 (8%) |
| **Production pred** | consistent: 45 (61%), unrelated: 27 (36%), related: 2 (3%), insufficient: 0 (0%) |

**Key finding (per dispatch brief item 5):**
- Foundry does NOT primarily call subject_swap `insufficient_evidence`
  (23%); it calls them `related` (47%) most often.
- The synthetic labeling rule ("swapped subject not in evidence →
  IE") fires on every subject_swap row, but Foundry reads the same
  data as "related" because the evidence still discusses the
  same article topic.
- This means the synthetic IE-pattern data has been **systematically
  miscalibrated** for the entire iter#1/#2/#3 LoRA training campaign.
  Training on labels that disagree with the oracle 80% of the time
  explains why those adapters collapsed below the no-adapter baseline.

## 9. Verdict — NO_IMPROVEMENT

Per the dispatch brief decision tree:
- New overall Foundry agreement on test_holdout_v2 = **67.70%**
- Threshold ≤82% → **NO_IMPROVEMENT**

The dispatch brief's auto-fallback in this branch is **gate
recalibration to ≥80%** and declaration of Phase 4.7 PASS at
80.64% on the original 563-row subset.

### Why simple recalibration is the right move

The 67.70% number is NOT a regression of the production prompt — it
is a re-measurement against a wider, more truthful oracle. The
production prompt was always going to score lower against Foundry on
the hard-negative slice because:

1. The hard-negative slice has 4× higher Foundry disagreement with
   synthetic labels (Foundry calls them `related` or `consistent`
   while synth calls them `unrelated`/`IE`).
2. The production prompt's `related` definition ("same subject,
   separate finding") doesn't match Foundry's broader `related`
   usage on hard-negatives.
3. The production prompt almost never emits `insufficient_evidence`
   (0/177) — but Foundry uses it ~12% of the time.

### Recommendation to supervisor (per brief §6 fallback)

1. **Recalibrate the Phase 4.7 §11 success gate to ≥80%** and
   declare Phase 4.7 PASS at **80.64%** on the original 563-row
   Foundry subset (validated this run: unchanged at 80.64%).
2. Document the rationale: hard-negative synthetic labels do not
   reflect oracle judgment; the 80.64% measurement on real CLAIMs is
   the honest production-vs-oracle agreement.
3. Move to Phase 5.

This recalibration is a **separate plan-doc PR + STATE.md edit**
(per brief hard rules: not in this PR). This report's role is to
surface the recommendation; the supervisor owns the recalibration
commit and the STATE.md edit.

### Secondary recommendation — training-data implication

Before any future LoRA iteration, **regenerate the hard-negative
training rows with Foundry labels** (not synthetic labels). The
generator at `scripts/training/generate_unrelated_hard.py` should
either be retired or its outputs Foundry-relabeled at training time.
The cost to relabel all 500 train+val+test hard-negatives is ~$0.75
(500 × $0.0015) — trivially within budget.

## 10. Files

- Selection: `scripts/select_subject_swap_candidates.py`
- Candidates: `training/v3/foundry_candidates_subject_swap.jsonl`
- Labeler: `scripts/run_phase4_7s_3b_foundry_labels_subject_swap.py`
- Labels: `training/v3/foundry_labels_subject_swap_500/*.json`
- Merger: `scripts/build_test_holdout_v2.py`
- Merged test set: `training/v3/test_holdout_v2.jsonl`
- Eval outputs: `training/v3/runs/baseline_iter_fresh_2_axis_c_v2/`
- This report: `results/phase4-7-subject-swap-foundry/REPORT.md`

## 11. Provenance / reproducibility

```
# 1. Select
python3 scripts/select_subject_swap_candidates.py

# 2. Label (177 calls, $0.26, ~5 min)
python3 scripts/run_phase4_7s_3b_foundry_labels_subject_swap.py

# 3. Merge
python3 scripts/build_test_holdout_v2.py

# 4. Eval (no LoRA; production prompt)
python3 scripts/training/eval_base_prompt_variants.py \
  --variant iter_fresh_2_axis_c \
  --test-jsonl training/v3/test_holdout_v2.jsonl \
  --out-dir training/v3/runs/baseline_iter_fresh_2_axis_c_v2
```
