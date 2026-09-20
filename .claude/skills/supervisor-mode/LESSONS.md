# SUPERVISOR_LESSONS

Generalizations paid for once, kept where every DISPATCHED AGENT can read them # subagents cannot read supervisor
memory, so a lesson filed there gets hand-transcribed into briefs, and two were corrupted that way on 2026-08-07/08.
A LESSON IS A GENERALIZATION YOU CAN APPLY SOMEWHERE ELSE. One that names a single incident and dies when that
incident is fixed is a commit message: it belongs in git, or in docs/BACKLOG.md carrying its measurement.

## GRADUATION_RULE [read before adding anything]
A lesson's goal is to STOP being a lesson: once encoded in a template, skill, hook or checklist -> DELETE it here and
  leave a pointer under ALREADY_ENCODED # a lesson enforced in two places drifts in one of them
ENCODING IS THE GRADUATION AND IT OUTRANKS THE CHECK: a GRADUATE_CHECK still failing is a statement about the
  ENFORCEMENT artifact, never a licence to keep the entry -- "over-retention is the cheap error" below is scoped to a
  check that could not RUN # measured 2026-09-20: three entries encoded into two templates were kept anyway, on that
  clause read backwards as a precondition
  AN EMPTY `## LESSONS` IS THE GOAL STATE, and `scripts/new-epic.sh` now reports it clean on three conditions --
  heading present, its section EMPTY, ALREADY_ENCODED populated # until 2026-09-20 it refused the empty section as
  unparseable, which made the one outcome this rule aims at the one shape that could never pass
SHRINK as often as you grow; add nothing enforced elsewhere. CITE anchors, never file:line # they rotted 3x in a day
WRITE_TRIGGER: a failure RECURS -> here, at the VERDICT and not the retro; a FIRST occurrence goes to docs/BACKLOG.md,
  which carries it across epics until it recurs. A METHOD failure leaves no artifact, so it never trips this by
  itself -- which is why this file filled with artifact facts and stayed silent on method, the thing it exists for.
  Its observables: a PR over its class budget (`review-discipline` ROUND_BUDGET), a proposition adjudicated a third
  time (`re-derive` PASS_COUNTER). WHICH STORE, never both: a DISPATCHED AGENT can act on it -> here; supervisor
  judgement or recall -> memory only.
RULE_MUST_BE_CHEAP: name the CHEAPEST SUFFICIENT action, not the most correct one # the correction lesson existed, was
  exactly on point, and did NOT fire: its remedy was "dispatch an agent" against a one-line refutation
FORMAT, all five or it is not an entry: the generalization in ONE line / EVIDENCE, re-checkable / RULE / GRADUATES,
  NAMING THE ARTIFACT THAT WILL HOLD THIS RULE and why it cannot hold it today / GRADUATE_CHECK. Naming the
  destination at WRITE time IS the out-flow -- the in-flow fires at every verdict and the audit only at ~10-day epic
  boundaries, so an entry that never named where it was going becomes the queue that invites `--evicted`. An entry
  whose destination exists TODAY is not an entry; it is an edit to that artifact.
GRADUATE_CHECK: a shell predicate, run from the repo ROOT, copy-pastable, EXITING 0 ONCE THE LESSON HAS GRADUATED;
  `none -- judgement` is the honest value and an absent line is silence wearing a verdict. `scripts/new-epic.sh` runs
  every one at each epic boundary and REFUSES the reset while any exits 0 -- promote that lesson and delete it, or
  record why not and re-run `--evicted`. Over-retention is the cheap error; a check that CANNOT RUN (rc 124/126/127)
  blocks as broken. MATCH THE ARTIFACT, NEVER THE PROSE ABOUT IT: scan FILES not directories, exclude comment lines
  where the clause says the artifact DOES something, require the WIRING where being wired is the point, and never end
  a listing pipeline with `xargs -r grep -q` whose empty input exits 0 and reads as GRADUATED # a directory grep once
  matched an agent's own audit-log prose about running a tool and reported the tool wired
  THESE LINES EXECUTE -- the same trust boundary as the script that runs them.

## ALREADY_ENCODED [go there, never restate here]
Every line NAMES the mechanism enforcing it now, so removing that mechanism removes a visible pointer instead of
  silently losing the lesson # a deleted entry with no pointer cannot be told from one that was never earned
