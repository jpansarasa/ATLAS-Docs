# ATLAS Supervisor STATE

Supervisor memory. Read first, write last. **Current truth + open items + pointers ONLY.** History → git log + PRs + tags + memories.

## EACH TURN START HERE
0. **FIRST TURN** — git audit (`worktree list / branch / status / fetch + log HEAD..origin/main`); on `main` & current. [[feedback_git_session_start_discipline]] [[feedback_main_always_current]]
1. Poll NTFY `atlas-claude-reply` (user input first).
2. Re-read this file + the active spec THIS turn; walk the pipeline backward before dispatching; plan-vs-prod mismatch → STOP + NTFY.

## CURRENT WORK
*(new investigation — scope TBD; populate here as it's defined.)*

## PARKED — #729 regime news-as-staleness redesign  (NOT current focus; spec = PR #729, DO-NOT-MERGE)
Intent: FRED/OFR benchmark = slow grounding anchor; Sentinel news = fast-decaying coincident perturbation weighted by benchmark STALENESS; measure economic significance, not coverage volume. Phases 1 / 2a / 2b built + deployed (Phase 2b in **shadow**) — #730–#736. Open:
- 🔴 **Phase 2c cutover** (`Matrix:NewsClustering:Mode=Authoritative`) — **HELD.** Embedding coverage still ramping (missing_embedding ~0.57) AND shadow `magnitude_ratio` ~2.5 (>1) ⇒ dedup AMPLIFIES / sign-flips the net (not gentle compression) → materially restructures news. Hold until coverage fills + Phase 3 revisited.
- ◯ **Phase 3** — staleness crossfade `news_weight = g(benchmark_age/cadence)` sigmoid + benchmark-anchored aggregation. The principled fix for the structural news-overweight (news MACRO cells ~2-4× FRED magnitude, net-negative → washes sector differentiation → over-neutral regime). Gated on 2c + backtest.
- ◯ **BACKTEST harness** = the validation layer that unblocks all of #729 (awaiting go). Shadow has no ground truth. Outcome data found: `finnhub_quotes` XLE/XLF/XLV/XLK ~6.5mo + SPY/QQQ + Yahoo EOD backfill for all 11; AV WTI/BRENT/NATGAS to 1986. First gate = establish a realized-sector-return series.
- ⏸ classifier **sign-fix** parked — DO NOT ship off the energy anecdote (n=6 peek showed news directionally right; needs real backtest N) [[project_energy_signal_not_misfire]]. `commercial-paper-stress` de-flagged interim (#733); spread-redesign `DCPN3M − DGS3MO` (add DGS3MO) proposed, awaiting go.

## DEPLOY / RUNTIME
- **Scoped deploy**: `ansible-playbook playbooks/deploy.yml --tags <svc> --skip-tags build -e "scoped_restart=true scoped_services=<svc>"` from `deployment/ansible/` (build image first: `{Project}/.devcontainer/build.sh --no-cache`; `--skip-tags build` uses the fresh `:latest`, avoids stale-cache rebuild). Non-scoped / `[always]` run = compose regen + full-stack restart (incl ~4min vLLM reload). Grafana provisioning is startup-loaded → `--tags alerting|dashboards --skip-tags always` + restart grafana. ✅ scoped restart is now TRULY scoped (#816, merged 2026-06-30): raw-rm the named target + `compose create --no-recreate` + start → `depends_on` peers (incl. timescaledb) AND vLLM stay up, no cascade / no transient-burst; fails loudly if a peer drops or the target isn't running after. Verified end-to-end on sentinel-collector (only target recreated). [[project_scoped_restart_cascade_gotcha]] ⚠️ `autofix-runner/-watcher` timers run a FULL deploy from main every few min → unpushed config reverts; MERGE to hold. Repeated `--no-cache` builds load the host → transient Npgsql cmd-timeouts in other collectors (benign). [[ansible_deploy_stale_image]] [[project_ansible_deploy_restart_behavior]]
- Boot: `atlas.service` + `otel.service` (enabled) bring the stack up; containers `restart: unless-stopped`. EF migration: `nerdctl compose exec -T <svc>-dev dotnet ef migrations add <Name> --context <Ctx> --output-dir Data/Migrations`. Prometheus reload: `sudo nerdctl exec prometheus kill -HUP 1`.
- DB inspect (SELECT only; never DML [[feedback_agent_no_raw_db_fixes]]): `sudo nerdctl exec timescaledb psql -U ai_inference -d atlas_data`. news obs `sentinel.extracted_observations` / `raw_content`; matrix feed `public.macro_observations`; cells `public.matrix_cells`; regime `public.sector_regimes`. SecMaster DB = `atlas_secmaster`.
- Inference: GPU vLLM `Qwen2.5-32B-Instruct-AWQ` 32K (extraction `VllmJson` + classifier); CPU `llama-cpu-rag` (qwen2.5:7b, SecMaster RAG, host port 11438; solo ~29 tok/s post-EXPO-5600), `llama-cpu-embed` (bge-m3), DSL `llama-server` — all memory-bandwidth-bound on the 9960X single IMC (per-request rate divides under concurrency; that's at-spec, not a bug). Trust running `printenv`, not commit narrative.
- Push: git-push-guard blocks direct main push except docs-only; commit + push = SEPARATE bash calls; docs/STATE push → `scripts/claude-mark-verified` then push. Merge gate: `pr-review-toolkit:review-pr <N>` writes `/tmp/atlas-test-markers/pr-reviewed-<N>` keyed to headRefOid; re-run after new commits. [[project_push_guard_main_block]] [[project_pr_review_marker_staleness]]

## SUPERVISOR OPERATING NOTES
- Land substantive work as GitHub PRs [[feedback_work_in_github_not_local]]. **Verify agent DATA vs DIAGNOSIS before surfacing/recording** — for a stuck/looping/slow symptom, pull the OTEL counters FIRST [[feedback_pull_counters_before_loop_hypothesis]] [[feedback_verify_agent_inferences]].
- Code/test/build/deploy → background subagents; worktree-isolate parallel code agents [[feedback_parallel_agents_use_worktrees]]; run `compile.sh` AS FINAL from within the worktree; silent 30-60min usually = mid-compile [[feedback_compile_slow_dont_kill]].
- Autonomy: dispatch/review/fix/merge/deploy autonomous; NTFY `atlas-claude-ask` only on genuine blocker / arch fork / milestone; ACK on receipt. End turns `▶` (autonomous) vs `🛑` (decision needed) [[feedback_signal_autonomous_vs_pause]].
