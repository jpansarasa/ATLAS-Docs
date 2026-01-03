# LLM Extraction Benchmark Results

Tracking extraction accuracy for SentinelCollector across models.

## Current Leaderboard (Best F1 per Model)

| Model | F1 | Precision | Recall | Value Acc | Date |
|-------|---:|----------:|-------:|----------:|------|
| qwen2.5:32b-instruct | 0.629 | 0.585 | 0.682 | 0.866 | 2025-12-30 |
| deepseek-r1:32b | 0.570 | 0.459 | 0.752 | 0.847 | 2025-12-29 |
| phi4:14b-q4_K_M | 0.567 | 0.456 | 0.752 | 0.801 | 2025-12-29 |
| qwen3:30b-a3b (MoE) | 0.526 | 0.458 | 0.618 | 0.794 | 2025-12-30 |
| mistral-small:24b | 0.512 | 1.000 | 0.344 | 0.500 | 2025-12-29 |
| gemma3:27b | 0.470 | 0.398 | 0.574 | 0.920 | 2025-12-29 |
| qwen3:32b (dense) | 0.201 | 0.178 | 0.231 | 0.764 | 2026-01-03 |
| qwen3:30b-instruct (MoE) | 0.062 | - | - | - | 2026-01-03 |
| llama3.3:70b-instruct-q2_K | 0.000 | 0.000 | 0.000 | 0.000 | 2025-12-29 |

## Metrics

- **F1**: Harmonic mean of precision and recall (primary metric)
- **Precision**: TP / (TP + FP) - how many extractions are correct
- **Recall**: TP / (TP + FN) - how many expected values were found
- **Value Acc**: Exact match accuracy for extracted numeric values

## Notes

- llama3.3:70b-instruct-q2_K failed (quantization too aggressive?)
- mistral-small:24b has perfect precision but poor recall (too conservative)
- qwen3:30b-a3b is MoE variant (3B active), not dense 32B
- deepseek-r1:32b strong recall but lower precision (over-extracts)
- qwen3:32b (dense) is likely a base model, not instruction-tuned - explains poor results
- qwen3:30b-instruct failed quick benchmark (F1=6.2%), full benchmark skipped
- qwen3 models significantly underperform qwen2.5 on structured extraction

## Pending Tests

- [ ] Fine-tuned variants

## Running Benchmarks

```bash
cd SentinelCollector/tests/SentinelCollector.LlmBenchmark
./run-benchmarks.sh
```

Results saved to `Results/` directory.
