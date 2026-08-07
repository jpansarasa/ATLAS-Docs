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
- The supervisor owns these files — do not touch:
  - STATE.md
  - .claude/skills/supervisor-mode/**

## Git ops hygiene (HARD_STOP — supervisor-edit preservation)
- If `git status --short` shows supervisor-owned files modified (e.g. ` M STATE.md`),
  DO NOT `git stash`, `git restore`, or `git checkout -- <path>` them.
- `git checkout -b <newbranch>` and `git pull --ff-only` BOTH preserve dirty
  tracked files when the new ref doesn't touch them — proceed as-is.
- The only valid action on supervisor-owned modifications is leaving them alone.
- If you literally cannot proceed (e.g. genuine merge conflict on a supervisor
  file), STOP and report the conflict — do not "resolve" it by reverting.
- Background: dispatched agents have wiped weeks of supervisor STATE.md edits
  via stash-and-never-pop; see stash list `git stash list | grep STATE`.

## Deliverables
{Numbered list of concrete artefacts. For DB work, always include the
EF migration step explicitly per CLAUDE.md HARD_STOP:
"Use `nerdctl compose exec -T {svc}-dev dotnet ef migrations add {Name} --project {path}` —
never hand-author migration .cs files."}

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
  the supervisor/human decides, not you.)

## Build / verify
- `bash {Service}/.devcontainer/compile.sh` (with tests). Per CLAUDE.md
  GIT_PUSH HARD_STOP: 0 errors / 0 warnings / all tests pass.
- Do NOT deploy. Do NOT push. Do NOT open PR.

## Reporting back (final reply, <=200 words)
- Branch + final commit hash
- Files touched (paths, <=20 lines)
- Build status: errors / warnings / tests pass
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
- Default to `run_in_background=true`. Supervisor gets a notification
  on completion; can dispatch parallel work meanwhile.
- For the same epic / same SecMaster project, sequence stories on the
  same branch. Across epics use parallel branches.
- Do NOT use worktree isolation when the story needs the
  `.devcontainer` build/test/migrate flow — the devcontainer mounts
  the fixed `/home/james/ATLAS` path.
