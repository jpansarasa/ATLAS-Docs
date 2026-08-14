# BACKLOG

Known defects, deferred follow-ups and parked epics — the things that outlive the epic that found them.

This file exists so `STATE.md` does not have to. STATE.md is disposable working memory for the epic at hand; anything
that must survive that epic belongs here, in a card, in a code comment, or in a PR body — never in STATE.md.

Each entry carries the measurement that makes it re-checkable. An entry with no measurement is an opinion, and the
next agent cannot tell whether it is still true. Close an entry in the PR that fixes it; do not leave tombstones.

It also holds FIRST occurrences. `LESSONS.md` takes a failure mode only once it has RECURRED — once is an incident,
twice is a lesson — and STATE.md is wiped at every epic boundary, so an incident parked there makes the second
occurrence unrecognisable and the lesson is never earned. Land it here, and a later repeat has something to match.

Routing for everything else: `CLAUDE.md` §WHERE_WORK_LANDS.

## KNOWN DEFECTS

**`SecMasterDiscoveryTimeoutsElevated` structurally cannot fire.** The rule is `timeout/total > 0.5`, but a
per-candidate deadline propagates through the `finally` (emitting `not_found`) and THEN emits `timeout` — one
candidate, two increments, ratio pinned at exactly 0.5. Only the pre-discovery semaphore arm can exceed it.
Fix the double-count, not the threshold; an alert tuned around a miscount hides the miscount.
Metric gotcha: OTEL appends `_total`, so alert on `secmaster_fred_search_skipped_total`, not the bare name.

**The merge gate reads a shell redirect as a PR number.** `gh pr merge <N> --squash 2>&1` denies with "names more
than one PR number: 2 <N>". Same redirect-parsing class the push guard already fixed; a third guard still carries it.
Workaround until fixed: drop the redirect.

**`SentinelExtractionDead` inhibits every sentinel warning if it fires** (`equal: ['service']`), including the
collapse alert. 0 inhibited to date, but the coupling is undocumented anywhere else.

**SentinelCollector.UnitTests parallel-isolation flake, ~50% on full runs.** Two tests fail only under xUnit
parallelism, because a process-global `ActivityListener` catches a concurrent test's state:
`MacroObservationRouterTests.should_count_log_and_tag_error_status_when_qualitative_substrate_write_throws` and
`ExtractionProcessorArticleEmbeddingTests.should_tag_failed_when_store_throws`. Both PASS in isolation.
Fix: serial `[Collection]` or a per-test-tag listener filter.

**`templates/implementation-fix.md` "WHERE TO WORK" is stale, and it costs parallelism.** It still says every
`.devcontainer` project resolves to `devcontainer` and that "no lock exists today" — both false since per-worktree
ownership landed (`scripts/devcontainer-owner.sh` exists; 12 of 13 `compile.sh` carry `devcontainer_own`, the 13th
being a Python syntax check with no compose). It tells dispatched agents to take NO worktree and to SEQUENCE behind
other compiles, which also pushes concurrent agents into the shared checkout — the "different branches without
worktrees -> COLLIDE" case. One-paragraph fix.

