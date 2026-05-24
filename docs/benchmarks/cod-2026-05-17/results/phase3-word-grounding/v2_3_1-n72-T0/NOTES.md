# Arm A — v2.3.1 n=72 at T=0 (deterministic)

**Date:** 2026-05-24
**Branch:** `experiment/v2.3.1-n72-T0`
**Model:** qwen3:30b-a3b-instruct-2507-q4_K_M
**Backend:** llama.cpp server (solo, `--parallel 1`, `--ctx-size 32768`)
**Sweep duration:** ~9h40m wall (06:33 → 16:15 local; ~8.81h cumulative model compute across 72 cells, shared with Arm B)
**Sampler:** `--temperature 0.0 --top-k 1` (deterministic greedy)

## Purpose

Arm A of the v2.3.1-vs-v15 T=0 A/B. Re-runs the v2.3.1 baseline (PR
#428: v14 word-hints prompt + v2.3 grammar, no chunking, no RAG) with
sampling RNG removed so the comparison against Arm B (v15 chunked
compound, T=0) becomes diagnostic of compound-design value, not
sampling variance.

PR #428 reported v2.3.1 stochastic at pooled word **0.8733** (d=2029).
This sweep re-runs the same 72 cells deterministically so the delta
against Arm B's v15 T=0 (this PR's sister) reflects only architectural
differences, not stochasticity.

## Source materials (all on main)

