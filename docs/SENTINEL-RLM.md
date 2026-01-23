# Sentinel Extraction: RLM Methodology

Reference: [arxiv:2512.24601](https://arxiv.org/html/2512.24601v1) - Recursive Language Models

## Overview

`sentinel-extraction-v5` uses the Recursive Language Model (RLM) approach for economic data extraction. This methodology treats input documents as environment variables that the model programmatically examines and decomposes, rather than processing in a single pass.

## Critical Configuration

| Parameter | Value | Rationale |
|-----------|-------|-----------|
| Model size | ≥30B | Smaller models lack coding ability for RLM decomposition |
| Context window | 32768 | Full-document decomposition; shorter causes context rot |
| NUM_PARALLEL | 1 | Fits model + 32K KV cache entirely on GPU |

## Why These Matter

### Context Window (32K)

- **Context rot**: Information-dense tasks (extraction) degrade faster than simple retrieval as context shrinks
- **RLM decomposition**: Model must see entire document to write code that examines/chunks it
- Reducing context breaks the extraction pipeline - model can't decompose what it can't see

### Model Size (30B+)

- RLM requires the model to write Python code examining context and making recursive sub-calls
- 8B models "struggled without sufficient coding abilities" (paper finding)
- Only frontier-class models effectively implement RLM strategies

### Parallelism (NUM_PARALLEL=1)

- 32K context × 2 parallel slots = 16GB KV cache → forces CPU offload
- 32K context × 1 parallel slot = 8GB KV cache → fits on GPU
- CPU offload causes 3-4x latency increase, leading to timeouts

## GPU Memory Budget (RTX 3090 / 32GB)

```
Model weights:     18.1 GB
KV cache (1 slot): 8.0 GB
Graph memory:      2.6 GB
─────────────────────────
Total:             ~29 GB (fits in 32GB)
```

With NUM_PARALLEL=2: KV cache doubles to 16GB → 41.5GB required → CPU offload

## Troubleshooting

| Symptom | Cause | Fix |
|---------|-------|-----|
| Extraction timeouts | CPU offload from NUM_PARALLEL>1 | Set NUM_PARALLEL=1 |
| Poor extraction quality | Context reduced below 32K | Restore full context |
| Model won't load | GPU OOM | Restart Ollama, verify NUM_PARALLEL=1 |
| Degraded accuracy | Wrong model (<30B) | Use sentinel-extraction-v5 |

## Do Not

- Reduce context window to save memory
- Increase NUM_PARALLEL for throughput
- Substitute smaller models for cost savings
- Enable flash attention (causes memory layout issues with this model)
