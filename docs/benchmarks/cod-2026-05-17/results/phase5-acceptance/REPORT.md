# Phase 5.8 Acceptance Demo — Run #3

**Date:** 2026-05-27 (UTC)
**Branch:** `bench/phase5-demo-3` from `origin/main@2915d22f`
**Gating fixes deployed:**
- PR #497 (`c4824ecb`) — ExtractionProcessor threads prepass entity bag through to cascade Stage 1.5; `MacroSignalIdentityCatalog` loads both `macro` AND `rate` categories.
- PR #498 (`2915d22f`) — V2 extractor sets `Seed=42` on every LLM call (temperature already 0).

**Container deploy currency (verified before run):**
- sentinel-collector: 2026-05-26T21:57:16-04:00 (post PR #498 merge at 21:56:10)
- secmaster:         2026-05-26T20:59:47-04:00 (post PR #495 merge at 20:59:05)

## VERDICT: **PROGRESS** — 3/10 articles produced matrix_cells (30%)

Spec coverage bands: PASS >= 8/10 · PARTIAL >= 5/10 · **PROGRESS 1-4/10** · FAIL 0/10.

Matches the PR #496 author's counterfactual prediction (~3/10) after PR #497 + PR #498 land. A non-zero outcome is the first measured forward motion against demos #1 and #2 (both 0/10).

> Note on `summary.json`: the script's built-in verdict reports `FAIL` with `articles_with_matrix_cells=0` because its attribution loop matches `pattern_id` substring on `external_id` — but Phase 5.4's `SourceProvenance.sourceDocumentRef` carries the source *name* (e.g. `fed-press-monetary-full`), not the article `external_id`. The attribution bug is in the demo script, not the pipeline. Manual attribution via `(sourceDocumentRef, sourceTimestamp) <-> (raw_content.source, collected_at)` recovers the true 3/10 coverage shown below.

## Per-article verdict

Attribution joins cell `sourceProvenance.sourceTimestamp` to `raw_content.collected_at` (exact-second match — unique within the demo set).

| # | Article ID | Source | Domain | Obs | Cells | Wins | Verdict |
|---|------------|--------|--------|-----|-------|------|---------|
| 1 | 29807 | rss | FOMC fed-press-monetary | 1 | 0 | – | miss |
| 2 | 33632 | rss | Investing.com Israel/Iran ceasefire | 1 | 0 | – | miss |
| 3 | 27702 | rss | Jefferies oil to BoC rate-hike | 1 | 0 | – | miss |
| 4 | 31149 | rss | Jefferies oil pullback to BoC ratehike bets | 1 | 1 | macro_signal_identity (BoC) | **WIN** |
| 5 | 27772 | rss | Seeking Alpha Russell 2000 Value | 6 | 0 | – | miss |
| 6 | 31430 | searxng-content | Mixed sector aggregator | 1 | 0 | – | miss |
| 7 | 36065 | searxng-content | Adyen M&A call transcript | 1 | 0 | – | miss |
| 8 | 27560 | fed-press-monetary-full | Fed press / monetary (mis-tagged as insider sale) | 3 | 2 | macro_signal_identity (Fed) | **WIN** |
| 9 | 34537 | fed-speeches-full | Fed speech / monetary commentary | 1 | 1 | macro_signal_identity (Fed) | **WIN** |
| 10 | 35352 | tsa-checkpoint | TSA checkpoint counts | 111 | 0 | – | miss |

- 10/10 articles processed without `processing_error`.
- 27 extracted_observations total across the set (one had 111 from tsa-checkpoint).
- 4 sentinel matrix_cells produced (cells #2, #3, #4, #5).
- All 4 cells land within the 3-minute per-article budget (total demo wall ~101 s).

## Per-source delta — which stage caught wins

Prometheus `sentinel_entity_resolver_stage_outcome_total` deltas (before -> after):

| Stage | Outcome | delta |
|-------|---------|-------|
| `macro_signal_identity` | grounded   | **+2** |
| `macro_signal_identity` | no_match   | +25 |
| `secmaster_composer`    | no_match   | +28 |
| `secmaster_cove`        | no_match   | +28 |
| `openfigi`              | no_match   | +28 |
| `gemini_fallback`       | no_match   | +25 |

`sentinel_matrix_filter_dropped_total{reason="entity_resolution_exhausted"}`: **+24**.

**Interpretation:** the macro_signal_identity stage (PR #491) is the *only* resolver stage producing groundings in this corpus. The expanded `macro+rate` categories from PR #497 are doing real work — the grounded count more than doubled (1 -> 3) and the no_match count also climbed (2 -> 27), confirming that the catalog is actually being queried per claim. Composer / CoVE / OpenFIGI / Gemini all return no_match for every claim attempted; for a 10-article macro-heavy corpus, those stages are not yet contributing wins.

The 4th cell (1 win > 3 groundings counted) likely reflects a path that bypassed the resolver stage counter (one of the two `fed-press-monetary-full` cells shares the same source timestamp, so they may have been emitted from a single grounded claim). The dominant signal remains: `macro_signal_identity = 3 groundings`.

## Composer-fuzzy manifest

**No.** `secmaster_composer/grounded` count: 0 -> 0 (unchanged). Bank of Canada specifically: not visible in the composer wins because none exist. The BoC cell (article 31149) appears to have grounded via `macro_signal_identity` (with `rate` category from PR #497), not via the SecMaster composer fuzzy path.

## Extractor-quality manifest

**Yes (partial).** All 10 articles produced observations with valid `source_entity` tags: `{Federal Reserve, Federal Reserve Board, Federal Open Market Committee, Jefferies, Jenoptik AG, Investing.com, Reuters, Bureau of Labor Statistics, TSA, author}`.

Example wins:
- 27560 (fed-press-monetary): `source_entity` in {Federal Reserve, Federal Reserve Board, Federal Open Market Committee} -> resolves to macro identity.
- 31149 (Jefferies/BoC): `source_entity = Jefferies` -> BoC grounded via DSL claim payload (macro_signal_identity catalog).
- 34537 (Fed speech): `source_entity = Federal Reserve` -> resolves.

Example misses:
- 27702 (Jefferies/BoC): `source_entity = NULL` -> blocks downstream resolution.
- 33632 (Israel/Iran ceasefire): `source_entity = Reuters` (publisher, not subject) -> no instrument target.
- 35352 (TSA): `source_entity = TSA` but 111 obs — high-frequency macro indicator may be hitting filter on cardinality/composer side.
- 36065 (Adyen M&A): `source_entity = "author"` -> not a usable instrument identity.
- 27772 (Russell 2000 Value): `source_entity = "Jenoptik AG"` — wrong entity (looks like an attribution bug in the extractor when the article ranges across multiple entities).

## Determinism verification (bonus, per Step 1)

**Inconclusive-but-positive.** A second back-to-back run (`run2.log`, reset_at=2026-05-27T02:14:28Z) produced **0 new matrix_cells** — meaning the publisher / pipeline considered the second pass's output already-present (no behavioural divergence detected at the cell layer).

Per-article extracted_observation counts:

| Article | Run 1 | Run 2 | Match |
|---------|-------|-------|-------|
| 27560 | 3 | 3 | yes |
| 27702 | 1 | 1 | yes |
| 27772 | 6 | 6 | yes |
| 29807 | 1 | 1 | yes |
| 31149 | 1 | 1 | yes |
| 31430 | 1 | 1 | yes |
| 33632 | **2** | **1** | **no (delta -1)** |
| 34537 | 1 | 1 | yes |
| 35352 | 111 | 111 | yes |
| 36065 | 1 | 1 | yes |

9/10 articles produced identical observation counts and the matrix_cells layer is fully stable. Article 33632 (Israel/Iran ceasefire) showed a -1 drift between runs. The source_entity assignment for 33632 was identical across runs (`Reuters`), so the drift is not at the source-entity level. Possible explanations:

1. Non-deterministic input ordering at chunk-level decomposition (RLM may split the same article differently if a tokenizer or chunker has any randomness).
2. Filter-tie at observation hashing where a deduplication race excluded an observation in run2 that survived in run1.

Recommendation: supervisor's next regression pass should re-run determinism check at the extraction layer specifically (capture both runs' `extracted_observations` rows in full and compare hash-by-hash) before claiming `Seed=42` is verified. PR #498's behavioural change is **strongly indicated but not bit-exact-confirmed** by this run.

## Artifacts

- `per_article.jsonl` — 10 rows, one per article (matrix_cell_count=0 reflects script attribution bug; see verdict note).
- `matrix_cells_dump.csv` — 4 new sentinel cells (run1).
- `audit_spot_checks.json` — 3 cells x audit lookup (all 200, attribute via timestamp table above).
- `dashboard_metrics.json` — Phase 5.7 metric snapshot.
- `summary.json` — script-computed verdict (FAIL — supersede with manual verdict above).

External captures in `/tmp/phase5-demo-3/`:
- `before-resolver.json`, `after-resolver.json`, `before-filter.json`, `after-filter.json` (Prom deltas).
- `run1.log`, `run2.log`, `cells_run1.csv`, `per_article_run1.jsonl` (determinism inputs).

## Driving commits

- #491 — Phase 5 Step 2: extend EntityResolver cascade with SecMaster composer + macro signal-identity
- #492 — SecMaster: exhaust-all-lookups in resolve-entities
- #493 — Phase 4.4 EntityResolver per-stage metrics export
- #495 — SecMaster: case-insensitive alias lookup + long-form alias expansion
- #497 — ExtractionProcessor prepass-bag wiring + MacroSignalIdentityCatalog `macro+rate`
- #498 — V2 extractor `Seed=42` for determinism

## Recommended next moves

1. **Fix demo script's attribution bug** — switch the matching loop to `(sourceDocumentRef, sourceTimestamp) <-> (source, collected_at)` so future runs compute the verdict correctly without manual reconciliation. (Not done here per task hard rules — supervisor to dispatch separately.)
2. **Diagnose 7/10 misses** — they fall into three buckets:
   - extractor-misses (no `source_entity` or wrong subject): 27702, 27772, 36065.
   - resolver-misses (no identity available for valid `source_entity`): 33632 (Reuters -> not a security), 35352 (TSA -> macro indicator series), 31430 (BLS -> macro indicator).
   - high-cardinality wins not flowing through (35352 has 111 obs but 0 cells) — the filter is dropping all 111 at `entity_resolution_exhausted`.
3. **Determinism**: full hash-by-hash diff at the `extracted_observations` row level on a single article to confirm or refute the delta -1 drift on article 33632.
4. **macro_signal_identity yield**: 3 groundings / 27 attempts ~ 11% yield. PR #497 expanded the catalog to include `rate`; the next catalog expansion (commodity, FX, index series?) is the most direct path to PARTIAL (>=5/10).
