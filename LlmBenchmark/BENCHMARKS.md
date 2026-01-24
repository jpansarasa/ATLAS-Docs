# LLM Extraction Benchmark Results

Tracking LLM extraction accuracy across models for ATLAS sentinel extraction.

## Current Leaderboard (Quick Benchmark - 2 Test Cases)

| Model | Backend | F1 | census_retail | fed_fomc | Mean Time | Date |
|-------|---------|---:|--------------:|---------:|----------:|------|
| qwen2.5:32b-instruct | llama.cpp | **64.6%** | 77% | 46% | 102s | 2026-01-24 |
| qwen3:30b-instruct | Ollama | 61.1% | 71% | 42% | 16s | 2026-01-24 |
| qwen2.5:32b-instruct | Ollama | 56.7% | 70% | 35% | 40s | 2026-01-24 |
| qwen3:32b | Ollama | 54.8% | 68% | 26% | 89s | 2026-01-24 |
| mistral-small:24b | Ollama | 52.1% | 54% | 48% | - | 2026-01-24 |
| GLM-4.7-Flash (30B MoE) | llama.cpp | 48.0% | 65% | TIMEOUT | 213s | 2026-01-24 |
| phi4:14b-q4_K_M | Ollama | 40.6% | 43% | 37% | 22s | 2026-01-24 |
| deepseek-r1:32b | Ollama | 25.7% | 23% | 31% | - | 2026-01-24 |
| Gemma 3 27B | llama.cpp | 0.0% | TIMEOUT | ERROR | FAIL | 2026-01-24 |
| llama3.3:70b-instruct-q2_K | Ollama | CRASH | - | - | - | 2026-01-24 |

## Key Findings

### Backend Comparison (Same Model)

- **llama.cpp outperforms Ollama** with identical Qwen 2.5 32B model (64.6% vs 56.7% F1)
- llama.cpp uses grammar-free generation with JSON extraction fallback
- Ollama uses structured output format parameter

### Model Family Performance

- **Qwen family dominates** - 4 of top 5 models are Qwen variants
- **qwen3:30b-instruct** is fastest (16s mean) with strong F1 (61.1%)
- **qwen3:32b** (dense) underperforms instruct variants
- **deepseek-r1:32b** reasoning model performs poorly on extraction (25.7%)

### Failed Models

- **Gemma 3 27B**: Too slow, times out on both test cases
- **GLM-4.7-Flash**: MoE architecture causes timeouts despite 30B parameter count
- **llama3.3:70b-instruct-q2_K**: Crashes (insufficient VRAM at q2_K quantization)

## Metrics

- **F1**: Harmonic mean of precision and recall (primary metric)
- **census_retail**: Census Bureau retail sales extraction (17 expected values)
- **fed_fomc**: Federal Reserve FOMC statement extraction (13 expected values)
- **Mean Time**: Average seconds per test case (300s timeout)

## Quick Benchmark Threshold

Pass criteria: F1 >= 40% AND mean time < 300s per entry

## Running Benchmarks

```bash
cd LlmBenchmark

# Ollama backend (default)
./run-benchmarks.sh

# Ollama with specific model
./run-benchmarks.sh --model "qwen3:30b-instruct"

# llama.cpp backend (requires llama-server running)
./run-benchmarks.sh --backend llamacpp
```

Results displayed in test output. Full benchmark logs in `/tmp/benchmark_*.txt`.

## Hardware

- GPU: NVIDIA RTX 4090 (24GB VRAM)
- All models run fully on GPU (no CPU offload)
- Context size: 32K tokens
