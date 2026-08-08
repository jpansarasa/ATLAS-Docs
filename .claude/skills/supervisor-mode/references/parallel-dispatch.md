# Reference: running agents in parallel without them compiling each other's code

Read this BEFORE dispatching two or more code agents at once, and especially before dispatching two
that will run `compile.sh` or `build.sh`. SKILL.md DISPATCH PARALLELISM carries the decision rule.

## Branch/worktree matrix
CANONICAL: SKILL.md DISPATCH PARALLELISM. Not restated here — it is a dispatch-time decision rule,
so it stays resident, and a matrix enforced in two places drifts in one of them.

## The devcontainer trap — check, never assume
Whether a worktree isolates the COMPILE flow depends on what identity the compile scripts derive.
If two runs resolve to the same compose project AND service they are ONE container identity, so the
second run's `compose exec` lands in the FIRST run's /workspace, compiles code it did not check
out, and attests it — silently, exit 0. The project name is path-independent (a `name:` key, or the
`.devcontainer` basename when absent), so adding `name:` is NOT the fix. Shared host ports and
globally-named volumes collide the same way, and a teardown trap in compile.sh can remove a
container another run is still using.

BEFORE dispatching parallel compiles, ask the tree rather than this file:

    command grep -l devcontainer_own */.devcontainer/compile.sh | wc -l
    ls */.devcontainer/compile.sh | wc -l

all covered -> each run owns its own identity, parallel is safe. Any gap -> SEQUENCE those.
# this states the MECHANISM only. Per-service ports, volume names and counts live in the compose
# files; an inventory copied here is wrong the next time someone edits one.

## Dispatch hygiene
ALWAYS TELL each agent which files belong to other in-flight agents.
DEFAULT for parallel code dispatch: pass `isolation: "worktree"` on the Agent tool call — the tool
  creates a temporary git worktree per agent and cleans it up on completion.
  rationale: shared working tree + concurrent git checkout = silent commit loss.
EXCEPTIONS: docs-only parallel work on disjoint files can skip worktrees; single-agent dispatches
  do not need them.
PRUNE the finished agent's worktree BEFORE dispatching the next agent onto that branch
  # LESSONS.md L4 — the agent half (take, never delete) only works if the supervisor does this half
