# Phase 3 v6 leaner prompt — n=9 sweep report

**Branch:** `experiment/phase3-v6-leaner-n9`
**Date:** 2026-05-22
**Model:** `qwen3:30b-a3b-instruct-2507-q4_K_M` (Class B MoE, 4B active)
**Variant:** `v2_1_spec`
**Prompt:** `cod_dsl_v6_leaner.txt`
**Baseline:** PR #397 v5 dedup (`phase3-v5-dedup-n9`)

## TL;DR

- **Prompt: 1033 → 634 lines (-399, -38.6%)**
- **Prompt tokens freed: 11,867 -> 8,081 mean (-3,786 per call, -31.9%)**
- **Aggregate verifier (n=9): 0.895 (v6) vs 0.782 (v5, n=8 valid; n=9 strict 0.695)**
- **Paired delta verifier: +0.110** on the 8 common cells (v6 n=8 = 0.892 vs v5 n=8 = 0.782).
  The naive aggregate diff 0.895 − 0.782 = +0.113 compares different cell counts.
- **All 9 cells grammar-valid in v6**; v5 had 33632 fail grammar (gv=False, stop=limit)
- **Verdict: PROGRESS** (aggregate 0.895 >= 0.84 threshold)

## Intervention

v6 collapses three v5 worked examples (`_anon:` DISCIPLINE, `_anon:` ID FORM ALIGNMENT,
PUNCTUATION-FREE IDENTIFIERS) plus the standalone PascalCase declare-first example into
**one decree #10 (ANON-REFERENCE DISCIPLINE)** with sub-rules a/b/c, and **one composite
WRONG/RIGHT worked example** that exercises all three patterns simultaneously.

All other v5 instructions preserved verbatim (GRAMMAR section, decrees 1-9, 11+).

## Per-cell results (v6)

| Cell  | gv | verifier | num | org | undecl | prompt_eval | eval | wall (s) | stop  |
| ----- | -- | -------- | --- | --- | ------ | ----------- | ---- | -------- | ----- |
| 27560 | T  | 0.667    | -   | -   | 0      | 7,209       | 793  | 139.5    | eos   |
| 27702 | T  | 1.000    | -   | -   | 0      | 7,240       | 162  | 74.9     | eos   |
| 29807 | T  | 0.812    | -   | -   | 7      | 7,262       | 1229 | 179.5    | eos   |
| 31149 | T  | 0.867    | -   | -   | 0      | 7,143       | 679  | 113.4    | eos   |
| 31430 | T  | 0.905    | -   | -   | 0      | 7,517       | 822  | 144.2    | eos   |
| 33632 | T  | 0.922    | -   | -   | 0      | 8,763       | 4686 | 544.7    | eos   |
| 34537 | T  | 1.000    | -   | -   | 0      | 7,049       | 599  | 119.7    | eos   |
| 35352 | T  | 0.991    | -   | -   | 0      | 9,432       | 6144 | 767.8    | limit |
| 36065 | T  | 0.892    | -   | -   | 3      | 11,111      | 2568 | 423.1    | eos   |

Note: `num`/`org` per-kind recall fields are `null` for all cells (scorer doesn't populate them
on v2.1 spec mode; verifier_pass_rate is the composite figure of merit).

## Side-by-side delta vs PR #397 v5 (paired)

