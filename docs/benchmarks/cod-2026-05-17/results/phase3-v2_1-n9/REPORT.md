# Phase 3 v2.1 — n=9 scale-confirmation acceptance run

**Date:** 2026-05-21
**Model:** `qwen3:30b-a3b-instruct-2507-q4_K_M` (4 B active / 30 B total MoE, q4_K_M GGUF)
**Backend:** `llama-server` (`http://localhost:11437`) + GBNF grammar-constrained decoding
**Schema:** v2.1 no-offset (`docs/grammars/cod-dsl-v2.1.gbnf` + `prompts/cod_dsl_v2_no_offset.txt`)
**Runner:** `scripts/run_cod_dsl.py --schema v2_1 --backend llama-server --n-predict 4096 --timeout 500`
**Corpus:** Phase 2 Arm B (n=10 minus `27772` known timeout) → n=9
**Branch:** `feat/dsl-poc/phase3-v2_1-n9` off main (`0d13090b`)

---

## 1. Verdict (TL;DR)

**Pattern from the n=3 probe does NOT hold at scale.** All four §6.4 acceptance gates either FAIL or are UNDETERMINABLE on this n=9 subset:

| Gate | Target | n=9 actual | Verdict |
|---|---|---|---|
| `numeric_exact_match_rate` | ≥ 0.85 | **n/a** (no numeric GT in any of the 9 cells) | UNDETERMINABLE |
| `ticker_recall` | ≥ 0.40 | **n/a** (no ticker GT in any of the 9 cells) | UNDETERMINABLE |
| `org_recall` | ≥ 0.50 | **n/a** (no org GT in any of the 9 cells) | UNDETERMINABLE |
| `verifier_pass_rate` per article (mean across n=9) | ≥ 0.95 | **0.944** (3 defined cells averaged) | FAIL (marginal; only 3/9 cells contribute, denominator-collapsed) |
| Implicit gv-rate prerequisite (`grammar_valid` for the verifier to even run) | high | **33.3%** (3/9) | DEGRADES vs n=3 probe (67%, 2/3) |

**Overall: §6.4 FAIL.** Recall gates can't even be assessed because the Phase 2 Arm B corpus subset used here has no `numeric` / `ticker` / `org` ground truth attached (gt-file rows for these articles only list ENT names, no per-category labels). The `verbatim_match_rate` metric collapses to 0.611 (vs n=3 probe's 0.821) once the `27702`-style empty-block edge case (verbatim=0 with no copy slots emitted) is included.

The dominant failure mode (6 of 9 cells) is the **`name:` slot omission** root cause already documented in the §11.2 path (4) note — the lenient v2.1 preprocessor doesn't auto-emit `name:` for synthesized inferred ENTs, and the GBNF doesn't enforce it. Three of those 6 cells additionally hit `stop=limit` at `n_predict=4096`, but the parse failure is upstream of the truncation (line 5-7 of output, before truncation could matter for two of them).

## 2. Per-cell results

| article | gv | verbatim | verifier | num_exact | org_rec | ticker_rec | wall_s | stop | eval_tokens |
|---|---|---|---|---|---|---|---|---|---|
| 27560 | True | 0.833 | 0.833 | n/a | n/a | n/a | 31.8 | eos | 317 |
| 27702 | True | 0.000 | 1.000 | n/a | n/a | n/a | 29.3 | eos | 130 |
| 29807 | False | n/a | n/a | n/a | n/a | n/a | 102.2 | eos | 1339 |
| 31149 | False | n/a | n/a | n/a | n/a | n/a | 54.6 | eos | 615 |
| 31430 | False | n/a | n/a | n/a | n/a | n/a | 60.4 | eos | 718 |
| 33632 | False | n/a | n/a | n/a | n/a | n/a | 427.6 | **limit** | 4096 |
| 34537 | True | 1.000 | 1.000 | n/a | n/a | n/a | 48.8 | eos | 542 |
| 35352 | False | n/a | n/a | n/a | n/a | n/a | 339.6 | **limit** | 4096 |
| 36065 | False | n/a | n/a | n/a | n/a | n/a | 426.9 | **limit** | 4096 |

**Aggregates (all 9 cells):**
- `grammar_valid` rate: **3/9 = 33.3%**
- `verbatim_match_rate` (mean over 3 defined cells): **0.611**
- `verifier_pass_rate` (mean over 3 defined cells): **0.944**
- Wall: total **1,521.3 s** (~25 min), mean **169.0 s/cell**, median 60.4 s/cell

