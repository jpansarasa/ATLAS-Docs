# Token-Level Source Grounding — v2.2 Schema Recon

**Date:** 2026-05-23
**Branch:** `chore/token-grounding-recon`
**Status:** RECON + PLAN only. No schema changes, no verifier changes. Doc-only PR.
**Source synthesis:** `/tmp/dsl-prior-art-review.md`, `docs/plans/three-hypotheses-scoping.md`,
`docs/benchmarks/cod-2026-05-17/dsl/{verifier.py,grammar-v2_1.lark}`,
`docs/grammars/cod-dsl-v2.1.gbnf`,
`docs/benchmarks/cod-2026-05-17/scripts/{run_cod_dsl.py,score_dsl.py}`.

## Reframe of v2 → v2.1 history

v2 schema asked for `source_span: [start, end]` (codepoint offsets).
`verbatim_match_rate = 0.0` across ALL backends — qwen3:30b-a3b MoE AND
qwen2.5:32b dense. Prior-art review (`/tmp/dsl-prior-art-review.md` Part 4 §1)
attributes this to **BPE tokenization breaking byte-level span grounding**
(Olsson induction heads, arxiv:2209.11895): copy-mode circuits operate on
token boundaries, not character boundaries. v2.1 dropped `source_span` to
OPTIONAL; verifier falls back to `input_text.find(value)`.

**User directive (2026-05-23):** *"If character counting gets in the way
because the LLM thinks in tokens, let's have it return the tokens. The tokens
are more important than the character count."*

v2.2 takes the directive literally: grounding unit = LLM's native unit (BPE
tokens), verifier validates via tokenizer-consistent subsequence match.

---

## 1. Schema design sketch

### Two variants

**v2.2a — `source_token_text`** (RECOMMENDED)
The model emits a list of verbatim token *strings* it copied from the article.
Verifier tokenizes the article with the SAME tokenizer the model used, then
checks the emitted strings appear as a contiguous sub-sequence in the
tokenized article.

```
NUM rate_hike_1_2
  - raw: "1.2 percentage points"
  - source_tokens: ["1", ".", "2", " percentage", " points"]
  - value: 1.2
  - unit: pct_points
```

Why preferred:
- **Natural for the model** — Qwen3 already emits these tokens during gen;
  we're asking for them in a structured slot. No new capability needed.
- **Human-readable** — diffs against the article are obvious.
- **Survives tokenizer drift** — if we swap models, strings still point at
  the same article substring.

**v2.2b — `source_token_ids`** (NOT recommended)
Model emits integer token IDs from tokenizer vocab (`[16, 13, 17, ...]`).
Rejected: unnatural to prompt (model can't introspect vocab IDs reliably —
induction heads operate on activations, not ID introspection); brittle
across model swap (qwen3 IDs ≠ qwen2.5 IDs); zero upside over 2.2a since
verifier work is identical either way.

### GBNF grammar additions (variant 2.2a)

Add ONE production per copy-bearing block (ENT / NUM / EVT), OPTIONAL like
v2.1 spans:

```gbnf
# ENT — copy zone gains optional source-tokens-slot
ent-block ::= "ENT " ent-id ent-alias-clause? " / " ent-type qualifiers? "\n" \
              ent-name-slot source-tokens-slot? source-span-slot? ent-derived-slot*

source-tokens-slot ::= "  - source_tokens: [" tok-string ("," ws tok-string){0,127} "]\n"
tok-string         ::= "\"" string-content "\""
```

Same shape for NUM (`raw` + optional `source_tokens` + optional
`context_tokens`) and EVT (`trigger` + optional `trigger_tokens`). v2.1's
optional `source_span` stays, so v2.2 is a strict superset: any v2.1 doc
parses as v2.2 with `source_tokens` omitted.

Lark side: mirrors GBNF — add `source_tokens_slot` to each copy-zone rule
+ one new SLOT_LINE terminal `SLOT_LINE_SOURCE_TOKENS`.

`{0,127}` token-list cap per slot (longest NUM raw in corpus tokenizes
to ~12-15 BPE tokens; 127 matches existing rep-bound convention).

---

## 2. Verifier changes

### Tokenizer choice

