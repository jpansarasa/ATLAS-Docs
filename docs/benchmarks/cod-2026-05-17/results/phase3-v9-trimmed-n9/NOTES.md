# Phase 3 v9 trimmed layered — n=9 sweep notes

**Branch:** `experiment/phase3-v9-trimmed-n9`
**Date:** 2026-05-22
**Model:** `qwen3:30b-a3b-instruct-2507-q4_K_M` (Class B MoE, 4B active)
**Variant:** `v2_1_spec`
**Prompt:** `cod_dsl_v9_trimmed_layered.txt` (773 lines)
**Server:** llama-server solo (`--parallel 1 / --ctx-size 32768`) per PR #407
**Baselines:** PR #410 v8 layered (0.907, gv 9/9), PR #400 v6 leaner (0.895, gv 9/9)

## TL;DR

- **Aggregate verifier (n=9): 0.8595** (v8 0.907; v6 0.895)
- **Paired delta vs v8: -0.0474 mean** (6 wins, 1 catastrophic loss, 2 ties)
- **gv pass rate: 8/9** (vs v8 9/9). Cell **33632 dup-decl regression** — the exact
  failure mode that v7 hit in PR #409. The DECLARE-ONCE preservation alone
  was insufficient.
- **Verdict: REGRESSION** (< 0.87). v9's safe-trim hypothesis is FALSIFIED for
  this model.

## Intervention

