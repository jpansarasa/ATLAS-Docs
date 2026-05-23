# Three-Hypotheses Scoping — DSL PoC Path Forward

**Date:** 2026-05-23
**Branch:** `chore/three-hypotheses-scoping`
**Status:** RECON + PLAN only — no implementation, no sweeps. User picks impl order.
**Source synthesis:** `/tmp/dsl-prior-art-review.md` (prior-art doc from agent a0c6a933c3cebf9e9)

## Context

v8 baseline (`prompts/cod_dsl_baseline.txt`, locked 2026-05-22) plateaued at
aggregate verifier **0.907** on Arm B n=9 and **0.7826** on the n=72 corpus
(qwen3:30b-a3b-instruct-2507-q4_K_M, llama-server v2.1 GBNF, `--num-ctx 32768`,
`--n-predict 8192`). Prompt-iteration has hit a ±0.03 noise floor.

Three architectural interventions are scoped here. The Path-C GPU pivot
(grammar-constrained AWQ on vLLM) already shipped (#418) — kind-flood was
**displaced** to `macro_indicator`, not eliminated. These three hypotheses target
the underlying mechanisms.

---

## A. Span-retrieval hints (extend existing CoVe substrate)

### What

For every numeric / quote slot the extraction LLM is about to emit, inject
*retrieved verbatim source spans* into the prompt — extending the existing
CoVe substring-grounding machinery (`CoveStatementVerifier`,
`CoveSymbolVerifier`) from post-hoc verification into in-context generation
hints. Same predicate, applied **before** the model speaks rather than after.

### Code locations

Substrate that exists today:

- `SentinelCollector/src/Services/CoveStatementVerifier.cs:50-300` — pure
  function, value/date/text grounding. Numeric tolerance ±0.5%; date forms
  ISO + long + quarter + prose. Length floor 4 chars for text.
- `SentinelCollector/src/Services/CoveSymbolVerifier.cs:40-163` — Symbol/Name
  word-boundary regex against `text_quote ∪ context_summary`.
- `SentinelCollector/src/Services/MergedExtractionService.cs:20-273` — v2
  pipeline (entity-anchored chunking + structured-output extraction). Wired
  to vLLM via `IOllamaClient.GenerateStructuredAsync`.
- `docs/benchmarks/cod-2026-05-17/scripts/run_cod_dsl.py:485-630` — DSL PoC
  runner (`run_cell`). Currently single-pass, no retrieval hints.
- `docs/benchmarks/cod-2026-05-17/scripts/retrieve_few_shot.py` — existing
  cosine-similarity retrieval (article-level via `sentence-transformers/all-MiniLM-L6-v2`).
  Closest precedent — wraps prompt with `<FEW_SHOT_EXAMPLES>` marker
  substitution (`prompts/cod_dsl_v6_rag.txt:613`). Note: v6+RAG REGRESSED
  (PR #408) — article-level retrieval injected noise. The span-level
  variant proposed here is finer-grained.
- `docs/benchmarks/cod-2026-05-17/prompts/cod_dsl_baseline.txt` (846 lines) —
  target for prompt-template diff.

### Implementation sketch

Two-script extension, no C# changes for the PoC:

1. **`scripts/retrieve_spans.py`** (new, ~200 LOC):
   - Pre-pass over `article_text`:
     - Extract numeric literals via the same regex CoveStatementVerifier
       uses (`NumericLiteralRegex` — port the pattern verbatim).
     - For each match, capture ±80-char window + position.
   - Bucket by surface form so the prompt block doesn't repeat.
   - Render as a `<SOURCE_NUMERIC_SPANS>` block:
     ```
     SOURCE_NUMERIC_SPANS (verbatim, copy from here):
       1.2 — "have eased to 1.2 hikes from 1.6 just last week"   @[412,415]
       1.6 — "have eased to 1.2 hikes from 1.6 just last week"   @[428,431]
       ...
     ```

2. **`scripts/prompt_assembly.py:assemble_prompt`** (existing) — add a
   `SOURCE_NUMERIC_SPANS` substitution marker. New prompt variant
   `cod_dsl_v12_span_hints.txt` (= v8 + decree "every NUM raw MUST appear in
   SOURCE_NUMERIC_SPANS block; if not, omit").

3. **`run_cod_dsl.py:run_cell`** — single conditional pass-through for
   `--variant v12_span_hints`. No new backend, no new flags.

Prompt template diff vs v8 (illustrative):
```diff
+SOURCE_NUMERIC_SPANS (copy-zone — every NUM raw value must appear here):
+<SOURCE_NUMERIC_SPANS>
+
 DECREES:
 8) DECLARE-ONCE — declare each _anon:<id> at most once.
+9) SPAN-MATCH — every NUM raw token must literally appear in
+   SOURCE_NUMERIC_SPANS above. If the value is not there, omit the NUM.
```

### Cost estimate

**2–3 dev days.**
- 0.5d: extract regex + window-builder script
- 1d: prompt template + assembly wiring
- 0.5d: n=9 smoke + grade
- 1d (buffer): debug retrieval-noise regression (v6+RAG burned 1 round)

### Risk

**MEDIUM.**
- ✓ Reuses validated CoVe predicates (low semantic risk).
- ✓ Pure prompt-augmentation, no infra change.
- ✗ Prompt-bloat risk on long articles — n=72 corpus has 5 cells >20K chars
  (id 27772 ~47KB, 32859 ~44KB, 35051 ~39KB). Span-blocks for those will
  inflate prompts; at n_predict=8192 + ctx=32768 we have ~24K input
  budget which should hold but needs verification.
- ✗ v6+RAG (article-level retrieval) regressed 0.215 vs v6 0.895 — the
  injection mechanism itself caused mode collapse. Span-level retrieval is
  more targeted, but the regression vector is real.

### Test plan

- **n=9 smoke** (Arm B cells): `variant=v12_span_hints`, baseline llama-server
  + v2.1 GBNF, model qwen3:30b-a3b. Pass gate: `verifier_pass_rate ≥ 0.907`
  AND `numeric_exact_match_rate > 0.5` (current baseline 0.0).
- **n=72 follow-up only if n=9 passes** — same gate at `≥ 0.85`.

### Dependencies

None. Pure script work. Can run on existing llama-server.

---

## B. Outlines FSM (stateful grammar at decode time)

### What

Move from `vLLM 0.21 structured_outputs.grammar` (xgrammar CFG backend,
stateless) to a stateful decoder that can enforce DECLARE-ONCE and per-kind
block caps (`ent_concept{0,5}`) at token-generation time, not post-hoc.
**Headline:** the recon below changes the calculus on this hypothesis.

### Code locations

- `docs/benchmarks/cod-2026-05-17/scripts/run_cod_dsl.py:382-470`
  (`_vllm_server_chat_completion`) — current vLLM grammar enforcement via
  `structured_outputs.grammar`. PR #417 documented this is the only
  decoder-enforced path in vLLM 0.19/0.21.
- `docs/benchmarks/cod-2026-05-17/scripts/run_cod_dsl.py:310-380`
  (`_llama_server_completion`) — alternative GBNF on llama-server.
- `/opt/ai-inference/compose.yaml` (vllm-server service) — would need restart
  with new image / launch flag if backend swap.
- `docs/benchmarks/cod-2026-05-17/dsl/` — current v2.1 grammar (GBNF). Would
  need full port to either xgrammar Lark/EBNF or Outlines regex-FSM.

### Recon findings (load-bearing — re-read before committing)

From `WebFetch docs.vllm.ai/en/stable/features/structured_outputs.html` and
`dottxt-ai.github.io/outlines/latest/` (2026-05-23):

1. **vLLM 0.21 supported backends are `xgrammar` and `guidance`** — Outlines
   is NOT a first-class vLLM backend. `lm-format-enforcer` is referenced but
   capability matrix is not documented.
2. **Outlines is a wrapper layer**, not a vLLM backend. It wraps vLLM (and
   transformers, llama.cpp, ollama, sglang, TGI, OpenAI, etc.). To use
   Outlines you'd run *outside* vLLM's structured-output path and lose the
   in-engine constraint optimizations.
3. **Outlines public API supports JSON Schema, regex, CFG.** **No documented
   support for stateful counting / declare-once / deduplication** in the
   current public API — same fundamental limitation as xgrammar.
4. **xgrammar EBNF/Lark** — same CFG-class expressiveness as our GBNF; no
   stateful counting either.

**Implication:** the "Outlines FSM" hypothesis in the prior-art doc
overstates Outlines' current capability. The actual stateful-decoding
options are:

- **Option B1:** xgrammar EBNF — port GBNF, no semantic upgrade. Pointless.
- **Option B2:** Custom Python LogitsProcessor in vLLM — write a stateful
  processor that tracks declared `_anon:` IDs and zeros their logits after
  first emission; also tracks per-kind block counts. vLLM's
  `logits_processors` parameter is supported. This is the **real
  stateful-decoding path**.
- **Option B3:** Move to Outlines standalone (drop vLLM structured-outputs)
  + write custom FSM with the same counting logic. Same logic as B2 but
  loses vLLM's in-engine throughput.

### Implementation sketch (Option B2 — recommended over Outlines)

```python
# scripts/dsl_stateful_processor.py (new, ~150 LOC)
class DeclareOnceProcessor:
    """Tracks _anon:<id> tokens seen during generation; zeros logits for
    the second emission of any ID. Per-kind block counter for ENT_concept,
    ENT_org, NUM, EVT, etc. with configurable caps."""
    def __init__(self, tokenizer, kind_caps: dict[str, int]):
        ...
    def __call__(self, token_ids: list[int], logits: torch.Tensor) -> torch.Tensor:
        # 1. Decode last N tokens to detect ENT/_anon declarations.
        # 2. Maintain set of declared ids; ban re-declaration tokens.
        # 3. Track kind counts; ban opening tokens once cap hit.
        return logits
```

Wire via vLLM's `SamplingParams(logits_processors=[...])`.

### Cost estimate

**Option B2: 2–3 weeks.**
- 1w: spike — verify vLLM 0.21 logits_processors API works with grammar
  backend simultaneously (open question — they may be mutually exclusive).
- 1w: implement + unit-test on a corpus of v8 outputs (replay).
- 0.5w: integrate into `run_cod_dsl.py --variant v12_stateful`.
- 0.5w: n=9 + n=72 sweep.

**Option B3 (Outlines standalone): 3–4 weeks** (rewrite vLLM call path,
lose in-engine optimization). Not recommended.

### Risk

**HIGH.**
- ✗ Unknown: whether vLLM's `structured_outputs.grammar` and
  `logits_processors` compose. If they don't, B2 collapses to "grammar OR
  stateful, not both" — losing the format-validity floor that the GBNF
  guarantees.
- ✗ vLLM throughput hit: every logits processor runs on every token, on
  CPU by default. On the n=72 corpus this is non-trivial.
- ✗ Tokenizer-boundary problem: `_anon:` is multi-token. Token-level
  bans require careful tokenizer-aware bookkeeping.
- ✓ If it works, it's the architecturally-correct fix and applies to all
  future stateful constraints (per-kind caps, max-EVT-per-ENT, etc.).

### Test plan

- **Spike (1 week):** dispatch a recon subagent to (a) confirm vLLM 0.21
  composes `structured_outputs.grammar` + `logits_processors`, (b) measure
  throughput hit on a 3-cell smoke. Pass gate: composes AND throughput hit
  <30%.
- **If spike passes:** n=9 smoke with explicit DECLARE-ONCE + per-kind cap.
  Pass gate: `dup_decl_rate = 0` AND `verifier_pass_rate ≥ 0.85`.
- **n=72 sweep** only after n=9 passes.

### Dependencies

- vLLM 0.21 on `vllm-server` (current). Possible upgrade to v0.22+ if
  logits_processor + grammar compatibility lands later.
- No infra changes if spike succeeds; new image / config if B3 forced.

---

## C. Chunked extraction (REINSTATE)

### What

Reinstate the chunked-extraction pattern that the C# production pipeline
already uses (`DocumentChunker` + `ChainOfVerification.ExtractChunkedAsync`
+ `ExtractionMerger.Merge`) into the DSL PoC runner. The PoC currently
runs **single-pass only** — long-output regressions are not a context-cap
issue (input fits) but an *output-token-budget exhaustion* issue (model
emits 100+ duplicate blocks → hits `n_predict=8192` mid-block).

### Git archaeology — WAS chunking dropped?

**Answer: No. Chunking is alive in C# production. The DSL PoC simply
never imported it.** Forensics:

| Commit / PR | What it did |
|---|---|
| `cc6e7a5b` PR #134 | Added structured output + validation retries + **initial chunking** |
| `30cd40bf` PR #126 | Collapsed `ExtractAndVerifyAsync` to single-pass (4 calls → 1). This is the "collapse" event but it dropped CoVe loops, **not chunking** (chunking arrived later in #134) |
| `ecf8b87e` PR #161 | Extended chunked extraction for large documents — current `DocumentChunker.cs`, `ChainOfVerification.ExtractChunkedAsync` |
| `8c03ccbf` PR #162 | Dual-backend GPU/CPU routing — preserved chunking |
| `1aca5e16` PR #171 | Persistent-kernel sandbox + tool-augmented — preserved chunking |

`git log --all --diff-filter=D -- '*chunk*'` returns **nothing reverting
chunking**. Current `main` has `DocumentChunker.cs`, `ExtractionMerger.cs`,
`ChainOfVerification.ExtractChunkedAsync`, `ChainOfDensity.DensifyChunkedAsync`,
and `MergedExtractionService.RunChunkedAsync` — all live, all wired.

The DSL PoC `docs/benchmarks/cod-2026-05-17/scripts/run_cod_dsl.py` was
greenfield single-pass from day one (Round-2 commit `539769ac` 2026-05-18).
The Round-2 REPORT, Round-3, Round-4, and Phase-3 docs make **no reference
to chunking** (`grep -rn "chunk\|split" docs/benchmarks/cod-2026-05-17/`
returns zero hits). It was never tried in the PoC.

**User correction is correct:** "the chunk CoT was something we were
already doing that did prove useful" — refers to the C# production pipeline
(Sentinel Stage 2: direct-vs-chunked CoD, documented in
`docs/sentinel-extraction-pipeline.md:17-30`). Reinstating it for the DSL
PoC is **cherry-pick + adapt**, not greenfield.

### Code locations (cherry-pick targets)

- `SentinelCollector/src/Extraction/DocumentChunker.cs` (135 LOC) — pure
  static, recursive splitter (paragraph → newline → period → word). 4
  chars/token estimator. Overlap percentage support. **Direct port to Python.**
- `SentinelCollector/src/Extraction/ChainOfVerification.cs:49-115`
  (`ExtractChunkedAsync`) — orchestration: chunk → per-chunk extract →
  `ExtractionMerger.Merge`. **Port to** `run_cod_dsl.py`.
- `SentinelCollector/src/Extraction/ExtractionMerger.cs` (52 LOC) —
  description + value + period key dedup, keep highest confidence. **Port
  to Python; adapt key tuple to DSL slot shape (ENT id / NUM raw / EVT
  trigger).**
- `SentinelCollector/src/Services/MergedExtractionService.cs:164-274`
  (`RunChunkedAsync`) — production-grade pattern: dedup by
  `(subject, desc, value, periodStart)` tuple, chunk-budget halving on
  `PromptTooLargeException`. Reference for retry policy.
- `SentinelCollector/src/Configuration/ExtractionOptions.cs:130-175` —
  current defaults: chunk threshold 15K tokens / 6K-token CoD chunks /
  15% overlap. Carry over verbatim.

### Implementation sketch

Two-file change:

1. **`docs/benchmarks/cod-2026-05-17/scripts/dsl_chunker.py`** (new, ~200 LOC):
   - Direct Python port of `DocumentChunker.Chunk` (paragraph → newline →
     period → word recursive splitter, overlap %).
   - `DslMerger.merge_dsl_blocks(per_chunk_outputs: list[str]) -> str`:
     parse each chunk output via existing `dsl.parser.parse_text_v2_1`,
     dedup ENT/NUM/EVT blocks by their declared id (or trigger for EVT),
     re-emit as a single v2.1 document. Reuses the existing parser — no
     re-implementation.

2. **`run_cod_dsl.py:run_cell`** — branch on
   `len(article_text) > chunk_threshold_chars` (default 32K chars ≈ 8K
   tokens, mirroring `CodChunkingThresholdTokens=8000`):
   - Single-pass: today's flow, unchanged.
   - Chunked: split → per-chunk `_llama_server_completion` / `_vllm_*` →
     `DslMerger.merge_dsl_blocks` → scored as one record.

Critical adaptation vs C#: the C# pipeline merges *parsed observation
records*. The DSL PoC needs to merge *raw v2.1 DSL text* — i.e. parse-merge-
re-emit. The existing `dsl.parser` (`docs/benchmarks/cod-2026-05-17/dsl/`)
already round-trips, so this is mechanical.

### Cost estimate

**3–5 dev days.**
- 1d: port DocumentChunker to Python (mechanical).
- 1–2d: DSL merger (parse + dedup + re-emit) — most novel piece.
- 0.5d: integrate `run_cell` branch.
- 0.5d: n=9 smoke on long-document cells (id 27772/32859/35051/38048/36065).
- 1d (buffer): tuning chunk-size / overlap if dedup loses signal.

### Risk

**LOW.**
- ✓ Pattern is battle-tested in C# production.
- ✓ Cell 33632 (the canonical hard case) is only ~9K chars — chunking
  alone doesn't help that one. But the n=72 long-document cells (27772 ~47KB,
  32859 ~44KB, 35051 ~39KB) DO benefit directly.
- ✓ User explicitly endorsed this path.
- ⚠ Dedup-key choice (id vs surface form) needs care: `_anon:` ids
  collide across chunks. Mitigation: namespace per-chunk during merge
  (`_anon:chunk1:oil_prices` → re-renumber post-merge).
- ⚠ Output truncation at `n_predict=8192` still applies per-chunk; if a
  single chunk emits 8K+ tokens (looping case) chunking doesn't prevent it.
  Pair with hypothesis B for full coverage.

### Test plan

- **n=9 smoke** including the 5 long-document cells. Pass gate:
  `verifier_pass_rate ≥ 0.85` AND no NEW regression on short-document cells
  (≥ baseline `0.907 ± 0.03`).
- **n=72 follow-up** if n=9 passes. Pass gate: `≥ 0.85` aggregate (vs v8
  baseline 0.7826 — should improve materially on the long-doc tail).

### Dependencies

None. Pure Python work. Existing llama-server / vLLM backend.

---

## Recommended sequence

### Dispatch order

1. **C (Chunked extraction) FIRST** — LOW risk, 3–5 days, user-endorsed,
   directly addresses the n=72 long-document tail (5 cells >20K chars).
   Reinstates known-good pattern. No infra change.

2. **A (Span-retrieval hints) SECOND, in parallel with C** — MEDIUM risk,
   2–3 days, addresses the `numeric_exact_match_rate=0.0` failure that
   neither chunking nor stateful-decoding solves. Different worktree, no
   shared code with C, fully parallelizable.

3. **B (Stateful decoding) THIRD, gated on spike** — HIGH risk, 2–3 weeks
   real cost. Before committing 2–3 weeks, run a 1-week **spike** to
   confirm vLLM 0.21 composes `structured_outputs.grammar` +
   `logits_processors`. If spike fails, the cost balloons to 3–4 weeks
   (B3 standalone Outlines path). Do this LAST so A+C's gains are banked
   first.

### Parallel-dispatch plan

| Step | Worktree | Concern | Can parallel with |
|---|---|---|---|
| C-impl | `epic/chunked-dsl` | `scripts/dsl_chunker.py`, `run_cod_dsl.py` | A-impl |
| A-impl | `epic/span-hints` | `scripts/retrieve_spans.py`, new prompt | C-impl |
| B-spike | `epic/stateful-spike` | `scripts/dsl_stateful_processor.py` smoke | A, C |
| B-impl | `epic/stateful-impl` | (gated on B-spike) | nothing — touches vLLM call path |

A and C can run as parallel subagents (per `feedback_parallel_agents_use_worktrees`).
B-spike can run in parallel with A/C since it only smoke-tests vLLM
behavior, no production code change.

### Cost / ROI summary

| Hypothesis | Cost | Risk | Hits |
|---|---|---|---|
| **A. Span-retrieval hints** | 2–3d | MEDIUM | `numeric_exact_match_rate` 0.0 → 0.5+ |
| **B. Stateful decoding (vLLM logits_processors)** | 1w spike + 2–3w impl | HIGH | DECLARE-ONCE eliminated, per-kind caps enforced |
| **C. Chunked extraction (reinstate)** | 3–5d | LOW | Long-doc n=72 cells (5 of 72) |

A + C combined: **~1 dev week**, both LOW–MEDIUM risk, addresses the two
distinct failure modes that v8 plateaued on (numeric grounding + long-doc).
B is the architecturally-correct fix but the cost ratio is 5–10× and the
spike has a real chance of falsifying the approach.

---

## Open questions for user

1. **Confirm A+C dispatch order vs B-first.** Plan recommends A+C first
   (parallel) because B has a 1-week spike with a real chance of
   falsification, and A+C address orthogonal failure modes. If user
   prefers to *prove out the architectural fix first* (B-spike) before
   investing in A+C, that's a legitimate alternative — but it gates 1
   week of dispatch budget on a HIGH-risk outcome.

2. **B-spike scope:** is a 1-week spike (compose-check + 3-cell smoke)
   the right granularity, or should we defer B entirely until A+C land
   and the noise floor is re-measured?

3. **C dedup-key policy:** when merging per-chunk DSL, the C# pipeline
   keys on `(subject, desc, value, periodStart)`. For DSL the natural key
   is the declared id (`ENT TerawulfInc`, `NUM rate_hike_1_2`). But
   `_anon:` ids collide across chunks. Should the merger (a) namespace
   per-chunk then rewrite anon refs post-merge, or (b) reject `_anon:`
   from cross-chunk merge entirely (each chunk's anons are local)? (a)
   is more work; (b) is simpler but may lose cross-chunk references.

---

## STATE.md one-liner

DSL PoC scoping: A (span hints, 2-3d, MED) + C (chunked extraction reinstate, 3-5d, LOW) recommended first in parallel worktrees; B (vLLM stateful logits_processors, 2-3wk, HIGH) gated on 1wk spike — Outlines is NOT a vLLM backend per recon, real path is custom LogitsProcessor.
