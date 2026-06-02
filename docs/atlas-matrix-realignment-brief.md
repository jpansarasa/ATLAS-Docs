# ATLAS Matrix Realignment — Decision Brief

*Written for your morning review. Goal: explain where the matrix work stands, how it works mechanically, and the handful of decisions I need from you to move forward. No jargon assumed — read top to bottom.*

---

## 1. What we're actually building (the 30-second version)

A **matrix**: rows = **macro signals**, columns = **the 11 sectors** (Energy, Materials, … Real Estate). Each cell is a number from **−3 to +3** — the sign is direction (defensive ← → offensive for that sector), the magnitude is strength. Every cell is a time series (daily).

It's a **"count the cards" tool** — situational awareness, not buy/sell advice. The whole point is to let you see *where the macro environment is*, and especially to surface **disagreement**: when the official data (FRED) and the unofficial data (news) say different things about the same signal, that's the interesting moment ("something changed and we don't know what yet").

That's the thesis from your handoff doc, and nothing below changes it.

---

## 2. How a number gets into a cell (the mechanics — read this or the rest won't land)

This is the part I think got muddy over NTFY, so here it is concretely.

**A "pattern" is a signal.** It's a small rule that reads some data and outputs one number in −3..+3. Example — the *yield-curve-inversion* pattern reads the 10Y-minus-2Y Treasury spread; when it inverts, it outputs a negative signal (recession risk).

**A signal projects onto sectors via a weight vector** (`sectorWeights`). The signal's −3..+3 value is multiplied by a per-sector weight to produce that sector's cell:

```
cell(signal, sector) = signal_value × sectorWeight[sector] × freshness × temporal × confidence × trust
```

So one signal fans out into up to 11 cells (one per sector), each scaled by how much that signal matters *to that sector*. Example — a "housing starts" signal might weight Materials high, Real Estate medium, Energy zero.

**The same signal can be fed by multiple sources.** This is the new part. "Unemployment" isn't just `UNRATE` from FRED — it's *also* a news signal when articles report layoffs. Both feed the **same row**. They differ on three knobs:

