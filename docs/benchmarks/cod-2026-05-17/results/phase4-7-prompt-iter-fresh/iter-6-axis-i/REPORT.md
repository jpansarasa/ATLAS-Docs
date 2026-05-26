# Phase 4.7s.4 diagnostic — baseline (no LoRA) eval REPORT — variant `iter_fresh_6_axis_i`

Generated: 2026-05-26T11:43:14Z

## Run

- variant: `iter_fresh_6_axis_i`
- endpoint: `http://localhost:8000/v1/completions`
- model: `Qwen/Qwen2.5-32B-Instruct-AWQ` (base, **no LoRA adapter**)
- max_tokens: 64 | temperature: 0.0
- concurrency: 8
- rows: completed 1560 / requested 1560 (errors 0)
- wall time: 229.1s (6.81 calls/s)

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
- overall accuracy: **0.5474**
- macro F1: 0.4469
- micro F1: 0.5615
- Foundry-agreement subset (n=563): **0.3641**

### Per-class metrics

| label | precision | recall | F1 | support |
|---|---:|---:|---:|---:|
| consistent | 0.722 | 0.909 | 0.805 | 419 |
| related | 0.309 | 0.778 | 0.442 | 198 |
| unrelated | 0.967 | 0.366 | 0.531 | 869 |
| insufficient_evidence | 0.008 | 0.014 | 0.010 | 74 |

### Confusion matrix

| true \ pred | consistent | related | unrelated | insufficient_evidence | _unknown_ |
|---|---:|---:|---:|---:|---:|
| consistent | 381 | 26 | 1 | 8 | 3 |
| related | 22 | 154 | 6 | 9 | 7 |
| unrelated | 75 | 303 | 318 | 108 | 65 |
| insufficient_evidence | 50 | 16 | 4 | 1 | 3 |
| _unknown_ | 0 | 0 | 0 | 0 | 0 |

### Top disagreement cells

| true | pred | count |
|---|---|---:|
| unrelated | related | 303 |
| unrelated | insufficient_evidence | 108 |
| unrelated | consistent | 75 |
| unrelated | _unknown_ | 65 |
| insufficient_evidence | consistent | 50 |

## 4-way comparison

| run | test set | accuracy | Foundry agreement |
|---|---|---:|---:|
| **this run — variant `iter_fresh_6_axis_i`** | v3/test_holdout n=1560 | **0.5474** | **0.3641** |
| baseline iter5 (PR #467) | v3/test_holdout n=1560 | 0.4904 | 0.3552 |
| LoRA iter #1 (Phase 4.7s.4) | v3/test_holdout n=1560 | 0.5187 | — |
| LoRA iter #2 (class-weighted) | v3/test_holdout n=1560 | 0.4227 | — |
| production iter-5 (prior n=50 acceptance) | n=50 mixed | 0.7153 | — |

## Decision rule (per dispatch brief)

Comparing this variant to baseline iter5 (49.04 % accuracy on the same set):

- Δ accuracy vs iter5 baseline: **+5.70 pp**
- Δ Foundry agreement vs iter5 baseline: **+0.89 pp**

Interpretation rules from the dispatch brief:

- If Foundry agreement ≥ +3 pp vs iter5: IMPORTANT clause IS the dominant bias.
  Recommend prompt iteration without the clause + minor refinement.
- If Foundry agreement decreases or stays flat: gap is more fundamental than one clause.
  Recommend broader Option 4 (verdict-vocabulary redesign) or Option 5 (accept structural).
- If `unrelated → related` confusion drops but a new dominant disagreement appears:
  report it; the supervisor designs the next iter accordingly.

