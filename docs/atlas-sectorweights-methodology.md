# ATLAS `sectorWeights` Methodology — REVIEW DRAFT (rev1)

*Status: rev1 — incorporates the #579 review. The five forks are now resolved (sign convention + 7-level scale + no ×3 LOCKED; cyclical-GDP composite kept; borderline CUTs kept; pure-macro reclassified — see the "Decisions" block at the end and §F). Still editable: glance over §F and the magnitudes, then bless. Nothing here is committed to a pattern file or to code — it is a methodology + a starter set of vectors. The blank-page problem (Decision 1 in the matrix-realignment brief) is what this document removes.*

**Companion doc:** the matrix realignment brief (the plain-English *why*; retired to git history — `git show 5cf265f7:docs/atlas-matrix-realignment-brief.md`). This doc is the *how-to-fill-the-cells*; current matrix mechanics live in [MATRIX.md](./MATRIX.md).

---

## 0. The one-paragraph version

The matrix is `cell(signal, sector) = signal_value × sectorWeight[sector] × freshness × temporal × confidence × trust`. Every pattern today ships an all-zero `sectorWeights` vector, so every cell is `0` and the matrix renders blank. This document proposes (A) a **discrete signed weight scale** (7-level, `[−1,+1]`, LOCKED), (B) a **repeatable economic-transmission rubric** for choosing each weight, (C) **starter weight vectors for the 33 KEEP signals** plus the obvious SPLIT children, (D) a default **breakeven-blend** for the collapsed inflation-expectations row, and (F) a research-grounded **rate-transmission vector for `fed-funds-rate`** that retires the old "pure-macro ⇒ flat-zero" treatment — no macro row is sector-neutral; the contrast just varies. These weights live in pattern JSON and are hot-reloadable — fast tweak loop, no rebuild (§E).

---

## A. Weight scale & convention

### A.1 Runtime supports SIGNED weights — verified, no code change needed

I confirmed against the live code that a negative `sectorWeights` value works end-to-end:

- **`ThresholdEngine/src/Services/CellProjector.cs`** (`Project`): `raw = commonFactor * weight` where `commonFactor = signal × freshness × temporal × confidence × patternWeight`, then `raw` is clamped to `[-3, +3]` (`CellClampMin = -3m`, `CellClampMax = 3m`). The weight is a plain `decimal` multiplier. **A negative weight flips the cell's sign relative to the signal** — exactly the "a signal is offensive for one sector, defensive for another" behaviour we want. There is **no `Math.Max(0, …)` or `≥ 0` guard anywhere on the path.**
- **`ThresholdEngine/src/Configuration/PatternConfigurationLoader.cs`** (`ValidatePattern`, ~L385–407): the only `sectorWeights` validation is **structural** — the dictionary must contain all 11 ATLAS sectors (D5 "explicit zeros, not nulls"). There is **no range or sign check** on the values. `WarnIfAllSectorWeightsZero` (~L247) only warns when *every* weight is zero (the current blank state); a mix of signs and zeros passes clean.
- **Field shape** (confirmed in `growth/housing-starts.json`, `recession/yield-curve-inversion.json`, `liquidity/credit-spread-widening.json`): `sectorWeights` is a flat JSON object whose **keys are the 11 ALL-CAPS DB-form codes** and whose **values are JSON numbers** deserialised to `decimal`. Today all 33+ patterns carry `0.0` for all 11.

> **No flag, no migration, no code change is required to ship signed weights.** The clamp, the loader, and the projector already handle the full `[-1, +1]` range proposed below (the product never approaches the `±3` clamp at sane signal magnitudes; see A.4). The only "change" is editing JSON.

### A.2 The 11 ATLAS sector codes (canonical, DB form)

Source of truth: `Events/src/Events.Client/AtlasSectorCode.cs` (enum + `ToDbString`) and the SecMaster seed `atlas-sector-rollup-v1.0.csv`. These are the **real** codes — use these exact strings as `sectorWeights` keys, in this order:

`ENERGY, MATERIALS, INDUSTRIALS, CONS_DISC, CONS_STAPLES, HEALTHCARE, FINANCIALS, INFOTECH, COMM_SVC, UTILITIES, REAL_ESTATE`

(Note: `CONS_DISC` = Consumer Discretionary, `CONS_STAPLES` = Consumer Staples, `COMM_SVC` = Communication Services. ATLAS has **no** standalone "Tech Hardware" or "Semis" sub-sector — semis/hardware/software all roll into `INFOTECH`.)

### A.3 The discrete signed scale — **LOCKED: 7-level**

> **Decision 2 (LOCKED).** The **7-level** scale below is the authoring scale. 5-level was considered and rejected as too coarse — `±0.5` collapses the "primary high-beta" vs "second-tier" distinction the rubric (§B) actually produces. The 5-level paragraph below is retained only as a *reading aid*, not as an option to switch to.
>
> **On the "multiply weights ×3?" question — LOCKED: NO.** The scale stays `[−1, +1]`; weights are **not** rescaled to `[−3, +3]`. `signal_value` is already `±3` and the weight is a *projection fraction*: `cell = signal_value × weight × freshness × temporal × confidence × trust`. At `weight = ±1` the projection term alone already reaches the full `±3` cell range, so the clamp is reachable with no rescale. Multiplying weights by 3 only drives `signal_value × weight` toward `±9`, so the `±3` clamp **binds almost always** for any high-exposure cell — which erases the entire `× freshness × temporal × confidence × trust` attenuation. That attenuation chain *is* the quality/disagreement machinery (a stale, low-confidence, low-trust source must land a *smaller* cell than a fresh high-trust one); saturating the clamp throws it away. **If finished cells feel muted, recalibrate the multiplier BASELINES globally** — e.g. FRED `trust = 1.0`, a gentler freshness-decay half-life, a confidence floor — **never** the per-sector weights. Tune the attenuation chain; keep the projection a fraction in `[−1, +1]`.

