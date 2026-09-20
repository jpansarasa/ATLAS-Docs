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
      && git ls-files -z | grep -zE '\.devcontainer/(compile|typecheck|dev)\.sh$' \
      | xargs -0r grep -lZE 'nerdctl|devcontainer_compose' \
      | xargs -0r grep -LE '^[^#]*devcontainer_own' )

silent -> each run owns its own identity, parallel is safe. Any name printed -> SEQUENCE those.
✗ never substitute a COUNT comparison (`grep -l devcontainer_own` vs `ls */.devcontainer/compile.sh`).
  It over-reports and orders the wrong action: a script that starts NO container has no identity to
  collide over, so its exemption reads as a gap. Measured on this tree 2026-09-20: 12 vs 13, the one
  difference being FinBertSidecar's pure-python compile.sh. The middle filter IS that exemption, keyed
  to a property the check can OBSERVE; a filename allowlist would rot the next service that is added.
✗ never drop the `^[^#]*` anchor off the LAST filter. Every one of these scripts carries a COMMENT
  naming devcontainer_own, so a bare `grep -L devcontainer_own` calls a commented-out ownership call
  "owned" and prints nothing — a gap reading as safe, which is the direction that dispatches N colliding
  compiles. The suite's `devcontainer_gap_verdict` has always used the anchored form; this pipeline used
  the bare one, so the two disagreed on exactly that input. Re-measured 2026-09-20: 15 verification
  scripts, 14 of them container-driving, and BOTH forms print nothing — latent, 0 of 14, not live.
  The `*/` glob is a second false reading — it cannot see edge/sentinel-edge/.devcontainer, two levels
  down, whose typecheck.sh does drive containers. `git ls-files` is depth-agnostic and skips the
  other agents' checkouts under .claude/worktrees/, which `find` from the root would audit.
✗ never run it UNANCHORED. `git ls-files` is CWD-relative: from SecMaster/ it enumerates that one
  subtree, the pipeline prints nothing, and silence reads as "parallel is safe" — a silent false green
  where the old `*/` glob at least errored loudly. The `cd "$(git rev-parse --show-toplevel)"` subshell
  is the anchor. Re-measured 2026-09-20: from the repo root and from SecMaster/, rc 0 and empty stdout on
  a clean tree, and both name `zzgap/.devcontainer/compile.sh` with a gap planted in the index.
✗ never run it UNQUOTED. Plain `xargs` splits on whitespace, so a path containing a space is handed to
  `grep` as two nonexistent files: the errors go to STDERR and stdout stays EMPTY, which under the
  contract above reads as "no gaps". Measured 2026-09-20 on a throwaway repo holding one planted gap at
  `my svc/.devcontainer/compile.sh` — the unquoted form printed nothing on stdout, the `-z`/`-0` form
  above printed the gap. Latent, not live: 0 of the 29 tracked `.devcontainer/*.sh` paths contain a
  space today (`git ls-files | grep -E '\.devcontainer/.*\.sh$' | grep -c ' '`).
✗ never wrap it in a SCRIPT that reads only stdout or only rc. Outside a repo it writes two `fatal:`
  lines to stderr and exits **0** with empty stdout (`cd ""` succeeds, so the subshell keeps going) —
  indistinguishable from a clean tree by either signal. It fails loudly for a human watching the
  terminal and silently for anything else; a caller that must automate this has to check stderr.
AUTHORITATIVE, and the copy carrying a known-bad control: scripts/test-devcontainer-owner.sh test 5 —
  the same selector and the same two filters, now including the `^[^#]*` anchor the pipeline above used to
  drop, driven on EVERY run over a PLANTED TREE carrying one fixture per
  audit branch, and scored on whether the per-file lines correspond ONE-TO-ONE to the names that tree enumerated
  AND on whether each branch emitted its own line. The REAL tree is pinned separately, by a NAME roster rather
  than a count floor — a floor with slack absorbs a path-scoped exclusion, and no planted fixture can match an
  excluded real path. A filter quietly matching nothing, a branch that stopped firing, a narrowed pattern that
  makes a check silently not apply, and a loop reading a file other than the one it names all go RED there by
  name. What it still cannot see: the check is STATIC TEXT — an ownership call present but unreachable, or written
  early and CALLED late, reads as owned and in-order (docs/BACKLOG.md KNOWN DEFECTS).
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
