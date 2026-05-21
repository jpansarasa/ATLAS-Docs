# Smoke test: git-push-guard global tree-hash marker scan

Verifies the 2026-05-21 change to `.claude/hooks/git-push-guard.sh` that
moved the tests-passed marker lookup from a single per-worktree file to
a global scan of every `/tmp/atlas-test-markers/tests-passed.*` file,
matching by tree-hash.

Driver: [`run-push-guard-smoke.sh`](run-push-guard-smoke.sh). It builds
a real git repo + bare remote in tempdirs (so the hook's
`git rev-parse <branch>^{tree}` and `git log origin/main..<branch>`
calls resolve), plants synthetic markers with sha1-shaped suffixes
that no real worktree would ever derive, drives the hook end-to-end
via the same JSON-on-stdin shape the Claude harness emits, and asserts
on the hook's JSON output.

Backs up every real marker before the run and restores them on exit
(trap-cleanup), so the production marker dir is never corrupted.

## Run

```
$ timeout 60s bash /home/james/ATLAS/.claude/hooks/test/run-push-guard-smoke.sh
```

## Transcript

```
=== SETUP ===
HOOK:       /home/james/ATLAS/.claude/worktrees/agent-a59f2f4f3f410f1d7/.claude/hooks/git-push-guard.sh
MARKER_DIR: /tmp/atlas-test-markers
TMP_REPO:   /tmp/tmp.lKZ0W0K850
TMP_BARE:   /tmp/tmp.D4bvLSx3zT
BACKUP_DIR: /tmp/tmp.nZlXwKHBTk

Backing up existing real markers...
  backed up: tests-passed.240724d82eec
  backed up: tests-passed.a6faf43df0d9

Building synthetic repo at /tmp/tmp.lKZ0W0K850 ...
  feat/smoke tree = dfa878bc599b0515d719920d6d755ee6301312fd

=== TEST 1: synthetic marker with non-matching filename covers current tree ===
Planted:
-rw-rw-r-- 1 james james 70 May 21 07:30 /tmp/atlas-test-markers/tests-passed.aaaa00000001
-rw-rw-r-- 1 james james 70 May 21 07:30 /tmp/atlas-test-markers/tests-passed.bbbb00000002
Contents:
  A: v2 tree dfa878bc599b0515d719920d6d755ee6301312fd 2026-05-21T11:30:37Z
  B: v2 tree deadbeefdeadbeefdeadbeefdeadbeef00000002 2026-05-21T11:30:37Z

Running hook with CWD=/tmp/tmp.lKZ0W0K850 ...
--- hook stdout/stderr ---
{"hookSpecificOutput":{"hookEventName":"PreToolUse","permissionDecision":"allow"}}
--------------------------
  PASS: hook allowed push when marker A covers feat/smoke tree

=== TEST 2: no synthetic marker covers current tree ===
Now both synthetic markers cover unrelated trees:
  A: v2 tree deadbeefdeadbeefdeadbeefdeadbeef00000001 2026-05-21T11:30:37Z
  B: v2 tree deadbeefdeadbeefdeadbeefdeadbeef00000002 2026-05-21T11:30:37Z
  temporarily removed real marker: tests-passed.240724d82eec
  temporarily removed real marker: tests-passed.a6faf43df0d9

Running hook with CWD=/tmp/tmp.lKZ0W0K850 ...
--- hook stdout/stderr ---
{
  "hookSpecificOutput": {
    "hookEventName": "PreToolUse",
    "permissionDecision": "deny",
    "permissionDecisionReason": "BLOCKED: Tests have not passed for the current tree.\n\nBranch:       feat/smoke\nCurrent tree: dfa878bc\n\nMarkers scanned in /tmp/atlas-test-markers:\n  - tests-passed.aaaa00000001 version=v2 tree=deadbeefdeadbeefdeadbeefdeadbeef00000001\n  - tests-passed.bbbb00000002 version=v2 tree=deadbeefdeadbeefdeadbeefdeadbeef00000002\n\nRun the compile.sh script for modified projects to regenerate the marker:\n  {Project}/.devcontainer/compile.sh\n\nIf no automated validator applies (docs/config/scripts only), use:\n  scripts/claude-mark-verified \"<reason>\""
  }
}
--------------------------
  PASS: hook blocked push when no marker covers the tree
  PASS: block message lists marker file A
  PASS: block message lists marker file B
  PASS: block message lists tree covered by A

=== TEST 3: regression — pre-existing real marker still satisfies lookup for its own tree ===
  Real marker:      tests-passed.240724d82eec
  Marker tree-hash: d81308237428bd8767415b88df7f2277d2476ef1
  PASS: lookup matched tests-passed.240724d82eec for its own tree

=== SUMMARY ===
ALL ASSERTIONS PASSED

=== CLEANUP ===
removed synthetic: /tmp/atlas-test-markers/tests-passed.aaaa00000001
removed synthetic: /tmp/atlas-test-markers/tests-passed.bbbb00000002
restored real: tests-passed.240724d82eec
restored real: tests-passed.a6faf43df0d9
exit rc=0
```

## What each test pins

- **Test 1** — `should_allow_push_when_any_marker_covers_current_tree`:
  proves the global scan matches by tree-hash, NOT by filename suffix.
  The marker that covered `feat/smoke^{tree}` was named
  `tests-passed.aaaa00000001` — a suffix no real worktree would ever
  produce. The pre-change hook would have looked only at
  `tests-passed.<sha1(/tmp/tmp.XXX)>` and missed it.

- **Test 2** — `should_block_and_enumerate_markers_when_none_cover_current_tree`:
  proves the diagnostic block message lists every marker file it
  considered and the tree each one covers, so the operator can see at
  a glance why the lookup missed.

- **Test 3** — `should_match_existing_real_marker_for_its_own_tree`:
  regression — picks the first real marker present in
  `/tmp/atlas-test-markers/`, runs the same lookup loop the hook uses
  with `CURRENT_TREE` set to that marker's tree-hash, asserts the loop
  returns the same marker. Guards against a future refactor that
  silently breaks marker recognition.

## Post-review verification (2026-05-21)

PR #388 review flagged the hardcoded HOOK path on line 14 (pointed at the
original creator-worktree, which would FNF after worktree teardown).
Fix: HOOK is now derived from the script's own location via
`HOOK="$(cd "$(dirname "$0")/.." && pwd)/git-push-guard.sh"`, so the
smoke driver works from any worktree (or a fresh clone) that ships
both files together.

Re-run from a different worktree (`agent-af7b80be61b53a5bf`):

```
HOOK: /home/james/ATLAS/.claude/worktrees/agent-af7b80be61b53a5bf/.claude/hooks/git-push-guard.sh
PASS: hook allowed push when marker A covers feat/smoke tree
PASS: hook blocked push when no marker covers the tree
PASS: lookup matched tests-passed.240724d82eec for its own tree
ALL ASSERTIONS PASSED
```

Real markers were backed up before the run and restored on exit; the
production marker dir was not corrupted.
