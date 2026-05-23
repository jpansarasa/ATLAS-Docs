# Phase 3 v11 anti-concept-flood — n=9 sweep notes

**Branch:** `experiment/phase3-v11-anti-concept-flood-n9`
**Date:** 2026-05-23 (sweep wall: 03:46Z–04:22Z, ~36 min)
**Model:** `qwen3:30b-a3b-instruct-2507-q4_K_M` (Class B MoE, 4B active)
**Variant:** `v2_1_spec`
**Prompt:** `cod_dsl_v11_anti_concept_flood.txt` (931 lines = v8 846 + 85-line guard)
**Backend:** llama-server (`--parallel 1 --ctx-size 32768`)
**Baseline:** PR #410 v8 layered Arm B n=9 (aggregate 0.907, gv 9/9)

## TL;DR — VERDICT: REGRESSION (and triggers the GPU pivot)

- **Aggregate verifier (n=9): 0.824** (vs v8 0.907 → −0.083, −9.1 %)
- **gv pass rate: 8/9** (vs v8 9/9)
- **Cell 33632 (the flagship flood case): gv=F, vpr=0.000** — cap successfully
  suppressed `concept`, but the failure mode DISPLACED into `kind=event`
  and triggered a `DECLARE-ONCE` violation via a runaway repeating
  `ENT _anon:energy_market_stability / event` loop. The hard-fail path
  was redirected, not eliminated.
- **Cell 36065: 9 `kind=concept` ENTs** — cap was VIOLATED. Self-check
  directive did not fire. (vpr still 0.981 because the 9 concepts were
  well-anchored, but the GUARD MECHANISM FAILED.)
- Per the task's hard verdict gate:
  - aggregate 0.824 < 0.87 → **REGRESSION** band
  - 33632 gv-fail AND 36065 concept count > 5 → **REGRESSION regardless**
- Per the `sentinel-llm-strength-layering` memory contingent: **HARD PIVOT
  to GPU DSL-trained extraction track**. Small-model prompt-engineering
  has been exhausted.

## Per-cell results

| Cell  | v11 gv | v11 vpr   | v11 ent | v11 undecl | v11 concept | v8 vpr | v8 concept | Δ vpr   | Note                                                |
| ----- | ------ | --------- | ------- | ---------- | ----------- | ------ | ---------- | ------- | --------------------------------------------------- |
| 27560 | T      | 0.609     | 12      | 0          | 0           | 0.667  | 1          | −0.058  | minor regress                                       |
| 27702 | T      | 1.000     | 0       | 0          | 0           | 1.000  | 0          |  0.000  | tie                                                 |
| 29807 | T      | 0.912     | 14      | 2          | 0           | 0.735  | 0          | **+0.176** | unexpected gain                                  |
| 31149 | T      | 0.957     | 9       | 0          | 1           | 0.958  | 2          | −0.002  | tie                                                 |
| 31430 | T      | 0.960     | 11      | 0          | 2           | 0.917  | 0          | +0.043  | small gain                                          |
| 33632 | **F**  | **0.000** | 0       | 0          | 0           | 0.969  | 0          | **−0.969** | **catastrophic — flood displaced to `event`**    |
| 34537 | T      | 1.000     | 0       | 0          | 0           | 1.000  | 0          |  0.000  | tie                                                 |
| 35352 | T      | 1.000     | 1       | 0          | 0           | 0.991  | 0          | +0.009  | marginal gain                                       |
| 36065 | T      | 0.981     | 18      | 1          | **9**       | 0.925  | 3          | +0.056  | **cap VIOLATED — self-check did not fire**          |

**Aggregate v11: 0.8242   v8: 0.9069   paired Δ: −0.0827 mean (−0.7439 sum)**

## Critical mechanism finding: 33632 cap displacement

The CONCEPT-FLOOD GUARD section achieved its narrow stated goal on the
flagship cell — v11 emitted **0 `kind=concept` ENTs** for 33632, exactly
what `9A(a)` HARD CAP and the FORBIDDEN-CONTEXT LIST prescribed for a
regulatory-typed article. But the failure mode did not disappear; it
**displaced** into `kind=event`, faithfully following the GUARD's
priority-(ii) re-classification instruction.

Output of 33632 (first 1500 chars):

```
NOTE article_type
  - type: regulatory

ENT _anon:energy_costs / macro_indicator
  - name: "energy costs"
ENT _anon:energy_crisis / event
ENT _anon:war_on_Iran / event
ENT _anon:energy_market_stability / event
ENT _anon:supply_shortages / event
ENT _anon:price_spikes / event
ENT _anon:emergency_measures / event
... (~20+ more `event`-kind anon ENTs)
```

