# Why vLLM dominates multi-request inference on high-end GPUs

**vLLM's core advantage over llama.cpp and Ollama is architectural: PagedAttention and continuous batching deliver 35–44× higher throughput under concurrent load, while llama.cpp's throughput stays flat regardless of how many requests queue up.** For a single RTX 5090 workstation serving multiple applications, agents, or parallel chains simultaneously, this difference is transformative. The tradeoff is real, though — llama.cpp remains ~5% faster for single-user interactive chat, starts in seconds rather than minutes, and offers unmatched quantization granularity via GGUF. On your Threadripper 9960X + RTX 5090 setup with 128GB DDR5, vLLM unlocks the GPU's full potential for any workload beyond one-request-at-a-time chat.

---

## PagedAttention eliminates 60–80% of KV cache waste

The fundamental innovation separating vLLM from llama.cpp is how each system manages the KV cache — the per-request memory that stores attention keys and values and grows linearly with sequence length.

**llama.cpp** pre-allocates a contiguous ring buffer for the entire maximum context window at startup. When you launch `llama-server -c 32768 --parallel 4`, it reserves four fixed-size KV cache slots, each sized for 32,768 tokens, whether or not any request actually uses that full context. This is simple and fast but wasteful: the vLLM paper demonstrated that traditional systems waste **60–80% of allocated KV cache memory** through internal fragmentation (reserved-but-unused slots), external fragmentation (non-contiguous free blocks), and redundant duplication of shared prefixes.

**vLLM's PagedAttention** borrows directly from OS virtual memory paging. The KV cache is divided into fixed-size blocks (default 16 tokens each), allocated on demand from a pre-initialized free pool. Each request maintains a block table mapping logical block indices to physical GPU memory addresses — blocks need not be contiguous. When a sequence grows, new blocks are popped from the free pool. When a request completes, blocks return immediately. This reduces KV cache waste to **under 4%**, with the only overhead being partially-filled final blocks.

The practical impact for your RTX 5090 (32GB GDDR7) is significant. For a Llama 3.1 8B model in FP8, the KV cache consumes roughly **128 KB per token** (2 × 32 layers × 8 KV heads × 128 head_dim × 1 byte). With PagedAttention managing ~20GB of free VRAM as a block pool, vLLM can sustain dozens of concurrent sequences with varying context lengths, dynamically sharing that 20GB. llama.cpp, by contrast, would pre-commit fixed allocations per slot — if you configure 4 parallel slots at 32K context, you commit ~16GB to KV cache whether those slots are busy or idle.

**Copy-on-write** further optimizes memory for beam search and parallel sampling. Multiple sequences sharing a common prefix reference the same physical blocks until one diverges, at which point only the divergent block is copied. This saves up to **55%** memory for those workloads. vLLM also enables **automatic prefix caching** by default in V1: a hash-based lookup matches incoming prompt blocks against cached KV data, skipping redundant prefill computation for shared system prompts — a major win for agentic workflows where every request starts with an identical system prompt.

---

## Continuous batching scales throughput linearly with concurrency

The second architectural pillar is **continuous batching** (iteration-level scheduling). After every single token generation step, vLLM's scheduler re-evaluates which requests to include in the next forward pass: completed requests are removed and their KV blocks freed, while waiting requests are added if resources permit. This eliminates the straggler problem inherent in static batching, where the GPU idles waiting for the longest sequence to finish.

The benchmark data from controlled Red Hat Developer tests on an H200 GPU (September 2025) illustrates the divide starkly:

| Concurrent users | vLLM throughput (tok/s) | llama.cpp throughput (tok/s) | vLLM advantage |
|-----------------|------------------------|-----------------------------|----|
| 1 | ~comparable | ~comparable | ~1× |
| 4 | Growing | Flat | ~2–3× |
| 16 | Significantly higher | Flat | ~10×+ |
| 64 | Peak | Flat | **35–44×** |

llama.cpp's throughput remained "almost perfectly flat" regardless of load. It uses a queuing model — excess requests wait in line while fixed slots process sequentially. Even tuning Ollama with `OLLAMA_NUM_PARALLEL=32` on an A100-40GB yielded only 41 tokens/sec total versus vLLM's **793 tokens/sec** — a 19× gap.

On your RTX 5090, independent DEV Community benchmarks (March 2026) with a Nemotron 9B model in BF16 showed vLLM achieving **~83 tok/s for a single request** scaling to **~630 tok/s at 10 concurrent requests** — a 7.5× throughput multiplier from continuous batching alone. RunPod benchmarks pushed small models to **65,000+ tok/s** at 1024 concurrent prompts, demonstrating near-linear scaling.

**The latency tradeoff** is worth understanding: at high concurrency, vLLM's inter-token latency (ITL) per request rises slightly because larger batches mean more compute per step. llama.cpp's ITL stays constant because it processes the same fixed workload regardless of queue depth. However, vLLM's **time to first token (TTFT)** stays remarkably stable under load while llama.cpp's TTFT grows exponentially as requests queue. For interactive applications serving multiple users, vLLM's behavior is strictly better.

