# Sentinel → Matrix break map (post-CoD-cutover)

**Date:** 2026-06-05
**Question:** Why is the daily digest empty of signal, and why is the matrix stale? Trace the *whole* break before fixing.

**STATUS (2026-06-05) — this doc is the durable diagnosis; live status is in `STATE.md`.** Fixes landed + LIVE: feed (#613), matrix-cell heal (#602), news decay/accumulation (#615), resolve-entities timeout (#616). The **signal** dimension is RESTORED + verified. Remaining: the **sector** dimension is blocked one layer up at SecMaster instrument-NAICS coverage (~10%); plus the `#5` pattern reconciliation (signal breadth) and the digest VIEW. See `STATE.md` → NEXT.

## Executive summary

The digest is a symptom. The real failure: **the CoD/DSL extraction cutover (`ff72eb8f`, Phase 5.8, live 05-29/05-30) plus the A3 matrix-realignment (`0be162dd`/#601, 06-02) together severed Sentinel's actual value — sustained news as a fast-decaying *leading counterweight* to lagging hard data — at every layer of the pipeline.** Six breaks, one root cause (the CoD path changed the *shape* of two extraction fields everything downstream keyed on) plus a design regression in the matrix scoring.

Intended pipeline (post-#601): `news → extracted_observations (sector+signal tagged) → macro_observations(source=sentinel) → ThresholdEngine ObservationCellProjector → matrix_cells (signal×sector, decayed/accumulated)`.

## The breaks

**1. News stopped feeding the matrix (the counterweight input).** `MacroObservationRouter.TryPlanMacroWrite` (`SentinelCollector/.../MacroObservationRouter.cs:79-136`) gates on `Value != null` AND `MacroSignalIdentityCatalog.TryResolve(Description)` — an **exact-match** alias lookup. V1 descriptions were canonical labels ("CPI", "initial jobless claims") and matched. CoD descriptions are CLAIM predicate phrases ("inflation firming", "above-target inflation") with degenerate `value` 1.0/0.0 → never match → **no `macro_observations` row, no `signal_identity`**. Last `source_collector='sentinel'` macro_observation: **2026-05-28**. *Regression, root = `ff72eb8f`.*

**2. News sector tagging → 0.** `atlas_sector_code` is written only via `ExtractedObservation.UpdateResolution(...)`, and the *only* path that ever passed a sector was V1 `ticker_in_quote` (`ExtractionProcessor.cs:544`), which needs an `EXCHANGE:TICKER` in `text_quote`. CoD `text_quote` is a bare value fragment ("$75 billion") with no ticker → 0 since 05-29. Compounding: the V2/CoD path `V2ExtractionPipeline.BuildObservationFromV2Result` (`:157`) **omits the `atlasSectorCode` argument entirely** → always writes `null`. The GPU `SectorTagger` runs as enrichment but its result is **discarded post-#601** (`ExtractionProcessor.cs:1138`) — never persisted to the row. *Regression + latent gap.*

**3. Signal-identity mapping dead** — same gate as #1 (catalog exact-match on the fragment description). No other signal-identity assignment exists for news. *Regression, same root.*

**4. Matrix `23505` errors are LOG SPAM, not the stall.** `ObservationCellProjector` (authoritative, 5-min cycle) reads `macro_observations`, groups by `(signal_identity_id, source_collector)`, writes `matrix_cells`. Live cycle: `CellsBuilt=55 CellsInserted=0 CellsIdempotentSkipped=55` — it does **not** crash; the ~220 `23505`/cycle are EF's internal logging of INSERTs that *are* correctly swallowed (`MatrixCellRepository.cs:62-77`, effective DO NOTHING). PR **#602** (`fix/ws3/heal-on-rewrite`, OPEN, 9 commits behind main, no CI run) converts this to `ON CONFLICT DO UPDATE` — kills the spam + enables value-heal, but does **not** fix freshness.

**5. Matrix staleness real cause = UNKNOWN-SIGNAL skips + legacy Path-1.** The projector skips 25 groups as `GroupsSkippedUnknownSignal` — fresh signal observations (`breakeven-*`, `initial-jobless-claims` reaching 06-03/07-08) have **no enabled ThresholdEngine pattern mapping their `signal_identity_id`** (`ObservationCellProjector.cs:417-437`) → dropped. The freshest *mappable* signal vintage is **05-21** → that's exactly why signal-keyed cells froze there. Separately, legacy **Path-1** (`MatrixCellPersistenceWorker`, FRED→event→persist) still runs and writes the 226 NULL-signal cells up to 06-01 — #601 only retired Path-2 (Sentinel gRPC bridge), not Path-1.

**6. The alpha-decay / sustained-flow / counterweight MODEL was lost in A3.** The deleted `SignalMagnitudeCalculator` (`git show 0be162dd^:.../SignalMagnitudeCalculator.cs`) implemented the news semantics: `weight = confidence × freshness × alphaFactor`, with **freshness = exp(−ln2 × ageHours/24h)** (per-article, 24h half-life, no floor) and **daily-UTC bucketing** → a *decaying time series of dated cells* = sustained-flow accumulation. The new `ObservationCellProjectorCore` (`:14-16,80-95`) instead: (a) reuses **FRED's** freshness (step + publication-frequency staleness, 10% floor, keyed to the *latest* obs age — not per-article); (b) collapses the 120-day group to a single **arithmetic mean** `Σoᵢ/n` (sustained flow **averages away**, doesn't accumulate); (c) `evaluated_at` = latest-obs timestamp + DO NOTHING → one cell per signal that jumps forward, **no time series**. The `sourceTrust` α-tier (FRED 1.0 / sentinel 0.7 / scrape 0.4) is the only surviving counterweight; the **fast-decay that encoded news's leading-but-perishable nature is gone** — news now decays on the same schedule as FRED. So even if news fed the matrix, a burst of sustained sector coverage ≈ a single stale article.

## Root cause

One upstream change — the **CoD/DSL backend cutover** — changed `Description` (canonical label → CLAIM phrase) and `text_quote` (source sentence → value fragment), breaking the macro-route, sector-tag, and signal-identity hooks (#1–#3). **#601** then made `macro_observations` the *sole* matrix feed (so those mismatches fully sever the signal) **and** replaced the news-aware magnitude model with FRED's hard-data contract (#6). #5 is an orthogonal pattern-config gap; #4 is cosmetic.

## Coherent fix sequence (dependency order)

1. **Restore the news→`macro_observations` feed** (#1/#3): route CoD observations to signal identities without depending on exact-match on fragment descriptions — classify CoD CLAIM/NUM blocks to signal identities (and emit a real magnitude/polarity, not 1.0/0.0). *Without this, nothing flows; everything else is moot.*
2. **Restore sector tagging** (#2): V2 pipeline must pass `atlasSectorCode` to `UpdateResolution`, sourced reliably for fragments (persist the SectorTagger result, or tag from article text).
3. **Restore the news magnitude/decay/accumulation model** (#6): re-implement per-article 24h-half-life freshness + daily-bucket accumulation (the sustained-flow counterweight) in the projector's news path — distinct from FRED's staleness contract.
4. **Matrix freshness/hygiene** (#4/#5): rebase + merge #602 (stop spam, enable heal); reconcile fresh signal identities → enabled patterns (the unknown-signal skips); decide Path-1's fate.
5. **Then the digest** (the view): with sustained news signal flowing into the matrix, render news-bias-vs-matrix (daily = today, weekly/monthly = sustained).

## Evidence appendix
- Feed/tag: `MacroObservationRouter.cs:79-136`, `MacroSignalIdentityCatalog.cs:104-119`, `ExtractionProcessor.cs:544,1138`, `V2ExtractionPipeline.cs:157`. DB: sentinel macro_observations stop 05-28; atlas_sector_code 0 since 05-29; `ticker_in_quote` 0 since 05-29.
- Projector: `ObservationCellProjector.cs:203-217,285-363,417-482`, `MatrixCellRepository.cs:56-77`, `MatrixCellPersistenceWorker.cs`, `CellProjector.cs:40-96`. PR #602 OPEN.
- Magnitude model: `git show 0be162dd^:SentinelCollector/src/Semantic/SignalMagnitudeCalculator.cs`; `ObservationCellProjectorCore.cs:14-95`; `SourceTrustConfig.cs:73-96`.