| Cell  | v5 verifier | v6 verifier | delta      | v5 gv | v6 gv | Notes                                              |
| ----- | ----------- | ----------- | ---------- | ----- | ----- | -------------------------------------------------- |
| 27560 | 1.000       | 0.667       | -0.333     | T     | T     | regression (clean baseline, v6 less verbatim)      |
| 27702 | 0.000       | 1.000       | +1.000     | T     | T     | v6 correctly extracts Fusion Media boilerplate     |
| 29807 | 0.957       | 0.812       | -0.144     | T     | T     | v6 has 7 undecl (regression vs v5's 2)             |
| 31149 | 0.905       | 0.867       | -0.038     | T     | T     | marginal regression                                |
| 31430 | 0.917       | 0.905       | -0.012     | T     | T     | stable                                             |
| 33632 | - (failed)  | 0.922       | n/a (gain) | **F** | **T** | v5 hit grammar limit; v6 closes with eos           |
| 34537 | 1.000       | 1.000       | 0.000      | T     | T     | identical                                          |
| 35352 | 0.500       | 0.991       | +0.491     | T     | T     | v6 dramatic gain (still stop=limit; eval=6144)     |
| 36065 | 0.976       | 0.892       | -0.083     | T     | T     | concept-latch avoided but undecl=3 (regression)    |

**Aggregate:** v6 mean (n=9) = **0.895**, v5 mean (n=8 valid) = **0.782**.
**Paired delta on the 8 common cells (excludes 33632, gv=False in v5): v6 = 0.892, v5 = 0.782, delta = +0.110.**
(The naive aggregate diff 0.895 − 0.782 = +0.113 compares different cell counts and is not paired.)

## Token budget freed

| Metric                          | v5         | v6         | delta            |
| ------------------------------- | ---------- | ---------- | ---------------- |
| Prompt size (lines)             | 1,033      | 634        | -399 (-38.6%)    |
| Mean prompt_eval (tokens)       | 11,867     | 8,081      | -3,786 (-31.9%)  |
| Cell 27560 prompt_eval (ref)    | 10,995     | 7,209      | -3,786           |

Per-cell delta is exactly 3,786 across all 9 cells (deterministic prompt-size diff;
input docs identical). At 16K `num_ctx`, this leaves ~8K headroom for generation
that previously held prompt - directly relevant for 33632 (dup-decl) and 35352
(verbose extraction) which both exceed 4K eval tokens.

## Mechanism preservation (critical)

### 33632 dup-decl (DECLARE-ONCE)

- **CLOSED.** v6 verifier=0.922, gv=True, 35 unique ENT lines, 0 duplicates, undecl=0,
  stop=eos at eval=4686.
- v5 (PR #397) regressed here: gv=False, stop=limit (hit 16K context cap), 0 entities
  parsed. v6 fits the same document in 8,763 prompt + 4,686 generation = ~13.5K, well
  under cap.
- **DECLARE-ONCE rule from decree #10(a) preserved despite consolidation.**

### 36065 kind-latch (was 70% concept failure mode)

- **PRESERVED but with sampling regression.** v6 verifier=0.892, kind distribution:
  `sector x7, equity x5, country x5, org x4, concept x4, person x1`. v5 had
  `macro_indicator x7, equity x5, org x4, concept x2, person x1, country x1`.
- The original v4 failure (`concept x90 / 128 entities, 70%`) does not recur; v6
  has 4 `concept` out of 26 = 15%, in normal range.
- However, v6 introduced 3 undeclared ENTs (v5 had 0) and verifier dropped 0.976 -> 0.892.
  The kind-latch fix from v5 decree priorities survived consolidation; entity-naming
  consistency slightly regressed.

### 27702 punctuation-ident (clean `_anon:` form)

- **CLOSED, with bonus.** v6 verifier=1.000 (vs v5 0.000), no `_anon:` refs needed
  (the doc is Fusion Media boilerplate; v6 correctly extracted that and skipped the
  fake Form-144 URL). v5 hallucinated `MonolithicPowerSystems` from the URL slug;
  v6 produced 2 `org` ENTs that match the actual article text verbatim.

## Wall-time

- Total sweep: 04:17Z -> 05:31Z = **74 minutes** (8 polls x ~9 min each)
- Mean per cell: 278.5s
- Longest: 35352 (767.8s, stop=limit, eval=6144), 33632 (544.7s, eval=4686)
- Shortest: 27702 (74.9s)

## Verdict: PROGRESS

- **Aggregate verifier 0.895 >= 0.84 threshold -> PROGRESS.**
- 6/9 cells >= 0.892, 9/9 cells gv=True (v5 had 7/8 gv=True among reported, plus 33632 failed grammar).
- Token budget freed (-31.9% prompt) lets 33632 fit at 16K `num_ctx`; v5 needed 32K for the same cell.
- Side-by-side wins: 27702 (+1.000), 33632 (gv recovery), 35352 (+0.491).
- Side-by-side losses: 27560 (-0.333), 29807 (-0.144), 36065 (-0.083, with undecl=3).

**Net interpretation:** consolidation of 3 worked examples into one decree #10 with
sub-rules a/b/c did not destroy the mechanism fixes. Aggregate improves materially.
Residual losses on clean-baseline cells (27560, 36065) suggest the composite worked
example is slightly less specific than three separate ones for entity-naming consistency;
this is the prompt-iteration ceiling on a 4B-active MoE for prose-heavy structured
extraction. Further gains likely require data-driven (training) interventions, not
prompt edits.
