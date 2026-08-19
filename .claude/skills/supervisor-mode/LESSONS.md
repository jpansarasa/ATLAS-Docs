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
WRITE_TRIGGER: a defect RECURS -> entry here. Once is an incident; twice is a lesson.
  A first occurrence goes to **docs/BACKLOG.md**, NOT to STATE.md # STATE.md is wiped at every epic boundary, so an
  incident parked there makes the SECOND occurrence unrecognisable and the lesson is never earned. BACKLOG is what
  carries an incident ACROSS epics until it recurs; keeping first occurrences out of here is what keeps this file
  short enough to be read.
  WHICH STORE [never both]: can a DISPATCHED AGENT act on it? -> here, and the memory entry becomes a POINTER.
    Is it the supervisor's own judgement or recall? -> memory only.
  Write at the VERDICT, not the retro: recording a BLOCK for a class you have blocked before IS the second occurrence.
RULE_MUST_BE_CHEAP: the RULE names the CHEAPEST SUFFICIENT action, not the most correct one # measured 2026-08-13:
  L2 existed, was exactly on point, and did NOT fire — its remedy was "dispatch an agent", disproportionate against a
  one-line refutation, so it was skipped. If the only sufficient remedy IS expensive, say so and expect it to be
  skipped under pressure — an unaffordable rule is a rule that does not exist.
FORMAT per entry: the lesson in one line / OCCURRENCES the instances that earned it (two minimum — this is what
  makes WRITE_TRIGGER auditable; L1-L11 carry theirs inside EVIDENCE) / EVIDENCE re-checkable in ~3-5 lines (date,
  command, measured outcome) / APPLIES where / RULE what to do instead / GRADUATES to what.
  GRADUATES IS NOT OPTIONAL and must be CHECKABLE — name the observable state that retires the entry, so a later
  session can test it. An entry with no exit condition is permanent by construction, which is how this file stops
  shrinking. # measured 2026-08-17: three of five new entries had none

## ALREADY_ENCODED [go there, never restate here]
Every line NAMES the mechanism that enforces it now, so removing that mechanism removes a visible pointer
  instead of silently losing the lesson # a deleted entry with no pointer cannot be told from one never earned
verify a claim before relaying it -> SKILL.md TIER1_CLAIM_CHECK + `templates/claim-verification.md`
two reviewers need DIFFERENT lenses -> SKILL.md TIER1_CLAIM_CHECK TWO REVIEWERS + REVIEW_FIX_LOOP LENSES
a measured number in a brief is a hypothesis -> `templates/implementation-fix.md` Notes for the supervisor
findings are claims, verify each before fixing -> `templates/implementation-fix.md` TRAJECTORY step 2
what a guard test must do to count -> `.claude/skills/intent-review/SKILL.md` GUARD_TEST_CONTRACT
what the verdict marker attests -> SKILL.md MERGE_GATE + `.claude/hooks/README.md` PR Review Verdict Gate
gate the ACT not a spelling; a FAST oracle answering a different question is still wrong (`@{push}` reports where the
  BRANCH would go, not what the push would WRITE) -> `.claude/skills/guard-change/SKILL.md` CHECKLIST item 9
find a recorded DECISION about a PR (BLOCKED, do-not-merge, superseded) BEFORE reviewing its code -> SKILL.md
  MERGE_GATE SEQUENCE, step 0 # the incident that earned it is in docs/BACKLOG.md, filed under #935
analysis is not a review record; the Skill invocation is what makes a verdict possible -> SKILL.md MERGE_GATE,
  fail-closed at both ends by `.claude/hooks/pr-review-marker.sh` (sole writer of the pending record) and
  `scripts/claude-pr-verdict`, which exits 1 without it # was L3
one merge act per Bash invocation -> `.claude/hooks/git-push-guard.sh` denies any command carrying more than one,
  and SKILL.md RED_FLAGS carries the stop-line # was L5
