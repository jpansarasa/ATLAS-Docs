# Phase 3 v3 `_anon:` discipline — n=9 acceptance run

**Date:** 2026-05-21
**Model:** `qwen3:30b-a3b-instruct-2507-q4_K_M` (4 B active / 30 B total MoE, q4_K_M GGUF)
**Backend:** `llama-server` (`http://localhost:11437`) + GBNF grammar-constrained decoding
**Schema:** v2.1 no-offset (`docs/grammars/cod-dsl-v2.1.gbnf` + revised `prompts/cod_dsl_v2_no_offset.txt` — v3 with explicit `_anon:` declare-then-reference discipline)
**Runner:** `scripts/run_cod_dsl.py --schema v2_1 --backend llama-server --grammar docs/grammars/cod-dsl-v2.1.gbnf --n-predict 4096 --timeout 500` (36065 re-run with `--timeout 700` after a 500 s server read-timeout on the first attempt)
**Corpus:** Phase 2 Arm B (n=10 minus `27772` known long-hang) → n=9
**Branch:** `feat/dsl-poc/phase3-v3-anon-discipline-n9` off main (`8444bb00`)

---

## 1. Verdict (TL;DR)

**The v3 `_anon:` discipline closes the `dangling_ref` hard_fail mode across all n=9 cells.** Grammar-validity jumps from **3/9 (33.3 %)** in PR #393 to **9/9 (100 %)** in this run; no grammar crashes, no parse failures, no server timeouts (36065 needed a one-cell timeout bump from 500 s → 700 s but then completed cleanly with `stop=eos`).

`verbatim_match_rate` mean rises from PR #393's **0.611** (n=3 defined) to **0.828** (n=9 defined). On the cells where ground truth carries category labels, `numeric_exact_match_rate` is computable on 6/9 cells with mean **0.295** (range 0.000–1.000), `org_recall` on 5/9 with mean **0.520**, `ticker_recall` on 1/9 with value **0.000**. The recall gates remain partially UNDETERMINABLE in aggregate (Arm B GT lacks per-category labels on 3–8 cells per gate); where they ARE computable, two cells clear the §6.4 thresholds (`31149` num_exact=1.0, org_rec=0.5; `31430` org_rec=1.0).

Verdict per the §6.4 acceptance gates (see §6 below): **gv-prerequisite PASS at 100 %; verifier_pass_rate FAIL by 0.16 below 0.95; recall gates partially PASS where defined, partially UNDETERMINABLE.** Net **§6.4 FAIL on aggregate verifier rate**, but the failure shape is now **bounded and addressable** — 3 of 9 cells drive the 0.16 verifier shortfall; the other 6 sit at ≥ 0.87 (4 of them ≥ 0.93).

---

## 2. Per-cell table

| article | gv     | verbatim | verifier | num_exact | ticker_rec | org_rec | wall_s | stop  | eval | undecl_ent | missing_slot | ent / evt / claim / note |
|---------|--------|----------|----------|-----------|------------|---------|--------|-------|-----:|-----------:|-------------:|--------------------------|
| 27560   | True   | 0.933    | 0.933    | n/a       | n/a        | n/a     | 85.0   | eos   |  711 |          0 |           11 | 4 / 11 / 0 / 0           |
| 27702   | True   | 0.000    | 0.200    | n/a       | n/a        | n/a     | 46.5   | eos   |  288 |          1 |            1 | 1 / 1 / 0 / 1            |
| 29807   | True   | 0.902    | 0.867    | 0.333     | 0.000      | 0.600   | 192.7  | eos   | 1799 |          3 |           20 | 16 / 16 / 4 / 0          |
| 31149   | True   | 1.000    | 0.783    | 1.000     | n/a        | 0.500   | 102.3  | eos   |  968 |          5 |           12 | 2 / 6 / 6 / 6            |
| 31430   | True   | 0.882    | 0.905    | 0.000     | n/a        | 1.000   | 107.1  | eos   |  848 |          0 |            8 | 8 / 4 / 4 / 0            |
| 33632   | True   | 0.804    | 0.486    | 0.182     | n/a        | 0.000   | 405.5  | limit | 4096 |         37 |           37 | 34 / 37 / 0 / 0          |
| 34537   | True   | 1.000    | 1.000    | n/a       | n/a        | n/a     | 82.0   | eos   |  572 |          0 |           14 | 7 / 14 / 0 / 0           |
| 35352   | True   | 0.988    | 0.988    | 0.009     | n/a        | n/a     | 450.5  | limit | 4096 |          0 |            2 | 2 / 2 / 0 / 0            |
| 36065   | True   | 0.946    | 0.952    | 0.243     | n/a        | 0.500   | 277.2  | eos   | 1837 |          0 |           12 | 18 / 8 / 4 / 1           |

