# SUPERVISOR_LESSONS

Lessons paid for once, kept where every dispatched agent can read them.

Why here and not in the supervisor's memories: subagents cannot read
`~/.claude/projects/.../memory/` — that is personal context, not repo content. So the supervisor
hand-transcribed lessons into every brief, which grew them past 700 words AND corrupted two of
them on 2026-08-07/08 (an `rm -rf` hazard that cannot occur, and an over-broad "ambiguity on
either side denies" that had to be measured and struck). A repo-local file removes the supervisor
as a lossy channel.

## GRADUATION_RULE [read this before adding anything]
A lesson's goal is to STOP being a lesson.
When one is encoded into a template, a hook, a guard test or a checklist -> DELETE it from here and
  name where it now lives. # a lesson enforced in two places drifts in one of them
This file must SHRINK as often as it grows.
  # rationale: an append-only lessons doc becomes the thing nobody reads, and then every lesson in
  # it is lost a second time
ADD nothing already enforced elsewhere -> cross-reference it under ALREADY_ENCODED instead.
CITE section anchors, never file:line. # numeric citations rotted three times in one day:
  # deploy.yml:972 (moved to 1043 by #924; 972 now names an unrelated finnhub build task),
  # supervisor-mode SKILL.md:109 (cited by git-push-guard.sh for GIT_OPS, which is not at 109),
  # autofix.sh:240
FORMAT per entry, in this order: the lesson in one line / EVIDENCE — the scar, in as many lines as
  it takes to RE-CHECK it (dates, the command, the measured outcome; ~3-5 lines in practice) /
  APPLIES where / RULE what to do instead / GRADUATES to what.
  # EVIDENCE is the one field that may run long: an unre-checkable scar is how a lesson rots into
  # folklore, and every correction to this file so far has landed on a too-thin evidence line.

## ALREADY_ENCODED [go there, never restate here]
verify a claim before relaying it, and how -> SKILL.md TIER1_CLAIM_CHECK + `templates/claim-verification.md`
two reviewers need DIFFERENT lenses -> SKILL.md TIER1_CLAIM_CHECK TWO REVIEWERS + REVIEW_FIX_LOOP LENSES
a measured number in a brief is a hypothesis -> `templates/implementation-fix.md` Notes for the supervisor
findings are claims, verify each before fixing -> `templates/implementation-fix.md` TRAJECTORY step 2
what a guard test must do to count -> `.claude/skills/intent-review/SKILL.md` GUARD_TEST_CONTRACT
what the verdict marker attests -> SKILL.md MERGE_GATE + `.claude/hooks/README.md` PR Review Verdict Gate
gate the ACT not a spelling, and a FAST oracle that answers a different question is still wrong
  (`@{push}` reports where the BRANCH would go, not what the push would WRITE) ->
  `.claude/skills/guard-change/SKILL.md` CHECKLIST item 9

## LESSONS

L1 Brief a MECHANISM as a hypothesis, never as a specification.
  EVIDENCE: 2026-08-07/08 briefs named a line number that had moved (`deploy.yml:972`), a root cause
    that was real but mislocated ("fixes an invalid-JSON fail-open" — main handles malformed stdin
    correctly), and a rule the code does not implement ("ambiguity on EITHER side denies", struck
    after measurement). Each was caught by the dispatched agent.
  APPLIES: every impl, fix and review brief — line numbers, root causes, severity rankings alike.
  RULE: label them "hypothesis, verify before acting"; an agent refuting one is the round's value,
    not a deviation. # a brief stated as fact is obeyed, and a wrong fact is obeyed into code
  GRADUATES: when `templates/implementation-fix.md` widens "every measured number" to "every
    measured number, cited line, root cause and severity".

L2 A CORRECTION is a claim. Verify it before amplifying it.
  EVIDENCE: 2026-08-07 the supervisor relayed a "50 survivors summing to 58" unit-switch defect as
    real; under the anchored command 50 survivors / 58 touches is the CORRECT pair and the
    accusation was withdrawn. 2026-08-08 a guard comment's `rm -rf "/"` hazard was relayed as a
    live mechanism; it cannot occur (the assignment happens even on failure, so the expansion is
    one empty argument that removes nothing).
  APPLIES: any report that CORRECTS an earlier claim — a correction arrives framed as the fix, so
    it skips the check the original claim would have got.
  RULE: run the correction through TIER1_CLAIM_CHECK exactly like the claim it corrects.
  GRADUATES: when `templates/claim-verification.md` names corrections as in-scope input.

L3 Analysis is not a review record. A review that never invokes the `review-pr` Skill leaves the
  merge blocked, however good the analysis was.
  EVIDENCE: `pr-review-marker.sh` is PostToolUse(Skill) and fires only on skill name
    `pr-review-toolkit:review-pr` (or `review-pr`); it alone writes `pr-review-pending-<N>`, and
    `scripts/claude-pr-verdict` exits 1 with "no review-invocation record" without that file.
    Fail-closed at both ends: no pending record -> verdict refused; no verdict -> merge blocked.
  APPLIES: every review dispatch that reaches a mergeable verdict.
  RULE: reviews that must end in a merge go through the Skill; ad-hoc review agents ADD lenses and
    can never produce a verdict. # dispatching three sharp agents and no Skill leaves zero record
  GRADUATES: when the review dispatch gets a template that names the Skill invocation as step 1.

L4 Prune the finished agent's worktree BEFORE dispatching the next agent onto that branch.
  EVIDENCE: on 2026-08-07/08 the fix-round branches of the guard-hardening chain were repeatedly
    still held by the worktree of the agent that had finished with them; an agent told to work
    `{branch}` finds it held elsewhere and either stalls or branches off a stale copy.
  APPLIES: every FIX ROUND and every re-dispatch onto a branch another agent has finished with.
  RULE: the supervisor prunes; the agent never deletes another agent's worktree — it takes the
    branch with `git checkout --ignore-other-worktrees`. # `templates/implementation-fix.md`
    WHERE TO WORK states the agent half, which only works if the supervisor does this half
  GRADUATES: when a pre-dispatch worktree check lands in the dispatch templates or a hook.

L5 One merge act per Bash invocation. Never chain two.
  EVIDENCE: measured 2026-08-07 with #921 approved and #923 carrying no verdict at all —
    `gh pr merge 921 ... && gh pr merge 923 ...` ALLOWED, merging the unverdicted PR under the
    other's approval; five further shapes did the same — six rows in all, recorded in the guard's
    own M1 comment block (git-push-guard.sh, "THE PR IDENTITY FOR A MERGE MUST COME FROM THE MERGE
    ITSELF"), including the two sharpest, where a `/pulls/<other>/merge` anywhere in the string —
    even inside a `--subject` — hijacked the verdict lookup. The gate now denies any command
    carrying more than one merge act, because one decision slot cannot carry two verdicts.
  APPLIES: every merge; also `gh api .../pulls/<N>/merge` and the `curl` form.
  RULE: merge, end the command, then merge the next. # chaining is denied now, so chaining is
    also just a wasted round
  GRADUATES: never — this one is enforced by the gate; it stays as the explanation of the denial.

L6 Prose that quotes a gated command form TRIPS that gate. Pass long text by path.
  EVIDENCE: the merge gate counts "one merge plus TEXT naming another" as two acts and refuses; a
    commit message merely QUOTING a push form used to produce an authoritative allow for a real
    push to main. Reproduced live while writing this file: a `command grep` whose PATTERN contained
    a merge form was denied by the merge gate.
  APPLIES: commit messages, PR comments, review briefs, dispatch prompts, and grep patterns.
  RULE: `git commit -F <file>`, `gh pr comment <N> --body-file <file>`, brief text in a file passed
    by path. # the gate reads the command string; it cannot know your text is only about a merge
  GRADUATES: never — the gate's own deny message names the remedy; this records why it is right.

L7 Squash merge makes "is this commit on a remote ref" answer NO for work that already landed.
  EVIDENCE: 2026-08-07 a reachability safety check flagged worktrees as holding unpushed work that
    had in fact already merged. Reproduce it any time: #931 is MERGED, and
    `git merge-base --is-ancestor <its headRefOid> origin/main` still returns 1. A squash merge
    rewrites the commits, so the originals are not ancestors and `--contains` finds nothing hours
    after the work landed. # the ORIGINAL count of flagged worktrees was never corroborated and is
    # deliberately not restated — the mechanism is the lesson, and it needs no headcount
  APPLIES: before pruning a worktree or branch, and any "has this landed" check.
  RULE: ask the PR's state (merged / mergeCommit) or compare CONTENT; commit reachability cannot
    distinguish "merged by squash" from "genuinely unpushed", and it fails toward "unpushed".
  GRADUATES: when the prune step gets a scripted check that reads PR state instead of reachability.

## ANTI [HARD_STOP @end for recency]
never state a brief's mechanism, line number or severity as settled fact
never amplify a correction you have not verified
never expect a merge verdict from a review that did not invoke the Skill
never dispatch onto a branch still held by a finished agent's worktree
never chain two merge acts in one command
never put a quoted push or merge form in a commit message, comment or grep pattern
never read commit reachability as proof that work did not land
never add an entry here that a template, hook or checklist already enforces
