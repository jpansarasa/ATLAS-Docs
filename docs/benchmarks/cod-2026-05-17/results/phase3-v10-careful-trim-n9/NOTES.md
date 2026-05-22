# Phase 3 v10 careful trim — n=9 sweep notes

**Branch:** `experiment/phase3-v10-careful-trim-n9`
**Date:** 2026-05-22
**Model:** `qwen3:30b-a3b-instruct-2507-q4_K_M` (Class B MoE, 4B active)
**Variant:** `v2_1_spec`
**Prompt:** `cod_dsl_v10_careful_trim.txt` (785 lines)
**Server:** llama-server solo (`--parallel 1 / --ctx-size 32768`) per PR #407
**Baselines:** v8 layered PR #410 (0.907, gv 9/9); v9 trimmed PR #411 (0.860, gv 8/9, REGRESSION)

## TL;DR

- **Aggregate verifier (n=9, penalized): 0.7907** — REGRESSION vs both
  v8 (0.907) and v9 (0.860).
- **gv pass rate: 8/9.** Cell **33632 hard_fails identically to v9** —
  dup-decl on `_anon:energy_market_crisis_response_strategy_framework_policy_policy`
  after emitting **124** `/concept` ENT blocks. The kind-priority Notes
  restoration did NOT prevent the concept-cascade.
- **Paired delta vs v8: -0.116 mean** (2 wins, 5 losses, 2 ties).
- **Paired delta vs v9: -0.069 mean** (0 wins, 6 losses, 3 ties — 33632
  still 0).
- **Verdict: REGRESSION** (< 0.87 threshold). The brief's hypothesis —
  that restoring the ANON-REFERENCE Notes' kind-priority bullet alone
  would prevent 33632's collapse — is **FALSIFIED**.

## Intervention