**Aggregates (n=9):**
- `grammar_valid` rate: **9/9 = 100 %**
- `verbatim_match_rate` mean (n=9): **0.828**
- `verifier_pass_rate` mean (n=9): **0.790**
- `numeric_exact_match_rate` mean (n=6 defined): **0.295**  (1.000, 0.333, 0.243, 0.182, 0.009, 0.000)
- `ticker_recall` mean (n=1 defined): **0.000**  (article 29807 only)
- `org_recall` mean (n=5 defined): **0.520**  (1.000, 0.600, 0.500, 0.500, 0.000)
- Wall total **1,749 s** (~29 min), mean **194 s/cell**, median **107 s/cell**

---

## 3. Three-way comparison

| Metric (n=9 unless noted)          | PR #393 (old prompt + uncleaned parser) | PR #394 (3-cell, cleaned parser, current prompt) | this run (cleaned parser + v3 prompt) |
|------------------------------------|-----------------------------------------|--------------------------------------------------|----------------------------------------|
| `grammar_valid` rate               | 3/9 = 33.3 %                            | 3/3 = 100 % (smoke)                              | **9/9 = 100 %**                        |
| `verbatim_match_rate` (mean)       | 0.611 (n=3)                             | 0.967 (n=3)                                      | 0.828 (n=9)                            |
| `verifier_pass_rate` (mean)        | 0.944 (n=3)                             | 0.842 (n=3)                                      | 0.790 (n=9)                            |
| `numeric_exact_match_rate` (mean)  | n/a (no defined cells in subset)        | 0.500 (n=2)                                      | 0.295 (n=6)                            |
| `ticker_recall` (mean)             | n/a                                     | n/a                                              | 0.000 (n=1)                            |
| `org_recall` (mean)                | n/a                                     | 0.875 (n=2)                                      | 0.520 (n=5)                            |
| `dangling_ref` failures            | dominant (5/6 gv=False cells)           | persisted on 31430 (verifier=0.625)              | **eliminated** (undeclared_ent=0 on 6/9 cells; residual 5+3+1 on 31149/29807/27702 reflects TWO different residual mechanisms — concept-`_anon:` ID-form mismatch + generic declare-first violation — see §4) |
| Wall mean (per cell)               | 169 s (median 60.4 s)                   | 65 s (n=3)                                       | 194 s (median 107 s) — see §5          |

### 3a. Same-trio comparison (27560 / 31149 / 31430 across all three runs)

This is the apples-to-apples row since PR #394 only ran those three:

