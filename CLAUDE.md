# CLAUDE.md [ATLAS project-level]

## PROJECT_OVERVIEW
name: ATLAS
purpose: financial data collection, processing, alerting, LLM extraction
stack: .NET10/C#14 | TimescaleDB | nerdctl/containerd | OTEL->Loki/Prom/Tempo->Grafana | Serilog | Polly

## EXECUTION_CONTEXT [HARD_STOP]
YOU_ARE_ON: mercury # this machine, the production server
✗ ssh mercury | ansible mercury # you are already here -> extra hop + templating breakage
✓ sudo nerdctl ... | sudo systemctl ... # direct
✓ ansible-playbook ... # deploys TO mercury, runs FROM mercury

## PROJECT_CONVENTIONS
compose.yaml never docker-compose.yml | Containerfile never Dockerfile | devcontainer never local install
rationale: runtime-agnostic (nerdctl|docker|podman) + compose v2 + clean host

## WHERE_WORK_LANDS [canonical routing — every other copy POINTS here, never restates]
you found or learned X -> write it HERE, in the SAME PR as the work:
  defect | measurement debt | deferred work | parked epic -> docs/BACKLOG.md, WITH the measurement that
    makes it re-checkable # an entry with no number is an opinion, and the next agent cannot tell
    whether it is still true. Close an entry in the PR that fixes it; no tombstones.
  engineering | deploy | observability rule                -> this file
  failure mode that RECURRED (twice, not once)             -> .claude/skills/supervisor-mode/LESSONS.md
    # a FIRST occurrence goes to docs/BACKLOG.md, which is what makes the SECOND one recognisable
  service shape | invariant | earned-exception precondition -> <Service>/AGENT_README.md D-entry
  phase | epic outcome                                      -> git tag + docs/RELEASES.md # PHASE_TAGS
  what happened                                             -> git log + the PR body # never a doc
