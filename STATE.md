# ATLAS Supervisor STATE

Supervisor memory. Read first, write last. **Current truth + open items + pointers ONLY.** History → git log + PRs + memories. WRITE_GATE: product/phase truth, deploy-todos, open-asks — NOT per-turn progress, review verdicts, or merged-PR narration.

## EACH TURN START HERE
0. **FIRST TURN ONLY** — git audit (`git -C /home/james/ATLAS worktree list / branch / status / fetch + log HEAD..origin/main`); main checkout on `main` & current. [[feedback_git_session_start_discipline]] [[feedback_main_always_current]]
1. Poll NTFY `atlas-claude-reply` (user input first).
2. RE-READ this file + active plan/relevant docs THIS turn (not "read earlier"). Walk the pipeline backward before dispatching; plan-vs-prod mismatch → STOP + NTFY.

## WHERE WE ARE [2026-06-07]
Sentinel→matrix counterweight, digest-quality, and the crash/disk fixes are ✅ DONE + LIVE. Matrix = **signal × sector** (Sentinel news = fast-decaying LEADING counterweight to lagging FRED hard data); both halves flow. Digest healthy (LlmStatus Ok, 23 sig×11 sector, FRED anchor 165 cells, commodity/FX recognized). System stable; repo pristine (main only). [[project_matrix_feed_classifier]]
**Active focus:** Sentinel extraction **THROUGHPUT** — CPU 30B CoD is model/DRAM-bound at ~20 art/hr (queue ~3 days behind, draining slowly). CPU tuning + HW are exhausted/maxed, so the remaining lever being pursued is **input size**, via the CoD truncation experiment (below).

## OPEN (current truth)
- 🔬 **CoD truncation experiment — PRIMARY ACTIVE.** Hypothesis: full article text adds little over headline+lead[+tail] (templated/inverted-pyramid news) → truncate long articles pre-CoD → collapse the ~133s prefill → throughput multiple, with NO GPU / model-swap / coverage-cut. Spec `docs/superpowers/specs/2026-06-07-cod-truncation-value-experiment-design.md` + supervisor exec plan `docs/plans/cod-truncation-experiment-plan.md` (P0 harness+matching-GATE → P1 pilot → P2 sweep → P3 policy[NTFY] → P4 rollout[user-gated]; heads-down P0–P2). **Fidelity invariant: harness drives the REAL CoD path.** **NEXT: launch P0** (awaiting user go). [[feedback_benchmark_not_production]]
- **Throughput baseline (concluded — context for the experiment):** CPU `qwen3-30b-a3b` CoD ≈ 20 art/hr; tuned config LIVE (`3a802551`: `--parallel 1` + flash-attn + ubatch 2048 + KV-q8 — kept for the 32K-ctx quality fix; aggregate throughput ~flat regardless of CPU knobs). Monitoring LIVE: #636 `queue_depth`/`frontier_lag` + #638 per-stage (`cod_llm`/`slot_wait`/tokens) + Grafana "Extraction Queue Health" panels. HW maxed (4-channel TRX50, healthy, no failure). Fallback levers if the experiment fails: GPU-offload / smaller-model (vs ≥30B rule) / accept-lag. [[project_atlas_inference_topology]]
- **Codify the core-dump `LimitCORE=512M` drop-in into deployment IaC** — manual host file (`/etc/systemd/system/containerd.service.d/core-limit.conf`), lost on rebuild.
- **Recalibrate flagged natgas/ECB sectorWeights** vs observed betas (optional, hot-reloadable).
- **Hard-data natgas/ECB → matrix wiring** — the hard-data pattern signals evaluate but don't produce `matrix_cells` (DHHNGSP/ECBDFR not pushed to `macro_observations`); matrix entry is news-`:sig:`-only.
- Verifier verdicts not persisted (blocks auditing verifier strictness).
- #622 doc-sweep (`~/.ansible_vault_pass` refs); hooks README "commit-hash-keyed" → "head-keyed" wording.
- break-map doc `docs/sentinel-matrix-break-map-2026-06-05.md` premise SUPERSEDED → update or retire.
- Track (minor): llama-server 900s-timeouts on a few large articles (Polly-retried, small minority).
- Optional: #619 strict per-source parallel refactor.

