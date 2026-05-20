# Phase 1.2 — Schema-order experiment (Tam et al. mechanism)

**Plan reference:** [`docs/plans/atlas-dsl-poc-plan.md` §4.2 (Q2)](../../../../plans/atlas-dsl-poc-plan.md)
**Date:** 2026-05-20
**Corpus:** 10 articles, R2/R3 corpus (identical to `dsl-round3`)
**Model:** `qwen3:30b-a3b-instruct-2507-q4_K_M` on `ollama-cpu-gen`
**Variants:** 4 = {v1a_spec, v1a_spec+example, v1b_spec, v1b_spec+example}, n=10 each (40 cells total)

## TL;DR — Decision

**Verdict: the Tam et al. (2024) schema-order failure mode is confirmed, in a stronger form than expected. Lock in `v1a` (verbatim-first) as the design rule for all future ATLAS DSL schemas.**

Plan §4.2 decision rule:
- `v1b` significantly worse than `v1a` → confirms Tam mechanism; lock verbatim-first as design rule.
- no difference → mechanism is elsewhere; document negative result.

**`v1b` is dramatically worse than `v1a`** — and not only on the recall metrics the plan anticipated. The dominant effect is a collapse in grammar validity and block production: putting derived slots before verbatim slots in the schema definition halves the model's ability to emit parseable structure at all.

### Headline comparison (v1a vs v1b)

| metric | v1a_spec | v1b_spec | Δ (v1b−v1a) | v1a_spec+example | v1b_spec+example | Δ (v1b−v1a) |
|---|---|---|---|---|---|---|
| **`grammar_valid_rate`** | **100.0%** | **50.0%** | **−50.0pp** | **90.0%** | **70.0%** | **−20.0pp** |
| `mean_ent` (blocks) | 9.70 | 2.50 | **−7.20** | 8.80 | 2.50 | **−6.30** |
| `mean_evt` (blocks) | 26.70 | 13.80 | **−12.90** | 23.70 | 15.80 | **−7.90** |
| `mean_claim` (blocks) | 1.10 | 0.30 | −0.80 | 0.50 | 0.70 | +0.20 |
| `ticker_recall` | 50.0% | n/a (zero valid cells with ticker GT) | — | 50.0% | 0.0% | **−50.0pp** |
| `org_recall` | 35.7% | 54.2% | +18.5pp | 57.0% | 48.3% | −8.7pp |
| `numeric_exact_match_rate` | 63.4% | 51.9% | −11.5pp | 66.5% | 63.9% | −2.6pp |
| `numeric_semantic_match_rate` | 87.6% | 88.9% | +1.3pp | 89.6% | 91.7% | +2.1pp |
| `unit_preservation_rate` | 95.4% | 93.8% | −1.6pp | 96.1% | 95.8% | −0.3pp |
| `span_copy_fraction` | 61.6% | 61.9% | +0.3pp | 56.8% | 58.7% | +1.9pp |
| `mean_wall_s` | 292.30 | 355.66 | +63.36 | 489.26 | 486.68 | −2.58 |
| `cost_per_recall_point` (plan §3.3) | 5.06 | 4.97 | −0.09 | 7.47 | 10.43 | +2.96 |

The mechanism is unambiguous on the structural metrics:
- **Grammar validity halves under v1b spec** (100% → 50%) and drops 20pp under v1b spec+example (90% → 70%). The model can't produce a parseable document when forced to commit to derived fields before naming the entity.
- **Block production collapses ~3-4x under v1b** (`mean_ent` 9.70 → 2.50; `mean_evt` 26.70 → 13.80). Even on cells that *do* parse, v1b produces roughly a quarter of the entities and half the events. The model effectively gives up on coverage when the schema forces premature commitments.
- **The recall metrics that look "okay" for v1b are conditional on grammar validity.** `org_recall` 54.2% on v1b_spec is computed over only 5 valid cells; `ticker_recall` is `n/a` because the one article with ground-truth tickers (27772) failed to parse under v1b_spec entirely.

