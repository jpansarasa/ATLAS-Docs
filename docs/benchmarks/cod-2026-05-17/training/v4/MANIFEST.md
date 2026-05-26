# Phase 4.7s.4 iter#3 — Training Data Manifest (v4)

Generated: 2026-05-26T12:29:40Z
Seed: 17 (inherited from v3 generators)

## Bucket counts (actual, train split)

| bucket | rows | notes |
|---|---:|---|
| v3 train (kept verbatim) | 1778 | iter#1/iter#2 input |
| synthetic IE produced | 300 | `generate_insufficient_evidence_synthetic.py` |
| synthetic IE kept (post v3-dedup) | 267 | 33 dropped |
| foundry IE produced | 391 | `run_foundry_ie_labels.py` |
| foundry IE kept (post dedup) | 249 | 142 dropped |
| **total v4 train** | **2294** | |

## Label distribution (train, four-value verdict enum)

- `consistent`: 509
- `related`: 237
- `unrelated`: 903
- `insufficient_evidence`: 645

## Split sizes

- train: 2294 (was 1778 in v3; iter#3 augmentation)
- val: 355 (UNCHANGED from v3)
- test_holdout: 1560 (UNCHANGED from v3)

## Source pointers

- v3 train: `training/v3/train.jsonl`
- synthetic IE: `training/v4/synthetic_ie.jsonl`
- foundry IE: `training/v4/foundry_ie.jsonl`
- val (mirrored from v3): `training/v3/val.jsonl`
- test_holdout (mirrored from v3): `training/v3/test_holdout.jsonl`

## Iter#3 rationale

iter#1 (run_001) + iter#2 (run_002_classweighted) both produced F1=0.000 for `insufficient_evidence` despite class-weighting in iter#2. RUN_NOTES diagnosed the gap as a representation problem: loss-side reweighting cannot conjure verdict tokens the model never emits. iter#3 attacks the gradient signal directly by expanding the IE bucket from 129 → 645 rows.

## Idempotency

Re-running `build_v4_training_data.py` with the same input JSONLs produces a byte-identical `train.jsonl`. The dedup key is `(label, subject, predicate, object_text, evidence_span)`; precedence on collision is foundry > synthetic > v3.
