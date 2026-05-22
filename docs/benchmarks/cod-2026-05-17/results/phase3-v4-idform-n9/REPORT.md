# Phase 3 v4 ID-form alignment + PascalCase declare-first — n=9 acceptance run

**Date:** 2026-05-21
**Model:** `qwen3:30b-a3b-instruct-2507-q4_K_M` (4 B active / 30 B total MoE, q4_K_M GGUF)
**Backend:** `llama-server` (`http://localhost:11437`) + GBNF grammar-constrained decoding
**Schema:** v2.1 no-offset (`docs/grammars/cod-dsl-v2.1.gbnf` + revised `prompts/cod_dsl_v2_no_offset.txt` — v4: v3 `_anon:` discipline + two new worked examples: `_anon:` ID FORM ALIGNMENT and GENERIC DECLARE-FIRST FOR PASCALCASE IDS)
**Runner:** `scripts/run_cod_dsl.py --schema v2_1 --backend llama-server --grammar docs/grammars/cod-dsl-v2.1.gbnf --n-predict 6144 --timeout 700` (35352 re-run with `--timeout 900` after a first-attempt ReadTimeout at 700 s)
**Corpus:** Phase 2 Arm B (n=10 minus `27772` known long-hang) → n=9
**Branch:** `feat/dsl-poc/phase3-v4-idform-n9` off main (`c38cf696`)

---

## 1. Verdict (TL;DR)

**v4's two interventions land but trigger orthogonal regressions on three other cells.** The §6.4 acceptance gate is **NOT** reached — verifier_pass_rate mean 0.808 (n=8 defined) is **0.142 below the 0.95 gate**, very close to v3's 0.790. Grammar-validity drops from v3's 9/9 = 100 % to **8/9 = 88.9 %** (one new gv=False from a `Duplicate ENT declaration` parse error on 33632).

**Verdict: PROGRESS, not ACCEPTANCE.** The two specific interventions DO close the diagnosed failure modes from PR #395 §6.4:

1. **ID FORM ALIGNMENT lands on 31149**: undecl drops 5 → 1 (4 closed), verifier 0.783 → 0.909 (+0.126). The model emits `ENT _anon:cost_of_living / macro_indicator` and matching colon-form refs — exactly the pattern the v4 worked example demonstrates.
2. **PASCALCASE DECLARE-FIRST lands on 29807**: `Reuters` and `InternationalEnergyAgency` (the two undecls the v4 worked example targets by name) are NOW declared as ENT blocks. Undecl drops 3 → 2, verifier 0.867 → 0.921, org_recall jumps 0.600 → 1.000.

But three cells regress for reasons orthogonal to the two interventions:

3. **33632 gv=False** with `Duplicate ENT declaration '_anon:energy_costs'` — v3 had truncation-driven 37-undecl issue; v4's bigger n_predict avoided that but the model emitted the same `_anon:` ENT twice, hard-failing the parse. The PascalCase fix DID land on 33632 (`Reuters` is declared, no PascalCase undecl) but the over-emission triggered a NEW failure mode.
4. **36065 verifier drops 0.952 → 0.711** — model emitted 128 ENT-only blocks (0 EVT / 0 CLAIM / 0 NOTE). Kind distribution: **70.3 % (90/128) `concept`**, the rest 18 `org` / 7 `country` / 6 `object` / 4 `person` / 2 `macro_indicator` / 1 `region`. The v4 worked example's use of `concept` as the `<kind>` value over-generalized into "declare every abstract phrase as a concept ENT and stop" (without 100 % latch — 28 % of blocks use other kinds), suppressing event emission.
5. **27560 verifier drops 0.933 → 0.722** — single-shot variance; model emitted fewer copy slots (7 ENT / 11 EVT, missing_slot=11). Undecl stayed at 0.

Net: the diagnosed mechanisms are closed; the prompt's emphasis on declaration discipline + the worked-example use of `concept` kind triggered over-declaration spirals on long-output cells. v4 trades v3's truncation/dangling-ref failures for over-declaration/duplicate-declaration failures.