| Knob | FRED / OFR / Finnhub | Sentinel (news) |
|---|---|---|
| **Temporal** (lead/coincident/lag) | FRED lags months; index prices coincident | news often **leads** |
| **Trust** (how rigorous) | high | low (it's news) |
| **Decay** (how fast it goes stale) | slow (CPI matters for weeks) | **fast** (last week's news ≈ useless) |

Because both sources land in the same cell on the same −3..+3 scale, **disagreement becomes a number you can compute**: FRED says +1 on employment, news says −2 → divergence flagged.

**Who attaches the signal + sector?** SecMaster ("classify"). Three-leg attribution stack:
- **OpenFIGI** → instrument identity (resolve "Microsoft" → MSFT).
- **NAICS → 11-sector rollup** → sector (MSFT → tech / INFOTECH).
- **signal-identity catalog** → the signal/row.

So *"Microsoft lays off 10,000"* becomes **{signal = employment, sector = INFOTECH}** — note it's about the **tech sector**, not the MSFT stock. *"Fed holds rates steady"* becomes **{signal = fed-funds-rate, sector = none}** (a pure-macro signal).

**ThresholdEngine ("TE") does all the scoring** — the multiplication above, combining sources, aggregating cells into sector scores, classifying regimes. The contract you locked: *collectors gather, SecMaster classifies, TE scores. If it's a calculation, it's in TE.*

---

## 3. Why the patterns "need to change"

Two reasons:

1. **The old organizing scheme is gone.** The 79 patterns were filed under 8 categories (Recession, Inflation, Liquidity, Growth, Valuation, Currency, Commodity, NBFI/OFR). Your handoff **deleted those categories** — they mixed unlike things and didn't sit on a clean axis. So the patterns need to stop being category-organized and instead each become a clean single signal keyed to a signal-identity.

2. **The matrix is literally empty right now.** Every one of the 79 patterns ships a **placeholder all-zero `sectorWeights`** vector. Since `cell = signal × sectorWeight × …`, and sectorWeight is 0 everywhere, **every cell computes to 0**. The matrix renders blank. Filling in real sector projections is the single biggest piece of work — and it's a judgment call (decision #1 below).

---

## 4. Where we are right now

**Done and in production (this whole session led here):**
- The Sentinel pipeline produces real cells (resolution, classification, polarity) — fast and thread-safe.
- A `macro_observations` store already exists where **both FRED and Sentinel write signal-identity-keyed observations**, plus a live "disagreement" dashboard. (Big deal — see §6.)
- **Phase 0 (coverage):** audited all 79 patterns against the new model; mapped which signals already resolve to signal-identities (77% did); found the **25 signals observable by both FRED and news** (the disagreement-capable rows).
- **Safe fixes (shipped today):** 5 catalog alias fixes that unblocked the CPI/PCE inflation core, plus a mis-aliased equipment-investment series. All verified live.

**Not done — waiting on you:** the four decisions in §7. The biggest is the `sectorWeights` methodology (without it the matrix stays blank).

---

## 5. The pattern audit (what's wrong with the 79, and what I propose)

I audited all 79. Verdicts:

| Verdict | Count | Meaning |
|---|---:|---|
| **KEEP** | 33 | Good signal; just needs sectorWeights + a signal-identity key |
| **REALIGN** | 22 | Keep, but fix something (wrong temporal flag, key to an identity, etc.) |
| **MERGE** | 11 | Redundant — same signal as another pattern, drop the duplicate |
| **CUT** | 9 | Remove — broken, no data, or conceptually wrong |
| **SPLIT** | 4 | One pattern doing two jobs — break into separate signals |

**The CUTs worth your eye** (these are the "make the cuts" calls):
- `vix-deployment-l1` / `l2` — these encode an **action** ("deploy $25–100K"). The matrix is supposed to be *state, never action*, so the deployment ladder has to go (a clean VIX *level* signal can stay).
- `nbfi-escalation` — also encodes an action ("move to 70–75% defensive") **and** fuses three different stress signals into one. Split + de-action.
- `bankruptcy-clusters`, `kre-underperformance` — inert stubs (`expression: false`, no data feed). Dead.
- `cape-attractive`, `equity-risk-premium`, `forward-pe-value` — reference data series that don't exist in FRED ("Phase 2" placeholders). Never fire.
- `equal-weight-indicator` — mislabeled (it measures price-vs-moving-average momentum, not valuation).

**A few MERGEs** (same signal, different threshold/units): HY credit spread appears twice; CPI level vs CPI "acceleration" are the same series; three INDPRO patterns are one continuous signal sliced three ways; the manual Sahm rule duplicates the official FRED Sahm series.

**One correction Phase 0 made to my own audit:** I'd proposed merging `eur-usd` into the dollar-index signal — that's **wrong**; the catalog correctly treats EUR/USD as a distinct signal from the trade-weighted dollar index. Don't merge those.

The full per-pattern table exists if you want it line-by-line; the above is the actionable summary.

---

## 6. The one architecture decision (who does the scoring math)

A wrinkle: there are currently **two ways** Sentinel news reaches the matrix.

- **Path 1 (the good one, already built):** Sentinel writes a raw observation `{signal, sector, polarity, magnitude, source, time}` into `macro_observations`. No math. This is the contract you want.
- **Path 2 (the legacy one, also live):** Sentinel computes the *score* itself (confidence × freshness × trust) and writes a finished cell with a one-off per-article id like `sentinel:doc123:block4`. This is what we built earlier this session to get cells flowing.

Path 2 has three problems: the scoring math is in the collector (you said it belongs in TE), the per-article ids can't overlay a FRED row (so no disagreement), and there are now two different decay formulas drifting apart.

**The decision — "who scores":**
- **Option A (recommended):** Retire Path 2. TE reads `macro_observations` and does *all* the scoring for every source. This honors your "calculation = TE" rule, and only this makes FRED-vs-news disagreement **magnitude-comparable in one cell** (the matrix's whole reason to exist). It's mostly *deletion* of redundant code, since Path 1's substrate already exists.
- **Option B:** Keep Path 2's split, just stabilize the row id. Smaller change, but leaves two scoring engines forever and only gives you sign-level (not magnitude) disagreement.

This only matters at "Phase 2," so it's **not urgent** — but A is my recommendation and I'd want your nod before I start deleting the legacy path.

---

## 7. The decisions I need from you

In priority order. For each: what it is, why it matters, and my recommendation.

### Decision 1 — `sectorWeights` methodology  ⭐ (the blocker)

**What:** every signal needs a vector of weights saying how it projects onto each of the 11 sectors. Right now they're all zero, so the matrix is blank.

**Why it's hard:** it's domain judgment. "Yield-curve inversion" hits Financials and rate-sensitive sectors (Real Estate, Utilities) hard, Energy less; "housing starts" hits Materials/Real Estate/Consumer Discretionary. There's no single source of truth — it's analyst judgment, possibly informed by historical correlation.

**Options:**
- **(a) I draft a proposed methodology + starter weight vectors** for the 33 KEEP signals (e.g. a 0/low/med/high scheme grounded in each signal's economic transmission), and you edit. *(My recommendation — gives you something concrete to react to instead of a blank page.)*
- **(b) You hand me a rule or rubric** and I apply it mechanically.
- **(c) Derive empirically** from historical cell correlations — but the matrix has to run first to generate that history, so this is a later refinement, not a starting point.

### Decision 2 — the cut list

**What:** sign off on the CUT / MERGE / SPLIT recommendations in §5 so I can clean up the pattern set (these are hot-reloadable, low-risk).

**Options:** approve as-is · approve with exceptions you name · or I produce the explicit file-by-file change list for a yes/no pass first.

### Decision 3 — breakeven trio

**What:** there are 5-year, 10-year, and 5y5y-forward inflation-breakeven signals. They're the same concept (market inflation expectations) at different horizons.

**Options:** keep all three as distinct rows · or collapse to one "inflation-expectations" signal with horizon as a parameter. *(Minor; either is fine.)*

### Decision 4 — who scores (A vs B)

Covered in §6. **A recommended.** Only needed before Phase 2, so you can defer it.

---

## 8. What happens after you decide

- **Decision 1 (a):** I dispatch a draft of the sectorWeights methodology + the 33 KEEP vectors for your review → once you bless it, the matrix starts rendering real numbers.
- **Decision 2:** I apply the cuts/merges/splits to the pattern files and hot-reload.
- Then the phased build proceeds: SecMaster schema (add temporal/decay to signals) → TE re-key + observation projector in shadow mode → cutover (delete the legacy path, your Decision 4) → Grafana sector×signal heatmap + disagreement view.

Each phase is independently shippable; only the legacy-path deletion is a "big" step, and it's reversible by a feature flag.

---

## TL;DR — what I need from you

1. **How do you want sectorWeights done?** (I suggest: let me draft them, you edit.) ← this unblocks the matrix.
2. **Approve the cut list?** (9 cut / 11 merge / 4 split — `eur-usd` stays separate.)
3. **Breakeven: one signal or three?**
4. **Who scores: A (TE does it all — recommended) or B?** (can defer)

Everything else is done and live. Reply with as much or as little as you want and I'll run with it.

*Full plan: `~/.claude/plans/harmonic-wobbling-manatee.md`. Phase 0 coverage detail: `/tmp/sentinel-remediation/phase0-coverage/`.*
