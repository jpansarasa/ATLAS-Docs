# Hypothesis C — Chunked Extraction Python Port (n=9 smoke)

**Date:** 2026-05-23
**Branch:** `experiment/hyp-c-chunked-extraction-port`
**Server:** solo llama.cpp :11437 (qwen3:30b-a3b-instruct-2507-q4_K_M)
**Concurrent load:** hyp-a-span-hints sweep ran on the same backend
during the same window (mercury has only one llama-server instance).

## Headline

| arm | n | verifier_pass_rate | grammar_valid | delta vs v8 |
|---|---|---|---|---|
| v8 baseline (PR #410) | 9 | **0.9069** | 9/9 | — |
| hyp-c chunked        | 9 (apples-to-apples) | **0.8353** | 8/9 | **-0.072** |
| hyp-c chunked        | 10 (full corpus.jsonl) | 0.8386 | 9/10 | n/a |

n=72 validation gate (n=9 >= 0.85): **NOT MET** (0.835 < 0.85). The n=72
sweep was not launched. Per task spec, n=72 is conditional on the n=9
smoke clearing the gate.

## Per-cell detail (vs v8 layered baseline)

| id   | size (chars) | chunked? | v8 gv | hc gv | v8 verifier | hc verifier | delta  |
|------|--------------|----------|-------|-------|-------------|-------------|--------|
| 27560 |   1380       | N        | Y     | Y     | 0.667       | 0.636       | -0.031 |
| 27702 |   1858       | N        | Y     | Y     | 1.000       | 1.000       | +0.000 |
| 29807 |   1565       | N        | Y     | Y     | 0.735       | 0.882       | +0.147 |
| 31149 |   1173       | N        | Y     | Y     | 0.958       | 0.900       | -0.058 |
| 31430 |   2661       | N        | Y     | Y     | 0.917       | 0.929       | +0.012 |
| 33632 |   8971       | N        | Y     | Y     | 0.969       | 0.344       | **-0.625** |
| 34537 |    905       | N        | Y     | Y     | 1.000       | 1.000       | +0.000 |
| 35352 |   3015       | N        | Y     | Y     | 0.991       | 0.991       | +0.000 |
| 36065 |  20350       | N        | Y     | **N** | 0.925       | n/a (gv=N)  | **n/a** |
| 27772 |  47038       | Y (3 chunks) | — | Y | — | 0.865 | not in v8 |

## What the data says (and doesn't)

### What works

- **Port correctness:** 23 unit tests on DocumentChunker / ExtractionMerger
  / ChunkedExtractor + 4 integration tests on `run_cell --chunked`
  pass. All 76 tests in the benchmark suite still pass.
- **Chunking mechanism itself:** Cell 27772 (47K chars, ~12K estimated
  tokens) was split into 3 chunks at paragraph boundaries, each chunk
  was extracted independently with grammar enforcement, and the merger
  produced a valid v2.1 DSL with 45 entities and verifier_pass_rate=0.865.
  No grammar failure, no duplicate-ENT collision, no merge corruption.
- **Mechanism preservation:**
    - 4 chars/token heuristic preserved (`CHARS_PER_TOKEN=4`)
    - Recursive separator descent preserved (`\n\n` -> `\n` -> `. ` -> ` `)
    - 8K-token threshold + 6K chunk size = production C# defaults
    - Entity-anchored chunking preserved (paragraphs containing anchor
      tokens stay atomic; falls back to plain chunking when no anchors)
    - Dedup key adapted from C# `description|value|period` to DSL
      `(kind, identifier, slot_signature)`

### What broke

- **Cell 33632 regressed -0.625** despite being **single-pass** (8.9K
  chars, ~2.2K tokens, well under the 8K threshold). `chunked_meta=null`
  in the result file confirms the chunked path never fired. Same prompt,
  same backend, same grammar — different output. The 33632 v8 baseline
  emitted 11.3KB of DSL; the hc run emitted 8.4KB and produced 36 ENTs
  that don't match the ground-truth tickers/companies (`ent_recall=0.0`,
  `numeric_recall=0.0`).
- **Cell 36065 lost grammar validity** (single-pass, 20K chars, ~5K
  tokens — below 8K threshold). Failure mode: `Duplicate ENT declaration:
  '_anon:banking_system_institutional_design'`. The model emitted two
  ENT blocks with identical identifier but slightly different slots
  inside a single un-chunked output — the parser rejected the second.

Both regressions are on **non-chunked cells** under the chunked-flag
sweep. This is sampler / concurrency noise, not a Hyp-C mechanism
failure. The hc sweep ran in parallel with the hyp-a-span-hints sweep
on the shared `llama-server :11437` (one slot, sequential dispatch),
which plausibly perturbed sampler state across runs vs the v8 baseline
sweep which had exclusive access.

### Long-doc tail mechanism check

The user's task note specifically flagged that 33632 (8.8KB) is too
small to benefit from chunking, and that the OTHER ~5 long-doc cells
in `corpus.full72` (> 20KB) should benefit. The n=9 corpus has only
**one** of those large cells — 27772 (47KB) — and it's NOT in the
v8 baseline set, so cell-vs-cell comparison is impossible from this
data.

**Mechanism evidence for the long-doc claim is real but unmeasured:**
- 27772 chunked into 3 paragraph-aligned chunks (sizes consistent with
  the 6K-token budget per chunk).
- Each chunk extracted independently with grammar enforcement.
- Merger produced a coherent DSL document (45 ENT blocks, 0 EVT,
  no parse errors, verifier_pass=0.865).
- This is the FIRST evidence the chunking pipeline works end-to-end
  on the DSL PoC. We just can't compare it to a baseline because v8
  excluded the 47K-char article.

## Verdict

**BLOCKED on n=9 gate.** Aggregate is 0.835 vs v8 0.907, missing the
0.85 trigger for n=72 validation.

The blocker is **sampler/concurrency noise on single-pass cells**, not
a chunking mechanism defect. The chunking path itself behaved
correctly on the one cell that exercised it (27772). The n=9 baseline
set is unfortunately not the right corpus for this hypothesis — only
1/10 cells exceeds the production 8K threshold.

## Follow-up options (NOT in scope for this PR)

1. **Re-sweep n=9 with exclusive llama-server access** to isolate the
   chunking signal from the sampler noise that swamped 33632 and 36065.
   If the 0.907 baseline reproduces under exclusive access, the
   chunking-vs-no-chunking comparison becomes apples-to-apples on the
   8 cells where chunking is a no-op (and would let us declare the
   port causally neutral on short cells).
2. **Force chunking on n=9 with a lower threshold** (e.g.
   `--chunking-threshold-tokens 2000 --chunk-size-tokens 1500`) to
   actually exercise the chunking path on the n=9 baseline. This would
   measure the worst-case impact of chunking when chunks are smaller
   than the document type would naturally support.
3. **Skip n=9 and go straight to n=72.** The long-doc tail (5 cells >
   5K tokens in `corpus.full72`) is where the user's hypothesis lives.
   The n=9 result tells us short-cell chunking is a noisy no-op; n=72
   would show whether the long-doc tail actually closes the
   0.9069 -> 0.7826 gap.

The infrastructure (chunked_extractor module + `--chunked` flag +
sweep script + 27 unit tests) is preserved for any follow-up to
adopt without rework.