---

## Quantization landscape favors FP8 on RTX 5090 with vLLM

Your RTX 5090's 5th-generation Tensor Cores with hardware FP8 and FP4 support create a clear performance advantage for vLLM's quantization approach over llama.cpp's.

**vLLM's FP8 W8A8** (weight and activation quantization) runs natively on Blackwell Tensor Cores at ~400 TFLOPS. It requires no calibration data for dynamic quantization, delivers **2× memory reduction** and up to **1.6× throughput improvement** over FP16, with minimal accuracy loss. vLLM also supports FP8 KV cache (`--kv-cache-dtype fp8`), halving KV cache memory and effectively doubling achievable context length. For an 8B model in FP8, model weights occupy ~8GB, leaving ~20GB for KV cache — enough for **~160K tokens of context** (or ~320K with FP8 KV cache).

**llama.cpp cannot directly leverage FP8 Tensor Cores.** Its quantization is weight-only: GGUF k-quants (Q4_K_M, Q5_K_M, etc.) dequantize weights to FP16 for computation. While llama.cpp's custom MMQ CUDA kernels are impressively efficient and exploit INT8 Tensor Cores where available, they don't access the RTX 5090's FP8 or FP4 hardware paths. This represents a growing architectural gap as NVIDIA pushes lower-precision tensor core operations.

The quantization options break down as follows for practical RTX 5090 usage:

- **vLLM FP8 W8A8** — Best overall for your hardware. ~8GB for an 8B model, maximum throughput, minimal quality loss, no calibration needed
- **vLLM AWQ/GPTQ W4A16** — For fitting 27B–34B models in 32GB. AWQ is 5–15% faster than GPTQ in vLLM's Marlin kernels. Provides +46% throughput improvement when VRAM-constrained
- **vLLM NVFP4** — Experimental Blackwell-native FP4; doubles throughput vs FP8 on compatible models but software support is still maturing
- **llama.cpp Q4_K_M** — The workhorse for fitting large models. ~4.8 bits per weight, 70% size reduction, good quality. Best for single-user interactive chat
- **llama.cpp Q6_K/Q8_0** — Near-lossless quality when models fit comfortably. Q8_0 on a Qwen3 8B achieves ~186 tok/s on your RTX 5090
- **llama.cpp IQ2/IQ3** — Ultra-low-bit codebook quantization for extreme compression. Quality degrades significantly but enables running 70B models (barely) in 32GB

**One critical note**: vLLM's GGUF support exists but is explicitly described as "highly experimental" with "poor performance" — it is not recommended. If migrating to vLLM, convert your workflow to AWQ, GPTQ, or FP8 models from HuggingFace rather than trying to serve existing GGUF files.

---

## vLLM's V1 engine rewrote the performance ceiling

The V1 engine, default since v0.8.0 (early 2025) and now the only engine in v0.19.0 (April 3, 2026), represents a ground-up architectural rewrite. The key changes directly impact single-GPU performance:

**Decoupled multi-process architecture** separates the scheduler, input preparation, detokenization, and API server into independent processes from the GPU execution loop. On smaller models where GPU forward passes complete in ~5ms (e.g., 8B on your RTX 5090), CPU overhead from Python scheduling had been consuming over 60% of wall-clock time. V1 eliminates this bottleneck with zero-copy DMA transfers and msgpack-based inter-process serialization, achieving **up to 24% throughput improvement** over V0.

**Automatic optimizations** mean you no longer need to manually tune `--num-scheduler-steps` or `--enable-chunked-prefill`. V1 always runs chunked prefill with dynamic budget allocation between prefill and decode stages. Prefix caching operates at zero overhead when enabled (default). A new `--performance-mode` flag (`balanced`, `interactivity`, or `throughput`) simplifies the remaining tuning.

Recent releases have added features that matter for your setup:

- **FlashAttention 4 integration** (v0.17.0, March 2026) for improved attention performance
- **Speculative decoding with CUDA graphs** — Eagle3 support with 8× faster spec decode (v0.19.0)
- **CPU KV cache offloading** (v0.19.0) — leverages your 128GB DDR5 as overflow for very long contexts
- **NVFP4 MoE support** for Blackwell SM120 GPUs
- **gRPC serving** (v0.18.0) for lower-overhead API calls
- **Anthropic API compatibility** alongside the existing OpenAI-compatible endpoint

The OpenAI-compatible API is production-ready: `/v1/chat/completions`, `/v1/completions`, `/v1/embeddings`, streaming, tool/function calling (with pluggable parsers for Hermes, Llama3, Mistral formats), vision inputs, and audio transcription. Switching from llama.cpp's server or Ollama requires only changing the `base_url`:

```python
from openai import OpenAI
client = OpenAI(base_url="http://localhost:8000/v1", api_key="token-abc123")
```

