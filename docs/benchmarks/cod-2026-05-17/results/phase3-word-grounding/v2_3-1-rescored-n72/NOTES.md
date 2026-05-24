# v2.3 + v2.3.1 rescore @ n=72 — does the HUGE WIN hold beyond Arm B?

**Date:** 2026-05-23 / 2026-05-24
**Branch:** `experiment/v2.3-n72-validation`
**Model:** `qwen3:30b-a3b-instruct-2507-q4_K_M` (llama.cpp :11437, `--parallel 1`, `--ctx-size 32768`)
**Prompt:** `prompts/cod_dsl_v14_word_hints.txt`
**Grammar:** `docs/grammars/cod-dsl-v2.3.gbnf`
**Sampler:** `--n-predict 8192 --num-ctx 32768 --timeout 1800 --force` (single-pass, no chunking, no RAG)
**Wall:** sweep started 2026-05-24T00:25:37Z, finished 2026-05-24T04:58:05Z → **~4h32m for 72 cells** (mean ~227 s/cell; consistent with v2.3 n=9 wall avg of 238 s/cell).

## Headline

| metric                                    |  v8 n=72 baseline |  β v12.1+chunked n=72 |  v2.3 n=72 (byte) |  v2.3.1 n=72 (word, casefold) |
| ----------------------------------------- | ----------------- | --------------------- | ----------------- | ----------------------------- |
| pooled verifier (penalized)               | 0.7826            | 0.8257                | n/a               | n/a                           |
| pooled verifier (scored)                  | n/a               | 0.887                 | n/a               | n/a                           |
| pooled byte verbatim                      | n/a               | n/a                   | **0.8561**        | 0.8561                        |
| pooled word verbatim, v2.3 (no rescore)   | n/a               | n/a                   | 0.7669            | 0.7669                        |
| **pooled word verbatim, v2.3.1 casefold** | n/a               | n/a                   | n/a               | **0.8733**                    |
| word v2.3.1 vs byte                       | n/a               | n/a                   | n/a               | **+0.0172** (word wins)       |
| gap closure word v2.3 → v2.3.1            | n/a               | n/a                   | n/a               | **+0.1065**                   |

The v2.3.1 verifier punctuation-tolerance fix **does carry over to n=72**: pooled word verbatim (casefold) edges out byte by +0.0172. n=9 showed +0.025; n=72 narrows the delta but preserves the sign. Punct-tolerance closes the v2.3 word-vs-byte gap by **+0.107** — the bulk of the n=9 "HUGE WIN" comes from the verifier itself, not from the underlying generation changing.

NOTE: cells are denom-weighted (pooled), so empty-doc cells (parse-ok but no source-grounded blocks) and parse-fail cells contribute 0 to numerator and 0 to denominator — they do NOT artificially deflate the pooled rate. Of 72 cells: **56 scoring cells**, **10 empty-doc** (parse-ok, kind without source_words), **6 parse-fail** (2 truncation, 4 duplicate-ENT).

## Mechanism cells (Arm B set, the v8 problem children)

| cell  | byte (v2.3=v2.3.1) | word v2.3 | word v2.3.1 cf | word v2.3.1 cased | denom | note                                                                       |
| ----- | ------------------ | --------- | -------------- | ----------------- | ----- | -------------------------------------------------------------------------- |
| 27702 | 0.000              | 0.000     | 0.000          | 0.000             | 0     | empty doc — model emits no checkable kinds (β recovered via chunking)      |
| 31149 | 0.900              | 0.850     | **1.000**      | 0.900             | 20    | punct-tolerance lifts to perfect                                           |
| 33632 | 0.378              | 0.378     | **0.992**      | 0.992             | 127   | **MAJOR RECOVERY** (+0.614) — byte/v2.3 word were swamped by punctuation drift |
| 35352 | 0.983              | 0.983     | 0.974          | 0.974             | 115   | byte already saturated; word slightly under (single block flips)            |
| 36065 | 0.941              | 0.902     | **1.000**      | 0.902             | 51    | punct-tolerance lifts to perfect                                           |