**request-log regression-guard gap (#886).** The DiagnosticContext re-registration in AlertService, SecMaster and
CalendarService — which preserves `UseSerilogRequestLogging` after `Host.UseSerilog` was dropped — has no
`// INTENT` tag and no test, so a future edit deleting it silently breaks request logging at request time.

**`verify-citations.py` still splits verdicts outside the card corpus.** `deployment/tests/alerts/.mutation-scratch/`
(left behind by the repo's own documented `matrix.sh` workflow) and `.venv-bench/` each produce false UNRESOLVED.
All three offenders — sibling worktrees, `.venv-bench`, `.mutation-scratch` — are GITIGNORED, and the documented
reason for rejecting `git ls-files` (the tests build throwaway trees with no git repo) does not rule out READING
`.gitignore`, which stays hermetic. Also: a missing input file crashes with a traceback and exits **1**, the tool's
own FINDING code — the same 1-vs-2 conflation fixed in four sibling scripts, each of which got a selftest case.

**Metric prefix inconsistency from a single service:** `sentinel_candidate_surface_*` vs
`sentinelcollector_semantic_signal_*`.

**Four uncorrected copies of the Finnhub transient-only claim.** `IdentifierConfirmationService.cs:361-362` is the
highest value because it states the dead-code conclusion outright (403s reaching a catch they cannot reach).

**#960's docs state `sum() - budget_exhausted` as "rows scored".** The true figure is
`accepted + rejected + below_floor`. One-line fix, and it must land before anyone builds that Grafana panel.

## MEASUREMENT DEBT [instruments that cannot report their own dullness]

**Empty-but-valid results grade CORRECT.** A schema-valid `qualitative_result` with `sentiment_polarity:
"unspecified"`, zero sectors, zero regimes and confidence 0.1 skips EVERY rubric rule on full-length content and is
graded CORRECT as a GRADED row — a full draw then reports `0.0% MAJOR, SHIP_FULL`, the best possible reading.
Already 2.1% of graded rows (3 of 146). Root cause: the rubric is a violation detector, so emptiness is
indistinguishable from perfection. Rubric design, not plumbing.

**`"stub"` is an unpinned cross-file contract that fails OPEN.** Writer `compare_base_vs_resolved.py:771` ->
reader `ab_scorecard.py:129`. A writer-side rename returns `usable_nonstub` to 6 on a dead run with zero tests
going red — the same shape as the false-green it was introduced to close.

**`[24h]` is pinned only from below.** `[26h]`, `[72h]` and `[8760h]` all ship GREEN in #952's suite. A widened
window is the MISS direction for a dead-man's switch, and the file's own banner requires a boundary pair per
calibrated constant.

**CI is advisory, not blocking** — branch protection 403s on this GitHub plan, so a red run does not stop a merge.

## DEFERRED WORK

**17 person-named catalog rows remain in the `GeminiFallback` bucket** — 16 active all-series, plus one inactive
series `MCRFPC1` = "Justin Trudeau". #961's repair allowlist keyed on `entity_resolution:gemini` only, so these were
out of scope by construction. **Scope any remedy on PROVENANCE, never on series.** Remedy (deactivate / rename /
delete) not yet chosen.

**#935 ansible-gate — BLOCKED. Do not merge and do not patch-round it.** Three successive rounds each claimed
"loosened writes = 0" and each was falsified by hand-probing outside the template sweep. The unsound carve-out is
demonstrated end-to-end: `hash-object -w` -> `mktree` -> `commit-tree` -> `git checkout <sha> -- <file>` writes
content into a tracked file with every step ungated, and the suite ASSERTS both halves as allow (`git restore -s
<sha> -W` is equivalent). The premise "content review has already seen it" fails because step 1 is ungated.
Main's gate is UNCHANGED by this PR, so main retains the 120 real-write holes the PR was fixing — the tightenings
are real and worth salvaging in a rewrite. Verified good and worth keeping: escaping/dequoting 14/14, HARD_STOP
matrix 33/33 deny with the ansible remedy as the reason, degraded-PATH fail-closed 27/27 across 9 externals.
The 5,005-cell sweep and 40-mutant battery live in an agent worktree, NOT the repo, so neither headline number is
reproducible by a later reviewer.

**Dependency debt, none runtime-exposed.** `SQLitePCLRaw.lib.e_sqlite3` 2.1.11 (NU1903, GHSA-2m69-gcr7-jv3q, HIGH)
is TEST-ONLY — SQLite is the unit-test DB provider, prod is TimescaleDB — transitive via `SecMaster.UnitTests` and
`Reports.UnitTests`. `System.Security.Cryptography.Xml` fleet pin sits at 10.0.8 with Reports bumped to 10.0.10.
NasdaqCollector's gRPC-Swagger chain is old but clean for CVE-2026-49451, and Nasdaq is DISABLED in prod anyway.

**Accepted risks, do not re-flag.** Plaintext DB password `atlas_secure_password_2025` in 10+ tracked files, and
`OfrCollector/.env` tracked with `DB_PASSWORD` / `SMTP_PASSWORD` / `FRED_API_KEY`. The user accepted both
explicitly: private repo, LAN-only, public and public-derived data. Rotating touches the DB user and every consumer.

## PARKED EPICS

**#729 — regime news-as-staleness redesign.** Spec is PR #729, DO-NOT-MERGE. Intent: FRED/OFR benchmark is the slow
grounding anchor; Sentinel news is a fast-decaying coincident perturbation weighted by benchmark STALENESS, so the
system measures economic significance rather than coverage volume. Phases 1 / 2a / 2b are built and deployed
(2b in shadow), #730-#736.
- **Phase 2c cutover HELD** — embedding coverage still ramping (`missing_embedding` ~0.57) AND shadow
  `magnitude_ratio` ~2.5, meaning dedup AMPLIFIES or sign-flips the net rather than gently compressing it.
- **Phase 3** — staleness crossfade `news_weight = g(benchmark_age/cadence)` plus benchmark-anchored aggregation.
  This is the principled fix for the structural news-overweight (news MACRO cells run ~2-4x FRED magnitude and
  net-negative, which washes out sector differentiation into an over-neutral regime). Gated on 2c.
- **Backtest harness** is the validation layer that unblocks all of #729 — shadow has no ground truth. Outcome data
  located: `finnhub_quotes` XLE/XLF/XLV/XLK ~6.5mo, SPY/QQQ, Yahoo EOD backfill for all 11, AV WTI/BRENT/NATGAS to
  1986. First gate is establishing a realized-sector-return series.
- **Classifier sign-fix is parked** — do not ship it off the energy anecdote; an n=6 peek showed news directionally
  right, but it needs a real backtest N. `commercial-paper-stress` de-flagged in the interim (#733).

## PRACTICE NOTES [cheap checks that were skipped three times or more]

**A defect found in a branch is not a defect on main.** Three fail-opens in one session were reported as live on
main and were not. The settling check is one command — `git branch --contains` on the commit that introduced the
line — and it was never run until a reviewer ran it.

**A tool's green run is not proof.** Ask what the tool cannot see before believing it; every instrument in the
MEASUREMENT DEBT section above reported success while blind. See CLAUDE.md TOOL_UPKEEP.