prose quoting a gated push or merge form trips that gate; pass long text by path -> `.claude/hooks/README.md`
  Accepted cost explains it. Whether the DENY says so depends which one fires: both merge denies name the remedy
  (`git commit -F`, `gh pr comment --body-file`), and so does the two-pushes-merged-into-one-span push deny; the
  three a quoted push actually reached in probing do not — `push origin main` answers "use a feature branch", a
  name main lacks answers "the branch does not exist … check 'git branch --list'", and a second `git push`
  anywhere in the line answers "Run the pushes as separate commands" # was L6; round 1 said the denies name it and
  round 2 said they do not, and the narrow form is the only true one
squash merge makes commit reachability answer NO for work that landed -> SKILL.md RED_FLAGS; ask the PR's state
  or compare CONTENT # was L7

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

L11 An alert rule is unproven until a test asserts it FIRES. Green promtool says only that it did not crash.
  EVIDENCE: 2026-08-14, second occurrence of the same class. `SentinelLowResolutionRate` stepped over 6h at 5m:
    9 of 18 samples NaN (empty `rate(...[5m])` denominator across the idle gaps between bursts), the other 9 exactly
    `0` — it oscillates pending -> inactive and never holds `for: 15m`. 24 pending cycles, 0 fires in 24h, through a
    real resolution rate of ~3%. FIRST occurrence already in docs/BACKLOG.md: `SecMasterDiscoveryTimeoutsElevated`,
    whose ratio a double-count pins at exactly 0.5 against a `> 0.5` test. Both rules live in files that HAVE
    promtool suites; neither had a positive assertion naming it.
  APPLIES: every rule in `deployment/artifacts/monitoring/alerts/` — and hardest to the ones written to catch a
    condition that is not currently happening, because nothing on the box will ever contradict them.
  RULE: adding or changing a rule -> ONE `promql_expr_test` asserting `alertstate="firing"` on an input series shaped
    like the real traffic, which for this fleet means BURSTS WITH GAPS, not a steady rate. A rule whose test only
    shows it staying silent has pinned nothing: silence is also what a rule that can never fire produces.
    # `deployment/tests/alerts/run.sh` already executes these and check-assertion-counts.py already ratchets the
    # count, so the cost is one fixture, not a harness
  WHY THE CHEAP VERSION IS ENOUGH: both defects here are visible the moment you feed the rule a gappy series — no
    soak, no prod data, no judgement about thresholds. The expensive version (measure the live distribution first) is
    what nobody will do under pressure.
  GRADUATES: when the alert-rules CI step fails any rule file containing an `alert:` with no positive assertion
    naming it — a coverage checker beside `check-matchers.py`, which already resolves alertname literals and so
    already holds both halves of the join.

L13 An instrument that dies silently scores its silence as a PASS.
  OCCURRENCES: five in one session, 2026-08-15 — and the `awk` one fired TWICE, the second time while writing the
    comment that warns about it.
  EVIDENCE: five distinct harness failures, EVERY one biased toward false success. A relative guard path under a
    fixture `cd` -> bash rc 127 -> every row scored invalid, reading as "the guard is broken" rather than "my probe
    is broken". An apostrophe inside a single-quoted `awk` program closed the program, so the guard emitted NO
    decision — which the permission layer reads as ALLOW. A suite that prints only a `PASS=<n>` summary scored 0
    under a per-row grep: a 2,138-assertion undercount that read as green. `$(git ls-files '*.md')` unquoted
    word-split on a path with spaces and aborted the tool two-thirds through with no summary, and the truncated
    sweep was read as a complete one. Parallel agents overwrote each other's harness in the shared scratchpad,
    producing an all-zero mutation result that read as "no coverage".
  APPLIES: every measurement brief, and every harness a dispatch builds for one job and throws away.
  RULE: absolute tool paths; abort on rc 127; syntax-check the thing under test before measuring; blob-check each
    copy against its git object; a uniquely-named private scratch dir; and make the harness ABORT BY NAME on a
    missing anchor rather than scoring the row. # three of the five were caught only because a harness aborted
    instead of scoring — a tool that fails toward SUCCESS cannot be caught by reading its output
  GRADUATES: when `templates/recon-measurement.md` carries those checks as a TRAJECTORY step, so a measurement brief
    cannot omit them — grep that file for `rc 127`, which returns nothing today (the file exists, the checks do
    not, so an empty result is not-yet-graduated rather than wrong-file).