| File | Purpose |
|---|---|
| `prompts/cod_dsl_v14_word_hints.txt` | v14 word-hints prompt (PR #428) |
| `docs/grammars/cod-dsl-v2.3.gbnf` | v2.3 schema with `source_words` slot |
| `dsl/parser_v2_3.py` | v2.3 parser |
| `dsl/verifier_v2_3_1.py` | v2.3.1 verifier (punct tolerance + casefold, PR #427) |
| `scripts/run_cod_dsl.py` | per-cell runner; supports `--temperature`, `--top-k` (no `--chunked` for Arm A) |
| `scripts/sweep_v2_3_1_n72_T0.sh` | this sweep's driver |
| `scripts/rescore_v2_3_1.py` | per-cell v2.3.1 rescore + pooled aggregate |

## TL;DR

**Arm A v2.3.1 T=0 pooled word (v2.3.1) = 0.8033 (d=3014)**. Trails
v2.3.1 stochastic baseline (0.8733) by **-0.070** and trails Arm B v15
T=0 (0.8261) by **-0.0228** ≥ 0.01 → **Arm B wins the T=0 A/B**. The
v2.3.1 stochastic-vs-deterministic delta itself (-0.070) is larger than
the v15-vs-v2.3.1 delta at T=0 (+0.023), so the dominant source of
variance in this experiment family is sampler RNG, not architecture.
3/72 cells parse-fail at T=0 (30802, 32859, 35336) — same as Arm B's
parse-fail set minus 27772 (which Arm A parses successfully and scores
0.937 byte / 0.992 word, recovering a chunked-only cell from PR #429).

## Per-cell table (Arm A v2.3.1 T=0 vs Arm B v15 T=0 vs v2.3.1 stoch)

```
   cell  gv_t0   byte_t0  word_t0  word_t0_c | byte_v15T0  word_v15T0 | byte_v23s word_v23s
---------------------------------------------------------------------------------------------------------
  27178    T    0.0000   0.0000    0.0000  |  0.0000    0.0000  |  0.0000    0.0000
  27548    T    0.9545   0.9545    0.9545  |  0.9545    0.9545  |  0.9730    0.9459
  27558    T    0.1417   0.1339    0.1339  |  0.1417    0.1339  |  0.8246    0.8070
  27560    T    0.8824   0.8824    0.8824  |  0.8824    0.8824  |  0.8824    0.8824
  27609    T    0.8095   0.8095    0.8095  |  0.8095    0.8095  |  0.9130    0.9130
  27702    T    0.0000   0.0000    0.0000  |  0.0000    0.0000  |  0.0000    0.0000
  27772    T    0.9370   0.9921    0.9370  |    N/A       N/A   |    N/A       N/A
  28398    T    0.3386   0.3307    0.3307  |  0.3386    0.3307  |  0.9167    0.8611
  29149    T    1.0000   1.0000    1.0000  |  1.0000    1.0000  |  0.9512    0.8049
  29163    T    1.0000   1.0000    1.0000  |  1.0000    1.0000  |  1.0000    1.0000
  29581    T    1.0000   1.0000    0.9565  |  1.0000    1.0000  |  0.7667    0.7667
  29807    T    0.9500   0.9750    0.9500  |  0.9500    0.9750  |  0.9000    0.9000
  29845    T    0.0000   0.0000    0.0000  |  0.0000    0.0000  |  0.0000    0.0000
  29846    T    0.0236   0.0236    0.0236  |  0.0236    0.0236  |  0.9091    0.9545
  29848    T    0.9764   0.9685    0.9685  |  0.9764    0.9685  |  0.9375    0.8750
  30273    T    1.0000   1.0000    1.0000  |  1.0000    1.0000  |  1.0000    1.0000
  30350    T    0.8873   0.9437    0.8873  |  0.8873    0.9437  |  0.9677    0.9839
  30596    T    0.0000   0.0000    0.0000  |  0.0000    0.0000  |  0.0000    0.0000
  30619    T    0.0000   0.0000    0.0000  |  0.0000    0.0000  |  0.0000    0.0000
  30674    T    0.6410   0.6923    0.6410  |  0.6410    0.6923  |  0.9783    0.9348
  30802    F      N/A      N/A       N/A   |    N/A       N/A   |  1.0000    0.9318
  31149    T    0.9167   1.0000    0.9167  |  0.9167    1.0000  |  0.9000    1.0000
  31430    T    0.9412   0.9412    0.8235  |  0.9412    0.9412  |  1.0000    0.9444
  31521    T    1.0000   1.0000    1.0000  |  1.0000    1.0000  |  1.0000    0.9375
  31527    T    1.0000   1.0000    1.0000  |  1.0000    1.0000  |  1.0000    1.0000
  31590    T    0.0000   0.0000    0.0000  |  0.0000    0.0000  |  0.0000    0.0000
  31629    T    1.0000   1.0000    1.0000  |  1.0000    1.0000  |  1.0000    0.9412
  31770    T    0.8740   0.9921    0.8740  |  0.8740    0.9921  |  0.7953    0.9606
  31852    T    1.0000   1.0000    1.0000  |  1.0000    1.0000  |  1.0000    0.9231
  32097    T    1.0000   1.0000    1.0000  |  1.0000    1.0000  |  1.0000    1.0000
  32146    T    0.8824   0.9608    0.9020  |  0.8824    0.9608  |  0.8475    0.9661
  32270    T    0.0000   0.0000    0.0000  |  0.0000    0.0000  |  0.0000    0.0000
  32360    T    1.0000   0.9750    0.9750  |  1.0000    0.9750  |  0.9762    1.0000
  32859    F      N/A      N/A       N/A   |    N/A       N/A   |    N/A       N/A
  33374    T    0.9048   1.0000    0.9524  |  0.9048    1.0000  |  1.0000    0.9444
  33569    T    0.8966   0.9310    0.8966  |  0.8966    0.9310  |  0.8621    0.8000
  33598    T    1.0000   1.0000    1.0000  |  1.0000    1.0000  |  1.0000    0.3000
  33632    T    0.4488   0.9921    0.9921  |  0.4488    0.9921  |  0.3780    0.9921
  33749    T    0.9886   0.9886    0.9886  |  0.9886    0.9886  |  0.9630    0.9753
  34230    T    0.1250   0.1250    0.1250  |  0.1250    0.1250  |  0.5000    0.5000
  34303    T    0.8571   0.8571    0.8571  |  0.8571    0.8571  |  0.8750    0.7500
  34310    T    0.0000   0.0000    0.0000  |  0.0000    0.0000  |  0.0000    0.0000
  34537    T    0.0000   0.0000    0.0000  |  0.0000    0.0000  |  0.0000    0.0000
  34666    T    0.9843   0.9921    0.9843  |  0.9843    0.9921  |  0.7736    0.7736
  34754    T    1.0000   1.0000    1.0000  |  1.0000    1.0000  |  0.9000    1.0000
  35051    T    0.0866   0.0787    0.0787  |  0.7540    0.7857  |    N/A       N/A
  35088    T    1.0000   1.0000    1.0000  |  1.0000    1.0000  |  1.0000    1.0000
  35185    T    0.0000   0.0000    0.0000  |  0.0000    0.0000  |  0.0000    0.0000
  35336    F      N/A      N/A       N/A   |    N/A       N/A   |    N/A       N/A
  35352    T    0.9912   1.0000    0.9912  |  0.9912    1.0000  |  0.9826    0.9739
  35354    T    0.9000   0.9333    0.9333  |  0.9000    0.9333  |  0.2366    0.2366
  35355    T    0.9412   0.9608    0.9216  |  0.9412    0.9608  |  0.9194    0.8710
  35441    T    1.0000   0.9231    0.9231  |  1.0000    0.9231  |  1.0000    1.0000
  35987    T    1.0000   1.0000    1.0000  |  1.0000    1.0000  |  1.0000    1.0000
  36031    T    1.0000   1.0000    1.0000  |  1.0000    1.0000  |  1.0000    1.0000
  36050    T    0.8571   1.0000    0.7143  |  0.8571    1.0000  |  0.8750    0.9375
  36065    T    1.0000   1.0000    0.9500  |  1.0000    1.0000  |  0.9412    1.0000
  36135    T    1.0000   0.9524    0.9524  |  1.0000    0.9524  |  0.9000    0.9500
  36184    T    0.9921   0.9843    0.9764  |  0.9921    0.9843  |  0.9149    0.8511
  36185    T    0.6842   0.6579    0.6579  |  0.6842    0.6579  |  0.7857    0.7500
  36445    T    0.9500   1.0000    1.0000  |  0.9500    1.0000  |  0.9500    0.9500
  36482    T    0.0000   0.0000    0.0000  |  0.0000    0.0000  |  0.0000    0.0000
  36509    T    0.0000   0.0000    0.0000  |  0.0000    0.0000  |  0.0000    0.0000
  36543    T    0.8667   0.8667    0.8667  |  0.8667    0.8667  |  1.0000    1.0000
  36558    T    0.9565   0.9565    0.9565  |  0.9565    0.9565  |  1.0000    1.0000
  36566    T    0.9490   0.9592    0.9490  |  0.9490    0.9592  |    N/A       N/A
  37333    T    0.9636   0.9455    0.9455  |  0.9636    0.9455  |  0.9512    0.8780
  37629    T    0.9310   0.9310    0.9310  |  0.9310    0.9310  |  0.9667    1.0000
  37730    T    0.9912   1.0000    0.9912  |  0.9912    1.0000  |  0.9914    0.9914
  38048    T    0.5591   0.5512    0.5354  |  0.5591    0.5512  |    N/A       N/A
  38157    T    1.0000   1.0000    1.0000  |  1.0000    1.0000  |  1.0000    1.0000
  38457    T    0.8750   1.0000    0.8750  |  0.8750    1.0000  |  0.8667    1.0000
```

Notes:
- `gv_t0` column = source-grammar `grammar_valid` per cell JSON. Three
  cells fail source parsing: **30802, 32859, 35336**.
- Cell `27772` parses cleanly under Arm A but parse-fails under Arm B
  (chunked merger artifact); Arm A captures it at 0.937 byte / 0.992
  word. Cell `35051` shows Arm A at 0.087/0.079 vs Arm B at 0.754/0.786
  — chunked extraction is the *only* path that recovers it.

## Pooled aggregates (denom-weighted)

| Run | pooled byte v2.3.1 | pooled word v2.3.1 | n_cells |
|---|---|---|---|
| Arm A v2.3.1 T=0 (this) | **0.7674** (d=3014) | **0.8033** (d=3014) | 72 |
| Arm B v15 T=0 (parallel) | 0.7893 (d=2886) | **0.8261** (d=2886) | 72 |
| v2.3.1 stochastic (PR #428, RNG=on) | 0.8530 (d=1980) | **0.8733** (d=2029) | 72 |
| v15 stochastic (PR #429, RNG=on) | 0.7581 (d=2795) | **0.7753** (d=2795) | 72 |

### Deltas (Arm A vs ...)
- vs Arm B v15 T=0 (0.8261): **-0.0228** (Arm B wins by ≥ 0.01)
- vs v2.3.1 stoch (0.8733, RNG=on baseline): **-0.0700** (RNG-off costs ~0.07 here)
- vs v15 stoch (0.7753): **+0.0280** (deterministic beats v15 stochastic — RNG hurts v15 more than it helps v2.3.1)

## Parse failures (3/72, 4.2%)

| Cell | Failure mode | Stoch baseline (PR #428) |
|---|---|---|
| 30802 | "Unexpected $END" line 596 (truncation mid-block) | 1.000 / 0.932 (passed) |
| 32859 | Duplicate ENT: '_anon:Strait_of_Hormuz' | N/A (also failed in stoch) |
| 35336 | Duplicate ENT: 'I' | N/A (also failed in stoch) |

30802 is a true T=0 regression — passed in stochastic, truncates at T=0.
The remaining two are persistent duplicate-ENT failures shared with both
the stochastic v2.3.1 baseline and Arm B v15 T=0.

## Verdict ladder (sister to Arm B's)

This Arm is the *denominator* for the v15 T=0 A/B verdict computed in
PR #432 (Arm B). For completeness, the verdict from the Arm A side:

| delta(v15_T0 − v23_T0) | verdict | implication |
|---|---|---|
| ≥ +0.01 | **DESIGN SOUND** | RNG was the killer; promote v15 |
| within ±0.01 | **COMPOUND DOES NOT HELP DETERMINISTICALLY** | abandon |
| ≤ -0.01 | **DESIGN WRONG** | chunking actively hurts beyond truncation cells |

**Computed: delta = +0.0228 → DESIGN SOUND.** Arm B's v15 chunked
compound beats this Arm A baseline by 0.0228 pooled word (v2.3.1
verifier) at matched T=0 sampling. Caveats:

- Larger result: stoch-vs-determ delta (-0.070 for v2.3.1) >> v15-vs-v2.3.1
  delta at T=0 (+0.023). Sampler RNG is the dominant axis of variance,
  not architecture, in this experiment family.
- v2.3.1 stochastic at 0.8733 remains the absolute SOTA. Promoting v15
  requires either (a) running v15 stochastic well enough to beat 0.8733
  on average (PR #429 showed it doesn't, at 0.7753), or (b) accepting
  that determinism is more valuable than the absolute peak — in which
  case v15 T=0 (0.8261) is the new ceiling.

## Methodology notes

- Per-cell scoring uses `dsl/verifier_v2_3_1.py` — same verifier used in
  PR #428, PR #429, and Arm B's NOTES.
- Pooled aggregate is denom-weighted: sum(match * denom) / sum(denom).
  Empty-doc and parse-fail cells contribute 0/0 so they don't deflate.
- Both arms share the same llama-server (`--parallel 1`), so cells
  alternate across arms. Wall time roughly doubles per arm.
- Arm A denom (3014) > Arm B denom (2886) because three Arm-B chunked
  cells parse-fail (27772, 30802, 32859), removing their denominators.
  Arm A only loses 30802, 32859, 35336 — so Arm A has more weight in
  the pooled stats.

## Source

Sweep script: `scripts/sweep_v2_3_1_n72_T0.sh`.
Rescore + aggregate: `scripts/rescore_v2_3_1.py`.
Comparison source: `results/phase3-word-grounding/v15-1-rescored-n72-T0/.../rescore_summary.json`
(Arm B, in PR #432 worktree).
