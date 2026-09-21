# scripts

Repo-root operator scripts. Mix of Claude Code helpers, ad-hoc auditing harnesses, and per-feature subdirectories with their own READMEs. Most of these are invoked directly by the operator (you); a few are referenced by `.claude/hooks/` or systemd timers.

## Files

| File | Purpose |
|---|---|
| `claude-mark-verified` | Manual "tests passed" marker for the `git-push-guard.sh` hook when changes are not amenable to a `compile.sh` validator (YAML, Markdown, dashboard JSON, shell). Writes a `v2-manual` marker that unblocks the push and logs an audit line. See `CLAUDE_MARK_VERIFIED.md` for full usage. |
| `CLAUDE_MARK_VERIFIED.md` | Companion doc for `claude-mark-verified` — when to use, when **not** to use, audit-log format, hook behaviour. |
| `devcontainer-owner.sh` | Per-agent ownership of devcontainer verification runs, sourced by every **container-starting** `.devcontainer/compile.sh` plus sentinel-edge's `typecheck.sh`/`dev.sh` — a container-less one (FinBertSidecar, pure python) owns nothing and cannot collide, and that exemption is NOT a gap (§DEVCONTAINER_OWNERSHIP). Each run gets its own compose project `atlas-<sha1(worktree)[0:12]>-<slug>` — the same key `mark-tests-passed.sh` uses — so containers, networks and the nuget volume are all per-owner and N agents verify at once. Previously container names were identical across worktrees (no `container_name:`, relative `/workspace` mounts), so a run could silently test another worktree's source and still write a push marker; this **supersedes** the host-wide flock that fixed that by serializing everything. It also resolves the repository's git **common dir** and hands it across the `sudo` boundary for the compose files to mount at its own host path: a linked worktree's `.git` is a pointer file naming a path under the MAIN checkout, which no `/workspace` mount carries, so without it `git ls-files` exits 128 inside the container and any tracked-tree sweep in a unit suite fails — in worktrees only. Also provides `devcontainer_verify_workspace` (inode proof that `/workspace` is the caller's own tree, gating the marker) and a reaper that removes `atlas-*` state whose worktree no longer exists — which is what covers SIGKILLed runs, since no trap can. |
| `test-devcontainer-owner.sh` | Guard test for the above (~3s, no containers): the project key matches the marker key, a foreign tree at `/workspace` is refused, `mark-tests-passed.sh` refuses a missing/stale/foreign attestation, **the reaper acts only on positively-identified absence and declines when it cannot tell** (including when run from a foreign repo), teardown fires exactly once, every compose-driving script owns *before* touching containers and verifies *before* building, no verification-path compose file pins a host port, and every base compose file sets a project name. No assertion count is quoted here on purpose — several sections scale with the file set, so a fixed number is drift waiting to happen; the suite prints its own totals. Discovery is depth-agnostic (`git ls-files`) and globs `compose*.ya?ml`, backed by a count floor and a declared-exception list, so neither a too-shallow glob nor a filter that matches nothing can pass silently. A script that starts **no** container is exempt on that observable property rather than by filename. Two roster assertions replace the count floors on the real tree — the population the loop WALKED, by name, and the exemptions it TOOK — because a floor with slack absorbs a path-scoped exclusion silently, and no planted fixture can match an excluded real path. The known-bad control drives the real enumerate-and-loop over a **planted tree** carrying one fixture per audit branch (an intact copy per selector arm; ownership stripped; container calls stripped; the compose form both plain and flag-layered; the `/workspace` verify removed, and separately moved AFTER the exec; the ownership call moved after the first container start; the marker path hardcoded) and scores three things: the one-to-one correspondence between enumerated names and emitted lines, each branch's own required line for **presence and for order**, and that every planted fixture is named by a required line or by a named assertion — a fixture scored by nothing, or a branch with no required line, is one a reader can delete for free. What the audit still cannot see is that it reads TEXT: an ownership call present but unreachable, or written early and *called* late, passes (`docs/BACKLOG.md` KNOWN DEFECTS). `--with-containers` chains the simultaneity proof. |
| `test-devcontainer-simultaneity.sh` | The real-container proof (~2 min): two different services from two worktrees concurrently, then the *same* service from two worktrees concurrently, asserting each run execs into its own `/workspace` and gets its own marker — plus a mutation that points one run at another's container and requires the refusal to fire. Section E then proves the suite's own `ATLAS_MARKER_DIR` redirect cannot widen the real push gate, end-to-end through the hook in both directions: a marker written under the override is honoured by a guard reading that directory and refused by one running with the default environment. It commits a nonce first so the branch's tree cannot be covered by any other marker on the host — otherwise a legitimate marker for this very branch would decide the assertion. Documents in-file what it cannot cover: passing runs sample interleavings, they do not prove the race is gone; the mutation and the disjoint-names argument carry that. |
| `new-epic.sh` | Resets `STATE.md` for a new epic from `.claude/skills/supervisor-mode/templates/STATE-scaffold.md`. STATE.md is gitignored — no undo, no git history — so the script archives it to `~/atlas-ops/state-archive/` and proves the copy identical with `cmp` *before* overwriting, then strips the scaffold header into a temp file **beside** STATE.md (same directory, so the final step is a real atomic rename; `/tmp` and `/home` are different devices here and a cross-device `mv` truncates the live file first), verifies the result is a complete usable body — including that it is byte-identical to the scaffold text after the sentinel — and only then renames it into place. If that verification fails the live file is never touched; the archive restore is the backstop for a failed *write*, and it refuses loudly if the restore itself fails rather than claiming it succeeded. It also audits the outgoing file for **four of the five bans in the WRITE_GATE** (canonical roster: `.claude/skills/supervisor-mode/templates/STATE-scaffold.md` — not re-enumerated here, since a fresh copy in a non-canonical file is the drift this tooling exists to stop; the one it cannot grep is judgement, not a pattern), plus migrated section headings and a non-blocking bloat note, and **refuses the reset while any matches**, except for lines that are verbatim scaffold boilerplate; `--evicted` overrides, `--check` audits and writes nothing. **The audit cannot judge durability** — that is judgement, routed by `CLAUDE.md` §WHERE_WORK_LANDS — so a clean audit means those patterns did not match, nothing more. Runs a known-bad control on every invocation: the strip verifier is re-proved against **two** deliberately broken scaffolds (sentinel removed; header marker after an intact sentinel), each proved to have been *built* before its verdict is scored. rc 1 = a finding or refusal, rc 2 = usage, environment, or an unsatisfiable precondition. |
| `verify-citations.py` | Resolves `file:line` citations in tracked Markdown (cards, `CLAUDE.md`, D-entries) and, with `--memory`, in the out-of-tree memory corpus that no `git ls-files` invocation can name. Findings are UNRESOLVED (including ambiguous basenames), REVERSED, OUT-OF-RANGE and BLANK. **Content-blind by design**: it checks that a line exists and is non-blank, never what it says, so a citation that has drifted onto a comment, a brace or an unrelated task reads GREEN, and ranges are blank-checked at the START only. A green sweep is not proof a card is sound. |
| `verify-pointers.py` | Resolves every `<path>.md` §CONSTRUCT anchor pointer in the files given to it. The house rule is to cite an ANCHOR rather than a `file:line` (`LESSONS.md` GRADUATION_RULE), which had moved every cross-document pointer OUT of any sweep: `verify-citations.py` resolves `file:line` and nothing else. It decides exactly two things -- the named file exists and is UNAMBIGUOUS, and the named construct begins some line of it. Known-bad control on every run (rc 3, no report, if it fails). **Run it the way CI does**, `python -m pytest scripts/tests -k tracked_corpus`, not by hand. What it CANNOT see, and the one case for invoking it directly: §POINTER_SWEEP below -- stated ONCE, there, because two copies in one file drift in one. |
| `verify-hosted-service-pins.py` | Keeps every `AddHostedService<T>` registration either PINNED by a test that goes RED when the line is removed, or FILED as unpinned with a measurement. The surface it watches is invisible to an ordinary suite: a worker's logic can be correct, tested and green while the one line that makes the host START it is gone — measured 2026-09-20, **50 of 53 live registrations deleted with every suite green**, which is the round PR #1073 lost. Two halves with very different costs. The **check** (default, no arguments) is pure static enumeration — no dotnet, no container, no dependency beyond the standard library, sub-second over the whole repo — and decides GROWTH only: a registration that is NEW, whose statement or recorded verdict CHANGED, or that VANISHED. The frozen set stays quiet, because a check that reddens CI on day one gets switched off (#1083). STALE is the quiet win: it turns the silent deletion of a registration nothing pins into an explicit baseline edit a reviewer sees. The **sweep** (`--sweep <Service>`) is the measurement — it removes registrations, drives the service's real `compile.sh`, and reads the verdict from dotnet's own summary line, never from the exit code (compile.sh works *after* the tests, and an rc-keyed verdict scored a 73/73 GREEN CalendarService run as PINNED). It proves the suite GREEN before it mutates anything, so a service carrying one unrelated broken test yields NO VERDICT rather than a whole set of meaningless PINNEDs. It removes a service's whole set in one run, because a suite still green with all of them gone observes none of them; only a reacting service pays for per-site runs. Known-bad controls run on **every** invocation, not just under pytest. What it CANNOT see is in its docstring and is the first thing to read: only `AddHostedService<T>`, so the factory overload, `AddSingleton<IHostedService, T>`, Quartz jobs, `AddMeter`/`AddSource`, middleware and anything reflection-registered are invisible — and PINNED means *some* test reacted, never that the test asserts anything useful. §HOSTED_SERVICE_PINS below. |
| `agent-stall-watchdog.sh` | Per-agent stall detector for supervisor sessions. Given a tasks dir (the `.output`-symlink directory under `/tmp/claude-<uid>/…/<uuid>/tasks`), reports each subagent transcript whose mtime is older than N minutes. Annotates `PROMPT?` when the tail contains `"stop_reason":"tool_use"` (agent proposed a tool, harness is paused waiting for approval). Suppresses false positives with `BUSY-COMPILE` when a compile/build process is found for the agent's worktree. Designed to run every supervisor poll cycle; output fits in <10 lines for a healthy session. |

## DEVCONTAINER_OWNERSHIP

Canonical home for the verification-run ownership model — `CLAUDE.md` §VERIFY carries the imperative
("N agents in N worktrees compile SIMULTANEOUSLY; never sequence them, never wait") and points here.

- **The key.** Every container-starting `.devcontainer/compile.sh`, plus sentinel-edge's `typecheck.sh`
  and `dev.sh`, sources `devcontainer-owner.sh` and takes the compose project
  `atlas-<sha1(worktree)[0:12]>-<slug>` — the same key `mark-tests-passed.sh` uses, which is what ties a
  marker to the tree that earned it. A container-less `compile.sh` (FinBertSidecar, pure python) owns
  nothing and cannot collide; that exemption is **not** a gap.
- **The attestation.** `compile.sh` proves `/workspace` is its OWN tree by inode match.
  `mark-tests-passed.sh` refuses to write a marker without that attestation, and refuses one more than
  3h old.
- **Cleanup.** Teardown on `EXIT`, plus a reaper on every start that removes `atlas-*` state whose
  worktree is gone — SIGKILL cannot be trapped, so the trap alone would leak.
- **No verification-path compose file publishes a host port** — these are exec-only. Two DECLARED
  interactive exceptions: sentinel-edge's `compose.ports.yaml` (8787) and WhisperService's
  `compose.dev.yaml` (8090). `test-devcontainer-owner.sh` asserts the rule and the exception list
  together, so adding a third port without declaring it turns the suite red.
- **Why:** before this, container names were identical across worktrees, so a concurrent run could
  silently test *another* worktree's source and still write a push marker.
- **Why per-WORKTREE and not per-INVOCATION.** A per-invocation project name was the rejected
  alternative, and the three objections raised against it were re-tested on this host rather than
  inherited (measured 2026-08-06; the numbers live in `devcontainer-owner.sh`'s header, not here).
  Only ONE holds: a SHARED nuget volume really does corrupt, because NuGet's cross-process lock is
  container-local, so two containers sharing a packages volume tear an extraction on a cold
  concurrent restore — and that is closed by giving each owner its own volume
  (`ATLAS_DEV_NUGET_VOLUME`), which is a property of the KEY EXISTING, not of its granularity. The
  other two do NOT hold, and neither should be re-raised as a reason: a shared `Events/` `obj` is
  not a cross-worktree hazard at all (every worktree has its own `Events/`; the UID collision below
  is same-worktree and is not a race), and "~1.5 GiB rebuilt and leaked per run" mistook a re-TAG of
  a content-addressed image for a rebuild — a second project name added zero bytes at an identical
  digest, and the one-time materialisation cost is paid by ANY project name and is reclaimable.
  What per-worktree buys that per-invocation cannot is the reaper: it identifies an owner by a
  worktree that either exists or does not, and pre-scheme names it cannot match are still on this
  host as fossils.

**KNOWN, and not a concurrency bug:** two DIFFERENT services in ONE worktree collide on
`Events/src/*/obj` by UID. The root-user devcontainers (AlphaVantageCollector, CalendarService,
FinnhubCollector, NasdaqCollector, Reports) versus `vscode`/uid-1000 produce
`Access to the path '/workspace/Events/.../obj/<guid>.tmp' is denied`. Recovery:
`sudo rm -rf <worktree>/Events/src/*/{obj,bin}`. Serializing does **not** fix it — it is file ownership,
not a race.

## TEST_FILTERS

Why a filtered `dotnet test` run can test nothing and still exit 0 — `CLAUDE.md` §VERIFY carries the
working form and points here. This belongs with the devcontainer verification flow above: the filter runs
inside the devcontainer, because `dotnet` exists on the host too and a bare `dotnet test` silently becomes
a host run against host-owned `obj/`.

xUnit exposes `DisplayName` and `FullyQualifiedName` — **never `Name`**. A `--filter 'Name~X'` matches ZERO
tests and STILL EXITS 0, so a run that tested nothing reads as a pass.

Four test projects set xunit `methodDisplay=method` — enumerate them, never recall them:

```bash
git grep -l '"methodDisplay": "method"' -- '*xunit.runner.json'
```

In those four, `DisplayName` is the bare method name, so `DisplayName~<ClassName>` ALSO matches zero tests
and exits 0. Filter a class with `FullyQualifiedName~<ClassName>`, which is correct in every project.

## POINTER_SWEEP

What `verify-pointers.py` CANNOT see — the single statement of it; the Files row above deliberately does
not repeat these. `CLAUDE.md` §TOOL_UPKEEP carries the CI-gate identity and the run-it-the-way-CI-does
rule and points here. **pytest is NOT installed on this host — use a venv.**

- It is a **NAME resolver, never a drift detector.** A pointer aimed at a construct that still exists
  and no longer says what the citing prose claims reads GREEN. Only UPPER-CASE construct names are
  parsed, because an upper-case run has an end a parser can find and a Title-Case English heading
  does not.
- A Title-Case heading (`§API Endpoints`) is **out of scope and UNCOUNTED**, so a run's
  "0 cannot resolve" is never a claim about those.
- Lines inside a fence are not constructs — a mermaid edge used to satisfy a pointer aimed at a heading.
- **It has no floor and no body check.** Deleting a section's BODY under a kept heading passes at rc 0,
  and `git rm --cached` on a destination drops the checked count silently at rc 0. A deletion test
  against it proves the ANCHOR is load-bearing, never the RULE (`docs/BACKLOG.md`).
- **The one reason to invoke it directly** — a file set the gate's tracked corpus does not cover, such
  as an untracked or out-of-tree draft. **NAME THE DRAFT ON THE COMMAND LINE.** `git ls-files` lists
  tracked and STAGED paths only, so the form below WITHOUT a trailing path sweeps everything except the
  one file you invoked it for, and reports a clean run — and this tool's clean state is rc **0**, with no
  rc-1 steady state to muddy the false green (measured 2026-09-21 at `7c94166b`, an untracked draft in
  the worktree root carrying one unresolvable pointer: the path-less form gave `209 file(s) swept, 158
  anchor pointer(s) checked, 0 cannot resolve` at rc 0 and never named the draft; the form below gave
  `210 swept, 159 checked, 1 cannot resolve` at rc 1; handed the draft alone it gave `1 cannot resolve`
  at rc 1, so the finding was always there to miss). It is never a substitute for the pytest run above:
  `mapfile -d '' F < <(git ls-files -z '*.md'); python3 scripts/verify-pointers.py "${F[@]}" path/to/draft.md`.

## HOSTED_SERVICE_PINS

The cost argument, the two ways to close a finding, and what the gate CANNOT see — the single statement
of it; the Files row above does not repeat the blind spots, and neither does `CLAUDE.md` §VERIFY, which
carries the rule and points here. **pytest is NOT installed on this host — use a venv.**

**Why CI pays nothing.** Measuring whether a registration is pinned means deleting it and running that
service's suite: 30s to a few minutes per run, and FredCollector and OfrCollector need a gitignored
`.env` no CI runner has. So the sweep is on-demand and the gate is static. The gate names the ONE service
to sweep, so the expensive half is scoped to the composition root you actually touched rather than to all
53 sites across 10 services.

**ADVISORY, not enforcement.** Branch protection returns 403 on this plan (verified 2026-08-12; the
evidence is in `.github/workflows/alert-rules.yml`), so neither this gate nor any other check here can
block a merge — a red run can only be SEEN. Read a green check as a report that was produced, never as a
gate that held. The same sentence is in `CLAUDE.md` §VERIFY and in the workflow, because a reader who
meets only one of the three takes a passing check for enforcement.

**Two ways to close a NEW finding, and only two.** Pin it — add a test that reads the descriptors the
composition root produced and goes RED when the line is removed (`FinnhubCollector/tests/Services/DependencyInjectionTests.cs`
and `FredCollector/tests/FredCollector.UnitTests/Telemetry/MetricWarmupHostedServiceTests.cs` are the two
worked examples, and the second drives the started service and asserts on the measurements). Or file it —
`--sweep <Service>` then `--freeze`, which records the verdict with the date, the suite result and the
control behind it. Filing is a real answer; an unfiled, unpinned registration is what the check refuses.
**`--freeze` on its own is not filing**: with no sweep result it writes an `UNMEASURED` row, and
UNMEASURED is itself reported — otherwise freezing would buy a green check for a row with nothing
behind it, which is "filed without a measurement" and is exactly what the property forbids.

- **`AddHostedService<T>` is the whole surface, and the gap is MEASURED not hypothetical.** Five live
  sites use the factory overload `AddHostedService(sp => sp.GetRequiredService<T>())` — all three Reports
  hosts, SecMaster's `EdgarIngestionBackgroundService` and FinnhubCollector's `BackgroundCollectionQueue`
  — so **Reports has no keyable registration at all and the whole service is outside the check**. Keying
  that one spelling was rejected rather than deferred: it would read as coverage of the construct while
  `AddHostedService(sp => new Foo())` and every other factory shape stayed silent. Coverage is instead
  counted from the DATA side — every `AddHostedService` token minus the ones keyed, **per token, not per
  line** — and printed as `NOT COVERED` on every run, a clean one included, so an unparseable spelling
  lands there rather than vanishing. Also unwatched, as different constructs rather than spellings:
  `AddSingleton<IHostedService, T>` (zero instances today, checked), Quartz job registration,
  `AddMeter`/`AddSource`, middleware, and anything registered by reflection or assembly scanning.
- **PINNED is not a quality claim.** It means *some* test in that service went RED when the line was
  removed. A test that asserts the registration exists while the component is broken scores PINNED.
- **Conditional registration is invisible.** A line moved inside `if (options.Enabled)` still enumerates
  as one registration; nothing here can tell that it now runs only sometimes.
- **A persistently red suite is caught; INTERMITTENT flake is not.** The sweep proves the suite green on
  an unmutated CONTROL run before it mutates anything, so a service carrying one broken unrelated test
  yields NO VERDICT rather than a whole set of PINNEDs whose evidence reads "went RED with the
  registration removed" — true, and about the wrong failure. Re-running never fixed that, because the
  second reason persists; it helps only against an intermittent failure, which can pass the control and
  then fail a mutation run. A control that reports green having run ZERO tests is also refused.
- **A pinning test deleted after the freeze is invisible.** The registration is unchanged, so its row
  still hashes correctly and still says PINNED. The check guarantees a verdict cannot change *quietly*,
  never that a recorded verdict is still true; only a re-sweep re-measures.
- **A registration inside a string literal counts.** Line and block comments are skipped (one live
  commented-out registration exists, `CalendarService/src/DependencyInjection.cs:53`); string contents
  are not parsed.
- **An interrupted sweep blocks everything until the tree is restored.** SIGKILL cannot be trapped, so a
  killed run can leave `// PIN-SWEEP` in a source file; enumeration then refuses outright rather than
  reading the missing registration as a STALE finding. Observed live on `NasdaqCollector/src/Program.cs`.

## Subdirectories (each with its own README)

| Directory | Purpose |
|---|---|
| `tests/` | Guard tests for the tools in this directory. `new-epic-selftest.sh` breaks `new-epic.sh` one documented way at a time and requires the matching guard to fire **with a message that names it** — including a mutation that kills the strip verifier outright and asserts the inline control notices it, a mutation that truncates the written body, and a check that each mutation actually landed before its verdict is believed. No case count is quoted here on purpose: the suite pins its own (`EXPECTED_CASES`) and prints its totals, so a number here would be one fact written twice. `test_verify_citations.py` covers `verify-citations.py`. |
| `claude-watchdog/` | Background watchdog (`scan.py` + `notify.sh`) that flags long-running Claude Code sessions sitting idle on user input. Publishes to NTFY `atlas-claude-ask`. |
| `sentinel-quality-check/` | Production weekly Sentinel qualitative-extraction quality-check harness (runs via `atlas-sentinel-quality-check.timer`). Also serves as the on-demand A/B audit harness for the F4.6.4 entity-resolution prompt-grounding feature. Renders a Markdown scorecard from a 50-row stratified sample. |
| `gemini-spend-calibration/` | Offline calibration harness for the surface gate in front of the paid Gemini resolver — captures a window of what reached the boundary, replays it through SearXNG and scores the issuer-probe signals. Sets no thresholds. The 11-surface reference draw is committed (`testdata/reference-cache/`), so `--offline` re-checks it byte for byte instead of re-drawing a different population. Stdlib only; `unittest discover` runs the suite (**no pytest on this host**) and `mutation-check.py` says what the suite is worth. |

## When to use which

- **Need to push a docs/YAML/shell-only PR?** Run `scripts/claude-mark-verified "<reason>"` first; the hook will accept the manual marker.
- **Dispatching a subagent for a Matrix epic story?** Templates live in `.claude/skills/supervisor-mode/templates/` (start from `story-implementation.md`).
- **Long-running Claude session stuck on permission prompts?** `claude-watchdog/scan.py` is what fires the NTFY ping; tail its log to debug false positives.
- **Supervisor subagent stalled overnight?** Run `scripts/agent-stall-watchdog.sh <tasks-dir>` to find which agent(s) have stopped making progress; `PROMPT?` flag identifies permission-prompt stalls specifically.
- **The hosted-service pin check REPORTS a registration NEW, CHANGED or GONE?** (It reports; it cannot block — §HOSTED_SERVICE_PINS.) Run `python3 scripts/verify-hosted-service-pins.py --sweep <Service>` for the one service the message names, then `--freeze`. Freezing alone does not close it — that records an `UNMEASURED` row, which the check still reports.
- **Investigating an F4.6.4 prompt-grounding regression?** Re-run the `sentinel-quality-check/` harness against the live extraction stack.

## See Also

- [.claude/hooks/](../.claude/hooks/) — pre-push hook that reads the `mark-verified` marker file
- [deployment/artifacts/scripts](../deployment/artifacts/scripts/README.md) — host-side scripts driven by systemd timers
