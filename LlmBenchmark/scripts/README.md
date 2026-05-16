# LlmBenchmark/scripts

Python harness scripts for the Sentinel extraction-LoRA acceptance-criteria pipeline. Build the eval substrate, run a vLLM-served model against it, and score against the pinned acceptance metrics.

## Files

| Script | Purpose |
|---|---|
| `build_eval_substrate.py` | Deterministically builds the evaluation substrate (positives + negatives) for the Sentinel extraction acceptance-criteria harness. Writes the merged substrate plus a sidecar `criteria.json` documenting construction. Substrate itself is ~5 MB and lives under `/opt/ai-inference/training-data/eval-substrates/` (not committed). |
| `eval_harness.py` | Computes the 18 pinned acceptance-criteria metrics against an eval substrate. Two-stage: `score_predictions(gold, pred)` is a pure function (no vLLM / GPU / network — unit-testable); `run_against_model(substrate, vllm-endpoint)` drives the live model and calls the scorer. |
| `test_eval_harness.py` | Unit tests over `score_predictions` and the substrate-construction logic. Fully offline. |

## Typical workflow

```bash
# 1. (Re)build the substrate from upstream training-data inputs.
python LlmBenchmark/scripts/build_eval_substrate.py

# 2. Run a candidate model and grade it.
python LlmBenchmark/scripts/eval_harness.py \
    --substrate /opt/ai-inference/training-data/eval-substrates/<dated>.json \
    --vllm-endpoint http://vllm-server:8000 \
    --model sentinel-extraction-v5

# 3. Unit-test changes to the grader.
python -m pytest LlmBenchmark/scripts/test_eval_harness.py
```

## See Also

- [LlmBenchmark](../) — parent project (C# benchmark runner + `BENCHMARKS.md`)
- [SentinelCollector](../../SentinelCollector/README.md) — owner of the extraction LoRA being evaluated
- [SentinelCollector/scripts](../../SentinelCollector/scripts/README.md) — training-data generation + QLoRA fine-tuning
