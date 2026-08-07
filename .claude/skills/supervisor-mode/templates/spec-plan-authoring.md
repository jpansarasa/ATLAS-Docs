# Template: spec / plan authoring

Writing a specification, an implementation plan, a revision of one, or a red-team of one — plus
the read-only research legs that feed them. Output is a repo doc that goes through a PR, so it is
code-review-shaped, not recon-shaped.

```
WRITE {A SPECIFICATION | AN IMPLEMENTATION PLAN} — {subject}. Deliver as
`docs/proposals/{slug}.md` on a new branch `docs/{slug}` off main (`{sha}`), own git worktree
(isolation set). Do NOT push, open a PR, merge, or implement anything.

DESIGN INTENT: none — a document. No production code, no behaviour change.
supersedes: none
guard_tests: none

TRAJECTORY
1. MEASURE FIRST, WRITE SECOND. The document is written after the evidence, not before it. If you
   cannot cite a number for a claim, it does not go in as a claim.
2. Read the in-scope `{Service}/AGENT_README.md` DECISIONS blocks and the relevant CLAUDE.md
   sections. A plan that contradicts a D-entry without naming the supersession is a STOP.
3. Every acceptance criterion needs FOUR things: a measure, a pass value, a FAIL value, and a
   negative control separating "broken" from "no data yet". A criterion that cannot fail is not a
   criterion. One plan's originally proposed metric would have PASSED while the path it measured
   had been dead for 37 days.
4. State the SEQUENCING and say which edges are load-bearing and why — "B blocks C because
   repairing before the faucet closes re-accumulates at the seeding rate" is a dependency;
   "B then C" is a preference.
5. Verify that every number you inherited from the brief or an earlier report still holds, and list
   the corrections explicitly in the doc. On one plan that section changed the outcome.
6. A "what could not be verified" section is MANDATORY. List it rather than smoothing it over.
7. DO-NOT-BUILD is a legitimate deliverable, and a spec that cannot reach it is advocacy, not
   analysis. Say so plainly when the evidence points there.
8. Say where the rollback lives, and whether it is real. ZFS snapshots cover both databases at once
   so they are a disaster floor, not a per-change undo; a per-row pre-image may be required.
9. Commit, then `scripts/claude-mark-verified "<reason>"` bare. The supervisor opens the PR.

CONSTRAINTS FOR THIS CLASS
- A benchmark score is not production capability. Never cite one as evidence the wired path works;
  check `compose.yaml`, the options class and the DI registration.
- Plans are not kept on main. At completion the plan is retired per CLAUDE.md PHASE_TAGS, with the
  recovery pointer recorded in `docs/RELEASES.md`. `.claude/hooks/plan-retirement-guard.sh`
  enforces this — write the doc knowing it is temporary.
- No feature flags. A default-OFF flag is tech debt plus an unwired path; propose the cutover.

FAILURE MODES SEEN HERE -> THE CHECK
- An acceptance metric that passes on a dead path. Check: step 3's negative control, and confirm
  the Prometheus series you are gating on EXISTS today — several proposed signals emit no series
  at all, so they cannot gate anything.
- Population bias in the evidence behind the plan. Check: state per measurement what the sample
  structurally cannot contain — the plan inherits every blind spot in the evidence it rests on.
- A recommendation that would destroy the evidence it rests on. Check: a proposed quarantine of
  two symbols would have deleted real instruments, and the corrupt name was the only surviving
  record of the surface that produced it.

Report (<=20 lines): branch + commit, the doc's verdict in one sentence, the acceptance criteria
with their fail values, and what you could not verify.
```

## Notes for the supervisor

- Red-team the document as a SEPARATE dispatch before the PR, with a different lens than the
  author's. That step is what turned one spec into a DO-NOT-BUILD instead of a build plan.
- Research legs are `recon-measurement.md` dispatches; keep them separate from authoring so the
  author is not also the one grading the evidence.
