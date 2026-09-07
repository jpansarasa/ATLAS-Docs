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

> **This is not a leaderboard. No row below is a model score you can rank.** The table tangles
> engine, model, quantization, decoding method and context in one F1 column — see
> [`MEASUREMENT_SPACE.md`](MEASUREMENT_SPACE.md) for the full rule. Three rows record a serving
> failure, not an extraction result, and are labelled COORDINATE FINDING rather than scored. The
> table below now carries a quantization and decoding column for every row so the confound is
> visible in the row, not only in this banner, and the "Key Findings" prose that used to draw
> family-level conclusions from these rows (including "Qwen family dominates") has been removed
> for the same reason it is banned in MEASUREMENT_SPACE.md: this table cannot show it, true or
> not. The one comparison that survives — same model, only the engine varies — is kept below,
> with its own caveat. Real re-qualification numbers on production's current engine and prompt
> path are in "2026-09-06 re-qualification outcomes" further down, sourced from `docs/BACKLOG.md`.

## Historical Leaderboard (Quick Benchmark — 2 Test Cases) — HISTORY, not a ranking

Kept as a record of what was run and when. Every row mixes at least engine, quantization (where
even named) and decoding method — do not compare across rows without reading the footnotes.

| Model (as run) | Backend | Weight quant (as named)† | Decoding‡ | F1 | census_retail | fed_fomc | Mean Time | Date |
|---|---|---|---|---:|---:|---:|---:|---|
| qwen2.5:32b-instruct | llama.cpp | not stated | salvage-parse | 64.6% | 77% | 46% | 102s | 2026-01-24 |
| qwen3:30b-instruct | llama.cpp | not stated | salvage-parse | 61.1% | 71% | 42% | 14s | 2026-04-03 |
| qwen3:30b-instruct | Ollama | not stated | Ollama structured-output | 61.1% | 71% | 42% | 16s | 2026-01-24 |
| Gemma 4 31B (Q4_K_M) | llama.cpp | Q4_K_M | salvage-parse | 59.7% | 69% | 43% | 193s | 2026-04-03 |
| qwen2.5:32b-instruct | Ollama | not stated | Ollama structured-output | 56.7% | 70% | 35% | 40s | 2026-01-24 |
| qwen3:32b | Ollama | not stated | Ollama structured-output | 54.8% | 68% | 26% | 89s | 2026-01-24 |
| mistral-small:24b | Ollama | not stated | Ollama structured-output | 52.1% | 54% | 48% | - | 2026-01-24 |
| GLM-4.7-Flash (30B MoE) | llama.cpp | not stated | salvage-parse | 48.0% (composite, not a score)§ | 65% | TIMEOUT | 213s | 2026-01-24 |
| phi4:14b-q4_K_M | Ollama | Q4_K_M | Ollama structured-output | 40.6% | 43% | 37% | 22s | 2026-01-24 |
| deepseek-r1:32b | Ollama | not stated | Ollama structured-output | 25.7% | 23% | 31% | - | 2026-01-24 |
| EXAONE 4.0 32B (Q4_K_M) | llama.cpp | Q4_K_M | salvage-parse | **COORDINATE FINDING** (recorded as 0.0% FAIL)⁋ | TIMEOUT | ERROR | FAIL | 2026-04-03 |
| Command-R 35B (Q4_K_M) | llama.cpp | Q4_K_M | salvage-parse | **COORDINATE FINDING** (recorded as 0.0%)⁋ | TIMEOUT | TIMEOUT | FAIL | 2026-04-03 |
| Gemma 3 27B | llama.cpp | not stated | salvage-parse | **COORDINATE FINDING** (recorded as 0.0%)⁋ | TIMEOUT | ERROR | FAIL | 2026-01-24 |
| llama3.3:70b-instruct-q2_K | Ollama | q2_K | Ollama structured-output | **COORDINATE FINDING** (recorded as CRASH)⁋ | - | - | - | 2026-01-24 |

