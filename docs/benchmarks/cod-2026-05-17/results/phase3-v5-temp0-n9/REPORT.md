# Phase 3 v5 T=0.0 deterministic probe — sampling-noise vs prompt-ceiling diagnostic

**Date:** 2026-05-22
**Model:** `qwen3:30b-a3b-instruct-2507-q4_K_M` (4 B active / 30 B total MoE, q4_K_M GGUF)
**Backend:** `llama-server` + GBNF grammar-constrained decoding
**Schema:** v2.1 no-offset (`docs/grammars/cod-dsl-v2.1.gbnf`, v5 prompt `prompts/cod_dsl_v2_no_offset.txt`)
**Runner:** `scripts/run_cod_dsl.py --schema v2_1 --backend llama-server --grammar docs/grammars/cod-dsl-v2.1.gbnf --n-predict 6144 --num-ctx 16384 --temperature 0.0 --top-k 1 --timeout 700`
**Corpus:** Phase 2 Arm B (n=9), same cells as PR #397 v5 baseline
**Branch:** `experiment/phase3-v5-temp0-n9`
**Sampling:** deterministic (T=0.0, top-k=1), single sample per cell
**Baseline:** PR #397 v5 stochastic (default Ollama sampling)

---

## 1. Verdict (TL;DR)

**BOUNDED — sampling noise did NOT meaningfully drive v5's aggregate verifier. The 0.782 result from PR #397 was the true prompt ceiling, not variance.**

On the 6 cells where BOTH runs produced a defined verifier, deterministic decoding mean = 0.7946 vs stochastic mean = 0.7963 — a delta of **-0.0017** (essentially identical). The aggregate T=0 verifier of **0.7946 (n=6 defined)** confirms v5's halt-criterion failure (0.782 from PR #397) was the real prompt ceiling. Three of the nine cells were gv=False under T=0: 33632 (gv=False under stochastic too, but T=0 produced 0 tokens vs 3835), 35352 (gv=True ver=0.500 under stochastic → 0-token HARD TIMEOUT under T=0), and 36065 (gv=True ver=0.976 under stochastic → concept-latch returned under T=0). I.e. **stochastic sampling was sometimes ESCAPING deterministic failure modes** (or, on 33632, producing some emission rather than none), not generating variance around a higher ceiling.

**Verdict: BOUNDED per directive ("0.78-0.83 → BOUNDED; prompt ceiling confirmed; v5 0.782 was real value").**

Per epic guidance this means PR #397's HALT call was correct — v5's prompt has plateaued and the diagnostic next step (CoD-as-symbolic-DSL / leaner prompt / model swap) is the right path.

---

## 2. Per-cell table (T=0.0 deterministic, n=9)

| article | gv     | verifier | numeric_sem | org_rec | undecl | ent / evt / cl / note | eval_tok | wall_s | stop  |
|---------|--------|---------:|------------:|--------:|-------:|-----------------------|---------:|-------:|------:|
| 27560   | True   |    1.000 |         n/a |     n/a |      0 | 6 / 5 / 0 / 0         |      350 |  174.1 | eos   |
| 27702   | True   |    0.000 |         n/a |     n/a |      0 | 2 / 1 / 0 / 0         |      198 |  149.9 | eos   |
| 29807   | True   |    0.906 |       0.222 |   1.000 |      4 | 14 / 7 / 2 / 0        |      985 |  236.1 | eos   |
| 31149   | True   |    0.957 |       1.000 |   0.500 |      0 | 12 / 7 / 0 / 0        |      805 |  349.2 | eos   |
| 31430   | True   |    0.905 |       0.556 |   1.000 |      0 | 9 / 4 / 3 / 0         |      805 |  305.6 | eos   |
| 33632   | False  |      n/a |         n/a |     n/a |      0 | 0 / 0 / 0 / 0         |        0 |  700.1 | TIMEOUT |
| 34537   | True   |    1.000 |         n/a |     n/a |      0 | 8 / 14 / 0 / 0        |      797 |  348.6 | eos   |
| 35352   | False  |      n/a |         n/a |     n/a |      0 | 0 / 0 / 0 / 0         |        0 |  700.1 | TIMEOUT |
| 36065   | False  |      n/a |         n/a |     n/a |      0 | 0 / 0 / 0 / 0         |     1487 |  423.5 | limit |