## 3. Comparison vs PR #392 n=3 probe (same model, same grammar, same prompt)

The n=3 probe (commit `0d13090b`, articles 27560 / 31149 / 31430) reported:

- `verbatim_match_rate` = **0.821** (mean of 2 gv-valid cells)
- `verifier_pass_rate` = **0.828** (mean of 2 gv-valid cells)
- gv-valid: **2/3 = 67%**
- Wall mean: **62.8 s**

The n=9 sweep on the **same 3 articles** produced different outcomes (nondeterminism within the grammar-constrained decode):

| article | n=3 probe (gv / verbatim / verifier) | n=9 sweep (gv / verbatim / verifier) | Δ |
|---|---|---|---|
| 27560 | True / 0.714 / 0.714 | True / 0.833 / 0.833 | **improved** |
| 31149 | False / n/a / n/a | False / n/a / n/a | unchanged (same `name:` failure mode) |
| 31430 | True / 0.929 / 0.941 | **False** / n/a / n/a | **regressed** |

So on the 3-cell intersection alone, gv rate slipped from 67% → 33% within nondeterminism band. Adding the 6 new cells (27702, 29807, 33632, 34537, 35352, 36065), only 2 are gv-valid (27702, 34537), and 27702's verbatim collapses to 0.000 (single NOTE block, no copy slots — `verbatim_match_rate` formula returns 0/total with total=0 → 0.0 rather than `None`; `verifier_pass_rate` returns 1.0 vacuously).

**Net:** the strong n=3 verbatim signal (0.821) was carried almost entirely by article 31430's single 0.929 outcome — a cell that regresses to gv=False in this run. The pattern is **not robust to corpus expansion or to rerun nondeterminism**.

## 4. Failure-mode breakdown (6 of 9 gv=False cells)

| article | parse_error line | dominant cause |
|---|---|---|
| 29807 | line 6 — `Token('ENT', 'ENT') ... Expected SLOT_LINE_NAME` | inline ENT decl missing `name:` slot |
| 31149 | line 6 — same | same |
| 31430 | line 6 — same | same |
| 33632 | line 7 — same | same; also hit `stop=limit` at 4096 tokens |
| 35352 | line 462 — `Token('$END', '') ... Expected SLOT_LINE_RAW` | truncated mid-slot at `stop=limit` |
| 36065 | line 6 — same `name:` issue + `stop=limit` | both stacked |

5 of 6 failures are the **same root cause** — the lenient-preprocessor `name:`-synthesis gap flagged for fix in §11.2 path (4). 1 of 6 (35352) is pure mid-slot truncation at `n_predict=4096`.

## 5. §6.4 acceptance verdict

| Gate | Target | Actual | Verdict |
|---|---|---|---|
| `numeric_exact_match_rate ≥ 0.85` | 0.85 | n/a (no GT) | UNDETERMINABLE |
| `ticker_recall ≥ 0.40` | 0.40 | n/a (no GT) | UNDETERMINABLE |
| `org_recall ≥ 0.50` | 0.50 | n/a (no GT) | UNDETERMINABLE |
| `verifier_pass_rate ≥ 0.95` (per-article mean) | 0.95 | 0.944 (n=3 defined) | FAIL (marginal; tiny sample) |
| gv-rate prerequisite | (high) | 33.3% | FAIL — regression vs n=3 |

**Overall verdict: §6.4 FAIL.**

The most defensible characterization: **scale-confirmation did not confirm the n=3 pattern.** The 0.821 verbatim signal does not survive corpus expansion. The first-order blocker is gv-rate, dominated by the `name:`-slot preprocessor gap (5/6 failure cells). The §6.4 recall gates can't be evaluated against this corpus subset at all — Phase 2 Arm B's GT rows lack per-category numeric / ticker / org labels for any of the 9 articles run.

## 6. Recommended next actions (NOT decided here)

1. **Land the preprocessor `name:`-synthesis fix** flagged in §11.2 path (4) — would convert 5/6 of the failure cells back to gv-eligible.
2. **Raise `n_predict` above 4096** or add explicit per-block budgeting to handle the long-form articles (33632, 35352, 36065 all hit `stop=limit`).
3. **Re-attach numeric / ticker / org ground truth** to the n=9 Phase 2 Arm B subset (or switch to the n=10 acceptance corpus that already has it) — otherwise §6.4 gates remain unmeasurable here regardless of model output.
4. **Rerun n=9 after (1)+(3)** before treating any of the recall gates as actually failed; the current FAIL is dominated by a fixable preprocessor bug + missing GT labels, not by model competence on the copy-mode hypothesis.

