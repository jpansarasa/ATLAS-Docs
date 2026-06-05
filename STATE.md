# ATLAS Supervisor STATE

Supervisor memory. Read first, write last. **Current truth + pointers only** — history lives in git log + PRs. Keep lean. VERIFY each status vs code/config/DB before writing.

## EACH TURN START HERE
0. **FIRST TURN ONLY** — git audit: `git worktree list`, `git branch`, `git status`, `git fetch origin && git log --oneline HEAD..origin/main`. Flag stale worktrees / drift; main checkout must be on `main` & current (else routine deploys build stale — see DEPLOY notes). [[feedback_git_session_start_discipline]] [[feedback_main_always_current]]
1. Poll NTFY `atlas-claude-reply` (user input first).
2. RE-READ the canonical diagnosis `docs/sentinel-matrix-break-map-2026-06-05.md` + this file. RE-READ, not "read earlier". [[feedback_read_readme_before_assuming]] (don't theorize a service's behavior — read its README/probe it).
3. Walk the pipeline backward from this turn's target before dispatching; plan-vs-prod mismatch → STOP + NTFY.

## WHERE WE ARE [2026-06-05] — Sentinel→matrix counterweight RESTORATION
The "empty digest" thread bottomed out at a severed pipeline: the CoD/DSL cutover (~05-28/29) + A3 (#601) cut Sentinel's news signal out of the matrix at every layer. **Sentinel's value = SUSTAINED news as a fast-decaying LEADING counterweight to lagging hard data (FRED).** Diagnosis: `docs/sentinel-matrix-break-map-2026-06-05.md`. The matrix is **signal × sector**.

- **SIGNAL dimension — RESTORED + LIVE + verified.** News flows into the matrix as signal-keyed, decaying, accumulating cells. Verified on real data: sustained same-direction coverage accumulates, single/stale decays, heal-in-place each cycle, cells bounded.
- **SECTOR dimension — partially fixed; BLOCKED at SecMaster.** The Sentinel pipeline is now correct, but the breadth doesn't land (see NEXT #1).

## DONE — merged + deployed + verified (2026-06-05)
- **#613 Fix #1 — news→`macro_observations` feed:** per-article vLLM classifier → signal-keyed/sector/tilt rows (`source='sentinel'`, `source_id` `…:sig:…`, `value=tilt×confidence` ∈[−1,1]). LIVE.
- **#602 — matrix-cell heal-on-rewrite:** `ON CONFLICT DO UPDATE` (was DO-NOTHING) → 23505 spam 462/30m→0; `MatrixCellWriteResult`; projector-error alerts. LIVE. (Prereq for #6.)
- **#615 Fix #6 — news decay/accumulation:** `ObservationCellProjector` news path = per-article 24h-half-life **decay-weighted SUM + tanh(S/K)**; FRED untouched; `:sig:`-only (excludes old raw-value rows); heal-in-place via #602. LIVE+§7-verified.
- **#616 — entity→sector grounding (L1 only landed the payoff):** fixed the `/api/resolve-entities` **100% timeout** (unbounded HTTP pool starvation → bounded `MaxConnectionsPerServer=16` + 15s fail-fast). L2/L3 (persist + matrix-feed) wiring is correct but produces nothing — see NEXT #1. LIVE.
- **#614/#611** — workstream docs (break-map) + ntfy-mcp spec-fix + STATE landed on main.

## NEXT (fresh session — prioritized)
1. **SECTOR breadth — SecMaster instrument NAICS enrichment [the real blocker].** EVIDENCE (#616 deploy verify): `/api/resolve-entities` resolves instruments via OpenFIGI but returns `naicsCode=null → atlasSectorCode=null` even for blue-chips (MSFT/XOM/TPL @0.9). NAICS→ATLAS map works; resolve-entities never SUPPLIES NAICS. **Only ~1,490/14,459 instruments NAICS-tagged → `extracted_observations.atlas_sector_code` still 0.** Investigate why the SecMaster OpenFIGI→NAICS **classification backfill** covers only ~10% (running? broken? coverage gap?), then fix it so resolve-entities returns sectors. This is the only thing between the (now-correct) Sentinel pipeline and the sector breadth.
2. **SIGNAL breadth — #5 pattern reconciliation [config-only, quick win].** Only 7 of 23 produced news signals map to enabled ThresholdEngine patterns; the rest are `unknown_signal`-skipped (Fix #6 decay invisible for them). **Option A** (recon done): add `metadata.signalIdentity` to ~8 enabled pattern files under `ThresholdEngine/config/patterns/` whose `RequiredSeries` already matches the news signal → recovers ~18 rows. Hot-reload via `ansible … --tags patterns` (no code, no restart). 3 ambiguous ids need a pick (yield-curve-2s10s, unemployment-rate, nonfarm-payrolls — assign each to ONE pattern, the map dedups). Watch headline/core + real/nominal mismatches (don't reconcile by FRED-series alone).
3. **Fix #4 — the digest VIEW (user's original ask).** Render the daily/weekly/monthly brief as **news-bias-vs-matrix**: the period's collection bias (volume/tilt by sector & signal) juxtaposed against the matrix values → divergences + coverage gaps. Emergent narrative themes; bias axes matrix-aligned. Reuse the closed #612's `DigestArticleSelector`/`DigestArticleTextProvider`. Daily=today, weekly/monthly=sustained.
4. **Deferred/housekeeping:** deploy the MacroSubstrate FRED/Sentinel `macro_observations` heal to the COLLECTORS (needs FredCollector/SentinelCollector rebuild); decide legacy Path-1 (`MatrixCellPersistenceWorker`) fate; `EntityResolutionPrepassTests` missing `[Collection("SentinelMeterStatic")]` (pre-existing flake); `docs/benchmarks/cod-2026-05-17/corpus.jsonl.macro-bak` untracked (commit-or-discard).

## DEPLOY / RUNTIME NOTES
- Scoped deploy: `ansible-playbook playbooks/deploy.yml --tags <svc> -e "atlas_repo_path=<repo> scoped_restart=true scoped_services=<svc>"`. **`atlas_repo_path` defaults to the main checkout** → a routine deploy builds whatever it's checked out on; keep main checkout on `main` & current. Verify running image digest + a real check after deploy. [[ansible_deploy_stale_image]]
- ⚠ **alert-service STAYS DOWN (user)** — a NON-scoped tag run resurrects it (the `[always]` infra `compose up -d`); always pass `scoped_restart=true scoped_services=<svc>` even on `dashboards`/`alerting`. Prometheus reload: `sudo nerdctl exec prometheus kill -HUP 1`.
- DB inspect: `sudo nerdctl exec timescaledb psql -U ai_inference -d atlas_data` (SELECT only; never DML [[feedback_agent_no_raw_db_fixes]]). Schemas: news obs `sentinel.extracted_observations`; matrix feed `public.macro_observations`; cells `public.matrix_cells`.
- vLLM = `Qwen2.5-32B-Instruct-AWQ`, 32K ctx. CPU CoD = llama-server (saturated, `--parallel 1`). LLM work ≠ CPU-extraction → dedicated VllmClient. SecMaster `/api/resolve-entities` ~4.9s/call (sequential candidate loop).

## SUPERVISOR OPERATING NOTES
- All code/test/build/deploy → background subagents (never direct). STATE.md supervisor-owned (annotate ≤30 lines/turn; full rewrite only when directed). Verify status vs code/DB; don't ask user to approve STATE. [[feedback_state_md_not_session_diary]]
- Autonomy: dispatch/review/fix/merge/deploy autonomous; NTFY `atlas-claude-ask` only on genuine blocker / architectural fork / milestone. End turns `▶ continuing autonomously` vs `🛑 DECISION NEEDED`. NTFY bodies must be SINGLE-LINE (newlines break the tool-call) [[feedback_tool_failure_debug_not_reroute]].
- Merge gate = INVOKE `/pr-review-toolkit:review-pr <N>`; re-run after EACH new commit (gate blocks on commits-since-review) [[project_pr_review_marker_staleness]]. Push to main BLOCKED → branch+PR always (even docs); non-.NET trees use `scripts/claude-mark-verified`. Git: selective `git add -- <paths>`.
- Worktree-isolate parallel code agents [[feedback_parallel_agents_use_worktrees]]. Subagents may commit a stray `SentinelCollector/STATE.md` (per-task doc) — keep it untracked.

## ARCHIVE (git log + tags)
counterweight restoration #602/#613/#614/#615/#616 · WS3 realign #552–#587 · DSL PoC #381–#510 · Matrix MVP #215–#261 · digest plumbing #607–#611. #612 (digest-qual) CLOSED-superseded.