**Aggregates:**
- `grammar_valid` rate: **6/9 = 66.7 %** (33632, 35352, 36065 invalid)
- `verifier_pass_rate` mean (n=6 defined): **0.7946**
- Wall total: **3,387.2 s** (~56.5 min), mean **376 s/cell**
- Total tokens generated: **5,427**

Note: 33632 and 35352 produced ZERO output (HTTP read timeout at 700 s, model never began emitting tokens that reached the runner). 36065 emitted 1,487 tokens but ran into the concept-latch failure mode (see §4).

---

## 3. Side-by-side delta vs PR #397 v5 stochastic

| article | v5 stoch ver | T=0.0 ver | Δ ver    | v5 tok | T=0 tok | v5 wall | T=0 wall | T=0 outcome                                |
|---------|-------------:|----------:|---------:|-------:|--------:|--------:|---------:|--------------------------------------------|
| 27560   |        1.000 |     1.000 |  +0.000  |    910 |     350 |   220.2 |    174.1 | identical-quality (faster)                 |
| 27702   |        0.000 |     0.000 |  +0.000  |    198 |     198 |   156.8 |    149.9 | **byte-IDENTICAL output** (see §4.1)       |
| 29807   |        0.957 |     0.906 |  -0.050  |    819 |     985 |   238.4 |    236.1 | within single-shot band                    |
| 31149   |        0.905 |     0.957 |  +0.052  |    758 |     805 |   216.6 |    349.2 | within single-shot band                    |
| 31430   |        0.917 |     0.905 |  -0.012  |    882 |     805 |   227.8 |    305.6 | within single-shot band                    |
| 33632   |          n/a |       n/a |     n/a  |   3835 |       0 |   649.0 |    700.1 | T=0 HARD TIMEOUT (no output)               |
| 34537   |        1.000 |     1.000 |  +0.000  |    726 |     797 |   213.2 |    348.6 | identical-quality (slower)                 |
| 35352   |        0.500 |       n/a |     n/a  |    209 |       0 |   200.7 |    700.1 | **T=0 REGRESSED** (timeout vs 209-tok eos) |
| 36065   |        0.976 |       n/a |     n/a  |   1487 |    1487 |   431.0 |    423.5 | **T=0 REGRESSED** (concept-latch returned) |

**Matched-cell aggregate (6 cells with defined verifier in both):**
- v5 stochastic: **0.7963** mean
- T=0.0:        **0.7946** mean
- Δ:            **-0.0017** (noise-band identical)

**Full-corpus aggregate (n=9 cell positions):**
- v5 stochastic: 0.782 (n=8 defined) — PR #397 halt-criterion value
- T=0.0:        0.7946 (n=6 defined) — but 3 cells became gv=False vs only 1 under stochastic

---

## 4. Variance-vs-noise diagnosis (the two cells PR #397 flagged)

### 4.1 27702 — v5 stochastic mean 0.250

PR #397 reported the **mean** verifier across stochastic runs was 0.250; the single n=9 acceptance run for that cell scored 0.000. T=0.0 deterministic also scores **0.000**, with **byte-identical raw output** (same eval_count=198, same trigger phrase "filed Form 144", same `_anon:shares` id form).

| field | v5 stochastic (n=9 acceptance) | T=0.0 |
|-------|--------------------------------|-------|
| verifier | 0.000 | 0.000 |
| eval_count | 198 | 198 |
| stop_type | eos | eos |
| ent/evt | 2/1 | 2/1 |
| raw_output | "...EVT m_and_a_announcement / trigger: \"filed Form 144\"..." | **identical** |

