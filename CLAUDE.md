# CLAUDE.md [ATLAS project-level]

## PROJECT_OVERVIEW
name: ATLAS
purpose: financial data collection, processing, alerting, LLM extraction
stack: .NET10/C#14 | TimescaleDB | nerdctl/containerd | OTEL->Loki/Prom/Tempo->Grafana | Serilog | Polly

## EXECUTION_CONTEXT [HARD_STOP]
YOU_ARE_ON: mercury # this machine, the production server
✗ ssh mercury # you are already here
✗ ansible mercury # you are already here
✓ sudo nerdctl ... # direct command
✓ sudo systemctl ... # direct command
✓ ansible-playbook -i inventory/hosts.yml ... # deploys TO mercury, runs FROM mercury
rationale: ssh/ansible to localhost = unnecessary hop + templating issues + confusion
REMEMBER: run commands directly with sudo, not via ssh or ansible targeting mercury

## PROJECT_CONVENTIONS
compose files: compose.yaml, never docker-compose.yml
dockerfile files: Containerfile, never Dockerfile
development: devcontainer, never local install
rationale: runtime-agnostic (nerdctl|docker|podman) + compose v2 standard + clean host

## VERIFY [before_commit]
IF code change THEN verify it compiles BEFORE commit
METHODS:
  dotnet: {Project}/.devcontainer/compile.sh [--no-test]
  container: {Project}/.devcontainer/build.sh [--no-cache]
  fallback: ASK_USER("Can't verify - use devcontainer or skip?")
ANTI-PATTERN: "straightforward" -> commit anyway # this is how bugs ship

## PHASE_TAGS [going_forward]
At phase / epic completion:
  1. tag the merge-completion commit: `git tag -a dsl-poc-phase{N}-done <sha> -m "<outcome summary>"`
  2. `git push origin tag/dsl-poc-phase{N}-done` (or `--tags` selectively)
  3. add a brief entry to `docs/RELEASES.md` — phase outcome + tag reference
  4. `git rm` phase-specific working/iteration docs (plans are not kept on main; record each retirement in `docs/RELEASES.md` with its recovery pointer)
rationale: main = current-state reference docs only (see docs/README.md curation policy). Iteration history recoverable via tag (`git checkout <tag>` | `git show <tag>:<path>`). git log is the archive; tags are the named index.

## GIT_PUSH [HARD_STOP]
✗ NEVER git push without first running ALL tests for modified projects
✓ ALWAYS run {Project}/.devcontainer/compile.sh (without --no-test) before push
✓ VERIFY: 0 errors AND 0 warnings AND all tests pass
ENFORCE: tests pass AND build clean -> push allowed
PROCESS:
  1. identify modified projects (git diff --name-only)
  2. FOR each project: run compile.sh (with tests)
  3. IF any failure -> fix before push
  4. ONLY after all pass -> git push
rationale: broken tests = broken code = broken trust
hook: .claude/hooks/git-push-guard.sh # enforced by tooling
  marker: tree-hash-keyed (HEAD^{tree}) # committed content only, not commit hash
  caveat: HEAD^{tree} reflects committed state — uncommitted edits do NOT change tree hash
  survives: commit-after-test | cherry-pick | rebase | unrelated commits
  scope: write=per-worktree (filename suffix = sha1(toplevel)); read=global scan by tree-hash # any worktree's marker satisfies the gate
  format: "v2 tree <hash> <iso8601>" # version prefix rejects old commit-hash markers
  blocks: tree content changed AND not retested # source changes require retest

## DEPLOYMENT [HARD_STOP]
✗ NEVER edit /opt/ai-inference/compose.yaml directly
✓ ALWAYS use ansible for deployments
  playbook: ansible-playbook playbooks/deploy.yml --tags {service}
  inventory: deployment/inventory/hosts
rationale: compose.yaml = ansible-managed AND direct edit = config drift

## CONTAINER_BUILD
IMAGE: {service-name}:latest # fred-collector ✓ fredcollector ✗
  verify: /opt/ai-inference/compose.yaml
