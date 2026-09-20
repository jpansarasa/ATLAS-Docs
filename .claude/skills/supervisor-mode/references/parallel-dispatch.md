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

BEFORE dispatching parallel compiles, ask the tree rather than this file. This NAMES the gaps — a
script that drives containers without owning an identity — and prints NOTHING when there are none:

    ( cd "$(git rev-parse --show-toplevel)" \
      && git ls-files | grep -E '\.devcontainer/(compile|typecheck)\.sh$' \
      | xargs -r grep -lE 'nerdctl|devcontainer_compose' \
      | xargs -r grep -L devcontainer_own )

silent -> each run owns its own identity, parallel is safe. Any name printed -> SEQUENCE those.
✗ never substitute a COUNT comparison (`grep -l devcontainer_own` vs `ls */.devcontainer/compile.sh`).
  It over-reports and orders the wrong action: a script that starts NO container has no identity to
  collide over, so its exemption reads as a gap. Measured on this tree 2026-09-20: 12 vs 13, the one
  difference being FinBertSidecar's pure-python compile.sh. The middle filter IS that exemption, keyed
  to a property the check can OBSERVE; a filename allowlist would rot the next service that is added.
  The `*/` glob is a second false reading — it cannot see edge/sentinel-edge/.devcontainer, two levels
  down, whose typecheck.sh does drive containers. `git ls-files` is depth-agnostic and skips the
  other agents' checkouts under .claude/worktrees/, which `find` from the root would audit.
✗ never run it UNANCHORED. `git ls-files` is CWD-relative: from SecMaster/ it enumerates that one
  subtree, the pipeline prints nothing, and silence reads as "parallel is safe" — a silent false green
  where the old `*/` glob at least errored loudly. The `cd "$(git rev-parse --show-toplevel)"` subshell
  is the anchor, and it still fails loudly outside a repo. Verified identical output from the repo root
  and from SecMaster/.
AUTHORITATIVE, and the copy carrying a known-bad control: scripts/test-devcontainer-owner.sh test 5 —
  the same two filters, plus a gap planted from the tree's own scripts on EVERY run that the audit
  must name, so a filter quietly matching nothing cannot read as a clean pass.
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
  # templates/implementation-fix.md, Notes for the supervisor — the agent half (take, never delete) only
  # works if the supervisor does this half, and the sweep is for STAGED indexes, not merely held branches
