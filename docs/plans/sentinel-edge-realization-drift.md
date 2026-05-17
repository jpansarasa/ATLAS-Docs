# Sentinel Edge Realization — Drift Tracking

**Purpose:** Track IDEAL design vs ACTUAL implementation at each pipeline stage. Capture the HISTORY of how reality diverged from intent so we never lose the chain of reasoning that led to current state.

## How to use this doc

- Each pipeline stage gets four subsections: IDEAL, ACTUAL, DRIFT, HISTORY.
- Updated when: any change to stage behavior, any decision that re-shapes a stage, any newly-discovered gap.
- Companion to [`sentinel-edge-realization.md`](sentinel-edge-realization.md) (which is the active plan); this is the longitudinal architectural memory.
- Cross-reference for stage definitions: [`../sentinel-extraction-pipeline.md`](../sentinel-extraction-pipeline.md).

## Stage 0: Sources

### IDEAL

- 11+ configured sources producing diverse `raw_content` (RSS, Fed press, searxng, challenger-rss).
- Steady throughput, source-level health visibility.

### ACTUAL (as of 2026-05-17)

- 11 V2EnabledSources active (rss, fed-speeches, fed-speeches-full, fed-press-all, fed-press-all-full, fed-press-monetary, fed-press-monetary-full, fed-press-bcreg, fed-press-bcreg-full, searxng-content, challenger-rss).
- ~37 raw_content/6h overnight; ~hundreds/hr during day.
- Per-source quality unknown — no per-source preservation/extraction metrics.

### DRIFT

- No per-source observability of CoD/extraction quality. Can't tell which sources produce useful signal.

### HISTORY

- (Capture decisions over time.)

## Stage 1: CoD (context_summary generation)

### IDEAL

- Faithfully preserves all signal-bearing tokens (tickers, company names, sector mentions, numerics with units) from source.
- Output structure adapted to model capability (small models: chunking + refinement; large models: single-pass).
- Per-source/per-model preservation metric visible in OTEL.
- Output format is structured symbolic DSL (v1) rather than English prose
  - Forces disambiguation; preserves signal-bearing tokens by construction
  - Each fact has source_span for downstream verification
  - Parseable by a deterministic 200-line parser
  - Versioned (DSL: v1 header); schema changes are migrations

### ACTUAL (as of 2026-05-17)

