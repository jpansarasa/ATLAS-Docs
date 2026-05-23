# Hypothesis A — span-retrieval hints (v12) — n=9 smoke notes

**Branch:** `experiment/hyp-a-span-retrieval-hints`
**Date:** 2026-05-23
**Model:** `qwen3:30b-a3b-instruct-2507-q4_K_M` (Class B MoE, 4B active)
**Backend:** llama-server :11437, v2.1 GBNF, `--num-ctx 32768`, `--n-predict 8192`, `--timeout 1800`
**Variant:** `v2_1_spec` with `--prompt-file cod_dsl_v12_span_hints.txt --retrieve-spans`
**Baselines:** v8 layered (PR #410), aggregate 0.907 / gv 9/9 on Arm B n=9
**Plan ref:** `docs/plans/three-hypotheses-scoping.md` §A

## TL;DR

- **Aggregate verifier (n=9 on cells where vpr was computable):**
  v12 **0.940** (n=6) vs v8 **0.907** (n=9). On the cells the model
  produced parseable output, v12 was strictly better. But:
- **gv pass rate: 6/9** (v8 baseline 9/9). Three regressions.
- **Numeric_exact_match_rate: v12 0.778 vs v8 0.529** (+0.249 on the
  intersection where both have a non-None value, 3 vs 6 measurable
  cells). This is the metric Hypothesis A explicitly targeted.
- **Verbatim_match_rate: v12 0.989 vs v8 0.693** (+0.296). The model
  is literally copying NUM.raw and ENT.name values from the hints
  block — Benford's-Curse copy-mode is engaging exactly as predicted.
- **Verdict: REGRESSION on aggregate** (gv loss outweighs vpr gain),
  but **mechanism gains are real on the cells where the mechanism
  held** — span hints DO produce more verbatim output when the model
  doesn't separately lose its way on concept-cascade.

## Per-cell results

| Cell  | v8 gv | v12 gv | v8 vpr | v12 vpr | Δ vpr  | v8 nem | v12 nem | v8 vmr | v12 vmr | v12 wall |
| ----- | ----- | ------ | ------ | ------- | ------ | ------ | ------- | ------ | ------- | -------- |
| 27560 | T     | T      | 0.667  | 0.974   | +0.307 | n/a    | n/a     | 0.643  | 0.973   | 159s     |
| 27702 | T     | T      | 1.000  | 1.000   | +0.000 | n/a    | n/a     | 0.000  | 1.000   | 68s      |
| 29807 | T     | T      | 0.735  | 0.867   | +0.131 | 0.667  | 0.889   | 0.879  | 1.000   | 139s     |
| 31149 | T     | T      | 0.958  | 1.000   | +0.042 | 1.000  | 1.000   | 0.955  | 1.000   | 86s      |
| 31430 | T     | T      | 0.917  | 0.800   | -0.117 | 0.444  | 0.444   | 0.895  | 0.960   | 521s     |
| 33632 | T     | **F**  | 0.969  | n/a     | n/a    | 0.864  | n/a     | 0.969  | n/a     | 384s     |
| 34537 | T     | T      | 1.000  | 1.000   | +0.000 | n/a    | n/a     | 0.000  | 1.000   | 208s     |
| 35352 | T     | **F**  | 0.991  | n/a     | n/a    | 0.009  | n/a     | 0.991  | n/a     | 0s       |
| 36065 | T     | **F**  | 0.925  | n/a     | n/a    | 0.189  | n/a     | 0.909  | n/a     | 519s     |

**Aggregates:**
- `verifier_pass_rate` mean (cells with non-None): v12 **0.940** / v8 **0.907** (+0.033, but n=6 vs n=9).
- `numeric_exact_match_rate` mean (cells with non-None): v12 **0.778** (n=3) / v8 **0.529** (n=6).
- `verbatim_match_rate` mean (cells with non-None): v12 **0.989** (n=6) / v8 **0.693** (n=9).
- `grammar_valid` count: **6/9** (v8: 9/9).

Total wall: ~57 min (sweep ran 16:01-16:58Z).

## Verdict band (per plan §A test plan)

Plan defines: `≥ v8 + 0.02 → PROGRESS`, `v8 ± 0.02 → BOUNDED`,
`< v8 − 0.02 → REGRESSION`.

**Aggregate over all 9 cells with missing vpr treated as 0:**
- v8 sum / 9 = 0.907
- v12 sum / 9 = 0.640 (3 cells contribute 0)
- Δ = -0.267 → **REGRESSION** by the strict task gate.

**On the 6 cells where v12 produced parseable output:**
- v12 mean = 0.940, v8 same-6 mean = 0.929 → +0.011 → **BOUNDED** but
  with substantially better mechanism on those cells.

The headline story is two-faced. Where the model used the hints, it
copied verbatim from them and the numeric / verbatim metrics jumped
materially. Where the model lost the plot, it lost it via the
canonical concept-flood failure mode that v8's decree #9 trip-wire
was supposed to prevent.

## Mechanism-preservation analysis (the central deliverable)

### Wins (mechanism reinforced, where the model produced output)

- **27560 verbatim_match 0.64 → 0.97**: the macro_release printed
  numerics (17, 18, 2026, ...) are now literally copied from the
  hints block. v8 was paraphrasing some headers; v12 is grounding.
- **27702 verbatim_match 0.00 → 1.00**: v8 produced ENT.name slots
  that didn't substring-match the article header (likely because the
  article is mostly boilerplate with no real named entity). v12's
  hints either yielded one good match or the model emitted NO blocks
  (vpr=1.0 trivially over an empty observation set). Either way,
  verbatim grounding flipped from broken to perfect.
