# Sentinel v2 Architecture — Review & Merge Log

> Phase 1.2 cross-check: Opus 4.6 reviewed the Opus 4.7 proposal at
> `/home/james/ATLAS/docs/sentinel-v2-architecture.md`.
> This file records Opus 4.6's notes and the supervisor's merge decisions.

## Dispatch summary

| Role | Model | Run label | Input tokens | Output tokens | Cost | Duration |
|---|---|---|---:|---:|---:|---:|
| Proposal | `claude-opus-4-7` | `v2-arch-proposal-A` | 3,508 | 7,856 | $0.6418 | 112.3 s |
| Review | `claude-opus-4-6` | `v2-arch-review-B` | 7,099 | 2,383 | $0.2852 | 55.6 s |
| **Total** | | | **10,607** | **10,239** | **$0.9270** | |

Verdict from review: `REQUEST_CHANGES` — driven almost entirely by the CertaintyWeight enum-name break. The proposal was otherwise comprehensive.

## Coverage audit (as reported by reviewer)

All 10 required topics were COVERED or PARTIAL. The only PARTIAL was the gate formula (certainty enum mismatch + missing `truflation-api` trust default). No topic was MISSING.

## Reviewer's top-5 edits — disposition

| # | Edit | Disposition | Where applied |
|---|---|---|---|
| 1 | Fix `CertaintyWeight` enum names (`Hypothetical`→`speculative`, `Denied`→`conditional`, lowercase). | **APPLIED** | Multi-signal gate formula, CertaintyWeight table + Approved predicate. |
| 2 | Add `truflation-api` to `SourceTrust` defaults (0.95). | **APPLIED** | SourceTrust defaults table. |
| 3 | Specify `CandidateRef` contract. | **APPLIED** | New table under Schema additions; fields Index/InstrumentId/Symbol/Name/AssetClass/Aliases, zero-based indexing noted. |
| 4 | Define how `QuoteFidelity` receives normalized text. | **APPLIED** | Added `NormalizedTextRef` field to `ExtractionResult`; MultiSignalGate section now documents the contract plus the missing-ref failure mode (ERROR log + 0.0). |
| 5 | Normalize certainty casing in Approved predicate. | **APPLIED** | Tier definition now reads `certainty IN {definite, expected}`; prose clarifies case-insensitive comparison and unknown-value → 0.0. |

All five top-priority edits applied.

## Reviewer's breaking-change risks — disposition

