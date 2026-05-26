# Phase 4.7s.4 diagnostic — baseline (no LoRA) eval REPORT — variant `iter_fresh_5_axis_j`

Generated: 2026-05-26T11:38:56Z

## Run

- variant: `iter_fresh_5_axis_j`
- endpoint: `http://localhost:8000/v1/completions`
- model: `Qwen/Qwen2.5-32B-Instruct-AWQ` (base, **no LoRA adapter**)
- max_tokens: 64 | temperature: 0.0
- concurrency: 8
- rows: completed 1560 / requested 1560 (errors 0)
- wall time: 73.1s (21.34 calls/s)

## Test set composition (v3/test_holdout.jsonl)

- label distribution:
  - `unrelated`: 869
  - `consistent`: 419
  - `related`: 198
  - `insufficient_evidence`: 74
- source kind distribution:
  - `foundry_labeled_partial`: 563
  - `synthetic_unrelated_strong`: 410
  - `synthetic_consistent_v15`: 389
  - `synthetic_unrelated_hard`: 177
  - `synthetic_consistent_pipeline`: 21

## Metrics — held-out v3 test set

- n = 1560
- overall accuracy: **0.2699**
- macro F1: 0.2109
- micro F1: 0.2699
- Foundry-agreement subset (n=563): **0.0320**

### Per-class metrics

| label | precision | recall | F1 | support |
|---|---:|---:|---:|---:|
| consistent | 0.569 | 0.857 | 0.684 | 419 |
| related | 0.078 | 0.020 | 0.032 | 198 |
| unrelated | 0.897 | 0.040 | 0.077 | 869 |
| insufficient_evidence | 0.027 | 0.311 | 0.050 | 74 |

### Confusion matrix

| true \ pred | consistent | related | unrelated | insufficient_evidence |
|---|---:|---:|---:|---:|
| consistent | 359 | 12 | 2 | 46 |
| related | 51 | 4 | 1 | 142 |
| unrelated | 176 | 30 | 35 | 628 |
| insufficient_evidence | 45 | 5 | 1 | 23 |

### Top disagreement cells

| true | pred | count |
|---|---|---:|
| unrelated | insufficient_evidence | 628 |
| unrelated | consistent | 176 |
| related | insufficient_evidence | 142 |
| related | consistent | 51 |
| consistent | insufficient_evidence | 46 |

## 4-way comparison

| run | test set | accuracy | Foundry agreement |
|---|---|---:|---:|
| **this run — variant `iter_fresh_5_axis_j`** | v3/test_holdout n=1560 | **0.2699** | **0.0320** |
| baseline iter5 (PR #467) | v3/test_holdout n=1560 | 0.4904 | 0.3552 |
| LoRA iter #1 (Phase 4.7s.4) | v3/test_holdout n=1560 | 0.5187 | — |
| LoRA iter #2 (class-weighted) | v3/test_holdout n=1560 | 0.4227 | — |
| production iter-5 (prior n=50 acceptance) | n=50 mixed | 0.7153 | — |

## Decision rule (per dispatch brief)

Comparing this variant to baseline iter5 (49.04 % accuracy on the same set):

- Δ accuracy vs iter5 baseline: **-22.05 pp**
- Δ Foundry agreement vs iter5 baseline: **-32.32 pp**

Interpretation rules from the dispatch brief:

- If Foundry agreement ≥ +3 pp vs iter5: IMPORTANT clause IS the dominant bias.
  Recommend prompt iteration without the clause + minor refinement.
- If Foundry agreement decreases or stays flat: gap is more fundamental than one clause.
  Recommend broader Option 4 (verdict-vocabulary redesign) or Option 5 (accept structural).
- If `unrelated → related` confusion drops but a new dominant disagreement appears:
  report it; the supervisor designs the next iter accordingly.

