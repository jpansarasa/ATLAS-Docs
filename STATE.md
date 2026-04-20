# Sentinel Extraction Quality — Remediation Plan

## GOAL

Close the six gaps surfaced by reviewing all 14 pending items in the Sentinel review queue on 2026-04-20. Raise auto-approve-ready observation rate; make human review faster.

**Accept criteria**
- `unit=count` no longer assigned to USD amounts.
- Analyst/forecast language no longer tagged `certainty=definite`.
- `text_quote` populated on ≥95% of new extractions.
- investing.com market-overview sidebar widgets no longer extracted as observations.
- SecMaster resolver hits on common tickers (LGO, JKS, NRGV, MRK, FFIV) that fail today.
- New items land in the queue with the above fixes in place, validated against a fresh sample of 14+ items.

## ARCH

Extraction pipeline: `RawContent → ContentNormalizer (trafilatura/markitdown) → ChainOfVerification (initial_extraction + verification via VllmClient structured output) → ExtractionResult → ExtractedObservation row → SecMaster resolution`.

Observations route to review queue when `confidence < 1.0` and `ReviewStatus=Pending`. Current pending-to-approved ratio: 14 / 12363 ≈ 0.1%, which is fine — the pending items are the *interesting* ones where quality gaps show.

**Prompts live at** `SentinelCollector/src/prompts/` and are host-mounted into the container per `deployment/artifacts/compose.yaml.j2` (edit host file, restart service, no rebuild).

## STATUS

- ✓ 1. Fix unit mislabeling (USD → `count`)
- ✓ 2. Fix certainty=definite on forecasts
- ✓ 3. Investigate + fix text_quote=null
- ✓ 4. Strip investing.com market-overview sidebar widgets pre-extraction
- ✓ 5. Deduplicate / investigate contradictory forecasts from same article
- ✓ 6. SecMaster resolver debug (NRGV, MRK, FFIV should resolve trivially)

## TASKS

### 1. Units: USD amounts tagged `count` (items 8, 10, 11)
**Symptom**: `value=-16950000, unit=count` for "Softbank sells $16.95M of NRGV"; same for F5 $294k, Strive $459k.
**Investigate**:
- Read `prompts/initial_extraction.txt` lines 25–28 — it already shows `"$73.8 billion" → billion_USD`. So prompt documents the convention; LLM adherence is the issue.
- Sample failing extractions from raw LLM output (Loki trace for one of these observation ids) to confirm it's LLM not-following vs parser bug.
**Fix (if prompt adherence)**:
- Add a "NEVER" rule block: `✗ USD_value → unit:count   # use USD or billion_USD`.
- Add 3 more ✓/✗ example pairs targeting insider-trade dollars and price-target dollars specifically.
**Fix (if parser bug)**: update parser; unlikely but verify.

### 2. Certainty on forecasts (items 1–5)
**Symptom**: "analysts predict 51% revenue growth" → `certainty=definite`.
**Investigate**: the prompt already shows `[{value: 150000, certainty: "definite"}, {value: 175000, certainty: "expected"}]` in a revision context. Check that the forecast/prediction signal words are listed under "expected".
**Fix**: add an explicit rule in `prompts/initial_extraction.txt` — "analyst predicts/forecast/expects/projects/estimates" → `certainty: expected`. Worked example.

### 3. text_quote=null across all 14 items
**Investigate first** (EVIDENCE_GATE — theory-first gets us the wrong fix):
- Query Loki for a raw LLM response for one of the 14 obs ids. Confirm whether `text_quote` is present in the model output.
- Read `ExtractionResultParser.TryParse` — does it handle missing/null quote field silently?
- Read VllmClient structured-output schema builder — is `text_quote` in the schema `required` list?
**Fix depends on where it's dropping**:
- LLM not emitting → schema `required: ["text_quote", ...]` fix.
- Parser dropping → parser fix.
- Prompt pass-through reminder (already says "verbatim sentence").

### 4. investing.com ticker-widget noise (items 6–7, 12–14)
**Symptom**: Extractions of Shanghai/CSI/Hang Seng/Nifty/Straits index moves from article pages that are *not about* those indices. LLM is picking them up from sidebar price tickers.
**Investigate**:
- Pull the raw HTML for one of the source URLs (already under `/opt/ai-inference/raw-data/sentinel/…`).
- Check what trafilatura actually returns for these pages — is it stripping sidebars, or passing them through?
- Confirm trafilatura's output contains the ticker snippet (root cause) vs. the LLM pulling from noise text (LLM hygiene).
**Fix path A** (if trafilatura output is clean): add to prompt an explicit exclusion rule — do not extract sidebar quotes/tickers that are unrelated to the article body.
**Fix path B** (if trafilatura leaks the sidebar): domain-specific pre-wash rule — strip known sidebar selectors for investing.com before trafilatura, or post-filter.
Path B is more robust; path A is cheaper but more fragile.