L14 A guard that matches TEXT instead of ACTS inherits the whole grammar — and every round attacks the NEWEST
  mechanism, so the oldest defect is never re-attacked.
  OCCURRENCES: `git-push-guard.sh` and `ansible-gate-guard.sh`, thirteen review rounds between them, 2026-08-15/17.
  EVIDENCE: every round found the previous round's fix broken by a piece of bash grammar it did not model — span cut
    at metacharacters, then act-bounding, then the nesting-depth counter, then escape handling; each fix correct and
    each creating the next surface. The tell runs BOTH ways: these guards refuse a DESCRIPTION of an act (a filename
    quoted in a verdict reason; `2>&1` and `Rule 1` read as PR numbers) while permitting the act under a spelling
    they do not parse. A bypass that let the gate approve one PR while the tool merged another was BISECTED to the
    third commit and had been live through all eight rounds — every brief said "attack what the last round
    introduced", which is correct and which left the original change permanently behind the frontier. Eleven rounds
    measured only whether the guard refuses enough; the two that also measured whether it still PERMITS ordinary
    work caught the swings in each direction, one of which would have locked the session out of its own repo.
  APPLIES: every round on a text-matching guard, and every review of one.
  RULE: bisect a confirmed bypass across the BRANCH'S OWN COMMITS, every time — it is cheap, it names the commit,
    and it is the only thing that finds a defect older than the review that keeps missing it. Measure over-denial
    EVERY round, not only bypasses. When a round's fix is "handle one more grammar construct", say so out loud and
    price it: that is an approximation converging on a reimplementation of bash. # the act-not-a-spelling half of
    this lesson is ALREADY_ENCODED above — go there, do not restate it here
  GRADUATES: when `guard-change/SKILL.md` CHECKLIST item 18 names the ACT ("bisect every confirmed bypass across the
    branch's own commits") and item 16 requires the over-denial count REPORTED beside the bypass count — grep that
    file for `bisect`, which returns NOTHING today: item 18 describes the check ("Check WHEN a surviving defect was
    introduced") without ever naming the act, so an empty grep means not-yet-graduated, not wrong-file.

L15 A "nothing loosened" claim is scoped to its author's imagination. Fixing the BASELINE is necessary and not
  sufficient.
  OCCURRENCES: #935's seven rounds, then the salvage round built specifically to avoid #935's error — which fixed
    the baseline, measured zero against MAIN, and was still wrong (2026-08-16).
  EVIDENCE: every zero was measured honestly and every one was falsified by the NEXT, bigger corpus. The series is
    the whole argument: 83 rows -> 14 loosened shapes · 181 rows -> those 14 plus 46 more · 342 rows -> 60 · 968
    rows -> 88, 52 of them landing in `/opt` or `/etc`. **Each ~3x corpus finds ~2-6x more. NOT converging.** #935's
    own seven rounds each measured against its PREVIOUS HEAD rather than main, were right every time, and the branch
    drifted anyway. Corpus sizes, per-round detail and the drift-against-main figures: `docs/BACKLOG.md`, the #935
    entries.
  THE TELL, and every miss that week fits it: **the shape that leaks is a SPELLING VARIANT of one the fixtures
    already cover.** Object-store rows spelled without `--`, so the `--` spelling leaked. The only destination-flag
    row unbundled, so `-rt` leaked. The tab-stripping heredoc rows without whitespace, so `<<- EOF` leaked. The
    heredoc row unterminated, so the terminated form — the one bash actually runs — leaked. A fixture set reads as
    coverage of a CONSTRUCT while covering one SPELLING.
  APPLIES: any self-authored negative offered as a merge signal ("nothing loosened", "no regressions", "no new
    findings"), and any corpus built by whoever wrote the fix — corpus author and fix author being the SAME MIND is
    the residual flaw once the baseline is right, and the holes are the shapes that mind was not thinking about.
  RULE: the corpus must come from a DIFFERENT mind than the fix — say so in the brief, literally: "build your own
    matrix; do not replay theirs". For each construct a rule names, enumerate its SPELLINGS and test each: separator
    present/absent, flag bundled/glued/spaced, delimiter quoted/unquoted, terminator present/absent. Require the
    corpus SIZE and the sentence "this number is only as good as this corpus" in the report.
  GRADUATES: when an adversarial corpus is generated from the guard's own rule table rather than by hand, so the
    spellings come from the code instead of from whoever is feeling thorough today.

L16 A guard that gates writes to ITSELF cannot be repaired by the isolation we default to.
  OCCURRENCES: PR #970, PR #974. Recorded once in `docs/BACKLOG.md` as a one-off; it is not one.
  EVIDENCE: gate-layer work dispatched with isolation:"worktree" deadlocks — ansible-gate-guard refuses every write
    to `.claude/hooks/**`, including the edit that FIXES the guard, and its only documented escape is a confirm file,
    so the agent's choices are "create a bypass" or "deliver nothing". The deadlock is invisible at dispatch time:
    the brief looks ordinary, the worktree is created normally, and the refusal appears only after the agent has
    done all the analysis, so the cost is paid in full before the blocker is discovered. #974 is what good looks
    like — the agent hit the wall, created NO bypass, and delivered the finished work as a patch verified with
    `git apply --check`.
  APPLIES: every dispatch that edits `.claude/hooks/**` or the suites that guard it.
  RULE: dispatch gate-layer work NON-ISOLATED, or fix `project_dir` resolution to use the actual toplevel. A blocked
    agent that hands back an applicable patch has lost nothing but the commit. Deciding to create the confirm file
    is the USER's, never the supervisor's — self-authorizing past a HARD_STOP is indistinguishable from routing
    around it, and it is the supervisor who is least able to see that difference in the moment.
  GRADUATES: when a gate-layer dispatch either resolves its own project_dir or is REFUSED AT DISPATCH, so the
    deadlock costs a turn instead of an agent-hour.

L17 An alert that fires by ACCIDENT is not coverage, and coverage cannot be enumerated from the alerts that exist.
OCCURRENCES: GeminiResolverNotResolving (CLAUDE.md §OBSERVABILITY) worked only because rejected calls consumed cap
  slots, so sustained rejection tripped the APPROACHING-CAP alert — nobody designed that, and fixing the cap
  accounting switched it off silently. 2026-08-19: the approaching-severe alert surfaced a four-month feed outage
  only because a pattern happened to reference the dead series.
EVIDENCE [2026-08-19]: `grep -rl ADP_EMPLOYMENT ThresholdEngine/config/patterns/` returns ZERO — same for
  INDEED_POSTINGS and REDBOOK_SALES. All three are Sentinel-primary (SentinelCollector/AGENT_README.md D-18), so no
  pattern means no `thresholdengine_pattern_data_overdue_days` series, means no alert is POSSIBLE. Last publish:
  2026-07-10, 2026-04-16, 2026-04-23. Two were already dead and nothing anywhere could have said so.
APPLIES: any alerting or observability change, and any claim that a subsystem is monitored.
RULE: enumerate coverage from the DATA side, never from the alert list — for the thing you care about, grep the
  config that would HAVE to mention it. One grep, before trusting that something is watched. The alert list is
  structurally blind to absence, so reading it can only ever confirm what already fires.
GRADUATES: delete this when every series in D-18's owned-series list has a freshness metric independent of whether
  any pattern references it — checkable by grepping the metric name per series and getting a non-empty result for all.

## ANTI [HARD_STOP @end for recency]
never state a brief's mechanism, line number or severity as settled fact
never amplify a correction you have not verified
never relay "X is false" without restating X's proposition and naming its subject
never repair citations before your last content edit, and never trust a green sweep as proof
never hand a fix round a list of sites — brief the class and require the count before the fixes
never let a round volunteer framing it did not measure, and never leave a cheap fact unqueried
never compile before the final commit — the marker keys to the tree, not the content
never dispatch onto a branch still held by a finished agent's worktree
never leave a worktree behind with a staged index — it is one commit from reverting merged work
never accept a self-authored "nothing loosened" as a merge signal — the corpus must come from a
  different mind than the fix, and the shape that leaks is a SPELLING VARIANT of one already covered
never dispatch gate-layer work worktree-isolated, and never create the confirm file yourself
never add an entry here that a template, hook or checklist already enforces