† As named in the run record only, never verified against the checkpoint's own `config.json`.
`MEASUREMENT_SPACE.md`'s 2026-09-06 audit of the checkpoints actually served in production found
the repo/tag name does **not** reliably state the weight-quantization scheme (two of the three
checked were misnamed) — the same caution applies retroactively to every "as named" cell here.
Quantization is stated in six of fourteen row names and absent from the other eight; that
inconsistency is itself axis 3 surviving as a naming convention, at scale.

‡ **salvage-parse** = grammar-free generation with the JSON extracted from free text after the
fact by the client. This project bans that mode (constrain the producer via `response_format` /
a real grammar; never salvage-parse free text). Every llama.cpp row above used it. The Ollama
rows used Ollama's own structured-output request parameter instead — a different mechanism, on
an engine this project no longer runs. Neither matches production's current vLLM
`response_format` json_schema path, so no row in this table was produced the way production
extracts today.

**Rule stated once, not per row: a composite metric with a non-scored case anywhere inside it
(TIMEOUT, ERROR, CRASH, OOM) is not itself a score.** When every case in the composite failed
that way, the row is a COORDINATE FINDING (⁋ below) — there is no feasible-point measurement in
it at all. When only some of the cases did, the composite still cannot be read as a score, even
though the surviving case is real data (§ below) — a number that is half quality measurement and
half wall clock is neither.

⁋ **COORDINATE FINDING, not a score** [`MEASUREMENT_SPACE.md`, "A FAILURE ON ANOTHER AXIS IS NOT
A MODEL SCORE"]:
- **Command-R 35B (Q4_K_M)** — "OOM at 32K context (35B too large for KV cache). At 8K both
  entries timeout." An axis-4/5 (KV cache / context length) result written into the model
  column; the model was never scored at a feasible point. The family's *current* build has GQA
  and serves the full 32K context cleanly — see "2026-09-06 re-qualification outcomes" below.
- **Gemma 3 27B** — "too slow, times out on both." A wall-clock/serving outcome, not a quality
  score. Re-qualified at a feasible point on production's current engine, it beats the model
  currently in production — see below.
- **llama3.3:70b-instruct-q2_K** — "insufficient VRAM at q2_K." An axis-3 (weight quantization)
  result: we chose the quant that did not fit and recorded the model as crashing. Sub-4-bit
  quantization is now banned outright (`CLAUDE.md` §SENTINEL) — "it produces tokens, not
  answers" — so this coordinate is not one this project would choose to retry.
- **EXAONE 4.0 32B (Q4_K_M)** — "census_retail timeout, fed_fomc extraction errors after 4
  retries." A timeout is a wall-clock outcome, same as Gemma 3's; "extraction errors" under
  salvage-parsing is a decoding artifact, not a model property. Re-qualified 2026-09-06: it
  **is** schema-compliant with 0 call errors, and fails only to terminate inside a 4,096-token
  completion budget on 8-9 of 40 articles — see "2026-09-06 re-qualification outcomes" below.

§ **COMPOSITE OF A SCORE AND A FAILURE, not a score:**
- **GLM-4.7-Flash (30B MoE)** — the 48.0% averages a real 65% on `census_retail` with a
  `TIMEOUT` on `fed_fomc`. Unlike the rows above, one half of this composite is genuine
  extraction data; the other half is a wall-clock outcome, and averaging the two produces a
  number that is not readable as either. Re-qualified 2026-09-06: degenerate at both grammar
  settings — non-terminating when unconstrained, empty arrays when constrained — a
  decoding/template interaction, not a family verdict. See "2026-09-06 re-qualification
  outcomes" below.

## Key Findings

### The one comparison this table can actually make: Backend Comparison (Same Model)

- **llama.cpp outperforms Ollama** with Qwen 2.5 32B: 64.6% vs 56.7% F1, a 7.9pp gap — the model
  (and, as far as the record states, its quantization) held fixed, only the engine differing.
- **qwen3:30b-instruct is identical on both backends**: 61.1% F1 either way (14s llama.cpp vs
  16s Ollama) — no engine advantage for this particular model.

This is, per `MEASUREMENT_SPACE.md`, "the one correctly designed comparison in that file":
single-axis, and it found a gap four times the measured weight-quant effect (0.019) and larger
than the ~0.05 effect the current model-swap work is trying to detect. The engine is not a
detail that can be held constant by luck.

**Caveat that keeps it honest:** both rows came from the retired `BenchmarkLlamaServerClient`,
which dropped the request seed on every call (see the top banner). The 7.9pp *magnitude* is
therefore not reproducible. What survives is that the engine axis is real and large, not that
7.9pp is the number — do not requote it as a precise effect size.

### Everything else in the leaderboard above is inadmissible as a model comparison

`MEASUREMENT_SPACE.md`'s admissibility rule allows a comparison only when exactly one axis
differs and the delta is attributed to it, or when it is declared a configuration (Q2) result
attributed to nothing. No other pair of rows above qualifies — engine, quantization (where even
named) and decoding method move together across every family boundary in this table.
"Qwen family dominates — 4 of top 5" was the exact inference that rule forbids, and it has been
removed from this file: it may still be true, but this table cannot show it — the four Qwen rows
sit on two different backends at unstated quants, the strongest Gemma row is on a decoding mode
this project bans, and half the field is Ollama, an engine that no longer exists to re-run.
The same is true of any claim shaped like "Gemma 4 is a massive improvement over Gemma 3" drawn
from this table: one side is a real score, the other is a wall-clock coordinate finding, and
they are not comparable numbers. See "2026-09-06 re-qualification outcomes" below for the
comparisons that were actually designed to survive this rule.

## 2026-09-06 re-qualification outcomes

Six of the nine families this leaderboard eliminated were re-run at a feasible point on
production's own CoD extraction path (40 gold articles, vLLM, `response_format` json_schema —
the decoding mode production actually uses, not salvage-parse). **This is `numbers_f1` on
production's CoD task, a different metric and a different corpus from the Quick Benchmark F1
above and from the substrate `aggregate_f1` in `docs/BACKLOG.md` "MODEL BASELINES" — the three
do not convert to one another.** Full coordinates, sampling, engine flags, sample sizes and every
caveat (including which arms are and are not served alike) live in `docs/BACKLOG.md`; they are
not restated here. See sections "THE FAMILY RE-QUALIFICATION", "The candidate BEATS the incumbent
on production's CoD path", "COLIBRI: NO ADMISSIBLE SCORECARD IS POSSIBLE" and "THE PRECISION
LADDER".

