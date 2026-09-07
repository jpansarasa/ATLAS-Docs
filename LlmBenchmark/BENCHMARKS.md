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

All three leaders beat production by more than the ~0.06 confound band, so each of those gains over
production is real. The order among the leaders is not: the top two are 0.0438 apart — inside that
same band — and were measured on different engines.

**Mistral-Small is the control.** It lands within noise of production despite being re-measured on
the same engine and decoding path as every row but Gemma 4 — which is what rules out "these gains
are an artifact of the new harness" for the rows on 0.19.0. It does not cover the engine axis on
Gemma 4, whose +0.2417 stays a configuration delta with engine and model moving together.

### Serving coordinates required to reproduce these numbers

Not preferences — without them the score above is not what you get.

- **Qwen3.8-27B requires thinking disabled.** Its vendor chat template enables reasoning by default;
  at production's 4,096-token completion budget that leaves the JSON unclosed on **110 of 120
  documents** and the run is unscoreable. The 0.7132 is the thinking-off arm.
- **Gemma 4 31B needs vLLM 0.28.0, and no *stock-derived* 0.19.0 image serves it** — re-pinned
  wheels, nothing else; patching vLLM source was not tried. Measured 2026-09-07; the arm above
  stands on stock `vllm/vllm-openai:v0.28.0` (`transformers` 5.15.1). 0.19.0 is not short a
  dependency, it is short a **capability**, so Gemma 4 stays the one arm off production's engine and
  the +0.2417 remains a configuration delta with engine and model moving together. Detail in
  *An engine axis that a version bump cannot cross* below.
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

## An engine axis that a version bump cannot cross

**Question:** does a *stock-derived* vLLM 0.19.0 — stock 0.19.0 with its wheels re-pinned, nothing
else — serve Gemma 4 31B, so the +0.2417 could be re-taken on production's own engine?

**Answer, measured 2026-09-07: no, and no re-pin can change it.** Recorded as a coordinate finding,
never as a score. The arm produced no `numbers_f1` because it never reached a first token.

The derived image the earlier note called untested (`vllm-requal:tx5161` = stock 0.19.0 +
`transformers` 5.16.1) *was* built and *was* pointed at Gemma 4. It failed, and re-running it at the
exact coordinate of the 0.7570 row — `--max-model-len 32768 --max-num-seqs 6
--gpu-memory-utilization 0.90 --kv-cache-dtype fp8_e4m3 --generation-config vllm`, same substrate
sha `008c338d`, same gold sha `bc9c5b4c`, same 69-byte chat template `7a8a39d2` — reproduces the
failure.

`tx5161` is one committed layer over the digest-pinned `vllm_image` in
`deployment/ansible/group_vars/all.yml` — `pip install --no-deps -U transformers==5.16.1 tokenizers
huggingface_hub safetensors`, torch left at 2.10.0+cu129. Rebuild from that line; the image is local
only and one prune from gone.

**The mechanism is not the one the note assumed.** `model_type: gemma4` parses fine, and 0.19.0
registers `Gemma4ForConditionalGeneration` and ships `gemma4.py`. Serving dies earlier, in
`ModelConfig.__post_init__`:

```
# vllm/transformers_utils/model_arch_config_convertor.py, get_head_size(), inside the image
head_dim = getattr(self.hf_text_config, "head_dim", 0)
-> transformers.integrations.heterogeneity ...
   AmbiguousGlobalPerLayerAttributeError: 'head_dim' is a per-layer attribute
```

Gemma 4 31B is genuinely heterogeneous — `head_dim` 256 with 16 KV heads on sliding-attention
layers, `global_head_dim` 512 with 4 on the 10 full-attention layers of 60. `Gemma4TextConfig`
inherits `HeterogeneousConfigMixin` and declares both fields per-layer, so **every** `transformers`
that knows `gemma4` refuses the flat read, and the one that permits it cannot parse the model:

| `transformers` | knows `gemma4`? | permits 0.19.0's flat `head_dim` read? |
|---|---|---|
| 4.57.6 (stock 0.19.0) | no | yes — no heterogeneity module at all |
| 5.15.1 (stock 0.28.0) | yes | **no** — raises |
| 5.16.1 (the derived image) | yes | **no** — raises |

Only the 5.16.1 row is a boot attempt — the one above. The other two were read out of the
`transformers` packages themselves: 4.57.6 ships no heterogeneity module, 5.15.1 carries the same
`Gemma4TextConfig`. Neither was put in front of Gemma 4 on 0.19.0.

That pincer is why no wheel re-pin exists. Of the two escapes one was tried and failed, the other
was ruled out unrun: `--hf-overrides '{"allow_global_per_layer_attribute_access": true}'` does not
reach the config before `ModelConfig` reads it, and setting the flag on the checkpoint would hand a
homogeneous engine one head size for a model with two — a point that loads without being feasible,
which is why it was not attempted.

0.28.0 clears it by *capability*, not by dependency: a per-layer `model_arch_config[layer_idx]`, a
`transformers_utils/configs/gemma4.py` that materialises the per-layer overrides, and
`get_num_kv_heads(arch_config=…)` sized one layer at a time. 0.19.0 represents head size as a single
scalar (`max(head_dim, global_head_dim)`) and has no per-layer path to fix. Closing this would mean
patching vLLM source, which is no longer a stock-derived image and no longer this question.

**What this does not establish.** It says nothing about Gemma 4's quality — 0.7570 stands unchanged
on 0.28.0. It does not price the engine step *for Gemma 4*; the incumbent controls put that step at
+0.0005 (0.4540 → 0.4545, well inside noise) on Qwen2.5-AWQ **at Gemma 4's serving point** — seqs 6
/ util 0.90, on the control arm's own chat template, not the seqs 16 / util 0.95 point the 0.5153
row above was taken at, which is why neither level is that row's — and whether an engine result
transfers between models is untested: the non-transfer rule below was measured on **precision**, not
on engines. And it is a statement about **0.19.0**,
not about every engine between it and 0.28.0 — the intermediate releases were not tested.

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
