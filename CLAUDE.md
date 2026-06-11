# CLAUDE.md [ATLAS project-level]

## PROJECT_OVERVIEW
name: ATLAS
purpose: financial data collection, processing, alerting, LLM extraction
stack: .NET10/C#14 | TimescaleDB | nerdctl/containerd | OTEL→Loki/Prom/Tempo→Grafana | Serilog | Polly

## EXECUTION_CONTEXT [HARD_STOP]
YOU_ARE_ON: mercury # this machine, the production server
✗ ssh mercury # you are already here
✗ ansible mercury # you are already here
✓ sudo nerdctl ... # direct command
✓ sudo systemctl ... # direct command
✓ ansible-playbook -i inventory/hosts.yml ... # deploys TO mercury, runs FROM mercury
rationale: ssh/ansible to localhost = unnecessary_hop + templating_issues + confusion
REMEMBER: run commands directly with sudo, not via ssh or ansible targeting mercury

## PROJECT_CONVENTIONS
compose_files: compose.yaml ¬ docker-compose.yml
dockerfile_files: Containerfile ¬ Dockerfile
development: devcontainer ¬ local_install
rationale: runtime_agnostic(nerdctl|docker|podman) + compose_v2_standard + clean_host

## VERIFY [before_commit]
IF code_change THEN verify(compiles) BEFORE commit
METHODS:
  dotnet: {Project}/.devcontainer/compile.sh [--no-test]
  container: {Project}/.devcontainer/build.sh [--no-cache]
  fallback: ASK_USER("Can't verify - use devcontainer or skip?")
¬PATTERN: "straightforward" → commit_anyway # this_is_how_bugs_ship

## PHASE_TAGS [going_forward]
At phase / epic completion:
  1. tag the merge-completion commit: `git tag -a dsl-poc-phase{N}-done <sha> -m "<outcome summary>"`
  2. `git push origin tag/dsl-poc-phase{N}-done` (or `--tags` selectively)
  3. add a brief entry to `docs/RELEASES.md` — phase outcome + tag reference
  4. `git rm` phase-specific working/iteration docs from `docs/plans/`
rationale: main = reference data + current/active plans. Iteration history recoverable via tag (`git checkout <tag>` | `git show <tag>:<path>`). git log is the archive; tags are the named index.

## GIT_PUSH [HARD_STOP]
✗ NEVER git push without first running ALL tests for modified projects
✓ ALWAYS run {Project}/.devcontainer/compile.sh (without --no-test) before push
✓ VERIFY: 0 errors AND 0 warnings AND all tests pass
ENFORCE: tests_pass ∧ build_clean → push_allowed
PROCESS:
  1. identify_modified_projects(git diff --name-only)
  2. FOR each project: run compile.sh (with tests)
  3. IF any_failure → fix_before_push
  4. ONLY after all_pass → git push
rationale: broken_tests = broken_code = broken_trust
hook: .claude/hooks/git-push-guard.sh # enforced_by_tooling
  marker: tree-hash-keyed (HEAD^{tree}) # committed_content_only, ¬commit_hash
  caveat: HEAD^{tree} reflects committed state — uncommitted edits do NOT change tree hash
  survives: commit_after_test | cherry-pick | rebase | unrelated_commits
  scope: write=per-worktree (filename suffix = sha1(toplevel)); read=global scan by tree-hash # any_worktree's_marker_satisfies_gate
  format: "v2 tree <hash> <iso8601>" # version prefix rejects old commit-hash markers
  blocks: tree_content_changed ∧ ¬retested # source_changes_require_retest

## DEPLOYMENT [HARD_STOP]
✗ NEVER edit /opt/ai-inference/compose.yaml directly
✓ ALWAYS use ansible for deployments
  playbook: ansible-playbook playbooks/deploy.yml --tags {service}
  inventory: deployment/inventory/hosts
rationale: compose.yaml = ansible-managed ∧ direct_edit = config_drift

## CONTAINER_BUILD
IMAGE: {service-name}:latest # fred-collector ✓ fredcollector ✗
  verify: /opt/ai-inference/compose.yaml
BUILD: {Project}/.devcontainer/build.sh [--no-cache]
  from: /home/james/ATLAS # monorepo context required
DEPLOY: ansible --tags {service} # ¬nerdctl_manual

