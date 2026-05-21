# Phase 3 v2 — Prompt-Strengthening Probe (path 1 per plan §11.2)

**Verdict (TL;DR):** **INSUFFICIENT.** Prompt-strengthening did NOT
close the verbatim-span gap. Average `verbatim_match_rate` across the
3 cells is **0.0** (one defined value at 0.0; two cells are null
because the grammar didn't validate). This is well below the
INSUFFICIENT threshold of 0.20, confirming path (3) — the Phase 4
GPU verifier per plan §7 — is the required next step.

## Probe design

PR #390 (commit `d6f35887`) established the v2 copy-mode baseline at
n=9, finding `verbatim_match_rate=0%` across every cell where the
metric was defined. The blocker hypothesis: **prompt-only signal is
not enough to teach the span semantics; the model emits plausible-
looking `[start, end]` integers that do not point at the substring
claimed.**

This probe tests that hypothesis with the cheapest possible
intervention: a strengthened v2 prompt (commit `2c66e2be`) adding two
worked examples with explicit per-span character-counting arguments
plus a hard "count, verify, REDUCE if unsure, OMIT if can't" rule.
No grammar or runner change; identical schema, model, backend,
corpus, and decoder settings as baseline.

**Decision rules** (set by the dispatcher before the run):

| Avg `verbatim_match_rate` (n=3) | Verdict          | Next step                                     |
| -------------------------------- | ---------------- | --------------------------------------------- |
| >= 0.50                          | WORKS            | full n=10 follow-up sweep                     |
| 0.20 – 0.50                      | PARTIAL          | supervisor judgment                           |
| < 0.20                           | INSUFFICIENT     | confirm path (3) Phase 4 GPU verifier needed  |

**Cells under test:** 27560, 31149, 31430 — the same articles that
prompted the PR #390 baseline FAIL verdict (representative of the
two failure modes seen there: grammar-invalid output and grammar-
valid output with zero verbatim matches).

**Run configuration:**
- Model: `qwen3:30b-a3b-instruct-2507-q4_K_M`
- Backend: `llama-server` on `:11437`
- Schema arm: `--schema v2` (routes prompt -> `cod_dsl_v2.txt`, grammar -> `docs/grammars/cod-dsl-v2.gbnf`, results -> `phase3-prompt-probe/`)
- `--n-predict 4096` (bumped from baseline's 2048 to remove the stop=limit confound)
- `--timeout 400` per cell; wall-clock budget cap 420s
- Temperature 0.2 (runner default)

## Per-cell metrics

| Article | Run      | grammar_valid | verbatim_match_rate | verifier_pass_rate | span_validity_rate | span_copy_fraction | ent / evt / claim | wall_s  | Failure mode                              |
| ------- | -------- | ------------- | ------------------- | ------------------ | ------------------ | ------------------ | ----------------- | ------- | ----------------------------------------- |
| 27560   | baseline | false         | —                   | —                  | —                  | 0.329              | 0 / 0 / 0         | 343.82  | dup ENT 'FOMC' at line -1                 |
| 27560   | probe    | false         | —                   | —                  | —                  | 0.347              | 0 / 0 / 0         | 129.97  | dup ENT 'FOMC' at line -1 (same)          |
| 31149   | baseline | false         | —                   | —                  | —                  | 0.224              | 0 / 0 / 0         | 138.72  | unexpected ENT token at line 6            |
| 31149   | probe    | **true**      | **0.0**             | 0.583              | 1.0                | 0.360              | 1 / 1 / 7         | 108.64  | grammar now parses; spans are fiction     |
| 31430   | baseline | true          | **0.0**             | 0.167              | 1.0                | 0.224              | 7 / 3 / 3         | 100.64  | grammar valid; vmr=0 across all 6 slots   |
| 31430   | probe    | **false**     | —                   | —                  | —                  | 0.160              | 0 / 0 / 0         | 152.00  | unexpected ENT token at line 6 (REGRESS)  |

## Aggregate

- **Cells with `verbatim_match_rate` defined** (i.e., gv=true and any
  copy slots emitted): probe **1/3**, baseline **1/3**.
  - Probe defined cell: 31149 -> vmr = **0.0**
  - Baseline defined cell: 31430 -> vmr = **0.0**
- **Average `verbatim_match_rate` across defined cells:** probe
  **0.0**, baseline **0.0**.
- **Average treating null cells as 0** (conservative): probe
  **0.0**, baseline **0.0**.
- **Grammar-valid rate:** probe **1/3**, baseline **1/3** — net zero;
  31149 went valid, 31430 went invalid. The set of cells that parse
  shifted but the proportion did not.

The single measurable comparison (gv=true ∧ slots emitted) is
**0.0 vs 0.0** — no movement. Even if every other span emitted in
the probe were a perfect match, the average would still sit below
the 0.20 INSUFFICIENT threshold.

## Qualitative inspection — why the model still fails

The probe's `31149.json` raw output is instructive. Excerpt:

    ENT Jefferies / analyst_firm
      - name: "Jefferies"
      - source_span: [106, 114]

`"Jefferies"` is 9 characters; `[106, 114]` is an 8-character span.
The two cannot match — no amount of prompt examples will make
`"Jefferies"` fit. Every other copy-slot in the same output exhibits
the same class of error: numeric `[start, end]` pairs are emitted in
syntactically valid grammar form, but the model is not actually
*locating* the substring in the source. It is *generating* plausible
integer pairs.

This is the failure mode the v2 schema was designed to detect
(verifier rejects mismatches) — and the failure mode the dispatcher
hypothesized in plan §11.2: **the model has no representation of
character offsets to ground these integers in.** A 4B-active-param
MoE has neither the architectural capacity nor the training-data
exposure to count characters into a 2-3KB article from a prompt
example.

The 31430 regression (grammar went valid -> invalid) is a separate
signal: the 169-line prompt extension perturbed the model's structural
output in ways unrelated to span learning. Two cells flipped in
grammar-validity (one each direction), suggesting the prompt change
operates well within the model's nondeterminism band on this metric.

## Verdict

**INSUFFICIENT.** Per the dispatcher's decision rule
(avg verbatim_match_rate < 0.20 -> INSUFFICIENT), prompt-only fixes
will not close the gap. Path (3) — the Phase 4 GPU verifier per plan
§7 — is the architecturally correct next step. The verifier must
either (a) ground spans externally via post-hoc lookup against the
source text, or (b) be served by a substantially larger / fine-tuned
model with span-awareness baked in.

## Artifacts

- Strengthened prompt: `docs/benchmarks/cod-2026-05-17/prompts/cod_dsl_v2.txt` (commit `2c66e2be`)
- Smoke results: `docs/benchmarks/cod-2026-05-17/results/phase3-prompt-probe/qwen3_30b-a3b-instruct-2507-q4_K_M/v2_spec/{27560,31149,31430}.json` (commit `c33f8dd3`)
- Baseline for comparison: `docs/benchmarks/cod-2026-05-17/results/phase3-v2/qwen3_30b-a3b-instruct-2507-q4_K_M/v2_spec/{27560,31149,31430}.json` (commit `d6f35887`, PR #390)