- `qwen2.5:7b-instruct` on `ollama-cpu-gen` (CPU).
- Violates CLAUDE.md SENTINEL ≥30B rule.
- NFP-template contamination just fixed (PR #342) — prompt exemplars rewritten as abstract placeholders.
- Token-preservation rate UNMEASURED (benchmark in flight).
- No OTEL metric for preservation quality.

### DRIFT

- Model size violates project standard. Reason: assumed 7B could handle CoD; never validated.
- Architectural complexity (token estimation + chunking + CoD-on-CoD) added for small-model constraints. May be unnecessary on larger models.
- No measurement infrastructure. Quality is inferred from downstream artifacts, not directly observed.
- Output format is English prose, not structured DSL — wrong target for an LLM consumer
  - Reason: original design assumed human readability; downstream consumer is actually another LLM
  - Implication: subsequent extraction step has to re-interpret prose, burning a model call to extract what CoD could have structured implicitly
  - DSL reframe captured in main planning doc §"CoD as Symbolic DSL (v1 design)"

### HISTORY

- 2026-05-17: NFP-template contamination discovered (literal numerics bleeding into unrelated articles). Fixed PR #342.
- 2026-05-17: Diagnosed that 7B model + literal-numeric exemplars = template echoing. Reframed: CoD model size matters.
- 2026-05-17: Decision to benchmark 9 candidate models on CoD with token-preservation as success measure.
- 2026-05-17 (evening): User reframed CoD target from English prose summarization to structured symbolic DSL. Rationale: English ambiguity, LLM consumer, small-model strength at structured output, verifiability. v1 grammar drafted; Round 2 benchmark deferred until grammar converged + parser ready.

## Stage 2: Structured Extraction

### IDEAL

- Per-extraction: `subject_entity` populated, `candidate_symbols` proposed, `value` + `unit` + `period` captured correctly in source context.
- Extraction confidence reflects actual reliability.
- Per-source/per-model extraction quality visible in OTEL.

### ACTUAL (as of 2026-05-17)

- V2 path active (`UseV2Pipeline=true`). V2Model=`sentinel-cove-v6.2` (base `qwen2.5-32B-AWQ` on vLLM). LoRA removed (PR #344).
- `subject_entity` 100% populated on V2 path (verified post-LoRA-removal).
- Symbol resolution rate ~7% (1.9-7% across recent hours).
- `extraction_confidence` ≥0.9 hit rate ~45% (152/335 in 30min sample).

### DRIFT

- V2 production path doesn't invoke `AutoApprovePolicy` (PR #343 fixed for ReExtract; V2 production path STILL missing — out-of-scope finding).
- "Value lifted out of context" pattern observed (row 110630 — $40M lifted from "$20M-$40M range"). May be model-pathology or prompt-design issue.
- No OTEL metric for per-stage extraction quality.

### HISTORY

- 2026-05-16: V1 path was macro-only (no `subject_entity` in schema); pre-PR-#266 LoRA actively degraded extraction.
- 2026-05-17: V2 flipped on; LoRA removed; `subject_entity` now populated.
- 2026-05-17: AutoApprove fixed on ReExtract path; V2 production path still missing the gate invocation.

## Stage 3: Resolution (RAG + CoVe Symbol grounding)

### IDEAL

- `subject_entity` → SecMaster cache lookup → on miss → Finnhub/OpenFIGI/Gemini fallback → cache result.
- CoVe grounds candidates in source text.
- Per-stage resolution metrics visible (cache hit rate, upstream call rate, CoVe rejection rate).

### ACTUAL (as of 2026-05-17)

- 60% `no_match`, 34% `rag_synthesis`, 3% real catalog hits, 0.8% exact/fuzzy SQL.
- Phase 7 Gemini fallback firing 146/30min via older `sentinel_gemini_resolver_calls_total`, all 100% `no_match` outcome.
- PR #328 wrapper metric `sentinel_gemini_fallback_calls_total` missing — observability gap.
- 0 new instruments auto-registered via Gemini fallback (lazy-load chain broken somewhere).
- `CatalogEnrichmentBackgroundService` failing 100% with duplicate-key transient error (sibling-DbContext pollution).

### DRIFT

- Lazy-load chain doesn't lazy-load. Either wrapper not invoked from V2, OR Gemini API genuinely returning `no_match`, OR auto-register step failing silently.
- 89% of catalog instruments unresolved. Was misdiagnosed as "structural narrowness" — actually expected lazy-load state.

### HISTORY

- 2026-05-17: Recon misdiagnosed catalog as structurally narrow; user corrected: lazy-load by design.
- 2026-05-17: Decision to defer all catalog/RAG/SecMaster work until CoD validation lands.

## Stage 4: Statement Validation (Phase 6)

### IDEAL

- Every emitted numeric/factual statement verified present in source.
- Markers persisted to `review_notes` on rejection.
- Per-field validation rate visible in OTEL.

### ACTUAL (as of 2026-05-17)

- Code shipped PR #329.
- `Extraction__StatementValidationEnabled=true` in container env.
- BUT: PR #329 metric `sentinel_cove_statement_outcome_total` doesn't exist in Prometheus.
- AND: 0 `[cove-statement]` markers in `review_notes` (last 30 min, 30 days).
- Conclusion: feature is silent. Either code path not invoked despite flag, OR metric never registered.

### DRIFT

- Flag is `true` but feature doesn't observably fire. Same pattern as AutoApprove silent failure pre-PR-#343.

### HISTORY

- 2026-05-17: User flipped flag with confidence; OTEL recon revealed silence.
- 2026-05-17: Deferred investigation until after CoD validation.

## Stage 5: AutoApprove

### IDEAL

- Rows meeting confidence + instrument-id gate auto-promote to `Approved`.
- Decision visible in audit trail.
- Gate invoked from EVERY extraction path (V1, V2, ReExtract).

### ACTUAL (as of 2026-05-17)

- `AutoApprovePolicy.Evaluate` now called from ReExtract resolve-only + ReExtract full re-extract (PR #343).
- V2 production path (`RunV2ProductionAsync`) STILL does not invoke `AutoApprovePolicy`.
- 31 approvals in last hour observed (3 in prior hour, 2 in hour before that) — only via ReExtract path.
- All approved rows have `subject_entity` + `instrument_id` populated (validation chain intact).

### DRIFT

- Three paths (V1, V2, ReExtract); only one of three (ReExtract) currently invokes the gate.

### HISTORY

- 2026-05-17: Discovered AutoApprove was silent (0/4940 in 6h pre-fix). Root caused to ReExtract path overwriting resolution without re-invoking gate. PR #343 fixed.
- 2026-05-17: PR #343 author also flagged V2 production path missing the same gate invocation. Deferred per priority order.
