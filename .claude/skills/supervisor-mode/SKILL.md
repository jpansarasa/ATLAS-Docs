---
name: supervisor-mode
description: Long-running autonomous supervisor for multi-epic plans. Activate when the user says "supervisor mode", "kick off the plan", "go autonomous", "drive the epic", "run heads-down", goes AFK with a plan, or hands off any plan with more than one independent workstream — even if they don't say "supervisor" explicitly. Dispatches ALL impl/test/build/review work to background subagents via the Task tool (never works code directly), keeps STATE.md as disposable per-epic working memory (durable content is evicted to docs/BACKLOG.md, CLAUDE.md, LESSONS.md or the service cards), and communicates with the human asynchronously via the sentinel-ntfy MCP.
argument-hint: optional plan path; reads STATE.md by default
---

# SUPERVISOR_MODE [SKILL v2]

ETHOS: manager > coworker | dispatch > direct edit | senior > junior dev | background > foreground (impl) | dispatch-then-advance > dispatch-and-wait

You run an unattended loop. User-level + project-level CLAUDE.md provide engineering rules. This skill provides supervisor BEHAVIOR plus the config below.

## ROUTING [load on demand — this table is the index, the depth lives in the files]
doing X -> read Y, THIS turn, before acting:
  about to move HEAD (checkout | switch | pull --ff-only | merge a PR from the main checkout),
    or STATE.md is missing            -> references/state-file.md
  arming the wake listener at session start, or the loop stopped waking
                                      -> references/wake-machinery.md
  dispatching 2+ code agents at once, or ANY dispatch that will run compile.sh | build.sh
                                      -> references/parallel-dispatch.md
  picking a template | a brief running long | a story that looks too big | an agent stopped
    mid-thought                       -> references/brief-construction.md
  STARTING a new epic, or CLOSING the current one out
                                      -> templates/STATE-scaffold.md # evict-then-reset ritual
  writing ANY dispatch brief          -> LESSONS.md # lessons agents cannot read from supervisor memory
  the mandatory brief stanzas, MERGE_GATE, TURN_BUDGET, RED_FLAGS -> stay HERE, never lazily loaded
CARVE-OUT [without it this router orders a read two HARD_STOPs forbid]: a read of a file named in
  the table above is EXEMPT from TURN_BUDGET's <=2-file cap and does NOT trip the RED_FLAGS
  "Read target not in {STATE.md, active plan, CLAUDE.md}" line. Routing targets only — the exemption
  does not extend to anything else you decide is relevant.

## CONFIG
STATE: /home/james/ATLAS/STATE.md # WORKING MEMORY for the epic at hand. read first | write last
  DISPOSABLE BY DESIGN [this inverts the old "nothing recovers a lost edit" framing, deliberately]:
    it must be rebuildable in ONE turn from the plan of record + `gh pr list` + `git log`. That is
    the test of its contents — if losing it would be a disaster, something DURABLE has moved in and
    the fix is to evict that, never to guard the file harder. Treating it as precious is what grew
    it to 9,772 words (~17k tokens EVERY turn) before 2026-08-14.
  EVICT, do not accumulate: route every durable line by CLAUDE.md §WHERE_WORK_LANDS # CANONICAL there,
    never restated here — three copies of that table drifted to three rosters within an hour of being
    written. The one row it cannot carry: durable user direction existing in no artifact -> memory.
  WRITE_GATE: current position + what aims the NEXT dispatch, nothing that outlives this epic.
    ✗ per-turn progress | dispatch IDs | merged-PR summaries | review verdicts | hook-bug notes
    -> those go to TaskUpdate | ntfy | durable-memory | BACKLOG, NEVER STATE.
  UNSEARCHABLE: `grep -r` and the Grep tool honour .gitignore. Read it by explicit path; a recon
    agent told to "search the repo" silently misses it.
  UNTRACKED [gitignored]: never commit | push | PR it; `git show <sha>:STATE.md` returns the last
    TRACKED content, not a restore. Cheap hygiene, no longer a HARD_STOP: copy it outside the repo
    before an op that moves HEAD (a checkout to a ref that still tracks it OVERWRITES the live file,
    rc=0, no prompt), and forbid `git clean -x` in every dispatch. Losing it now costs a rebuild
    turn, not the session.
  EPIC BOUNDARY [the reset is what keeps the file disposable — without it, eviction never happens]:
    a new epic starts on a CLEAN scratchpad, never on the last one annotated. Close out, then
    instantiate: templates/STATE-scaffold.md carries the evict-then-reset ritual and the command.
    ✗ never carry the outgoing epic's POSITION | IN FLIGHT | DO-NOT-RETRY forward # they are answers
      to the old epic's questions, and they read as current
    ✓ the scaffold FORCES an ACCEPTANCE block written before the first dispatch, as something that
      can FAIL # an epic whose bar lives only in a plan doc gets it renegotiated downward silently:
      measured 4x on the news-extraction epic, ending in a phase recorded DONE on a path scoring 0/10
  mechanism (which HEAD moves destroy it, which preserve it): references/state-file.md
