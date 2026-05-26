# Phase 4.7s.4 diagnostic — baseline (no LoRA) eval REPORT — variant `iter_fresh_4_axis_h`

Generated: 2026-05-26T11:37:11Z

## Run

- variant: `iter_fresh_4_axis_h`
- endpoint: `http://localhost:8000/v1/completions`
- model: `Qwen/Qwen2.5-32B-Instruct-AWQ` (base, **no LoRA adapter**)
- max_tokens: 64 | temperature: 0.0
- concurrency: 8
- rows: completed 1560 / requested 1560 (errors 0)
- wall time: 173.4s (9.00 calls/s)

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
- overall accuracy: **0.7917**
- macro F1: 0.5639
- micro F1: 0.7917
- Foundry-agreement subset (n=563): **0.7993**

### Per-class metrics

| label | precision | recall | F1 | support |
|---|---:|---:|---:|---:|
| consistent | 0.749 | 0.826 | 0.785 | 419 |
| related | 0.759 | 0.510 | 0.610 | 198 |
| unrelated | 0.817 | 0.907 | 0.860 | 869 |
| insufficient_evidence | 0.000 | 0.000 | 0.000 | 74 |

### Confusion matrix

| true \ pred | consistent | related | unrelated | insufficient_evidence |
|---|---:|---:|---:|---:|
| consistent | 346 | 10 | 63 | 0 |
| related | 11 | 101 | 86 | 0 |
| unrelated | 60 | 20 | 788 | 1 |
| insufficient_evidence | 45 | 2 | 27 | 0 |

### Top disagreement cells

| true | pred | count |
|---|---|---:|
| related | unrelated | 86 |
| consistent | unrelated | 63 |
| unrelated | consistent | 60 |
| insufficient_evidence | consistent | 45 |
| insufficient_evidence | unrelated | 27 |

## 4-way comparison

| run | test set | accuracy | Foundry agreement |
|---|---|---:|---:|
| **this run — variant `iter_fresh_4_axis_h`** | v3/test_holdout n=1560 | **0.7917** | **0.7993** |
| baseline iter5 (PR #467) | v3/test_holdout n=1560 | 0.4904 | 0.3552 |
| LoRA iter #1 (Phase 4.7s.4) | v3/test_holdout n=1560 | 0.5187 | — |
| LoRA iter #2 (class-weighted) | v3/test_holdout n=1560 | 0.4227 | — |
| production iter-5 (prior n=50 acceptance) | n=50 mixed | 0.7153 | — |

## Decision rule (per dispatch brief)

Comparing this variant to baseline iter5 (49.04 % accuracy on the same set):

- Δ accuracy vs iter5 baseline: **+30.13 pp**
- Δ Foundry agreement vs iter5 baseline: **+44.41 pp**

Interpretation rules from the dispatch brief:

- If Foundry agreement ≥ +3 pp vs iter5: IMPORTANT clause IS the dominant bias.
  Recommend prompt iteration without the clause + minor refinement.
- If Foundry agreement decreases or stays flat: gap is more fundamental than one clause.
  Recommend broader Option 4 (verdict-vocabulary redesign) or Option 5 (accept structural).
- If `unrelated → related` confusion drops but a new dominant disagreement appears:
  report it; the supervisor designs the next iter accordingly.

