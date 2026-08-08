# Reference: STATE.md — why it disappears, and what does not restore it

Read this BEFORE any operation that moves HEAD (checkout, switch, `pull --ff-only`, merging a PR
from the main checkout), or when STATE.md is missing and you need to know which case you are in.
The hard rules live in SKILL.md CONFIG STATE; this file is the mechanism behind them.

UNTRACKED [gitignored, since #922]: never commit | push | PR it. It no longer propagates into agent
  worktrees, and that propagation was the whole mechanism by which dispatched agents reverted
  supervisor edits. Nothing recovers a lost edit now — no commit, no stash. Write it, keep it.

UNSEARCHABLE: `grep -r` and the Grep tool both honour .gitignore, so a repo-wide search does NOT
  see STATE.md (verified: `grep` is a ugrep wrapper carrying --ignore-files). Read it by explicit
  path; a recon agent told to "search the repo" will silently miss it.

DESTROYABLE [two families, not one]: `git clean -x` deletes ignored files. The other family is a
  HEAD move, and BOTH ends decide it: OVERWRITTEN if the ref you move TO tracks STATE.md, DELETED
  if only the ref you are ON does, PRESERVED only when NEITHER does (all verified live, git 2.43):
  - a ref that still TRACKS it: checkout SILENTLY OVERWRITES the live file with that ref's
    committed copy, rc=0, no prompt. Git REFUSES this when the file is merely untracked; being
    IGNORED is what removes the protection. This is the case that will actually bite — every
    branch cut before the untracking PR still tracks STATE.md.
  - a ref where it is also untracked: PRESERVED. Two post-PR branches switch freely, so "any move
    to a ref where STATE.md is untracked" overstated it.
  - away from a ref that tracks it (the `git pull --ff-only` after merging the untracking PR):
    DELETED silently, but only from a clean working copy; uncommitted edits abort it.

Forbid `clean -x` in dispatches; the checkout family cannot be forbidden, so guard it instead.
`git show <sha>:STATE.md` recovers only the last TRACKED content and loses every edit made since,
so it is not a restore.

WRITE_GATE and VERIFY-don't-copy are CANONICAL in SKILL.md (CONFIG STATE, CONTEXT_HYGIENE) and are
not restated here — they gate every write, so they stay resident, and a rule enforced in two places
drifts in one of them. This file is the destruction MECHANISM only.
