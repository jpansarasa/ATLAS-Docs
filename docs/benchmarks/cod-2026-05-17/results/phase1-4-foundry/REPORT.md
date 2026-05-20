# Phase 1.4 — Frontier closed-weight reference (Claude 4.7 via Azure Foundry)

**Plan reference:** [`docs/plans/atlas-dsl-poc-plan.md` §4.4 (Q4)](../../../../plans/atlas-dsl-poc-plan.md)
**Date:** 2026-05-20
**Corpus:** 10 articles, R2/R3 corpus (identical to `dsl-round3`)
**Model:** `claude-opus-4-7` via Azure Foundry, Class C spec template
**Total Foundry spend:** **$1.41** (run-label total in `azure-oracle-ledger.jsonl`; well under the $5 cap)
**Mean wall time per cell:** 43.1s

## TL;DR — Decision

**Verdict: qwen3:30b is at the capability frontier on this task. Focus on Phase 2-4 rather than chasing a bigger model.**

Plan §4.4 decision rule: *"if Claude crushes qwen3:30b by 30+pp on numeric_exact_match_rate, the gain is real and worth chasing; if Claude is only marginally better, qwen3:30b is at the capability frontier."*

| metric | qwen3:30b spec (R3) | claude-opus-4-7 spec | Δ (Claude − qwen) | 30pp gap? |
|---|---|---|---|---|
| `numeric_exact_match_rate` | 66.9% | 70.8% | **+3.9pp** | **no** — gap well below 30pp threshold |
| `numeric_semantic_match_rate` | 88.6% | 94.6% | +6.0pp | no |
| `unit_preservation_rate` | 82.2% | 100.0% | +17.8pp | no |
| `org_recall` | 31.0% | 97.2% | +66.2pp | yes — but org_recall, not numeric |
| `ticker_recall` | 50.0% | **0.0%** | −50.0pp | Claude REGRESSES |
| `sector_recall` | 100.0% | 100.0% | 0.0pp | tied |
| `industry_recall` | 60.8% | 62.5% | +1.7pp | no |
| `grammar_valid_rate` | 100.0% | 70.0% | **−30.0pp** | Claude REGRESSES |
| `span_copy_fraction` | 62.5% | 50.2% | −12.3pp | Claude copies less |
| `cost_per_recall_point` | 6.01 | **0.67** | 9× cheaper on wall | (latency, not capability) |

The plan's decision rule keys on `numeric_exact_match_rate` (the metric tied to mechanism — induction-head copy mode vs FFN regeneration). On that axis the gap is **+3.9pp**, well under the 30pp threshold. **Q4 answered: qwen3:30b's gain is at the capability frontier; not a 30B-MoE artifact.**

## Headline asymmetries (interesting beyond the headline metric)

