# Sentinel Extraction: Configuration & Constraints

Reference: [arxiv:2512.24601](https://arxiv.org/html/2512.24601v1) - Recursive Language Models

> **Pipeline architecture lives in [`SentinelCollector/README.md`](../SentinelCollector/README.md)** (with the read-first card in `SentinelCollector/AGENT_README.md`) and the system view in [`ARCHITECTURE.md`](./ARCHITECTURE.md). This document covers the model/backend/VRAM constraints and troubleshooting only.

## Overview

The Sentinel extraction pipeline uses Chain-of-Density (CoD) structured-output prompting to extract economic data from news and documents. While the original design was inspired by the RLM paper's recursive decomposition approach, the production implementation uses prompt-based extraction with JSON schema enforcement rather than code generation. Since the GPU-CoD role-flip (`gpu-cod-roleflip-2026-06-09`), CoD emission runs on the GPU as a single JSON-schema pass (`Extraction__Backend=VllmJson`); the former GPU claim verifier (CoVe `SemanticVerifier`) was removed as dead code in PR #647.

## Extraction Pipeline

| Stage | Backend | Technique | Purpose |
|-------|---------|-----------|---------|
| Content normalization | Trafilatura / Markitdown | HTML->markdown, PDF->markdown | Clean input for LLM |
| JSON-CoD emission | GPU (vLLM, Qwen2.5-32B-AWQ) | JSON-schema `response_format` decode | Structured CoD document (entities / numbers / events / claims) |
| Parse + grounding | dsl-parser-mcp `/parse_json` | Deterministic Lark/AST lift + v2.3.1 verifier | Same `DocumentAst` contract as the CPU DSL path |
| News-signal classification | GPU (vLLM) | Per-article classification | `:sig:` signal identities -> `macro_observations` -> matrix projector |

Rollback path: CPU `llama-server` (qwen3-30b-a3b) GBNF-constrained DSL emission (`Extraction__Backend=LlamaServerDsl`), parsed by the same sidecar via `/parse`.

## Current Configuration

**GPU Backend**: vLLM serving `Qwen/Qwen2.5-32B-Instruct-AWQ` (base model; the sentinel-cove LoRA adapter is no longer served — retained on disk as a forensic record only)

| Parameter | Value | Rationale |
|-----------|-------|-----------|
| Model | Qwen2.5-32B-Instruct-AWQ | 30B+ required for extraction quality |
| Context window | 32,768 tokens | Full-document processing; shorter causes quality degradation |
| Temperature | 0 | Deterministic extraction |
| Max tokens | 4,096 | Loop guard (with repetition penalty 1.1) — a JSON-CoD doc unclosed by ~4K is looping |
| Concurrency | `Extraction__MaxConcurrentExtractions=8`, continuous-streaming dispatch | Keeps the vLLM batch (`max_num_seqs=16`) continuously fed |
| Backend | vLLM with PagedAttention | Continuous batching, structured output support |
| Structured output | JSON schema via `response_format` | Enforces extraction schema at decode time (openai-standard; NOT `guided_json`) |

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
- The CPU rollback path also uses a >=30B-class model (qwen3-30b-a3b MoE, ~3B active params)

## Troubleshooting

| Symptom | Cause | Fix |
|---------|-------|-----|
| Extraction timeouts | vLLM overloaded or model not loaded | Check `curl http://vllm-server:8000/health` |
| Poor extraction quality | Wrong model or context reduced | Verify `Model` in appsettings.json matches served model |
| Schema validation failures | Structured output not enforced | Verify vLLM supports `response_format` for this model |
| Empty extractions | Content normalization failed | Check Trafilatura/Markitdown logs |
| GPU OOM | Long-context spike | Restart vLLM, check `gpu_memory_utilization` |

## Do Not

- Reduce context window to save memory (degrades extraction quality)
- Substitute models smaller than 30B for extraction
- Bypass structured output enforcement (produces unparseable JSON)
- Run extraction on CPU outside the documented `LlamaServerDsl` rollback path (timeout city)
