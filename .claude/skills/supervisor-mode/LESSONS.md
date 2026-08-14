# SUPERVISOR_LESSONS

Lessons paid for once, kept where every dispatched agent can read them.
Here and not in supervisor memory because subagents cannot read `~/.claude/projects/.../memory/` — so the supervisor
hand-transcribed them into briefs, which grew past 700 words and corrupted two of them on 2026-08-07/08.

## GRADUATION_RULE [read before adding anything]
A lesson's goal is to STOP being a lesson. Encoded into a template, hook, guard test or checklist -> DELETE it here and
  name where it now lives # a lesson enforced in two places drifts in one of them
This file must SHRINK as often as it grows # an append-only lessons doc becomes the thing nobody reads, and then every
  lesson in it is lost a second time
ADD nothing already enforced elsewhere -> cross-reference under ALREADY_ENCODED instead.
CITE section anchors, never file:line # numeric citations rotted three times in one day
WRITE_TRIGGER: a defect RECURS -> entry here. Once is an incident; twice is a lesson. A first occurrence goes to
  STATE.md or nowhere — that is what keeps this file short enough to be read.
  WHICH STORE [never both]: can a DISPATCHED AGENT act on it? -> here, and the memory entry becomes a POINTER.
    Is it the supervisor's own judgement or recall? -> memory only.
  Write at the VERDICT, not the retro: recording a BLOCK for a class you have blocked before IS the second occurrence.
RULE_MUST_BE_CHEAP: the RULE names the CHEAPEST SUFFICIENT action, not the most correct one # measured 2026-08-13:
  L2 existed, was exactly on point, and did NOT fire — its remedy was "dispatch an agent", disproportionate against a
  one-line refutation, so it was skipped. If the only sufficient remedy IS expensive, say so and expect it to be
  skipped under pressure — an unaffordable rule is a rule that does not exist.
FORMAT per entry: the lesson in one line / EVIDENCE re-checkable in ~3-5 lines (date, command, measured outcome) /
  APPLIES where / RULE what to do instead / GRADUATES to what.

## ALREADY_ENCODED [go there, never restate here]
verify a claim before relaying it -> SKILL.md TIER1_CLAIM_CHECK + `templates/claim-verification.md`
two reviewers need DIFFERENT lenses -> SKILL.md TIER1_CLAIM_CHECK TWO REVIEWERS + REVIEW_FIX_LOOP LENSES
a measured number in a brief is a hypothesis -> `templates/implementation-fix.md` Notes for the supervisor
findings are claims, verify each before fixing -> `templates/implementation-fix.md` TRAJECTORY step 2
what a guard test must do to count -> `.claude/skills/intent-review/SKILL.md` GUARD_TEST_CONTRACT
what the verdict marker attests -> SKILL.md MERGE_GATE + `.claude/hooks/README.md` PR Review Verdict Gate
gate the ACT not a spelling; a FAST oracle answering a different question is still wrong (`@{push}` reports where the
  BRANCH would go, not what the push would WRITE) -> `.claude/skills/guard-change/SKILL.md` CHECKLIST item 9

## LESSONS

L1 Brief a MECHANISM as a hypothesis, never as a specification.
  EVIDENCE: 2026-08-07/08 briefs named a line number that had moved (`deploy.yml:972`), a root cause that was real but
    mislocated, and a rule the code does not implement ("ambiguity on EITHER side denies", struck after measurement).
    Each was caught by the dispatched agent.
  APPLIES: every impl, fix and review brief — line numbers, root causes, severity rankings alike.
  RULE: label them "hypothesis, verify before acting"; an agent refuting one is the round's value, not a deviation.
    # a brief stated as fact is obeyed, and a wrong fact is obeyed into code
  GRADUATES: when `templates/implementation-fix.md` widens "every measured number" to "every measured number, cited
    line, root cause and severity".

L2 A CORRECTION is a claim. Verify it before amplifying it.
  EVIDENCE: 2026-08-13, and this entry EXISTED and did not fire. A reviewer called "the console bounds the token figure
    from above" false, testing it against the $1.76-2.86/day BAND (a value inside an interval bounds it from neither
    side) when the sentence's subject was the token LEG (console = token cost + an unseen fee >= 0, so console >= token
    cost, true by construction). Every number in the refutation was right; only the SUBJECT differed. Relayed as
    CRITICAL, it deleted a true sentence from four files including a D-entry's INTENT. Two rounds to detect and undo.
    Earlier: a "50 survivors summing to 58" defect (50/58 is the CORRECT pair); an `rm -rf "/"` hazard that cannot
    occur (the expansion is one empty argument).
  APPLIES: any report that CORRECTS an earlier claim — a correction arrives framed as the fix, so it skips the check
    the original claim would have got.
  RULE: run it through TIER1_CLAIM_CHECK like the claim it corrects — AND when it says "X is false", restate X's
    proposition in your own words and NAME ITS SUBJECT first. One sentence of work. Arithmetic-flavoured refutations
    are the dangerous ones: every number can be right while the quantity they are about is the wrong one.
  ASYMMETRY [why this outranks an ordinary claim check]: acting on a false ordinary claim ADDS something wrong and the
    next review catches it. Acting on a false refutation DELETES something right — and deletion is the one edit later
    review CANNOT audit, because reviewers read what is in the tree, not what used to be. Raise the bar highest when
    the refutation licenses a delete. # three agents disagreed across three rounds over `0.81%`; it was right from the
    start. Chained refutations converge on noise unless someone RE-DERIVES the quantity instead of adjudicating the
    previous opinion.
  GRADUATES: when `templates/claim-verification.md` names corrections as in-scope AND carries the restate-the-subject step.

