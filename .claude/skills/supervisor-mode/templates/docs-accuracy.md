# Template: docs / card / memory accuracy audit

Factual accuracy of the docs agents read first: CLAUDE.md, the skills, the cards, the hooks, the
memory index. Two runnable audit scripts cover most of the surface, and a brief that does not
name them gets a hand-rolled grep sweep instead — which is what step 1 exists to prevent.

```
AUDIT AND FIX — factual accuracy of {target}. Branch `docs/{slug}` off main (`{sha}`), own git
worktree (isolation set). Commit per layer. Do NOT push, merge, or deploy.

DESIGN INTENT: none — documentation accuracy. No production code, no behaviour change.
supersedes: none
guard_tests: none — but every path, script, command and skill name you WRITE must be verified to
exist. A doc naming a script that is not there is worse than no doc.

TRAJECTORY
1. RUN THE AUDIT SCRIPTS FIRST — sub-second, file-level findings. Invoke each with `bash`, one
   per command (`bash a.sh b.sh` runs only the first). A script stored 100644 dies with
   `Permission denied` in a worktree or fresh clone, and `core.fileMode=false` hides that in the
   main checkout; `bash` is correct either way. Committed mode: `git ls-files -s <script>` —
   on 2026-08-06 `readme-consistency`'s two were 100644, `architecture-cards`' 100755.
   - `bash .claude/skills/architecture-cards/scripts/enumerate-services.sh` -> one service dir
     per line (source of truth is CLAUDE.md `## SERVICES`, not a glob), then
     `bash .claude/skills/architecture-cards/scripts/audit.sh <ServiceDir>` per service.
   - same pair under `readme-consistency/scripts/`: `enumerate-projects.sh`, then `audit.sh`.
   Exit 0 clean / 1 with findings, ending `summary: critical=N high=N medium=N low=N`. Exit 1 is
   FINDINGS, not an error — do not let `set -e` swallow the run.
   THE AUDITS CAN REPORT A PHANTOM FINDING: a lone `missing_block` HIGH naming a block the card
   plainly contains, gone on re-run. MECHANISM UNCONFIRMED — the block test is a plain `cat` into
   `grep` with no race, so the leading explanation is the card being rewritten underneath the
   read by another agent or a branch switch. The rate is not stable (2 of 25 runs one day, 0 of
   25 the next), which is why no trigger is wired here. Re-run any lone `missing_block` before
   acting — one that does not reproduce is absent — and never audit a card someone is editing.
2. For memory files, invoke the `compact-memories` skill; do not sweep by hand.
3. VERIFY EACH FINDING AGAINST THE CODE — the audits are grep heuristics. A missing
   `GUARD <class.method> @ file:line` on a D-entry may mean the citation is absent, or the guard
   is. Those need opposite fixes.
4. For every factual claim you touch: does the cited path/file:line exist and say that? Run the
   command, `ls` the path, open the line. Nothing turns red when a citation rots — line numbers
   drift with every edit above them, and a moved path leaves the reference syntactically intact.
5. If the doc is right and the CODE is wrong, say so and STOP — never edit the doc to match
   broken code. An incomplete D-entry atomic set (entry + `INTENT(D-n):` comment + guard + guard
   test) is a code finding to report, not a doc bug to paper over.
6. Changed an audit script? Run its tests, each `bash`-invoked:
   `.claude/skills/architecture-cards/tests/test-audit.sh` and the same path under
   `readme-consistency/`. Baseline them BEFORE your edit and diff after — a suite can already be
   RED on main (readme-consistency was, one case `stale-readme: MEDIUM`, on 2026-08-07), and
   without a baseline you adopt someone else's failure as yours. Hook changes: the smokes under
   `.claude/hooks/test/`, same rules.
7. Commit per layer, `git add -- <paths>` explicitly. Then, as the FINAL step:
   `scripts/claude-mark-verified "<reason>"` — run it BARE.

CONSTRAINTS FOR THIS CLASS
- Contradicting a documented CLAUDE.md rule without a named supersession -> STOP and report.
- Cards are agent-facing: plain ASCII, one rule per line, negative space (does-NOT / on-miss /
  invariants) over restated catalogs. Unicode operators and snake_case compression cost the same
  tokens as plain words and break literal-string grep in hooks.
- Do not touch files another in-flight agent owns; the supervisor names them in the brief.

FAILURE MODES SEEN HERE -> THE CHECK
- `scripts/claude-mark-verified "…" | tail` suppresses the marker write and it never reaches the
  audit log, leaving a stale marker that blocks the NEXT push. Check: run it bare, then confirm the
  marker's tree hash equals `git rev-parse HEAD^{tree}`.
- A docs mark-verified clobbers a code branch's marker (one marker file per worktree). Check: push
  the code branch first, then mark the docs branch.
- Fixing a doc to match a heuristic that was wrong. Check: step 3.
```

## Notes for the supervisor

- Both `audit.sh` scripts return in under a second across the whole monorepo, and they find real
  work: unaddressed HIGH findings (missing GUARD/TEST citations on live D-entries) were
  outstanding on 2026-08-06. Run them for the current set rather than trusting that. The scripts
  are not the bottleneck — naming them in the brief is.

## Falsifier — check this template's own claim, then delete these lines

- CLAIM: naming a runnable tool in the brief is what gets it run. This class is the clean test,
  because it names `audit.sh` with NO gate behind it — unlike PR review, where the merge is
  blocked without a marker, so non-invocation was mechanically impossible and proves nothing.
- OBSERVE: across the next 5 docs-accuracy dispatches, count how many ran an `audit.sh` rather
  than hand-grepping.
- THRESHOLD: fewer than 4 of 5 -> naming is insufficient; the next iteration pastes a real run
  and its finding lines instead of a trajectory. 0 of 5 while PR review stays at 100% -> the
  gate was the cause all along, and the fix is gates, not templates.
- CHECK BY 2026-09-06, owner = whoever runs the supervisor loop. Still unchecked after that
  date means nobody measured it: record that outcome and delete this section either way.
- Docs/config-only pushes are exempt from the tests-passed marker per
  `.claude/hooks/README.md` Git Push Guard — but `scripts/claude-mark-verified` is still the honest
  record, and mixed doc+code branches are NOT exempt.