Proposed **7-level discrete signed scale**:

| Weight | Label | Meaning |
|---:|---|---|
| **+1.00** | strong offensive | this sector is a primary, high-beta beneficiary when the signal is positive |
| **+0.66** | moderate offensive | clearly helped, second-tier |
| **+0.33** | weak offensive | mild tailwind |
| **0.00** | neutral / no transmission | signal has no first-order channel to this sector |
| **−0.33** | weak defensive | mild headwind / mild beneficiary on the *down*-signal |
| **−0.66** | moderate defensive | clearly hurt when signal is positive (or a clear safe-haven that rallies when the signal turns negative) |
| **−1.00** | strong defensive | primary high-beta loser / strongest counter-cyclical haven |

Discrete thirds keep the table human-authorable and editable (you can eyeball "is this a +1 or a +0.66?") without pretending to a false precision that empirical correlation would later overwrite (Decision 1(c) in the brief — empirical refinement is a *later* pass, once the matrix has run long enough to generate history).

**If 7 levels feel too fine on first read,** collapse to a **5-level** `{−1, −0.5, 0, +0.5, +1}` — the rubric in §B is written to be scale-agnostic; the starter vectors in §C use the 7-level scale but round cleanly to 5 levels (drop `±0.33`→`±0.5`, `±0.66`→`±0.5`). Pick one before iterating so the table stays internally consistent.

### A.4 Cell-sign semantics (defensive ↔ offensive, per sector) — **LOCKED: relative-rotation**

> **Decision 1 (LOCKED).** The **relative-rotation** reading is the convention for the whole table. The matrix is a *macro-environment / rotation* map, not an absolute-return forecast: a defensive sector earns a **positive** weight on a **risk-off/stress** signal (it *outperforms in the rotation*) and a **negative** weight on a **risk-on/growth** signal (it *lags*). The absolute-return alternative is rejected. Every §C sign below is derived under this convention.


The **signal** is signed `±3` on a *macro-direction* convention (the pattern's `signalExpression` decides what "+" means — e.g. `yield-curve-inversion` emits **negative** as the curve inverts deeper = more recession risk; `housing-starts` emits **positive** as starts rise). The **weight** then says how that macro direction lands on a given sector:

- `weight > 0` (**offensive for this sector**): the sector moves *with* the signal. Cell sign = signal sign. Example: a positive growth signal × `+1` on `INDUSTRIALS` ⇒ positive cell ⇒ "tailwind for industrials".
- `weight < 0` (**defensive for this sector**): the sector moves *against* the signal. Cell sign = opposite of signal sign. Example: a positive *inflation* signal × `−0.66` on `CONS_DISC` ⇒ negative cell ⇒ "inflation is a headwind for discretionary". Conversely a positive *risk-off / stress* signal × `+0.33` on `CONS_STAPLES`/`UTILITIES` ⇒ positive cell ⇒ "stress is a relative tailwind for defensives" (rotation, not absolute return).
- `weight = 0`: no first-order channel — explicit zero, not omission (D5).

**Reading a finished cell:** sign tells you the *direction of pressure on that sector right now*; magnitude tells you *how hard*. A green +2 on `FINANCIALS` and a red −2 on `UTILITIES` in the same column-day is the matrix doing its job — surfacing a rotation, not a contradiction.

> **Convention (locked above):** relative-rotation throughout §C — a defensive sector gets a **positive** weight on a **risk-off/stress** signal (it *outperforms* in the rotation), and a **negative** weight on a **risk-on/growth** signal (it *lags*). The absolute-return alternative (defensives small-negative even in risk-off because everything falls) is not used; if it were ever adopted, the §C signs for `CONS_STAPLES`/`UTILITIES`/`HEALTHCARE` on stress rows would flip.

---

## B. Assignment rubric (economic-transmission grounded)

A weight is chosen by walking **six transmission channels** and summing their contributions, then rounding to the nearest scale step. This is a *rule*, not ad-hoc taste — two authors applying it should land within one scale step.

For each `(signal, sector)` pair, score each channel that applies as `{−1, −0.5, 0, +0.5, +1}` (its directional contribution to *this* sector when the *signal* is positive), then take the **dominant channel** (not a naive average — the strongest economic story wins; average only to break near-ties). Round to the scale.

| # | Channel | Question | Sectors it dominates |
|---|---|---|---|
| 1 | **Rate sensitivity** | Does the signal move rates / discount factors? Long-duration & leverage-funded sectors react hardest. | `REAL_ESTATE`, `UTILITIES`, `INFOTECH` (long-duration cash flows), `FINANCIALS` (NIM — *opposite* sign to the others) |
| 2 | **Commodity exposure** | Is the signal a commodity price / input cost? Producers benefit on the up; heavy consumers are hurt. | `ENERGY`, `MATERIALS` (producers, +); `INDUSTRIALS`, `CONS_DISC` (consumers, −) |
| 3 | **Cyclicality** | Is the signal a growth/activity gauge? High-beta cyclicals amplify; defensives dampen. | `INDUSTRIALS`, `MATERIALS`, `CONS_DISC`, `ENERGY` (+); `CONS_STAPLES`, `UTILITIES`, `HEALTHCARE` (−, relative-rotation) |
| 4 | **Defensive / staple character** | On a risk-off/stress signal, which sectors are the rotation destination? | `CONS_STAPLES`, `UTILITIES`, `HEALTHCARE` (+ on stress); cyclicals (− on stress) |
| 5 | **USD sensitivity** | Does the signal move the dollar? Strong USD hurts exporters/commodity & multinationals; helps importers. | `MATERIALS`, `ENERGY`, `INFOTECH`, `INDUSTRIALS` (− on USD-strength); domestic `UTILITIES`, `REAL_ESTATE` (≈0) |
| 6 | **Credit sensitivity** | Does the signal move credit spreads / funding? Credit-funded & financial-intermediary sectors react hardest. | `FINANCIALS`, `REAL_ESTATE`, `CONS_DISC` (high-yield-funded), `UTILITIES` (capital-intensive) |

**Worked example — `yield-curve-inversion` (signal goes negative as curve inverts = recession warning):**
- Ch.1 rate-sensitivity: inversion = falling long rates relative to short → `FINANCIALS` NIM compression dominates → strong negative-for-financials channel. Because the *signal itself* is negative on inversion, financials want a **positive weight** (signal− × weight+ = negative cell = "financials under pressure"). Rate-sensitive long-duration `REAL_ESTATE`/`UTILITIES` get a mild benefit from falling long rates but the recession-warning cyclicality (Ch.3) dominates → mild positive weight.
- Ch.3 cyclicality: an inversion warns of recession → cyclicals (`INDUSTRIALS`, `MATERIALS`, `CONS_DISC`, `ENERGY`) get **positive** weights (signal− × + = negative cell = headwind). Defensives (`CONS_STAPLES`, `UTILITIES`, `HEALTHCARE`) get **negative** weights (signal− × − = positive cell = relative haven).
- Result vector in §C row 1.

This is the pattern for the whole table: **identify the dominant channel, set the sign by the relative-rotation convention (A.4), set the magnitude by how concentrated the channel is in that sector.**

---

## C. Starter vectors — the 33 KEEP signals (+ obvious SPLIT children)

**How to read each row.** Columns are the 11 sectors in canonical order. Each value is on the 7-level scale (§A.3) and assumes the **relative-rotation convention** (§A.4) and the **signal's existing sign convention** (from its `signalExpression`, noted in the rationale where it's counter-intuitive). The "signal +" column states what a *positive signal value* means so you can read the signs. These are **starting points for your edit pass** — magnitudes especially are first-draft.

