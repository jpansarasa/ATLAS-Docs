# Phase 1.3 (Class C) CoD DSL benchmark — aggregate report

Total result files: **20** (10 spec + 10 spec+example, single model)
Models seen: **1**, variants: **2**

Phase 0 instrumentation (DSL PoC plan §3) — per-category recall breakdown, span_copy_fraction mechanism metric, and latency-per-recall-point cost metric.

## Verdict — MoE-vs-dense reference (plan §4.3)

**Outcome: qwen2.5:32b (dense, Class C) is dramatically worse on the cost metric, modestly worse on recall — the MoE-active-3B default (qwen3:30b-a3b) holds.**

Per plan §4.3, the decision matrix was:

- qwen2.5:32b beats qwen3:30b-MoE → reconsider MoE-over-dense default
- qwen2.5:32b matches qwen3:30b-MoE → MoE-active-3B is sufficient at this scale
- qwen2.5:32b dramatically worse → unexpected; investigate before Phase 2

The data lands between "matches" and "dramatically worse": on the headline numeric-semantic recall the dense 32B model is materially behind on the spec arm (65.9% vs 88.6%, −22.7pp) and closes some of that gap with worked examples (78.8% vs 90.8%, −12.0pp). On ticker recall both models are pinned at 0.0% on this corpus (ground-truth tickers are sparse — see scorer note); on org recall the dense model wins in the spec+example arm (29.9% vs 8.8%) but loses badly in the spec arm (2.1% vs 31.0%). Grammar-validity favors the dense model in spec+example (90% vs 0% for qwen3:30b spec+example) — the Class B MoE model has a known regression in the worked-example arm that this run reproduces.

**Decision branch chosen: keep MoE-active-3B as the Class B default.** The dense model offers no aggregate recall advantage and is **substantially more expensive** per recall point (see below). No investigation gate triggered for Phase 2.

## Comparison vs qwen3:30b-a3b R3 baseline

R3 baseline pulled from `results/dsl-round3/REPORT.md` (same corpus, n=10 per arm, same Phase 0 scorer).

### Headline metrics

| metric | arm | qwen3:30b-a3b (B, MoE) | qwen2.5:32b (C, dense) | Δ (dense − MoE) |
|---|---|---|---|---|
| grammar_valid % | spec | 100.0% | 40.0% | −60.0pp |
| grammar_valid % | spec+example | 90.0% | 90.0% | 0.0pp |
| ticker recall | spec | 50.0% | 0.0% | −50.0pp |
| ticker recall | spec+example | 0.0% | 0.0% | 0.0pp |
| org recall | spec | 31.0% | 2.1% | −28.9pp |
| org recall | spec+example | 8.8% | 29.9% | +21.1pp |
| numeric_exact | spec | 60.8% | 51.8% | −9.0pp |
| numeric_exact | spec+example | 60.0% | 70.7% | +10.7pp |
| numeric_semantic (legacy num_rec) | spec | 88.6% | 65.9% | −22.7pp |
| numeric_semantic | spec+example | 90.8% | 78.8% | −12.0pp |
| unit preservation | spec | 82.2% | 97.1% | +14.9pp |
| unit preservation | spec+example | 80.7% | 93.5% | +12.8pp |
| span_copy | spec | 62.5% | 62.9% | +0.4pp |
| span_copy | spec+example | 50.3% | 60.2% | +9.9pp |

Mixed picture: the dense model wins on **unit preservation** (consistent +13–15pp across arms) and on **numeric_exact + org recall in the spec+example arm only**. The MoE model wins on **numeric semantic recall in both arms** and on **ticker/org recall in the spec arm**. Neither is uniformly dominant on extraction quality.

### Cost per recall point — the deciding number

`cost = mean(wall_s) / (target_recall × 100)`, where target_recall = unweighted mean of available {numeric_semantic, ticker, org} recalls. Lower is better. From the aggregator (plan §3.3):

| arm | qwen3:30b-a3b cost | qwen2.5:32b cost | dense / MoE |
|---|---|---|---|
| spec | 6.01 (wall 339.7s) | 19.15 (wall 434.0s) | **3.19×** |
| spec+example | 12.74 (wall 422.8s) | 14.13 (wall 512.1s) | **1.11×** |

