# Claude Code Hooks

Context-aware hooks that inject patterns when working on specific file types.

## Active Hooks

| Hook | Trigger | Purpose |
|------|---------|---------|
| `testing-context.sh` | Edit/Write `*Tests.cs`, `*_test.*`, `test_*.*` | Inject AAA, naming, assertions patterns + outbound-boundary guard-test contract |
| `benchmark-context.sh` | Edit/Write `*Benchmark*.cs`, `*_bench.*` | Inject BenchmarkDotNet, Release mode |
| `observability-context.sh` | Edit/Write `*Service.cs`, `*Worker.cs`, `*Repository.cs`, `*Telemetry*.cs` | Inject OTEL/Serilog patterns |
| `git-push-guard.sh` | Bash: `git push` and `gh pr merge` in leading-token position | **BLOCK** - requires a tests-passed marker to push, a recorded review VERDICT to merge. Matches the command STRING, so it raises the cost of an unreviewed merge; it does not prevent one — and only these two spellings are matched (see [KNOWN GAP: entry spellings](#known-gap-entry-spellings)) |
| `dotnet-guard.sh` | Bash `dotnet build/test/run/...` | **BLOCK** - use `{Project}/.devcontainer/compile.sh` |
| `node-guard.sh` | Bash `npm/npx/yarn/pnpm/wrangler` | **BLOCK** - run node tooling in a container |
| `pr-review-marker.sh` | Skill `review-pr` (PostToolUse) | **RECORD** - writes `pr-review-pending-<N>` only. It fires at INVOCATION and cannot see the review's findings, so it never records a verdict |
| `ef-migration-guard.sh` | Write `Migrations/*_*.cs` | **BLOCK** - prevents manual migration creation |
| `ansible-gate-guard.sh` | Edit/Write on a deployment/CI gate file. **Not Bash** — a `sed -i` on the same file is not seen | **BLOCK** - was ASK until 2026-08-06, which is inert here (see [`ask` is inert](#ask-is-inert-on-this-host)). Escape for legitimate gate work: `touch .claude/.ansible-gate-confirmed` |
| `deploy-smoke-reminder.sh` | Bash deploy/restart commands (PostToolUse) | **ADVISE** - inject smoke-test reminder |
| `memory-density-guard.sh` | Write/Edit `*/projects/*/memory/*.md` (PostToolUse) | **ADVISE** - nudge when MEMORY.md hook line or memory-file description violates the density bar. Scope is the `projects/*/memory/` corpus specifically, not any directory named `memory` |
| `design-intent-dispatch-guard.sh` | Agent dispatch with impl-shaped prompt | **BLOCK** - requires a DESIGN INTENT stanza in the brief. Presence-only and content-agnostic, with one structural exception: an EMPTY label is denied. The stanza is the rest of the label's line plus following lines up to the first BLANK line, so `DESIGN INTENT:` followed by a blank line and then the brief body is an empty label, not a stanza. The phrase in prose, in a filename, or as a bare substring (`redesign intentionally`) still satisfies the gate — KNOWN GAP, see the hook header. Harness-shape anomaly -> open + loud; jq missing -> degraded raw-grep, verdict-identical to the jq path |
| `service-decisions-context.sh` | Edit/Write `<Service>/src/**` | **ADVISE** - inject the service card's DECISIONS block (skipped on `DECISIONS: none`). Neutral (empty output) on no-op; anomalies neutral + loud |
| `plan-retirement-guard.sh` | Bash `git rm` of `docs/proposals/**`, `*PLAN*.md`, `*-design.md` | **ADVISE** - injects the migration checklist via `additionalContext`. Deliberately not a block: a script cannot verify migration semantics, and PHASE_TAGS step 4 makes `git rm` of a plan doc a normal step. Was ASK until 2026-08-06, so the checklist was never actually shown (see [`ask` is inert](#ask-is-inert-on-this-host)). Scope is `git rm` only: plain `rm` is silent (KNOWN GAP, asserted by `run-intent-fidelity-smoke.sh`) even though `rm` + `git commit -a` orphans the decisions identically. Anomaly -> open + loud; jq missing -> degraded raw-grep, still advises on match |

## Failure Direction (intent-fidelity hooks)

Infrastructure failure inside a hook must pick a direction per hook level,
and must never die silently — every anomaly path emits a stderr line naming
the hook and the reason (`[hook-name] ANOMALY: <reason> — failing open` /
`[hook-name] DEGRADED: ...`).

- **BLOCK** (`design-intent-dispatch-guard.sh`): harness-shape drift (empty
  stdin, non-JSON, missing `.tool_input.prompt`) fails **open + loud** —
  harness drift must not brick dispatching. Losing jq fails **closed where
  scoped**: a degraded raw-stdin grep still denies stanza-less impl briefs.
- **ADVISE** (`plan-retirement-guard.sh`): all infra failures fail **open +
  loud** (this hook runs on every Bash call). jq-less degraded mode raw-greps
  stdin and still emits the checklist on a plan-doc match — it never falls back
  to advising on everything. It returns `allow` + `additionalContext`, never a
  blocking decision.
- **ADVISE** (`service-decisions-context.sh`): no-op paths emit **nothing**
  (empty stdout, exit 0) so the permission flow is untouched — an active
  `allow` would widen permissions. The injection path emits
  `additionalContext` without a `permissionDecision`. Infra failures
  (jq missing, unreadable card, awk failure) are neutral + loud.

**Harness timeout caveat**: if the harness kills a hook on timeout, that is
fail-open by harness semantics — a killed hook emits no decision. Mitigated
by keeping these hooks subsecond (no network, no repo scans) and by the live
smoke suite (`test/run-intent-fidelity-smoke.sh`), which exercises the jq,
degraded and anomaly paths of the intent-fidelity hooks — not "every path" of
every hook, as this line previously claimed. See
[Guard test suites](#guard-test-suites) for what is and is not covered.

## How It Works

1. Claude calls a tool (Edit, Write, Bash, etc.)
2. `PreToolUse` hook runs before execution
3. Hook checks input and decides: allow/deny/modify
4. If denied: tool execution blocked with message
5. If allowed: tool executes normally

## KNOWN GAP: entry spellings

`git-push-guard.sh` decides from the command STRING the Bash tool was asked to
run. It never observes the act. **It raises the cost of an unreviewed push or
merge; it does not prevent one**, and nothing else in this README should be
read as claiming otherwise. There is also no permission layer behind it:
`~/.claude/settings.json` sets `"defaultMode": "bypassPermissions"` and allows
`Bash(git:*)`, so this hook is the only gate.

Matching is by leading token, so only `git push` and `gh pr merge` are caught.
These all reach the act **ungated** (each verified against this hook):

| Spelling | Why it misses |
|---|---|
| `git -C <dir> push` | a global option sits between `git` and `push` |
| `gh -R o/r pr merge <N>` | a global option sits between `gh` and `pr` |
| `gh api …/pulls/N/merge` | merge by REST path, not by subcommand |
| `curl -X PUT …/pulls/N/merge` | same act, different client |
| `gh api graphql … mergePullRequest` | same act, GraphQL mutation |

`git -C <dir> …` is the shape `.claude/skills/supervisor-mode/SKILL.md` tells
every supervisor to use, so the gap covers the most-used git form in this repo.
**This is unchanged from `main`** — closing it means matching the ACT across
global-flag prefixes and alternate transports inside a command string, which is
command-string parsing and is deliberately not in this change.

## A right answer for the wrong reason

**A right answer for the wrong reason is the dominant failure mode here** — it
has twice nearly shipped as a passing test:

- `json_ok` was added in response to an invalid-JSON bug and then fed nine
  static-text cases, none of which could express that bug.
- `scripts/claude-pr-verdict` shipped non-executable while every test invoked it
  as `bash <path>` — the one spelling that does not need the bit.

The habit that catches both: **mutate the fix and confirm the test goes red**,
and when a test asserts a refusal, assert *which* refusal. A guard test that
cannot distinguish "blocked for the reason under test" from "blocked for an
unrelated reason" is pinning nothing.

## KNOWN GAP: MCP write paths

`PreToolUse` fires for MCP tools, but a matcher made only of
`[A-Za-z0-9_\- ,|]` is compared as an exact name or a `|`-separated list, so
`Bash` can never match an MCP tool name. (Anything outside that character set
is treated as an unanchored regex — which is why the `mcp__.*` fix below works.
An earlier revision said matchers are always exact-compared, then proposed a
regex matcher two sentences later.) No hook block in any settings file carries
an `mcp__` matcher. These are in every dispatched agent's tool set and reach the outcomes
this layer guards with **no marker consulted**:

| Tool | Effect |
|------|--------|
| `mcp__plugin_github_github__merge_pull_request` | merges — no verdict marker |
| `mcp__plugin_github_github__push_files` | writes to main — no PR, no marker |
| `mcp__plugin_github_github__create_or_update_file` | writes to main — no PR, no marker |
| `mcp__plugin_github_github__delete_file` | commits a deletion straight to main |
| `mcp__plugin_github_github__pull_request_review_write` | posts a GitHub APPROVE a human may read as the verdict |
| `mcp__plugin_github_github__update_pull_request` | can retarget the base branch of an open PR |

The Bash-side merge routes are **not** all gated either — only leading-token
`gh pr merge` is; see [KNOWN GAP: entry spellings](#known-gap-entry-spellings).
So MCP is one of several ungated routes to a merge, not the sole one.

**Fix shape** (not built here): a second `PreToolUse` entry with the regex
matcher `mcp__.*`, routed to a sibling hook that reads the structured
`.tool_input` (owner/repo/pullNumber/branch) instead of `.tool_input.command`.
It is recorded rather than built because the deny-binds-for-MCP interaction has
not been exercised on this host, and a gate whose deny may not bind would
advertise protection that does not exist.

## Git Push Guard

**Purpose**: Prevent pushing code without running tests first.

**Behavior**: Blocks any `git push` command unless a tests-passed marker for
the current working tree exists. The marker is **tree-hash-keyed** (stores
the tree hash of the most recent commit — committed content only, uncommitted
edits do not change the tree hash), so it survives:

- `compile.sh` -> `git commit` (commit hash changes, tree unchanged)
- `git cherry-pick` / `git rebase` of a tested commit (tree preserved)
- Unrelated commits (e.g. STATE.md edits) that don't change source content

Different tree content -> different tree hash -> marker mismatch -> re-test
required. This is the safety property: untested source changes are blocked,
but workflow churn that doesn't change source content is not.

**Caveat (HEAD^{tree} is committed-only)**: the tree hash is computed from
`HEAD^{tree}`, which reflects the tree of the most recent commit. Uncommitted
edits in the working directory do NOT change the tree hash. If you run
`compile.sh` with a dirty working tree, the marker records HEAD's tree — not
the actually-tested content. `mark-tests-passed.sh` emits a stderr warning
when it detects a dirty working tree at marker-write time; commit before
relying on the marker.

**Marker scope (write=per-worktree, read=global)**: the marker file is
written with a suffix derived from the worktree toplevel path
(`sha1(git rev-parse --show-toplevel)`), so two worktrees writing markers
cannot overwrite each other. Path:
`/tmp/atlas-test-markers/tests-passed.<12-hex>`. However, the push guard
READS by scanning every marker file in that directory and matching by
tree-hash — so a marker written by any worktree (including an agent
worktree that has since been torn down) satisfies the gate for any
subsequent push of the same tree content. Note: `/tmp` is cleared on
reboot — markers need to be regenerated after host reboot (this is
intended; re-running tests after a reboot is cheap insurance).

**Marker format (v2)**: the marker is a single line
`v2 tree <tree_hash> <iso8601_timestamp>`. The push guard rejects any other
format (empty file, partial write, pre-v2 commit-hash-keyed marker) with a
"regenerate" message rather than silently accepting — pre-v2 markers were
40-char hex strings indistinguishable from tree hashes by shape.

Also enforces:
- No direct push to `main` / `master` (refspec form too)
- No deletion of the `main` / `master` remote branch
- Docs/config-only pushes are exempt from the marker requirement, keyed by
  EXTENSION and never by an open tree prefix: `CLAUDE.md`, any `*.md`,
  `.gitignore`, `.editorconfig`, `.claude/**.json`, `.claude/.gitignore`, and
  `deployment/artifacts/monitoring/**.{json,yml,yaml}`. Executables under those
  trees are NOT exempt. Until 2026-08-06 this read `.claude/**`, which exempted
  every tracked shell script under `.claude/` — every hook in this gating
  layer, `git-push-guard.sh` included — so a push that changed nothing but the
  gates skipped verification entirely. Pinned by
  `test/run-push-exemption-smoke.sh`.
- `gh pr merge` requires a `pr-reviewed-{N}` marker carrying an explicit
  **verdict** — see "PR Review Verdict Gate" below.

**Configuration**: tracked `.claude/settings.json` (also still registered in
the gitignored `settings.local.json` — see
[Migration status](#migration-status-registrations-are-duplicated-on-purpose))
```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "/home/james/ATLAS/.claude/hooks/git-push-guard.sh",
            "timeout": 5
          }
        ]
      }
    ]
  }
}
```

## PR Review Verdict Gate

**Purpose**: `gh pr merge` must be gated on what the review CONCLUDED, not on
the fact that a review was started.

**The defect this replaces (fixed 2026-08-06)**: `pr-review-marker.sh` is a
`PostToolUse(Skill)` hook. It fires the instant `review-pr` is invoked — the
Skill tool's response is the review PROMPT being loaded, and the analysis is
performed afterwards by Task agents. No hook event in the harness ever carries
the review's conclusions. The hook nonetheless wrote `pr-reviewed-<N>`
unconditionally, and the merge guard read that file as "reviewed and approved".
A DO-NOT-MERGE review therefore left a fresh, passing marker behind and the
merge was unblocked. Observed live on PRs #908, #911 and #913; #913's marker
matched its live `headRefOid` exactly. Nothing shipped only because a human read
the verdict text.

**Mechanism (pending + explicit promotion)**:

1. `pr-review-marker.sh` writes `pr-review-pending-<N>` = `v1 pending <sha> <ts>`.
   That is all it can honestly attest: *review-pr was invoked while the head was
   `<sha>`*. It never writes `pr-reviewed-<N>`.
2. The reviewer reads the findings and records the outcome deliberately:
   ```
   scripts/claude-pr-verdict <N> approve "<what was verified>"
   scripts/claude-pr-verdict <N> block   "<the blocking finding>"
   ```
   Preconditions, all fail-closed: a pending marker must exist; its SHA must
   equal the PR's *current* `headRefOid`; at least 60s must have elapsed since
   invocation (the toolkit fans out to several agents and never returns faster,
   so an invoke-then-stamp is mechanically refused); the reason must be
   at least 20 characters. Every verdict is appended to
   `~/.claude/atlas-pr-verdict.log` (override with `ATLAS_PR_VERDICT_LOG`).
   The log is deliberately NOT worktree-relative: it used to be, and gitignored,
   so a background agent's approval record died with its throwaway worktree —
   the least supervised approvals were the ones that left no trace.

   **These four checks are friction, not control.** Read them as raising the
   cost of an absent-minded approval, never as evidence that a review happened.
   Each was defeated deliberately during the review of PR #918: the pending
   record is an ordinary `/tmp` file that a single `Write` forges (no hook
   covers `Write` to `/tmp`); backdating it defeats the elapsed-time check —
   it was backdated and a verdict recorded with twenty `a`s as the reason,
   zero review performed; the reason check measures LENGTH, not content; and
   nothing stops one agent being both reviewer and approver, since there is no
   second party anywhere in this flow. What they genuinely buy is that approval
   is a deliberate, logged, separately-typed act rather than a side effect of
   invoking the review skill — which is the specific failure that let #908,
   #911 and #913 merge. That is worth keeping, and it is not an assurance that
   anyone read the code.
3. `git-push-guard.sh` allows the merge only for
   `v2 approved <sha> <ts> <reason>` where `<sha>` is the current head. Everything
   else denies: missing file, empty file, `v2 blocked`, a stale SHA, a
   pre-2026-08-06 `<sha> <ts>` marker, or a `gh` failure that leaves the head
   unconfirmable. **A review that states no verdict leaves the merge blocked.**

Pre-2026-08-06 markers are deliberately rejected rather than migrated — they
attest only invocation, which is exactly the claim being removed.

**Guard test**: `test/run-pr-verdict-smoke.sh` (stubs `gh`, uses PR numbers in
the 999xx range, cleans up on exit). TEST 3 plants the exact legacy marker shape
and asserts the merge is DENIED — restore the old "marker exists -> allow" logic
and it goes RED.

## Adding New Hooks

1. Create script in `.claude/hooks/`
2. **Make it executable**: `chmod +x script.sh`
3. Read `tool_input` from stdin JSON
4. Output JSON with `hookSpecificOutput`:
   - `permissionDecision`: "allow" | "deny"
   - `permissionDecisionReason`: message shown to Claude
   - `additionalContext`: extra context for allowed operations
5. Register it in the TRACKED `.claude/settings.json` hooks block, as a
   `$CLAUDE_PROJECT_DIR`-relative path — never `settings.local.json` (gitignored,
   so the registration would be invisible to review) and never an absolute path
   (wrong for any other clone). Then add a `wired` assertion for it in
   `test/run-wiring-smoke.sh`.

**Important**: Scripts must have execute permission. Without it, you'll see
`PreToolUse:Bash hook error` on every command but tools will still run.

## Hook Output Format

```json
{
  "hookSpecificOutput": {
    "hookEventName": "PreToolUse",
    "permissionDecision": "deny",
    "permissionDecisionReason": "Reason shown to Claude"
  }
}
```

## Exit Codes

- `0`: Continue (allow or with modifications)
- `2`: Block command, show stderr to Claude

## EF Migration Guard

**Purpose**: Prevent manual creation of EF migration files.

**Problem**: Creating migration `.cs` files manually (without `dotnet ef migrations add`)
produces incomplete migrations missing the required `Designer.cs` file. EF Core then:
1. Records the migration in `__EFMigrationsHistory`
2. But does NOT apply the schema changes
3. Runtime errors like "column X does not exist"

**Correct Process**:
```bash
nerdctl compose exec -T {svc}-dev dotnet ef migrations add {Name} --project src/Data
```

**Blocked Pattern**: Any Write to `Migrations/[timestamp]_[Name].cs` that isn't
a Designer.cs or ModelSnapshot.cs. `[Name]` is `[A-Za-z0-9_]+` — it was
letters-only until 2026-08-06, which silently exempted every generated name
containing a digit or an underscore (e.g. the repo's own
`20260526195853_Phase5SentinelMatrixColumns.cs`).

**Not covered**: this is a `PreToolUse(Write)` hook, so a migration created by
a Bash heredoc or `cat >` never reaches it. The gate is a guardrail against the
easy mistake, not a seal.

**Guard test**: `test/run-command-guards-smoke.sh` (also covers `dotnet-guard.sh`
and `node-guard.sh`, both of which matched only at the start of the command
string until 2026-08-06 — `cd /x && dotnet test` and `cd /x && npm ci` walked
straight past them. Both now match after a shell operator too, and both stand
down when the command already goes through nerdctl/docker/podman, which is the
behaviour they exist to encourage).

## Ansible Gate Guard

**Purpose**: Per `ARCH_PREF` — edits to deployment/CI gate files should prompt
for explicit intent so gates are fixed at the root, not worked around.

**Matched paths**: `deployment/inventory/*`, `playbooks/*`, any
`.devcontainer/compile.sh`, any `.claude/hooks/**.sh` (the whole gating layer
and its own guard tests), `.claude/settings.json` / `settings.local.json`
(where the hooks are wired — un-wiring one there is the most complete bypass
available), and the two marker-writing tools `scripts/claude-mark-verified` and
`scripts/claude-pr-verdict`.

Until 2026-08-06 the list named only `git-push-guard.sh`, so every other file
that can weaken the gates — the other hooks, the settings that wire them, the
scripts that write the markers they read — was editable with no prompt at all
(verified).

**Decision**: `deny`. Fail-CLOSED — missing jq denies, because a guard that
cannot evaluate the gate cannot know the write is safe. (This section said
`ask` and fail-open until 2026-08-06; both were wrong, and the inverted failure
direction was the dangerous half. `ask` is inert here — see
[`ask` is inert](#ask-is-inert-on-this-host).)

**KNOWN GAP: this is wired at `Edit|Write` only, so Bash reaches the same
files.** `sed -i s/deny/allow/ .claude/settings.json` — the single most complete
way to unwire every gate in this layer — is not seen by this hook (verified). A
guard on the Edit tool is not a guard on the file. Closing it requires deciding,
from a command STRING, which token is a write target and which is merely the
file a runner runs; that is command-string parsing and is deliberately not in
this change, so that a parser regression cannot take this Edit|Write
enforcement down with it.

**Session bypass**: `touch $CLAUDE_PROJECT_DIR/.claude/.ansible-gate-confirmed`
disables the guard for **every** gate file. It now EXPIRES after 4h
(`ATLAS_GATE_BYPASS_TTL`) and warns when a command actually touches a gate
path — not on every tool call, which briefly made this hook emit an
unconditional `allow` for unrelated commands like `ls -la /etc`, speaking
authoritatively about calls it has no opinion on. The warning arrives via
`systemMessage` **and** `additionalContext`, so the warning reaches the user and
the model rather than only the debug log, which is all `echo >&2` from a hook
exiting 0 ever reached. Expiry is announced rather than silently honoured.
It is gitignored — committing
it would ship the gating layer switched off. Creating it is deliberately *not*
gated: this hook's own deny message instructs it, and gating the documented
remedy is the same self-contradiction as blocking the build it demands.

## Deploy Smoke Reminder

**Purpose**: Per `VERIFY_TEST` — every deploy must be followed by an
(unsolicited) smoke test. This `PostToolUse` hook injects a reminder into
Claude's context whenever a real deploy/restart command completes.

**Matched commands** — the executable token after any `sudo`, `env` or
`VAR=value` prefixes, basenamed (so `/usr/local/bin/nerdctl`, the form ansible
invokes, matches); then the first non-flag word, skipping flags and their
values (so `compose -f <file> up -d` matches, which is what `deploy.yml:485,490`
actually runs):
- `ansible-playbook ...deploy.yml`
- `nerdctl|docker|podman compose up|restart|start`
- `nerdctl|docker|podman restart|start|rm` — the surgical-deploy shapes
- `compose up|restart|start`
- `systemctl restart|start`

Not matched, deliberately: `compose build`, `compose config`, `ps`, `logs`.

Substring matching on the full command was deliberately avoided to prevent
false positives from `echo`, `grep`, `git log` etc.

**Guard test**: `test/run-advisory-guards-smoke.sh` (also covers
`ansible-gate-guard.sh` and `plan-retirement-guard.sh`).

## Settings resolution and hook merge semantics

Point 1 is **documented**; points 2 and 3 are **our own empirical findings** —
do not cite the docs for those. (An earlier revision of this section claimed all
three were undocumented. That was wrong about the first.)

1. **Hooks MERGE across settings levels rather than replacing each other** —
   user, project and local settings each add their own hooks. Documented, and
   confirmed here: same event (`PreToolUse`), two files, both fire —
   `design-intent-dispatch-guard.sh` (tracked `settings.json`, `[Agent]`)
   blocked a dispatch in the same session that `git-push-guard.sh`
   (`settings.local.json`, `[Bash]`) was gating pushes.
2. **A `deny` from the tracked file is honoured** under `bypassPermissions`
   (empirical — the docs cover ask *rules*, not hook-returned decisions).
3. **Settings resolve from the main project directory, not from a git worktree**
   (empirical).
   An agent running in `.claude/worktrees/agent-X/` — which contains no
   `settings.local.json` at all — was still blocked by `dotnet-guard.sh`, which
   is registered *only* in `/home/james/ATLAS/.claude/settings.local.json`.
   So a hook registration edited on a feature branch is **not live** until it
   merges to the main checkout. This is the trap that makes "I changed the
   wiring and re-probed" prove nothing from inside a worktree.

### Migration status: registrations are duplicated, on purpose

`git-push-guard`, `dotnet-guard`, `node-guard` and `ef-migration-guard` are
registered in **both** the tracked `settings.json` (added here) and the
gitignored `settings.local.json` (pre-existing). That duplication is deliberate
and safe — these are pure decision functions with no side effects, a repeated
deny is still a deny, and `pr-review-marker.sh` has been double-registered all
along with no ill effect.

**Do not delete the `hooks` block from `settings.local.json` until this branch
is merged to main.** Per finding 3, the tracked registrations are not live on
the host until then; deleting the local copy first would leave the machine with
**no push guard, no dotnet guard, no node guard and no EF-migration guard** —
all four, for every agent. After the
merge, delete that block (keep `permissions`) and confirm with `dotnet vstest`
— it must still be blocked.

## Guard test suites

| Suite | Covers |
|-------|--------|
| `test/run-wiring-smoke.sh` | every hook is registered in the TRACKED `settings.json`, the registered set has not drifted, and the marker writers are executable in the index. Static — safe beside live agents |
| `test/run-push-guard-smoke.sh` | `git-push-guard.sh` marker lookup: global tree-hash scan, orphaned markers, block diagnostics. Runs against an isolated `ATLAS_MARKER_DIR` |
| `test/run-push-exemption-smoke.sh` | the docs-only / docs-config allowlists on the push path |
| `test/run-pr-verdict-smoke.sh` | `claude-pr-verdict` preconditions and the merge gate's verdict parsing (stubs `gh`, uses PR numbers 99901-99903) |
| `test/run-command-guards-smoke.sh` | `ef-migration-guard.sh`, `dotnet-guard.sh`, `node-guard.sh` |
| `test/run-advisory-guards-smoke.sh` | `ansible-gate-guard.sh`, `plan-retirement-guard.sh`, `deploy-smoke-reminder.sh` |
| `test/run-intent-fidelity-smoke.sh` | `design-intent-dispatch-guard.sh`, `service-decisions-context.sh`, `plan-retirement-guard.sh`, `testing-context.sh`, incl. degraded (jq-less) parity |
| `test/run-memory-density-smoke.sh` | `memory-density-guard.sh` |

Run them all with `for f in .claude/hooks/test/run-*.sh; do bash "$f"; done`.

**KNOWN GAP — nothing runs these automatically.** `.github/workflows/` contains
only `sync-docs.yml`; there is no CI job, no pre-commit hook and no scheduled
task that executes any suite. They run when a human or an agent remembers to
run them. This materially weakens the practice of pinning a KNOWN GAP as a test
assertion: an unrun test pins nothing, and a guard regression stays green until
someone looks. Wiring these into CI is the single highest-value follow-up in
this layer.

## `ask` is inert on this host

**Established 2026-08-06 by live probe — our own finding, not documented
behaviour.** `~/.claude/settings.json` sets `"defaultMode": "bypassPermissions"`.
A `permissionDecision: "ask"` is never surfaced under it; a `deny` is honoured
normally, from either settings file.

Evidence, with the invalid probes excluded:

| Probe | Valid? | Result |
|-------|--------|--------|
| Write to a path matching `ansible-gate-guard`'s live gate set (`*/playbooks/*`), no bypass file present; hook fed the harness's exact JSON returns `ask` | **yes** | no prompt, write completed |
| `git rm docs/proposals/…` — `plan-retirement-guard` live-wired at `PreToolUse:Bash`, script identical, returns `ask` | **yes** | no prompt, command ran |
| `rm -f .claude/hooks/…` expecting `ansible-gate-guard` | **no** — it is wired at `Edit\|Write`, not `Bash`; a Bash command can never reach it | silent, proves nothing |
| `nerdctl restart …` expecting `deploy-smoke-reminder` | **no** — the live script matches only `compose up`; `restart` handling is unmerged | silent, proves nothing |

The two invalid probes are recorded because they are the trap: *a matcher that
never matched* and *a decision the harness ignored* look identical from outside.
Only the first two probes are evidence.

**Consequence, and why this mattered more than a wiring detail**:
`ansible-gate-guard` is the mechanical enforcement behind CLAUDE.md's
DEPLOYMENT HARD_STOP. While it returned `ask` it enforced **nothing** — the hook
fired, its unit tests passed, and no edit was ever stopped. This is the class of
bug where the guard exists, the test is green, and nothing is protected.

**Both `ask` hooks were converted**, and *not* both to the same thing:

- `ansible-gate-guard.sh` -> **`deny`**. It gates HARD_STOPs; blocking is the
  honest semantics. Escape hatch unchanged and now load-bearing:
  `touch $CLAUDE_PROJECT_DIR/.claude/.ansible-gate-confirmed`.
- `plan-retirement-guard.sh` -> **`allow` + `additionalContext`**, *not* `deny`.
  Its header states "Deliberately ASK, not BLOCK: a script cannot verify
  migration SEMANTICS", and CLAUDE.md PHASE_TAGS step 4 makes `git rm` of a
  plan doc a required routine step with no escape hatch. Converting it to `deny`
  would contradict a documented decision and break the workflow. Its purpose is
  to *show a checklist*, so it now uses the delivery mechanism that demonstrably
  works (the same one `testing-context.sh` uses).

Pinned by `run-advisory-guards-smoke.sh` -> "DECISION-VALUE PIN": a static check
that no hook emits `"permissionDecision": "ask"`, plus a per-guard dynamic pin
of the expected decision. Mutation-verified — reverting either hook to `ask`
turns the suite RED.

**Related pin, from a real bug this caught**: every hook's output must be
parseable JSON. `ansible-gate-guard`'s new deny message embedded
`\$CLAUDE_PROJECT_DIR` via a heredoc and emitted a literal `\$`, which is not a
valid JSON escape. It looked fine under `cat` and failed the moment anything
parsed it — an unparseable deny is an absent deny, failing exactly as silently
as the `ask` did.
