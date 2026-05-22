# Phase 3 v6 + context-aware (article-type to kind-priority steering) — n=9 sweep report

**Branch:** `experiment/phase3-v6-context-aware-n9`
**Date:** 2026-05-22
**Model:** `qwen3:30b-a3b-instruct-2507-q4_K_M` (Class B MoE, 4B active)
**Variant:** `v2_1_spec`
**Prompt:** `cod_dsl_v6_context_aware.txt` (769 lines, +135 vs v6 leaner 634)
**Baseline:** PR #400 v6 leaner (`phase3-v6-leaner-n9`, aggregate verifier 0.895)

## TL;DR

- **Aggregate verifier on grammar-valid cells (n=7): 0.895** — identical to v6
- **Paired delta on 7 common gv=True cells: +0.017** (0.895 vs 0.878)
- **Naive aggregate including gv=False as 0 (n=9): 0.696** — DOWN from v6's 0.895
- **2/9 grammar failures** vs v6's 0/9: 33632 (duplicate ENT) and 35352 (n_predict limit)
- **9/9 type assignments match human intuition** — classification is not the bottleneck
- **Verdict: REGRESSION** — naive aggregate 0.696 < 0.87 threshold

## Intervention

v6_context_aware = v6 leaner + two additive sections:

1. **ARTICLE TYPE CLASSIFICATION** preamble (after the intro, before
   GRAMMAR). Closed-set taxonomy of 7 types:
   `earnings_announcement | macro_release | analyst_action | m_and_a |
    guidance | regulatory | other`. Model emits the type as a
   `NOTE article_type` block — the FIRST block after the
   `DSL/SOURCE/TIMESTAMP` header.

2. **Decree #9 KIND PRIORITY is TYPE-CONDITIONAL**: per-type table
   listing preferred ENT kinds. e.g., `earnings_announcement` ->
   `org / instrument / num-ref / person`; `macro_release` ->
   `macro_indicator / country / region / num-ref`. `regulatory` is the
   one type that PERMITS `concept` as a preferred kind (for
   doctrine-shaped rule names like "MiFID II" or "Section 871(m)").

**Grammar approach: zero grammar changes.** NOTE blocks already accept
arbitrary `- type:` slot lines per v2.1 GBNF
(`note-block ::= "NOTE" note-label-opt "\n" source-span-slot? generic-slot*`).
The classification rides as a single labeled NOTE; the parser ingests
it as a regular block and the report tooling greps `NOTE article_type`
from `raw_output` for the type assignment.

## Per-cell results (v6_context_aware)

| Cell  | TYPE                | gv | verifier | ent | evt | note | undecl | eval | prompt | wall (s) | stop  |
| ----- | ------------------- | -- | -------- | --- | --- | ---- | ------ | ---- | ------ | -------- | ----- |
| 27560 | macro_release       | T  | 0.800    | 2   | 14  | 1    | 0      | 1305 | 8,745  | 1085.2   | eos   |
| 27702 | other               | T  | 1.000    | 0   | 0   | 1    | 0      | 86   | 8,776  | 469.5    | eos   |
| 29807 | macro_release       | T  | 0.943    | 14  | 12  | 1    | 1      | 1126 | 8,798  | 1465.8   | eos   |
| 31149 | analyst_action      | T  | 0.682    | 1   | 6   | 1    | 11     | 1090 | 8,679  | 878.4    | eos   |
| 31430 | macro_release       | T  | 0.889    | 8   | 3   | 1    | 0      | 653  | 9,053  | 71.7     | eos   |
| 33632 | regulatory          | F  | n/a      | 0   | 0   | 0    | 0      | 3731 | 10,299 | 242.0    | eos   |
| 34537 | other               | T  | 1.000    | 0   | 0   | 1    | 0      | 61   | 8,585  | 43.6     | eos   |
| 35352 | macro_release       | F  | n/a      | 0   | 0   | 0    | 0      | 5416 | 10,968 | 398.1    | limit |
| 36065 | other               | T  | 0.951    | 11  | 9   | 1    | 1      | 1692 | 12,647 | 329.3    | eos   |

