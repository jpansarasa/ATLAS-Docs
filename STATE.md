# ATLAS Supervisor STATE

Supervisor memory. Read first, write last. **Current truth + open items + pointers ONLY.** History → git log + PRs + tags + memories. NOT per-turn progress, review verdicts, or merged-PR narration.

## EACH TURN START HERE
0. **FIRST TURN ONLY** — git audit (`git -C /home/james/ATLAS worktree list / branch / status / fetch + log HEAD..origin/main`); main on `main` & current. [[feedback_git_session_start_discipline]] [[feedback_main_always_current]]
1. Poll NTFY `atlas-claude-reply` (user input first).
2. RE-READ this file + active plan THIS turn. Walk the pipeline backward before dispatching; plan-vs-prod mismatch → STOP + NTFY.

## GOAL [PINNED — check every turn AND every dispatch against THIS]
**Both major deliverables DONE 2026-06-10 (see ARCHIVE):**
- **Throughput:** Sentinel CoD extraction ~20 → ~383/hr sustained (~19×), backlog drained to 0.
- **Digest overhaul:** daily/weekly/monthly reports useful — real sectors, de-noised word cloud, News Signal Momentum, cross-collector "all data collection" coverage; macro-noise Top-Sectors count-table dropped. Live + verified at digest 149.
**Standing digest quality bar:** DAILY = good picture of last 24h across ALL collectors (FRED/AV/Finnhub/OFR/Nasdaq/news); WEEKLY/MONTHLY = representative 7d/month summaries.
**Now: MONITOR-MODE** — watch prod health (errors/crashes/backlog/throughput), address issues, await next task.
**HARD CONSTRAINTS:** quality must not regress; land substantive work as **GitHub PRs** [[feedback_work_in_github_not_local]]; **verify agent DATA vs DIAGNOSIS before surfacing** (this session's agents repeatedly misread metrics — the `ollama-cpu-gen`=SecMaster red herring tripped 3) [[feedback_verify_agent_inferences]].

## WHERE WE ARE [2026-06-10]
Throughput (~19×) + the full digest overhaul both DELIVERED, deployed, verified, and merged (#640–#652); GitHub clean (no open PRs, main==origin). Monitor-mode; production healthy (queue drained, metrics verified representative `a142d89a`; extraction = GPU vLLM VllmJson, continuous-streaming). One optional digest residue + a few standing TODOs below.

## OPEN (current truth)
- 🔭 **Digest follow-ups (optional / low-pri):** (i) the ntfy-push "· sectors:" preview line does Take(3)-by-count → still leads "Uncategorized" (the macro bucket) THERE; the digest BODY table is already removed. Offer to drop/relabel in the push line too — **awaiting user**. (ii) Testcontainers integration test for the 4 cross-collector raw-SQL queries (currently unit-mocked). (iii) notable-moves floor is per-collector ABSOLUTE → tune to RELATIVE if small-base %-moves (e.g. RRPONTSYD +140%) read as noisy.
- 🔭 **OTEL cleanup (low-pri):** rename `sentinel_cpu_extraction_outcome_total` (GPU work despite the "cpu" name — caused this session's misreads) engine-neutral + update dashboards/alerts; instrument the un-timed `resolve` stage (0 samples).
- 🧹 **Matrix data-hygiene (`a5d730d0`):** 226 legacy `matrix_cells` rows breach the ±3 invariant (max +28) + 3 stray-sector (`51`/`52`) rows; all pre-WS3-projector (NULL `signal_identity_id`). The ±3 / 11-sector invariant is CODE-ONLY (no DB CHECK). Fix: one-time cleanup of pre-WS3 rows + EF migration adding a DB CHECK. Spec: `docs/atlas-matrix-handoff-v2.md`.
- 🐛 **SecMaster 7b RAG retry-storm (separate, orthogonal):** `ollama-cpu-gen` pegged ~1500% CPU in an 8s-timeout client-abort loop (SecMaster hybrid-resolve under saturation) — unrelated to Sentinel throughput; a SecMaster-side fix worth doing.
- **Codify the core-dump `LimitCORE=512M` drop-in into deployment IaC** — manual host file (`/etc/systemd/system/containerd.service.d/core-limit.conf`), lost on rebuild.
- **Recalibrate flagged natgas/ECB `sectorWeights`** vs observed betas (optional, hot-reloadable).
- **Hard-data natgas/ECB → matrix wiring** — DHHNGSP/ECBDFR evaluate but don't reach `macro_observations`; matrix entry is news-`:sig:`-only.
- #622 doc-sweep (`~/.ansible_vault_pass` refs); hooks README "commit-hash-keyed" → "head-keyed". · break-map doc `docs/sentinel-matrix-break-map-2026-06-05.md` SUPERSEDED → update/retire. · Optional #619 strict per-source parallel refactor.

## DEPLOY / RUNTIME NOTES
- Scoped deploy: `ansible-playbook playbooks/deploy.yml --tags <svc> -e "scoped_restart=true scoped_services=<svc>"`. **Always scoped** — a non-scoped/`[always]` run regenerates compose + resurrects `alert-service` (must STAY DOWN) — incl. `--tags dashboards` (`compose up`): `sudo nerdctl stop alert-service` after. `--no-cache` build + verify running digest. A cutover must redeploy ALL services the new path touches. [[ansible_deploy_stale_image]] [[project_ansible_deploy_restart_behavior]]
- ansible vault: ansible.cfg uses absolute path (`/home/james/.ansible_vault_pass`) → run from `deployment/ansible/` (inventory is `inventory/hosts.yml`). Prometheus rule reload: `sudo nerdctl exec prometheus kill -HUP 1`.
- DB inspect: `sudo nerdctl exec timescaledb psql -U ai_inference -d atlas_data` (SELECT only; never DML [[feedback_agent_no_raw_db_fixes]]). SecMaster DB = `atlas_secmaster`. Schemas: news obs `sentinel.extracted_observations`; matrix feed `public.macro_observations`; cells `public.matrix_cells`. Digest served at `http://localhost:5091/sentinel/digests/<id>`.
- **CoD = GPU vLLM `Qwen2.5-32B-Instruct-AWQ` 32K-ctx, `Extraction__Backend=VllmJson`, `MaxConcurrent=8`, `ContinuousStreaming=ON`, loop-guard rep1.1/maxItems60/maxtok4096** (`/parse_json` on dsl-parser-mcp). Classifier also on GPU vLLM. CPU `llama-server` = OLD CoD path, still wired as rollback (CpuInference disabled). **READER-TRAPS:** metric `sentinel_cpu_extraction_outcome_total{path=gpu_json}` = GPU work despite "cpu" name; "MaxConcurrent=1/#639/llama-server" = STALE config — trust running `printenv`, not commit narrative.
- Tag-push: the git-push-guard hook can't parse tag pushes → tag via `gh release create <tag> --target main` (API). Docs/STATE-only push → `scripts/claude-mark-verified` then push.

## SUPERVISOR OPERATING NOTES
- All code/test/build/deploy → background subagents (worktree-isolate parallel code agents). **Verify agent DATA vs DIAGNOSIS** before recording/surfacing. [[feedback_verify_agent_inferences]]
- Implementation-agent briefs: forbid `setsid`/`nohup`/detached procs; run `compile.sh` AS FINAL after committing, **FROM WITHIN THE WORKTREE** (`cd <worktree>` first) — running it from the main repo keys the push-marker to MAIN's tree → push blocked (recurring this session, root cause `a6467309`). A code agent silent 30-60min is usually mid-`compile.sh` (≈30-40min) — check worktree git status before TaskStop. [[feedback_compile_slow_dont_kill]]
- Autonomy: dispatch/review/fix/merge/deploy autonomous; NTFY `atlas-claude-ask` only on genuine blocker / architectural fork / milestone. End turns `▶ continuing autonomously` vs `🛑 DECISION NEEDED`.
- Merge gate: `pr-review-toolkit:review-pr <N>` Skill writes `/tmp/atlas-test-markers/pr-reviewed-<N>` keyed to headRefOid; re-run after new commits (let GitHub head propagate first — head-lag mis-keys it; bit this session on #647/#652). [[project_pr_review_marker_staleness]]
- gh auth (2026-06-10): OAuth token can expire mid-session → 401 even though `gh auth status` shows logged-in; user re-auths (`gh auth login`) OR use `GH_TOKEN=<pat>` inline.

## ARCHIVE (git log + tags)
2026-06-10 — **Digest overhaul** (#649–#652, live+verified @ digest 149): real sectors (re-sourced from classifier-fed `macro_observations` vs ~99%-NULL `extracted_observations.atlas_sector_code`) + de-noised word cloud (per-document freq + scalar-series exclusion, killed the TSA flood) + **News Signal Momentum** (replaced the stale bias-vs-matrix that compared fresh news to a 2-month-old FRED matrix) + **cross-collector "Across All Data Collection"** rollup (raw-hypertable scoreboard + notable-moves table + macro tape; only FRED/OFR-STFM/OFR-FSI/Finnhub-quotes are live, rest footnoted) + dropped the macro-noise Top-Sectors count-table (the "Uncategorized 36471/420" residue was scattered macro signals — kept the BySector data feed for the push/narrative/metric).
2026-06-09/10 — **Extraction throughput ~20→~383/hr sustained (~19×), backlog drained to 0.** Chain: GPU-CoD role-flip (tag `gpu-cod-roleflip-2026-06-09`, #640–#646: CoD CPU llama-server→GPU vLLM JSON-schema, recall gate 0.79) + **verifier removal** (#647: dead SemanticVerifier was ~93% of GPU calls, computed-and-discarded — its matrix consumer retired by design WS3-A3 #601, matrix is classifier-fed) + **continuous-streaming dispatch** (#648: killed the ~40-batch + 5s-`Task.Delay` barrier → GPU continuously fed). Observability verified representative (`a142d89a`); user banked. Lessons: `ollama-cpu-gen`=SecMaster-RAG (NOT Sentinel) red herring tripped agents 3×; CoVe stays GPU (n=563); slow compile ≠ stuck agent.
2026-06-07 — extraction throughput: queue/frontier monitoring (#636) · per-stage OTEL timing (#638) · CPU tuning `3a802551` · memory-hygiene skill+hook (#637).
2026-06-06/07 — digest-quality + reliability sweep: classifier catalog commodity/fx + FRED-anchor (#633) · ThresholdEngine SIGSEGV fix (#634) · 30d retention (#635) · classifier-recall + cascade fixes (#630/#631/#632) · architecture-cards 11/11 (#626/#628/#629).
2026-06-05/06 — matrix-health restoration: #618–#625 · #589 cpuset · #603 verifier-alert. Earlier: counterweight #602/#613–#616 · WS3 #552–#587 · DSL PoC #381–#510 · Matrix MVP #215–#261.