Legend: `EN`=ENERGY `MAT`=MATERIALS `IND`=INDUSTRIALS `CD`=CONS_DISC `CS`=CONS_STAPLES `HC`=HEALTHCARE `FIN`=FINANCIALS `IT`=INFOTECH `CSV`=COMM_SVC `UTL`=UTILITIES `RE`=REAL_ESTATE

### C.1 Recession / labour-market signals

| signal-identity | signal + means | EN | MAT | IND | CD | CS | HC | FIN | IT | CSV | UTL | RE | one-line transmission rationale |
|---|---|--:|--:|--:|--:|--:|--:|--:|--:|--:|--:|--:|---|
| `yield-curve-2s10s` (yield-curve-inversion) | curve *steeper/positive*; signal **−** on inversion | +.66 | +.66 | +1 | +1 | −.66 | −.33 | +1 | +.66 | +.33 | −.33 | +.66 | inversion warns recession → cyclicals + FIN (NIM) hurt; defensives are the haven |
| `recession-probability` | higher recession prob; signal **−** as prob rises | +.66 | +.66 | +1 | +1 | −.66 | −.33 | +.66 | +.66 | +.33 | −.33 | +.66 | direct cyclical-vs-defensive recession axis |
| `sahm-rule` (→ `sahm-rule-official`, MERGE) | recession trigger; signal **−** as it fires | +.33 | +.66 | +1 | +1 | −.66 | −.33 | +.66 | +.66 | +.33 | −.33 | +.66 | labour-recession confirm; same rotation as recession-prob, slightly lower magnitude (lagging) |
| `initial-jobless-claims` (initial-claims-spike) | claims rising; signal **−** as claims spike | +.33 | +.33 | +.66 | +1 | −.66 | −.33 | +.66 | +.33 | +.33 | −.33 | +.33 | consumer/labour stress hits CD hardest, FIN via credit |
| `continuing-jobless-claims` | persistence of weakness; signal **−** as claims rise | +.33 | +.33 | +.66 | +1 | −.66 | −.33 | +.66 | +.33 | +.33 | −.33 | +.33 | same channel as initial, confirmation lag |
| `nonfarm-payrolls` (jobs-contraction) | jobs growing; signal **+** on strength | +.33 | +.66 | +1 | +1 | −.33 | −.33 | +.66 | +.66 | +.33 | −.33 | +.66 | broad cyclical employment; defensives lag the up-move |
| `job-openings` (jolts-job-openings) | openings rising; signal **+** on strength | +.33 | +.33 | +.66 | +.66 | −.33 | 0 | +.33 | +.66 | +.33 | −.33 | +.33 | labour demand → cyclical hiring, IT services |
| `um-consumer-sentiment` (consumer-confidence-collapse) | sentiment rising; signal **+** on strength | +.33 | +.33 | +.66 | +1 | −.66 | −.33 | +.33 | +.33 | +.66 | −.33 | +.33 | confidence → discretionary spend & media; staples are the inverse |
| `manufacturing-employment` (cyclical-payrolls-decline) | mfg jobs growing; signal **+** | +.33 | +.66 | +1 | +.66 | −.33 | 0 | +.33 | +.33 | 0 | −.33 | +.33 | factory-floor cyclical, materials/industrials core |
| `stagflation-warning` | stagflation risk; signal **−** as it worsens | +1 | +.66 | −.33 | −1 | +.33 | +.33 | −.33 | −.66 | −.33 | −.33 | −.66 | high inflation + weak growth: only commodity/staple/HC hold; rate-sensitive & discretionary crushed |

### C.2 Inflation signals

