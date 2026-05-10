# LLM Extraction Reliability — Acceptance Criteria (D15)

**Status:** RATIFIED  2026-05-10 by James.
Per `atlas-matrix-mvp-plan.md` D15, this document is the source of
truth for the inherited LoRA acceptance thresholds.

This consolidates thresholds that are currently scattered across:

| Source | Location | Current threshold(s) |
|---|---|---|
| ExtractionAccuracyTests | `LlmBenchmark/ExtractionAccuracyTests.cs:18-23` | text_quote_precision ≥ 0.85; text_quote_recall ≥ 0.80; value_accuracy ≥ 0.90; certainty_accuracy ≥ 0.85; period_accuracy ≥ 0.80 |
| BENCHMARKS.md | `LlmBenchmark/BENCHMARKS.md:62-64` | F1 ≥ 0.40 (quick benchmark, 2 cases) AND mean_time < 300s |
| Eval pipeline 2026-05-03 | `/opt/ai-inference/training-data/eval-pipeline-20260503T200815Z-report.md` | symbol_exact_match ≥ 0.90; null_precision ≥ 0.95; JSON-valid = 100% (implied) |
| Eval pipeline 2026-04-26 (baseline) | `/opt/ai-inference/training-data/v7-eval-20260426T103004Z-report.md` | reference baseline (v7 base, pre-iteration) |

## Proposed unified bar (numeric extraction — Feature 4.2 verification + Feature 4.6)

| Metric | Threshold | Source |
|---|---|---|
| `text_quote_precision` | ≥ 0.85 | ExtractionAccuracyTests |
| `text_quote_recall` | ≥ 0.80 | ExtractionAccuracyTests |
| `value_accuracy` | ≥ 0.90 | ExtractionAccuracyTests |
| `certainty_accuracy` | ≥ 0.85 | ExtractionAccuracyTests |
| `period_accuracy` | ≥ 0.80 | ExtractionAccuracyTests |
| `symbol_exact_match` | ≥ 0.90 | eval-pipeline May 2026 |
| `null_precision` (correctly returns empty when no signal) | ≥ 0.95 | eval-pipeline May 2026 |
| `json_valid` | = 1.00 | eval-pipeline May 2026 |
| `selectivity_precision` | ≥ 0.85 | (proposed; today 0.45 on holdout — gap to close) |
| `selectivity_recall` | ≥ 0.80 | (proposed; today 0.82 — meets) |
| Per-document mean latency | < 300s | BENCHMARKS.md |
| Aggregate F1 (numeric outputs) | ≥ 0.40 (initial); ≥ 0.60 (target) | BENCHMARKS.md |

## Proposed bar (qualitative extraction — Feature 4.4)

D15 does not pin qualitative metrics — those are net-new for Feature
4.4. Proposed initial bar (open for ratification):

| Metric | Threshold |
|---|---|
| `sentiment_polarity_accuracy` | ≥ 0.75 |
| `sector_affinity_accuracy` (when sector specified) | ≥ 0.80 |
| `lead_lag_directional_accuracy` (sign correct) | ≥ 0.70 |
| `regime_hint_top1_accuracy` | ≥ 0.65 |
| `cove_consistency_rate` (CoVe verification passes) | ≥ 0.80 |
| `null_precision_qualitative` (don't fabricate signals) | ≥ 0.95 |

The qualitative bars are intentionally lower than numeric — qualitative
extraction is harder, and Feature 4.6 explicitly accepts that the
*trust-attribute gate* on the consumption side (D3) makes lower
extraction confidence usable. Below-threshold qualitative observations
remain queryable; only TE's read adapter filters at the gate.

## Eval substrate decision (open)

Plan claims ~5,500 examples with ~550 negative. Recon found:
- `sentinel-v6.2-cove.json` — 70,895 records (production)
- `sentinel-v4-diverse.json` — ~250K records (historical diverse)
- `sentinel-extraction-v3-validated.json` — ~250K records

**Open question for the user:** which of these is the Feature 4.6 eval
substrate? Recommendation: subset v6.2-cove down to the ~5,500
strongest examples + ensure ≥10% negative coverage, document the
selection criteria, and keep v6.2-cove full for training.

## Iteration budget (D16)

- 4-week timebox of on-prem effort with weekly checkpoints
- $500 Azure cap for any Opus 4.7 relabeling
- Spend tracked at weekly checkpoints in
  `/opt/ai-inference/training-data/azure-oracle-ledger.jsonl`
- Failure mode at exhaustion: architect call on extend / ship constrained
  (high trust-gate threshold) / defer (Feature 4.4 ships with
  reduced scope)

## Ratification

User ratifies by replacing this `Status: DRAFT` line with
`Status: RATIFIED YYYY-MM-DD by James`. Story 4.6.1 reads this file
and pins these values into the LoRA training artifact metadata.