---

# Aggregate report (auto-generated by `aggregate_report.py`)

# phase3-v2_1-n9 CoD DSL benchmark — aggregate report

Total result files: **9**
Models seen: **1**, variants: **1**

Phase 0 instrumentation (DSL PoC plan §3) — per-category recall breakdown, span_copy_fraction mechanism metric, and latency-per-recall-point cost metric.

Legacy columns (preserved for apples-to-apples vs R1/R2/R3): `n`, `gv%` (grammar-valid rate), `ent`/`evt`/`claim`/`note` (mean block counts), `ent_rec` (legacy combined ticker+company recall), `num_rec` (legacy lenient numeric recall, == `num_sem` below), `undecl`, `miss_slot`.

Phase 0 columns: `tick`/`org`/`pers`/`loc`/`inst`/`sect`/`ind` (per-category recall; n/a when ground truth lacks the category), `num_exa` (numeric exact-match rate), `num_sem` (numeric semantic recall — full literal OR any digit-token), `unit_pre` (unit-token preservation), `span_cp` (n=4 char n-gram fraction of output appearing verbatim in input), `wall_s` (mean wall seconds), `cost` (wall_s / (mean(num+tick+org recall) × 100), plan §3.3).

## Per-model averages — legacy view (R1/R2/R3 parity)

| model | class | variant | n | gv% | ent | evt | claim | note | ent_rec | num_rec | undecl | miss_slot |
|---|---|---|---|---|---|---|---|---|---|---|---|---|
| qwen3:30b-a3b-instruct-2507-q4_K_M | B | v2_1_spec | 9 | 33.3% | 1.00 | 1.89 | 0.00 | 0.11 | n/a | n/a | 0.00 | 1.89 |

## Per-model averages — Phase 0 per-category recall

| model | class | variant | n | tick | org | pers | loc | inst | sect | ind | num_exa | num_sem | unit_pre | span_cp |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| qwen3:30b-a3b-instruct-2507-q4_K_M | B | v2_1_spec | 9 | n/a | n/a | n/a | n/a | n/a | n/a | n/a | n/a | n/a | n/a | 43.9% |

## Per-model averages — Phase 3 v2 verifier metrics

`verbatim` = fraction of v2 copy slots whose value matches `input[source_span]` byte-for-byte. `span_val` = fraction of v2 spans that are well-formed (int, int) within input bounds. `verifier` = fraction of blocks scored `pass` by the deterministic verifier (no hard_fail). All three are `n/a` for v1 cells.

| model | class | variant | n | verbatim | span_val | verifier |
|---|---|---|---|---|---|---|
| qwen3:30b-a3b-instruct-2507-q4_K_M | B | v2_1_spec | 9 | 61.1% | 0.0% | 94.4% |

## Per-model averages — latency × recall cost (plan §3.3)

`cost` = mean(wall_s) / (mean target_recall × 100); lower is better. Target recall = unweighted mean of available {numeric, ticker, org} recalls. ``n/a`` when no target-recall component is available.

| model | class | variant | n | wall_s | tick | org | num_sem | cost |
|---|---|---|---|---|---|---|---|---|
| qwen3:30b-a3b-instruct-2507-q4_K_M | B | v2_1_spec | 9 | 169.03 | n/a | n/a | n/a | n/a |

## Per-class averages

| class | variant | n | gv% | ent | evt | claim | note | ent_rec | num_rec | undecl | miss_slot |
|---|---|---|---|---|---|---|---|---|---|---|---|
| A | v2_1_spec | 0 | n/a | n/a | n/a | n/a | n/a | n/a | n/a | n/a | n/a |
| B | v2_1_spec | 9 | 33.3% | 1.00 | 1.89 | 0.00 | 0.11 | n/a | n/a | 0.00 | 1.89 |
| C | v2_1_spec | 0 | n/a | n/a | n/a | n/a | n/a | n/a | n/a | n/a | n/a |

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
| numeric_exact_match_rate | n/a | 0 |
| numeric_semantic_match_rate | n/a | 0 |
| org_recall | n/a | 0 |

---

Generated by `aggregate_report.py` from `docs/benchmarks/cod-2026-05-17/results/phase3-v2_1-n9`.
