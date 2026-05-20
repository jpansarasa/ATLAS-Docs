# Round 3 CoD DSL benchmark — aggregate report

Total result files: **140**
Models seen: **7**, variants: **2**

Phase 0 instrumentation (DSL PoC plan §3) — per-category recall breakdown, span_copy_fraction mechanism metric, and latency-per-recall-point cost metric.

Legacy columns (preserved for apples-to-apples vs R1/R2/R3): `n`, `gv%` (grammar-valid rate), `ent`/`evt`/`claim`/`note` (mean block counts), `ent_rec` (legacy combined ticker+company recall), `num_rec` (legacy lenient numeric recall, == `num_sem` below), `undecl`, `miss_slot`.

Phase 0 columns: `tick`/`org`/`pers`/`loc`/`inst`/`sect`/`ind` (per-category recall; n/a when ground truth lacks the category), `num_exa` (numeric exact-match rate), `num_sem` (numeric semantic recall — full literal OR any digit-token), `unit_pre` (unit-token preservation), `span_cp` (n=4 char n-gram fraction of output appearing verbatim in input), `wall_s` (mean wall seconds), `cost` (wall_s / (mean(num+tick+org recall) × 100), plan §3.3).

## Per-model averages — legacy view (R1/R2/R3 parity)

| model | class | variant | n | gv% | ent | evt | claim | note | ent_rec | num_rec | undecl | miss_slot |
|---|---|---|---|---|---|---|---|---|---|---|---|---|
| qwen2.5:7b-instruct | A | spec | 10 | 90.0% | 13.80 | 7.10 | 0.60 | 0.40 | 11.5% | 65.2% | 0.00 | 0.10 |
| qwen2.5:7b-instruct | A | spec+example | 10 | 80.0% | 5.80 | 5.40 | 0.90 | 0.20 | 15.3% | 78.6% | 0.00 | 0.10 |
| qwen3:8b | A | spec | 10 | 70.0% | 3.10 | 2.60 | 12.10 | 0.10 | 19.5% | 88.8% | 0.00 | 0.00 |
| qwen3:8b | A | spec+example | 10 | 90.0% | 8.80 | 12.00 | 0.90 | 0.00 | 18.4% | 86.4% | 0.00 | 0.00 |
| granite4.1:8b | A | spec | 10 | 40.0% | 6.40 | 6.30 | 0.20 | 0.10 | 16.7% | 88.9% | 0.00 | 0.00 |
| granite4.1:8b | A | spec+example | 10 | 90.0% | 4.30 | 15.20 | 0.80 | 0.70 | 21.8% | 73.9% | 0.00 | 0.00 |
| phi4-mini:3.8b | A | spec | 10 | 20.0% | 1.60 | 0.80 | 0.00 | 0.00 | 25.0% | 88.9% | 0.00 | 0.00 |
| phi4-mini:3.8b | A | spec+example | 10 | 70.0% | 3.80 | 3.70 | 0.60 | 0.10 | 6.2% | 15.8% | 0.00 | 0.00 |
| gemma4:e4b | A | spec | 10 | 100.0% | 10.00 | 8.10 | 13.10 | 0.10 | 36.9% | 85.7% | 0.00 | 0.00 |
| gemma4:e4b | A | spec+example | 10 | 90.0% | 8.10 | 7.60 | 0.30 | 0.20 | 28.2% | 66.5% | 0.00 | 0.10 |
| gemma4:26b-a4b-it-q4_K_M | B | spec | 10 | 30.0% | 2.10 | 2.50 | 0.00 | 0.00 | 7.7% | 23.6% | 0.00 | 0.00 |
| gemma4:26b-a4b-it-q4_K_M | B | spec+example | 10 | 20.0% | 1.10 | 2.60 | 0.00 | 0.00 | n/a | n/a | 0.00 | 0.00 |
| qwen3:30b-a3b-instruct-2507-q4_K_M | B | spec | 10 | 100.0% | 10.90 | 27.70 | 1.70 | 0.20 | 29.4% | 88.6% | 0.00 | 0.00 |
| qwen3:30b-a3b-instruct-2507-q4_K_M | B | spec+example | 10 | 90.0% | 9.40 | 24.00 | 1.00 | 0.10 | 8.8% | 90.8% | 0.00 | 0.00 |

## Per-model averages — Phase 0 per-category recall

