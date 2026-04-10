# Recursive Language Models and Long-Context Extraction Techniques for Local LLMs

MIT's Recursive Language Model (RLM) represents a breakthrough inference-time approach that wraps existing LLMs to navigate massive contexts programmatically rather than expanding context windows—achieving **2x+ accuracy improvements** on long-context extraction while being cheaper than base model calls. For your Qwen2.5:32b setup, the most impactful immediate fix is likely a configuration issue: **Ollama defaults to only 2048-4096 tokens context**, which almost certainly explains your "context rot." Increasing `num_ctx` to 32K-65K, combined with structured output enforcement and chunking strategies, should dramatically improve extraction reliability.

---

## MIT's Recursive Language Models tackle context rot directly

The December 2025 paper from MIT CSAIL (arXiv:2512.24601, authored by Alex Zhang, Tim Kraska, and Omar Khattab) introduces RLMs not as a new neural architecture but as an **inference-time scaffold** that wraps existing LLMs. The core insight: instead of feeding massive prompts directly into a Transformer, RLMs treat long contexts as external environment variables that the model can programmatically navigate.

The system works by loading input context into a Python REPL environment. The root LLM (which never sees the full context) can write and execute Python code to inspect, grep, slice, and partition the data—then spawn recursive sub-LLM calls to process specific chunks and aggregate results. This "context-centric" decomposition dramatically outperforms standard approaches on extraction tasks.

**Benchmark results show remarkable improvements**: On OOLONG (132K tokens), RLM boosted GPT-5-mini from **30 to 64 points** (114% improvement). On BrowseComp+ with 6-11M token contexts, base GPT-5 scored 0% while RLM achieved **91.33%**. The approach handles 10M+ tokens without performance collapse because no single LLM call ever processes the entire context.

The official implementation at github.com/alexzhang13/rlm (MIT license, 1.1k stars) supports vLLM for local inference. While primarily tested with API models, community implementations exist for quantized models. For your homelab, the scaffold itself is lightweight—the bottleneck is running your local LLM, which you already have. Current limitations include no prefix caching and synchronous blocking calls, making runtime variable from seconds to minutes per query.

---

## Your Ollama configuration likely caps context at 2K-4K tokens

A critical discovery from this research: **Ollama's default context window is severely limited (2048-4096 tokens)**, which would explain degraded accuracy over long documents regardless of Qwen2.5's native 128K capability. This single configuration change may resolve most of your issues.

Create a custom Modelfile to increase context:
```bash
FROM qwen2.5:32b-instruct
PARAMETER num_ctx 65536
PARAMETER temperature 0
PARAMETER num_predict 4096
```

Then build with `ollama create qwen-fin -f Modelfile`. With 32GB VRAM on your RTX 5090, you can safely use 32K-65K context. Each 4K context increase adds roughly 1GB VRAM, so you have substantial headroom.

For API calls, set context dynamically:
```python
response = ollama.generate(
    model="qwen2.5:32b-instruct",
    prompt="...",
    options={"num_ctx": 65536}
)
```

---

## Chunking and hierarchical summarization preserve accuracy across long documents

Research from NVIDIA on FinanceBench found optimal settings for financial documents: **1,024 tokens per chunk with 15% overlap**. The recursive character splitter respects document structure by using separators in order (paragraphs → sentences → words), preventing mid-thought breaks.

For documents exceeding your effective context window, the **Map-Reduce pattern** processes chunks independently before combining:

1. **Map phase**: Extract from each chunk independently
2. **Reduce phase**: Combine and deduplicate results
3. **Recursive collapse**: If combined results exceed context, summarize in tiers

Hierarchical merging works particularly well for financial documents. Create a pyramid: page-level summaries (2000 tokens each) → section summaries (1000 tokens) → document summary (500 tokens). Query the appropriate level based on extraction granularity needed.

The **"Lost in the Middle" problem** (Stanford research) shows models exhibit U-shaped attention—highest accuracy at document beginning and end, significant degradation in the middle. Solutions include strategic ordering (most relevant content at start/end, less important in middle), two-stage prompts that explicitly label primary vs. supplementary passages, and windowed processing with overlap to ensure critical content falls near chunk boundaries.

---

## Constrained decoding guarantees valid JSON at the token level

JSONSchemaBench (Microsoft/EPFL, January 2025) tested 10,000 real-world schemas across six frameworks. **Constrained decoding speeds up generation by 50%** while improving task accuracy by up to 4%—it's strictly better than unconstrained generation.

Ollama's structured output (v0.5+) uses llama.cpp's grammar-based sampling, masking invalid tokens during generation:

```python
from pydantic import BaseModel, Field
from ollama import chat

class FinancialExtraction(BaseModel):
    reasoning: str = Field(description="Step-by-step extraction logic")  # FIRST field
    vendor_name: str
    invoice_total: float = Field(gt=0)
    currency: str = Field(pattern=r'^[A-Z]{3}$')
    line_items: list[dict]

response = chat(
    model='qwen2.5:32b-instruct',
    messages=[{'role': 'user', 'content': document}],
    format=FinancialExtraction.model_json_schema(),
    options={'temperature': 0}
)
validated = FinancialExtraction.model_validate_json(response.message.content)
```

