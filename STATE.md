# ATLAS Supervisor STATE [2026-05-11]

## ACTIVE [2026-05-11]
- **Phase 2:** Epic 1 + Epic 2 + Epic 3 substrate triad ✓ DONE. Epic 4 LLM track running end-to-end on validation-content via `ExtractionProcessor`.
- **Latest merges (this session):** PR #246 (4.4.2 qualitative persistence), PR #247 (4.4.3 CoVe verifier), PR #248 (qualitative dispatch in ExtractionProcessor), PR #249 (Feature 5.5.1 `SectorThresholdCrossedEvent` contract).
- **In flight:** Feature 5.5.2 — TE emission of `SectorThresholdCrossedEvent` (per-sector threshold config, hot-reloadable). Background agent on `epic/5.5.2-sector-threshold-emission`.
- **Next after 5.5.2:** Story 4.5 — Sentinel subscription to sector-keyed events + narrative search per sector.
- **Hook redesign landed** (PRs #230 + #238): tree-hash keyed, per-worktree, format-versioned markers at `/tmp/atlas-test-markers/`. Survives commit-after-test + cherry-pick + rebase.
- **Recurring multi-agent race:** parallel agents in the same toplevel share the marker file (`sha1(toplevel)`); mitigate by sequencing pushes between parallel branches or using `isolation: "worktree"`.

## PHASE STATUS
- **Phase 1** — Spine — ✓ DONE (Epic 1 merged as `a08b806`)
- **Phase 2** — Substrate / TE schema / LLM track — → IN FLIGHT
  - Epic 2 (TE schema rework) — ✓ DONE (sectorWeights + cell projection + macro-score retire)
  - Epic 3 (`macro_observations` substrate) — ✓ DONE (MacroSubstrate triad + FRED/OFR dual-write + Sentinel macro routing)
  - Epic 4 LLM track — qualitative pipeline live end-to-end (4.4.1 + 4.4.2 + 4.4.3 + #248 dispatch); 4.5 pending F5.5.2
  - Epic 4 Feature 4.6 (LoRA reliability) — D15 ratified 2026-05-10; 4.6.1 dispatched separately (multi-week)
- **Phase 3** — Sentinel rework + matrix — partially advanced via Epic 4 wiring; Epic 5 in flight (F5.5.1 ✓; F5.5.2 → background; F5.5.3 pending)
- **Phase 4** — Surfaces — ⧗ blocked on Phase 3
  - Epic 6 (MCP, dashboard, reports)

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

## EPIC 4 — IN FLIGHT [Phase 2 LLM track]
- ✓ 4.1.1 Sentinel exits scoring (PR #243) — deleted `MultiSignalGate` + `CorroborationScanner` + `QuoteFidelity`.
- ✓ 4.3.1 Macro routing (PR #244) — Sentinel routes macro extractions to `macro_observations`.
- ✓ 4.4.1 Qualitative extraction prompt + schema + service (PR #245).
- ✓ 4.4.2 Qualitative persistence into `macro_observations` (PR #246).
- ✓ 4.4.3 CoVe verification layer over qualitative extraction (PR #247).
- ✓ Qualitative pipeline dispatch wired in `ExtractionProcessor` (PR #248, validation-content branch).
- ✓ 4.6.1 D15 acceptance criteria pinned into LoRA training metadata (PR #234).
- ✓ 4.6.2 Eval harness + substrate builder + mock baseline scorecard (PR #236).
- ◯ 4.5 Sentinel subscription to sector-keyed events + narrative search per sector — blocked on Epic 5 F5.5.2.
- ◯ 4.6.x LoRA reliability followups — multi-week, dispatched separately.

## EPIC 5 — IN FLIGHT [matrix / sector-keyed events]
- ✓ F5.5.1 `SectorThresholdCrossedEvent` contract (PR #249).
- → F5.5.2 TE emission of `SectorThresholdCrossedEvent` — background agent on `epic/5.5.2-sector-threshold-emission`.
- ◯ F5.5.3 pending (matrix integration step — unblocks Epic 4 Story 4.5).

## DEPLOYMENT TODOS
- Wire `ConnectionStrings__AtlasData=...atlas_data...` into the **SecMaster** container in `deployment/artifacts/compose.yaml.j2` + ansible vault (Story 1.7.2 dedup-grouping needs cross-DB raw-SQL access). Migrator already has it (line 74); SecMaster still only has `ConnectionStrings__SecMaster` (line 619). Could not verify ansible-vault contents (no vault password in scope).
- Wire `SECMASTER_REST_ENDPOINT` env var into **FredCollector + OfrCollector** containers (Stories 1.6.1/1.6.2). Code reads it (`FredCollector/src/DependencyInjection.cs:112,147`, `OfrCollector/src/DependencyInjection.cs:108`); compose only sets `SECMASTER_GRPC_ENDPOINT`.
- Bridge devcontainer↔timescaledb network (recurring blocker for integration tests across 4 stories — long-lived infra debt independent of any single epic).

## OPEN ASKS (for user)
- Eval substrate decision (Feature 4.6.x) — v6.2-cove (70,895) vs. subset. Not yet settled.

## DEFERRED FOLLOW-UPS (non-blocking)
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
- Epic 3 owner doc: `docs/plans/atlas-matrix/epic-3-owner-recommendation.md`

## SUPERVISOR NOTES
- All implementation/review/validation goes to subagents per CLAUDE.md SUPERVISOR_MODE.
- Templates dir `/home/james/ATLAS/scripts/agent-prompts/` with `story-implementation.md`.
- supervisor-mode skill v2 + readme-consistency skill both wired; supervisor-mode references readme-consistency in REVIEW_FIX_LOOP suite.
- supervisor-mode skill discipline tightened (PRs #217 + #218): TOUCHES vs AUTHORS distinction (≤30-line cap), `NEVER_READ` list, `run_in_background=true` default for impl dispatches, `agent_returns_BLOCKED → ntfy_publish + end_turn`, RATIONALIZATION_TABLE + RED_FLAGS.
- NTFY channels: publish to `atlas-claude-ask`, poll `atlas-claude-reply`. WAKEUP_STEP_0 is poll-first.
- Race-guard discipline: selective `git add -- <paths>` throughout; no recent regression.
