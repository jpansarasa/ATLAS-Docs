# Phase 3 v7 retry (solo mode) — n=9 sweep notes

**Branch:** `experiment/phase3-v7-leaner-n9-retry`
**Date:** 2026-05-22
**Model:** `qwen3:30b-a3b-instruct-2507-q4_K_M`
**Variant:** `v2_1_spec`
**Prompt:** `cod_dsl_v7_even_leaner.txt` (528 lines, -106 vs v6's 634 = -16.7%)
**Baseline:** PR #400 v6 leaner (`phase3-v6-leaner-n9`, mean verifier 0.895)
**Server:** llama-server :11437 solo mode (`--parallel 1 --ctx-size 32768`),
shared with sibling sweeps Exp H (v6+RAG retry) and Exp I (v8 layered) —
queued serially.

## TL;DR

- **Prompt-leanness hypothesis: CONFIRMED.** Mean prompt_eval 6101.7 (v6 8080.7)
  = **-1979 tokens / -24.5%** per call. v7 trim landed exactly on input tokens.
- **Mechanism preservation: PARTIAL REGRESSION.** v7 prompt lost the
  DECLARE-ONCE rule strongly enough that **33632 (dup-decl canary) failed
  grammar with a duplicate ENT line** (v6 had this cell verifier=0.922).
- **Aggregate verifier (n=9 strict, failures=0) = 0.626 → REGRESSION.**
  Aggregate verifier on the 6 paired gv=True cells = **0.940 vs v6 0.875,
  delta +0.065** — v7 is stronger *where it parses*, but loses 1 mechanism
  cell to dup-decl and 2 cells to harness read-timeout under contention.
- 27702 (punctuation-ident) PRESERVED. 33632 (dup-decl) BROKEN.
  36065 (kind-latch) UNKNOWN due to timeout.

## Per-cell table (v7)

| Cell  | gv | verifier | num  | org  | undecl | prompt_eval | eval | wall (s) | stop    |
| ----- | -- | -------- | ---- | ---- | ------ | ----------- | ---- | -------- | ------- |
| 27560 | T  | 1.000    | -    | -    | 0      | 5,856       | 4722 | 217.0    | eos     |
| 27702 | T  | 0.941    | -    | -    | 0      | 5,887       | 973  | 411.1    | eos     |
| 29807 | T  | 0.944    | 1.00 | 1.00 | 0      | 5,909       | 1390 | 460.5    | eos     |
| 31149 | T  | 0.944    | 1.00 | 0.50 | 0      | 5,790       | 802  | 558.9    | eos     |
| 31430 | T  | 0.808    | 0.67 | 0.67 | 0      | 6,164       | 1099 | 435.1    | eos     |
| 33632 | **F** | -     | -    | -    | -      | 7,410       | 3247 | 360.7    | eos     |
| 34537 | T  | 1.000    | -    | -    | 0      | 5,696       | 542  | 708.6    | eos     |
| 35352 | F  | -        | -    | -    | -      | -           | -    | 900.1    | timeout |
| 36065 | F  | -        | -    | -    | -      | -           | -    | 900.0    | timeout |

33632 stop=eos but gv=False: model emitted duplicate
`ENT _anon:coal_fired_power_generation_utilisation_target_80_percent_policy`
twice. parse_errors confirms `"Duplicate ENT declaration"`. This is the
DECLARE-ONCE failure mode v6 closed.

35352 / 36065: harness 900s ReadTimeout (raw_output empty). Infra-side,
not v7-side. Both are long-output cells (v6 35352 eval=6144 stop=limit;
v6 36065 wall=423s); under solo-mode + 2 sibling sweeps queuing they exceeded
the 900s harness cap. Treat as MISSING DATA, not v7 failure.

## Side-by-side delta vs PR #400 v6

| Cell  | v6 verifier | v7 verifier | delta   | v6 gv | v7 gv | Notes                                            |
| ----- | ----------- | ----------- | ------- | ----- | ----- | ------------------------------------------------ |
| 27560 | 0.667       | 1.000       | +0.333  | T     | T     | v7 recovers v5's clean baseline                   |
| 27702 | 1.000       | 0.941       | -0.059  | T     | T     | tiny regression, still strong                     |
| 29807 | 0.812       | 0.944       | +0.132  | T     | T     | gain; undecl 7→0                                  |
| 31149 | 0.867       | 0.944       | +0.077  | T     | T     | gain                                              |
| 31430 | 0.905       | 0.808       | -0.097  | T     | T     | regression                                        |
| 33632 | 0.922       | (gv=F)      | -loss-  | T     | **F** | **DECLARE-ONCE broken under v7 condensation**     |
| 34537 | 1.000       | 1.000       | 0.000   | T     | T     | identical                                         |
| 35352 | 0.991       | (timeout)   | n/a     | T     | F     | infra timeout, not comparable                     |
| 36065 | 0.892       | (timeout)   | n/a     | T     | F     | infra timeout, not comparable                     |

**Paired n=6 (cells with both v6 gv=T and v7 gv=T): v7 = 0.940, v6 = 0.875,
delta = +0.065.** Strict n=9 (failures=0): v7 = 0.626, v6 = 0.895.

## Mechanism preservation

### 33632 dup-decl (DECLARE-ONCE) — **BROKEN**

v6 closed this with decree #10(a) declare-once sub-rule (verifier 0.922,
gv=True, 35 unique ENTs). v7 condensed prompt further; model emitted same
`_anon:coal_fired_power_generation_utilisation_target_80_percent_policy`
ENT twice and verifier rejected as duplicate declaration. **The leanness
trim crossed the line at which DECLARE-ONCE survives without an explicit
verbatim example.**