## DEPLOY / RUNTIME NOTES
- Scoped deploy: `ansible-playbook playbooks/deploy.yml --tags <svc> -e "scoped_restart=true scoped_services=<svc>"`. **Always pass scoped flags** — a non-scoped/`[always]` run regenerates compose + resurrects `alert-service` (must STAY DOWN, user) — **incl. `--tags dashboards` (does a `compose up`): `sudo nerdctl stop alert-service` after every dashboards deploy.** `atlas_repo_path` defaults to main checkout → keep it current; verify running digest + `--no-cache` build. [[ansible_deploy_stale_image]] [[project_ansible_deploy_restart_behavior]]
- ansible vault: ansible.cfg uses the absolute path (`/home/james/.ansible_vault_pass`, #622) → plain `sudo ansible-playbook` resolves it. Prometheus rule reload: `sudo nerdctl exec prometheus kill -HUP 1`.
- DB inspect: `sudo nerdctl exec timescaledb psql -U ai_inference -d atlas_data` (SELECT only; never DML [[feedback_agent_no_raw_db_fixes]]). SecMaster DB = `atlas_secmaster`. Schemas: news obs `sentinel.extracted_observations`; matrix feed `public.macro_observations`; cells `public.matrix_cells`.
- vLLM = `Qwen2.5-32B-Instruct-AWQ`, 32K ctx (classifier + GPU). CPU CoD = llama-server (**`--parallel 1`, 32K-ctx/slot, flash-attn on, ubatch 2048, KV-q8**, in `deployment/ansible/group_vars/all.yml`). cpuset: llama-server 0-23, ollama-cpu-gen 24-39, ollama-cpu-embed 40-47. IaC changes must be COMMITTED in the same step (uncommitted drift blocks the next deploy's pre-flight).

## SUPERVISOR OPERATING NOTES
- All code/test/build/deploy → background subagents (worktree-isolate parallel code agents). Verify agent DATA vs DIAGNOSIS (trust data, independently check inferences) before recording/surfacing. [[feedback_verify_agent_inferences]]
- Autonomy: dispatch/review/fix/merge/deploy autonomous; NTFY `atlas-claude-ask` only on genuine blocker / architectural fork / milestone (SINGLE-LINE bodies). End turns `▶ continuing autonomously` vs `🛑 DECISION NEEDED`.
- Merge gate: see SKILL `MERGE_GATE` — the `pr-review-toolkit:review-pr <N>` Skill (with a valid cwd) writes `/tmp/atlas-test-markers/pr-reviewed-<N>`; re-run after new commits. Push to main BLOCKED → branch+PR always; selective `git add -- <paths>`. [[project_pr_review_marker_staleness]]

## ARCHIVE (git log + tags)
2026-06-07 — extraction throughput: queue/frontier monitoring (#636) · per-stage OTEL timing #638 (V2-path + real `timings`) · CPU tuning `3a802551` (`--parallel 1`+flash-attn+ubatch+KVq8, 32K-ctx restored) · bottleneck read = model-bound CPU 30B ~20/hr · memory-hygiene skill+hook (#637).
2026-06-06/07 — digest-quality + reliability sweep: classifier catalog `commodity/fx` + digest FRED-anchor 90d lookback (#633) · ThresholdEngine Roslyn-SIGSEGV serialize+pre-warm (#634, +containerd `LimitCORE=512M` cap) · 30-day `raw_content` retention freed 11.1GB (#635) · classifier-recall natgas/ECB legs + SecMaster-prune/Fred-delete-cascade fixes (#630/#631/#632) · supervisor SKILL.md sharpening (#627) · architecture-cards 11/11 services (#626/#628/#629).
2026-06-05/06 — matrix-health restoration: #618 signal→pattern · #619 grounding resolver · #620 23505 · #621 weights · #622 vault · #623 digest VIEW · #624 FRED reconciliation · #625 classifier-recall · #589 cpuset · #603 verifier-alert. Earlier: counterweight #602/#613/#614/#615/#616 · WS3 #552–#587 · DSL PoC #381–#510 · Matrix MVP #215–#261 · digest plumbing #607–#611.