| signal-identity | signal + means | EN | MAT | IND | CD | CS | HC | FIN | IT | CSV | UTL | RE | one-line transmission rationale |
|---|---|--:|--:|--:|--:|--:|--:|--:|--:|--:|--:|--:|---|
| `cpi-headline-yoy` (cpi-yoy-level) | inflation rising; signal **+** as CPI rises | +1 | +.66 | −.33 | −.66 | −.33 | −.33 | +.33 | −.66 | −.33 | −.66 | −.66 | inflation = commodity tailwind, cost headwind for consumers, duration-killer for IT/UTL/RE; FIN mild + (nominal) |
| `cpi-core-yoy` (core-cpi-sticky) | sticky core rising; signal **+** | +.66 | +.33 | −.33 | −.66 | −.33 | −.33 | +.33 | −.66 | −.33 | −.66 | −.66 | core sticky → durable rate pressure; less commodity-direct than headline |
| `pce-core-yoy` (pce-above-target) | Fed-target gauge above; signal **+** | +.66 | +.33 | −.33 | −.66 | −.33 | −.33 | +.33 | −.66 | −.33 | −.66 | −.66 | Fed's preferred gauge → policy-rate transmission, same duration story as core CPI |
| `inflation-expectations` (breakeven trio MERGE — see §D) | mkt expects more inflation; signal **+** | +1 | +.66 | −.33 | −.33 | −.33 | −.33 | +.33 | −.66 | −.33 | −.66 | −.66 | forward expectations move real rates → duration-sensitive sectors hit, commodities bid |
| `truflation-vs-cpi` (REALIGN → disagreement) | real-time vs lagged CPI **gap**; signal = divergence | +.33 | +.33 | 0 | −.33 | 0 | 0 | 0 | −.33 | 0 | −.33 | −.33 | treat as *disagreement* magnitude, not inflation level — low weights, flags "official data is stale" |

### C.3 Growth / activity signals

| signal-identity | signal + means | EN | MAT | IND | CD | CS | HC | FIN | IT | CSV | UTL | RE | one-line transmission rationale |
|---|---|--:|--:|--:|--:|--:|--:|--:|--:|--:|--:|--:|---|
| `gdp-real` (gdp-acceleration) | growth accelerating; signal **+** | +.66 | +.66 | +1 | +1 | −.33 | 0 | +.66 | +.66 | +.33 | −.33 | +.33 | broad cyclical lift; defensives lag |
| `industrial-production` (MERGE of expansion/contraction/IP) | output rising; signal **+** | +.66 | +1 | +1 | +.33 | −.33 | 0 | +.33 | +.33 | 0 | −.33 | 0 | factory output → materials/industrials core, energy demand |
| `retail-sales` (retail-sales-surge) | consumer spend rising; signal **+** | +.33 | +.33 | +.66 | +1 | +.33 | 0 | +.33 | +.33 | +.66 | 0 | +.33 | discretionary + comm/media direct; staples mildly + (still spending) |
| `durable-goods-consumption` (durable-goods-consumption-growth) | durables demand rising; signal **+** | +.33 | +.66 | +1 | +1 | 0 | 0 | +.33 | +.66 | +.33 | 0 | +.33 | big-ticket cyclical: autos/appliances (CD), capex (IND) |
| `equipment-investment` (equipment-investment-growth) | business capex rising; signal **+** | +.33 | +.66 | +1 | +.33 | 0 | +.33 | +.33 | +1 | +.33 | +.33 | +.33 | capex cycle → industrials + IT (hardware/software capex) |
| `private-residential-investment` (residential-investment-growth) | housing investment rising; signal **+** | +.33 | +1 | +.66 | +.66 | +.33 | 0 | +.66 | +.33 | 0 | +.33 | +1 | residential build → materials, homebuilders (CD), RE, mortgage FIN |
| `cyclical-gdp-share-declining` (composite — **LOCKED: kept as one composite**) | cyclical share of GDP falling; signal **−** as it declines | +.33 | +.66 | +1 | +1 | −.33 | −.33 | +.33 | +.33 | 0 | −.33 | +.33 | composite cyclical-rotation gauge; **Decision 4 LOCKED** — stays one composite (not retired into its components), this vector stands |

### C.4 Housing — SPLIT children (the handoff worked example)

`housing-starts` splits into three lagged children sharing series `HOUST` but different `temporalType`/`leadTimeMonths`. Each gets its **own** narrow vector (the whole point of the split — one output per signal):

| signal-identity (split child) | lag | EN | MAT | IND | CD | CS | HC | FIN | IT | CSV | UTL | RE | rationale |
|---|---|--:|--:|--:|--:|--:|--:|--:|--:|--:|--:|--:|---|
| `housing-starts-to-materials` | t (coincident) | 0 | +1 | +.33 | +.33 | 0 | 0 | 0 | 0 | 0 | 0 | +.33 | starts consume lumber/cement/copper immediately → MATERIALS dominant |
| `housing-starts-to-homebuilders` | t−60d | 0 | +.33 | +.33 | +1 | 0 | 0 | +.33 | 0 | 0 | 0 | +.33 | homebuilders are classified `CONS_DISC`; starts lead builder revenue ~1 qtr |
| `housing-starts-to-real-estate` | t−180d | 0 | +.33 | 0 | +.33 | 0 | 0 | +.66 | 0 | 0 | 0 | +1 | completed supply flows to RE / mortgage credit at longer lag |

### C.5 Liquidity / rates / credit signals