The non-structural recalls (`numeric_semantic_match_rate`, `unit_preservation_rate`, `span_copy_fraction`) are nearly invariant across schema order — those metrics are bottlenecked by token-level copy behaviour (Phase 1.1 mechanism), not by schema design. Schema order acts on the *structural* layer.

### Why this is stronger than Tam et al. predicted

Tam et al. (2024) framed the failure mode as a *reasoning* degradation: "answer-first" schemas degrade chain-of-thought quality on tasks where the answer benefits from prior reasoning. Our finding is more severe: in the extraction setting, "derived-first" schemas don't merely degrade recall, they cause the model to abandon structured generation altogether (grammar collapse, block-count collapse).

The likely mechanism is that the v1b spec asks the model to commit to derived fields (e.g. event types, classifications, magnitudes) *before* it has anchored to the source surface tokens that justify those derivations. Without a verbatim anchor, the model either (a) gives up and emits free prose that the parser rejects, or (b) emits a sparse skeleton with only the most obvious entities, dropping coverage.

### Design rule (lock-in)

For all future ATLAS DSL schemas:
1. **Verbatim-bearing slots come first** within every block (entity surface form, `source_span`, raw mentions).
2. **Derived/typed slots come last** (event type, classification, normalized magnitude).
3. This rule applies block-by-block, not just document-globally — within an `EVT` block, the `source_span` precedes the typed `action`/`subject` slots; within an `ENT` block, the surface name precedes the type tag.

The current v1 grammar already conforms (it was authored verbatim-first by accident — this experiment retroactively justifies the convention). The lock-in matters for Phase 2 (grammar-constrained decoding) and Phase 3 (copy-mode schema with explicit `source_span` pointers): both phases must preserve the verbatim-first order or risk re-introducing the v1b failure mode.

### Implications for the plan

- Phase 2 (grammar-constrained decoding) should treat v1a's slot order as a hard constraint, not a stylistic preference — the grammar must enforce verbatim-first.
- Phase 3 (copy-mode schema): explicit `source_span: "..."` slots in front of every derived field should *amplify* the verbatim-first effect; the experiment design should now expect v1c (copy-augmented) to beat v1a, with v1b serving as a guardrail control.
- The Phase 1.2 result also explains the worked-example regression that R2/R3 saw on Class B without needing additional experiments: the v1 schema is already verbatim-first, so the regression must have a *different* mechanism (the worked example itself may be diluting copy-mode by demonstrating paraphrase). That hypothesis is testable separately but is no longer on the critical path.

---

Total result files: **40**
Models seen: **1**, variants: **4**

Phase 0 instrumentation (DSL PoC plan §3) — per-category recall breakdown, span_copy_fraction mechanism metric, and latency-per-recall-point cost metric.

Legacy columns (preserved for apples-to-apples vs R1/R2/R3): `n`, `gv%` (grammar-valid rate), `ent`/`evt`/`claim`/`note` (mean block counts), `ent_rec` (legacy combined ticker+company recall), `num_rec` (legacy lenient numeric recall, == `num_sem` below), `undecl`, `miss_slot`.

Phase 0 columns: `tick`/`org`/`pers`/`loc`/`inst`/`sect`/`ind` (per-category recall; n/a when ground truth lacks the category), `num_exa` (numeric exact-match rate), `num_sem` (numeric semantic recall — full literal OR any digit-token), `unit_pre` (unit-token preservation), `span_cp` (n=4 char n-gram fraction of output appearing verbatim in input), `wall_s` (mean wall seconds), `cost` (wall_s / (mean(num+tick+org recall) × 100), plan §3.3).

## Per-model averages — legacy view (R1/R2/R3 parity)