---

## 2. Per-cell table

| article | gv     | verbatim | verifier | num_exact | ticker_rec | org_rec | wall_s | stop  | eval | undecl | missing_slot | ent / evt / claim / note |
|---------|--------|----------|----------|-----------|------------|---------|--------|-------|-----:|-------:|-------------:|--------------------------|
| 27560   | True   | 0.722    | 0.722    | n/a       | n/a        | n/a     | 141.7  | eos   |  874 |      0 |           11 | 7 / 11 / 0 / 0           |
| 27702   | True   | 0.000    | 0.250    | n/a       | n/a        | n/a     | 114.7  | eos   |  245 |      1 |            1 | 1 / 1 / 0 / 1            |
| 29807   | True   | 0.971    | 0.921    | 0.111     | 0.000      | 1.000   | 177.7  | eos   | 1240 |      2 |           16 | 14 / 13 / 3 / 0          |
| 31149   | True   | 0.944    | 0.909    | 1.000     | n/a        | 0.750   | 137.1  | eos   |  837 |      1 |           10 | 9 / 6 / 4 / 0            |
| 31430   | True   | 0.947    | 0.958    | 0.222     | n/a        | 1.000   | 158.7  | eos   |  851 |      0 |            8 | 11 / 3 / 5 / 0           |
| 33632   | False  | n/a      | n/a      | n/a       | n/a        | n/a     | 408.7  | eos   | 3207 |      0 |            0 | 0 / 0 / 0 / 0            |
| 34537   | True   | 1.000    | 1.000    | n/a       | n/a        | n/a     | 122.9  | eos   |  695 |      0 |           14 | 8 / 14 / 0 / 0           |
| 35352   | True   | 0.991    | 0.991    | 0.009     | n/a        | n/a     | 760.8  | eos   | 5812 |      0 |            2 | 1 / 2 / 0 / 0            |
| 36065   | True   | 0.711    | 0.711    | 0.000     | n/a        | 0.438   | 378.8  | eos   | 2196 |      0 |            0 | 128 / 0 / 0 / 0          |

**Aggregates (n=9 unless noted):**
- `grammar_valid` rate: **8/9 = 88.9 %**
- `verbatim_match_rate` mean (n=8 defined): **0.786**
- `verifier_pass_rate` mean (n=8 defined): **0.808**
- `numeric_exact_match_rate` mean (n=5 defined): **0.268**  (1.000, 0.222, 0.111, 0.009, 0.000)
- `ticker_recall` mean (n=1 defined): **0.000**  (article 29807 only)
- `org_recall` mean (n=4 defined): **0.797**  (1.000, 1.000, 0.750, 0.438)
- Wall total **2,401 s** (~40 min), mean **267 s/cell**, median **159 s/cell**

(33632 is excluded from numeric aggregates because gv=False -> scoring returns Nones.)

---

## 3. Four-way comparison

| Metric (n=9 unless noted)            | PR #393 (old prompt + uncleaned parser) | PR #394 (3-cell, cleaned parser, current prompt) | PR #395 / v3 (cleaned parser + v3 prompt) | v4 (this run) |
|--------------------------------------|-----------------------------------------|--------------------------------------------------|-------------------------------------------|---------------|
| `grammar_valid` rate                 | 3/9 = 33.3 %                            | 3/3 = 100 % (smoke)                              | **9/9 = 100 %**                           | 8/9 = 88.9 %  |
| `verbatim_match_rate` (mean)         | 0.611 (n=3)                             | 0.967 (n=3)                                      | 0.828 (n=9)                               | 0.786 (n=8)   |
| `verifier_pass_rate` (mean)          | 0.944 (n=3)                             | 0.842 (n=3)                                      | 0.790 (n=9)                               | **0.808** (n=8) |
| `numeric_exact_match_rate` (mean)    | n/a                                     | 0.500 (n=2)                                      | 0.295 (n=6)                               | 0.268 (n=5)   |
| `ticker_recall` (mean)               | n/a                                     | n/a                                              | 0.000 (n=1)                               | 0.000 (n=1)   |
| `org_recall` (mean)                  | n/a                                     | 0.875 (n=2)                                      | 0.520 (n=5)                               | **0.797** (n=4) |
| `dangling_ref` failures (`undecl`)   | dominant (5/6 gv=False cells)           | persisted on 31430 (verifier=0.625)              | 9 total across 3 cells (5+3+1)            | **4 total across 3 cells (1+2+1)** |
| Wall mean (per cell)                 | 169 s (median 60 s)                     | 65 s (n=3)                                       | 194 s (median 107 s)                      | 267 s (median 159 s) |