| signal-identity | signal + means | EN | MAT | IND | CD | CS | HC | FIN | IT | CSV | UTL | RE | one-line transmission rationale |
|---|---|--:|--:|--:|--:|--:|--:|--:|--:|--:|--:|--:|---|
| `fed-funds-rate` *(see §F — revised, no longer "pure-macro flat")* | policy rate; signal **−** as rate rises/hikes (restrictive = bearish, per `real-rates.json` convention) | +.33 | +.33 | +.33 | +.66 | −.33 | −.33 | **−.66** | +.66 | +.33 | +.66 | **+1** | full rate-transmission vector — see §F.2. Signal is **−** on a hike: RE/UTL/IT *hurt* want **positive** weights (signal− × + = negative cell); FIN (NIM) and defensive CS/HC (relative haven) want **negative** weights (signal− × − = positive cell) |
| `walcl-fed-balance` (fed-liquidity-contraction) | balance sheet shrinking (QT); signal **−** on contraction | −.33 | −.33 | −.33 | −.66 | 0 | 0 | −.33 | −1 | −.66 | −.33 | −.66 | liquidity drain hits long-duration/high-multiple (IT, CSV growth) hardest |
| `m2-money-supply` (m2-growth-rate) | money supply growing; signal **+** | +.33 | +.33 | +.33 | +.66 | 0 | 0 | +.33 | +.66 | +.66 | +.33 | +.66 | liquidity tailwind → risk assets, long-duration growth |
| `ust-10y-real` (real-rates-tips, MERGE of real-rates) | real yields rising; signal **+** as real rates rise | 0 | −.33 | −.33 | −.33 | 0 | −.33 | +.33 | −1 | −.66 | −.66 | −1 | rising real rates = discount-rate shock → RE/IT/UTL crushed, FIN + |
| `hy-credit-spread` (credit-spread-widening; MERGE nbfi-hy-spreads) | spreads widening; signal **−** as spreads widen | +.33 | +.33 | +.33 | +.66 | −.33 | 0 | +1 | +.33 | +.33 | +.33 | +.66 | credit stress → FIN + RE (funding) most exposed; defensives relative haven |
| `nfci-chicago-fed` (chicago-conditions-index) | conditions tightening; signal **−** as it tightens | +.33 | +.33 | +.66 | +.66 | −.33 | 0 | +1 | +.66 | +.33 | +.33 | +.66 | broad financial-conditions tightening → leverage-sensitive sectors |
| `stlfsi-financial-stress` (stlouis-stress-index) | stress rising; signal **−** as stress rises | +.33 | +.33 | +.66 | +.66 | −.33 | 0 | +1 | +.66 | +.33 | +.33 | +.66 | same stress channel as NFCI, St-Louis variant |
| `ofr-fsi-stress` (composite; sub-indices REALIGN as children) | system stress rising; signal **−** as it rises | +.33 | +.33 | +.66 | +.66 | −.33 | 0 | +1 | +.66 | +.33 | +.33 | +.66 | OFR system-stress composite; children inherit, scaled to sub-index focus |
| `vix-index` (collapsed VIX *level*; CUT the deployment ladder) | volatility rising; signal **−** as VIX spikes | +.33 | +.33 | +.66 | +.66 | −.66 | −.33 | +.66 | +.66 | +.33 | −.33 | +.33 | risk-off vol spike → high-beta hurt, defensives the rotation |

### C.6 Currency signals

| signal-identity | signal + means | EN | MAT | IND | CD | CS | HC | FIN | IT | CSV | UTL | RE | one-line transmission rationale |
|---|---|--:|--:|--:|--:|--:|--:|--:|--:|--:|--:|--:|---|
| `dxy-dollar-index` (dxy-risk-off; MERGE dollar-weakness-regime) | USD strengthening; signal **+** as USD rises | −.66 | −.66 | −.33 | −.33 | 0 | 0 | 0 | −.33 | 0 | 0 | 0 | strong USD = commodity headwind + multinational FX drag; domestic/defensive ≈ neutral |
| `eur-usd` *(KEEP SEPARATE — do not merge into DXY; Phase-0 correction)* | EUR strengthening vs USD; signal **+** | +.33 | +.33 | +.33 | 0 | 0 | 0 | 0 | +.33 | 0 | 0 | 0 | EUR-up ≈ USD-down for the EU-exposed slice; distinct from trade-weighted DXY |

> Brief §5 explicitly corrected the earlier proposal to merge `eur-usd` into the dollar index. `eur-usd` stays a distinct row; its vector is the near-mirror of `dxy-dollar-index` but EU-trade-weighted, hence lower magnitudes.

### C.7 Commodity signals

| signal-identity | signal + means | EN | MAT | IND | CD | CS | HC | FIN | IT | CSV | UTL | RE | one-line transmission rationale |
|---|---|--:|--:|--:|--:|--:|--:|--:|--:|--:|--:|--:|---|
| `copper-price` (`cu-au-ratio`; REALIGN — **LOCKED: KEEP**) | copper rising / risk-on; signal **+** | +.66 | +1 | +.66 | +.33 | 0 | 0 | +.33 | 0 | 0 | 0 | +.33 | "Dr. Copper" growth proxy → materials/industrials direct |
| `baltic-dry-index` (`baltic-freight-recession`; REALIGN — **LOCKED: KEEP**) | shipping demand rising; signal **+** | +.33 | +.66 | +1 | +.33 | 0 | 0 | 0 | 0 | 0 | 0 | 0 | dry-bulk freight = global goods-trade pulse → industrials/materials |

> **Decision 5 (LOCKED): keep both** `cu-au-ratio` and `baltic-freight-recession` (not cut); the vectors above apply. **Operational follow-ups (do not block the methodology, track separately):**
> - `cu-au-ratio` is currently `enabled: false` — flip to `true` when its REALIGN lands.
> - `baltic-freight-recession` needs a **BDIY (Baltic Dry Index) data feed** wired up before it can emit; the vector is authored and waits on the feed.

