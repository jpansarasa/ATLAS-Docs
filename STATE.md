# ATLAS Supervisor STATE [2026-05-16]

## ACTIVE [2026-05-17]
- **CoD reframe + DSL workstream** is the upstream-of-everything beat per planning doc §7 priority order. Sentinel extraction quality is bottlenecked by CoD; CoD output is being reframed from English prose to a structured symbolic DSL. Canonical: `docs/plans/sentinel-edge-realization.md` (§6 v1 spec + 8 locked decisions) + `docs/plans/sentinel-edge-realization-drift.md` (per-stage drift).
- **2026-05-17 shipped:** §6 spec + decisions (PRs #347/#348), Python parser + 21 tests (PR #349), per-class CoD prompts (PR #350), Round 2 benchmark harness (PR #351). Also today: WhisperServiceMcp consolidation (#353), `scripts/` reorg (#356, #358), `agent/` → SentinelCollector (#360), `sentinel-mcp/` → `ntfy-mcp/` (#361), atlas-matrix doc sweep (#359), 6 stale-doc deletions (#354/#355).
- **In flight:** Round 1 CoD prose-baseline benchmark (9 models, ~80% done). Round 2 DSL benchmark kickoff is gated on Round 1 completion (sequential — Ollama CPU runners would contend). Also in flight: Round 2 model roster research (Gemma 4 + Qwen3.6 + qwen3:30b candidates).
- **Runtime posture:** Sentinel ingest healthy (250-1000 rows/hr); AutoApprove disciplined (5-18/hr, 100% with subject_entity + instrument_id); Phase 6/7 wired but silent metrics (planning doc §5 gaps 3+4 still open).
- **Older arc (symbol-identification remediation, 2026-05-15→16)** retained below as supporting history — phases completed but the "ACTIVE" focus has moved upstream to CoD.

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
