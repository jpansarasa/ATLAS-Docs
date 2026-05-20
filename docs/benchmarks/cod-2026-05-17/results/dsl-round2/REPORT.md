# Round 2 CoD DSL benchmark — aggregate report

Total result files: **140**
Models seen: **7**, variants: **2**

Phase 0 instrumentation (DSL PoC plan §3) — per-category recall breakdown, span_copy_fraction mechanism metric, and latency-per-recall-point cost metric.

Legacy columns (preserved for apples-to-apples vs R1/R2/R3): `n`, `gv%` (grammar-valid rate), `ent`/`evt`/`claim`/`note` (mean block counts), `ent_rec` (legacy combined ticker+company recall), `num_rec` (legacy lenient numeric recall, == `num_sem` below), `undecl`, `miss_slot`.

Phase 0 columns: `tick`/`org`/`pers`/`loc`/`inst`/`sect`/`ind` (per-category recall; n/a when ground truth lacks the category), `num_exa` (numeric exact-match rate), `num_sem` (numeric semantic recall — full literal OR any digit-token), `unit_pre` (unit-token preservation), `span_cp` (n=4 char n-gram fraction of output appearing verbatim in input), `wall_s` (mean wall seconds), `cost` (wall_s / (mean(num+tick+org recall) × 100), plan §3.3).

## Per-model averages — legacy view (R1/R2/R3 parity)

| model | class | variant | n | gv% | ent | evt | claim | note | ent_rec | num_rec | undecl | miss_slot |
|---|---|---|---|---|---|---|---|---|---|---|---|---|
| qwen2.5:7b-instruct | A | spec | 10 | 80.0% | 6.50 | 4.00 | 1.00 | 0.50 | 16.3% | 51.7% | 0.00 | 0.00 |
| qwen2.5:7b-instruct | A | spec+example | 10 | 90.0% | 8.30 | 8.30 | 0.30 | 0.30 | 15.9% | 80.5% | 0.00 | 0.00 |
| qwen3:8b | A | spec | 10 | 100.0% | 8.00 | 10.90 | 1.10 | 0.00 | 17.1% | 81.9% | 0.00 | 0.00 |
| qwen3:8b | A | spec+example | 10 | 100.0% | 7.40 | 9.30 | 0.30 | 0.00 | 32.9% | 87.7% | 0.00 | 0.00 |
| granite4.1:8b | A | spec | 10 | 60.0% | 2.60 | 2.80 | 0.70 | 0.30 | 25.0% | 85.2% | 0.00 | 0.00 |
| granite4.1:8b | A | spec+example | 10 | 90.0% | 7.40 | 8.80 | 1.00 | 0.80 | 16.3% | 68.4% | 0.00 | 0.00 |
| phi4-mini:3.8b | A | spec | 10 | 20.0% | 0.80 | 0.50 | 0.00 | 0.00 | 50.0% | 100.0% | 0.00 | 0.00 |
| phi4-mini:3.8b | A | spec+example | 10 | 80.0% | 5.10 | 4.10 | 1.10 | 0.00 | 6.2% | 19.2% | 0.00 | 0.10 |
| gemma4:e4b | A | spec | 10 | 90.0% | 9.20 | 17.10 | 3.10 | 0.00 | 26.5% | 88.2% | 0.00 | 0.00 |
| gemma4:e4b | A | spec+example | 10 | 100.0% | 20.80 | 19.40 | 0.60 | 0.10 | 19.2% | 87.7% | 0.00 | 0.00 |
| gemma4:26b-a4b-it-q4_K_M | B | spec | 10 | 0.0% | 0.00 | 0.00 | 0.00 | 0.00 | n/a | n/a | 0.00 | 0.00 |
| gemma4:26b-a4b-it-q4_K_M | B | spec+example | 10 | 20.0% | 2.10 | 3.40 | 0.10 | 0.00 | 50.0% | 46.8% | 0.00 | 0.00 |
| qwen3:30b-a3b-instruct-2507-q4_K_M | B | spec | 10 | 90.0% | 8.60 | 23.50 | 1.70 | 0.40 | 31.7% | 91.7% | 0.00 | 0.00 |
| qwen3:30b-a3b-instruct-2507-q4_K_M | B | spec+example | 10 | 80.0% | 9.60 | 13.20 | 1.10 | 0.00 | 11.2% | 89.0% | 0.00 | 0.00 |

## Per-model averages — Phase 0 per-category recall

