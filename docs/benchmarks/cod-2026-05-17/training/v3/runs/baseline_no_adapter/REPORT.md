# Phase 4.7s.4 — baseline (no LoRA) eval REPORT

Generated: 2026-05-26T10:44:12Z

## Run

- endpoint: `http://localhost:8000/v1/completions`
- model: `Qwen/Qwen2.5-32B-Instruct-AWQ` (base, **no LoRA adapter**)
- max_tokens: 64 | temperature: 0.0
- concurrency: 8
- rows: completed 1560 / requested 1560 (errors 0)
- wall time: 181.5s (8.59 calls/s)

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
- overall accuracy: **0.4904**
- macro F1: 0.4039
- micro F1: 0.4904
- Foundry-agreement subset (n=563): **0.3552**

### Per-class metrics

| label | precision | recall | F1 | support |
|---|---:|---:|---:|---:|
| consistent | 0.813 | 0.759 | 0.785 | 419 |
| related | 0.232 | 0.949 | 0.372 | 198 |
| unrelated | 0.989 | 0.298 | 0.458 | 869 |
| insufficient_evidence | 0.000 | 0.000 | 0.000 | 74 |

### Confusion matrix

| true \ pred | consistent | related | unrelated | insufficient_evidence |
|---|---:|---:|---:|---:|
| consistent | 318 | 95 | 0 | 6 |
| related | 1 | 188 | 1 | 8 |
| unrelated | 43 | 486 | 259 | 81 |
| insufficient_evidence | 29 | 43 | 2 | 0 |

### Top disagreement cells

| true | pred | count |
|---|---|---:|
| unrelated | related | 486 |
| consistent | related | 95 |
| unrelated | insufficient_evidence | 81 |
| unrelated | consistent | 43 |
| insufficient_evidence | related | 43 |

## 3-way comparison (baseline vs LoRA iters)

| run | test set | accuracy |
|---|---|---:|
| **baseline (no LoRA), this run** | v3/test_holdout n=1560 | **0.4904** |
| LoRA iter #1 (Phase 4.7s.4) | v3/test_holdout n=1560 | 0.5187 |
| LoRA iter #2 (class-weighted) | v3/test_holdout n=1560 | 0.4227 |
| production prompt iter-5 (prior n=50 acceptance) | n=50 mixed | 0.7153 |

Caveat: the n=50 acceptance number was measured on the iter-1..10
contaminated pool; the n=1560 baseline above is on a fresh held-out
split (the same one the LoRA iters were evaluated on), so the
baseline-vs-iter rows here are apples-to-apples while the n=50
number is informational only.

## Verdict

See dispatch brief NTFY `O241G1X7X4Iy` — interpretation rules:

- if baseline >= 85%: adapter is hurting; production prompt suffices.
- if baseline ~= LoRA iters (~50%): gap to 85% is structural; SFT can't close it.
- if baseline meaningfully > LoRA iters but < 85%: adapter is regressing
  against the base — fix the training signal (label noise / loss / weighting)
  before any more adapter iteration.

