# CLAUDE.md [ATLAS project-level]

## PROJECT_OVERVIEW
purpose: financial data collection, processing, alerting, LLM extraction
stack: .NET10/C#14 | TimescaleDB | nerdctl/containerd | OTEL->Loki/Prom/Tempo->Grafana | Serilog | Polly

## EXECUTION_CONTEXT [HARD_STOP]
YOU_ARE_ON: mercury # this machine, the production server
✗ ssh mercury | ansible mercury # you are already here -> an extra hop to yourself
✓ sudo nerdctl ... | sudo systemctl ... # direct
✓ ansible-playbook ... # runs FROM mercury and deploys TO it; inventory sets ansible_connection: local

## PROJECT_CONVENTIONS
compose.yaml never docker-compose.yml | Containerfile never Dockerfile | devcontainer never local install
rationale: runtime-agnostic (nerdctl|docker|podman) + compose v2 + clean host

## WHERE_WORK_LANDS [canonical routing — every other copy POINTS here, never restates]
you found or learned X -> write it HERE, in the SAME PR as the work:
  defect | measurement debt | deferred work | parked epic -> docs/BACKLOG.md, WITH the measurement that
    makes it re-checkable # without a number the next agent cannot tell whether it is still true.
    Close an entry in the PR that fixes it; no tombstones.
  engineering | deploy | observability rule                -> this file
  failure mode that RECURRED (twice, not once)             -> .claude/skills/supervisor-mode/LESSONS.md
    # a FIRST occurrence goes to docs/BACKLOG.md, which is what makes the SECOND one recognisable
  service shape | invariant | earned-exception precondition -> <Service>/AGENT_README.md D-entry
  phase | epic outcome                                      -> git tag + docs/RELEASES.md # PHASE_TAGS
  what happened                                             -> git log + the PR body # never a doc