### 36065 kind-latch — UNKNOWN

Harness read-timeout (900s) with empty raw_output. Cannot evaluate
kind-distribution. Requires re-run under non-contended server to score.

### 27702 punctuation-ident — **PRESERVED**

v7 verifier=0.941, gv=True, no punctuation in `_anon:` IDs. The
`ANON-REFERENCE DISCIPLINE` consolidation in v7 still carries the
punctuation rule.

## Token budget delta

| Metric                          | v6         | v7         | delta            |
| ------------------------------- | ---------- | ---------- | ---------------- |
| Prompt size (lines)             | 634        | 528        | -106 (-16.7%)    |
| Mean prompt_eval (tokens)       | 8,080.7    | 6,101.7    | -1,979 (-24.5%)  |
| Cell 27560 prompt_eval (ref)    | 7,209      | 5,856      | -1,353 (-18.8%)  |

v7's per-cell prompt_eval delta vs v6 is consistent (~-1300 to -1600 per
cell), confirming the input-token savings are deterministic and real.
Expectation of ~6500-7000 tokens from the brief slightly beat: actual
mean 6101.7 is leaner than forecast.

## Wall-time

- Total sweep: 10:24:28Z → 12:13:41Z = **109 min** (vs v6 74 min; solo-mode
  + 2 sibling sweeps add ~50% wall overhead via queue contention).
- Mean per cell where measured (n=7 non-timeout): 450s.
- Longest: 34537 (708s — anomalously slow vs v6 120s, queue contention)
  and 31149 (559s).
- 2 cells (35352, 36065) hit the harness 900s ReadTimeout; raw_output empty.
  These are *long-output* cells where solo-mode latency + queue contention
  combined to exceed the cap.

## Verdict: BOUNDED-REGRESSION

- Aggregate (strict n=9, failures=0) = **0.626** — below the 0.87 floor →
  REGRESSION by the brief's threshold.
- Aggregate (paired n=6 gv=T) = **0.940**, +0.065 vs v6 on same cells.
- Token-leanness landed: -24.5% prompt_eval per call (vs forecast ~10-20%).
- Mechanism canary 33632 (DECLARE-ONCE) **broke** under v7 trim — this is
  the load-bearing failure: v7 cut past the worked example that v6 needed
  to teach declare-once.
- Two cells lost to harness timeout under concurrent server pressure; not
  v7's fault but blocks a clean n=9 baseline.

**Net interpretation:** v7's leanness hypothesis is partially validated
(massive input-token win, +0.065 on the 6 cells that parsed) but the
condensation also crossed a load-bearing decree boundary (DECLARE-ONCE
worked example removed → 33632 dup-decl recurs). The right next step is
**v7.1: re-add a minimal DECLARE-ONCE worked example without re-bloating
the rest** — keep the -24.5% input win, restore the mechanism the trim
stripped. Also re-run 35352/36065 under non-contended server to close
the data gap on kind-latch.
