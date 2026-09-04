# LLM Extraction Benchmark Results

Tracking LLM extraction accuracy across models for ATLAS sentinel extraction.

> ## READ THIS BEFORE CITING A NUMBER BELOW
>
> **These figures are stale, and two of them describe software this box no longer runs.**
> Measured 2026-09-03:
>
> | | |
> |---|---|
> | llama.cpp this box **runs** | build **10603** (image built 2026-08-24) |
> | Newest llama.cpp figure in this table | **2026-04-03** |
> | Engines compared in "Backend Comparison" below | llama.cpp vs **Ollama** — retired 2026-06-11 |
> | vLLM (the engine that serves **production** extraction) | never scored in this table |
>
> Five months of llama.cpp development sit between the best number here (64.6% F1) and the
> binary actually installed, so "llama.cpp scores 64.6%" is not a statement about our
> llama.cpp. Nothing in a result file recorded an engine build, which is why none of that
> was visible from the results themselves.
>
> **This table cannot answer "which engine should we use".** It compares one engine we still
> run against one we deleted, on data from before either moved. For a current, engine-attributed
> number use the Python track — [`scripts/`](scripts/README.md) — whose `run_model.py` drives
> any OpenAI-compatible engine (vLLM, SGLang, llama.cpp `/v1`), records the engine build in
> every scorecard, and **refuses to emit a result it cannot attribute**. `check_staleness.py`
> compares a scorecard's recorded build against what is live now.
>
> The C# harness below can now drive vLLM as well: it builds SentinelCollector's own
> `VllmClient`/`LlamaServerClient` from a real `ExtractionOptions`. **Every row in this table
> predates that**, and no row was produced by a client this repo still contains. The llama.cpp
> rows came from the replaced `BenchmarkLlamaServerClient`, which posted to `/completion` for
> plain generation and to `/v1/chat/completions` for structured output — hardcoding
> `model: "default"` and `max_tokens: -1` on that second endpoint — and dropped the seed on
> both, so none of those numbers is seed-reproducible or on production's completion budget.
> The seven `Ollama` rows — half of the fourteen-row table, the other seven being llama.cpp —
> are older still: Ollama was retired 2026-06-11 and no engine remains to reproduce them on.
> Read this leaderboard as history, not as a comparison.

## Current Leaderboard (Quick Benchmark - 2 Test Cases)

| Model | Backend | F1 | census_retail | fed_fomc | Mean Time | Date |
|-------|---------|---:|--------------:|---------:|----------:|------|
| qwen2.5:32b-instruct | llama.cpp | **64.6%** | 77% | 46% | 102s | 2026-01-24 |
| qwen3:30b-instruct | llama.cpp | 61.1% | 71% | 42% | 14s | 2026-04-03 |
| qwen3:30b-instruct | Ollama | 61.1% | 71% | 42% | 16s | 2026-01-24 |
| Gemma 4 31B (Q4_K_M) | llama.cpp | 59.7% | 69% | 43% | 193s | 2026-04-03 |
| qwen2.5:32b-instruct | Ollama | 56.7% | 70% | 35% | 40s | 2026-01-24 |
| qwen3:32b | Ollama | 54.8% | 68% | 26% | 89s | 2026-01-24 |
| mistral-small:24b | Ollama | 52.1% | 54% | 48% | - | 2026-01-24 |
| GLM-4.7-Flash (30B MoE) | llama.cpp | 48.0% | 65% | TIMEOUT | 213s | 2026-01-24 |
| phi4:14b-q4_K_M | Ollama | 40.6% | 43% | 37% | 22s | 2026-01-24 |
| deepseek-r1:32b | Ollama | 25.7% | 23% | 31% | - | 2026-01-24 |
| EXAONE 4.0 32B (Q4_K_M) | llama.cpp | 0.0% | TIMEOUT | ERROR | FAIL | 2026-04-03 |
| Command-R 35B (Q4_K_M) | llama.cpp | 0.0% | TIMEOUT | TIMEOUT | FAIL | 2026-04-03 |
| Gemma 3 27B | llama.cpp | 0.0% | TIMEOUT | ERROR | FAIL | 2026-01-24 |
| llama3.3:70b-instruct-q2_K | Ollama | CRASH | - | - | - | 2026-01-24 |

## Key Findings

### Backend Comparison (Same Model)

