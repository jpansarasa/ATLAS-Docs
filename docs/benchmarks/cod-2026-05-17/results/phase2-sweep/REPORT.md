# Phase 2 sweep — grammar-constrained decoding (DSL PoC plan §5)

Three arms × n=10 articles on qwen3:30b-a3b-instruct-2507-q4_K_M, R2/R3 corpus. Aims to validate §5.3 acceptance gates and the §5.4 prompt-simplification hypothesis.

## Arms

- **arm-a** (`grammar_taught_free_ollama`) — Class B prompt, ollama backend, free decoder (Phase 1 best baseline)
- **arm-b** (`grammar_taught_constrained_llamaserver`) — Class B prompt, llama-server backend, v1.1 GBNF constraint
- **arm-c** (`content_only_constrained_llamaserver`) — minimal content-only prompt, llama-server backend, v1.1 GBNF constraint

## Per-arm aggregates

| arm | n | gv% | num_rec | num_exa | num_sem | ent_rec | tick | org | span_cp | wall_s | cost |
|---|---|---|---|---|---|---|---|---|---|---|---|
| arm-a | 10 | 100.0% | 88.6% | 66.9% | 88.6% | 29.4% | 50.0% | 31.0% | 62.5% | 339.72 | 6.01 |
| arm-b | 10 | 40.0% | 95.5% | 86.4% | 95.5% | 20.8% | 0.0% | 28.3% | 57.1% | 305.05 | 7.39 |
| arm-c | 10 | 20.0% | 0.0% | 0.0% | 0.0% | 0.0% | 0.0% | 0.0% | 32.7% | 269.51 | n/a |

Columns: `gv%` grammar-valid rate · `num_rec` numeric_recall (legacy, lenient) · `num_exa` numeric_exact_match_rate · `num_sem` numeric_semantic_match_rate · `ent_rec` ent_recall (legacy combined ticker+company) · `tick`/`org` per-category recall · `span_cp` span_copy_fraction (n=4 char n-gram) · `wall_s` mean wall seconds · `cost` cost_per_recall_point (wall_s / (mean(num+tick+org)·100), DSL-PoC plan §3.3).

## §5.3 acceptance verdict (Arm B)

| Gate | Result | Detail |
|---|---|---|
| gv%=100% under constrained decoding | **FAIL** | observed: 40.0% |
| numeric_recall ≥ 91.7% (Phase 1 best) | **PASS** | observed: 95.5% |
| latency overhead < 30% vs free decoder | **PASS** | observed: Arm A 339.7s vs Arm B 305.0s → -10.2% overhead |

**Overall: FAIL** — Phase 2 does NOT clear the §5.3 gates on qwen3:30b-a3b-instruct-2507-q4_K_M.

## §5.4 prompt-simplification verdict (Arm C vs Arm B)

- numeric_recall: Arm C 0.0% vs Arm B 95.5% → -95.5pp
- ent_recall: Arm C 0.0% vs Arm B 20.8% → -20.8pp
- grammar_valid: Arm C 20.0% vs Arm B 40.0%
- mean wall_s: Arm C 269.5s vs Arm B 305.0s
**Verdict: FAIL** — content-only loses 95.5pp on numeric_recall vs grammar-taught. The prompt still carries load beyond what GBNF enforces; keep the grammar-taught prompt.

## Caveats

- llama-server runs with `--ctx-size 16384` (server-side) and `--n-predict 4096` (per-request budget). The grammar permits up to 128 blocks per document and the model does not self-stop under constrained decoding — every constrained-arm cell hits the 4096-token output cap (and the long Jenoptik earnings-call cell 27772 hits the 540s `requests` timeout entirely, producing an empty result). Arm A used ollama's free decoder, which has no equivalent cap; the model self-stops at ~800-7900 tokens depending on article.
- `grammar_valid` failures in arms B and C are mid-block truncation artifacts, not GBNF rejections: the GBNF decoder emits syntactically valid block prefixes right up to the cap, but the post-hoc Lark parser sees `Unexpected $END` because the last block's slot list isn't terminated. The GBNF guarantee is per-token, not per-document.
- Arm A files are reused verbatim from `dsl-round3/` (same corpus IDs, same Class B prompt, same model, ollama backend). Tagged in-file with `phase2_arm` and `phase2_provenance`.
- The corpus is the same n=10 R2/R3 set; ground truth is the Claude-generated `ground-truth.jsonl` already used for Phase 1.x scoring.

## Terminator validation (v1.2 GBNF, n=3 quick re-run)

Quick mechanism check: re-ran Arm B with v1.2 GBNF + Class B prompt-with-END
instruction on n=3 previously-failing cells (`31149`, `27560`, `31430`).
Results dir: `arm-b-terminator/qwen3_30b-a3b-instruct-2507-q4_K_M/spec/`.

| article | gv | eval | stop | wall_s | failure mode (if any) |
|---|---|---|---|---|---|
| 27560 | True  | 623 | eos | 33.1 | — (was truncation in v1.1) |
| 31149 | True  | 687 | eos | 57.2 | — (was malformed-string-truncation in v1.1) |
| 31430 | False | 451 | eos | 39.6 | `Duplicate ENT 'etsy'` (degenerate-loop, pre-existing in v1.1 arm-b) |

gv% = 2/3 = **66.7%** (HARD CHECK ≥95% NOT MET).

**Mechanism verdict: validated** — every cell stopped voluntarily on `eos`
after emitting `END` (no more `stop=limit` at n_predict=4096). The two
truncation-class failures cleared. The third failure is a model
degenerate-loop pathology (article 31430 emits `ENT etsy / org` 30-59x
in a row) that was present in the v1.1 arm-b output too and is orthogonal
to the terminator fix — the parser rejects on a semantic dedup check, not
a structural truncation.

**Quality verdict: gv% gate not passed on this n=3 sample** — but the
remaining failure class (degenerate loops) is not a grammar/decoder
problem; further iteration on the terminator design will not fix it.
Path forward: a full n=10 re-run is needed to characterize how many
of the original 6 v1.1 arm-b failures were truncation (terminator
should clear) vs degenerate loops (terminator can't fix). Supervisor
to decide whether to (a) accept the mechanism win + carry degenerate-loop
issue separately, or (b) widen the grammar to bound repeated-ENT
emissions.

