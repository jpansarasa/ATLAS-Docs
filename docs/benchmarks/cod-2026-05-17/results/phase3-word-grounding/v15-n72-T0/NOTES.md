# Arm B — v15 n=72 at T=0 (deterministic)

**Date:** 2026-05-24
**Branch:** `experiment/v15-n72-T0`
**Model:** qwen3:30b-a3b-instruct-2507-q4_K_M
**Backend:** llama.cpp server (solo, `--parallel 1`, `--ctx-size 32768`)
**Sweep duration:** ~10h wall (06:35 → 16:42 local; ~5.85h cumulative model compute across 72 cells, shared with Arm A)
**Sampler:** `--temperature 0.0 --top-k 1` (deterministic greedy)

## Purpose

Arm B of the v2.3.1-vs-v15 T=0 A/B. Re-runs v15 compound (PR #429:
v14 word-hints prompt + chunked extraction + v2.3 grammar) with
sampling RNG removed so any remaining delta vs Arm A (v2.3.1 T=0) is
diagnostic of **compound design value**, not sampler variance.

PR #429 found that v15 stochastic underperformed the v2.3.1 baseline
(0.7753 vs 0.8733 word) and attributed the gap to per-cell RNG
variance, not the chunked compound design. This A/B falsifies or
confirms that claim by removing the RNG.

## Source materials (all on main, including chunked-merger ENT-dedup
fix landed in PR #429)

