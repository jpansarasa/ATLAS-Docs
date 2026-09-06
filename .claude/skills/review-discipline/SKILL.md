---
name: review-discipline
description: Deciding whether a PR needs ANOTHER review round, dispatching a review round, or recording a merge verdict. Supplies the STOP condition the review skills do not - a round budget declared by artifact class before round 1 and derived from this repo's own verdict log, a severity bar tied to consequence, a convergence test over finding CLASSES, a bounded scope for the pass a merge gate forces after every fix, and a termination record naming the findings accepted unfixed. review-pr, intent-review and observability-review say what to look FOR; this one says when to stop looking.
---

# REVIEW_DISCIPLINE [SKILL v1]

BUDGET: PROSE 2 | TOOLING 2 | PRODUCTION 4. A ROUND = one verdict recorded at a NEW head.
A TRIPWIRE, NOT A CAP - exhausting it forces a written re-declaration, never a merge over an open
critical. Everything below is the derivation, for the reader who disputes a number.

A careful reviewer always finds something. So "the last round found a real defect" is a
CONTINUATION signal that never runs out - it refuels itself, and a loop gated on it does not
terminate. It has run here: PR #980 changed ONE markdown file, recorded four verdicts across what
its own approving reviewer called six review rounds, and its third BLOCK landed on a paragraph the
previous round's fix had introduced.
Nothing in that sequence was wrong, and that is the point. What was missing is that no class and no
budget were ever declared, so no round could be judged against one - and the round count was not
knowable at all until someone read the audit log.
This skill supplies the stop condition: declared before round 1, judged on CONSEQUENCE, counted in a
unit that exists, and ended by a record rather than by silence.

## ETHOS
declare the budget BEFORE round 1        # a stop rule invented at round 4 fits whatever happened
consequence, not correctness             # the question is what WRONG ACTION a fix prevents
a new finding CLASS means sweeping       # converging rounds find less of the SAME thing
findings accepted unfixed are NORMAL     # an empty residue means the bar was never applied
terminate by RECORDING                   # an unrecorded stop cannot be told from abandonment

## THE_MOVE [the whole procedure; every section below is the depth behind one of these]
  1. BEFORE round 1: name the artifact CLASS and write its budget down.
  2. Each round: report SCOPE, NOT_EXAMINED, and per finding a CLASS plus the reader-and-wrong-action
     sentence. No sentence -> not a round's worth of finding.
  3. Count the round: one verdict at a new head. The gate-forced pass after a fix counts too.
  4. At the tripwire: RE-DECLARE in writing what is unconverged. Never merge over an open critical.
  5. The round that APPROVES re-attacks the OLDEST at full scope - never the fix diff.
  6. End by RECORDING rounds/budget, the RESIDUE, and the last NOT_EXAMINED.

## ROUND_BUDGET [declare it, and the class, before round 1]