## DATABASE [ef_core]
SCHEMA: EF migrations only # ¬raw_sql_scripts
SEED_DATA: EF HasData() | app-level seeding on startup
DEPLOYMENT: app handles own migrations/seeding
  rationale: single_source_of_truth + version_controlled + testable
DEBUG_ONLY: raw psql queries for inspection
  ✓ SELECT to verify state
  ✗ INSERT/UPDATE/DELETE to fix state # fix_in_code_instead
PSQL_CONNECTION: sudo nerdctl exec timescaledb psql -U ai_inference -d atlas_data
  rationale: container=timescaledb, user=ai_inference, db=atlas_data
MIGRATIONS [HARD_STOP]:
  ✗ NEVER manually create migration .cs files
  ✓ ALWAYS use: dotnet ef migrations add {Name} --project {path}
  rationale: manual_migration → missing_Designer.cs → migration_recorded_but_not_applied
  required_files: {Migration}.cs + {Migration}.Designer.cs + ModelSnapshot.cs
  failure_mode: EF records in __EFMigrationsHistory but schema unchanged
  command: nerdctl compose exec -T {svc}-dev dotnet ef migrations add {Name} --project src/Data
ANTI:
  ✗ raw SQL scripts during deployment
  ✗ bypassing EF to seed/migrate
  ✗ manual database fixes # fix_root_cause_in_app
  ✗ manually creating migration files # missing Designer.cs

## BUILD [devcontainer]
compile: {Project}/.devcontainer/compile.sh [--no-test]
image: {Project}/.devcontainer/build.sh [--no-cache]
deploy: ansible-playbook playbooks/deploy.yml --tags {service}
dashboards: ansible-playbook playbooks/deploy.yml --tags dashboards # grafana_auto_reloads
patterns: ansible-playbook playbooks/deploy.yml --tags patterns # hot_reload_no_rebuild
filter_test: nerdctl compose exec -T {svc}-dev dotnet test --filter 'Name~{Test}'

## DATA_ML_CONTEXT
TRAINING_DATA: assume(≥500 good_docs available) ¬ lowball("30-50")
  rationale: high_yield_sources_abundant → estimate_generously
VLLM_STRUCTURED: response_format(openai_standard) ¬ guided_json
  rationale: guided_json broken in vLLM 0.19
PROMPTS: host_mount(/prompts) → edit_host_file ¬ container_only
  rationale: container edits lost on restart
ESTIMATE_GATE [data | vram | model_tradeoff]:
  enumerate(repo + filesystem) FIRST → estimate
  ¬anchor: generic_defaults ("30-50 docs", "LoRA hurts general quality")
  check: prior_results(this_project) before claiming tradeoffs
  rationale: conservative_defaults ≠ this_project_reality → wastes_correction_turn

## SENTINEL [llm_extraction] [arxiv:2512.24601]
MODEL_SIZE: ≥30B parameters required # RLM needs coding ability
  rationale: extraction_quality_requires_large_models
  ✓ qwen2.5:32b-instruct, qwen3:32b, sentinel-extraction-v5
  ✗ 7b, 8b, 14b models # insufficient_quality
CONTEXT: 32K required # RLM full-document decomposition
  ✗ reduce_context # breaks extraction, causes context_rot
OLLAMA: NUM_PARALLEL=1 # fits 32K context + model on GPU
  ✗ NUM_PARALLEL>1 # forces CPU offload, degrades performance
GPU_OOM: restart_ollama_first ¬ downgrade_model ¬ reduce_context

## SERVICES [monorepo]
collectors: FredCollector, AlphaVantageCollector, NasdaqCollector, FinnhubCollector, OfrCollector
processing: ThresholdEngine
alerting: AlertService
calendar: CalendarService
metadata: SecMaster
substrate: MacroSubstrate
mcp: FredCollector/mcp, ThresholdEngine/mcp, FinnhubCollector/mcp, OfrCollector/mcp, SecMaster/mcp, WhisperService/mcp
shared: Events/, deployment/, docs/

## SERVICE_ARCHITECTURE [read-first] [HARD_STOP]
BEFORE reasoning about a service's architecture / API / data-model / resolution flow:
  READ <Service>/AGENT_README.md (the dense agent card) FIRST.
  rationale: card front-loads negative-space (does-NOT / on-miss / invariants / DISTINCTIONS / GOTCHAS) an endpoint catalog can't convey.