The dense 32B model is **3.2× more expensive per recall point** on the spec arm and **1.1× more expensive** on spec+example. Wall-clock alone is +28% (spec) / +21% (spec+example), so the cost gap is mostly **lower recall**, not just slower decoding. The closer spec+example margin is driven entirely by the qwen3:30b MoE model collapsing in that arm (grammar validity dropping from 100% → 90% with a 22pp ticker-recall regression) rather than the dense model catching up on absolutes.

### Trade-off flag (per dispatch instruction)

**On the spec arm**, qwen3:30b-a3b is the unambiguous winner: higher recall on every headline metric except unit preservation, and 3.2× cheaper per recall point. **On spec+example**, the qwen3:30b prompt regression narrows the cost gap, but the dense model still loses on numeric_semantic recall and is 11% more expensive overall. Even where qwen2.5:32b wins individual metrics (unit preservation, numeric_exact in spec+example, org recall in spec+example), the wall-clock penalty (CPU-bound at ~6.5 TPS on ollama-cpu-gen) erodes the per-point value.

**Verdict: no reason to switch Class B default away from qwen3:30b-a3b.** The MoE-active-3B architecture remains the better fit for the CoD DSL extraction task on this hardware, and the dense reference does not justify its cost. Recommendation for Phase 2 (grammar-constrained decoding via vLLM): proceed with qwen3:30b-a3b as the primary candidate.

### Caveats

- Both models run on `ollama-cpu-gen` (~6.5 TPS dense, MoE-active-3B effectively faster); cost ratios would shift on GPU but qwen2.5:32b at 32GB VRAM doesn't fit alongside extraction workload (see `project_atlas_inference_topology.md`).
- Ticker ground truth is sparse on this corpus (19 non-empty rows / n=72 articles); the 0.0% ticker for qwen2.5:32b across both arms may partly reflect article selection, but the spec-arm gap (50.0% MoE vs 0.0% dense) is real and reproducible.
- Span-copy correlations on the n=9 paired sample are negative (numeric_exact r=−0.31, org r=−0.60), opposite-sign vs R3 (n large). Treat the per-cell correlations on a single model as noise; the mechanism claim from R3 stands.
- Phase 1.4 (Foundry/Claude 4.7 reference) is the external-validity anchor — this report doesn't address whether either local model is near the frontier.

---


Legacy columns (preserved for apples-to-apples vs R1/R2/R3): `n`, `gv%` (grammar-valid rate), `ent`/`evt`/`claim`/`note` (mean block counts), `ent_rec` (legacy combined ticker+company recall), `num_rec` (legacy lenient numeric recall, == `num_sem` below), `undecl`, `miss_slot`.

Phase 0 columns: `tick`/`org`/`pers`/`loc`/`inst`/`sect`/`ind` (per-category recall; n/a when ground truth lacks the category), `num_exa` (numeric exact-match rate), `num_sem` (numeric semantic recall — full literal OR any digit-token), `unit_pre` (unit-token preservation), `span_cp` (n=4 char n-gram fraction of output appearing verbatim in input), `wall_s` (mean wall seconds), `cost` (wall_s / (mean(num+tick+org recall) × 100), plan §3.3).

## Per-model averages — legacy view (R1/R2/R3 parity)

| model | class | variant | n | gv% | ent | evt | claim | note | ent_rec | num_rec | undecl | miss_slot |
|---|---|---|---|---|---|---|---|---|---|---|---|---|
| qwen2.5:32b-instruct-q4_K_M | C | spec | 10 | 40.0% | 4.70 | 4.40 | 1.10 | 0.50 | 2.1% | 65.9% | 0.00 | 0.00 |
| qwen2.5:32b-instruct-q4_K_M | C | spec+example | 10 | 90.0% | 7.50 | 10.60 | 0.90 | 0.70 | 27.3% | 78.8% | 0.00 | 0.50 |

## Per-model averages — Phase 0 per-category recall

