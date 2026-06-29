# ATLAS Supervisor STATE

Supervisor memory. Read first, write last. **Current truth + open items + pointers ONLY.** History → git log + PRs + tags + memories.

## EACH TURN START HERE
0. **FIRST TURN** — git audit (`worktree list / branch / status / fetch + log HEAD..origin/main`); on `main` & current. [[feedback_git_session_start_discipline]] [[feedback_main_always_current]]
1. Poll NTFY `atlas-claude-reply` (user input first).
2. Re-read this file + the active spec THIS turn; walk the pipeline backward before dispatching; plan-vs-prod mismatch → STOP + NTFY.

## OPEN ITEMS [check every turn]

### A. #729 — regime news-as-staleness redesign (the big ongoing effort; spec = PR #729, DO-NOT-MERGE design surface)
Intent: benchmark (FRED/OFR) = slow grounding anchor; Sentinel news = fast-decaying COINCIDENT perturbation weighted by benchmark STALENESS; measure economic SIGNIFICANCE, not coverage volume. 5 phased shadow-gated pieces. Autonomous supervisor execution authorized 2026-06-16.
- ✅ Phase 1 — per-signal `matrix-benchmark` flag (SecMaster = single source of truth) + FredCollector gate repoint. Done/deployed (#730; unit-bug fix #731). 52 signals flagged.
- ✅ Phase 2a per-article embeddings (#732, IaC #734) + 2b occurrence-clustering in SHADOW + dashboard/alert (#735, #736). Done/deployed shadow.
- 🔴 **Phase 2c cutover** (flip `Matrix:NewsClustering:Mode=Authoritative`) — **HELD / NO-GO.** Coverage still ramping (missing_embedding ~0.57) AND `magnitude_ratio` settled **~2.5 (>1)** → dedup AMPLIFIES/sign-flips the net (not gentle compression) → materially restructures news. Hold until coverage fills AND Phase 3 is revisited.
- ◯ **Phase 3** (staleness crossfade `news_weight = g(benchmark_age/cadence)` sigmoid + benchmark-anchored aggregation) — NOT built. The principled fix for the structural news-overweight (audit-confirmed: news MACRO cells ~2-4× FRED magnitude, net-negative → washes sector differentiation → regime over-neutral). Gated on 2c + backtest.
- ◯ **BACKTEST harness = the missing validation layer that unblocks ALL of #729** (awaiting user go). Shadow has no ground truth (`magnitude_ratio` uninterpretable). Outcome data FOUND: `finnhub_quotes` XLE/XLF/XLV/XLK ~6.5mo + SPY/QQQ/IWM; AV WTI/BRENT/NATGAS to 1986; Yahoo EOD backfill for all 11. Signal side OK (sentinel→2023, fred→2025-10). First gate = establish a realized-sector-return outcome series.
- ⏸ **PARKED — classifier sign-fix** (news SIGN wrong on market-priced signals: oil-news −0.98 vs FRED +0.39; all 5 sign-disagreers are market-priced). **DO NOT ship off the energy anecdote** — n=6 peek showed news was directionally RIGHT (XLE mean-reverting from the Hormuz spike). Revisit only with real backtest N. [[project_energy_signal_not_misfire]] "Divergence-as-signal" = unvalidated idea, survives.
- ◯ **commercial-paper-stress** DE-FLAGGED interim (#733; uses CP rate LEVEL not a spread, unit-bugged). Spread-redesign PROPOSED (awaiting go): `DCPN3M − DGS3MO` (must add DGS3MO to FredCollector), `signal = clamp(−(spread−0.25)/0.4, ±3)`.

### B. Sentinel SEC/paywall + feed-tuning (this session — DONE / deployed / stable; pointers only)
- **W1/W2/W4 feed-tuning** — dead-feed prune, 11 gap feeds, VIX/valuation news extraction. Live + validated (#789–791 etc.).
- **SEC/paywall mirror-search** (#793–801): on a terminal-4xx article-fetch, query SearXNG by headline → fetch an open mirror's full body (MSN/Yahoo syndicate Bloomberg/MarketWatch) → extract; title+summary last-resort; timeout/transient give-ups capped + try fallback first. Root cause of the early divergence was the SearXNG `time_range=day` query returning 0 → false soft-outage → spin (fixed #800). Converged + stable. Flags ON: collector `MirrorSearch__Enabled=true` (compose env), edge `ENABLE_RSS_FALLBACK=true`.
- Verifier "rows dropped" Warn→Info (#802). SecMaster keyword extraction constrained via llama.cpp `json_schema`, hand-rolled JSON-salvage parser deleted (#803). [[feedback_constrain_llm_output_not_salvage_parse]]

### C. Open follow-ons
- 🔄 **RAM EXPO 4800→6400** — user rebooting mercury to enable (4×DDR5; 9960X supports 6400 native). Post-reboot verify: `sudo dmidecode -t memory | grep "Configured Memory Speed"` = 6400, then re-measure `llama-cpu-rag` SOLO decode (`curl localhost:11438/completion … | jq .timings.predicted_per_second`) ≈ **32 tok/s** (was ~24; +33%, lifts ALL CPU inference runners). `atlas.service` (enabled, `ExecStart=nerdctl compose up -d`) brings the stack up on boot.
- SecMaster **4 tok/s = AT SPEC**, not a bug: decode is memory-bandwidth-bound; aggregate ~24 tok/s = the documented benchmark; per-request "4" = `--parallel 4` division + embed BW steal under load. Optional cheap *latency* lever: `--parallel` 4→2. [[feedback_pull_counters_before_loop_hypothesis]]
- SecMaster "discovery failed" warnings = the collectors' own 30s HTTP timeouts (low-impact) — demote-or-accept, user's call.

## DEPLOY / RUNTIME
- **Scoped deploy**: `ansible-playbook playbooks/deploy.yml --tags <svc> --skip-tags build -e "scoped_restart=true scoped_services=<svc>"` from `deployment/ansible/` (build image first: `{Project}/.devcontainer/build.sh --no-cache`; `--skip-tags build` uses the fresh `:latest`, avoids stale-cache rebuild). A non-scoped / `[always]` run regenerates compose + full-stack restart (incl ~4min vLLM reload). Grafana provisioning is startup-loaded → `--tags alerting|dashboards --skip-tags always` + restart grafana. ⚠️ even scoped, `compose up <svc>` recreates the `depends_on` chain → brief transient-error burst, self-heals (Polly). ⚠️ `autofix-runner/-watcher` timers run a FULL deploy from main every few min → unpushed config reverts; MERGE to hold. Repeated `--no-cache` builds load the host → transient Npgsql cmd-timeouts in other collectors (benign). [[ansible_deploy_stale_image]] [[project_ansible_deploy_restart_behavior]]
- Boot: `atlas.service` + `otel.service` (enabled) bring the stack up; containers `restart: unless-stopped`. EF migration: `nerdctl compose exec -T <svc>-dev dotnet ef migrations add <Name> --context <Ctx> --output-dir Data/Migrations`. Prometheus reload: `sudo nerdctl exec prometheus kill -HUP 1`.
- DB inspect (SELECT only; never DML [[feedback_agent_no_raw_db_fixes]]): `sudo nerdctl exec timescaledb psql -U ai_inference -d atlas_data`. news obs `sentinel.extracted_observations` / `raw_content`; matrix feed `public.macro_observations`; cells `public.matrix_cells`; regime `public.sector_regimes`; mirror `sentinel.mirror_attempts`. SecMaster DB = `atlas_secmaster`.
- Inference: GPU vLLM `Qwen2.5-32B-Instruct-AWQ` 32K (extraction `VllmJson` + classifier); CPU `llama-cpu-rag` (qwen2.5:7b, SecMaster RAG, port 11438), `llama-cpu-embed` (bge-m3), DSL `llama-server` — all memory-bandwidth-contended on the 9960X single IMC. Trust running `printenv`, not commit narrative.
- Push: git-push-guard blocks direct main push except docs-only; commit + push = SEPARATE bash calls; docs/STATE push → `scripts/claude-mark-verified` then push. Merge gate: `pr-review-toolkit:review-pr <N>` writes `/tmp/atlas-test-markers/pr-reviewed-<N>` keyed to headRefOid; re-run after new commits. [[project_push_guard_main_block]] [[project_pr_review_marker_staleness]]

## SUPERVISOR OPERATING NOTES
- Land substantive work as GitHub PRs [[feedback_work_in_github_not_local]]. **Verify agent DATA vs DIAGNOSIS before surfacing/recording** [[feedback_verify_agent_inferences]] — two investigations this session corrected an earlier wrong inference (the timeout-not-the-cause; the 4tok/s-is-at-spec).
- Code/test/build/deploy → background subagents; worktree-isolate parallel code agents [[feedback_parallel_agents_use_worktrees]]; run `compile.sh` AS FINAL from within the worktree; silent 30-60min usually = mid-compile [[feedback_compile_slow_dont_kill]].
- Autonomy: dispatch/review/fix/merge/deploy autonomous; NTFY `atlas-claude-ask` only on genuine blocker / arch fork / milestone; ACK on receipt. End turns `▶` (autonomous) vs `🛑` (decision needed) [[feedback_signal_autonomous_vs_pause]].
