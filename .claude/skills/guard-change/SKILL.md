---
name: guard-change
description: Changing a hook or gate under .claude/hooks, reviewing a guard/gate PR, or writing or reviewing the tests for a guard. Front-loads the recurring first-pass defects measured on git-push-guard.sh and ansible-gate-guard.sh across six PRs (#921, #925, #928-#931) and the seventeen adversarial ROUNDS the first five of them took - rounds and PRs are different counts - fixtures varying the wrong axis, sweeps with no teeth, decision-only assertions, unverified mutations, ambiguity resolving to allow, enumerations that grow another member.
---

# GUARD_CHANGE [SKILL v1]

Guard PRs on this layer took 8, 3, 3, 2 and 1 rounds (#921, #925, #928, #929, #930).
Roughly half of those rounds existed to fix a defect the PREVIOUS round introduced, and the
same small set of mistakes recurred. Rounds-to-merge is the metric this skill is judged by.

## ETHOS
ambiguity resolves to DENY               # every regression here reached ALLOW by failing to FIND something
gate the ACT, not a spelling             # an enumeration always grows another member
prove the harness, then trust the zero   # a blind sweep reads exactly like a clean one
assert the REASON, not the decision      # a right answer via the wrong rule passes a decision-only row

## READ_FIRST
`.claude/hooks/README.md`      # what each guard decides, the failure direction, and the scars already recorded
`.claude/hooks/test/run-*.sh`  # the ten suites; nothing runs them automatically (KNOWN GAP, README)

## ALREADY_COVERED [go there, not restated here]
guard-test contract, tautological tests -> `.claude/skills/intent-review/SKILL.md` GUARD_TEST_CONTRACT
mutation-verify step, findings-are-claims, take-never-delete a held worktree -> `.claude/skills/supervisor-mode/templates/implementation-fix.md`
re-deriving a reported number, population bias -> `.claude/skills/supervisor-mode/templates/claim-verification.md`

## IMPLEMENTER [first pass]
1. Name what the CODE READS, then vary THAT axis in the fixtures.
   # #928 shipped two deny-to-allow regressions: fixtures varied main's CONTENT while the code read the repo's REF LIST; #929 then had to add a CONFIG-STATE axis
   rationale: testing the wrong dimension looks identical to testing thoroughly

2. Prove the harness has teeth BEFORE trusting a zero.
   run it against the KNOWN-BROKEN version -> it must light up; swap the two guards -> the count must invert
   # #931's first sweep reported a clean 0/0 because relative guard paths failed to open after a `cd` - every cell read allow on both sides

3. Assert the REASON, not just the decision.
   # #930: `git --zzz stash push` denied correctly and told the operator "Direct push to main is not allowed" - the wrong remedy
   # #929: config-injection rows passed while denying via a different rule than the one under test

4. Every comment asserting a MECHANISM must be demonstrated, never reasoned.
   # #927: a comment claimed an empty variable would make the EXIT trap run `rm -rf "/"` - impossible; the assignment happens even on failure
   # git-push-guard.sh, "WHEN THE TEST FAILS" header: returning the ORIGINAL ref spelling was called "safe" for exactly the refs the gate exists to protect

5. Test the REAL invocation shape - the production call site, not an equivalent one.
   # #927's suite fed the prompt on stdin while production passed it in argv; green suite over a pipeline that never started a session

6. A row that passes for a SECOND reason proves nothing about the clause under test.
   # run-entry-shape-smoke.sh, "refspec push to main denies even when the TREE IS TESTED": the rows above it ran against an EMPTY marker dir, so they denied on the missing marker - delete the refspec block and they stay green
   check what else in the fixture could produce the same verdict, and make the control row prove the second reason is absent

7. Mutation-verify every new clause and REPORT THE COUNT per mutation.
   # hooks/README.md, "A command is a SET of acts": a chained-act fail-open survived its own deletion with all 507 assertions green
   rationale: a test cannot detect its own removal; the count, not the word "verified", is the deliverable

8. Ambiguity resolves to DENY, never ALLOW.
   every regression in this work reached ALLOW by failing to find something:
   # `head -1` over the raw command applied one repo redirect to every span (#921 C1)
   # decide-and-exit settled the command on its first act; later acts went unevaluated (#921, README "A command is a SET of acts")
   # an rc read on one of two resolver calls let a push land on main with no reason emitted (#928)
   # a redirect stripped by STREAM NUMBER instead of TARGET let `cat forged &> <marker>` land a forged tests-passed marker (#921; git-push-guard.sh "REGRESSION THIS REPLACES")

9. Gate the ACT, not a spelling - prefer a generic arm, or ask the tool.
   # #930: `-p` walked through because only `--paginate` was enumerated; the fix replaced the alternation with a generic dash-prefixed arm
   ok: `git rev-parse --symbolic-full-name` applies git's own rules (#928); `git push --dry-run --porcelain` as the test ORACLE (#929)
   but: check the tool answers the question you asked - `@{push}` reports where the BRANCH would go, not what the push would WRITE (#929, fast and wrong)

10. A shared test override binds EVERY participant - assert it in BOTH directions.
    # #921: only two of five marker writers honoured `ATLAS_MARKER_DIR`, so a suite wrote real verdict markers into the directory every agent's push gate reads; the fix asserts set->temp AND unset->live, because a fix hardcoding the TEMP path passes a one-sided test and silently disarms production (run-wiring-smoke.sh `marker_dir_isolation`). #930, one round later: the new suite ignores `PUSH_GUARD_HOOK`, which its sibling honours, so its teeth cannot be shown the documented way

## ENVIRONMENT [each has cost a round]
11. Use `command grep` for any exhaustive sweep.
    # the shell's `grep` is a function wrapping ugrep with `--ignore-files --hidden`; it honours ignore files and produced false negatives in a security sweep
12. Any "how often did this fire" count is a FLOOR, not a census - no guard writes a run log, and the only trace is a rotating per-session transcript.
13. A rename can silently break a grep-based dependency.
    # #931: a renamed constant dropped every `-c` from the replication and fail-opened the config-destination rule; the sweep missed it, the suite's two composed rows caught it
14. Prune the finished agent's worktree BEFORE dispatching the next one onto the same branch.
    # supervisor-side half of implementation-fix's take-never-delete rule; stale worktrees repeatedly blocked these branches on 2026-08-07/08

## REVIEWER
15. Re-derive independently; never audit the author's reasoning.
    # the rounds that found real defects re-ran the measurement (#928's review found 18 loosenings against the pre-fix head; #930's built its own 4,340-cell corpus); the diff-reading rounds found least
16. Weigh over-denial as heavily as under-denial on any path ordinary work flows through.
    # #925: the guard refused reviewed IaC edits and offered only a 4h global bypass - a guard that makes correct work require disabling all guards trains people to disable all guards
17. A GREEN mutation is a finding, not noise - investigate it and say WHICH it is.
    # #928: the drop-the-second-rc mutation measures 0, not 84 - a genuine equivalence, so the header and the battery both overstated the fix
18. Check WHEN a surviving defect was introduced before calling a PR non-convergent.
    # #927 round 5: the remaining flaw dated to the first commit and was untouched by rounds 3-5 - a surviving flaw is not evidence the latest round regressed

## ANTI [HARD_STOP @end for recency]
never ship a sweep whose SWAPPED direction was never run     # a blind harness reads as clean
never enumerate a spelling class                             # add the generic arm, or ask the tool
never decide-and-exit on the first act in a command          # a command is a SET of acts; any deny wins
never assert a mechanism in a comment you did not demonstrate
never trust a green suite over an invocation shape production does not use
never call a round converged without per-mutation counts
never narrow what you INSPECT to fix a false deny            # narrow the EXEMPTION instead - `head -1`, the ansible execute exemption, the text carve-out (README "narrow what you EXEMPT")
never let a fixture's second reason carry a row              # add the control that proves it absent
