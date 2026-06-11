# Fix #6 — News-signal decay / accumulation model (design)

**Date:** 2026-06-05
**Branch:** `feat/news-signal-decay`
**Status:** DESIGN ONLY — no implementation, no build in this branch.
**Scope:** ThresholdEngine `ObservationCellProjector` news path (`source_collector='sentinel'`).
**Context:** matrix health restored via #618–#625; break-map retired — history recoverable via git log; architecture context: `docs/atlas-matrix-handoff-v2.md`.
**Predecessor fixes:** #613 (Fix #1, restores news→`macro_observations` feed — LIVE);
`docs/superpowers/specs/2026-06-05-news-signal-feed-design.md` (feed value contract).

---

## 1. Goal

Restore Sentinel's intended matrix semantics: **sustained news coverage of a
signal accumulates / reinforces a cell, while a single stale event decays
toward zero**, bounded and sane for the ±3 matrix clamp. News is the
fast-decaying *leading counterweight* to lagging hard data — a single day
should barely move a cell; a week of consistent same-direction coverage should
move it substantially.

The A3 cutover (#601) broke this by routing news through FRED's hard-data
contract: the projector groups `(signal, collector)` over a 120-day lookback
into a single **arithmetic mean** `Σoᵢ/n`
(`ObservationCellProjectorCore.cs:76-86`) and stamps one cell at the latest
observation time. Two defects follow:

1. **The mean averages sustained flow away.** Ten consecutive days of `+0.6`
   tilt produce the same cell as one day of `+0.6` — volume of coverage is
   erased. This is the exact inverse of the "sustained = signal" intent.
2. **News decays on FRED's schedule.** Freshness reuses
   `CellProjector.CalculateFreshness` (`CellProjector.cs:60-82`) — a
   publication-frequency step with a 10% floor keyed to the *latest* obs age,
   not a per-article 24h half-life. A 5-day-old burst of coverage stays ~fully
   fresh; the fast-perishable nature of news is gone
   (break-map §break-6c — retired doc, recoverable via git log).

The deleted `SignalMagnitudeCalculator`
(`git show 0be162dd^:SentinelCollector/src/Semantic/SignalMagnitudeCalculator.cs`)
encoded the lost semantics: per-article
`freshness = exp(−ln2 × ageHours / 24h)` (24h half-life, no floor, lines
260-279), `weight = confidence × freshness × alphaFactor` (line 220),
`DailyUtcMidnightBucket` (lines 247-251), one `MatrixCellUpdate` per CLAIM, and
"aggregation rule downstream = weighted-mean" (doc lines 48-49).

---

## 2. The core decision — aggregation formula

### 2.1 Why the obvious candidates fail

Let the news group hold observations `v₁…vₙ` with ages (now − obsₜ) and
per-article decay weights `wᵢ = exp(−ln2 · ageᵢ_hours / 24)`.
`wᵢ ≈ 1.0` today, `0.5` at 24h, `0.25` at 48h, `0.125` at 72h.

| Candidate | Formula | Fails because |
|---|---|---|
| **Flat mean** (current) | `Σvᵢ / n` | Sustained flow averages away; no decay (the bug). |
| **Decay-weighted *mean*** (deleted model's downstream rule) | `Σ(wᵢvᵢ) / Σwᵢ` | Normalizes out *volume*: 1 fresh article and 50 fresh articles give the same value. Sustained ≠ reinforced. The deleted model only got accumulation because it emitted a **dated daily-cell time series** (one bucket per day) that a downstream reader integrated — not from this single value. We are collapsing to one cell (§4), so a weighted mean here still does not accumulate. |
| **Pure decay-weighted sum** | `Σ(wᵢvᵢ)` | Accumulates AND decays correctly, but **unbounded**: 30 same-direction articles × tilt 0.6 ≈ 18 → permanently saturates the ±3 clamp, destroying gradation between "moderate" and "overwhelming" coverage. |

### 2.2 Recommendation — decay-weighted sum with soft saturation (`tanh` squash)

Compute the raw decay-weighted sum, then squash it through a saturating
function before the existing scoring chain:

```
S      = Σ_i ( vᵢ · wᵢ )                       # signed decay-weighted accumulation
         where wᵢ = exp(−ln2 · ageᵢ_hours / H),  H = 24h
newsMagnitude = K · tanh( S / K )              # soft clamp into (−K, +K), K = saturation scale
```

Then feed `newsMagnitude` in place of `alphaMean` into the *unchanged* cell
chain:

```
cell = newsMagnitude · sourceTrust · temporal · confidence · sectorWeight
       clamped to [−3, +3]   (D12, CellProjector.CellClampMin/Max)
```

**Why this shape:**

- **Sustained accumulates.** Each fresh same-direction article adds ~`v·1.0`
  to `S`. Five consecutive days of `+0.6` (decayed at +0d…+4d:
  1, .5, .25, .125, .0625 ≈ Σw 1.94) → `S ≈ +1.16`, vs one day `S ≈ +0.6`.
  Coverage volume is preserved (a weighted mean would flatten both to ~`+0.6`).
- **Single stale event decays toward zero.** One article at age 3 days →
  `S = v·0.125`. After ~1 week the contribution is < 1% — `S → 0` with **no
  10% floor** (the FRED floor is wrong for news; perishable-by-design).
- **Bounded & graded.** `tanh` is ~linear near zero (small `S` ≈ `S`, so
  light coverage scales naturally) and asymptotes to `±K` for large `S`, so a
  burst of 30 articles saturates *gracefully* near `K` instead of slamming the
  hard ±3 clamp and losing all gradation. Choosing **K = 2.0** keeps
  `newsMagnitude ∈ (−2, +2)`; after `sourceTrust` (sentinel 0.7) × temporal ×
  confidence the cell lands comfortably inside ±3, so the hard clamp becomes a
  true safety rail, not the operating point. `K` is a config knob.
- **Self-healing on each cycle.** Because `S` is recomputed every cycle from
  the lookback window with fresh ages, yesterday's high cell *decays on its own*
  the next cycle if no new coverage arrives — exactly "a single day barely
  moves a cell, and stops mattering as it ages."

**Half-life and floor:** `H = 24h` (matches the deleted model and the Fix #1
feed spec). **No freshness floor** on the news path — the 10% floor in
`CalculateFreshness` exists so a stale *hard indicator* never silently
vanishes; news is *supposed* to vanish. The decay lives entirely inside `S`
via per-article `wᵢ`; the news path does **not** call `CalculateFreshness`.

**Lookback vs. half-life:** the projector's 120-day lookback
(`ObservationCellProjector.Options.Lookback`, `ObservationCellProjector.cs:83`)
far exceeds the news decay horizon — anything older than ~10 half-lives
(~10 days) contributes `< 0.1%` to `S` and is numerically irrelevant. The wide
lookback stays (it is shared with FRED and the idempotency dedup makes the
overlap free); it does not need to shrink. Persisting the per-article weights
is unnecessary — `S` is cheap to recompute each cycle.

### 2.3 Value-scale dependency (RISK — owned by Fix #1, not here)

The decay math is only interpretable if the per-article contributions `vᵢ` are
the **signed tilt × confidence ∈ [−1, +1]** that the Fix #1 feed spec
mandates (`…news-signal-feed-design.md` lines 36, 60). **Live data currently
violates this:** `macro_observations(source_collector='sentinel')` holds raw
economic values leaking through — e.g. `continuing-jobless-claims` up to
`1.82e6`, `cpi-headline-yoy` to `332`, `retail-sales` to `187393`. With those
inputs `S` is dominated by a single huge `vᵢ` and `tanh` saturates instantly —
the model degenerates. **Fix #6 assumes Fix #1's value contract holds;** this
calibration bug is Fix #1's to close. Verification (§7) MUST first confirm
`|vᵢ| ≲ 1` on the sentinel rows, else the decay model's output is meaningless.
Flag in the cycle log if any `|vᵢ| > 1.5` is seen on the news path so the
contract violation is visible.

---

## 3. News-vs-FRED split

The new aggregation applies **only** to `source_collector='sentinel'` (and
future news collectors). FRED / hard-data groups keep their existing contract
unchanged:

- **FRED** → flat mean (`Σoᵢ/n`) + `CalculateFreshness` step-decay with the 10%
  floor, keyed to publication frequency. A monthly CPI release at 20 days old
  is still meaningful; it must NOT decay on a 24h half-life.
- **News** → decay-weighted-sum + `tanh` squash, **no** `CalculateFreshness`
  call, no floor.

**Split point:** `ObservationCellProjectorCore.Project` already receives the
group's `SourceCollector` (via `ObservationGroup.SourceCollector`,
`ObservationCellProjection.cs:67`). Branch there on a predicate
`IsNewsCollector(sourceCollector)` (initially `== "sentinel"`, table-driven for
the named/unnamed RSS tiers in `SourceTrustConfig`,
`SourceTrustConfig.cs:89-95`). The news branch computes `newsMagnitude` and
skips the freshness call; the FRED branch is the existing code verbatim. The
two paths converge at the shared `commonFactor · weight → clamp` tail, so the
clamp, sector loop, and provenance plumbing are untouched.

`sourceTrust` (FRED 1.0 / sentinel 0.7 / scrape 0.4,
`SourceTrustConfig.cs:77-95`) survives on **both** paths — it is orthogonal to
decay (it is *who said it*, not *how fresh*). It stays as a multiplicative
factor after `newsMagnitude`.

---

## 4. `matrix_cells` write semantics — one recomputed cell per cycle

**Two options:**

| | A. One current cell, recomputed per cycle | B. Restore dated daily buckets |
|---|---|---|
| `evaluated_at` | latest-obs ts — **one** row per `(signal, collector, sector)` | UTC-midnight per day — **one row per day** (the deleted model's shape) |
| Accumulation source | inside `S` (decay-weighted sum over the window) | the *series* of dated cells; reader integrates |
| Idempotency | re-projection hits `ux_matrix_cells_idem`; value changes each cycle as ages advance → **requires #602 DO-UPDATE to heal** (DO-NOTHING freezes the first value) | each day's bucket is write-once; stable key, DO-NOTHING is fine |
| Storage | 1 row/signal/sector — tiny | N rows/signal/sector over the window — fatter, backfill-shaped |
| Reader | reads the single current cell | must integrate the dated series (today/weekly/monthly views) |

**Recommendation: Option A — one recomputed cell per cycle.**

The accumulation now lives **inside the aggregation** (`S`), so we no longer
need a dated-cell time series to encode "sustained." A single cell that is
recomputed every 5-minute cycle and *moves as coverage accumulates and decays*
delivers the intent directly, with far less storage and a trivial reader (the
digest reads one current value per signal). This matches the projector's
existing one-cell-per-group design (`ObservationCellProjector.cs:347-363`) — no
structural change.

**`evaluated_at` choice:** keep the **latest-observation timestamp** the
projector already uses (`ObservationCellProjector.cs:349-350`), NOT cycle time.
Rationale: it keeps the idempotency key stable *across cycles while no new
article arrives*, so an idle signal's cell is one row that gets DO-UPDATE-healed
in place as its decayed value falls — rather than minting a new row every 5
minutes (cycle-time `evaluated_at` would explode the table with 288 rows/day per
signal). When a fresh article lands, `evaluated_at` advances to the new latest
obs and the cell jumps, which is correct.

**Hard dependency on #602.** Option A only works with last-write-wins. The cell
value changes every cycle (ages advance → `S` decays) **even when no new
observation arrives**, but `evaluated_at` is pinned to the latest obs, so the
key `(pattern_id, sector_code, evaluated_at)` is *stable* and every recompute is
a CONFLICT. Under today's DO-NOTHING swallow
(`MatrixCellRepository.cs:62-77`) the **first** value written for that key
sticks forever and the decay never lands in the table — the cell freezes. PR
**#602** (`fix/ws3/heal-on-rewrite`, OPEN) converts this to a single
parameterized `INSERT … ON CONFLICT (pattern_id, sector_code, evaluated_at) DO
UPDATE … WHERE <value cols> IS DISTINCT FROM EXCLUDED` — exactly the heal Fix #6
needs. **Fix #6 MUST merge after, or together with, #602.** Shipping Fix #6 on
top of DO-NOTHING produces a silently-frozen first-value cell — a worse failure
than the current averaging bug because it *looks* fixed.

Secondary #602 coupling: #602 changes `IMatrixCellRepository.WriteBatchAsync`
from `Task<int>` to `Task<MatrixCellWriteResult>` (insert vs. heal counts). The
projector's call site (`ObservationCellProjector.cs:236`,
`inserted/skipped` logging at `:237,246-249`) must adopt the new return shape.
This is a merge-order coupling, not new logic — implement Fix #6 on top of the
#602 branch (or rebase #602 first), don't duplicate the int contract.

---

## 5. Code integration points (file:line)

1. **`ObservationCellProjectorCore.Project`**
   (`ThresholdEngine/src/Services/ObservationCellProjectorCore.cs:65-138`) —
   the one substantive change. Replace the unconditional `alphaMean = Σoᵢ/n`
   (lines 76-86) + `CalculateFreshness` (lines 88-91) with a branch:
   - **news** (`IsNewsCollector(group.SourceCollector)`): compute
     `S = Σ vᵢ·exp(−ln2·ageᵢ_h/H)`, `newsMagnitude = K·tanh(S/K)`, set
     `freshness = 1m` (decay already inside `S`), use `newsMagnitude` as the
     magnitude term. **Needs per-article ages**, which `ObservationGroup.Values`
     (`ObservationCellProjection.cs:69`) does NOT carry today — see point 2.
   - **FRED / default**: existing `alphaMean` + `CalculateFreshness` path
     verbatim.
   `K` and `H` arrive via `SignalScoringInputs` or a new `NewsDecayOptions`
   record (config-bound, defaults `K=2.0`, `H=24h`). `tanh` is `double`-only in
   .NET, so cast `S` → `double` for the squash and back to `decimal`; the
   intermediate `decimal` `S` is exact, the squash is the only float step.

2. **`ObservationGroup`** (`ObservationCellProjection.cs:66-71`) — `Values` is
   `IReadOnlyList<decimal>`; the news path needs **(value, age)** pairs. Add a
   parallel `IReadOnlyList<(decimal Value, double AgeHours)>` (or carry the raw
   `ObservationTime`) so the core can weight per article. FRED ignores ages.
   The projector already has each row's `ObservationTime`
   (`ObservationCellProjector.cs:349,353-356`) — thread it through instead of
   discarding it at the `.Select(r => r.ValueNumeric!.Value)` projection
   (`:353-356`).

3. **`ObservationCellProjector.BuildCellsAsync`**
   (`ObservationCellProjector.cs:347-371`) — stop flattening to bare values;
   build the `(value, ageHours)` list from `groupRows` (age = `nowUtc − r.ObservationTime`).
   Leave the latest-obs `evaluatedAt` pin (`:349-350`) unchanged (§4).

4. **News-collector predicate** — new helper, sourced from the
   `SourceTrustConfig` tier table (`SourceTrustConfig.cs:73-95`: `sentinel`,
   `sentinel-rss-named`, `sentinel-rss-unnamed`, `searxng`) so it tracks the
   trust tiers rather than a hard-coded literal.

5. **Write path** (`ObservationCellProjector.cs:236`) — adopt #602's
   `MatrixCellWriteResult` (insert/heal counts) once #602 is merged; update the
   cycle log (`:246-249`) to report healed vs. inserted.

6. **Untouched:** `CellProjector.cs` (FRED-shared, do NOT add news decay here);
   the ±3 clamp + sector loop in `ObservationCellProjectorCore` (lines 97-124);
   `ToEntities` provenance plumbing (`ObservationCellProjector.cs:447-482`) —
   `Freshness=1m` on news rows is correct and audit-legible.

---

## 6. Testing strategy

**Unit (`ObservationCellProjectorCore`), the high-value surface:**

- *Sustained reinforces.* 5 obs of `+0.6` at ages 0..4d → `newsMagnitude`
  strictly greater than 1 obs of `+0.6`. (A flat/weighted mean fails this — it
  is the regression guard.)
- *Single stale decays.* 1 obs of `+0.6` at age 7d → `newsMagnitude ≈ 0`
  (< 0.01), **no 10% floor**. Contrast with a FRED group at the same age →
  freshness ≥ 0.10.
- *Bounded.* 50 obs of `+1.0` at age 0 → `newsMagnitude → K (≈2.0)`, never
  exceeds `K`; final cell within ±3.
- *Sign preserved.* mixed `+`/`−` obs net the correct sign; symmetric
  cancellation → ~0.
- *Decay-over-cycles.* same group projected at `now` vs `now+24h` (no new obs)
  → magnitude halves. This is the "self-heal as it ages" property.
- *Split.* identical `(value, age)` set tagged `fred` vs `sentinel` →
  different magnitudes (FRED step-decay+floor vs news 24h half-life).
- *Half-life exactness.* one obs at 24h → weight `0.5±1e-9`; at 48h → `0.25`.
- *Future-stamped guard.* obs with `age < 0` (clock skew / bad feed date) →
  `w` clamps to 1.0, no negative-exponent blowup.

**Integration (TimescaleDB, gated by #602):**

- Project a seeded sustained group across two cycles → assert the cell row is
  **DO-UPDATE-healed in place** (same `(pattern_id, sector_code, evaluated_at)`,
  changed `cell_value`), NOT a frozen first value and NOT a duplicate row. This
  is the #602-dependency proof; on a DO-NOTHING substrate it must fail.

**No** test on `CellProjector`/FRED freshness changes — that path is unchanged;
re-testing it is coverage-chase.

---

## 7. Mandatory real-data verification (post-deploy)

Per COMPLETION_GATE — run unsolicited after deploy, report pass/fail with
actual cell values:

1. **Input-scale gate (Fix #1 dependency).** Confirm sentinel `value_numeric`
   is bounded to ~[−1, +1]:
   `SELECT signal_identity_id, min(value_numeric), max(value_numeric) FROM
   macro_observations WHERE source_collector='sentinel' AND observation_time >
   now()-interval '10 days' GROUP BY 1;`
   If any `|v|>1.5`, **STOP** — the decay model output is meaningless until
   Fix #1's calibration is corrected. (Today this gate FAILS: live max is 1.8e6.)
2. **Sustained vs single-event cell values.** Pick a signal with sustained
   recent coverage (e.g. `cpi-headline-yoy`, 14 obs) and one with a single
   recent obs (e.g. `challenger-job-cuts`, 1 obs). Read their projector cells
   (`pattern_id LIKE 'obs:%:sentinel'`):
   - sustained-signal cell magnitude **>** single-event cell magnitude for
     comparable per-article tilt → the accumulation works.
   - a signal whose only obs is > 1 week old → cell ≈ 0 → the decay works.
3. **Heal-in-place.** Across two projector cycles ~24h apart with no new obs for
   an idle signal, confirm its cell row's `cell_value` **dropped** (decayed)
   while `evaluated_at` stayed fixed → DO-UPDATE healed (proves #602 live).
4. **FRED unaffected.** A FRED signal's cell is unchanged vs. pre-Fix-#6
   (its freshness still steps on publication frequency, not 24h).
5. **Loki:** no `23505` spam regression (#602), no `invalid_sector_weights` /
   `read cap` warnings introduced; cycle log shows news groups projecting with
   `Freshness=1.0` and a sane magnitude.

---

## 8. Interaction with matrix-hygiene items (#602 / #4 / #5)

- **#602 heal-fix — HARD prerequisite** (§4). Fix #6 on DO-NOTHING = silently
  frozen first-value cell. Merge #602 first (or land Fix #6 on the #602 branch).
  Also a compile coupling via `WriteBatchAsync`'s new `MatrixCellWriteResult`.
- **Unknown-signal skips (#5).** Orthogonal but gating *visibility*: a news
  signal with no enabled ThresholdEngine pattern is dropped as
  `unknown_signal` (`ObservationCellProjector.cs:327-335`) **before** the
  aggregation ever runs. Fix #6 makes accumulation correct *for signals that
  reach the projector*; fresh news signals (`breakeven-*`,
  `initial-jobless-claims` at 06-03/07-08) still need pattern mappings (the #5
  reconciliation) or the improved decay is invisible. Note, don't fix here.
- **Future-dated obs (data hygiene).** Live data shows
  `initial-jobless-claims` at **2026-07-08** (future) — `ageHours < 0` →
  `wᵢ` must clamp to 1.0 (deleted model's clock-skew guard, lines 274-277).
  Carry the same `age ≤ 0 → w = 1.0` guard so a future-stamped row doesn't
  produce a weight > 1 or a negative-exponent blowup. The bad timestamp itself
  is an upstream feed defect, not Fix #6's to repair.
- **Path-1 (`MatrixCellPersistenceWorker`).** Untouched by Fix #6 (FRED→event,
  NULL-signal cells). Its fate (#5) is out of scope.

---

## 9. Open questions / risks

1. **`K` calibration.** `K=2.0` is a reasoned default (keeps post-trust cells
   inside ±3 with gradation), not empirically tuned. Validate against real
   sustained-coverage volumes in §7.2; expose as config.
2. **Fix #1 value-scale (highest risk).** The decay model is a no-op-to-garbage
   unless `vᵢ ∈ [−1,1]`. Live data violates this today. Fix #6 cannot be
   verified until Fix #1's `Tilt × Confidence` calibration lands cleanly. Hard
   gate in §7.1.
3. **`tanh` vs alternatives.** `tanh` chosen for smooth saturation + linear
   near-zero. A logistic or a soft-clamp (`S/(1+|S|/K)`) would also work; `tanh`
   is the simplest with the right asymptotics. Not load-bearing — swap-able
   behind the `K` knob if a sharper knee is wanted.
4. **Cross-source coexistence.** A signal with BOTH a FRED group and a sentinel
   group writes **two** cell rows (distinct `obs:{signal}:{collector}`
   pattern ids) — by design (#601 magnitude-comparability). The decay split
   keeps them independently scored; the matrix reader must continue to treat
   the FRED and sentinel contributions to one signal as separate, comparable
   rows. No change, but the spec must not accidentally merge them.
5. **Half-life per source-tier.** Deferred. The deleted model's options carried
   a per-source half-life table (24h default). All news tiers share `H=24h`
   for now; a future override (flash earnings 4h, slow macro commentary 7d) is
   a config extension, not part of this fix.

---

## 10. Summary

Replace the news path's flat 120-day mean with a **per-article 24h-half-life
decay-weighted sum, soft-saturated through `K·tanh(S/K)` (K≈2)**, fed into the
unchanged `sourceTrust · temporal · confidence · sectorWeight → ±3 clamp` tail.
Split strictly by `source_collector`: FRED keeps `CalculateFreshness`
step-decay + 10% floor; news gets no floor (perishable by design). Persist one
recomputed-per-cycle cell at the latest-obs `evaluated_at`, **healed in place by
#602's DO-UPDATE** — which is a **hard merge prerequisite**. Accumulation now
lives inside the aggregation, so no dated-bucket time series is needed.
Verification must first gate on Fix #1's `[−1,1]` value contract (today
violated), then prove sustained-cell > single-event-cell and stale-cell ≈ 0 on
real data.