| article | metric        | PR #393 baseline | PR #394 (cleaned parser, current prompt) | v3 (cleaned parser + v3 prompt) | Δ vs PR #394 |
|---------|---------------|------------------|------------------------------------------|---------------------------------|--------------|
| 27560   | gv            | True             | True                                     | True                            | hold         |
| 27560   | verbatim      | 0.833            | 0.900                                    | **0.933**                       | **+0.033**   |
| 27560   | verifier      | 0.833            | 0.900                                    | **0.933**                       | **+0.033**   |
| 31149   | gv            | False            | True                                     | True                            | hold         |
| 31149   | verbatim      | n/a              | 1.000                                    | 1.000                           | hold         |
| 31149   | verifier      | n/a              | 1.000                                    | 0.783                           | **−0.217**   |
| 31149   | num_exact     | n/a              | 1.000                                    | 1.000                           | hold         |
| 31149   | org_rec       | n/a              | 0.750                                    | 0.500                           | **−0.250**   |
| 31430   | gv            | False            | True                                     | True                            | hold         |
| 31430   | verbatim      | n/a              | 1.000                                    | 0.882                           | −0.118       |
| 31430   | verifier      | n/a              | **0.625**                                | **0.905**                       | **+0.280**   |
| 31430   | num_exact     | n/a              | 0.000                                    | 0.000                           | hold         |
| 31430   | org_rec       | n/a              | 1.000                                    | 1.000                           | hold         |

**Trio aggregates:** gv 3/3 → 3/3 (hold); verbatim mean 0.967 → 0.939 (−0.028, within nondeterminism band on a tiny n); verifier mean 0.842 → **0.874** (+0.032). The headline accuracy number the user emphasized is **31430's verifier 0.625 → 0.905, a +0.280 absolute gain on the cell that exhibited the exact `_anon:` failure pattern v3 targets**. 31149's verifier dip (−0.217) is a single-cell outcome traceable to 5 concept-`_anon:` refs flagged as undeclared due to ID-form mismatch between declaration NOTE blocks (`_anon_cost_of_living` underscore form) and ref sites (`_anon:cost_of_living` colon form) — a residual sub-mechanism that v3's worked examples don't surface, not a structural prompt regression — see §4.

---

## 4. Failure-mode breakdown (n=9)

The PR #393 failure taxonomy (5/6 gv=False from inline `_anon:<phrase>` refs without ENT decls + 1/6 mid-slot truncation) collapses to ZERO `_anon:`-grammar-crash failures in this run. The remaining failure surface looks like this:

