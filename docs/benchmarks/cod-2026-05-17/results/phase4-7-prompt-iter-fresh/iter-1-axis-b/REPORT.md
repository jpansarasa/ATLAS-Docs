# Phase 4.7s.4 diagnostic — baseline (no LoRA) eval REPORT — variant `iter_fresh_1_axis_b`

Generated: 2026-05-26T11:26:44Z

## Run

- variant: `iter_fresh_1_axis_b`
- endpoint: `http://localhost:8000/v1/completions`
- model: `Qwen/Qwen2.5-32B-Instruct-AWQ` (base, **no LoRA adapter**)
- max_tokens: 64 | temperature: 0.0
- concurrency: 8
- rows: completed 1560 / requested 1560 (errors 0)
- wall time: 171.0s (9.13 calls/s)

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
- overall accuracy: **0.7776**
- macro F1: 0.5651
- micro F1: 0.7776
- Foundry-agreement subset (n=563): **0.7300**

### Per-class metrics

| label | precision | recall | F1 | support |
|---|---:|---:|---:|---:|
| consistent | 0.764 | 0.902 | 0.827 | 419 |
| related | 0.492 | 0.742 | 0.592 | 198 |
| unrelated | 0.898 | 0.792 | 0.842 | 869 |
| insufficient_evidence | 0.000 | 0.000 | 0.000 | 74 |

### Confusion matrix

| true \ pred | consistent | related | unrelated | insufficient_evidence |
|---|---:|---:|---:|---:|
| consistent | 378 | 19 | 22 | 0 |
| related | 14 | 147 | 37 | 0 |
| unrelated | 55 | 126 | 688 | 0 |
| insufficient_evidence | 48 | 7 | 19 | 0 |

### Top disagreement cells

| true | pred | count |
|---|---|---:|
| unrelated | related | 126 |
| unrelated | consistent | 55 |
| insufficient_evidence | consistent | 48 |
| related | unrelated | 37 |
| consistent | unrelated | 22 |

## 4-way comparison

| run | test set | accuracy | Foundry agreement |
|---|---|---:|---:|
| **this run — variant `iter_fresh_1_axis_b`** | v3/test_holdout n=1560 | **0.7776** | **0.7300** |
| baseline iter5 (PR #467) | v3/test_holdout n=1560 | 0.4904 | 0.3552 |
| LoRA iter #1 (Phase 4.7s.4) | v3/test_holdout n=1560 | 0.5187 | — |
| LoRA iter #2 (class-weighted) | v3/test_holdout n=1560 | 0.4227 | — |
| production iter-5 (prior n=50 acceptance) | n=50 mixed | 0.7153 | — |

## Decision rule (per dispatch brief)

Comparing this variant to baseline iter5 (49.04 % accuracy on the same set):

- Δ accuracy vs iter5 baseline: **+28.72 pp**
- Δ Foundry agreement vs iter5 baseline: **+37.48 pp**

Interpretation rules from the dispatch brief:

- If Foundry agreement ≥ +3 pp vs iter5: IMPORTANT clause IS the dominant bias.
  Recommend prompt iteration without the clause + minor refinement.
- If Foundry agreement decreases or stays flat: gap is more fundamental than one clause.
  Recommend broader Option 4 (verdict-vocabulary redesign) or Option 5 (accept structural).
- If `unrelated → related` confusion drops but a new dominant disagreement appears:
  report it; the supervisor designs the next iter accordingly.