BUILD: {Project}/.devcontainer/build.sh [--no-cache]
  from: /home/james/ATLAS # monorepo context required
DEPLOY: ansible --tags {service} # never manual nerdctl

## DATABASE [ef_core]
SCHEMA: EF migrations only # no raw SQL scripts
SEED_DATA: EF HasData() | app-level seeding on startup
DEPLOYMENT: app handles own migrations/seeding
  rationale: single source of truth + version controlled + testable
DEBUG_ONLY: raw psql queries for inspection
  ✓ SELECT to verify state
  ✗ INSERT/UPDATE/DELETE to fix state # fix in code instead
PSQL_CONNECTION: sudo nerdctl exec timescaledb psql -U ai_inference -d atlas_data
  rationale: container=timescaledb, user=ai_inference, db=atlas_data
MIGRATIONS [HARD_STOP]:
  ✗ NEVER manually create migration .cs files
  ✓ ALWAYS use: dotnet ef migrations add {Name} --project {path}
  rationale: manual migration -> missing Designer.cs -> migration recorded but not applied
  required_files: {Migration}.cs + {Migration}.Designer.cs + ModelSnapshot.cs
  failure_mode: EF records in __EFMigrationsHistory but schema unchanged
  command: nerdctl compose exec -T {svc}-dev dotnet ef migrations add {Name} --project src/Data
ANTI:
  ✗ raw SQL scripts during deployment
  ✗ bypassing EF to seed/migrate
  ✗ manual database fixes # fix the root cause in the app
  ✗ manually creating migration files # missing Designer.cs

## BUILD [devcontainer]
compile: {Project}/.devcontainer/compile.sh [--no-test]
image: {Project}/.devcontainer/build.sh [--no-cache]
deploy: ansible-playbook playbooks/deploy.yml --tags {service}
dashboards: ansible-playbook playbooks/deploy.yml --tags dashboards # grafana auto-reloads
patterns: ansible-playbook playbooks/deploy.yml --tags patterns # hot reload, no rebuild
filter test: nerdctl compose exec -T {svc}-dev dotnet test --filter 'Name~{Test}'

## DATA_ML_CONTEXT
TRAINING_DATA: assume >=500 good docs available, never lowball ("30-50")
  rationale: high-yield sources are abundant -> estimate generously
VLLM_STRUCTURED: response_format (openai standard), never guided_json
  rationale: guided_json broken in vLLM 0.19
PROMPTS: host mount (/prompts) -> edit the host file, never container-only
  rationale: container edits lost on restart
ESTIMATE_GATE [data | vram | model tradeoff]:
  enumerate repo + filesystem FIRST -> then estimate
  never anchor on generic defaults ("30-50 docs", "LoRA hurts general quality")
  check: prior results from this project before claiming tradeoffs
  rationale: conservative defaults != this project's reality -> wastes a correction turn

