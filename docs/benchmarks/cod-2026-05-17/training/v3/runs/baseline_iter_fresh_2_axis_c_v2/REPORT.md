# Phase 4.7s.4 diagnostic — baseline (no LoRA) eval REPORT — variant `iter_fresh_2_axis_c`

Generated: 2026-05-26T16:54:32Z

## Run

- variant: `iter_fresh_2_axis_c`
- endpoint: `http://localhost:8000/v1/completions`
- model: `Qwen/Qwen2.5-32B-Instruct-AWQ` (base, **no LoRA adapter**)
- max_tokens: 64 | temperature: 0.0
- concurrency: 8
- rows: completed 1560 / requested 1560 (errors 0)
- wall time: 185.9s (8.39 calls/s)

## Test set composition (v3/test_holdout.jsonl)

- label distribution:
  - `unrelated`: 791
  - `consistent`: 450
  - `related`: 298
  - `insufficient_evidence`: 21
- source kind distribution:
  - `foundry_labeled_partial`: 740
  - `synthetic_unrelated_strong`: 410
  - `synthetic_consistent_v15`: 389
  - `synthetic_consistent_pipeline`: 21

## Metrics — held-out v3 test set

- n = 1560
- overall accuracy: **0.7987**
- macro F1: 0.5579
- micro F1: 0.7987
- Foundry-agreement subset (n=740): **0.6770**

### Per-class metrics

| label | precision | recall | F1 | support |
|---|---:|---:|---:|---:|
| consistent | 0.813 | 0.831 | 0.822 | 450 |
| related | 0.725 | 0.433 | 0.542 | 298 |
| unrelated | 0.806 | 0.939 | 0.867 | 791 |
| insufficient_evidence | 0.000 | 0.000 | 0.000 | 21 |

### Confusion matrix

| true \ pred | consistent | related | unrelated | insufficient_evidence |
|---|---:|---:|---:|---:|
| consistent | 374 | 11 | 65 | 0 |
| related | 64 | 129 | 105 | 0 |
| unrelated | 10 | 38 | 743 | 0 |
| insufficient_evidence | 12 | 0 | 9 | 0 |

### Top disagreement cells

| true | pred | count |
|---|---|---:|
| related | unrelated | 105 |
| consistent | unrelated | 65 |
| related | consistent | 64 |
| unrelated | related | 38 |
| insufficient_evidence | consistent | 12 |

## 4-way comparison

| run | test set | accuracy | Foundry agreement |
|---|---|---:|---:|
| **this run — variant `iter_fresh_2_axis_c`** | v3/test_holdout n=1560 | **0.7987** | **0.6770** |
| baseline iter5 (PR #467) | v3/test_holdout n=1560 | 0.4904 | 0.3552 |
| LoRA iter #1 (Phase 4.7s.4) | v3/test_holdout n=1560 | 0.5187 | — |
| LoRA iter #2 (class-weighted) | v3/test_holdout n=1560 | 0.4227 | — |
| production iter-5 (prior n=50 acceptance) | n=50 mixed | 0.7153 | — |

## Decision rule (per dispatch brief)

Comparing this variant to baseline iter5 (49.04 % accuracy on the same set):

- Δ accuracy vs iter5 baseline: **+30.83 pp**
- Δ Foundry agreement vs iter5 baseline: **+32.18 pp**

Interpretation rules from the dispatch brief:

- If Foundry agreement ≥ +3 pp vs iter5: IMPORTANT clause IS the dominant bias.
  Recommend prompt iteration without the clause + minor refinement.
- If Foundry agreement decreases or stays flat: gap is more fundamental than one clause.
  Recommend broader Option 4 (verdict-vocabulary redesign) or Option 5 (accept structural).
- If `unrelated → related` confusion drops but a new dominant disagreement appears:
  report it; the supervisor designs the next iter accordingly.