- **Gemma 3 27B** — leaderboard above: `0.0% FAIL` (COORDINATE FINDING, wall-clock, on a decoding
  mode this project bans). Re-qualified: `numbers_f1` **0.6177**, and it **beats the model
  currently in production by +0.1024** (11.5x se, disjoint runs) on production's own engine
  (vLLM 0.19.0).
- **Gemma 4 31B** — not in the original leaderboard at all. `numbers_f1` **0.7570**, served on
  vLLM 0.28.0 — **not** production's pinned engine (0.19.0); `docs/BACKLOG.md` carries the
  caveat this difference costs.
- **Mistral-Small 24B** — leaderboard above: 52.1% on retired Ollama. Re-qualified on vLLM at
  0.5105, indistinguishable from the incumbent's 0.5153. This is deliberately **the control**: it
  shows the Gemma gains above are not a universal "just move it to vLLM" uplift.
- **Command-R** — leaderboard above marks the 35B v01 build a COORDINATE FINDING (no GQA, an
  8K-only checkpoint that was never a 32K model). The family's *current* build serves the full
  32K context cleanly (0 errors) and scores **0.3178**.
- **EXAONE 4.0 32B** — leaderboard above: `0.0% FAIL, "can't follow extraction format"`.
  Re-qualified: it **is** schema-compliant (0 call errors) but fails to terminate inside
  production's 4,096-token completion budget on 8-9 of 40 articles, scoring **0.2218**.
- **GLM-4.7-Flash** — leaderboard above: 48.0% (one of two test cases timed out). Re-qualified:
  degenerate at **both** grammar settings — non-terminating when unconstrained, empty arrays when
  constrained. A second coordinate finding, not an F1.

None of the above is a shipping decision by itself — `docs/BACKLOG.md` names the blockers
(context floor, a ticker-recall regression, provisional criteria) for the one candidate closest
to a swap. `CLAUDE.md` §MODEL_ACCEPTANCE governs what a swap requires.

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