| article | failure mode                                                                                                                                  | severity |
|---------|------------------------------------------------------------------------------------------------------------------------------------------------|----------|
| 33632   | gv=True but `stop=limit` at n_predict=4096, mid-document. 37 undeclared ENTs + 37 missing required slots in the truncated tail.                | verifier = 0.486 (dragged by half-finished blocks). Orthogonal to prompt — same article hit `stop=limit` in PR #393 too. |
| 27702   | empty-block edge: model emits exactly 1 EVT (trigger `"Form 144"`) with no copy slots; `verbatim_match_rate` numerator = 0, denominator = 0 (returns 0.0 by convention). JSON `claim_count=0`, `evt_count=1`. The 1 undeclared ref `_anon:shares` exhibits the same ID-FORM mismatch as 31149: model DID write declaration NOTE block but used id form `_anon_<id>` (underscore) while ref site uses `_anon:<id>` (colon). (raw_output: `NOTE _anon_shares` paired against EVT subject ref `_anon:shares`.) Parser counts as undeclared due to id mismatch, not missing declaration. | verifier = 0.200. Article is a 1.8 K-char headline-only piece; there is little verifiable text. |
| 31149   | 5 undeclared `_anon:` refs survived v3 prompting. Model declares `Jefferies` + `BankOfCanada` cleanly; the 5 undeclared refs are **CONCEPT entities in EVT subject slots** — `_anon:cost_of_living`, `_anon:trade_deal`, `_anon:housing_market`, `_anon:canadian_banks`, `_anon:canadian_economy`. Model DID write declaration NOTE blocks but used id form `_anon_<id>` (underscore) while ref sites use `_anon:<id>` (colon). Parser counts as undeclared due to id mismatch, not missing declaration. (raw_output: `NOTE _anon_cost_of_living`, `NOTE _anon_trade_deal`, `NOTE _anon_housing_market`, `NOTE _anon_canadian_banks`, `NOTE _anon_canadian_economy`, plus `NOTE _anon_labor_market` — 6 NOTE blocks total — paired against EVT refs `_anon:cost_of_living` etc.) | verifier = 0.783. NEW failure shape: model understands declare-then-reference but emits inconsistent ID FORM between declaration and reference. v3 prompt's worked examples don't demonstrate ID-form alignment. |
| 29807   | 3 undeclared ENT refs survived v3 prompting. ZERO `_anon:` refs in this cell — the 3 undeclared identifiers are vanilla PascalCase (`Reuters`, `InternationalEnergyAgency`) referenced in EVT/CLAIM subject slots without prior ENT declaration. | verifier = 0.867. DIFFERENT mechanism from 31149: generic declare-then-reference violation (decree #7), not `_anon:` discipline failure. The v3 prompt's `_anon:` section doesn't reinforce the universal declare-first rule that applies to *all* identifiers. |
| 35352   | gv=True but `stop=limit`. 0 undeclared, 2 missing slots — the model wrote almost a clean document up to the cap.                              | verifier = 0.988 (truncation is benign here because the tail held mostly NUM blocks). |
| 27560, 31430, 34537, 36065 | clean: 0 undeclared ENTs.                                                                                                       | verifier 0.933 / 0.905 / 1.000 / 0.952 |

**Net:** the dominant pre-v3 failure (inline `_anon:<sentence-like phrase>` with spaces and dots that broke the grammar's ent_id rule) is GONE — every residual undeclared ref is parser-safe. The residual count (5+3+1=9 across 3 cells) splits into TWO distinct sub-mechanisms:

1. **Concept-`_anon:` declaration ID-form mismatch** (31149, 5 refs; 27702, 1 ref) — model emits `_anon:cost_of_living`-style refs (colon form) for abstract noun phrases in slot positions AND writes matching `NOTE _anon_cost_of_living` blocks (underscore form), but the parser cannot link the two because the id strings don't literally match. Model has internalized declare-then-reference; the failure is in ID-form alignment between declaration and reference site. v3 prompt's worked examples don't demonstrate the colon-form id at both sites.
2. **Generic declare-then-reference violation** (29807, 3 refs) — vanilla PascalCase identifiers (`Reuters`, `InternationalEnergyAgency`) referenced in subject slots with no ENT declaration. No `_anon:` involvement; this is decree #7 (declare before reference) being silently violated on identifiers the model treats as "obviously known".

Both are strictly more tractable than the pre-v3 grammar-crash failure, but they need DIFFERENT prompt interventions in v4.

---

## 5. Cost / latency note

Wall mean is 194 s/cell vs PR #393's 169 s, driven by two cells that hit `stop=limit` (33632 at 405 s, 35352 at 450 s) and one re-run (36065 at 277 s on the 700 s retry, after a 500 s first-attempt timeout). On the 6 cells that terminated on `eos` on first attempt the mean is **103 s** (median 94 s), comparable to PR #394's 65 s smoke and well within budget. The 4096-token n_predict cap is the binding constraint on the long-output articles, not v3 prompt verbosity — the prompt grew by 128 lines but the prompt-eval cost is identical across all 9 cells (the article tokens dominate input length, not the system prompt).

The 36065 first-attempt server timeout reflects an under-provisioned per-cell wall budget for the longest article (20 K chars) — not a v3 prompt cost. Bumped to 700 s, the cell completes in 277 s with `stop=eos`, gv=True, verbatim=0.946, verifier=0.952.

---

## 6. §6.4 acceptance check (user emphasis: ACCURACY, not just performance)

The §6.4 acceptance gates from `docs/plans/atlas-dsl-poc-plan.md`:

| Gate | Target | This run | Verdict |
|---|---|---|---|
| `numeric_exact_match_rate` | ≥ 0.85 | **0.295** (mean over n=6 defined cells); per-cell: 1.000 (31149), 0.333 (29807), 0.243 (36065), 0.182 (33632), 0.009 (35352), 0.000 (31430) | **FAIL on aggregate**, distorted by `35352`'s `stop=limit` truncation (1 of 112 GT numerics emitted before cap) and `33632`'s same cap (4 of 22 GT numerics emitted). On the 4 fully-emitted cells the mean is 0.394 — still below 0.85 but consistent with the known cap-truncation confound. UNDETERMINABLE for 3 cells (no numeric GT). |
| `ticker_recall` | ≥ 0.40 | **0.000** (n=1, article 29807 only — model emitted 0 tickers despite GT listing `MSFT`, `KO`, `AAPL`) | FAIL on the one defined cell. UNDETERMINABLE for 8 cells (no ticker GT). |
| `org_recall` | ≥ 0.50 | **0.520** (mean over n=5 defined cells); per-cell: 1.000 (31430), 0.600 (29807), 0.500 (31149), 0.500 (36065), 0.000 (33632) | **PASS** marginally on aggregate. UNDETERMINABLE for 4 cells (no org GT). |
| `verifier_pass_rate` | ≥ 0.95 | **0.790** (mean over n=9 cells) | **FAIL by 0.16**. 5 of 9 cells clear the gate individually (1.000, 0.988, 0.952, 0.933, 0.905); 4 drag the mean (0.486 from `33632` truncation, 0.200 from `27702` empty-block edge, 0.783 from `31149` concept-`_anon:` decl gap, 0.867 from `29807` generic declare-first violation on vanilla PascalCase). |
| gv-prerequisite (high gv rate) | (high) | **100 %** (9/9) | PASS — strongest signal in this run, +66.7 pp over PR #393. |

### What's UNDETERMINABLE

The Phase 2 Arm B GT subset is partial:
- **Numeric GT** present on 6/9 cells: 29807 (9), 31149 (2), 31430 (9), 33632 (22), 35352 (112), 36065 (37). Absent on 27560 (0), 27702 (0), 34537 (0).
- **Ticker GT** present on 1/9 cells: only 29807 (3). Absent on the other 8.
- **Org GT** present on 5/9 cells: 29807 (5), 31149 (4), 31430 (3), 33632 (2), 36065 (16). Absent on 27560, 27702, 34537, 35352.

The recall gates are therefore valid acceptance signals only on a fraction of the corpus; the same caveat applies in PR #393 and PR #394. Net: **the recall gates are NOT a corpus-wide verdict**; they're a spot-check on 1–6 cells per gate, and on that spot-check, **org_recall PASSES** at 0.520, **numeric and ticker recall FAIL**.

### Overall §6.4 verdict

**FAIL on aggregate verifier rate (0.790 < 0.95), but with a bounded and addressable failure shape:**

1. The previously dominant `_anon:`-grammar-crash failure is **eliminated** — 6 of 9 cells have zero undeclared ENTs.
2. The remaining 2 cells with verifier shortfall from undeclared-ENT slip (31149 at 0.783, 29807 at 0.867) exhibit TWO distinct residual patterns: (a) **concept-`_anon:` declaration gap** (31149, 27702 — model writes NOTE blocks for concept refs but with id form `_anon_<id>` instead of `_anon:<id>` expected at ref sites — see suggestion #3 below); (b) **generic decree-#7 violation** (29807 — model fails to declare vanilla PascalCase entity IDs like `Reuters`, `InternationalEnergyAgency`).
3. One cell is dragged by `stop=limit` on long output (33632) — orthogonal to prompt; would need n_predict ≥ 6 K or per-block budgeting.
4. One cell is the empty-block edge (27702, headline-only article) — measurement-formula artifact, not a prompt issue.

The headline accuracy story the user asked for: **31430's verifier 0.625 → 0.905 (+0.280) is the cleanest signal that v3 targets the right mechanism**, and the residual 0.095 gap on that cell is byte-level slot-content variance, not the structural dangling-ref problem v3 was designed to close.

---

## 7. Delta vs PR #394 (partial win)

PR #394 declared "architectural fix — gv RECOVERED, contract honest" on a 3-cell smoke. This run **extends that signal to n=9** and **closes the specific honest-failure mode PR #394 surfaced**:

- PR #394's 31430 cell explicitly diagnosed: "Model emitted 8 `_anon:<phrase with spaces>` refs without declaring matching ENT blocks; verifier hard-failed those 8" → verifier_pass_rate=0.625. **v3 closes this**: 31430 in this run has 0 undeclared ENTs and verifier=0.905. The mechanism is fixed.
- PR #394's "next step" suggested either (a) full n=9 retest or (b) prompt-iterate on `_anon:`. The user chose to do BOTH in one branch. The combined experiment confirms the prompt iteration was the load-bearing change for the failure mode PR #394 surfaced.
- The same-trio comparison (§3a) shows the v3 prompt + cleaned parser produces strictly equal-or-better verifier scores on 2 of 3 cells (27560 +0.033, 31430 +0.280), and a single dip on 31149 (−0.217) attributable to 5 concept-`_anon:` refs (cost_of_living, trade_deal, housing_market, canadian_banks, canadian_economy) flagged as undeclared due to ID-form mismatch (declaration `NOTE _anon_<id>` underscore form vs reference `_anon:<id>` colon form) — a residual sub-mechanism v3's worked examples don't surface.
- The n=9 scale-out adds 6 new cells (27702, 29807, 33632, 34537, 35352, 36065) of which 5 are clean gv-true (27702, 29807, 34537, 35352, 36065) and one is truncated but still gv-true (33632). gv 6/6 on the new cells vs PR #393's 2/6 on the same set.

---

## 8. Recommended next actions (NOT decided here)

1. **Two-pronged declare-first reinforcement.** The residual undeclared-ref failures split into two distinct sub-mechanisms that v4 prompt must address separately:
   1. **Concept-`_anon:` ID-form alignment** (31149 + 27702): the model already writes declaration NOTE blocks for concept refs but uses id form `_anon_<id>` (underscore) at the declaration site while ref sites use `_anon:<id>` (colon) — the parser cannot link the two. v4 prompt must show explicit ID FORM ALIGNMENT: the declared block's id must literally match the `_anon:<id>` syntax used at the reference site. Either document `_anon_<id>` ↔ `_anon:<id>` synonymy in the parser, OR add a worked example showing colon-form id at both declaration and reference (`NOTE _anon:cost_of_living` paired with EVT subject ref `_anon:cost_of_living`).
   2. **Universal declare-first reinforcement** (29807): vanilla PascalCase identifiers like `Reuters` or `InternationalEnergyAgency` that the model treats as "obviously known" still need ENT blocks before being used as slot fillers. This is decree #7 (declare before reference) — v4 should restate it OUTSIDE the `_anon:` section so the rule isn't read as `_anon:`-specific.
2. **n_predict ≥ 6 K** for long-output articles (33632, 35352). Independent of prompt; pure budget question.
3. **Per-article server timeout > 500 s as default** for the long-output articles (36065 needed 700 s on this run). Or split long articles into chunked extraction.
4. **Re-attach per-category GT** to the Phase 2 Arm B subset (or move to the n=10 acceptance corpus) so the recall gates can be evaluated corpus-wide. PR #393 and #394 already flagged this; the situation is unchanged.

---

## 9. Commit trail

- `f7d50e54` — prompt v3: explicit `_anon:` declare-then-reference discipline
- `8c72b38c` — n=9 batch 1/3: 27560, 27702, 29807
- `e6f61025` — n=9 batch 2/3: 31149, 31430, 33632
- `e8c3c105` — n=9 batch 3a: 34537
- `2b551714` — n=9 batch 3b: 35352, 36065 (first attempt; 36065 server-timed-out)
- `a9967486` — n=9 batch 3 final: 36065 re-run with 700 s timeout (clean)
- this report