| model | class | variant | n | tick | org | pers | loc | inst | sect | ind | num_exa | num_sem | unit_pre | span_cp |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| qwen2.5:7b-instruct | A | spec | 10 | 0.0% | 14.7% | n/a | n/a | n/a | 100.0% | 50.0% | 57.5% | 65.2% | 79.8% | 57.6% |
| qwen2.5:7b-instruct | A | spec+example | 10 | 0.0% | 16.9% | n/a | n/a | n/a | 0.0% | 40.0% | 53.2% | 78.6% | 77.1% | 56.8% |
| qwen3:8b | A | spec | 10 | 50.0% | 23.3% | n/a | n/a | n/a | 0.0% | 68.8% | 62.4% | 88.8% | 76.1% | 59.7% |
| qwen3:8b | A | spec+example | 10 | 50.0% | 19.9% | n/a | n/a | n/a | 100.0% | 50.4% | 79.0% | 86.4% | 96.8% | 52.4% |
| granite4.1:8b | A | spec | 10 | 0.0% | 21.7% | n/a | n/a | n/a | 0.0% | 100.0% | 88.9% | 88.9% | 100.0% | 54.8% |
| granite4.1:8b | A | spec+example | 10 | 0.0% | 23.6% | n/a | n/a | n/a | 100.0% | 61.2% | 47.6% | 73.9% | 76.8% | 55.9% |
| phi4-mini:3.8b | A | spec | 10 | 0.0% | 40.0% | n/a | n/a | n/a | 0.0% | n/a | 66.7% | 88.9% | 100.0% | 25.0% |
| phi4-mini:3.8b | A | spec+example | 10 | 0.0% | 6.2% | n/a | n/a | n/a | 0.0% | 13.3% | 5.9% | 15.8% | 50.6% | 25.2% |
| gemma4:e4b | A | spec | 10 | 0.0% | 40.0% | n/a | n/a | n/a | 100.0% | 17.1% | 64.9% | 85.7% | 82.7% | 59.2% |
| gemma4:e4b | A | spec+example | 10 | 0.0% | 28.3% | n/a | n/a | n/a | n/a | 50.4% | 59.0% | 66.5% | 80.5% | 58.1% |
| gemma4:26b-a4b-it-q4_K_M | B | spec | 10 | 0.0% | 8.3% | n/a | n/a | n/a | n/a | 0.0% | 9.3% | 23.6% | 44.7% | 47.8% |
| gemma4:26b-a4b-it-q4_K_M | B | spec+example | 10 | n/a | n/a | n/a | n/a | n/a | n/a | n/a | n/a | n/a | n/a | 61.5% |
| qwen3:30b-a3b-instruct-2507-q4_K_M | B | spec | 10 | 50.0% | 31.0% | n/a | n/a | n/a | 100.0% | 60.8% | 66.9% | 88.6% | 82.2% | 62.5% |
| qwen3:30b-a3b-instruct-2507-q4_K_M | B | spec+example | 10 | 0.0% | 8.8% | n/a | n/a | n/a | 100.0% | 60.0% | 68.2% | 90.8% | 80.7% | 50.3% |

## Per-model averages — latency × recall cost (plan §3.3)

`cost` = mean(wall_s) / (mean target_recall × 100); lower is better. Target recall = unweighted mean of available {numeric, ticker, org} recalls. ``n/a`` when no target-recall component is available.

| model | class | variant | n | wall_s | tick | org | num_sem | cost |
|---|---|---|---|---|---|---|---|---|
| qwen2.5:7b-instruct | A | spec | 10 | 311.52 | 0.0% | 14.7% | 65.2% | 11.71 |
| qwen2.5:7b-instruct | A | spec+example | 10 | 169.20 | 0.0% | 16.9% | 78.6% | 5.32 |
| qwen3:8b | A | spec | 10 | 547.31 | 50.0% | 23.3% | 88.8% | 10.13 |
| qwen3:8b | A | spec+example | 10 | 350.06 | 50.0% | 19.9% | 86.4% | 6.72 |
| granite4.1:8b | A | spec | 10 | 235.78 | 0.0% | 21.7% | 88.9% | 6.40 |
| granite4.1:8b | A | spec+example | 10 | 226.51 | 0.0% | 23.6% | 73.9% | 6.97 |
| phi4-mini:3.8b | A | spec | 10 | 156.04 | 0.0% | 40.0% | 88.9% | 3.63 |
| phi4-mini:3.8b | A | spec+example | 10 | 130.23 | 0.0% | 6.2% | 15.8% | 17.73 |
| gemma4:e4b | A | spec | 10 | 243.83 | 0.0% | 40.0% | 85.7% | 5.82 |
| gemma4:e4b | A | spec+example | 10 | 186.15 | 0.0% | 28.3% | 66.5% | 5.89 |
| gemma4:26b-a4b-it-q4_K_M | B | spec | 10 | 1001.33 | 0.0% | 8.3% | 23.6% | 93.93 |
| gemma4:26b-a4b-it-q4_K_M | B | spec+example | 10 | 1144.82 | n/a | n/a | n/a | n/a |
| qwen3:30b-a3b-instruct-2507-q4_K_M | B | spec | 10 | 339.72 | 50.0% | 31.0% | 88.6% | 6.01 |
| qwen3:30b-a3b-instruct-2507-q4_K_M | B | spec+example | 10 | 422.77 | 0.0% | 8.8% | 90.8% | 12.74 |