L3 Analysis is not a review record. A review that never invokes the `review-pr` Skill leaves the merge blocked,
  however good the analysis was.
  EVIDENCE: `pr-review-marker.sh` is PostToolUse(Skill), fires only on `pr-review-toolkit:review-pr` (or `review-pr`),
    and alone writes `pr-review-pending-<N>`; `scripts/claude-pr-verdict` exits 1 without it. Fail-closed at both ends.
  APPLIES: every review dispatch that reaches a mergeable verdict.
  RULE: reviews that must end in a merge go through the Skill; ad-hoc review agents ADD lenses and can never produce a
    verdict. # dispatching three sharp agents and no Skill leaves zero record
  GRADUATES: when the review dispatch gets a template naming the Skill invocation as step 1.

L4 Prune the finished agent's worktree BEFORE dispatching the next agent onto that branch.
  EVIDENCE: 2026-08-07/08 fix-round branches were repeatedly still held by the worktree of the agent that had finished
    with them — the next agent stalls or branches off a stale copy. 2026-08-13 it became a DESTRUCTION hazard: FOUR
    abandoned worktrees held a live branch with the whole change STAGED AS DELETIONS (1,301 on one, 357 on another).
    A bare `git commit` in any of them rewrites the branch and silently reverts merged work. HEAD follows the ref so it
    LOOKS current; the stale thing is the index. My own cleanup missed a second instance after reporting it closed.
  APPLIES: every fix round, every re-dispatch onto a finished branch, and every dispatch that ENDS — the hazard is what
    it leaves behind.
  RULE: the supervisor prunes; the agent takes the branch with `git checkout --ignore-other-worktrees` and never
    deletes another agent's worktree. That flag is what permits several worktrees on ONE branch, so the instruction
    that fixes the dispatch is what creates the hazard — sweep for STAGED CHANGES across ALL worktrees, not just for
    held branches, and require every dispatch to release its branch leaving nothing staged.
  GRADUATES: when a pre-dispatch check lands in the templates or a hook AND it checks `diff --cached`, not merely which
    branch is held.

L5 One merge act per Bash invocation. Never chain two.
  EVIDENCE: 2026-08-07 with #921 approved and #923 carrying no verdict, chaining both merges in one command was
    ALLOWED, merging the unverdicted PR under the other's approval. Six shapes in all, recorded in the guard's own M1
    comment block (git-push-guard.sh) — sharpest being a `/pulls/<other>/merge` anywhere in the string, even inside a
    `--subject`, hijacking the verdict lookup. The gate now denies any command carrying more than one merge act.
  APPLIES: every merge; also the `gh api .../pulls/<N>/merge` and `curl` forms.
  RULE: merge, end the command, then merge the next. # chaining is denied now, so it is also a wasted round
  GRADUATES: never — enforced by the gate; this stays as the explanation of the denial.

L6 Prose that quotes a gated command form TRIPS that gate. Pass long text by path.
  EVIDENCE: the merge gate counts "one merge plus TEXT naming another" as two acts; a commit message merely QUOTING a
    push form used to produce an authoritative allow for a real push to main. Reproduced while writing this file: a
    `command grep` whose PATTERN contained a merge form was denied by the merge gate.
  APPLIES: commit messages, PR comments, review briefs, dispatch prompts, grep patterns.
  RULE: `git commit -F <file>`, `gh pr comment <N> --body-file <file>`, brief text passed by path. # the gate reads the
    command string; it cannot know your text is only ABOUT a merge
  GRADUATES: never — the gate's deny message names the remedy; this records why it is right.

L7 Squash merge makes "is this commit on a remote ref" answer NO for work that already landed.
  EVIDENCE: 2026-08-07 a reachability check flagged worktrees as holding unpushed work that had already merged.
    Reproducible any time: #931 is MERGED and `git merge-base --is-ancestor <its headRefOid> origin/main` returns 1.
  APPLIES: before pruning a worktree or branch, and any "has this landed" check.
  RULE: ask the PR's state (merged / mergeCommit) or compare CONTENT. Commit reachability cannot distinguish "merged by
    squash" from "genuinely unpushed", and it fails toward "unpushed".
  GRADUATES: when the prune step gets a scripted check reading PR state instead of reachability.

