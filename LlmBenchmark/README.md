# LlmBenchmark

C# xUnit harness that measures Sentinel LLM extraction quality (CoVe / CoD / epistemic markers) against a golden dataset, across configurable models and backends (Ollama, llama.cpp).

## Overview

LlmBenchmark is the GPU-extraction track's accuracy gate for ATLAS Sentinel. It exercises the exact production code paths in `SentinelCollector` (`ChainOfVerification`, `ChainOfDensity`, the same `src/prompts` directory) against a pinned golden dataset and emits per-entry + aggregate scores (precision / recall / F1, timing, epistemic-marker recall). It is **not** a service: no container, no ports, no deployment — it runs inside the `SentinelCollector` devcontainer via `dotnet test`. The current leaderboard lives in `BENCHMARKS.md`; the Python LoRA-acceptance harness for vLLM-served Sentinel models lives in `scripts/` (separately documented).

## Architecture

```mermaid
flowchart LR
    subgraph Inputs
        GD[golden_dataset.json<br/>+ raw_content/]
        PR[SentinelCollector/src/prompts]
    end

    subgraph LlmBenchmark
        FX[OllamaFixture<br/>backend selector]
        BM[Benchmarks/<br/>CoVe + CoD + Epistemic tests]
        SC[Metrics/<br/>scorers + report]
    end

    subgraph Backends
        OL[Ollama]
        LC[llama.cpp server]
    end

    subgraph Outputs
        RJ[Results/benchmark_results_*.json]
        LD[BENCHMARKS.md<br/>leaderboard]
    end

    GD --> BM
    PR --> BM
    FX --> OL
    FX --> LC
    BM --> SC
    SC --> RJ
    RJ -.curated.-> LD
```

`OllamaFixture` health-checks the chosen backend, loads/unloads models (Ollama only; llama.cpp pins a single model at server start), and constructs `ChainOfVerification` / `ChainOfDensity` against the production prompt directory. Each test runs all golden entries, scores them, and writes a per-run JSON report.

## Features

- **Three benchmark tracks**: CoVe extraction (`ExtractionAccuracyTests`), CoD summarization (`CoDAccuracyTests`), epistemic-marker JSON extraction (`EpistemicMarkerTests`)
- **Quick-screen mode**: 2-entry / 5-min-per-entry / F1 ≥ 40% gate before committing to a full run
- **Dual backend**: Ollama (per-test model load/unload) or llama.cpp server (fixed model, no swapping)
- **Production prompts**: reads from `SentinelCollector/src/prompts` — no test-local copies to drift
- **Per-entry JSON reports**: written to `Results/` with full score breakdown + timing stats
- **Convenience scripts**: backend selection, sequential multi-model runs, top-5 with VRAM recycling, tokens/sec validator

## Test Categories

xUnit traits used to filter runs via `--filter`:

| Trait | Value | Tests included |
|---|---|---|
| `Category` | `LlmBenchmark` | All extraction + CoD tests (full run) |
| `Category` | `QuickBenchmark` | `QuickBenchmark_ScreenModel` only |
| `Category` | `EpistemicBenchmark` | `EpistemicMarkerTests` only |
| `Strategy` | `CoVe` | `ExtractionAccuracyTests` |
| `Strategy` | `CoD` | `CoDAccuracyTests` |

## Configuration

Driven by environment variables read by `BenchmarkConfiguration.Default`:

| Variable | Description | Default |
|----------|-------------|---------|
| `BENCHMARK_BACKEND` | `ollama` or `llamacpp` (also `llama.cpp`, `llama-cpp`) | `ollama` |
| `BENCHMARK_MODEL` | Ollama model name to test (single-model run) | `qwen2.5:32b-instruct` |
| `OLLAMA_ENDPOINT` | Ollama base URL | `http://localhost:11434` |
| `LLAMA_SERVER_ENDPOINT` | llama.cpp server base URL | `http://localhost:8080` |
| `LLAMA_CHAT_TEMPLATE` | Chat template for llama.cpp completions | Qwen2.5 ChatML |
| `LLAMA_STOP_TOKENS` | Comma-separated stop tokens for llama.cpp | `<\|im_end\|>,<\|endoftext\|>` |
| `OLLAMA_USE_SCHEMA` | `true` enables Ollama `format` schema enforcement (fine-tuned models only) | `false` |

### Thresholds (hard-coded in test classes)

| Track | Metric | Threshold |
|---|---|---|
| Full CoVe (`FullPipeline_MeetsAccuracyThreshold`) | Overall precision | ≥ 85% |
| Full CoVe | Overall recall | ≥ 80% |
| Full CoVe | Avg value-exact-match | ≥ 90% |
| Quick (`QuickBenchmark_ScreenModel`) | F1 | ≥ 40% |
| Quick | Mean seconds per entry | ≤ 300s |
| CoD (`CoD_MeetsSummaryQualityThreshold`) | Entity coverage | ≥ 80% |
| CoD | Value coverage | ≥ 75% |
| CoD | Overall CoD score | ≥ 75% |
| Epistemic | JSON parse success rate | ≥ 90% |

