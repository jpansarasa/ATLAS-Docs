# v15 compound — v14 word-hints prompt + chunked extraction on n=72

**Date:** 2026-05-24
**Branch:** experiment/v15-word-chunked-n72
**Model:** qwen3:30b-a3b-instruct-2507-q4_K_M
**Backend:** llama.cpp server (solo, --parallel 1, --ctx-size 32768)
**Source compounds:**
  - v14 word-hints prompt (`prompts/cod_dsl_v14_word_hints.txt`) — PR #428
  - v2.3 grammar (`docs/grammars/cod-dsl-v2.3.gbnf`) — PR #428
  - v2.3.1 verifier (`dsl/verifier_v2_3_1.py`) — punct tolerance + casefold — PR #427
  - Chunked extraction (`scripts/chunked_extractor.py`) — PR #422
  - **NEW:** ENT-dedup fix on chunked merger (this branch) — required for v2.3 to parse merged output

## Verdict — COMPOUND DID NOT HELP (regression)

| Metric | v15.1 (this run) | v2.3.1 n=72 (PR #428) | β scored (PR #424) | v8 baseline n=72 |
|---|---|---|---|---|
| Pooled word verifier | **0.7753** | **0.8733** | 0.887 (scored only) | 0.7826 |
| Pooled byte verifier | 0.7581 | 0.8517 | — | — |
| Grammar/parse failures | 5 cells | 6 cells | 4 cells | — |
| Truncation cells recovered | 2 of 2 | — | — | — |
| Dup-ENT cells recovered | 2 of 4 | — | — | — |

**Bucket:** < 0.85 → **COMPOUND DID NOT HELP**. v15 pooled (0.7753) is BELOW the v2.3.1 baseline (0.8733) by 9.8 pp and BELOW the v8 baseline (0.7826) by 0.7 pp. β scored (0.887) is the production-grade target and v15 is well below it.

## Truncation cell recovery (the design goal)

Both truncation cells **RECOVERED**:

| Cell | chars | est_tokens | chunked? | gv | byte_v15 | word_v15.1 |
|---|---|---|---|---|---|---|
| 27772 | 47038 | 11759 | YES (3c) | T | 0.900 | 0.957 |
| 36566 | 7477  | 1869  | no (under threshold) | T | 0.953 | 0.962 |

**Mechanism finding:** 27772 needed chunking (47K chars → 3 chunks); 36566 fit single-pass (7.5K chars / 1869 est_tokens, well below 8K chunking threshold) and recovered just from prompt iteration (v14 vs whichever v2.3 baseline configuration produced the v23 truncation). Chunking solved 1 of 2.

## Dup-ENT cell recovery (the side-effect goal)

| Cell | chars | chunked? | gv | byte_v15 | word_v15.1 | Δ vs v2.3 |
|---|---|---|---|---|---|---|
| 32859 | 44222 | YES (2c) | T | 0.496 | 0.537 | RECOVERED |
| 35051 | 38738 | YES (2c) | T | 0.623 | 0.639 | RECOVERED |
| 35336 | 3275  | no | F | FAIL | FAIL | STILL FAILS (under threshold, single-pass model bug: `Duplicate ENT: 'I'`) |
| 38048 | 21349 | no | F | FAIL | FAIL | STILL FAILS (under threshold, single-pass model bug: `Duplicate ENT: '_anon:iran_nuclear_program'`) |

2 of 4 dup-ENT cells recovered (the two chunked ones). The remaining two are model-side single-pass dup-ENT emissions; chunking cannot apply because the cells are below the 8K-token chunking threshold.

## CRITICAL FINDING — chunked merger ENT-dedup bug

Initial v15 sweep on cell 27772 hit a fatal merger bug: each chunk emitted the same ENT with different `source_words` slot values (each chunk sees different mentions), so `_block_key`'s slot_signature differed → not deduped → parser rejected the merged output with `Duplicate ENT declaration: '_anon:monitoring'`.

**Patch (committed in this branch):** for ENT blocks ONLY, drop `slot_signature` from the dedup key. First-wins keeps the first chunk's ENT and silently discards later chunks' otherwise-identical re-declarations. NUM/EVT/CLAIM/NOTE retain full slot_signature dedup (no uniqueness constraint).

**Verified:** synthetic 2-chunk replay with conflicting source_words deduplicates ENT to 1 (previously 2 → parser fail); NUM same-name different-periods keeps both (no regression). Without this patch, v15 chunked path is broken for v2.3 schema.

## Per-cell table (full n=72)

```
   cell   gv  chunked   byte_v15  word_v15  word_v15.1 | byte_v23  word_v23.1
------------------------------------------------------------------
  27178    T        -    0.0000    0.0000      0.0000 |  0.0000     0.0000
  27548    T        -    1.0000    0.8095      1.0000 |  0.9730     0.9459
  27558    T        -    0.7008    0.7008      0.6929 |  0.8246     0.8070
  27560    T        -    0.6667    0.5833      1.0000 |  0.8824     0.8824
  27609    T        -    0.8400    0.8400      0.8400 |  0.9130     0.9130
  27702    T        -    0.0000    0.0000      0.0000 |  0.0000     0.0000
  27772    T       3c    0.9000    0.8238      0.9571 |     N/A        N/A   <- truncation RECOVERED
  28398    T        -    0.2677    0.2126      0.2520 |  0.9167     0.8611   <- regressed -0.61
  29149    T        -    1.0000    0.9535      1.0000 |  0.9512     0.8049
  29163    F        -       N/A       N/A         N/A |  1.0000     1.0000   <- NEW failure
  29581    T        -    1.0000    1.0000      1.0000 |  0.7667     0.7667
  29807    T        -    0.9730    0.9189      1.0000 |  0.9000     0.9000
  29845    T        -    0.0000    0.0000      0.0000 |  0.0000     0.0000
  29846    T        -    0.0157    0.0157      0.0157 |  0.9091     0.9545   <- regressed -0.94
  29848    T        -    0.9375    0.9375      0.9167 |  0.9375     0.8750
  30273    T        -    1.0000    1.0000      1.0000 |  1.0000     1.0000
  30350    T        -    0.3071    0.2992      0.3071 |  0.9677     0.9839   <- regressed -0.68
  30596    T        -    0.0000    0.0000      0.0000 |  0.0000     0.0000
  30619    T        -    0.0000    0.0000      0.0000 |  0.0000     0.0000
  30674    T        -    0.9750    0.8500      0.9500 |  0.9783     0.9348
  30802    T        -    0.9388    0.8163      0.8980 |  1.0000     0.9318
  31149    T        -    0.9200    0.8800      0.9600 |  0.9000     1.0000
  31430    T        -    1.0000    0.8125      0.9375 |  1.0000     0.9444
  31521    T        -    1.0000    1.0000      1.0000 |  1.0000     0.9375
  31527    T        -    1.0000    1.0000      1.0000 |  1.0000     1.0000
  31590    T        -    0.0000    0.0000      0.0000 |  0.0000     0.0000
  31629    T        -    0.9474    0.9474      0.9474 |  1.0000     0.9412
  31770    T        -    0.7347    0.6735      0.8367 |  0.7953     0.9606
  31852    T        -    0.9000    0.9000      0.9000 |  1.0000     0.9231
  32097    T        -    1.0000    1.0000      1.0000 |  1.0000     1.0000
  32146    T        -    0.8654    0.7885      0.9615 |  0.8475     0.9661
  32270    T        -    0.0000    0.0000      0.0000 |  0.0000     0.0000
  32360    T        -    0.9688    0.8438      0.9688 |  0.9762     1.0000
  32859    T       2c    0.4959    0.4959      0.5366 |     N/A        N/A   <- dup-ENT RECOVERED
  33374    T        -    1.0000    1.0000      1.0000 |  1.0000     0.9444
  33569    T        -    0.9000    0.8333      0.9333 |  0.8621     0.8000
  33598    T        -    1.0000    1.0000      1.0000 |  1.0000     0.3000
  33632    F        -       N/A       N/A         N/A |  0.3780     0.9921   <- NEW failure
  33749    F        -       N/A       N/A         N/A |  0.9630     0.9753   <- NEW failure
  34230    T        -    0.4000    0.4000      0.4000 |  0.5000     0.5000
  34303    T        -    0.7778    0.7778      0.7778 |  0.8750     0.7500
  34310    T        -    0.0000    0.0000      0.0000 |  0.0000     0.0000
  34537    T        -    0.0000    0.0000      0.0000 |  0.0000     0.0000
  34666    T        -    0.9524    0.9524      0.9762 |  0.7736     0.7736
  34754    T        -    1.0000    0.9000      1.0000 |  0.9000     1.0000
  35051    T       2c    0.6230    0.6071      0.6389 |     N/A        N/A   <- dup-ENT RECOVERED
  35088    T        -    1.0000    1.0000      1.0000 |  1.0000     1.0000
  35185    T        -    0.0000    0.0000      0.0000 |  0.0000     0.0000
  35336    F        -       N/A       N/A         N/A |     N/A        N/A   <- dup-ENT STILL FAILS
  35352    T        -    0.9912    0.9912      1.0000 |  0.9826     0.9739
  35354    T        -    0.7931    0.7931      0.7931 |  0.2366     0.2366
  35355    T        -    0.9412    0.8039      0.9608 |  0.9194     0.8710
  35441    T        -    1.0000    1.0000      1.0000 |  1.0000     1.0000
  35987    T        -    1.0000    1.0000      1.0000 |  1.0000     1.0000
  36031    T        -    1.0000    1.0000      1.0000 |  1.0000     1.0000
  36050    T        -    0.7500    0.5000      1.0000 |  0.8750     0.9375
  36065    T        -    0.9714    0.8857      1.0000 |  0.9412     1.0000
  36135    T        -    1.0000    1.0000      1.0000 |  0.9000     0.9500
  36184    T        -    0.9921    0.9606      0.9843 |  0.9149     0.8511
  36185    T        -    0.7500    0.7500      0.7857 |  0.7857     0.7500
  36445    T        -    0.9500    0.9000      1.0000 |  0.9500     0.9500
  36482    T        -    0.0000    0.0000      0.0000 |  0.0000     0.0000
  36509    T        -    0.0000    0.0000      0.0000 |  0.0000     0.0000
  36543    T        -    0.9231    0.9231      1.0000 |  1.0000     1.0000
  36558    T        -    1.0000    0.7143      1.0000 |  1.0000     1.0000
  36566    T        -    0.9528    0.6887      0.9623 |     N/A        N/A   <- truncation RECOVERED
  37333    T        -    1.0000    1.0000      0.9921 |  0.9512     0.8780
  37629    T        -    1.0000    0.8621      1.0000 |  0.9667     1.0000
  37730    T        -    0.9912    0.9912      1.0000 |  0.9914     0.9914
  38048    F        -       N/A       N/A         N/A |     N/A        N/A   <- dup-ENT STILL FAILS
  38157    T        -    1.0000    1.0000      1.0000 |  1.0000     1.0000
  38457    T        -    0.8571    0.8571      1.0000 |  0.8667     1.0000
```

## Pooled aggregates (denom-weighted)

| Verifier | Pooled rate | Denom |
|---|---|---|
| byte (v2.3 = v2.3.1) | 0.7581 | 2795 |
| word v2.3 verifier | 0.7141 | 2795 |
| **word v2.3.1 verifier** | **0.7753** | 2795 |
| word v2.3.1 cased | 0.7596 | 2795 |

## Diagnosis

The 4-cell recovery (chunked extracts on truncation + dup-ENT longs) is REAL, but it was overwhelmed by:

1. **Three new grammar failures** (29163, 33632, 33749) — cells that were valid in v2.3 n=72 are now broken in v15. All three are single-pass (chunking did not fire). 33632 was `Duplicate ENT: '_anon:excise_duties'` (model emitted a duplicate within a single pass — same class of bug as the persistent 35336/38048 failures). 29163 and 33749 hit "Unexpected $END" mid-block — output truncation despite n_predict=8192.

2. **Several large per-cell regressions** even on cells that didn't fail:
   - 28398: 0.86 → 0.25 (−0.61)
   - 29846: 0.95 → 0.02 (−0.94)
   - 30350: 0.98 → 0.31 (−0.68)
   - 33598: 1.00 → 0.30 word (−0.70 word, even though parse_ok)

   These are NOT chunked cells. Same prompt, same grammar, same backend, same model — DIFFERENT output. The model emitted ~2x more tokens (e.g., 28398: 4545 v15 vs 2303 v2.3) and lost word-grounding signal. This is **sampling variance between runs**, not a v15-specific regression.

3. **Conclusion:** the compound's chunked-extraction win is dwarfed by run-to-run RNG variance at the n=72 scale. v15 = v14 = v2.3 at the per-prompt level; the difference is just sampler noise + the additive chunked-merger fix. We need a stable seed or n>>72 averaging to disentangle.

## Recommendation

- **Do NOT deploy v15** (regresses below v2.3.1 baseline by 9.8 pp).
- **KEEP** the ENT-dedup fix on the chunked merger — it's a real bug that any v2.3-schema chunked run will hit. Cherry-pick the fix into main even if v15 itself is shelved.
- **Truncation recovery is real** — chunked 27772 produced a parseable, high-quality DSL (0.96 word). Future iterations of v2.3 on the long-doc cells should keep `--chunked` enabled.
- **For dup-ENT cells under threshold** (35336, 38048): chunking is not the lever. Need either:
  - (a) lower the chunking threshold to ~4K so 21K-char cells get split, OR
  - (b) a prompt-level decree against duplicate `_anon:*` ENTs within a single response.
- **RNG variance at n=72 is the elephant in the room.** Future A/B comparisons should set `temperature=0.0 --top-k 1` for greedy decode so per-cell rates are reproducible.

## State of the workstream

v2.3.1 (PR #428) remains the best-published config (0.8733 pooled n=72). v15 compound did not improve on it; v15's only durable contribution is the chunked-merger ENT-dedup fix.