…and the tail of the output:

```
ENT _anon:energy_market_stability / event
ENT _anon:energy_market_stability / event
ENT _anon:energy_market_stability / event
... (~15 consecutive identical re-emissions)
END
```

The verifier hard-failed with:
`Duplicate ENT declaration: '_anon:energy_market_stability'` →
`grammar_valid = False`, `verifier_pass_rate = 0.000`.

**Diagnosis:** The model dutifully obeyed the FORBIDDEN-CONTEXT LIST
("regulatory rule names → `event`"), recategorized every concept candidate
as `kind=event`, then — under the same long-document flood pressure that
previously produced 124 concept ENTs in Phase A — entered a degenerate
repeat loop on a single `event`-kind ID. Decree #8 DECLARE-ONCE caught
it, but the cost was a complete cell loss. The guard did not address the
**root cause** (the model's inability to bound its own output length on
long regulatory articles); it only renamed the symptom.

## Mechanism preservation across 4 flagship cells

| Cell  | failure mode targeted        | v8 result   | v11 result               | preserved? |
| ----- | ---------------------------- | ----------- | ------------------------ | ---------- |
| 33632 | concept flood (124 in n=72)  | gv=T 0.969  | **gv=F 0.000** (event flood) | **NO — DISPLACED** |
| 36065 | concept over-emit            | gv=T 0.925 (3 concepts) | gv=T 0.981 (9 concepts)  | **NO — cap violated** |
| 27702 | clean baseline               | gv=T 1.000  | gv=T 1.000               | YES        |
| 31149 | undecl-ref fix from v6+CA    | gv=T 0.958  | gv=T 0.957               | YES        |

Two of four flagship cells regressed on their targeted mechanism. The
guard worked on cells where the failure wasn't actually present (27702,
31149) and failed where it was needed most (33632, 36065).

## Why the guard failed

1. **Cap violation on 36065** — the self-check directive `9A(c)` is a
   *verbal* instruction at the END of the prompt, identical in kind to
   v8's decree #9 ">5 concept STOP" trip-wire. It suffers the same
   weakness: the model treats it as guidance, not enforcement. Adding a
   second verbal directive does not turn a verbal directive into a
   counted cap; only a grammar constraint or post-hoc validator-driven
   retry can do that.

2. **Displacement on 33632** — the FORBIDDEN-CONTEXT LIST `9A(b)`
   correctly identified `kind=concept` as wrong for regulatory rules,
   but pointed the model at `kind=event` without ALSO bounding the
   number of `event`-kind ENTs. The model traded one flood (concept) for
   another (event), and DECLARE-ONCE caught it harder than the verifier
   would have caught the concept flood.

3. **Root cause confirmed** — the issue is NOT classification choice
   (which `kind` to assign); it is **output-length unboundedness on long
   regulatory articles**. The model cannot self-regulate its emission
   rate. A prompt directive cannot fix this; a grammar that counts ENT
   emissions or a sliding-window summarizer ahead of the extractor can.

## Verdict & contingent action

- Per-spec verdict bands:
  - aggregate ≥ 0.91 → PROGRESS — **NO** (0.824 < 0.91)
  - 0.87 ≤ aggregate < 0.91 → BOUNDED — **NO** (0.824 < 0.87)
  - aggregate < 0.87 → **REGRESSION** — **YES**
  - 33632 concept > 5 → REGRESSION regardless — **N/A** (33632 concept = 0
    but **gv=F** is an even worse failure that the verdict gate did not
    enumerate; should be treated as REGRESSION by the spirit of the rule)
  - 36065 concept > 5 → secondary REGRESSION signal — **YES** (9 concepts)

- **Final verdict: REGRESSION.**

- Per the user directive in the brief ("last small-model attempt;
  if this fails: HARD PIVOT to GPU DSL-trained extraction track") and
  the `sentinel-llm-strength-layering` memory, the small-model
  prompt-engineering workstream is **closed**. Next phase: GPU
  DSL-trained extraction (LoRA-adapted large model emitting CoD DSL
  directly, freed from class-B's output-length and self-regulation
  limits).

## Runtime artifacts

- **Results dir**: `docs/benchmarks/cod-2026-05-17/results/phase3-v11-anti-concept-flood-n9/qwen3_30b-a3b-instruct-2507-q4_K_M/v2_1_spec/`
- **Sweep log**: `/tmp/v11_sweep.log`
- **Per-cell wall times**: 65–855 s; total 36 min serial.
- **Anomalous cell**: 36065 took 855 s (vs v8 238 s) — likely the model
  attempting to satisfy the self-check, generating, revising, and
  ultimately exceeding the cap anyway.
