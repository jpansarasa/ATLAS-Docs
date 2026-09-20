# Subagent template — story implementation (<=400 words)

Use this for ATLAS Matrix epic stories that involve code changes. The
supervisor fills in the placeholders and dispatches via the Agent tool
with `subagent_type=general-purpose` and `run_in_background=true`.

```
You're implementing Story {N.M.K} of ATLAS Matrix Epic {N} — {short title}.

## Mission
{One paragraph describing the story's goal, the AC summary, and why the
story matters to the epic.}

## Working tree
- Repo: /home/james/ATLAS, currently on branch `{parent_branch}`
- Branch policy: `{branch_policy}`
  (typical: `git checkout -b epic/{N}-{slug}` if not already there;
  commit small reviewable commits; do NOT push; do NOT open PR)
- Supervisor-owned, do not touch:
  - .claude/skills/supervisor-mode/**
  - STATE.md — gitignored; `git clean -x` deletes it unrecoverably

## Git ops hygiene (HARD_STOP — supervisor-edit preservation)
- If `git status --short` shows supervisor-owned files modified (e.g.
  ` M .claude/skills/supervisor-mode/SKILL.md`), DO NOT `git stash`,
  `git restore`, or `git checkout -- <path>` them.
- `git checkout -b <newbranch>` and `git pull --ff-only` BOTH preserve dirty
  tracked files when the new ref doesn't touch them — proceed as-is.
- The only valid action on supervisor-owned modifications is leaving them alone.
- If you literally cannot proceed (e.g. genuine merge conflict on a supervisor
  file), STOP and report the conflict — do not "resolve" it by reverting.
- Background: agents wiped weeks of supervisor edits via stash-and-never-pop
  (`git stash list | grep STATE`); untracking STATE.md closed that path, not
  the reflex.

## Deliverables
{Numbered list of concrete artefacts. For DB work, name the EF migration
step and send the agent to `CLAUDE.md` §MIGRATIONS for the command
VERBATIM — never restate it here. `--project` is forbidden there by name,
and that block carries the `cd` wrapper, `--output-dir` and the
`dotnet tool restore` precondition, and routes the per-service dev-service
name and the `--context {Svc}DbContext` rule to
`.claude/hooks/README.md` §EF_MIGRATION_TRAPS. Never hand-author a migration .cs.}

## Design intent (MANDATORY stanza — supervisor fills VERBATIM, never paraphrases)
- decisions: {the in-scope D-entries copied VERBATIM from
  <Service>/AGENT_README.md DECISIONS block — full lines, not summaries;
  or "none — no D-entries in scope for the touched code"}
- supersedes: {D-n | none}
- guard_tests: {one deliverable per new/changed guard — construct the
  violation, assert refusal AT the boundary through the real flow, mock
  ONLY the external client, RED if the guard is deleted; contract:
  .claude/skills/intent-review/SKILL.md §GUARD_TEST_CONTRACT}
- Conflict rule: if this brief contradicts a D-entry without a named
  supersession above -> STOP and report; never route around it, never
  obey the stale entry. (The entry may be outdated OR the brief wrong —
  the supervisor/human decides, not you.) The same stop applies to
  a rule stated in a skill, a template or CLAUDE.md, not only a D-entry:
  NAME the rule and the contradiction, and never silently obey a written
  rule you believe is stale — say so — CLAUDE.md INTENT_FIDELITY CONFLICT.

## Before the first edit
- ENUMERATE THE SURFACE the change touches -- every site, and every artifact
  asserting a fact the change makes false -- each PINNED by a test, FILED with
  a measurement, or OUT OF SCOPE and why; hand that list back with the work.
  Dispositions, the fix-the-claim-SET rule and the #1073 measurement:
  `implementation-fix.md` TRAJECTORY step 3

## Build / verify
- `bash {Service}/.devcontainer/compile.sh` (with tests), AFTER the final commit.
  Per CLAUDE.md GIT_PUSH HARD_STOP: 0 errors / 0 warnings / all tests pass.
  The marker keys to `HEAD^{tree}`, which every tracked path feeds — so a commit
  after the build (comments and docs included) remaps it and the push gate
  refuses a tree nobody built.
- Do NOT deploy. Do NOT push. Do NOT open PR.

## Pre-handback (run every one, report each result)
- SELF-ATTACK, and it covers this first draft, not only a fix round: try to
  BREAK what you are handing back, then report the LIST -- what you attacked,
  what broke, what you tried that did NOT break. A line claiming it with no
  list is worth nothing, and a reviewer is not the detector. The moves and the
  measured round counts: `implementation-fix.md` PRE-HANDBACK, the SELF_ATTACK block
- DELETE THE ENABLING LINE of every guard or control you added — the
  registration/view, the startup seeding or zero-init, the evidence file a
  gate reads, the consumer of a flag — re-run, NAME what failed. Nothing
  failed -> decorative; fix it before handing back. Mutating the guard's
  LOGIC is not a substitute (six PRs, 2026-09-20, all mutated logic and
  all shipped an unwired guard).
- AIM EACH CONTROL AT THE ACT, not at the function you were already
  reading: drive the tool END TO END and assert on its OUTPUT and its
  EXIT CODE; let no assertion rest on a SECOND COPY of the rule the
  shipped code decides; build the fixture where the two candidate rules
  DISAGREE, not one they both pass. Contract + the five measured cases:
  `.claude/skills/intent-review/SKILL.md` §AIMED AT THE ACT.
- ABSENT IS NOT ZERO, and a missing path is not a short count. A lazily
  created series does not exist until something fails, so `or vector(0)`
  paints absent as a healthy 0; a re-check pointed at a path that is not
  there must FAIL, never report the count it reached. Say which of absent
  and zero each control you add can tell apart.
- SCALE-MATCHED MUTATION — mutate at the scale the control SHIPS at:
  poison one unit, pad with clean ones to the documented usage and one
  past it, and require the complaint to NAME the offending unit. At
  n=1 the aggregate IS the unit; read an unstated n as n=1. One mutant per condition, each breaking
  EXACTLY ONE — a fixture breaking several pins none of them.
- ENUMERATE COVERAGE FROM THE DATA SIDE, not from the instruments that
  exist: grep the config or rule file that would HAVE to mention the thing
  you care about. A census of instruments cannot show the missing one.
- For every FIELD you write, name its READER, or do not write the field.
- Every number carries the COMMAND that produced it and the POPULATION it
  came from, and you state that the population CAN contain what you claim
  — a post-restart counter holds no pre-restart samples (#1071).
- Every file/line reference re-derived at the final commit, never copied.
- NAME the rule or contract clause that changed what you did, or say
  plainly that none applied — the absence is data.

## Reporting back (final reply, <=200 words)
- Branch + final commit hash, and the attested tree hash = `HEAD^{tree}`
- Files touched (paths, <=20 lines)
- Build status: errors / warnings / tests pass
- Surface list + dispositions; self-attack list (attacked / broke / did not break)
- Deviations from spec + rationale
- Anything blocked / needed from supervisor

## Reference docs
- {epic plan file path} (Story {N.M.K})
- /home/james/ATLAS/CLAUDE.md (HARD_STOPS — especially MIGRATIONS,
  GIT_PUSH, DEPLOYMENT)

## Stop conditions
- Hit a blocker the supervisor must resolve -> stop and report.
- Build fails after a reasonable fix attempt -> stop and report.
- Spec ambiguous in a way that materially changes the result ->
  pick the lower-risk option, document the choice, continue.

## Standing rules (include verbatim in every brief)
- EVIDENCE POPULATION: before concluding anything from a sample, state
  what that sample structurally CANNOT contain. A review queue holds
  only surfaces that FAILED, so measuring "does this rule drop real
  issuers" against it is unfalsifiable by construction — the
  counter-examples were never eligible to enter. The recurring shapes:
  a failure-only journal, success-only residue, a single-path enqueue,
  and a window chosen because it is where you already looked. Name the
  blind spot, then pick a population that can contain it.
- MECHANISMS IN THIS BRIEF ARE HYPOTHESES, NOT INSTRUCTIONS. Any
  specific pattern, threshold, line number, count or API the supervisor
  names is unverified unless it says otherwise. Verify it; if it is
  wrong or insufficient, say so and do the right thing instead. Agents
  correcting the brief on evidence is the expected outcome, not a
  deviation.
- CHECK IT ALREADY EXISTS FIRST. This system is heavily built out, so
  the capability you are about to propose is usually present and merely
  unused — the index nothing queries, the client nobody wired, the job
  nobody reads. Before proposing to build, or explaining why something
  is hard, search the repo and list the running containers.
```

## Notes for the supervisor
- Keep prompts under ~600 words including the placeholders. The user's
  rule: short and focused.
- Recording a BLOCK on this story's PR writes the next brief: name the
  PROPERTY the artifact must guarantee, never the field you noticed -
  `.claude/skills/review-discipline/SKILL.md` §BLOCK_TEXT.
- Default to `run_in_background=true`. Supervisor gets a notification
  on completion; can dispatch parallel work meanwhile.
- For the same epic / same SecMaster project, sequence stories on the
  same branch. Across epics use parallel branches.
- Worktree isolation IS safe for the `.devcontainer` build/test/migrate
  flow: each compile.sh owns a compose project keyed to its worktree
  (scripts/devcontainer-owner.sh) and attests /workspace by inode match.
  N stories in N worktrees compile simultaneously — never sequence them.