- **`org_recall` 31% → 97.2%.** Claude pulls 66pp more organization names. This is the largest single gap and it is qualitative: qwen3:30b is under-extracting orgs. Phase 3 (copy-mode schema with verbatim `name` slot + `source_span` pointer) is the right place to address this — make org names a verbatim copy-slot rather than letting the model regenerate them.
- **`ticker_recall` 50% → 0%.** Claude does NOT emit tickers in DSL identifier form despite the Class C spec template defining `TICKER:EXCHANGE` identifiers (`TH:NYSE`, etc). The scorer matches surfaces literally; Claude prefers `verbatim_name` (company names) over identifier-style tickers. The legacy `ent_recall` (82.4%) shows the entities are captured — they are just encoded under company names, not tickers. Mechanism-relevant for prompt design, not a capability gap.
- **`grammar_valid_rate` 100% → 70%.** Claude produces invalid DSL on 3/10 cells (`31149`, `31430`, `36065`). Root cause: Claude used the `_anon:<short_desc>` shorthand documented in the Class C spec template (`cod_dsl_classC.txt` line 35: "Local-only refs (not in any catalog): `_anon:<short_desc>`") in ENT identifier slots, e.g. `ENT Toronto / _anon:city`, `ENT _anon:sme_segment / sector`, `ENT BoC = "Bank of Canada" / org (country=_anon:Canada)`. The parser rejects this form: `Grammar error: Unexpected token Token('QUAL_VAL', ':<word>')`. qwen3:30b R3 didn't trigger this because it didn't emit `_anon:` shorthand at all. This is parser-spec drift (parser stricter than the documented spec) — same class as the leniency batch in PR #376 (header/nested-list/narrative). Not in scope for Phase 1.4; follow-up parser-leniency PR recommended. It is also still a Phase-2 argument: grammar-constrained decoding would have prevented Claude from emitting a form the parser rejects, regardless of whether the spec or the parser is wrong.
- **`cost_per_recall_point` 6.01 → 0.67.** Claude runs ~8× faster per cell (43.1s vs 339.7s for qwen3:30b on the same corpus). Production-wall economics differ from capability economics; this gap belongs in the "production latency budget" open decision (plan §10 #1), not in the Q4 verdict.

## Span-copy mechanism check (plan §3.2)

| target metric | Pearson r (Claude) | Pearson r (qwen3:30b R3, for reference) |
|---|---|---|
| `numeric_exact_match_rate` | +0.901 (n=4) | +0.630 (n=66) |
| `numeric_semantic_match_rate` | +0.728 (n=4) | +0.363 (n=66) |

Claude's per-cell correlations point the same direction as qwen's: higher span-copy fraction tracks higher numeric-exact recall. Sample is small (n=4 cells with both metrics defined) so the magnitudes are not directly comparable, but the sign agreement is consistent with the induction-head copy-mode hypothesis. **No counter-evidence from the frontier model.**

## Cost breakdown

Per-cell from `foundry_meta.cost_est_usd` (Foundry list price: $5/Mtok input,
$25/Mtok output for `claude-opus-4-7`):

| article | input tok | output tok | cost ($) | wall (s) |
|---|---|---|---|---|
| 27560 | 2,518 | 2,054 | 0.0639 | 24.5 |
| 27702 | 2,516 | 136 | 0.0160 | 27.4 |
| 27772 | 17,600 | 5,814 | 0.2334 | 67.8 |
| 29807 | 2,557 | 1,856 | 0.0592 | 50.9 |
| 31149 | 2,382 | 1,433 | 0.0477 | 15.4 |
| 31430 | 2,859 | 1,330 | 0.0475 | 20.2 |
| 33632 | 5,115 | 8,000 | 0.2256 | 78.7 |
| 34537 | 2,336 | 1,476 | 0.0486 | 13.9 |
| 35352 | 4,132 | 8,000 | 0.2207 | 79.2 |
| 36065 | 8,638 | 4,778 | 0.1626 | 53.0 |
| **10-cell total** | **52,653** | **34,877** | **$1.1252** | **mean 43.1s** |

Run-label total in the Foundry ledger (`/opt/ai-inference/training-data/azure-oracle-ledger.jsonl`,
filter `run_label="phase1-4-frontier-claude-opus-4-7"`): **$1.41**. The difference
($0.28) comes from 2 cells (29807, 33632) that had two billed API calls each
during script bring-up — the result files on disk are from the second call;
the first call's ledger row reflects an interim run that was overwritten.

Budget cap: **$5.00** — used 28% of cap.

---

## Aggregator output (auto-generated)

Total result files: **10**
Models seen: **1**, variants: **1**

Phase 0 instrumentation (DSL PoC plan §3) — per-category recall breakdown, span_copy_fraction mechanism metric, and latency-per-recall-point cost metric.

Legacy columns (preserved for apples-to-apples vs R1/R2/R3): `n`, `gv%` (grammar-valid rate), `ent`/`evt`/`claim`/`note` (mean block counts), `ent_rec` (legacy combined ticker+company recall), `num_rec` (legacy lenient numeric recall, == `num_sem` below), `undecl`, `miss_slot`.

Phase 0 columns: `tick`/`org`/`pers`/`loc`/`inst`/`sect`/`ind` (per-category recall; n/a when ground truth lacks the category), `num_exa` (numeric exact-match rate), `num_sem` (numeric semantic recall — full literal OR any digit-token), `unit_pre` (unit-token preservation), `span_cp` (n=4 char n-gram fraction of output appearing verbatim in input), `wall_s` (mean wall seconds), `cost` (wall_s / (mean(num+tick+org recall) × 100), plan §3.3).

## Per-model averages — legacy view (R1/R2/R3 parity)

| model | class | variant | n | gv% | ent | evt | claim | note | ent_rec | num_rec | undecl | miss_slot |
|---|---|---|---|---|---|---|---|---|---|---|---|---|
| claude-opus-4-7 | ? | spec | 10 | 70.0% | 11.20 | 9.30 | 13.30 | 0.90 | 82.4% | 94.6% | 0.00 | 0.00 |

## Per-model averages — Phase 0 per-category recall

| model | class | variant | n | tick | org | pers | loc | inst | sect | ind | num_exa | num_sem | unit_pre | span_cp |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| claude-opus-4-7 | ? | spec | 10 | 0.0% | 97.2% | n/a | n/a | n/a | 100.0% | 62.5% | 70.8% | 94.6% | 100.0% | 50.2% |

## Per-model averages — latency × recall cost (plan §3.3)

`cost` = mean(wall_s) / (mean target_recall × 100); lower is better. Target recall = unweighted mean of available {numeric, ticker, org} recalls. ``n/a`` when no target-recall component is available.

| model | class | variant | n | wall_s | tick | org | num_sem | cost |
|---|---|---|---|---|---|---|---|---|
| claude-opus-4-7 | ? | spec | 10 | 43.10 | 0.0% | 97.2% | 94.6% | 0.67 |

## Per-class averages

| class | variant | n | gv% | ent | evt | claim | note | ent_rec | num_rec | undecl | miss_slot |
|---|---|---|---|---|---|---|---|---|---|---|---|
| A | spec | 0 | n/a | n/a | n/a | n/a | n/a | n/a | n/a | n/a | n/a |
| B | spec | 0 | n/a | n/a | n/a | n/a | n/a | n/a | n/a | n/a | n/a |
| C | spec | 10 | 70.0% | 11.20 | 9.30 | 13.30 | 0.90 | 82.4% | 94.6% | 0.00 | 0.00 |

## Variant deltas (`spec+example` minus `spec`)

Positive means the worked-example arm improved that metric. Cells with `n/a` indicate one or both arms were empty.

| model | class | Δ gv% | Δ ent | Δ evt | Δ claim | Δ ent_rec | Δ num_rec | Δ undecl | Δ miss_slot |
|---|---|---|---|---|---|---|---|---|---|
| claude-opus-4-7 | ? | n/a | n/a | n/a | n/a | n/a | n/a | n/a | n/a |

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
| numeric_exact_match_rate | +0.901 | 4 |
| numeric_semantic_match_rate | +0.728 | 4 |
| org_recall | -0.225 | 3 |

---

Generated by `aggregate_report.py` from `results/phase1-4-foundry`.