| model | class | variant | n | tick | org | pers | loc | inst | sect | ind | num_exa | num_sem | unit_pre | span_cp |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| qwen2.5:7b-instruct | A | spec | 10 | 0.0% | 20.4% | n/a | n/a | n/a | 100.0% | 59.4% | 43.7% | 51.7% | 69.1% | 51.8% |
| qwen2.5:7b-instruct | A | spec+example | 10 | 0.0% | 16.0% | n/a | n/a | n/a | 100.0% | 8.3% | 57.4% | 80.5% | 83.0% | 58.0% |
| qwen3:8b | A | spec | 10 | 50.0% | 18.5% | n/a | n/a | n/a | 100.0% | 35.4% | 59.1% | 81.9% | 80.3% | 58.4% |
| qwen3:8b | A | spec+example | 10 | 50.0% | 34.8% | n/a | n/a | n/a | 100.0% | 48.3% | 63.4% | 87.7% | 83.0% | 48.9% |
| granite4.1:8b | A | spec | 10 | 0.0% | 32.5% | n/a | n/a | n/a | 0.0% | 100.0% | 52.1% | 85.2% | 66.7% | 59.5% |
| granite4.1:8b | A | spec+example | 10 | 0.0% | 17.9% | n/a | n/a | n/a | 100.0% | 59.2% | 60.8% | 68.4% | 80.1% | 58.7% |
| phi4-mini:3.8b | A | spec | 10 | n/a | 50.0% | n/a | n/a | n/a | n/a | 0.0% | 100.0% | 100.0% | 100.0% | 34.2% |
| phi4-mini:3.8b | A | spec+example | 10 | 0.0% | 6.2% | n/a | n/a | n/a | 100.0% | 13.3% | 5.5% | 19.2% | 56.3% | 22.9% |
| gemma4:e4b | A | spec | 10 | 0.0% | 29.7% | n/a | n/a | n/a | 100.0% | 44.2% | 65.5% | 88.2% | 80.8% | 47.9% |
| gemma4:e4b | A | spec+example | 10 | 0.0% | 24.4% | n/a | n/a | n/a | 100.0% | 48.8% | 65.9% | 87.7% | 82.7% | 52.6% |
| gemma4:26b-a4b-it-q4_K_M | B | spec | 10 | n/a | n/a | n/a | n/a | n/a | n/a | n/a | n/a | n/a | n/a | 58.7% |
| gemma4:26b-a4b-it-q4_K_M | B | spec+example | 10 | n/a | 50.0% | n/a | n/a | n/a | n/a | 20.0% | 35.6% | 46.8% | 48.5% | 48.0% |
| qwen3:30b-a3b-instruct-2507-q4_K_M | B | spec | 10 | 0.0% | 36.2% | n/a | n/a | n/a | 100.0% | 60.0% | 68.2% | 91.7% | 81.2% | 58.9% |
| qwen3:30b-a3b-instruct-2507-q4_K_M | B | spec+example | 10 | 0.0% | 12.8% | n/a | n/a | n/a | 100.0% | 60.0% | 84.6% | 89.0% | 96.9% | 54.2% |

## Per-model averages — latency × recall cost (plan §3.3)

`cost` = mean(wall_s) / (mean target_recall × 100); lower is better. Target recall = unweighted mean of available {numeric, ticker, org} recalls. ``n/a`` when no target-recall component is available.

| model | class | variant | n | wall_s | tick | org | num_sem | cost |
|---|---|---|---|---|---|---|---|---|
| qwen2.5:7b-instruct | A | spec | 10 | 210.63 | 0.0% | 20.4% | 51.7% | 8.76 |
| qwen2.5:7b-instruct | A | spec+example | 10 | 207.07 | 0.0% | 16.0% | 80.5% | 6.44 |
| qwen3:8b | A | spec | 10 | 431.04 | 50.0% | 18.5% | 81.9% | 8.60 |
| qwen3:8b | A | spec+example | 10 | 327.78 | 50.0% | 34.8% | 87.7% | 5.70 |
| granite4.1:8b | A | spec | 10 | 170.82 | 0.0% | 32.5% | 85.2% | 4.35 |
| granite4.1:8b | A | spec+example | 10 | 195.07 | 0.0% | 17.9% | 68.4% | 6.78 |
| phi4-mini:3.8b | A | spec | 10 | 171.55 | n/a | 50.0% | 100.0% | 2.29 |
| phi4-mini:3.8b | A | spec+example | 10 | 197.70 | 0.0% | 6.2% | 19.2% | 23.30 |
| gemma4:e4b | A | spec | 10 | 213.27 | 0.0% | 29.7% | 88.2% | 5.43 |
| gemma4:e4b | A | spec+example | 10 | 218.99 | 0.0% | 24.4% | 87.7% | 5.86 |
| gemma4:26b-a4b-it-q4_K_M | B | spec | 10 | 815.61 | n/a | n/a | n/a | n/a |
| gemma4:26b-a4b-it-q4_K_M | B | spec+example | 10 | 683.68 | n/a | 50.0% | 46.8% | 14.13 |
| qwen3:30b-a3b-instruct-2507-q4_K_M | B | spec | 10 | 321.20 | 0.0% | 36.2% | 91.7% | 7.53 |
| qwen3:30b-a3b-instruct-2507-q4_K_M | B | spec+example | 10 | 330.93 | 0.0% | 12.8% | 89.0% | 9.76 |

