# News → matrix signal feed (Fix #1) — design

**Date:** 2026-06-05
**Fixes break-map #1/#3** (`docs/sentinel-matrix-break-map-2026-06-05.md`): news stopped feeding `macro_observations` because the macro router gates on an exact-match of the catalog against the (now fragmented) CoD description, and the qualitative path hardcodes `signal_identity_id = null` (which the projector skips).

## Goal

Restore Sentinel's news signal into the matrix: every news article is classified into the matrix's macro signal(s) + ATLAS sector + a signed tilt, written to `macro_observations` (signal-keyed, sector-tagged, with a real signed magnitude), **continuously** — so the projector can accumulate it as the sustained, fast-decaying leading counterweight to lagging hard data.

## Cadence (load-bearing)

Per-article, in the extraction/semantic tier — **not** at digest time. The value is *sustained* flow; a single article barely moves a cell (large alpha decay). So all ~250 articles/day must be classified as they arrive and accumulated/decayed in the matrix. The digest is a downstream *view* that reads the result.

## Decided approach

LLM classifies from article text (vLLM — the GPU arm the SectorTagger/ClaimVerifier already use, which has capacity; CPU llama-server is saturated by extraction). Per article:

```
NewsSignalClassification {
  IReadOnlyList<NewsSignal> Signals   // 0..N — an article can bear on multiple signals
}
NewsSignal {
  string SignalIdentityId   // chosen from the matrix signal-identity catalog (fixed/known set)
  AtlasSectorCode? Sector   // the ATLAS macro sector
  double Tilt               // signed, -1..+1 (the news direction: e.g. "inflation firming" = +)
  double Confidence         // 0..1
}
```

The prompt is given the **catalog of known signal identities** (so it picks from the matrix's fixed set, not free-form) + the ATLAS sector list + the article excerpt, and emits structured output (vLLM OpenAI structured response, the proven path). An article that bears on no tracked signal → empty (correctly contributes nothing).

## Routing → macro_observations

Per `(article × signal)`, write a `macro_observations` row:
- `source_collector = "sentinel"`, `signal_identity_id = <classified>`, `atlas_sector_code = <classified sector>`
- `value_numeric = Tilt × Confidence` (signed magnitude — the news's directional contribution; NOT the 1.0/0.0 polarity stub)
- `observation_time` = article publication/extraction time; idempotency `(source_collector, source_id, observation_time)` with `source_id = {raw_content_id}:sig:{signal_identity_id}`.

This is a new "news-signal" macro path: numeric value-bearing + signal-keyed, but the signal comes from the **LLM classification**, not the exact-match catalog lookup. Reuse `IMacroObservationWriter`; add `MacroObservationRouter.TryPlanNewsSignalWrite(...)` (or a dedicated planner) — distinct from the dead exact-match numeric path and the null-signal qualitative path.

## Components

- **`INewsSignalClassifier`** (new, vLLM): article excerpt + catalog → `NewsSignalClassification`. Dedicated `VllmClient` (mirror SectorTagger DI). New prompt `prompts/news_signal_classify.md`.
- **Signal-identity catalog provider**: enumerate the known signal identities (from `MacroSignalIdentityCatalog` / SecMaster `list_signal_identities`) to feed the prompt + validate the LLM's choices (reject hallucinated signal ids).
- **Routing**: `MacroObservationRouter.TryPlanNewsSignalWrite` + the worker that runs the classifier per article in the semantic tier and flushes the rows.
- **Wiring**: invoke the classifier where the SectorTagger runs (the per-article semantic enrichment, `ExtractionProcessor`/`SemanticVerifier` tier), so it's continuous.

## Pairs with #6 (not in this spec, but required for the matrix to be right)

The feed gets signal-keyed news into `macro_observations`. For the matrix to reflect *sustained* flow, the projector (#6) must accumulate with per-article 24h-half-life decay + daily bucketing (the deleted `SignalMagnitudeCalculator` semantics), not the current 120-day arithmetic mean + last-write-wins. **The feed is the prerequisite; #6 makes the accumulation correct.** Build feed → verify rows land → then #6.

## Verification (real-data, mandatory)

After deploy: confirm `macro_observations` `source_collector='sentinel'` rows resume with non-null `signal_identity_id` + `atlas_sector_code` + signed `value_numeric`; confirm the projector picks them up (cells gain `signal_identity_id` for news signals). Spot-check classification quality on real articles (does "Fed holds, signals patience" map to the right rate signal with the right tilt?).

## Open / risks

- Classifier latency × ~250 articles/day on vLLM (shared with ClaimVerifier/SectorTagger) — measure; batch if needed.
- Signal-identity catalog coverage: if the matrix tracks few signals, much news maps to nothing (correctly) — surfaces a catalog-coverage gap.
- Magnitude calibration (Tilt × Confidence) is the input to #6's decay model — keep the scale sane (−1..+1) so the projector math is interpretable.
- Sector classification here also fixes break-map #2 (news sector tags) for the macro feed, independent of the broken `atlas_sector_code` resolution path.
