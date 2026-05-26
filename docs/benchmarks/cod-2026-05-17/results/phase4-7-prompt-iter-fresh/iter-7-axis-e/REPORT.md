# Phase 4.7s.4 diagnostic — baseline (no LoRA) eval REPORT — variant `iter_fresh_7_axis_e`

Generated: 2026-05-26T11:45:18Z

## Run

- variant: `iter_fresh_7_axis_e`
- endpoint: `http://localhost:8000/v1/completions`
- model: `Qwen/Qwen2.5-32B-Instruct-AWQ` (base, **no LoRA adapter**)
- max_tokens: 64 | temperature: 0.0
- concurrency: 8
- rows: completed 1560 / requested 1560 (errors 0)
- wall time: 99.9s (15.62 calls/s)

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
- overall accuracy: **0.6654**
- macro F1: 0.5475
- micro F1: 0.6654
- Foundry-agreement subset (n=563): **0.6146**

### Per-class metrics

| label | precision | recall | F1 | support |
|---|---:|---:|---:|---:|
| consistent | 0.788 | 0.790 | 0.789 | 419 |
| related | 0.726 | 0.414 | 0.527 | 198 |
| unrelated | 0.833 | 0.694 | 0.757 | 869 |
| insufficient_evidence | 0.073 | 0.297 | 0.117 | 74 |

### Confusion matrix

| true \ pred | consistent | related | unrelated | insufficient_evidence |
|---|---:|---:|---:|---:|
| consistent | 331 | 19 | 35 | 34 |
| related | 13 | 82 | 69 | 34 |
| unrelated | 43 | 10 | 603 | 213 |
| insufficient_evidence | 33 | 2 | 17 | 22 |

### Top disagreement cells

| true | pred | count |
|---|---|---:|
| unrelated | insufficient_evidence | 213 |
| related | unrelated | 69 |
| unrelated | consistent | 43 |
| consistent | unrelated | 35 |
| consistent | insufficient_evidence | 34 |

## 4-way comparison

| run | test set | accuracy | Foundry agreement |
|---|---|---:|---:|
| **this run — variant `iter_fresh_7_axis_e`** | v3/test_holdout n=1560 | **0.6654** | **0.6146** |
| baseline iter5 (PR #467) | v3/test_holdout n=1560 | 0.4904 | 0.3552 |
| LoRA iter #1 (Phase 4.7s.4) | v3/test_holdout n=1560 | 0.5187 | — |
| LoRA iter #2 (class-weighted) | v3/test_holdout n=1560 | 0.4227 | — |
| production iter-5 (prior n=50 acceptance) | n=50 mixed | 0.7153 | — |

## Decision rule (per dispatch brief)

Comparing this variant to baseline iter5 (49.04 % accuracy on the same set):

- Δ accuracy vs iter5 baseline: **+17.50 pp**
- Δ Foundry agreement vs iter5 baseline: **+25.94 pp**

Interpretation rules from the dispatch brief:

- If Foundry agreement ≥ +3 pp vs iter5: IMPORTANT clause IS the dominant bias.
  Recommend prompt iteration without the clause + minor refinement.
- If Foundry agreement decreases or stays flat: gap is more fundamental than one clause.
  Recommend broader Option 4 (verdict-vocabulary redesign) or Option 5 (accept structural).
- If `unrelated → related` confusion drops but a new dominant disagreement appears:
  report it; the supervisor designs the next iter accordingly.