L8 Repair citations LAST, and treat every fact-shaped claim as a citation.
  EVIDENCE: 2026-08-12/13, one PR chain. Round 2 repaired ONE line citation and broke FOURTEEN (its own edits shifted
    `server.py`), in a commit whose message NAMED that class. Round 4 re-ran its own sweep and still shipped two,
    because the sweep resolved `file:line` while the new breakage was in COUNTS and ROSTERS ("the four tests beside it"
    after the list grew to 6; a doc saying a card holds D-1..D-4 after D-5/D-6 landed, which would make the next agent
    mint a colliding D-6). Round 5's edits shifted 27 more, including D-6's own GUARD.
  APPLIES: any edit to a file a D-entry, card or CLAUDE.md cites INTO — which is most service code.
  RULE: content edits first, citations last, then re-run over your OWN diff. Enumerate the KINDS of fact-shaped claim
    before sweeping (line numbers, counts, rosters, "N tests", "the three files"). Best of all DELETE the derived
    spelling: a bare count beside an enumerated list is one fact written twice, which IS the defect.
  A GREEN SWEEP IS NOT PROOF: `scripts/verify-citations.py` is content-blind by design — it flags BLANK and
    out-of-range only, so a citation drifted onto a comment, a brace or an unrelated task reads GREEN, and ranges are
    blank-checked at the START only. Every instance found was found by READING the entry. Two were wrong the DAY THEY
    WERE WRITTEN, off by 4 lines — citations ship broken, they do not only rot.
  GRADUATES: when the tool resolves a citation to the NAMED CONSTRUCT rather than a live line, and a hook runs it on
    changed files. Until then this is judgement, not tooling.

L9 A fix round fixes exactly what you NAME and adds what you did NOT ask for. Brief the class, ban the framing.
  EVIDENCE: 2026-08-13, two PRs, seven rounds. NAMING: a reviewer named 3 stale citations; the round fixed 3 and left 3.
    Named 3 more; found 6 — the two worst in neither list, incl. an INTERFACE CONTRACT declaring "does not throw" for a
    client that throws and whose D-entry guard fires ONLY because it throws. The round that worked was told "do NOT fix
    the three I name — search for the PROPOSITION and report the COUNT before the fixes."
    FRAMING: successive rounds shipped 7, then 5, then 3, then 1 NEW false claims, every one from explanatory prose the
    round volunteered while fixing something else. The round that added zero was told "correct these sentences and add
    NOTHING."
  APPLIES: every fix round, and every sweep for a defect with more than one instance.
  RULE: brief the CLASS with a searchable predicate, never a list of sites; require the count BEFORE the fixes; treat
    "more than you named" as the useful result. Forbid volunteered framing — the ONE justified addition is a clause
    closing a contradiction your own edit opens. A list in the brief is the ceiling on what comes back.
  ALSO: "report it, do not fix it" is right for JUDGEMENT and wrong for a CHEAP FACT. A round surfaced "if any inactive
    row were a series this count is wrong" and left it open; one SELECT settled it and CHANGED THE ANSWER (16 -> 17).
    If one query closes it, close it and report.
  GRADUATES: when `templates/implementation-fix.md` carries a "state the class, report the count first, add nothing" stanza.

L10 Run the build AFTER the final commit. The marker keys to the TREE.
  EVIDENCE: 2026-08-13, twice in one evening. An agent edited, compiled green, then committed the card and its INTENT
    comments — comments still change `HEAD^{tree}`, so the marker keyed to the PRE-commit tree and the push gate
    refused a tree nobody had built. Content was identical each time; only the tree hash moved.
  APPLIES: every dispatch that both compiles and commits — most of them.
  RULE: final commit, THEN `compile.sh`, then report the attested tree hash and check it equals `HEAD^{tree}`.
    # one line in the brief; it removes a whole re-verification round
  GRADUATES: when the dispatch templates carry the ordering and the report asks for the tree hash.

## ANTI [HARD_STOP @end for recency]
never state a brief's mechanism, line number or severity as settled fact
never amplify a correction you have not verified
never relay "X is false" without restating X's proposition and naming its subject
never repair citations before your last content edit, and never trust a green sweep as proof
never hand a fix round a list of sites — brief the class and require the count before the fixes
never let a round volunteer framing it did not measure, and never leave a cheap fact unqueried
never compile before the final commit — the marker keys to the tree, not the content
never expect a merge verdict from a review that did not invoke the Skill
never dispatch onto a branch still held by a finished agent's worktree
never leave a worktree behind with a staged index — it is one commit from reverting merged work
never chain two merge acts in one command
never put a quoted push or merge form in a commit message, comment or grep pattern
never read commit reachability as proof that work did not land
never add an entry here that a template, hook or checklist already enforces