---

## Practical migration path for your RTX 5090 workstation

For serving a model on your single-GPU setup, the simplest migration path:

```bash
# Install (pip or Docker)
pip install vllm  # Requires CUDA 12.x+, Python ≥3.10

# Serve Llama 3.1 8B in FP8 (optimal for RTX 5090)
vllm serve meta-llama/Llama-3.1-8B-Instruct \
    --dtype auto \
    --kv-cache-dtype fp8 \
    --max-model-len 32768 \
    --gpu-memory-utilization 0.92 \
    --performance-mode balanced

# Or serve a larger quantized model
vllm serve TechxGenus/Qwen2.5-32B-Instruct-AWQ \
    --quantization awq \
    --max-model-len 16384 \
    --gpu-memory-utilization 0.92
```

Key configuration parameters for your hardware: set `--gpu-memory-utilization` to **0.90–0.95** (vLLM pre-allocates this fraction as the KV block pool), `--max-model-len` to your actual maximum needed context (over-provisioning wastes blocks), and `--max-num-seqs 64` (default) for the concurrent sequence limit. Docker is recommended for production to avoid Blackwell driver/CUDA compatibility issues: `vllm/vllm-openai:latest` with `--gpus all`.

**Model format matters**: abandon GGUF files for vLLM. Search HuggingFace for AWQ or GPTQ variants of your preferred models — nearly every popular model has them. For FP8, many models work with vLLM's dynamic FP8 quantization automatically (no special model file needed; just add `--quantization fp8`).

**Startup time warning**: vLLM takes **30 seconds to several minutes** to initialize (model download, weight loading, CUDA graph compilation, profiling run). This is a one-time cost per server start, vastly different from llama.cpp's near-instant loading. Plan for persistent serving rather than on-demand model swapping — if you frequently switch between models, keep Ollama installed alongside vLLM for quick experimentation.

---

## When llama.cpp remains the better choice

Despite vLLM's throughput advantages, llama.cpp wins decisively in several scenarios relevant even to high-end workstations:

**CPU/GPU layer offloading** is llama.cpp's killer feature for oversized models. Running a 70B model at Q4_K_M (~35GB) on 32GB VRAM is impossible in vLLM, which requires the entire model in GPU memory. llama.cpp's `-ngl` flag gracefully offloads excess layers to your 128GB DDR5, running at reduced but functional speed (~10–15 tok/s for overflow layers). This makes your Threadripper's massive RAM directly useful for inference — vLLM cannot do this for model weights (only KV cache offloading was added in v0.19.0).

**Single-user interactive latency** favors llama.cpp by a consistent **4–6% margin**. The scheduling overhead, Python runtime, and HTTP server stack in vLLM add measurable latency that doesn't exist in llama.cpp's lean C++ path. For a personal chat interface with one active conversation, llama.cpp delivers slightly snappier responses.

**Quantization flexibility** is dramatically wider in llama.cpp. The GGUF format offers **15+ quantization levels** from IQ1_S (1.5-bit) through Q8_0 (8-bit), with importance-matrix-aware quantization that preserves quality on critical layers. vLLM offers FP8, AWQ (4-bit), GPTQ (4/8-bit), and experimental FP4 — fewer options with less granularity.

**Platform support** is incomparable: llama.cpp runs on Windows natively, Apple Silicon with Metal acceleration, AMD consumer GPUs via Vulkan, and even Raspberry Pi. vLLM requires Linux (WSL2 on Windows), an NVIDIA or AMD datacenter GPU, and a substantial Python environment. Ollama wraps llama.cpp with a one-command install that makes model management trivial — `ollama run llama3.1` versus vLLM's multi-step setup.

**Rapid model switching** matters for experimentation. llama.cpp loads a GGUF file in seconds; vLLM's initialization pipeline takes minutes. If you're evaluating many models quickly, Ollama's `ollama run` workflow is unbeatable.

---

## Conclusion

For your Threadripper 9960X + RTX 5090 workstation, the decision hinges on concurrency. **If you're running agentic workflows, parallel tool calls, multiple chat sessions, or any scenario with 2+ simultaneous inference requests, vLLM's continuous batching and PagedAttention deliver throughput gains that compound dramatically** — 7.5× at 10 concurrent requests, 35×+ at 64. The RTX 5090's FP8 Tensor Cores are a direct accelerator for vLLM's quantization strategy, delivering quality-preserving 2× memory savings that llama.cpp simply cannot access.

The optimal setup for your hardware is likely **both tools coexisting**: vLLM as a persistent server (FP8 quantization, 92% GPU memory utilization, OpenAI-compatible endpoint) for production workloads and multi-request scenarios, with Ollama kept installed for quick model evaluation, CPU-offloaded experimentation with 70B+ models across your 128GB DDR5, and single-user chat where its sub-second startup and marginally lower latency provide a better experience. The two serve genuinely different niches that your hardware is well-positioned to exploit.