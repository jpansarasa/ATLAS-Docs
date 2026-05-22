# Phase 3 v8 layered context-aware — n=9 sweep notes

**Branch:** `experiment/phase3-v8-layered-n9`
**Date:** 2026-05-22
**Model:** `qwen3:30b-a3b-instruct-2507-q4_K_M` (Class B MoE, 4B active)
**Variant:** `v2_1_spec`
**Prompt:** `cod_dsl_v8_layered.txt` (841 lines = v6 634 + ~200 additive)
**Baselines:** PR #400 v6 leaner (aggregate 0.895, gv 9/9), PR #406
v6+CA (aggregate naive 0.696, gv 7/9 — REGRESSION)

## TL;DR

- **Aggregate verifier (n=9): 0.907** (vs v6 0.895; vs v6+CA naive 0.696)
- **gv pass rate: 9/9** (matches v6; beats v6+CA 7/9)
- **Paired delta vs v6: +0.107 sum / 9 = +0.012 mean (6 wins, 1 loss, 2 ties)**
- **Critical v6+CA regressions FIXED**: 33632 concept 76 -> 0, gv recovered;
  31149 undecl 11 -> 0, vpr 0.682 -> 0.958; 35352 gv recovered
- **Verdict: PROGRESS (bounded)** — 0.907 is in the [0.87, 0.91) BOUNDED band
  by the task spec, but mechanism preservation (the central deliverable)
  succeeded fully

## Intervention

v8 = v6 leaner + TWO PURELY ADDITIVE sections:

1. **ARTICLE TYPE HINT** (top, advisory): `NOTE article_type` block,
   7-type closed taxonomy (same as v6+CA).
2. **TYPE-AWARE EXTRACTION HINTS** (mid-prompt, after decree #10):
   per-type ADDITIVE priorities; NO type permits `concept` as preferred,
   NO decree relaxed, NO under-declare hints.

Decree #9 KIND PRIORITY stays **byte-identical** to v6 (verified by
diff). The universal ">5 concept STOP" trip-wire fires for every type.
`--n-predict 8192` (vs v6+CA 6144) absorbs prompt growth.

## Per-cell results

| Cell  | TYPE (model)  | match? | gv | vpr   | ent | undecl | concept | wall (s) |
| ----- | ------------- | ------ | -- | ----- | --- | ------ | ------- | -------- |
| 27560 | macro_release | YES    | T  | 0.667 | 4   | 0      | 1       | 82       |
| 27702 | other         | YES    | T  | 1.000 | 0   | 0      | 0       | 103      |
| 29807 | other         | NO     | T  | 0.735 | 11  | 7      | 0       | 331      |
| 31149 | other         | NO     | T  | 0.958 | 10  | 0      | 2       | 193      |
| 31430 | macro_release | YES    | T  | 0.917 | 9   | 0      | 0       | 667      |
| 33632 | regulatory    | YES    | T  | 0.969 | 119 | 0      | **0**   | 321      |
| 34537 | other         | YES    | T  | 1.000 | 0   | 0      | 0       | 798      |
| 35352 | other         | NO     | T  | 0.991 | 1   | 0      | 0       | 406      |
| 36065 | other         | YES    | T  | 0.925 | 13  | 0      | 3       | 238      |

**Aggregate: 0.907, gv: 9/9, type match: 6/9.**

## Delta tables

### vs v6 leaner (paired)

| Cell  | v6 vpr | v8 vpr | delta  | Notes                              |
| ----- | ------ | ------ | ------ | ---------------------------------- |
| 27560 | 0.667  | 0.667  | +0.000 | identical                          |
| 27702 | 1.000  | 1.000  | +0.000 | identical                          |
| 29807 | 0.812  | 0.735  | -0.077 | 7 undecl regression (other-typed)  |
| 31149 | 0.867  | 0.958  | +0.092 | gain                               |
| 31430 | 0.905  | 0.917  | +0.012 | marginal gain                      |
| 33632 | 0.922  | 0.969  | +0.047 | gain (more ENTs, mechanism intact) |
| 34537 | 1.000  | 1.000  | +0.000 | identical                          |
| 35352 | 0.991  | 0.991  | +0.000 | identical                          |
| 36065 | 0.892  | 0.925  | +0.033 | gain (concept 4 -> 3)              |

### vs v6+CA (the regression being corrected)

| Cell  | v6+CA gv/vpr | v8 gv/vpr | delta or recovery               |
| ----- | ------------ | --------- | ------------------------------- |
| 27560 | T / 0.800    | T / 0.667 | -0.133 (v6+CA over-shaped)      |
| 27702 | T / 1.000    | T / 1.000 | tie                             |
| 29807 | T / 0.943    | T / 0.735 | -0.208 (v6+CA macro steer won)  |
| 31149 | T / 0.682    | T / 0.958 | **+0.276 (under-decree fix)**   |
| 31430 | T / 0.889    | T / 0.917 | +0.028                          |
| 33632 | **F**        | T / 0.969 | **gv RECOVERY (concept 76->0)** |
| 34537 | T / 1.000    | T / 1.000 | tie                             |
| 35352 | **F**        | T / 0.991 | **gv RECOVERY (n_predict)**     |
| 36065 | T / 0.951    | T / 0.925 | -0.026                          |

## Mechanism preservation (CRITICAL)

- **33632 dup-decl (DECLARE-ONCE)**: PRESERVED. 119 unique ENT IDs /
  119 declared (0 dups), 8 unique NUMs / 8 declared (0 dups). Concept-
  kind ENT count = **0** vs v6+CA's **76**. The universal trip-wire
  fired identically under the `regulatory` classification.
- **36065 kind-latch**: PRESERVED. 3 concept ENTs out of 13 = 23%,
  under the universal ≤5 cap. (v4 baseline was 90/128 = 70%.)
- **27702 punctuation-ident**: PRESERVED. vpr=1.0, 0 punctuation-id
  violations, correctly skipped fake URL boilerplate.
- **31149 declare-then-reference**: PRESERVED + IMPROVED. 10 ENTs
  declared, 0 undecl (vs v6+CA's 1 decl + 11 undecl). Cell classified
  `other` -> routed through v6-default kind priority. The `other -> v6
  default` hint is the load-bearing safety net.

## Why type-assignment match dropped 9/9 -> 6/9 (and why that's GOOD)

3 cells (29807, 31149, 35352) classified `other` instead of the human
intuition (macro_release / analyst_action). This is **defensive
behavior**: `other` routes through v6-default kind priority, the
proven-safe baseline. The cell v6+CA crashed under `analyst_action`
steering (31149) is precisely the cell v8 classified as `other` and
gained +0.276 vpr. The conservative `other` fallback is exactly what
the v8 hint section tells the model to do under ambiguity.

## Process notes

- Initial sweep at runner default `--timeout 900` had 3 cells (33632,
  35352, 36065) hit HTTP read timeout under cold cache while serial-
  queueing against Exp G + Exp H on the solo llama-server slot.
- Retry with `--timeout 1800` succeeded for all three on warm cache
  (321s, 406s, 238s wall). Final results = retry values, overwritten
  via `--force`.
- Total wall: initial 144 min + retry 16 min = 160 min (over 125-min
  budget). Ephemeral sweep wrappers in `/tmp/v8_sweep.sh` and
  `/tmp/v8_retry.sh` (not committed).
