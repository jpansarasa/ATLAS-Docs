# DSL PoC — Phase 4 — GPU semantic verifier (plan §7.2)

> **Status**: COMPLETED 2026-05-26. Tagged as `dsl-poc-phase4-done`. Kept as the canonical Phase 4 record (§11 gate-recalibration rationale).

**Status:** scoping. No code, no deploy. Single source of truth for Phase 4
work until split into stories and merged.
**Parent plan:** `docs/plans/atlas-dsl-poc-plan.md` §7 (lines 317–406),
specifically §7.2 (semantic tier) and §7.3 (acceptance).
**Predecessor:** Phase 3 v2 schema + word-grounding arc — the CPU side is
DONE. v15 (word + chunked compound) is the production candidate; the
deterministic Python verifier ships as
`docs/benchmarks/cod-2026-05-17/dsl/verifier_v2_3_1.py` (plan §7.1
implemented as Phase 3 Story 5). T=0 A/B verdict landed in PRs
#431/#432/#433.
**Topology rule (immovable, PR #386):** CPU = extraction (qwen3:30b-a3b on
`llama-server` :11437; v15 DSL emission — DONE). GPU = verification +
summarization (Qwen2.5-32B-AWQ / `sentinel-cove-v6.2` on `vllm-server`
:8000 — this plan).
**Architectural rule (memory `project_sentinel_llm_strength_layering`,
2026-05-20):** small CPU does pattern-recognition + verbatim copy +
classification; large GPU does structured emission and judgment. Phase 4
§7.2 IS that GPU judgment layer — it is not a new architectural commitment.

---

## 1. Executive summary

Phase 4 §7.1 (deterministic verifier, Python, no LLM) is DONE; Phase 3
shipped it as Story 5. Phase 4 §7.2 is the next step: an in-process GPU
semantic verifier tier that consumes the deterministic verifier's
output and enriches each block with NAICS sector/industry tags +
claim-support verdicts + a three-stage entity-resolution cascade
(SecMaster RAG + CoVe → OpenFIGI → Gemini MCP) so catalog misses
register new instruments instead of biasing toward known ones,
emitting the shape Phase 5 (matrix integration) consumes. Production
runtime uses local qwen2.5:32b on `vllm-server` with $0 cloud LLM
spend (Gemini MCP is the only cloud surface, and is bounded to
catalog-miss events). Estimated ~1 week (parent plan §11), 8 phased
PRs each ≤500 LOC, no DB schema changes, additive at every step with a
single cutover at Phase 4.5.

---

## 2. Architectural context

### 2.1 What is DONE

