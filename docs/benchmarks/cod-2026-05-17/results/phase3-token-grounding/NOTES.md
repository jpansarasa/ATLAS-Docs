# α — v2.2 Token-Grounding NOTES

Branch: `experiment/alpha-v2.2-token-grounding`
Recon doc: PR #423 (merged 2026-05-22)
This NOTES.md: 2026-05-23 (sweep completed 22:34 UTC)

## TL;DR

**Verdict: BYTE GROUNDING WINS** (delta -0.446 across n=9 Arm B cells).
Token grounding via verbatim `source_tokens:` slot collapses to ~0.17
mean match rate vs ~0.62 for byte grounding on the same runs. Three
cells (27560, 33632, 36065 — 3/9) regressed from `grammar_valid=True`
in v8 to `grammar_valid=False` in v2.2, with cell 27560 hitting the
8192-token limit looping `EVT macro_release` blocks. The v2.2 grammar
+ prompt did NOT preserve mechanism for previously-working cells.

Verdict criteria (per task brief):
* `token_verbatim_match_rate ≥ byte_verbatim_match_rate + 0.10` → TOKEN GROUNDING WINS
* within ±0.10 → COMPARABLE
* lower → BYTE GROUNDING WINS

Actual: byte=0.616, token=0.170, delta=-0.446. **Byte wins**, with
a margin > 4x the verdict threshold.

## 1. v2.2 schema + grammar + verifier design

### What v2.2 changes vs v2.1 (one bit, intentionally)

Every copy-bearing block (ENT, NUM, EVT, CLAIM, NOTE) gains an
**OPTIONAL** `source_tokens:` slot **alongside** the existing
optional `source_span:` slot. Both remain optional, so the same
run scores byte-level (find/equality) AND token-level (subseq
match) verbatim rates side-by-side — no separate sweep needed for
A/B.