### C.8 Valuation / leverage signals (the survivors)

| signal-identity | signal + means | EN | MAT | IND | CD | CS | HC | FIN | IT | CSV | UTL | RE | one-line transmission rationale |
|---|---|--:|--:|--:|--:|--:|--:|--:|--:|--:|--:|--:|---|
| `buffett-indicator` (MERGE buffett-indicator-daily) | mkt-cap/GDP elevated; signal **−** as it stretches | −.33 | −.33 | −.33 | −.66 | 0 | 0 | −.33 | −1 | −.66 | 0 | −.33 | rich valuation = downside risk concentrated in high-multiple IT/CSV |
| `ofr-hfm-leverage` (hfm-leverage; REALIGN) | HF leverage elevated; signal **−** as it builds | +.33 | +.33 | +.33 | +.33 | 0 | 0 | +1 | +.33 | +.33 | 0 | +.33 | NBFI leverage = deleveraging/forced-selling risk → FIN-intermediation core |

**Rows intentionally NOT given vectors here** (CUT or disagreement rows, per the audit): `vix-deployment-l1/l2` (CUT — action-encoded), `cape-attractive`/`equity-risk-premium`/`forward-pe-value` (CUT — no FRED series), `equal-weight-indicator` (CUT — mislabeled), `bankruptcy-clusters`/`kre-underperformance` (CUT — inert stubs), `freight-recession` (CUT — self-contradictory). The foreign-CB rates (`boc/ecb/boe/boj`) in the catalog have no KEEP pattern yet — when promoted they take the **same rate-transmission lens as §F** (an ECB hike hits EU-exposed Financials / Real-Estate / long-duration the same way), not a flat zero.

> **Note — "pure-macro = flat zeros" is retired.** Earlier drafts treated `fed-funds-rate` (and the policy/rate family) as *sector-less* and left them all-zero. §F shows that is wrong: a rate move has a sharp, empirically-grounded **differential** transmission vector. The fed-funds row in §C.5 above now carries that vector; §F also re-examines every other row previously called "pure-macro" and reclassifies them.

---

## D. Breakeven trio → `inflation-expectations` (Decision 3)

The audit collapses three breakeven patterns — `breakeven-5y-elevated` (`T5YIE`), `breakeven-10y-elevated` (`T10YIE`), `breakeven-5y5y-forward` (`T5YIFR`) — into **one** `inflation-expectations` row. Their sector vector is the single row in §C.2 (`inflation-expectations`). The remaining question is **how to blend the three horizons into one signal value.**

