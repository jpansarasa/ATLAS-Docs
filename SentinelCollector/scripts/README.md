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
| `build_golden_corpus.py` | **The one script here whose OUTPUT is committed and read by the xUnit suite** (the suite never invokes it). Builds `tests/SentinelCollector.UnitTests/Fixtures/GoldenCorpus/` from `corpus.spec.json` — raw file plus every landed observation per article. Rebuild after the identity work moves a baseline. `--check` writes nothing and returns TWO named verdicts: **CORPUS DRIFT** (the committed corpus disagrees with itself — hermetic, no database, **exits 1**) and **PRODUCTION MOVED** (the live rows a fixture recorded have since changed — advisory, **exits 0**). Production rows mutate retroactively as the resolver re-resolves, and raw files prune at 30 days, so the second verdict goes dirty on its own while the fixture stays valid; never rebuild to quiet it. `--write-manifest` refreshes MANIFEST.md from the committed fixtures alone. |

### Resolution regression (live catalog, not hermetic)

| Script | Purpose |
|---|---|
| `resolution-regression/` | `run.sh` resolves `corpus.tsv` (real `(subject, description)` pairs lifted verbatim from production observations, positives AND refusals) against the RUNNING SecMaster via `/api/semantic/resolve-local` and asserts each lands where it should. Default mode is a **catalog probe: the Rule 2b query at SecMaster's default floor with no context** (it answers whether the catalog still holds the instrument near the row's words, not what production does); `--live` is the production mirror, `q=subject&context=description` at the `Extraction__V2VectorMinScore` read from `deployment/artifacts/compose.yaml.j2` at run time -- the Rule 2 leg `SecMasterClient.ResolveLocalFromQuoteAsync` sends. rc 0 needs every row as expected, at least one row resolving a SYMBOL (seven negative rows mean a dead catalog answers NONE everywhere, so passes are not the signal) and every positive resolved; any `PARSE_ERROR` (non-200, non-JSON, no curl) is rc 2. The corpus carries ONE EXPECTATION PER MODE (`expected_probe`, `expected_live`), each the value a healthy catalog returns today, so both modes exit 0 on a healthy catalog: row 1 is NONE under `--live` by construction (the `NameAppearsInContext` gate needs the catalog name verbatim in the context), rows 2-4 resolve under the probe (recorded so the D-13 AFTER run measures it, not asserted correct), and a Federal Funds Effective Rate row is the live-mode positive that keeps the nothing-resolved and all-positives clauses falsifiable. `--selftest` appends a known-bad control (the first row positive in the current mode's column, that expectation rewritten to NONE) to the real corpus and runs the full harness: rc must be 1, the control's FAIL line must carry the corpus symbol as the OBSERVED value (`-> CHALLENGER_JOB_CUTS (expected NONE)` in probe mode, `-> DFF (expected NONE)` live), the summary must count it, else rc 2. `neighbourhood.sh` emits the unfloored top-5 `/api/semantic/search` neighbourhood per row as TSV with a `retired` flag (`(DISCONTINUED)` in the name only -- the endpoint carries no source-mapping indicator, so there is no `unmapped` column) and a summary of the retired share, with a stdout `# WARNING` when any query failed; its `--selftest` pushes a synthetic 1-of-2 retired body through the same derivation AND the same summary function the measurement prints (rc 2 if it does not count 1), and a BEFORE-only arm reports whether a named corpus row still scores a retired slot live (skips with a reason once #1049 has emptied the vector population of retired rows). The acceptance run for SecMaster D-13 (SecMaster/AGENT_README.md D-13) is `run.sh --live` plus `neighbourhood.sh` before and after the deploy, diffed. |

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