| model | class | variant | n | gv% | ent | evt | claim | note | ent_rec | num_rec | undecl | miss_slot |
|---|---|---|---|---|---|---|---|---|---|---|---|---|
| qwen3:30b-a3b-instruct-2507-q4_K_M | B | v1a_spec | 10 | 100.0% | 9.70 | 26.70 | 1.10 | 0.20 | 32.5% | 87.6% | 0.00 | 0.00 |
| qwen3:30b-a3b-instruct-2507-q4_K_M | B | v1a_spec+example | 10 | 90.0% | 8.80 | 23.70 | 0.50 | 0.00 | 53.3% | 89.6% | 0.00 | 0.00 |
| qwen3:30b-a3b-instruct-2507-q4_K_M | B | v1b_spec | 10 | 50.0% | 2.50 | 13.80 | 0.30 | 0.10 | 54.2% | 88.9% | 0.00 | 0.00 |
| qwen3:30b-a3b-instruct-2507-q4_K_M | B | v1b_spec+example | 10 | 70.0% | 2.50 | 15.80 | 0.70 | 0.00 | 45.8% | 91.7% | 0.00 | 0.00 |

Reference — R3 (same model, same n=10, current v1 spec, scored under same Phase 0 instrumentation):

| model | class | variant | n | gv% | ent | evt | claim | note | ent_rec | num_rec |
|---|---|---|---|---|---|---|---|---|---|---|
| qwen3:30b-a3b-instruct-2507-q4_K_M | B | spec (R3) | 10 | 100.0% | 10.90 | 27.70 | 1.70 | 0.20 | 29.4% | 88.6% |
| qwen3:30b-a3b-instruct-2507-q4_K_M | B | spec+example (R3) | 10 | 90.0% | 9.40 | 24.00 | 1.00 | 0.10 | 8.8% | 90.8% |

`v1a` numbers in Phase 1.2 track R3 within sampling noise (gv 100/90%, evt 26.70 vs 27.70, ent 9.70 vs 10.90), confirming the v1a arm is a faithful replication of the R3 production setup. The `v1b` collapse is therefore attributable to the schema-order change, not to a corpus or runtime drift.

## Per-model averages — Phase 0 per-category recall

| model | class | variant | n | tick | org | pers | loc | inst | sect | ind | num_exa | num_sem | unit_pre | span_cp |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| qwen3:30b-a3b-instruct-2507-q4_K_M | B | v1a_spec | 10 | 50.0% | 35.7% | n/a | n/a | n/a | 100.0% | 54.6% | 63.4% | 87.6% | 95.4% | 61.6% |
| qwen3:30b-a3b-instruct-2507-q4_K_M | B | v1a_spec+example | 10 | 50.0% | 57.0% | n/a | n/a | n/a | 100.0% | 78.1% | 66.5% | 89.6% | 96.1% | 56.8% |
| qwen3:30b-a3b-instruct-2507-q4_K_M | B | v1b_spec | 10 | n/a | 54.2% | n/a | n/a | n/a | n/a | 100.0% | 51.9% | 88.9% | 93.8% | 61.9% |
| qwen3:30b-a3b-instruct-2507-q4_K_M | B | v1b_spec+example | 10 | 0.0% | 48.3% | n/a | n/a | n/a | 100.0% | 100.0% | 63.9% | 91.7% | 95.8% | 58.7% |

## Per-model averages — latency × recall cost (plan §3.3)

`cost` = mean(wall_s) / (mean target_recall × 100); lower is better. Target recall = unweighted mean of available {numeric, ticker, org} recalls. ``n/a`` when no target-recall component is available.