UNIT [name it every time you quote a number]. A ROUND is one review pass that records a verdict at a
head no earlier verdict on that PR named. That is one head-deduplicated line of
`~/.claude/atlas-pr-verdict.log`, and it is the only round count this repo can actually produce.
  a round is NOT a review dispatch # guard-change/SKILL.md:8 counts #921 at 8 ROUNDS; the audit log
    holds 2 verdicts for it, having existed for TWO HOURS at that point (log opens 2026-08-07T10:57:52Z,
    #921's first verdict 12:53:25Z). Two real counts of different things on different windows -
    never summed, never compared.
  a round is NOT a commit # #950 carries 27 commits across 8 rounds, #1002 41 across 7.
  IT COUNTS HEAD CHURN, and that is not a defect: a rebase or amend re-fires the gate and records a
    verdict with no reviewable delta. The budgets below were derived in THIS SAME UNIT, so the
    measured distributions already absorb whatever churn this repo actually has.

DERIVATION, over a PINNED SNAPSHOT because the log is live and appends under you:
  awk '$1 < "2026-09-06T04:00:00Z"' ~/.claude/atlas-pr-verdict.log
200 lines, 101 PRs, of which 100 are classified. Only #935 is dropped, and it is CLOSED UNMERGED -
not a tooling gap. #974 and #975 landed as MERGE commits rather than squashes, so recover their diffs
with `git diff <mergeCommit>^1 <mergeCommit>`; an earlier pass dropped both for "no squash commit",
which excluded two SIX- and FOUR-round PRs from a derivation whose whole subject is the tail.
Class comes from the merged diff's paths. Each budget is the smallest number covering at least 85%
of that class's measured PRs.

  PROSE      2 rounds - docs, comments, PR and backlog text; nothing in the diff executes
  TOOLING    2 rounds - a script or harness whose OUTPUT a human reads and acts on
  PRODUCTION 4 rounds - code on a live path, a migration, a deploy artifact, or a gate
A DIFF SPANNING CLASSES TAKES THE HIGHEST, ordering PROSE = TOOLING < PRODUCTION. One deploy
  artifact under a hundred lines of docs is a PRODUCTION review # the paths decide, never the ratio

  PROSE      n=22, spread 1x17 2x2 3x2 4x1.            <=2 covers 19/22 = 86.4%
  TOOLING    n=20, spread 1x15 2x3 3x2.                <=2 covers 18/20 = 90.0%, max 3
  PRODUCTION n=58, spread 1x33 2x8 3x5 4x4 5x2 6x2 7x2 8x2.
                                                       <=4 covers 50/58 = 86.2%
                                                       <=3 only 79.3%, <=2 only 70.7%
THE CLASS BOUNDARIES ARE JUDGEMENT CALLS AND THE BUDGETS DO NOT DEPEND ON THEM. Two independent
  re-classifications under the same stated rule both returned PROSE 23, and they disagreed with each
  other about where the displaced 1-round PR goes - one to TOOLING, one to PRODUCTION. NO BUDGET
  MOVED in either. A third test moved `deployment/tests/**` wholesale from TOOLING to PRODUCTION,
  defensible since CI runs it as a gate: the 85% table stayed 2 / 2 / 4.
  Trust the budgets; treat the percentages as accurate to about one PR, and say which side
  `deployment/tests/**` falls on when you re-derive - this table counts it TOOLING.
TOOLING GETS NO +1 OVER PROSE, though the reasoning that it should is sound and worth keeping out of
  the next editor's way: a tool fails toward SUCCESS (CLAUDE.md TOOL_UPKEEP), so it needs more
  looking. Measured, TOOLING is the TIGHTEST of the three - most likely because tool PRs here ship
  with selftests, and a selftest finds findings before a reviewer does. The theory is right about
  tools and wrong about ROUNDS, which is the only thing it was being used to set.
PRODUCTION alone earns headroom: at 3 it covers 79.3% and at 2 only 70.7%.

WHY 85%, and the argument is about what STOPS FIRING, not about tidiness. Raise the bar to 90% and
the table becomes PROSE 3 / PRODUCTION 6. The tripwire then never fires on #977 or #982, nor on any
PRODUCTION PR under seven rounds - #949, #971, #975, #1004 at four and #954, #961 at five all go
quiet, and #948 and #974 at six with them. Those are the cases the budget exists to catch.
  # do NOT argue this from class ORDERING. An earlier draft did - "at 90% PROSE sits above TOOLING"
  # - and that rests on TOOLING's <=2 landing at exactly 90.00% (18/20). The independent
  # re-classification above puts it at 17/19 = 89.47%, which flips TOOLING to 3 and erases the
  # inversion. A reason that turns on one PR is not a reason; the stop-firing list holds either way.
The threshold is a CHOICE, written down so the next reader can move it against the same numbers
instead of arguing from taste.

MEASURED OVERRUNS - the loop this skill ends, in the log that recorded it:
  #980  PROSE, one markdown file, 4 rounds against a budget of 2. Its own approving reason says
    "Six review rounds". Its third BLOCK landed on a paragraph the fix pushed before it had
    introduced - "Delta commit 2f8d0055 introduced a defective SecMaster/CFIGY paragraph".
  #950  PRODUCTION, 8 rounds. The rule's SEMANTICS were pinned from round 2, which proves the
    rule-file diff comment-only and the parsed YAML hash-identical; rounds 6-8 re-prove it
    byte-identical (blob d27480a8). Rounds 3-8 then ran on the TEST file - round 2 still blocked on
    the rule file's own comment text - and round 8's delta was comments only.
  #951  PRODUCTION, 7 rounds. All four code blockers closed at round 2; rounds 3 through 7 ran on
    doc-only, comments-only and prose-only deltas - and round 4's whole blocking finding was that
    "the new ACCEPTED COST sentence is itself false", a sentence round 3's fix had just written.
NOT OVERRUNS, and they are this skill's own founding pair. #1016 ran 3 rounds against TOOLING 2,
  and the round that earned it was round 2, NOT round 3: round 2 BLOCKED on "known-bad control
  averages its floor across runs, so a breaching run is masked by clean ones" - textbook
  ACTION-CHANGING by CALIBRATION below. Round 3 is the APPROVE that verified the fix, which took the
  selftest from ten CONTROLS to fourteen (units: controls, not runs - the reason attests "14 of 14
  rc 0" and a diff "confined to the four LOW findings plus four selftest controls").
  SO THE THIRD ROUND IS STRUCTURAL, and this is the sharpest thing the log says about a budget of 2:
  the merge gate forces a pass after EVERY fix push, so a PR that blocks TWICE cannot finish in
  fewer than 3 rounds. A budget of 2 is met only by a PR with at most ONE blocking round. That is
  not an argument for raising it - 86.4% of PROSE and 90.0% of TOOLING PRs do finish inside it -
  but a second block is a re-declaration, by construction, and the tripwire is doing its job when
  it fires there. #1017 ran 3 against PRODUCTION 4, inside its budget.
  Neither shows a budget exceeded. Both show what NO declared budget costs: this file's own first
  draft published 5 rounds and 3 for PRs the log records at 3 and 3, and classed a 941-line Python
  tool as PROSE. The count was unknowable until someone read the log.

THE BUDGET IS A TRIPWIRE, NOT A CAP. Exhausting it neither authorises a merge nor forbids another
round. It forces a WRITTEN re-declaration naming what is still unconverged and how many more rounds
that needs. A critical finding open at exhaustion is an escalation, never a merge.
  # a hard cap would trade one silent failure for a worse one - merging on a schedule

## SEVERITY_BAR [consequence, not correctness]
For every finding name two things: the READER or CALLER, and the WRONG ACTION they take unfixed.
  both nameable -> ACTION-CHANGING -> fix it, this round
  not nameable  -> IMPERFECT       -> backlog entry or accepted residue, NEVER a round of its own
This is CLAUDE.md TOOL_UPKEEP's MISLEADS-vs-IMPERFECT test applied to a review finding; the
rationale lives there and is not restated here.
CALIBRATION, generalised from the rounds above:
  a wrong attribution in a headline or summary         -> ACTION-CHANGING # headlines quote forward
  a control that cannot see the breach it guards       -> ACTION-CHANGING # every green now worthless
  a guard whose subject empties, selftest still green  -> ACTION-CHANGING # the guard is decorative
  a defect on a path that only ever prints diagnostics -> IMPERFECT
  formatting, line length, spelling                    -> IMPERFECT
BORDERLINE - misleading, but only to a reader already inside that file (an inverted rationale in a
comment): it rides along with a push that is already happening. It does not BUY a round, and a
round opened for it is the loop restarting.

## CONVERGENCE_TEST [make the sweep observable]
Every round labels EVERY finding with one CLASS:
  FACT       a claim the artifact makes is untrue
  BLINDNESS  a check, control or test cannot see what it claims to
  SCOPE      the artifact does or touches something it does not say it does
  MECHANISM  the logic does not do what its own description says
  STYLE      form only
Then compare round N against round N-1:
  same classes, fewer or smaller findings -> CONVERGING; continue inside the budget
  a class absent from every prior round   -> the review is SWEEPING NEW GROUND, not converging.
    Round 1's scope was wrong. Say that in the round report - the finding is real AND the process
    is defective, and the second half is invisible unless the classes are written down.
  two consecutive rounds each introducing a new class -> STOP and escalate. Either the artifact is
    under-specified for review or the reviewer is unbounded, and another round decides neither.
rationale: in a converging review severity trends down and the CLASS SET narrows. On #951 the class
  set MOVED instead of narrowing - the code closed at round 2 and five prose rounds followed, at
  least one of them (round 4) blocking on a sentence round 3's fix had just written. Nothing
  recorded the classes, so nothing could read either signal.

## ROUND_REPORT [what a round must state to be a round]
  1. SCOPE examined - paths, sections, aspects
  2. NOT_EXAMINED: <what was skipped> because <reason>
  3. per finding: CLASS + severity + the reader-and-wrong-action sentence
  4. the verdict, and this round's number against the declared budget
TWO AXES, NOT ONE. CLASS feeds the convergence test and sets NO severity. Severity comes from
SEVERITY_BAR alone, and maps onto the review skills' {critical, important, suggestion}:
  ACTION-CHANGING, wrong action taken by someone OUTSIDE this PR -> critical   -> verdict block
  ACTION-CHANGING, wrong action confined to whoever finishes it  -> important  -> fix, then approve
  IMPERFECT                                                      -> suggestion -> residue
One open critical blocks; a round that found five importants and no critical is an approve.
A LENS WITH ITS OWN SEVERITY_MAP OUTRANKS THIS ONE for the findings it types. intent-review/SKILL.md
  SEVERITY_MAP fixes CHECK_3, a missing or tautological guard test, at IMPORTANT - while CALIBRATION
  above would read the same shape as ACTION-CHANGING, hence critical. Both lenses run in ONE fan-out
  (supervisor-mode REVIEW_FIX_LOOP step 1), so an unresolved collision is one finding with two
  verdicts depending on which file the aggregator read last. The lens that OWNS the check wins;
  SEVERITY_BAR governs everything no lens has already typed.
NOT_EXAMINED is what makes the NEXT pass bounded instead of a fresh sweep of the same artifact -
and a fresh sweep is precisely the act that always finds something. `NOT_EXAMINED: nothing` claims
total coverage: rarely true, and it hands the residual risk to nobody.

## FORCED_PASS [the round the merge gate demands after every fix]
A verdict gate keyed to the CURRENT head (ATLAS: supervisor-mode SKILL.md MERGE_GATE) forces a
review invocation after every fix push. It CONSUMES A ROUND - it records a verdict at a new head,
which is the definition above. Calling it free is what the first draft did, and with a round defined
as a pass plus its fix push, every pass after the first was then free and the counter could never
reach 2 # keep both halves aligned or the budget silently stops being able to trip
What this pass gets is not exemption from the count but a NARROW SCOPE and a licence to find nothing.
  CHOOSE THE SCOPE EX ANTE, and this is the trap: scope is picked BEFORE the round runs, while
  terminality is only known AFTER. So take the narrow scope ONLY when you already know this round
  cannot approve - a critical is open, or the budget has >=2 rounds left. Otherwise go full scope.
  AT A BUDGET OF 2 THE NARROW SCOPE NEVER APPLIES IN BUDGET: round 1 is full scope and round 2 is
  both the forced pass and the round that approves. That is 42 of the 100 classified PRs. Using the
  licence there does not save a round, it spends an extra one - and buys the #970 failure below.
  BASELINE: the head of this PR's LAST recorded verdict, from the append-only audit log:
    awk -v t=PR#<N> '$2==t{print $4}' ~/.claude/atlas-pr-verdict.log | tail -1
    # exact field, not `grep "PR#<N> "` - a reason mentioning another PR would match the loose form
    NOT the review-pending marker - it is written with a truncating redirect and holds only the
    NEWEST head, so a diff against it is <head>..<head> and returns empty.
    EMPTY on a PR's first review, correctly - this section is only ever the round AFTER a fix.
  SCOPE: that diff, plus whatever the fix CLAIMED to change.
  QUESTION: did the fix do what it claimed, and did it break what it touched? Nothing else.
  LICENCE: "nothing outside the claimed scope" is a COMPLETE and CORRECT result for this pass.
  Manufacturing a finding to justify the round is itself a defect - it converts a bounded check
  back into the unbounded sweep the budget exists to end.
  THE ONE ROUND THAT MAY NOT HAVE THIS SCOPE IS THE LAST, and the reason is a property of the
  METHOD, not of any one PR: EVERY ROUND ATTACKS THE NEWEST CHANGE, SO THE OLDEST DEFECT IS NEVER
  RE-ATTACKED. A review that always aims at the frontier leaves the original change permanently
  behind it, and nothing in the loop notices, because each round is doing exactly what it was
  briefed to do. Measured on #970: a bypass that let the gate approve one PR while the tool merged
  another bisected to the THIRD commit and had been live through all eight rounds, every brief
  saying "attack what the last round introduced". supervisor-mode SKILL.md RE-ATTACK THE OLDEST
  exists for it. A diff scope answers "did this fix work"; it cannot answer "is the ORIGINAL change
  still sound", and no other round asks. So the round that carries the APPROVE re-attacks the oldest
  at full artifact scope, always. Diff-scoping the terminal round is the #970 failure with a budget
  bolted on. # the ACT that finds a defect older than the review is a bisect across the branch's own
  # commits, every round - `.claude/skills/guard-change/SKILL.md` item 18

## TERMINATION [a decision someone records]
End by writing the decision where the next reader finds it - on ATLAS that is the reason field of
`scripts/claude-pr-verdict <N> approve|block "<reason>"`, which refuses anything under 20 chars.
THE FIELD IS FLATTENED BEFORE IT IS STORED. claude-pr-verdict:141 maps newline, CR and tab to
spaces, DELETES every double-quote and backslash, and squeezes runs of spaces - silently, not as a
refusal. So the record cannot be structured text; its structure has to be WORD ORDER, on one line:
  ROUNDS <n>/<budget> <CLASS>. RESIDUE: <finding -> where it went>; <finding -> where>.
  NOT_EXAMINED: <what> because <why>.
  # the literal word RESIDUE is the marker, so one grep finds every terminated review. #1019 and
  # #1022 already open with "RESIDUE accepted unfixed" (colon and comma respectively) - the word
  # is the marker, so either punctuation satisfies the check.
  # no quotes, no backslashes, no line breaks - they vanish, and a reader cannot tell they were there
  # RESIDUE: none is legal and suspicious. NOT_EXAMINED: nothing claims total coverage.
THE TERMINAL ROUND IS THE APPROVE THE MERGE CONSUMES, not the first approve. A PR can be approved
and reopened - #951 runs block, approve, block, block, approve, block, approve, and #950 carries
three approvals. Only the last one ends anything.
Accepting findings unfixed is the NORMAL ending, not an exception granted under time pressure. The
session behind this skill ended with two open deliberately (an escaped mutation, a fix shipping
without a control) and that was correct. A review whose residue is empty either found nothing or
fixed things that did not need fixing.

## SELF_CHECK [how you would know this skill is failing]
SOURCE: `~/.claude/atlas-pr-verdict.log`, append-only, one line per recorded verdict. It IS a round
log - the first draft said none existed. Rounds per PR, deduplicated by head:
  awk '{print $2, $4}' ~/.claude/atlas-pr-verdict.log | sort -u \
    | awk '{n[$1]++} END {for (p in n) printf "%d %s\n", n[p], p}' | sort -rn
FAILING when any of:
  in-budget share for a class, its trailing 20 PRs, below 80%  # measured baselines 86.4 / 90.0 / 86.2
    # 20 PROSE or TOOLING PRs is about a month at the measured rate; 20 PRODUCTION PRs is about
    # eleven days. Window by CLASS COUNT, never by calendar, or the classes fall out of step.
  a PR over budget whose verdict reasons never re-declare      # the tripwire has become decorative
  a round OPENED BY CHOICE whose findings are all IMPERFECT    # the bar was not applied to the round itself
    # NOT "a final round whose findings are all STYLE". A gate-FORCED terminal pass that finds
    # nothing, or only form, is the LICENSED normal ending (FORCED_PASS) - and "all STYLE" is
    # vacuously true of an empty finding set, so that phrasing failed the healthy case and would
    # have pushed a reviewer to manufacture a finding, which ANTI forbids two screens down.
  a verdict reason carrying no RESIDUE field                   # unrecorded termination
    # BASELINE: in the pinned snapshot above, 4 of 200 reasons mention a residue and NONE carries
    # the field. The convention is in use since 2026-09-06 - count it, never quote a frozen number:
    #   grep -c RESIDUE ~/.claude/atlas-pr-verdict.log
    # This test reads ~98% red against history, so score only verdicts recorded after adoption.
    # A LIVE-LOG FIGURE NEEDS A PIN OR A COMMAND: this line first shipped "exactly 2", measured
    # minutes before a third landed. The command cannot go stale; the number could not stay true.
NOT THE MEDIAN. Measured, the median is 1 round in EVERY class, so a median test passes forever and
cannot see #950 at 8. The tail is the observable here; the middle carries no signal at all.
HONEST LIMITS, both measured, neither fixable from this log:
  it is a FLOOR, not a census - #980 holds 4 verdict lines while its own approving reason says "Six
    review rounds", and #974 holds 6 while its own says "seven". A round reaching no verdict is invisible.
  a PR merged with NO verdict is invisible entirely - #1008 is on main with zero lines in the log.

## CONTROL [run it whenever you run the check; a green check with no control is an opinion]
CLAUDE.md TOOL_UPKEEP: a tool must report its own dullness. This one is a query, so its control is a
pair it must SEPARATE - a known overrun and a known in-budget PR, scored by the same command.
  KNOWN OVERRUN   PR#980,  PROSE, 4 rounds against a budget of 2
  KNOWN IN BUDGET PR#1010, PROSE, 1 round
  for p in PR#980 PR#1010; do printf '%s ' $p; \
    awk -v t=$p '$1 < "2026-09-06T04:00:00Z" && $2==t{print $4}' \
      ~/.claude/atlas-pr-verdict.log | sort -u | wc -l; done
  # SAME PIN as DERIVATION. Unpinned, a later verdict on either PR moves the expected pair and the
  # control fails for a reason that has nothing to do with the check being broken.
  REQUIRED: 4 and 1. Anything scoring them the SAME is broken whatever else it reports, and its
  output may not be believed until it separates them again.
MUTATION, run 2026-09-06, and it is why this section exists at all. The first draft's SELF_CHECK
counted commits pushed after the head in the review-pending marker. That file is written with a
truncating redirect, so it only ever holds the FINAL head. Scored through it: #980 = 0, #1010 = 0,
#1016 = 0, #1017 = 0. A four-round overrun and a one-round merge read IDENTICALLY, every median came
out 0, and the check passed on the exact evidence that motivated the skill.

## ANTI [HARD_STOP at end for recency]
never continue because the last round found something # a careful reader always does; that is not non-convergence
never open round 1 without a declared class and budget
never quote a round count without its UNIT # verdicts, dispatches and commits are three different numbers
never spend a round on a finding whose reader and wrong action you cannot name
never narrow a gate-forced pass that COULD approve # narrow scope only when a critical is open or
  the budget has >=2 rounds left; otherwise full scope, because terminality is known only afterwards
never diff-scope the round that carries the approve # every round attacks the newest change, so the
  oldest defect is never re-attacked unless the terminal round does it at full scope
never manufacture a finding to justify a round # the round was already counted, and an empty
  forced pass is the licensed normal ending
never read a NEW finding class as thoroughness # it is evidence round 1 was scoped wrong
never merge on an exhausted budget with a critical finding open # tripwire, not merge authority
never terminate silently # the residue is the deliverable and silence loses it
never treat accepted-unfixed findings as a failed review # that is the normal ending
never ship this check without its control # it was green through the whole failure it was written for
never freeze a LIVE-log figure into prose # pin the snapshot or ship the command; this file shipped
  "exactly 2" and a third landed ninety-three seconds before the commit
never drop a PR from the derivation for a TOOLING reason # #974 and #975 merged without a squash
  commit and were excluded as if absent; both are tail PRs, and the tail is the whole subject
