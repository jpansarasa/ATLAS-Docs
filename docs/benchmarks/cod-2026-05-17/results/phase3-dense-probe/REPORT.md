# Phase 3 v2 — Dense-Model Probe (path 2 per plan §11.2)

**Verdict (TL;DR):** **ARCHITECTURE.** Swapping the 30B / 3.3B-active MoE
(`qwen3:30b-a3b-instruct-2507-q4_K_M`) for a 32B fully-dense base
(`qwen2.5:32b-instruct-q4_K_M`) did NOT close the verbatim-span gap.
Average `verbatim_match_rate` across all 3 cells is **0.0** — identical
to both the baseline (PR #390) and the prompt-strengthening probe
(commit `dac82695`). This is below the INSUFFICIENT threshold of 0.20
and confirms path (3) — the Phase 4 GPU verifier per plan §7 — is the
architecturally correct next step.

## Probe design

PR #390 (`d6f35887`) established the v2 copy-mode baseline FAIL at n=9
with `verbatim_match_rate = 0%` on every gv=true cell. The
prompt-strengthening probe (`c33f8dd3` + `dac82695`) added two worked
examples plus a hard "REDUCE or OMIT if unsure" rule and produced no
movement (vmr 0.0).

The hypothesis under test here: **is the MoE's 3.3B active-parameter
footprint the bottleneck?** Two plausible failure modes were on the
table — (a) **MODEL CAPACITY** — active params insufficient to ground
integer offsets in source text, in which case a fully-dense 32B should
fix it; (b) **ARCHITECTURE** — no Transformer LM at this scale (with
character-blind BPE tokenization) can count bytes into a 2-3 KB
article from a prompt example, in which case 32B dense fails the same
way and we need an external verifier.

This probe runs the same 3 prior-fail cells, same prompt (`2c66e2be`),
same grammar, same runner under a fully-dense 32B base. **One
controlled variable: parameter footprint.**

**Decision rules** (set by the dispatcher before the run):

| Avg `verbatim_match_rate` (n=3) | Verdict          | Next step                              |
| -------------------------------- | ---------------- | -------------------------------------- |
| >= 0.50                          | MODEL CAPACITY   | qwen2.5:32b becomes new v2 default     |
| 0.20 – 0.50                      | PARTIAL          | supervisor judgment                    |
| < 0.20                           | ARCHITECTURE     | confirm path (3) Phase 4 GPU verifier  |

**Cells under test:** 27560, 31149, 31430 — identical to PR #390 +
prompt-probe (representative of the two failure modes there: grammar-
invalid output and grammar-valid output with zero verbatim matches).

**Run configuration:**
- Model: `qwen2.5:32b-instruct-q4_K_M` (fully dense 32.76 B params,
  loaded from `/opt/ai-inference/models/models/blobs/sha256-eabc98a9...`)
- Backend: `llama-server` sidecar `llama-server-dense` on `:11438`
  (existing MoE `:11437` left untouched)
- Sidecar launch: `docs/benchmarks/cod-2026-05-17/scripts/launch_dense_sidecar.sh`
- Schema arm: `--schema v2` (variant `v2_spec`, grammar
  `docs/grammars/cod-dsl-v2.gbnf`, results `phase3-dense-probe/`)
- `--n-predict 4096`, `--num-ctx 16384`, `--timeout 900`,
  temperature 0.2 (runner default)
- `--threads 16`, `--n-gpu-layers 0` (CPU-only — GPU was busy with
  vLLM, irrelevant to the test since the comparison is per-cell
  correctness, not throughput)

Wall-clock note: dense 32B serializes on CPU (Phase 1.3 measured
6.51 TPS at small ctx). All three cells in this probe ran
~420–490 s each — well above the MoE arm's 100–150 s but inside the
900 s timeout budget. No cell came close to the 20 min hard-stop.

## Per-cell metrics — three-way comparison

| Article | Run                | Model                                | gv     | verbatim_match_rate | verifier_pass_rate | span_validity_rate | span_copy_fraction | ent/evt/claim | wall_s | Failure mode                                          |
| ------- | ------------------ | ------------------------------------ | ------ | ------------------- | ------------------ | ------------------ | ------------------ | ------------- | ------ | ----------------------------------------------------- |
| 27560   | baseline (PR #390) | qwen3:30b-a3b (MoE, 3.3B active)     | false  | —                   | —                  | —                  | 0.329              | 0 / 0 / 0     | 343.82 | dup ENT `FOMC` at line -1                             |
| 27560   | prompt-probe       | qwen3:30b-a3b (MoE, 3.3B active)     | false  | —                   | —                  | —                  | 0.347              | 0 / 0 / 0     | 129.97 | dup ENT `FOMC` at line -1 (same)                      |
| 27560   | **dense-probe**    | **qwen2.5:32b (dense, 32.8B)**       | **true** | **0.0**           | **0.0**            | **1.0**            | 0.396              | 2 / 14 / 0    | 453.51 | grammar valid; ALL 16 copy slots miss source bytes    |
| 31149   | baseline (PR #390) | qwen3:30b-a3b (MoE, 3.3B active)     | false  | —                   | —                  | —                  | 0.224              | 0 / 0 / 0     | 138.72 | unexpected ENT token at line 6                        |
| 31149   | prompt-probe       | qwen3:30b-a3b (MoE, 3.3B active)     | true   | 0.0                 | 0.583              | 1.0                | 0.360              | 1 / 1 / 7     | 108.64 | grammar parses; spans are fiction                     |
| 31149   | **dense-probe**    | **qwen2.5:32b (dense, 32.8B)**       | **true** | **0.0**           | **0.333**          | **1.0**            | 0.418              | 2 / 4 / 4     | 426.21 | grammar parses; spans are fiction (same class as MoE) |
| 31430   | baseline (PR #390) | qwen3:30b-a3b (MoE, 3.3B active)     | true   | 0.0                 | 0.167              | 1.0                | 0.224              | 7 / 3 / 3     | 100.64 | grammar valid; vmr=0 across all 6 slots               |
| 31430   | prompt-probe       | qwen3:30b-a3b (MoE, 3.3B active)     | false  | —                   | —                  | —                  | 0.160              | 0 / 0 / 0     | 152.00 | unexpected ENT token at line 6 (REGRESS vs base)      |
| 31430   | **dense-probe**    | **qwen2.5:32b (dense, 32.8B)**       | **true** | **0.0**           | **0.0625**         | **1.0**            | 0.214              | 5 / 5 / 1     | 486.98 | grammar valid; vmr=0 across all 16 copy slots         |

## Aggregate

- **Grammar-valid rate:** dense-probe **3/3**, prompt-probe **1/3**,
  baseline **1/3**. The dense model is strictly more reliable at
  emitting parseable output — the 32B dense weight kernel has the
  representational headroom to keep the v2 schema's structural rules
  (declare-before-use, unique ENT names, required slots) in working
  context that the 3.3B-active MoE lost.
- **Avg `verbatim_match_rate` across defined cells (gv=true):**
  dense-probe **0.0** (n=3, all three defined), prompt-probe **0.0**
  (n=1), baseline **0.0** (n=1).
- **Avg `verbatim_match_rate` treating null as 0** (conservative):
  dense-probe **0.0**, prompt-probe **0.0**, baseline **0.0**.
- **Avg `verifier_pass_rate` across defined cells:** dense-probe
  **0.132** (n=3: 0.0 + 0.333 + 0.0625, mean 0.132), prompt-probe
  **0.583** (n=1, single cell — not directly comparable), baseline
  **0.167** (n=1, same caveat).
- **Avg `span_validity_rate`:** dense-probe **1.0** (n=3) — every
  emitted span is syntactically well-formed `[int, int]` within
  `[0, len(article))`. The model is NOT producing garbage integers; it
  is producing **plausible** integers that simply don't point at the
  claimed substring.

The grammar-validity win is real (3/3 vs 1/3) but irrelevant to the
v2 schema's payoff. The v2 schema exists to drive `verbatim_match_rate`
toward 1.0 (every claimed span verifiable), and on that metric the
dense 32B is indistinguishable from the 3.3B-active MoE: both score
**0.0** across every defined cell.

## Qualitative inspection — same failure class as MoE

Dense `31149.json` raw output, ENT block 1:

    ENT Jefferies / analyst_firm
      - name: "Jefferies"
      - source_span: [112, 120]

`"Jefferies"` is 9 characters; `[112, 120]` is an 8-character span.
This is the **same byte-level miscount** found in the prompt-probe's
MoE output (which emitted `[106, 114]` — same 9-vs-8 error class,
slightly different offsets). Every other copy slot in the dense
output exhibits the same failure: well-formed integers, within
bounds, syntactically valid — and locating the wrong substring.

## Why dense doesn't fix it (mechanism)

Two compounding architectural constraints:

1. **BPE tokenization is character-blind.** The model never sees
   "characters" — it sees BPE subword tokens. A request to emit
   `[start, end]` *character* offsets requires it to maintain a
   running character-count of its own input as it decodes, which no
   pretraining signal teaches it to do. This is independent of
   parameter count.
2. **Position embeddings encode token positions, not byte positions.**
   The model has direct attention access to "the 47th token" but no
   built-in addressing for "byte 112". Synthesizing byte offsets from
   token positions is a derived computation the model is not trained
   on at any scale.

The 3.3B MoE failed by emitting fabricated spans inside a partially-
malformed structural envelope. The 32B dense fails by emitting
fabricated spans inside a perfectly-formed structural envelope. The
*fabrication* class is the failure under test, and parameter scaling
moved it not at all.

## Verdict

**ARCHITECTURE.** Per the dispatcher's decision rule (avg
`verbatim_match_rate` < 0.20 → ARCHITECTURE), no in-model fix at
this scale or family will close the gap. The v2 schema's copy-mode
design is sound — the verifier correctly distinguishes claimed-vs-
actual spans — but the LLM cannot produce the integers it claims.

**Recommended next step (per plan §11.2):** path (3) — Phase 4 GPU
verifier per plan §7. The verifier must ground spans **externally**
via post-hoc substring lookup against the source text (i.e., the
model emits `name:` only and a deterministic post-processor locates
the span, OR the verifier rewrites the emitted `source_span:` to the
first-match byte offset of the emitted `name:` before scoring). The
v2 schema's `span_validity_rate = 100%` across all 3 cells means the
runtime envelope is reliable; only the byte arithmetic needs an
external prosthesis.

## Comparison summary (three-way, mean across defined cells)

|                                       | baseline (PR #390) | prompt-probe    | **dense-probe (this)**    |
| ------------------------------------- | ------------------ | --------------- | ------------------------- |
| Model                                 | qwen3:30b-a3b      | qwen3:30b-a3b   | **qwen2.5:32b (dense)**   |
| Active params                         | 3.3 B              | 3.3 B           | **32.8 B**                |
| Prompt                                | v2 baseline        | v2 strengthened | v2 strengthened           |
| Grammar-valid rate (3 cells)          | 1/3 (33%)          | 1/3 (33%)       | **3/3 (100%)**            |
| Avg `verbatim_match_rate` (def cells) | **0.0**            | **0.0**         | **0.0**                   |
| Avg `verifier_pass_rate` (def cells)  | 0.167              | 0.583           | 0.132                     |
| Avg `span_validity_rate` (def cells)  | 1.0                | 1.0             | 1.0                       |
| Avg wall_s                            | 194 s              | 130 s           | 456 s                     |

**Bottom line:** dense-32B fixed grammar reliability (the symptom)
but did not move the verbatim-span metric one decimal place (the
disease). The disease is architectural, not capacity-bound.

## Artifacts

- Sidecar launch script: `docs/benchmarks/cod-2026-05-17/scripts/launch_dense_sidecar.sh`
- Sidecar identity verified: `n_params: 32_763_876_352`,
  `n_ctx: 16384`, `format: gguf`, `quantization: q4_K_M`
- Dense-probe smoke results:
  `docs/benchmarks/cod-2026-05-17/results/phase3-dense-probe/qwen2.5_32b-instruct-q4_K_M/v2_spec/{27560,31149,31430}.json`
- Prior-arm baselines for comparison:
  - PR #390 baseline: `docs/benchmarks/cod-2026-05-17/results/phase3-v2/qwen3_30b-a3b-instruct-2507-q4_K_M/v2_spec/{27560,31149,31430}.json`
  - Prompt-probe (`c33f8dd3`): `docs/benchmarks/cod-2026-05-17/results/phase3-prompt-probe/qwen3_30b-a3b-instruct-2507-q4_K_M/v2_spec/{27560,31149,31430}.json`
- Strengthened v2 prompt (`2c66e2be`):
  `docs/benchmarks/cod-2026-05-17/prompts/cod_dsl_v2.txt` (identical to
  prompt-probe run — controlled variable is the model only)