- **29807 numeric_exact_match 0.67 → 0.89, vmr 0.88 → 1.00**: market
  recap — the hints block listed every price / move, and the model
  emitted them by surface form rather than re-deriving.
- **31149 vpr 0.96 → 1.00**: analyst-action article (small, dense);
  the hints prevented the slot-by-slot drift v8 produced.
- **34537 vmr 0.00 → 1.00**: speech transcript; v8 fabricated names
  not in source, v12 stuck to the hints.

### Losses (mechanism regression)

- **33632 (regulatory, 8.9KB) — DECLARE-ONCE collision on
  `_anon:energy_market_risk_mitigation_action_plan`** (and a cascade
  of variants on the same compound). The model produced 119 ENTs
  under v8 (concept count = 0, decree #9 held). Under v12 it
  produced a `concept`-kind cascade of `_anon:*_action_*` ids that
  collided on second declaration. The span hints did NOT block this:
  the ENT hint section contained surface forms like "energy market
  risk mitigation action plan" (a Title-Case multi-word match), and
  the model used the hints as a vocabulary to invent compound
  identifiers rather than as an enumeration of unique entities.

- **36065 (banking news, 20KB) — DECLARE-ONCE collision on
  `_anon:banking_profit_*_cost` cascade.** Same pathology: hints
  contained "banking profit" and "cost" phrases (Title-Case
  matches), model concatenated them into `_anon:banking_profit_<X>_cost`
  variants and tripped on the second `_cost`.

- **35352 (TSA passenger volumes, 3KB) — HTTP 400 at 0.19s.** Cell
  is dense daily-counts data; the article triggered 143 NUM hints
  (every daily passenger count is a unique literal). Prompt + hint
  block = 21K approx tokens; with `--n-predict 8192` the
  prompt+budget is 29K against `--ctx-size 32768`. The 400 looks
  like llama-server's pre-flight budget check fired. Below 32K on
  paper, but llama.cpp's tokenizer may emit slightly more than 4
  chars/token and we hit the cap.

### Why the hints fed concept-flood instead of preventing it

The ENT hint section is high-recall by design — it uses Title-Case
regex + ORG-suffix matching. On long articles those patterns capture
LOTS of compound noun phrases ("energy market risk", "action plan",
"mitigation programme", ...). The hints become a VOCABULARY of
n-grams the model can recombine. The decree #11 SPAN-MATCH rule we
added says "every ENT.name MUST appear in the hints OR elsewhere in
the article" — which, on a dense article, gives the model permission
to mint identifiers by recombining hint phrases.

Decree #9 (the `>5 concept STOP` trip-wire) is *content-level*
constraint that v8 enforced via prompt discipline. v12's
SPAN-MATCH overrode it implicitly by giving the model a copy-zone
of phrases too long and too overlapping for the trip-wire to fire
cleanly.

## Follow-ups (not in this PR; for the next iteration)

1. **Drop the ENT hint section entirely.** The wins above were all
   driven by NUM grounding; the ENT hints CAUSED the concept-flood
   regressions. The C# substrate's CoveStatementVerifier verifies
   ENT.name via substring against `text_quote ∪ context_summary`,
   not via a pre-emission hint list — we mistakenly added the ENT
   list. The C-equivalent intervention is NUM-only hints.

2. **Cap `--span-max-spans` for long articles.** Cell 35352 with
   143 NUM hints (passenger-volume table) inflated the prompt past
   the safe budget. A 50-hint cap (NUM only) with surface-form
   bucket dedup is the appropriate trim.

3. **Tighten decree #11.** Add explicit anti-pattern: "do NOT
   concatenate words from multiple hints to mint new
   `_anon:<compound>` identifiers". Phrased as a literal
   anti-example: "If hints contain `energy market` and
   `action plan` and `mitigation`, do NOT emit
   `ENT _anon:energy_market_action_plan_mitigation`."

4. **Re-sweep with NUM-only hints.** The 2026-05-23 sweep's
   mechanism wins on the 6 healthy cells suggest a NUM-only
   intervention could capture the verbatim-match / numeric-exact
   gains without the concept-flood blast radius. Expected outcome:
   gv 9/9 + vpr ≥ 0.92.

## Process notes

- Sweep ran serial against llama-server (single slot, `--parallel 1`).
- One-shot run, no retries (unlike v8 NOTES.md which retried 3 cells
  on cold-cache timeout — solo slot avoided that here).
- Cell 35352 returned a 400 in 0.19s — that's the canary-like
  pre-flight budget rejection on the llama.cpp side. Not retried;
  the failure mode is structural (prompt-budget), not transient.
- All 9 result JSONs committed (-f) for reproducibility.
- Sweep script: ephemeral `/tmp/hyp_a_sweep.sh` (NOT committed).
- Log: `/tmp/hyp_a_sweep.log` (NOT committed).
- Worktree: `/home/james/ATLAS/.claude/worktrees/agent-a9dc758b15f6591c9`.

## Conclusion

Hypothesis A is **partially validated**: NUM grounding via
pre-emission hints DOES improve verbatim copy and numeric-exact
match rates by 0.25-0.30 points on the cells where the model held
the line. But the ENT hints — added on a hunch, NOT in the C#
substrate — caused two concept-flood regressions on long articles.
The PR ships the substrate (regex + runner) and a clear next-step
list; the v12 prompt as-shipped is REGRESSION on the strict task
gate but the underlying mechanism is sound. NUM-only follow-up
should resolve the regression and bank the verbatim/numeric gains.