- **llama.cpp outperforms Ollama** with Qwen 2.5 32B (64.6% vs 56.7% F1)
- **qwen3:30b-instruct identical on both backends** (61.1% F1, 14s vs 16s) - no llama.cpp advantage
- llama.cpp uses grammar-free generation with JSON extraction fallback
- Ollama uses structured output format parameter

### Model Family Performance

- **Qwen family dominates** - 4 of top 5 models are Qwen variants
- **qwen3:30b-instruct** is fastest (16s mean) with strong F1 (61.1%)
- **qwen3:32b** (dense) underperforms instruct variants
- **deepseek-r1:32b** reasoning model performs poorly on extraction (25.7%)

### Gemma 4 vs Gemma 3

- **Gemma 4 31B**: Massive improvement over Gemma 3 27B (59.7% F1 vs 0% - timeouts/errors)
- 100% recall on census_retail (finds all expected values) but lower precision (53%)
- Slower than Qwen models (193s mean vs 102s for qwen2.5 on llama.cpp) - near the 300s timeout on census_retail (291s)
- Ran on llama.cpp with Q4_K_M quantization (18GB GGUF, 22.4GB VRAM with 32K context)

### Failed Models

- **EXAONE 4.0 32B**: census_retail timeout, fed_fomc extraction errors after 4 retries. Model produces output but can't follow extraction format.
- **Command-R 35B**: OOM at 32K context (35B model too large for KV cache). At 8K context both entries timeout.
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

`run-benchmarks.sh` takes `--filter` and nothing else — any other flag exits 1. It documented
`--backend llamacpp` and `--model` for months after both were removed, on a backend (Ollama)
retired 2026-06-11, and a `--debug` shortcut that set `--filter Category=Debug` against a trait
no test carries. The model is whatever the server has loaded, because both engines pin theirs at
server start.

**All three `run-*.sh` wrappers are the llama.cpp arm and only the llama.cpp arm.** They
health-check `llama-server` and pin `BENCHMARK_BACKEND=LlamaServer` for the run, forwarding no
host variable — so `BENCHMARK_BACKEND=VllmServer ./run-benchmarks.sh` used to run llama.cpp and
file the number under a vLLM banner. All three now refuse that invocation and print the vLLM
form below; the refusal is case-insensitive, matching `ParseEnumOrThrow`, so `llamaserver` names
this arm and is accepted rather than refused with a consequence that cannot happen to it.

```bash
cd LlmBenchmark

# QuickBenchmark screen against the running llama.cpp server (2 entries)
./run-benchmarks.sh

# Full extraction run
./run-benchmarks.sh --filter "Category=LlmBenchmark"

# The vLLM arm: same harness, production's default backend. NOT run-benchmarks.sh --
# the variable has to reach the container the tests run in. (LlmBenchmark/README.md, "vLLM run")
cd ../SentinelCollector/.devcontainer
sudo nerdctl compose up -d
sudo nerdctl compose exec -T \
    -e BENCHMARK_BACKEND=VllmServer \
    -e VLLM_ENDPOINT=http://vllm-server:8000 \
    -e BENCHMARK_MODEL=Qwen/Qwen2.5-32B-Instruct-AWQ \
    sentinel-collector-dev \
    dotnet test /workspace/LlmBenchmark/LlmBenchmark.csproj --filter "Category=QuickBenchmark"
```

An unrecognised `BENCHMARK_BACKEND` is now REFUSED rather than defaulted away — `llamacpp`,
`llama` and `LlamaCpp` all used to fall through to vLLM silently and file the numbers under a
llama.cpp banner.

To compare models, restart the server with each candidate and re-run. Results are displayed in
the test output; `run-all-benchmarks.sh` and `run-top5-benchmark.sh` write timestamped logs
beside themselves.

## Hardware

- GPU: NVIDIA RTX 5090 (32GB VRAM)
- CPU: Threadripper-class, 128GB DDR5 RAM (used for CoD / RAG / parallel small-model fan-out)
- All models in this benchmark ran fully on GPU (no CPU offload). Note: ATLAS production CoD/RAG generation runs on CPU via llama.cpp (`llama-cpu-rag`, which replaced the retired `ollama-cpu-gen` on 2026-06-11); this benchmark file specifically tracks the GPU-served extraction track.
- Context size: 32K tokens