| # | Risk | Disposition |
|---|---|---|
| 1 | CertaintyWeight enum runtime break | **APPLIED** (see Top-5 #1). |
| 2 | Approval predicate casing | **APPLIED** (see Top-5 #5). |
| 3 | `Confidence` back-projection misleads v1 consumers (using `ExtractionConfidence`). | **APPLIED** — back-projection changed to `Confidence = QualityScore`, with an explicit note that dashboard owners are notified. |
| 4 | Shadow-table schema drift. | **APPLIED** — migration path now mandates creation in the same EF migration via `LIKE ... INCLUDING ALL` with a rule that future column additions modify both tables atomically. |
| 5 | CorroborationWorker might promote shadow-table rows if `ShadowMode` flips mid-window. | **APPLIED** — CorroborationWorker section now states the worker only scans prod; shadow rows are never migrated to prod; cutover triggers a fresh v2 re-extraction against prod. |

## Reviewer's under-specified contracts — disposition

| # | Item | Disposition |
|---|---|---|
| 1 | Shape of `CandidateRef`. | **APPLIED** (Top-5 #3). |
| 2 | `QuoteFidelity` input path. | **APPLIED** (Top-5 #4). |
| 3 | `CandidateSymbolsJson` contents. | **APPLIED** — schema table clarifies it stores the exact shortlist snapshot passed into the prompt (not the raw SecMaster response), enabling prompt replay. |
| 4 | `CrossSourceCount` on re-extraction. | **DEFERRED** — acknowledged in Risks/Phase 2 items (#8 Re-extraction row management). Requires a superseding policy that is Phase 2, not a doc edit. |
| 5 | Resolution-confidence 0.7 magic number. | **APPLIED** — promoted to `Extraction__ResolverCandidatePickThreshold` with default 0.7; added to feature-flag table and NEW-config row. |
| 6 | `truflation-api` missing from SourceTrust. | **APPLIED** (Top-5 #2). |
| 7 | Partial index DDL. | **ADJUSTED** — concrete DDL (`CREATE INDEX CONCURRENTLY ... WHERE cross_source_count IS NULL`) added to Risks §Phase 1.3 #1 with expected cardinality. |

## Reviewer's optional improvements — disposition

| # | Suggestion | Disposition |
|---|---|---|
| 1 | State-transition diagram | **REJECTED for this doc** — the three sections already cover transitions; adding a diagram pads word count without new information. Can be added as a follow-up if engineers flag confusion. |
| 2 | Tighten `application/json` route semantics | **PARTIAL** — captured as Phase 2 item #9 in Risks; doc flags that undeclared payloads fall back to LLM. Full per-feed mapping is a Phase 2 deliverable. |
| 3 | Row-volume estimates for index sizing | **PARTIAL** — approximate cardinality noted on the partial-index risk item; per-source daily volumes are tracked in collector telemetry and out of scope for an architecture doc. |
| 4 | Observability (structured log fields, metrics, alerts) | **REJECTED for this doc** — observability is governed by project-level CLAUDE.md rules and is a Phase 2 implementation concern, not an architectural decision. Implementers will follow those rules. |
| 5 | Add `resolution_method` explicitly to schema table | **APPLIED** — added to `ExtractedObservation` table; noted that v2 extends the existing enum with `llm_candidate_pick` and `hybrid_subject`. |

## Reviewer's terminology-drift items — disposition

All three drift items (`Hypothetical`, `Denied`, title-case certainty) **APPLIED** via Top-5 #1 and #5. Canonical casing is now lowercase throughout the gate section, matching the v7 prompt contract (plan §1.1).

## Residual concerns the doc explicitly hands to other phases

These are called out inline in the final doc's "Risks and open items" section:

- **Phase 1.3**: partial-index DDL + composite-index design + shadow-table creation + back-projection communication to dashboards + explicit "no future NOT NULL" policy.
- **Phase 2**: candidate-cache invalidation, `SubjectEntity` normalization, re-extraction row dedup, `application/json` per-feed parser mapping, deterministic-path `QuoteFidelity=1.0` short-circuit.

None of these are in-scope for the architecture doc itself.

## Rollup

- Applied: 14
- Adjusted: 2 (partial-index DDL became concrete; `application/json` behavior bounded)
- Rejected: 2 (state-transition diagram, observability section — both better handled elsewhere)
- Deferred: 1 (re-extraction policy — Phase 2 ticket, already called out)

## Schema-migration input for Phase 1.3

The architecture doc's Risks §Phase 1.3 items 1–5 constitute the Phase 1.3 backlog. Explicit deliverables the Phase 1.3 ticket must ship:

1. EF migration `AddV2ObservationFields` with all 8 nullable columns (`SubjectEntity`, `CandidateSymbolsJson`, `SelectedCandidateIndex`, `ExtractionConfidence`, `ResolutionConfidence`, `CrossSourceCount`, `QualityScore`, and a `ResolutionMethod` enum extension row).
2. `CREATE INDEX CONCURRENTLY ix_eo_corrob_pending ON sentinel.extracted_observations (extracted_at) WHERE cross_source_count IS NULL`.
3. `CREATE INDEX ix_eo_subject_period ON sentinel.extracted_observations (subject_entity, period_start)` — composite for corroboration matching; `value` filtered in-process.
4. `sentinel.extracted_observations_shadow` created in the same migration via `LIKE ... INCLUDING ALL`.
5. Communication in the migration PR description that `Confidence` back-projection semantics change (`= QualityScore` in v2 rows).
