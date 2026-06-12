# Digest Matrix-First Redesign — 5-Phase Plan

The Sentinel digest is rebuilt around the signal×sector matrix (`matrix_cells`,
ThresholdEngine WS3 projector output) as its spine, replacing the theme taxonomy.
Each phase ships green and independently deployable.

STATUS: Phase 1 merged (#693) · Phase 2 merged (#695) · Phase 3 merged (#697) ·
Phase 5 merged (#694) · Phase 4 = this PR (final).

## Phase 1 — merged (#693): matrix read layer + sector heat table — additive

- New `MatrixSectorQueryService` (raw Npgsql, SELECT-only) reading the latest
  `matrix_cells` per (pattern, sector) over a lookback window (`obs:` prefix
  filters legacy NULL-collector rows) and the latest `sector_regimes` per sector.
- New pure `SectorMatrixBuilder` → exactly 11 `SectorMatrixRollup` records (one
  per `AtlasSectorCodes.All`): NetScore = Σ latest cell values, ContributingSignals
  = non-zero cells, TopSignals by |cell_value| joined to news momentum by signal id.
- Regime optional: `sector_regimes` is EMPTY in prod (publisher dead until Phase 5);
  regimes attach only when fresh (RegimeMaxAgeDays) AND contributing_pattern_count > 0.
- `DigestStats.MatrixSectors` nullable (JSON `matrix_sectors`; old blobs round-trip);
  same resilience contract as momentum/cross-collector — any failure logs +
  `digest.matrix_failed` span event, digest ships with null MatrixSectors.
- Renderer: Sector Heat markdown table after the Theme Radar (radar untouched this
  phase); explicit "matrix unavailable" / "no matrix signal" states.
- Aggregate serializer: compact sector-matrix block for the LLM prompt (top 3
  signals per sector).
- `DigestOptions.SectorMatrix`: LookbackDays=7, RegimeMaxAgeDays=7,
  TopSignalsPerSector=5, DetailedSectorCount=6 (+ validator clauses).

## Phase 2 — merged (#695): article→sector grounding

- `ArticleSignalsSql` gains `atlas_sector_code`.
- `ArticleSectorResolver` precedence: max-|tilt| signal sector → first non-null
  signal sector → row.AtlasSectorCode → Macro-wide bucket.
- Selector + narrative `ArticleBlock` keyed by sector; `NarrativePerThemeCap` →
  `NarrativePerSectorCap`.

## Phase 3 — merged (#697): sector-skeleton digest

- Renderer reordered: narrative → sector heat strip → per-sector blocks (top
  `DetailedSectorCount` by ArticleCount + |NetScore|, then Macro-wide) → momentum
  → cross-collector → word cloud → certainty → sources.
- DELETE Theme Radar + ByTheme sections.
- `narrative.md` prompt rewritten: Executive Take / Sector Watch / Cross-Sector
  Signals / Noteworthy One-Offs.
- Push notification leads with top sector.

## Phase 4 — this PR (final): deletion sweep

- Delete `ThemeClassifier.cs`, `DigestTheme`, `IThemeClassifier`, theme metric
  `sentinel_digest_theme_observations_total`, DI line, tests; grep gate zero.
- Grafana `sentinel.json` theme panel → sector panel.
- AGENT_README / docs/MATRIX.md updates.

## Phase 5 — merged (#694) (ThresholdEngine, parallel after Phase 1): regime publisher

- New `SectorRegimeProjectionWorker` publishing `SectorScoreEvent` per projector
  cycle from latest `matrix_cells` aggregates — the hosted
  `SectorRegimePersistenceWorker` then persists `sector_regimes`.
- Do NOT wire `ObservationEventSubscriber` wholesale (ThresholdEngine card
  forbids: duplicate cell-write path).

## Risks

- Regime dead today → render n/a (honest sparsity).
- Sparse matrix (~30 signals project) → render ContributingSignals explicitly.
- Heal-on-rewrite means historical regeneration approximates.
- Macro-wide bucket capped + excluded from heat ranking.
- Momentum SQL vs article-signal SQL intentionally diverge (do not unify).
- Prompt size bounded by DetailedSectorCount + AssembleWithinBudgetAsync.