a brief's numbers, cited lines, root causes and severities are HYPOTHESES, and findings are claims verified before they are fixed -> `templates/implementation-fix.md` TRAJECTORY step 2 # was L1
a CORRECTION is a claim, and a refutation licensing a DELETE gets a derivation -> `.claude/skills/re-derive/SKILL.md` + `templates/claim-verification.md` step 7 # was L2
prune a finished agent's worktree, sweeping for STAGED indexes and not only for held branches -> `templates/implementation-fix.md` Notes for the supervisor # was L4
a fix round fixes what you NAME and adds what you did not ask for -> `templates/implementation-fix.md` CONSTRAINTS, the ceiling-not-floor bullet # was L9
build AFTER the final commit; the marker keys to the TREE -> CLAUDE.md GIT_PUSH + both dispatch templates + `.claude/hooks/commit-marker-staleness.sh` # was L10
a hand-rolled measurement harness fails toward SUCCESS unless you stop it -> `templates/recon-measurement.md` TRAJECTORY step 3 # was L13
a guard matching a DESCRIPTION of an act inherits the whole grammar of the description, so a fix that handles one more grammar construct is priced out loud as converging on a reimplementation of bash, and a bypass is bisected across the branch's OWN commits with over-denial reported beside it -> `.claude/skills/guard-change/SKILL.md` item 9 (GRAMMAR INHERITANCE and PRICE IT OUT LOUD), 18 and 16; the review-method half -> `review-discipline` FORCED_PASS # was L14
a guard whose scope includes its own SOURCE makes its own repair unreachable -> `guard-change` item 14; the open defect is docs/BACKLOG.md, the gate-layer deadlock entry # was L16
an alert that fires by ACCIDENT is not coverage -> CLAUDE.md OBSERVABILITY; the six ungauged D-18 series are docs/BACKLOG.md # was L17, whose enumerate-from-the-DATA-side half is NOT there: it is the PRE-HANDBACK item of that name in both code-dispatching templates, named on the SILENCE line below
a command reporting its own limit, warning or truncation has ANSWERED you -> `templates/claim-verification.md` step 8 # was L18
verify a claim before relaying it, and give two reviewers DIFFERENT lenses -> SKILL.md TIER1_CLAIM_CHECK + REVIEW_FIX_LOOP LENSES + `templates/claim-verification.md`
a block names the PROPERTY the artifact must guarantee and not the field the reviewer noticed, demands the INPUT-SPACE enumeration before the fix, scales the round to the delta, and carries no figure its author did not derive -> `.claude/skills/review-discipline/SKILL.md` §BLOCK_TEXT, pointed at from `templates/implementation-fix.md`, `templates/story-implementation.md`, SKILL.md REVIEW_FIX_LOOP step 3 and `references/brief-construction.md`; the anchor is resolved by `scripts/tests/test_verify_pointers.py::test_tracked_corpus_resolves`, so deleting the section turns CI red naming all five citing files, this line included # #1078, seven declared rounds against four
what a guard test must do to count -> `.claude/skills/intent-review/SKILL.md` GUARD_TEST_CONTRACT
a guard test that mutates only LOGIC cannot see an UNWIRED guard: delete the line that INSTALLS or CONSUMES it -> the same GUARD_TEST_CONTRACT, WIRING NOT ONLY LOGIC + the PRE-HANDBACK block in `templates/implementation-fix.md` and `templates/story-implementation.md`
a control proves nothing about the code path its author did not AIM it at, and the aim lands where they were already looking -- drive the ACT end to end asserting OUTPUT and EXIT CODE, rest no assertion on a SECOND COPY of the rule, and vary the axis the CODE READS by finding the fixture where the two candidate rules DISAGREE -> `.claude/skills/intent-review/SKILL.md` §AIMED AT THE ACT, pointed at from the PRE-HANDBACK block of both code-dispatching templates, `guard-change` ALREADY_COVERED and CLAUDE.md TOOL_UPKEEP; the anchor is resolved by `scripts/tests/test_verify_pointers.py::test_tracked_corpus_resolves`, which the python-tests workflow runs on any `**/*.md` change, so renaming the block turns CI red naming all SIX pointers at it # five instances in one day, 2026-09-20, none caught by its author
what the verdict marker attests -> SKILL.md MERGE_GATE + `.claude/hooks/README.md` PR Review Verdict Gate
find a recorded DECISION about a PR (BLOCKED, do-not-merge, superseded) BEFORE reviewing its code -> SKILL.md MERGE_GATE SEQUENCE step 0
analysis is not a review record -> SKILL.md MERGE_GATE, fail-closed by `.claude/hooks/pr-review-marker.sh` and `scripts/claude-pr-verdict` # was L3
one merge act per Bash invocation -> `.claude/hooks/git-push-guard.sh` denies any command carrying more than one + SKILL.md RED_FLAGS # was L5
prose quoting a gated push or merge form trips that gate; pass long text by path -> `.claude/hooks/README.md` Accepted cost # was L6; only the merge denies and the two-pushes deny name the remedy
squash merge makes commit reachability answer NO for work that landed -> SKILL.md RED_FLAGS; ask the PR's state or compare CONTENT # was L7
verify against the THING and never the PROXY -- the landing TEXT of a reference, an artifact you proved rebuilt -> `templates/implementation-fix.md` step 5 (show one mutant that compiled, `touch` after mutate AND restore) + CLAUDE.md TOOL_UPKEEP ANTI (compare the unresolved SET against a pristine baseline, never a count and never an rc) # was L8
an instrument proven only by SILENCE or at n=1 has pinned nothing, and coverage is enumerated from the DATA side -> all three halves are in BOTH code-dispatching templates, `templates/implementation-fix.md` and `templates/story-implementation.md`: the alert half at step 7 (the rule's case asserts `alertstate="firing"` on bursty input), SCALE-MATCHED MUTATION at step 5 / the Pre-handback bullet that now BEARS that name (it did not until 2026-09-20, so this pointer named text no grep could find), and ENUMERATE COVERAGE FROM THE DATA SIDE as the PRE-HANDBACK item of that name # was L11. Named as three destinations because two of them were claimed to survive "on this line", which is the entry pointing at itself -- a retirement whose rule landed nowhere. The `consumed cap` phrase two hook fixtures cite as their provenance lives on this line
a self-authored negative is scoped to its author's imagination -> `templates/implementation-fix.md` CONSTRAINTS (a corpus from a DIFFERENT MIND than the fix; its SIZE and the BASELINE it was measured against, with the #935 series that falsified every honest zero) + `guard-change` item 1 for the spellings half # was L15
a per-worktree devcontainer does not isolate what its tests share on a server outside it: a fixed database name dropped on the shared timescaledb collides across worktrees and across services -> each integration project's `Infrastructure/IntegrationDatabaseName`, which refuses to run without the owner key `scripts/devcontainer-owner.sh` hands over. A default `compile.sh` runs its `IntegrationDatabaseNameTests` only in SecMaster; the other six services run them only under `compile.sh --integration`, which the push marker does not require. A new fixture that CREATEs or DROPs a database names it there

## LESSONS

L19 A METRIC STEP CARRIES ITS OWN TIMESTAMP, and a deploy is a BUNDLE -- so "the deploy did it" is not an answer
  until you have read WHERE the step is and WHICH change in that deploy owns it. The deploy nearest your
  attention is the one you did not measure.
  EVIDENCE: twice on `secmaster_vector_search_duration_milliseconds`, the same metric both times.
    2026-09-17 -- a latency rise attributed to D-13's join; it was #1030 raising `hnsw.ef_search` 40 -> 400 in
    the SAME deploy, and the A/B that settled it put the two SQL variants at 7.14-8.97 ms and 8.04-15.72 ms in
    one arm, so the join was never the size of the step.
    2026-09-20 -- a step attributed to the 2026-09-17T14:08Z identity deploy. Re-derived at 1h steps, the share
    at or under 5 ms reads 84.5-87.6% through 22:00Z on 2026-09-16, 17.8% in the 23:00-00:00Z hour and 2.5-10.5%
    after, while p50/p95 either side of 14:08Z are 6.92/21.29 vs 7.12/21.18 -- unchanged. The cause sat 15 hours
    earlier and the docs/RELEASES.md entry "Vector-SQL latency rose with the deploy" already named it.
    Re-check either by plotting:
    `sum(increase(secmaster_vector_search_duration_milliseconds_bucket{le="5.0"}[30m])) / sum(increase(secmaster_vector_search_duration_milliseconds_bucket{le="+Inf"}[30m]))`
  RULE: before attributing a metric step to a deploy -- PLOT the metric as a RANGE across the whole candidate
    window, at a step finer than the gap between deploys, and read off where the step IS. Then read
    docs/RELEASES.md and the recent deploy log for everything else that shipped near it. A deploy carries every
    change in it, so name the CHANGE or say you cannot. Two candidates inside one deploy are separated by holding
    one fixed -- an A/B, a GUC set through PGOPTIONS -- never by which one you were already reviewing. The plot
    is minutes and free; it is the cheapest discriminator on the board and it went unrun both times.
  GRADUATES: when deploys ANNOTATE Grafana -- `deploy.yml` POSTing to `/api/annotations` with the deployed tags,
    so every latency panel draws the deploy lines and "where is the step relative to what shipped" is answered by
    looking instead of by recall. No such task is reachable from deploy.yml today, which is exactly why the deploy
    a human remembers wins over the one that moved the metric. The claim-verification template cannot hold this:
    it checks numbers a report ASSERTS, and here the defect is a CAUSE attached to a number that reproduced
    perfectly.
    THE CHECK BELOW asks ansible what deploy.yml REACHES, never what a file CONTAINS -- a grep for the task greens
    on one sitting in a playbook nothing imports, and `--list-tasks` executes nothing and needs no host. What it
    still does NOT prove: that the task SUCCEEDS at deploy time, or that its tags and timestamp are right. It
    fails closed -- no ansible, or a broken playbook, feeds grep an empty stream and the lesson stays. This note
    sits in GRADUATES because the parser reads the predicate as everything after `GRADUATE_CHECK:` ON THAT LINE:
    spelled `GRADUATE_CHECK [note]:` with the predicate below, the gate saw NO check at all and told the operator
    to add one, which would have made it a DUPLICATE. Measured 2026-09-20 against main's own script.
  GRADUATE_CHECK: ( cd deployment/ansible && ansible-playbook playbooks/deploy.yml --list-tasks 2>/dev/null ) | grep -qi annotation
## ANTI [HARD_STOP @end for recency]
never state a brief's mechanism, cited line, root cause or severity as settled fact
never relay "X is false" without restating X's proposition and naming its subject # RELAYING is not DISPUTING, which
  is why this line stays here while the rest lives in `.claude/skills/re-derive/SKILL.md`
never open a THIRD pass on one proposition with an opinion -- re-derive it # `.claude/skills/re-derive/SKILL.md`
never open review round 1 without a declared artifact class and budget, and never diff-scope the round that carries
  the approve # `.claude/skills/review-discipline/SKILL.md`
never spend an EXPENSIVE observation while a FREE one bearing on the same question is unrun
  # `.claude/skills/cheapest-discriminator/SKILL.md`
never hand a fix round a list of sites -- brief the CLASS, require the count before the fixes, then add nothing
never compile before the final commit -- the marker keys to the tree, not the content
never dispatch onto a branch a finished agent's worktree still holds, or leave one behind with a staged index
never dispatch gate-layer work worktree-isolated, and never create the confirm file yourself
never interpret a result whose row count EQUALS the limit you passed, or whose output carried a warning you did not
  read -- the instrument has already told you the answer is not an answer
never repair a reference before your last content edit, and never believe a PROXY over the thing it stands for:
  not a green sweep, not a FALLING cannot-land count, not a test result on a build you did not prove rebuilt
never accept a rule, alert or control proven only by silence, only at n=1, or only from the alert list
never accept a self-authored "nothing loosened", or a corpus built by the fix's own author
never attribute a metric step to a deploy you have not located the step against, or to a deploy rather than to a
  named CHANGE inside it -- plot the range first, then read RELEASES.md for what else shipped nearby [L19]
never add an entry that a template, skill, hook or checklist already enforces
never add an entry without a GRADUATES clause NAMING THE ARTIFACT that will hold it and a GRADUATE_CHECK -- `none --
  judgement` is an answer, an absent line is not, and `scripts/new-epic.sh` refuses the next epic reset over either
  that or a check that now PASSES
