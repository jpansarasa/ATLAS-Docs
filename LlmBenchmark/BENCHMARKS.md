# LLM Extraction Benchmarks

Extraction quality for ATLAS Sentinel, measured on production's own CoD path.
Method and comparability rules: [`MEASUREMENT_SPACE.md`](MEASUREMENT_SPACE.md). Raw coordinates,
controls and sample-level detail: `docs/BACKLOG.md`. Swap criteria: `CLAUDE.md` §MODEL_ACCEPTANCE.

---

## Results

`numbers_f1`, 40 gold articles, production's CoD prompt path, vLLM with `response_format`
json_schema. Higher is better. Coordinate columns are part of the measurement — a score belongs to
a configuration, not to a model.

| Model | numbers_f1 | sd | n | vs production | Engine | Weight quant | Context |
|---|---:|---:|---:|---:|---|---|---:|
| **Gemma 4 31B** | **0.7570** | 0.0009 | 3 | **+0.2417** | vLLM 0.28.0 | compressed-tensors w4a16 QAT | 32,768 |
| **Qwen3.8-27B** | **0.7132** | 0.0109 | 3 | **+0.1979** | vLLM 0.19.0 | compressed-tensors INT4 g32 | 32,768 |
| **Gemma 3 27B** | **0.6177** | 0.0130 | 3 | **+0.1024** | vLLM 0.19.0 | compressed-tensors w4a16 | 32,768 |
| Qwen2.5-32B-AWQ | 0.5153 | 0.0106 | 5 | — *(production)* | vLLM 0.19.0 | AWQ 4-bit g128 | 32,768 |
| Mistral-Small 24B | 0.5105 | 0.0032 | 3 | −0.0048 | vLLM 0.19.0 | AWQ 4-bit | 32,768 |
| Command-R 08-2024 | 0.3178 | 0.0098 | 3 | −0.1975 | vLLM 0.19.0 | AWQ 4-bit | 32,768 |
| EXAONE 4.0 32B | 0.2218 | 0.0089 | 3 | −0.2935 | vLLM 0.19.0 | Q4_K_M | 32,768 |
| GLM-4.7-Flash | *no score* | — | — | — | vLLM 0.19.0 | Q4_K_M | 32,768 |

All three leaders beat production by more than the ~0.06 confound band, so the ranking is real.

**Mistral-Small is the control.** It lands within noise of production despite being re-measured on
the same engine and decoding path as everything else — which is what rules out "these gains are an
artifact of the new harness".

### Serving coordinates required to reproduce these numbers

Not preferences — without them the score above is not what you get.

- **Qwen3.8-27B requires thinking disabled.** Its vendor chat template enables reasoning by default;
  at production's 4,096-token completion budget that leaves the JSON unclosed on **110 of 120
  documents** and the run is unscoreable. The 0.7132 is the thinking-off arm.
- **Gemma 4 31B requires `transformers` ≥ 5.16.1.** The stock vLLM image cannot parse
  `model_type: gemma4`, so the model does not load at all. vLLM 0.19.0 itself cannot serve it;
  0.28.0 is the measured engine.
- **EXAONE 4.0 fails to terminate** inside the 4,096-token budget on 8–9 of 40 articles. Those
  score zero and are included in its 0.2218.

---

## Not viable on this hardware

Recorded as coordinate findings, not scores — the constraint is ours, not the model's.

| Model | Binding constraint |
|---|---|
| llama3.3-70B | 4-bit weights alone are 37–41 GiB on a 31.8 GiB card. Sub-4-bit is banned. No feasible point. |
| Command-R 35B **v01** | No GQA (640 KiB/token) and `max_position_embeddings` 8192 — never a 32K model. Superseded by the 08-2024 build above. |
| GLM-4.7-Flash | Non-terminating unconstrained, empty arrays constrained. A decoding/template interaction. |

---

## Other measurement tracks

**These do not convert to the table above.** Different task, metric or engine.

| Track | What it measures | Qwen3.8 | Gemma 3 | Incumbent |
|---|---|---:|---:|---:|
| Substrate `aggregate_f1` | 16 instruction blocks, not production's task | 0.764 | — | 0.443 |
| GGUF ladder, llama.cpp | chat mode, 8,192/slot, Q4_K_M | 0.6745 | 0.6263 | — |
| GGUF ladder, llama.cpp | Q6_K | 0.6862 | 0.5985 | — |
| GGUF ladder, llama.cpp | Q8_0 | 0.6554 | — | — |

Two findings from the ladder that bear on configuration choices:

- **Precision is model-dependent.** Q6_K beats Q4_K_M on Qwen3.8 by +0.0117 (overlapping ranges — no
  measurable difference) but *loses* on Gemma 3 by −0.0278 (12.2× se, disjoint). A precision result
  does not transfer between models.
- **llama.cpp is a measurement path, not a deployment one.** 2.54× slower than vLLM on the same 40
  articles at the same concurrency. It is used here only because vLLM has no usable rung above
  4-bit at 27B.

---

## Comparability rules, short form

Full statement in [`MEASUREMENT_SPACE.md`](MEASUREMENT_SPACE.md).

1. A score belongs to a **configuration**, not a model. Quote the coordinate with the number.
2. A comparison is valid only if the arms differ in **one axis**, or if it declares itself a
   configuration comparison and attributes the delta to nothing.
