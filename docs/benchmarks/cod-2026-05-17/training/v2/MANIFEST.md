# Phase 4.7s.1 — Training Data Manifest (v1)

Generated: 2026-05-25T21:16:54Z
Seed: 17

## Bucket counts (actual)

| bucket | actual | target | status |
|---|---:|---:|---|
| consistent | 1000 | 1000 | OK |
| unrelated_strong | 1000 | 1000 | OK |
| unrelated_hard | 500 | 500 | OK |
| partial/related (Foundry) | 137 | 1500 | SHORT (Phase 4.7s.3 needed) |

## Bucket counts (by `source.kind`)

- `foundry_labeled_partial`: 137
- `synthetic_consistent_pipeline`: 137
- `synthetic_consistent_v15`: 863
- `synthetic_unrelated_hard`: 500
- `synthetic_unrelated_strong`: 1000

## Label distribution (four-value verdict enum)

- `consistent`: 1014
- `insufficient_evidence`: 244
- `related`: 97
- `unrelated`: 1282

## Split sizes

- train: 1304
- val: 314
- test_holdout: 1019

## Held-out article pool (OK)

- target: 100
- actual: 100
- excluded iter sweep: iter-1..10 v2_pipeline article_ids
- pool source: corpus.full72.jsonl
- sample IDs (first 10): 27070, 27078, 27364, 27548, 28815, 29067, 29246, 29405, 29546, 29581

## Foundry candidate pool (Phase 4.7s.3 input)

- 486 CLAIMs identified for cloud labeling
- written to: `foundry_candidates.jsonl`
- selection: low word-grounding overlap OR non-verbatim object_text

## Source pointers

- claim_inputs: `results/phase4-7-acceptance/v2_pipeline/*.json`
- v15 T=0 raw DSL: `results/phase3-word-grounding/v15-n72-T0/qwen3_30b-a3b-instruct-2507-q4_K_M/v2_3_spec/*.json`
- corpus: `corpus.full72.jsonl`
- existing Foundry labels: `results/*/foundry_labels/*.json`

## Notes

* The partial/related bucket FELL SHORT of 1500. Only 137 distinct (article_id, claim_block_id) pairs have existing Foundry labels (137 claims × 17 iter runs = many labels, but only 137 distinct claims). Phase 4.7s.3 must run NEW Foundry calls against the 137 candidates in `foundry_candidates.jsonl` AND/OR a corpus expansion is required.