STATE.md [supervisor-owned, repo root]: DISPOSABLE working memory for the epic in flight.
  UNSEARCHABLE: untracked + gitignored, and `grep -r` / the Grep tool honour .gitignore, so a repo-wide
    search silently misses it # read it by explicit path before reporting "not found in the repo"
  ✗ never commit | push | PR | `git clean -x` it # no CURRENT copy in git to restore from
  ✗ never stash | restore | checkout supervisor-owned files (.claude/skills/supervisor-mode/**) to get a
    clean tree # `git checkout -b` and `git pull --ff-only` already preserve dirty tracked files when the
    new ref does not touch them — proceed as-is
  ✗ nothing durable goes in it — if it would outlive this epic, it belongs at a row above # a store with
    an in-flow and no out-flow becomes a diary
  epic-boundary reset, and the audit that gates it: `scripts/new-epic.sh --help`

## VERIFY [before_commit]
IF code change THEN verify it compiles BEFORE commit # "straightforward -> commit anyway" is how bugs ship
  dotnet: {Project}/.devcontainer/compile.sh [--no-test]
  container: {Project}/.devcontainer/build.sh [--no-cache]
  can't verify -> ASK_USER
FILTERED RUN: nerdctl compose exec -T {svc}-dev dotnet test --filter 'DisplayName~{Test}' # xUnit exposes
  DisplayName and FullyQualifiedName, NOT Name. `Name~` matches ZERO tests and STILL EXITS 0, so a run
  that tested nothing reads as a pass. Run it in the devcontainer: dotnet exists on the HOST too, and a
  bare `dotnet test` silently becomes a host run
OWNED: each compile.sh (+ sentinel-edge typecheck.sh/dev.sh) owns a compose project keyed to its worktree
  (scripts/devcontainer-owner.sh): atlas-<sha1(worktree)[0:12]>-<slug>, the same key mark-tests-passed.sh uses.
  N agents in N worktrees compile SIMULTANEOUSLY # never sequence them, never wait
  compile.sh proves /workspace is its OWN tree (inode match); mark-tests-passed.sh refuses without that
    attestation, and refuses one over 3h old
  cleanup: teardown on EXIT + a reaper on every start removing atlas-* state whose worktree is gone # SIGKILL cannot trap
  ✗ no .devcontainer compose file publishes a host port # exec-only. Two DECLARED interactive exceptions:
    sentinel-edge compose.ports.yaml (8787) and WhisperService compose.dev.yaml (8090)
  rationale: concurrent runs used to test another worktree's tree and still write a push marker
KNOWN [pre-existing, not concurrency]: two DIFFERENT services in ONE worktree collide on Events/src/*/obj by UID
  root-user devcontainers (AlphaVantageCollector, CalendarService, FinnhubCollector, NasdaqCollector, Reports)
    vs vscode/uid-1000 -> "Access to the path '/workspace/Events/.../obj/<guid>.tmp' is denied"
  recovery: sudo rm -rf <worktree>/Events/src/*/{obj,bin} # ✗ not fixed by serializing: file ownership, not a race

## PHASE_TAGS [at phase / epic completion]
  1. `git tag -a <epic-slug>-done <sha> -m "<outcome>"` # insert `-phase<N>` only for a phased epic
  2. `git push origin <epic-slug>-done` # bare tag name; `refs/tags/<name>` behaves identically
     ✗ `git push origin tag <name>` # the guard DENIES this spelling
     ✗ `git push --tags` # gates on the CURRENT BRANCH, not on the tag
     ✗ create and push in ONE bash call # the guard resolves the refspec before the chain runs, so the
       push is denied AND the tag is never created. Tag, then push, as two separate calls.
  3. entry in `docs/RELEASES.md` — outcome + tag reference
  4. `git rm` the phase's working/iteration docs; record each retirement in RELEASES.md with its
     recovery pointer `git show <tag>:<path>`
EXCEPTION [never retire]: a DO-NOT-BUILD spec is permanent and ADR-shaped -> step 4 never fires for it
  # an implemented plan is survived by its code; a rejected one is survived by nothing
curation, and that merging a plan is NOT approving it (its Status line governs): docs/README.md §Curation policy

## GIT_PUSH [HARD_STOP]
✗ NEVER push without running ALL tests for modified projects: compile.sh (no --no-test), 0 errors AND 0 warnings AND green
PROCESS: git diff --name-only -> compile.sh per project -> fix any failure -> only then push
hook: .claude/hooks/git-push-guard.sh
  marker "v2 tree <hash> <iso8601>" -- a git TREE, matched against the PUSHED BRANCH's tree; CWD's HEAD
    only for a bare `git push`
  committed content only # compile.sh on a dirty tree attests HEAD, NOT what it ran (stderr warning only)
  survives: commit-after-test | cherry-pick | rebase
  does NOT survive an unrelated commit: the root tree covers EVERY tracked path, so a docs-only commit
    remaps it too. Docs pushes get through on the hook's "Rule 1.5" docs/config exemption (feature branch),
    never on tree survival.
  scope: write=per-worktree (suffix sha1(toplevel)); read=global scan by tree-hash # any worktree's marker satisfies it
rationale: broken tests = broken code = broken trust

## DEPLOYMENT [HARD_STOP]
✗ NEVER edit /opt/ai-inference/compose.yaml directly # ansible-managed; direct edit = config drift
compose-service tag [SCOPED — the default]:
  ansible-playbook playbooks/deploy.yml --tags {service} --skip-tags build -e "scoped_restart=true scoped_services={service}"
  scoped_services must name a COMPOSE service, not merely an ansible tag # the scoped task filters
    `label=com.docker.compose.service=${svc}` then fails "SCOPED-RESTART FAILED" if the target is not running,
    so a tag-only name matches nothing and FAILS. Grep those two strings, never a line number.
    Most ansible tags name no compose service. Derive the set, never recall it:
      sudo nerdctl compose -f /opt/ai-inference/compose.yaml config --services
    It will not warn you that nasdaq-collector has none, or that macro-substrate's is migrate-macro-substrate.
  --skip-tags build deploys the CURRENT :latest, it does NOT build # 18 of the 27 DEPLOYABLE service tags
    carry ONLY the build task -> this form runs zero tag-scoped tasks. Build first (CONTAINER_BUILD).
    The 27 excludes macro-substrate and nasdaq-collector, which are build-only (a one-shot migrator, and
    commented out of compose). The 9 that also carry non-build tasks: alert-service, llama-cpu-embed,
    llama-cpu-rag, llama-server, secmaster, sentinel-collector, threshold-engine, trafilatura (which
    rebuilds on a changed context even under --skip-tags build), vllm-server.
non-service tag [dashboards | patterns | alerting | monitoring | sentinel-prompts | ...]: --tags {tag}
  --skip-tags always # only that tag's own tasks; command catalogue in deployment/README.md
✗ bare `--tags {anything}` # UNCONDITIONAL full-stack restart, not a conditional one: the task
  "Remove existing compose.yaml to force regeneration" runs state:absent under tags:[always] with no `when:`,
  the next task re-templates it, so compose_file.changed is ALWAYS true and the atlas systemd state resolves
  to 'restarted'. = compose down/up of EVERY service incl a ~4min vLLM GPU reload, and it
  RESURRECTS a deliberately-stopped alert-service. The two escapes are `--skip-tags always` and `-e scoped_restart=true`.
grafana alerting: `--tags alerting --skip-tags always` THEN `sudo nerdctl restart grafana` # provisioning is
  startup-loaded AND grafana lives in the separate OTEL stack, so no ansible task restarts it.
  Dashboards differ — they auto-reload (updateIntervalSeconds: 30), no restart.
inventory: deployment/ansible/inventory/hosts.yml # ansible.cfg default; run from deployment/ansible/
VERIFY_TRAP: `nerdctl inspect <svc>` RETURNS THE IMAGE, NOT THE CONTAINER # every service shares a name between the
  two and bare inspect resolves the image first, yielding a plausible .Created that is the BUILD time -> a deploy
  "verified" that way compared the fresh image to itself. Use `nerdctl container inspect`.
AUTOFIX [runner ARMED, deployer DISARMED — check, never assume]:
  autofix-runner NEVER deploys: alert -> queue -> a scoped Claude session opens a PR and STOPS # autofix.sh
    `deny_spellings ansible ... systemctl`, and DENY is the side the harness actually enforces
  it IS running: alert-service Up, Channels__AutoFix__Enabled=true, autofix-runner.timer enabled+active, queue
    non-empty. Each run DEFERS (exit 75) while an interactive `claude` lives, so it looks idle and is not.
  autofix-watcher is the ONLY deploying half, on a HUMAN merge of an autofix PR: `git checkout main; git pull`
    in the SHARED working tree (no dirty-tree check) then `deploy.yml --tags "$services"` — no --skip-tags, no
    scoped_restart = FULL stack + ~4min vLLM reload, retried every 5min UNBOUNDED on failure.
  its timer is `disabled` and deploy.yml re-enforces that on every --tags autofix|alert-service run, so NO
    alert can arm it — only a human `systemctl enable --now autofix-watcher.timer`. Check is-enabled, assume nothing.

## CONTAINER_BUILD
IMAGE: {compose-service}:latest, sole exception migrate-macro-substrate -> macro-substrate-migrator;
  MCP names drop the hyphen (fredcollector-mcp) — read the `image:` line in /opt/ai-inference/compose.yaml
BUILD: {Project}/.devcontainer/build.sh [--no-cache] # runnable from any cwd
DEPLOY: build first, then the scoped form above, never manual nerdctl # 12 of the 14 build.sh print a
  "To deploy:" line carrying the BARE `--tags` form this file forbids

## DATABASE [ef_core]
SCHEMA: EF migrations only # single source of truth, versioned, testable
SEED: EF HasData() | app-level seeding on startup; the app runs its own migrations
PSQL [DEBUG_ONLY] [HARD_STOP — no hook gates psql; this prose is the entire guard]:
  sudo nerdctl exec timescaledb psql -U ai_inference -d atlas_data
  ✓ SELECT, and read-only catalog meta-commands (\dt, \d, \l), to verify state
  ✗ ANY other statement or meta-command # the CLASS, not a verb list: INSERT | UPDATE | DELETE | TRUNCATE |
    ALTER (incl ALTER SYSTEM) | CREATE | DROP | GRANT | COPY ... FROM | VACUUM FULL | REINDEX | LOCK |
    SELECT pg_terminate_backend() | SELECT setval() | \gexec | \! are EXAMPLES. Fix the cause in the app.
  restate "psql is SELECT-only" VERBATIM in every dispatch brief # necessary but NOT sufficient:
    an agent that HAD the rule still ALTERed prod
TABLES: atlas_data -> sentinel.extracted_observations (news obs) | sentinel.raw_content |
  public.macro_observations (matrix feed) | public.matrix_cells | public.sector_regimes ; atlas_secmaster -> source_mappings | instruments
MIGRATIONS [HARD_STOP]:
  ✗ NEVER hand-write a migration .cs # missing Designer.cs -> EF records it in __EFMigrationsHistory, schema unchanged
  ✓ nerdctl compose exec -T {dev-svc} sh -c "cd /workspace/{Svc}/src && dotnet ef migrations add {Name} --output-dir Data/Migrations"
    {dev-svc} = the service name in {Svc}/.devcontainer/compose.yaml # plain `dev` for CalendarService and FinnhubCollector, an undeducible slug elsewhere
    cd + --output-dir exceptions: CalendarService -> src, Migrations | MacroSubstrate -> src/MacroSubstrate, Data/Migrations
    ✗ `--project src/Data` # Data/ is a FOLDER, not a project: `git ls-files | grep -E '/src/Data/.*\.csproj$'`
      returns 0. It resolves to {Svc}/src/src/Data, dies MSB1009, and leaves a stray src/src/obj
    `--context {Svc}DbContext` REQUIRED where the reference graph reaches a SECOND DbContext # MEASURED on
      SentinelCollector (references MacroSubstrate.csproj -> bare form is ambiguous); INFERRED, not executed,
      for FredCollector | OfrCollector | ThresholdEngine, which hold the same reference
    `dotnet tool restore` FIRST if dotnet-ef is missing # local tool manifest, not a global install
  required: {Migration}.cs + {Migration}.Designer.cs + ModelSnapshot.cs
  PARTIAL INDEX: EF expresses it natively -- .HasFilter("\"col\" = TRUE"), no raw-SQL escape needed
    ✗ bare `ON CONFLICT (col)` is NOT backed by a partial unique index # the arbiter needs a predicate implying
      the index's, else 42P10. Write `ON CONFLICT (col) WHERE <pred>` -- worked case SecMaster/AGENT_README.md
ANTI: ✗ raw SQL during deployment ✗ bypassing EF to seed/migrate ✗ manual DB fixes
      ✗ backfill-to-green # writing data to turn a red metric green is fixing the dashboard, not the cause

## DATA_ML_CONTEXT
VLLM_STRUCTURED: response_format (openai standard), never guided_json # guided_json broken in vLLM 0.19
PROMPTS: edit the REPO, never the host mount and never the container
  ✓ SentinelCollector/src/prompts/     -> /opt/ai-inference/prompts/sentinel -> container /prompts
  ✓ SentinelCollector/src/cod-prompts/ -> /opt/ai-inference/prompts/cod      -> container /prompts/cod
  ✗ /opt/ai-inference/prompts/** # deploy.yml's "Sync ... prompts from repo (overwrites host edits)" tasks
    copy with force:true, so host edits are CLOBBERED next deploy; ansible-gate-guard denies the write
  ✗ inside the container # lost on restart; the host mount is what the container reads
  ✗ never version a prompt in its FILENAME; an unreferenced prompt is DELETED, not parked # git is the history
  hot-tune on the host to iterate; tuning worth keeping must land in the repo path
ESTIMATE_GATE [data | vram | model tradeoff]: enumerate the repo and filesystem FIRST, then estimate, and
  check THIS project's prior measurements before claiming a tradeoff # generic defaults ("30-50 docs",
  "LoRA hurts quality") are not our reality -- high-yield sources have been abundant every time anyone
  counted. COUNT, never swap one unmeasured number for another

## GIGO [garbage_in_garbage_out] [HARD_STOP]
Clean at the SOURCE where garbage is BORN; never gate each destination # derivation: ~/.claude/CLAUDE.md
  §BOUNDARY_HANDLING (machine-local, untracked). Broken twice here, both as destination gates:
  #818 FRED series-search, #823 paid resolver
$ is ONE symptom, not the frame # a junk "entity" resolving to the WRONG instrument corrupts
  public.matrix_cells, and the free wrong-ticker resolutions cost more than the bill did
ROOT: reject non-entity surfaces at extraction INGRESS (SentinelCollector CandidateSurfaceFilter); destination
  gates stay as defense-in-depth # what it deliberately does NOT catch: SentinelCollector/AGENT_README.md

## INTENT_FIDELITY [code_embodies_the_spec's_why] [HARD_STOP]
PRINCIPLE: every line traces to a design decision, and the justification lives NEXT TO the code (card or comment), not
  only in a plan # code that inherits the WHAT without the WHY drifts into violating the design's ethic
  a privileged/expensive/EXCEPTION path (frontier last-resort, raw-DB write, host restart, --user flag) exists for a
  SPECIFIC EARNED case -> GUARD it so it cannot silently become a primary path, and WRITE the precondition at the code
  # gemini-resolver kept the mechanism (call-on-miss) and lost the precondition (earned only when all-cheap-failed) -> frontier firehose, invisible until the bill
ENFORCE at a scarce-resource boundary ($/GPU/quota; as warranted, not dogmatic): gate(eligible-only) +
  fail-closed-cap(refuse past budget, never silent-pass) + burn-alert BEFORE depletion (never ship "calls>0 AND
  cost=$0" — that is a corpse-detector, it fires after the money is gone) + honest-health(exercise the real work path,
  not reachability) + business-test(RED-on-unfixed [[feedback_tests_validate_business_outcomes]]).
MECHANICS [format + scope = .claude/skills/architecture-cards/CARD_TEMPLATE.md §DECISIONS BLOCK, never restated here]:
  WHEN: exception path | scarce-resource boundary | non-obvious precondition -> D-n entry in <Service>/AGENT_README.md
  ATOMIC_SET [all-or-none]: D-entry + `// INTENT(D-n):` at the guard site + guard code + guard test
  SUPERSESSION: rewrite the entry in the SAME PR as the code change; briefs name "supersedes D-n"; no tombstones
  CONFLICT [HARD_STOP]: brief contradicts a D-entry without named supersession -> STOP + report # never route around,
    never obey the stale entry; a human arbitrates, not the implementing agent
  GUARD_TEST: violation -> refusal AT the boundary through the real flow; mock ONLY the external client; RED if the
    guard is deleted # contract .claude/skills/intent-review/SKILL.md §GUARD_TEST_CONTRACT

## OBSERVABILITY [user scar tissue: "too many services non-functional due to lack of observability"]
✗ never demote a visible signal to Info+metric without a WIRED alert
✓ keep a VISIBLE Warning on persistent dependency-unavailability; startup banners STAY Warning # boot-loop visibility
A SIGNAL CAN ALSO BE DEMOTED WITH NOBODY DECIDING TO: before removing or changing a mechanism, enumerate what was
  OBSERVING it and pin each with a test that fires on the REAL path # a signal riding on a bug dies with the fix, and
  the fix looks correct -- missed TWICE on GeminiResolverNotResolving, the second time by the round that fixed the first
HEALTH IS TEMPO, NOT LOKI: prod log level defaults to Warning, so a HEALTHY container emits NOTHING — silence is the
  designed steady state, never a defect. Health = Tempo span status + Prometheus metrics; Loki carries the CONTENT
  once something is known wrong. MCP sidecars deliberately rely on parent-service telemetry.

## TOOL_UPKEEP [sharpen while you cut] [HARD_STOP]
Tools are maintained DURING the work that uses them, never batched into a phase of their own # that is regrinding, after months of dull cuts
PRECONDITION WE DO NOT GET FREE: our tools fail toward SUCCESS -- a harness scoring KILLED on a suite that ran zero
  assertions reports itself as sharp -- so each must REPORT ITS OWN DULLNESS: carry a KNOWN-BAD CONTROL exercised when
  the tool runs, broken one documented way, requiring the matching guard to complain BY NAME (worked example:
  deployment/tests/alerts/selftest.sh). A green run without a control is an opinion.
TRIGGER: you USED a tool -> leave it sharper # not "it broke" -- a tool that has visibly broken was already blunt for every job before it
SHARP ENOUGH, NOT RAZOR: judge a remaining defect by whether it MISLEADS (a reader or agent takes a wrong action) or is
  merely IMPERFECT. Stop at the first # a round trading three cosmetic fixes for one new false claim is a net loss
ANTI: ✗ read a green run as proof # ask what the tool CANNOT see -- verify-citations.py is content-blind, so a
        citation drifted onto a comment reads GREEN
      ✗ judge a citation sweep by its COUNT or its rc # both are proxies that fail toward SUCCESS -- rc 1 is this
        repo's steady state, and a citation going WRONG has been measured to LOWER the cannot-land count. Compare the
        unresolved SET and the LANDING TEXT against a pristine baseline; re-derive, never quote # full rule + evidence:
        .claude/skills/supervisor-mode/LESSONS.md L8
        `mapfile -d '' F < <(git ls-files -z '*.md'); python3 scripts/verify-citations.py --quiet "${F[@]}"`
        # ✗ pipe it through xargs: xargs remaps the child rc to 123, so the rc above is unobservable that way
      ✗ ship a tool whose docstring claims coverage it does not have # the defect, moved into the tool

## INFERENCE [shared GPU/CPU serving — EXTRACTION rules live in SentinelCollector/AGENT_README.md]
TOPOLOGY [what is INSTALLED, never what is permitted -- the engine is an AXIS
  (LlmBenchmark/MEASUREMENT_SPACE.md); run_model.py drives any OpenAI-compatible engine BY DESIGN]:
  GPU: vllm-server (Qwen/Qwen2.5-32B-Instruct-AWQ) -> Sentinel extraction + Reports narrative
  CPU: llama-server(GBNF DSL rollback) | llama-cpu-rag(SecMaster RAG) |
    llama-cpu-embed(bge-m3, shared SecMaster + SentinelCollector)
  ✗ propose ollama # no container remains; its GGUF store is a frozen ro-mounted artifact the
    llama.cpp runners read from -- a deployment fact, and the only one here
✗ CHANGE THE SERVED MODEL, ITS QUANTIZATION, ITS KV DTYPE OR `--max-model-len` AS A DEPLOY # each is a
  SCORED acceptance decision, not config -> SentinelCollector/AGENT_README.md §MODEL_ACCEPTANCE.
  Sites that look like config and are not: `vllm_base_model`, `vllm_image` and `vllm_max_model_len` in
  deployment/ansible/group_vars/all.yml, and the vllm-server `command:` in deployment/artifacts/compose.yaml.j2
GPU_OOM: restart vLLM first # model, quantization and context are then measurable tradeoffs, each with a
  scorecard path -- none is off the table, and none is a free edit (line above)
VLLM_UPGRADE [HARD_STOP before bumping `vllm_image` in deployment/ansible/group_vars/all.yml]:
  ✗ carry `--kv-cache-dtype fp8_e5m2` past 0.19 # 0.28.0 starts fine and serves ONE request, then faults
    under concurrent decode (CUDA illegal memory access) and stays 503 -- and the deploy gate is a /health
    wait plus one SEQUENTIAL 1-token completion; the fault needs concurrency >= 2, so the gate cannot see
    it. `fp8_e4m3` is the one-flag fix at NO quality cost (measured single-axis, null). Isolation table ->
    docs/BACKLOG.md
  ✓ re-score BOTH models on production's CoD path before ANY engine bump # an engine change is a silent
    quality change until scored
TRACK LATEST, ROLL BACK ON FAULT [user direction 2026-09-07. This SUPERSEDES "score before you bump",
  which still made staying the default and is how 0.19.0 became a floor nobody chose. The HARD_STOP above
  is scoped to the FLAG, never to upgrading]:
  THE DEFAULT IS THE LATEST RELEASE. Staying needs a reason; upgrading does not # inverted deliberately
  rollback IS the safety mechanism, and here it is one variable: revert `vllm_image` in
    deployment/ansible/group_vars/all.yml and redeploy # bounded, ~4min, no data at risk. When rollback
    is that cheap, making each upgrade earn its way in is pure loss
  STAYING has a cost that appears on no dashboard: architectures the engine cannot serve (Gemma 4, whose
    infeasibility on 0.19.0 is a MISSING CAPABILITY, not a dependency pin), throughput never claimed, and
    a migration that grows with every version skipped
  ✗ never price an engine bump as a tax charged against the model that needs it # independently worth
    doing, and doing it DECOUPLES the engine decision from the model decision
  ✗ never let the DEPLOYED engine bound the option space # the axis is what is PERMITTED, not INSTALLED
  PRECONDITION WE CURRENTLY FAIL [roll-back-on-fault needs faults to be VISIBLE, and ours are not]:
    the deploy gate is a /health wait plus ONE SEQUENTIAL 1-token completion, so it cannot see the
    fp8_e5m2 concurrency fault above, nor any fault needing concurrency >= 2 -- an upgrade PASSES that
    gate and is still broken. FIX THE GATE (exercise CONCURRENT decode on the real path), never slow
    the upgrades # the gate is the thing that is wrong here, not the cadence
  SCORE AFTER THE BUMP, AS A DETECTOR, NEVER AS A GATE # a crash rolls itself back loudly; a silent
    quality regression does not, and only the harness sees it
  ✓ llama.cpp is already deployed here and is a legitimate GPU arm to SCORE, not only the CPU rollback
    path # user direction 2026-09-07
VLLM_METRICS [for QUERYING; the rule, dashboard and compose files carry their own edit notes]:
  scraped as job="vllm" from vLLM's NATIVE /metrics # NO OTLP metric exporter, only traces (Tempo,
    service vllm-server), so no vllm: series exists under job="otel-collector"
  ✗ vllm:gpu_cache_usage_perc # does not exist on 0.19 and every published vLLM dashboard and example rule
    uses it -- a query or rule against it is silent forever and reads exactly like a healthy engine. The
    real name is vllm:kv_cache_usage_perc; verify any vllm: name against the live endpoint before using it

## SERVICES [monorepo] # the card-audit set, machine-read by .claude/skills/architecture-cards/scripts/enumerate-services.sh
  # one role per line, comma-separated — a pipe-joined line silently drops that line's services from the audit
collectors: FredCollector, AlphaVantageCollector, NasdaqCollector, FinnhubCollector, OfrCollector, SentinelCollector
processing: ThresholdEngine
alerting: AlertService
calendar: CalendarService
metadata: SecMaster
substrate: MacroSubstrate

## SERVICE_ARCHITECTURE [HARD_STOP]
READ {Service}/AGENT_README.md before reasoning about a service's architecture, API, data model or
  resolution flow # the card front-loads negative space (does-NOT / on-miss / invariants /
  DISTINCTIONS / GOTCHAS) that an endpoint catalog cannot convey
every service in SERVICES has one at that exact path; the sole card off the roster is
  gemini-resolver-mcp/AGENT_README.md (a host systemd unit, not a compose service)
✗ guess a service's shape from method names or the endpoint table
✗ "fix" a symptom by violating a card INVARIANT

## DATA_FLOW
Collectors ->gRPC:5001-> ThresholdEngine ->OTLP-> otel-collector -> Prometheus -> Alertmanager -> AlertService -> ntfy|email|autofix
Collectors ->gRPC:5001-> SecMaster RegisterSeries (fire-and-forget)
ThresholdEngine ->gRPC:5001-> SecMaster ResolveBatch | SentinelCollector ->HTTP:8080-> SecMaster /api/resolve-entities
arrows are DATA direction, never call direction # on arrow 1 the collector SERVES the stream and TE is the client
gRPC and HTTP 8080 are container-internal. Of the ROSTER services only sentinel-collector publishes a host
  port (5091, Review UI); MCP sidecars 31xx
