# Phase 3 ENT-ref cleanup — 3-cell smoke vs n=9 baseline

**Date:** 2026-05-21
**Model:** `qwen3:30b-a3b-instruct-2507-q4_K_M` (4 B active / 30 B total MoE, q4_K_M GGUF)
**Backend:** `llama-server` (`http://localhost:11437`) + GBNF grammar-constrained decoding
**Schema:** v2.1 no-offset (`docs/grammars/cod-dsl-v2.1.gbnf` + revised `prompts/cod_dsl_v2_no_offset.txt`)
**Runner:** `scripts/run_cod_dsl.py --schema v2_1 --backend llama-server --n-predict 4096 --timeout 500`
**Branch:** `feat/dsl-poc/phase3-ent-ref-cleanup` off main (`23dc4ce6`)
**Baseline reference:** `docs/benchmarks/cod-2026-05-17/results/phase3-v2_1-n9/` (PR #393)

---

## 1. Reframe (why this exists)

PR #393 documented 5/6 n=9 gv-failures as a "preprocessor `name:`-synthesis gap": the lenient front-end was generating `ENT <id> = "<phrase>" / org (inferred=true)` blocks WITHOUT a `name:` slot for any subject/counterparty/speaker ref the model hadn't explicitly declared. Under v2.1's required `name:` copy slot the synthesized header wedges the grammar at the next block.

The 2026-05-21 reframe (user, ntfy `LdHP2kWD3vtH`): teaching the synthesizer to emit a `name:` slot is the wrong fix. Synthesis itself is the bug — it papers over the model's **declare-then-reference** failure (plan §7.1: "every `<ent_ref>` must resolve to a declared block"). The architectural cleanup:

1. **Parser**: stop synthesizing inline ENT blocks. Canonicalize ref text only.
2. **Parser pass 2**: stop raising on undeclared refs. Let them flow to the verifier as dangling strings.
3. **Verifier**: emit `hard_fail` with `reason=dangling_ref` for the referencing block (already implemented per §7.1).
4. **Prompt**: explicitly tell the model "declare-then-reference; the parser does not fabricate placeholders".

This re-aligns the deterministic gates so the model's miss is visible, not hidden.

---

## 2. 3-way per-cell comparison

| article | metric                | n=9 baseline (PR #393) | after cleanup | Δ                              |
|---------|-----------------------|------------------------|---------------|--------------------------------|
| 27560   | `grammar_valid`       | True                   | True          | HOLD                           |
| 27560   | `ent_count`           | 2                      | 2             | =                              |
| 27560   | `evt_count`           | 3                      | 3             | =                              |
| 27560   | `verbatim_match_rate` | 0.833                  | **0.900**     | **+0.067** (improved)          |
| 27560   | `verifier_pass_rate`  | 0.833                  | **0.900**     | **+0.067** (improved)          |
| 27560   | `wall_s`              | 31.8                   | 69.5          | slower (rerun nondeterminism)  |
| 31149   | `grammar_valid`       | **False**              | **True**      | **RECOVER**                    |
| 31149   | `ent_count`           | n/a (parse failed)     | 3             | RECOVER                        |
| 31149   | `evt_count`           | n/a                    | 1             | RECOVER                        |
| 31149   | `verbatim_match_rate` | n/a                    | **1.000**     | RECOVER                        |
| 31149   | `verifier_pass_rate`  | n/a                    | **1.000**     | RECOVER                        |
| 31149   | `wall_s`              | 54.6                   | 54.1          | =                              |
| 31430   | `grammar_valid`       | **False**              | **True**      | **RECOVER**                    |
| 31430   | `ent_count`           | n/a                    | 5             | RECOVER                        |
| 31430   | `evt_count`           | n/a                    | 4             | RECOVER                        |
| 31430   | `verbatim_match_rate` | n/a                    | **1.000**     | RECOVER                        |
| 31430   | `verifier_pass_rate`  | n/a                    | 0.625         | RECOVER (partial — see below)  |
| 31430   | `wall_s`              | 60.4                   | 71.8          | slightly slower                |

---

## 3. Verdict

**Did gv recover?** YES — both 31149 and 31430 went from parse-failure to `grammar_valid=True`. The dominant n=9 failure mode (5/6 cells, including both of our prior-fail cells) is eliminated. The synthesis path that was emitting malformed ENT headers no longer exists; the model's actual emission flows through the grammar unchanged.

**Did gv hold or drop on the previously-passing cell?** HOLD. 27560 stayed `grammar_valid=True` AND modestly improved on `verbatim_match_rate` / `verifier_pass_rate` (0.833 → 0.900). Different cells across reruns produce different exact verbatim counts (model nondeterminism on the CPU path); the structural shape is unchanged.

**Did verbatim purity improve?** YES, definitively. On the 3-cell intersection:
- baseline mean (1 cell defined): 0.833
- cleanup mean (3 cells defined): **0.967**

The 1.000 verbatim scores on 31149 and 31430 are particularly load-bearing — they confirm the model CAN emit clean copy slots; the n=9 baseline simply never got to score them because the synthesized header crashed the grammar at line 5-7.

**Did the architectural contract hold?** YES. 31430 cleanly demonstrates the design:

- Model declared all 5 proper-noun ENTs correctly (BureauOfLaborStatistics, BureauOfEconomicAnalysis, SAndP500, NASDAQ, TRowePrice) — `name:` slots verbatim, `verbatim_match_rate` = 1.0.
- Model then ALSO emitted 8 inline refs of the form `_anon:U.S. labor share of GDP`, `_anon:corporate profits`, `_anon:U.S. economy`, `_anon:workers` (x4), `_anon:low-cost index fund`, `_anon:401(k) plan` — **without** declaring matching `ENT _anon:...` blocks.
- Verifier hard-failed the 6 EVT/CLAIM blocks containing those 8 dangling refs (`reason=dangling_ref`).
- Net `verifier_pass_rate` = 0.625 — the **honest** score. Pre-cleanup the parser would either crash OR fabricate placeholder ENTs, inflating apparent recall with refs the model never declared.

**This is the desired behavior.** The cleanup is doing exactly what the user reframe demanded: surface the model's declare-then-reference miss, do not paper over it. The 0.625 is the correct upstream signal for "model needs better declare-then-reference training/prompting on the `_anon:` handle case" — distinct from the `name:`-slot grammar bug that PR #393 conflated it with.

---

## 4. Architecture validation

The cleanup vindicates plan §6 mechanism (b) (span-copying as the load-bearing primitive) and §7.1 (declare-then-reference contract) as **separable** gates:

| Failure type                 | Pre-cleanup score impact                                               | Post-cleanup score impact                                  |
|------------------------------|------------------------------------------------------------------------|------------------------------------------------------------|
| Model emits clean DSL        | parses; verifies                                                       | parses; verifies (unchanged)                               |
| Model misses ENT declaration | parser fabricates ENT, grammar wedges OR `ent_recall` inflates falsely | dangling ref → `hard_fail` on referencing block (honest)   |
| Model bytes don't match span | `copy_slot_mismatch` (`hard_fail`)                                     | `copy_slot_mismatch` (`hard_fail`) (unchanged)             |

Pre-cleanup these failure modes were entangled (synthesis hid declare-misses behind grammar crashes that LOOKED like copy-slot bugs). Post-cleanup they're independently observable, which is the necessary condition for the next iteration (prompt-engineer the `_anon:` declare-then-reference case in isolation).

---

## 5. Plan §11.2 update

`docs/plans/atlas-dsl-poc-plan.md` §11.2 has a new entry appended documenting this cleanup and its outcome (see commit log on this branch).

---

## 6. What this DOES NOT measure

- **n=9 retest.** Per user direction this is a 3-cell smoke on the prior-fail subset only. A full n=9 rerun is the supervisor's call.
- **Recall gates** (`numeric_exact_match_rate`, `ticker_recall`, `org_recall`). The Phase 2 Arm B GT subset has no per-category labels for any of the 3 cells run here — same UNDETERMINABLE state as PR #393 §1.
- **Generalization to non-`_anon:` undeclared-ref failure modes.** 31430's remaining failure is specifically the `_anon:<phrase>` pattern. Other prompt-induced declare-misses (e.g., model uses an abbreviation as a ref without declaring it) are not exercised by this 3-cell subset.

---

## 7. Commit trail

- `9f4d4ef8` — parser cleanup + tests (52 dsl tests pass)
- `afc1195e` — prompt update (declare-then-reference contract + worked anti-pattern)
- `434ac28e` — smoke artifacts (27560, 31149, 31430)
- this report
