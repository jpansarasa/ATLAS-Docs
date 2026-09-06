# SUPERVISOR_LESSONS

Generalizations paid for once, kept where every DISPATCHED AGENT can read them # subagents cannot read supervisor
memory, so a lesson filed there gets hand-transcribed into briefs, and two were corrupted that way on 2026-08-07/08.
A LESSON IS A GENERALIZATION YOU CAN APPLY SOMEWHERE ELSE. One that names a single incident and dies when that
incident is fixed is a commit message: it belongs in git, or in docs/BACKLOG.md carrying its measurement.

## GRADUATION_RULE [read before adding anything]
A lesson's goal is to STOP being a lesson: once encoded in a template, skill, hook or checklist -> DELETE it here and
  leave a pointer under ALREADY_ENCODED # a lesson enforced in two places drifts in one of them
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
an alert that fires by ACCIDENT is not coverage -> CLAUDE.md OBSERVABILITY; the six ungauged D-18 series are docs/BACKLOG.md # was L17, whose enumerate-from-the-DATA-side half is NOT there and survives in L11 below
a command reporting its own limit, warning or truncation has ANSWERED you -> `templates/claim-verification.md` step 8 # was L18
verify a claim before relaying it, and give two reviewers DIFFERENT lenses -> SKILL.md TIER1_CLAIM_CHECK + REVIEW_FIX_LOOP LENSES + `templates/claim-verification.md`
what a guard test must do to count -> `.claude/skills/intent-review/SKILL.md` GUARD_TEST_CONTRACT
what the verdict marker attests -> SKILL.md MERGE_GATE + `.claude/hooks/README.md` PR Review Verdict Gate
find a recorded DECISION about a PR (BLOCKED, do-not-merge, superseded) BEFORE reviewing its code -> SKILL.md MERGE_GATE SEQUENCE step 0
analysis is not a review record -> SKILL.md MERGE_GATE, fail-closed by `.claude/hooks/pr-review-marker.sh` and `scripts/claude-pr-verdict` # was L3
one merge act per Bash invocation -> `.claude/hooks/git-push-guard.sh` denies any command carrying more than one + SKILL.md RED_FLAGS # was L5
prose quoting a gated push or merge form trips that gate; pass long text by path -> `.claude/hooks/README.md` Accepted cost # was L6; only the merge denies and the two-pushes deny name the remedy
squash merge makes commit reachability answer NO for work that landed -> SKILL.md RED_FLAGS; ask the PR's state or compare CONTENT # was L7

## LESSONS

