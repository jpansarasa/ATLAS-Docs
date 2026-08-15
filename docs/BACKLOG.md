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

**`SentinelLowResolutionRate` cannot fire, and fixing only the window would make it scream instead.** Measured
2026-08-14 over 6h at 5m steps: 9 of 18 samples are NaN (`sum(rate(...[5m]))` denominator empty during the idle
gaps between bursts) and the other 9 are exactly `0` — so it oscillates pending -> inactive and never holds the
`for: 15m` dwell. 24 pending cycles and 0 fires in 24h, straight through a real resolution rate of ~3%.
THREE defects, and the window is only the first. (2) The denominator includes the sector-grounding statuses
`no_subject_match` (4,263/24h) and `matched_no_sector` (1,622/24h), emitted by `DeterministicResolver.LiftSector`
with `resolution_state="no_sector"` — those can never carry `status="resolved"`, so they structurally depress the
ratio. (3) The numerator misses the successes: `ResolutionWorker` resolves with method `async_finnhub` and
increments `SecMasterResolutionCounter` only on its REJECTION paths, so `sentinel_secmaster_resolution_total` is a
failure-biased counter — `status="resolved"` totalled 2 in 24h while the DB recorded ~180 real resolutions/day.
Fix needs a per-observation outcome counter at the persist boundary, landed WITH the rule; a window-only fix swaps
a silent alert for a permanently-firing one on a ratio that does not mean what it says.
Both DB figures above (~3% real rate, ~180 real resolutions/day) are POST-erasure `instrument_id` readings and are
therefore FLOORS — see the `ReExtractBackgroundService` entry below. The three alert defects are unaffected; the
magnitudes understate by an unknown margin.

**Sentinel's reported resolution rate read 67% while the honest column has not read above 4.21% since April — and both
are POST-erasure readings** (see the `ReExtractBackgroundService` entry below). Scope of that caveat: the HONEST-rate
figures are FLOORS, each understating the true rate by an unknown margin. The 67.11% reported rate is NOT a floor —
it is inflated by the mislabel, which is what this entry is for and which the erasure does not affect. Honest rate
(`instrument_id IS NOT NULL`) by month: Apr 4.21%, May 3.09%, Jun 0.79%, Jul 0.34%, Aug 2.58%. The reported rate
(`resolution_state='Resolved'`) read 67.11% in April because rows were stamped Resolved with a NULL instrument —
28,298 of them that month, 21,671 in May. PR #854 (2026-07-05) fixed the mislabel, and from July the two figures
agree, which is why the June "collapse" in any dashboard built on `resolution_state` is an artifact of the FIX, not
a regression. Re-check: compare the two percentages in the same query before concluding anything moved.