## GIGO [garbage_in_garbage_out] [HARD_STOP]
Not a new rule — it's BOUNDARY_HANDLING applied to the OTHER direction. We validate the inputs a function RECEIVES; we dropped that same defensiveness for what a CALL SENDS. A call (fn | service | model | paid API) is a boundary too — validate what you hand it, symmetrically, or garbage propagates. Junk admitted at ANY boundary costs everything downstream — $, GPU/CPU, log signal, OUTPUT CORRECTNESS (a junk "entity" resolving to the WRONG instrument corrupts the matrix), stored-data integrity. $-cost is ONE symptom, not the frame.
  ✓ clean at the SOURCE where garbage is BORN — one fix covers every consumer
  ✗ gate each destination — each new consumer re-learns the lesson
  RECURRING (same disease, each time gated at the destination instead of cleaned at source):
    news-NER junk -> FRED search (#818 — gated at FRED via allowEconomicDiscovery)
    news-NER junk -> paid Gemini resolver (#823 — gated at the resolver) + wrong-ticker resolutions (FREE, silent, worse than the bill)
  -> ROOT fix: reject non-entity surfaces at extraction INGRESS (SentinelCollector CandidateSurfaceFilter) so NO downstream — FRED, Gemini, storage, the matrix — is fed garbage. The destination gates stay as defense-in-depth.

## INTENT_FIDELITY [code_embodies_the_spec's_why] [HARD_STOP]
PRINCIPLE: every line is intentional, traceable to a design decision WITH its justification — and the justification lives in the card/comment NEXT TO the code, not only in a plan. A spec that captures WHY is lost when code inherits only the WHAT; then code drifts into violating the design's ETHIC.
  a privileged/expensive/EXCEPTION path (frontier last-resort, raw-DB write, host restart, --user flag) exists for a SPECIFIC EARNED case -> GUARD it so it can't silently become a primary path, and WRITE the precondition where the code lives.
  worked example: gemini-resolver INTENT = "cheap lookups fan out in parallel; the frontier call is the RARE exception, earned only when all-cheap-failed on a genuinely hard entity, because THERE it pays dividends." Code kept the mechanism (call-on-miss) but lost the precondition (rare AND hard AND dividend-paying) -> trash firehose to a frontier model = ethic violation, invisible until it hit a bill.
ENFORCE (at a scarce-resource boundary — $/GPU/quota; apply as warranted, not dogmatic): gate(eligible-only) + fail-closed-cap(refuse past budget, never silent-pass) + burn-alert(BEFORE depletion; never ship a "calls>0 AND cost=$0" check as the alert — that is a corpse-detector, it fires after the money is gone) + honest-health(exercise the real work path, not cheap reachability) + business-test(RED-on-unfixed [[feedback_tests_validate_business_outcomes]]).
MECHANICS [D-entries — format spec: .claude/skills/architecture-cards/CARD_TEMPLATE.md §DECISIONS BLOCK]:
  D_ENTRY: `D-n <slug>: INTENT <why> / PRECOND <condition> / GUARD <class.method> @ file:line / TEST <TestClass.TestName>` in <Service>/AGENT_README.md DECISIONS block # scoped: exception paths | scarce-resource boundaries | non-obvious preconditions; `DECISIONS: none` legal; >~6 = smell
  ATOMIC_SET [change-all-or-none]: D-entry + `// INTENT(D-n):` comment at guard site + guard code + guard test # same discipline as the `:sig:` infix
  SUPERSESSION: rewrite the entry in the SAME PR as the code change; briefs name "supersedes D-n" explicitly; no tombstones
  CONFLICT [HARD_STOP]: brief contradicts a D-entry without named supersession -> STOP + report; never route around, never obey the stale entry # human/supervisor arbitrates, not the implementing agent
  GUARD_TEST: required per D-entry — construct the violation, assert refusal AT the boundary through the real flow, RED if the guard is deleted # contract: .claude/skills/intent-review/SKILL.md §GUARD_TEST_CONTRACT

## SENTINEL [llm_extraction] [arxiv:2512.24601]
MODEL_SIZE: >=30B parameters required # RLM needs coding ability
  rationale: extraction quality requires large models
  ✓ qwen2.5:32b-instruct, qwen3:32b, sentinel-extraction-v5
  ✗ 7b, 8b, 14b models # insufficient quality
CONTEXT: 32K required # RLM full-document decomposition
  ✗ reduce context # breaks extraction, causes context rot
INFERENCE_TOPOLOGY: vLLM(GPU) + llama.cpp(CPU) # ollama fully retired 2026-06-11 (ollama exodus)
  GPU: vllm-server (Qwen2.5-32B-AWQ; continuous batching/PagedAttention — no per-slot NUM_PARALLEL tuning)
  CPU: llama-server(DSL/GBNF rollback) | llama-cpu-rag(SecMaster RAG gen) | llama-cpu-embed(bge-m3 embeddings)
  ✗ propose ollama # no ollama container/engine remains; GGUF blobs live in the frozen ollama-format store, ro-mounted
GPU_OOM: restart vLLM first; never downgrade the model or reduce context

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
cards carry DECISIONS (numbered, guarded D-entries): touching a guard or contradicting an entry without a named "supersedes D-n" -> STOP # rule: INTENT_FIDELITY MECHANICS
cards:
  SecMaster:         SecMaster/AGENT_README.md         # resolve-entities != ResolveBatch; identity indep of collection; fuzzy-proposes/authoritative-confirms; ✗NotFound="not-in-table"; ✗bulk-preload; ✗gate-non-Equity-sector
  ThresholdEngine:   ThresholdEngine/AGENT_README.md   # WS3-projector=ONLY wired matrix_cells writer; ObservationEventSubscriber=UNWIRED/dead; Confidence XML-doc"informational only"=FALSE; ✗live-FRED-gRPC-writes-matrix_cells; ✗ascending-projector-read
  SentinelCollector: SentinelCollector/AGENT_README.md # news->matrix pipeline spans MacroSubstrate; `:sig:` infix=string contract change-all-or-none; signal-dim gates projection sector-dim does NOT; Shadow != Off (same cells written); ✗gate-entry-on-sector; ✗check-Mode-before-concluding-broken
  FredCollector:     FredCollector/AGENT_README.md     # catalog indep of instrument (SeriesId=FRED mnemonic != instr-id); AlfredBackfillService deliberately does NOT touch LastCollectedAt; ObservationChannel no reader=memory-growth; ✗expect-WARN-GRPC-unset; ✗ALFRED-backfill=advances-LastCollectedAt
  FinnhubCollector:  FinnhubCollector/AGENT_README.md  # candle/social/insider/calendars=dead-schema (tables empty); ObservationChannel FullMode=Wait+no-reader->BLOCKS; GetLatestEventTime=UtcNow-on-empty (not a new-data signal); ✗assume-non-Quote-data-flows
  AlphaVantageCollector: AlphaVantageCollector/AGENT_README.md # stream=scalar-only(Commodity/Economic); OHLCV never emits events; quota=in-mem(restart-wipes); TechnicalIndicator=scaffold (not collected); ✗assume-OHLCV-events ✗expect-values-in-Event ✗quota-survives-restart
  NasdaqCollector:   NasdaqCollector/AGENT_README.md   # DISABLED prod(NDL WAF); EventId=Ulid-per-read (not a stable dedup key); EventTypes filter=dead param; SecMaster Economic GUARD may silently reject; ✗treat-EventId-stable ✗assume-prod-running
  OfrCollector:      OfrCollector/AGENT_README.md      # FSI indep of gRPC-register; FSI composite+4 patterned subindices->macro_observations (raw; _EQUITY/_SAFE_ASSETS/_US/_AE excluded — no pattern); circuit-breaker=5 CONSECUTIVE fails->60s (not sliding-window); dual-write non-fatal; gRPC GetLatestEventTime=now() placeholder; ✗conflate-gRPC-register-with-REST-tag ✗assume-ALL-FSI-cols->matrix
  AlertService:      AlertService/AGENT_README.md      # UP by design (re-enabled+verified end-to-end Alertmanager->alert-service->ntfy `atlas-alert`, 2026-06-10 #656); appsettings routing != RoutingOptions class default; dedup=fingerprint-only; autofix rate-limit=static process-wide; ✗assume-202-means-sent
  CalendarService:   CalendarService/AGENT_README.md   # HTTP-only (no gRPC :5001); FRED allow-list ~18 releases (not all); event_time=synthetic DST-unaware; market endpoints bypass DB; Finnhub worker disabled; ✗assume-gRPC ✗trust-FRED-event_time-real
  MacroSubstrate:    MacroSubstrate/AGENT_README.md    # write=DO UPDATE heal-on-rewrite (not DO NOTHING); QueryAsync(AsOfDate+MappingVersionLabel simultaneously)->ArgumentException; not a running service (library+migrator only); ✗trust-README-DO-NOTHING ✗set-both-version-axes

## DATA_FLOW
Collectors ->gRPC:5001-> ThresholdEngine ->metrics-> Prometheus -> Alertmanager -> AlertService -> ntfy|email
Collectors ->gRPC:5001-> SecMaster (registration, fire-and-forget)
ThresholdEngine ->gRPC:5001-> SecMaster (resolution, context-based routing)
gRPC: internal only (container-to-container)
HTTP: 8080 internal, 50xx host
