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
a change's SURFACE is enumerated in ONE pass before the first edit -- every site AND every artifact asserting a fact the change makes false, each PINNED by a test, FILED with a measurement or OUT OF SCOPE -- and a claim is fixed as a SET, never as the copy a finding quoted -> `templates/implementation-fix.md` TRAJECTORY step 3, pointed at from `templates/story-implementation.md` and `.claude/skills/review-discipline/SKILL.md` §BLOCK_TEXT, whose DEMAND-the-enumeration rule is the block-side twin # #1073: 13 sites surfaced 2 at a time across rounds 2 and 4, a round each
a finding cites the RULE, PATTERN or INTENT it violates or it is a PREFERENCE, and a manufactured reader sentence does not promote one -> `.claude/skills/review-discipline/SKILL.md` §FINDING_BAR, pointed at from its own BLOCK_TEXT and ANTI # user direction: a review agent cannot say "I don't like this, rewrite it"; for-loop-versus-while is not a substantive change
an agent that hands back work a reviewer has to debug has FAILED, so attack your own work first and report what you attacked, what broke and what did NOT -> `templates/implementation-fix.md` PRE-HANDBACK, the SELF_ATTACK block, pointed at from `templates/story-implementation.md`; both templates' report sections carry the list as a field # measured 2026-09-20: agents that self-attacked and said so closed in 1-2 rounds, unattacked handbacks in 4-7
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
a brief ordering a RUN names its POPULATION (the commands THIS branch authored, from the merge-base diff) and its MODE (anything unauthored is QUOTED with what it would do, never executed) -> `templates/docs-accuracy.md` TRAJECTORY step 4 and `templates/claim-verification.md` step 1, each derived for its OWN population because they differ -- the REPORT under verification versus the DOC -- and one shared line would have been wrong for one of them # was L21. Three instances: prod ALTER 2026-05-30 (its reconciling migration, SecMaster/src/Data/Migrations/20260530213012_WidenNaicsVintageForSicCrosswalk.cs, is the repo-side trace), CREATE TEMP TABLE 2026-09-17, and 2026-09-21 an agent that harvested and executed every backticked span in docs/BACKLOG.md. THE CLASS IS WORSE THAN DATA LOSS, and the confirm file needs THREE quantities because one number cannot carry them: 6 span occurrences (5 distinct texts) NAME `.claude/.ansible-gate-confirmed`; 3 of those CREATE it (one `touch`, two `printf > `); and of the 42 below, exactly ONE does -- the `touch`, because both `printf` forms put `printf` in command position. That one is the whole bypass: the same document records BOTH `printf` forms as DENIED by the guard, "leaving only `touch .claude/.ansible-gate-confirmed`, a four-hour bypass of the WHOLE layer". Harvesting that document DISARMS the guards that would have refused the rest. It also carries 2 spans that pipe a removal of the production compose file, 6 that pipe a removal of `.claude/settings.local.json` -- which holds `permissions`, NOT hooks, and is untracked and mode 600, so nothing in git restores it -- and a merge API call against an attacker-named repo. No hook can see any of it -- every guard here reads the command STRING, and .claude/hooks/README.md already names "the act performed by something the command merely starts" as uncatchable. HOW BIG IS THE HAZARD: a published pipeline -- recoverable at `git show d1a08b10:.claude/skills/supervisor-mode/LESSONS.md`, which is the only tree holding it -- returns 42, UNIT span occurrences, POPULATION the 4,104 spans IT scans; a property of THAT PIPELINE, never of the file, and a FLOOR three ways. It is line-based, so a span WRAPPED across lines is invisible: 144 span occurrences wrap under whole-document sequential pairing of backtick runs, 114 if only single-backtick runs are paired -- the count moves with the pairing rule, which is why the bare number was the wrong thing to ship. On the incident's own line it LOSES the real `git clean -fdx` and emits two PHANTOM prose fragments instead, so it under-counts AND injects noise. It reads only COMMAND POSITION, so every `echo <verb> ... | bash` form above is absent from the 42. And one member, `git apply --check`, is read-only. Re-derive it, never quote it. DELIBERATELY NOT DE-FANGED in the doc: that is destination-gating a growing population against CLAUDE.md GIGO, mangling the guard's own quoted bypass would break it for the blocked human who needs it, and the span that CAUSED the incident is one the pipeline cannot see -- so a de-fanging pass keyed on the 42 would have missed it
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
L20 AN INSTRUMENT MUST REPORT ITS OWN FAILURE IN A SIGNAL ITS FINDING CANNOT PRODUCE. Where one channel
  carries both -- RED means "this line is covered" AND "the file no longer parses"; RED means "the mutant
  died" AND "the runner is broken"; GREEN means "no defect" AND "my fixture cannot see this axis" -- the
  run yields NO VERDICT, whatever it printed.
  EVIDENCE: three in ONE PR's review chain (#1089); two re-checkable from the repo today, the third
    disclosed in-session and NOT verifiable from it -- PR #1089 carries no reviews and no comments.
    1. commit 8863c5c2, finding F2: weakening the builder's `bool(labelled and accepted)` to
       `bool(labelled)` survived all four suites, because the control's fixture varied only the labelled
       axis. A green that meant "my fixture cannot see this", read as "the mutant is dead".
    2. round 2's own deletability table recorded `build_attach_gold.py`'s unmeasured-gold warning as
       PINNED. Its `print` is the ONLY statement in `if unmeasured:`, so deleting that line left an empty
       block: the interpreter refused the file and every suite went red. Re-derived as a whole statement it
       is DELETABLE -- the corrected table, 14 of 176 units, is docs/BACKLOG.md MEASUREMENT DEBT.
    3. NOT RE-CHECKABLE, and kept anyway because it is the sharpest form: the reviewer's own mutation
       runner returned RED for its NULL control, so every KILLED it reported was worthless until it was
       rebuilt. The reviewer disclosed it in-session; nothing in the repo or PR #1089 records it, so this
       one is testimony, not evidence -- 1 and 2 above are the re-checkable pair.
  RULE: before reading a measurement, name the OTHER thing that produces this same signal, and make the
    instrument decide which -- cheapest sufficient form, in this order: (a) a NULL control in the SAME
    invocation, unmutated input that must come back with the OPPOSITE verdict, which is what catches a
    broken runner; (b) a validity check on the mutated artifact BEFORE the suite runs -- parse, compile,
    load -- whose failure is its own verdict; (c) a third value the tool is allowed to return, NO VERDICT,
    distinct from both PASS and FAIL. A known-bad control alone does not do this: it proves the tool can
    say FAIL, never that its FAIL means what you are about to read.
  GRADUATES: `.claude/skills/intent-review/SKILL.md` §AIMED AT THE ACT, as a fourth shape beside its three.
    DO NOT RE-LITIGATE COLLAPSING IT INTO THAT SECTION TODAY [supervisor direction 2026-09-21]: different
    SURFACE and different REMEDY -- that section is a pass/fail contract on a control that appears IN A DIFF
    and is therefore read by a review, while this rule's subject is an instrument no diff contains and no
    review reads (a scratch mutation runner, an ad-hoc deletability sweep, a reviewer's own harness) and its
    remedy is a THIRD verdict, NO VERDICT, rather than a stricter pass.
    `templates/recon-measurement.md` step 3 already carries the narrow half ("syntax-check the thing under
    test before measuring") for RECON harnesses only, which is why instance 2 was built by an agent that had
    never read it. It graduates when either artifact binds ANY instrument a verdict rests on.
  GRADUATE_CHECK: grep -qi "distinct failure channel" .claude/skills/intent-review/SKILL.md .claude/skills/supervisor-mode/templates/recon-measurement.md
L22 A ROUTING IS NOT A WRITE, AND AN APPROVE VERDICT IS THE WRONG PLACE TO PROMISE ONE: the median approval
  merges SIX SECONDS after its verdict is recorded, so a filing the reason promises has no PR left to land
  in -- CLAUDE.md WHERE_WORK_LANDS requires the SAME PR. The claim reads as the act and nothing checks it.
  EVIDENCE [all figures at ~/.claude/atlas-pr-verdict.log = 366 lines, 2026-09-21; it APPENDS, so re-derive
    rather than quote]. Counts come from the VERDICT FIELD (`awk '$3=="approved"'`): 187 approved, 179
    blocked. NOT from a whole-line `grep -c ' approved '`, which returns 191 -- four BLOCKED verdicts
    (log lines 99, 105, 127, 298) whose REASON PROSE contains the word.
    TWO DIFFERENT QUANTITIES, and conflating them overstates the practice 3x:
      MATCHED SET = 12 approve reasons whose text matches the filing pattern.
      PRACTICE COUNT = 4 of those 12 are an actual filing claim -- #962, #978, #1090, #1093.
      The other 8 are over-matches in SIX modes, all on the word "filed": negation ("corrected rather
      than filed"), counterfactual (#984 "I would have filed a false blocking finding"), CLOSING a
      pre-existing entry (#999), a filing made EARLIER in the same PR (#1085, #1091), document LAYOUT
      (#1036 "filed under a hardware heading", #1089 "filed by date"), and category vocabulary or tool
      behaviour (#1086, #1087). READ EVERY WINDOW, NOT THE FIRST: #978 matches on a negation and, later
      in the SAME reason, on a real claim, so one window per verdict undercounts the practice.
    ALL 12 merged 4-8s after their verdict; the 4 genuine ones at 4s, 8s, 5s, 7s.
    THE MECHANISM IS BROADER THAN THE MATCHED SET AND IS NOT UNIVERSAL [the merge half resolves against a
    GIT REF, so stamp it: at origin/main 20d1540d]: of 183 approvals whose PR has a merge commit, 151
    (82.5%) merge within 10s and the median gap is 6s -- but 12 exceed 600s, the longest just under 10
    hours. So "the window is usually seconds", never "structurally impossible". Resolved against THIS
    branch the same figures read 182 and 150, the one-PR delta being #1092, unreachable from here.
    WORKED CASE #1090: the reason says three blindness items were "routed to the backlog with re-checkable
    measurements"; approve 10:33:28Z, merge 10:33:33Z. The branch DID add 101 lines to docs/BACKLOG.md,
    which is what makes this hard to see -- those are the round 2-3 residue, written earlier, and NONE of
    the three round-4 items' distinguishing phrases (cannot-check, drives the tool's own command line)
    appears in docs/BACKLOG.md at 56a3eba1. A non-empty destination diff is not evidence THIS claim landed.
    AND #1090 IS NOT THE OLDEST -- all three genuine claims tested are unlanded, a month apart: #962's merge
    does not touch docs/BACKLOG.md AT ALL, and #978's adds 88 lines while none of its claim's distinguishing
    words (re-wrapped, mv-plumbing, misattribute, archive filename) occurs anywhere in that file at
    origin/main. So the practice count is 4 and the landed count, so far as three tests can show, is 0.
    Same shape, different store: the 2026-09-17 psql breach was routed to THIS FILE in-session and, as its
    memory entry records, was still unwritten on 2026-09-18.
    Re-check, read-only, from the repo root -- one row per approve reason mentioning a filing:
      awk '$3=="approved"' ~/.claude/atlas-pr-verdict.log | grep -iE 'routed to the backlog|FILED|to be filed' | awk '{print $1, $2}' | while read -r ts pr; do n=${pr#PR#}; m=$(git log -1 --format='%aI' --grep="(#$n)"); echo "$pr verdict=$ts merge=$m"; done
    READ EVERY ROW BEFORE COUNTING IT: the pattern over-matches the six ways above, so the rows are
    CANDIDATES. Only the TIMING half is decidable without judgement. It also cannot see a routing promised
    anywhere but this log -- the LESSONS.md case above is invisible to it, because no store records it.
  RULE: land the entry on the branch BEFORE recording the verdict, never in the reason. A reviewer holding
    residue at terminal budget either commits the backlog entry first, or BLOCKS -- those are the two
    answers, and "routed to the backlog" in an approve reason is neither. Reading side, standing: a record
    saying work was filed is not evidence it was, and a non-empty diff of the destination is not evidence
    either; grep the destination for THIS claim's own distinguishing words before believing it.
  GRADUATES: `scripts/claude-pr-verdict` as a FIFTH precondition beside its four, refusing an approve whose
    reason claims a filing unless docs/BACKLOG.md differs from the merge-base on that branch. The tool is
    the right home and already holds both halves it needs -- it parses the reason
    (BACKLOG_DECISION_PATTERNS) and resolves the canonical store (canonical_backlog) -- but it consults the
    backlog only for a prior BLOCKED decision, never for whether the filing this reason promises exists.
    It must classify on the SIX over-match modes above or it will refuse honest approvals; that is the work,
    and it is why the entry is not simply an edit. Prose cannot hold this one: the promise is made at the
    moment the work stops, which is the moment nobody re-reads anything.
  GRADUATE_CHECK: grep -qE '^[^#]*FILING_CLAIM' scripts/claude-pr-verdict

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
never read a RED or a GREEN whose other cause you have not ruled out -- name what else emits this
  signal and make the instrument say which, or the run has no verdict [L20]
never execute a command because a document contains it, and never order a RUN without naming the
  POPULATION and the MODE -- an unbounded "run them verbatim" has named the whole file
never believe a record that says work was FILED, or a non-empty diff of the destination -- grep the
  destination for that claim's own words, and never promise a filing in an approve reason [L22]
never add an entry that a template, skill, hook or checklist already enforces
never add an entry without a GRADUATES clause NAMING THE ARTIFACT that will hold it and a GRADUATE_CHECK -- `none --
  judgement` is an answer, an absent line is not, and `scripts/new-epic.sh` refuses the next epic reset over either
  that or a check that now PASSES