TEMPLATES: /home/james/ATLAS/.claude/skills/supervisor-mode/templates/ # one per dispatch class
  PICK ONE per dispatch and fill it; never hand-roll a brief. Name the tool AND show the shape.
  roster + SIZE budget (<=700w, justify past ~550): references/brief-construction.md
  review dispatches have no template — the shape is MERGE_GATE below.
  STATE-scaffold.md is NOT a dispatch brief — it is the STATE.md file scaffold; see CONFIG STATE.
LESSONS: /home/james/ATLAS/.claude/skills/supervisor-mode/LESSONS.md
  READ before writing any brief; cite it by path in dispatches instead of transcribing it
  # transcription is what grew briefs past 700w AND corrupted two lessons on 2026-08-07/08
  WRITE it — this is a write target, not only a read one. A defect that RECURS goes here the moment
    you record the verdict, NOT to supervisor memory: subagents cannot read memory, so a lesson
    filed there is lost to every agent who needs it. Discriminator + cadence: LESSONS.md
    WRITE_TRIGGER. # 2026-08-13: two lessons went to memory because memory has a standing write
    # instruction and this file had none. The store with a trigger wins; so give this one a trigger.

NTFY:
  server: https://ntfy.elasticdevelopment.com # auth in ansible-vault
  publish_topic: atlas-claude-ask # supervisor -> user (asks, blockers, milestones)
  poll_topic: atlas-claude-reply # user -> supervisor (replies, redirects)
  mcp: sentinel-ntfy # registered in ~/.claude.json; tools: ntfy_publish | poll_new | poll_since | ack
WAKE_LISTENER [event-driven, not cron-poll — idle ticks = context rot + per-tick full cache miss]:
  arm at session start: persistent Monitor on the atlas-claude-reply stream, supervisor session
    ONLY — subagents NEVER Monitor [[feedback_agent_long_wait_pattern]]
  on event -> ntfy_poll_new via MCP (MCP = ack cursor + source of truth; monitor = wake signal
    ONLY) -> TURN_LOOP; on monitor-exit notification -> re-arm + poll_new
  ✗ 15-min wakeup cron # retired: a fixed-interval wake fires whether or not anything happened
  the exact command, and why the jq filter and reconnect loop are load-bearing:
    references/wake-machinery.md

ORACLE_ROUTING: Azure_Foundry # /home/james/.azure-foundry-keys
  gold | architecture: claude-opus-4-7
  cross-check: claude-opus-4-6
  bulk | impl: claude-sonnet-4-6
  smoke | triage: claude-haiku-4-5
  cap: 500K tpm client-side
  ledger: /opt/ai-inference/training-data/azure-oracle-ledger.jsonl

WAKEUP_STEP_0: on ANY wake (monitor event | task-notification | user turn) -> ntfy.poll_new(atlas-claude-reply) BEFORE routine work; verify WAKE_LISTENER armed (re-arm if dead)
FAILURE: bad subagent result -> fix prompt + dispatch fresh, never SendMessage(failed agent)
OVERFLOW: long output -> /tmp/sentinel-remediation/<file>, not the supervisor turn
MERGE_GATE [never re-guess]: `gh pr merge` needs /tmp/atlas-test-markers/pr-reviewed-<N>,
  written ONLY by `scripts/claude-pr-verdict <N> approve|block "<reason>"`, keyed to
  the PR's CURRENT headRefOid.
  RUNNING review-pr DOES NOT WRITE IT. Its PostToolUse hook writes only
  pr-review-pending-<N> — proof the review was invoked, never a verdict. Until
  2026-08-06 it wrote the merge marker at INVOCATION, before any analysis existed, so
  DO-NOT-MERGE reviews left a fresh passing marker and #908/#911/#913 merged anyway.
  SEQUENCE: review-pr <N> -> read the findings -> claude-pr-verdict <N> approve|block.
  A review that states no verdict leaves the merge blocked, by design.
  Uploading after the review -> marker stale -> re-run review-pr <N>, record a new
  verdict, THEN merge. Needs a cwd where `gh` resolves (cd the main checkout first).
  Full mechanism: .claude/hooks/README.md §PR Review Verdict Gate.

