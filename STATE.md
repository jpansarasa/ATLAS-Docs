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

- ◯ 1. Fix unit mislabeling (USD → `count`)
- ◯ 2. Fix certainty=definite on forecasts
- ◯ 3. Investigate + fix text_quote=null
- ◯ 4. Strip investing.com market-overview sidebar widgets pre-extraction
- ◯ 5. Deduplicate / investigate contradictory forecasts from same article
- ◯ 6. SecMaster resolver debug (NRGV, MRK, FFIV should resolve trivially)

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

*(empty — plan phase)*