llama-server runs `qwen3-30b-a3b-instruct-2507-q4_K_M.gguf` (per
`/opt/ai-inference/compose.yaml:90`). Qwen3 uses a **tiktoken-style BPE
tokenizer** (vocab ~151K, same family as Qwen2.5). Two Python access paths:

1. **`tokenizers` (HF Rust)** — load
   `Qwen/Qwen3-30B-A3B-Instruct-2507`'s `tokenizer.json`. Fast, exact
   match to what llama.cpp uses (llama.cpp embeds the GGUF tokenizer
   verbatim from the same source).
2. **`transformers` AutoTokenizer** — heavier dep but identical output.

Verifier picks (1). The tokenizer file is ~7MB; cache once at import time.

### New verifier pipeline (per NUM/ENT/EVT block)

```python
def _verify_num_v22(num, input_text, tokenizer):
    # Existing: raw / source_span check (v2.1 fallback path unchanged)
    # NEW: if num.source_tokens is set, run subsequence check
    if num.source_tokens is None:
        return _v21_path(num, input_text)  # unchanged
    article_token_strs = [tokenizer.decode([tid]) for tid in tokenizer.encode(input_text)]
    needle = num.source_tokens  # list[str]
    if _is_contiguous_subsequence(needle, article_token_strs):
        return BlockVerdict("pass")
    return BlockVerdict("hard_fail", reason="source_tokens_not_contiguous")
```

**`_is_contiguous_subsequence`** = O(n*m) naive scan; for n=72 longest
article (47KB ≈ 12K tokens) × 15-token needle × 100 blocks/doc = ~18M
comparisons per doc, <100ms. Fine.

### New metric: `token_verbatim_match_rate`

Analog of byte-level `_verbatim_match_rate` in `score_dsl.py:647`:

```python
def _token_verbatim_match_rate(doc, article_text, tokenizer):
    # Denominator: copy slots with non-empty source_tokens
    # Numerator: those that match contiguously in tokenized article
```

Reports alongside (not replacing) existing `verbatim_match_rate` — both
together let us A/B the two grounding modes on the same run.

### Verifier rewrite cost

- 0.5d — HF `tokenizers` dep + cache (~50 LOC, single import-time load)
- 0.5d — `_is_contiguous_subsequence` + tests, `_token_verbatim_match_rate`
- 0.5d — `_verify_*` extensions; preserve v2.1 fallback for mixed docs
- 0.5d — score_dsl.py wiring + smoke on 3-cell n=3

**Subtotal: ~2 dev days.**

---

## 3. Runner changes

### llama-server `/completion`

Per llama.cpp `tools/server/README.md`: accepts `"return_tokens": true` and
returns `tokens: [int, ...]`; per-token breakdown via `completion_probabilities`
when `n_probs > 0`. Returned IDs match the embedded GGUF tokenizer's IDs.
Verifier doesn't strictly need them (it re-tokenizes the article
independently), but capturing is cheap drift insurance.

### vLLM `/v1/chat/completions`

`logprobs: 0` (or `top_logprobs: 1`) returns per-token IDs + log-probs in
`choices[0].logprobs.content[].token` (string) and `.bytes`. Same drift-
insurance value; no production config change.

### Runner additions

Minimal — one optional flag, default off:

```python
# run_cod_dsl.py:_llama_server_completion
if capture_tokens:
    payload["return_tokens"] = True
# ... in return dict:
"output_tokens": body.get("tokens"),  # list[int] or None
```

Persisted to the per-cell JSON output alongside `content`. Verifier ignores
it for v2.2a (re-tokenizes independently); used only for drift detection.

**Runner cost: 0.5d** for both backends + JSON-schema bump on the per-cell
record. Total back-end work ~2.5d.

---

## 4. Cost + risk estimate

### Total implementation cost (v2.2a, full path)

| Workstream | Days |
|---|---|
| GBNF grammar v2.2 + Lark grammar | 0.5 |
| Parser extensions (v2_2 module mirroring v2_1) | 0.5 |
| Tokenizer integration + verifier rewrite | 2.0 |
| Runner token-capture + JSON schema bump | 0.5 |
| Prompt template v13 with worked example | 1.0 |
| n=9 smoke + grade + 1d debug buffer | 2.0 |
| n=72 sweep (gated on n=9 pass) | 1.0 |
| **Subtotal** | **7.5 dev days (~1.5 wks)** |

