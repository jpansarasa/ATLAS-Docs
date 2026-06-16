# Design Spec — Regime News-as-Staleness-Perturbation Redesign

**Status:** PROPOSAL / for review — NOT approved, NOTHING deploys from this doc.
**Origin:** user ↔ supervisor design conversation, 2026-06-16 (follow-on from the regime-pipeline remediation #722/#724/#725/#726/#727).
**Author:** Claude (supervisor), capturing user-stated design intent.
**Gate before any cutover:** shadow-run comparison vs. the current flat-sum + explicit user approval.

> Curation note: this is a *plan*, not current-state reference. Per `docs/README.md` / CLAUDE.md PHASE_TAGS, plans are not kept on `main` — this PR is the **review surface only**; on approval it drives implementation PRs and is retired (recorded in `docs/RELEASES.md`), not merged to main.

---

## 1. Problem

The sector regime (`SectorRegimeProjectionWorker`) currently lets **news dominate and override** the lagging fundamental benchmark, and it measures **press coverage volume**, not **economic significance**. Two consequences:

- News is ~**64%** of the total net tilt and **flips the benchmark's sign in 6 of 11 sectors** (live, 2026-06-16):

  | Sector | Benchmark (FRED/OFR) | News | Result |
  |---|---|---|---|
  | FINANCIALS | +2.02 | −6.64 | −2.31 severe_contraction (flipped) |
  | ENERGY | +2.44 | −5.33 | −1.45 contraction (flipped) |
  | MATERIALS | +3.14 | −4.96 | −0.91 contraction (flipped) |
  | INDUSTRIALS | +2.83 | −4.14 | −0.66 contraction (flipped) |
  | CONS_DISC | +0.56 | −3.56 | −1.50 contraction (flipped) |
  | UTILITIES | −0.06 | +4.76 | +2.35 overheating (news is the entire signal) |
  | INFOTECH | +2.01 | −0.41 | +0.80 expansion (benchmark wins) |

  Aggregate |contribution|: news 85.8 vs benchmark 48.3 across 283 vs 195 cells.

- The most influential news signals are exactly the heavily-covered macro topics, and **they are sourced *only* from news — no real series behind them** (live, 30d):

  | signal | only source | 30d obs | real series that exists |
  |---|---|---|---|
  | oil-price | sentinel (news) | 802 | FRED `DCOILWTICO` (daily) |
  | natural-gas-price | sentinel | 177 | FRED `DHHNGSP` (daily) |
  | ust-10y-yield | sentinel | 257 | FRED `DGS10` (daily) |
  | dxy-dollar-index | sentinel | 331 | FRED `DTWEXBGS` (daily) |
  | fed-funds-rate | sentinel | 165 | FRED `DFF` (daily) |
  | cpi-headline-yoy | sentinel | 569 | FRED `CPIAUCSL` (monthly print) |

  i.e. the system estimates the **price of oil by counting how many articles mention it** instead of reading the daily print.

### Three compounding failure modes
1. **Coverage amplification** — one event ("X lays off 10,000") → many articles → many summed contributions.
2. **Breadth/depth inversion** — 10 firms × 1,000 (broader, more significant) can score ≤ 1 firm × 10,000 that got more coverage. The pipeline can't distinguish "1 event, 10 articles" from "10 events, 1 article."
3. **No anchoring / no staleness coupling** — the inflated news sum is flat-added to the benchmark and dominates, regardless of whether the benchmark was printed yesterday or 90 days ago.

---

## 2. Design intent (from user)

- Each sector has a **grounding benchmark** from FRED/OFR. These are **at best monthly, often quarterly, lagging 30–60–90 days** behind reality.
- **Sentinel/news** is a **fast-decaying coincident perturbation** that nudges the lagging benchmark toward a coincident value (e.g. UNRATE says one thing, but sustained layoff news perturbs the sector toward the current reality).
- **News decays fast** (a single story fades in hours/days); **sustained** news indicates a coming change to the benchmark itself.
- **Keystone:** news influence must scale with **benchmark staleness** — minimal right after a fresh print, growing as the anchor ages toward/past its next expected release. Staleness *is* the principled governor (no arbitrary cap).
- Measure **economic significance** — distinct occurrences × magnitude × net direction — not article count.
- **Not everything has a daily benchmark.** Oil / yields / currency do (use the real series); materials / staples / infotech do not (news is the only coincident signal — keep it, measure it right).
- **Sustained ≠ duplicate:** oil up Mon/Tue/Wed = three real occurrences that should accumulate; 10 outlets covering one layoff = one occurrence. Dedup is **per occurrence, not per topic**.

---

## 3. Current state (code-grounded)

- **Cell formula** (`ObservationCellProjectorCore`): `cell = magnitude × sourceTrust × freshness × temporal × confidence × sectorWeight`, clamped ±3.
- **Hard-data magnitude** = compiled `signalExpression` eval (#718); `freshness` = `CellProjector.CalculateFreshness` (publication-frequency step-decay, 10% floor) — this is the **`(1 − staleness)` term on the benchmark** and already shrinks the anchor as it ages.
- **News magnitude** (Fix #6) = `K·tanh(S/K)`, where `S = Σ vᵢ·exp(−ln2·ageᵢ/H)` over **per-article** contributions; `K = 2.0` (`SaturationScale`), `H = 24h` (`HalfLifeHours`); `freshness = 1.0` (decay lives in `S`).
  - Note: the per-signal `tanh` bounds a single signal to ±K, but that **worsens** the breadth/depth inversion — an over-covered single event and a broad genuine signal **both saturate to ~2**, becoming indistinguishable.
- **Regime aggregation** (`SectorRegimeProjectionWorker.Aggregate`): `score = clamp(Σ cell_value × RegimeTiltScale, ±3)` — a **flat cross-signal sum**; news cells outnumber benchmark cells and news weight is **not coupled to benchmark staleness**.
- **News observations** are keyed `{rawContentId}:sig:{signalId}` — **one row per (article, signal)**, no event-level dedup (1,530 distinct articles → 3,090 obs in 7d).
- **Daily-observable quantities are news-only** (table in §1).

---

## 4. The redesign — five pieces

### Piece 1 — Daily-observable quantities → real FRED/OFR series
Wire the actual series and **remove these from the news channel**:
`DCOILWTICO` (oil), `DHHNGSP` (nat-gas), `DGS10` (10y), `DTWEXBGS` (broad dollar), `DFF` (fed funds), real `CPIAUCSL` print.
- Effect: accurate daily value replaces hundreds of article-counts; gives daily-driven sectors (energy←oil/gas, financials & real-estate←yields, etc.) a genuine **fast benchmark**; removes the worst coverage-volume offenders in one move; and — via the staleness crossfade (Piece 4) — these never-stale signals get ~zero news weight automatically.
- Lives in: FredCollector series config (+ SecMaster signal-identity registration + sector mapping). Largely **additive / low-risk**.
- **Verify at impl time:** exact FRED mnemonics + availability; current FredCollector series set; which sectors each series drives.

### Piece 2 — News dedup per *occurrence* (not topic)
Collapse redundant coverage of one occurrence; preserve distinct occurrences across time/entity.
- Method (recommended): **tight time-windowed semantic clustering** (bge-m3 embeddings already in stack; cluster within ~a day by similarity → one cluster per occurrence). Per [[feedback_rag_over_deterministic_gates]] — semantic, not a brittle entity+keyword key.
- Preserves: oil up Mon/Tue/Wed = 3 clusters; 10 firms = 10 clusters; 10 outlets on 1 layoff = 1 cluster.
- Lives in: SentinelCollector (ingestion → event) and/or the news→observation mapping.

### Piece 3 — News aggregated by significance, not article count
Replace the per-article `S` with a per-**occurrence** aggregation:
- distinct-entity **breadth** + **net direction** + decay; **magnitude** (headcount/$) as **v2** pending extraction-schema support.
- Re-examine the `tanh(S/K)` saturation so genuine breadth is distinguishable from a single over-covered event (today both saturate to ~2).
- Lives in: `ObservationCellProjectorCore` news path (`ComputeNewsMagnitude`) + `NewsDecayOptions`.

### Piece 4 — Staleness crossfade (keystone)
News weight = `g(benchmark_age / expected_cadence)`:
- Benchmark side `(1 − staleness)` **already exists** via `CalculateFreshness`.
- **Missing half:** news weight is coupled only to *news* recency (24h decay), not *benchmark* staleness. Add `news_contribution × g(staleness)` where `g` rises as the anchor ages toward/past its next expected release.
- Fresh anchor ⇒ news suppressed; maximally-stale anchor ⇒ news leads. This is the principled bound that replaces "caps."
- Needs: per-signal **expected release cadence** (CalendarService carries the FRED release schedule) → staleness as a fraction of the period.
- Lives in: regime aggregation (`SectorRegimeProjectionWorker.Aggregate`) + a staleness input per signal/sector.

### Piece 5 — Regime aggregation = benchmark base + staleness-scaled news perturbation
`score(sector) = clamp( benchmark_net + Σ_news[ news_contribution × g(staleness) ] , ±3 )` (net-directional; trust still baked into cell_value). Replaces the undifferentiated flat sum.

---

## 5. Phasing (each shadow-run before cutover)

- **Phase 1** — Piece 1 (real daily series, remove from news). Highest leverage, lowest risk, mostly additive. Likely resolves a large fraction of the domination on its own.
- **Phase 2** — Pieces 2+3 (occurrence-dedup + significance aggregation) for the no-series sectors.
- **Phase 3** — Pieces 4+5 (staleness crossfade + benchmark-anchored aggregation).

Shadow mode: compute the redesigned regime alongside the live flat-sum, compare per-sector over ≥ a few cycles, then cut over. The `te-sector-regime-all-neutral` alert (already deployed) guards the cutover.

---

## 6. Resolved decisions (user, 2026-06-16)
1. **Magnitude extraction** — **v2.** v1 ships dedup + distinct-entity breadth + net-direction; magnitude (headcount/$) deferred to v2 pending extraction-schema support.
2. **`g()` shape** — **sigmoid** (flat while the benchmark is fresh, ramps near the next expected release). Not linear.
3. **Cadence source** — **inferred historical cadence** (derive each signal's period from its own observation history). Not CalendarService → removes a cross-service dependency.
4. **Dedup method** — **embedding-cluster first** (soft preference); keep `entity + event-type + time-key` as a documented fallback if the embedding clustering skews.
5. **Daily-series → sector mapping** — **research required** (see §8). User hypothesis: each sector has a well-established benchmark. Pending user validation of the proposed mapping.

## 7. Risks
- Over-dedup killing sustained signals → mitigated by per-occurrence (not per-topic) clustering + net-direction accumulation.
- Inferred-cadence error on sparse/new signals → default staleness conservatively (treat unknown cadence as "fresh" so news doesn't over-lead) until enough history accrues.
- Phase 1 sector remap could shift regimes → shadow-run + the all-neutral alert.
- Embedding-cluster cost/latency in the news path → batch within the existing Sentinel pipeline; deterministic-key fallback (§6.4) if clustering skews.

## 8. Research — sector → benchmark mapping

### 8.1 Current-state finding (grounded, live 2026-06-16)
**There are no sector-specific benchmarks today.** All 11 ATLAS sectors draw from the *same* shared panel of ~16–20 generic macro signals (`gdp-real`, `cpi-core/headline-yoy`, `unemployment-rate`, `nonfarm-payrolls`, `initial/continuing-jobless-claims`, `retail-sales`, `industrial-production`, `pce-headline-yoy`, `sahm-rule`, `job-openings`, `um-consumer-sentiment`, `equipment-investment`, `global-pmi`, `ofr-fsi-composite`, `sofr-repo-dispersion`, `hf-leverage-gav-median`, `manufacturing-employment`, `housing-starts`), differentiated **only by `sectorWeights`**. So:
- Energy's benchmark contains **no oil**; financials' contains **no yield curve / credit spreads**; infotech's contains **no semiconductor/tech series**; materials' contains **no metals/PPI**.
- The user hypothesis ("each sector has a well-established benchmark") is correct *in principle* but **not implemented** — the regime currently grounds every sector in the same broad macro reading.

This reframes Piece 1: it's not merely "move oil from news to data," it's **establish a sector-specific fast/regular benchmark per sector** where a well-known one exists.

### 8.2 Proposed per-sector benchmark mapping — DRAFT, needs user validation
Cadence: D=daily, W=weekly, M=monthly, Q=quarterly. FRED mnemonics are candidates to verify against the FredCollector config at impl time.

| Sector | Proposed sector-specific benchmark (candidate series) | Fastest cadence | Today? |
|---|---|---|---|
| ENERGY | WTI `DCOILWTICO` + Henry Hub `DHHNGSP` (+ EIA inventories W) | **D** | none (oil is news-only) |
| FINANCIALS | yield curve `T10Y2Y` + 10y `DGS10` + IG credit `BAMLC0A0CM` + OFR FSI | **D** | OFR FSI only; no rates/credit |
| UTILITIES | rates `DGS10` (rate-sensitive) + electricity demand/price (EIA) | **D** (rates) | none |
| REAL_ESTATE | mortgage `MORTGAGE30US` (W) + `DGS10` (D) + housing starts `HOUST` (M) + Case-Shiller (M) | **W/D** | housing-starts (M) only |
| MATERIALS | industrial metals (copper) + PPI metals/chemicals (M) | **D (metals) / M** | none sector-specific |
| INDUSTRIALS | ISM Mfg PMI (M) + `INDPRO` (M) + durable-goods orders (M) | **M** | IP + global-pmi (generic) |
| CONS_DISC | retail sales `RSAFS` (M) + `UMCSENT` (M) + auto sales (M) | **M** | retail-sales (generic) |
| CONS_STAPLES | staples CPI components (M) + `UMCSENT` (M) | **M** | generic only |
| HEALTHCARE | healthcare CPI/PCE + healthcare employment (M) | **M** | generic only |
| INFOTECH | semiconductor billings/book-to-bill (M) + tech capex | **M** | generic only |
| COMM_SVC | ad spend + subscriber/usage proxies | **M/none** | generic only |

Takeaway consistent with user's "oil/yield/currency yes; materials/staples/infotech no": **~4 sectors (energy, financials, utilities, real-estate) have a genuine daily/weekly market benchmark to wire; the rest are monthly-fundamental** — which is exactly where news is the sole coincident layer and the dedup/significance/staleness work (Pieces 2–4) matters most.

### 8.3 Open for user
- Validate / correct 8.2 (your domain call — especially materials, infotech, comm-svc where the "well-established benchmark" is least obvious).
- For monthly-only sectors: confirm the generic macro panel + news (no daily series) is acceptable, or name a sector-specific monthly series to add.
- Whether sector-specific benchmarks **replace** or **augment** the shared macro panel in each sector's weighting.