L8 A PROXY DOES NOT MOVE WITH THE THING IT STANDS FOR, and when it stops tracking it fails in the direction that
  reads as SUCCESS. A reference is a proxy for the content at a position; an mtime is a proxy for freshness.
  EVIDENCE: #1002, 2026-09-04 -- 8 lines above a card's DECISIONS block moved D-18 from `:85` to `:93`. Two docs
    cite `:84`: BLANK on main and FLAGGED, landing on D-13 on head and QUIET, so cannot-land FELL BY ONE as the
    citation went wrong. #1004, 2026-09-05 -- a `mv` restore put content below the built mutant's DLL mtime, so
    every later run scored the FIRST mutant's binary.
  RULE: verify against the THING, never the proxy -- the CONTENT a reference should land on, the ARTIFACT a build
    should have produced. Content edits first and references LAST, then sweep your OWN diff and compare the
    unresolved SET against a pristine baseline (never a count, never an rc: both sides are rc 1 here). `touch` after
    BOTH a mutate and a restore, then require one mutant SHOWN to have compiled -- never the test result, which is
    the thing under suspicion.
  GRADUATES: TWO halves, both open, so graduating one is not graduating the entry -- split it then. (a) when
    `scripts/verify-citations.py` resolves to the NAMED CONSTRUCT rather than a live line AND a hook runs it on
    changed files (the `D-n` class shipped in #1002, no hook is wired). (b) when a SHARED mutation helper asserts
    the built artifact CHANGED, instead of a harness hand-rolled per round.
  GRADUATE_CHECK: H=$(find .claude/hooks -type f -perm -u+x -exec grep -l '^[^#]*verify-citations' {} +); grep -q WRONG-D-ENTRY scripts/verify-citations.py && [ -n "$H" ] && printf '%s\n' "$H" | sed 's|.*/||' | grep -qFf - .claude/settings.json && test -x scripts/mutate-verify.sh && grep -v '^[[:space:]]*#' scripts/mutate-verify.sh | grep -qwE 'cmp|strings|sha256[a-z]*'

L11 An instrument that dies silently scores its silence as a PASS -- and coverage can never be enumerated from the
  instruments that EXIST, because the missing one is precisely what a census of them cannot show.
  EVIDENCE: an alert rule oscillating pending -> inactive through a real resolution rate of ~3%: 24 pending cycles, 0
    fires in 24h, in a file whose promtool suite asserted only silence. GeminiResolverNotResolving fired only because
    rejected calls consumed cap slots -- undesigned, and switched off silently by fixing the accounting. A
    shuffled-gold control averaged a 4x breach into a pass, invisible to its own mutation test: at n=1 the aggregate
    IS the unit (#1016).
  RULE: adding or changing an alert rule -> ONE `promql_expr_test` asserting `alertstate="firing"` on input shaped
    like the real traffic, BURSTS WITH GAPS for this fleet; a test showing only silence has pinned nothing. Mutate
    any control at THE SCALE IT SHIPS AT -- poison one unit, pad with clean ones to the documented usage AND one past
    it, make the complaint NAME the offending unit, and read an unstated n as n=1. Enumerate coverage from the DATA
    side: grep the config that would HAVE to mention the thing you care about.
  GRADUATES: BOTH -- the alert-rules CI step fails any rule file holding an `alert:` with no positive assertion
    naming it, AND intent-review GUARD_TEST_CONTRACT requires a scale-matched mutation. Neither artifact can hold the
    rule today: no coverage checker exists beside `check-matchers.py`, the contract is silent on aggregates. Keep the
    `consumed cap` clause above -- two hook fixtures cite it by that phrase as their provenance.
  GRADUATE_CHECK: S=.claude/skills/intent-review/SKILL.md; grep -v '^[[:space:]]*#' deployment/tests/alerts/run.sh | grep -qE 'check-[a-z-]*coverage' && grep -qiE 'scale-matched|pad(ded)? with clean' "$S" && grep -qiE 'names? the (offending|breaching) (unit|run|record)' "$S"

L15 A self-authored negative -- "nothing loosened", "no regressions", "no new findings" -- is scoped to its author's
  imagination, and fixing the BASELINE is necessary but not sufficient.
  EVIDENCE: #935, then the salvage round built specifically to avoid #935's error, which fixed the baseline, measured
    zero against MAIN and was still wrong (2026-08-16). Every zero was honest and every one was falsified by the
    next, bigger corpus: 83 rows -> 14 loosened shapes, 181 -> 60, 342 -> 60, 968 -> 88. Not converging.
  RULE: the corpus must come from a DIFFERENT MIND than the fix -- say it in the brief, literally: "build your own
    matrix; do not replay theirs". Require the corpus SIZE and the sentence "this number is only as good as this
    corpus" in the report. The guard-specific instance (enumerate the SPELLINGS of every construct a rule names) is
    `guard-change` item 1; this entry is the general form, which no artifact holds.
  GRADUATES: when an adversarial corpus is GENERATED from the guard's own rule table rather than written by hand, so
    the spellings come from the code and not from whoever is feeling thorough today. The check keys on an artifact's
    NAME because that tool does not exist and has no other observable; name it `*corpus*` when you build it.
  GRADUATE_CHECK: find .claude/hooks/test -type f -perm -u+x -iname '*corpus*' | grep -q .

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
  not a green sweep, not a FALLING cannot-land count, not a test result on a build you did not prove rebuilt [L8]
never accept a rule, alert or control proven only by silence, only at n=1, or only from the alert list [L11]
never accept a self-authored "nothing loosened", or a corpus built by the fix's own author [L15]
never add an entry that a template, skill, hook or checklist already enforces
never add an entry without a GRADUATES clause NAMING THE ARTIFACT that will hold it and a GRADUATE_CHECK -- `none --
  judgement` is an answer, an absent line is not, and `scripts/new-epic.sh` refuses the next epic reset over either
  that or a check that now PASSES
