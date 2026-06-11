# SentinelCollector/scripts

Operator + research scripts for Sentinel's extraction pipeline. Mix of one-shot training-data builders, LoRA fine-tuning drivers, A/B evaluation harnesses, and ad-hoc analysis utilities. **Most of these are research/eval tooling — not production code paths**. Nothing here runs in CI or against production tables without explicit invocation.

## Files — grouped by intent

### Training data generation

| Script | Purpose |
|---|---|
| `generate_training_data.py` | Synthetic training-data generator for early bring-up. |
| `generate_diverse_training.py` | Diverse-bucket training-data generator. |
| `generate_production_training.py` | Sends real production documents to Claude Sonnet for high-quality labeling. Emits instruction/input/output pairs for CoVe + CoD tasks. |
| `generate_training_from_production.py` | Variant that bootstraps training pairs from already-extracted production observations. |
| `export_symbol_training_data.py` | Exports symbol-resolution training pairs from the production database. |
| `sample_training_candidates.py` | Stratified sampling of training candidates by bucket / source / date. |
| `compose_labeling_prompts.py` | v7-style labeling prompts for Opus gold-labeling (Phase 3.2a). |
| `merge_labeled_output.py` | Merges oracle raw output with candidate metadata (Phase 3.2a). |
| `build_v7_train_holdout.py` | Assembles the v7 LoRA training set + 10% holdout (Phase 3.3). |
| `build_v7_audit_input.py` | Builds Phase 3.2c audit-input JSONL. |

### LoRA training drivers

| Script | Purpose |
|---|---|
| `train_qlora.py` | QLoRA fine-tuning driver. Trains a LoRA adapter on top of Qwen2.5:7b for economic-data extraction. 4-bit quantization for consumer GPU memory budgets. |
| `train_qlora_unsloth.py` | Unsloth-backed variant of `train_qlora.py` — faster wall-clock for the same training data. |

### Evaluation + A/B

| Script | Purpose |
|---|---|
| `evaluate_lora_v7.py` | Evaluates the v7 LoRA adapter (`sentinel-cove-v7`) against the pinned acceptance criteria. |
| `evaluate_pipeline_v2.py` | Sentinel v2 full-pipeline evaluator. |
| `experiment_a_lora_extraction.py` | Experiment A — LoRA-only A/B between v6.2 and v7-e3. |
| `experiment_b_pipeline_resolution.py` | Experiment B — pipeline-only A/B; feed perfect Opus labels in. |
| `experiment_c_end_to_end.py` | Experiment C — full end-to-end on the same docs as Exp B but with v7-e3 LoRA. |
| `dryrun_v7_prompt.py` | v7 prompt dry-run for Phase 1.1 acceptance measurement. |
| `test_acceptance_criteria.py` | Smoke checks for the pinned LLM extraction acceptance criteria. |
| `shadow_diff_report.py` | Sentinel v2 shadow-mode parity + lift report. |

### Audit / triage / cross-check

| Script | Purpose |
|---|---|
| `analyze_audit.py` | Phase 3.2c diversity-audit analyzer. |
| `analyze_v7_audit.py` | Phase 3.2c audit analysis (v7-specific). |
| `analyze_crosscheck_diff.py` | Phase 3.2b cross-check diff analyzer. |
| `v7_crosscheck_sample.py` | 10% bucket-stratified sample of Phase 3.2a gold labels. |
| `v7_crosscheck_diff.py` | Diff Opus 4.7 gold labels vs Opus 4.6 cross-check labels. |
| `v7_triage_build.py` | Stratified-50 disagreement triage prompts (Phase 3.2b). |
| `v7_triage_report.py` | Aggregates triage-50 verdicts into a Markdown report + recommendation. |

### SecMaster bring-up + reprocessing

| Script | Purpose |
|---|---|
| `load_secmaster_curated.py` | Loads the curated index / ETF / commodity / crypto / equity universe into SecMaster. |
| `load_secmaster_fred.py` | Loads the FRED catalog (top ~5000 series by popularity) into SecMaster. |
| `mass_reprocess.py` | Bulk re-extraction driver for already-collected raw content. |
| `scope_nextdata_reprocess.py` | Scopes `raw_content` rows whose on-disk HTML contains a Next.js `__NEXT_DATA__` SSR fragment (targets a known extraction blind spot). |

### External oracle client

| Script | Purpose |
|---|---|
| `azure_oracle_client.py` | Azure Foundry Claude oracle client (Sentinel v2 Phase 0.4). Cheaper per-call than direct Anthropic API for bulk labeling — see `feedback_azure_foundry_vs_anthropic_cost` memory note. |

## When to run these

These scripts have specific phases attached to them (`v7_*`, `Phase 3.2*`, etc.). If the phase isn't currently active, the script is historical reference — don't re-run blindly. The plans were retired to git history at phase completion (recovery via `docs/RELEASES.md`) — treat unreferenced scripts as historical.

## See Also

- [SentinelCollector](../README.md) — service README
- [LlmBenchmark/scripts](../../LlmBenchmark/scripts/README.md) — eval-substrate harness used by the LoRA acceptance pipeline
