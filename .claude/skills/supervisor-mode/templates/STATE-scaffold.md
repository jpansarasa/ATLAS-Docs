<!--
NOT A DISPATCH BRIEF. Every other file in templates/ is one; this is a file scaffold for STATE.md.

INSTANTIATE (strips this header, so nothing here reaches the live file):
  sed '1,/^SCAFFOLD-HEADER-END$/d' .claude/skills/supervisor-mode/templates/STATE-scaffold.md > STATE.md

FIRST, CLOSE OUT THE OUTGOING EPIC. STATE.md is gitignored: the copy has no undo and no git history.
Nothing below rescues content you failed to evict, so evict before you copy.

  1. archive it:  cp STATE.md ~/atlas-ops/state-archive/STATE.md.<epic-slug>-<YYYYMMDD>
  2. EVICT, entry by entry — anything that outlives this epic:
       defect | measurement debt | deferred work | parked epic -> docs/BACKLOG.md, WITH its measurement
       engineering | deploy | observability rule                -> CLAUDE.md
       failure mode that RECURRED (twice, not once)             -> LESSONS.md, per its WRITE_TRIGGER
       service shape | invariant | precondition                 -> <Service>/AGENT_README.md D-entry
       durable user direction that exists in no artifact        -> supervisor memory
       what happened                                            -> git log, PR bodies, tag, docs/RELEASES.md
     # the test: read each line and ask "does this change what someone does NEXT, in the NEW epic?"
     # no -> it is history or it is durable; either way it does not belong in the new file
  3. phase/epic tag + RELEASES.md entry: CLAUDE.md PHASE_TAGS
  4. only then instantiate, and fill every <angle-bracket> placeholder
  5. delete any section you left empty — an empty heading reads as "nothing here yet" when the truth
     is "nobody filled it in", and the next agent cannot tell those apart

THE ACCEPTANCE BLOCK IS THE POINT OF THIS SCAFFOLD, not boilerplate. An epic whose bar lives only in a
plan doc gets its bar renegotiated downward and nobody notices: measured on the news-extraction epic,
where >90% completeness became per-stage recall, became `cells > 0`, became a phase recorded DONE on
"2 cells from 1/10 articles" on a path that had scored 0/10. Four silent narrowings, each locally
reasonable. Write the bar HERE, as something that can FAIL, before the first dispatch.
SCAFFOLD-HEADER-END
# ATLAS Supervisor STATE

**Working memory for the epic at hand. Disposable by design.**
If this file were deleted, it should be rebuildable in ONE turn from the plan of record, `gh pr list`
and `git log`. That is the test of whether it is holding the right things — if losing it would be a
disaster, something durable has moved in and needs evicting. Treating it as precious is what grew the
previous one to 9,772 words (~17k tokens EVERY turn) before it was reset.

WHERE THINGS LIVE — never restate these here, point at them:
  defects, deferred work, parked epics -> `docs/BACKLOG.md`
  engineering + deploy + observability rules -> `CLAUDE.md`
  supervisor behaviour, turn loop, merge gate -> `.claude/skills/supervisor-mode/SKILL.md`
  recurring failure modes agents must read -> `.claude/skills/supervisor-mode/LESSONS.md`
  service shape, invariants, D-entries -> `<Service>/AGENT_README.md`
  what happened -> git log, PR bodies, tags, `docs/RELEASES.md`
WRITE_GATE: current position and what aims the next dispatch. ✗ merged-PR summaries ✗ review verdicts
  ✗ per-turn progress ✗ dispatch IDs ✗ anything that outlives this epic.
UNTRACKED + gitignored, so `grep -r` cannot see it — read it by explicit path.

## STANDING DIRECTIVES
<Live user direction governing THIS epic — scope, autonomy, cost ceilings, hard prohibitions.
 Quote the user verbatim and date it. If a directive is durable rather than epic-scoped, it belongs in
 supervisor memory instead: it would not survive the next reset, and a reset is not a repeal.>

## ACTIVE EPIC — <name>
PLAN OF RECORD: <path to the plan, spec or PR that is canonical for this work>

**ACCEPTANCE — what "done" means, written before the first dispatch.**
<State the bar as something that can FAIL, and name the thing that runs it: a test, a harness, a query,
 a scorecard. "Prose criteria in a doc nobody executes" is the failure mode, not the cure.
 If no executable bar exists yet, BUILDING ONE IS THE FIRST STORY — say so here rather than starting
 the work with the bar unstated.>
CHANGING THIS IS AN EXPLICIT ACT. Narrowing the bar mid-epic is legitimate and often correct — a PoC
gate is not a product gate — but it is an EDIT to this block with the reason, never a quiet
substitution downstream. If you are about to accept a weaker result than this block asks for, change
this block first, or you are renegotiating the bar rather than meeting it.

POSITION:
✓ <phase done — one line, and the durable consequence if there is one>
-> <phase in flight — what is running, and the measurement that aims the next dispatch>
◯ <phase not started — the precondition it is waiting on>

**NEXT DECISION:** <the one real choice in front of the supervisor, and what evidence would settle it.
 "Unchosen, pending shadow data on live traffic" is a legitimate and useful entry. "None" is also
 legitimate — then the next action is a dispatch, not a decision.>

**DO NOT RETRY — measured and rejected inside this epic:**
<Starts empty. Every rejected approach lands here WITH its number the moment it is measured, so the
 epic cannot re-run an experiment it already paid for. This section is what makes a long epic cheaper
 rather than more expensive as it goes.>

## IN FLIGHT
<Open PRs and their state, live worktrees, anything dispatched and not yet returned. This is the only
 section that is genuinely per-turn, and it should be short enough to rewrite rather than amend.>
