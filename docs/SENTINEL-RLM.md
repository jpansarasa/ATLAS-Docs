# Sentinel Extraction: Architecture & Configuration

Reference: [arxiv:2512.24601](https://arxiv.org/html/2512.24601v1) - Recursive Language Models

## Overview

The Sentinel extraction pipeline uses structured-output prompting with Chain of Verification (CoVe) and Chain of Density (CoD) techniques to extract economic data from news and documents. While the original design was inspired by the RLM paper's recursive decomposition approach, the production implementation uses prompt-based extraction with JSON schema enforcement rather than code generation.

## Extraction Pipeline

| Stage | Backend | Technique | Purpose |
|-------|---------|-----------|---------|
| Content normalization | Trafilatura / Markitdown | HTML→markdown, PDF→markdown | Clean input for LLM |
| Summarization (CoD) | CPU (Ollama, qwen2.5:7b) | 5-iteration Chain of Density | Dense context summary |
| Extraction (CoVe) | GPU (vLLM, sentinel-cove-v6.2) | Structured output + validation retry | JSON data extraction |
| Epistemic markers | CPU (Ollama, qwen2.5:7b) | Single-pass extraction | Uncertainty, conditions, attributions |

CoD and CoVe run in parallel — summarization on CPU doesn't block GPU extraction.

## Current Configuration

**GPU Backend**: vLLM serving Qwen2.5-32B-Instruct-AWQ with sentinel-cove-v6.2 LoRA adapter

| Parameter | Value | Rationale |
|-----------|-------|-----------|
| Model | sentinel-cove-v6.2 (Qwen2.5-32B-Instruct-AWQ + LoRA) | 30B+ required for extraction quality |
| Context window | 32,768 tokens | Full-document processing; shorter causes quality degradation |
| Temperature | 0 | Deterministic extraction |
| Max tokens | 4,096 | Extraction output limit |
| Backend | vLLM with PagedAttention | Continuous batching, structured output support |
| Structured output | JSON schema via `response_format` | Enforces extraction schema at decode time |

**CPU Backend**: Ollama serving qwen2.5:7b-instruct for CoD summarization.

## GPU Memory Budget (RTX 5090 / 32GB)

vLLM with PagedAttention manages KV cache dynamically — no fixed slot allocation like Ollama's `NUM_PARALLEL`. The `gpu_memory_utilization` parameter (set to 0.92) controls how much VRAM vLLM reserves for the KV cache pool.

```
Model weights (AWQ 4-bit): ~18 GB
KV cache pool (dynamic):   ~11 GB (0.92 * 32GB - weights - overhead)
Overhead (CUDA, graphs):   ~3 GB
```

PagedAttention allocates KV cache in pages on demand per request, avoiding the fixed-slot fragmentation that forced `NUM_PARALLEL=1` on Ollama.

## Critical Constraints

### Context Window (32K)

- Information-dense extraction tasks degrade as context shrinks
- Model must see the full document to extract all observations
- Reducing context causes missed data points and hallucinated dates

### Model Size (30B+)

- Extraction quality requires strong instruction-following and JSON formatting ability
- 7B/8B models produce more schema violations and lower confidence scores
- The 7B CPU model is adequate for summarization (CoD) but not extraction (CoVe)

## Troubleshooting

| Symptom | Cause | Fix |
|---------|-------|-----|
| Extraction timeouts | vLLM overloaded or model not loaded | Check `curl http://vllm-server:8000/health` |
| Poor extraction quality | Wrong model or context reduced | Verify `Model` in appsettings.json matches served model |
| Schema validation failures | Structured output not enforced | Verify vLLM supports `response_format` for this model |
| Empty extractions | Content normalization failed | Check Trafilatura/Markitdown logs |
| GPU OOM | LoRA + long context spike | Restart vLLM, check `gpu_memory_utilization` |

## Do Not

- Reduce context window to save memory (degrades extraction quality)
- Substitute models smaller than 30B for extraction (use only for CoD)
- Bypass structured output enforcement (produces unparseable JSON)
- Run extraction on CPU (timeout city)