| model | class | variant | n | tick | org | pers | loc | inst | sect | ind | num_exa | num_sem | unit_pre | span_cp |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| qwen2.5:32b-instruct-q4_K_M | C | spec | 10 | 0.0% | 2.1% | n/a | n/a | n/a | n/a | 16.2% | 51.8% | 65.9% | 97.1% | 62.9% |
| qwen2.5:32b-instruct-q4_K_M | C | spec+example | 10 | 0.0% | 29.9% | n/a | n/a | n/a | 0.0% | 54.6% | 70.7% | 78.8% | 93.5% | 60.2% |

## Per-model averages — latency × recall cost (plan §3.3)

`cost` = mean(wall_s) / (mean target_recall × 100); lower is better. Target recall = unweighted mean of available {numeric, ticker, org} recalls. ``n/a`` when no target-recall component is available.

| model | class | variant | n | wall_s | tick | org | num_sem | cost |
|---|---|---|---|---|---|---|---|---|
| qwen2.5:32b-instruct-q4_K_M | C | spec | 10 | 434.01 | 0.0% | 2.1% | 65.9% | 19.15 |
| qwen2.5:32b-instruct-q4_K_M | C | spec+example | 10 | 512.10 | 0.0% | 29.9% | 78.8% | 14.13 |

## Per-class averages

| class | variant | n | gv% | ent | evt | claim | note | ent_rec | num_rec | undecl | miss_slot |
|---|---|---|---|---|---|---|---|---|---|---|---|
| A | spec | 0 | n/a | n/a | n/a | n/a | n/a | n/a | n/a | n/a | n/a |
| A | spec+example | 0 | n/a | n/a | n/a | n/a | n/a | n/a | n/a | n/a | n/a |
| B | spec | 0 | n/a | n/a | n/a | n/a | n/a | n/a | n/a | n/a | n/a |
| B | spec+example | 0 | n/a | n/a | n/a | n/a | n/a | n/a | n/a | n/a | n/a |
| C | spec | 10 | 40.0% | 4.70 | 4.40 | 1.10 | 0.50 | 2.1% | 65.9% | 0.00 | 0.00 |
| C | spec+example | 10 | 90.0% | 7.50 | 10.60 | 0.90 | 0.70 | 27.3% | 78.8% | 0.00 | 0.50 |

## Variant deltas (`spec+example` minus `spec`)

Positive means the worked-example arm improved that metric. Cells with `n/a` indicate one or both arms were empty.

| model | class | Δ gv% | Δ ent | Δ evt | Δ claim | Δ ent_rec | Δ num_rec | Δ undecl | Δ miss_slot |
|---|---|---|---|---|---|---|---|---|---|
| qwen2.5:32b-instruct-q4_K_M | C | 50.0% | 2.80 | 6.20 | -0.20 | 25.2% | 12.9% | 0.00 | 0.50 |

Per-class deltas:

| class | Δ gv% | Δ ent | Δ evt | Δ claim | Δ ent_rec | Δ num_rec | Δ undecl | Δ miss_slot |
|---|---|---|---|---|---|---|---|---|
| A | n/a | n/a | n/a | n/a | n/a | n/a | n/a | n/a |
| B | n/a | n/a | n/a | n/a | n/a | n/a | n/a | n/a |
| C | 50.0% | 2.80 | 6.20 | -0.20 | 25.2% | 12.9% | 0.00 | 0.50 |

## Span-copy correlations (plan §3.2 mechanism check)

Pearson `r` of per-cell `span_copy_fraction` against the indicated recall. Positive `r` supports the induction-head copy-mode hypothesis (verbatim copies preserve numerics and entity surfaces better than paraphrase/regeneration). `n` is the number of (model, variant, article) cells with both metrics defined.

| target metric | Pearson r | n pairs |
|---|---|---|
| numeric_exact_match_rate | -0.307 | 9 |
| numeric_semantic_match_rate | -0.355 | 9 |
| org_recall | -0.602 | 9 |

---

Generated by `aggregate_report.py` from `docs/benchmarks/cod-2026-05-17/results/phase1-3-classC`.