STATE.md [supervisor-owned, repo root]: DISPOSABLE working memory for the epic in flight.
  reset at each epic boundary: `scripts/new-epic.sh <epic-slug>` (`--check` audits any time and writes
    nothing; `--evicted` overrides). It audits BOTH STATE.md and `.claude/skills/supervisor-mode/LESSONS.md`
    and REFUSES on any finding: a lesson whose `GRADUATE_CHECK` passes must be promoted into a rule, skill or
    hook and DELETED, or its refusal recorded. Mechanism: `--help`, the scaffold header, LESSONS.md's own
    header. Guards: scripts/tests/new-epic-selftest.sh
    ✗ a clean audit is NOT a certificate that eviction is done # the fifth ban is judgement, ungreppable
    rationale: a store with an in-flow and no out-flow becomes a diary # measured 2026-09-05 on all three of
      them -- STATE.md, docs/BACKLOG.md and LESSONS.md; all three counts are in docs/BACKLOG.md
  UNSEARCHABLE: untracked + gitignored, and `grep -r` honours .gitignore — a repo-wide search silently
    misses it. Read it by explicit path; never report "not found in the repo" without that check.
  ✗ never commit | push | PR it
  ✗ never `git clean -x` # it deletes ignored files, STATE.md among them, with no recovery
  ✗ never stash | restore | checkout supervisor-owned files (.claude/skills/supervisor-mode/**) to get a
    clean tree # `git checkout -b` and `git pull --ff-only` already preserve dirty tracked files when the
    new ref does not touch them — proceed as-is
  ✗ nothing durable goes in it — if it would outlive this epic, it belongs at a row above

## VERIFY [before_commit]
IF code change THEN verify it compiles BEFORE commit # "straightforward -> commit anyway" is how bugs ship
  dotnet: {Project}/.devcontainer/compile.sh [--no-test]
  container: {Project}/.devcontainer/build.sh [--no-cache]
  can't verify -> ASK_USER
OWNED: each compile.sh (+ sentinel-edge typecheck.sh/dev.sh) owns a compose project keyed to its worktree
  (scripts/devcontainer-owner.sh): atlas-<sha1(worktree)[0:12]>-<slug>, the same key mark-tests-passed.sh uses.
  N agents in N worktrees compile SIMULTANEOUSLY # never sequence them, never wait
  compile.sh proves /workspace is its OWN tree (inode match); mark-tests-passed.sh refuses without that attestation
  cleanup: teardown on EXIT + a reaper on every start removing atlas-* state whose worktree is gone # SIGKILL cannot trap
  ✗ no .devcontainer/compose.yaml publishes a host port # exec-only; sole exception sentinel-edge compose.ports.yaml
  rationale: concurrent runs used to test another worktree's tree and still write a push marker
KNOWN [pre-existing, not concurrency]: two DIFFERENT services in ONE worktree collide on Events/src/*/obj by UID
  root-user devcontainers (CalendarService, FinnhubCollector, NasdaqCollector, Reports, AlphaVantageCollector)
    vs vscode/uid-1000 -> "Access to the path '/workspace/Events/.../obj/<guid>.tmp' is denied"
  recovery: sudo rm -rf <worktree>/Events/src/*/{obj,bin} # ✗ not fixed by serializing: file ownership, not a race

## PHASE_TAGS [at phase / epic completion]
  1. `git tag -a dsl-poc-phase{N}-done <sha> -m "<outcome>"` # tags are named bare; `tag/<name>` is not resolvable
  2. `git push origin dsl-poc-phase{N}-done`
  3. entry in `docs/RELEASES.md` — outcome + tag reference
  4. `git rm` the phase's working/iteration docs; record each retirement in RELEASES.md with its recovery pointer
EXCEPTION [never retire]: a DO-NOT-BUILD spec is permanent and ADR-shaped # it never completes a phase, so step 4
  never fires for it. An implemented plan is survived by its code; a rejected one is survived by nothing, and the
  next person with the same idea cannot learn it was already measured and declined.
rationale: main = current-state docs PLUS live working docs in docs/proposals/ + docs/superpowers/ (docs/README.md
  is the authority). Merging a plan is not approving it — the doc's Status line governs. History: `git show <tag>:<path>`.

## GIT_PUSH [HARD_STOP]
✗ NEVER push without running ALL tests for modified projects: compile.sh (no --no-test), 0 errors AND 0 warnings AND green
PROCESS: git diff --name-only -> compile.sh per project -> fix any failure -> only then push
hook: .claude/hooks/git-push-guard.sh
  marker "v2 tree <hash> <iso8601>", keyed to HEAD^{tree} # committed content only; uncommitted edits do NOT move it
  survives: commit-after-test | cherry-pick | rebase
  does NOT survive an unrelated commit: the root tree covers EVERY tracked path, so a docs-only commit remaps
    HEAD^{tree} too. Docs pushes get through on the docs/config exemption, never on tree-hash survival.
  scope: write=per-worktree (suffix sha1(toplevel)); read=global scan by tree-hash # any worktree's marker satisfies it
rationale: broken tests = broken code = broken trust

## DEPLOYMENT [HARD_STOP]
✗ NEVER edit /opt/ai-inference/compose.yaml directly # ansible-managed; direct edit = config drift
compose-service tag [SCOPED — the default]:
  ansible-playbook playbooks/deploy.yml --tags {service} --skip-tags build -e "scoped_restart=true scoped_services={service}"
  scoped_services must name a COMPOSE service, not merely an ansible tag # deploy.yml filters on
    `label=com.docker.compose.service=${svc}` and then asserts the container is running, so a
    tag-only name matches nothing and FAILS. Grep those two strings, never a line number.
    No compose service exists for dashboards | patterns | monitoring | otel | alerting | instruments | models |
    nasdaq-collector; macro-substrate's service is migrate-macro-substrate.
  --skip-tags build deploys the CURRENT :latest, it does NOT build # for 21 of 26 service tags the only task
    carrying the tag IS the build task -> this form runs zero service tasks. Build first (CONTAINER_BUILD).
    Only alert-service | secmaster | sentinel-collector | threshold-engine | vllm-server carry config/sidecar tasks too.
non-service tag [dashboards | patterns | alerting | monitoring]: --tags {tag} --skip-tags always # only that tag's tasks
✗ bare `--tags {anything}` # UNCONDITIONAL full-stack restart, not a conditional one: the task
  "Remove existing compose.yaml to force regeneration" runs state:absent under tags:[always] with no `when:`,
  the next task re-templates it, so compose_file.changed is ALWAYS true and the restart state resolves to
  'restarted'. = compose down/up of EVERY service incl a ~4min vLLM GPU reload, and it
  RESURRECTS a deliberately-stopped alert-service. The two escapes are `--skip-tags always` and `-e scoped_restart=true`.
grafana alerting: `--tags alerting --skip-tags always` THEN `sudo nerdctl restart grafana` # provisioning is
  startup-loaded AND grafana lives in the separate OTEL stack, so no ansible form reloads it.
  Dashboards differ — they auto-reload (updateIntervalSeconds: 30), no restart.
inventory: deployment/ansible/inventory/hosts.yml # ansible.cfg default; run from deployment/ansible/
filter test: nerdctl compose exec -T {svc}-dev dotnet test --filter 'DisplayName~{Test}' # xUnit exposes
  DisplayName | FullyQualifiedName, NOT Name. `Name~` matches ZERO and STILL EXITS 0 — bare, no pipe needed —
  so a filtered run that tested nothing reads as a pass. Measured: Name~should_return 0 tests, DisplayName~ 304.
VERIFY_TRAP: `nerdctl inspect <svc>` RETURNS THE IMAGE, NOT THE CONTAINER # every service shares a name between the
  two and bare inspect resolves the image first, yielding a plausible .Created that is the BUILD time -> a deploy
  "verified" that way compared the fresh image to itself. Use `nerdctl container inspect`.
AUTOFIX [dormant but armed]: autofix-runner NEVER runs ansible (alert -> queue -> Claude opens a PR and STOPS).
  Only autofix-watcher deploys, and ONLY when a HUMAN merges an autofix PR: `git checkout main; git pull` in the
  SHARED working tree (no dirty-tree check) then `deploy.yml --tags "$services"` with NO --skip-tags and no
  scoped_restart = FULL stack + ~4min vLLM reload, retrying every 5min UNBOUNDED on failure.
  Dormant since 2026-06-18, but alert-service is Up with AutoFix enabled, so one critical alert re-arms it.

## CONTAINER_BUILD
IMAGE: {service-name}:latest # fred-collector ✓ fredcollector ✗ — verify against /opt/ai-inference/compose.yaml
BUILD: {Project}/.devcontainer/build.sh [--no-cache] from /home/james/ATLAS # monorepo context required
DEPLOY: the scoped form above # never manual nerdctl, never bare --tags

## DATABASE [ef_core]
SCHEMA: EF migrations only # single source of truth + versioned + testable
SEED: EF HasData() | app-level seeding on startup; the app runs its own migrations
PSQL [DEBUG_ONLY]: sudo nerdctl exec timescaledb psql -U ai_inference -d atlas_data
  ✓ SELECT to verify state
  ✗ INSERT/UPDATE/DELETE/ALTER to fix state # fix the root cause in the app
  restate "psql is SELECT-only" VERBATIM in every dispatch brief # this HARD_STOP alone has proven necessary
    but NOT sufficient: an agent that HAD the rule still ALTERed prod
TABLES: news obs sentinel.extracted_observations | raw_content · matrix feed public.macro_observations ·
  cells public.matrix_cells · regime public.sector_regimes · SecMaster DB atlas_secmaster (source_mappings, instruments)
MIGRATIONS [HARD_STOP]:
  ✗ NEVER hand-write a migration .cs # missing Designer.cs -> EF records it in __EFMigrationsHistory, schema unchanged
  ✓ nerdctl compose exec -T {svc}-dev sh -c "cd /workspace/{Svc}/src && dotnet ef migrations add {Name} --output-dir Data/Migrations"
    `--context {Svc}DbContext` is REQUIRED where the project reference graph pulls in a SECOND DbContext
      -- measured 2026-08-27: SentinelCollector references MacroSubstrate, so MacroSubstrateDbContext is in
      scope and the bare form above fails as ambiguous. The service having one DbContext of its OWN is not
      the test; what `dotnet ef` sees across every referenced project is.
    ✗ `--project src/Data` # no service has such a PROJECT (Data/ is a FOLDER under {Svc}/src/; `git ls-files | grep -E '/src/Data/.*\.csproj$'` -> 0):
      it resolves to {Svc}/src/src/Data and dies MSB1009, leaving a stray src/src/obj
    --output-dir is relative to the project: Data/Migrations for all but CalendarService (Migrations) and
      MacroSubstrate (its project is src/MacroSubstrate/, so cd there and use Data/Migrations).
    `dotnet tool restore` FIRST if dotnet-ef is missing # it is a local tool manifest, not a global install
  required: {Migration}.cs + {Migration}.Designer.cs + ModelSnapshot.cs
  PARTIAL INDEX: EF expresses it natively via .HasFilter("\"col\" = TRUE") — no raw-SQL escape needed
    ✗ a partial unique index does NOT back a bare `ON CONFLICT (col)` # arbiter inference needs a predicate implying
      the index's; bare raises 42P10 "no unique or exclusion constraint matching". Write `ON CONFLICT (col) WHERE <pred>`
ANTI: ✗ raw SQL during deployment ✗ bypassing EF to seed/migrate ✗ manual DB fixes

## DATA_ML_CONTEXT
VLLM_STRUCTURED: response_format (openai standard), never guided_json # guided_json broken in vLLM 0.19
PROMPTS: edit the REPO upstream, never the host mount and never the container
  ✓ SentinelCollector/src/prompts/     -> /opt/ai-inference/prompts/sentinel -> container /prompts
  ✓ SentinelCollector/src/cod-prompts/ -> /opt/ai-inference/prompts/cod      -> container /prompts/cod
  ✗ /opt/ai-inference/prompts/** # the deploy.yml tasks named "... prompts from repo (overwrites host edits)" sync
    force:true -> host edits are CLOBBERED next deploy; ansible-gate-guard denies the write
  ✗ inside the container # lost on restart; the host mount is what the container reads
  hot-tune on the host to iterate, but tuning worth keeping must land in the repo path
ESTIMATE_GATE [data | vram | model tradeoff]: enumerate repo + filesystem FIRST, then estimate; check THIS
  project's prior results before claiming a tradeoff # generic defaults ("30-50 docs", "LoRA hurts quality") are
  not our reality, and high-yield sources here have been abundant every time anyone counted -- so COUNT, and
  never substitute one unmeasured number for another

## GIGO [garbage_in_garbage_out] [HARD_STOP]
BOUNDARY_HANDLING in the OTHER direction: we validate what a function RECEIVES and drop that rigor for what a call
SENDS. A call (fn | service | model | paid API) is a boundary — validate what you hand it or garbage propagates.
Junk admitted at ANY boundary costs $, GPU/CPU, log signal, stored-data integrity, and OUTPUT CORRECTNESS (a junk
"entity" resolving to the WRONG instrument corrupts the matrix). $-cost is ONE symptom, not the frame.
  ✓ clean at the SOURCE where garbage is BORN — one fix covers every consumer
  ✗ gate each destination — every new consumer re-learns the lesson # done twice (#818 FRED, #823 paid resolver),
    and the free wrong-ticker resolutions were worse than the bill
  ROOT: reject non-entity surfaces at extraction INGRESS (SentinelCollector CandidateSurfaceFilter);
    destination gates stay as defense-in-depth

## INTENT_FIDELITY [code_embodies_the_spec's_why] [HARD_STOP]
PRINCIPLE: every line traces to a design decision WITH its justification, and the justification lives in the
card/comment NEXT TO the code, not only in a plan. Code that inherits the WHAT without the WHY drifts into violating
the design's ethic.
  a privileged/expensive/EXCEPTION path (frontier last-resort, raw-DB write, host restart, --user flag) exists for a
  SPECIFIC EARNED case -> GUARD it so it cannot silently become a primary path, and WRITE the precondition at the code.
  worked example: gemini-resolver kept the mechanism (call-on-miss) and lost the precondition (earned only when all-cheap-failed) -> frontier firehose, invisible until the bill
ENFORCE at a scarce-resource boundary ($/GPU/quota; as warranted, not dogmatic): gate(eligible-only) +
  fail-closed-cap(refuse past budget, never silent-pass) + burn-alert BEFORE depletion (never ship "calls>0 AND
  cost=$0" — that is a corpse-detector, it fires after the money is gone) + honest-health(exercise the real work path,
  not reachability) + business-test(RED-on-unfixed [[feedback_tests_validate_business_outcomes]]).
MECHANICS [format spec: .claude/skills/architecture-cards/CARD_TEMPLATE.md §DECISIONS BLOCK]:
  D_ENTRY: `D-n <slug>: INTENT <why> / PRECOND <condition> / GUARD <class.method> @ file:line / TEST <Class.Test>`
    in <Service>/AGENT_README.md # scoped to exception paths | scarce-resource boundaries | non-obvious preconditions;
    `DECISIONS: none` is legal; >~6 = smell
  ATOMIC_SET [all-or-none]: D-entry + `// INTENT(D-n):` at the guard site + guard code + guard test
  SUPERSESSION: rewrite the entry in the SAME PR as the code change; briefs name "supersedes D-n"; no tombstones
  CONFLICT [HARD_STOP]: brief contradicts a D-entry without named supersession -> STOP + report # never route around,
    never obey the stale entry; a human arbitrates, not the implementing agent
  GUARD_TEST: construct the violation, assert refusal AT the boundary through the real flow, RED if the guard is
    deleted # contract: .claude/skills/intent-review/SKILL.md §GUARD_TEST_CONTRACT

## OBSERVABILITY [user scar tissue: "too many services non-functional due to lack of observability"]
✗ never demote a visible signal to Info+metric without a WIRED alert
✓ keep a VISIBLE Warning on persistent dependency-unavailability; startup banners STAY Warning # boot-loop visibility
A SIGNAL CAN BE DEMOTED WITH NOBODY DECIDING TO DEMOTE IT: before removing or changing a mechanism, enumerate
  what was OBSERVING it and pin each with a test that fires on the REAL path # a signal riding on a bug dies with
  the fix, and the fix looks correct -- missed TWICE on GeminiResolverNotResolving, the second time by the round that fixed the first
HEALTH IS TEMPO, NOT LOKI: prod log level defaults to Warning, so a HEALTHY container emits NOTHING — silence is the
  designed steady state, never a defect. Health = Tempo span status + Prometheus metrics; Loki carries the CONTENT
  once something is known wrong. MCP sidecars deliberately rely on parent-service telemetry.

## TOOL_UPKEEP [sharpen while you cut] [HARD_STOP]
Tools are maintained DURING the work that uses them, never in a separate phase — a dull tool makes worse work every cut.
PRECONDITION WE DO NOT GET FREE: at the bench dullness is FELT; our tools fail toward SUCCESS instead (a harness scoring
  KILLED on a suite that ran zero assertions reports itself as sharp). So each tool must REPORT ITS OWN DULLNESS:
  carry a KNOWN-BAD CONTROL exercised when the tool runs — break it one documented way, require the matching guard to
  complain BY NAME (worked example: deployment/tests/alerts/selftest.sh). A green run without a control is an opinion.
TRIGGER: you USED a tool -> leave it sharper. Not "it broke" — a tool that has visibly broken was already blunt for
  every job before it. A defect that RECURS belongs in .claude/skills/supervisor-mode/LESSONS.md at the VERDICT, not
  in a retro # once is an incident, twice is a lesson
SHARP ENOUGH, NOT RAZOR: judge a remaining defect by whether it MISLEADS (a reader or agent takes a wrong action) or is
  merely IMPERFECT. Stop at the first # a round trading three cosmetic fixes for one new false claim is a net loss
ANTI: ✗ batch maintenance into its own phase # that is regrinding, after months of dull cuts
      ✗ read a green run as proof # ask what the tool CANNOT see — verify-citations.py is content-blind, so a
        citation drifted onto a comment reads GREEN
      ✗ check citations with a bare rc after an edit that SHIFTS LINE NUMBERS — diff the LANDING TEXT against
        a pristine baseline, never the counts # they matched EXACTLY across the a8a0ed5d drift that put three
        citations on real-but-wrong lines, and both sides are rc 1, this repo's steady state. Re-derive, never quote:
        `mapfile -d '' F < <(git ls-files -z '*.md'); python3 scripts/verify-citations.py --quiet "${F[@]}"`
        # 195 files/496 cites/28 cannot-land @ a8a0ed5d. Piping through xargs instead reports the SAME
        # numbers at rc 123, never 1 — xargs remaps a child's rc, so the rc above is unobservable that way
      ✗ ship a tool whose docstring claims coverage it does not have # the defect, moved into the tool

## SENTINEL [llm_extraction] [arxiv:2512.24601]
MODEL_ACCEPTANCE [SCORED ON PERFORMANCE — the a priori limits were dropped 2026-09-06]:
  THE BAR: a candidate ships on a SCORECARD that BEATS the incumbent's at an ADMISSIBLE coordinate.
    Admissibility is the whole rule and it lives in LlmBenchmark/MEASUREMENT_SPACE.md: arms differ in
    exactly ONE axis and the delta is attributed to it, OR the comparison declares itself a
    CONFIGURATION result and attributes the delta to nothing.
  ✓ evidence = scripts/run_model.py --task cod + eval_harness.py --task cod --cod-gold, LOCAL engine,
    `production_prompt_path: true`, n>=3 an arm, both arms served alike
  ✗ never swap on a publisher's claim, a benchmark from elsewhere, or any number whose coordinate is unstated
  NO A PRIORI LIMITS. Size, engine, quantization, KV dtype, context and topology are AXES TO BE
    MEASURED, never floors to be asserted. A limit may be ADDED only with the measurement that
    establishes it, and it is scoped to what was measured -- a quantization result on one model
    says nothing about another. Which axes have been measured, and to what: LlmBenchmark/BENCHMARKS.md.
  VALIDITY CONDITIONS [not limits — these bound what a scorecard MEANS]:
    - criteria are PROVISIONAL (`ratified_by: null`) -> a `pass: true` is not a ratified pass
    - the gold's arrays do NOT weigh equally: NUMBERS and ENTITIES carry it, events are usable on
      `subject` only, CLAIMS measure noise # two careful human labellers score 0.302 and 0.138
    - `source_entity` for macro series: the macro SERIES owns the number, not the country, not blank
      (DECIDED 2026-09-05 by the user; 490 of 518 conform, 6 knowingly non-conformant, 22 open)
    - ✗ NEVER select a model on entity_ticker_accuracy # most of its gold tickers appear nowhere in
      their article, so it largely scores PRETRAINING RECALL of ticker symbols. Entity resolution is
      SecMaster's job and resolves from the NAME. -> docs/BACKLOG.md KNOWN DEFECTS
  measured results -> docs/BACKLOG.md. ✗ NEVER quote a model figure from THIS file # a figure without
    its coordinate is not a run, and this file cannot carry a coordinate
CONTEXT [MEASURED, not asserted]: serve more than the longest real document, with headroom.
  MEASURED: the 40-article gold's worst case is 7,617 tokens, and truncation is 0 in every arm run
    at 8,192 per slot or above.
  UNMEASURED, and what must set any floor: production's REAL article length distribution. Set the
    floor from that, never from a round number.
PROMPT_HYGIENE [HARD_STOP]:
  ✗ never version a prompt in its FILENAME # no cod_json_final_final_v3.txt. git is the history
  ✓ edit the prompt in place; the diff, the commit message and the scorecard carry the story
  a prompt that is no longer referenced by code or compose is DELETED, not renamed or parked
INFERENCE_TOPOLOGY [what RUNS today — a fact about what is installed, NOT a fence]:
  GPU: vllm-server (Qwen2.5-32B-AWQ) | CPU: llama-server(GBNF) | llama-cpu-rag | llama-cpu-embed(bge-m3)
  THE ENGINE IS AN AXIS, and this project has run three: llama.cpp, then ollama, now vLLM. None of the
    switches was re-measured. run_model.py drives any OpenAI-compatible engine BY DESIGN.
  ollama is RETIRED (no container remains, GGUF blobs ro-mounted) # a deployment fact, not a ban
  PRICED 2026-09-06: llama.cpp CUDA is 2.5x slower than vLLM here (wall 260.9s vs 102.7s, same 40
    articles at concurrency 6) -- a MEASUREMENT path, not a deployment one. It is also the only engine
    with rungs above 4-bit at 27B, which is why the precision question had to be asked there.
  SCREENED, and the screen is cheap: an engine must take `seed` AND do REAL constrained decoding
    (a grammar that MASKS sampling, not one that only drafts speculatively) or no admissible scorecard
    can exist on it. colibri fails both -> docs/BACKLOG.md. NEXT: SGLang (takes both; same-GPU
    single-axis vs vLLM), then ktransformers for the >32B RAM-resident region.
GPU_OOM: restart vLLM first. Then treat model, quantization and context as measurable tradeoffs --
  none of the three is off the table, and each has a scorecard path.
VLLM_UPGRADE [a STABILITY finding, not a performance limit — the crash is real and the fix is FREE]:
  QUALITY COST: NONE. `fp8_e5m2` vs `fp8_e4m3` measured single-axis and null -> docs/BACKLOG.md.
  ✗ carry `--kv-cache-dtype fp8_e5m2` past 0.19 # measured 2026-09-04: 0.28.0 starts fine, serves ONE request,
    then faults under concurrent decode (CUDA illegal memory access) and stays 503. Isolated to the KV DTYPE at
    matched context and concurrency -- not sm_120, not structured output, not CUDA graphs (--enforce-eager still
    crashes), not a dependency (our flashinfer is NEWER than the one in the sibling bug vllm#41651).
    `--kv-cache-dtype fp8_e4m3` is the one-flag fix: 0 errors at concurrency 6, output matched known-good at n=18.
  rationale: vLLM benchmarks FP8 KV as FA3-on-Hopper and FlashInfer-on-B200(SM100), documents e4m3 only, and
    never mentions SM120 or e5m2 -- our config is outside the tested matrix on all three axes. It works on
    0.19.0; nobody upstream validates it.
  ✓ before ANY engine bump: run the levers in docs/BACKLOG.md, then re-measure BOTH models on the substrate
    (LlmBenchmark/scripts/run_model.py) -- an engine change is a silent quality change until scored
VLLM_OBSERVABILITY: metrics = DIRECT prometheus scrape job="vllm" (vllm-server:8000) | traces = OTLP -> otel-collector:4317 -> Tempo
  ✗ look for a vllm series under job="otel-collector" # vLLM has NO OTLP METRIC exporter, only traces; the
    metric side is its own native /metrics endpoint and reaches Prometheus by scrape, not by push
  ✗ vllm:gpu_cache_usage_perc # does NOT exist on 0.19 — every published vLLM dashboard and example rule uses
    it, and a rule written against it is silent forever and reads exactly like a healthy engine.
    The real name is vllm:kv_cache_usage_perc. Verify any vllm: name against the live endpoint before using it.
  ✗ ship the traces flag without OTEL_SERVICE_NAME # vLLM never sets service.name, so spans land in Tempo
    as the literal `unknown_service` — the mechanism works and the spans are unfindable
  rules deployment/artifacts/monitoring/alerts/vllm.yml (tests deployment/tests/alerts/vllm_test.yml) |
    dashboard uid vllm-inference | VllmServerDown dwell 10m is HEADROOM over a cold model load, not a
    derived number (78s measured warm 2026-09-01; ~170s budgeted in deploy.yml) — a dwell under the real
    load time pages on every scoped deploy and on the GPU_OOM remedy above
  DEPLOY TAKES TWO RUNS, and the second one costs a GPU reload:
    metrics+rules+dashboard -> `--tags monitoring --skip-tags always` # no restart of anything
    traces flag             -> `--tags vllm-server --skip-tags build -e "scoped_restart=true scoped_services=vllm-server"`
                               # recreates vllm-server = ~4min GPU reload; needs a deliberate window

## SERVICES [monorepo]
collectors: FredCollector, AlphaVantageCollector, NasdaqCollector, FinnhubCollector, OfrCollector, SentinelCollector
processing: ThresholdEngine | alerting: AlertService | calendar: CalendarService | metadata: SecMaster
substrate: MacroSubstrate | shared: Events/, deployment/, docs/
mcp: FredCollector/mcp, ThresholdEngine/mcp, FinnhubCollector/mcp, OfrCollector/mcp, SecMaster/mcp, WhisperService/mcp

## SERVICE_ARCHITECTURE [read-first] [HARD_STOP]
BEFORE reasoning about a service's architecture / API / data-model / resolution flow: READ <Service>/AGENT_README.md.
  rationale: the card front-loads negative space (does-NOT / on-miss / invariants / DISTINCTIONS / GOTCHAS) that an
  endpoint catalog cannot convey.
✗ guess a service's shape from method names or the endpoint table # read the card
✗ "fix" a symptom by violating a card INVARIANT (bulk-preload, backfill-to-green, raw DB fix)
cards carry numbered D-entries: touching a guard or contradicting one without a named "supersedes D-n" -> STOP
cards:
  SecMaster:         SecMaster/AGENT_README.md         # resolve-entities != ResolveBatch; identity indep of collection; fuzzy-proposes/authoritative-confirms; ✗NotFound="not-in-table"; ✗bulk-preload; ✗gate-non-Equity-sector
  ThresholdEngine:   ThresholdEngine/AGENT_README.md   # WS3-projector=ONLY wired matrix_cells writer; ObservationEventSubscriber=UNWIRED/dead; Confidence XML-doc"informational only"=FALSE; ✗live-FRED-gRPC-writes-matrix_cells; ✗ascending-projector-read
  SentinelCollector: SentinelCollector/AGENT_README.md # news->matrix pipeline spans MacroSubstrate; `:sig:` infix=string contract change-all-or-none; signal-dim gates projection sector-dim does NOT; Shadow != Off (same cells written); ✗gate-entry-on-sector; ✗check-Mode-before-concluding-broken
  FredCollector:     FredCollector/AGENT_README.md     # catalog indep of instrument (SeriesId=FRED mnemonic != instr-id); AlfredBackfillService deliberately does NOT touch LastCollectedAt; ObservationChannel no reader=memory-growth; ✗expect-WARN-GRPC-unset; ✗ALFRED-backfill=advances-LastCollectedAt
  FinnhubCollector:  FinnhubCollector/AGENT_README.md  # candle/social/insider/calendars=dead-schema (tables empty); quotes reach TE via the TABLE not a push (upsert -> EventRepository -> gRPC), so a "publish" path is NOT the data flow; deadman must watch COLLECTION work, never finnhub_api_requests_total (other services hold it >0 — that is why a 16-day stall was invisible); GetLatestEventTime=UtcNow-on-empty (not a new-data signal); ✗assume-non-Quote-data-flows
  AlphaVantageCollector: AlphaVantageCollector/AGENT_README.md # stream=scalar-only(Commodity/Economic); OHLCV never emits events; quota=in-mem(restart-wipes); TechnicalIndicator=scaffold; ✗assume-OHLCV-events ✗expect-values-in-Event ✗quota-survives-restart
  NasdaqCollector:   NasdaqCollector/AGENT_README.md   # DISABLED prod(NDL WAF); EventId=Ulid-per-read (not a stable dedup key); EventTypes filter=dead param; SecMaster Economic GUARD may silently reject; ✗treat-EventId-stable ✗assume-prod-running
  OfrCollector:      OfrCollector/AGENT_README.md      # FSI indep of gRPC-register; FSI composite+4 patterned subindices->macro_observations (_EQUITY/_SAFE_ASSETS/_US/_AE excluded — no pattern); circuit-breaker=5 CONSECUTIVE fails->60s (not sliding-window); dual-write non-fatal; gRPC GetLatestEventTime=now() placeholder; ✗conflate-gRPC-register-with-REST-tag ✗assume-ALL-FSI-cols->matrix
  AlertService:      AlertService/AGENT_README.md      # UP by design (Alertmanager->alert-service->ntfy `atlas-alert`); appsettings routing != RoutingOptions class default; dedup=fingerprint-only; autofix rate-limit=static process-wide; ✗assume-202-means-sent
  CalendarService:   CalendarService/AGENT_README.md   # HTTP-only (no gRPC :5001); FRED allow-list ~18 releases (not all); event_time=synthetic DST-unaware; market endpoints bypass DB; Finnhub worker disabled; ✗assume-gRPC ✗trust-FRED-event_time-real
  MacroSubstrate:    MacroSubstrate/AGENT_README.md    # write=DO UPDATE heal-on-rewrite (not DO NOTHING); QueryAsync(AsOfDate+MappingVersionLabel simultaneously)->ArgumentException; not a running service (library+migrator only); ✗trust-README-DO-NOTHING ✗set-both-version-axes
  gemini-resolver:   gemini-resolver-mcp/AGENT_README.md # HOST systemd unit :9300 (not compose, speaks HTTP not MCP) — the CODE is not ansible-deployed (a git pull reaches prod on next restart) but its ALERT RULES are (`--tags monitoring --skip-tags always`), so a code-only pull ships none of them; 3 outcomes not 2 (rejected/accepted/neutral); a Google 429 reaches the caller as 200+symbol=null; burn gauges FALL during a credit outage; ✗read-quiet-burn-as-not-spending ✗restart-to-clear-fail-closed ✗treat-symbol-null-as-failure

## DATA_FLOW
Collectors ->gRPC:5001-> ThresholdEngine ->metrics-> Prometheus -> Alertmanager -> AlertService -> ntfy|email
Collectors ->gRPC:5001-> SecMaster (registration, fire-and-forget)
ThresholdEngine ->gRPC:5001-> SecMaster (resolution, context-based routing)
gRPC internal only (container-to-container) | HTTP 8080 internal, 50xx host
