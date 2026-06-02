# ATLAS Supervisor STATE

Supervisor memory. Read first, write last, each turn. **Current truth + pointers only** — arc/iteration history lives in git log + phase tags (see ARCHIVE), NOT here. Keep it lean so the live numbers don't get buried. **Maintenance rule: VERIFY each status against code/config/DB before writing it — never copy-forward inferred status.**

## EACH TURN START HERE
0. **FIRST TURN OF SESSION ONLY** — git audit before any dispatch: `git worktree list`, `git branch`, `git status`, `git fetch origin && git log --oneline HEAD..origin/main`. Flag stale worktree-agent branches, detached HEADs, dirty main, behind-main drift. Safe cleanup direct; destructive (worktree remove, branch/stash delete) needs user approval. Per [[feedback_git_session_start_discipline]] + [[feedback_git_verify_each_step]] + [[feedback_main_always_current]].
1. Poll NTFY `atlas-claude-reply` (user input first).
2. RE-READ the active driver — `docs/atlas-matrix-realignment-brief.md` §6/§7 + plan `~/.claude/plans/harmonic-wobbling-manatee.md` + this file. ("RE-READ", not "I read it earlier".) DSL-PoC pipeline plan `docs/plans/atlas-dsl-poc-plan.md` §8 = completed-pipeline reference.
3. Walk the pipeline backward from this turn's target before dispatching.
4. Plan vs production mismatch → STOP + NTFY architectural, don't dispatch.

Dispatching without re-grounding → [[feedback_walk_pipeline_each_turn]]. Citing benchmark scores as production capability → [[feedback_benchmark_not_production]].

**Canonical pipeline (deployed — verified 2026-06-02):**
SentinelCollector → CPU CoD extraction (qwen3:30b-a3b on `llama-server`, v8 prompt + v2.3 GBNF) → deterministic verifier (`dsl-parser-mcp`, v2_3_1) → `DslToMergedExtractionAdapter` → GPU semantic tier (`EntityResolver` cascade incl SecMaster name-resolve + self-seed; `ClaimVerifier` on vllm-qwen2.5:32b; `SectorTagger` NAICS on vllm) → `SignalFilter` (§8.2 gate) → gRPC `MatrixCellEnrichment` → ThresholdEngine `MatrixCellSentinelWorker` → `matrix_cells`. TE applies `sectorWeights` to render the macro×sector matrix. **NB: this is still the legacy per-article "Path 2"; the Option-A `macro_observations` cutover is decided but not built (see WS3 DECISION STATUS / D4).**

## WHERE WE ARE NOW [2026-06-02]
Two arcs since the 2026-05-28 baseline, both landed:

- **0-cells blocker chain — RESOLVED.** All wiring/config gaps AND the previously-parked precision decisions were built + merged (PRs #552–#571): claim delivery (#552), evidence-span + real object (#554), resolver name-route (#555), sector→GPU vllm (#556), CPU throughput / scoped restart + cpuset (#557), Fix-A SecMaster self-seed (#558/#559), RAG-tier bound (#560), CLAIM-subject ENT-ref + ticker-collision (#561/#562), §8.2 Gate-B provenance-anchored evidence for paraphrased objects (#563/#565), catalog-sector bind (#564), polarity threading (#566/#567), retired benchmark allowlist gate (#568), parallel breadth-first discovery + lazy sector classify (#571). Pipeline produces real cells end-to-end.
- **WS3 Matrix Realignment — ACTIVE; matrix RENDERS.** Verified in prod (DB `matrix_cells` 2026-06-02): **198 cells, 178 non-zero, latest 2026-06-01.** Pattern set realigned (cut-list, signal-identity catalog with temporal/decay, canonical-id reconcile) and **blessed `sectorWeights` populated → real signed values** (#586: 40 patterns, 7 sign-inversions fixed). Full context: `docs/atlas-matrix-realignment-brief.md` (UNCOMMITTED — see DEFERRED).
- ⚠ **Cells are NOT yet signal-identity-keyed** — verified: `count(DISTINCT signal_identity_id) = 0` across all 198 cells. The Option-A cutover (TE scores from `macro_observations`) is decided + has additive groundwork (#585) but is not live; legacy Path-2 still writes every cell. See D4 below.

## CPU CoD EXTRACTION QUALITY — VALIDATED GOOD  [reference — read carefully, do NOT call it low]
The cutover proceeded *because* extraction is good. Three scoring conventions (this is what I kept conflating):
- **scored / pooled** = quality on cells that produced output → the TRUE extraction quality (~0.85–0.89).
- **penalized** = empty-doc / parse-fail cells counted as 0 → UNDERSTATES (corpus edge cases drag it down).
- **n=9** = small Arm-B subset → OVERSTATES scale (v8 was 0.907 on n=9 but 0.7826 penalized on n=72). Always cite **n=72**.

Authoritative n=72 figures:
- **ABSOLUTE SOTA = v2.3 stochastic, pooled 0.8733.**
- β v12.1: scored **0.887** / penalized 0.8257.
- v15 (word + chunked) T=0: **0.8261** — deterministic ceiling.
- v8 baseline: penalized **0.7826** — lowest mature prompt.
- n=9 trajectory (subset, not scale): v3 0.790 → v4 0.808 → v5 0.782 → v6 0.895.

**Production recommendation (prior benchmarking): v15 leading, v2.3 runner-up.**
**Deployed in prod: v8 baseline prompt + v2.3 GBNF.** Diagnostic confirms v8 already emits `source_entity` and v2.3 grammar permits it → the prompt is NOT the yield problem. (Upgrading the deployed prompt to the v15/v2.3 family is a separate, optional gain — not a fix.)

## WS3 REALIGNMENT — DECISION STATUS [VERIFIED 2026-06-02 against code/config/DB]
Brief §7 decisions:
- **D1 sectorWeights methodology** — ✅ **DONE.** Methodology drafted (#579), blessed weights populated (#586); DB confirms 178/198 non-zero cells (was all-zero). 
- **D2 cut/merge/split list** — ✅ **DONE** (#580 cut-list + #581 ValidationTrigger reconcile). `eur-usd` kept distinct (Phase-0 correction).
- **D3 breakeven trio** — ✅ **DONE: collapsed to ONE** `inflation-expectations` signal. Verified: single `ThresholdEngine/config/patterns/inflation/inflation-expectations.json` (`signalIdentity: inflation-expectations`); no standalone 5y/10y/5y5y pattern files remain.
- **D4 who scores (A vs B)** — ✅ **DECIDED = Option A** (TE scores every source from `macro_observations`), ⚠ **NOT complete.** Verified: 0 cells carry `signal_identity_id`; `ObservationCellProjector` does not exist; legacy Path-2 files all present (`SentinelCollector/src/Semantic/SignalMagnitudeCalculator.cs`, `Events/src/Events/Protos/matrix_updates.proto`, `ThresholdEngine/src/Workers/MatrixCellSentinelWorker.cs`). #585 (WS3 P2a) added the additive `signal_identity_id` + source-provenance keying but cells aren't populated with it yet. **REMAINING:** Phase-2 shadow projector → Phase-3 cutover (delete Path-2), flag-reversible (plan §Part 4). Confirm whether #585 is deployed (cells dated 06-01 post-date its 05-31 merge yet carry no signal id).

Items the prior STATE carried as "open asks" that were **already resolved** (verified in deployed `/opt/ai-inference/compose.yaml`; all flipped 2026-05-16, before the prior STATE was written):
- **AutoApprove** — ✅ ENABLED (`Extraction__AutoApproveEnabled=true`; gated behind Phase-6/7 validators + CoVE Symbol cascade).
- **Statement-level CoVe (#329)** — ✅ ENABLED (`Extraction__StatementValidationEnabled=true`).
- **Gemini Google-Finance fallback (#328)** — ✅ ENABLED (`Extraction__GeminiFallbackEnabled=true` + `GeminiResolverEnabled=true` + `GeminiAutoRegisterNewInstruments=true`).

## IN FLIGHT [2026-06-02]
- **PR #589 OPEN** — deploy: rebalance CPU cpuset partition (restore embed island capacity). Worktree `agent-a8322bae…` (locked).
- **WS3 D4 cutover** — remaining implementation (decided Option A), not a user-ask. Next concrete step: confirm #585 deploy + build `ObservationCellProjector` in shadow mode.

## REMAINING USER DECISIONS (genuinely open — plan `harmonic-wobbling-manatee.md` §"Open decisions"; NOT re-verified this pass, flag don't assert)
- **Macro score's role** (handoff §8 open-Q1): derive-from-sector-scores / keep-separate-new-weights / retire (its categories are gone).
- **Regime granularity** (§8 open-Q2): per-sector 11 regimes / single global / both; relation to the existing 6-state.
- **Borderline CUTs:** `cu-au-ratio` (valid FRED, `enabled:false`), `baltic-freight-recession` (needs BDIY feed) — enable+realign or drop.
- **`cyclical-gdp-share-declining`** — retire to components or keep as one explicit composite.

## DEPLOY / RUNTIME NOTES
- Scoped per-service restart now EXISTS (#557/#569/#570): freshness-gated stop+rm+up (nerdctl 1.7.7; `--force-recreate` invalid). The old "ansible full restart cycles GPU + starts alert-service" footgun is mitigated — still verify the running digest after deploy ([[ansible_deploy_stale_image]]).
- **alert-service STAYS DOWN** (user; output not useful now).
- `nerdctl images` CreatedSince unreliable for buildkit images — use `inspect .Created`.
- DB inspection: `sudo nerdctl exec timescaledb psql -U ai_inference -d atlas_data` (SELECT only; never DML — [[feedback_agent_no_raw_db_fixes]]).
- Pre-existing (predates WS3): threshold-engine CCSA/ICSA unregistered + a few patterns overdue — re-verify post-#580 cut-list.

## DEFERRED FOLLOW-UPS (non-blocking)
- **Untracked in repo:** `docs/benchmarks/cod-2026-05-17/corpus.jsonl.macro-bak` — commit-or-discard. (The realignment brief is now tracked, landed via PR #588.)
- **Orphan worktree dir** `.worktrees/epic-3-macro-observations/` — NOT in `git worktree list` (stale/unregistered) → investigate + `git worktree prune` / remove.
- Test-hardening: #552 interleaved-blank-subject index-drift; #554 direct `BuildEvidenceWindow` snap/multi-`\n`; #555 `SecMasterNameResolver` wire-shape (stub HttpMessageHandler + canned JSON).
- README-TEMPLATE 6-variant router verification (PR #543).
- Embedding cache hit-rate alerting (~57% live) if ceiling stalls.
- readme-consistency skill: sub-component template detection; `:` vs `__` key equivalence; Jinja port resolution.
- Parking lot: benchmark llama.cpp vs vLLM-on-CPU on existing corpus.

## CONTEXT / POINTERS
- **Active driver:** `docs/atlas-matrix-realignment-brief.md` (decisions) + plan `~/.claude/plans/harmonic-wobbling-manatee.md`; matrix plans `docs/atlas-matrix-mvp-plan.md`, `docs/atlas-matrix-handoff-v2.md`; epic `docs/plans/atlas-matrix/`. Phase-0 coverage: `/tmp/sentinel-remediation/phase0-coverage/`.
- **Completed-pipeline reference:** DSL PoC plan `docs/plans/atlas-dsl-poc-plan.md`; Phase 4 sub-plan `docs/plans/atlas-dsl-poc-phase4-gpu-semantic-verifier.md`.
- Benchmark scores (authoritative): `docs/benchmarks/cod-2026-05-17/results/**/NOTES.md` (v8 / β v12.1 / v2.3 / v15).
- Acceptance: `docs/llm/ACCEPTANCE_CRITERIA.md`. CoD→CoVe harness: `docs/benchmarks/cod-cove-acceptance/` (PR #553).

## SUPERVISOR OPERATING NOTES
- All code/test/build/deploy work → background subagents (never direct). STATE.md / plans / CLAUDE.md = supervisor-owned (annotate ≤30 lines/turn; full rewrites only when directed, as here). STATE.md is *my* document to maintain — verify, don't ask the user to approve it.
- Merge gate marker is set by INVOKING `/pr-review-toolkit:review-pr <N>` (the SKILL), not by dispatching review agents directly. Invoke once before merging a supervisor PR.
- NTFY: publish `atlas-claude-ask`, poll `atlas-claude-reply`. Templates: `.claude/skills/supervisor-mode/templates/`.
- Git: selective `git add -- <paths>`; never `-A`/`-u`/`.` with agents in flight.

## ARCHIVE (recoverable via git log + phase tags — not carried here)
- **0-cells blocker chain (2026-05-28→30):** full CoD→verifier→resolver→sector→matrix wiring saga + parked-decision build-out, PRs #552–#571 (5 handoff gaps + Fix-A self-seed + §8.2 Gate-B paraphrased-object evidence). CoD→CoVe acceptance harness PR #553. Prior STATE overnight diary recoverable via git history of this file.
- **WS3 Matrix Realignment (2026-05-30→31):** PRs #571–#587 — pattern audit/cut-list (#580/#581), signal-identity catalog temporal/decay (#582), canonical-id reconcile (#584), additive matrix-cell keying + provenance (#585), blessed sectorWeights (#586), otel-compliance sweep (#575/#576/#577), ansible pattern-prune (#587).
- **DSL PoC Phases 0–5**: PR chain #381–#510 (Phase 0 instrumentation → Phase 1 disambiguation → Phase 2 GBNF → Phase 3 v2/word-grounding schema → Phase 4 GPU semantic verifier → Phase 5 matrix integration + CPU CoD cutover). Prompt arc v3→v15 + n=72 scale-outs. Tags `dsl-poc-phase{4-7,5}-done`; RELEASES.md.
- **Symbol-identification remediation** (2026-05-15/16): rich RAG embeddings v5 + top-N retrieval + CoVe grounding + Gemini catalog-of-last-resort + similarity floor. Plan `docs/plans/symbol-identification-remediation.md`. (AutoApprove now ENABLED 2026-05-16 — see WS3 DECISION STATUS.)
- **Matrix MVP Epics 1–6** (2026-05-10/11, PRs #215–#261): TE schema rework, MacroSubstrate `macro_observations`, matrix-storage MVP, REST/MCP surfaces, sector×phase dashboards. All merged.
- Push-guard hook fix (PR #388, tree-hash-keyed markers). Supervisor-mode skill v2 + PLAN_GROUNDING + STATE anchor (PR #511).
