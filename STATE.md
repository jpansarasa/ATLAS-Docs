# ATLAS Supervisor STATE

Supervisor memory. Read first, write last. **Current truth + pointers only** — history lives in git log. Keep lean. VERIFY each status vs code/config/DB before writing.

## EACH TURN START HERE
0. **FIRST TURN OF SESSION ONLY** — git audit before dispatch: `git worktree list`, `git branch`, `git status`, `git fetch origin && git log --oneline HEAD..origin/main`. Flag stale worktree branches / drift. Safe cleanup direct; destructive needs user approval. [[feedback_git_session_start_discipline]]
1. Poll NTFY `atlas-claude-reply` (user input FIRST).
2. RE-READ the active plan — break-map `docs/sentinel-matrix-break-map-2026-06-05.md` (canonical diagnosis) + the current Fix spec + this file. RE-READ, not "read earlier".
3. Walk the pipeline backward from this turn's target before dispatching.
4. Plan vs production mismatch → STOP + NTFY (architectural), don't dispatch. [[feedback_walk_pipeline_each_turn]]

## WHERE WE ARE NOW [2026-06-05] — Sentinel→matrix counterweight RESTORATION
The "empty digest" thread bottomed out at a severed pipeline. **Sentinel's value = SUSTAINED news as a fast-decaying LEADING counterweight to lagging/coincident hard data (FRED); a single day barely moves a cell (large alpha decay).** The CoD/DSL cutover (~05-28/29) + A3 (#601) cut that news signal out of the matrix at every layer. Canonical diagnosis: **`docs/sentinel-matrix-break-map-2026-06-05.md`**.

**The break (6 layers, one root + a design regression):** CoD changed extraction field shapes — `Description` (canonical label → CLAIM phrase) and `text_quote` (source sentence → value fragment) — which broke the macro-route, sector-tag, and signal-identity hooks; #601 then made `macro_observations` the SOLE matrix feed (so the mismatch fully severs the signal) AND replaced the news magnitude model (deleted `SignalMagnitudeCalculator`: 24h-half-life + daily-bucket accumulation) with FRED's staleness contract + a 120-day mean. DB proof: sentinel `macro_observations` stop 05-28; `atlas_sector_code` 0 since 05-29; signal-keyed cells froze 05-21; matrix latest 06-01.