3. A cross-family gap **below ~0.06** is not a model result — the chat template alone is worth that,
   and it cannot be held fixed across families.
4. **OOM, TIMEOUT and CRASH are not scores.** They are statements about our configuration.
5. Never infer a quantization scheme from a repo name — two of three checkpoints we serve are named
   for a scheme they do not use.

---

## Archived: 2026-01/04 quick-benchmark leaderboard

<details>
<summary>14 rows, retained as history — not usable for a decision (click to expand)</summary>

Every row mixes engine, quantization and decoding in one F1 column. All were produced with
**salvage-parsing** (grammar-free generation, JSON scraped from free text afterwards) or Ollama's
structured-output parameter — neither is how production extracts today. Half the rows ran on Ollama,
retired 2026-06-11. Seeds were dropped, so no row is reproducible.

Six of the nine families here were re-qualified on 2026-09-06 and appear in the tables above. **Not
one of their eliminations turned out to be a measurement of the model.** Gemma 3, recorded `0.0%
FAIL — too slow`, now beats production.

| Model (as run) | Backend | F1 | census_retail | fed_fomc | Mean Time | Date |
|---|---|---:|---:|---:|---:|---|
| qwen2.5:32b-instruct | llama.cpp | 64.6% | 77% | 46% | 102s | 2026-01-24 |
| qwen3:30b-instruct | llama.cpp | 61.1% | 71% | 42% | 14s | 2026-04-03 |
| qwen3:30b-instruct | Ollama | 61.1% | 71% | 42% | 16s | 2026-01-24 |
| Gemma 4 31B (Q4_K_M) | llama.cpp | 59.7% | 69% | 43% | 193s | 2026-04-03 |
| qwen2.5:32b-instruct | Ollama | 56.7% | 70% | 35% | 40s | 2026-01-24 |
| qwen3:32b | Ollama | 54.8% | 68% | 26% | 89s | 2026-01-24 |
| mistral-small:24b | Ollama | 52.1% | 54% | 48% | - | 2026-01-24 |
| GLM-4.7-Flash (30B MoE) | llama.cpp | *not a score* | 65% | TIMEOUT | 213s | 2026-01-24 |
| phi4:14b-q4_K_M | Ollama | 40.6% | 43% | 37% | 22s | 2026-01-24 |
| deepseek-r1:32b | Ollama | 25.7% | 23% | 31% | - | 2026-01-24 |
| EXAONE 4.0 32B (Q4_K_M) | llama.cpp | *no feasible point* | TIMEOUT | ERROR | FAIL | 2026-04-03 |
| Command-R 35B (Q4_K_M) | llama.cpp | *no feasible point* | TIMEOUT | TIMEOUT | FAIL | 2026-04-03 |
| Gemma 3 27B | llama.cpp | *no feasible point* | TIMEOUT | ERROR | FAIL | 2026-01-24 |
| llama3.3:70b-instruct-q2_K | Ollama | *no feasible point* | - | - | - | 2026-01-24 |

The one comparison in this table that was correctly designed holds the model fixed and varies only
the engine: qwen2.5:32b at 64.6% on llama.cpp vs 56.7% on Ollama — a 7.9pp engine effect. The
magnitude is not reproducible (dropped seeds); the size of the axis is the point.

</details>

---

## Running the benchmarks

**Python track** (`scripts/`) — production's CoD path, any OpenAI-compatible engine, records the
full coordinate in every scorecard and refuses to emit a result it cannot attribute. This produces
the numbers in the tables above.

```bash
python3 LlmBenchmark/scripts/run_model.py --task cod --endpoint-mode completions \
    --prompt-file cod_json_v1.txt --schema-file cod_json_schema_v1.json --chat-template ...
python3 LlmBenchmark/scripts/eval_harness.py --task cod --cod-gold
```

**C# track** — the quick-benchmark screen, drives SentinelCollector's own clients.

```bash
cd LlmBenchmark && ./run-benchmarks.sh                       # llama.cpp arm, 2 entries
./run-benchmarks.sh --filter "Category=LlmBenchmark"          # full run

# vLLM arm — the variable must reach the container, so NOT run-benchmarks.sh
cd ../SentinelCollector/.devcontainer && sudo nerdctl compose up -d
sudo nerdctl compose exec -T -e BENCHMARK_BACKEND=VllmServer \
    -e VLLM_ENDPOINT=http://vllm-server:8000 \
    -e BENCHMARK_MODEL=Qwen/Qwen2.5-32B-Instruct-AWQ \
    sentinel-collector-dev \
    dotnet test /workspace/LlmBenchmark/LlmBenchmark.csproj --filter "Category=QuickBenchmark"
```

`run-benchmarks.sh` takes `--filter` and nothing else. All three `run-*.sh` wrappers are the
llama.cpp arm only and refuse a `BENCHMARK_BACKEND=VllmServer` invocation rather than silently
running llama.cpp under a vLLM banner.

## Hardware

RTX 5090, 32 GB VRAM (31.8 GiB usable) · Threadripper 9960X, 24 cores / 48 threads · 125 GB RAM ·
Context 32K. All benchmark models run fully on GPU. Production CoD/RAG generation also uses CPU
llama.cpp (`llama-cpu-rag`); this file tracks the GPU-served extraction track.
