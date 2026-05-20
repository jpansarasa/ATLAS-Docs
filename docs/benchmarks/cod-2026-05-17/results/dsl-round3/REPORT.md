# Round 3 CoD DSL benchmark — aggregate report

Total result files: **140**
Models seen: **7**, variants: **2**

Metrics columns: `n`, `gv%` (grammar-valid rate), `ent` (mean ENT count), `evt`, `claim`, `note`, `ent_rec` (ent recall, mean), `num_rec` (numeric recall, mean), `undecl` (mean undeclared ENT refs), `miss_slot` (mean missing required slots).

## Per-model averages

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

---

## Round 2 (rescore) vs Round 3 (fresh) — qwen3:30b headline comparison

Baselines: Round 1 prose for `qwen3:30b-a3b-instruct-2507-q4_K_M` reported
`ent_recall = 25%` (single-pass extraction). The hypothesis under test is
whether the structured CoD DSL — with the widened parser from PRs
#375 / #376 — clears that bar end-to-end on a fresh inference run.

| metric (qwen3:30b) | R1 prose | R2 rescore (widened parser) | R3 fresh (widened parser, TIMEOUT=1800) | R3 − R2 | R3 − prose |
|---|---|---|---|---|---|
| spec `gv%`         | n/a      | 90.0%  | 100.0% | +10.0pp | n/a |
| spec `ent_recall`  | 25.0%    | 31.7%  | 29.4%  | -2.3pp  | +4.4pp |
| spec `num_recall`  | n/a      | 91.7%  | 88.6%  | -3.1pp  | n/a |
| spec+ex `gv%`      | n/a      | 80.0%  | 90.0%  | +10.0pp | n/a |
| spec+ex `ent_recall` | n/a    | 11.2%  | 8.8%   | -2.4pp  | n/a |
| spec+ex `num_recall` | n/a    | 89.0%  | 90.8%  | +1.8pp  | n/a |

### Resolution of the 47K-char outlier (article 27772)

The previously-timed-out earnings transcript (47K chars, Jenoptik Q4 2025):

- `qwen3:30b` **spec/27772**: completed in **1095s** wall, `grammar_valid=true`,
  `ent_recall=0.31`, `num_recall=0.73` — **rescued by the 1800s budget.**
- `qwen3:30b` **spec+example/27772**: **still timed out** at 1800s. The
  worked-example arm expands the prompt enough that this single cell
  exceeds the new ceiling; this contributes to the spec+ex `ent_recall`
  drop (timeouts score as `null`).

### Run-failure inventory (10 ReadTimeouts at 1800s, 0 other failures)

- `gemma4:26b-a4b-it-q4_K_M`: 7 (3 spec, 4 spec+example) — model is
  pathological on long inputs at this quantization.
- `qwen3:30b` spec+example/27772 — as above.
- `qwen2.5:7b-instruct` spec/36065 — single edge case.
- `qwen3:8b` spec/36065 — single edge case.

Timeouts are scored as `grammar_valid=false` with null recall, so they
*do* drag the per-model averages above, but legitimately so.

## Hypothesis verdict — **TRENDING**

Decision rule (per the user-authorized sequence):
- **CONFIRMED**: best-model DSL ent_recall ≥ +5pp over prose, repeatable.
- **TRENDING**: above prose baseline but within run-to-run noise.
- **NEEDS-MORE**: at or below prose baseline.

Result:
- qwen3:30b spec arm clears prose by **+4.4pp** (29.4% vs 25%) — just
  below the +5pp threshold for CONFIRMED.
- Run-to-run drift R2→R3 is **-2.3pp** on the same parser at n=10,
  which is larger than the headline gap over prose. We cannot
  distinguish "real improvement" from noise without a third run.
- Worked-example arm regression vs spec (-20.6pp) is now **confirmed
  across two independent runs**. This is no longer a fluke — the
  current worked example actively harms recall.

### Recommended next step — **prompt-tune the worked example**

Both data points (R2 rescore -20.4pp, R3 fresh -20.6pp) say the same
thing: the current `spec+example` worked example is degrading recall on
the best model. Before either shipping the widened DSL or rethinking
the approach, the cheapest informative experiment is to revise the
worked example (shorter, simpler ENT block, fewer EVT slots) and re-run
just the `spec+example` arm. If a revised example pulls qwen3:30b
ent_recall above 31.7%, the DSL stack is doing real work and we ship.
If it stays under 15%, the DSL itself is the bottleneck and we rethink.

Threshold tripped for this recommendation: **repeatable spec+example
regression** (-20pp vs spec on both runs) + **spec arm gap over prose
under +5pp threshold** for CONFIRMED.

---

Generated by `aggregate_report.py` from `results/dsl-round3`.