### 5. Duplicate/contradictory forecasts from one article (items 3–5)
**Symptom**: Two different "JinkoSolar module shipment forecast" numbers (85 GW and 75 GW) from one article; same pattern for Largo items 1–2.
**Investigate**:
- Pull the source article. Are both numbers actually in the text (one for 2026, one for 2025)?
- If yes, extractions are correct but lack period distinction — fix is `period` population.
- If no, LLM hallucination — fix via verification pass in ChainOfVerification.
**Fix depends on finding**: likely period-hygiene rule, not hallucination.

### 6. SecMaster resolver failure (all 14)
**Symptom**: `instrumentId=null` on every item despite clean tickers (MRK, JKS, NRGV, FFIV, LGO, FFIV).
**Investigate**:
- Call SecMaster `/resolve` directly for "MRK", "Merck", "NYSE:MRK" — which form fails?
- Check resolver input — is the LLM-extracted `source_entity` blank or malformed, so SecMaster has nothing to match on?
- Check ticker-extraction path in ChainOfVerification.
**Fix depends on finding**. Separate from LLM quality; schedule after items 1–5.

## CONTEXT

- Branch: `fix/sentinel-extraction-quality`
- Prompts host-mounted — edit + `nerdctl restart sentinel-collector` (no rebuild).
- Review queue endpoint: `GET http://localhost:5091/admin/review/?limit=20` (JSON only).
- Memory note: project prefers evidence-first per CLAUDE.md EVIDENCE_GATE — every task above starts with "Investigate" for that reason.
- Order of work: 1, 2 first (prompt-only, trivial). 3 before 5 (both need LLM output inspection). 4 before broad re-validation. 6 last (separate subsystem).
- Re-validation gate: after 1–5 ship, sample the review queue over a 24h window and confirm the failure classes listed above are absent.

## ATTEMPTED

- PR #183 (2026-04-20): shipped fixes for tasks 1–6 in STATE above. Ready to merge/deploy.

---

# Architecture — Async Resolution Pipeline (follow-on, post-#183)

## GOAL

Pull SecMaster resolution OFF the synchronous extraction hot path. Resolution becomes eventually-consistent via a rate-limit-aware worker that drains a `resolution_pending` queue against upstream providers (Finnhub primary, AlphaVantage nightly sweep).

**Accept criteria**

- `ExtractionProcessor.ProcessBatchAsync` never calls a SecMaster upstream provider directly — only local SecMaster (ExactSQL + FuzzySQL + Vector+RAG hypothesis).
- Extraction pipeline completes even if Finnhub and AlphaVantage are both unreachable.
- Observations persist with one of three resolution states: `resolved`, `pending`, `no_resolution`.
- Resolution worker respects Finnhub's 60/min quota (token bucket, ≤50/min sustained).
- AlphaVantage's 25/day budget is spent on a nightly sweep of `no_resolution` rows ranked by mention frequency × confidence.
- New metric `sentinel_resolution_latency_seconds` (histogram) tracks time from observation insert → `resolved`.
- Existing behavior for already-resolved local hits unchanged (no latency regression).

## ARCH