## FIX SEQUENCE (dependency order)
1. **Fix #1 — news→macro_observations FEED [✅ LIVE + VERIFIED 2026-06-05].** Per-article vLLM classifier → `macro_observations` (signal-keyed, signed `value=tilt×confidence`), CONTINUOUS (semantic tier). Merged #613 (main `4b058ae2`), deployed (image `2662d95b`), VERIFIED in prod: 6 signal-keyed rows, `value_numeric` [−0.70,+0.72] signed, classifier per-article (0 lost/dropped); quality excellent (services-growth+prices→gdp/cpi/pce +0.48, curve +0.28; strong-jobs→payrolls +0.72/jobless −0.70 opposing-signs ✓); vLLM 2.3GB free 0 OOM; Loki clean. `atlas_sector_code` NULL = correct-by-design (economy-wide signals). Observability hardened (drop-counter reason{nan,sub_floor}, write-result {written,idempotent_skip,lost}, NewsSignal band-guard, isolation tests, 2 alerts). Spec: `docs/superpowers/specs/2026-06-05-news-signal-feed-design.md`.
   ⚠ **DEPLOY REGRESSION RISK:** `atlas_repo_path` defaults to the main checkout (`/home/james/ATLAS`), currently on `feat/digest-qual-from-text` (lacks #613) → a PLAIN `--tags sentinel-collector` deploy would REBUILD STALE + drop the prompt, re-silencing the feed. This deploy used `-e atlas_repo_path=/tmp/deploy613` (origin/main worktree). **Must return main checkout to a #613-main before any routine deploy** (git-cleanup task below).
2. **Fix #6 — news decay/accumulation** in `ObservationCellProjector`: restore per-article 24h-half-life + daily-bucket accumulation (the deleted semantics), not the 120-day mean + last-write-wins.
3. **Matrix hygiene** — rebase+merge **PR #602** (`fix/ws3/heal-on-rewrite`: 23505 log-spam → `ON CONFLICT DO UPDATE`; OPEN, 9 behind main, no CI); reconcile fresh signal-ids → enabled patterns (unknown-signal skips are the real freshness stall, not the 23505); decide legacy Path-1 (`MatrixCellPersistenceWorker`) fate.
4. **Digest = the VIEW (last):** news-bias-vs-matrix — divergence + coverage gaps; daily=today, weekly/monthly=sustained. Emergent narrative themes; bias axes matrix-aligned.

## DONE — digest plumbing (merged + deployed; the symptom layer, NOT the substance)
#607 confidence threading · #608 narrative payload cap + rejection/missing alerts · #609 narrative→dedicated vLLM (~7s, Ok) · #610 article-grounded narrative · #611 STATE doc. origin/main = `3eeac5cb`+. See [[project_cod_cutover_zeroed_digest]]. Un-emptied the digest but the SUBSTANCE (news→matrix) is the Fix Sequence above.
**PR #612 (digest qualitative panels from article text) — OPEN, mid-review, SUPERSEDED by Fix #4.** Selector/article-reading is reusable foundation; its regex-themes/radar are replaced by the LLM-classification approach. Disposition pending: close vs merge-as-foundation.

## DEPLOY / RUNTIME NOTES
- Scoped deploy: `ansible-playbook playbooks/deploy.yml --tags <svc> -e "scoped_restart=true scoped_services=<svc>"` (freshness-gated stop+rm+up; nerdctl 1.7.7 — no `--force-recreate`/`--no-deps`). `compose up -d <svc>` recreates the depends_on chain (DB+secmaster) — harmless. Always verify running image digest + a real digest after deploy. [[ansible_deploy_stale_image]]
- alert-service STAYS DOWN (user). DB inspect: `sudo nerdctl exec timescaledb psql -U ai_inference -d atlas_data` (SELECT only; never DML [[feedback_agent_no_raw_db_fixes]]).
- vLLM = `Qwen2.5-32B-Instruct-AWQ`, 32K ctx, `Extraction:VllmEndpoint` `http://vllm-server:8000`. CPU CoD = llama-server (saturated, `--parallel 1`). Pattern: LLM work that isn't CPU-extraction → dedicated VllmClient (#609 / SectorTagger).

## SUPERVISOR OPERATING NOTES
- All code/test/build/deploy → background subagents (never direct). STATE.md/plans/CLAUDE.md supervisor-owned (annotate ≤30 lines/turn; full rewrite only when directed). Verify status vs code/DB; don't ask user to approve STATE. [[feedback_state_md_not_session_diary]]
- Autonomy: dispatch/review/fix/merge/deploy autonomous; NTFY `atlas-claude-ask` only on genuine blocker / architectural fork / milestone. End turns `▶ continuing autonomously` vs `🛑 DECISION NEEDED`. [[feedback_signal_autonomous_vs_pause]]
- Merge gate = INVOKE `/pr-review-toolkit:review-pr <N>` (skill) before merge [[project_pr_review_marker_staleness]]. Direct push to main BLOCKED (branch protection) → feature branch + PR always; even docs-only needs a branch+PR (root `.md` not in the docs-only allowlist). Git: selective `git add -- <paths>`, never `-A`/`-u`/`.`.
- NTFY: publish `atlas-claude-ask`, poll `atlas-claude-reply`.

## OPEN HOUSEKEEPING (non-blocking)
- Untracked docs to land (branch+PR): `docs/sentinel-matrix-break-map-2026-06-05.md`, `docs/superpowers/specs/2026-06-05-news-signal-feed-design.md`, `docs/superpowers/specs/2026-06-05-digest-qualitative-from-article-text-design.md` (in #612), `corpus.jsonl.macro-bak`.
- Worktrees: `agent-a023e66` (fix/ws3/heal-on-rewrite = PR #602), `agent-a8322bae` (cpuset, PR #589, locked) — pre-existing.

## ARCHIVE (git log + phase tags)
WS3 matrix realignment #552–#587 · DSL PoC Phases 0–5 #381–#510 · Matrix MVP #215–#261 · digest plumbing #607–#611. Recoverable via git history of this file + tags.
