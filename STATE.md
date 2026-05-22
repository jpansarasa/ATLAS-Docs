# ATLAS Supervisor STATE [2026-05-16]

## ACTIVE [2026-05-21]
- **DSL PoC plan** (`docs/plans/atlas-dsl-poc-plan.md`) drives current work. **Inference topology rule (PR #386):** CPU = extraction (qwen3:30b-a3b on `ollama-cpu-gen`, port 11435). GPU = summarization + verification (sentinel-cove-v6.2 on `vllm-server`). Benchmark work targets CPU; do NOT lift plan §5's vLLM-specific text verbatim.
- **Phase 0 + Phase 1 complete (PRs #381–#385 merged 2026-05-20):**
  - **#381** Phase 0 instrumentation — per-category recall (ticker/org/person/loc/instrument/sector/industry/numeric_exact/numeric_semantic/unit_preservation), `span_copy_fraction`, `cost_per_recall_point`. R2/R3 rescored under new metrics.
  - **#382** Phase 1.4 — Claude 4.7 frontier reference. qwen3:30b only +3.9pp behind on numeric_exact; **qwen3:30b at capability frontier — don't chase a bigger model.** Foundry cost $1.41/$5.
  - **#383** Phase 1.1 — direct-extraction arm. `direct ≈ spec` on mechanistic metrics; **structure is the whole CoD win — retire prose-CoD prompt path.**
  - **#384** Phase 1.2 — schema-order experiment. v1b (derived-first) collapses gv% 100%→50%. **Tam et al. mechanism CONFIRMED; verbatim-first locked as design rule for Phase 2+3.**
  - **#385** Phase 1.3 — Class C qwen2.5:32b dense. qwen3:30b-a3b (MoE) beats qwen2.5:32b on every recall AND is 3.19× cheaper. **MoE-active-3B is the right shape.**
  - Mechanism: `span_copy_fraction` ↔ `numeric_exact_match` Pearson r=+0.6 (R2/R3) — induction-head copy-mode story has direct mechanistic evidence (#381 correlation analysis).
- **Phase 2 infrastructure DONE (PR #387 squash-merged 2026-05-21 → `25cb33fa`):** llama.cpp `llama-server` CPU sibling on :11437 serving qwen3-30b-a3b via bind-mounted ollama blob; v1.1 GBNF enforces (repetition bounds lowered to satisfy llama.cpp's `MAX_REPETITION_THRESHOLD=2000` + combinatorial rule-expansion cap); runner gains `--backend {ollama,llama-server}` + CANARY_OK pipeline-health probe (default stays ollama, byte-additive); 35/35 runner tests. Empirically established during scoping: ollama 0.24 silently drops raw GBNF via `format` (use of `--ollama-engine` Go runner).
- **Phase 2 sweep + terminator fix MERGED** (PR #389 → `cbf96102`, 2026-05-21): 3-arm sweep diagnosed §5.3 gv% gate FAIL (40%, root cause = `block-list` permissive + no model stop criterion). v1.2 GBNF adds explicit `END` block terminator; n=3 validation shows all cells voluntarily stop on EOS (mechanism validated). gv% gate hit 2/3 on n=3 — single failure (article 31430) is an orthogonal degenerate-loop pathology, not a terminator issue. §5.4 simplification gate FAIL (content-only prompt collapses to 0% num_rec) → keep grammar-taught prompt. Numeric recall (95.5% on Arm B, +6.9pp over Phase 1) and latency (-10.2%) gates PASS. Full n=10 v1.2 deferred until publication trail needs it.
- **Phase 3 v2 schema: baseline + probe arc MERGED (PRs #390 → `d6f35887`, #391 → `f7aa178c`)** — both 2026-05-21:
  - #390 baseline: full v2 implementation (grammar+prompt+AST+verifier+scoring), n=9 acceptance OVERALL FAIL on `verbatim_match_rate=0%` across all gv-valid cells.
  - #391 probe bundle: path (1) MoE + strengthened prompt + path (2) qwen2.5:32b dense (sidecar on :11438) — both vmr=0 too. 10× active-param scaling fixed gv (33%→100%) but didn't move spans one decimal.
  - **ARCHITECTURE verdict (not model-capacity)**: BPE tokenization is character-blind; position embeddings encode token positions not byte positions. No in-model fix closes the byte-arithmetic gap.
  - v2 SLOT VOCABULARY remains useful; spans need external prosthesis.
- **Phase 3 v2.1 no-offset probe MERGED** (PR #392 → `0d13090b`, 2026-05-21): **PATTERN VALIDATED.** v2.1 grammar makes `source_span` OPTIONAL; v2.1 prompt drops byte-arithmetic ask; verifier extended with `find()` fallback. n=3 smoke on prior-fail articles: verbatim_match avg **0.821** (vs 0.0 across all 3 prior arms), 5× faster wall. **3b-cheap design empirically confirmed** — small CPU model emits verbatim text, deterministic Python computes spans. No GPU needed for this layer. Bonus parser quote-strip bug fix included (silent perma-fail across PR #390/#391 arms). 94 tests pass.
- **Phase 3 v2.1 n=9 scale-confirmation MERGED (PR #393 → `23dc4ce6`, 2026-05-21)**: **§6.4 FAIL — pattern does NOT hold at scale.** gv collapses to 3/9 = 33% (vs 67% on n=3); verbatim 0.611 (vs 0.821); verifier_pass 0.944 (just below 0.95 gate). Recall gates (numeric/ticker/org) UNDETERMINABLE on this subset — Arm B corpus lacks per-category GT. 3-cell intersection with PR #392 probe showed 31430 regressed gv=True→False, indicating the n=3 verbatim signal was carried by a nondeterminism-sensitive cell. Dominant failure (5/6): lenient preprocessor synthesizes inline ENT blocks for `<ent_ref>` referents without emitting `name:` slot (preprocessor papering over a model declare-then-reference gap); 1/6: `n_predict=4096` truncation on 5K-token outputs.
- **Phase 3 ENT-ref cleanup MERGED (PR #394 → `8444bb00`, 2026-05-21)**: research-aligned reframe approved by user. Parser no longer synthesizes inline ENT blocks for `<ent_ref>` referents; verifier hard_fails undeclared refs per plan §7.1 (was already implemented); prompt teaches declare-then-reference. **Validated on 3 prior-fail cells**: 27560 HOLD+improve (0.833→0.900), 31149 RECOVER fully (verbatim=1.0), 31430 RECOVER partially with HONEST verifier signal (0.625 = 6 EVT/CLAIM blocks hard_fail for 8 dangling `_anon:` refs — model declare-then-reference gap now visible instead of papered over). Verbatim purity 0.833 → 0.967 mean. 52 dsl tests pass.
- **Phase 3 prompt iteration v3 → v4 → v5 (2026-05-21)** — **HALT per "iterate until no more progress" directive.** Trajectory verifier (n=9): 0.790 → 0.808 → 0.782.
  - **#395 v3 `_anon:` discipline MERGED**: gv 100% restored after ENT-ref cleanup; verifier 0.790.
  - **#396 v4 ID-form + PascalCase MERGED → `a694ecbb`**: verifier +0.018 (near-noise); org_recall +0.277 real.
  - **#397 v5 dedup + kind-priority + punctuation-ident MERGED → `438fb764`**: **all 3 targeted mechanisms CLOSED.** 36065 concept-latch verifier 0.711 → 0.976 (+0.265) — largest single-intervention win in Phase 3. 33632 dup-decl closed (51/51 distinct ENT). 27702 punctuation-ident closed. Aggregate 0.782 ≤ 0.82 halt threshold; dragged below by 2 non-intervention residuals (35352 sampling variance -0.491, 27702 empty-article -0.250). Excluding those: **mean 0.959 (n=6, above CONTINUE)** — Phase 3 ceiling is well above the aggregate floor.
  - **Architectural follow-ups (NOT prompt-iterable on 4B-active MoE)** — user direction pending (NTFY `l9nciqW0t9L7`): (a) bump server `--ctx-size` 16K→32K (33632 hit cap with ~10K-token prompt); (b) n>1 sampling per cell to dampen variance (35352-class); (c) upstream corpus filter for empty-article rows (27702-class); (d) move to plan §7 GPU verifier-as-judge; (e) merge #397 and pause.
- **Phase 4 GPU verifier deferred (but back on the table per follow-up (d))**: 3b-cheap (deterministic, pure Python, no GPU) is the better path per no-offset evidence; 3b-llm (plan §7.2) only if downstream semantic-claim verification needs it. Aggregate-floor mop-up could escalate this.
- **Parking lot**: user raised idea of benchmarking llama.cpp vs vLLM-on-CPU on the existing benchmark corpus (separate workstream).
- **Push-guard hook fix MERGED** (PR #388 → `6721bff2`, 2026-05-21): marker lookup scopes globally by tree-hash; agent-worktree teardown no longer orphans test markers. `claude-mark-verified` no longer needed as a workaround for main-worktree pushes of agent-worktree-tested branches.
- **Runtime posture (PoC testing window):** sentinel-collector, secmaster, secmaster-mcp, alert-service, and all MCP containers (finnhub/threshold-engine/markitdown/whisper/ofr/fred) are STOPPED to free ollama-cpu-gen for benchmarking. Restart when DSL PoC test window ends. User direction: alert-service stays down (output not useful right now).
- **Older arc (symbol-identification remediation, 2026-05-15→16)** retained below as supporting history.

## PHASE STATUS
- **Phase 1** — Spine — ✓ DONE (Epic 1 merged as `a08b806`)
- **Phase 2** — Substrate / TE schema / LLM track — ✓ DONE (modulo 4.6.x)
  - Epic 2 (TE schema rework) — ✓ DONE
  - Epic 3 (`macro_observations` substrate) — ✓ DONE
  - Epic 4 LLM track — ✓ DONE through 4.5 (Sentinel subscription); 4.6.x LoRA reliability pending (multi-week, dispatched separately)
- **Phase 3** — Sentinel rework + matrix — ✓ DONE (Epic 5 matrix-storage MVP complete: F5.1 + F5.4 + F5.5.1 + F5.5.2 + F5.5.3)
- **Phase 4** — Surfaces — → IN FLIGHT (Epic 6: 6.1.x ✓, 6.2.x active)

## SENTINEL SYMBOL-IDENTIFICATION REMEDIATION [arc 2026-05-15 → 2026-05-16]

Canonical plan: `docs/plans/symbol-identification-remediation.md`. Architecture: `docs/sentinel-extraction-pipeline.md`. Trigger: 30K+ hallucinated-Symbol Approved rows discovered 2026-05-15 (LoRA-era damage; root cause = cosine-similarity nearest-neighbor over field-poor RAG with no source-grounding check). Defense-in-depth target: rich embeddings + top-N retrieval + CoVe grounding + Gemini catalog-of-last-resort + statement-level validation.

### Phases
- **Phase 1 — Rich RAG embeddings (v5: ticker + name + description + industry + sector)** — ✓ DONE. PR #311. 100% v5 coverage on 9,073 active instruments; EDGAR sicDescription backfill populated meaningful business descriptions on equities.
- **Phase 2 — Top-N RAG retrieval (default 5, cap 10)** — ✓ DONE. PR #312. `LocalResolutionDto.Candidates` array with similarity-score ordering. Backwards-compatible with single-candidate callers.
- **Phase 3 — CoVe Symbol grounding on structured-extraction path** — ✓ DONE.
  - PR #313: `CoveSymbolVerifier.PickFirstGrounded` wired into `ExtractionProcessor.ApplyLocalResolutionAsync`. Iterates candidates similarity-descending, picks first whose Symbol or Name literally appears in `text_quote ∪ context_summary`; otherwise sets Symbol/InstrumentId to null + `ResolutionState=NoResolution` + `[cove] no-symbol-grounded-in-source` review note.
  - PR #314: predicate hardening — word-boundary regex for ≥4-char Symbols, Name-only (≥4-char substring) for ≤3-char Symbols; persists `NoGroundingNote` to `review_notes`.
  - PR #317: ResolutionWorker CoVe wiring + cascade-exhausted marker so the v2 deterministic-resolver path is also covered.
  - PR #331: tightened CoVe Name predicate to close the TXNM/SFIX/SNX leak class.
- **Phase 4 — Re-extract backfill BackgroundService** — ✓ CODE LIVE.
  - PR #318: feature-flag-gated BackgroundService (default OFF).
  - PRs #321/#322: resolve-only mode (skip LLM extraction, re-run RAG+CoVe against existing rows) + ops-enable.
  - PR #323: skip RAG LLM-pick in resolve-only mode → 40s/call → <1s (~85× throughput).
  - Status today: running, ~218 rows/min, ~65K rows remaining.
- **Phase 5 — AutoApprove re-engage** — ◯ HOLD. Awaiting fresh readiness recon after Phase 4 backfill drains a meaningful post-all-fixes cohort. Gate: true grounding rate ≥90% before flipping `Extraction__AutoApproveEnabled=true`.
- **Phase 6 — Statement-level CoVe validation (structured-extraction path)** — ✓ CODE LIVE default-OFF. PR #329. Operator decision pending to activate.
- **Phase 7 — Gemini MCP Google Finance fallback (catalog-of-last-resort)** — ✓ CODE LIVE default-OFF. PR #328. Closes the catalog-gap failure mode (foreign tickers `.PA/.TO/.V/.SI/.WA/.MC`, newly-listed equities). Operator decision pending to activate.

### Supporting fixes (this arc)
- **Catalog poisoning fix** — SecMaster guard against extraction-emitted symbols being self-registered + soft purge of 365 LoRA-era rows registered as instruments (PR #330).
- **`ticker_in_quote` bypass guard** — closes the code path that skipped CoVe grounding (PR #331).
- **Vector similarity floor (default 0.5)** in SecMaster RAG so nearest-neighbor returns null instead of "closest" under that floor (PR #332).
- **SecMaster perf + observability cleanup** — embedding cache, metric split (`vector_search_duration`), suffix dedup, method-tag on requests (PR #325).
- **SecMaster Resolve & RAG latency dashboard** (PR #324).
- **Finnhub upstream circuit breaker** — bounds the 15s-timeout cost on upstream 403s (PR #326).
- **Supervisor-mode skill** — default parallel code dispatches to worktree isolation (PR #327).
- **Docs** — pipeline diagram + Phase 6/7 plan additions (PRs #315/#316).
- **README sweep** — 22 READMEs touched, closing 40 audit findings (PRs #333/#334).

## EPIC 1 — DONE [merged 2026-05-10 as `a08b806`, PR #215]
- 14 stories merged: 1.1.1, 1.1.2, 1.2.1, 1.2.2, 1.3.1, 1.3.2, 1.4.1, 1.4.2, 1.4.3 (destructive migration, 4 phases), 1.5.1, 1.6.1, 1.6.2, 1.7.1, 1.7.2.
- 2 review-fix passes (29 + 9 commits) + obs-fix sweep (3 commits) + deferred suggestions (2 commits).
- New skill landed mid-epic: `readme-consistency` (wired into supervisor-mode REVIEW_FIX_LOOP); 22 README updates across 22 projects.
- **Tests at merge:** SecMaster 744 / FredCollector 243 / OfrCollector 144 / SentinelCollector 664 = **1,795 unit tests, all green**.
- **Schema:** removed `instruments.sector|industry` + `extracted_observations.sector`; added SecMaster tables `naics_codes`, `atlas_sectors`, `naics_sector_rollup`, `mapping_versions`, `signal_identities`, `edgar_filers`, `instrument_sector_overrides`; added cols `naics_code|atlas_sector_code|rollup_version_id|classification_source|classified_at` (instruments), `naics_code|atlas_sector_code|mapping_version_label|signal_identity_id` (FRED), `signal_identity_id` (OFR HFM/STFM), `atlas_sector_code` (Sentinel `extracted_observations`).
- **New types:** `AtlasSectorCode` enum (Events), `ClassificationSource` + `SignalIdentityCategory` enums (SecMaster), `IdentifierQuery` factories. Embedding template v2→v4 (re-embed automatic on deploy via staleness check).

## EPIC 2 — DONE [merged 2026-05-10, PRs #221–#229]
- 9 stories: 2.1.1 (TE schema drop `category`) → 2.1.2 (TE code path) → 2.2.1 (`sectorWeights` on pattern schema) → 2.2.2 (cell projection) → 2.3.1 (multi-output rejection + linter) → 2.4 (per-sector score aggregator) → 2.5 (burst-window + read-time dedup) → 2.6.1 (retire macro-score + 6-state regime, Epic 2 complete).
- Followups: PR #223 (2.1.2 review findings), PR #231 (retire `MacroRegime`/`ApplicableRegimes`/`GetByRegimeAsync` residual), PR #232 (dashboard panels + alerts for cell/sector-score/dedup counters).
- Net: TE evaluates pattern → cell → per-sector score. No more macro-score, no `PatternCategory`, no `MacroRegime`.

## EPIC 3 — DONE [merged 2026-05-11, PRs #237, #239, #240, #242]
- 4 stories: 3.1.1 (MacroSubstrate scaffolding + `macro_observations` DDL, new project Option 2) → 3.1.3 (versioned-mapping query support) → 3.2.1 (FRED vertical slice, UNRATE) → 3.2.3 (OFR HFM+STFM write path).
- Owning-project decision ratified 2026-05-10: **Option 2 = new `MacroSubstrate` project** (cross-DB topology killed Option 1). All 7 Epic 3 open risks resolved in the implementation PRs.
- Migrator runs Variant A (one-shot init container `migrate-macro-substrate` wired in `compose.yaml.j2:66-83`).
- FredCollector parity fix PR #241 (DataCollection + Backfill `ActivitySource` OTEL registration).

## EPIC 4 — IN FLIGHT [Phase 2 LLM track — only 4.6.x pending]
- ✓ 4.1.1 Sentinel exits scoring (PR #243) — deleted `MultiSignalGate` + `CorroborationScanner` + `QuoteFidelity`.
- ✓ 4.3.1 Macro routing (PR #244) — Sentinel routes macro extractions to `macro_observations`.
- ✓ 4.4.1 Qualitative extraction prompt + schema + service (PR #245).
- ✓ 4.4.2 Qualitative persistence into `macro_observations` (PR #246).
- ✓ 4.4.3 CoVe verification layer over qualitative extraction (PR #247).
- ✓ Qualitative pipeline dispatch wired in `ExtractionProcessor` (PR #248, validation-content branch).
- ✓ 4.5 Sentinel subscription to sector-keyed events + narrative search per sector (PR #252).
- ✓ 4.6.1 D15 acceptance criteria pinned into LoRA training metadata (PR #234).
- ✓ 4.6.2 Eval harness + substrate builder + mock baseline scorecard (PR #236).
- ◯ 4.6.x LoRA reliability followups — multi-week, dispatched separately (recon in flight).

## EPIC 5 — DONE [matrix-storage MVP, PRs #249, #250, #253, #254, #255]
- ✓ F5.1 `matrix_cells` hypertable persistence (PR #253).
- ✓ F5.4 per-sector regime classifier + `sector_regimes` time-series (PR #254).
- ✓ F5.5.1 `SectorThresholdCrossedEvent` contract (PR #249).
- ✓ F5.5.2 TE emission of `SectorThresholdCrossedEvent` + per-sector hot-reloadable threshold config (PR #250).
- ✓ F5.5.3 sector × phase derived view (materialized view + 7-day refresh worker, PR #255).
- ✓ Sector-threshold admin REST endpoint (PR #256).

## EPIC 6 — IN FLIGHT [Phase 4 surfaces, PRs #257–#261]
- ✓ 6.1.1 `macro_observations` REST + MCP tool (PR #257).
- ✓ 6.1.2 `matrix_cells` REST + MCP tool (PR #258).
- ✓ 6.1.3 `sector_regimes` + `sector_phase_cells` REST + MCP tools (PR #259).
- ✓ Post-Epic-6 cleanup — M1 catch-all parity + OCE rethrow + worker test stability + DOC ANTI scrub (PR #260).
- ✓ 6.2.1 sector × phase matrix primary dashboard (PR #261).
- → 6.2.x next dashboard — background agent on `epic/6.2.2-trajectory-dashboard` (sector score & regime trajectory).
- ◯ 6.2.x remaining dashboards + 6.3.x reports — pending.

## DEPLOYMENT TODOS
_All long-standing items cleared 2026-05-16: SecMaster cross-DB conn-string + Fred/OFR `SECMASTER_REST_ENDPOINT` resolved in PR #279 (2026-05-14, predated this audit); devcontainer↔timescaledb network bridged for 4 affected services (CalendarService, Reports, FinnhubCollector, AlphaVantageCollector) in PR #336._

## OPEN ASKS (for user)
- **Phase 5 readiness recon** — gates AutoApprove re-engagement. Pending: re-run after Phase 4 backfill drains a meaningful post-all-fixes cohort. Decision needed: flip `Extraction__AutoApproveEnabled=true` once true grounding rate ≥90%.
- **Phase 6 / Phase 7 flag activation** — both shipped default-OFF. Operator decision on enabling statement-level CoVe validation (#329) and Gemini Google Finance fallback (#328); recommend after Phase 5 is stable.
- **TEI bge-m3 GPU experiment** — optional, user authorized as "willing to test". Verify whether GPU-served bge-m3 (via TEI) materially beats CPU bge-m3 baseline (~216ms p50 / ~1,474ms p95).
- **Finnhub API key rotation** — deferred until the system validates; circuit breaker (PR #326) bounds the upstream 403 cost in the meantime.

## DEFERRED FOLLOW-UPS (non-blocking)
- **`vector_search_duration_ms` sunset** — renamed to `vector_search_duration` in PR #325; old name still in Prom retention window until it ages out.
- **Embedding cache hit-rate alerting** — `secmaster_embedding_cache_hits_total` / `secmaster_embedding_cache_misses_total` exist; live hit rate ~57%. If the ceiling stalls, tune target / add a stall-detection alert.
- **readme-consistency** sub-component template detection — audit.sh applies 10-section check uniformly; should detect `*/mcp/`, `*/config/`, `*/scripts/` and apply lighter template.
- **readme-consistency** S5 broadening — current regex misses `IConfiguration` parameter-named accesses (e.g. `configuration["FOO"]` lowercase); broaden to catch any `IConfiguration`-typed local.
- **readme-consistency** S5 `:` vs `__` config-key equivalence — .NET treats them as same; literal-token comparison flags both as separate.
- **readme-consistency** Jinja port resolution — S6 currently skips `{{ ports_external.X }}` templated ports; future enhancement could resolve against `deployment/ansible/group_vars/all.yml`.
- **AtlasSectorCodes** compile-time exhaustiveness — `switch` expression with no default arm (CS8509 catches new members); pin if a new sector is added.

## CONTEXT
- Canonical product plan: `docs/atlas-matrix-mvp-plan.md`
- Handoff (philosophy): `docs/atlas-matrix-handoff-v2.md`
- Execution meta-plan: `~/.claude/plans/we-are-going-to-purrfect-sedgewick.md`
- Plans index: `docs/plans/atlas-matrix/README.md`
- Acceptance criteria (ratified 2026-05-10): `docs/llm/ACCEPTANCE_CRITERIA.md`

## SUPERVISOR NOTES
- All implementation/review/validation goes to subagents per CLAUDE.md SUPERVISOR_MODE.
- Templates dir `/home/james/ATLAS/.claude/skills/supervisor-mode/templates/` with `story-implementation.md`.
- supervisor-mode skill v2 + readme-consistency skill both wired; supervisor-mode references readme-consistency in REVIEW_FIX_LOOP suite.
- supervisor-mode skill discipline tightened (PRs #217 + #218): TOUCHES vs AUTHORS distinction (≤30-line cap), `NEVER_READ` list, `run_in_background=true` default for impl dispatches, `agent_returns_BLOCKED → ntfy_publish + end_turn`, RATIONALIZATION_TABLE + RED_FLAGS.
- NTFY channels: publish to `atlas-claude-ask`, poll `atlas-claude-reply`. WAKEUP_STEP_0 is poll-first.
- Race-guard discipline: selective `git add -- <paths>` throughout; no recent regression.
