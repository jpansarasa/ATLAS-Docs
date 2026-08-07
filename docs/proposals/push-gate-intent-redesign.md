# Push/Merge Gate — Intent Redesign

**Status:** SPEC for a build decision. Nothing implemented; no production code written; no hook,
settings file or script was modified in producing it.
**Verdict:** **BUILD, with the docs-only main-push allowance KEPT BUT MOVED.** Delete it from the
command-string guard (making string-level main-push denial unconditional and non-authoritative),
and re-express it at the `pre-push` boundary, where git *hands* the hook the refs and the diff
instead of a parser inferring them from an arbitrary shell string. The hypothesis in the brief —
"delete the allowance outright" — is **half right**: the parser-side allowance must go, but the
capability is neither dead weight nor safely deletable as a workflow, and at the act boundary it
costs nothing to keep because there is nothing left to mis-parse.
**One dependency is unverified and the verdict rests partly on it:** Layer 2 (the `mcp__` matcher,
§5) assumes such a matcher fires, which no hook on this host currently demonstrates (§10.2). If it
does not, the MCP write surface stays open and this verdict changes. Prove it before relying on it.
**Origin:** user, verbatim — *"This was to stop agents from pushing broken code to main. Early on
in the development cycle, agents would vibe code crap, not review, break tests, then push directly
to main. This is \*ALWAYS\* unacceptable. The INTENT is to stop any agent from pushing to main
without passing certain gates first. I don't care about implementation. I care about the intent of
the hook."*
**Measured:** 2026-08-07 on mercury. All probes fed command strings to the guards as **stdin data**;
no push to ATLAS's origin and no merge was performed at any point. Fixture pushes ran against
throwaway bare repos under the session scratchpad (`git remote -v` recorded in each fixture).
**Retirement:** working doc, retired per CLAUDE.md PHASE_TAGS on completion.

---

## 0. PROVENANCE — read before any number below

Every figure is tagged. Untagged assertions are arguments, not measurements.

| tag | meaning |
|---|---|
| `[P]` | I probed it in this session; command and fixture named |
| `[L]` | live observation on this host, reproducible now |
| `[A]` | measured by a dispatched agent this session, citation given |
| `[H]` | derived from git history by re-classifying commits; a projection, not an observation |
| `[U]` | **unverified** — inference or inherited claim I could not test. Collected in §10 |

**Citation form: name-primary, line as a pinned hint.** Line numbers in this document have rotted
twice in a single day — `SKILL.md`'s `GIT_OPS [cwd_drift_guard]` sat at `:128` when §3.1 was written
and at `:152` on `origin/main` hours later, and PR #921's head moved twice more while §2.3 was being
corrected. So every reference here names the **stable key** first — a rule name, a function, a
heading — and carries the line only as a parenthetical hint pinned to a sha. Two deliberate
exceptions keep bare line lists, because they are static enumerations an implementer has to *count*
rather than *find*: AC2's inventory of allow sites, and the §7 salvage list (whose line numbers refer
to PR #921's frozen artifacts, not to main). **Unless stated otherwise, every `git-push-guard.sh`
line hint below is as of `7675e93f`.** Pure by-name would
break those; pure by-line has already broken twice. Name-primary makes rot **visible but harmless** —
a stale hint next to a findable name is a nuisance, whereas a stale line number alone silently points
at the wrong code.

**The population caveat that governs §4.** The workflow-cost numbers come from re-classifying
485 commits on `main`. That population **structurally cannot contain** a push that was *attempted
and denied* — the gate leaves no record of what it refused. So §4 measures what the allowance
*carried*, never what it *cost*. AC10 exists specifically because that denominator is missing.

---

## 1. The governing principle