**Diagnosis: NOT variance-driven, NOT noise-driven, NOT sampling-noise — the deterministic baseline IS this output.** The PR #397 "0.250 mean across multiple stochastic samples" reflects ~25 % chance the model luckily diverges to an output that scores 1.0 on this empty-article (Fusion Media boilerplate) cell. The default Ollama sampling occasionally finds a verbatim-zero hallucination escape; T=0.0 always hits the same hallucinated trigger. **Stochastic noise was occasionally helping here, not hurting.**

### 4.2 35352 — v5 stochastic 0.500 ± 0.491 (high variance flagged)

PR #397 reported very high variance on this cell. Under v5 stochastic the n=9 acceptance sample scored 0.500 with 209 tokens / eos stop. Under T=0.0 the cell hit a **hard HTTP read timeout** (700 s) with **zero tokens reaching the runner**.

| field | v5 stochastic | T=0.0 |
|-------|---------------|-------|
| verifier | 0.500 | undefined (gv=False) |
| eval_count | 209 | 0 (timeout) |
| stop_type | eos | ReadTimeout 700 s |
| ent/evt | 2/2 | 0/0 |

**Diagnosis: T=0.0 went WORSE.** Stochastic sampling, on this cell (TSA passenger-volumes page, very dense numeric table), occasionally finds a clean short emission (eos in 209 tokens). The T=0 deterministic decode path apparently goes into a long-tail generation that doesn't terminate within 700 s — the high stochastic variance (0.500 ± 0.491) on this cell was not "sampling around a fixed deterministic point" but rather **sampling between a productive short-emission attractor and a long-tail/failure attractor**. The deterministic path lands in the failure attractor.

### 4.3 Bonus: 36065 (concept-latch)

PR #397 v5 reported 36065 verifier = 0.976 (the v5 KIND PRIORITY decree closed the v4 "concept-latch" failure where the model emits dozens of `concept`-kind ENTs). Under T=0.0, the model **returns to the concept-latch failure mode**: raw output ends with a runaway sequence "financial_stability / concept", "financial_resilience / concept", "financial_innovation / concept", "financial_inclusion / concept", "financial_access / concept", "financial_exclusion / concept", "financial_equity / concept", "financial_justice / concept" — hits the n-predict cap at 1487 tokens with `Unexpected token $END` parse error.

**Diagnosis: T=0 destroys the v5 decree #9 (KIND PRIORITY) intervention on this cell.** The intervention only works if sampling randomness selects against the concept token at the kind position; the deterministic top-1 path defeats the decree. This is **anti-variance**: random sampling was helping the prompt's prescriptions land.

---

## 5. Conclusions

1. **v5 prompt verifier ceiling is real, not a variance artifact.** Matched-cell mean (v5 stochastic 0.7963 vs T=0 0.7946) is identical to within 0.002. PR #397's HALT call at 0.782 was correct.

2. **Stochastic sampling was helping, not hurting, on 2 of 3 "variance-driven" cells.** 35352 (timeout failure mode escape) and 36065 (concept-latch escape) both show stochastic sampling produced strictly better outcomes than deterministic decoding. 27702 was byte-identical (no variance to diagnose).

3. **T=0 is a worse decode setting for v5 prompt on this corpus.** 3/9 cells went grammar-invalid under T=0 vs 1/9 under stochastic. Wall time is 33 % higher (3,387 s vs 2,554 s) due to the two 700 s timeouts.

4. **Implication for next workstream:** Variance reduction (T=0 / multi-sample averaging) is NOT a productive direction for v5. The architectural next step (per ATLAS LLM-strength-layering note 2026-05-20) — splitting CoD pattern-recognition (small CPU) from DSL emission (large GPU, trained adapter) — addresses the actual ceiling, which is prompt design, not sampling noise.

---

## 6. Files

- 9 cell JSONs: `qwen3_30b-a3b-instruct-2507-q4_K_M/v2_1_spec/{27560,27702,29807,31149,31430,33632,34537,35352,36065}.json`
- Sweep log: `/tmp/exp_b_sweep.log` (not committed)
- Branch: `experiment/phase3-v5-temp0-n9`
- Runner change: `676d9211` — additive `--temperature` and `--top-k` flags on `run_cod_dsl.py` (default behavior unchanged)
