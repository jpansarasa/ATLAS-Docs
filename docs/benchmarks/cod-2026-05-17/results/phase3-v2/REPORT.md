# phase3-v2 CoD DSL benchmark — aggregate report

Total result files: **9**
Models seen: **1**, variants: **1**

Phase 0 instrumentation (DSL PoC plan §3) — per-category recall breakdown, span_copy_fraction mechanism metric, and latency-per-recall-point cost metric.

Legacy columns (preserved for apples-to-apples vs R1/R2/R3): `n`, `gv%` (grammar-valid rate), `ent`/`evt`/`claim`/`note` (mean block counts), `ent_rec` (legacy combined ticker+company recall), `num_rec` (legacy lenient numeric recall, == `num_sem` below), `undecl`, `miss_slot`.

Phase 0 columns: `tick`/`org`/`pers`/`loc`/`inst`/`sect`/`ind` (per-category recall; n/a when ground truth lacks the category), `num_exa` (numeric exact-match rate), `num_sem` (numeric semantic recall — full literal OR any digit-token), `unit_pre` (unit-token preservation), `span_cp` (n=4 char n-gram fraction of output appearing verbatim in input), `wall_s` (mean wall seconds), `cost` (wall_s / (mean(num+tick+org recall) × 100), plan §3.3).

## Per-model averages — legacy view (R1/R2/R3 parity)

| model | class | variant | n | gv% | ent | evt | claim | note | ent_rec | num_rec | undecl | miss_slot |
|---|---|---|---|---|---|---|---|---|---|---|---|---|
| qwen3:30b-a3b-instruct-2507-q4_K_M | B | v2_spec | 9 | 33.3% | 1.56 | 1.89 | 0.33 | 1.22 | 100.0% | 55.6% | 0.00 | 2.22 |

## Per-model averages — Phase 0 per-category recall

| model | class | variant | n | tick | org | pers | loc | inst | sect | ind | num_exa | num_sem | unit_pre | span_cp |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| qwen3:30b-a3b-instruct-2507-q4_K_M | B | v2_spec | 9 | n/a | 100.0% | n/a | n/a | n/a | n/a | n/a | 44.4% | 55.6% | 87.5% | 28.5% |

## Per-model averages — Phase 3 v2 verifier metrics

`verbatim` = fraction of v2 copy slots whose value matches `input[source_span]` byte-for-byte. `span_val` = fraction of v2 spans that are well-formed (int, int) within input bounds. `verifier` = fraction of blocks scored `pass` by the deterministic verifier (no hard_fail). All three are `n/a` for v1 cells.

| model | class | variant | n | verbatim | span_val | verifier |
|---|---|---|---|---|---|---|
| qwen3:30b-a3b-instruct-2507-q4_K_M | B | v2_spec | 9 | 0.0% | 100.0% | 38.9% |

## Per-model averages — latency × recall cost (plan §3.3)

`cost` = mean(wall_s) / (mean target_recall × 100); lower is better. Target recall = unweighted mean of available {numeric, ticker, org} recalls. ``n/a`` when no target-recall component is available.

| model | class | variant | n | wall_s | tick | org | num_sem | cost |
|---|---|---|---|---|---|---|---|---|
| qwen3:30b-a3b-instruct-2507-q4_K_M | B | v2_spec | 9 | 177.82 | n/a | 100.0% | 55.6% | 2.29 |

## Per-class averages

| class | variant | n | gv% | ent | evt | claim | note | ent_rec | num_rec | undecl | miss_slot |
|---|---|---|---|---|---|---|---|---|---|---|---|
| A | v2_spec | 0 | n/a | n/a | n/a | n/a | n/a | n/a | n/a | n/a | n/a |
| B | v2_spec | 9 | 33.3% | 1.56 | 1.89 | 0.33 | 1.22 | 100.0% | 55.6% | 0.00 | 2.22 |
| C | v2_spec | 0 | n/a | n/a | n/a | n/a | n/a | n/a | n/a | n/a | n/a |

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
| numeric_exact_match_rate | n/a | 1 |
| numeric_semantic_match_rate | n/a | 1 |
| org_recall | n/a | 1 |

## Per-cell breakdown (n=9, 27772 skipped)