Per-entry HTTP timeout: 10 min (CoVe full run, `OllamaFixture.PerEntryTimeout`). Model load timeout: 5 min.

## Project Structure

```
LlmBenchmark/
├── Benchmarks/                 # xUnit test classes
│   ├── ExtractionAccuracyTests.cs   # CoVe: full + quick + debug
│   ├── CoDAccuracyTests.cs          # CoD summarization tests
│   ├── EpistemicMarkerTests.cs      # JSON marker-parse tests
│   └── OllamaFixture.cs             # Backend health-check + client factory
├── Infrastructure/             # Config + client adapters
│   ├── BenchmarkConfiguration.cs    # Env-var driven config
│   ├── BenchmarkLlamaServerClient.cs
│   ├── BenchmarkLlmClientAdapter.cs
│   ├── TestPromptProvider.cs
│   └── TestDensityPromptProvider.cs
├── GoldenDataset/              # Dataset loader + entry record
├── Metrics/                    # Scorers + BenchmarkReport JSON writer
│   ├── ExtractionAccuracyScorer.cs
│   ├── CoDAccuracyScorer.cs
│   ├── TextQuoteMatcher.cs
│   └── BenchmarkReport.cs
├── Results/                    # Historical benchmark JSON outputs (committed)
├── TestData/                   # golden_dataset.json + raw_content/ (copied to build output)
├── eval-substrate/             # vLLM-acceptance criteria + scorecard sidecars
├── scripts/                    # Python harness for vLLM LoRA acceptance (see scripts/README.md)
├── run-benchmarks.sh           # Single run with backend / model / filter flags
├── run-all-benchmarks.sh       # Sequential overnight run across 5 models (in-container)
├── run-top5-benchmark.sh       # QuickBenchmark with container recycling for clean VRAM
├── validate-models.sh          # Tokens/sec smoke test via /api/generate (pass: ≥ 20 tok/s)
├── BENCHMARKS.md               # Current leaderboard + key findings
└── LlmBenchmark.csproj         # net10.0 xUnit test project; refs SentinelCollector
```

## Running Benchmarks

The harness builds against `SentinelCollector`, so it runs inside the `SentinelCollector` devcontainer. The `run-benchmarks.sh` wrapper handles `nerdctl compose up/down` and env passthrough.

### Quick benchmark, Ollama default (qwen2.5:32b-instruct)

```bash
cd LlmBenchmark
./run-benchmarks.sh
```

### Specific Ollama model

```bash
./run-benchmarks.sh --model "qwen3:30b-a3b"
```

### llama.cpp backend (model pinned at server start)

```bash
./run-benchmarks.sh --backend llamacpp
```

### Alternate Ollama endpoint (e.g. CPU instance)

```bash
./run-benchmarks.sh --endpoint http://ollama-cpu:11434 --model "qwen2.5:7b-instruct"
```

### Custom filter (full extraction run)

```bash
./run-benchmarks.sh --filter "Category=LlmBenchmark"
```

### Debug shortcut

```bash
./run-benchmarks.sh --debug          # equivalent to --filter "Category=Debug"
```

### Other scripts

| Script | Use |
|---|---|
| `./run-all-benchmarks.sh` | Sequential full-extraction run across 5 hard-coded models, writes timestamped log. Must already be inside the devcontainer (`/workspace` paths). |
| `./run-top5-benchmark.sh` | QuickBenchmark across top 5 models with `docker restart ollama-gpu` between each for clean VRAM. Writes per-model logs under `results_<ts>/`. |
| `./validate-models.sh [results.json]` | Sends a representative extraction prompt to each model via `/api/generate`, reports tokens/sec, pass gate = 20 tok/s. |

### Building / compiling separately

The benchmark is a normal C# test project — same `compile.sh` flow as any ATLAS service:

```bash
SentinelCollector/.devcontainer/compile.sh           # build + tests (no benchmarks run unless filtered in)
SentinelCollector/.devcontainer/compile.sh --no-test # build only
```

## Outputs

- **xUnit console output**: per-entry precision/recall/F1, timing, aggregate, pass/fail vs threshold.
- **JSON reports**: `Results/benchmark_results_<model>_<UTC-timestamp>.json` (full CoVe runs only) — schema in `Metrics/BenchmarkReport.cs` (`BenchmarkRun` record).
- **Leaderboard**: hand-curated into `BENCHMARKS.md` after notable runs.
- **Convenience-script logs**: `benchmark_run_<ts>.log`, `benchmark_top5_<ts>.log`, `results_<ts>/<model>.log`.

## See Also

- [BENCHMARKS.md](./BENCHMARKS.md) — current leaderboard, key findings, hardware notes
- [scripts/README.md](./scripts/README.md) — Python LoRA-acceptance harness (vLLM-served Sentinel models)
- [SentinelCollector](../SentinelCollector/README.md) — owner of `ChainOfVerification`, `ChainOfDensity`, and the prompt directory under test
- [SentinelCollector/scripts](../SentinelCollector/scripts/README.md) — training-data generation and QLoRA fine-tuning
