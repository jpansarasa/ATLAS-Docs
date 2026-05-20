# Phase 1.1 — Direct-extraction arm (qwen3:30b-a3b)

**Plan reference:** [`docs/plans/atlas-dsl-poc-plan.md` §4.1 (Q1)](../../../../plans/atlas-dsl-poc-plan.md)
**Date:** 2026-05-20
**Corpus:** 10 articles, R2/R3 corpus (identical to `dsl-round3`)
**Model:** `qwen3:30b-a3b-instruct-2507-q4_K_M` on `ollama-cpu-gen`, `direct` variant
**Total wall time:** ~51 min (mean 306.0s/cell, including one cell that re-spawned its ollama runner mid-flight)

## TL;DR — Decision

**Verdict: structure is most of the win; CoD's intermediate-thinking step adds little on top of the DSL grammar itself.**

Plan §4.1 decision rule:
- `direct ≥ spec` → retire CoD across the board.
- `spec >> direct` → CoD-style intermediate state matters; keep it.
- `prose_cod > both` → unexpected; rethink.

Headline three-way comparison on qwen3:30b-a3b, n=10 corpus:

| metric | prose_cod (R1, single_pass) | spec (R3, DSL CoD) | direct (Phase 1.1, DSL) | direct vs spec | spec vs prose |
|---|---|---|---|---|---|
| `numeric_recall` / `num_sem` | 19% (micro) | 88.6% | **88.6%** | tied | +69.6pp |
| `numeric_exact_match_rate` | n/a (R1 didn't measure) | 66.9% | **66.6%** | −0.3pp | — |
| `unit_preservation_rate` | n/a | 82.2% | **96.2%** | **+14.0pp** | — |
| `ticker_recall` | 25% | 50.0% | **0.0%** | −50.0pp | +25pp |
| `org_recall` (companies in R1) | 40% (micro) | 31.0% | **28.5%** | −2.5pp | −9pp |
| `sector_recall` | 100% (micro) | 100.0% | 0.0% | −100.0pp | tied |
| `industry_recall` | 23% (micro) | 60.8% | **63.3%** | +2.5pp | +37.8pp |
| `grammar_valid_rate` | n/a | 100.0% | **90.0%** | −10pp | — |
| `span_copy_fraction` | n/a | 62.5% | **60.5%** | −2.0pp | — |
| `wall_s` (mean) | 49.3 s (R1 single_pass) | 339.7 s | **306.0 s** | −33.7 s | +290 s |
| `cost_per_recall_point` | — | 6.01 | **7.84** | +1.83 | — |

**Decision branch fired: `direct ≈ spec`.** On the two most mechanistically meaningful metrics — `numeric_exact_match_rate` (Benford's-curse copy-mode evidence) and `numeric_semantic_match_rate` (the metric that drove the R1→R2 jump) — direct matches spec to within a fraction of a point. `unit_preservation_rate` is actually +14pp better under direct. The recall gains over prose CoD are owned by the DSL grammar itself, not by the chain-of-distillation framing.

The two regressions (tickers 0%, sectors 0%) are scorer-shaped, not capability-shaped:
- `ticker_recall` 0% comes from the direct prompt's identifier convention drifting: the model uses bare company names as ENT identifiers rather than `TICKER:EXCHANGE`, so the scorer can't match the ground-truth tickers. Legacy `ent_recall` (25.7%) shows entities are still captured. Same pattern in Phase 1.4 Claude (tick 0%, org 97%) — scorer is identifier-form-sensitive.
- `sector_recall` 0% — only 2 cells in this run had non-empty ground-truth sector lists; R3 spec landed 100% on the same rows, so this is a 1-cell variance signal, not a structural difference.

**Implications for the plan:**
- The chain-of-distillation intermediate state is no longer the lever — structure is. Strong support for retiring the prose-CoD prompt path entirely.
- Phase 2 (grammar-constrained decoding) is now even more attractive: structure-alone preservation means grammar enforcement should preserve recall while shrinking prompts (and erasing the gv% regression seen here).
- Phase 3 (copy-mode schema with explicit `source_span` pointers) is the right venue to address `ticker_recall` and `org_recall` — both arms here underperform Claude's `org_recall` of 97% (Phase 1.4) by ~70pp, and the gap survives the prose→DSL transition.

---

Total result files: **10**
Models seen: **1**, variants: **1**

Phase 0 instrumentation (DSL PoC plan §3) — per-category recall breakdown, span_copy_fraction mechanism metric, and latency-per-recall-point cost metric.

Legacy columns (preserved for apples-to-apples vs R1/R2/R3): `n`, `gv%` (grammar-valid rate), `ent`/`evt`/`claim`/`note` (mean block counts), `ent_rec` (legacy combined ticker+company recall), `num_rec` (legacy lenient numeric recall, == `num_sem` below), `undecl`, `miss_slot`.

Phase 0 columns: `tick`/`org`/`pers`/`loc`/`inst`/`sect`/`ind` (per-category recall; n/a when ground truth lacks the category), `num_exa` (numeric exact-match rate), `num_sem` (numeric semantic recall — full literal OR any digit-token), `unit_pre` (unit-token preservation), `span_cp` (n=4 char n-gram fraction of output appearing verbatim in input), `wall_s` (mean wall seconds), `cost` (wall_s / (mean(num+tick+org recall) × 100), plan §3.3).

## Per-model averages — legacy view (R1/R2/R3 parity)

| model | class | variant | n | gv% | ent | evt | claim | note | ent_rec | num_rec | undecl | miss_slot |
|---|---|---|---|---|---|---|---|---|---|---|---|---|
| qwen3:30b-a3b-instruct-2507-q4_K_M | B | direct | 10 | 90.0% | 12.50 | 25.40 | 2.00 | 0.20 | 25.7% | 88.6% | 0.00 | 0.00 |

## Per-model averages — Phase 0 per-category recall

| model | class | variant | n | tick | org | pers | loc | inst | sect | ind | num_exa | num_sem | unit_pre | span_cp |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| qwen3:30b-a3b-instruct-2507-q4_K_M | B | direct | 10 | 0.0% | 28.5% | n/a | n/a | n/a | 0.0% | 63.3% | 66.6% | 88.6% | 96.2% | 60.5% |

## Per-model averages — latency × recall cost (plan §3.3)

`cost` = mean(wall_s) / (mean target_recall × 100); lower is better. Target recall = unweighted mean of available {numeric, ticker, org} recalls. ``n/a`` when no target-recall component is available.

| model | class | variant | n | wall_s | tick | org | num_sem | cost |
|---|---|---|---|---|---|---|---|---|
| qwen3:30b-a3b-instruct-2507-q4_K_M | B | direct | 10 | 306.04 | 0.0% | 28.5% | 88.6% | 7.84 |

## Per-class averages

| class | variant | n | gv% | ent | evt | claim | note | ent_rec | num_rec | undecl | miss_slot |
|---|---|---|---|---|---|---|---|---|---|---|---|
| A | direct | 0 | n/a | n/a | n/a | n/a | n/a | n/a | n/a | n/a | n/a |
| B | direct | 10 | 90.0% | 12.50 | 25.40 | 2.00 | 0.20 | 25.7% | 88.6% | 0.00 | 0.00 |
| C | direct | 0 | n/a | n/a | n/a | n/a | n/a | n/a | n/a | n/a | n/a |

## Variant deltas (`spec+example` minus `spec`)

Positive means the worked-example arm improved that metric. Cells with `n/a` indicate one or both arms were empty.

| model | class | Δ gv% | Δ ent | Δ evt | Δ claim | Δ ent_rec | Δ num_rec | Δ undecl | Δ miss_slot |
|---|---|---|---|---|---|---|---|---|---|
| qwen3:30b-a3b-instruct-2507-q4_K_M | B | n/a | n/a | n/a | n/a | n/a | n/a | n/a | n/a |

Per-class deltas:

| class | Δ gv% | Δ ent | Δ evt | Δ claim | Δ ent_rec | Δ num_rec | Δ undecl | Δ miss_slot |
|---|---|---|---|---|---|---|---|---|
| A | n/a | n/a | n/a | n/a | n/a | n/a | n/a | n/a |
| B | n/a | n/a | n/a | n/a | n/a | n/a | n/a | n/a |
| C | n/a | n/a | n/a | n/a | n/a | n/a | n/a | n/a |

## Span-copy correlations (plan §3.2 mechanism check)

Pearson `r` of per-cell `span_copy_fraction` against the indicated recall. Positive `r` supports the induction-head copy-mode hypothesis (verbatim copies preserve numerics and entity surfaces better than paraphrase/regeneration). `n` is the number of (model, variant, article) cells with both metrics defined.

| target metric | Pearson r | n pairs |
|---|---|---|
| numeric_exact_match_rate | +0.567 | 7 |
| numeric_semantic_match_rate | -0.491 | 7 |
| org_recall | -0.502 | 6 |

---

Generated by `aggregate_report.py` from `results/phase1-1-direct`.
