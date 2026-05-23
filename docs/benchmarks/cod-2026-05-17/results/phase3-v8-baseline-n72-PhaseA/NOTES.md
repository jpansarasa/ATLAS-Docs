# Phase A — v8 baseline at scale (n=72) — NOTES

**Branch:** `experiment/phase3-v8-baseline-n72-PhaseA`
**Date:** 2026-05-22 / 23
**Model:** `qwen3:30b-a3b-instruct-2507-q4_K_M` (Class B MoE, 4B active)
**Variant:** `v2_1_spec`
**Prompt:** `cod_dsl_baseline.txt` (locked = v8, PR #413, `f549ef8c`)
**Corpus:** `corpus.full72.jsonl` (72 articles)
**GT:** `ground-truth.jsonl` (72 rows)
**Backend:** llama-server solo (`--parallel 1`, `--ctx-size 32768`, `--n-predict 8192`, `--timeout 1800`)
**Wall:** sweep 19:50:18Z → 00:55:53Z = **5h 5min** (vs ~7h projected)

## TL;DR

- **Penalized verifier (n=72): 0.7826** (vs v8 n=9: **0.907**, Δ −0.124)
- **Scored-only verifier (n=66 of 72): 0.8538**
- **gv-fails: 6/72 (8.3%)** — vs v8 n=9: 0/9
  - 5 are **DECLARE-ONCE violations** on `_anon:concept` symbols (concept-flood)
  - 1 is a **1800s timeout** (cell 28398)
- **n_predict (8192) hits: 0** — every grammar-valid output stopped on `eos`
- **Verdict: DEGRADES AT SCALE** (penalized 0.7826 < 0.80 threshold)

The 0.907 aggregate held only on the curated 9-cell Arm B set.
Broadening to 72 articles surfaces failure modes — concept-flood on
ideological / dementia / energy-policy prose — that the v8 mechanism
guards do not suppress.

## Aggregate

| Metric                                  | v8 n=9  | v8 n=72 | Δ       |
| --------------------------------------- | ------- | ------- | ------- |
| Penalized mean (gv-fail = 0)            | 0.907   | 0.7826  | −0.124  |
| Scored-only mean                        | 0.907   | 0.8538  | −0.053  |
| gv pass rate                            | 9/9     | 66/72   | −8.3 pp |
| Median wall (s)                         | 321     | 116     | −205    |
| Mean wall (s)                           | 349     | 254     | −95     |
| Max wall (s)                            | 798     | 1800    | +1002   |

(The faster median on n=72 is corpus mix — the n=9 set was deliberately
heavy on long earnings-call-style cells. Median <60s for 6/72 cells
indicates the 72-set includes a tail of short articles that finish in
seconds at solo throughput.)

## Cell stratification

### Wall distribution (n=72)

| Bucket          | Count | Notes                                  |
| --------------- | ----- | -------------------------------------- |
| <60s            | 6     | short articles                         |
| 60–180s         | 43    | bulk                                   |
| 180–600s        | 17    |                                        |
| 600–1200s       | 3     |                                        |
| 1200–1700s      | 2     |                                        |
| ≥1700s (near-TO)| 1     | cell 28398, hit 1800s read-timeout     |

### gv-fail cells (6)

| ID    | Wall (s) | Failure mode                                | First duplicate symbol                                     |
| ----- | -------- | ------------------------------------------- | ---------------------------------------------------------- |
| 28398 | 1800     | **ReadTimeout** (HTTP, llama-server)        | n/a (no raw output captured)                               |
| 33598 | 255      | Duplicate ENT `_anon:concept`                | `U_S_future_self_reflection_preservation_action…`          |
| 33632 | 370      | Duplicate ENT `_anon:concept` (mech cell!)  | `malaysia_energy_supply_disruption_response`               |
| 35051 | 360      | Duplicate ENT `_anon:concept`                | `market_forecasts`                                         |
| 36050 | 223      | Duplicate ENT `_anon:concept`                | `living_well_with_dementia`                                |
| 36566 | 386      | Duplicate ENT `_anon:concept`                | `market_moving_reasons`                                    |

**Pattern**: 5/6 gv-failures are the same mechanism — model emits the
same `_anon:concept` symbol twice, violating DECLARE-ONCE. Concept-flood
is the v6+CA failure mode that v8 was specifically designed to suppress;
v8 succeeded on the n=9 set, but the suppression does not generalize.

### Prompt eval (n=71, excluding timeout cell with no metadata)

- p50: 9894 tokens
- mean: 10618 tokens
- max: 20135 tokens
- All under 32K ctx; no truncation pressure observed.

### n_predict cap

- 0 cells hit `stop_type == "limit"`. 8192 was sufficient for every
  cell that produced output.

## Mechanism cells (the 4 v6-flagship cells, all present in n=72)

| Cell  | v8 n=9 gv/vpr   | v8 n=72 gv/vpr      | Preserved?                              |
| ----- | --------------- | ------------------- | --------------------------------------- |
| 27702 | T / 1.000       | T / 1.000           | yes (identical)                         |
| 31149 | T / 0.958       | T / 0.885           | yes, slight drop                        |
| 33632 | T / 0.969       | **F / None**        | **NO — concept-flood REGRESSION**       |
| 36065 | T / 0.925       | T / 0.909           | yes, slight drop                        |

**33632 broke**: this is the `regulatory` cell where v8 had explicitly
fixed v6+CA's concept-flood failure (concept count 76 → 0). At n=72
that fix did not hold; same cell, same prompt, but emits a duplicate
`_anon:malaysia_energy_supply_disruption_response`. Indicates the
fix was either run-noise-sensitive (sampler temp 0.2) or the surrounding
n=9 ordering / cache state mattered.

## Bottom 10 vpr cells (signal, not noise)

| ID    | gv | vpr   | ent | undecl | wall (s) | Likely cause                                       |
| ----- | -- | ----- | --- | ------ | -------- | -------------------------------------------------- |
| 35185 | T  | 0.200 | 2   | 0      | 127      | severe under-extraction                            |
| 27558 | T  | 0.219 | 20  | 0      | 253      | low recall vs GT                                   |
| 34230 | T  | 0.312 | 2   | 0      | 154      | severe under-extraction                            |
| 27772 | T  | 0.328 | 127 | 0      | 607      | the Jenoptik earnings call (47K), entity-flood    |
| 36482 | T  | 0.333 | 2   | 0      | 64       | severe under-extraction                            |
| 35336 | T  | 0.391 | 13  | 28     | 146      | UNDECL-explosion (28 entities referenced never declared) |
| 29163 | T  | 0.469 | 127 | 0      | 634      | entity-flood (>100 ENTs)                           |
| 36184 | T  | 0.519 | 24  | 24     | 144      | UNDECL-explosion                                   |
| 37629 | T  | 0.541 | 13  | 18     | 111      | UNDECL-explosion                                   |
| 27560 | T  | 0.667 | 4   | 0      | 83       | (was also bottom on n=9, same vpr)                 |

**Three distinct failure modes** beyond concept-flood:

1. **Under-extraction** (ent_count ≤ 4 when GT has many entities)
   — 35185, 34230, 36482, 27560: model emits a near-empty document.
2. **UNDECL-explosion** (entities referenced in EVT/CLAIM but never
   declared via ENT) — 35336, 36184, 37629: v8 decree #2 not enforced
   well on these.
3. **Entity-flood** (ent_count ≥ 100) — 27772 (Jenoptik), 29163: long
   articles trip the model into emitting every name as an ENT.

## Verdict

**DEGRADES AT SCALE.**

- penalized 0.7826 < 0.80 → fails the "HOLDS" (≥0.85) and "MOSTLY HOLDS"
  (0.80–0.85) bands; falls into DEGRADES.
- 5 new failure modes surface (concept-flood × 5, timeout × 1) that the
  9-cell Arm B did not exhibit.
- Mech cell 33632 regression — even the cell where v8's intervention was
  designed and demonstrated to work does not hold on a fresh run within
  a larger corpus.

Implications:

- The 0.907 v8 number is **Arm B-specific**, not a general property.
  Future prompts must be validated at n≥30 before claiming progress.
- The DECLARE-ONCE rule is brittle on `_anon:concept` declarations
  when the model is generating long lists of similar concepts.
- ~8% gv-fail rate on broadly-sampled production corpus is the floor
  v8 establishes; subsequent prompts must beat this on n=72, not n=9.

## Run mechanics (no length-cap filter applied; outlier behavior is data)

- All 72 cells ran; none excluded.
- 1 timeout (28398) counted as gv-fail (penalized = 0.0) per the
  honest scoring rubric.
- No prompt-eval truncation; no n_predict-cap hits.
- Concept-flood failures emit clean grammar tokens, then a duplicate
  `_anon:` symbol — model is "trying to comply" but loses track of
  which anonymous concepts it has already declared.

## Files

- 72/72 cell JSONs at `qwen3_30b-a3b-instruct-2507-q4_K_M/v2_1_spec/`.
- Sweep log preserved at `/tmp/v8_n72_sweep.log` (host-local, not committed).
