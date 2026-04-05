# LoRA v6: Production-Based Training Data

## Problem

The sentinel-extraction-v5 LoRA regressed extraction accuracy (F1: 64.6% → 32.6%) because its training data was fundamentally misaligned with production reality:

- **99.97% single-extraction examples** — model learned to extract one item and stop
- **114-char average content** — single synthetic sentences, never multi-paragraph documents
- **Simplified instruction prompt** — differed from the production `initial_extraction.txt`

Production documents average 7.2 extractions per document with content spanning thousands of characters.

## Solution

Use real production documents as inputs but generate fresh extraction labels using the Claude API (Sonnet). This avoids contamination from the broken v5 LoRA which produced 48.6% single-extraction documents in the production database.

Train two adapters: a 32B adapter for CoVe (GPU extraction) and a 7B adapter for CoD (CPU summarization).

## Data Pipeline

### Why Not Use Production Extractions

48.6% of production documents have exactly 1 extraction — contaminated by the v5 LoRA which was trained on single-extraction synthetic data. Training on these labels would reintroduce the same bias. The raw documents on disk are clean; only the extraction labels are contaminated.

### Source Documents

- `sentinel.raw_content` — raw documents on disk (1,344 with extractions, many more without)
- Select ~500 documents with real financial content (RSS articles, fed speeches, searxng-content)
- Normalize HTML to markdown via markitdown (or use already-normalized content)
- Filter: file exists, readable, content length 500-30,000 chars

### Label Generation via Claude API

For each raw document, send to Claude Sonnet with the production `initial_extraction.txt` prompt. Claude produces the extraction labels — multiple data points per document, matching the production schema exactly.

```
Claude API call per document:
  system: <initial_extraction.txt prompt with source/content_type filled>
  user: <normalized document content>
  response_format: json array of extractions
```

Estimated cost: ~$5-15 for 500 documents (Sonnet pricing, ~2K input + ~1K output tokens per doc).

### CoVe Training Format (32B)

```json
{
  "instruction": "<contents of initial_extraction.txt with {{source}}, {{content_type}} filled>",
  "input": { "source": "BLS", "content_type": "text/html", "content": "<normalized document text>" },
  "output": [
    { "description": "...", "text_quote": "...", "value": 256000, "unit": "thousands", "period": "2024-12", ... },
    { "description": "...", "text_quote": "...", "value": 4.1, "unit": "percent", ... },
    ...
  ]
}
```

The instruction is the **production prompt** (`initial_extraction.txt`), not a simplified version.
The output is the **Claude-generated extraction**, not the production database extraction.

### CoD Training Format (7B)

For CoD, we also use Claude to generate dense summaries from the raw documents:

```json
{
  "instruction": "<contents of cod_initial_summary.txt with {{source}}, {{target_word_count}} filled>",
  "input": { "source": "BLS", "content_type": "text/html", "content": "<normalized document text>" },
  "output": "<Claude-generated dense summary>"
}
```

### Validation & Filtering

After Claude labels each document:
- Parse JSON output, discard if invalid
- Require 2+ extractions per document (discard single-extraction results)
- Validate required fields present (description, value, text_quote, period)
- Verify text_quote appears in source document
- Include ~10% negative examples: send non-financial documents (methodology, demographics) and expect empty arrays

### Golden Dataset Integration

Include the 12 golden dataset entries in training (hand-labeled, known-good). Hold out 2 entries (census_retail, fed_fomc) for quick validation during training. Use all 12 for the full benchmark after training.

## Training Configuration

### 32B Adapter (CoVe — GPU extraction)

| Parameter | Value | Rationale |
|-----------|-------|-----------|
| Base model | Qwen/Qwen2.5-32B-Instruct | Production model, fits RTX 5090 |
| LoRA rank | 32 | Higher than v5's 16 — more capacity for multi-extraction patterns |
| Alpha | 64 | 2x rank (standard) |
| Targets | q_proj, k_proj, v_proj, o_proj | Full attention — v5's q/v-only was too conservative |
| Epochs | 3 | Real data, lower overfitting risk |
| Learning rate | 5e-5 | Gentler than v5's 1e-4 for 32B model |
| Max seq length | 8192 | Real documents exceed 4096 |
| Batch size | 1 | GPU memory constraint |
| Gradient accumulation | 8 | Effective batch = 8 |
| Quantization | 4-bit NF4 (QLoRA) | Memory efficiency |
| Dropout | 0.05 | Regularization |

### 7B Adapter (CoD — CPU summarization)

| Parameter | Value | Rationale |
|-----------|-------|-----------|
| Base model | Qwen/Qwen2.5-7B-Instruct | CPU model for CoD pipeline |
| LoRA rank | 64 | Higher rank compensates for smaller model capacity |
| Alpha | 128 | 2x rank |
| Targets | q_proj, k_proj, v_proj, o_proj | Full attention |
| Epochs | 3 | |
| Learning rate | 1e-4 | Standard for 7B |
| Max seq length | 8192 | |
| Batch size | 1 | |
| Gradient accumulation | 8 | |
| Quantization | 4-bit NF4 | |
| Dropout | 0.05 | |

## Benchmark Plan

Run **full benchmark** (all 12 golden dataset entries) for each configuration:

1. **Baseline**: base qwen2.5:32b-instruct (no adapter) — current 59.7% F1
2. **v6-32B**: 32B + new CoVe adapter
3. **Baseline 7B**: base qwen2.5:7b-instruct — current 55-64% F1
4. **v6-7B**: 7B + new CoD adapter

Use `--filter "Category=LlmBenchmark"` for the full 12-entry suite. Compare F1, precision, recall, and per-entry scores.

**Success criteria**: v6 adapter F1 > base model F1 on the full benchmark. If not, the adapter is discarded and the base model continues in production.

## Implementation

### New Script: `generate_production_training.py`

Single script that:
1. Connects to TimescaleDB to get raw_content metadata (source, content_type, file paths)
2. Reads and normalizes raw files from disk
3. Calls Claude API (Sonnet) with production prompt for each document
4. Validates Claude's extraction output (JSON parse, required fields, text_quote in source)
5. Filters: keeps documents with 2+ valid extractions
6. Outputs two files: `sentinel-v6-cove.json` (for 32B) and `sentinel-v6-cod.json` (for 7B)
7. Reports statistics: document count, extraction distribution, content length distribution
8. Resumable: tracks processed document IDs to allow restart without re-calling the API

### Modified: `train_qlora.py`

- Add `--target-modules` CLI arg (default: `q_proj,k_proj,v_proj,o_proj`)
- Add `--lora-rank` CLI arg (default: 32)
- Update default `max_seq_length` to 8192
- No other changes — the format_example function already handles the instruction/input/output format

### GGUF Export

After training, convert adapter to GGUF for llama-server / ollama:
```bash
python -m llama_cpp.convert_lora --base <base_model> --lora <adapter_dir> --outfile <output.gguf>
```

## Files

| File | Action |
|------|--------|
| `SentinelCollector/scripts/generate_production_training.py` | New — Claude API labeling pipeline |
| `SentinelCollector/scripts/train_qlora.py` | Modify — add CLI args for targets/rank/seq_length |
| `/opt/ai-inference/training-data/sentinel-v6-cove.json` | Output — 32B CoVe training data |
| `/opt/ai-inference/training-data/sentinel-v6-cod.json` | Output — 7B CoD training data |

## Cost Estimate

- ~500 documents x ~3K tokens/doc (input + output) = ~1.5M tokens
- Claude Sonnet: ~$4.50 input + ~$7.50 output = ~$12 total
- Training: GPU time on RTX 5090, ~2-4 hours per adapter