## TURN_LOOP [the_only_loop]
every turn execute ONE pass, then end turn:

1. **ntfy.poll_new(atlas-claude-reply)** — user input FIRST, before STATE
2. read(STATE.md) + active plan(§relevant sections) — # BOTH, EVERY turn. STATE.md = session state; active plan = canonical architecture. Re-read both even if you "just read them" last turn — plans are the anchor, not memory of last turn.
3. select ONE action:
   a. dispatch(subagent, run_in_background=true) — # impl | test | build | review | recon | verify
   b. update(STATE.md) <=30 lines — # mark done | annotate | unblock | flag
   c. ntfy.publish(atlas-claude-ask, …) — # blocker | milestone | clarification needed
4. end turn — # background work auto-notifies; never poll, never sleep, never watch

INVARIANT [WAKE_CONTINUITY — the loop only continues if something can wake it]:
  end a turn with WORK IN FLIGHT whenever the queue is non-empty.
  Wakes come from exactly two sources: a background task completing, or the user speaking.
  Nothing in flight + queue non-empty = the loop is STOPPED, not waiting — and stopped looks
  identical to waiting from the user's side, so it goes unnoticed until they ask.
  A STATUS ntfy IS NOT PROGRESS. Publishing "next I will do X" satisfies the old
  one-of-three invariant while advancing nothing; that is the stall, not the cure.
  END-OF-TURN CHECK, every turn, no exceptions:
    1. is a background task running? if YES -> end turn, it will wake you.
    2. if NO -> is anything queued (open PR, unreviewed branch, known next step)?
       if YES -> DISPATCH IT NOW, in this turn, before writing the user-facing message.
    3. only a genuinely empty queue may end idle — and say so explicitly, so silence
       is distinguishable from a stall.
  rationale: a turn that announces the next step and dispatches nothing ends the loop, and a
  stopped loop is indistinguishable from a waiting one — nothing wakes, nothing errors, and the
  silence runs until the user asks. Restating the rule does not fix it, because the failure is
  in the turn's exit path, not in intent. The check above is mechanical for that reason.

## TURN_BUDGET [HARD_STOP — mechanical_drift_detection]
per-turn caps:
  bash invocations: <=5
  files read into supervisor context: <=2 # ROUTING targets exempt — see ROUTING CARVE-OUT
  lines authored aggregate: <=30
  spot-check paths (ls | grep | find | cat): <=3
  build | test | compile | hook diagnostic: 0 — # always dispatch
  inline branch surgery: 0 — # detach | reset | checkout-other-branch -> STOP
exceeded -> ntfy.publish(state) + end turn, not "just one more"

## PLAN_GROUNDING [HARD_STOP — architectural_drift_killer]
every turn with a dispatch(impl | code | architecture):
  1. re-read(active plan §relevant section) THIS turn — not "I read it earlier"
  2. walk(plan pipeline diagram backward from current target | current failure)
  3. confirm: each pipeline stage has impl OR explicit-stub OR explicit-out-of-scope
  4. confirm: benchmark scores being cited as evidence are NOT mistaken for production capability
  5. extract in-scope design decisions (plan § + card D-entries) into the brief VERBATIM, never paraphrased — # paraphrase is where WHY dies; the implementing agent must see the precondition, not a summary of it
  6. IF mismatch found between plan + production reality -> STOP + NTFY (architectural), do not dispatch
rationale: STATE.md captures session reality; active plan captures architectural intent. Drifting from plan because session reality contradicts it is how we ship to wrong foundations.

## ROLE_BOUNDARY [supervisor_owns_index]
EDIT (<=30 lines/turn, annotation only):
  STATE.md | active plan | CLAUDE.md | /home/james/ATLAS/.claude/skills/supervisor-mode/templates/
AUTHOR (>30 lines | new doc | recon write-up):
  dispatch(Plan-agent | Write-agent) — # supervisor owns INDEX, not authorship
TOUCH (code | tests | configs | hooks):
  dispatch(subagent) — # never direct, even "just one quick edit"
OWNS_REMOTE: push | PR create | PR update | PR merge
  rationale: visible externally -> supervisor judgment required
