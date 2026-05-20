# Subagent template — story implementation (≤400 words)

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
  - docs/plans/atlas-matrix/**
  - docs/llm/**
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

## Build / verify
- `bash {Service}/.devcontainer/compile.sh` (with tests). Per CLAUDE.md
  GIT_PUSH HARD_STOP: 0 errors / 0 warnings / all tests pass.
- Do NOT deploy. Do NOT push. Do NOT open PR.

## Reporting back (final reply, ≤200 words)
- Branch + final commit hash
- Files touched (paths, ≤20 lines)
- Build status: errors / warnings / tests pass
- Deviations from spec + rationale
- Anything blocked / needed from supervisor

## Reference docs
- {epic plan file path} (Story {N.M.K})
- /home/james/ATLAS/CLAUDE.md (HARD_STOPS — especially MIGRATIONS,
  GIT_PUSH, DEPLOYMENT)
- /home/james/ATLAS/docs/atlas-matrix-mvp-plan.md (canonical AC)

## Stop conditions
- Hit a blocker the supervisor must resolve → stop and report.
- Build fails after a reasonable fix attempt → stop and report.
- Spec ambiguous in a way that materially changes the result →
  pick the lower-risk option, document the choice, continue.
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