v9 = v8 (PR #410, 0.907) with v7's prose collapses applied to the BASIC SHAPE
and ANON-REFERENCE worked examples. DECLARE-ONCE worked example preserved
byte-identical (the section v7 dropped that broke 33632 in PR #409).

Trims applied:
- **BASIC SHAPE**: "What this example demonstrates" + "CRITICAL — anti-patterns"
  numbered list (28 lines) → v7's compact "Key points" paragraph (10 lines)
- **ANON-REFERENCE**: "What goes wrong, block by block" + "Notes" +
  "ANTI-PATTERN SUMMARY for ANON-REFERENCE DISCIPLINE" (~58 lines) → v7's
  compact "Key points" paragraph (9 lines)
- Header label tweaks + inline-comment collapses (~5 lines)

Preserved byte-identical from v8:
- All v8 additive type-aware decrees (lines 1-460)
- **DECLARE-ONCE WORKED EXAMPLE body** (every word)
- All trailing OUTPUT DIRECTIVE / INPUT directives

## Prompt size

| version | lines | rel-v8 |
| ------- | ----- | ------ |
| v6 leaner   | 634 | -207 |
| v7 even lean | 528 | -313 |
| v8 layered  | 841 |    — |
| **v9 trimmed** | **773** | **-68** |

## Per-cell results (v9)

| Cell  | gv | vpr    | undecl | ent | concept-block | num_recall | pe    | ev   | wall (s) | stop |
| ----- | -- | ------ | ------ | --- | ------------- | ---------- | ----- | ---- | -------- | ---- |
| 27560 | T  | 0.8750 | 0      | 3   | 0             | 0.000      | 8735  | 7997 | 959      | eos  |
| 27702 | T  | 1.0000 | 0      | 0   | 0             | 0.000      | 8766  | 86   | 47       | eos  |
| 29807 | T  | 1.0000 | 0      | 9   | 0             | 0.667      | 8788  | 1148 | 92       | eos  |
| 31149 | T  | 0.9600 | 0      | 11  | 5             | 1.000      | 8669  | 956  | 81       | eos  |
| 31430 | T  | 0.9583 | 0      | 10  | 2             | 0.556      | 9043  | 959  | 85       | eos  |
| **33632** | **F** | **0.0000** | **0** | **0** | **125** | **0.000** | **10289** | **2501** | **175** | **eos** |
| 34537 | T  | 1.0000 | 0      | 0   | 0             | 0.000      | 8575  | 61   | 44       | eos  |
| 35352 | T  | 1.0000 | 0      | 1   | 0             | 0.009      | 10958 | 5529 | 539      | eos  |
| 36065 | T  | 0.9420 | 0      | 32  | 17            | 0.243      | 12637 | 2408 | 634      | eos  |

## Side-by-side vpr deltas

| Cell  | v6     | v8     | v9     | v9 - v8  | v9 - v6  |
| ----- | ------ | ------ | ------ | -------- | -------- |
| 27560 | 0.6667 | 0.6667 | 0.8750 | +0.2083  | +0.2083  |
| 27702 | 1.0000 | 1.0000 | 1.0000 | +0.0000  | +0.0000  |
| 29807 | 0.8125 | 0.7353 | 1.0000 | +0.2647  | +0.1875  |
| 31149 | 0.8667 | 0.9583 | 0.9600 | +0.0017  | +0.0933  |
| 31430 | 0.9048 | 0.9167 | 0.9583 | +0.0417  | +0.0535  |
| **33632** | **0.9219** | **0.9688** | **0.0000** | **-0.9688** | **-0.9219** |
| 34537 | 1.0000 | 1.0000 | 1.0000 | +0.0000  | +0.0000  |
| 35352 | 0.9911 | 0.9913 | 1.0000 | +0.0087  | +0.0089  |
| 36065 | 0.8923 | 0.9250 | 0.9420 | +0.0170  | +0.0497  |
| **AGG**   | **0.895**  | **0.907**  | **0.8595** | **-0.0474** | **-0.0355** |

Paired sum vs v8 = +0.5421 from 8 cells, -0.9688 from cell 33632 → net -0.4267.
**If 33632 had matched its v8 score (0.9688), v9 would aggregate to 0.9671.**

## Mechanism preservation (vs brief checklist)

| Cell  | mechanism                | v9 status                          | v8 status |
| ----- | ------------------------ | ---------------------------------- | --------- |
| **33632** | **dup-decl (CRITICAL)** | **FAIL** — `_anon:market_intervention` (and 9 others) declared 4-5x each, gv=F | PASS — 119 ENTs, 0 dups |
| 36065 | kind-latch (concept cap) | PASS (17 concept, vpr 0.942 ≥ v8)  | PASS (3 concept) |
| 27702 | punctuation-ident        | PASS (vpr 1.0)                     | PASS |
| 31149 | declare-then-reference   | PASS (0 undecl, vpr 0.96)          | PASS (0 undecl) |

### 33632 failure analysis

- Output emitted 127 ENT declarations; 90 of them were **`_anon:` `/ concept`** blocks
  (kind-priority decree #9 violation)
- Top duplicates (each declared 4-5 times):
  `_anon:energy_security`, `_anon:price_stabilisation`, `_anon:consumer_protection`,
  `_anon:supply_chain_resilience`, `_anon:market_intervention`, `_anon:price_cap`
- Parser bailed on second `ENT _anon:market_intervention` → `grammar_valid` False → vpr 0
- This is the SAME class of failure that broke v7 in PR #409 on
  `_anon:coal_fired_power_generation_utilisation_target_80_percent_policy`

**Root cause hypothesis**: the ANON-REFERENCE Notes/ANTI-PATTERN section I
trimmed contained an explicit reinforcement of decree #9 kind-priority:
*"Choose the MOST SPECIFIC kind per decree #9: aggregate economic variable →
macro_indicator; common-noun group of people → person; program shape → org.
NOT concept."* That sentence was load-bearing, not just stylistic. v8's
worked-example prose holds kind-priority discipline together on long-form
regulatory articles; collapsing it to "Key points" loses the anchor.

The DECLARE-ONCE worked example WAS preserved byte-identical, so its
mechanism guard alone is insufficient — the model needs both the
declare-once AND the kind-priority reinforcement to stay disciplined on
the 33632-class article.

## Token budget delta vs v8

| Cell  | v8 pe  | v9 pe  | Δ pe   |
| ----- | ------ | ------ | ------ |
| 27560 | 9664   | 8735   | -929   |
| 27702 | 9695   | 8766   | -929   |
| 29807 | 9717   | 8788   | -929   |
| 31149 | 9598   | 8669   | -929   |
| 31430 | 9972   | 9043   | -929   |
| 33632 | 11218  | 10289  | -929   |
| 34537 | 9504   | 8575   | -929   |
| 35352 | 11887  | 10958  | -929   |
| 36065 | 13566  | 12637  | -929   |

Prompt-eval delta: **-929 tokens/cell** (consistent across cells, matches
v8 - v9 prompt size of -68 lines × ~13.7 tokens/line). Output budget freed
as designed, but freed budget did not translate to better verifier pass rate.

## Verdict

**REGRESSION** (aggregate 0.8595 < 0.87 threshold).

- 6/9 cells improved vs v8 (5 of those by ≥0.01)
- 2/9 cells tied at 1.0
- **1/9 cell (33632) collapsed** to 0.0 from dup-decl, swamping aggregate
- Hypothesis falsified: DECLARE-ONCE worked-example preservation alone is
  not sufficient; the ANON-REFERENCE Notes/ANTI-PATTERN section also
  carries load-bearing kind-priority reinforcement
- Next iteration should preserve **both** the DECLARE-ONCE worked example
  AND the ANON-REFERENCE Notes paragraph (the "Choose the MOST SPECIFIC
  kind" sentence in particular)

## One-line STATE.md

v9 (v8 + safe-trim minus DECLARE-ONCE) REGRESSED at 0.8595 — cell 33632 collapsed via concept-cascade dup-decl; the ANON-REFERENCE Notes were also load-bearing for decree-#9 kind-priority on long regulatory articles.