✗ guess a service's shape from method names / the endpoint table # read the card
✗ "fix" a symptom by violating a card INVARIANT (e.g. bulk-preload, backfill-to-green, raw DB fix)
cards:
  SecMaster:         SecMaster/AGENT_README.md         # resolve-entities≠ResolveBatch; identity⊥collection; fuzzy-proposes/authoritative-confirms; ✗NotFound="not-in-table"; ✗bulk-preload; ✗gate-non-Equity-sector
  ThresholdEngine:   ThresholdEngine/AGENT_README.md   # WS3-projector=ONLY wired matrix_cells writer; ObservationEventSubscriber=UNWIRED/dead; Confidence XML-doc"informational only"=FALSE; ✗live-FRED-gRPC-writes-matrix_cells; ✗ascending-projector-read
  SentinelCollector: SentinelCollector/AGENT_README.md # news→matrix pipeline spans MacroSubstrate; `:sig:` infix=string contract change-all-or-none; signal-dim gates projection sector-dim does NOT; Shadow≠Off(same cells written); ✗gate-entry-on-sector; ✗check-Mode-before-concluding-broken
  FredCollector:     FredCollector/AGENT_README.md     # catalog⊥instrument(SeriesId=FRED mnemonic≠instr-id); AlfredBackfillService deliberately¬touches LastCollectedAt; ObservationChannel no reader=memory-growth; ✗expect-WARN-GRPC-unset; ✗ALFRED-backfill=advances-LastCollectedAt
  FinnhubCollector:  FinnhubCollector/AGENT_README.md  # candle/social/insider/calendars=dead-schema(tables ∅); ObservationChannel FullMode=Wait+no-reader→BLOCKS; GetLatestEventTime=UtcNow-on-empty(¬new-data-signal); ✗assume-non-Quote-data-flows
  AlphaVantageCollector: AlphaVantageCollector/AGENT_README.md # stream=scalar-only(Commodity/Economic); OHLCV ¬emit-events; quota=in-mem(restart-wipes); TechnicalIndicator=scaffold(¬collected); ✗assume-OHLCV-events ✗expect-values-in-Event ✗quota-survives-restart
  NasdaqCollector:   NasdaqCollector/AGENT_README.md   # DISABLED prod(NDL WAF); EventId=Ulid-per-read(¬stable-dedup-key); EventTypes filter=dead param; SecMaster Economic GUARD may silently reject; ✗treat-EventId-stable ✗assume-prod-running
  OfrCollector:      OfrCollector/AGENT_README.md      # FSI⊥macro/register; circuit-breaker=5 CONSECUTIVE fails→60s(¬sliding-window); dual-write non-fatal; gRPC GetLatestEventTime=now() placeholder; ✗conflate-gRPC-register-with-REST-tag ✗assume-FSI-macro
  AlertService:      AlertService/AGENT_README.md      # UP by design (re-enabled+verified end-to-end Alertmanager→alert-service→ntfy `atlas-alert`, 2026-06-10 #656); appsettings routing≠RoutingOptions class default; dedup=fingerprint-only; autofix rate-limit=static process-wide; ✗assume-202-means-sent
  CalendarService:   CalendarService/AGENT_README.md   # HTTP-only(¬gRPC :5001); FRED allow-list ~18 releases(¬all); event_time=synthetic DST-unaware; market endpoints bypass DB; Finnhub worker disabled; ✗assume-gRPC ✗trust-FRED-event_time-real
  MacroSubstrate:    MacroSubstrate/AGENT_README.md    # write=DO UPDATE heal-on-rewrite(¬DO NOTHING); QueryAsync(AsOfDate+MappingVersionLabel simultaneously)→ArgumentException; ¬a running service(library+migrator only); ✗trust-README-DO-NOTHING ✗set-both-version-axes

## DATA_FLOW
Collectors →gRPC:5001→ ThresholdEngine →metrics→ Prometheus → Alertmanager → AlertService → ntfy|email
Collectors →gRPC:5001→ SecMaster (registration, fire-and-forget)
ThresholdEngine →gRPC:5001→ SecMaster (resolution, context-based routing)
gRPC: internal_only (container-to-container)
HTTP: 8080 internal, 50xx host
