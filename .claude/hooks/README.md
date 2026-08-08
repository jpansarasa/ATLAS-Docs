# Claude Code Hooks

Context-aware hooks that inject patterns when working on specific file types.

## Active Hooks

| Hook | Trigger | Purpose |
|------|---------|---------|
| `testing-context.sh` | Edit/Write `*Tests.cs`, `*_test.*`, `test_*.*` | Inject AAA, naming, assertions patterns + outbound-boundary guard-test contract |
| `benchmark-context.sh` | Edit/Write `*Benchmark*.cs`, `*_bench.*` | Inject BenchmarkDotNet, Release mode |
| `observability-context.sh` | Edit/Write `*Service.cs`, `*Worker.cs`, `*Repository.cs`, `*Telemetry*.cs` | Inject OTEL/Serilog patterns |
| `git-push-guard.sh` | Bash: any `git … push`, `gh … pr merge`, or `gh api …/pulls/N/merge` — including **any** dash-prefixed `git` global option before the subcommand (with or without its value), `gh -R`/`--repo` prefixes, command substitution, subshells and `sh -c` | **BLOCK** - requires a tests-passed marker to push, a recorded review VERDICT to merge. Matches the command STRING, so it raises the cost of an unreviewed merge; it does not prevent one (see [What the entry regexes cannot catch](#what-the-entry-regexes-cannot-catch)) |
| `dotnet-guard.sh` | Bash `dotnet build/test/run/...` | **BLOCK** - use `{Project}/.devcontainer/compile.sh` |
| `node-guard.sh` | Bash `npm/npx/yarn/pnpm/wrangler` | **BLOCK** - run node tooling in a container |
| `pr-review-marker.sh` | Skill `review-pr` (PostToolUse) | **RECORD** - writes `pr-review-pending-<N>` only. It fires at INVOCATION and cannot see the review's findings, so it never records a verdict |
| `ef-migration-guard.sh` | Write `Migrations/*_*.cs` | **BLOCK** - prevents manual migration creation |
| `ansible-gate-guard.sh` | Edit/Write on a deployment/CI gate file, **or** a Bash command that both names one and carries a write construct (`sed -i`, `>`, `rm`, `cp`, …) | **BLOCK** - was ASK until 2026-08-06, which is inert here (see [`ask` is inert](#ask-is-inert-on-this-host)). Reads stay frictionless. Escape for legitimate gate work: `touch .claude/.ansible-gate-confirmed` |
| `deploy-smoke-reminder.sh` | Bash deploy/restart commands (PostToolUse) | **ADVISE** - inject smoke-test reminder |
| `memory-density-guard.sh` | Write/Edit `*/projects/*/memory/*.md` (PostToolUse) | **ADVISE** - nudge when MEMORY.md hook line or memory-file description violates the density bar. Scope is the `projects/*/memory/` corpus specifically, not any directory named `memory` |
| `design-intent-dispatch-guard.sh` | Agent dispatch that WRITES to a service carrying D-entries | **BLOCK** - requires a DESIGN INTENT stanza in the brief. Fires only when all four hold: write-shaped judged AFTER prohibition clauses are stripped, a clause ending at `.`, `;`, or a comma that an instruction-verb list says begins a new instruction (a bare comma terminator was measured and rejected — it leaves `open a PR, or deploy` write-shaped); an agent type whose remit is writing code — the exempt types are ENUMERATED (`Explore`, `Plan`, `claude-code-guide`, `statusline-setup`, and the five `pr-review-toolkit` reviewers), never matched by namespace, because `pr-review-toolkit:code-simplifier` holds All tools and exists to MODIFY code, and compared ANCHORED so `Explorer` cannot inherit `Explore`'s exemption; no read-only self-declaration in the brief's opening 400 chars; and the brief names a service whose `AGENT_README.md` DECISIONS block has >=1 D-entry. Everything else passes silently — a stanza with no D-entries in scope is `none`, which carries nothing. Presence-only and content-agnostic, with one structural exception: an EMPTY label is denied. The stanza is the rest of the label's line plus following lines up to the first BLANK line, so `DESIGN INTENT:` followed by a blank line and then the brief body is an empty label, not a stanza. The phrase in prose, in a filename, or as a bare substring (`redesign intentionally`) still satisfies the gate — KNOWN GAP, see the hook header. Harness- or repo-shape anomaly -> open + loud; jq missing -> degraded raw-grep, verdict-identical to the jq path |
| `service-decisions-context.sh` | Edit/Write `<Service>/src/**` | **ADVISE** - inject the service card's DECISIONS block (skipped on `DECISIONS: none`). Neutral (empty output) on no-op; anomalies neutral + loud |
| `plan-retirement-guard.sh` | Bash `git rm` of `docs/proposals/**`, `*PLAN*.md`, `*-design.md` | **ADVISE** - injects the migration checklist via `additionalContext`. Deliberately not a block: a script cannot verify migration semantics, and PHASE_TAGS step 4 makes `git rm` of a plan doc a normal step. Was ASK until 2026-08-06, so the checklist was never actually shown (see [`ask` is inert](#ask-is-inert-on-this-host)). Scope is `git rm` only: plain `rm` is silent (KNOWN GAP, asserted by `run-intent-fidelity-smoke.sh`) even though `rm` + `git commit -a` orphans the decisions identically. Anomaly -> open + loud; jq missing -> degraded raw-grep, still advises on match |

## Failure Direction (intent-fidelity hooks)

Infrastructure failure inside a hook must pick a direction per hook level,
and must never die silently — every anomaly path emits a stderr line naming
the hook and the reason (`[hook-name] ANOMALY: <reason> — failing open` /
`[hook-name] DEGRADED: ...`).

- **BLOCK** (`design-intent-dispatch-guard.sh`): harness-shape drift (empty
  stdin, non-JSON, missing `.tool_input.prompt`) and repo-shape drift (no
  service card carrying a D-entry) fail **open + loud** — drift must not brick
  dispatching, and failing closed on an unreadable repo would gate every
  dispatch, which is the pathology the scope predicate exists to remove.
  Losing jq fails **closed where scoped**: a degraded raw-stdin grep still
  denies stanza-less in-scope impl briefs. The degraded path is restricted to
  bash/cat/grep/head/awk/sed, and the hook is written to stay inside that set
  (parameter expansion rather than `dirname`, regex rather than `tr`) because
  forking a binary it lacks would abort the gate with `command not found`
  instead of degrading it. This is a PRESERVED invariant, not a repaired bug:
  run under a PATH of exactly those six, the pre-rewrite hook already emitted
  zero `command not found` and denied correctly (verified 2026-08-08). Pinned
  by `run-intent-fidelity-smoke.sh` so a future edit cannot introduce one.
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

## What the entry regexes cannot catch

`git-push-guard.sh` decides from the command STRING the Bash tool was asked to
run. It never observes the act. **It raises the cost of an unreviewed push or
merge; it does not prevent one**, and nothing else in this README should be
read as claiming otherwise. There is also no permission layer behind it:
`~/.claude/settings.json` sets `"defaultMode": "bypassPermissions"` and allows
`Bash(git:*)`, so this hook is the only gate.

Caught (one case per form in `test/run-entry-shape-smoke.sh`): plain
`git push` / `gh pr merge`; `git` carrying **any** dash-prefixed global option
before the subcommand, in either the `--opt=value` or the `--opt value`
spelling, with the value quoted, backslash-escaped or bare;
`gh` with `-R`, `--repo`, `--hostname`; command substitution `$(…)`; subshells
`(…)`; nested shells `sh -c "…"`; and `gh api …/pulls/<N>/merge`.

The `git` option set is **not enumerated**, and that is the point — see
[Every dash token is a global option](#every-dash-token-is-a-global-option).

Not caught, and no regex over a command string can be:

- a tool that merges or writes with **no Bash command at all** — see the MCP
  known gap below; there is no string to match
- the act performed by something the command merely starts: a script file, a
  Makefile target, `xargs`, `find -exec`, a shell alias, a git alias, or a
  wrapper earlier on `$PATH`
- obfuscation of the literal tokens — `eval`, a variable holding the
  subcommand, base64
- a push from a non-Bash surface (an editor's VCS integration, a daemon)

**Accepted cost: writing ABOUT a gated act is gated.** Because the anchor is
gone, the literal tokens trip the gate wherever they appear — including in a
commit message or a review comment. **Pass such text by path:**

```sh
gh pr comment <N> --body-file findings.md
git commit -F message.txt
```

### The text-authoring exemption: tried, withdrawn, do not reintroduce

An exemption for text-authoring segments (`git commit -m …`, `gh pr comment …`)
existed for two rounds and caused a fail-OPEN regression in **each** of them:

| round | defect | effect |
|---|---|---|
| 4 | it suppressed a segment's text but not its **execution** | `echo "git push origin main" \| sh` allowed |
| 5 | making it quote-aware across newlines needed `RS="\0"`, and the splitter has no backslash-escape handling, so a leading segment with an **odd number of quotes** opened a quote that never closed and swallowed every following line | derived command came out **empty**; a following push to main, merge, `--delete main`, `curl -X PUT …/merge` and the graphql auto-merge **all allowed** (nine rows reproduced) |

Round 5's failure took out two routes closed in that same round. The exemption
is a **convenience**, and its cost is a component whose failure mode is silent
and total: reintroducing it means reintroducing a shell-quoting parser inside
this hook, and both attempts at that parser fail-OPENed the whole gate. The
question is not whether the parser can be made correct — it is whether saving
one flag is worth that. Pinned by inverted assertions in
`run-entry-shape-smoke.sh` plus the nine reproduced rows.

### Every dash token is a global option

The entry pattern used to **enumerate** git's global options. An enumeration of
someone else's vocabulary is a defect generator: every round closed the options
that had been noticed and left the rest, and an entry miss is the worst shape in
this file — the hook does not run, so **every** rule is skipped, not merely the
one an option would have confused.

Nine shapes were still open at `5b127704` — a tenth, an env-assignment
truncation, is described with the config table below. Each was reproduced
against a **local bare remote** with an honestly-earned marker for the feature
tree: the guard returned **allow** and git really wrote the remote's `main`.

| shape | why it escaped |
|---|---|
| `git -c user.name=A\ B push origin main` | a **backslash-escaped** space is the quoted-space case with no quote characters in it, so the value pattern ended at the backslash |
| `git -c "user.name=A⏎B" push origin main` | a newline **inside quotes** is one character to git and a line break to every line-oriented grep here |
| `git -p push …` | the short spelling of `--paginate`, whose long form *was* listed |
| `git --no-optional-locks push …` | never listed |
| `git --namespace=ns push …` | never listed |
| `git --icase-pathspecs` / `--glob-pathspecs` / `--noglob-pathspecs` | never listed (`--literal-pathspecs` was) |
| `git --attr-source=HEAD push …` | never listed |
| `git --namespace=ns -c remote.origin.push=HEAD:refs/heads/main push origin` | composed: the entry miss also hid the `-c` from the config-destination rule |
| `GIT_CONFIG_PARAMETERS="'remote.origin.push=…'" git push origin` | in neither the replicated set nor the refusal set — see the config table below |

`--namespace` deserves a note: in a **local** fixture the write lands under
`refs/namespaces/ns/refs/heads/main`, because a local transport spawns
`receive-pack` as a child that inherits `GIT_NAMESPACE`. A real server does not
inherit it. Modelled with
`--receive-pack='env -u GIT_NAMESPACE git-receive-pack'`, the same command writes
`refs/heads/main` outright (measured).

The alternation is gone. The pattern is **any dash-prefixed token, optionally
followed by one separate word**:

- the generic arm needs no maintenance when git grows an option, and cannot
  under-match a spelling nobody thought of;
- the **optional second word is load-bearing, not tidiness**: `-c`, `-C`,
  `--namespace`, `--git-dir` and `--attr-source` take their value as a separate
  argument, so dropping it would regress `git -c user.name=x push`, caught since
  2026-08-06.

**Measured cost — corrected 2026-08-08. The first account of this was wrong.**
A *valueless* global option can swallow a subcommand as if it were a value, so
`git --no-pager stash push` and `git --paginate subtree push` enter the gate.
That much held. The claim that they are "evaluated, not refused" and "pass on an
ordinary marker" did not: it is true only on a feature branch that **has** a
valid marker. Measured on a `main` checkout and on a marker-less feature branch,
every such pair **denies** — and on `main` the reason read *"Direct push to
main/master is not allowed"*, a false statement about a command that pushes
nothing. The claim that the one genuine new denial was `--config-env=` was wrong
for the same reason.

Neither side of that cross is a short list, either. **Any** dash-prefixed token
does it — `git --zzz stash push` denies, as do `-p`, `-P`, `--bare`,
`--paginate`, `--namespace=`, `--no-optional-locks`, `--git-dir=`, `--no-advice`,
`--html-path` and a bare `--` — and the subcommand side reaches past
`stash push` / `subtree push` to `remote push` and `notes push`. Enumerating
either is the mistake the generic arm removed.

The **verdict is kept**: this gate cannot know which global options take a
separate value, so it cannot rule the push out, and ambiguity resolves to DENY.
What changed is the **reason** — `deny()` now appends an `ON THE VERB:` note
naming the swallowed word and saying the `push` may belong to it. The
discriminator is the word's *shape* (a bare lowercase word), not a list of
subcommands, so the real separate-word values keep their message unqualified:
`-C /tmp push` (holds a slash) and `-c user.name=x push` (holds a dot and an
equals) are byte-for-byte unchanged. `git stash push` and `git -C <dir> stash
push`, the spellings used here, still do not match at all (`-C` consumes
`<dir>`).

Swept across 3,899 command strings under **two** branch conditions — feature
branch without a marker, and a `main` fixture — crossing env prefix ×
global-option form × separator (space/tab/CR) × destination × chaining, plus the
gh option/route cross, the composed `-c` config shapes, and the newline and
continuation shapes: **0 loosenings**, 288 tightenings per condition (7,798
evaluations). The harness is proved able to see a loosening by running it
**swapped** (new as baseline), where those 288 reappear as loosenings in both
conditions — a sweep that cannot report one is not evidence of anything, and an
earlier run of this harness reported a clean `0/0` only because a relative guard
path failed to open after the `cd`, so every cell read back "allow" on both
sides.

The composed `-c` axis is in that corpus because its **absence** let a real
loosening through: an earlier draft renamed the option-value constant without
repointing the `-c` replication grep, which silently dropped every `-c` and made
the config-destination rule answer from the repository's config. The sweep said
`0/0`; `run-entry-shape-smoke.sh`'s two "composed" rows caught it. A corpus that
contains no cell where the thing you changed decides the verdict cannot clear
you.

Per-call latency on the common path is **12 ms median** (n=400 interleaved A/B
over a `git status` / `ls` / `dotnet build` mix, p95 13 ms, mean 11.6 ms) —
unchanged by this round. The earlier "**11 ms**, unchanged" claim was measured
against a single `git status` payload, which is faster than the mix and rounds
down; the honest figure for the common path is 12 ms, and it has been 12 ms
since before that claim was written.

### `gh` had the same enumeration, one round later

The git side was made generic on 2026-08-08; `GH_GLOBAL_OPT_RE` beside it stayed
a three-spelling list (`-R`, `--repo`, `--hostname`) with an **unquoted** value
class, so any `-R`/`--repo` value containing a space took the whole command out
of the gate — and an entry miss on the merge route means the **recorded-verdict
rule never runs at all**. Reproduced as ALLOW at `967e0d57`, on the `pr merge`
route *and* on `api …/pulls/<N>/merge`:

| shape | why it escaped |
|---|---|
| `gh -R "$(echo owner/repo)" pr merge 930 --squash` | a command-substituted value contains spaces — **a real, working merge** |
| `gh -R 'o/r x'` / `gh -R "o/r x"` / `gh -R o/r\ x` | quoted or backslash-escaped space, same as the git side's closed case |
| `gh --repo='o/r x'`, `gh --hostname 'a b'` | ditto, on the other two listed spellings |
| `gh --version pr merge N`, `gh --help pr merge N` | never in the list at all |

`gh` now shares the git grammar rather than carrying a second dialect that
drifts out of step with it: `CLI_GLOBAL_OPT_RE` is defined once and both
`GIT_GLOBAL_OPT_RE` and `GH_GLOBAL_OPT_RE` are set from it. Identity still comes
from the merge itself — the canonicalising `sed` strips the option run first, so
a substituted or space-carrying `-R org2/repo7` contributes no digits to the PR
number (asserted). Five ordinary `gh` controls (`pr view`, `pr list`,
`repo view`, `auth status`, `pr create`) plus ten more are **byte-identical**
before and after, not merely same-verdict.

### The word-split class is bash's IFS, not POSIX space

Bash splits on space, tab and newline and on nothing else. `[[:space:]]` also
matches **CR, VT and FF**, so a value carrying one of those was one argument to
git and two tokens to the entry regex — `git -c user.name=A⏎B push origin main`
(with a CR) returned **allow**, and a shim in place of git confirms bash hands
over five arguments (`-c`, the joined value, `push`, `origin`, `main`). The class
now excludes exactly what bash splits on. Separators stay `[[:space:]]+`:
treating a CR as a separator can only make *more* commands match.

A **line continuation inside a value** was the same defect one layer up. The
continuation join replaced backslash-newline with a **space**, splitting a token
bash would have joined — `git -c user.name=A\⏎B push origin main` is
`-c user.name=AB push origin main` to git, five arguments, and it wrote
`refs/heads/main`. The replacement is now empty, which is simply what bash does.
The `git push origin --delete \⏎main` case that motivated the join is
unaffected: what separates those tokens is the space *before* the backslash.

**A newline is tested, not substituted.** Substituting the joined command would
run the span regex across former line boundaries and merge spans that are
genuinely separate — and a merged span is an ungated push. So the joined form
drives the entry test and the derivation backstop's count only: a command whose
push is visible only once the lines are joined **enters** the gate, isolates no
span, and is refused with the parse message. The backstop takes the **max** of
the raw and joined counts, because joining can also merge two adjacent pushes
into one match; both can only raise the count, so the comparison stays at least
as strict as before.

### Which global options repoint the repository

`-C <dir>` and `--git-dir=<d>` genuinely redirect, so the guard resolves them
and inspects **that** repository. `--work-tree=<dir>` does **not**: it relocates
the working tree only, and `git --work-tree=<other> rev-parse
--absolute-git-dir` still returns the current repo's `.git` (verified, git
2.43). Resolving it as a redirect made the guard inspect the wrong repository
and emit "supervisor docs-only push to main" for a code push. It is still
matched for **entry** purposes — the push is evaluated — but the repo lookup
ignores it.

Canonicalising a span into `git push …` deliberately strips those global
options, so **each span's prefix is captured separately and index-matched to
it**. Recovering the prefix from the raw command with `head -1` bound *one*
redirect to *every* span: with the CWD holding unpushed code, `git commit -m "…
git -C <docs> push origin main …" && git push origin main` returned an
authoritative allow reading "supervisor docs-only push to main" for a real code
push, and `git -C <docs> push … && git -C <code> push …` allowed on both spans.
If the two derivations disagree on how many pushes there are they cannot be
paired, and a mis-paired prefix binds a span to the wrong repository — so that
disagreement denies.

**Line continuations are joined before any rule reads the command.** Every rule
here is a line-oriented grep, so `git push origin --delete \` + newline + `main`
matched neither the delete check nor the span regex and was allowed.

### Fail-closed derivation backstop

Every rewrite between the raw command and the text a rule inspects can silently
drop content — which is exactly how both regressions above happened. So: **if
the raw command names N push acts and the derivation yields fewer, DENY.** A
mis-parsed command is refused rather than evaluated. The former
`PUSH_SPANS=("git push")` fallback was the same bug in miniature — it invented a
span rather than admitting it had none — and is gone.

There are **two** such backstops and they are **mutually masking**, which is why
both were removable with the whole suite still green until 2026-08-07 (the string
`could not parse` appeared in no suite at all):

- **A** — the span count disagrees with the `git … push` prefix count, so spans
  cannot be index-matched to the prefixes that qualify them;
- **B** — the raw command names more push acts than could be isolated.

B can **never fire alone**. `RAW_PUSH_N` can never exceed the prefix count,
because `GIT_PUSH_RE` consumes a leading *and* a trailing boundary character and
its matches are therefore longer, so grep finds no more of them than of the
unbounded prefix regex. Hence `spans < RAW_PUSH_N` implies
`spans != prefixes`: whenever B fires, A has already fired and answered. No
single command can isolate B, so the only honest contract for it is a
**joint-removal** assertion — remove both and a tested push carrying a trailing
`# git push …` comment flips from deny to allow. That is the last row of the
"two push-derivation backstops" block in `test/run-entry-shape-smoke.sh`.

**Known false deny, deliberately kept.** Two `git push` mentions with no `|`,
`;` or `&&` between them merge into one span while the raw count sees both, so
this also fires when one of them is *text* — a trailing `# git push …` comment,
or a commit message quoting a push form. Telling those apart needs a
shell-quoting parser; this hook has had two, and **both fail-opened the entire
gate**. Relaxing the count comparison instead would forgive the same shape that
hides `git push origin a $(git push origin --delete main)`, which is what the
backstop exists to catch. So the deny stays and the *message* names the shape
and the way out (`git commit -F <file>`, `gh pr create --body-file <file>`).
A false deny costs a retry and announces itself; the alternatives cost a silent
ungated push.

**The rule these follow, applied throughout this layer**: *narrow what you
EXEMPT, never what you INSPECT.* Three regressions came from the opposite move —
scoping the inspection down for precision and silently dropping enforcement with
it: `head -1` made a second push in the same command invisible; the ansible
execute exemption skipped the whole segment and left the redirect target
unguarded; the text carve-out removed a segment's text but not its execution.
Each was a narrowing that looked like an improvement and cost a gate.

## A right answer for the wrong reason

**A right answer for the wrong reason is the dominant failure mode here** —
three times in this layer it nearly shipped as a passing test:

- Chained-push cases asserted `deny` and passed, but the *first* span denied on
  a missing marker, so the second span — the one under test — was never
  reached. Mutation-checked and found green; only rebuilding the fixture so the
  first push is legitimately permitted, and asserting the deny REASON, made it
  real.
- `json_ok` was added in response to an invalid-JSON bug and then fed nine
  static-text cases, none of which could express that bug.
- `scripts/claude-pr-verdict` shipped non-executable while every test invoked it
  as `bash <path>` — the one spelling that does not need the bit.

The habit that catches all three: **mutate the fix and confirm the test goes
red**, and when a test asserts a refusal, assert *which* refusal. A guard test
that cannot distinguish "blocked for the reason under test" from "blocked for an
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

Bash-side merge routes are all gated: `gh pr merge` (any global-flag prefix),
`gh api …/pulls/N/merge`, `curl -X PUT …/pulls/N/merge`, and the GraphQL
`mergePullRequest` / `enablePullRequestAutoMerge` mutations (refused outright —
they carry a node ID, so no verdict can be looked up). Each of the three
number-bearing routes derives the PR from its OWN span, and only one merge act
per command is accepted — see [Which PR is being merged](#which-pr-is-being-merged-fixed-2026-08-07).


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

Different tree content -> different tree hash -> marker mismatch -> re-test
required. The survival condition is exactly "the root tree is byte-identical",
which is NARROWER than "no source file changed": the root tree covers EVERY
tracked path, so a commit touching any tracked file — docs included — changes
`HEAD^{tree}` and mismatches the marker (verified by comparing `HEAD^{tree}`
across a docs-only commit; an amend that rewrites only the message does not
change it). That is why the list above stops at operations which rewrite
history without rewriting content, and why it used to wrongly include
"unrelated commits (e.g. STATE.md edits)".

Docs pushes therefore get through on the separate docs/config-only exemption
below, never on tree-hash survival. STATE.md is now gitignored, so it no longer
enters a commit incidentally — but `git add -f` still forces it in (git even
advertises that escape hatch in the error it prints), so this is narrower than
"cannot be committed".

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

**Marker directory override (`ATLAS_MARKER_DIR`)**: one variable, honoured by
BOTH `mark-tests-passed.sh` (write) and `git-push-guard.sh` (read), so a test
that redirects it moves the whole flow and cannot half-isolate. Each side needs
it for its own reason — the writer so an identity-only guard test does not mint
a marker for a tree nothing compiled, the reader so `run-push-guard-smoke.sh`
no longer has to delete the live markers and restore them from an EXIT trap (a
kill mid-run used to leave every agent on the host unable to push). It cannot
widen the gate: hooks run with the harness's environment, so a marker written
under an override lands where the real guard never scans. Asserted
behaviourally — override-written marker allowed by a guard reading that dir,
refused by one with the default environment — in
`scripts/test-devcontainer-simultaneity.sh` section E. That assertion replaced a
grep for a hardcoded path in `git-push-guard.sh`, which went RED when that
literal legitimately became the override and would have been softened away.

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
- A push that names **no refspec** is judged from git **config**, not from the
  command — see "A bare push takes its destination from config" below.

### A bare push takes its destination from config

**The defect (fixed 2026-08-08).** Every other rule reads the destination off
the command string, and #928 closed the last spelling that could be *typed*.
This route types nothing:

```
git config remote.origin.push HEAD:refs/heads/main
git push origin
```

`git config` is an ordinary unblocked command; the setting lands in the repo's
**common** config, shared by every worktree, and from then on every routine bare
push from that repo retargets main. Reproduced against a local bare remote: the
guard returned **allow** — in the feature lane, unlocked by an honestly-earned
`compile.sh` marker for the feature tree — and git wrote the feature branch's
`.cs` file onto the remote's `main`. No evasion and no privilege the agent
lacked. Accident and convenience, which is what this layer exists for.

**Why not just ask git.** Both ways of asking were measured (git 2.43):

| Mechanism | Latency | Verdict |
|---|---|---|
| `git push --dry-run --porcelain` | **465-579 ms** against `origin`; 2008 ms to fail against an unreachable host with an explicit 2 s cap (git sets no default cap) | Authoritative but **rejected** — the guard's own budget is 12 ms median / 35 ms p95 / 278 ms max over 791 runs, and offline it must either deny every push or fall back to allow. A network outage that unlocks main is worse than no gate, because it is silent. |
| `git rev-parse --symbolic-full-name @{push}` | 1 ms | Local but **rejected on correctness**. It answers "where would the current *branch* go", not "what refs would this push write". With `remote.origin.push=refs/heads/*:refs/heads/*`, with a second `+HEAD:refs/heads/master` refspec, and with `push.default=matching`, git writes main/master while `@{push}` reports `refs/remotes/origin/feat/x` — an authoritative-looking **safe** answer for a command that lands on main. |
| Read the config keys (**chosen**) | +3 ms, and only on a bare push | Local, offline-proof, fail-closed. |

**The rule.** When a push span carries no refspec, the effective remote is
resolved by git's own chain (positional, `--repo`, `branch.<n>.pushRemote`,
`remote.pushDefault`, `branch.<n>.remote`, `origin`) and the push is DENIED —
with the cause named — if any of these hold. All four were measured really
writing main on a bare push, with local main one commit ahead of remote main:

| Config | Why it reaches main |
|---|---|
| `remote.<r>.mirror` = anything but a provably-false value | mirrors every local ref, main included, and **deletes** remote refs absent locally |
| `remote.<r>.push` set at all | globs and a second `+`-forced refspec both reach main, so the **presence** of the key is the trigger — parsing globs, `+`, `^` negatives and destination DWIM here would be a fifth hand-rolled parser in a file whose previous two both fail-OPENed it |
| `push.default` = `matching` (and a local main/master exists) | updates every branch present on both ends |
| `push.default` = `upstream` or `tracking`, with `branch.<n>.merge` naming main | pushes the current branch's commits onto main. `tracking` is a live deprecated synonym and was measured, not read off the man page. Skipped when the current branch already **is** main — there Rule 1 judges the same push and judges it better, so the supervisor docs-only path is untouched. |

Three values that look like routes are **not**, so they change nothing:
`simple` (the default when unset) — git refuses when the upstream name differs
from the branch name, so it can only reach main *from* main; `current` — the
destination is always the branch's own name; `nothing` — git refuses outright.

**And the config need not be on disk — that is the fifth route.** The four rows
above are the four config *keys*. The rule reads them out of the repository, but
the command being judged can set any of them **for that command alone**, writing
nothing anywhere:

```
git -c remote.origin.push=HEAD:refs/heads/main push origin
git -c push.default=matching push origin
git -c remote.origin.mirror=true push origin
GIT_CONFIG_COUNT=1 GIT_CONFIG_KEY_0=remote.origin.push \
  GIT_CONFIG_VALUE_0=HEAD:refs/heads/main git push origin
```

All four returned **allow** while git really wrote the feature branch onto the
remote's main (local bare remote, honestly-earned marker for the feature tree).
Nothing was missing except the replication: `-c` has always been in the entry
regex, so the push genuinely *was* evaluated — against the wrong config —
and `GIT_C_ARGS` already replicated `-C` and `--git-dir` precisely so that the
guard's lookups match the command. It replicated every option except the one
that changes the config this rule reads.

Every `-c <k>=<v>` on the span now goes into `GIT_C_ARGS`, and
`GIT_CONFIG_COUNT`/`KEY_<n>`/`VALUE_<n>` is translated into the same thing —
git documents the two as equivalent. Three details are load-bearing:

| Spelling | Handling | Why |
|---|---|---|
| `-c foo=bar` (no section) | **dropped**, not replicated | git refuses to run at all on one, so it reaches no destination — but passing it on makes every lookup exit 128 and return `""`, which reads as *nothing configured*. Fail-open. |
| `-c remote.origin.mirror` (no `=`) | **kept** | git's own boolean true, and it mirrors — measured, it wrote main. It also exposed a second bug: `config --get` reports a valueless key as the empty string with rc 0, so the mirror and push-refspec rules were testing **text** where they meant **presence**. They read the exit status now. |
| `--config-env`, `GIT_CONFIG_GLOBAL`, `GIT_CONFIG_SYSTEM`, `GIT_CONFIG_NOSYSTEM`, `GIT_CONFIG_PARAMETERS` | **deny** | none can be expressed as `-c` — they name an environment variable a hook cannot read, replace whole config files, or carry a git-internal serialisation this gate does not parse. What cannot be replicated is not guessed at. A fail-closed probe backs this generally: one unreadable option used to turn every risky config into *nothing configured*. |

**This list is only ever as complete as the last audit.** `GIT_CONFIG_PARAMETERS`
sat in **neither** set — not replicated, not refused — until 2026-08-08, while
this table presented the set as closed. It is `-c` by a third spelling (git
documents it as how `-c` reaches subprocesses) and it wrote main on a bare push
with nothing looking at it. It is refused rather than parsed because its payload
is a git-internal serialisation with its own quoting rules, and a mis-parse here
does not fail loudly — it returns *nothing configured*, which is an allow. Treat
a new `GIT_CONFIG_*` or config-bearing option as a **hole until proven
otherwise**, not as covered because the table looks tidy.

`GIT_DIR` is mapped to `--git-dir`, for the same reason `--git-dir` is resolved
at all. The environment prefix is captured **with its span** and matched
separately from the option half, so `MSG=--git-dir=/evil git push` cannot bind
the guard to `/evil`.

**An assignment's value may be quoted too**, and until 2026-08-08 the assignment
pattern had no quoted arm while the option pattern beside it had just gained one.
That asymmetry was a bug in its own right: *one unrelated* assignment carrying a
space truncated the whole prefix chain, because `b'` is not an assignment, so the
captured prefix began after it —

```
GIT_CONFIG_COUNT=1 MSG='a b' GIT_CONFIG_KEY_0=remote.origin.push \
  GIT_CONFIG_VALUE_0=HEAD:refs/heads/main git push origin      # ALLOW, main written
```

The command **did** enter the gate; the rule simply could no longer see
`GIT_CONFIG_COUNT` and answered from the repository's config for a push whose
destination came from the command's. Without the `MSG` assignment the identical
command already denied, which is what made the shape invisible: the protection
looked present.

**Three entry defects, found in the same audits and strictly worse**, because the
hook did not run *at all* and every rule in the file was bypassed rather than
just this one:

```
git --config-env=x=Y push origin main     # option absent from the entry regex
git -c 'user.name=A B' push origin main   # quoted value with a space ended the match
git -c user.name=A\ B push origin main    # backslash-escaped space did the same
```

All three are ordinary code pushes to main, and all three returned **allow**. The
entry regex now accepts quoted, backslash-escaped and newline-bearing option
values, and matches any dash-prefixed token rather than a list. Widening what
the entry *inspects* can only make more commands evaluated, never fewer — the
same "narrow what you exempt, never what you inspect" rule the span loop follows,
and it is **measured** rather than asserted: a 7,808-cell sweep against the
pre-fix guard reports 0 deny→allow transitions in this direction and 2,117 in the
reverse.

**Cost.** Nothing in this repo, `~/.gitconfig`, `~/.config/git/config` or
`/etc/gitconfig` sets any of these keys (audited), so the measured false-deny
cost today is zero. An ordinary `git push`, `git push origin feat/x`,
`git push -u origin feat/x` and the supervisor docs-only main path all behave
exactly as before. The rule runs **before** the `autofix/*` allowance: that
allowance rests on autofix only ever opening a PR, and a configured refspec
writes main directly and opens none.

Pinned by `test/run-push-config-destination-smoke.sh`.

**Concurrency (marker integrity)**: `mark-tests-passed.sh` derives both the tree
hash and the worktree id from `$PWD` via `git rev-parse`, so it can only ever write
a marker for its own caller's tree — one run cannot write another run's marker.
That one is structural, not a lucky timing observation: attribution follows the
caller's directory by construction.

The exposure was upstream of it. Every `.devcontainer/compile.sh` used to run
concurrently against shared `nerdctl compose` state, and no compose file sets
`container_name:` while every `/workspace` mount is relative — so container names
are identical across worktrees while the tree behind them differs. A second run of
the same service therefore either **re-creates** the first run's container bound to
its own `/workspace`, or is handed the existing one; either way a run ends up
building and testing the *other* worktree's source, exiting 0, and writing a
perfectly well-formed marker for a tree that was never tested.

Each run therefore **owns** its compose project, keyed by the worktree it was
launched from — `atlas-<sha1(worktree)[0:12]>-<slug>`, the same key the marker
filename uses (`scripts/devcontainer-owner.sh`). Distinct project names make the
container names disjoint, so there is no shared object left to race over and N
agents verify simultaneously. This supersedes the host-wide flock that briefly
served the same purpose by serializing every run.

Ownership alone would still be trusted rather than checked, so `compile.sh` now
**proves** the container has its own tree: it compares the host worktree's inode
with the one the container reports for `/workspace`, and refuses to build on a
mismatch. `mark-tests-passed.sh` will not write a marker without that attestation
(fresh, and for a path inside the caller's own worktree).

Deliberately **not** a tree hash, which is what the hooks audit first proposed:
two worktrees branched from the same commit have identical tree hashes, so a hash
comparison agrees precisely when the mix-up is most likely. Identity has to match,
not content.

Note the original failure was **silent** — it exits 0 and looks like a clean pass.
Markers written before these guards existed cannot be assumed to have tested their
own tree; there is no log that would show it either way.

`scripts/test-devcontainer-owner.sh` is the guard test (fast, no containers);
`--with-containers` adds `scripts/test-devcontainer-simultaneity.sh`, which runs
two worktrees of the same service at once and injects a foreign-tree container to
confirm the refusal fires.

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

### Which PR is being merged (fixed 2026-08-07)

A verdict is per-PR, so the gate is only as good as its answer to "which PR is
this?". Both PR-number derivations used to scan the WHOLE command string and take
`head -1`, so a command naming two PRs was judged by whichever number appeared
**first** — never necessarily by the one being merged. With #921 approved at its
current head and #923 carrying no verdict at all, every one of these returned
**allow**, merging an unreviewed PR under a different PR's approval:

| Command | Judged as |
|---------|-----------|
| `gh pr merge 921 --squash && gh pr merge 923 --squash` | #921 |
| `git commit -m "see gh pr merge 921 notes" && gh pr merge 923` | #921 |
| `echo "gh pr merge 921" ; gh pr merge 923 --squash` | #921 |
| `gh api …/pulls/921/merge && gh api …/pulls/923/merge` | #921 |
| `curl -X PUT …/pulls/921/merge && curl -X PUT …/pulls/923/merge` | #921 |
| `gh pr merge --subject "re 921" 923 --squash` | #921 |
| `gh pr merge 923 --squash --subject "cf …/pulls/921/merge"` | #921 |
| `git push origin <tested-branch> && gh pr merge 923 --squash` | not judged at all |

This is the same defect the push path fixed as C1 (a `head -1` over the raw
command applying one binding to every span); only the push side had been given
span machinery. The merge path now uses it too:

- a merge act is isolated into a **span**, and its PR identity may come only from
  that span — `gh`'s global options are canonicalised away first, so a repo slug
  like `-R org2/repo7` cannot contribute digits;
- **more than one merge act denies.** Each PR carries its own verdict and this
  hook returns one decision, so two merges need two decisions and there is one
  slot. Unlike a push-then-cleanup chain, a chained merge is not routine;
- **more than one candidate number inside one span denies** — there is no correct
  guess available there, only a lucky one. A whole-word integer is required, so a
  hex `--match-head-commit` contributes nothing;
- **a push act and a merge act in one command denies.** The push block decided
  and exited, so the merge was never evaluated — no approval at all, rather than
  the wrong one.

The last row of the table is the reason the rule is stated as **ambiguity
resolves to DENY, never to ALLOW**: the alternative to a refusal is not a
correct answer, it is a confident wrong one that says nothing when it is wrong.

#### An identity the gate cannot READ is not an absent one

Span-scoping still left one way through, because only literal ASCII digits were
ever looked for. `gh pr merge` takes the PR as a **positional**, so a positional
the hook cannot evaluate left the candidate set empty and fell through to the
no-number fallback — which answers with the **current branch's** PR. The gate
then approved one PR while the shell merged another. With #921 approved at its
head and #923 carrying no verdict, all of these returned **allow**:

| Command | Gate judged | Shell would merge |
|---------|-------------|-------------------|
| `N=923; gh pr merge $N --squash --delete-branch` | #921 | #923 |
| `gh pr merge "$PR" --squash` | #921 | whatever `$PR` holds |
| `gh pr merge ${PR} --squash` | #921 | whatever `$PR` holds |
| `gh pr merge $(cat n.txt) --squash` | #921 | whatever the file holds |
| ``gh pr merge `cat n.txt` --squash`` | #921 | whatever the file holds |
| `gh pr merge ９２３ --squash` (full-width) | #921 | — |
| `gh pr merge * --squash` | #921 | whatever the glob expands to |

The **push path already refused the identical shape** — `git push origin $BRANCH`
denies — so this was an asymmetry, not a limit of what a hook can know.

The fallback was **not** made cleverer. Resolving `$N` means predicting an
expansion that happens after this hook has already answered, which is guessing.
The **ambiguity path was removed** instead: the fallback may run only when the
span provably names no PR, and that holds only when every token after the verb
is a flag or a known flag's value. Anything else is an identity argument that
was not read, and it denies.

Only `gh pr merge`'s **own** value-taking flags are consulted (`-b/--body`,
`-F/--body-file`, `-t/--subject`, `--match-head-commit`, `--author-email`,
`-R/--repo`, `--hostname`) — applying another command's flag semantics here is
the mistake C3 already cost this file. An **unknown** flag is treated as boolean,
so a future value-taking flag leaves its value standing as an opaque positional
and this denies: fail-closed in the direction that costs a retry, not a merge.

Quoting and subshell punctuation is stripped first, exactly as the push path
does, so `OUT=$(gh pr merge <N>)` and `(gh pr merge <N>)` still resolve `<N>`
rather than reading `<N>)` as unreadable. `$(` and a backtick deliberately
survive that strip: they mark a substitution whose value is the unknown.

**Guard test**: `test/run-pr-verdict-smoke.sh` (stubs `gh`, uses PR numbers in
the 999xx range, cleans up on exit). TEST 3 plants the exact legacy marker shape
and asserts the merge is DENIED — restore the old "marker exists -> allow" logic
and it goes RED. TEST 12 covers the identity rule in both directions: rows 5-15
are the table above, rows 1-2 and 16-17 keep the normal merge path working, and
rows 3-4 are **decoys** — `gh -R jpansarasa/repo7 pr merge <N>` and a hex
`--match-head-commit` both carry digits that are not PR numbers and must still
ALLOW. Without them, "deny whenever a second integer appears anywhere" would
pass every deny row while breaking the merge command the operator actually uses;
a suite of deny rows alone cannot tell a gate that discriminates from a gate that
refuses everything.

Rows 22-23 exist because rows 5-15 all chain **two** acts, so the "more than one
merge act" refusal answers them and the span-scoping itself went unexercised —
both derivations could be re-pointed at the whole `$COMMAND` with every one of
those rows still passing. Rows 22-23 carry exactly **one** merge act, so the only
thing that can decide them is where the number is read from. Rows 24-31 cover the
unreadable-identity rule and 32-34 are its decoys (a `--body-file` path, a `-t`
subject, flags before the number — none of them an identity); 35-37 cover the
no-number fallback's own guards, which were unreachable while the stubbed
`gh pr view --json number` answered with a SHA instead of a number.

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

Two independent rules reach the deny, and they carry different remedies. Both
apply through **`Edit`/`Write` and through `Bash`** — the hook is wired to all
three. A guard on the Edit tool is not a guard on the file: while it was wired
`Edit|Write` only, `sed -i .../.claude/settings.json` — the most complete way to
unwire every gate here — reached the file with no prompt. The two tool paths
share `is_gate_path` and `is_deployed_path` rather than keeping their own lists,
because the first version of the Bash path consulted only rule 1 and left rule 2
open through Bash while `Edit` held it shut.

**Rule 1 — the GATE LAYER** (`is_gate_path`): any `.devcontainer/compile.sh`,
any `.claude/hooks/**.sh` (the whole gating layer and its own guard tests),
`.claude/settings*.json` (where the hooks are wired — un-wiring one there is the
most complete bypass available), and the two marker-writing tools
`scripts/claude-mark-verified` and `scripts/claude-pr-verdict`.

Until 2026-08-06 the list named only `git-push-guard.sh`, so every other file
that can weaken the gates — the other hooks, the settings that wire them, the
scripts that write the markers they read — was editable with no prompt at all
(verified).

**Rule 2 — the DEPLOYED artifact** (`is_deployed_path`, added 2026-08-07): any
path under `/opt/**` or `/etc/**`. This is the rule `CLAUDE.md` DEPLOYMENT
actually names — "NEVER edit `/opt/ai-inference/compose.yaml` directly … direct
edit = config drift". Roots are matched whole rather than enumerated
(`/opt/ai-inference`, `/opt/otel`, `/etc/systemd`, `/etc/sudoers.d`,
`/etc/containerd`, `/etc/apcupsd` are the live `dest:` values) so a NEW ansible
destination is covered the day it is added instead of escaping until someone
extends a pattern. Ambiguity resolves to deny.

**NOT matched: repo-tracked IaC source.** `deployment/ansible/**` — playbooks,
inventory, roles, `group_vars` — is deliberately ungated; PR review is the
control there, and changing that source *is* how a deployment is meant to
change. `*/playbooks/*` and `*deployment/inventory/*` were in rule 1 until
2026-08-07 and are gone: gating the reviewed source bought nothing while the
deployed copy it generates was writable with no prompt, so the guard enforced
the drift rule exactly backwards. (`*deployment/inventory/*` never matched
anything either — the real path is `deployment/ansible/inventory/`.)

**Decision**: `deny`. Fail-CLOSED — missing jq denies, because a guard that
cannot evaluate the gate cannot know the write is safe. (This section said
`ask` and fail-open until 2026-08-06; both were wrong, and the inverted failure
direction was the dangerous half. `ask` is inert here — see
[`ask` is inert](#ask-is-inert-on-this-host).)

**Running a gate file is not writing to it.** `<path>/compile.sh > log 2>&1` and
`ansible-playbook -i inventory/hosts.yml playbooks/deploy.yml > /tmp/deploy.log`
are ALLOWED; only writes are blocked. Both were denied before 2026-08-06 —
first because `(>|>>)` matched `2>&1`, then because a runner's flag VALUE
(`-i <inventory>`) consumed the slot where the playbook argument was expected,
so the playbook itself was judged a write target. That is CLAUDE.md's mandated
deploy invocation, and denying it also blocked the test marker, and so the push.
Reads (`cat`, `grep`, `ls`) are never blocked.

Flag-value skipping is keyed to the **executable**, not to "is this a runner":
`-f` is `--forks` to `ansible-playbook` but `--force` to `rm`, and skipping it
unconditionally left `rm -f <hook>` ungated. Keying it to the runner test was
the same bug one layer deeper — that test also matches `bash|sh|zsh|source|.`,
for which `-e -i -u -f -l -t -T -M` take no value, so the skip ate the *script*
argument and the next token (the redirect target) was consumed as "the file
being run": `bash -e /tmp/x.sh >.claude/settings.json` was ALLOWED. The list is
ansible's, so only `ansible` and `ansible-playbook` consult it.

**Matched writes** include the bare-basename form: `cd .claude/hooks && sed -i
… git-push-guard.sh` was allowed before, because only full paths were tested.
They also include the target inside a **dash token**: `--output=`, `--backup-dir=`
and `--target-directory=` name a write target that the blanket "skip flags" rule
never tested at all, so `patch --output=.claude/settings.json` was allowed. The
text after the first `=` is tested. The bare hooks **directory** is in the gate
set for the same reason `--target-directory=` needs it.


**Session bypass**: `touch $CLAUDE_PROJECT_DIR/.claude/.ansible-gate-confirmed`.
Note the path: `CLAUDE_PROJECT_DIR` is the **shared checkout**, not the worktree
an agent happens to be running in, so one agent's bypass is every concurrent
agent's bypass.

Since 2026-08-07 the file's **content scopes it**. Path fragments, one per line
(`#` comments and blank lines ignored), and the bypass applies only to paths
containing one of them — substring, not glob, because the fragment a human
writes is a path piece and not a pattern:

```
printf '%s\n' .claude/hooks/ansible-gate-guard.sh > .claude/.ansible-gate-confirmed
```

An **empty** file keeps the documented `touch` meaning — bypass everything —
because that spelling is what the deny message has always instructed. A
**non-empty** file matching nothing ENFORCES: a scope that does not cover the
path did not authorise it. So does a scope file that cannot be read, or one
whose only fragment lacks a trailing newline (`printf '%s'`); both read as zero
fragments, and until 2026-08-07 both were therefore treated as *unscoped*, i.e.
a global bypass covering `/opt/ai-inference/compose.yaml`. Ambiguity resolves to
DENY, never to ALLOW — that is the rule the whole file is written to.

It EXPIRES after 4h
(`ATLAS_GATE_BYPASS_TTL`) — expiry beats any scope — and warns when a command
actually touches a gate path — not on every tool call, which briefly made this hook emit an
unconditional `allow` for unrelated commands like `ls -la /etc`, speaking
authoritatively about calls it has no opinion on. The warning arrives via
`systemMessage` **and** `additionalContext`, so the warning reaches the user and
the model rather than only the debug log, which is all `echo >&2` from a hook
exiting 0 ever reached. Expiry is announced rather than silently honoured.
It is gitignored — committing
it would ship the gating layer switched off. Creating it is deliberately *not*
gated: this hook's own deny message instructs it, and gating the documented
remedy is the same self-contradiction as blocking the build it demands.

### A command is a SET of acts; one uncovered act refuses all of them

`refuse()` emits a decision and **exits**, so the first guarded token settled the
whole command and later acts in it went unevaluated. That is harmless while the
answer is deny — and a fail-**open** the moment a bypass is live, because the
first *covered* token returns an authoritative `allow` and the acts behind it are
never looked at. With the scope naming only `ansible-gate-guard.sh`:

| Command | Was | Now |
|---------|-----|-----|
| `sed -i … ansible-gate-guard.sh && sed -i s/deny/allow/ … git-push-guard.sh` | allow | deny |
| `cp .claude/hooks/ansible-gate-guard.sh .claude/hooks/git-push-guard.sh` | allow | deny |
| `sed -i … ansible-gate-guard.sh && sed -i … /opt/ai-inference/compose.yaml` | allow | deny |

The second row is one **segment**, so segment-splitting cannot help: two guarded
tokens, and only the first was consulted. The third crosses the rules — and with
the documented **empty** `touch`, a gate act followed by a deployed one was
allowed too, inverting the invariant that an unscoped bypass must not reach
deployed config. The deployed write *alone* was correctly denied; only the chain
slipped, which is exactly why 507 green assertions could not see it.

This is the same class as the `head -1` over push spans in `git-push-guard.sh`,
and the same fix: **narrow what you EXEMPT, never what you INSPECT**. Findings
now accumulate across the whole walk and the decision is taken once, after it —
the token is the unit of evaluation, the command is the unit of decision, and any
deny wins. The bypass may speak only when its scope covers **every** finding, and
the announcement then names them **all**; it used to name only the first, which
under-reported a multi-act command to the one human who could stop it.

**Guard test**: `test/run-advisory-guards-smoke.sh`, section "a command is a SET
of acts". The CONTROL rows carry the weight — the in-scope act alone must still
ALLOW, or a deny on the chain rows would prove nothing.

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
| `test/run-entry-shape-smoke.sh` | `git-push-guard.sh` entry shapes — one case per bypass form verified ALLOW before 2026-08-06. Marker-free — safe beside live agents. `PUSH_GUARD_HOOK` points it at another guard, as in the config-destination suite; until 2026-08-08 it ignored that variable and always read its own sibling, so the documented way to demonstrate its teeth did not work |
| `test/run-wiring-smoke.sh` | every hook is registered in the TRACKED `settings.json`, the registered set has not drifted, and the marker writers are executable in the index. Static — safe beside live agents |
| `test/run-push-guard-smoke.sh` | `git-push-guard.sh` marker lookup: global tree-hash scan, orphaned markers, block diagnostics. Runs against an isolated `ATLAS_MARKER_DIR` |
| `test/run-push-exemption-smoke.sh` | the docs-only / docs-config allowlists on the push path |
| `test/run-push-config-destination-smoke.sh` | the bare-push route: 1020 cells over config state x HEAD x content x marker x command shape, plus 14 targeted and 21 config-injection rows, each cross-checked against what `git push --dry-run --porcelain` says the fixture would really write. Liveness is asserted in both directions — a cell whose fixture *cannot* push proves nothing, and six (state, HEAD) pairs were in that condition until 2026-08-08. `PUSH_GUARD_HOOK` points it at another guard — against the pre-fix one it must go RED, which is how its teeth are demonstrated |
| `test/run-pr-verdict-smoke.sh` | `claude-pr-verdict` preconditions, the merge gate's verdict parsing, and which PR a merge is judged as (stubs `gh`, uses PR numbers 99901-99904) |
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
| `rm -f .claude/hooks/…` expecting `ansible-gate-guard` | **no** — *at the time* it was wired at `Edit\|Write`, not `Bash`, so a Bash command could never reach it (it is wired at `Bash` since 2026-08-07, and this probe would now deny) | silent, proves nothing |
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