### Risk

**Evidence the model emits verbatim tokens reliably:** YES, indirect.
- v8 baseline `verbatim_match_rate=0.989` on Hyp A v12 healthy cells —
  the `name:` slot copy (verbatim string emission, no offset) succeeds
  ~99% of the time. Since BPE copy IS the model's native operation
  (induction heads, Olsson 2022), asking for the token list explicitly
  is asking the model to *report* what it just did, not to *compute*
  something new.
- The byte-offset failure (`verbatim_match_rate=0.0` on v2) was at the
  arithmetic level (count bytes), not at the token-emission level. v2.2a
  removes the arithmetic ask.

**Residual risks:**
1. **Token-boundary surprise on user-facing tokens.** Qwen3 BPE encodes
   leading-space-as-prefix (`" percentage"` is ONE token, not `[" ",
   "percentage"]`). Worked examples must train the model on this convention
   or the emitted list will mis-tokenize.
   *Mitigation:* prompt includes 2-3 worked examples with explicit
   leading-space tokens; verifier accepts a `whitespace_normalize=True`
   mode as a fallback.
2. **Tokenizer-version drift llama.cpp ↔ HF.** llama.cpp ships
   GGUF-embedded tokenizer; HF ships `tokenizer.json`. Both come from the
   same Qwen3 source, but there's a non-zero chance of edge-case
   divergence (BPE merges, byte-fallback).
   *Mitigation:* runner captures output tokens; verifier asserts
   `tokenizer.encode(output_text) == captured_tokens` as a sanity gate.
3. **Prompt-bloat.** A token list is verbose: 15 tokens × ~10 chars/token
   = 150 chars per NUM, vs. `[start, end]` ~ 12 chars. On 100-block docs
   this is +14KB of output tokens. At `n_predict=8192` cap this is real
   pressure.
   *Mitigation:* could move to comma-joined string `"1|.|2| percentage| points"`
   (~50% smaller) if budget pressure shows up in n=9 smoke. Defer pending
   evidence.

### Comparison vs A+C compound (Path β)

A+C compound per `docs/plans/three-hypotheses-scoping.md`:
- A (span-retrieval hints) = 2-3d, MEDIUM risk
- C (chunked extraction reinstate) = 3-5d, LOW risk
- Combined wall: ~3-4h (parallel dispatch, both workstreams shippable
  same day per scoping doc)
- Combined dev: ~1 dev week

A+C **does not address root cause** (BPE-vs-byte mismatch); both work
within the v2.1 `find()` fallback. They land NUM/verbatim wins per Hyp A
(numeric_exact_match_rate 0.0 → 0.5+) and long-doc wins per Hyp C, but
the underlying grounding remains string-find heuristic — fragile under
duplicate surface forms (e.g. "1.2" appearing 5 times in one article;
`find()` always returns position 0).

v2.2 redesign **addresses root cause** at +0.5-1 wk extra effort over
A+C. The two are NOT mutually exclusive — v2.2 grounding could fold
in the A-style span-hint prompt block (pointing at TOKENIZED spans
instead of byte spans) and the C-style chunked extraction (verifier
runs per-chunk, merge step unchanged).

---

## 5. Decision matrix

