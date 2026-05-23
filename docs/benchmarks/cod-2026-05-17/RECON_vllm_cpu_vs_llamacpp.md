# Recon — vLLM CPU vs llama.cpp for CoD DSL extraction

Date: 2026-05-22 (UTC). Branch: `experiment/recon-vllm-cpu-vs-llamacpp`.
Smoke artefacts: `smoke/recon_vllm_vs_llamacpp/results.jsonl`.

## 1. vLLM CPU image options

Source: <https://docs.vllm.ai/en/stable/getting_started/installation/cpu/#pre-built-images>

| Item | Finding |
|---|---|
| Official image (x86) | `vllm/vllm-openai-cpu:latest-x86_64`, also `v0.21.0-x86_64` / `v0.21.0` (8 days old, 1.15 GB compressed, ~3.8 GiB unpacked) |
| Architectures | x86 (AVX-512 recommended, AVX2 limited), ARM AArch64, IBM Z s390x, Apple Silicon (build-from-source) |
| Quantization on CPU | AWQ (x86 only), GPTQ (x86 only), compressed-tensor INT8 W8A8 (x86 + s390x). **No GGUF on CPU.** |
| Structured outputs | xgrammar + guidance (auto-routed); `response_format: {"type":"json_schema",...}` works via OpenAI endpoint. `guided_json` deprecated → `structured_outputs: {json: ...}`. **No GBNF.** |
| Launch flags used | `--max-model-len 32768 --dtype bfloat16 --quantization awq --enforce-eager`, env `VLLM_CPU_KVCACHE_SPACE=8 VLLM_CPU_OMP_THREADS_BIND=0-23`, `--cap-add SYS_NICE --shm-size=4g` |
| Mercury CPU compatibility | AMD Ryzen Threadripper 9960X, 24c/48t, AVX-512F + AVX-VNNI + AVX-512-BF16 — fully supported |
| RAM footprint observed | KV-cache reservation 8 GiB + ~5.2 GiB weights ~ resident ≤14 GiB; mercury has 81 GiB free |
| Container size on disk | 3.8 GiB unpacked (`vllm-openai-cpu:v0.21.0`) vs ~140 MiB for `llama.cpp:server` |
| Startup time | 50 s from `nerdctl run` → `/health 200` (cold, no torch.compile, eager) |

Cross-check:
* `project_vllm_sentinel` memory — vLLM AWQ + LoRA on GPU works; 32B local quant OOMs RTX 5090. This recon stays under 8B and runs on **CPU only**, so GPU OOM constraints don't apply.
* `project_turboquant_blocked` — irrelevant on CPU build.
* `project_data_ml_context` — `guided_json` broken in vLLM 0.19; we're on **v0.21.0** and the docs confirm `guided_json` is deprecated in favour of `structured_outputs`/`response_format`. Did not exercise structured outputs here (CoD DSL is custom text, not JSON).

## 2. 8B model candidates

Pulled: `Qwen/Qwen2.5-7B-Instruct-AWQ` (W4A16, 5.2 GiB on disk at `/opt/ai-inference/models-vllm/`).
Not pulled: `Qwen3-8B-AWQ` — no official AWQ build on Hugging Face as of this recon.
Loaded successfully into vLLM CPU on first try; weights load in 0.25 s after mmap (EXT4, no prefetch).

## 3. Smoke benchmark

Setup: v8 baseline prompt (`prompts/cod_dsl_baseline.txt`, 846 lines), 3 small Arm-B-style cells.
Backends both single-replica, single in-flight request.

