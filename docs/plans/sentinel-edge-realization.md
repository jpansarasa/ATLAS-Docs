# Sentinel Edge Realization — Planning Doc

**Created:** 2026-05-17 (post-session pause; pre-dispatch planning artifact)
**Status:** Discussion artifact, **not** a fix plan. Pulls the whole picture together so we can methodically decide what to dispatch next.
**Context:** Two days of intensive Sentinel work + deep diagnostics have exposed a layered set of architectural and operational gaps. This doc steps back from fix-mode into plan-mode. No work should be dispatched against this doc without an explicit decision in §7 / §8.

## 1. Operating principles

These principles came out of the 2026-05-17 planning conversation. They frame every downstream decision in this doc. Anything that violates one of these is wrong-shape work regardless of how plausible it looks in isolation.

- **Per-stage measures, not end-state metric.** Each pipeline stage has its own success criteria. The chain succeeds only if every link does. We do not collapse "is Sentinel working?" into one number — we ask "is CoD preserving tokens? is extraction recovering them? is resolution grounding them? is statement validation catching drift?" stage-by-stage.
- **Token preservation as CoD success.** CoD output isn't subjective prose quality — it's whether signal-bearing tokens (tickers, company names, sector mentions, numerics with units) from source are preserved. Lose-prose-keep-tokens fine; lose-tokens fail. This is the dimension to benchmark and to instrument.
- **Observability before raw log observation.** Any measurement we can't get from OTEL means OTEL is wrong, not "go grep logs." If a benchmark dimension can't become a runtime metric, it's the wrong dimension. Treat the benchmark as a forcing function for instrumentation — what we measure offline must also be measurable online.
- **Stocks→sector→macro chain.** Tickers/company names are signal-bearing tokens. Their role: feed lookup (SecMaster cache OR upstream authority) → sector/industry attribution → per-sector macro score. The macro score is the actual edge. SecMaster is a CACHE of instruments — the authorities are Finnhub, OpenFIGI, Gemini. Designs that treat SecMaster as the source-of-truth invert the architecture.
- **Right tool per problem.** RegEx is fine as a filter (80% of well-shaped patterns) but not a solution; "if RegEx is the answer you now have two problems." Prompt architecture should match model capabilities (small models need chunking; large/sliding-context models don't). Don't apply one tool uniformly across stages with different characteristics.
- **Drop feature flags entirely.** Each "default-OFF + flip later" hedge became technical debt because the flip surfaced unwired gates. ZFS snapshots + git revert + ansible redeploy = real rollback. Future features ship enabled; bugs become git-revert events. Implication: remove `Extraction__StatementValidationEnabled`, `Extraction__GeminiFallbackEnabled`, `Extraction__GeminiAutoRegisterNewInstruments`, `Extraction__AutoApproveEnabled`, `ReExtract:Enabled`, `ReExtract:Mode` (and any others). Always-on; bugs = git-revert.
- **Infinite-resources prior.** Mercury has 128GB RAM + RTX 5090 + 6×8TB nvme on bench + sata-bulk. Don't pre-optimize for compute/disk. Pull all candidate models, use the largest defensible sample, let the test harness consume what it needs. Dial back only on observed ceilings.

## 2. Goal

> Sentinel is the edge. We are card counters, not by-the-book players. The whole point is to surface signals from unstructured prose that FRED/OFR baselines can't deliver. **Validation IS the edge** — without trustworthy validation, we're just charting FRED.

Every architectural decision in this doc is judged against that frame. The pipeline doesn't have to be cheap or fast; it has to be _correct enough that we trust an AutoApproved row_. Anything that erodes that trust — silent gates, parallel paths drifting apart, lazy-load that doesn't lazy-load — is a direct hit on the edge.

## 3. Architectural intent vs implementation gap

Canonical reference: [`docs/sentinel-extraction-pipeline.md`](../sentinel-extraction-pipeline.md). The Mermaid pipeline below is reproduced from that doc; each stage is annotated with what's actually happening in production today per the morning's OTEL recon.

```mermaid
flowchart TD
    Start([Start: Document Input]) --> EstimateTokens{Estimate Token Size}
    EstimateTokens -->|Less than Context| DirectCoD[Chain of Density]
    EstimateTokens -->|Greater than Context| SplitDoc[Split Document with<br/>Overlapping Windows]
    SplitDoc --> ChunkCoD[CoD Processing<br/>on Each Chunk]
    ChunkCoD --> FinalCoD[Final Overview CoD]
    DirectCoD --> ExtractSymbols[Extract Stock Symbols &<br/>Financial Instruments]
    FinalCoD --> ExtractSymbols
    ExtractSymbols --> RAGLookup[RAG Lookup:<br/>Cosine Similarity]
    RAGLookup --> TopResults[Top 5 Results]
    TopResults --> VerifyStep[Chain of Verification]
    VerifyStep --> CheckHallucinations{Check RAG Symbols<br/>in Original Document}
    CheckHallucinations -->|Not Found| GeminiFallback[Gemini MCP<br/>Google Finance Lookup]
    GeminiFallback --> GeminiResult{Gemini Found?}
    GeminiResult -->|Yes| RegisterInstrument[Register New Instrument<br/>+ Use as Resolution]
    GeminiResult -->|No| FlagHallucination[Flag Hallucination]
    RegisterInstrument --> ValidateStatements[Validate Every Statement<br/>in CoD Summary]
    CheckHallucinations -->|Found| ValidateStatements
    ValidateStatements --> ValidateResult{All Statements<br/>Valid?}
    ValidateResult -->|No| FlagInvalid[Flag Invalid Statement]
    ValidateResult -->|Yes| Success([Pipeline Complete])
    Success -.->|10% sampled,<br/>async, non-blocking| GeminiCrossCheck[Gemini Cross-Check<br/>Ticker ↔ Company]
    GeminiCrossCheck --> CrossCheckMetric[Emit Quality Metric]
    FlagHallucination --> Success
    FlagInvalid --> Success
```

### Stage-by-stage: intent vs reality

| Stage | Intent (per canonical doc) | Actually happening today | Gap |
|---|---|---|---|
| **1. Token estimate** | Route oversize docs to chunk path, in-budget to direct path. | Working as designed. | — |
| **2. Chain of Density** | Dense summary that preserves entities + numbers. Per SENTINEL rule, ≥30B model. | CoD runs on **qwen2.5:7b on `ollama-cpu-gen`**. NFP-contamination prompt bug fixed (PR #342). | **Rule violation.** 7B violates the ≥30B SENTINEL constraint. Quality of the downstream extraction is bottlenecked by CoD summary quality. |
| **3. Final overview CoD** | Coherent merged summary for chunked path. | Working. Same 7B caveat as Stage 2. | Same as Stage 2. |
| **4. Extract symbols + structured fields** | `sentinel-cove-v6.2` base on 32B-AWQ. JSON-schema-enforced decode. `subject_entity` populated. | V2 path active. `subject_entity` 100% populated. LoRA removed (PR #344). | — (structurally healthy on V2; V1 path divergence — see §5 gap 1) |
| **5a. RAG top-N** | Cosine similarity over v5 embeddings (`ticker + name + description + industry + sector`); top-5 default, cap 10. | Working. | — |
| **5b. CoVe Symbol grounding** | Pick first top-N candidate whose Symbol or Name appears literally in `text_quote ∪ context_summary`. Length-aware predicate (≥4-char Symbol → `\bSYM\b`; ≤3-char → Name-only). PR #314, #331 tightening. | Active. ~60% rows → `no_match`, 34% → `rag_synthesis`, 3% real hits. | The high `no_match` rate is _expected_ from the CoVe predicate **if lazy-load is doing its job**. It isn't — see Stage 5c. |
| **5c. Phase 7 Gemini fallback** | On cascade-exhausted, call Gemini MCP Google Finance → if hit, register new instrument in SecMaster (`discovery_source='GeminiFallback'`) and use ticker as resolution. | Wrapper firing **146× / 30min**. Result: **100% `no_match`. Zero auto-registrations.** | **Critical.** The lazy-load chain is broken end-to-end. Either wrapper isn't being invoked from V2, Gemini is returning useless responses, or the register-step is silently failing. PR #328 metrics never registered → can't tell which. |
| **6. Statement-level validation** | Validate every numeric/factual field against source. Failed fields → `[cove] statement-ungrounded:<field>` + force `review_status = Pending`. PR #329 landed. | **Metric never emitted as Prometheus series.** Silent in logs too despite `Sentinel__StatementValidationEnabled=true`. Code path may not be invoked from V2. | **Critical.** Validation IS the edge per §2. If Stage 6 is silently no-op'd, every row that passes Stage 5b is AutoApproved on extraction-confidence alone. |
| **AutoApprove** | Single gate (`AutoApprovePolicy.cs`): `extraction_conf ≥ 0.9 ∧ resolved ∧ resolution_conf ≥ 0.8 ∧ instrument_id != null`. Invoked in **all** persistence paths. | Just wired into `ReExtract` path (PR #343). **V2 production path (`RunV2ProductionAsync`) still doesn't invoke the gate.** | The production hot path bypasses the central policy. This is what §5 gap 1 is about. |
| **Catalog** | Seeded narrow from FRED/Finnhub. Lazy-load + auto-register fills it from real article traffic. See §4. | 8,379 / 9,420 rows (89%) unresolved. Catalog has only seeded sources. Lazy-load chain broken (see Stage 5c). | **Not a structural narrowness problem.** This is the lazy-load chain not lazy-loading. Bulk catalog expansion would be wrong-architecture. |

## 4. SecMaster lazy-load design — the corrected understanding

Reference: [`SecMaster/README.md`](../../SecMaster/README.md), "Catalog Discovery" + "Embedding Backfill" features; and the Phase 7 description in `docs/sentinel-extraction-pipeline.md` §"Stage 5c".

**How the system is supposed to work:**

1. **Catalog seeded narrow.** SecMaster is populated from upstream collectors (FRED, Finnhub, AlphaVantage, OFR, EDGAR) at startup. This covers the universe those collectors care about — US macro series, US equities with material SEC filings, etc. By design it does **not** cover every globally-listed ticker.
2. **Extraction surfaces an entity.** Sentinel processes an article and extracts a candidate Symbol or company name.
3. **RAG cosine lookup against seeded catalog.** Top-N similarity search. If the right answer is already in the catalog, CoVe grounds it and we're done.
4. **Miss → Phase 7 Gemini fallback.** When CoVe rejects all top-N (or top-N is empty), the Phase 7 wrapper calls the Gemini MCP Google Finance integration with the article's company-name context.
5. **Gemini returns ticker → auto-register.** On a verified hit, SecMaster registers the new instrument with `discovery_source = 'GeminiFallback'`. Embedding backfill picks it up. The row uses the ticker as its resolution.
6. **Future references resolve correctly.** Next article mentioning that company finds it in the now-expanded catalog via normal RAG.

**Consequence:** 89% unresolved is _not_ a bug. It's the "we haven't seen these yet" state — exactly the entry condition for lazy-load. As real article traffic flows through and Phase 7 fires, the catalog _earns_ its expansion driven by what the market is actually writing about. We don't seed Wikidata's global universe up front; we let real signal pull instruments into existence.

**The actual bug:** the lazy-load is not lazy-loading. Phase 7 fires 146× in 30 minutes and registers **zero** instruments. The cascade is failing somewhere between "wrapper fires" and "instrument persists in SecMaster." Fixing that closes the loop and the unresolved-rate trends down naturally with traffic.

This is the architectural correction that drove the rest of this doc.

## 5. Five compounding gaps — the integration story

The diagnostic pattern over the last two days has been: fix one thing, discover that the next gate downstream is silently no-op on the production path, fix that, discover the next one. This isn't five independent bugs. It's one architectural shape — **features built incrementally, each landed in V1 path or standalone code, never refactored into common middleware. V2 was built in parallel and didn't pick up V1's accumulated gates.** The five gaps that fall out of that shape:

### Gap 1 — Parallel-path inconsistency

V1, V2 `RunV2ProductionAsync`, and `ReExtract` paths each invoke different subsets of the gates. AutoApprovePolicy, Phase 6 statement validation, Phase 7 Gemini fallback are each wired in some subset of paths. Last night's flip to V2 silenced any gate that lived only in V1.

**Why it compounds:** every new gate added in the future re-runs this risk. Each PR has to remember "wire it into all three paths"; the diff doesn't enforce that. The fix is structural: a common middleware that all paths invoke.

### Gap 2 — Lazy-load not lazy-loading

Phase 7 wrapper firing 146× / 30min with 100% `no_match` and 0 auto-registrations. Hypothesis space (need evidence to narrow):

- **(a)** Wrapper not actually invoked from V2 — the "146 calls" metric might be coming from a different code path (e.g. the old `sentinel_gemini_resolver_calls_total` series, not the PR #328 wrapper).
- **(b)** Gemini API returning no useful ticker — prompt issue, MCP timeout, or genuinely no result for the article's context.
- **(c)** Auto-register step failing silently — gRPC call to SecMaster swallowed, DbContext duplicate-key, etc.
- **(d)** Firing metric is the OLD `sentinel_gemini_resolver_calls_total`, NOT the new PR #328 wrapper. In which case "0 registrations" means PR #328 isn't actually running at all.

**Why it compounds:** until (a)-(d) are disambiguated by metrics, we're guessing. See Gap 3.

### Gap 3 — Observability gaps on the very things we built

PR #328 (Phase 7 fallback), PR #329 (Phase 6 statement validation), and PR #332 metrics were added in code but **never registered as Prometheus series** in production. We can't measure whether the features fire. This is the meta-failure that makes Gap 2 hard to diagnose — we shipped instrumentation that doesn't instrument.

**Why it compounds:** every fix attempt for Gap 2 needs Gap 3 fixed first, or we're flying blind. And every _future_ gate addition risks the same silent-metric pattern unless registration is enforced.

### Gap 4 — Validation layers built but not all firing end-to-end

Phase 6 statement validation: feature-flag-gated, flag set to `true`, but silent in metrics **and** logs. Code path may not be invoked from V2. The qualitative-extraction path has it wired (`QualitativeVerificationService.cs:299-403`); the structured-extraction path was the explicit Phase 6 target and the wiring may not have crossed paths after the V2 flip.

**Why it compounds:** Phase 6 _is_ the validation half of "validation IS the edge." Without it, AutoApprove approves on extraction-confidence + grounded-Symbol alone — the numeric `value`, `period_end_date`, `metric_label` are taken on faith. That defeats the point.

### Gap 5 — Quality thresholds set by guess

`extraction_confidence ≥ 0.9` AND `resolution_confidence ≥ 0.8`. These numbers came from intuition, not calibration. Morning recon: 4 of 7 `cove_VectorSearch` hits hovered at 0.791-0.795 — _just under_ the gate. We have no ground-truth labels, so we don't know whether 0.8 is "right" or whether we're rejecting good rows / approving bad ones.

**Why it compounds:** every other fix in §6 changes the distribution of confidence scores at the gate. Thresholds tuned today against the broken-pipeline distribution will be wrong tomorrow when Phase 6 + Phase 7 are firing correctly. This is the **last** thing to tune, not the first.

## 6. Corrected priority order

Re-sequenced 2026-05-17 evening. Old order put Phase 7 lazy-load first; that put the cart before the horse — lazy-load enrichment is downstream of CoD, and CoD itself is unvalidated. Catalog/RAG/Phase 7 work is now explicitly deferred until upstream stages (CoD, then extraction) are trusted on per-stage measures.

| # | Workstream | Why this order |
|---|---|---|
| 1 | **CoD validation benchmark** (token-preservation measures, all candidate models, observability-first) | Upstream-of-everything. Garbage CoD → wasted downstream work. |
| 2 | **CoD instrumentation** (whatever the benchmark surfaces becomes production metrics) | Closes "observability before log-observation" gap for the most critical stage. |
| 3 | **Extraction validation** (same discipline, after CoD trusted) | Next stage downstream; only meaningful once CoD output is reliable. |
| 4 | **Extraction instrumentation** | Same pattern. |
| 5 | **THEN — only then — RAG/SecMaster/Phase 7 lazy-load + catalog work** | All downstream of trustworthy upstream. Catalog enrichment now would be optimizing the wrong stage. |
| 6 | **Feature-flag removal sweep** (separate workstream, can run anytime) | Cleanup, low risk. |
| 7 | **V2 path unification + missing-metrics fixes** | Hygiene; can land alongside #5. |
| 8 | **Threshold calibration** | After above stable. |
| 9 | **Container tagging discipline** (requires release-notes process) | Operational improvement after Sentinel works. |

## 7. Open questions — for discussion before dispatch

Resolved this session:

- **Q3 (target volume "edge realized")** — reframed: success measured PER-STAGE, not single end-state metric. Per-stage measures TBD per-stage; CoD = token preservation rate.
- **Q4 (threshold calibration method)** — defer until catalog/RAG/Phase 7 stable. Premature to calibrate before fixing what feeds the thresholds.
- **Runtime question (Ollama vs vLLM-CPU vs llama.cpp-server)** — defer until AFTER CoD model benchmark (B then A order per user). If runtime needs swap, top 2-3 models re-validated on new runtime.

Still open:

- **Per-stage measures for stages beyond CoD** (extraction, resolution, statement-validation) — discuss as each stage's validation comes up.
- **V2 path unification scope** — common middleware refactor vs continue patching one-at-a-time.

## 8. Decision log

| Date | Decision | Rationale | Status |
|---|---|---|---|
| 2026-05-17 | Remove LoRA from extraction pipeline | F4.6.3 diagnostic + morning recon showed LoRA degrades extraction vs `sentinel-cove-v6.2` base | Live (PR #344) |
| 2026-05-17 | Flip Phase 6 + Phase 7 + AutoApprove ON | User: "no risk in PoC, validation IS the edge" | Live (PR #341), but V2-path wiring incomplete — see §5 |
| 2026-05-17 | Defer bulk catalog expansion (OpenFIGI + Wikidata) | User correction citing `SecMaster/README.md` lazy-load + discover-on-demand design intent. The 89% unresolved rate is the entry condition for lazy-load, not a structural narrowness problem. | Captured in this plan §4 |
| 2026-05-17 | Reject bulk catalog seeding | SecMaster lazy-load is by design; bulk seed = wrong-architecture | Captured in §4 |
| 2026-05-17 | CoD benchmark on Ollama (B), runtime investigation deferred (A) | Forward momentum on model quality; runtime swap can re-validate top 2-3 | Captured in §6 |
| 2026-05-17 | Approved 9-model roster (4 to pull, 5 loaded) | Refreshed against current SOTA, balanced size tiers, dropped reasoning/MoE variants unfit for CoD | Captured in §6 |
| 2026-05-17 | Disk cleanup: 8 superseded models deleted | Reduce clutter; recon's disk constraint was wrong, no actual blocker | Done |
| 2026-05-17 | Drop feature-flag pattern entirely | Each flag was tech debt; rollback via git+ansible/ZFS is sufficient | Operating principle |
| 2026-05-17 | Token preservation (regex+LLM-validate) as CoD success measure | Subjective dimensions rejected; signal-bearing tokens are what downstream needs | Operating principle |
| 2026-05-17 | Infinite-resources prior for design | Stop pre-optimizing for compute/disk; mercury has the headroom | Operating principle |
| 2026-05-17 | Container tagging deferred | Needs release-notes discipline; not just a tag scheme tweak | Captured in §6 |