**Critical implementation details**: Temperature must be 0 for consistent schema adherence. The reasoning field should be first in the schema—research shows adding a chain-of-thought field boosted accuracy from 4.5% to 95% on structured tasks. Ollama's grammar doesn't inject schema into the prompt, so describe expected structure in your prompt text as well.

---

## Validate-retry loops outperform pure self-correction for smaller models

MIT research (arXiv:2310.01798) found that LLMs struggle with intrinsic self-correction without external feedback—sometimes degrading after self-correction attempts. However, **validate-retry loops with external feedback work excellently across all model sizes** because the bottleneck is error detection, not correction capability.

The **Instructor library** (3M+ monthly downloads) provides production-ready implementation for Ollama:

```python
import instructor
from pydantic import BaseModel

client = instructor.from_provider("ollama/qwen2.5:32b-instruct")

result = client.create(
    messages=[{"role": "user", "content": document_text}],
    response_model=FinancialExtraction,
    max_retries=3,
    timeout=60.0
)
```

Instructor automatically retries with specific validation error messages ("price: ensure this value is greater than 0"), which dramatically improves correction rates. Qwen2.5 uses TOOLS mode automatically for better function calling.

Interestingly, weaker models achieve **1.6-1.7× higher correction rates** than stronger models on some benchmarks—stronger models make fewer but "deeper" errors that resist correction, while smaller models make more surface-level errors that feedback easily fixes.

---

## Tree-of-Thought is overkill for extraction; iterative chunking wins

Research on Tree-of-Thought (NeurIPS 2023) found it excels at tasks requiring exploration and strategic lookahead—document extraction typically isn't such a task. ToT incurs significant compute overhead (multiple model calls per decision) with minimal benefit for structured extraction. **Self-consistency** (generate N extractions, pick the most common) provides similar benefits at lower cost.

For long documents, **iterative extraction with text splitting outperforms single-pass** according to research on information extraction quality (arXiv:2404.04068). The optimal approach:

1. Split document into distinct chunks
2. Extract sequentially with iterated LLM calls
3. Maintain extraction history from previous chunks (prevents duplicates)
4. Merge and deduplicate final results

This addresses both "Lost in the Middle" and output token limits while achieving more thorough extraction. The MINEA score (Multiple Infused Needle Extraction Accuracy) provides evaluation without labeled data using synthetic "needles" inserted into documents.

---

## Practical tool stack for your homelab deployment

**Immediate configuration changes** (highest impact):
- Increase Ollama `num_ctx` to 32K-65K
- Set `temperature=0` for extraction tasks
- Enable structured output via `format` parameter

**Recommended tool stack**:
- **Instructor** (pip install instructor): Pydantic validation with automatic retries
- **LlamaIndex** (llama-index-llms-ollama): RAG pipelines with semantic chunking
- **LlamaSherpa** (pip install llmsherpa): Layout-aware PDF chunking preserving tables

**For very long documents (>65K tokens)**:
- Implement Map-Reduce extraction pipeline
- Consider MIT RLM scaffold for documents exceeding practical context limits
- Use hierarchical summarization before extraction

**Financial document-specific recommendations**:
- Chunk at 1000-2000 tokens with 200-token overlap
- Extract tables separately from narrative (Qwen2.5 excels at table understanding)
- Use two-phase approach for complex documents: free reasoning → structured output

The FinBen benchmark (NeurIPS 2024) reveals a critical gap: **all LLMs struggle with fine-grained taxonomy alignment** in financial documents—FinTagging showed 0% precision/recall on full XBRL taxonomy matching. Focus extraction on high-level financial metrics rather than attempting exact taxonomy classification.

---

## Implementation checklist for immediate improvements

Start with these changes in priority order:

- **Configure context properly**: Create Modelfile with `num_ctx 65536` and rebuild model
- **Add schema enforcement**: Use Ollama's `format` parameter with Pydantic schemas  
- **Implement retry logic**: Add Instructor library with `max_retries=3`
- **Chunk long documents**: 1024 tokens with 15% overlap for financial content
- **Validate and repair**: Use Pydantic validation plus json_repair for edge cases

For documents consistently causing failures, the MIT RLM approach offers a sophisticated fallback—but the configuration and tooling changes above should resolve most "context rot" issues without requiring the complexity of a full RLM deployment.

The research consensus is clear: **iterative approaches with external validation outperform single-pass extraction**, constrained decoding is strictly better than unconstrained, and proper context configuration is foundational. Your Qwen2.5:32b on RTX 5090 has substantial capability—the challenge is ensuring that capability is properly accessed through correct configuration and extraction architecture.