* **llama.cpp** (production baseline): port 11437, qwen3:30b-a3b-instruct-2507-q4_K_M (~3B active MoE), `--parallel 1 --ctx-size 32768`, GBNF v2.1 grammar enforced.
* **vLLM CPU** (recon): port 11438, qwen2.5-7B-AWQ dense, `--max-model-len 32768`, **no grammar enforcement** (vLLM CPU has no GBNF; CoD DSL is not JSON so json_schema doesn't fit).

Grammar validity = `dsl.parser.parse_text_v2_1(content)` returns no errors.

| id | chars | backend | wall (s) | prompt toks | eval toks | tok/s (eval) | stop | grammar valid |
|---|---|---|---|---|---|---|---|---|
| 27560 | 1380 | llama.cpp + GBNF | 146.6 | 9784 | 2048 | 21.7 | **limit** | **false** (truncated mid-block) |
| 27560 | 1380 | vLLM CPU 7B AWQ | 194.0 | 9810 | 1513 | 7.8 | stop | false (`QUAL_VAL '/ macro_indicator'` parse error) |
| 27702 | 1858 | llama.cpp + GBNF | 54.1 | 9791 | 54 | 24.6 | eos | true (article_type=other, no entities) |
| 27702 | 1858 | vLLM CPU 7B AWQ | 10.2 | 9817 | 53 | 5.2 | stop | true (article_type=other) |
| 29807 | 1565 | llama.cpp + GBNF | 135.0 | 9815 | 1844 | 21.9 | eos | true |
| 29807 | 1565 | vLLM CPU 7B AWQ | 78.2 | 9841 | 917 | 11.7 | stop | false (`QUAL_VAL '/ COUNT'` — emitted ent-type after qualifier value) |

Grammar-valid rate: llama.cpp **2/3 (66%)**, vLLM CPU **1/3 (33%)**.
Per-token speed: llama.cpp ~**22 tok/s**, vLLM CPU ~**8-12 tok/s** (~50% of llama.cpp).
Per-cell wall is mixed because vLLM emits **fewer tokens** (the dense 7B is less verbose / less complete), not faster.

## 4. Quality observation

* `27702` (trivial "article_type=other") — both backends got it. vLLM was 5× faster in wall because both stopped at 53 tokens, and vLLM's prompt-eval is faster per cell (no GBNF overhead, AVX-512 BLAS on dense weights).
* `27560` (Fed press release) — llama.cpp hit the 2048-token n_predict cap and got truncated mid-block (grammar would have accepted if completed; truncation is the failure mode, not grammar drift). vLLM emitted plausible-looking DSL but **violated the grammar twice** on canonical CoD slot shapes (qualifier vs ent-type position). Without GBNF the 7B dense model invents shapes the v2.1 grammar rejects.
* `29807` (Iran/Hormuz) — llama.cpp produced valid DSL with named entities. vLLM produced more aggressive entity extraction (Stoxx 600, Brent, WTI, LVMH) but again violated the qualifier-vs-type slot shape (`oil_prices_brent / COUNT` is not legal v2.1).

**Read:** the 30B MoE + GBNF stack is producing **structurally valid** extractions; the 7B AWQ stack produces **structurally invalid** extractions because there is no constraint backstop on CPU. The qualifier-vs-type position confusion is exactly the kind of error GBNF prevents and that pure-prompt prompting struggles with.

## 5. Recommendation: **NOT WORTH a full n=9 benchmark right now.**

Reasons:

1. **No grammar enforcement on vLLM CPU.** vLLM's structured outputs (xgrammar/guidance) target JSON/regex/EBNF/Lark; CoD DSL is a custom textual grammar served via llama.cpp's GBNF. To compare apples-to-apples we'd need to either (a) convert v2.1 GBNF → Lark and use vLLM's `structured_outputs: {grammar: ...}` backend (xgrammar supports a subset of EBNF), or (b) keep vLLM unconstrained and accept lower validity. This recon shows (b) costs roughly **half the grammar-valid rate** on small cells; on a larger corpus where structural complexity scales we'd expect that gap to widen.

2. **Throughput is comparable, not better.** vLLM CPU tok/s is **~40-50% of llama.cpp** on this hardware/model class. The reason is that llama.cpp's q4_K_M MoE with only 3B active params is decode-time efficient on CPU, while vLLM CPU runs the full 7B dense (no expert routing) every step. The MoE-3B-active vs 7B-dense math favours MoE on CPU.

3. **Quality is not obviously better.** vLLM produced more entities on 29807 but in invalid shapes. To know whether the 7B dense beats 30B-MoE on *correctly-shaped* DSL we'd need vLLM under grammar constraint — which is the missing piece in (1).

4. **Operational delta is real and bad.** vLLM CPU image is 3.8 GiB (vs 140 MiB llama.cpp), KV-cache reservation 8 GiB (vs ~2 GiB llama.cpp), no GGUF (we'd hold both AWQ and GGUF copies of every model), and the cold-start path is 50 s (vs ~5 s).

5. **The MoE active-param argument was correct.** Per `project_atlas_inference_topology` and `project_sentinel_llm_strength_layering`: small-CPU CoD is for pattern-recognition + verbatim copy + classification — not for emitting structured DSL. The current DSL emitter is on llama.cpp by design. Moving to a larger dense model on the same path violates the layering rule.

## 6. When to revisit

* If vLLM CPU adds first-class GBNF (currently EBNF/Lark via xgrammar — re-test in v0.22+).
* If we successfully port v2.1 GBNF → Lark and want a structured-output A/B vs llama.cpp GBNF.
* If we move the **extraction** path off MoE-on-CPU back onto a GPU dense model — but per layering rule that path is already on GPU.

## 7. Costs (this recon)

* Image pull: ~25 s (1.15 GiB compressed).
* Model download: ~50 s (5.2 GiB safetensors).
* Startup: 50 s cold to /health=200.
* Smoke benchmark: 3 cells × 2 backends = ~10 minutes wall.
* No production swap. Container `vllm-cpu-recon` left running on :11438 — `sudo nerdctl rm -f vllm-cpu-recon` to clean.
