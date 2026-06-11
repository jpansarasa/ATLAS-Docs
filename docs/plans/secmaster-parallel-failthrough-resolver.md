# SecMaster parallel fail-through resolver — design mapping + reconciliation

**Branch:** `feat/secmaster/parallel-failthrough-resolver`
**Date:** 2026-06-05
**Status:** in-progress (see STATUS at bottom)

## Documented design (the target)

- `docs/f4.6.4-prepass-rollout.md` — F4.6.4 entity-resolution pre-pass ops reference.
  References phase-1 spec `docs/plans/atlas-matrix/f4.6.4-openfigi-naics-rag-phase1.md` (not on disk;
  recoverable via tag/git-log). Line 68: "Phase 2 handles [foreign issuers] via the existing
  `GeminiResolverClient`."
- Break-map STATUS line (retired doc, recoverable via git log): "the **sector** dimension is blocked one
  layer up at SecMaster instrument-NAICS coverage (~10%)". Architecture context: `docs/atlas-matrix-handoff-v2.md`.

The DOCUMENTED design is a **fail-through cascade** that exhausts local catalog → instruments →
edgar → OpenFIGI → signal-identity → upstream-discovery (parallel fan-out) → identifier-confirmation
(OpenFIGI→Finnhub→**Gemini**) → self-seed, surfacing canonical identity + NAICS + ATLAS sector back
to the caller. Most of this EXISTS in `SecMaster/src/Services/EntityResolutionService.cs`.

## User spec mapped onto the impl

| User-spec element | Impl status | Action |
|---|---|---|
| Parallel fan-out ACROSS sources per entity | per-CANDIDATE parallel; per-source SEQUENTIAL cascade. Upstream-discovery (Step 6) already fans out Finnhub/AV/FRED inside CatalogService | KEEP cascade order (it encodes precision ranking + cost); the existing discovery fan-out IS the parallel multi-source leg. No rewrite to a flat fan-out — that would discard the precision/cost ordering the cascade deliberately encodes and is not a MATERIAL doc conflict. |
| Gemini LAST RESORT (costs money, rarely fails) | EXISTS as the 3rd confirmation source inside discovery (`IdentifierConfirmationService`) | Gemini already runs only after OpenFIGI+Finnhub miss, AND only after the whole local+signal cascade misses → it IS effectively last-resort. Make the "all-missed → review queue" routing explicit (Layer C). |
| Review queue (near-empty, alertable) | DOES NOT EXIST | NET-NEW (Layer B). |
| Surface resolved classification back to grounding (grounded>0 for cache-HIT + NEW-symbol MISS) | **BROKEN** — proven root cause below | FIX (Layer A). |
| Lazy-load / self-seed | EXISTS (`PersistConfirmedInstrumentAsync`) | keep. |
| Per-source telemetry, Gemini count, review-queue depth, grounded-vs-fallback | per-source + Gemini + self-seed EXIST; grounded-vs-fallback + review-queue depth DO NOT | ADD (Layers A + B). |

## Proven root cause (the WHY) — VERIFIED 2026-06-05 against live `atlas_secmaster`

`resolve-entities` drops the persisted classification for a classified instrument that has **no
active source-mapping**.

Live probe (`atlas_secmaster.instruments` JOIN `source_mappings`):

| symbol | naics_code | atlas_sector_code | active_mappings |
|---|---|---|---|
| AAPL | 334111 | INFOTECH | 1 |
| JPM | 522110 | FINANCIALS | 1 |
| MSFT | 511210 | INFOTECH | 1 |
| **WM** | **562212** | **INDUSTRIALS** | **0** |

Mechanism:
1. `ResolutionService.ResolveAsync` populates `NaicsCode`/`AtlasSectorCode` on BOTH branches
   (`no_sources` and source-mapped `BuildResolution`) — but only the source-mapped branch sets
   `ResolvedSource`.
2. `HybridResolutionService.ResolveLocalAsync` STEP 1 gates the exact-SQL hit on
   `sqlResult?.ResolvedSource != null` (HybridResolutionService.cs:63). For WM (0 active mappings)
   `ResolvedSource` is null → the exact match is **rejected** and the classification never surfaces
   via the cheapest, highest-confidence path.
3. AAPL/JPM/MSFT have a mapping → pass. WM and every classified-but-uncollected blue-chip silently
   fail the exact path. This is the "naicsCode=null even for classified blue-chips" symptom.

It slipped because there was no grounded-vs-fallback telemetry at the resolution boundary.

## Build layers (commit per layer)

- **A. Root-cause fix + grounded telemetry.** Make the exact/fuzzy SQL path accept a
  classification-bearing resolution even without an active source-mapping (the instrument IS
  identified + classified — source-mappings are about *collection*, orthogonal to *identity*). Add a
  grounded-vs-fallback counter at the EntityResolutionService boundary
  (`secmaster_entity_resolution_grounding_total{result=grounded|identified_no_sector|fallback}`).
- **B. Review queue.** EF entity `EntityResolutionReviewItem` + repository + enqueue on a true
  exhausted miss (after Gemini). Telemetry: enqueue counter + observable-gauge depth (alertable,
  expected ~0). MCP/REST read surface for operator drain.
- **C. Gemini last-resort routing.** Ensure a fully-exhausted candidate (local + signal + discovery
  + confirmation incl. Gemini all miss) routes to the review queue rather than a silent null;
  trace span status=Error on the true miss.

## STATUS
- ✓ A: root-cause fix (exact-match surfaces classification for uncollected
  instruments) + grounded-vs-fallback telemetry. Verified against live DB +
  unit-tested.
- ✓ B: review queue — entity + migration + repository + telemetry (enqueue
  counter + depth gauge + sampler) + operator REST surface. Unit + integration
  tested against real PostgreSQL.
- ✓ C: Gemini-last-resort routing — terminal enqueue after the full
  confirmation cascade (incl. Gemini) misses; span status=Error on a true miss.

### Deliberately NOT done (no material doc conflict; would discard precision/cost ordering)
- Rewriting the per-candidate SEQUENTIAL cascade into a flat per-source parallel
  fan-out. The cascade order encodes a precision + cost ranking
  (local-exact 0.95 → … → Gemini 0.85), and the upstream-discovery step already
  fans out Finnhub/AlphaVantage/FRED in parallel inside CatalogService. A flat
  fan-out would lose that ordering and the short-circuit cost savings for no
  grounding benefit. Candidates already resolve in parallel
  (MaxConcurrentCandidates). If a future requirement insists on per-source
  parallelism within a candidate, it is an isolated refactor of ResolveOneAsync.

### Verification
- compile.sh (build + unit): 0 errors, 0 code warnings, 962 unit pass.
- integration: 129 pass (incl. 5 review-queue repo tests, one driving a
  concurrent same-surface enqueue) against real PG;
  AddEntityResolutionReviewQueue migration applies cleanly.
- Pre-existing NU1903 NuGet advisories on the test project's transitive
  System.Security.Cryptography.Xml 9.0.0 are unrelated to this change.