Aggregate verifier on grammar-valid (n=7): **0.895**.
Naive aggregate with gv=False as 0 (n=9): **0.696**.

## Type-assignment intuition check

| Cell  | Source                                                    | Model TYPE          | Human intuition       | Match?  |
| ----- | --------------------------------------------------------- | ------------------- | --------------------- | ------- |
| 27560 | Fed monetary press release feed                           | macro_release       | macro_release         | YES     |
| 27702 | Fusion Media boilerplate / fake Form 144 URL              | other               | other                 | YES     |
| 29807 | macro / multi-index summary article                       | macro_release       | macro_release         | YES     |
| 31149 | Jefferies on Canadian banks (sell-side outlook)           | analyst_action      | analyst_action        | YES     |
| 31430 | macro release with multiple central-bank refs             | macro_release       | macro_release         | YES     |
| 33632 | fuel-tax-cut policy / govt action (Philippines, Malaysia) | regulatory          | regulatory            | YES     |
| 34537 | executive list / corporate boilerplate                    | other               | other                 | YES     |
| 35352 | BLS daily passenger volume series                         | macro_release       | macro_release         | YES     |
| 36065 | broad sector / market recap                               | other               | other                 | YES     |

**9/9 type assignments match human intuition.** Classification capability
is not the bottleneck — the model picks plausible types reliably under
this prompt. The mechanism issue is what happens DOWNSTREAM of the
classification.

## Side-by-side delta vs v6 leaner (paired)

| Cell  | v6 verifier | new verifier | delta      | v6 gv | new gv | Notes                                                       |
| ----- | ----------- | ------------ | ---------- | ----- | ------ | ----------------------------------------------------------- |
| 27560 | 0.667       | 0.800        | +0.133     | T     | T      | macro_release steering helped (still only 2 ENT, was 5)     |
| 27702 | 1.000       | 1.000        | +0.000     | T     | T      | `other` -> no-op; identical                                 |
| 29807 | 0.812       | 0.943        | +0.130     | T     | T      | macro_release: more diverse kinds (instrument/region added) |
| 31149 | 0.867       | 0.682        | -0.185     | T     | T      | analyst_action regression: 1 ENT decl + 11 undecl refs      |
| 31430 | 0.905       | 0.889        | -0.016     | T     | T      | marginal, within noise                                      |
| 33632 | 0.922       | n/a          | **n/a (LOSS)** | T  | **F**  | `concept x76` (regulatory exception) + DECLARE-ONCE break   |
| 34537 | 1.000       | 1.000        | +0.000     | T     | T      | `other` -> no-op; identical                                 |
| 35352 | 0.991       | n/a          | **n/a (LOSS)** | T  | **F**  | stop=limit (n_predict 6144) — 1K-token extra prompt budget  |
| 36065 | 0.892       | 0.951        | +0.059     | T     | T      | concept x4 -> x1 (good); kind-latch tightened               |

**Paired (n=7 common gv=True): v6_context_aware = 0.895, v6 = 0.878, delta = +0.017.**
**Naive aggregate (n=9, gv=False as 0): v6_context_aware = 0.696, v6 = 0.895, delta = -0.199.**

## Kind-distribution shift (top kinds, v6 vs v6_context_aware)

