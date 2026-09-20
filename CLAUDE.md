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
  ✗ nothing durable goes in it — if it would outlive this epic, it belongs at a row above # a store with
    an in-flow and no out-flow becomes a diary
  ✗ never commit | push | PR | `git clean -x` | stash | restore | checkout it, or any supervisor-owned file
    (.claude/skills/supervisor-mode/**), to get a clean tree # untracked + gitignored: NOTHING in git restores
    a lost edit. `git checkout -b` and `git pull --ff-only` already preserve dirty tracked files when the new
    ref does not touch them -- proceed as-is
  UNSEARCHABLE to `grep -r` and the Grep tool, which honour .gitignore # read it by explicit path before
    reporting "not found in the repo". What else destroys it, and what does NOT:
    .claude/skills/supervisor-mode/references/state-file.md §UNTRACKED/DESTROYABLE
  epic-boundary reset, and the audit that gates it: `scripts/new-epic.sh --help`

## VERIFY [before_commit]
IF code change THEN verify it compiles BEFORE commit # "straightforward -> commit anyway" is how bugs ship
  dotnet: {Project}/.devcontainer/compile.sh [--no-test]
  container: {Project}/.devcontainer/build.sh [--no-cache]
  can't verify -> ASK_USER
FILTERED RUN [in the devcontainer -- dotnet exists on the HOST too and a bare `dotnet test` becomes a host run]:
  nerdctl compose exec -T {svc}-dev dotnet test --filter 'FullyQualifiedName~{Class}'
  ✗ `Name~` anything, and ✗ `DisplayName~<ClassName>` in the 4 projects setting xunit methodDisplay=method
    # both match ZERO tests and STILL EXIT 0, so a run that tested nothing reads as a pass
    which 4, and how to enumerate them rather than recall them: scripts/README.md §TEST_FILTERS
OWNED: each CONTAINER-STARTING compile.sh (+ sentinel-edge typecheck.sh/dev.sh) owns a per-worktree compose
  project (scripts/devcontainer-owner.sh), the same key mark-tests-passed.sh uses.
  N agents in N worktrees compile SIMULTANEOUSLY # never sequence them, never wait
  the key, the inode attestation gating the marker, the reaper, the no-host-port rule with its two DECLARED
    exceptions, and the Events/src/*/obj UID collision with its recovery: scripts/README.md §DEVCONTAINER_OWNERSHIP
    # that collision is file ownership, not a race -- serializing does not fix it

## PHASE_TAGS [at phase / epic completion]
  1. `git tag -a <epic-slug>-done <sha> -m "<outcome>"` # insert `-phase<N>` only for a phased epic
  2. `git push origin <epic-slug>-done` # bare tag name. Tag, then push, as two SEPARATE bash calls --
     chaining them denies the push AND never creates the tag. The three spellings that FAIL and why each
     one does: .claude/hooks/README.md §TAG_PUSH_SPELLINGS
  3. entry in `docs/RELEASES.md` — outcome + tag reference
  4. `git rm` the phase's working/iteration docs; record each retirement in RELEASES.md with its
     recovery pointer `git show <tag>:<path>`
EXCEPTION [never retire]: a DO-NOT-BUILD spec is permanent and ADR-shaped -> step 4 never fires for it
  # that exception, curation, and that merging a plan is NOT approving it (its Status line governs) are
  the AUTHORITY's, not this file's: docs/README.md §Curation policy -- keep the two consistent

## GIT_PUSH [HARD_STOP]
✗ NEVER push without running ALL tests for modified projects: compile.sh (no --no-test), 0 errors AND 0 warnings AND green
PROCESS: git diff --name-only -> compile.sh per project -> fix any failure -> only then push
hook: .claude/hooks/git-push-guard.sh -- the marker is a git TREE, COMMITTED content only, matched against
  the PUSHED BRANCH's tree # compile.sh on a dirty tree attests HEAD, NOT what it ran
  survives commit-after-test | cherry-pick | rebase; does NOT survive an unrelated commit, because the root
    tree covers EVERY tracked path -- so a docs push gets through on the docs/config exemption, never on survival
  marker format, the two different refs the write and read sides resolve, the exemption's exact extension
    list, and the per-worktree-write / global-read scan: .claude/hooks/README.md §PUSH_MARKER
rationale: broken tests = broken code = broken trust

## DEPLOYMENT [HARD_STOP]
✗ NEVER edit /opt/ai-inference/compose.yaml directly # ansible-managed; direct edit = config drift
compose-service tag [SCOPED — the default]:
  ansible-playbook playbooks/deploy.yml --tags {service} --skip-tags build -e "scoped_restart=true scoped_services={service}"
  scoped_services must name a COMPOSE service, not merely an ansible tag # a tag-only name matches nothing
    and FAILS "SCOPED-RESTART FAILED". Most ansible tags name no compose service. Derive the set, never recall it:
      sudo nerdctl compose -f /opt/ai-inference/compose.yaml config --services
    It will not warn you that nasdaq-collector has none, or that macro-substrate's is migrate-macro-substrate.
  --skip-tags build deploys the CURRENT :latest, it does NOT build # 18 of the 27 DEPLOYABLE service tags
    carry ONLY the build task -> this form runs zero tag-scoped tasks. Build first (CONTAINER_BUILD).
non-service tag [dashboards | patterns | alerting | monitoring | sentinel-prompts | ...]: --tags {tag}
  --skip-tags always # only that tag's own tasks
✗ bare `--tags {anything}` # UNCONDITIONAL full-stack restart, not a conditional one: compose down/up of
  EVERY service incl a ~4min vLLM GPU reload, and it RESURRECTS a deliberately-stopped alert-service.
  The two escapes are `--skip-tags always` and `-e scoped_restart=true`.
the tag catalogue, the task chain that makes a bare --tags unconditional, which tags name no compose service,
  and which 9 carry non-build tasks: deployment/README.md §TAG_MECHANICS
grafana alerting: `--tags alerting --skip-tags always` THEN `sudo nerdctl restart grafana` # provisioning is
  startup-loaded AND grafana lives in the separate OTEL stack, so no ansible task restarts it.
  Dashboards differ — they auto-reload (updateIntervalSeconds: 30), no restart.
inventory: deployment/ansible/inventory/hosts.yml # ansible.cfg default; run from deployment/ansible/
VERIFY_TRAP: `nerdctl inspect <svc>` RETURNS THE IMAGE, NOT THE CONTAINER # every service shares a name between the
  two and bare inspect resolves the image first, yielding a plausible .Created that is the BUILD time -> a deploy
  "verified" that way compared the fresh image to itself. Use `nerdctl container inspect`.
AUTOFIX [runner ARMED, deployer DISARMED — check `systemctl is-enabled`, never assume]: autofix-runner NEVER
  deploys (it opens a PR and STOPS) and it looks idle when it is not; autofix-watcher is the ONLY deploying
  half, its timer is `disabled`, and deploy.yml re-enforces that on every --tags autofix|alert-service run, so
  NO alert can arm it — only a human can # deployment/README.md §AUTOFIX_HALVES

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
    ✗ `--project src/Data` # Data/ is a FOLDER, not a project -- it resolves to {Svc}/src/src/Data, dies
      MSB1009, and leaves a stray src/src/obj
    `dotnet tool restore` FIRST if dotnet-ef is missing # local tool manifest, not a global install
    the three this command does NOT template -- the {dev-svc} slug (undeducible), the cd + --output-dir
      exceptions, and when `--context {Svc}DbContext` is REQUIRED: .claude/hooks/README.md §EF_MIGRATION_TRAPS
      # the guard's own deny message prescribes that same form
  required: {Migration}.cs + {Migration}.Designer.cs + ModelSnapshot.cs
  PARTIAL INDEX: EF expresses it natively -- .HasFilter("\"col\" = TRUE"), no raw-SQL escape needed
    ✗ bare `ON CONFLICT (col)` is NOT backed by a partial unique index # the arbiter needs a predicate implying
      the index's, else 42P10. Write `ON CONFLICT (col) WHERE <pred>` -- worked case SecMaster/AGENT_README.md
ANTI: ✗ raw SQL during deployment ✗ bypassing EF to seed/migrate ✗ manual DB fixes
      ✗ backfill-to-green # writing data to turn a red metric green is fixing the dashboard, not the cause

## DATA_ML_CONTEXT
VLLM_STRUCTURED: response_format (openai standard), never guided_json # guided_json broken in vLLM 0.19
PROMPTS: edit the REPO (SentinelCollector/src/prompts/ and src/cod-prompts/), never /opt/ai-inference/prompts/**
  and never inside the container # deploy.yml syncs with force:true so host edits are CLOBBERED next deploy
  (ansible-gate-guard denies the write), and a container edit is lost on restart
  the two mount chains, hot-tuning, and why a prompt is never versioned in its FILENAME:
    deployment/README.md §PROMPT_SYNC
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
    THE SAME STOP COVERS a rule stated in a skill, a template or this file -- `.claude/skills/**`, their
    `templates/**`, CLAUDE.md: name the rule and the contradiction, never obey a brief over a written rule, and
    never silently obey a written rule you believe is STALE -- say so # `a rule stated in a skill` is the
    greppable phrase every artifact handing an agent this stop carries VERBATIM; the measured case and the one
    narrow site still missing it are in docs/BACKLOG.md
  GUARD_TEST: the contract is at its CANONICAL home and is never restated here --
    .claude/skills/intent-review/SKILL.md §GUARD_TEST_CONTRACT

## OBSERVABILITY [user scar tissue: "too many services non-functional due to lack of observability"]
✗ never demote a visible signal to Info+metric without a WIRED alert
✓ keep a VISIBLE Warning on persistent dependency-unavailability; startup banners STAY Warning # boot-loop visibility
A SIGNAL CAN ALSO BE DEMOTED WITH NOBODY DECIDING TO: before removing or changing a mechanism, enumerate what was
  OBSERVING it and pin each with a test that fires on the REAL path # a signal riding on a bug dies with the fix, and
  the fix looks correct -- missed TWICE on GeminiResolverNotResolving, the second time by the round that fixed the first
HEALTH IS TEMPO, NOT LOKI: prod log level defaults to Warning, so a HEALTHY container emits NOTHING — silence is the
  designed steady state, never a defect. Health = Tempo span status + Prometheus metrics; Loki carries the CONTENT
  once something is known wrong. MCP sidecars deliberately rely on parent-service telemetry.
LOKI service_name HAS NO DERIVABLE PATTERN: enumerate `list_loki_label_values` for `service_name` FIRST, never
  infer it from a container or tag name # a wrong label returns the SAME empty result as a healthy
  Warning-level service. The measured value set: docs/OBSERVABILITY.md §LOKI_SERVICE_NAME

## TOOL_UPKEEP [sharpen while you cut] [HARD_STOP]
Tools are maintained DURING the work that uses them, never batched into a phase of their own # that is regrinding, after months of dull cuts
PRECONDITION WE DO NOT GET FREE: our tools fail toward SUCCESS -- a harness scoring KILLED on a suite that ran zero
  assertions reports itself as sharp -- so each must REPORT ITS OWN DULLNESS: carry a KNOWN-BAD CONTROL exercised when
  the tool runs, broken one documented way, requiring the matching guard to complain BY NAME (worked example:
  deployment/tests/alerts/selftest.sh). A green run without a control is an opinion.
AND A CONTROL MUST BE AIMED AT THE ACT, or it is a green run about a path nobody tested: drive the tool
  END TO END asserting its OUTPUT and its EXIT CODE (never an internal function's return), rest no
  assertion on a SECOND COPY of the rule the shipped code decides, and build the fixture where the two
  candidate rules DISAGREE # contract + five measured cases: `.claude/skills/intent-review/SKILL.md`
  §AIMED AT THE ACT -- it sits here too because a control and a known-bad control fail the same way
ANCHOR POINTERS ARE GATED IN CI, file:line ones are not the same check: `scripts/verify-pointers.py`
  resolves every `<path>.md` §CONSTRUCT pointer and DENIES an ambiguous path. The GATE is
  `scripts/tests/test_verify_pointers.py::test_tracked_corpus_resolves`, swept on any `**/*.md` change, so
  a renamed construct turns CI red without anyone remembering to look. Run it the way CI does, never by
  hand-invoking the tool: `python -m pytest scripts/tests -k tracked_corpus` # pytest is NOT installed on
  this host -- use a venv. What it CANNOT see, and the arbitrary-file-set form: scripts/README.md §POINTER_SWEEP
TRIGGER: you USED a tool -> leave it sharper # not "it broke" -- a tool that has visibly broken was already blunt for every job before it
SHARP ENOUGH, NOT RAZOR: judge a remaining defect by whether it MISLEADS (a reader or agent takes a wrong action) or is
  merely IMPERFECT. Stop at the first # a round trading three cosmetic fixes for one new false claim is a net loss
ANTI: ✗ read a green run as proof # ask what the tool CANNOT see -- verify-citations.py is content-blind, so a
        citation drifted onto a comment reads GREEN
      ✗ judge a citation sweep by its COUNT or its rc # both are proxies that fail toward SUCCESS -- rc 1 is this
        repo's steady state, and a citation going WRONG has been measured to LOWER the cannot-land count. Compare the
        unresolved SET and the LANDING TEXT against a pristine baseline; re-derive, never quote # full rule + evidence:
        .claude/skills/supervisor-mode/LESSONS.md ALREADY_ENCODED, the verify-against-the-THING line (was L8)
        `mapfile -d '' F < <(git ls-files -z '*.md'); python3 scripts/verify-citations.py --quiet "${F[@]}"`
        # ✗ pipe it through xargs: xargs remaps the child rc to 123, so the rc above is unobservable that way
      ✗ ship a tool whose docstring claims coverage it does not have # the defect, moved into the tool

## INFERENCE [shared GPU/CPU serving — EXTRACTION rules live in SentinelCollector/AGENT_README.md]
TOPOLOGY [what is INSTALLED, never what is permitted -- the engine is an AXIS
  (LlmBenchmark/MEASUREMENT_SPACE.md); run_model.py drives any OpenAI-compatible engine BY DESIGN]:
  GPU vllm-server -> Sentinel extraction + Reports narrative | CPU llama-server (GBNF DSL rollback),
    llama-cpu-rag (SecMaster RAG), llama-cpu-embed (bge-m3, shared) # the served models, ports, context
    and roles are docs/ARCHITECTURE.md §INFERENCE_TOPOLOGY -- a model AND its serving flags are ONE
    SCORED coordinate, so read them there and never retype one here (SentinelCollector/AGENT_README.md D-29)
  ✗ propose ollama # no container remains; its GGUF store is a frozen ro-mounted artifact the
    llama.cpp runners read from -- a deployment fact, and the only one here
✗ CHANGE THE SERVED MODEL, ITS QUANTIZATION, ITS KV DTYPE OR `--max-model-len` AS A DEPLOY # each is a
  SCORED acceptance decision, not config -> SentinelCollector/AGENT_README.md §MODEL_ACCEPTANCE.
  The sites that LOOK like config and are NOT -- in group_vars/all.yml, in compose.yaml.j2, and the
  CLIENT-side ExtractionOptions.ChatTemplate -- are enumerated at deployment/README.md §SCORED_NOT_CONFIG.
  Read it BEFORE editing any of those three files
GPU_OOM: restart vLLM first # model, quantization and context are then measurable tradeoffs, each with a
  scorecard path -- none is off the table, and none is a free edit (line above)
VLLM_UPGRADE [HARD_STOP before bumping `vllm_image` in deployment/ansible/group_vars/all.yml]:
  ✗ NEVER REINTRODUCE `--kv-cache-dtype fp8_e5m2` # past 0.19 it starts fine and serves ONE request, then
    faults under concurrent decode (CUDA illegal memory access) and stays 503 -- and the deploy gate is a
    /health wait plus one SEQUENTIAL 1-token completion; the fault needs concurrency >= 2, so the gate
    CANNOT SEE IT. `fp8_e4m3` is the one-flag fix at NO quality cost. Re-check the RUNNING engine, never
    the repo, before claiming anything about it # `sudo nerdctl container inspect vllm-server`
TRACK LATEST, ROLL BACK ON FAULT [user direction 2026-09-07, SUPERSEDING "score before you bump"; the
  e5m2 stop above is scoped to the FLAG, never to upgrading. group_vars/all.yml, compose.yaml.j2, D-29 and
  a unit test all navigate to this file BY THIS LABEL -- never retire it without repointing them]:
  THE DEFAULT IS THE LATEST RELEASE -- STAYING needs a reason, upgrading does not. Rollback IS the safety
  mechanism and is ONE variable (revert `vllm_image`, redeploy, ~4min, no data at risk).
  SCORE AFTER THE BUMP, as a DETECTOR, never as a gate # the cadence argument, the harness that catches
  this fault class PRE-DEPLOY, what neither it nor the deploy gate can see, and the e5m2 decision record:
  LlmBenchmark/MEASUREMENT_SPACE.md §ENGINE_POLICY
VLLM_METRICS: query job="vllm" (NATIVE /metrics), never job="otel-collector" # no OTLP metric exporter
  ✗ vllm:gpu_cache_usage_perc # does not exist on 0.19, and every published vLLM dashboard and rule uses
    it -- silent forever, and reads exactly like a healthy engine. Real name + the verify-it-live rule:
    docs/OBSERVABILITY.md §VLLM_METRICS

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

## DATA_FLOW [diagrams, ports, the CalendarService caveat: docs/ARCHITECTURE.md §DATA_FLOW]
Collectors ->gRPC:5001-> ThresholdEngine ->OTLP-> otel-collector -> Prometheus -> Alertmanager -> AlertService -> ntfy|email|autofix
Collectors ->gRPC:5001-> SecMaster RegisterSeries | TE ->gRPC-> SecMaster ResolveBatch | Sentinel ->HTTP:8080-> SecMaster /api/resolve-entities
arrows are DATA direction, never call direction # on arrow 1 the collector SERVES the stream and TE is the client
gRPC and HTTP 8080 are container-internal. Of the ROSTER services only sentinel-collector publishes a host
  port (5091, Review UI); MCP sidecars 31xx