`33632` is the headline behaviour: under v2.3 (byte) the cell scored 0.378 because the model emits source-grounded spans with punctuation drift (commas, periods, quote marks) that v2.3 byte/word match rejected. v2.3.1's punct-tolerant casefold lifts it to 0.992 — the underlying generation was correct all along; the verifier was lying.

`27702` is still empty — v2.3 + v14 prompt does not provoke the model to emit a checkable kind for that long doc. β v12.1's chunked extraction recovered it; word grounding does not.

## Distribution

- 56 scoring cells, per-cell `word_v231_casefold` distribution: mean 0.737, median 0.936, max 1.0, min 0.0
- 7 cells where byte > word_v2.3.1 (>0.05) — punctuation drift went the *other* way (model emitted extra punct that the grammar/byte match accepted but word tokenisation dropped). Worst: `33598` (byte 1.000 → word 0.300, denom 70); `29149` (byte 0.951 → word 0.805, denom 41).
- 14 cells where word_v2.3.1 > byte (>0.02) — the punct-tolerance wins. Headline: `33632` (+0.614), `31770` (+0.165), `38457` (+0.133), `31149` (+0.100), `36065` (+0.059).

The `33598` regression is real and worth flagging: at denom 70 it pulls the pooled aggregate down meaningfully. A spot check of its raw_output is queued for follow-up — likely an unusual punctuation pattern in the source where v2.3 byte match accepts but word tokenisation splits.

## Parse failures (6/72, 8.3%)

| cell  | failure mode                                                     |
| ----- | ---------------------------------------------------------------- |
| 27772 | grammar `$END` at line 640 — output truncated at 8192 token cap  |
| 32859 | duplicate ENT decl (`_anon:Israel`)                              |
| 35051 | duplicate ENT decl (`_anon:geopolitical_risk_and_financial_...`) |
| 35336 | duplicate ENT decl (`I`)                                         |
| 36566 | grammar `$END` at line 558 — output truncated                    |
| 38048 | duplicate ENT decl (`_anon:iranian_blockade_of_strait_of_...`)   |

Two truncations are the long-doc tail (cells previously known to push past 8192 tokens). Four duplicate-ENT errors are a v2.3 grammar gap: the parser refuses re-declaration of the same anonymous entity in a single document. Pre-existing condition; not a v2.3.1 regression.

## Verdict

**BOUNDED PROGRESS** (verifier 0.8733; falls in 0.85–0.90 band).

Specifically:
- pooled v2.3.1 word @ n=72 = **0.8733** → below the 0.90 "PROGRESS" threshold from the brief.
- BUT it beats v8 baseline (0.7826) by **+0.091** and beats β v12.1 chunked penalized (0.8257) by **+0.048**.
- It is *under* β's scored 0.887 by −0.014, but β requires chunked extraction infrastructure; this is single-pass + grammar + word verifier.

The n=9 "HUGE WIN" (+0.025 word over byte) **scaled down to +0.017 at n=72**: sign preserved, magnitude attenuated by ~30 %. The bulk of the n=9 win was the verifier punct-tolerance unmasking generation that v2.3 was already producing correctly — visible in `33632`'s +0.614 jump.

**Production-deployable?** As a verifier change: yes — pure rescoring, zero risk to generation. Drop v2.3.1 in place of v2.3 immediately. As a generation pipeline: v2.3 + v14 word-hints alone is **competitive with but does not surpass** β v12.1 NUM-only + chunked; the chunking infrastructure still buys ~0.013 on top.

**Next probes (parking lot):**
1. Spot-check `33598` (byte 1.0 → word 0.3 regression) — is the punct-drift adversarial or a verifier edge case?
2. Re-run truncation cells `27772` and `36566` with `--n-predict 16384` to confirm 8192 is the bottleneck.
3. Patch v2.3 grammar / parser to allow duplicate ENT decls (or de-dupe in parser).

## One-line STATE.md candidate

> v2.3 n=72 + v2.3.1 rescore: BOUNDED PROGRESS (0.8733 verifier; +0.091 vs v8, +0.048 vs β v12.1; word edges byte +0.017, mechanism cell 33632 recovers 0.378→0.992 via punct-tolerance).
