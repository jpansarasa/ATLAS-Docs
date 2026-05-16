# Symbol-Identification Remediation Plan

**Created:** 2026-05-16 ~00:30Z (overnight execution authorized by user)
**Memory ref:** `~/.claude/projects/-home-james-ATLAS/memory/project_rag_symbol_identification.md`

## Context

Today (2026-05-15) we discovered that ~30,000 of 53,140 Approved rows in `sentinel.extracted_observations` had hallucinated Symbols (Symbol does not appear anywhere in source article). Concrete: row 103535 stamped Symbol=`USAF` on a Terawulf $900M equity raise article. Cause: cosine-similarity nearest-neighbor over a field-poor RAG returns "wrong-but-close" candidates when the right answer isn't in the catalog.

99% of damage was LoRA-era; reverting to base (`sentinel-cove-v6.2`, PR #266) reduced the rate but base still hallucinates ~50% on field-poor RAG. The architectural intent (CoD + CoVe + RAG defense-in-depth) was incomplete on the structured-extraction path.

Already done today:
- PR #309: AutoApprove env disabled (no new poisonous Approved rows)
- PR #310: 30,111 hallucinated Approved rows quarantined (`QuarantinedAt` stamped, not deleted)

## Architectural target (the durable fix)

Three layers must be in place for safe Symbol attribution:

1. **Rich RAG embeddings** — every instrument embedded as `ticker + name + description + industry + sector` (currently ~ticker+name only). With rich descriptions, an article about "Terawulf $900M crypto-miner equity raise" semantically matches Terawulf's record (which carries name + Bitcoin-mining industry + Technology sector), not just token-similar tickers.
2. **Top-N retrieval** — RAG returns top N candidates (default 5), not just single nearest neighbor. Gives the verification layer options.
3. **CoVe-style disambiguation in source text** — for the top-N candidates, pick the one that literally appears in the article text (`text_quote` ∪ `raw_content.context_summary`). If none appear → return NoMatch (Symbol=NULL).

Don't re-engage AutoApprove until all three are in place.

## Phases (overnight execution)

### Phase 1 — Rich RAG embeddings (~2-3h)
- 1a. **Backfill missing instrument fields:** `description`, `industry`, `sector` for instruments lacking them. Currently ~25% atlas_sector / 25% naics. Most equities are field-poor. Source from EDGAR (already wired), OpenFIGI metadata where available, fallback null.
- 1b. **Update `InstrumentEmbeddingTemplate` to v5** to include all 5 fields (ticker + name + description + industry + sector) in the prose template.
- 1c. **Re-embed all active instruments at v5** via `EmbeddingBackgroundService`. Existing v4 rows will be detected as stale by template-version mismatch and re-embedded.
- 1d. **Verify v5 coverage = 100%** of active instruments. Confirm SecMaster RAG queries hit v5.

**Dispatch:** 1 agent, end-to-end (PR + review + merge + deploy + monitor backfill + verify).

### Phase 2 — Top-N retrieval API (~1-2h)
- 2a. Update `SemanticSearchEndpoints.HybridResolveLocal` and `RagService.QueryAsync` to support `topN` parameter (default 5, cap at 10).
- 2b. Response shape: include `candidates` array (each with Symbol, instrument_id, similarity score, name).
- 2c. Add unit tests for top-N behavior + similarity-score ordering.
- 2d. Deploy. Backwards-compatible with single-candidate callers.

**Dispatch:** 1 agent, end-to-end after Phase 1 lands.

### Phase 3 — CoVe-style Symbol verification in structured-extraction path (~2-3h)
- 3a. In `ExtractionProcessor`, after extraction model emits `Symbol`, call SecMaster RAG with `topN=5` (using article context, not just the model's Symbol guess).
- 3b. For each top-N candidate, check whether its Symbol or company name literally appears in `raw_content.context_summary` (also try `text_quote` for narrower scope).
- 3c. Pick first candidate appearing in text. If none of the 5 appear → set Symbol=NULL, leave instrument_id NULL, mark `review_notes` with "no-symbol-grounded-in-source" so review UI can filter.
- 3d. Add unit tests covering: candidate-in-text picked, no-candidate-in-text → null, candidate ordering by similarity.
- 3e. Deploy.

**Dispatch:** 1 agent, end-to-end after Phase 2 lands.

## Phases requiring user direction (NOT autonomous)

### Phase 4 — Re-process limbo (deferred)
- 41K quarantined rows: optionally re-extract with new pipeline (recover what's recoverable)
- 36K Pending rows: optionally re-process to clear backlog
- Risk surface: large UPDATE volumes, semantic drift between old-and-new extractions
- **Hold for user awake**

### Phase 5 — Re-engage AutoApprove (deferred)
- Run quality spot-check: post-fix Symbol attribution rate
- If clean: flip `Extraction__AutoApproveEnabled=true`
- Monitor 24h
- **Hold for user awake**

## Communication plan

- **Per phase:** ntfy low-priority notification when each phase lands successfully
- **Blocker:** ntfy default-priority if any phase BLOCKED (test failure, deploy failure, ambiguous design choice)
- **Critical:** ntfy high-priority if any phase reveals a deeper architectural issue (e.g. EDGAR doesn't have descriptions for foreign tickers — need to source elsewhere)
- **Not paged:** routine progress, branch creation, dispatch handoffs (user can read STATE.md / plan in morning)

## Operating constraints (per CLAUDE.md + supervisor-mode skill)

- All implementation/test/build/deploy work goes to dispatched subagents — supervisor never edits code directly
- Each phase agent end-to-end: branch + edit + test + PR + review + merge + deploy + verify
- Selective `git add -- <paths>` always; PR flow only; no direct push to main
- AutoApprove stays OFF throughout (don't re-engage until Phase 3 lands and quality is verified)
- Quarantined rows stay quarantined (don't unquarantine)
- ZFS snapshots taken automatically by ansible deploy

## Status tracking

This file is the canonical plan. Per-phase outcomes recorded inline below as each lands.

### Phase 1 status
- Dispatched: 2026-05-16 ~00:30Z
- PR: #311 (merged 2026-05-16T04:14:02Z, SHA `748f329b`)
- Deploy: ansible `--tags secmaster` succeeded 2026-05-16T04:14:50Z
- Description backfill (EDGAR sicDescription): ran on startup, 5036 instruments now carry meaningful business descriptions (AMD="Semiconductors & Related Devices", TSLA="Motor Vehicles & Passenger Car Bodies", AAPL="Electronic Computers", APP="Services-Computer Programming, Data Processing, Etc.", etc.). Coverage of equities with curated descriptions exceeds Phase 1 50% goal.
- Re-embed: in flight via EmbeddingBackgroundService — template-version mismatch detected v4→v5, ~30 rows/min via ollama-cpu-embed bge-m3, expected ~5h to complete all 9051 active instruments
- Backfill-unresolved-rate stays ~89% (8073/9053) — expected, classification coverage is unchanged in this phase; only the description column was the focus
- Status: code LIVE; awaiting full re-embed

### Phase 2 status
- Dispatched: 2026-05-16 ~04:25Z
- PR: #312 (merged 2026-05-16T04:32:00Z, SHA `4b212336`)
- Deploy: ansible `--tags secmaster` succeeded 2026-05-16T04:32:50Z; container re-up at 04:32:48Z
- Continued in-flight branch (`feat/rag-top-n-candidates`) from prior agent: kept `LocalResolutionOptions.TopN` + endpoint `topN` plumbing, finished response-shape contract (`LocalResolutionCandidate` record, `BuildCandidates` projection), updated existing tests to new positional signature, added 9 new top-N tests + 1 over-fetch test in `HybridResolutionService`. 830/830 unit tests pass.
- Smoke probe `GET /api/semantic/resolve-local?q=microsoft&topN=5&enableRag=true&limit=5` → returns `candidates` array w/ MSFT (FuzzySql fast-exit, single-element list per design); backward-compat probe (no `topN`) returns identical shape. Vector-path probe `q=cloud computing software` returns 5 candidates ordered by `similarityScore` desc.
- Backward-compat verified: `SentinelCollector.LocalResolutionDto` ignores the new `candidates` field via System.Text.Json default behaviour.
- Status: code LIVE; ready for Phase 3 to consume top-N for CoVe disambiguation

### Phase 3 status
- Dispatched: 2026-05-16 ~00:35Z
- PR: #313 (merged 2026-05-16T04:49:55Z, SHA `1907783f`)
- Deploy: ansible `--tags sentinel-collector` succeeded 2026-05-16T04:51:13Z; container re-up 04:51:18Z
- Implementation: `CoveSymbolVerifier` (pure-function helper) + wiring in
  `ExtractionProcessor.ApplyLocalResolutionAsync`. After RAG returns the
  description path's DTO (with Phase 2's `Candidates` list, default top-5),
  the verifier iterates candidates in similarity-descending order and picks
  the first whose `Symbol` or `Name` literally appears (case-insensitive)
  in `text_quote` ∪ `raw_content.context_summary`. On rejection:
  `Symbol`/`InstrumentId` set to null, `ResolutionState=NoResolution`,
  `review_notes="[cove] no-symbol-grounded-in-source"`. Rationale for
  NoResolution (not Pending): re-running the same RAG would return the same
  un-grounded candidates — better to expose for human review than to loop.
- `ISecMasterClient.ResolveLocalFromQuoteAsync` gained an optional `topN`
  parameter (default null → SecMaster server default 5, cap 10). Backward-
  compat: `LocalResolutionDto.Candidates` defaults to null so older callers
  and test fixtures compile unchanged.
- Tests: +20 new (12 verifier unit tests covering null/empty/whitespace
  inputs, Symbol-in-quote, Name-in-quote, context_summary-only, case-
  insensitive, top-1-skip = WULF→USAF scenario, no-match, list-order,
  blank-field defense, public marker constant; 9 wiring tests covering
  grounded Symbol set, ungrounded null behaviour, marker append, second-
  candidate pick, name-only match, NoResolution state, context_summary
  path, defensive missing-Candidates DTO). 3 existing tests updated to
  provide `Candidates` for the now-required grounding check. 954/954
  pass, 0 errors, 0 net new warnings.
- Status: code LIVE; smoke verification in progress
- Note: scope intentionally limited to v1 path (`ApplyLocalResolutionAsync`)
  — v2 pipeline (`DeterministicResolver.TryHybridResolveAsync`) is shadow-
  only for `rss` and lower-impact; revisit when v2 is opted-in for more
  sources.

### Phase 3 follow-up — CoVe predicate tightening
- PR: #314 (merged 2026-05-16T10:48:43Z, SHA `aa604a8d`)
- Tightens the CoVe Symbol predicate to word-boundary match for ≥4-char
  Symbols, Name-only (substring, ≥4 chars) for ≤3-char Symbols (Symbols
  like "F" or "GE" are noise-prone as substrings). Persists
  `[cove] no-symbol-grounded-in-source` to `review_notes` so the review UI
  can filter the un-grounded cohort.

### Phase 6 — Statement-level CoVe validation on structured-extraction path (deferred)

**Intent:** Symbol grounding (Phase 3) verifies the *who* — the Symbol attribution. Phase 6 verifies the *what* — every individual numeric/factual statement in the extracted observation must be grounded in the source document. The model's CoD summary often contains numbers (EPS, revenue, dates, ratios) that the model invented or transcribed from a different paragraph. Today there is no validator for those on the structured path.

**Where the gap is today:**
- `QualitativeVerificationService.cs:299-403` already implements statement-level validation for the qualitative-extraction (`macro_observations`) flow.
- `ExtractionProcessor` (structured-extraction → `extracted_observations`) has no equivalent step. Symbol grounding is the only verification it runs.

**Architectural target:**
- After CoVe Symbol grounding passes for a row, run a statement-level pass: for each emitted numeric/factual field (e.g. `value`, `period_end_date`, `metric_label`), check whether the value appears verbatim (or within a small tolerance for numeric fields) in `text_quote ∪ raw_content.context_summary`.
- If any field fails grounding, mark the row with a `[cove] statement-ungrounded:<field>` marker in `review_notes` and set `review_status` to `Pending` (do not auto-approve).
- Implementation can extend `CoveSymbolVerifier` into a `CoveExtractionVerifier` or sit as a sibling service called from `ExtractionProcessor.ApplyLocalResolutionAsync` after the Symbol-grounding branch.

**Acceptance:**
- 100% of fields in any Approved row are literally present (or numeric-within-tolerance) in `text_quote ∪ context_summary`.
- New unit tests covering: numeric grounded, numeric near-match within tolerance, numeric not in source, date grounded, date wrong format but same value, label grounded, label paraphrased (reject).

**Hold for user direction** — same status as Phases 4 and 5.