### 3a. Same-trio comparison (27560 / 31149 / 31430)

| article | metric        | PR #393 baseline | PR #394 (3-cell) | PR #395 / v3 | v4 (this run) | Delta vs v3 |
|---------|---------------|------------------|------------------|--------------|---------------|-------------|
| 27560   | gv            | True             | True             | True         | True          | hold        |
| 27560   | verbatim      | 0.833            | 0.900            | 0.933        | **0.722**     | -0.211      |
| 27560   | verifier      | 0.833            | 0.900            | 0.933        | **0.722**     | -0.211      |
| 31149   | gv            | False            | True             | True         | True          | hold        |
| 31149   | verbatim      | n/a              | 1.000            | 1.000        | 0.944         | -0.056      |
| 31149   | verifier      | n/a              | 1.000            | 0.783        | **0.909**     | **+0.126**  |
| 31149   | num_exact     | n/a              | 1.000            | 1.000        | 1.000         | hold        |
| 31149   | org_rec       | n/a              | 0.750            | 0.500        | **0.750**     | **+0.250**  |
| 31430   | gv            | False            | True             | True         | True          | hold        |
| 31430   | verbatim      | n/a              | 1.000            | 0.882        | 0.947         | +0.065      |
| 31430   | verifier      | n/a              | 0.625            | 0.905        | **0.958**     | **+0.053**  |
| 31430   | num_exact     | n/a              | 0.000            | 0.000        | **0.222**     | **+0.222**  |
| 31430   | org_rec       | n/a              | 1.000            | 1.000        | 1.000         | hold        |

**Trio aggregates:** gv 3/3 (hold); verbatim mean 0.939 -> 0.871 (-0.068, driven by 27560 only); verifier mean 0.874 -> **0.863** (-0.011). The headline is **31149's verifier 0.783 -> 0.909 (+0.126)** — the cell that surfaced the ID-form mismatch in v3 — closes per v4 intervention #1. 31430 also gains +0.053 (the v3 acid-test cell continues to improve).

### 3b. Six new-in-v3 cells (27702, 29807, 33632, 34537, 35352, 36065) vs v4

| article | metric     | v3      | v4      | Delta    | mechanism                                                         |
|---------|------------|---------|---------|----------|-------------------------------------------------------------------|
| 27702   | verifier   | 0.200   | 0.250   | +0.050   | residual underscore-form NOTE for `_anon:80,000 shares` — id-form fix did NOT land when short_ident needs punctuation-normalization |
| 27702   | undecl     | 1       | 1       | hold     | same                                                              |
| 29807   | verifier   | 0.867   | **0.921** | **+0.054** | PascalCase fix lands: `Reuters` + `InternationalEnergyAgency` declared |
| 29807   | undecl     | 3       | 2       | -1       | Reuters + IEA closed; BrentCrudeFutures + WTI new residuals       |
| 29807   | org_rec    | 0.600   | **1.000** | **+0.400** | every news org / supranational org now declared                  |
| 33632   | gv         | True    | **False** | **regress** | new `Duplicate ENT declaration` parse failure (`_anon:energy_costs` declared twice); v3 was gv=True but truncated at n_predict=4096 with 37 undecls |
| 34537   | verifier   | 1.000   | 1.000   | hold     | perfect -> perfect                                                |
| 35352   | verifier   | 0.988   | 0.991   | +0.003   | n_predict bump 4096 -> 6144 closed v3's `stop=limit` truncation; wall 450 -> 761 s |
| 36065   | verifier   | 0.952   | **0.711** | **-0.241** | model emitted 128 ENT-only blocks (0 EVT / 0 CLAIM / 0 NOTE) — `concept` kind generalized into over-declaration spiral |
| 36065   | org_rec    | 0.500   | 0.438   | -0.062   | over-declaration produced lower-quality org matches              |