**Rule 1's outcome is ERASED downstream, not never produced — and the column that says otherwise cannot tell the
difference.** `sum by (reason)(sentinel_resolver_rule1_decision_total)` read 2026-08-15T01:10Z: `picked` 108,
`no_index` 26 — cumulative since the container was created 2026-08-15T00:17:47Z and last incremented 00:32Z, so ~14
minutes of counting on a counter that the restart zeroed. **Not comparable to the 24h DB figures below**; no
108-against-6,099 ratio can be formed from them. `no_confidence`, `below_threshold` and `index_out_of_range` are
ABSENT series, not measured zeros — the query returns two rows. The instrument itself is exporting (two of its reason
values are present), so absent means those branches never executed since that restart, not that the meter is missing.
Measured in the DB 2026-08-15 (UTC) over the preceding 24h, 6,099 `sentinel.extracted_observations` rows carry
`OriginalResolutionMethod='llm_candidate_pick'` — the resolver fires at scale — and **1,421 of them (23%) lost an
instrument they already held** (`OriginalInstrumentId` NOT NULL, `instrument_id` NULL), with 0 quarantined.
**The erasure hits EVERY `DeterministicResolver` leg, not only Rule 1**: the same window loses 549 `hybrid_subject`,
128 `gemini_fallback` and 1 `llm_candidate_exact` row, which with Rule 1's 1,421 sum to the ALL-METHODS total of
2,099 in the `ReExtractBackgroundService` entry below (identical under `extracted_at` and `re_extracted_at` windows).
`hybrid_subject` and `gemini_fallback` fire in production too. Do not read 2,099 as a Rule 1 figure: against Rule 1's
6,099 rows it implies a 34% loss rate where the measured one is 23%. Worked
examples, ids 689274 / 689275 / 689284: `OriginalResolutionMethod=llm_candidate_pick`, `resolution_method` NULL,
`QuarantinedAt` NULL, `review_notes="[re-extract] processed 2026-08-15T00:29:03Z"`. Mechanism: the
`ReExtractBackgroundService` entry immediately below.
THE TRAP, and it cost hours of wrong root-cause search: **any query over `resolution_method` reads POST-erasure state
and cannot distinguish "never set" from "overwritten".** This entry previously read "no `DeterministicResolver` outcome
has ever been persisted" off 0 occurrences of `llm_candidate_pick` / `llm_candidate_exact` / `hybrid_subject` /
`gemini_fallback` in 658,167 rows — literally true of the column, false about the resolver, and it sent the
investigation into the extraction path instead of the writer. Read `OriginalResolutionMethod` alongside
`resolution_method`, always; the legacy-leg counts recorded in the same round (`cove_VectorSearch` 3,977,
`ticker_in_quote` 6,702, `cove_FuzzySql` 235) are readings of that same post-erasure column and carry the same caveat.
A SECOND column carries the same circularity, and it is a distinct trap: the value Rule 1 gates on is never PERSISTED
(`extracted_observations.resolution_confidence` holds the resolver OUTCOME's value —
`DeterministicResolver.cs:293-297`), so that column cannot answer what Rule 1 received, and querying it is circular.
It is observable, just not in the DB: PR #963 added `sentinel_resolver_rule1_input_confidence` ("ResolutionConfidence
as received by Rule 1", `SentinelMeter.cs:1614-1616`, recorded at `DeterministicResolver.cs:315`), which is the only
thing that sees the input value — and is where the 0.850 reading below comes from.
Not to be re-derived: the `ExtractionSchemaV2 required[]` hypothesis was DISPROVEN by probing vLLM with the shipped
schema, which emitted `resolution_confidence` non-null 5/5.
One cross-check is structurally inert and will agree forever: all 108 observations of
`sentinel_resolver_rule1_input_confidence_bucket` sit at exactly 0.850 (the entire count lands in `le="0.9"`, none at
or below 0.8), which IS `DslPreselectionConfidence`, a hardcoded constant — so the `< 0.7` gate can never trip (an
absent `below_threshold` is a property of the constant, not evidence about the data) and `bucket{le="0.7"}` reads
0=0 indefinitely.

**`ReExtractBackgroundService` strips instruments off brand-new live rows, and no counter exists for the loss.** It is
running in prod as a backfill (`/opt/ai-inference/compose.yaml:1212-1216`: `ReExtract__Enabled=true`,
`Mode=resolve-only`, `Cohort=all`, `RowsPerMinuteResolveOnly=600`) but it is not confined to legacy rows:
`ApplyCohortPredicate("all")` filters only on `ReExtractedAt IS NULL`, so every freshly-inserted row qualifies and is
claimed **~10-180s after insert (2026-08-15 UTC, n=189 over 2h: min 10.2s, p50 ~50s, p95 133s)**, well inside the
live path.
`ApplyReExtraction` then assigns `ResolutionMethod` unconditionally from its own one-shot resolve — NULL on a miss,
overwriting a good value rather than declining to write — and recomputes `ResolutionState` from that. Measured in the
DB 2026-08-15 (UTC) over the preceding 24h: **2,099 rows lost an instrument they already held against 127 that gained
one — 16:1 destructive**, 0 quarantined. That 2,099 is ALL resolution methods; the per-method split is in the Rule 1
entry above. Do not re-check the ratio against `sentinel_reextract_rows_processed_total{outcome="recovered"}`:
`ReExtractBackgroundService.cs:685-696`
(mirrored verbatim at `:496-507`) emits `Recovered` only when the SYMBOL CHANGES, so a row that gains an instrument
under an unchanged symbol is classed `Unchanged` — the counter undercounts recoveries by construction, and a
cumulative read of it is worthless for hours after any container restart.
It ran unseen because `ReExtractOutcome` has no counter for "had an instrument, lost it": the outcome enum cannot
express the regression, so no dashboard or alert could have shown it, and the loss surfaced only by diffing
`OriginalInstrumentId` against `instrument_id` in SQL. Land that counter WITH any fix — a fix verified on the same
blind instrument proves nothing. `NoResolutionSweepWorker` is EXONERATED and should not be re-suspected: it only calls
`SetReviewStatus`, and its predicate requires `extracted_at < now-7d`, which no row of this age can satisfy.

