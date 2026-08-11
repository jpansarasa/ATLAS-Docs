# Template: implementation of a measured fix — FRESH branch or FIX round

The most repeated code class. Two entry states, one
trajectory: FRESH is a defect MEASURED first on a new branch off a named main sha, ending
PR-ready; FIX ROUND has the branch already open behind a PR, bounded by a findings list.
Distinct from `story-implementation.md`: no epic, no story number. Take the boilerplate from
that file (Working tree, Git ops hygiene, Design intent, Stop conditions, Standing rules) — this
adds only the moves. Those stanzas push the realized prompt to ~600 words; budget for it.

```
FRESH:     IMPLEMENTATION — {defect in one line}. New branch `{branch}` off main (`{sha}`).
FIX ROUND: FIX round {K} — PR #{N}, branch `{branch}`, head `{sha}`. Findings verbatim below.
Commit per layer (FRESH) / per finding (FIX ROUND). Do NOT push, open a PR, merge, deploy or
restart any service.

{Design intent stanza — verbatim D-entries, per story-implementation.md}

WHERE TO WORK — decide first
Running compile.sh or build.sh? Take NO worktree: work in /home/james/ATLAS and SEQUENCE behind
any other agent compiling. Worktrees isolate git state, not compose — every .devcontainer
project resolves to `devcontainer`, so a second worktree's `compose exec` runs in the FIRST
worktree's /workspace (wrong code compiled, then attested), and compile.sh's
`trap 'nerdctl compose down' EXIT` kills the shared container mid-run. No lock exists today.
Docs/config only? A worktree is fine. FIX ROUND: `git worktree list` first; if a stale worktree
holds `{branch}`, take it with `git checkout --ignore-other-worktrees {branch}` — never delete
another agent's worktree, never branch off a copy.

TRAJECTORY
1. Read `{Service}/AGENT_README.md`, the WHOLE DECISIONS block, before touching code. A guard you
   are about to move may be a D-entry GUARD site. Contradiction, no named supersession -> STOP.
2. FRESH: reproduce the defect, record the number — the PR's opening evidence and the acceptance
   measure. FIX ROUND: VERIFY EACH FINDING BEFORE FIXING IT. Findings are claims, not facts:
   re-derive the number, open the cited file:line (lines drift between rounds). One that does not
   reproduce is REJECTED WITH EVIDENCE — a valid outcome, not a deviation. DB SELECT-only;
   Loki/Prometheus anchored to actual `date -u`.
3. The fix, plus an `// INTENT(D-n):` comment at the guard site if a guard is involved. Commit.
4. The guard test: construct the violation, assert refusal AT the boundary through the real flow,
   mock ONLY the external client. Contract: `.claude/skills/intent-review/SKILL.md`
   GUARD_TEST_CONTRACT. Commit.
5. MUTATION-VERIFY each guard or alert rule you added or moved: delete or invert it, re-run,
   confirm RED, restore. A test that stays green is the bug, not the proof. Comment-only rounds
   skip this, never step 6.
6. `bash {Service}/.devcontainer/compile.sh` — every compile.sh is 100644 in git, so `bash`,
   never bare. CAPTURE THE FULL LOG AND THE REAL `$?`; `| grep | tail` hides Permission-denied
   and swallows the exit code. 0 errors AND 0 warnings AND all tests pass.
7. Alert rules -> `bash deployment/tests/alerts/run.sh`: promtool over the REPO's rules plus the
   committed unit tests, inside the prometheus image (promtool is not on the host PATH, and the
   running container holds the DEPLOYED rules). One `*_test.yml` case per new rule. Ansible
   playbooks under `deployment/ansible/playbooks/` -> `bash deployment/tests/ansible/run.sh`:
   syntax-check + check-mode + tag selection over the zfs rollback floor, creating and destroying
   nothing, so it is safe on the live host. It cannot see `command`/`shell` behaviour — read its
   DOES NOT CATCH header before trusting a green run. Schema change
   -> `nerdctl compose exec -T {svc}-dev dotnet ef migrations add {Name} --project {path}`.
8. Report: commit hash per layer/finding, each finding {addressed | rejected with evidence |
   deferred}, the before/after number, mutation results, compile counts.

CONSTRAINTS
- NARROW (FIX ROUND): no re-architecting, no adjacent refactors, no scope the findings did not
  raise. Scope added mid-round is scope the review never saw, so it restarts the loop it was
  meant to close.
- The supervisor owns the remote; a push invalidates the review marker, keyed to headRefOid.
- `gemini-resolver-mcp`: `pytest` is hermetic by default and its conftest FAILS the run on any
  outbound attempt. Never set `GEMINI_LIVE_TESTS=1` — that opts `tests/test_smoke.py` back into
  real Gemini calls against a 1500/day shared quota. (`SKIP_NETWORK` is dead; it gated nothing
  and a plain `pytest` used to spend.)
- `git add -- <paths>`, never `-A`/`-u`/`.`.
- Scarce resource ($/GPU/quota) -> gate + fail-closed cap + burn alert BEFORE depletion. A
  "calls>0 AND cost=$0" check is a corpse-detector, not an alert.

FAILURE MODES -> THE CHECK
- A fix that reopens the hole it closed. Check: name the adversarial case it now admits, answer
  it in the same commit — a narrowed guard reads as a fix and passes the same tests.
- A guard test that cannot fail. Check: step 5, no exceptions.
- Fixing a finding that is itself wrong. Check: step 2. A headline figure quoted without its
  run-to-run spread can sit entirely inside the noise floor; when it does, keep the decision and
  rewrite the claim rather than changing code to chase it.
- Junk cleaned at the destination instead of the source. Check: name where it is BORN, reject it
  there; destination gates are defence in depth only.

--- FINDINGS ---  [FIX ROUND only]
{paste the review findings verbatim, with severities}
```

## Notes for the supervisor

- Paste findings VERBATIM. Summarising re-introduces the supervisor's own compression errors —
  exactly what step 2 exists to catch.
- Give the CURRENT head sha. Without it the agent has to infer which branch state it is on, and
  spends its first tool calls guessing between a stale worktree and the live branch.
- Every measured number in the brief is a hypothesis; an agent correcting one is the round's
  value, not a deviation.
- Convergence is the goal, not one-shot. Several rounds on one PR is normal and not a failed
  dispatch; each round should shrink the findings list, and that is the signal to watch.
- Long output to `/tmp/sentinel-remediation/{slug}/`, not the report.
- If per-agent devcontainer ownership lands (compose project keyed to the launching worktree),
  the no-worktree rule relaxes to "confirm the workspace check passed" and compiles parallelise.
  Until then, sequencing is the only mutual exclusion there is.