## Per-class averages

| class | variant | n | gv% | ent | evt | claim | note | ent_rec | num_rec | undecl | miss_slot |
|---|---|---|---|---|---|---|---|---|---|---|---|
| A | spec | 50 | 64.0% | 6.98 | 4.98 | 5.20 | 0.14 | 22.7% | 81.4% | 0.00 | 0.02 |
| A | spec+example | 50 | 84.0% | 6.16 | 8.78 | 0.70 | 0.24 | 18.5% | 65.9% | 0.00 | 0.04 |
| B | spec | 20 | 65.0% | 6.50 | 15.10 | 0.85 | 0.10 | 26.3% | 74.2% | 0.00 | 0.00 |
| B | spec+example | 20 | 55.0% | 5.25 | 13.30 | 0.50 | 0.05 | 8.8% | 90.8% | 0.00 | 0.00 |
| C | spec | 0 | n/a | n/a | n/a | n/a | n/a | n/a | n/a | n/a | n/a |
| C | spec+example | 0 | n/a | n/a | n/a | n/a | n/a | n/a | n/a | n/a | n/a |

## Variant deltas (`spec+example` minus `spec`)

Positive means the worked-example arm improved that metric. Cells with `n/a` indicate one or both arms were empty.

| model | class | Δ gv% | Δ ent | Δ evt | Δ claim | Δ ent_rec | Δ num_rec | Δ undecl | Δ miss_slot |
|---|---|---|---|---|---|---|---|---|---|
| qwen2.5:7b-instruct | A | -10.0% | -8.00 | -1.70 | 0.30 | 3.7% | 13.4% | 0.00 | 0.00 |
| qwen3:8b | A | 20.0% | 5.70 | 9.40 | -11.20 | -1.0% | -2.4% | 0.00 | 0.00 |
| granite4.1:8b | A | 50.0% | -2.10 | 8.90 | 0.60 | 5.2% | -14.9% | 0.00 | 0.00 |
| phi4-mini:3.8b | A | 50.0% | 2.20 | 2.90 | 0.60 | -18.8% | -73.1% | 0.00 | 0.00 |
| gemma4:e4b | A | -10.0% | -1.90 | -0.50 | -12.80 | -8.7% | -19.2% | 0.00 | 0.10 |
| gemma4:26b-a4b-it-q4_K_M | B | -10.0% | -1.00 | 0.10 | 0.00 | n/a | n/a | 0.00 | 0.00 |
| qwen3:30b-a3b-instruct-2507-q4_K_M | B | -10.0% | -1.50 | -3.70 | -0.70 | -20.7% | 2.2% | 0.00 | 0.00 |

Per-class deltas:

| class | Δ gv% | Δ ent | Δ evt | Δ claim | Δ ent_rec | Δ num_rec | Δ undecl | Δ miss_slot |
|---|---|---|---|---|---|---|---|---|
| A | 20.0% | -0.82 | 3.80 | -4.50 | -4.2% | -15.5% | 0.00 | 0.02 |
| B | -10.0% | -1.25 | -1.80 | -0.35 | -17.6% | 16.7% | 0.00 | 0.00 |
| C | n/a | n/a | n/a | n/a | n/a | n/a | n/a | n/a |

## Span-copy correlations (plan §3.2 mechanism check)

Pearson `r` of per-cell `span_copy_fraction` against the indicated recall. Positive `r` supports the induction-head copy-mode hypothesis (verbatim copies preserve numerics and entity surfaces better than paraphrase/regeneration). `n` is the number of (model, variant, article) cells with both metrics defined.

| target metric | Pearson r | n pairs |
|---|---|---|
| numeric_exact_match_rate | +0.630 | 66 |
| numeric_semantic_match_rate | +0.363 | 66 |
| org_recall | +0.106 | 56 |

---

Generated by `aggregate_report.py` from `results/dsl-round3`.