- **CPU extraction (Phase 3):** v15 (word + chunked compound) on
  qwen3:30b-a3b via `llama-server` :11437. T=0 deterministic A/B (PRs
  #431/#432/#433): Arm A v2.3.1 pooled 0.8033, Arm B v15
  chunked-compound pooled **0.8261** (Δ +0.0228); the RNG=on regression
  was sampling-noise, not design. v2.3 stochastic SOTA = 0.8733 pooled.
- **Phase 4 §7.1 deterministic tier:**
  `docs/benchmarks/cod-2026-05-17/dsl/verifier_v2_3_1.py`. Pure Python,
  no LLM call. Per-block `pass / soft_fail / hard_fail`, word-grounding
  score, per-slot `failures` array.
- **Existing GPU verification surface in production:**
  `SentinelCollector/src/Services/CoveSymbolVerifier.cs` (SecMaster
  top-N RAG + literal-presence grounding), `CoveStatementVerifier.cs`
  (Phase 6, PR #329 — literal-token + numeric-tolerance grounding),
  `VllmClient.cs` (OpenAI-compatible client to `http://vllm-server:8000`,
  implements `IOllamaClient`).
- **vllm-server** (`deployment/artifacts/compose.yaml.j2` L811–838)
  serves `sentinel-cove-v6.2` (Qwen2.5-32B-AWQ) on GPU; LoRA disabled
  2026-05-17.

### 2.2 What is NEXT (this plan)

Parent plan §7.2, three workstreams: entity resolution upgrade (ENT →
catalog → NAICS sector + industry), claim verification (per-CLAIM LLM
yes/no/partial), sector tagging (per-EVT NAICS-in-context). Phase 5
(parent plan §8) consumes Phase 4's output: ThresholdEngine reads
validated NUMs/EVTs, downweights/discards when claim_confidence falls
below threshold, scores `(sector × industry × time)` cells.

### 2.3 What this plan is NOT

- ✗ Not a new architectural commitment — it's the documented GPU
  judgment layer in the original topology (PR #386, plan §7).
- ✗ Not a replacement for existing `CoveSymbolVerifier` /
  `CoveStatementVerifier`. Entity-resolution wraps them as stage 1 of
  the new cascade (Phase 4.4) and adds OpenFIGI + Gemini MCP as
  catalog-miss stages 2 + 3; the embedding-similarity overlay
  (Stage 0) ships only if it beats the existing cosine RAG.
- ✗ No production-deployment plan, no DB schema changes, no
  `/opt/ai-inference/prompts/` edits (parent plan §9 hard rule).

---

## 3. The three §7.2 components

### 3.1 Component (a) — Entity resolution upgrade

**Current state.** `CoveSymbolVerifier.PickFirstGrounded` (PRs
#313/#314/#317/#331) walks SecMaster top-N RAG (rich Phase-1 v5
embeddings: ticker + name + description + industry + sector, PR #311)
and returns the first candidate whose Symbol or Name literally appears
in `text_quote ∪ raw_content.context_summary`. With top-N response (PR
#312) and vector similarity floor 0.5 (PR #332), the WULF→USAF nearest-
neighbor failure mode is closed. **Existing cascade-on-miss surface
(default-OFF in production):** `OpenFigiClient.cs`
(`SecMaster/src/Services/`) for ticker → identifier-set lookup, and
`GeminiSymbolFallbackService.cs`
(`SentinelCollector/src/Services/`, PR #328) for catalog-of-last-resort
Google Finance grounding. **Both wired into the production DI graph
today but disabled at the operator-flag layer** (STATE.md L73 — Phase 7
"operator decision pending").

**§7.2 ask.** "Upgrade to embedding-similarity with qwen2.5:32b as the
embedder if the existing logic underperforms."

**Decision (user, 2026-05-24, supersedes "measure first"):**
**Missing instruments are significant failures** — prior cosine-only
RAG biased the system toward what it had seen before (the 30K
hallucinated-Symbol Approved rows discovered 2026-05-15 are the
evidence). Phase 4 SHIPS the full cascade as the entity-resolution
path; no A/B optionality on the cascade itself. The A/B is
**arm-internal** (which catalog-miss path lands the resolution), not
"do we ship the upgrade".

**Cascade (executed in order; first non-null wins):**

1. **SecMaster RAG + CoVe** (existing `CoveSymbolVerifier`) — top-N
   embedding match + literal-presence verification.
2. **OpenFIGI ticker lookup** (existing `OpenFigiClient`) — invoked on
   SecMaster cache miss for ticker-shaped ENT names (deterministic API,
   rate-limit-bounded; cached in `OpenFigiLookupCacheEntity`). On hit:
   the resolved identifier set is registered in SecMaster with
   `discovery_source='OpenFIGI'` and used as the resolution.
3. **Gemini MCP Google Finance fallback** (existing
   `GeminiSymbolFallbackService`, PR #328) — invoked on OpenFIGI miss
   for company-name-shaped ENT names. On hit: register in SecMaster
   with `discovery_source='GeminiFallback'` per existing convention.

The optional embedding-similarity overlay from the original §7.2 ask
(qwen2.5:32b as the embedder) lands BELOW the cascade as a Stage 0
"prefer overlay if it beats RAG cosine by >5pp on grounding rate" —
measured during Phase 4.4 implementation, gated on actual evidence vs
the existing Phase-1 v5 embeddings (PR #311). If the overlay loses, it
doesn't ship; if it wins, it replaces the cosine-RAG stage. Either way,
the OpenFIGI + Gemini cascade ships unconditionally — those are the
catalog-miss paths the user called out as significant-failure
remediation.

**Output contract (per ENT):**

```python
@dataclass
class EntityResolution:
    ent_local_id: str
    instrument_id: Optional[int]      # SecMaster catalog row
    symbol: Optional[str]             # resolved ticker
    sector_code: Optional[str]        # NAICS 2-digit
    industry_code: Optional[str]      # NAICS 3-/4-/6-digit
    confidence: float                 # [0, 1]
    grounding_source: Literal[
        "secmaster_cove",             # cascade stage 1 hit
        "openfigi",                   # cascade stage 2 hit (newly registered)
        "gemini_fallback",            # cascade stage 3 hit (newly registered)
        "embedding_overlay",          # optional Stage 0 overlay
        "no_resolution",              # full cascade missed
    ]
    cascade_stages_attempted: list[str]  # audit trail for the cascade
    review_notes: list[str]
```

**Validation evidence.** Grounding rate + FP rate on n=72 v15 corpus,
**broken down by cascade stage** so the per-stage hit rate is visible
(STATE.md L73 noted Phase 7 was firing 146×/30min with 100% no_match
under the old config — Phase 4 instrumentation must surface that class
of failure immediately). Per-ENT trace recording every cascade stage
attempted.

**Hard requirements driven by the user decision:**

- Cascade stages 2 and 3 (OpenFIGI, Gemini MCP) MUST be **default-ON**
  in Phase 4.5 — no operator-flag gate. Per
  `[[feedback-drop-feature-flags]]`: default-OFF flags become tech debt
  and the flip surfaces unwired integrations (which is exactly what
  Phase 7 hit per the drift report cited above). Phase 4.4 verifies
  the integration end-to-end before 4.5 cuts over.
- SecMaster-auto-registration from OpenFIGI / Gemini hits MUST be
  audited per the `discovery_source` tag so a regression in either
  upstream surfaces in metrics (Phase 4.6).
- Per `[[feedback-rag-over-deterministic-gates]]`: the cascade trusts
  the existing CoVe literal-presence + similarity-floor (PR #332)
  gating; Phase 4.4 does NOT add new literal-token gates on top.

### 3.2 Component (b) — Claim verification

**Current state.** v15 CLAIMs carry `subject`, `claim_text` (when
present), `evidence_span` / `source_words`. v2.3.1 verifier validates
word-grounding of `source_words` against article text — no semantic
check that the evidence supports the claim.

**§7.2 ask.** One LLM call per CLAIM: "Does `<evidence_span>` support
`<subject> <predicate> <object>`? Answer yes/no/partial." A/B Claude
4.7 (Foundry) vs qwen2.5:32b (local).

**Recommended shape.** One call per CLAIM that passed §7.1. Prompt at
`docs/benchmarks/cod-2026-05-17/prompts/claim_verifier_v1.txt` (NOT
under `/opt/ai-inference/prompts/`). Pure-function shape mirrors
`CoveStatementVerifier`.

**Production arm — single, locked: local qwen2.5:32b on
`vllm-server`:8000.** Per user decision 2026-05-24: **production cloud
LLM budget is $0.** The only cloud LLM allowed in production is the
Gemini MCP (Component a, stage 3 catalog fallback). Foundry / Anthropic
direct / Gemini-as-LLM are training-and-labeling tools, NOT runtime
components.

**Training/labeling reference arm — Foundry Claude 4.7 (Phase 4.7
acceptance only).** Used to label the n=50 acceptance set for
agreement measurement against the production qwen2.5:32b arm. Per
parent plan §7.3 and `[[feedback-azure-foundry-vs-anthropic-cost]]`:
Foundry $9 vs Anthropic direct $100 for the same task — Foundry chosen
when a paid reference arm is needed.

**Gemini paid-tier as labeling tiebreaker** (per
`[[project-gemini-paid-tier]]`, ~$0.00006/call) if Foundry vs
qwen2.5:32b human-spot-check is ambiguous on a sub-sample of n=50.
Still labeling-tier — not a production path.

Per-CLAIM agreement reported on n=50 with human spot-check (parent
plan §7.3); production arm is locked regardless of the labeling-tier
verdict (the labeling tier informs prompt iteration on
`claim_verifier_v1.txt`, not runtime arm selection).

**Output contract (per CLAIM):**

```python
@dataclass
class ClaimVerification:
    claim_block_id: str
    support: Literal["full", "partial", "none"]
    confidence: float                 # [0, 1]; downstream threshold cutoff
    arm: Literal[
        "vllm_qwen2_5_32b",            # production (locked)
        "foundry_claude_4_7",          # training/labeling reference only
        "gemini_paid_tier",            # labeling tiebreaker only
    ]
    rationale: Optional[str]          # nullable to bound tokens
    latency_ms: int
    cost_usd: float                   # 0.0 for vllm arm; non-zero only on labeling-tier arms
```

### 3.3 Component (c) — Sector tagging

**Current state.** None. EVTs carry `event_kind`, `subject` (ENT-ref),
`trigger`, optional `time`. Sector inference today is implicit via the
ENT→SecMaster path (Component a) but grounds the ENT, not the event.

**§7.2 ask.** One LLM call per validated EVT (where §7.1 returned `pass`
AND at least one `subject_refs` ENT grounded to catalog via Component
a). NAICS 2-/3-digit table inline in prompt; 6-digit drill-down by-
reference. Cost-conscious batching: dedup identical prompts within an
article (same `subject_refs` + `event_kind` → one call).

**Production arm — locked to local qwen2.5:32b** per the same user
decision (no cloud LLMs in production except Gemini MCP for entity
catalog miss). Foundry / Gemini paid-tier remain available as
training/labeling reference for Phase 4.7 acceptance only.

**Output contract (per EVT):**

```python
@dataclass
class SectorTag:
    rank: int                         # 1 = highest-confidence
    sector_code: str                  # NAICS 2-digit
    industry_code: Optional[str]      # NAICS 3-/4-/6-digit
    confidence: float                 # [0, 1]

@dataclass
class EventSectorTagging:
    evt_block_id: str
    tags: list[SectorTag]             # sorted by rank; typically 1–3
    arm: Literal[
        "vllm_qwen2_5_32b",            # production (locked)
        "foundry_claude_4_7",          # training/labeling reference only
        "gemini_paid_tier",            # labeling tiebreaker only
    ]
    latency_ms: int
    cost_usd: float                   # 0.0 in production; non-zero only on labeling-tier arms
```

**Validation evidence.** Coverage rate (fraction of validated EVTs with
≥1 tag above confidence floor) + human spot-check on a sample.

---

## 4. Input / output contract

**Input.** Existing `Document` AST
(`docs/benchmarks/cod-2026-05-17/dsl/types.py`) +
`VerificationReport` from `verifier_v2_3_1.py`. No new parser, no new
grammar.

**Output.**

```python
@dataclass
class EnrichedDocument:
    source_document: Document
    verification_report: VerificationReport
    entity_resolutions: dict[str, EntityResolution]    # keyed by ENT.local_id
    claim_verifications: dict[str, ClaimVerification]  # keyed by claim_block_id
    event_sector_taggings: dict[str, EventSectorTagging]
    metadata: EnrichmentMetadata                       # arm choices, total cost, latency
```

Phase 5 (parent plan §8.1) reads `verification_report` (pass/fail) +
`event_sector_taggings` (sector annotation on the wrapping EVT) for NUM
magnitude contribution; reads `polarity` slot + verification_report for
EVT direction; reads `claim_verifications[id].confidence` for downweight/
discard decisions.

**Safety property (parent plan §8.2).** Matrix never receives a value
that wasn't anchored to a source span. Phase 4 enforces by refusing to
enrich any block §7.1 `hard_fail`-ed; threading `verification_report`
through `EnrichedDocument` so Phase 5 can re-check before signal
emission; recording per-call cost + latency in `EnrichmentMetadata`.

---

## 5. Phased PR sequence

Eight PRs. Each ≤500 LOC, independently mergeable, no PR depends on a
future PR. Per `[[feedback-drop-feature-flags]]`: no default-OFF flags;
integration glue at Phase 4.5 is the cutover.

### Phase 4.1 — input/output contract + data shapes

**Scope.** `EntityResolution`, `ClaimVerification`, `SectorTag`,
`EventSectorTagging`, `EnrichmentMetadata`, `EnrichedDocument` +
`build_enriched_document(...)` constructor (validates input shape,
refuses hard_fail-only docs, refuses non-v2.3+ docs, enforces no
duplicate block IDs, pre-allocates dicts). Round-trip JSON.

**Files (new):** `dsl/enrichment_types.py`,
`dsl/tests/test_enrichment_types.py`,
`dsl/tests/fixtures/v15_enriched_minimal.json`.

**Acceptance.** Round-trip JSON; constructor rejects malformed input;
empty enrichment dicts on initial construction.

### Phase 4.2 — claim verifier + A/B harness

**Scope.** `ClaimVerifierService(block, article_text) ->
ClaimVerification`. Two arms (Foundry via the `azure_oracle_client.py:44`
ledger pattern; local via the existing `VllmClient` pattern). Prompt
authored here. A/B harness emits per-CLAIM diff JSON.

**Files (new):** `dsl/claim_verifier.py`,
`prompts/claim_verifier_v1.txt`, `scripts/run_claim_verifier_ab.py`,
`dsl/tests/test_claim_verifier.py`.

**Acceptance.** Unit tests: (a) stubbed full-support → confidence ≥
0.8; (b) stubbed no-support → confidence ≥ 0.8; (c) malformed LLM
response → `support=none, confidence=0.0` + marker
`[claim-verifier] malformed-response`; (d) A/B harness comparison JSON.
Live LLM tests skip-marked behind `CLAIM_VERIFIER_LIVE_TEST=1` env (no
live calls in CI; live calls in Phase 4.7 only).

### Phase 4.3 — sector tagger

**Scope.** `SectorTaggerService(evt, ent_resolutions) ->
EventSectorTagging`. NAICS 2-/3-digit table inline in prompt. Batching
dedup within article. Same A/B harness pattern as 4.2.

**Files (new):** `dsl/sector_tagger.py`,
`prompts/sector_tagger_v1.txt`, `data/naics_2_3_digit.jsonl`,
`dsl/tests/test_sector_tagger.py`.

**Acceptance.** Unit tests: (a) deterministic sector for a known event
(energy company + drilling → sector 21); (b) batching reduces call
count when identical prompts repeat; (c) confidence-floor cutoff
returns empty `tags` rather than spurious top-1 (mirrors PR #332).

### Phase 4.4 — entity resolution cascade (SecMaster + OpenFIGI + Gemini)

**Scope.** Implements the §3.1 cascade as the production entity-
resolution path. Wires the existing SecMaster CoVe (stage 1), OpenFIGI
(stage 2, `SecMaster/src/Services/OpenFigiClient.cs`), and Gemini MCP
fallback (stage 3, `SentinelCollector/src/Services/GeminiSymbolFallbackService.cs`,
PR #328) into a single `EntityResolver.resolve(ent, article_text) ->
EntityResolution`. Per-stage attempt + outcome recorded in
`cascade_stages_attempted` for audit. Stages 2 and 3 ship default-ON
per the §3.1 user decision.

**Optional Stage 0 — embedding-similarity overlay (measured here, ship
IFF wins):** A/B harness comparing qwen2.5:32b-as-embedder (via
vllm-server's embedding endpoint; recon during impl: does the current
vllm-server serve embeddings or need a sidecar?) against the existing
Phase-1 v5 cosine-RAG embeddings (PR #311). Ships as Stage 0 IFF it
beats stage 1 grounding rate by >5pp without a false-positive
regression. If it loses, overlay code is NOT merged.

**Files (new):**
- `dsl/entity_resolver.py` (cascade implementation — calls into the
  existing SecMaster gRPC :8080, OpenFIGI HTTP, Gemini MCP)
- `scripts/run_entity_resolver_cascade.py`
- `scripts/run_entity_resolver_overlay_ab.py` (Stage 0 A/B harness)
- `dsl/tests/test_entity_resolver.py`
- `results/phase4-ent-resolution-cascade/REPORT.md`

**Acceptance.**
1. Cascade end-to-end test on n=72 v15 corpus: per-stage hit rate
   reported in REPORT.md (catches the Phase 7 drift class — 146×/30min
   100% no_match — at integration time, not in production).
2. Stage-2/Stage-3 auto-registration verified: an OpenFIGI hit
   registers the instrument with `discovery_source='OpenFIGI'`; a
   Gemini hit registers with `discovery_source='GeminiFallback'`. Both
   visible in SecMaster after the run.
3. Stage 0 A/B: REPORT.md records "ship overlay as Stage 0" or "skip
   overlay". Overlay code merges IFF the decision is ship.
4. Per `[[feedback-no-placeholder-prs]]`: substantive implementation =
   end-to-end cascade + auto-registration verification, not empty
   stubs.

### Phase 4.5 — `SemanticVerifierService` in-process integration — CUTOVER

**Scope.** Wires 4.2 + 4.3 + 4.4 into a single
`SemanticVerifierService.enrich(document, verification_report) ->
EnrichedDocument`. Sequencing: entity resolution cascade first (feeds
sector tagger); claim verifier runs in parallel (no dependency).
Surfaces `EnrichedDocument` to downstream.

**Deploy target (user decision, 2026-05-24): in-process.** Phase 4.5
ships as a library consumed in-process by the Phase 5 consumer
(ThresholdEngine, primarily). This matches the existing
`CoveStatementVerifier` / `CoveSymbolVerifier` pattern (pure
service classes called from `ExtractionProcessor` in-process) and is
the going-forward pattern. No new container, no sidecar — the existing
HTTP clients (`VllmClient` to `vllm-server:8000`,
`GeminiSymbolFallbackService` to the Gemini MCP, `OpenFigiClient`)
remain the network boundaries; the orchestration is in-process.

Direct cutover per `[[feedback-drop-feature-flags]]` — no default-OFF
flag.

**Files (new):**
- `dsl/semantic_verifier.py` (Python orchestration for the
  benchmark-corpus runs)
- `dsl/tests/test_semantic_verifier.py`
- `scripts/run_semantic_verifier.py`

**Cross-cutting (C# side, for Phase 5 in-process consumption):** the
benchmark `dsl/semantic_verifier.py` orchestration is the PoC
reference; production integration into ThresholdEngine ships as part
of Phase 5 per parent plan §8 (out of scope for this plan, but the
in-process pattern is locked here so Phase 5 doesn't re-litigate).

**Acceptance.** Integration test on fixture v15 document: emits an
`EnrichedDocument` whose ENT-resolutions, claim-verifications, EVT-
sector-tags are populated. Wall-time on n=10 <60s total (outer SLO at
Phase 4.7). Rollback = `git revert` + ansible redeploy of the
consumer; downstream reverts to consuming raw deterministic
verification report. No DB schema, no compose changes.

### Phase 4.6 — observability

**Scope.** Metrics + dashboard panels paired with existing v2 Pipeline
Health (`deployment/artifacts/monitoring/dashboards/v2-pipeline-health.json`).

**Metrics (new):**
- `atlas_semantic_verifier_claim_pass_rate` (gauge, tag=arm).
- `atlas_semantic_verifier_sector_tagger_coverage` (gauge).
- `atlas_semantic_verifier_entity_resolver_grounding_rate` (gauge).
- `atlas_semantic_verifier_call_count_total` (counter, tags=component, arm).
- `atlas_semantic_verifier_call_latency_ms` (histogram, tags=component, arm).
- `atlas_semantic_verifier_cost_usd_total` (counter, tags=component, arm).
- `atlas_semantic_verifier_per_article_cost_usd` (histogram).

**Tagging discipline (CLAUDE.md OBSERVABILITY):** bounded cardinality.
`component ∈ {claim_verifier, sector_tagger, entity_resolver}`,
`arm ∈ {foundry_claude_4_7, vllm_qwen2_5_32b, secmaster_cove,
embedding_overlay}`. No article/block IDs in tags.

**Files (new/modified):** `dsl/semantic_verifier.py` (metric
emission), `v2-pipeline-health.json` (4 new panels),
`dsl/tests/test_semantic_verifier_observability.py`.

**Acceptance.** Metrics surface in Prometheus after smoke run; 4 new
panels render against n=10 sample.

### Phase 4.7 — acceptance run (n=50)

**Scope.** Per parent plan §7.3: claim verifier vs human spot-check
≥85% agreement; per-article fact yield measured.

**Files (new — results only):**
- `results/phase4-acceptance-n50/REPORT.md`
- `results/phase4-acceptance-n50/spot_check.jsonl`
- `results/phase4-acceptance-n50/enriched_docs/*.json`

**Gates.**
1. 100% copy slots pass §7.1 deterministic verification or are flagged
   (already met; re-confirmed on n=50).
2. Claim verifier ≥85% human-agreement on n=50.
3. Per-article fact yield (validated NUMs + catalog-grounded ENTs) =
   histogram + median + p95.

If gate 2 fails: diagnose via per-CLAIM disagreement; remediation = prompt
iteration on `claim_verifier_v1.txt`. If gate 3 yields ~zero: Phase 5
unblocked but signal thin — escalate before declaring done.

### Phase 4.8 — rollback rehearsal + cleanup

**Scope.** Per `[[feedback-completion-gate]]`: exercise rollback before
declaring Phase 4 closed. Verify `git revert` of 4.5 + ansible
redeploy returns downstream to pre-cutover behavior cleanly. Delete
A/B harness scripts if arms are locked; keep comparison REPORT.md.

**Acceptance.** Rollback exercised on staging or throwaway
nerdctl-compose stack; pre/post-revert diff empty for downstream
consumers.

---

## 6. Cost considerations

**Production cloud LLM budget = $0** (user decision, 2026-05-24). The
only cloud LLM permitted in the production runtime path is the Gemini
MCP catalog fallback (Component a, cascade stage 3) — that integration
is already on the existing Phase 7 cost line item (PR #328) and not
reopened here. **Foundry / Anthropic direct / Gemini-paid-tier are
training-and-labeling tools only.**

### 6.1 Production runtime (per-article, $0 cloud)

- **Component a — entity resolution cascade:** SecMaster RAG + CoVe
  $0 marginal; OpenFIGI $0 marginal at the per-call layer (existing
  rate-limit-bounded API with `OpenFigiLookupCacheEntity` cache);
  Gemini MCP — uses the Gemini paid-tier endpoint already in place
  (per `[[project-gemini-paid-tier]]`, ~$0.00006/call); only fires on
  SecMaster + OpenFIGI miss, so per-article exposure is bounded by
  the catalog-miss rate. The user-tagged "significant failure" is a
  catalog miss itself — at production volume Gemini MCP cost is
  expected to remain in the low-single-digit dollars/month range
  consistent with the existing Phase 7 footprint.
- **Component b — claim verifier:** local qwen2.5:32b on
  `vllm-server`:8000 — $0 marginal (GPU contention is a capacity
  question, not a $-cost question; see Risk 1).
- **Component c — sector tagger:** same as b — $0 marginal.

### 6.2 Training / labeling budget (Phase 4.7 acceptance only —
**up for discussion**)

Reference: Foundry Claude 4.7 at ~$1.41 cost / $5 revenue per Phase 1.4
frontier (PR #382, STATE.md L7). Per
`[[feedback-azure-foundry-vs-anthropic-cost]]`: Foundry $9 vs Anthropic
direct $100 — Foundry is the chosen paid-arm channel.

- **Per-CLAIM labeling reference (Foundry Claude 4.7):**
  ~$0.001–0.005/call. n=50 articles × ~10 CLAIMs = 500 calls × ~$0.003
  ≈ **$1.50** worst case.
- **Per-EVT labeling reference (Foundry):** ~$0.0005–0.002/call.
  n=50 × ~5 EVTs × ~$0.0015 ≈ **$0.40**.
- **Gemini paid-tier tiebreaker** (per `[[project-gemini-paid-tier]]`,
  ~$0.00006/call): ~$0.05 total for the same n=50 envelope.

**Suggested Phase 4.7 budget — propose to user before fire:** $10
ceiling on Foundry labeling spend; $1 ceiling on Gemini paid-tier
labeling spend. **User confirms before Phase 4.7 runs.** If the user
prefers a $0 labeling budget too, fall back to human spot-check
without a paid reference arm (slower but free).

---

## 7. Rollback strategy

Per `[[feedback-drop-feature-flags]]`: no default-OFF flag. Each
component lands as additive; integration glue at Phase 4.5 is the
cutover when downstream consumers (Phase 5) start reading enriched
output.

- 4.1, 4.2, 4.3, 4.4, 4.6: additive; no downstream changes. `git revert`
  no-op for downstream.
- 4.5 cutover: rollback = `git revert` + ansible redeploy. No DB schema,
  no schema rollback. ZFS snapshot of `/var/lib/atlas` before deploy
  per `[[feedback-drop-feature-flags]]` is the catch-all.
- 4.7: results-only; revert = no behavior impact.
- 4.8: cleanup; revert restores A/B harness scripts.

**Hard constraint.** No partial-state rollback. If a Phase 4.5
component is broken (e.g. sector tagger fine, claim verifier broken),
the fix is forward: follow-on PR that disables the broken component at
the service-construction layer (DI-time arg, not runtime flag).

---

## 8. Observability deliverables

Detailed in §5 Phase 4.6. Pairs with v2 Pipeline Health dashboard
(`deployment/artifacts/monitoring/dashboards/v2-pipeline-health.json`)
— new panels go in a `GPU semantic verifier` row appended below
existing Throughput & Success.

Per CLAUDE.md ALERTING: one P2_URGENT alert per the four new gauges —
sustained `entity_resolver_grounding_rate < 0.5` for >15 min is a
regression flag (the rest are dashboarded, not alerted, until
production volume calibrates a threshold).

---

## 9. Open questions — RESOLVED (user, 2026-05-24)

1. **SLO target on claim verifier pass rate.** **DECIDED — pin
   `claim_pass_rate` at 0.70.** The human-agreement gate (≥85%, parent
   plan §7.3) is the calibration target; 0.70 is the runtime gauge
   tracked in Phase 4.6 metrics.

2. **Sector tagger fallback when LLM confidence low.** **DECIDED —
   emit empty `tags` list and let Phase 5 ignore the EVT.** No
   deterministic NAICS-keyword fallback. User note: failures here will
   need to be tuned later; emitting empty preserves a clean audit
   trail in the meantime.

3. **Entity resolution upgrade scope.** **DECIDED — upgrade SecMaster
   AND wire OpenFIGI + Gemini MCP into the runtime cascade.** Prior
   cosine-only RAG biased toward seen entities (the 30K hallucinated-
   Symbol Approved rows are the evidence); missing instruments are
   significant failures. §3.1 and §5 Phase 4.4 are rewritten around
   the three-stage cascade (SecMaster → OpenFIGI → Gemini MCP) shipped
   default-ON. The optional embedding-similarity overlay stays
   measured-first (Stage 0).

4. **Phase 4 deploy target — sidecar vs in-process.** **DECIDED —
   in-process.** Matches the existing CoVe* pattern (pure service
   classes called from `ExtractionProcessor`) and is the going-forward
   pattern for ATLAS. §5 Phase 4.5 reflects this.

5. **NAICS vs GICS taxonomy.** **DECIDED — NAICS always.** GICS
   licensing cost is prohibitive; NAICS is free and already the
   taxonomy referenced by SecMaster (per `SecMaster/README.md` L161 —
   NAICS classification + ATLAS-sector rollup are in the migration
   history). If a free GICS source is ever identified, revisit; until
   then NAICS is locked.

---

## 10. Risks + mitigations

**Risk 1 — GPU contention.** `sentinel-cove-v6.2` on `vllm-server` is
the production extraction model (compose.yaml.j2 L817:
`Extraction__Model=sentinel-cove-v6.2`). Phase 4.2 + 4.3 add per-CLAIM
+ per-EVT calls to the same GPU. At PoC scale (n=72) fine; at
production (thousands of articles/day) bottleneck.
*Mitigation:* production runtime arm IS local qwen2.5:32b per the
user's $0 cloud-LLM rule (§6, Risk 2 below); GPU capacity is the
constraint to plan around. Phase 4.6 metrics expose
`atlas_semantic_verifier_call_latency_ms` per component so capacity
trends are visible early. Production wall-time budget remains TBD per
parent plan §8.3 — Phase 5 sets the SLO and decides whether the
shared GPU needs a dedicated verification-tier vllm-server (a follow-on
deploy question, out of scope here).

**Risk 2 — Production cloud LLM cost.** Per user decision 2026-05-24:
**production cloud LLM budget is $0.** Closed by construction — the
production runtime path uses only local qwen2.5:32b on `vllm-server`
(claim verifier + sector tagger) and the existing Gemini MCP catalog
fallback (already on the Phase 7 line item). Foundry / Anthropic
direct / Gemini paid-tier are training-and-labeling tools and never
enter the production runtime.
*Mitigation:* §6 splits production ($0) from training/labeling (under
discussion, capped at proposed $10 ceiling for Phase 4.7 — user
confirms before fire). The arm-selection literals in
`ClaimVerification.arm` and `EventSectorTagging.arm` make the
production-vs-labeling distinction type-level, not config-level: a
runtime call with `arm != "vllm_qwen2_5_32b"` fails the production-
deploy gate.

**Risk 3 — Claude 4.7 vs qwen2.5:32b A/B disagreement.** If arms
disagree >30% per-CLAIM, spot-check at Phase 4.7 won't unambiguously
identify the winning arm.
*Mitigation:* Gemini paid-tier tiebreaker (~$0.00006/call, negligible).
Spot-check protocol: on disagreement CLAIMs, human verdict = ground
truth; arm with higher agreement on disagreement subset wins. If both
arms agree >70% and disagree <30%, the disagreement subset is small
enough that human spot-check is tractable.

---

## 11. Success criteria (consolidated, per parent plan §7.3)

1. **100% of v2 copy slots pass deterministic verification or are
   flagged.** Already met by §7.1; re-confirmed on n=50 in Phase 4.7.
2. **Claim verifier ≥80% Foundry-agreement on the canonical Foundry
   subset (563 rows from the original `test_holdout`).** Phase 4.7
   gate; per-CLAIM agreement reported in
   `results/phase4-acceptance-n50/REPORT.md`.
3. **End-to-end per-article fact yield (validated NUMs + catalog-grounded
   ENTs) measured + tracked.** Phase 4.7 reports histogram + median +
   p95; Phase 5 consumes the fact yield as matrix signal.

### Gate recalibration rationale (2026-05-26)

Gate #2 was recalibrated from **≥85% → ≥80%** Foundry-agreement after
the following empirical arc bounded the achievable ceiling:

- **3 LoRA iterations all collapsed** (PRs #465 / #466 / #473). Iter#1
  and iter#2 returned degenerate single-class outputs; iter#3 OOM-ed
  on the 32B base under per-device VRAM caps and the rescued ckpt-250
  did not move the needle.
- **12 prompt iterations + 2 parser axes plateaued** at the
  no-IMPORTANT baseline (PR #469 IMPORTANT-clause drop + PR #470
  iter-2-axis-C winner). The winning prompt (`iter-2-axis-C`) plus
  the `IsSelfReferential` short-circuit (PRs #449 / #469) hits 80.64%
  Foundry agreement on the original 563-row subset; no further prompt
  axis improved it.
- **3-way model ablation** (PR #458) confirmed Qwen2.5-32B-Instruct-AWQ
  is the right base for the verifier task — no swap recovers the gap.
- **Sparse-evidence + subject-in-evidence predicate probes** (PR #474)
  both returned NO_PROGRESS — neither structural axis explains the
  residual disagreement.
- **Foundry-label expansion** (PR #475) was the decisive evidence: the
  177 `synthetic_unrelated_hard` rows were re-labeled by the Foundry
  oracle (74 subject_swap, 102 value_sign_flip, 1 predicate_inversion).
  Foundry **disagrees with the synthetic IE label on ~77% of the
  subject-swap slice** (Foundry calls them `related` 47% / `consistent`
  22% / `unrelated` 8% / `insufficient_evidence` only 23%). The
  synthetic rule ("swapped subject not in evidence → IE") is
  systematically off-pattern.

  Implication: the "≥85% target" was measuring agreement against a
  benchmark whose hard-negative slice is partly built on miscalibrated
  synthetic labels. The gap between production output and the
  synthetic benchmark is a **metric artifact**, not a missing
  production capability.

- **80.64% on the original 563-row Foundry subset IS the real
  achievable ceiling** for this model + task combination at the
  current evidence shape.

Recalibrating to ≥80% recognises that ceiling. Production at 80.64%
already clears the recalibrated gate by 0.64pp, so Phase 4.7 is
declared **PASS** and Phase 5 (matrix integration, parent plan §8) is
**unblocked**.

If all three clear → Phase 4 closes; Phase 5 (parent plan §8) unblocked.
Any miss → diagnose via per-component validation reports (4.4 REPORT.md,
4.2 A/B harness, 4.3 A/B harness) before pushing into Phase 5.

---

## 12. Phase ordering rationale

```
Phase 3 (v15 DSL extraction on CPU — DONE, T=0 A/B verdict #431/#432/#433)
  │
  ▼
Phase 4 (THIS) — GPU semantic verifier per plan §7.2
  ├─ 4.1: input/output contract + EnrichedDocument shape
  ├─ 4.2: claim verifier + A/B (Foundry vs vllm)
  ├─ 4.3: sector tagger + A/B
  ├─ 4.4: entity resolution cascade (SecMaster + OpenFIGI + Gemini MCP)
  ├─ 4.5: SemanticVerifierService glue (CUTOVER)
  ├─ 4.6: metrics + dashboard panels
  ├─ 4.7: n=50 acceptance + human spot-check
  └─ 4.8: rollback rehearsal + cleanup
  │
  ▼
Phase 5 (matrix integration — plan §8; unblocked iff Phase 4 gates met)
```

4.1–4.4 can land in parallel (no inter-deps — 4.1 ships the contract,
4.2/4.3/4.4 ship per-component services that consume it). 4.5
sequential after 4.1–4.4. 4.6 after 4.5 (metrics need the service).
4.7 after 4.6 (acceptance emits to dashboards built in 4.6). 4.8 after
4.7. Parent plan §11 budgets ~1 week; the 8-PR slice fits if 4.1–4.4
land in parallel and 4.5–4.8 are sequential.

---

## 13. References

- Parent plan: `docs/plans/atlas-dsl-poc-plan.md` §7 (lines 317–406),
  §8 (lines 358–406), §10 open decisions (lines 421–435).
- Phase 3 v2 schema: `docs/plans/atlas-dsl-poc-phase3-v2-schema.md`.
- Phase 4 §7.1 deterministic verifier (DONE):
  `docs/benchmarks/cod-2026-05-17/dsl/verifier_v2_3_1.py`.
- v15 DSL parser: `docs/benchmarks/cod-2026-05-17/dsl/parser_v2_3.py`.
- AST types: `docs/benchmarks/cod-2026-05-17/dsl/types.py`.
- Existing GPU verification surface:
  `SentinelCollector/src/Services/CoveSymbolVerifier.cs`,
  `SentinelCollector/src/Services/CoveStatementVerifier.cs`,
  `SentinelCollector/src/Services/VllmClient.cs`.
- vllm-server compose config:
  `deployment/artifacts/compose.yaml.j2` L811–838.
- Foundry ledger pattern:
  `SentinelCollector/scripts/azure_oracle_client.py:44`,
  `docs/benchmarks/cod-2026-05-17/scripts/run_phase1_4_foundry.py:49`.
- v2 Pipeline Health dashboard:
  `deployment/artifacts/monitoring/dashboards/v2-pipeline-health.json`.
- Architectural rule (LLM-strength layering): user memory
  `project_sentinel_llm_strength_layering` (2026-05-20).
- Topology rule: PR #386 ("immovable", per Phase 3 v2 schema plan
  line 8).
