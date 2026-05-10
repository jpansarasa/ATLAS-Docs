# ATLAS Supervisor STATE [2026-05-10]

## ACTIVE
- **Epic:** 1 — SecMaster Sector Taxonomy & Classification — **MERGED to main**
- **Phase:** 1 (spine) — ✓ DONE; Phase 2 unblocked (Epic 2, Epic 3, Epic 4 Feature 4.6)
- **Branch:** `main` (squash-merge `a08b806`); local epic branch can be deleted
- **PR:** https://github.com/jpansarasa/ATLAS/pull/215 — **MERGED**
- **Story 2.1.2 ✓ MERGED** (PR #222 → `ee5f7c5` on main, 2026-05-10). Review suite landed 3 critical + ~6 important findings; fix loop ran on the original branch but DNS hiccup caused merge-before-push of the fix commit.
- **Recovery in flight:** fix commit `4381068` cherry-picked onto branch `fix/te-2.1.2-review-followup`. Build-agent re-testing for hook marker; will push + open follow-up PR + merge once green.
- **Out-of-scope deferrals (still on 2.6.1 punch list):** category= tags on gauges/meters in `PatternEvaluationService` + `ThresholdEngineMeter`, `GetPatternContributionsAsync` + DTOs, `MacroScoreCalculator` + `RegimeTransitionDetector` + `PatternCategory` enum deletion.
- **Next:** Story 2.2.1 (`sectorWeights` on pattern schema) dispatches after fix PR merges (avoids conflict on `PatternConfiguration.cs`/`PatternEvaluationResult.cs`).
- **Recurring infra issue:** git-push-guard marker is keyed by exact commit hash; agents that compile-then-commit, or any cherry-pick/rebase, force a re-test cycle. 3rd hit this session — flag for future hook redesign (commit-keyed marker placement should happen post-commit, not pre).

## EPIC 1 STORY QUEUE — all done
1. ✓ 1.1.1 NAICS taxonomy import (commits dac5b95→fa46b29)
2. ✓ 1.1.2 NAICS lookup API (adacc53→14bc804; gRPC deferred)
3. ✓ 1.2.1 Initial 11-sector rollup mapping (2f047b4→c52fd84; 1012 codes)
4. ✓ 1.2.2 Rollup lookup API (a50326f→41bcc67)
5. ✓ 1.3.1 Mapping versioning schema (7dc330b→c7c3abf)
6. ✓ 1.3.2 Version-aware lookup (129a1f4→a5f5cf9; v1.1 demo green)
7. ✓ 1.5.1 OpenFIGI verify (b951f08→c5b4edc)
8. ✓ 1.7.1 Signal identity catalog (e7a7f5b→b29e12b; 70 signals)
9. ✓ 1.7.2 Dedup grouping API (3a5635d→6969a8d)
10. ✓ 1.4.1 EDGAR ingestion (e0d4b83→09e5bac; ~200 SIC codes)
11. ✓ 1.4.2 Manual override mechanism (02b848e→84a4019)
12. ✓ 1.6.1 Industry-aggregate series tagging (9fede4d→0e10bc1)
13. ✓ 1.6.2 Pure-macro signal identity (ba7fa1e→e48bd63)
14. ✓ 1.4.3 Destructive migration (PS3) — 4 phases (568d744→0d365ed)

## EPIC 1 REVIEW-FIX SWEEPS — all done
**1st pass** (29 commits across D1-D5 + Obs-fix + Layer 2 enum cascade):
- ✓ D1: Type-design hardening (4f4f42e, a16071b, ff56444)
- ✓ D2: Silent-failure fixes (708de60→e677005)
- ✓ D3: Comment rot + missing test (04109f7→09832af)
- ✓ Obs-fix B+C+D (19fa445, 8657348, 6550df8)
- ✓ D4: Test gaps + comment fix (dfec7c2→63ea8cd)
- ✓ D5: Story-ID sweep + ClassificationSource enum (bc3463a, 7c84469)
- ✓ Layer 2: AtlasSectorCode enum cascade (83f51e3→d5b7a4d, 4 stages)

**2nd pass** (9 commits across Dispatch A + B):
- ✓ Dispatch A: 4 critical doc fixes + ToDbString switch + lift converters to Events.EntityFrameworkCore + Sentinel parse counter + migration prod-data check + importer failure counter + 2nd story-ID sweep + Down() data-loss remarks (bc09acc, 6e84fb1, 35ef674, 870f7fb)
- ✓ Dispatch B: EDGAR duration histogram tests + importer narrow-catch tests + AtlasSectorCodes roundtrip tests + IdentifierQuery factories + ClassificationSource converter file + AtlasSectorDisplayResolver per-code log (b28ccec, df541c0)

**2nd-pass obs review fix** (3 commits):
- ✓ SignalIdentityCategory enum + boundary validation (16652a3)
- ✓ Backfill cycle catch logs Warning not Error (b6a9e7f)
- ✓ Sentinel parse-unknown SetTag + display-load AddEvent (6c07967)

**Deferred 1st-pass suggestions** (2 commits):
- ✓ Cardinality verification tests across 4 meters (aee60a6, +20 tests)
- ✓ Timeout/cancellation tests on EDGAR/SecMasterClient/FredSeriesSectorResolver (768b568)

## NEW SKILL LANDED — readme-consistency
- 16-task plan executed via subagent-driven-development
- Skill at `.claude/skills/readme-consistency/` with SKILL.md + TEMPLATE.md + audit.md + bash scripts + fixtures + 14 tests
- Wired into supervisor-mode REVIEW_FIX_LOOP suite + TEMPLATE_LIBRARY
- Smoke run against branch: 92 findings (13 CRITICAL / 16 HIGH / 63 MEDIUM)
- Phase 3 fix dispatch landed 22 README updates across 22 projects (CRITICALs + HIGHs + top services' MEDIUMs all resolved)
- 9 CRITICAL skipped (config/scripts dirs that don't warrant full READMEs)

## EPIC 1 EXIT STATUS
- **Branch:** 100 commits ahead of main
- **Tests (latest, all 0/0/all green):** SecMaster 744 / FredCollector 243 / OfrCollector 144 / SentinelCollector 664 = **1,795 unit tests**
- **Build:** clean across all 4 .NET projects
- **Schema state:**
  - Removed: `instruments.sector`, `instruments.industry` (+ 2 indexes); `extracted_observations.sector`
  - Added (SecMaster): `naics_codes`, `atlas_sectors`, `naics_sector_rollup`, `mapping_versions`, `signal_identities`, `edgar_filers`, `instrument_sector_overrides`; `instruments` cols `naics_code` + `atlas_sector_code` + `rollup_version_id` + `classification_source` + `classified_at`
  - Added (FredCollector): `naics_code` + `atlas_sector_code` + `mapping_version_label` + `signal_identity_id`
  - Added (OfrCollector): `signal_identity_id` on HFM/STFM series
  - Added (Sentinel): `atlas_sector_code` on `extracted_observations`
- **New shared packages:** `Events.EntityFrameworkCore` (lifted enum value-converters)
- **New types:** `AtlasSectorCode` enum (Events), `ClassificationSource` enum (SecMaster), `SignalIdentityCategory` enum (SecMaster), `IdentifierQuery` factories
- **Embedding template:** v2 → v4 (re-embed scheduled automatically post-deploy via staleness check)

## DEPLOYMENT TODOS (gate Epic 1 → main merge)
- Wire `ConnectionStrings__AtlasData=Host=timescaledb;Database=atlas_data;...` into SecMaster container in `deployment/artifacts/compose.yaml.j2` + ansible vault (Story 1.7.2 dedup-grouping needs cross-DB raw-SQL access).
- Wire `SECMASTER_REST_ENDPOINT` env var into FredCollector + OfrCollector containers (Stories 1.6.1/1.6.2).
- Bridge devcontainer↔timescaledb network (recurring blocker for integration tests across 4 stories — long-lived infra debt independent of this epic).
- After deploy: re-embed kicks automatically (template v3→v4 staleness check); monitor `embedding_background_service` Loki logs for completion.

## OPEN ASKS (for user)
- D15 ratification: `docs/llm/ACCEPTANCE_CRITERIA.md` DRAFT → RATIFIED (blocks Phase 2 Feature 4.6 only).
- Eval substrate decision (Feature 4.6) — v6.2-cove (70,895) vs. subset.
- Epic 3 owning-project decision — ✓ ratified Option 2 (new `MacroSubstrate` project). Risks 1–7 in `docs/plans/atlas-matrix/epic-3-owner-recommendation.md` still need pre-Story-3.1.1 ratification.

## PHASE 2 KICKOFF (decided 2026-05-10)
- Strategy: **parallel branches** Epic 2 + Epic 3 (Feature 4.6 deferred until D15 ratified).
- Branches on origin from `main@a08b806`:
  - `epic/2-te-schema-rework` — TE track (sectorWeights + cell projection + macro-score retire)
  - `epic/3-macro-observations` — substrate track (owning-project pending user ratification)

### Recon outputs (2026-05-10)
- **Epic 3 owning-project** — Plan-agent recommends **Option 2 (new `MacroSubstrate` project)**.
  - Doc: `docs/plans/atlas-matrix/epic-3-owner-recommendation.md` (DRAFT, awaiting user ratification).
  - Decisive constraint: cross-DB topology kills Option 1 (substrate must live in `atlas_data`, not `atlas_secmaster`).
  - 7 open risks worth pre-ratifying — see doc §"Open risks" (variant A/B migrator, MacroSubstrate.Client, soft FK, D4 CHECK constraints, idempotency key, retention default, devcontainer bootstrap).
  - Spot-checks ✓: Events triad / FredObservationSourceProvider / AddInstrumentEmbeddings / AddEventsTable all exist as cited.

- **Epic 2 Story 2.1 blast radius** — audit-recon (read-only Explore agent).
  - Report: `/tmp/sentinel-remediation/epic2-story-2.1-blast-radius.md` (235 lines).
  - 14 plan-cited files ✓ verified at expected line numbers.
  - **Plan understated by ~3-4 files** — adds `ThresholdEngine/mcp/Tools/ThresholdEngineTools.cs` (HIGH severity; MCP `evaluate()` + `list_patterns()` group-by-category), `tests/Workers/DataWarmupServiceTests.cs`, `tests/Endpoints/PatternEndpointsTests.cs`, `tests/Data/ThresholdEventRepositoryTests.cs`.
  - Dashboard `deployment/artifacts/monitoring/dashboards/thresholdengine.json` flagged as needing inspection (recommend pre-dispatch grep).
  - **One audit error** — claimed `ThresholdEngine/config/patterns/` directory "does not exist"; it does, with 79 JSON files in 11 subdirectories. Other findings hold.
  - No cross-service references to `category | PatternCategory | MacroScore | MacroRegime` (verified in Sec​Master, FredCollector, OfrCollector, FinnhubCollector, AlertService, CalendarService).

## DEFERRED FOLLOW-UPS (non-blocking)
- **readme-consistency** sub-component template detection — audit.sh applies 10-section check uniformly; should detect `*/mcp/`, `*/config/`, `*/scripts/` and apply lighter template.
- **readme-consistency** S5 broadening — current regex misses `IConfiguration` parameter-named accesses (e.g. `configuration["FOO"]` lowercase); broaden to catch any `IConfiguration`-typed local.
- **readme-consistency** S5 `:` vs `__` config-key equivalence — .NET treats them as same; literal-token comparison flags both as separate.
- **readme-consistency** Jinja port resolution — S6 currently skips `{{ ports_external.X }}` templated ports; future enhancement could resolve against `deployment/ansible/group_vars/all.yml`.
- **AtlasSectorCodes** compile-time exhaustiveness — currently uses `switch` expression with no default arm (CS8509 catches new members); already addressed in Dispatch A but worth pinning if a new sector is added.

## PHASE STATUS
- **Phase 1** — Spine — ✓ DONE (Epic 1 merged to main as `a08b806`)
- **Phase 2** — Substrate / TE schema / LLM track — → IN FLIGHT
  - Epic 2 (TE schema rework) — → Story 2.1.1 background agent in flight on `epic/2-te-schema-rework`
  - Epic 3 (`macro_observations` substrate) — → Story 3.1.0 ✓ done (PR #219); 3.1.1 awaiting risk ratification
  - Epic 4 Feature 4.6 only (LLM reliability track) — ⧗ blocked on D15 ratification
- **Phase 3** — Sentinel rework + matrix — ⧗ blocked on Phase 2
  - Epic 4 Features 4.1–4.5
  - Epic 5 (matrix storage, vector queries)
- **Phase 4** — Surfaces — ⧗ blocked on Phase 3
  - Epic 6 (MCP, dashboard, reports)

## CONTEXT
- Canonical product plan: `docs/atlas-matrix-mvp-plan.md`
- Handoff (philosophy): `docs/atlas-matrix-handoff-v2.md`
- Execution meta-plan: `~/.claude/plans/we-are-going-to-purrfect-sedgewick.md`
- Plans index: `docs/plans/atlas-matrix/README.md`
- Acceptance criteria draft: `docs/llm/ACCEPTANCE_CRITERIA.md`

## SUPERVISOR NOTES
- All implementation/review/validation went to subagents per CLAUDE.md SUPERVISOR_MODE.
- Templates dir `/home/james/ATLAS/scripts/agent-prompts/` exists (created Epic 1) with `story-implementation.md`.
- supervisor-mode skill v2 + readme-consistency skill both wired in; supervisor-mode references readme-consistency in REVIEW_FIX_LOOP suite.
- supervisor-mode skill discipline tightened mid-session (PRs #217 + #218): TOUCHES vs AUTHORS distinction (≤30-line cap), `NEVER_READ` list, `run_in_background=true` default for impl dispatches, `agent_returns_BLOCKED → ntfy_publish + end_turn`, RATIONALIZATION_TABLE + RED_FLAGS sections. Driven by live failures observed in this session.
- NTFY channels: publish to `atlas-claude-ask`, poll `atlas-claude-reply`. WAKEUP_STEP_0 is poll-first.
- Race-guard discipline practiced: selective `git add -- <paths>` throughout; one race incident early in session (resolved by reset+recommit), no regression since.