---

## 4. Specific tests of v4's two interventions

### 4.1 `_anon:` ID FORM MISMATCH (31149 + 27702)

**Did the id-form mismatch close (zero undecl on 31149 / 27702)?**

| cell  | v3 declaration form                   | v3 ref form               | v3 undecl | v4 declaration form                         | v4 ref form               | v4 undecl |
|-------|---------------------------------------|---------------------------|----------:|---------------------------------------------|---------------------------|----------:|
| 31149 | `NOTE _anon_cost_of_living` (x 6)     | `_anon:cost_of_living`    |         5 | `ENT _anon:cost_of_living / macro_indicator` (+2 sibling colon-form ENT decls) | `_anon:cost_of_living` (+ siblings) |       1 |
| 27702 | `NOTE _anon_shares`                   | `_anon:shares`            |         1 | `NOTE _anon_80_000_shares`                  | `_anon:80,000 shares`     |       1 |

**Verdict on intervention #1:**
- 31149: **PARTIAL PASS** — 4 of 5 id-form mismatches closed (cost_of_living, labour_markets, housing_market now declared with colon form; canadian_economy + trade_deal subsumed). Residual 1 undecl is `_anon:Canadian_banks` (different from v3's residuals) — model declared 3 of 4 abstract concepts but missed one.
- 27702: **DID NOT LAND** — the residual undecl persists with the SAME mechanism class but a DIFFERENT triggering scenario. Model declared `NOTE _anon_80_000_shares` (underscore form) while ref site is `_anon:80,000 shares` (colon AND verbatim space + comma). The v4 worked example shows colon-form id alignment for short alphanumeric idents (`cost_of_living`); it does NOT cover what to do when the natural short_ident would contain punctuation. Model normalizes (commas+space -> underscores) but only at the declaration site, breaking the literal match.

Net: 31149 closes (5 -> 1, primary mechanism resolved); 27702 holds (model's punctuation-normalization at decl site is a new sub-pathology v4 doesn't cover).

### 4.2 GENERIC DECLARE-FIRST FOR PASCALCASE IDS (29807)

**Did the generic PascalCase undecl close (zero undecl on 29807)?**

| 29807 PascalCase ref            | v3 status | v4 status   |
|---------------------------------|-----------|-------------|
| `Reuters`                       | undecl    | **DECLARED as `ENT Reuters / org`** |
| `InternationalEnergyAgency`     | undecl    | **DECLARED as `ENT InternationalEnergyAgency / org`** |
| `WhiteHouse` (was `USTheWhiteHouse` in v3) | declared as `USTheWhiteHouse` | **DECLARED as `ENT WhiteHouse / org`** (casing/spelling shift toward the prompt's worked example) |
| `BrentCrudeFutures`             | declared as `ENT BrentCrudeFutures / equity` | **referenced but ENT decl was DROPPED** — NEW undecl |
| `WTI`                           | declared as `ENT WTI / equity`               | **referenced but ENT decl was DROPPED** — NEW undecl |

**Verdict on intervention #2:**
- The TWO PascalCase idents the v4 worked example targets BY NAME (`Reuters`, `InternationalEnergyAgency`) **PASS** — both are now declared, and 29807's `org_recall` jumped 0.600 -> 1.000.
- BUT the model dropped declarations for `BrentCrudeFutures` and `WTI` — commodity-futures PascalCase ids the v4 worked example doesn't enumerate. Net undecl 3 -> 2 (1 closed, 2 swapped).
- The verifier_pass_rate gain (+0.054) and org_recall gain (+0.400) confirm the intervention is correctly targeted; the regression on commodity tickers suggests the worked-example enumeration needs to extend to commodities / futures / instrument-class PascalCase, not just news/supranational orgs.

29807 does NOT reach zero undecl, but the SPECIFIC failure mode v4 targets is closed.

---

## 5. Failure-mode breakdown (n=9)

| article | failure mode                                                                                                                                                              | severity                                            |
|---------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------|
| 27560   | model under-emits copy slots (missing_slot=11) — verifier counts each missing required slot against the block. Undecl=0. Likely single-shot variance band.                | verifier=0.722; regression vs v3 0.933, no failure mechanism beyond fewer slots emitted. |
| 27702   | 1 residual `_anon:` ID-form mismatch with extra wrinkle (short_ident contains comma+space -> model normalizes at decl site only, breaking literal match).                 | verifier=0.250; verbatim=0.000 (headline-only article; same edge as v3). |
| 29807   | 2 residual undecls — `BrentCrudeFutures` and `WTI` (commodity-futures PascalCase ids the v4 worked example didn't enumerate).                                             | verifier=0.921; +0.054 vs v3. The two TARGETED undecls (Reuters, IEA) closed. |
| 31149   | 1 residual undecl `_anon:Canadian_banks` (id-form aligned but one sibling concept dropped from decl set).                                                                 | verifier=0.909; +0.126 vs v3. Primary id-form mechanism resolved. |
| 31430   | clean — undecl=0, verifier=0.958. Continued v3 trajectory (0.625 -> 0.905 -> 0.958).                                                                                      | verifier=0.958.                                     |
| 33632   | gv=FALSE: `Duplicate ENT declaration '_anon:energy_costs'` — model declared the same `_anon:` ENT block twice. Tail also shows duplicate NUM blocks (NUM `fuel_tax_cut_1_percent` x 20 of 33 total NUM blocks, 14 distinct ids). Looping over the article tail. | gv=False; loses this cell from numeric aggregates. PascalCase fix DID land (`Reuters` declared). New failure mode INDUCED by v4's emphasis on emitting colon-form `_anon:` decls. |
| 34537   | clean — perfect 1.000.                                                                                                                                                    | verifier=1.000.                                     |
| 35352   | clean — n_predict bump closed v3's truncation; 0 undecl; wall 760.8 s (within 15-min hard stop but above 700 s default timeout).                                          | verifier=0.991.                                     |
| 36065   | model emitted 128 ENT blocks (70.3 % / 90 of kind `concept`, e.g. `ENT regulatory_will_absence / concept`, `ENT economic_cost_of_profit_extraction / concept`; remainder 18 org / 7 country / 6 object / 4 person / 2 macro_indicator / 1 region) and 0 EVT / 0 CLAIM. Over-declaration spiral induced by v4's use of `concept` as a worked-example kind — over-generalization, not 100 % latch. Undecl=0 (nothing references anything). | verifier=0.711; regression -0.241 vs v3. Largest single-cell regression. |

**Failure surface shift:** v3's dominant failure surface was `_anon:` ID-form mismatch (31149: 5 undecls) + generic PascalCase undecl (29807: 3 undecls) + truncation (33632, 35352 at `stop=limit`). v4 closes most of those but introduces:
- **Duplicate-declaration** (33632) — new gv=False mode.
- **Over-declaration spiral** (36065) — model treats every abstract phrase as a `concept` ENT and stops emitting events.
- **Single-shot variance regression on a previously-clean cell** (27560) — verifier 0.933 -> 0.722, no clear mechanism beyond fewer slots emitted.

The closed-mechanism count is HIGHER than the new-mechanism count (closed: 4 of 5 31149 id-form, 2 of 3 29807 PascalCase; introduced: 33632 dup-decl, 36065 over-declaration), but the new mechanisms drag the verifier aggregate harder because they zero out cells rather than just dinging individual blocks.

---

## 6. Cost / latency note

Wall mean 267 s/cell (median 159 s). Total 40 minutes for the n=9 sweep. The n_predict bump (4096 -> 6144) closed v3's `stop=limit` truncations (33632, 35352) but pushed wall times up:
- 33632: 405 s -> 408 s (similar wall but eval down 4096 -> 3207; the bigger budget let the model run further before duplicate-decl tripped the parser).
- 35352: 450 s -> 761 s (eval 4096 -> 5812; full document emission).
- 36065: 277 s -> 378 s (over-declaration spiral consumed more tokens than v3's clean output).

The 700 s HTTP timeout was sufficient for 8 of 9 cells; 35352's first attempt at 700 s ReadTimeouted and was retried at 900 s. Within the 15-min HARD STOP (900 s) on every cell.

---

## 7. §6.4 acceptance check

The §6.4 acceptance gates from `docs/plans/atlas-dsl-poc-plan.md`:

| Gate                          | Target  | This run                                                                                              | Verdict |
|-------------------------------|---------|-------------------------------------------------------------------------------------------------------|---------|
| `numeric_exact_match_rate`    | >= 0.85 | **0.268** (mean over n=5 defined cells); per-cell: 1.000 (31149), 0.222 (31430), 0.111 (29807), 0.009 (35352), 0.000 (36065) | **FAIL on aggregate**. 33632 dropped from n (gv=False). One cell at 1.000, four below 0.25. |
| `ticker_recall`               | >= 0.40 | **0.000** (n=1, article 29807 only — model emitted 0 tickers despite GT listing `MSFT`, `KO`, `AAPL`) | FAIL on the one defined cell. UNDETERMINABLE for 8 cells (no ticker GT). |
| `org_recall`                  | >= 0.50 | **0.797** (mean over n=4 defined cells); per-cell: 1.000 (29807), 1.000 (31430), 0.750 (31149), 0.438 (36065) | **PASS** comfortably. (vs v3 0.520 — biggest gate-level gain in this run.) |
| `verifier_pass_rate`          | >= 0.95 | **0.808** (mean over n=8 defined cells)                                                               | **FAIL by 0.142**. 4 of 8 defined cells clear the gate individually (1.000, 0.991, 0.958, 0.921); 4 drag the mean (0.250 from 27702, 0.711 from 36065, 0.722 from 27560, 0.909 from 31149). |
| gv-prerequisite (high gv rate) | (high) | 8/9 = 88.9 %                                                                                          | **REGRESSION from v3's 100 %**. One new gv=False from 33632's `Duplicate ENT declaration`. |

### Overall §6.4 verdict: **FAIL on verifier_pass_rate aggregate; PROGRESS on the two specific intervention tests, NOT ACCEPTANCE.**

Per intervention test (§4):
- **ID FORM ALIGNMENT (31149 + 27702):** PARTIAL PASS — 31149 closes 4 of 5 mismatches (+0.126 verifier); 27702 does not close (new wrinkle: short_ident punctuation normalization).
- **PASCALCASE DECLARE-FIRST (29807):** PARTIAL PASS — both targeted ids (`Reuters`, `InternationalEnergyAgency`) declared; org_recall 0.6 -> 1.0; 2 commodity-futures PascalCase ids (`BrentCrudeFutures`, `WTI`) newly undeclared.

Per gate:
- **gv-prerequisite:** REGRESSION (100 % -> 88.9 %); one new dup-decl failure on 33632.
- **verifier_pass_rate:** marginal gain 0.790 -> 0.808 (+0.018) but still FAIL by 0.142.
- **org_recall:** SUBSTANTIAL GAIN 0.520 -> 0.797 (+0.277) — the strongest gate-level signal.
- **numeric_exact_match_rate:** marginal degradation 0.295 -> 0.268.

### Progress delta vs v3

| dimension                  | v3      | v4      | delta    | direction |
|----------------------------|---------|---------|----------|-----------|
| gv rate                    | 100 %   | 88.9 %  | -11.1 pp | REGRESS   |
| verbatim mean              | 0.828   | 0.786   | -0.042   | REGRESS   |
| verifier mean              | 0.790   | 0.808   | +0.018   | MARGINAL  |
| num_exact mean             | 0.295   | 0.268   | -0.027   | MARGINAL  |
| org_recall mean            | 0.520   | 0.797   | +0.277   | **PROGRESS** |
| ticker_recall mean         | 0.000   | 0.000   | hold     | hold      |
| total undecl across cells  | 9       | 4       | -5       | **PROGRESS** |
| cells with verifier >= 0.90 | 5/9    | 5/8     | hold     | hold      |

**Net interpretation:** v4 is **marginally better on verifier (+0.018), substantially better on org_recall (+0.277), but introduces new failure modes (dup-decl on 33632, over-declaration on 36065) that the v3 prompt didn't trigger.** The two specific interventions land where targeted but generalize unfavorably to neighboring failure shapes.

---

## 8. What v4 actually proved

1. **The ID-form-alignment hypothesis is correct.** When the model emits colon-form `_anon:` decls (as the worked example shows), the parser links ref <-> decl. 31149's 4-of-5 closure is the cleanest signal.
2. **The PascalCase decree-#7 hypothesis is correct for named orgs.** When the worked example calls out `Reuters` and `InternationalEnergyAgency` BY NAME, the model declares them. Org_recall on 29807 jumped 0.4.
3. **`concept` as a worked-example kind generalizes too broadly.** 36065's 128-ENT-only output is the model picking up the `concept` kind from the v4 worked example and using it as a default kind for every abstract noun phrase. The prompt should either (a) use a different kind in the worked example, or (b) explicitly cap the ENT count for `concept`-kind blocks.
4. **The model can be pushed into duplicate declarations.** 33632's `_anon:energy_costs` x 2 + duplicate NUM blocks at the document tail suggest the model loses track of which `_anon:` ids it already declared on long-output cells.
5. **Punctuation-normalization at decl site without ref-site update is a new sub-mechanism.** 27702's `_anon_80_000_shares` vs `_anon:80,000 shares` is a case the v4 worked example doesn't address — the model has internalized "decl site should use grammar-safe chars" from the v3 `_anon:` discipline section but doesn't propagate back to the ref site.

---

## 9. Recommended next actions (NOT decided here)

1. **Swap `concept` for `macro_indicator` / `org` in v4's worked examples.** The model latched onto `concept` as a default kind for any abstract phrase and over-declared on 36065. Using kinds already enumerated in the v2.1 ent-type set should suppress the spiral.
2. **Explicit anti-pattern: duplicate `_anon:` declaration.** v4's `_anon:` section doesn't say "DECLARE ONCE — don't re-emit the same `_anon:<id>` block twice." 33632's dup-decl is a v4 regression; an explicit prohibition in v5 should close it.
3. **Extend PascalCase declare-first to commodity / futures / instrument PascalCase.** `BrentCrudeFutures`, `WTI`, `Stoxx600` etc. are not orgs but the same declare-first rule applies. v4's worked example only enumerates org/news/govt classes.
4. **Punctuation-bearing short_idents.** Add a worked example for the case where the natural short_ident contains a comma/space (e.g. `"80,000 shares"`) — either showing the canonical normalization (decl AND ref both use `_anon:eighty_thousand_shares`) or extending the grammar to allow these chars in ent-id.
5. **27560's single-shot regression.** Likely temperature-driven (T=0.2); investigating with T=0.0 on a re-run would distinguish prompt-induced regression from sampling noise.

---

## 10. Commit trail

- `dff4cd8c` — prompt v4: `_anon:` ID-form alignment + generic PascalCase declare-first
- `04669b9c` — n=9 batch 1/3: 27560, 27702, 29807
- `9a221d81` — n=9 batch 2/3: 31149, 31430, 33632
- `f9fb179f` — n=9 batch 3/3: 34537, 35352, 36065
- this report