A gate that decides from parsing an arbitrary shell string cannot parse reliably. That is the
nature of the input, not a bug awaiting a fix. Five rounds (#918 → #921) each closed one hole and
opened another because each tuned parser **accuracy**.

Accuracy was never the requirement. The requirement is **asymmetry**:

> **Ambiguity must resolve to DENY. A false deny costs seconds and announces itself. A false allow
> is broken code on main, silently.**

The design move that makes this structural rather than aspirational:

> **The command-string guard must lose the ability to say YES.**

Under this spec the string guard emits only `deny` or *no opinion*. Its `allow` stops unlocking
anything, because a second gate — one that does no parsing — still runs behind it. A mis-parse can
then only ever cost a false deny. That is not a better parser; it is a parser that is no longer
load-bearing for the dangerous direction.

---

## 2. CONSTRAINTS — each re-verified, not taken on faith

### 2.1 Server-side enforcement is UNAVAILABLE — CONFIRMED `[P]`

```
gh api repos/jpansarasa/ATLAS/branches/main/protection
  -> 403 {"message":"Upgrade to GitHub Pro or make this repository public to enable this feature."}
gh api repos/jpansarasa/ATLAS/rulesets
  -> 403 {"message":"Upgrade to GitHub Pro or make this repository public to enable this feature."}
gh api repos/jpansarasa/ATLAS -> {"private":true,"visibility":"private","allow_auto_merge":false}
gh api user -> {"login":"jpansarasa","plan":null}
```

**Consequence:** the local layer is the only boundary, with no backstop. This spec is sound without
server-side protection. What changes IF the user upgrades to Pro is confined to §8 and is treated
as a variable throughout — no rule here assumes it.

### 2.2 STATE.md untracking — LANDED `[P]`

An earlier revision of this section described `chore/untrack-state-md` as local-only and unmerged,
and built §4's survivor projection on that. **It has since merged**: `9ed5709a` — *"chore: untrack
STATE.md (supervisor working memory, not product) (#922)"*. On `origin/main` today
`git ls-tree origin/main -- STATE.md` returns empty and `.gitignore:122` reads `/STATE.md`. Every
clause of the old text — local-only, unpushed, "in flight" — is now false and has been removed
rather than annotated.

**What this changes for §4, and what it does not.** The regime is no longer hypothetical, so §4 is
not projecting a future. But it landed too recently to have produced an observable push rate, and
§4's survivor counts are still a re-classification of *history* with `STATE.md` removed from each
commit's file set. They therefore keep their `[H]` tag, correctly. §10.7 states this precisely.

**`STATE.md` is gitignored and has no git backup.** Nothing in this spec restores, commits or
removes it.

### 2.3 PR #921 is BLOCKED — CONFIRMED, with corrections `[P][A]`

The recorded verdict for #921 reads `v2 blocked f5dc93d9… 2026-08-07T12:53:25Z`, citing an
authoritative-allow regression. Branch `fix/push-guard-command-parsing`, state OPEN.

**The recorded sha is now stale, and the reason that does not matter is worth stating.** `[P]` The
branch has been rewritten repeatedly during this round: the verdict names `f5dc93d9`, and the PR's
head has since been `5996e4fe` and then `2d69829b` (sole parent `5996e4fe`) — it moved again between
the round that measured it and the round that wrote this sentence. So the recorded BLOCK no longer
describes the code that would merge.

The merge nevertheless stays denied **via the BLOCK path, not the staleness path**, and this is a
property of the guard's ordering rather than luck: the `blocked` branch of the verdict check returns
and exits *before* the head-staleness comparison is reached — the staleness check guards only the
**approved** path, as the guard's own comment there says ("Verdict is approved — it must still
describe the code that would merge"). Confirmed against an isolated `ATLAS_MARKER_DIR` carrying a
deliberately stale `blocked` verdict: the deny reads "PR #921 was reviewed and the verdict is BLOCK"
and quotes `f5dc93d9`, a commit no longer in the PR `[P]`. A BLOCK is therefore permanent until a
new verdict is recorded, and it denies citing stale evidence — in the safe direction. That
asymmetry (stale ⇒ still-deny on the block path, stale ⇒ deny on the approve path) is the one place
the existing merge gate already gets §1's rule right, and it should survive the redesign intact.

Corrections to numbers inherited from the brief are in §7.

---

## 3. WHAT MAIN ACTUALLY DOES — the measurement that reframes the problem

All probes fed command strings to `/home/james/ATLAS/.claude/hooks/git-push-guard.sh` on stdin
against a fixture whose local `main` is **ahead of `origin/main` by a code-only commit**. Any
`allow` there is code reaching main.

### 3.1 The entry condition does not fire for the project's own mandated idiom `[P]`

The guard's **push entry condition** (`git-push-guard.sh:182`) admits on
`(^|\s|&&|;)\s*git\s+push(\s|$)` — `git` immediately followed by
`push`. Any git global option between them defeats it. **10 of 10 probes returned `allow` with an
empty reason** — i.e. the guard fell through to its terminal "Allow all other commands" at :644 and
never entered the push path at all:

| probe | command | verdict |
|---|---|---|
| P01–P02 | `git -C <path> push origin main` | allow, no reason |
| P03 | `git -C . push origin main` | allow, no reason |
| P04 | `git --git-dir=<...> push origin main` | allow, no reason |
| P05 | `git --work-tree=<...> push origin main` | allow, no reason |
| P06 | `git --no-pager push origin main` | allow, no reason |
| P07 | `git -c user.name=x push origin main` | allow, no reason |
| P08 | `git -C . push origin :main` | allow — **deletion of main** |
| P09 | `git -C . push origin --delete main` | allow — **deletion of main** |
| P10 | `git -C . push origin feat/code` | allow — test marker never consulted |

This is a **non-parse, not a mis-parse.** No accuracy improvement reaches it; the condition simply
never fires. And it is not an exotic spelling:

> `.claude/skills/supervisor-mode/SKILL.md`, the `GIT_OPS [cwd_drift_guard]` rule (`:152` on
> `origin/main` today) — `every supervisor git command -> `git -C /home/james/ATLAS <op>``

The repo's own operating doctrine **mandates** the spelling that bypasses the gate. The four lines
below that rule already document it ("so `git -C <path> push` is UNGATED … verified 2026-08-06"),
and `.claude/hooks/README.md` lists it under *KNOWN GAP: entry spellings* with the note "This is
unchanged from `main`".

### 3.2 The guard fails OPEN when its own deny message is malformed — VERIFIED LIVE `[L]`

Main builds deny messages with expanding heredocs that interpolate a **parsed** branch name raw —
the **"Tests have not passed" deny** (`git-push-guard.sh:503-511`) and the **"verdict is BLOCK"
deny** (`:596-604`). A branch name containing `"` — which is exactly what the
parser produces when it mis-reads a quoted string — yields invalid JSON.

Verified end-to-end with a payload that pushes nothing:

```
command fed to Bash:  echo "harmless: git push origin feat/x"
guard stdout:         {"…","permissionDecision": "deny",
                       "permissionDecisionReason": "…Branch:       feat/x"\nCurrent tree: feat/x"^…
                      -> INVALID JSON
observed:             the command RAN. Output: harmless: git push origin feat/x
```

The guard decided **deny** and the harness executed anyway. `.claude/hooks/README.md:583` names this
class in its own words: *"an unparseable deny is an absent deny, failing exactly as silently as the
`ask` did."*

This is the asymmetry violation in its purest form: **the parser's own confusion is the trigger, and
the failure direction is ALLOW.** Three probe forms reproduce it (`echo "…" && git push origin main`,
`git commit -m "…" ; git push origin main`, and a `\`+newline refspec).

`[A]` The same defect voids the **merge** gate: a `blocked` verdict whose reason text contains a
double quote — reviewer free prose, interpolated into the same "verdict is BLOCK" deny — emits
invalid JSON, so the block is
silently void. Only the deny paths are affected; the approve path emits a static string and can
never break. **The gate fails open precisely on the reviews that found something.**

### 3.3 Degenerate stdin fails open `[P]`

| stdin | decision |
|---|---|
| empty (0 bytes) | **allow** |
| `null` | **allow** |
| `{"tool_input":{}}` | **allow** |
| `not json at all` | deny |
| truncated JSON | deny |
| `[]` | deny |

`[A]` corrects the brief here: main does **not** fail open on malformed stdin — the
`COMMAND=$(… | jq …)` / `if [[ $? -ne 0 ]]` pair in the **stdin parse preamble** (`:146-150`)
works, because a plain
`VAR=$(cmd)` assignment does propagate the substitution's status. The fail-open is **empty/null**
input, where jq exits 0 and `COMMAND` is empty.

### 3.4 Inside the allowance, a decoy binds the decision `[P]`

The **push-span extraction** (`:206`) runs `grep -oE 'git\s+push\b[^|;&]*' | head -1` — **only the
first push-shaped substring is ever evaluated.**

| probe | command (fixture main carries CODE) | verdict |
|---|---|---|
| D02 | `git commit -m "note: git push origin feat/docs is the supervisor form" && git push origin main` | **allow** — "docs/config-only push — test marker not required" |
| D05 | `git push origin feat/docs && git push origin main` | **allow** — same reason |
| D10 | `git commit -m "x" && git push origin feat/docs && git push origin main` | **allow** — same reason |
| G04 | `git push origin feat/docs && git push origin feat/code` | **allow** — marker never checked for `feat/code` |

An **authoritative allow**, reading "docs/config-only push", for a command whose real effect is a
code push to main.

### 3.5 Refspec spellings walk past Rule 1 `[P]`

With a valid `tests-passed` marker present for the code tree (the ordinary state after any
`compile.sh` run):

| probe | command | verdict |
|---|---|---|
| H02 | `git push origin refs/heads/main` | **allow** — code onto main |
| H08 | `git push origin heads/main` | **allow** — code onto main |

`heads/main` resolves identically to `refs/heads/main` `[P]` (`git rev-parse heads/main` ==
`git rev-parse refs/heads/main` == `0d07aed0`). Without a marker these deny — but for the *wrong
reason* ("Tests have not passed"), which is why no existing assertion catches them.

### 3.6 Deleting main is allowed via a line continuation `[P]`

| probe | command | verdict |
|---|---|---|
| F04 | `git push origin --delete \`⏎`main` | **allow** — "branch deletion exempt from test marker" |

`grep` is line-oriented, so the inner check of **Rule 0.5** (branch deletion, `:299`) never sees
`--delete` and `main` together. The
one-line form correctly denies. This is the only one of PR #921's five named defects that is
**also** a main defect (§7).

### 3.7 The merge gate: 5 of 12 spellings ungated `[P]`

| ungated | reason |
|---|---|
| `gh -R jpansarasa/ATLAS pr merge 921 --squash` | global option between `gh` and `pr` |
| `gh --repo … pr merge 921 --squash` | same |
| `gh api -X PUT repos/…/pulls/921/merge` | merge by REST path |
| `gh api graphql -f query='mutation { mergePullRequest… }'` | GraphQL mutation |
| `curl -X PUT …/pulls/921/merge` | different client |

Gated correctly: bare `gh pr merge`, `--squash` before the number, `cd … && gh pr merge`,
`GH_REPO=… gh pr merge`, double-spaced, `--admin`, and `gh pr merge` with no number (denies).

### 3.8 The MCP write path has no matcher at all `[P]`

Dumping every hook matcher across all three settings files:

| event | matcher | hooks |
|---|---|---|
| PreToolUse | `Edit\|Write` | testing-context, benchmark-context, observability-context, ansible-gate-guard, service-decisions-context |
| PreToolUse | `Agent` | design-intent-dispatch-guard |
| PreToolUse | `Bash` | **git-push-guard**, dotnet-guard, node-guard, plan-retirement-guard |
| PreToolUse | `Write` | ef-migration-guard |
| PostToolUse | `Skill` / `Bash` / `Write\|Edit` | pr-review-marker, deploy-smoke-reminder, memory-density-guard |

**No `mcp__` matcher exists anywhere** — and note before reading Layer 2 that *whether one would
fire if added is itself unverified* (§10.2). Every dispatched agent carries
`mcp__plugin_github_github__push_files`, `create_or_update_file`, `delete_file` and
`merge_pull_request` — each of which writes to main or merges a PR **with no marker consulted and no
string for any guard to inspect.** `~/.claude/settings.json` and `.claude/settings.local.json` both
set `"defaultMode": "bypassPermissions"`, so there is no permission layer behind the hook either.

### 3.9 Direct GitHub HTTP writes — a route class no layer covers `[P]`

`gh api` and `curl` reach `main` without git transport and without an `mcp__` tool name. Fed to the
guard on stdin, **8 of 8 returned `allow` with an empty reason** — the terminal fall-through at
`:644`, the same non-parse as §3.1:

| command | verdict |
|---|---|
| `gh api -X PUT repos/…/contents/README.md -f branch=main …` | allow — commits a file straight to main |
| `gh api --method PUT repos/…/contents/docs/x.md -f branch=main …` | allow |
| `gh api -X PATCH repos/…/git/refs/heads/main -F force=true -f sha=…` | allow — **force-moves main to any sha** |
| `gh api -X DELETE repos/…/git/refs/heads/main` | allow — **deletes main outright** |
| `gh api -X POST repos/…/git/refs -f ref=refs/heads/main …` | allow |
| `curl -X PATCH …/git/refs/heads/main -d '{"sha":…,"force":true}'` | allow |
| `curl -X DELETE …/git/refs/heads/main` | allow — **deletes main outright** |
| `curl -X PUT …/contents/README.md -d '{"branch":"main"}'` | allow |

**No layer as specified covers this.** Layer 1 never sees it — there is no git transport, so
`pre-push` is not invoked. Layer 2 matches `mcp__` tool names, and these are Bash. Layer 3's
keep-list gates `gh api …/pulls/N/merge`, graphql `mergePullRequest` and `curl …/merge` — **merge
routes only**; nothing in it mentions `contents` or `git/refs`.

**Layer 3 owns it**, on the same grounds the merge routes are already there: these are fixed-token
denies against a URL path, not branch/tree/diff inference, so they add no parsing that could return
an allow. The keep-list must extend to `…/contents/…` and `…/git/refs/…` writes for `gh api`,
`curl` and any other client, matching on the path rather than the client. Note also that the two
`DELETE` rows remove `main` server-side in one non-interactive command, which lands on the row §11
already marks unrecoverable ("server-side ref history → none") and on §10.6's untested
force-deleted-main recovery. §3.6 found a local-transport route to the same outcome; this is a
second, simpler one that no proposed layer sees.

### 3.10 Summary — main's defect set

| # | defect | direction | §|
|---|---|---|---|
| M1 | git globals (`-C`, `--git-dir`, `--work-tree`, `-c`, `--no-pager`) → not seen as a push | **ALLOW** | 3.1 |
| M2 | malformed emitted JSON voids the deny | **ALLOW** | 3.2 |
| M3 | empty/null stdin | **ALLOW** | 3.3 |
| M4 | `head -1` span binding; decoy binds the verdict | **ALLOW (authoritative)** | 3.4 |
| M5 | `refs/heads/main`, `heads/main` walk past Rule 1 | **ALLOW** | 3.5 |
| M6 | `--delete \`⏎`main` deletes main | **ALLOW** | 3.6 |
| M7 | 5 of 12 merge spellings ungated | **ALLOW** | 3.7 |
| M8 | no `mcp__` matcher; 4 write tools ungated | **ALLOW** | 3.8 |
| M9 | Rule 0 unanchored `printf\|echo` false-denies **reads** | DENY | `[A]`, hit live twice |
| M10 | `gh api` / `curl` writes to `…/contents/…` and `…/git/refs/heads/main`; 8/8 allow, incl. deleting main | **ALLOW** | 3.9 |
| M11 | merge decoy: the `head -1` PR-number bind lets an approved PR's verdict authorise a **BLOCKED** one | **ALLOW (authoritative)** | 6, AC7 |

Eleven defects; **ten fail toward allow.** That distribution is the whole argument.

M11 is listed here rather than in a §3 subsection because it was found while building AC7's decoy
rows, not during the §3 sweep — which is itself the point: §3.7 enumerated merge *routes* and had no
decoy in it, so the class could not surface there.

---

## 4. WHAT THE ALLOWANCE ACTUALLY CARRIES `[H]`

Commits on `main` since 2026-06-02 (the date the allowance was added), classified by whether the
subject carries a `(#N)` squash-merge suffix and whether every changed file matches the live
allowlist regex (the `non_doc` filter in `is_supervisor_docs_only()`, `git-push-guard.sh:134` as of
`7675e93f`), applied verbatim. Measured at `0d07aed0` — the commit this document was written
against — merge commits included, via the **anchored** command:

```
git log 0d07aed0 --since=2026-06-02T00:00:00
```

**The `T00:00:00` is load-bearing; do not drop it.** `[P]` A bare `--since=<date>` is an
*approxidate*, and git fills the missing time-of-day from the **current clock**. The same command
therefore returns a different population depending on when it is run: measured in one session,
`--since=2026-06-02` gave **474** at 11:52, **473** at 11:56 and **485** under
`--since=2026-06-02T00:00:00` — while `--since=2026-06-02T23:59:59` gives 466. Every figure this
section previously carried (477, and a reviewer's 476) was the same command at a different clock
time, not a different classification and not an off-by-one. Anchoring the timestamp is what makes
the block reproducible; re-deriving it against a bare date will silently disagree again.

| | count |
|---|---|
| total commits on main | 485 |
| arrived via PR squash-merge | 329 |
| direct-push shaped | 156 |
| …of those, docs-only under the live regex | **152** |
| …of those, not docs-only | 4 |

The four are named rather than characterised, because the earlier gloss ("3 predate/skirt the rule")
is incoherent inside a window that *starts* on the rule's own date — nothing in it can predate it:
`1c2ef5e5` (a `.gitignore`), `3a802551` and `9cfcb871` (both
`deployment/ansible/group_vars/all.yml`, the first also `deployment/artifacts/compose.yaml.j2`), and
`6bf8f9b2` (a merge commit, no files of its own — an empty changed set cannot be shown to be
docs-only, so the classifier counts it here). Dropping merge commits gives
484 / 329 / 155 / 152 / **3** — the classification is sensitive to that one switch, which is why the
exact command is stated.

Of the 152:

| | count | share |
|---|---|---|
| touch `STATE.md` at all | 109 | 72% |
| **`STATE.md` and nothing else** | **102** | **67%** |
| carry content other than `STATE.md` | **50** | 33% |

**Untracking STATE.md removes just over two-thirds of the allowance's traffic.** What survives —
each window anchored the same way, `--since=<date>T00:00:00` at `0d07aed0`:

| window | `--since` | docs-only direct | survives untracking | survivors/day |
|---|---|---|---|---|
| 7d | `2026-07-31T00:00:00` | 9 | 7 | 1.00 |
| 14d | `2026-07-24T00:00:00` | 10 | 8 | 0.57 |
| 30d | `2026-07-08T00:00:00` | 29 | 26 | 0.87 |
| 66d | `2026-06-02T00:00:00` | 152 | 50 | 0.76 |

The 66d row is the base population by construction — 2026-08-07 minus 66 days *is* 2026-06-02 — so
it agrees with the table above rather than drifting from it. An unanchored window subtracting 66×24h
from the anchor commit's own timestamp returns 151, dropping the one commit that landed between
midnight and 07:54 on 2026-06-02: the same approxidate hazard, one level down.

The 50 survivors touch **58 files**. The class breakdown below counts **file-touches, not commits** —
a single commit can touch more than one class, so these sum to 58 and not to 50.

| file-touches | class |
|---|---|
| 25 | `docs/**` narrative docs (excluding `RELEASES.md`) |
| 15 | `docs/RELEASES.md` — **mandated by CLAUDE.md PHASE_TAGS step 3** |
| 12 | `.claude/skills/**.md` — supervisor-owned per SKILL.md's EDIT block |
| 5 | service READMEs (4: SentinelCollector ×2, NasdaqCollector, CalendarService) + `.claude/commands/**.md` (1) |
| 1 | `CLAUDE.md` |

Spread over **18 distinct calendar days out of 66** (27%). Extension histogram across the 58:
**`.md` × 58, nothing else.**

*An earlier revision's "50 survivors / 58 file-touches" was right and was wrongly "corrected" to
48 / 56.* The correction was made under the unanchored command, so it swapped a correct pair for one
drawn from a smaller population; the note that then accused that revision of "silently switching
unit mid-table" was itself the error and is withdrawn. The unit was never switched — the population
was.

**The decisive property:** every one of the 50 is a *commit on main, then push main*. **Not one
requires a user-supplied refspec.** There is nothing for a parser to get wrong except which branch
is meant — and at the `pre-push` boundary, git states that outright.

**Verdict on the hypothesis.** ~0.57–1.00/day across 27% of days, concentrated in five known path
classes, two of which are mandated routine steps, is not dead weight — deleting it outright imposes
a full branch/push/mark-verified/PR/review/verdict/merge cycle on a typo fix in `RELEASES.md`. But
the allowance as *currently sited* is indefensible: §3.4 shows it is the single surface where a
mis-parse becomes an authoritative "yes". Both facts are satisfied by moving it, not by choosing
between them.

---

## 5. THE DESIGN

Three layers. **No layer parses an arbitrary shell string to decide ALLOW for a PUSH.**

One exception survives by necessity, and is stated here rather than left to be discovered: the
**merge** gate must read a PR number out of the command string in order to look up that PR's
verdict, so it does reach an `allow` after parsing (the **PR-number bind**, `:531` → the
**merge-approved allow**, `:639`). It cannot move to Layer 1, because a merge uses no git transport.
§6's AC2 is therefore
scoped to the push paths, and AC7 carries decoy rows — because §3.4's `head -1` span-binding defect
survives on precisely that path.

### Layer 1 — the act boundary: a `pre-push` git hook `[P]`

git invokes `pre-push` with the refs on stdin as
`<local ref> <local sha> <remote ref> <remote sha>`. **Measured** against a throwaway fixture, with
a hook that denies `refs/heads/main`:

| spelling | push exit | what the hook received |
|---|---|---|
| `git push origin main` | **1 denied** | `rref=refs/heads/main` |
| `git -C <path> push origin main` | **1 denied** | `rref=refs/heads/main` |
| `git --git-dir=… --work-tree=… push origin main` | **1 denied** | `rref=refs/heads/main` |
| `git push origin HEAD:refs/heads/main` | **1 denied** | `lref=HEAD rref=refs/heads/main` |
| `git push origin refs/heads/main` | **1 denied** | `rref=refs/heads/main` |
| `git push origin heads/main` | **1 denied** | `rref=refs/heads/main` |
| `git push origin --delete main` | **1 denied** | `lref=(delete) lsha=0000…` |
| `git push origin :main` | **1 denied** | `lref=(delete) lsha=0000…` |
| `git push origin feat/x` | 0 allowed | `rref=refs/heads/feat/x` |
| `git push --no-verify origin main` | **0 — hook skipped** | — |

**Every spelling that defeats the string guard arrives here already normalised to
`refs/heads/main`.** git does the DWIM; the hook does zero parsing. Deletion arrives as an explicit
`(delete)` / all-zero sha. Confirmed the remote ref did not move on any denial.

**Eight further spellings, measured on the same fixture, all normalise identically** `[P]` — this is
the strongest single piece of evidence for the design, so it is stated in full rather than asserted:

| spelling | push exit | what the hook received |
|---|---|---|
| `git push origin +main` (force refspec) | **1 denied** | `rref=refs/heads/main` |
| `git push origin main:main` | **1 denied** | `rref=refs/heads/main` |
| `git push origin feat/x:main` | **1 denied** | `rref=refs/heads/main` |
| `git push origin HEAD` | **1 denied** | `rref=refs/heads/main` |
| `git push -f origin main` | **1 denied** | `rref=refs/heads/main` |
| `git push --all origin` | **1 denied** | `refs/heads/main` **and** `refs/heads/feat/x` |
| `git push --mirror origin` | **1 denied** | `refs/heads/main`, `refs/heads/feat/x`, `refs/remotes/origin/main` |
| `git push <file-URL> main` (URL instead of a named remote) | **1 denied** | `rref=refs/heads/main` |

`refs/heads/main` byte-identical before and after all eight. Two things follow. First, the rules
**must be evaluated per ref line, not per invocation** — `--all` and `--mirror` present main as one
line among several, and a hook that inspects only the first would pass them. §5's rule list already
says "evaluated per ref line, in order"; this is why that phrasing is load-bearing rather than
stylistic. Second, `--mirror` and `--all` need **no special-casing at all** at this boundary, which
retires §7's SALVAGE item 4 (`Rule 0.45 --mirror/--all`) along with the rest of the span machinery —
git enumerates the refs, so there is no flag to recognise.

For contrast, the same eight fed to the string guard split three ways `[P]`: `+main`, `main:main`,
`feat/x:main`, `-f` and the URL form deny; **`--all`, `--mirror` and `HEAD` return an authoritative
`allow` reading "docs/config-only push — test marker not required"** (measured from a worktree whose
own diff is docs-only — the verdict is computed from the *current tree*, not from the refs being
pushed, which is precisely the inference Layer 1 removes).

Rules, evaluated per ref line, in order:

1. `rref` is `refs/heads/{main,master}` **and** `lsha` is all-zeros → **DENY** (deletion).
2. `rref` is `refs/heads/{main,master}` → compute `rsha..lsha` changed files. **All** match the
   doc-extension allowlist → allow. Anything else, or a range that cannot be computed (unknown
   `rsha`, all-zero `rsha`, empty diff) → **DENY**.
3. Any other `rref` → require a `tests-passed` marker for `lsha^{tree}`, unless the changed set
   versus `origin/main` is docs/config-only.
4. Anything not matched above, any error, any non-zero from an internal git call → **DENY**.

This is the docs-only allowance, unchanged in policy, sited where its inputs are **given** rather
than inferred. §3.4's entire defect class ceases to exist because there is no span, no `head -1`,
and no branch to guess.

**The gap is not one token, and one member of it defeats BOTH layers.** Measured on a throwaway
fixture (git 2.43.0) with a `pre-push` that always exits 1, each spelling paired with the string
guard's verdict on the same command `[P]`:

| spelling | Layer 1 (pre-push) | string guard | net |
|---|---|---|---|
| `git push origin main` (control) | ran, rc=1, ref HELD | deny | gated |
| `git -c core.hooksPath=/dev/null push origin main` | skipped, rc=0, main **MOVED** | **allow** | **DOUBLE bypass** |
| `git -c core.hooksPath=/nonexistent push origin main` | skipped, rc=0, main **MOVED** | **allow** | **DOUBLE bypass** |
| `GIT_CONFIG_COUNT=1 GIT_CONFIG_KEY_0=core.hooksPath … push origin main` | skipped, rc=0, main **MOVED** | deny | single |
| `git push --no-verify origin main` | skipped, rc=0, main **MOVED** | deny | single |

Four spellings skip `pre-push`, not one. Two of them are **single** bypasses — `GIT_CONFIG_COUNT`
and `--no-verify` defeat Layer 1 but the string guard still denies them, so the pair holds. Only the
two `-c core.hooksPath=` spellings defeat both layers, and those are the ones that matter.

**The root cause is M1, not a missing `core.hooksPath` rule.** The entry regex
of the **push entry condition** (§3.1) requires `push` to follow `git` immediately, so a `-c`
between them defeats the match and the script falls through to the terminal `allow` at `:644`. A
"deny `core.hooksPath` mutation" rule written **inside** that entry condition would therefore never
run on the one spelling it exists to stop. The fixed-token denies must be evaluated **before** the
entry regex, against the whole command string — see §9 step 2. The same fall-through is why
`git -C /home/james/ATLAS push origin main` returns **allow** `[P]`: the idiom SKILL.md's
`GIT_OPS [cwd_drift_guard]` MANDATES is ungated by the string layer today (§3.1).

### Layer 2 — the tool-name boundary: an `mcp__` matcher

A `PreToolUse` matcher on the fixed names
`mcp__plugin_github_github__(push_files|create_or_update_file|delete_file|merge_pull_request|update_pull_request_branch)`
returning `deny` unconditionally. Fixed tool names; nothing to parse. Read-only GitHub MCP tools
are untouched. `[U]` — see §10.2: this layer rests on an unverified assumption.

### Layer 3 — the string boundary: the existing hook, radically shrunk

It keeps only checks that are **unambiguous token tests**, and it loses every branch, tree, diff and
extension computation:

| keep | why it is safe |
|---|---|
| deny `--no-verify` / `--no-hooks` alongside any git invocation | fixed token; closes one of Layer 1's four bypass spellings (§5) |
| deny mutation of `core.hooksPath`, **incl. the `-c` and `GIT_CONFIG_*` forms** | fixed token; closes the other three — and must be evaluated ahead of the entry regex (§9 step 2) or it inherits M1 and never fires |
| Rule 0 marker hand-writing, with #921's read/write split | fixed token set |
| `gh pr merge` verdict gate | **cannot** move to Layer 1 — a merge uses no git transport. The one place a string parse still gates an ALLOW; its **PR-number bind** (`:531`) must lose the `head -1` (deny when the string carries more than one distinct PR number) or §3.4's decoy class survives here — see AC7 and M11 |
| deny `gh api …/pulls/N/merge`, graphql `mergePullRequest`, `curl …/merge`, `gh -R`/`--repo` | closes §3.7 |
| deny `gh api`/`curl` writes to `…/contents/…` and `…/git/refs/…` (any method, any client) | closes §3.9 — the class no other layer can see |
| **`deny()` via `jq -n --arg`** for every message | closes §3.2 |
| empty/null stdin → deny | closes §3.3 |

| delete | replaced by |
|---|---|
| `is_supervisor_docs_only()` and both call sites | Layer 1 rule 2 |
| the whole `PUSH_TOKENS` / `PUSH_POSITIONAL` walk | Layer 1 — git supplies the ref |
| Rule 1 / Rule 1.5 allowlist evaluation | Layer 1 rules 2–3 |
| the `tests-passed` marker scan | Layer 1 rule 3 |
| Rule 0.4 / 0.5 delete handling | Layer 1 rule 1 |

Main-push denial at the string layer becomes **unconditional and non-authoritative**: the guard
either denies on a fixed token or expresses no opinion. It can no longer emit an `allow` that
unlocks anything.

**No feature flag.** This is a cutover: Layer 1 lands and is verified first; Layer 3's deletions
follow only after AC1 and AC8 pass (§9).

---

## 6. ACCEPTANCE CRITERIA

Each carries a measure, a pass value, a **fail** value, and a negative control that separates
"broken" from "no data yet". A criterion that cannot fail is not a criterion.

**AC1 — act-boundary coverage.**
*Measure:* the 10 spellings of §5 against a fixture whose main carries code; record push exit code
**and** whether the bare remote's `refs/heads/main` moved.
*Pass:* the **8 main-targeting** spellings each exit non-zero with `refs/heads/main` byte-identical
before and after, **AND** `git push origin feat/x` exits 0 and moves `refs/heads/feat/x`.
*Fail:* any main-targeting spelling exits 0, or main moves, or the `feat/x` row is denied.
*Today:* the string guard allows 10/10 of the global-carrying forms (§3.1).
*Negative control:* re-run with the hook uninstalled — **8 of the 10 must exit 0**, and the
non-deletion main-targeting rows must move `refs/heads/main`. If the control does not move the ref,
the fixture is not exercising a push and the pass proves nothing.

*The control is 8/10, not 10/10, and the two exceptions are not the hook* `[P]`. Measured on a fresh
bare repo with `core.hooksPath=/dev/null` and `receive.denyDeleteCurrent` unset: rows 01–06, 09 and
10 exit 0, but the two **deletion** spellings — `git push origin --delete main` and
`git push origin :main` — exit **1** with *"deleting the current branch is denied"*. That is git's
own receive-side refusal to delete the branch the bare repo's `HEAD` points at, and it fires with no
hook installed at all. Writing the control as "all 10 must exit 0" makes it **unpassable**, and an
unpassable negative control gets quietly downgraded to "the fixture must be broken" — which is the
precise failure this AC's control exists to prevent, reappearing one level up. Either accept 8/10
with rows 07/08 expected non-zero for a hook-independent reason, or point the fixture's `HEAD` at an
unrelated branch so the deletions can proceed; state which was done.

*Why not "9/9 non-`--no-verify` spellings exit non-zero".* That was this AC's earlier pass value and
it is **unsatisfiable by a correct implementation.** The 9 non-`--no-verify` rows include
`git push origin feat/x`, whose own row in the same table records `0 allowed` — so 9/9 demands that
a legitimate feature push be denied. The only thing that passes it is a deny-everything hook, which
is exactly the failure this AC's negative control exists to catch. An acceptance criterion that
fails a correct build and passes a broken one is worse than no criterion, so the pass value is
stated per class: deny the 8, allow the 1.

*Coverage limit — AC1 does not measure bypass.* It measures §5's 10 spellings and nothing else.
`git -c core.hooksPath=…` is not among them, and it skips `pre-push` outright (§5's gap table), so
**AC1 can pass in full while Layer 1 remains bypassable.** AC1 is act-boundary coverage; the bypass
set is gated by the fixed-token denies of §9 step 2 and must be measured with them. Reading a green
AC1 as "Layer 1 is unbypassable" is the specific misreading this note exists to prevent.

**AC2 — the string guard is never authoritative for allow on a PUSH path.**
*Measure:* static check that no `permissionDecision":"allow"` emission on a **push** path is
reachable after any branch/tree/diff computation.
*Pass:* zero such sites. Main has 8 allow sites (`git-push-guard.sh:262,311,322,354,418,523,639,644`,
as of `7675e93f`); the only ones that may survive are the terminal no-opinion emission (`:644` —
"Allow all other commands", outside both the push and the merge block) and the merge-verdict allow
(`:639`). *Fail:* ≥1 push-path allow reachable after a branch/tree/diff computation.
*Negative control:* re-introduce one docs-only allow branch — the check must go RED.

*Why `:523` is not a survivor.* It is the **tests-passed authoritative allow** — inside the push
block (`:182`–`:525`), reached only after `CURRENT_TREE=$(git rev-parse "${CURRENT_BRANCH}^{tree}")`
at `:395` — so it satisfies this AC's own *Fail* clause, and §5's delete table already orders it
removed ("the `tests-passed` marker scan → Layer 1 rule 3"). The other five are push-path allows for
the same reason: `:262` after the branch inference at `:257`, and `:311`, `:322`, `:354`, `:418`
after a branch or diff computation. Listing `:523` as a "no-opinion emission" made the *Pass* clause
permit exactly what the *Fail* clause condemns.

*Scope, stated because AC7 depends on it.* `:639` is excluded **by construction, not by oversight.**
It is the merge-approved allow that AC7's own negative control REQUIRES to fire, and it is reachable
only after parsing a PR number out of the command string. Worded as "no allow after **any**
parsing", AC2 and AC7 could not both pass: AC2 would demand the deletion of the very site AC7
demands be reachable. The merge gate cannot move to Layer 1 (no git transport), so this residue is
permanent under this design. AC7 is where it is measured.

**AC3 — adversarial corpus.**
*Measure:* ≥40 command strings, each pairing a real push to main with at least one of {quoted
mention, `#` comment, chained second push, git global, line continuation}; run each through the
fixture with Layer 1 installed.
*Pass:* zero **adversarial** rows move the remote main ref, **AND** every control row does. The
string verdict is explicitly **not** asserted — it is no longer load-bearing. *Fail:* any
adversarial row moves the ref, or any control row fails to.
*Negative control:* the corpus carries ≥3 rows that are **legitimate docs-only main pushes, which
MUST move the ref.** If none moves, the corpus proves only that everything is blocked.
*Stated because the two halves otherwise read as a contradiction:* the corpus holds two disjoint
classes and each is scored against its own expectation. "Zero rows move the ref" governs the ≥40
adversarial rows **only** — the control rows are required to move it. Scoring both classes against
a single "zero rows move" is exactly what a deny-everything hook passes.

**AC4 — emitted JSON is valid under adversarial text.**
*Measure:* drive every deny path with a payload whose interpolated field contains `"`, `\` and a
newline; parse stdout with a strict JSON parser.
*Pass:* 100% parse AND decision == `deny`. *Fail:* any unparseable output. *Today:* 3 forms
unparseable, and the command demonstrably runs (§3.2).
*Negative control:* revert one `deny()` to a heredoc — must go RED.

**AC5 — degenerate stdin fails closed.**
*Measure:* empty, `null`, `[]`, `not json`, truncated, `{"tool_input":{}}`.
*Pass:* all 6 → `deny`. *Fail:* any → `allow`. *Today:* 3 of 6 allow (§3.3).
*Negative control:* a well-formed benign payload must still reach `allow`, else the criterion passes
by denying everything.

**AC6 — MCP write path.**
*Precondition, and it is not established:* that an `mcp__` matcher fires at all (§10.2 — the single
largest unverified assumption in this spec). **AC6 cannot be run until a throwaway logging hook has
proved it.** If it does not fire, AC6 is not "failing" — it is *unmeasurable*, Layer 2 is
unimplementable as specified, and the verdict changes. Do not record a pass or a fail without that
sub-step's result.
*Measure:* matcher present in settings; feed each of the 5 write tool names.
*Pass:* the matcher demonstrably fires, AND 5/5 deny. *Fail:* any allow, or no `mcp__` matcher
present. *Today:* no matcher exists.
*Negative control:* a read-only MCP tool (`pull_request_read`) must **allow** — otherwise the
matcher is over-broad and the pass is meaningless.

**AC7 — merge-route coverage.**
*Measure:* the 12 spellings of §3.7, **plus the 2 decoy rows below.**
*Pass:* 12/12 deny absent a current approved verdict, AND both decoy rows deny.
*Fail:* any allow. *Today:* 5/12 allow, **and both decoy rows allow** `[P]`.
*Negative control:* with a valid, current, approved verdict marker, bare `gh pr merge <N> --squash`
must **allow** — otherwise the suite is pinning "deny everything".

*The decoy rows, and why they belong to this AC.* The PR-number bind (the `head -1` on the
`gh pr merge <N>` scan, `git-push-guard.sh:531` as of `7675e93f`) is §3.4's `head -1` class,
unchanged, on the merge path. Measured against an isolated marker dir (`ATLAS_MARKER_DIR`) holding
an **approved** marker for #923 and a **`blocked`** marker for #921 `[P]`:

| row | verdict |
|---|---|
| `gh pr merge 921 --squash` (control) | deny — "PR #921 was reviewed and the verdict is BLOCK" |
| decoy-chain: an approved PR merged first, then #921 in the same command | **allow** |
| decoy-quoted: an approved PR named only inside a quoted commit message, then #921 | **allow** |

An approved PR named first — in a chained merge, or merely inside a **quoted commit message** —
binds the verdict lookup, and **a PR the reviewer explicitly BLOCKED merges under someone else's
approval.** That is the load-bearing form of this defect, and it is strictly worse than the
unreviewed case the earlier wording described: the gate is not merely uninformed about #921, it
holds a recorded refusal for it and issues an authoritative allow anyway. §3.7 counts routes that
reach the merge *ungated*; this is the distinct and worse case where the gate runs, consults the
wrong PR, and overrides its own recorded BLOCK. §3.7's 12 rows contain no decoy, which is why the
class was invisible to the merge-side count. Tabulated as **M11** in §3.10.

*Instrument note — the fixture must carry full 40-char OIDs, or this defect is invisible.* The row
that binds the decision is the **approved** marker, and the approved path is the only one that
reaches the head-staleness check (`gh pr view <N> --json headRefOid`, compared against the marker's
sha). An **abbreviated** sha there can never equal the 40-char `headRefOid`, so the guard denies
with "has new commits since the recorded verdict" — a deny for an unrelated reason that masks the
decoy completely and reads, to a careless eye, as a pass. Round 1's fixture made exactly that
mistake. Any re-measurement of these rows must record the marker sha's length alongside the verdict.

**AC8 — worktree universality.** *(load-bearing — see §9)*
*Measure:* from ≥3 worktrees whose branches were cut **before** the hook landed, attempt a main push
against a fixture remote.
*Pass:* all deny. *Fail:* any allow.
*Measured today:* common `.git/hooks` → all deny ✓; **relative `core.hooksPath` → the pre-existing
worktree ALLOWS** ✗; absolute `core.hooksPath` → all deny ✓. `core.hooksPath` is repo-shared, not
per-worktree.
*Negative control:* hook uninstalled → all must allow.

**AC9 — the suite can actually fail.**
*Measure:* mutation-test every rule: neuter it, run the suite.
*Pass:* every mutation turns the suite RED **and the failing assertion names the mutated rule.**
*Fail:* any mutation leaves it green, or turns it red via an unrelated assertion.
*Today `[A]`:* all 7 runnable suites are green — 252/252, 0 fail — while all eleven §3.10 defects are
open. Only 4 of 8 suites touch this guard at all; only **three syntactic shapes** of push string are
ever fed to it (`git push origin feat/smoke`, `git push origin $branch`, `git push origin main`),
every one a bare single push. Zero decoy cases, zero `#` comments, zero multi-span, zero `git -C`,
zero negative controls. All five `RED-ON-NEUTERED` markers in the suites are **prose instructions to
a human, never executed.** 15 of 15 exemption assertions test the decision value only, never the
reason — so an allow for entirely the wrong reason passes today.
*This AC is the negative control for the whole suite.*

**AC10 — workflow cost (the criterion with a real "no data yet" state).**
*Measure:* for 14 days post-cutover, count (a) docs-only main-push **attempts**, (b) attempts
denied, (c) denials that were legitimate docs-only changes.
*Pass:* (c) == 0 AND (a) ≥ 7 (i.e. the path was actually exercised at roughly the measured 0.57–1.00
/day). *Fail:* (c) > 0.
*No-data-yet:* (a) < 7 → the criterion is **UNMEASURED, not passed.** Reporting (b)==0 without (a)
is the trap this AC exists to prevent — the current gate leaves no record of refusals, which is why
§4's population cannot contain them. **Layer 1 must log every decision, allow and deny, or AC10 is
unmeasurable by construction.**

---

## 7. PR #921 — WHAT TO SALVAGE, WHAT TO DROP `[A]`

First, an artifact caveat that affects citations: `pr921-review/old-guard.sh` (34,447 B) is **not**
main's guard. `pr921-bypass/main-guard.sh` (34,837 B) is, and is byte-identical to the live file.
They differ by a 5-line comment at line 91, so `old-guard.sh:N ↔ main-guard.sh:N+5` for N ≥ 92.

### Numbers from the brief, re-verified

| claim | verdict |
|---|---|
| "86 probes" | **CORRECT** — 70 + 10 + 6 = 86, matching the TSV row counts exactly |
| "closes 20 forms main allows" | **CORRECT** in the no-marker scenario; **27** with a marker present |
| "fixes an invalid-JSON fail-open" | **CORRECT but mislocated.** Not the stdin path — main handles malformed stdin correctly. The fail-open is the guard's **own emitted** JSON (§3.2) |
| "none of the 5 baseline fail-open defects is closed" | **CORRECTED.** C1, C2, C3 and I4 are **PR-only** defects — main has no repo-redirect mechanism at all, and C2/C3 belong to `ansible-gate-guard.sh`, which on main is `Edit\|Write`-only. Only the `--delete`+line-continuation defect is present in **both** |
| brief: "all 7 hook suites pass green" | **CORRECTED.** There are **8** suite files. 7 ran (252/252 pass). The 8th, `run-pr-verdict-smoke.sh` (29 assertions), **cannot be run safely** — it hardcodes the live `/tmp/atlas-test-markers` at `:25` and writes into it at 9 sites, and unlike the guard it has no `ATLAS_MARKER_DIR` override |

### SALVAGE — take independently, zero coupling to the parser

1. **`deny()` via `jq -n --arg`** (`new-guard.sh:254-269`) plus every heredoc→`deny()` conversion.
   Closes the §3.2 fail-open on both the branch-name path and the reviewer-BLOCK-reason path. The
   pattern is already precedented on main in `ansible-gate-guard.sh:95-99`. **Take this first — see
   §9.**
2. **Rule 0 read/write regex split** (`:335-343`). Pure false-deny removal (M9); writes stay blocked.
3. **Alternate merge routes** — `GH_API_MERGE_RE` `:201`, graphql `:208`, `CURL_MERGE_RE` `:211`,
   deny block `:816-827`. These gate acts main does not gate **at all** and depend on no span
   machinery. Directly satisfies AC7.
4. **Rule 0.45 `--mirror`/`--all`** (`:556-574`) and `${LOCAL_REF#refs/heads/}` (`:505`) — small and
   self-contained, **but take BOTH only if Layer 1 is deferred.** Layer 1 retires them together:
   git normalises the ref (`:505`), and it enumerates `--all`/`--mirror` into one ref line per ref,
   main among them, so there is no flag left to recognise (§5, measured). Under Layer 1 these are
   not merely redundant — re-adding them would reintroduce flag-recognition into a layer whose whole
   point is that it recognises nothing.

Abandoning these would be a regression of its own — items 1 and 3 close live fail-opens.

### DROP

- **The repo-redirect mechanism** (`:448-454` + `GIT_C_ARGS` at `:516, 676, 687, 689, 694`). Its
  line 442 comment claims "Repo redirection comes from THIS span" while `:449` reads `$COMMAND` —
  the whole raw string — with `head -1`. It converts main's *silent misses* into **authoritative
  allows** on 5 probes. Layer 1 makes it unnecessary: git states the repo.
- **The `RAW_PUSH_N` backstop** (`:392-408`) without comment stripping — unactionable false denies
  (I4).
- **`ansible-gate-guard.sh`'s new Bash coverage — drop the MECHANISM, keep the OBJECTIVE.** What
  fails is #921's implementation: C2 and C3 let through the exact write class it exists to stop, and
  its shipped test shape does not exercise either. The objective is untouched by that and remains
  unmet. Post-#925 (`ca67ea5a`) the guard has `canon` (`:69`) and `is_deployed_path` (`:207`) but is
  still wired to `Edit|Write` only, so `sed -i … /opt/ai-inference/compose.yaml` is gated by
  **nothing**.

  State the obligation accurately: #925 records this as a **KNOWN GAP, deliberately deferred** —
  `ansible-gate-guard.sh:26-32` and `.claude/hooks/README.md:454-461` both name it and both give the
  same reason, that closing it means deciding from a command string which token is a write target,
  and it "is deliberately not in this change, so that a parser regression cannot take this
  `Edit|Write` enforcement down with it." That is a recorded gap with a stated rationale. **It is
  not a formal obligation on #921, and #925 must not be cited as imposing one.** Whoever picks the
  objective up inherits #925's constraint, which is also §1's: it is command-string parsing, so it
  must be a fixed-token deny resolving ambiguity to DENY — never a write-target inference that can
  return an allow.
- **The entire span-loop / canonicalisation programme** (`:375`, `:390`, `:433-798`). Not because it
  is bad work — it is careful work — but because Layer 1 obsoletes the problem it solves. This is
  the single largest deletion and the point of the redesign.

---

## 8. IF THE USER UPGRADES TO GITHUB PRO — treated as a variable `[U]`

Nothing above assumes it. If it happens:

| gains | still required |
|---|---|
| A ruleset can enforce main-push denial **server-side**, surviving any local bypass. Layer 1 becomes defence-in-depth rather than the boundary. | Layer 1 still earns its place: it denies *before* the network round-trip and gives an actionable local message. |
| A ruleset restriction covers the MCP write tools and `gh api`/`curl` alike, since all terminate at GitHub. Layer 2's exposure shrinks to a usability concern. | Layer 2 stays until measured otherwise — see §10.2. |
| — | **The merge gate does not improve.** Required approvals must stay 0 to preserve autonomous operation, and GitHub cannot express "approved by a reviewer who is not the author" with 0 required approvals. §5 Layer 3's verdict gate remains the only merge control, with the honesty caveat below. |

**The merge gate's floor, stated plainly.** `scripts/claude-pr-verdict`'s own header already says it:
the pending record is an ordinary `/tmp` file forgeable with a single Write; backdating defeats the
60-second check; the 20-char reason check measures length, not content; and nothing prevents one
agent being both reviewer and approver. What the mechanism genuinely delivers is that an approval
becomes a **deliberate, separately-typed, logged act** rather than a side effect of invoking the
review skill — the specific failure that let #908/#911/#913 merge. This spec does not improve that
floor and should not be read as claiming to. It only ensures the gate is not **additionally** voided
by its own malformed JSON (§3.2), which today it is.

---

## 9. SEQUENCING — and which edges are load-bearing

| # | step | depends on | load-bearing? |
|---|---|---|---|
| 1 | `deny()` JSON escaping + degenerate-stdin fail-closed (AC4, AC5) | — | **YES** |
| 2 | **M1 entry-regex repair** — hoist the fixed-token denies ahead of the entry condition (§3.1, §5) | 1 | **YES** |
| 3 | Rule 0 read/write split (M9) | — | no |
| 4 | Layer 2 `mcp__` matcher — **prove the matcher fires first** (AC6, §10.2) | — | no |
| 5 | Merge-route additions incl. §3.9's `contents`/`git refs` class and AC7's decoy rows | 1, 2 | no |
| 6 | Layer 1 `pre-push` + **absolute** `core.hooksPath` + the four-spelling bypass denies (AC1, AC8) | 1, 2 | **YES** |
| 7 | Layer 3 demotion — delete the allowance and all push parsing (AC2, AC3) | **6 verified** | **YES** |
| 8 | Mutation-test the suite (AC9); begin AC10's 14-day window | 7 | no |

**Edge 1 → everything. Load-bearing.** Until the emitted-JSON fix lands, *any* deny added afterwards
can be silently voided by its own message, and no subsequent measurement is trustworthy. An
unparseable deny is an absent deny; you cannot honestly measure a gate whose failures are invisible.

**Step 2 — the M1 repair, newly scheduled, and why it is load-bearing now.** M1 (§3.1) is the oldest
defect in the set and no round from #918 to #921 ever scheduled it for repair. That was survivable
while Layer 3 still did real work by other routes; it stops being survivable at step 7, because the
demotion removes everything that was incidentally covering for it. After the demotion Layer 3's
*entire* remaining job is fixed-token denies — `--no-verify`, `core.hooksPath` mutation, the merge
and `gh api`/`curl` routes, Rule 0 — and **every one of them is dead code on any command the entry
regex does not match.** `git -c core.hooksPath=/dev/null push origin main` is the case that matters:
it defeats Layer 1 *and* Layer 3 today (§5), and a `core.hooksPath` deny written inside the
`git\s+push` block would not fire on it either. Concretely: evaluate the fixed-token denies against
the whole command string, before the entry condition, so they cannot inherit M1. Nothing else
backstops the entry regex once step 7 lands.

**Edge 6 → 7. Load-bearing, and the classic cutover trap.** Step 7 deletes the only gate that exists
today. If it lands before Layer 1 is *verified* — not merely written — main is completely
unprotected in the interval. AC1 and AC8 must both pass before step 7 begins. This ordering is not
stylistic.

**Within step 6: absolute vs relative `core.hooksPath`. Load-bearing.** `[P]` A **relative**
`core.hooksPath` pointing at a tracked directory **fails on any worktree whose branch was cut before
the hook landed** — the directory simply is not there, and the push is ALLOWED. ATLAS runs 11 live
worktrees at the time of writing (the count churns by the hour as agents are dispatched and reaped —
treat it as an order of magnitude, not a constant), most on branches predating this work. Two
install sites pass AC8: the common
`.git/hooks/pre-push` (untracked, invisible to review) and an **absolute** `core.hooksPath` pointing
at a tracked directory in the main checkout (version-controlled **and** universal *across
worktrees*). Take the latter — with its limit stated, because "universal" overstates it:

**An absolute host path does not resolve inside a devcontainer.** 11 `.devcontainer/compose.yaml`
files bind-mount the monorepo as `../..:/workspace` `[P]`, so a hook installed at
`/home/james/ATLAS/<dir>/pre-push` is simply absent at that path inside the container — while
`core.hooksPath`, being repo-shared config, still points there. Any push issued from a devcontainer
is therefore **ungated by Layer 1**, silently and in the allow direction. Whether that matters turns
on whether agents push from containers or from the host; this spec assumes neither, and **AC8 must
add a from-container row** so the answer is measured rather than assumed. If they do, the install
needs a container-resolvable path (a repo-relative directory that exists on every branch, which
reintroduces the relative-path problem above) — that tension is unresolved here and should not be
discovered during the cutover.

**Within step 6: the whole Layer-1 bypass set must ship in the same change** `[P]`. Not one flag —
four spellings skip `pre-push` entirely (§5): `--no-verify`, the `GIT_CONFIG_COUNT` env form, and
`git -c core.hooksPath=` in both its `/dev/null` and its `/nonexistent` variants. Two of the four
are already ungated by the string guard as well, so shipping Layer 1 without them leaves main
exactly as reachable as it is today via `git -c core.hooksPath=/dev/null push origin main`. And
these denies must be evaluated **ahead of the entry regex** (step 2), or they inherit M1 and never
fire on the very spellings that carry a `-c`.

**Step 4 is independent of everything** and addresses the largest ungated surface (§3.8), so it
should not wait on the git work. **It does not close that surface on landing.** §10.2 is the single
largest unverified assumption in this spec, and step 4's own first sub-step is to discharge it: wire
a throwaway logging hook on an `mcp__` matcher and confirm it fires. If it does not, Layer 2 is
unimplementable as specified, the MCP surface stays open, and the overall verdict changes. Recording
the surface as closed before that sub-step returns would be the same corpse-detector error this spec
objects to elsewhere — a check that reports success without having exercised the thing it measures.

---

## 10. WHAT COULD NOT BE VERIFIED — mandatory

1. **Harness behaviour on hook timeout.** The guard is wired with `"timeout": 5`. Whether a killed
   hook fails open was not probed; doing so means hanging a hook that every live agent's Bash calls
   pass through. `.claude/hooks/README.md` asserts it is fail-open "by harness semantics" — I did not
   confirm it. `[U]`
2. **Whether an `mcp__` matcher actually fires.** *This is the single largest unverified assumption
   in the proposal — Layer 2 rests entirely on it.* The README reasons that a matcher containing
   characters outside `[A-Za-z0-9_\- ,|]` is treated as an unanchored regex, so `mcp__.*` should
   work; but **no `mcp__` hook exists on this host to test**, and I did not add one. **Step 4 of §9
   must begin by proving this with a throwaway logging hook.** If it does not fire, Layer 2 is
   unimplementable as specified and the MCP surface stays open — which would materially change the
   overall verdict. `[U]`
3. **Whether `pre-push` covers MCP-initiated writes.** Almost certainly not — the GitHub MCP server
   uses the REST API, not git transport — but I did not inspect its implementation. `[U]`
4. **Everything about the GitHub Pro path (§8)** is inference from documentation. Both endpoints
   return 403, so no ruleset behaviour could be observed. `[U]`
5. **Harness behaviour on a valid JSON body with an unknown `permissionDecision` value.** Not
   probed. `[U]`
6. **GitHub-side recovery of a force-deleted `main`.** Not testable without doing it. `[U]`
7. **The post-untracking push rate.** The untracking itself has **landed** (`9ed5709a` / #922 —
   see §2.2), so the regime is no longer hypothetical. What remains unverified is its *rate*: it
   landed too recently to have generated one, and §4's survivor counts are a re-classification of
   history with `STATE.md` removed from each commit's file set, not an observation of the new
   regime. AC10's 14-day window is the thing that would measure it. `[H]`
8. **`run-pr-verdict-smoke.sh` was not run** (29 assertions unmeasured) because it writes into the
   live marker directory with no override. Its current pass/fail state is unknown. `[A]`
9. **Whether the 156 "direct-push shaped" commits were all actually direct pushes.** The classifier
   keys on the absence of a `(#N)` subject suffix; a squash-merge with an edited subject would be
   miscounted. 152 of 156 are docs-only under the live regex, which is consistent, but not proof.
   `[H]`

**Live finding, incidental but worth surfacing — and since expired.** During measurement,
`/home/james/ATLAS/.claude/.ansible-gate-confirmed` existed with that day's mtime (~09:21, TTL 4h)
`[A]`: the ansible gate — the mechanical enforcement behind CLAUDE.md's DEPLOYMENT HARD_STOP — was
bypassed on this host at the time. **The file no longer exists** (re-checked: no such file), so the
gate is armed again and the bypass was transient, exactly as its TTL intends. Do not read the
original wording as describing a standing condition. It is retained because of *how* it surfaced —
it silently altered a probe result mid-measurement, which is the general hazard worth carrying
forward: a confirm-file with a TTL is a state a measurement can sit inside without noticing, and
nothing in the probe output says which side of it you are on.

---

## 11. ROLLBACK — and whether it is real

**The ZFS floor does not cover this asset, and the reason is not the playbook.** `[P]`

```
findmnt -no FSTYPE,SOURCE,TARGET /home  ->  ext4  /dev/nvme5n1p4  /home
```

`/home/james/ATLAS` is on **ext4**. Every ZFS dataset on this host is under `/opt/ai-inference/*`,
`/nvme-fast` or `/sata-bulk`; `zfs list -t snapshot` shows **no snapshot covering `/home`**. That is
the load-bearing fact and it is unchanged: **no snapshot covers the git repository, so ZFS is not a
rollback path for anything in this spec.** Do not cite it as one.

The ZFS repair is no longer "in flight" — it merged as `4dfee35d` (*"fix(deploy): repair the ZFS
rollback floor and stop it reporting false success"*, #924). It is irrelevant *to this spec*, but
the earlier phrasing "irrelevant to this work in either direction" understated what sits on the
other side of that line: the datasets it repairs include `nvme-fast/timeseries`, which is
TimescaleDB's PGDATA (`/opt/ai-inference/compose.yaml:318` mounts it at
`/var/lib/postgresql/data`). The ZFS floor protects the production database; it simply never
extended to `/home`. "Irrelevant here" is not "irrelevant".

**What is real:**

| asset | rollback | real? |
|---|---|---|
| the hook/script files | ordinary tracked files; `git revert` | **yes** |
| `core.hooksPath` | **local config, NOT tracked.** A `git revert` of the hook does **not** restore prior behaviour — the config must be unset by hand | **partial — name this in the runbook** |
| a bad commit on main | `git revert` via a PR | yes |
| a force-pushed/lost main | local reflog — 723 entries on `main`, but the window is **30 days, not 90** `[P]` | **local only, agent-mutable, and half the assumed window** |
| server-side ref history | none — no protection, no rulesets (§2.1) | **no** |

**The reflog row, corrected.** An earlier revision cited git's 90-day default. That is
`gc.reflogExpire`, which governs entries **reachable** from the ref. A force-pushed or lost tip is
by definition *unreachable*, so `gc.reflogExpireUnreachable` governs instead — and it is **unset on
this repo** `[P]`, giving git's 30-day default. The recovery window for the exact case this row
describes is therefore half what was claimed.

The `core.hooksPath` row is the one that bites: it is the only piece of this design that lives
outside version control, and reverting the commit leaves it set. The install step must be
idempotent and paired with a documented uninstall.

---

## 12. ALTERNATIVES CONSIDERED AND REJECTED

| option | why rejected |
|---|---|
| **Keep tuning the parser (the #918→#921 path)** | Six rounds, five regressions, all from one place. §3.1 is a *non-parse*: no accuracy improvement reaches `git -C`, and §3.8 is not a string at all. The parser cannot reach the act. |
| **Delete the allowance outright** (the brief's hypothesis) | Correct about the parser-side allowance; wrong as a whole. §4: 50 commits / 18 days survive untracking, two classes of them mandated routine steps. At the act boundary the allowance costs nothing to keep, so the trade the hypothesis assumes does not exist. |
| **A blessed script minting a one-shot token** | Considered and rejected as unnecessary. Layer 1 computes docs-only-ness from `rsha..lsha`, which git hands it — no token to forge, no script to invoke, one fewer artifact. It would have been the right answer if `pre-push` did not receive the refs; it does `[P]`. |
| **Server-side branch protection** | Unavailable — 403, verified twice (§2.1). Revisit per §8. |
| **Requiring PR approvals** | Must stay 0 to preserve autonomous operation (user constraint). |
| **A feature flag / default-OFF rollout** | Prohibited by user rule, and it would be actively harmful here: a default-OFF gate is an unwired path, and §9's edge 6→7 already requires a verified sequence rather than a toggle. |

---

## 13. THE ONE-LINE ANSWER TO THE USER'S QUESTION

The hook was built to observe a **string**. The intent is about an **act**. Every defect in §3 is
that gap, and every round of parser work has been an attempt to reconstruct the act from the string.
`pre-push` is handed the act. Move the decision there, let the string guard keep only the checks
that are single fixed tokens, and take away its ability to say yes.