Variant choice (per recon PR #423 §1): **2.2a `source_token_text`**
— `|`-delimited verbatim token strings (vs 2.2b vocab-IDs). 2.2a
is robust to tokenizer-ID drift between llama.cpp BPE and HF
`tokenizers`; only the surface strings must match.

### Grammar (docs/grammars/cod-dsl-v2.2.gbnf)

```
ent-block ::= ... ent-name-slot source-span-slot? source-tokens-slot? ent-derived-slot*
num-block ::= ... num-raw-slot source-span-slot? source-tokens-slot? context-span-slot? num-derived-slot*
evt-block ::= ... evt-trigger-slot evt-trigger-span-slot? source-tokens-slot? evt-derived-slot+
claim-block ::= ... evidence-span-slot? source-tokens-slot? claim-derived-slot+
note-block ::= ... source-span-slot? source-tokens-slot? generic-slot*

source-tokens-slot ::= "  - source_tokens: " token-list "\n"
token-list ::= token ("|" token){0,127}
token ::= token-char token-char{0,63}
token-char ::= [^|\n]
```

Bounds: ≤128 tokens per list, ≤64 chars per token.

### Verifier (dsl/verifier_v2_2.py)

`verify_document_v2_2(doc, input_text)` returns
`V22VerifierReport(byte_verbatim_match_rate, token_verbatim_match_rate,
byte_denom, token_denom, blocks)`:

* **byte_verbatim_match_rate** — apples-to-apples with v2.1's
  `verbatim_match_rate`: byte-equality when `source_span` present
  + in bounds, `find()` fallback when omitted.
* **token_verbatim_match_rate** — NEW. For each block with a
  non-empty `source_tokens:` list: tokenize the article once via
  HF `Tokenizer.from_pretrained("Qwen/Qwen3-30B-A3B-Instruct-2507")`,
  then check the parsed token list as a contiguous sub-sequence
  of the article tokens.

Tokenizer pinned to Qwen3-30B-A3B-Instruct-2507. `ATLAS_VERIFIER_TOKENIZER`
env override for alt-backend sweeps.

### Score wiring fix (dsl/verifier.py, 2026-05-23)

During post-sweep analysis the v2.1 axis verifier returned `pass_rate=0`
for every v2.2 doc because `verify_document` (dsl/verifier.py line 136)
had a hard `dsl_version not in ("v2", "v2.1")` early-return that
rejected v2.2. The new `source_tokens:` slot is invisible to the v2.1
verifier (which only looks at name/raw/trigger/source_span), so
extending the gate to `("v2", "v2.1", "v2.2")` is a backward-compatible
fix that lets the v2.1 axis be measured on v2.2 docs. Patch applied,
all 79 unit tests still pass, and the existing results were rescored
in place via the new score function (no need to re-run the sweep).

## 2. n=9 smoke: byte_verbatim_match vs token_verbatim_match

### Per-cell results

| cell  | gv  | verif | verb_v21 | byte_v22 | tok_v22 | b_d | t_d | prompt | eval | stop  |
|-------|-----|-------|----------|----------|---------|-----|-----|--------|------|-------|
| 27560 | Fal |  None |     None |     None |    None |   0 |   0 |  10728 | 8192 | limit |
| 27702 | Tru | 1.000 |    0.000 |    0.000 |   0.000 |   0 |   0 |  10759 |   86 |   eos |
| 29807 | Tru | 0.919 |    0.917 |    0.917 |   0.389 |  36 |  36 |  10781 | 1812 |   eos |
| 31149 | Tru | 0.909 |    0.895 |    0.895 |   0.368 |  19 |  19 |  10662 | 1150 |   eos |
| 31430 | Tru | 0.917 |    0.895 |    0.895 |   0.263 |  19 |  19 |  11036 | 1365 |   eos |
| 33632 | Fal |  None |     None |     None |    None |   0 |   0 |  12282 | 4618 |   eos |
| 34537 | Tru | 1.000 |    0.000 |    0.000 |   0.000 |   0 |   0 |  10568 |   61 |   eos |
| 35352 | Tru | 0.991 |    0.991 |    0.991 |   0.000 | 112 | 112 |  12951 | 8192 | limit |
| 36065 | Fal |  None |     None |     None |    None |   0 |   0 |  14630 | 3795 |   eos |

### Aggregates (mean over n=9 cells; `None` rows excluded from rate means)

| metric                          | value |
|---------------------------------|-------|
| grammar_valid                   | 6/9 |
| verifier_pass_rate (mean)       | 0.956 (denom=6) |
| verbatim_match_rate (v2.1 axis) | 0.616 |
| byte_verbatim_match_rate (v2.2) | 0.616 |
| token_verbatim_match_rate       | 0.170 |
| byte_denom (sum)                | 186 |
| token_denom (sum)               | 186 |

The byte_v22 column matches verbatim_match_rate (v2.1 axis) exactly on
every cell — confirming the verifiers agree on the byte semantics
when `source_tokens:` is the only added slot. Note that the model
NEVER emits `source_span:` (it elides the optional slot on both v8
and v2.2 runs), so all byte checks fall through to the `find()`
fallback path.

### Token mismatch deep-dive (cell 29807)

On the cell with the highest token-mismatch density (byte=0.917
vs token=0.389), 19/36 blocks were `byte_OK_but_token_FAIL`.
Sample failures:

```
ENT U_S:      expected ['ĠU', 'S']                — model fabricated split
ENT Dior:     expected ['ĠDior']                  — single-token claim, not in article tokens
ENT LVMH:     expected ['ĠLVMH']                  — likely tokenizes as ['ĠL', 'V', 'MH']
ENT Stoxx600: expected ['ĠStoxx', 'Ġ600']         — fabricated leading-space tokens
ENT CAC40:    expected ['ĠCAC', 'Ġ40']            — same pattern
ENT FTSE100:  expected ['ĠFTSE', 'Ġ100']          — same pattern
```

The model is emitting plausible-looking BPE tokens (correct `Ġ` prefix
for word-initial positions) but its mental model of BPE boundaries
doesn't match the HF tokenizer's actual output. **The model can't
reliably emit verbatim Qwen3 BPE tokens** — even on cells where it
gets the byte-level verbatim copy right (byte=0.917+), its
`source_tokens:` lists fail the contiguous-subseq check ~60% of
the time.

This is the central negative result of v2.2: forcing tokenization
into the surface DSL costs accuracy without buying a stronger
verifiability gate.

## 3. Prompt-bloat impact (per Q3)

### Static prompt-file size
* v8 baseline (cod_dsl_baseline.txt): 37,082 bytes
* v13 token-hints (cod_dsl_v13_token_hints.txt): 40,866 bytes
* Δ: +3,784 bytes (+10.2%)

### Runtime prompt_eval_count

| stat | v8 baseline | v2.2 v13 | Δ | Δ% |
|------|------------|---------|---|-----|
| prompt_eval mean (n=9) | 10535.7 | 11599.7 | +1064.0 | +10.1% |

Prompt overhead matches the file-size growth exactly (every cell
shows Δp=+1064 tokens). No surprises here.

### Runtime eval_count (output tokens)

| stat | v8 baseline | v2.2 v13 | Δ | Δ% |
|------|------------|---------|---|-----|
| eval mean (n=9) | 1652.1 | 3252.3 | +1600.2 | +96.9% |

**Output nearly DOUBLED.** Per-cell breakdown:

| cell  | v8 eval | v22 eval | Δe |
|-------|---------|----------|-----|
| 27560 |     733 |     8192 | +7459 (limit; loop-fail) |
| 27702 |      86 |       86 | +0 (empty doc both runs) |
| 29807 |    1201 |     1812 | +611 |
| 31149 |     877 |     1150 | +273 |
| 31430 |     968 |     1365 | +397 |
| 33632 |    3387 |     4618 | +1231 |
| 34537 |      61 |       61 | +0 (empty doc both runs) |
| 35352 |    5956 |     8192 | +2236 (limit) |
| 36065 |    1600 |     3795 | +2195 |

Two cells (27560, 35352) hit `n_predict=8192` limit under v2.2 vs
not hitting it under v8. Cell 27560 was 733→8192 — a 11x explosion
caused by the model getting stuck emitting `EVT macro_release` blocks
in a tight loop after burning ~1000 tokens of valid output. The
extra `source_tokens:` slot on every block roughly doubles each
copy block's footprint, plus the model lost the v8 length discipline.

### Timeout-class concerns
Two cells hit n_predict cap (27560, 35352). Cell 27560 became
infinite-loop-degenerate. Cell 35352 was a long-output cell at v8
already (5956 eval) so the +37% bloat pushed it past 8192 but
the truncated output was still structurally valid (gv=True,
verifier=0.991). The 8192 cap is a hard limit on v2.2 viability
for long financial-rich documents.

### Wall-time

| stat | v8 baseline | v2.2 v13 | Δ |
|------|------------|---------|---|
| wall mean (n=9) | 348.9s | 309.6s | -39.3s (-11.3%) |

Wall improved slightly — but only because two cells (27702, 34537)
emitted near-zero output in both runs and v8's long-output cells
(35352, 36065) were already slow. The wall metric is dominated
by output_eval throughput; v2.2's slower-per-token output
(longer prompt cache miss + longer source_tokens decoding) is
offset by the looping cells getting clipped at 8192. Not a real
speed win.

## 4. Parallel impact analysis (per Q4)

**Status: SKIPPED — requesting supervisor approval.**

Per `ps -ef` at sweep launch and again at conclusion, the live
llama-server :11437 is running with `--parallel 1` (PID 106658,
running since 2026-05-22 with `--n-gpu-layers 0` CPU-only). Bumping
`--parallel` to 2 or 3 requires editing
`/opt/ai-inference/compose.yaml` and re-deploying via ansible —
this is a production service config change that touches the same
file as #403/#407 and would briefly disconnect any in-flight client
of :11437. The brief's Q4 fallback clause:

> If risky, document the experiment design but skip the actual runs
> and request supervisor approval.

### Experiment design (for future execution under controlled conditions)

1. **Cells**: 27560, 31149, 36065 (Arm B n=9 healthy + short subset).
2. **Conditions**: same prompt (v8 baseline or v13 v2.2 — TBD with
   supervisor), three llama-server configurations:
   * `--parallel 1` (current)
   * `--parallel 2` (restart required)
   * `--parallel 3` (restart required; #403 setting)
3. **Measurements per cell × condition**:
   * `eval_count` (output tokens generated)
   * `prompt_eval_count` (prompt tokens processed)
   * `total_duration_ns` (wall time)
   * tok/s (derived: eval_count / (total_duration_ns / 1e9))
   * `verifier_pass_rate` (quality should be invariant; sanity check)
4. **Risk**: each restart drops in-flight client requests. The
   sweep should be staged when no other consumer is using :11437.
5. **Expected outcome**: per #403 commit message, --parallel 3
   should ~3x sweep throughput on independent cells but not affect
   per-cell latency. Verify this empirically — and verify
   per-cell quality is invariant (no GPU OOM at higher concurrency
   on the q4_K_M model).

### Baseline (`--parallel 1`) latencies from this sweep — for future comparison

| cell  | wall_s | eval | prompt_eval | tok/s (eval) |
|-------|--------|------|-------------|--------------|
| 27560 |  519.9 | 8192 |       10728 | 15.8 |
| 27702 |   63.5 |   86 |       10759 |  1.4 |
| 29807 |  335.7 | 1812 |       10781 |  5.4 |
| 31149 |  111.4 | 1150 |       10662 | 10.3 |
| 31430 |  126.8 | 1365 |       11036 | 10.8 |
| 33632 |  331.4 | 4618 |       12282 | 13.9 |
| 34537 |   59.0 |   61 |       10568 |  1.0 |
| 35352 |  618.9 | 8192 |       12951 | 13.2 |
| 36065 |  619.9 | 3795 |       14630 |  6.1 |

Request: supervisor approval + a maintenance window (no other
consumers on :11437) before executing this experiment. Note the
question is now mostly moot since v2.2 is being shelved (byte wins);
parallel-impact analysis on v8 baseline would still be useful and
should be filed as a separate ticket.

## 5. Mechanism preservation (v8 → v2.2 per cell)

| cell  | v8 verifier | v2.2 verifier | Δ | mechanism delta |
|-------|------------|---------------|---|------------------|
| 27560 | 0.667 | None | grammar fail | **regressed** (loop-fail; n_predict cap) |
| 27702 | 1.000 | 1.000 | +0.000 | preserved (empty doc both runs) |
| 29807 | 0.735 | 0.919 | +0.184 | **improved** (verifier on more strict v2.2 schema) |
| 31149 | 0.958 | 0.909 | -0.049 | within noise |
| 31430 | 0.917 | 0.917 | +0.000 | preserved |
| 33632 | 0.969 | None | grammar fail | **regressed** (was the previously-fragile cell) |
| 34537 | 1.000 | 1.000 | +0.000 | preserved (empty doc both runs) |
| 35352 | 0.991 | 0.991 | -0.000 | preserved |
| 36065 | 0.925 | None | grammar fail | **regressed** |

**Net:** 4 cells preserved, 1 noisy delta, 1 actually improved
(29807's verifier rate went up), and **3 cells regressed to
grammar fail**. The 33/3 regression rate (33%) is a strong negative
signal — v2.2's extra slot cost real grammar viability on borderline-
length cells, not just metric points. Cell 33632 was previously v6's
fragility canary and v2.2 made it worse, not better.

## 6. Verdict

**BYTE GROUNDING WINS** with delta -0.446, far outside the ±0.10
COMPARABLE band. Three additional concerns compound the verdict:

1. **Grammar regression** — 3/9 cells lost grammar validity vs v8.
   The `source_tokens:` slot increases per-block weight enough to
   bend the model into pathological loops on cells with many copy
   targets (FOMC release, 36065 long doc).
2. **Output bloat 2x** — even on grammar-valid cells, the v2.2 prompt
   nearly doubles eval_count. Two cells hit the 8192 cap; cell 27560
   spent 7459 extra tokens looping.
3. **Token-fabrication mode** — the model emits BPE tokens that look
   right but fail the contiguous-subseq check ~60% of the time even
   on cells where its byte-level copy is fine. Token surface forms
   are not robustly memorizable — the LLM doesn't have a stable
   tokenizer mental model.

### Recommendation

Shelve v2.2 token-grounding. Pursue alternative verifiability gates
that don't require the model to emit tokenization artifacts:

* **char-offset spans** (already in v2.1 grammar as `source_span:` —
  bring it back as REQUIRED rather than optional and measure)
* **char-ngram verifiability** (the existing span_copy_fraction is
  a parser-output-vs-source mechanism check; promote it to a
  per-block gate)
* **post-hoc retrieval** (β's PR #424 BOUNDED-PROGRESS chunked
  extraction shows a different attack vector: split documents, not
  slots)

The v2.2 grammar/parser/verifier code is preserved on this branch
for reference but should NOT be merged to `main` as an active
schema arm — keep it as a recon artifact alongside PR #423.

## Files touched

* `docs/grammars/cod-dsl-v2.2.gbnf` (new)
* `docs/benchmarks/cod-2026-05-17/dsl/grammar-v2_2.lark` (new)
* `docs/benchmarks/cod-2026-05-17/dsl/parser_v2_2.py` (new)
* `docs/benchmarks/cod-2026-05-17/dsl/verifier_v2_2.py` (new)
* `docs/benchmarks/cod-2026-05-17/dsl/verifier.py` (patched: v2.2 allowed on v2.1 axis)
* `docs/benchmarks/cod-2026-05-17/scripts/prompt_assembly.py` (extended)
* `docs/benchmarks/cod-2026-05-17/scripts/run_cod_dsl.py` (extended)
* `docs/benchmarks/cod-2026-05-17/scripts/score_dsl.py` (extended)
* `docs/benchmarks/cod-2026-05-17/prompts/cod_dsl_v13_token_hints.txt` (new)
* `docs/benchmarks/cod-2026-05-17/scripts/sweep_alpha_v2_2_n9.sh` (new)
* `docs/benchmarks/cod-2026-05-17/results/phase3-token-grounding/NOTES.md` (this file)
* `docs/benchmarks/cod-2026-05-17/results/phase3-token-grounding/alpha-v2_2-n9/...` (n=9 result JSONs + sweep.log)
