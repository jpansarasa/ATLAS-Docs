# Path C — GPU grammar-constrained AWQ — n=9 Arm B smoke — NOTES

**Branch:** `experiment/path-c-grammar-constrained-gpu`
**Date:** 2026-05-23
**Model:** `sentinel-cove-v6.2` (vLLM alias = base `Qwen/Qwen2.5-32B-Instruct-AWQ`, no LoRA)
**Variant:** `v2_1_spec`
**Prompt:** `cod_dsl_baseline.txt` (locked = v8, PR #413, `f549ef8c`)
**Corpus:** `corpus.jsonl` (n=9 Arm B; skip 27772 by brief)
**GT:** `ground-truth.jsonl`
**Backend:** **vllm-server** (production), `/v1/chat/completions`,
            `structured_outputs.grammar` = `docs/grammars/cod-dsl-v2.1.gbnf`
**Wall:** sweep 14:30:03 → 14:43:26 UTC = **13.4 min** total
            (mean 89.2 s/cell, median 29.4 s/cell, max 293.6 s)

## TL;DR — Path C n=9 smoke FAILS the 0.85 early-exit gate. n=72 SKIPPED.

- **Penalized verifier (n=9): 0.7358** — vs v8 n=9 baseline **0.907**, Δ **−0.171**
- **Scored-only verifier (8/9): 0.8278** — vs v8 n=9 **0.907**, Δ **−0.079**
- **gv pass: 8/9 (89%)** — vs v8 n=9: **9/9 (100%)**, Δ **−11 pp**
- **Mechanism preservation (4 v8 flagships):** 1/4 preserved (27702),
  2/4 regressed but still gv-valid (31149 Δ −0.058, 36065 Δ −0.144),
  **1/4 hard gv-fail (33632)** — the same concept-flood failure mode
  that killed v8 at n=72, reproduced here at n=9 under decoder grammar
  enforcement.
- **n=9 < 0.85 threshold → STOP per brief.** n=72 validation NOT run.

## Per-cell table

| Cell  | gv  | vpr   | ent | num | evt | claim | note | wall_s | out_tok | stop | concept-kind-ENT |
| ----- | --- | ----- | --- | --- | --- | ----- | ---- | ------ | ------- | ---- | ---------------- |
| 27560 | T   | 0.857 | 3   | 0   | 3   | 0     | 1    |  15.1  |   230   | stop | 1                |
| 27702 | T   | 1.000 | 1   | 0   | 0   | 5     | 1    |   5.7  |   340   | stop | 0                |
| 29807 | T   | **0.164** | 12 | 9 | **106** | 0 | 1 | 293.6 | 4908   | stop | 0                |
| 31149 | T   | 0.900 | 11  | 3   | 5   | 0     | 1    |  29.4  |   584   | stop | **5**            |
| 31430 | T   | 0.929 | 5   | 4   | 4   | 0     | 1    |  26.2  |   540   | stop | 0                |
| 33632 | **F** | None | scoring counts 0 — gv-fail aborts scoring path | — | — | — | — | 101.8 | 2610 | stop | **126 macro_indicator-kind ENT** |
| 34537 | T   | 1.000 | 1   | 0   | 0   | 0     | 1    |   1.9  |    76   | stop | 0                |
| 35352 | T   | 0.991 | 1   | **111** | 0 | 0   | 1    | 274.6  | 6178    | stop | 0                |
| 36065 | T   | 0.781 | 12  | 12  | 7   | 0     | 1    |  54.2  |  1180   | stop | 2                |

> The "concept-kind-ENT" column counts ENT lines whose kind segment
> is the literal token `concept`. Cell 33632's flood used kind
> `macro_indicator` (126 lines, 24 distinct names — see "33632
> concept-count check" below). The kind label varies; the flood
> mechanism does not.

Aggregate gv pass: 8/9 = 88.9%. Penalized mean (gv-fail = 0): 0.7358.
Scored-only mean: 0.8278.

## Delta vs v8 Arm B n=9 baseline (PR #410)

| Metric                          | v8 n=9    | Path C n=9 | Δ        |
| ------------------------------- | --------- | ---------- | -------- |
| Penalized mean (gv-fail = 0)    | **0.907** | **0.7358** | −0.171   |
| Scored-only mean                | 0.907     | 0.828      | −0.079   |
| gv pass rate                    | 9/9       | 8/9        | −11.1 pp |
| Median wall (s)                 | 321       | 29.4       | −291     |
| Mean wall (s)                   | 349       | 89.2       | −260     |
| Total sweep wall                | ~52 min   | **13.4 min** | −38.6 min |

Path C is **~3-4× faster** end-to-end (GPU AWQ + paged attention vs CPU MoE)
but loses 0.171 verifier on penalized aggregate. Throughput is not the bottleneck.

## Mechanism cell preservation (the 4 v8 flagship cells in n=9)

| Cell  | v8 n=9 gv/vpr | Path C n=9 gv/vpr | Preserved?                              |
| ----- | ------------- | ----------------- | --------------------------------------- |
| 27702 | T / 1.000     | T / 1.000         | **YES** (identical)                     |
| 31149 | T / 0.958     | T / 0.900         | DROP (−0.058)                           |
| 33632 | T / 0.969     | **F / None**      | **NO — concept-flood REGRESSION**       |
| 36065 | T / 0.925     | T / 0.781         | DROP (−0.144)                           |

**Only 1/4 mechanism cells preserved.** Two drop within tolerance but
non-trivially (≥0.05). One hard gv-fail.

Same cell (33632) that broke at v8 n=72 ALSO breaks under Path C n=9
with the SAME failure mode — duplicate `_anon:concept`-style DECLARE-ONCE
violation. The decoder grammar constraint did not prevent the model
from emitting a duplicate symbol (because v2.1 GBNF enforces SHAPE,
not declare-once semantics).

## 33632 concept-count check (key signal per brief)

- **Brief asked:** "should be ≤5 if grammar caps work; if grammar
  doesn't cap concept count, note that GBNF needs `ent_concept_block{0,5}`
  style limits"
- **Observed:** 33632 emitted **127 ENT blocks total** — 1 of kind
  `org`, **126 of kind `macro_indicator`** (24 distinct names; 102
  duplicate emissions). Verifier's first detected duplicate was
  `_anon:energy_price_spike`. v8 baseline on this cell emitted 0
  concept-kind ENTs (NOTE-only output).
- **Verdict:** **GBNF DOES NOT CAP KIND COUNT.** v2.1 GBNF
  (`block-list ::= ("\n"* block){0,128}`) bounds total blocks to 128
  but is agnostic to kind. The kind-flood failure mode that killed
  v8 at n=72 reproduces here at n=9 with decoder grammar
  enforcement — same shape (model emits 100+ near-identical
  same-kind blocks), different kind label (`macro_indicator` here,
  `concept` in v8 n=72). GBNF cannot express:
  - "≤5 ENT blocks where type is `concept`/`macro_indicator`/<any
    kind in {concept, macro_indicator, generic_topic, ...}>"
  - "declare-once across the document" (a semantic / state-tracking
    constraint, not a context-free-grammar one)
  Both would require either a stateful decoder layer or a structural
  partition of `block-list` by kind plus a per-kind cap.

The mechanism layer is the same: model wants to enumerate every
nuance / synonym of a topic and the prompt's anti-flood guards
under-suppress on this particular article. Moving to GPU did not
change this behavior — the prompt is the same v8 prompt; the model
strength is comparable (32B AWQ dense vs 30B MoE 4B-active); the
new constraint (GBNF) does not bind the failure surface.

## Other concept / event / NUM flood patterns observed

Beyond the 33632 hard fail, two cells show NEW flood modes that hurt
vpr or would hurt it on a different scoring:

- **29807 (EVT-flood):** 106 EVT blocks (mostly `EVT market_movement`
  with near-identical triggers). vpr = **0.164** — penalized for
  over-extraction. Source article is short (~1.5 KB) but model
  emits 4908 output tokens (mean 89s, this cell 293s = 3× mean).
- **35352 (NUM-flood):** 111 NUM blocks (all distinct since the
  source IS a numeric table). vpr 0.991 by accident — verifier
  doesn't penalize NUM volume here because the GT row also has
  many numerics. But the wall (274s) and output (6178 tokens) are
  outliers that suggest the model is exhaustively enumerating
  every figure in the table rather than picking salient ones.

Pattern: **the v8 prompt's mechanism guards under-suppress on
specific article shapes (energy policy prose, ideological prose,
short equity-trade articles, dense numeric tables).** Grammar
enforcement at decoder layer does not address these — the failure
moves from "gv-fail because no terminator" to "gv-pass with floods
that crash vpr" or "gv-fail because DECLARE-ONCE".

## Verdict — Path C as-currently-specified DOES NOT CLEAR THE GATE

**Path C smoke verifier 0.7358 < 0.85 threshold → STOP per brief.**

n=72 validation NOT run. Path C as-currently-specified (v8 prompt
verbatim + v2.1 GBNF verbatim) does not fix the scale failure mode
that motivated the GPU pivot — it reproduces the same concept-flood
failure at n=9 that PR #414 documented at n=72 on the v8 baseline.

### What works under Path C

- **Grammar pipeline correctness:** vLLM 0.19 + AWQ +
  `structured_outputs.grammar` accepts the full `cod-dsl-v2.1.gbnf`
  as-is. 8/9 cells produced syntactically-valid DSL — the one
  failure (33632) was a DECLARE-ONCE semantic violation, not a GBNF
  parser failure.
- **Throughput:** 13.4 min wall for n=9 vs ~52 min on v8 baseline.
  **~3-4× faster end-to-end.** GPU AWQ paged attention dominates
  CPU MoE on this workload even with a single-stream pattern
  (`--max-num-seqs 16` slot reuse was effective).
- **Production-share safety:** zero retries triggered
  (`vllm_attempts == 1` on every cell). vllm-server handled the
  burst alongside sentinel-collector traffic without
  rate-limiting.

### What does NOT work — root cause

The v8 prompt was iterated against a context-free behaviour model:
"model emits structure when told to". The actual failure surface
is semantic / stateful: declare-once across a document, kind-priority
caps that depend on running counts, and "stop enumerating synonyms"
heuristics. GBNF — by construction — cannot express any of these
because they require state. Decoder enforcement of v2.1 GBNF
catches structural drift; it cannot stop the model from emitting
a 35-element list of related concepts and then duplicating an
earlier symbol.

### Recommended next steps (NOT in this PR's scope)

Two viable next moves; both require new artifacts before another
sweep:

1. **Grammar-side mitigation (smallest delta from Path C):** add
   kind-bounded block list rules to v2.1 GBNF, e.g.
   `ent-concept-block-list ::= ent-block{0,5}`. Requires a new
   grammar file (call it `cod-dsl-v2.2.gbnf`) that partitions the
   block list by kind. Does NOT address DECLARE-ONCE (state) but
   would block the 35-concept enumeration.

2. **Validator-driven retry (was Path D in recon):** wire the
   verifier into the runner — if `grammar_valid: false` or
   `verifier_pass_rate < 0.9`, prepend a per-cell complaint string
   and re-emit. ≤3 retries per cell. Layered on Path C, this gets
   the GPU throughput AND the stateful constraint at extra wall.

3. **Fine-tune path (Path A/B from recon):** the original recon
   recommended this only if Path C landed below 0.80. Penalized
   0.7358 is below that bar by 0.064. Now justified, but blocked
   on bias-detection harness work (see recon §3 Path A risk).

Path C-only is closed for the moment. The runner backend is in
place and reusable for any of the above three follow-ons.

## Reproduction

```bash
# Same invocation as the brief, one cell at a time (resumable):
/home/james/ATLAS/.venv-bench/bin/python \
  docs/benchmarks/cod-2026-05-17/scripts/run_cod_dsl.py \
  sentinel-cove-v6.2 \
  --schema v2_1 --backend vllm-server \
  --grammar docs/grammars/cod-dsl-v2.1.gbnf \
  --n-predict 8192 --num-ctx 16384 \
  --prompt-file docs/benchmarks/cod-2026-05-17/prompts/cod_dsl_baseline.txt \
  --variant v2_1_spec \
  --results-root docs/benchmarks/cod-2026-05-17/results/path-c-gpu-n9-arm-b \
  --timeout 600 --force \
  --article-id <id>
```

Arm B cell ids: `29807 33632 27702 31149 31430 36065 27560 34537 35352`
(27772 skipped per brief).

## Live data points

- **vllm-server (production):** `sentinel-cove-v6.2` →
  `Qwen/Qwen2.5-32B-Instruct-AWQ`, `--max-model-len 32768`,
  `--max-num-seqs 16`, `--gpu-memory-utilization 0.92`.
  Smoke baseline preserved (no config touched, no restart).
- **Concurrent extraction load:** sentinel-collector continued to
  serve over the 13.4 min sweep window. No rate-limit / 5xx
  responses observed (`vllm_attempts == 1` on every cell).
- **GBNF accepted as-is:** 8/9 cells produced gv-valid output
  → confirms vLLM 0.19 + llguidance accepts the v2.1 grammar
  without re-encoding.
- **Failure modes catalogued:** 1 concept-flood DECLARE-ONCE
  (33632), 1 EVT-flood (29807), 1 NUM-flood (35352, vpr OK by
  coincidence), 2 verifier-quality drops on flagship cells
  (31149, 36065).