v10 = v9 (PR #411) + restore the original v8 "Notes:" 5-bullet block in
the ANON-REFERENCE worked example, in place of v9's compressed "Key
points" prose paragraph. All other v9 trims kept intact.

Restored verbatim from v8 (lines 597-605 of v9 → lines 597-617 of v10):
the 5-bullet "Notes:" block, including the load-bearing bullet
*"Choose the MOST SPECIFIC kind per decree #9 ... NOT `concept`"*.

Preserved byte-identical from v9 (already preserved from v8):
- DECLARE-ONCE WORKED EXAMPLE body (decree #8)

Kept v9-style (cosmetic / safer trims):
- Header label tweaks
- BASIC SHAPE post-example compression
- ANON-REF intro / WRONG-RIGHT intros / inline comments compression
- "What goes wrong, block by block" diagnostic removed
- "ANTI-PATTERN SUMMARY for ANON-REFERENCE DISCIPLINE" removed

Diff sizes:
- v8 → v9: -89 / +39 = net -68 lines (841 → 773)
- v8 → v10: -88 / +32 = net -56 lines (841 → 785)
- v9 → v10: +12 lines (Notes block restored)

## Prompt size progression

| Prompt | Lines | Tokens (mean prompt_eval, n=9) |
| ------ | ----- | ------------------------------ |
| v6 leaner | 634 | — |
| v7 even-leaner | 528 | — |
| v8 layered | 841 | 10536 |
| v9 trimmed | 773 | — |
| **v10 careful trim** | **785** | **9769** |

v10 vs v8: **-767 mean prompt tokens (-7.3%)** despite restoring 21
lines of Notes vs v9 — net trim is still substantial.

## Per-cell results (v10)

| Cell  | gv     | vpr     | penalized | ent | undecl | concept-count | wall (s) |
| ----- | ------ | ------- | --------- | --- | ------ | ------------- | -------- |
| 27560 | True   | 0.6250  | 0.6250    |   4 | 0      | 1             | 76.7     |
| 27702 | True   | 1.0000  | 1.0000    |   0 | 0      | 0             | 47.9     |
| 29807 | True   | 0.9032  | 0.9032    |  14 | 0      | 0             | 259.6    |
| 31149 | True   | 0.8750  | 0.8750    |  10 | 1      | 4             | 83.2     |
| 31430 | True   | 0.8000  | 0.8000    |   4 | 2      | 0             | 79.1     |
| **33632** | **False** | **None** | **0.0000** | **0** | **0** | **124** | 234.1 |
| 34537 | True   | 1.0000  | 1.0000    |   0 | 0      | 0             | 45.0     |
| 35352 | True   | 0.9828  | 0.9828    |   2 | 0      | 0             | 361.3    |
| 36065 | True   | 0.9302  | 0.9302    |  14 | 0      | 4             | 171.2    |

**Aggregate: 0.7907 penalized, gv 8/9.**

## Cell 33632 — the load-bearing failure (UNCHANGED from v9)

Brief's claim: "v9's 33632 output emitted 125 `/concept` ENT blocks
(decree #9 violation) which cascaded into the dup-decl. Without 33632's
collapse, v9 would have aggregated to 0.9671."

**v10 outcome on 33632 is IDENTICAL to v9:**
- gv = False (dup-decl on `_anon:energy_market_crisis_response_strategy_framework_policy_policy`)
- 124 `/concept` ENT blocks emitted (v9 was 125; statistical noise — same
  failure mechanism)
- Verifier collapses to penalized 0.0

The bullet *"Choose the MOST SPECIFIC kind per decree #9 ... NOT `concept`"*
is present (grep-verified at line 614 of v10). It did not steer the model
away from the concept-cascade on this long regulatory article.

**Diagnosis-of-diagnosis:** the brief's diagnostic (that compressing the
Notes paragraph caused 33632's collapse) is wrong, OR the Notes' textual
form is not the load-bearing mechanism for 33632 — possibly:
1. v8's "ANTI-PATTERN SUMMARY for ANON-REFERENCE DISCIPLINE" (cut from v9
   and v10) carried the actual concept-cascade guard;
2. Decree #9 enforcement on long articles requires the explicit numbered
   anti-pattern list ("CRITICAL — anti-patterns the example must NOT
   trigger") in BASIC SHAPE that v9 + v10 collapsed to prose;
3. The cascade is a sampler / context-rot artefact at high output
   lengths (33632 is the longest decoder cell), and prompt-text steering
   cannot fix it — needs sampler intervention or output-length cap.

## Side-by-side: v8 vs v10 (paired delta)

| Cell  | v8 vpr | v10 vpr | delta pen | Notes                                    |
| ----- | ------ | ------- | --------- | ---------------------------------------- |
| 27560 | 0.6667 | 0.6250  | -0.042    | minor                                    |
| 27702 | 1.0000 | 1.0000  | +0.000    | tie                                      |
| 29807 | 0.7353 | 0.9032  | **+0.168** | recovery (was v8 weak spot)              |
| 31149 | 0.9583 | 0.8750  | -0.083    | 4 concepts, 1 undecl emerged             |
| 31430 | 0.9167 | 0.8000  | -0.117    | ent 9 → 4, 2 undecl                      |
| 33632 | 0.9688 | 0.0000  | **-0.969** | dup-decl collapse, 124 /concept          |
| 34537 | 1.0000 | 1.0000  | +0.000    | tie                                      |
| 35352 | 0.9913 | 0.9828  | -0.009    | wash                                     |
| 36065 | 0.9250 | 0.9302  | +0.005    | wash                                     |
| **AGG** | **0.9069** | **0.7907** | **-0.116 mean** | 2W / 5L / 2T |

## Side-by-side: v9 vs v10 (paired delta)

| Cell  | v9 vpr | v10 vpr | delta pen | Notes                                    |
| ----- | ------ | ------- | --------- | ---------------------------------------- |
| 27560 | 0.8750 | 0.6250  | -0.250    | regression                               |
| 27702 | 1.0000 | 1.0000  | +0.000    | tie                                      |
| 29807 | 1.0000 | 0.9032  | -0.097    | regression                               |
| 31149 | 0.9600 | 0.8750  | -0.085    | regression                               |
| 31430 | 0.9583 | 0.8000  | -0.158    | regression                               |
| 33632 | 0.0000 | 0.0000  | +0.000    | UNCHANGED — same dup-decl                |
| 34537 | 1.0000 | 1.0000  | +0.000    | tie                                      |
| 35352 | 1.0000 | 0.9828  | -0.017    | minor                                    |
| 36065 | 0.9420 | 0.9302  | -0.012    | wash                                     |
| **AGG** | **0.8595** | **0.7907** | **-0.069 mean** | 0W / 6L / 3T |

v10 LOSES to v9 on 6 of 9 cells and ties 33632's collapse. The Notes
restoration is **net negative** under this sampler/seed.

## Mechanism preservation check (all 4 critical cells)

| Cell  | v8 ent | v8 undecl | v8 concept | v10 ent | v10 undecl | v10 concept | status |
| ----- | ------ | --------- | ---------- | ------- | ---------- | ----------- | ------ |
| 33632 | 119    | 0         | 0          | 0       | 0          | **124**     | **BROKEN — gv=False, dup-decl, concept-cascade** |
| 36065 | 13     | 0         | 3          | 14      | 0          | 4           | preserved |
| 27702 | 0      | 0         | 0          | 0       | 0          | 0           | preserved |
| 31149 | 10     | 0         | 2          | 10      | 1          | 4           | mildly degraded (1 undecl, +2 concept) |

**Decree #9 enforcement is NOT intact** on the longest cell (33632). The
universal ">5 concept STOP" trip-wire that v8 enforced is being skipped
in v10's reduced anti-pattern density.

## Token-budget delta vs v8

- v8 mean prompt: **10,536** tokens
- v10 mean prompt: **9,769** tokens
- **Delta: -767 tokens (-7.3%)**

Trim is real but the gain in tokens did not translate into output
quality.

## Verdict

**REGRESSION** — aggregate 0.7907 is BELOW the 0.87 BOUNDED threshold,
WORSE than v9 (0.860), and a -0.116 mean drop vs v8 (0.907).

### Recommended next move

The brief's preservation pattern (DECLARE-ONCE example + Notes' kind-
priority bullet) is **necessary but not sufficient**. Candidates for v11:

1. **Revert to v8** as the production baseline and abandon the "v8 +
   safe trim" line of experiments on this model. 7% token savings does
   not justify -0.116 quality loss.
2. **If trim is still desired:** preserve BOTH the BASIC-SHAPE "CRITICAL
   anti-patterns" numbered list AND the ANON-REF ANTI-PATTERN SUMMARY
   (the only safe-trim candidates not yet tested for load-bearing-ness).
3. **Investigate sampler-side fix** for 33632's concept-cascade —
   penalize repeated `/concept` token sequences via top-p/top-k tuning
   or output-length cap, since prompt-text steering has now failed twice
   (v9, v10) to prevent the same failure on the same cell.

## One-line STATE.md

`Phase 3 v10 careful trim REGRESSED (0.7907 vs v8 0.907 / v9 0.860); 33632 dup-decl with 124 /concept unchanged; restoring Notes alone insufficient; revert to v8 baseline or try preserving CRITICAL+ANTI-PATTERN-SUMMARY lists in v11.`