**gemini-resolver runs at 100% of its daily cap while its gate rejects ~1 call in 3,000.** Measured 2026-08-14:
`gemini_resolver_live_calls_24h` 1500 against `gemini_resolver_daily_cap` 1500, `gemini_resolver_gated_24h` = 1 of
3,076 total calls, and 877 of SecMaster's 3,425 dispatches/24h refused as `cap_exhausted`. Refusal is
first-come-first-served, so genuine resolutions are dropped at random once the window is spent. `_company_gate`
(gemini_resolver/server.py) is purely syntactic — it rejects money, markup, code-slugs and 13 abbreviations, and
cannot reject a well-formed noun phrase that is not a tradeable issuer, which is what the junk is
("Birmingham Legion", "Hellenic Shipping News World", "Focus On Inflation"). Not a matrix-corruption event as of
this measurement: recent Gemini self-seeds are legitimate issuers. Re-check with `curl :9300/health`.

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

**A third histogram still carries the SDK default buckets a [0,1] value cannot use.**
`sentinel_chunk_extraction_dedup_ratio` (`SentinelCollector/src/Telemetry/SentinelMeter.cs:253`, unit `{ratio}`) has
no `AddView`, so it keeps the SDK boundaries `[0, 5, 10, 25, ...]` and every observation of a `1 - post/pre` fraction
would land in `le=5.0` — the identical collapse #963 fixed on `sentinel_dsl_adapter_resolution_confidence` and
`sentinel_resolver_rule1_input_confidence`. Nothing is misled TODAY: measured 2026-08-15 UTC, the metric has NO series
in prod (`count({__name__=~"sentinel_chunk_extraction_dedup_ratio.*"})` empty, against the sibling confidence
histogram returning all 16 default-bucket series in the same query shape), because the v2 chunked path is not
emitting. The entry exists so that the first time it does, the collapse is already known rather than rediscovered.
Fix: add the name to the same `AddView` list in `SentinelCollector/src/Program.cs` that already applies
`confidenceBuckets` — the [0,1] boundaries suit a ratio unchanged.

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

**`Extraction__GuardsEnabled=false` — AWAITING AN OWNER DECISION.** This is a deliberate experiment, not an accident:
`/opt/ai-inference/compose.yaml:1155` carries it dated 2026-05-03, on the rationale that the Phase 4.3 client-side
guards (token-overlap + asset-class) were defensive bandaids that had become precision sinks, with the PR-3 prose
template plus cosine expected to disambiguate without literal-token rules. This entry records what that buys and costs
today, and prejudges nothing. Exposure, measured live 2026-08-15T00:46Z: `/api/semantic/resolve-local?q=OpenAI`
returned `symbol=BBAI` (BigBear.ai) WITH an instrument id, at confidence 0.85 — a private company resolving to an
unrelated public issuer with an id attached, not a low-confidence near-miss.
`SubjectNameNormalizer.SharedTokenCount("OpenAI", "BigBear.ai Holdings")` scores 0 and WOULD reject it; the guard
simply does not run. So the call is a real tradeoff and belongs to the owner: re-enabling recovers this class of false
accept and re-imposes the precision cost the experiment was set up to measure. Re-check with the same probe — it needs
no deploy.

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