**Critical distinction (call this out explicitly so it doesn't get conflated with §C):** the horizon blend is a **TE *scoring* weight inside the pattern's `signalExpression`** — it produces the single `±3` `signal_value`. It is **NOT** a `sectorWeight`. The `sectorWeight` vector (§C.2) is applied *after* the blend, projecting the one blended signal onto the 11 sectors. Two different weight concepts; don't cross them.

**Proposed default blend — 10Y-primary:**

```
signal = 0.50·norm(T10YIE) + 0.25·norm(T5YIE) + 0.25·norm(T5YIFR)
```

where `norm(x)` is the existing per-series normalisation each pattern's `signalExpression` already does (deviation-from-anchor → `±3`, then the blend is re-clamped to `±3`).

**Rationale for 10Y-primary over equal-thirds:**
- The **10Y breakeven is the market's headline inflation-expectations number** and the most-traded TIPS tenor — deepest, least-noisy, the one the financial press and the Fed cite.
- The **5Y** captures the near-term path (more policy-reactive, noisier); the **5Y5Y forward** is the Fed's preferred *long-run anchor* (de-noised of the near-term). Weighting each at `0.25` keeps both as informative wings without letting the noisier 5Y or the slow-moving 5Y5Y dominate.
- This is a **horizon-importance** weighting, not a data-quality one — all three are high-trust FRED/TIPS series.

**Alternative if you prefer simplicity:** equal blend `Σ/n = (T5YIE + T10YIE + T5YIFR)/3`. Defensible, removes the "why 0.50?" question, costs a little signal (averages the noisy 5Y in at full weight). Either is fine; **10Y-primary is the recommendation, equal-thirds is the no-argument fallback.** Whichever you pick is one number in the `signalExpression`, trivially re-tunable.

---

## E. Where these live & how to iterate (the fast loop)

- **Location:** the `sectorWeights` object in each pattern JSON under `ThresholdEngine/config/patterns/<category-dir>/<patternId>.json`. (The category *folders* are slated to be dropped per the audit, but the field shape is unaffected — `sectorWeights` is per-file regardless of directory.)
- **Shape to write** (all 11 DB-form keys, signed decimals; example for `yield-curve-inversion` row C.1):
  ```json
  "sectorWeights": {
    "ENERGY": 0.66, "MATERIALS": 0.66, "INDUSTRIALS": 1.0, "CONS_DISC": 1.0,
    "CONS_STAPLES": -0.66, "HEALTHCARE": -0.33, "FINANCIALS": 1.0, "INFOTECH": 0.66,
    "COMM_SVC": 0.33, "UTILITIES": -0.33, "REAL_ESTATE": 0.66
  }
  ```
- **Hot-reload, no rebuild:** `ansible-playbook playbooks/deploy.yml --tags patterns`. The `PatternConfigurationWatcher` picks up the changed JSON; this is the fast tweak loop — edit a vector, deploy `--tags patterns`, watch the cell change in the sector×signal heatmap. No container rebuild needed.
- **Verification after a reload** (from the plan's Verification section): query `matrix_cells` for `WHERE cell_value != 0` (was all-zero before), and confirm `thresholdengine.patterns.evaluated` + non-zero cells in Grafana.
- **Iteration discipline:** the **scale** (7-level, A.3) and the **sign convention** (relative-rotation, A.4) are now **locked** — both ripple through every cell, so they were fixed first. Magnitudes are the remaining free variable; tune them row-by-row, they're independent edits.

---

## Decisions — settled in review (rev1)

The five forks are now resolved. Listed for the record; the body sections carry the detail.

1. **Sign convention (A.4) — LOCKED: relative-rotation.** Absolute-return rejected. See §A.4.
2. **Scale granularity (A.3) — LOCKED: 7-level**, weights stay `[−1, +1]` (no ×3 rescale). See §A.3.
3. **`fed-funds-rate` / pure-macro rows — RESOLVED: not sector-less.** "Pure-macro = flat zeros" is retired; rate (and most other "macro") rows get proper differential transmission vectors. Full analysis + the fed-funds vector in **§F**.
4. **`cyclical-gdp-share-declining` — LOCKED: kept as one composite.** The C.3 vector stands; not retired into components.
5. **Borderline CUTs `cu-au-ratio` / `baltic-freight-recession` — LOCKED: keep both.** C.7 vectors apply. Operational follow-ups: `cu-au-ratio` is `enabled:false` (flip on REALIGN); `baltic-freight-recession` needs a BDIY feed.

---

## F. Pure-macro vs differential transmission (Research — Q3)

> **Question (your #579 review):** is it right to treat `fed-funds-rate` and the other "pure-macro" rows as sector-neutral? You flagged that rates hit banking/Financials first. This section answers (a) whether *any* macro signal is truly sector-neutral, (b) builds a grounded `fed-funds-rate` vector from rate-sensitivity by sector, and (c) re-examines every row previously treated as pure-macro.

### F.1 Is any macro signal truly sector-neutral?

**No first-order macro signal is sector-neutral.** "Sector-neutral" and "pure-macro" are two different claims that the earlier draft conflated:

- **The brief's "`{signal=fed-funds-rate, sector=none}`" is a *classification* statement, not a *projection* statement.** It lives on the **SecMaster side**: a news article "Fed holds rates steady" has no *company/instrument* to resolve, so SecMaster attaches **no specific sector tag to that observation**. That is correct — there is no ticker to classify. But the brief's §2 contract is explicit that **TE does the scoring/projection**. The absence of a classification-time sector tag does **not** mean the signal has no transmission vector when TE projects it across the matrix. Those are different layers: SecMaster says "this observation isn't about one company"; TE still says "a rate hike lands hardest on Real Estate and Financials." Leaving the `sectorWeights` all-zero collapses the *entire row to nothing* — the macro signal would then contribute zero to every cell and be invisible in the sector×signal heatmap, which defeats the purpose of having the row at all.

- **Every macro variable transmits through a discount-rate, cash-flow-timing, leverage, or input-cost channel that is unevenly distributed across sectors.** Rates → duration & NIM (very uneven). Liquidity → multiple/leverage (uneven). USD → export/commodity mix (uneven). The *only* way a signal is genuinely sector-neutral is if it has **no economic channel to equities at all** — and then it shouldn't be a matrix row in the first place.

**The right model:** a "pure-macro" signal is one whose *cause* is economy-wide (a policy lever, an aggregate price) — but its *effect* is always a **differential vector**. The vector may be flatter (a broad-liquidity signal touches everything somewhat evenly) or sharp (a rate move is extremely concentrated), but it is never literally flat-zero. **Replace "pure-macro ⇒ no vector" with "pure-macro ⇒ derive the transmission vector from the dominant channel (§B), it just may be lower-contrast than a commodity or labour signal."**

### F.2 `fed-funds-rate` transmission vector (grounded)

**Signal-sign convention (locked for this row):** following `liquidity/real-rates.json` (`signal = −(realRate − 1)` ⇒ restrictive rates emit a **negative** signal), `fed-funds-rate` emits **signal − as the policy rate rises / on a hike** (tightening = restrictive = bearish macro direction). Under the relative-rotation convention (§A.4), the cell sign is `signal × weight`, so for a hike (`signal < 0`):

- a sector **hurt** by a hike must show a **negative cell** ⇒ needs a **positive weight** (`(−) × (+) = −`);
- a sector that **benefits** must show a **positive cell** ⇒ needs a **negative weight** (`(−) × (−) = +`).

**Evidence base (sector rate-sensitivity / rate-beta):**

- **Financials — benefit (the only clear winner).** Higher policy rates lift net interest margin; Schwab's *When Yields Talk, Sectors Listen* finds Financials' relative performance "almost mirrors" the 10Y yield (strongest positive rate-beta of any sector). During the 2022 hiking cycle bank net interest income rose ~17.9% on a 425 bp move. ⇒ strong **negative weight** (so the hike-signal produces a positive cell). Magnitude `−0.66` (not `−1`: the benefit is real but partly offset by recession/credit risk when hikes are aggressive).
- **Real Estate (REITs) — worst-hit.** Heavy borrowers + statutory high-payout ⇒ pure bond-proxy; the most rate-sensitive sector. ⇒ strong **positive weight** `+1`.
- **Utilities — bond-proxy, capital-intensive.** Schwab: a 1 pp rise in the 10Y drove Utilities to underperform the market by ~2–9 pp across cycles. ⇒ **positive weight** `+0.66`.
- **InfoTech & Comm Services — long-duration cash flows.** High-P/E, distant cash flows discounted harder; the Nasdaq's −33% in 2022 as the 10Y went 1.5→4.3% is the canonical case. INFOTECH carries the heavier load (semis/hardware/software all roll into it). ⇒ INFOTECH `+0.66`, COMM_SVC `+0.33` (mixed — large-cap media/telco are less pure-growth).
- **Consumer Discretionary — rate-sensitive demand.** Autos, housing-linked, big-ticket financed purchases contract as financing costs rise. ⇒ **positive weight** `+0.66`.
- **Cyclicals on the standalone rate channel (Energy, Materials, Industrials) — mild headwind.** Higher borrowing/capex cost is a real drag, but these sectors are dominated by the *growth* channel, not the *rate* channel; in isolation a hike is a mild negative. ⇒ **positive weight** `+0.33` each (low-contrast).
- **Consumer Staples & Healthcare — mild defensive *relative winner*.** Short-duration, inelastic-demand cash flows ⇒ the rate channel barely touches them, and in a tightening they *outperform* the rate-sensitive growth/RE complex. Under the locked relative-rotation convention a relative winner on a hike wants a **positive cell**, so with the signal **−** on a hike they take a **negative weight** `−0.33` each (`(−) × (−) = +` ⇒ small green = "relative haven as rates bite"). Low magnitude — they're a *mild* haven, not a primary beneficiary like Financials.

**Resulting `fed-funds-rate` vector (7-level, relative-rotation, signal − on a hike):**

| EN | MAT | IND | CD | CS | HC | FIN | IT | CSV | UTL | RE |
|--:|--:|--:|--:|--:|--:|--:|--:|--:|--:|--:|
| +.33 | +.33 | +.33 | +.66 | **−.33** | **−.33** | **−.66** | +.66 | +.33 | +.66 | **+1** |

```json
"sectorWeights": {
  "ENERGY": 0.33, "MATERIALS": 0.33, "INDUSTRIALS": 0.33, "CONS_DISC": 0.66,
  "CONS_STAPLES": -0.33, "HEALTHCARE": -0.33, "FINANCIALS": -0.66, "INFOTECH": 0.66,
  "COMM_SVC": 0.33, "UTILITIES": 0.66, "REAL_ESTATE": 1.0
}
```

Reading it: on a **hike** (signal −), Real Estate gets the most-negative cell (`+1 × signal = −`), Utilities / InfoTech / Cons-Disc next, while Financials *and* the Staples/Healthcare defensives go green (Financials on NIM, defensives on relative rotation). On a **cut** (signal +) every sign flips — Financials and defensives red, rate-sensitives green — exactly the easing-rally rotation. **This is the single highest-contrast row in the whole matrix**, the opposite of "sector-neutral."

> **Caveat to note in your edit pass:** FINANCIALS rate-beta is regime-dependent. Gradual hikes into a strong economy = NIM tailwind (negative weight correct). A *panic* hiking cycle that inverts the curve and triggers credit stress flips banks negative (see `yield-curve-inversion` C.1, where FIN takes a positive weight = headwind on inversion). The `fed-funds-rate` row captures the **level/direction** channel; the curve-shape and credit-stress channels live in their own rows and compose additively in the column. Don't try to make this one row express all three.

### F.3 Re-examining the other "pure-macro" rows

Applying the same lens to every row the earlier draft (or the brief) treated as pure-macro / sector-less:

| row | earlier treatment | verdict | reclassified vector |
|---|---|---|---|
| `fed-funds-rate` | "pure-macro, populate cautiously / maybe zero" | **NOT neutral — sharpest vector in the matrix** | §F.2 above (RE +1, FIN/CS/HC −, rate-sensitives +) |
| `walcl-fed-balance` (QT/QE liquidity) | already had a vector in C.5 | confirm **not neutral** — liquidity drain is concentrated in long-duration/high-multiple (IT, growth CSV) | C.5 row stands |
| `m2-money-supply` | already had a vector in C.5 | confirm **not neutral** — broad-liquidity tailwind to risk/long-duration; *lower-contrast* than rates but still signed | C.5 row stands |
| `ust-10y-real` (real rates) | already had a vector in C.5 | confirm **not neutral** — discount-rate shock, RE/IT/UTL crushed, FIN + | C.5 row stands (sharpest after fed-funds) |
| foreign-CB rates `boc/ecb/boe/boj` | "stay sector-less for now" | **same rate lens applies** when promoted — EU/global rate moves hit EU-exposed Financials, Real-Estate, long-duration; weight the export/multinational slice | author with §F.2 lens, EU-trade-weighted (lower magnitudes), when a KEEP pattern exists |

**The one genuinely low-contrast case** is a *pure broad-liquidity* aggregate like `m2-money-supply`: its channel (system-wide liquidity → risk appetite) really does touch most risk assets in the same direction, so its vector is legitimately *flatter* (mostly `+0.33`/`+0.66`, no strong negatives). But "flatter" still isn't "flat" — even there, long-duration growth (IT, CSV) and leverage-sensitive RE/FIN load more than Staples/Healthcare. **Conclusion: zero rows remain genuinely sector-neutral. Every macro row gets a vector; only the *contrast* varies** — sharp for rate/real-rate rows, moderate for credit/USD, flat-ish for broad-liquidity. The "pure-macro ⇒ all-zero" treatment is replaced wholesale by "derive the vector from the dominant channel; let the contrast fall where the economics put it."

**Sources (rate-sensitivity by sector):**
- Charles Schwab / Advisor Perspectives, *When Yields Talk, Sectors Listen* — sector relative-performance beta to the 10Y yield (Financials ≈ mirror of yields; Utilities −2 to −9 pp per 1 pp; REITs inverse).
- Schwab Sector Views, *What to Expect When Rates Rise* — Financials NIM benefit, Utilities/REIT rate vulnerability.
- CME OpenMarkets / Columbia Business School (Kim, 2024, *Interest Rate Sensitivities, Firm Growth Rates, and Stock Returns*) — long-duration/high-growth equities discount-rate sensitivity; 2022 Nasdaq −33% on the 10Y 1.5→4.3% move.