Wall in seconds. `stop` = llama-server `stop_type` (`eos` = clean,
`limit` = `n_predict=2048` exhausted mid-document). `gv` = parser
`grammar_valid`. Metric cells `n/a` when `gv=False` (parser drops
the document so per-slot recall is undefined).

| id | wall | stop | gv | verifier | num_exa | ticker | org | span_val | verbatim | eval_tok |
|---|---|---|---|---|---|---|---|---|---|---|
| 27560 | 343.8 | eos   | False | n/a  | n/a  | n/a | n/a   | n/a  | n/a  | 4370 |
| 27702 |  75.8 | eos   | True  | 1.00 | n/a  | n/a | n/a   | 1.00 | 0.00 |  772 |
| 29807 | 150.0 | eos   | False | n/a  | n/a  | n/a | n/a   | n/a  | n/a  | 1956 |
| 31149 | 138.7 | eos   | False | n/a  | n/a  | n/a | n/a   | n/a  | n/a  | 1791 |
| 31430 | 100.6 | eos   | True  | 0.17 | 0.44 | n/a | 1.00  | 1.00 | 0.00 |  964 |
| 33632 | 213.3 | limit | False | n/a  | n/a  | n/a | n/a   | n/a  | n/a  | 2048 |
| 34537 |  88.1 | eos   | True  | 0.00 | n/a  | n/a | n/a   | 1.00 | 0.00 | 1006 |
| 35352 | 221.5 | limit | False | n/a  | n/a  | n/a | n/a   | n/a  | n/a  | 2048 |
| 36065 | 268.4 | limit | False | n/a  | n/a  | n/a | n/a   | n/a  | n/a  | 2048 |

`27772` skipped — known 540s+ timeout case from Phase 2 Arm B
(dispatch instruction).

## Acceptance verdicts (plan §11.2 gates)

n=9 (full-corpus n=10 minus skipped 27772). The org / numeric /
ticker recalls are computed over the subset of cells with
`grammar_valid=True` (n=3): 27702, 31430, 34537. The other 6 cells
have `grammar_valid=False` so all per-slot metrics are `None` and
excluded from the means by the aggregator's `_mean` filter.

| gate                          | threshold | observed (n=9) | observed (gv-only n=3) | verdict |
|---|---|---|---|---|
| `numeric_exact_match_rate`    | ≥ 0.85    | 0.44           | 0.44                    | **FAIL** |
| `ticker_recall`               | ≥ 0.40    | n/a (no ticker GT in corpus subset) | n/a | **N/A** |
| `org_recall`                  | ≥ 0.50    | 1.00           | 1.00                    | **PASS** (single ground-truth org cell) |
| `verifier_pass_rate`          | ≥ 0.95    | 0.389          | 0.389                   | **FAIL** |

Secondary observations:
- `grammar_valid` rate: **33.3%** (3/9). Two failure modes dominate:
  (a) the v2 GBNF permits documents that the v2 lark parser rejects
  (e.g. duplicate ENT declarations on 27560, ENT block without a
  required `name` slot on 31149); (b) `n_predict=2048` truncates
  mid-document on long inputs (33632, 35352, 36065 — all
  `stop=limit`).
- `verbatim_match_rate`: **0.0%** across all 3 gv-valid cells. The
  model emits `name: "..."` slot values that the verifier deems
  non-verbatim against the supplied `source_span`. Plausible
  causes: span-boundary off-by-one in the model's offset arithmetic,
  or the model emitting normalized/quoted forms instead of literal
  bytes. Mechanism not investigated here per dispatch instructions
  (no grammar/prompt iteration; report honestly).
- `span_validity_rate`: **100%** in gv-valid cells — every emitted
  span is a well-formed `(int, int)` pair within bounds. The span
  *bytes* don't match `name`, but the spans themselves are
  structurally valid.

## Overall verdict

**FAIL** on the two gates with sufficient data. The v2 schema arm
does not clear acceptance at n=9 on `qwen3:30b-a3b-instruct-2507-q4_K_M`
under the current grammar + prompt + `n_predict=2048` budget. The
33.3% `grammar_valid` rate is the primary failure: roughly two-
thirds of cells either (a) produce documents the parser rejects on
shape rules the grammar fails to enforce, or (b) get truncated by
the predict-token cap before completing.

Per Phase 2 convention: no grammar/prompt iteration in this report.
The failure surface is documented for the upstream story.

---

Generated by `aggregate_report.py` from `results/phase3-v2`.