| Cell  | TYPE                | v6 kinds (top 6)                                          | new kinds (top 6)                                         |
| ----- | ------------------- | --------------------------------------------------------- | --------------------------------------------------------- |
| 27560 | macro_release       | org:3, macro_indicator:1, concept:1                       | org:2                                                     |
| 27702 | other               | org:2                                                     | (no ENTs)                                                 |
| 29807 | macro_release       | index:6, org:3, equity:2, country:2                       | org:5, index:4, instrument:2, country:2, region:1         |
| 31149 | analyst_action      | macro_indicator:2, analyst_firm:1, org:1, region:1, sector:1 | analyst_firm:1                                         |
| 31430 | macro_release       | org:4, index:2, macro_indicator:2, person:1, concept:1    | org:3, index:2, macro_indicator:2, person:1               |
| 33632 | regulatory          | country:33, macro_indicator:1, org:1                      | **concept:76**, country:34, macro_indicator:1, sector:1   |
| 34537 | other               | person:7, org:1                                           | (no ENTs)                                                 |
| 35352 | macro_release       | macro_indicator:1                                         | macro_indicator:1                                         |
| 36065 | other               | sector:7, equity:5, country:5, org:4, concept:4, person:1 | org:7, macro_indicator:3, concept:1                       |

**Did kinds shift in expected directions per the per-type preferred table?**

- **macro_release cells (27560, 29807, 31430, 35352)**: mixed signal.
  29807 ADDED `instrument:2` and `region:1` (both in the macro_release
  preferred set), and `country` kept its share — directionally correct.
  27560 actually LOST entities (5 -> 2), suggesting the steering
  over-pruned rather than re-shaped. 31430 and 35352 are stable.

- **analyst_action cell (31149)**: REGRESSION. v6 had a richer mix
  (`macro_indicator:2, analyst_firm:1, org:1, region:1, sector:1`);
  new dropped to a single `analyst_firm:1` and then references 11
  entities that were never declared. The preferred-kinds table biased
  toward `{org, person, instrument, num-ref}` but the model under-
  declared rather than re-declared with the new kinds — net loss.

- **regulatory cell (33632)**: CATASTROPHIC REGRESSION. The per-type
  exception that permits `concept` as a preferred kind for
  `regulatory` produced **concept:76 entities** — exactly the
  concept-latch failure mode that decree #9's ">5 concept ENTs ->
  STOP" trip-wire was added to prevent. Combined with a DECLARE-ONCE
  violation (`_anon:excise_duties_cut` declared twice), the cell
  hard-failed at grammar.

- **other cells (27702, 34537, 36065)**: mixed. 36065 actually
  IMPROVED — `concept:4 -> concept:1`, suggesting the `other`-type
  fallback to v6 default priority works. 27702 and 34537 produced
  zero ENTs in both v6 and the new prompt (correctly identifying them
  as boilerplate / non-substantive).

## Mechanism analysis (gv failures)

### 33632 — regulatory exception backfired (concept x76)

- Article: long fuel-tax-cut policy article spanning multiple
  countries (Philippines, Malaysia, India, ...) with repeated mentions
  of "fuel tax cut", "excise duties", etc.
- TYPE: classified as `regulatory` (CORRECT).
- The per-type rule for `regulatory` says `concept` IS a preferred
  kind for "doctrine-shaped rule names like Section 871(m), MiFID II,
  Reg SHO". The model interpreted this VERY liberally and emitted
  `concept:76` for every fuel-tax-related noun phrase
  ("excise_duties_cut", "fuel_subsidy", "policy_response", etc.).
- DECLARE-ONCE then fired: `_anon:excise_duties_cut` got declared
  twice -> parser hard_fails with
  `Duplicate ENT declaration '_anon:excise_duties_cut'`.
- **Root cause:** the regulatory-permits-concept exception is too
  permissive. The model treats it as "free pass on concept-latch"
  rather than "specific rule names only". v6's blanket
  ">5 concept STOP" trip-wire was the load-bearing mechanism;
  removing the trip-wire for ONE type took down the whole cell.

### 35352 — stop=limit (n_predict ceiling hit)

- Article: BLS daily passenger volume series (long table of figures).
- TYPE: classified as `macro_release` (CORRECT).
- The new prompt added ~1K tokens (8,776 -> 10,968 prompt_eval); with
  num_ctx=16384 and n_predict=6144, the available output budget for
  this cell shrank by the prompt delta.