Today (post-#183):
`ExtractionProcessor → IDescriptionResolver (wraps ISecMasterClient.ResolveAsync) → SecMaster HybridResolutionService (ExactSQL | FuzzySQL | Vector+RAG | UpstreamDiscovery)` — all synchronous, all inline with extraction.

Target:
- `ExtractionProcessor` calls a new `ILocalResolver` that returns `{resolution | rag_candidate | null}` using ONLY local steps (no upstream calls, no network I/O beyond the local embedding model).
- A new `ResolutionWorker` (BackgroundService) drains rows where `resolution_state = pending` against Finnhub via `FinnhubClient` with a token-bucket rate limiter. On resolve: update row + `_catalog.AddAsync` + `_embedding.EmbedInstrumentAsync`.
- A new scheduled job `AlphaVantageSweepJob` runs nightly, picks top-N `no_resolution` rows by mention frequency × extracted confidence, spends AV's daily budget on the ranked list.

Upstream providers:
- Finnhub: hot enrichment path (60/min quota, ~1/sec sustained). Already wired in `FinnhubCollector`; reuse its client with rate-limit wrapper.
- AlphaVantage: cold enrichment path (25/day). Reuse `AlphaVantageCollector` client; extend for single-symbol lookups if needed.
- Explicit non-goal: no external provider is reached from the synchronous path.

## STATUS

- ✓ 1. Schema: add `resolution_state` + `resolution_hypothesis` to `extracted_observations`
- ✓ 2. Split `HybridResolutionService` into `LocalHybridResolutionService` (no upstream) + `UpstreamResolutionService` (Finnhub/AV)
- ✓ 3. `ExtractionProcessor` calls local-only path; marks `resolution_state = pending` when RAG produces a hypothesis
- ✓ 4. `ResolutionWorker` BackgroundService draining pending queue through Finnhub with token-bucket rate limiter
- ✓ 5. `AlphaVantageSweepJob` scheduled daily against `no_resolution` backlog
- ✓ 6. Observability: `sentinel_resolution_latency_seconds`, `sentinel_resolution_queue_depth`, method-tagged resolution counter
- ✓ 7. Alert rule: `resolution_queue_depth` above sustained threshold → worker stuck / provider down
- ✓ 8. Review-UI cosmetic: show `pending` / `no_resolution` states distinctly from `resolved`

## TASKS

### 1. Schema migration
- New column `extracted_observations.resolution_state` (enum: `resolved | pending | no_resolution`) default `pending`.
- New column `extracted_observations.resolution_hypothesis` (nullable string — the RAG-proposed ticker awaiting verification).
- EF Core migration via `dotnet ef migrations add AddResolutionState` per CLAUDE.md DATABASE rules.
- Backfill: existing rows where `instrument_id IS NOT NULL` → `resolved`; else → `no_resolution` (not `pending`; we don't want the worker re-enqueueing 10k historical misses on first boot).

### 2. Service split
- Extract interface `ILocalResolver` returning `LocalResolutionResult(Resolution?, Candidate?, VectorMatches, RagResponse)`.
- `HybridResolutionService` implements local-only cascade (Exact → Fuzzy → Vector → RAG). Remove the STEP 6 upstream-discovery branch from this path.
- New `UpstreamResolutionService` that takes a ticker string and queries Finnhub then (optionally) AV. Owns rate-limit state.

### 3. Hot-path rewrite
- `ExtractionProcessor.ProcessBatchAsync` uses `ILocalResolver`:
  - Resolution non-null → mark `resolved`, populate `instrument_id`.
  - Candidate non-null, resolution null → mark `pending`, store `resolution_hypothesis`.
  - Both null → mark `no_resolution`, no instrument_id.
- Remove existing `DescriptionResolver` (its parenthetical-ticker logic moves into the hypothesis layer or stays as a pre-RAG normalization step — TBD; investigate before deciding).
- Delete the `EnableDiscovery=true` flag from SentinelCollector's Finnhub calls — there's no upstream discovery on the hot path anymore.

### 4. ResolutionWorker
- BackgroundService that polls `extracted_observations WHERE resolution_state = 'pending' ORDER BY extracted_at ASC LIMIT N`.
- Token-bucket rate limiter pre-configured to 50/min Finnhub (headroom for other callers of the same key).
- For each row: try Finnhub with `resolution_hypothesis` first, then raw `description` as fallback.
- On success: update row + catalog insert + embed.
- On Finnhub not-found: mark `no_resolution` (AV sweep picks it up).
- On Finnhub error (429/5xx): leave row `pending`, backoff, retry next tick. Structured log + span error per CLAUDE.md.

### 5. AlphaVantageSweepJob
- Quartz job, fires daily at 04:00 UTC (low-activity window).
- Pulls top 25 rows where `resolution_state = 'no_resolution'`, ranked by `COUNT(*) OVER description × avg(confidence)`.
- Spends AV's 25/day budget one lookup per top row.
- Same success/failure handling as the worker.

### 6. Observability
- `sentinel_resolution_latency_seconds` histogram, tagged `{method: local|finnhub|alphavantage}`.
- `sentinel_resolution_queue_depth` gauge (runs every 30s, counts pending rows).
- Resolution counter gains a `method` dimension consistent with the worker stages.
- Grafana panel: stacked area of method breakdown over 24h — self-surfacing health signal.

### 7. Alerting
- `sentinel_resolution_queue_depth > 500 for 15m` → warning (worker falling behind).
- `rate(sentinel_resolution_errors{provider=\"finnhub\"}[5m]) > 0.1` → warning.
- Both route to alert-service + autofix per existing pipeline.

### 8. UI state
- Review queue endpoint returns `resolution_state` alongside existing fields. Cosmetic; non-blocking for the rest of the work.

## CONTEXT

- Depends on PR #183 merged and deployed (cascade fix is the foundation; re-architecting a broken cascade would be silly).
- Touches three services: `SentinelCollector`, `SecMaster`, and the resolution worker (probably lives in `SentinelCollector` since it's the owner of `extracted_observations`).
- Potential collaborator bottleneck: `FinnhubCollector`'s existing rate-limit budget. Need to confirm a 50/min reservation doesn't starve its primary collection workload. Investigate before integration.
- Deferred until after this: AV batch lookup optimization — AV supports batch `SYMBOL_SEARCH`, ~5× quota efficiency. Nice-to-have, not required.
- Non-goals for this scope: multi-upstream conflict resolution (e.g. Finnhub says X, AV says Y), upstream caching layer, rate-limit sharing across services.

## ATTEMPTED

- PR #184 (2026-04-20): shipped all 8 tasks for the async resolution pipeline — schema migration (`AddResolutionState`), local/upstream resolver split, hot-path rewrite in `ExtractionProcessor`, `ResolutionWorker` BackgroundService, nightly `AlphaVantageSweepJob`, metrics + queue-depth gauge, alert rules, and review-UI cosmetic state surfacing.
- Follow-up commit (2026-04-20): review-driven narrow fixes for the Resolved/InstrumentId-null state-machine bug in ResolutionWorker + AlphaVantageSweepJob, AdminEndpoints backfill missing SetResolutionState, migration backfill race safety (BEGIN/COMMIT), plus happy-path parse tests and options-validator tests.