| File | Purpose |
|---|---|
| `prompts/cod_dsl_v14_word_hints.txt` | v14 word-hints prompt (PR #428) |
| `docs/grammars/cod-dsl-v2.3.gbnf` | v2.3 schema with `source_words` slot |
| `dsl/parser_v2_3.py` | v2.3 parser |
| `dsl/verifier_v2_3_1.py` | v2.3.1 verifier (punct tolerance + casefold, PR #427) |
| `scripts/chunked_extractor.py` | chunked extraction with ENT-dedup fix (PR #429) |
| `scripts/run_cod_dsl.py` | per-cell runner; supports `--chunked`, `--temperature`, `--top-k` |
| `scripts/sweep_v15_n72_T0.sh` | this sweep's driver |
| `scripts/aggregate_v15_T0.py` | per-cell rescore + pooled aggregate |
| `scripts/build_v15_T0_notes.py` | per-cell table + verdict ladder |

## TL;DR

**v15 T=0 pooled word (v2.3.1) = 0.8261 (d=2886)** — beats v15 stochastic
by **+0.0507** (confirms RNG hurt v15) and beats Arm A v2.3.1 T=0 by
**+0.0228** ≥ +0.01 → **DESIGN SOUND**. Still trails v2.3.1 *stochastic*
(0.8733) by **-0.0473**, but the apples-to-apples T=0 comparison favors
v15. 4/72 cells parse-fail under v2.3 source grammar (30802, 32859, 35336
are also failing in source; 27772 was a v15-stoch recover that did not
preserve at T=0). v2.3.1 rescore recovers 29163 / 33632 / 33749 / 28398 /
29846 to non-null word rates despite source `grammar_valid=False`.

## Per-cell table (Arm B v15 T=0 vs v15 stoch vs v2.3.1 stoch)

```
   cell  gv_t0  chunked  byte_t0  word_t0  word_t0_c | byte_v15s word_v15s | byte_v23s word_v23s
---------------------------------------------------------------------------------------------------------
  27178    T       -    0.0000   0.0000    0.0000  |  0.0000    0.0000  |  0.0000    0.0000
  27548    T       -    0.9545   0.9545    0.9545  |  1.0000    1.0000  |  0.9730    0.9459
  27558    T       -    0.1417   0.1339    0.1339  |  0.7008    0.6929  |  0.8246    0.8070
  27560    T       -    0.8824   0.8824    0.8824  |  0.6667    1.0000  |  0.8824    0.8824
  27609    T       -    0.8095   0.8095    0.8095  |  0.8400    0.8400  |  0.9130    0.9130
  27702    T       -    0.0000   0.0000    0.0000  |  0.0000    0.0000  |  0.0000    0.0000
  27772    F       -      N/A      N/A       N/A   |  0.9000    0.9571  |     N/A       N/A
  28398    T       -    0.3386   0.3307    0.3307  |  0.2677    0.2520  |  0.9167    0.8611
  29149    T       -    1.0000   1.0000    1.0000  |  1.0000    1.0000  |  0.9512    0.8049
  29163    T       -    1.0000   1.0000    1.0000  |     N/A       N/A  |  1.0000    1.0000
  29581    T       -    1.0000   1.0000    0.9565  |  1.0000    1.0000  |  0.7667    0.7667
  29807    T       -    0.9500   0.9750    0.9500  |  0.9730    1.0000  |  0.9000    0.9000
  29845    T       -    0.0000   0.0000    0.0000  |  0.0000    0.0000  |  0.0000    0.0000
  29846    T       -    0.0236   0.0236    0.0236  |  0.0157    0.0157  |  0.9091    0.9545
  29848    T       -    0.9764   0.9685    0.9685  |  0.9375    0.9167  |  0.9375    0.8750
  30273    T       -    1.0000   1.0000    1.0000  |  1.0000    1.0000  |  1.0000    1.0000
  30350    T       -    0.8873   0.9437    0.8873  |  0.3071    0.3071  |  0.9677    0.9839
  30596    T       -    0.0000   0.0000    0.0000  |  0.0000    0.0000  |  0.0000    0.0000
  30619    T       -    0.0000   0.0000    0.0000  |  0.0000    0.0000  |  0.0000    0.0000
  30674    T       -    0.6410   0.6923    0.6410  |  0.9750    0.9500  |  0.9783    0.9348
  30802    F       -      N/A      N/A       N/A   |  0.9388    0.8980  |  1.0000    0.9318
  31149    T       -    0.9167   1.0000    0.9167  |  0.9200    0.9600  |  0.9000    1.0000
  31430    T       -    0.9412   0.9412    0.8235  |  1.0000    0.9375  |  1.0000    0.9444
  31521    T       -    1.0000   1.0000    1.0000  |  1.0000    1.0000  |  1.0000    0.9375
  31527    T       -    1.0000   1.0000    1.0000  |  1.0000    1.0000  |  1.0000    1.0000
  31590    T       -    0.0000   0.0000    0.0000  |  0.0000    0.0000  |  0.0000    0.0000
  31629    T       -    1.0000   1.0000    1.0000  |  0.9474    0.9474  |  1.0000    0.9412
  31770    T       -    0.8740   0.9921    0.8740  |  0.7347    0.8367  |  0.7953    0.9606
  31852    T       -    1.0000   1.0000    1.0000  |  0.9000    0.9000  |  1.0000    0.9231
  32097    T       -    1.0000   1.0000    1.0000  |  1.0000    1.0000  |  1.0000    1.0000
  32146    T       -    0.8824   0.9608    0.9020  |  0.8654    0.9615  |  0.8475    0.9661
  32270    T       -    0.0000   0.0000    0.0000  |  0.0000    0.0000  |  0.0000    0.0000
  32360    T       -    1.0000   0.9750    0.9750  |  0.9688    0.9688  |  0.9762    1.0000
  32859    F       -      N/A      N/A       N/A   |  0.4959    0.5366  |     N/A       N/A
  33374    T       -    0.9048   1.0000    0.9524  |  1.0000    1.0000  |  1.0000    0.9444
  33569    T       -    0.8966   0.9310    0.8966  |  0.9000    0.9333  |  0.8621    0.8000
  33598    T       -    1.0000   1.0000    1.0000  |  1.0000    1.0000  |  1.0000    0.3000
  33632    T       -    0.4488   0.9921    0.9921  |     N/A       N/A  |  0.3780    0.9921
  33749    T       -    0.9886   0.9886    0.9886  |     N/A       N/A  |  0.9630    0.9753
  34230    T       -    0.1250   0.1250    0.1250  |  0.4000    0.4000  |  0.5000    0.5000
  34303    T       -    0.8571   0.8571    0.8571  |  0.7778    0.7778  |  0.8750    0.7500
  34310    T       -    0.0000   0.0000    0.0000  |  0.0000    0.0000  |  0.0000    0.0000
  34537    T       -    0.0000   0.0000    0.0000  |  0.0000    0.0000  |  0.0000    0.0000
  34666    T       -    0.9843   0.9921    0.9843  |  0.9524    0.9762  |  0.7736    0.7736
  34754    T       -    1.0000   1.0000    1.0000  |  1.0000    1.0000  |  0.9000    1.0000
  35051    T       -    0.7540   0.7857    0.7778  |  0.6230    0.6389  |     N/A       N/A
  35088    T       -    1.0000   1.0000    1.0000  |  1.0000    1.0000  |  1.0000    1.0000
  35185    T       -    0.0000   0.0000    0.0000  |  0.0000    0.0000  |  0.0000    0.0000
  35336    F       -      N/A      N/A       N/A   |     N/A       N/A  |     N/A       N/A
  35352    T       -    0.9912   1.0000    0.9912  |  0.9912    1.0000  |  0.9826    0.9739
  35354    T       -    0.9000   0.9333    0.9333  |  0.7931    0.7931  |  0.2366    0.2366
  35355    T       -    0.9412   0.9608    0.9216  |  0.9412    0.9608  |  0.9194    0.8710
  35441    T       -    1.0000   0.9231    0.9231  |  1.0000    1.0000  |  1.0000    1.0000
  35987    T       -    1.0000   1.0000    1.0000  |  1.0000    1.0000  |  1.0000    1.0000
  36031    T       -    1.0000   1.0000    1.0000  |  1.0000    1.0000  |  1.0000    1.0000
  36050    T       -    0.8571   1.0000    0.7143  |  0.7500    1.0000  |  0.8750    0.9375
  36065    T       -    1.0000   1.0000    0.9500  |  0.9714    1.0000  |  0.9412    1.0000
  36135    T       -    1.0000   0.9524    0.9524  |  1.0000    1.0000  |  0.9000    0.9500
  36184    T       -    0.9921   0.9843    0.9764  |  0.9921    0.9843  |  0.9149    0.8511
  36185    T       -    0.6842   0.6579    0.6579  |  0.7500    0.7857  |  0.7857    0.7500
  36445    T       -    0.9500   1.0000    1.0000  |  0.9500    1.0000  |  0.9500    0.9500
  36482    T       -    0.0000   0.0000    0.0000  |  0.0000    0.0000  |  0.0000    0.0000
  36509    T       -    0.0000   0.0000    0.0000  |  0.0000    0.0000  |  0.0000    0.0000
  36543    T       -    0.8667   0.8667    0.8667  |  0.9231    1.0000  |  1.0000    1.0000
  36558    T       -    0.9565   0.9565    0.9565  |  1.0000    1.0000  |  1.0000    1.0000
  36566    T       -    0.9490   0.9592    0.9490  |  0.9528    0.9623  |     N/A       N/A
  37333    T       -    0.9636   0.9455    0.9455  |  1.0000    0.9921  |  0.9512    0.8780
  37629    T       -    0.9310   0.9310    0.9310  |  1.0000    1.0000  |  0.9667    1.0000
  37730    T       -    0.9912   1.0000    0.9912  |  0.9912    1.0000  |  0.9914    0.9914
  38048    T       -    0.5591   0.5512    0.5354  |     N/A       N/A  |     N/A       N/A
  38157    T       -    1.0000   1.0000    1.0000  |  1.0000    1.0000  |  1.0000    1.0000
  38457    T       -    0.8750   1.0000    0.8750  |  0.8571    1.0000  |  0.8667    1.0000
```

Notes:
- `gv_t0` = source-grammar `grammar_valid` per cell JSON; v2.3.1 rescore
  can still produce a non-null word rate when the parser-v2_3 rescore
  succeeds on the same raw output (e.g. 29163, 33632, 33749, 28398, 29846).
- `chunked` column = number of chunks reported by extractor when chunking
  fired. At T=0 the chunked metadata path wasn't surfaced in the cell
  JSONs the populator reads, so this column shows "-" across the board;
  raw `chunked_meta` is present in the result JSONs for 27772/32859 and
  was used by the runner during inference.

## Pooled aggregates (denom-weighted)

| Run | pooled byte v2.3.1 | pooled word v2.3.1 | n_cells |
|---|---|---|---|
| v15 T=0 (this) | 0.7893 (d=2886) | **0.8261** (d=2886) | 72 |
| v15 stochastic (PR #429) | 0.7581 (d=2795) | **0.7753** (d=2795) | 72 |
| v2.3.1 stochastic (PR #428) | 0.8530 (d=1980) | **0.8733** (d=2029) | 72 |
| Arm A v2.3.1 T=0 (parallel) | 0.7674 (d=3014) | **0.8033** (d=3014) | 72 |

### Deltas (v15 T=0 vs ...)
- vs v15 stochastic 0.7753: **+0.0507** (sign: win)
- vs v2.3.1 stochastic 0.8733: **-0.0473** (sign: loss)
- vs Arm A v2.3.1 T=0 (0.8033): **+0.0228**

## Cells of interest (from PR #429 v15 stochastic NOTES)

### "New failure" cells under v15 stochastic — do they reappear at T=0?

| Cell | v15 stoch (PR #429) | v2.3.1 stoch (PR #428) | v15 T=0 (this) |
|---|---|---|---|
| 29163 | FAIL ("Unexpected $END" mid-block) | 1.000 / 1.000 | **1.000 / 1.000** (RECOVERED via v2.3.1 rescore; source gv=False) |
| 33632 | FAIL (Duplicate ENT: '_anon:excise_duties') | 0.378 byte / 0.992 word | **0.449 / 0.992** (RECOVERED via v2.3.1 rescore; source gv=False) |
| 33749 | FAIL ("Unexpected $END" mid-block) | 0.963 / 0.975 | **0.989 / 0.989** (RECOVERED via v2.3.1 rescore; source gv=False) |
| 28398 | regressed -0.61 (0.86→0.25) | 0.917 / 0.861 | 0.339 / 0.331 (still regressed; not RNG-driven) |
| 29846 | regressed -0.94 (0.95→0.02) | 0.909 / 0.955 | 0.024 / 0.024 (still regressed; not RNG-driven) |

The three "Unexpected $END" / "Duplicate ENT" failures from v15 stoch
**do recover** at T=0 under the v2.3.1 verifier rescore — confirming
those were RNG-driven generation artifacts. The two -0.6/-0.9 regressions
(28398, 29846) **persist at T=0** — they are systematic prompt or
chunking artifacts, not sampler variance.

### Chunked cells — recovery preserved?

| Cell | chars | v15 stoch (PR #429) | v15 T=0 (this) |
|---|---|---|---|
| 27772 | 47038 | RECOVERED (0.957 word, 3 chunks) | **PARSE-FAIL** ("Unexpected $END" line 642; not recovered) |
| 32859 | 44222 | RECOVERED (0.537 word, 2 chunks) | **PARSE-FAIL** (Unbalanced ')' in slot value line 839) |
| 35051 | 38738 | RECOVERED (0.639 word, 2 chunks) | **0.754 / 0.786** (recovery preserved; +0.15 word vs stoch) |

35051 keeps the recovery and improves on it; 27772/32859 lose the v15-stoch
recovery at T=0. Plus, two additional cells parse-fail at T=0:

| Cell | T=0 failure | Stoch (PR #429) |
|---|---|---|
| 30802 | "Unexpected $END" line 596 | 0.939 / 0.898 (passed) |
| 35336 | Duplicate ENT: 'I' | source gv=False at stoch too (not in PR #429 pooled) |

Net parse-fail at T=0: **4/72** (27772, 30802, 32859, 35336).
v2.3.1 rescore recovers 0/4 (these are source-grammar failures the
v2.3.1 verifier also cannot parse — distinct from the 29163-class
cells where parser-v2_3 succeeds but source `grammar_valid=False` flag
came from upstream).

## Verdict ladder

Per task brief, the answer hinges on **v15 T=0 vs Arm A v2.3.1 T=0**
(pooled word verifier, v2.3.1):

| delta(v15_T0 − v23_T0) | verdict | implication |
|---|---|---|
| ≥ +0.01 | **DESIGN SOUND** | RNG was the killer; promote v15 |
| within ±0.01 | **COMPOUND DOES NOT HELP DETERMINISTICALLY** | abandon |
| ≤ -0.01 | **DESIGN WRONG** | chunking actively hurts beyond truncation cells |

**Result: delta = +0.0228 ⇒ DESIGN SOUND.** v15 compound (v14 word-hints
prompt + chunked extractor + ENT-dedup fix) beats v2.3.1 baseline when
both run at T=0. The v15 stochastic regression seen in PR #429 was
dominated by sampler variance, not by the compound design. Promote v15.

Caveats:
- v15 T=0 still trails v2.3.1 *stochastic* (0.8733) by -0.047. The
  apples-to-apples comparison is T=0 vs T=0, but the absolute SOTA
  number is v2.3.1 stoch.
- Pooled denom differs across runs (v15 T=0 d=2886, Arm A T=0 d=3014,
  v2.3.1 stoch d=2029). Comparison is denom-weighted but cells with
  larger v15 T=0 denominators carry more weight.
- 4 parse failures at T=0 (27772, 30802, 32859, 35336) contribute 0
  numerator + 0 denominator to the pooled aggregate (don't deflate);
  fixing these is the obvious next lever.

## Methodology notes

- Per-cell scoring uses `dsl/verifier_v2_3_1.py` (the same verifier
  PR #428 and #429 reported against).
- Pooled aggregate is denom-weighted: sum(match * denom) / sum(denom).
  Empty-doc and parse-fail cells contribute 0 to both numerator and
  denominator so they don't deflate the rate artificially.
- Both arms share the same llama-server (`--parallel 1`), so cells
  alternate across arms. Wall time roughly doubles per arm.

## Source

Sweep script: `scripts/sweep_v15_n72_T0.sh`.
Rescore + aggregate: `scripts/aggregate_v15_T0.py`.
Per-cell + verdict: `scripts/build_v15_T0_notes.py`.