- v6 ALSO hit n_predict=6144 on this cell (eval=6144, stop=limit)
  but its raw_output happened to terminate cleanly with `END`
  inside the cap. The new prompt's tighter budget pushed past the
  cleanup point: the parser raised
  `Grammar error: Unexpected token Token('$END', '') at line 566,
   column 34. Expected one of: SLOT_LINE_TRIGGER` — i.e. EVT block
  truncated mid-slot.
- **Root cause:** prompt-size inflation eats into the deterministic
  output budget. The classification preamble + per-type table cost
  ~1K tokens, and on long-output cells the model has nowhere to
  recover.

### 31149 — undecl explosion under analyst_action steering

- Article: Jefferies sell-side outlook on Canadian banks; mentions
  multiple banks, the Bank of Canada, CUSMA, AI, labour markets,
  oil prices.
- TYPE: `analyst_action` (CORRECT).
- New prompt declared only ONE entity (`Jefferies`) and then
  referenced 11 others (`BankOfCanada`, `_anon:CUSMA`, etc.) without
  declaring them — DECLARE-THEN-REFERENCE violation produced 11
  `undeclared_ent_count`, dropping the verifier from 0.867 to 0.682.
- The `analyst_action` preferred-kinds table biases toward
  `{org, person, instrument, num-ref}` — directionally fine for this
  article — but the model interpreted "bias toward these kinds" as
  "only declare these kinds, reference everything else" rather than
  "declare these kinds when ambiguous". The DECLARE-THEN-REFERENCE
  decree was not strengthened to counter this regression.

## Wall-time

- Total sweep: 10:49Z -> 12:28Z = **~100 minutes** (11 polls; mid-run
  script restart at 11:55Z after a background-task termination at
  11:40Z; the sweep is resumable because `out_path.exists()`
  short-circuits already-completed cells).
- 3-way contention with Exp D (v7 leaner) and Exp F (v6+RAG) sharing
  llama-server slots inflated per-cell wall by ~3x vs solo v6
  baseline.

## Verdict: REGRESSION

- **Naive aggregate verifier 0.696 < 0.87 threshold -> REGRESSION.**
- Aggregate on gv-valid subset (0.895) ties v6, but two grammar
  failures cancel that signal.
- gv rate dropped from 9/9 (v6) to 7/9 — the mechanism guards from
  decree #8 (DECLARE-ONCE) and decree #9 (concept-latch trip-wire)
  are FRAGILE to the type-conditional intervention.
- Type-assignment capability is solid (9/9 plausible).
- Kind-distribution shifts were directionally correct on macro_release
  cells but produced regressions on regulatory (concept-latch
  exception backfired) and analyst_action (under-declare + over-ref).

**Net interpretation:** The intervention proved that the model CAN
classify financial-news templates reliably (9/9 plausible). But the
type-conditional preferred-kinds steering is too coarse a knob —
on cells where v6's blanket mechanism guards were load-bearing
(33632 concept-latch, 31149 declare-then-ref), the type-specific
steering instructions DISPLACED rather than ADDED to the guards. The
net token-budget cost (~1K extra prompt) also tipped one long-output
cell (35352) past the n_predict ceiling that v6 was already grazing.

## Possible next steps (NOT taken in this experiment)

- Tighten the type-conditional table to ADD preferences without
  REMOVING guards — e.g., keep the ">5 concept STOP" trip-wire ON
  for `regulatory` and require concept names to match a regex like
  `(Section \d+|Reg [A-Z]+|MiFID|Basel)` rather than a free-text noun.
- Bump n_predict to 8192 to recover the long-output budget the
  classification preamble consumed.
- For `analyst_action` cells, add a DECLARE-MANY-OR-NONE counter-
  rule: if you reference >3 entities, declare ALL of them, do not
  short-circuit to "the firm only declared, the rest implied".