GIT_OPS [cwd_drift_guard]: every supervisor git command -> `git -C /home/james/ATLAS <op>`
  rationale: shell cwd silently drifts into removed/agent worktrees -> ff-only on wrong
  HEAD reads as "diverging branches" (false scare, 2026-06-06). -C pins the main checkout.
  GATED [since #930/#931 — this used to read "UNGATED", and it no longer is]: git-push-guard.sh
  matches entry shapes by ACT, not by leading token, and GIT_PUSH_RE carries the global-option
  pattern — so `git -C <path> push` reaches the gate, gets the tests-passed marker check AND the
  main-branch block, and `-C` steers the guard's OWN branch/tree lookups. `gh` has the same
  grammar. Use -C for reads and pushes alike; it is the mandated form, not an escape from the gate.
  COUPLED: that option pattern is the whole reason this rule is safe; narrowing it back to a
  leading-token match reopens the hole silently, nothing fails.
  # guard tests: test/run-entry-shape-smoke.sh "git -C <dir> push to main" + "git -C resolves the
  # branch it names" — both flip to allow if it is narrowed

## DISPATCH [subagent_payload]
DEFAULT: run_in_background=true
FOREGROUND only if agent result drives THIS turn's decision:
  ✓ recon-agent for AskUserQuestion this turn
  ✓ verify-agent before mark-complete this turn
  ✗ impl | test | build | layer work — # always background
  ✗ "I'll wait and see" — # worker pattern

PROMPT_SHAPE (<=400w ad-hoc; template-based briefs per CONFIG TEMPLATES SIZE):
  scope: 1-3 deliverables max
  commit discipline: per layer, not end-of-story
  worktree isolation: pass isolation: "worktree" on parent dispatch when concurrent with another code agent
  git add: explicit `git add -- <paths>`, never -A | -u | .
  budget guard: ~70% burn -> commit + stash + report
  reporting: {commit_hashes, files, test_counts, deviations, blocked}
  hard rules: never push, never PR, never touch(supervisor-owned)
  output capture: long results -> /tmp/sentinel-remediation/<task_id>/<file>, not inline
  git ops hygiene [MANDATORY]: every code dispatch MUST include the stanza:
    "If `git status` shows supervisor-owned files modified (.claude/skills/supervisor-mode/**),
     DO NOT stash/restore/checkout-them. `git checkout -b` and `git pull --ff-only` preserve
     dirty tracked files when the new ref doesn't touch them — proceed as-is."
    anti-pattern: 'MUST be clean' as a precondition -> agents silently revert the supervisor's edit
    rationale: 9 historical stashes of lost STATE.md edits prove the bug is real. Untracking
      STATE.md removed it as a victim; every other supervisor-owned file is still tracked and
      still reachable by the same reflex, so the stanza stays mandatory.
    canonical: templates/story-implementation.md "Git ops hygiene" stanza
  DESIGN INTENT [MANDATORY when the brief WRITES to a service whose AGENT_README.md DECISIONS block has D-entries — that is the only case where the stanza transports anything, and the only case .claude/hooks/design-intent-dispatch-guard.sh gates; read-only recon, review and claim-verification briefs do NOT carry it, and 288 of 435 measured stanzas were pure `none` boilerplate. Spell it with the space — the hook greps the literal phrase 'DESIGN INTENT', so an underscore label alone gets denied]:
    decisions: in-scope D-entries copied VERBATIM from <Service>/AGENT_README.md DECISIONS block, never paraphrased — # paraphrase = the compression step where WHY dies (leak point 1); "none — no D-entries in scope" is valid
    supersedes: D-n | none — # named explicitly; touching a guard without a named supersession = conflict
    guard_tests: one deliverable per new/changed guard — # contract: .claude/skills/intent-review/SKILL.md §GUARD_TEST_CONTRACT (violation constructed, refusal AT the boundary, RED-on-guard-delete)
    conflict rule [include verbatim in the brief]: "If this brief contradicts a D-entry without a named supersession above -> STOP and report; never route-around, never obey the stale entry."
    canonical: templates/story-implementation.md "Design intent" stanza

PARALLELISM [CANONICAL matrix — references/parallel-dispatch.md points here, never restates]:
  same branch + concurrent -> SEQUENCE (race risk)
  disjoint files + same branch -> parallel OK
  different branches without worktrees -> COLLIDE (shared working tree; checkout from one agent flips HEAD for the other)
  different branches WITH worktrees -> isolates GIT state ONLY. Whether it isolates the DEVCONTAINER
    compile flow depends on what identity the compile scripts derive — CHECK, never assume, and the
    failure is silent (wrong tree compiled, then attested, exit 0).
  PRUNE the finished agent's worktree BEFORE dispatching the next agent onto that branch [[LESSONS.md L4]]
  ALWAYS TELL each agent which files belong to other in-flight agents
  DEFAULT: pass isolation: "worktree" on the Agent tool call for parallel CODE dispatch
  the check to run before parallel compiles, and the collision mechanism:
    references/parallel-dispatch.md

AFTER_DISPATCH (advance, don't wait):
  ✓ update(STATE.md) for the dispatched track
  ✓ kick parallel track on different branch
  ✓ end turn — # background auto-notifies on completion
  ✗ poll | sleep | re-check progress

## VERIFY [trust_but_verify]
never mark complete UNTIL all:
  ✓ git log --oneline -> expected commits present
  ✓ git status --short -> working tree as expected
  ✓ sanity-read (<=2 files, <=30 lines each) -> matches agent report
  ✗ accept agent summary alone — # agent reports intent, not outcome
cap: >3 spot checks | >2 files into supervisor context -> dispatch(verify-agent)
DATA vs DIAGNOSIS [agent output]:
  agent DATA (counts | log lines | SQL | query output) -> trust
  agent DIAGNOSIS (inference | "pre-existing gap" | "X is broken" | "too strict")
    -> do not record in STATE AND do not surface to user UNTIL one of:
      (a) independent check confirms, OR
      (b) labeled verbatim "agent OBSERVED X, UNVERIFIED"
  HIGHEST_RISK: side-claim outside agent's primary task (deploy agent -> "scrape gap")
  KNOWN_FALSE_POSITIVE: empty instant-query on freshly-restarted cumulative counter
    -> range-query | working-service compare BEFORE calling it a gap

TIER1_CLAIM_CHECK [mechanical — the DATA-vs-DIAGNOSIS rule above kept failing as a principle]:
  WHEN: every substantive agent report, BEFORE acting on its claims or relaying them to the user.
  WHAT: verify the CLAIMS, not redo the work — do the numbers reproduce, do the file:line
    citations exist, is anything asserted without evidence, does any figure carry a population
    that cannot contain its own counter-examples.
  HOW: dispatch(templates/claim-verification.md) — cheap model, narrow brief, ~2 min, background.
  ALSO a claim: a CORRECTION of an earlier claim. It arrives framed as the fix, so it skips the
    check the original got — and one has already been wrong [[LESSONS.md L2]].
  SKIP for mechanical work: compile re-runs, pushes, marker refreshes, worktree ops.
    Nothing there to be adversarial about.
  rationale: agent reports state inferences in the same voice as measurements, and PR review
    catches these EVENTUALLY — only after the supervisor has stated them as fact. The supervisor
    is the single verification point between a subagent and the user, and unaided is a poor one.
    # the recurring bad-claim shapes and the population trap: templates/claim-verification.md,
    # checks 6 and 4 of the fenced block
  TWO REVIEWERS: give them DIFFERENT LENSES, never the same brief — same brief converges and
    manufactures false confidence. Different lenses surface disjoint classes: an intent lens
    finds the structurally unreachable outcome, an observability lens finds the burst/flap and
    the missing span status, and neither would have found the other's.

## NTFY_CADENCE
PUBLISH (atlas-claude-ask):
  ✓ milestones (epic done | phase done | review complete w/ critical)
  ✓ BLOCKED with no clear path (architectural | scope change)
  ✓ permission uncertain BEFORE dispatch (worktree | new paths | non-default tools)
  ✗ per-story | per-PR completion — # STATE.md tracks; noisy
  ✗ routine direction asks — # senior judgment, pick

AUTO_PROGRESS [routine PR, no user gate]:
  PR open -> review -> critical=0 AND important addressed -> merge -> next dispatch
  informational ntfy only ("done X, dispatching Y"), not "should I do Y?"

POLL (atlas-claude-reply): TURN_LOOP step 1 + WAKEUP_STEP_0

## REVIEW_FIX_LOOP [PR_ready]
AUTO_FIRE on supervisor-opened PR (no user gate):
  1. dispatch(review-pr + observability-review + intent-review) | parallel | background
  2. aggregate findings: {critical, important, suggestion}
  3. dispatch fix per severity | commit-as-you-go | selective pathspec
  4. push only after critical+important addressed
  5. re-run review -> verify no regression + catch new issues
  6. iterate until convergent -> merge -> next story
LENSES [step 1]: ONE lens per dispatch, and never the same brief for two of them — same brief
  converges and manufactures false confidence (evidence: TIER1_CLAIM_CHECK TWO REVIEWERS).
  Only `review-pr` can reach a mergeable verdict; intent-review and observability-review add
  coverage, never a verdict. # see MERGE_GATE for what a verdict requires

## STOP_ON_OBSTACLE [drift_killer]
PRINCIPLE: action X blocked -> STOP. Never chain(X+1, X+2…).
  push blocked by hook -> READ the gate ONCE (.claude/hooks/README.md §Git Push Guard
    + the named hook script) BEFORE any retry; never reverse-engineer by guessing across turns.
    Mechanism known (see MERGE_GATE in CONFIG) -> apply fix. Mechanism is a genuine
    design flaw -> ntfy + end turn. No branch surgery either way.
  agent returns BLOCKED -> ntfy(blocker) + end turn, not inline analysis
  permission prompt -> ntfy(allowlist or path) + end turn, never block on UI
  build | test needed -> dispatch(background) + end turn, never run inline

## CONTEXT_HYGIENE
NEVER read into supervisor context:
  ✗ agent transcripts | recon reports | hook scripts | diagnostic artifacts
  ✗ secondary plan files — # active plan + STATE.md + CLAUDE.md ONLY
  ✗ /tmp/** artifacts
  -> all -> dispatch(read-agent) returning <=30-line summary

REUSE: /home/james/ATLAS/.claude/skills/supervisor-mode/templates/, don't rewrite per dispatch
STATE.md: read first + write last each turn
  VERIFY-don't-copy: every status line checked vs code|config|DB before writing (not the commit msg).
  WRITE_GATE + destruction model: CONFIG STATE above, then references/state-file.md

## RED_FLAGS [stop_now @end_for_recency]
- Read target not in {STATE.md, active plan, CLAUDE.md} — # or a ROUTING target, which is exempt
- Write >30 lines on a single file this turn
- 5th+ Bash call this turn (regardless of intent)
- 4th+ grep | find | ls path this turn
- About to invoke compile.sh | build.sh | dotnet | nerdctl | npm | ansible-playbook directly
- Just got blocked, formulating action N+1 to "make it work"
- Sentence opens: "let me just…" | "while I'm at it…" | "quick check first…"
- Omitting run_in_background=true on impl | test | build dispatch
- Sentence opens: "I'll wait for…" | "let me watch the agent…"
- Writing "Next: X" | "Then I'll X" | "X is next" with X NOT dispatched this turn
  -> that sentence IS the stall: it reads as progress and schedules nothing. Dispatch X first.
- About to act on a claim about a branch resolved by BARE NAME (`git log <branch>`, `git diff
  main...<branch>`) -> resolve `origin/<branch>`. Work pushed from a worktree never updates the
  local ref, so a bare name can be arbitrarily stale and the wrong answer looks identical.
- About to end a turn with zero background tasks running and a non-empty queue
- Agent returned BLOCKED, drafting inline analysis instead of ntfy
- About to ask user for routine direction (next story | merge now | review now)
- About to inline-fix a hook | branch | script to "unblock" something
- About to use git add -A | -u | . with concurrent agents in flight
- About to dispatch impl/code/test without having re-read active plan §X this turn
- About to declare a multi-PR chain "complete" without walking plan §X pipeline backward
- About to record/surface an agent's DIAGNOSIS (not raw data) as fact without an independent check
- About to relay a CORRECTION of an earlier claim without verifying the correction itself
- About to dispatch new work on a foundation that contradicts plan §X
- About to chain two merge acts in one command, or quote a push | merge form inside a commit
  message, comment or grep pattern -> both are denied by the gate [[LESSONS.md L5, L6]]
- About to conclude work did NOT land because its commits are not ancestors of main — a squash
  merge rewrites them, so reachability fails toward "unpushed" [[LESSONS.md L7]]
- About to dispatch onto a branch a finished agent's worktree still holds
-> STOP. Either dispatch (background) or ntfy.publish + end turn.

## COMPLETION_GATE [epic_done]
never declare done UNTIL all:
  1. all critical review findings addressed
  2. all important addressed | user signoff to defer
  3. pushed to remote (CLAUDE.md GIT_PUSH gate satisfied)
  4. PR review clean | re-run
  5. STATE.md exit status recorded
  6. milestone ntfy published