| Path | Cost | Risk | Addresses root cause? | Banks A/C wins? |
|---|---|---|---|---|
| **α** v2.2 token grounding redesign | 1.5 wk | MEDIUM | YES | NO (parallel-incompat with A's existing prompt; C orthogonal) |
| **β** A+C compound on n=72 (v2.1) | 1 wk | LOW–MED | NO (still `find()` fallback) | YES (by definition) |
| **γ** α + β in parallel worktrees | 1.5 wk wall | MED (two ongoing experiments) | YES (α) + YES (β bank) | YES |

### Recommendation: **Path γ** (both in parallel)

Rationale:
- α and β touch **disjoint files**. α: new `dsl/parser_v2_2.py`,
  `verifier_v2_2.py`, `grammar-v2_2.lark`, prompt `cod_dsl_v13_token_grounding.txt`,
  runner flag. β: `scripts/retrieve_spans.py` (new), `scripts/dsl_chunker.py`
  (new), prompt `cod_dsl_v12_span_hints.txt`, runner conditional. Zero overlap.
- α has +0.5-1 wk extra cost vs β alone; running in parallel costs
  zero additional wall time (different worktrees, different subagents per
  `feedback_parallel_agents_use_worktrees`).
- If α n=9 smoke fails to beat β baseline, we lose ~5 dev days but β
  still ships. If α succeeds, we have the architecturally-correct
  grounding mechanism unblocked for v3 / Phase 4.
- Resource constraint: both α and β use llama-server (CPU, serial) for
  n=9 sweeps. Need to sequence the sweeps even if implementations run
  parallel. Wall budget: ~4-6h llama-server time across both.

### Recommendation if user vetos γ: **Path α** (token grounding first)

Rationale: β is a tactical noise-floor push (banks +0.1-0.15 verifier);
α targets the architectural wedge that's been blocking gv=true on the
hard cells since v2. The 0.989 verbatim signal on Hyp A v12 healthy
cells is strong evidence the model CAN do token-level copy reliably —
we'd be foolish not to test it.

---

## 6. Open questions for supervisor

1. **Tokenizer-drift risk acceptable?** Verifier loads HF `tokenizer.json`
   from `Qwen/Qwen3-30B-A3B-Instruct-2507`; llama.cpp embeds the GGUF
   tokenizer at build time. Both come from the same source but a
   point-version mismatch is possible. The runner-side
   `assert tokenizer.encode(output) == captured_tokens` sanity gate
   catches it, but the user should confirm we're OK adding the HF
   tokenizers dep to the PoC runner. (~10MB wheel, no GPU.)

2. **Prompt-bloat budget.** Token lists are 5-10× longer than `[start,
   end]` pairs in characters. At n=100 blocks/doc this adds ~14KB to the
   output budget. `n_predict=8192` currently leaves room but a worst-case
   loop (33632 long-doc) could hit the cap earlier. Tolerate or
   pre-emptively switch to compact `"tok1|tok2|tok3"` separator format?

3. **Does v2.2 replace or supplement v2.1?** Two viable schemas:
   (a) v2.2 OBSOLETES v2.1, runner stops emitting `source_span`.
   (b) v2.2 ADDS `source_tokens` alongside `source_span` (both optional);
       verifier accepts either as grounding evidence. (b) lets us A/B
       on the same run; (a) is cleaner long-term. Decision should be
       made before v13 prompt-writing.

4. **Path γ resource contention.** llama-server has `--parallel 1` and
   is shared with other Phase 3 sweeps. Running α and β n=9 smokes
   back-to-back is fine; running them simultaneously requires bumping
   `--parallel` (last bumped to 3 in #403). Confirm we have the
   capacity OR sequence the sweeps despite parallel implementation.

---

## 7. STATE.md one-liner

DSL PoC v2.2 token-grounding recon: variant 2.2a (`source_token_text`)
recommended over 2.2b (token IDs); 7.5 dev days vs A+C's 5 days but
addresses BPE-vs-byte root cause; Path γ (α+β parallel worktrees)
recommended — disjoint files, no wall-time penalty, banks A+C wins
while testing architectural fix.

---

## Citations

- arXiv:2209.11895 (Olsson et al., Induction Heads — copy mode is token-level)
- arXiv:2509.06902 (Hakim et al., Proof-Carrying Numbers — value reproduction
  unreliability at the digit level)
- llama.cpp `/completion` API: `tools/server/README.md` (return_tokens,
  completion_probabilities)
- vLLM OpenAI server `logprobs`: `vllm/entrypoints/openai/protocol.py`
  (`ChatCompletionLogProb` with `.token` + `.bytes`)
- Existing v2.1 verifier: `docs/benchmarks/cod-2026-05-17/dsl/verifier.py:320-410`
  (`_check_span_and_copy` `find()` fallback path)
- Existing verbatim metric: `docs/benchmarks/cod-2026-05-17/scripts/score_dsl.py:647`
  (`_verbatim_match_rate` — analog target for `_token_verbatim_match_rate`)
