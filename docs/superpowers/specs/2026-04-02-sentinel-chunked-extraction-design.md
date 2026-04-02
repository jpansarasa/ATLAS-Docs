# Sentinel Chunked Extraction for Large Documents

## Problem

Sentinel's extraction pipeline fails on large documents (e.g., 754KB liveblog HTML) because both Chain of Density (CoD) and Chain of Verification (CoVe) send full document content to llama-server in a single request. Documents exceeding the 32K context window cause HTTP 400 responses, which the system misclassifies as transient errors, retrying 3 times and amplifying alert noise.

## Solution

Apply chunked processing to both CoD and CoVe pipelines with techniques from long-context extraction research. Enable the existing (dormant) CoVe chunking infrastructure and build a new hierarchical CoD path that preserves the iterative densification that makes CoD effective.

## Design

### 1. Chunked Chain of Density

**Current behavior:** `ChainOfDensity.DensifyAsync()` runs 5 iterations over the full content. Large documents exceed context window and fail.

**New behavior:** When content exceeds a token threshold, CoD splits into two phases:

**Phase 1 — Per-chunk densification:**
- Chunk the content using `DocumentChunker` with CoD-specific parameters (large chunks, no overlap)
- Run full 5-iteration CoD on each chunk independently
- Each chunk produces one maximally dense 80-word summary

**Phase 2 — Coalescing densification:**
- Concatenate all chunk summaries into a single document (~80 words x n_chunks)
- Run another 5-iteration CoD pass on the concatenated summaries
- Produces the final 80-word summary for the whole document

**Flow:**

```
EstimateTokens(content) > CodChunkingThresholdTokens?
    |                          |
    No                        Yes
    |                          |
    v                          v
  Standard 5-iteration      Chunk (large, no overlap)
  CoD (unchanged)              |
                               v
                         Per-chunk: 5-iteration CoD
                         -> 80-word dense summary per chunk
                               |
                               v
                         Concatenate chunk summaries
                               |
                               v
                         Coalescing: 5-iteration CoD
                         -> final 80-word summary
```

**Epistemic markers** are extracted once from the final coalesced summary, same as today.

**Total LLM calls for chunked path:** `5 * n_chunks + 5` (+ 1 for epistemic markers). For a 10-chunk document: 56 calls.

**Rationale:** CoD's value comes from successive refinement, not from seeing the whole document in one pass. A single-pass summary per chunk would lose the density gain. Running full CoD per chunk then coalescing preserves extraction quality.

### 2. Enable CoVe Chunking

**Current behavior:** `EnableChunking = false` in `ExtractionOptions`. The chunking code in `ChainOfVerification` exists but is disabled.

**Changes:**
- Set `EnableChunking = true` by default
- Keep existing defaults: 1024-token chunks, 15% overlap, 24000-token threshold

**Existing infrastructure that requires no changes:**
- `DocumentChunker.Chunk()` — recursive character splitting with configurable separators
- `ExtractionMerger.Merge()` — deduplication by (description, value, period), keeps highest confidence
- `ChainOfVerification.ExtractChunkedAsync()` — per-chunk extraction with merge
- `ChunkExtractionCounter` metric — already instrumented

**Effect:** Documents under 24K tokens (~96KB of text) are completely unaffected. Larger documents get chunked, extracted per-chunk, and deduplicated automatically.

### 3. Separate Chunk Configuration for CoD and CoVe

CoD and CoVe have fundamentally different chunking needs:
- **CoVe** needs small chunks with overlap to avoid missing observations that span boundaries
- **CoD** is compressing — overlap creates redundancy that the coalescing pass must deduplicate

**New configuration fields in `ExtractionOptions`:**

| Field | Default | Purpose |
|-------|---------|---------|
| `CodChunkingThresholdTokens` | 8000 | Token count that triggers chunked CoD |
| `CodChunkSizeTokens` | 6000 | Target chunk size for CoD (large) |
| `CodChunkOverlapPercent` | 0 | No overlap for CoD |

**Existing CoVe configuration (unchanged):**