## Per-class averages

| class | variant | n | gv% | ent | evt | claim | note | ent_rec | num_rec | undecl | miss_slot |
|---|---|---|---|---|---|---|---|---|---|---|---|
| A | spec | 50 | 70.0% | 5.42 | 7.06 | 1.18 | 0.16 | 22.3% | 78.0% | 0.00 | 0.00 |
| A | spec+example | 50 | 92.0% | 9.80 | 9.98 | 0.66 | 0.24 | 18.5% | 70.1% | 0.00 | 0.02 |
| B | spec | 20 | 45.0% | 4.30 | 11.75 | 0.85 | 0.20 | 31.7% | 91.7% | 0.00 | 0.00 |
| B | spec+example | 20 | 50.0% | 5.85 | 8.30 | 0.60 | 0.00 | 17.7% | 77.0% | 0.00 | 0.00 |
| C | spec | 0 | n/a | n/a | n/a | n/a | n/a | n/a | n/a | n/a | n/a |
| C | spec+example | 0 | n/a | n/a | n/a | n/a | n/a | n/a | n/a | n/a | n/a |

## Variant deltas (`spec+example` minus `spec`)

Positive means the worked-example arm improved that metric. Cells with `n/a` indicate one or both arms were empty.

| model | class | Δ gv% | Δ ent | Δ evt | Δ claim | Δ ent_rec | Δ num_rec | Δ undecl | Δ miss_slot |
|---|---|---|---|---|---|---|---|---|---|
| qwen2.5:7b-instruct | A | 10.0% | 1.80 | 4.30 | -0.70 | -0.5% | 28.8% | 0.00 | 0.00 |
| qwen3:8b | A | 0.0% | -0.60 | -1.60 | -0.80 | 15.8% | 5.7% | 0.00 | 0.00 |
| granite4.1:8b | A | 30.0% | 4.80 | 6.00 | 0.30 | -8.7% | -16.8% | 0.00 | 0.00 |
| phi4-mini:3.8b | A | 60.0% | 4.30 | 3.60 | 1.10 | -43.8% | -80.8% | 0.00 | 0.10 |
| gemma4:e4b | A | 10.0% | 11.60 | 2.30 | -2.50 | -7.3% | -0.6% | 0.00 | 0.00 |
| gemma4:26b-a4b-it-q4_K_M | B | 20.0% | 2.10 | 3.40 | 0.10 | n/a | n/a | 0.00 | 0.00 |
| qwen3:30b-a3b-instruct-2507-q4_K_M | B | -10.0% | 1.00 | -10.30 | -0.60 | -20.4% | -2.7% | 0.00 | 0.00 |

Per-class deltas:

| class | Δ gv% | Δ ent | Δ evt | Δ claim | Δ ent_rec | Δ num_rec | Δ undecl | Δ miss_slot |
|---|---|---|---|---|---|---|---|---|
| A | 22.0% | 4.38 | 2.92 | -0.52 | -3.7% | -7.9% | 0.00 | 0.02 |
| B | 5.0% | 1.55 | -3.45 | -0.25 | -14.0% | -14.8% | 0.00 | 0.00 |
| C | n/a | n/a | n/a | n/a | n/a | n/a | n/a | n/a |

## Span-copy correlations (plan §3.2 mechanism check)

Pearson `r` of per-cell `span_copy_fraction` against the indicated recall. Positive `r` supports the induction-head copy-mode hypothesis (verbatim copies preserve numerics and entity surfaces better than paraphrase/regeneration). `n` is the number of (model, variant, article) cells with both metrics defined.

| target metric | Pearson r | n pairs |
|---|---|---|
| numeric_exact_match_rate | +0.505 | 69 |
| numeric_semantic_match_rate | +0.421 | 69 |
| org_recall | -0.038 | 58 |

---

Generated by `aggregate_report.py` from `results/dsl-round2`.
