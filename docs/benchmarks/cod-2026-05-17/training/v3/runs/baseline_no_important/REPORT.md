# Phase 4.7s.4 diagnostic — baseline (no LoRA) eval REPORT — variant `no_important`

Generated: 2026-05-26T10:55:00Z

## Run

- variant: `no_important`
- endpoint: `http://localhost:8000/v1/completions`
- model: `Qwen/Qwen2.5-32B-Instruct-AWQ` (base, **no LoRA adapter**)
- max_tokens: 64 | temperature: 0.0
- concurrency: 8
- rows: completed 1560 / requested 1560 (errors 0)
- wall time: 195.6s (7.97 calls/s)

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
- overall accuracy: **0.6994**
- macro F1: 0.5183
- micro F1: 0.6996
- Foundry-agreement subset (n=563): **0.5933**

### Per-class metrics

| label | precision | recall | F1 | support |
|---|---:|---:|---:|---:|
| consistent | 0.791 | 0.866 | 0.827 | 419 |
| related | 0.347 | 0.823 | 0.488 | 198 |
| unrelated | 0.910 | 0.650 | 0.758 | 869 |
| insufficient_evidence | 0.000 | 0.000 | 0.000 | 74 |

### Confusion matrix

| true \ pred | consistent | related | unrelated | insufficient_evidence | _unknown_ |
|---|---:|---:|---:|---:|---:|
| consistent | 363 | 40 | 14 | 1 | 1 |
| related | 6 | 163 | 27 | 2 | 0 |
| unrelated | 49 | 249 | 565 | 6 | 0 |
| insufficient_evidence | 41 | 18 | 15 | 0 | 0 |
| _unknown_ | 0 | 0 | 0 | 0 | 0 |

### Top disagreement cells

| true | pred | count |
|---|---|---:|
| unrelated | related | 249 |
| unrelated | consistent | 49 |
| insufficient_evidence | consistent | 41 |
| consistent | related | 40 |
| related | unrelated | 27 |

## 4-way comparison

| run | test set | accuracy | Foundry agreement |
|---|---|---:|---:|
| **this run — variant `no_important`** | v3/test_holdout n=1560 | **0.6994** | **0.5933** |
| baseline iter5 (PR #467) | v3/test_holdout n=1560 | 0.4904 | 0.3552 |
| LoRA iter #1 (Phase 4.7s.4) | v3/test_holdout n=1560 | 0.5187 | — |
| LoRA iter #2 (class-weighted) | v3/test_holdout n=1560 | 0.4227 | — |
| production iter-5 (prior n=50 acceptance) | n=50 mixed | 0.7153 | — |

## Decision rule (per dispatch brief)

Comparing this variant to baseline iter5 (49.04 % accuracy on the same set):

- Δ accuracy vs iter5 baseline: **+20.90 pp**
- Δ Foundry agreement vs iter5 baseline: **+23.81 pp**

Interpretation rules from the dispatch brief:

- If Foundry agreement ≥ +3 pp vs iter5: IMPORTANT clause IS the dominant bias.
  Recommend prompt iteration without the clause + minor refinement.
- If Foundry agreement decreases or stays flat: gap is more fundamental than one clause.
  Recommend broader Option 4 (verdict-vocabulary redesign) or Option 5 (accept structural).
- If `unrelated → related` confusion drops but a new dominant disagreement appears:
  report it; the supervisor designs the next iter accordingly.