| Field | Default | Purpose |
|-------|---------|---------|
| `ChunkingThresholdTokens` | 24000 | Token count that triggers chunked CoVe |
| `ChunkSizeTokens` | 1024 | Target chunk size for CoVe (small) |
| `ChunkOverlapPercent` | 15 | Overlap for boundary coverage |

CoD threshold is lower than CoVe because CoD sends the full content on every iteration (5x), so it needs more headroom below the context window.

### 4. HTTP 400 Classification Fix

**Current behavior:** `ExtractionProcessor` classifies all `HttpRequestException` as transient, retrying 3 times.

**New behavior:** Check the HTTP status code:
- **4xx (client errors)** — permanent. The request is malformed or content is unparseable. Retrying won't help.
- **5xx or null (server errors, network errors)** — transient. Retry up to 3 times.

**Effect:** Poison pill items fail once instead of 4 times, reducing alert noise by ~75% for this failure class.

### 5. New Prompt: Coalescing Densification

A new prompt variant for `IDensityPromptProvider`:
- `GetCoalescingPrompt(source, chunkSummaries, targetWordCount)` — instructs the model that the input is a set of dense summaries from consecutive sections of a single document, and to synthesize them into one coherent summary
- Used as the initial prompt (iteration 1) for Phase 2 of chunked CoD
- Subsequent iterations (2-5) in Phase 2 use the existing `GetDensificationPrompt` with the concatenated chunk summaries as the "original content" parameter

### 6. Observability

**New:**
- Counter `sentinel_cod_chunked_total` — tracks when CoD takes the chunked path, with chunk count as a tag
- Log (Information) when entering chunked CoD: document token estimate, chunk count
- Log (Information) coalesced input size before final CoD pass

**Existing (already covers the rest):**
- `ChunkExtractionCounter` — CoVe per-chunk tracking
- `ExtractionSuccessCounter` / `ExtractionErrorCounter` — outcome tracking
- `OllamaLatencyHistogram` — per-call latency
- `ExtractionDurationHistogram` — end-to-end timing

No new dashboard panels. Existing SentinelCollector dashboard already shows extraction success rate, duration heatmap, and chunk extraction rate.

## What Doesn't Change

- **ContentNormalizer / Markitdown** — HTML-to-markdown conversion untouched
- **ExtractionService orchestration** — CoD and CoVe still run in parallel. Chunking decisions are internal to each pipeline.
- **Existing prompts** — extraction and density prompts unchanged. One new coalescing prompt added.
- **DocumentChunker algorithm** — reused as-is with different parameters per pipeline
- **ExtractionMerger** — reused as-is for CoVe deduplication
- **Database schema** — no changes. `RawContent.ContextSummary` stores the final 80-word summary regardless of production method.
- **Downstream systems** — SecMaster resolution, event publishing, review queue all unaffected
- **Documents under threshold** — identical behavior to today

## Files to Modify

| File | Change |
|------|--------|
| `src/Extraction/ChainOfDensity.cs` | Add chunked path with per-chunk CoD + coalescing CoD |
| `src/Extraction/IDensityPromptProvider.cs` | Add `GetCoalescingPrompt` method |
| `src/Extraction/FileBasedDensityPromptProvider.cs` | Implement coalescing prompt loading |
| `src/prompts/cod_coalescing.txt` | New prompt template for coalescing phase |
| `src/Configuration/ExtractionOptions.cs` | Add CoD-specific chunking config fields |
| `src/Workers/ExtractionProcessor.cs` | Fix HTTP 400 transient/permanent classification |
| `appsettings.json` | Enable chunking, set CoD chunk defaults |
| `src/Metrics/SentinelMeter.cs` | Add `CodChunkedCounter` |

## Decisions Considered and Deferred

- **Weighted processing queue** (small items first, large items when idle): Not needed with `MaxConcurrentExtractions = 1`. Could revisit if large docs cause noticeable delays.
- **Full RLM scaffold** (MIT recursive language model with Python REPL): Most powerful approach per research, but overkill for current failure mode. Chunked CoD gets 80% of the benefit at 20% of the complexity. Candidate for future iteration.
- **Content truncation**: Simpler but loses information. Not aligned with goal of accurate extraction from large documents.