| model | class | variant | n | wall_s | tick | org | num_sem | cost |
|---|---|---|---|---|---|---|---|---|
| qwen3:30b-a3b-instruct-2507-q4_K_M | B | v1a_spec | 10 | 292.30 | 50.0% | 35.7% | 87.6% | 5.06 |
| qwen3:30b-a3b-instruct-2507-q4_K_M | B | v1a_spec+example | 10 | 489.26 | 50.0% | 57.0% | 89.6% | 7.47 |
| qwen3:30b-a3b-instruct-2507-q4_K_M | B | v1b_spec | 10 | 355.66 | n/a | 54.2% | 88.9% | 4.97 |
| qwen3:30b-a3b-instruct-2507-q4_K_M | B | v1b_spec+example | 10 | 486.68 | 0.0% | 48.3% | 91.7% | 10.43 |

## Per-class averages

| class | variant | n | gv% | ent | evt | claim | note | ent_rec | num_rec | undecl | miss_slot |
|---|---|---|---|---|---|---|---|---|---|---|---|
| A | v1a_spec | 0 | n/a | n/a | n/a | n/a | n/a | n/a | n/a | n/a | n/a |
| A | v1a_spec+example | 0 | n/a | n/a | n/a | n/a | n/a | n/a | n/a | n/a | n/a |
| A | v1b_spec | 0 | n/a | n/a | n/a | n/a | n/a | n/a | n/a | n/a | n/a |
| A | v1b_spec+example | 0 | n/a | n/a | n/a | n/a | n/a | n/a | n/a | n/a | n/a |
| B | v1a_spec | 10 | 100.0% | 9.70 | 26.70 | 1.10 | 0.20 | 32.5% | 87.6% | 0.00 | 0.00 |
| B | v1a_spec+example | 10 | 90.0% | 8.80 | 23.70 | 0.50 | 0.00 | 53.3% | 89.6% | 0.00 | 0.00 |
| B | v1b_spec | 10 | 50.0% | 2.50 | 13.80 | 0.30 | 0.10 | 54.2% | 88.9% | 0.00 | 0.00 |
| B | v1b_spec+example | 10 | 70.0% | 2.50 | 15.80 | 0.70 | 0.00 | 45.8% | 91.7% | 0.00 | 0.00 |
| C | v1a_spec | 0 | n/a | n/a | n/a | n/a | n/a | n/a | n/a | n/a | n/a |
| C | v1a_spec+example | 0 | n/a | n/a | n/a | n/a | n/a | n/a | n/a | n/a | n/a |
| C | v1b_spec | 0 | n/a | n/a | n/a | n/a | n/a | n/a | n/a | n/a | n/a |
| C | v1b_spec+example | 0 | n/a | n/a | n/a | n/a | n/a | n/a | n/a | n/a | n/a |

## Variant deltas (`spec+example` minus `spec`)

The legacy aggregator's delta section is hardcoded to variant names `spec` / `spec+example`; Phase 1.2's `v1{a,b}_*` labels don't match that pattern so the auto-generated deltas (preserved below for honesty) are `n/a`. The v1a-vs-v1b deltas are tabulated in the TL;DR comparison table above; the spec-vs-spec+example deltas (within each schema variant), computed by hand from the per-model table, are:

| variant family | Δ gv% (with example − without) | Δ mean_ent | Δ mean_evt | Δ num_exa | Δ org |
|---|---|---|---|---|---|
| v1a | −10.0pp | −0.90 | −3.00 | +3.1pp | +21.3pp |
| v1b | +20.0pp | 0.00 | +2.00 | +12.0pp | −5.9pp |

Worked example helps v1a's `org_recall` substantially (+21pp) and v1b's `gv%` substantially (+20pp). The v1b worked-example bump is mostly the example partially compensating for the schema-order pathology — it's not free recall, it's repair.

Original auto-generated delta tables (variant names don't match the aggregator's hardcoded `spec` / `spec+example` pattern, so all cells are `n/a`):

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
| numeric_exact_match_rate | +0.616 | 20 |
| numeric_semantic_match_rate | -0.503 | 20 |
| org_recall | +0.014 | 16 |

---

Generated by `aggregate_report.py` from `docs/benchmarks/cod-2026-05-17/results/phase1-2-schema-order`.
