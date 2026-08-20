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

**A worktree-isolated agent CANNOT do gate-layer work at all: the two guards deadlock.** `ansible-gate-guard.sh`
refuses every write under `.claude/hooks/**` and names one sanctioned escape — a bypass file at
`$CLAUDE_PROJECT_DIR/.claude/.ansible-gate-confirmed`, ideally scoped by path fragment. But `project_dir` is
`"${CLAUDE_PROJECT_DIR:-$PWD}"` (`:98-99`) with NO worktree handling, so for an agent in
`.claude/worktrees/<id>/` it resolves to the SHARED checkout — which worktree isolation then refuses to let that
agent write ("Edit the worktree copy of this file instead"), and the worktree copy is never read. Measured
2026-08-15: creating `<worktree>/.claude/.ansible-gate-confirmed` activates the bypass when the guard is invoked by
hand from the worktree, and does nothing for the real hook, which reported the shared path in its deny message on
every attempt. Net effect: PR #970's own review findings — three guard fixes in `git-push-guard.sh` plus rows in two
suites — could be measured but not landed, and the round returned docs only. Two further consequences worth keeping:
the gate treats a path FRAGMENT match anywhere in the command, so it also blocks a scratch copy under `/tmp` whose
path merely contains `.claude/hooks/…`, closing the develop-and-verify-elsewhere route as well; and it gate-tests
every non-flag token of a `cp`, so READING a gate file into scratchpad is refused alongside writing one (deliberate
— `cp <gate> <gate>` is the motivating attack at `:412` — but it removes the last unprivileged way to work on a
copy). Options, cheapest first: honour a worktree-local bypass by resolving `project_dir` through
`git rev-parse --show-toplevel` before falling back to `CLAUDE_PROJECT_DIR`; or have the supervisor dispatch
gate-layer work non-isolated. Re-check: from a worktree, attempt any edit to `.claude/hooks/git-push-guard.sh` after
creating a scoped `<worktree>/.claude/.ansible-gate-confirmed` — it must be allowed.
**THE HUMAN-PLACED BYPASS UNBLOCKS THE HALF THAT IS UNTESTABLE ALONE.** Measured 2026-08-15 on the round that
followed: a bypass written into the SHARED checkout does reach a worktree agent's edits, so the deadlock above is
escapable by hand — but it is scoped by PATH FRAGMENT, and the natural fragment for guard work is the guard's own
filename. `git-push-guard.sh` therefore admits every edit to the rule and refuses every edit to
`.claude/hooks/test/run-*.sh`, because a suite's path does not contain the guard's name. That round landed four
merge-gate rules and not one of their rows; the round after it added the rows only because a human widened the
bypass to include the fragment `-smoke`, which substring-matches every `run-*-smoke.sh`. The shape generalises past
this instance: the gate counts a guard's TESTS as gate layer, so any fragment narrow enough to scope a bypass to one
guard is also narrow enough to exclude that guard's tests, and the bypass silently permits exactly the half of an
atomic change that must not ship alone. Whatever fixes the worktree resolution should decide this too — the
cheapest form is to let a fragment match the suite that guards it, e.g. by scoping on the guard's basename minus
its extension, so that one fragment admits the rule and its rows together rather than needing a second by hand.
Re-check: with a bypass containing `git-push-guard.sh` ONLY, edit
`.claude/hooks/test/run-pr-verdict-smoke.sh` — it must be refused today.

**`SecMasterDiscoveryTimeoutsElevated` structurally cannot fire.** The rule is `timeout/total > 0.5`, but a
per-candidate deadline propagates through the `finally` (emitting `not_found`) and THEN emits `timeout` — one
candidate, two increments, ratio pinned at exactly 0.5. Only the pre-discovery semaphore arm can exceed it.
Fix the double-count, not the threshold; an alert tuned around a miscount hides the miscount.
Metric gotcha: OTEL appends `_total`, so alert on `secmaster_fred_search_skipped_total`, not the bare name.

**`SentinelLowResolutionRate` spends its life oscillating pending -> inactive, and fixing only the window would
make it scream instead.** Measured 2026-08-14 over 6h at 5m steps: 9 of 18 samples are NaN
(`sum(rate(...[5m]))` denominator empty during the idle gaps between bursts) and the other 9 are exactly `0` — so
it rarely holds the `for: 15m` dwell. 24 pending cycles and 0 fires in 24h, straight through a real resolution
rate of ~3%.
THAT ZERO-FIRE COUNT IS TRUE FOR ITS WINDOW AND ONLY FOR ITS WINDOW — re-confirmed 2026-08-20,
`count_over_time(ALERTS{alertname="SentinelLowResolutionRate",alertstate="firing"}[2d])` at 2026-08-15T00:00:00Z
is empty, so nothing fired on 08-13 or 08-14. RUN THE CONTROL ALONGSIDE IT — empty is ambiguous on its face
between "did not fire" and "the series was never recorded". Re-run the identical query at `alertstate="pending"`:
it returned **1,423** at that same instant, which proves the series WAS being recorded and is what makes the
emptiness mean something. The rule is NOT structurally unable to fire: the entry below
measures five firing episodes in the 7 days to 2026-08-20. Do NOT widen the window to make it fire — it already
does, on burst shape rather than on resolution health.
THREE defects, and the window is only the first. (2) The denominator includes the sector-grounding statuses
`no_subject_match` (4,263/24h) and `matched_no_sector` (1,622/24h), emitted by `DeterministicResolver.LiftSector`
with `resolution_state="no_sector"` — those can never carry `status="resolved"`, so they structurally depress the
ratio. (3) The numerator misses the successes: `ResolutionWorker` resolves with method `async_finnhub` and
increments `SecMasterResolutionCounter` only on its REJECTION paths, so `sentinel_secmaster_resolution_total` is a
failure-biased counter — `status="resolved"` totalled 2 in 24h while the DB recorded ~180 real resolutions/day.
THE MECHANISM IN (3) IS RIGHT BUT THE RESOLVER NAMED IS NOT: the entry below measures `async_finnhub` at **0** rows
over 7 days and identifies `DeterministicResolver` as the live leg, with the shortfall an order larger.
Fix needs a per-observation outcome counter at the persist boundary, landed WITH the rule; a window-only fix swaps
a near-silent alert for a permanently-firing one on a ratio that does not mean what it says.
Both DB figures above (~3% real rate, ~180 real resolutions/day) are POST-erasure `instrument_id` readings and are
therefore FLOORS — see the `ReExtractBackgroundService` entry below. The three alert defects are unaffected; the
magnitudes understate by an unknown margin.

**`SentinelLowResolutionRate` does not measure a resolution rate: 49 of 16,030 real resolutions — 0.31% — ever
touch the counter it divides.** The rule is Prometheus-native — loaded by the `rule_files` glob at
`deployment/artifacts/monitoring/prometheus.yml:12`, not Grafana unified alerting.
`deployment/artifacts/monitoring/alerts/sentinel.yml:277-288`:
`sum(rate(sentinel_secmaster_resolution_total{status="resolved"}[5m]))` divided by
`sum(rate(sentinel_secmaster_resolution_total[5m]))`, `< 0.5`, `for: 15m`. The other 99.7% of real resolutions are
invisible to BOTH numerator and denominator, so the ratio is not a degraded measurement of resolution — it measures
something else.
CORRECTS THE ENTRY ABOVE, which named the wrong resolver and understated the magnitude by an order.
`ResolutionWorker`'s `async_finnhub` is not the active leg: **0** rows in 7 days, in `resolution_method` AND in
`"OriginalResolutionMethod"`. That entry's three MECHANISMS all stand — the 5m rate genuinely does go NaN and
break the dwell — and its 2026-08-14 zero-fire count is correct for the window it measured. What does NOT stand is
the generalisation drawn from it: "cannot fire" / "never holds the dwell" is falsified by the five firing episodes
below. The numerator's cause and size change as well.
- The live leg is `DeterministicResolver`, called at `SentinelCollector/src/Services/V2ExtractionPipeline.cs:77`.
  Its only consumer is `BuildObservationFromV2Result` (`SentinelCollector/src/Services/V2ExtractionPipeline.cs:172-235`),
  an `internal static` PURE BUILDER — it does NOT touch the database. It sets `instrument_id` and
  `resolution_state` on an in-memory `ExtractedObservation` (`UpdateResolution` / `SetResolutionState`), returns it
  into `observations` at `SentinelCollector/src/Services/V2ExtractionPipeline.cs:97`, and the row reaches
  `ExtractionProcessor` on `V2PipelineResult` to be persisted downstream. It calls
  `SentinelMeter.SecMasterResolutionCounter.Add` for **no** outcome — neither success nor failure; the whole file
  has **zero** call sites (`grep -c SecMasterResolutionCounter` = 0). That is the whole defect: the resolution
  OUTCOME is decided here and metered nowhere, so nothing between the decision and the persist is counted.
- Inside `DeterministicResolver` the counter has FIVE emission sites and exactly one carries `status="resolved"`:
  `SentinelCollector/src/Services/DeterministicResolver.cs:449` (`TryExactCandidateMatchAsync` success,
  `llm_candidate_exact`). The other four are refusals or non-resolutions — `:253` (`LiftSector`; ONE site whose
  status is a ternary over `no_subject_match` / `matched_no_sector`, always `resolution_state="no_sector"`), `:437`
  (`TryExactCandidateMatchAsync` co-mention rejection, `exact_rejected_name`), `:521` and `:538`
  (`TryHybridResolveAsync` guard rejections).
- `ExtractionProcessor.cs` never calls `DeterministicResolver` — **zero** grep hits — and its own two
  `status="resolved"` emissions (`SentinelCollector/src/Workers/ExtractionProcessor.cs:1195` `ticker_in_quote`,
  `:1313` `cove_*`) have not fired in prod for 30 days. Those two cite the `status` label line, one BELOW their
  `.Add(`; the `DeterministicResolver` citations above cite the `.Add(1,` line itself. Both land inside the correct
  emission block — do not "fix" either to match the other. A sixth site outside the resolver,
  `SentinelCollector/src/Workers/ReExtractResolutionAdapter.cs:190`, emits only `comention_rejected`.
- Over 30 days the metric carries EIGHT `(method, status)` pairs in total, and `llm_candidate_exact`/`resolved` is
  the only resolved one; `ticker_in_quote` appears solely as `comention_rejected`. Re-check:
  `count by (method, status) (increase(sentinel_secmaster_resolution_total[30d]))`.

METRIC VS GROUND TRUTH, 7 days to 2026-08-20T17:27Z. The expression evaluates to NaN, exactly 0, or small
positives; `max_over_time(<expr>[7d:5m])` = **0.111**. `sum(increase(...{status="resolved"}[7d]))` = **50.0**
(raw cumulative counter **49**, all `llm_candidate_exact`) against `sum(increase(...[7d]))` = **39,523** — a
counter-side ratio of **0.13%**. **30,802 of that 39,523 (78%) is `sector_grounding`**, which by construction can
never carry `status="resolved"`. The DB over the same window says **36.1%** (16,030 resolved / 44,410 total),
computed correcting for the re-extraction erasure trap; the naive current-`instrument_id` read gives **30.1%**
(13,365 / 44,410), itself an undercount. SQL, SELECT only — note the ABSOLUTE bounds: under a `now() - interval
'7 days'` moving window the two clipped edge buckets drift within minutes (measured: 20.3 -> 20.8 and 40.8 -> 41.4
over 28 minutes), so a re-check would not land on the window these figures came from.
```sql
SELECT count(*) AS total,
       count(*) FILTER (WHERE CASE WHEN re_extracted_at IS NULL
                        THEN instrument_id ELSE "OriginalInstrumentId" END IS NOT NULL) AS resolved_corrected,
       count(*) FILTER (WHERE instrument_id IS NOT NULL)                                AS resolved_naive
FROM sentinel.extracted_observations
WHERE extracted_at >= timestamptz '2026-08-20T17:27:00Z' - interval '7 days'
  AND extracted_at <  timestamptz '2026-08-20T17:27:00Z';
```
Daily (same two bounds, `GROUP BY date_trunc('day', extracted_at)`; 8 buckets because a 7-day window clips both
ends): 20.3, 27.9, 29.7, 32.7, 40.4, 40.2, 38.5, 40.8 — chronically below the 50% threshold, and trending UP while
the metric sat near zero. Those are UTC days only because the psql session inside the `timescaledb` container runs
`TimeZone = UTC` (`SHOW timezone`); the 2-arg `date_trunc('day', <timestamptz>)` takes its day boundaries FROM
the session TimeZone rather than from UTC — measured: the same instant truncates to 2026-08-20 under `UTC` and to
2026-08-19 under `America/New_York`, which is mercury's host TZ — so verify the session rather than assume it.
Both queries re-run 2026-08-20T18:0xZ and
reproduced 44,410 / 16,030 / 13,365 and all eight buckets unchanged. Carry the caveat from the `ReExtractBackgroundService` entry below: `"OriginalInstrumentId" IS NOT NULL` means
held-at-EARLIEST-snapshot, not at extraction, so 36.1% is not a clean upper bound either — but every reading here is
two orders above what the counter reports.

RESOLUTION_METHOD BREAKDOWN of the 16,030, erasure-corrected
(`CASE WHEN re_extracted_at IS NULL THEN resolution_method ELSE "OriginalResolutionMethod" END`), summing to 16,030
exactly: `llm_candidate_hybrid` **9,337**, `hybrid_subject` **3,552**, `llm_candidate_pick` **2,309**,
`gemini_fallback` **781**, `llm_candidate_exact` **51**. The naive `resolution_method` column instead sums to 13,365
over eight methods, and is the ONLY place `cove_VectorSearch` (544), `ticker_in_quote` (46) and `cove_FuzzySql` (1)
appear — those are what the re-extract adapter wrote, not what resolved the row. **Never mix the two columns in one
total**: a breakdown taken from the naive column under the corrected headline is short by 2,665 and still reads as a
plausible list. `async_finnhub` is **0** in both.

WHY IT FIRED ON 2026-08-20. Pending 16:32:30 to 16:47:30 — **900s, exactly the `for: 15m`** — bridged by a rare
uninterrupted extraction burst; firing 16:47:30 to ~16:49:00, then inactive the moment the burst ended and
`rate[5m]` went NaN. Two notifications, one fire: the 16:52:55 notification is Alertmanager's group re-flush at
`group_interval` 5m while `resolve_timeout` 5m still held it active. Across the 7 days the rule spent **7,935**
scrape samples pending (~33h) against **118** firing (~30 min) over **five** separate episodes — so it is not that
the dwell timer never completes, it is that completion is decided by burst shape rather than by resolution health.
Re-check: `count_over_time(ALERTS{alertname="SentinelLowResolutionRate",alertstate="firing"}[7d])`.

CONSEQUENCE, and this is the point. The exact row shape PR #980 documented — a single publisher-organisation
candidate, no resolution — is wired to NEITHER side of the ratio, so **a source dying 100% does not move this metric
at all**. The expression is a global `sum()` with no per-feed label, so even a correct metric could not name which
feed died. It could not have caught the 2026-04-23 outage. That last counterfactual is INFERRED, not measured:
Prometheus retention does not reach 2026-04-23 (an instant query at 2026-04-23T12:00:00Z returns empty; earliest
confirmed non-empty is 2026-07-01).

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
128 `gemini_fallback` and 1 `llm_candidate_exact` row, which with Rule 1's 1,421 sum to an ALL-METHODS total of
2,099 for that window (identical under `extracted_at` and `re_extracted_at` windows). The window is what makes that
number move: the same measurement re-run 2026-08-15T15:00Z read 1,046 all-methods (726 / 256 / 63 / 1) — the RATIO
and the per-method COMPOSITION reproduce, the absolutes do not, so cite the composition.
`hybrid_subject` and `gemini_fallback` fire in production too. Do not read 2,099 as a Rule 1 figure: against Rule 1's
6,099 rows it implies a 34% loss rate where the measured one is 23%. Worked
examples, ids 689274 / 689275 / 689284: `OriginalResolutionMethod=llm_candidate_pick`, `resolution_method` NULL,
`QuarantinedAt` NULL, `review_notes="[re-extract] processed 2026-08-15T00:29:03Z"`. Mechanism: the
`ReExtractBackgroundService` entry below.
THE TRAP, and it cost hours of wrong root-cause search: **any query over `resolution_method` reads POST-erasure state
and cannot distinguish "never set" from "overwritten".** This entry previously read "no `DeterministicResolver` outcome
has ever been persisted" off 0 occurrences of `llm_candidate_pick` / `llm_candidate_exact` / `hybrid_subject` /
`gemini_fallback` in 658,167 rows — literally true of the column, false about the resolver, and it sent the
investigation into the extraction path instead of the writer. Read `OriginalResolutionMethod` alongside
`resolution_method`, always; the legacy-leg counts recorded in the same round (`cove_VectorSearch` 3,977,
`ticker_in_quote` 6,702, `cove_FuzzySql` 235) are readings of that same post-erasure column and carry the same caveat.
A SECOND column carries the same circularity, and it is a distinct trap: the value Rule 1 gates on is never PERSISTED
(`extracted_observations.resolution_confidence` holds the resolver OUTCOME's value —
`DeterministicResolver.cs:324-328`), so that column cannot answer what Rule 1 received, and querying it is circular.
It is observable, just not in the DB: PR #963 added `sentinel_resolver_rule1_input_confidence` ("ResolutionConfidence
as received by Rule 1", `SentinelMeter.cs:1625-1627`, recorded at `DeterministicResolver.cs:346`), which is the only
thing that sees the input value — and is where the 0.850 reading below comes from.
Not to be re-derived: the `ExtractionSchemaV2 required[]` hypothesis was DISPROVEN by probing vLLM with the shipped
schema, which emitted `resolution_confidence` non-null 5/5.
One cross-check is structurally inert and will agree forever: all 108 observations of
`sentinel_resolver_rule1_input_confidence_bucket` sit at exactly 0.850 (the entire count lands in `le="0.9"`, none at
or below 0.8), which IS `DslPreselectionConfidence`, a hardcoded constant — so the `< 0.7` gate can never trip (an
absent `below_threshold` is a property of the constant, not evidence about the data) and `bucket{le="0.7"}` reads
0=0 indefinitely.

**Rule 1's pick was never the wrong row — the resolver SUBSTITUTED a different instrument after it, and #969 fixed
that.** Framing first, because the wrong one costs hours: the entry above establishes that Rule 1 fires and that its
outcome is erased downstream, and the natural next reading — "then the LLM must be picking the wrong candidate out of
the article-wide list" — is REFUTED. The pick is correct. `DeterministicResolver` then fuzzy-resolved the picked
candidate's `Symbol`, which on the production V2 producer is a model-authored slug of the DSL `local_id`, not an
identifier, and SecMaster's fuzzy/RAG stage returned whatever that string happened to look like. Do not re-search the
candidate list; the defect lives between the pick and the write.

WINDOW AND COLUMN RULE, stated before any figure because every number below is on THIS window and nothing else. The
figures here stood as "the 31 days to 2026-08-15", which is ROLLING: it re-derives differently every day, and the
`Sensex 704` / `yen 230` that used to sit in the table reproduce only under `now() - interval '31 days'`. Numbers on
different windows inside one entry is what took #968 four rounds. FIXED:

    extracted_at >= '2026-07-15 00:00:00+00' AND extracted_at < '2026-08-15 00:00:00+00'   -- 31 days
    Rule 1 population: coalesce("OriginalResolutionMethod", resolution_method)
                         IN ('llm_candidate_pick','llm_candidate_hybrid')
                       AND selected_candidate_index IS NOT NULL AND candidate_symbols_json IS NOT NULL
    picked Name  = candidate_symbols_json -> selected_candidate_index ->> 'Name'
    picked slug  = candidate_symbols_json -> selected_candidate_index ->> 'Symbol'
    attaches     = coalesce("OriginalInstrumentId", instrument_id) IS NOT NULL
    persisted    = coalesce("OriginalSymbol", "Symbol")

`Original*`-preferred throughout, because ReExtract overwrites the live columns (see above) while leaving `Original*`
holding the pre-re-extract answer. On that window: 134,612 Rule 1 rows, 31,392 attaching, 103,220 attaching nothing.

    picked Name | picked slug | persisted | n
    S&P 500     | S_P_500     | S         | 683   <- SentinelOne
    S&P 500     | S_P_500     | SP500     | 528   <- same slug, different answer
    Sensex      | Sensex      | SNSE      | 675   <- Sensei Biotherapeutics
    yen         | yen         | U         | 234   <- Unity Software

**29,140 of 31,392 instrument-attaching Rule 1 rows (92.83%)** persisted a symbol matching NEITHER surface the pick
named; 103,220 further rows resolved to nothing and still stored the raw slug in a column called `Symbol`. The
figures this replaces (93.6%, 28,377/30,326, 102,411) were the rolling window; on the rolling window measured
2026-08-15 they now read 28,877/31,122 = 92.79% and 101,149, which is the point — they move daily.
NOT a same-denominator correction: 92.83% moved BOTH the numerator rule (matching neither surface, not merely not
the slug) and the denominator, so it cannot be differenced against the superseded 95.47% "by 576" and that
subtraction must not be re-cited. The figure also falls MONOTONICALLY BY CONSTRUCTION, which is a property of the
column rule rather than of the pipeline: the 817 rows the re-extract path attached enter the denominator through
`coalesce("OriginalInstrumentId", instrument_id)` while their `"OriginalSymbol"` — all 817 of them — is still the
picked slug from the original null-instrument resolution, so they can never enter the numerator. Every re-extract
pass therefore drags the percentage down without anything changing at Rule 1. (Only 66 of those 817 have
Name == slug; the mechanism is the SYMBOL column, not the Name/slug relation.)
TWO CAVEATS the headline does not carry: 12,227 of the attaching rows (38.95%) have Name == slug, so the fix is a
BYTE-IDENTICAL no-op for them — including `Sensex` and `yen` above, whose wrong answers the fix does NOT address —
and `subject_entity` equals the picked Name in 31,390 of 31,392 rows (not all of them, as previously written),
because the V2 adapter selects the candidate BY matching subject to ENT name, so post-fix Rule 1 sends the same
string Rule 2 would.
STILL OPEN, one pair, and the re-check is two HTTP calls rather than a probe nobody kept. `#969`'s pre-merge
blast-radius probe reported an aggregate improvements-vs-regressions count; it left behind no script, no captured
output and no query, and re-deriving its two named regressions on 2026-08-15 contradicted one of them, so the
aggregate is STRUCK rather than restated — do not reinstate it from the PR body or the commit message, and do not
treat its absence as evidence the fix is one-directional. What reproduces, against live SecMaster
(`nerdctl exec secmaster curl -s "http://localhost:8080/api/semantic/resolve-local?q=<surface>&enableRag=true&limit=5"`,
the same endpoint `ResolveLocalFromQuoteAsync` calls):
`Intel Corporation` -> `INL.DEX` (`FuzzySql`, 0.9, catalog name "Intel Corporation" — the German line) while the slug
`INTC` -> `INTC` (`ExactSql`, 1.0, "Intel Corp"). That IS a regression, and **its in-window exposure is 24 rows, not
155** — on the fixed window AND on the rolling one, which agree exactly here. The 155 is every row with picked Name
`Intel Corporation`; 131 of them attached NO instrument and reach 155 only because they are counted by the persisted
`Symbol` column, which the UNFIXED code fills with the raw slug on non-resolution. Both slug and persisted value are
the string `INTC`, so the defect under repair inflated the measurement of itself 6.5x. Under this entry's own
attaching rule the number is 24.
The companion claim — `Dow Jones` `DIA` -> `DOW`, "the chemical company", 91 rows — was struck here as
NOT REPRODUCING, and that strike was itself wrong. The `91` was right; the MECHANISM was not. Measured on both
windows: 136 in-window rows, of which `Dow Jones`|`Dow_Jones`|`DIA` **77** and `Dow Jones`|`Dow_Jones`|`DOW` **14**
= **91 attaching**, plus 45 that attached nothing. The slug's endpoint response is indeed `RagSynthesis` with a null
`instrumentId` — but the endpoint is not the outcome: it returns hypothesis `DIA`, and
`TryHybridResolveAsync`'s materialisation branch looks that up (`GetInstrumentBySymbolAsync`,
`DeterministicResolver.cs:505`) and attaches it. `DIA` is the catalog's quote stub, literally named "DIA (Quote)";
`DOW` is `DOW INC`, the chemicals company (`MATERIALS`, NAICS 325211), which is the RAG candidate list's top entry at
0.730. So one slug produces three different outcomes — `DIA`, `DOW`, nothing — and the pair still IMPROVES, because
the Name resolves deterministically to `DJIA` ("Dow Jones Industrial Average", `FuzzySql` 0.9), the index the surface
names. It improves by removing a nondeterministic wrong answer, NOT by replacing a null.
The direction the same probe was right about is re-checkable the same way: `S&P 500` -> `SP500` (`FuzzySql`, 0.9,
deterministic) against slug `S_P_500` -> `RagSynthesis`/`VectorSearch`, whose answer moves between runs — 683 rows
landed on `S` (SentinelOne) and 528 on `SP500` off the SAME slug, which is the nondeterminism the fix removes.
Follow-up, NOT decided here: an exact-symbol-first leg at Rule 1 would keep `INTC`, but nothing measures what it
would cost — it is a new ungated exact path and would need D-8's subject-overlap companion, which is its own PR.
NOT DONE and deliberately so: `SubjectNameNormalizer.SharedTokenCount` scores 0 for all four bad pairs above and is
already invoked at `DeterministicResolver.cs:430` and `:509`. "Never on this branch" was the wrong compression and
is corrected here to match D-22 in the card: `:430` is D-8's leg, which Rule 1 never reaches. `:509` IS reachable
from Rule 1 — it sits on the RagSynthesis hypothesis-materialisation branch inside `TryHybridResolveAsync`, which
the id-less Rule 1 leg calls — but a DTO already carrying an instrument id bypasses it, and the whole guard is
behind `Extraction__GuardsEnabled=false` (`/opt/ai-inference/compose.yaml:1155`), so it is inert on both counts.
The 77 `DIA` rows above went straight through it: `SharedTokenCount("Dow Jones", "DIA (Quote)")` is 0, so with the
flag on they would have been refused. Adding a THIRD call behind that same disabled flag would read as protection
that does not exist. Deciding the flag's fate is the prerequisite, and it is its own entry's worth of work.
TWO GAPS #969 LEFT OPEN, recorded rather than fixed because both need a decision this PR is not the place for.
(a) The `!candidate.InstrumentId.HasValue &&` exemption on the blank-Name refusal (`DeterministicResolver.cs:361`)
has NO test. It is dead code today — 0 non-null candidate ids across 503,446 rows carrying candidates — so nothing
exercises it, and it activates the day SecMaster's search endpoint starts returning ids: a SERVER-SIDE change with
no compile-time signal here, on a branch whose whole point is that the two producers disagree. Same blind spot as
the id-carrying Rule 1 branch, which at least has a test.
(b) The `ResolutionConfidence` contract for the new hybrid leg (`IDeterministicResolver.cs:63-79`) has no test
either. The FALSE half of that comment is fixed in #969 — it claimed res_conf is null on "the null-instrument
hybrid/gemini fall-throughs", which is not true of the leg #969 created (it coalesces to
`input.ResolutionConfidence`, so a null-instrument `llm_candidate_hybrid` row still carries the LLM's PICK
confidence) — but a comment is not a guard. The >= 0.8f event-publish predicate reads this field, so "instrument is
null, therefore confidence is null" is exactly the inference a consumer would make and exactly the one that is wrong.

**Two unit tests fail nondeterministically on static-meter pollution, and the suite reads GREEN on a re-run.**
Measured 2026-08-15 on `fix/rule1-resolve-on-candidate-name`, two consecutive full runs of
`SentinelCollector/.devcontainer/compile.sh` on an identical tree: run 1 `Failed: 2, Passed: 2222`, run 2
`Failed: 0, Passed: 2224`. The two were `ExtractionProcessorArticleEmbeddingTests.should_tag_failed_when_store_throws`
("Expected capture.Results to contain a single item, but found {"failed", "failed"}") and
`ExtractionProcessorStreamingTests.should_degrade_to_retry_not_host_die_when_producer_warmup_throws`. Both classes
pass 155/155 when run alone. Mechanism: `ExtractionProcessorArticleEmbeddingTests` is in
`[Collection("SentinelMeterStatic")]` (line 21) and `ExtractionProcessorStreamingTests` is in NO collection, so it
runs in PARALLEL with the meter-bound classes and its measurements land in their open `MeterListener`s — the exact
cross-pollution that collection exists to prevent. Pre-existing, NOT introduced by #969, and the dangerous half is
the direction of the failure: a re-run turns it green, so the standing incentive is to re-run rather than to fix,
and a REAL regression in any meter-asserting test would be indistinguishable from this. Fix is one attribute on
`ExtractionProcessorStreamingTests`; the audit worth doing with it is which other classes emit SentinelMeter
instruments without joining the collection.

**`ReExtractBackgroundService`'s overwrite is still destructive — the age floor bounds WHO it reaches, not WHAT it
does.** The live-traffic half is CLOSED (D-21: `MinRowAgeDays` default 7 on the cohort predicate, plus the
`instrument_lost` outcome the enum previously could not express). What is NOT closed: `ApplyReExtraction` still
assigns `ResolutionMethod`/`InstrumentId` unconditionally from its own one-shot resolve — NULL on a miss, overwriting
a good value rather than declining to write. `ReExtractResolutionAdapter.ResolveOnlyAsync` is a STRICTLY NARROWER
cascade than the live one (ticker-in-quote plus `ResolveLocalFromQuoteAsync` with `enableRag=false`, and none of the
`DeterministicResolver` legs), so it structurally cannot reproduce what those legs ground. Measured 2026-08-15 over
all 671,571 rows: of the 84,531 claimed more than 7d after their `extracted_at` — i.e. genuinely aged rows, the
population the floor still admits — **49,616 lost an instrument against 213 that gained one**. So a row resolved by
`llm_candidate_pick` is still stripped, just 7 days later. The fix is to make the overwrite conditional on the new
resolve being BETTER (never null out a held instrument on a miss); it is a separate decision from the floor and was
deliberately not bundled with it. Re-check with the `instrument_lost` outcome now that it exists —
`sum(rate(sentinel_reextract_rows_processed_total{outcome="instrument_lost"}[1h]))` — rather than by diffing
`OriginalInstrumentId` against `instrument_id`, which is how this had to be found the first time.
Two traps that survive the fix. (1) Do not re-check the ratio against `outcome="recovered"`: `ClassifyOutcome` emits
`Recovered` only when the SYMBOL CHANGES, so a row that gains an instrument under an unchanged symbol is classed
`Unchanged` — the counter undercounts recoveries by construction, and a cumulative read is worthless for hours after
any container restart. (2) `NoResolutionSweepWorker` is EXONERATED and should not be re-suspected: it only calls
`SetReviewStatus`.
Historical note for the POST-erasure caveats referenced above: the erasure already happened, so `instrument_id`
readings taken before 2026-08-15 understate the real resolution rate by an unknown margin. At the time of the fix the
historical backfill was DRAINED (0 of 671,571 rows had a null watermark) and every row claimed in the preceding 7 days
was extracted the same day — 100% of the worker's throughput was live traffic, which is what the floor stopped.

**The candidate surface filter gates 4.3% of the rows that attach instruments; 95.7% resolve without ever meeting
it.** The guard is not broken and does not need fixing — `EntityResolutionPrepass.ApplySurfaceFilter` is
unconditional (`const string mode = "enforce"`, `EntityResolutionPrepass.cs:396`, no flag) and live: 30d
`sentinel_candidate_surface_filtered_total{mode="enforce"}` carries 12 reason series (institution 9,190,
gpe_country 167). It is POSITIONED wrong. `Classify` has three production call sites — the NER-candidate prepass
(`EntityResolutionPrepass.cs:404`), Rule 2.5's paid-Gemini leg (`DeterministicResolver.cs:604`, D-6) and its V1
mirror (`GeminiSymbolFallbackService.cs:85`, D-12) — while the LLM-extracted `SubjectEntity` reaches
`DeterministicResolver` through Rule 1 (`:60`) and Rule 2 (`:124`, raw `SubjectEntity` straight to hybrid resolve),
neither of which consults it. Measured over `extracted_at` [2026-07-15, 2026-08-15) reading
`OriginalInstrumentId`/`OriginalResolutionMethod` — **never the live columns; ReExtract erases those, see the two
entries above** — **45,831 of 47,891 instrument-attaching rows (95.7%) take an unfiltered leg** (llm_candidate_pick
30,575 + hybrid_subject 14,675 + llm_candidate_exact 581; only gemini_fallback's 2,060 passed the filter), and
**7,957 (16.6%) carry a subject the filter already has a verdict on** (gpe_country 7,184 over 38 distinct surfaces,
crypto 747, institution 26). That 16.6% is a FLOOR: it was replayed in SQL from the three exact-match sets only,
the shape classes (byline, garbled, multiline, bare-suffix) were not re-run. Consequence in the same window:
`U.S.` -> `U` (Unity Software) 2,470 times, `S&P 500` -> `S` 683, `Sensex` -> `SNSE` 677, `Wall Street` -> `IEP`
243, `yen` -> `U` 237 — **3,060 country-subject rows land on `U` alone**, and not one of those pairs arrives via
`gemini_fallback`, the one leg that is filtered.
SAME-ARTICLE CONTROL, and it is the cheapest re-check: `raw_content_id=146707` has 12 rows, every one
`subject_entity='U.S.'`, one process. Its `extracted_at` is 2026-08-15T03:26-03:27Z — **3.4h AFTER the window
above closes**, so re-running the window query will NOT return it; it is a separate, still-decisive observation,
not one of the counts above and not a fabrication. Nine attached `U` via `hybrid_subject`; Loki carries exactly
three `leg=sentinel-v2-direct decision=rejected reason=gpe_country surfaceJson="U.S."` lines for that id
(same timestamps, trace `b9dd745b5aa5d29976eaf84055a88298`). Identical string, identical source, opposite
outcomes — the three rejected are precisely the rows Rule 2 failed to resolve and which therefore fell through to
Rule 2.5 where the filter finally ran. The filter sits AFTER Rule 2, so it only ever sees Rule 2's misses.
THREE CAVEATS, because the obvious fix — "hoist the filter, country subjects are junk" — is wrong on all three:
(1) **country subjects also produce DEFENSIBLE resolutions**, so the discriminator is country -> single-issuer
EQUITY, never country -> anything. Same window: `Brazil` -> `EWZ` 446, `Germany` -> `DAX` 345, `Middle East` ->
`EIS` 248, `Israel` -> `EIS` 242, `China` -> `GXC` 190, `South Korea` -> `EWY` 176, `Mexico` -> `EWW` 133,
`Taiwan` -> `EWT` 89, plus FRED macro series (`India` -> `DEXINUS` 123, `China` -> `NGDPXDCCNA` 73). A
country -> reject rule destroys these.
(2) **the filter false-positives on live issuers today.** `IsInstitution`'s narrow-generic arm rejects any name
with no corporate suffix whose last word is in `GenericLastWords` — which includes `Association`. `Bancorporation`
is absent from `CorporateSuffixes` (only `Bancorp` is there), so `Zions Bancorporation, National Association`
(5 rows in-window, 7 all-time) and `Flagstar Bank, National Association` (5 in-window, 7 all-time) both classify
as `institution` — and both are ACTIVE catalog issuers, held under the abbreviated form of the very words that
trip the rule (`ZION` = `ZIONS BANCORP NA`, `FLG` = `FLAGSTAR BANK NA`, both `is_active`). The bare
`Zions Bancorporation` (4 in-window, 12 all-time) Keeps: same company, two surfaces, opposite verdicts. Hoisting
the filter onto the resolution path promotes that false-positive from "skips a paid call" to "silently drops a
real resolution" — the exact error D-1/D-5's PRECOND is built to avoid.
(3) **`Sensex` / `S&P 500` / `yen` are unfixable at either ingress.** They arrive on `llm_candidate_pick`: same
window and same Original-column rule, `GROUP BY subject_entity, coalesce("OriginalResolutionMethod",
resolution_method)` over every instrument-attaching row bearing the surface — so the denominator is the SUBJECT'S
WHOLE POPULATION, not the single-symbol pairs listed above — `Sensex` 675/677, `S&P 500` 1,221/1,230, `yen`
234/238; the balance is `hybrid_subject` (2, 9, 3) plus one `gemini_fallback` `yen` -> `DEXJPUS`. And Rule 1's PICK
IS CORRECT — "Rule 1 picked the wrong row" is REFUTED, measured over the same window across the WHOLE
`llm_candidate_pick` leg (the 30,575 of the 95.7% count above, not a sub-slice of it): `subject_entity` equals the
picked candidate's `Name` (`candidate_symbols_json -> selected_candidate_index ->> 'Name'`) in 30,575 of 30,575
rows case-insensitively and 30,573 exactly, the 2 exceptions being case-only
(`NASDAQ`/`Nasdaq`, `Blackrock`/`BlackRock`). The wrong answer is a SUBSTITUTION
AFTER the pick — the resolver fuzzy-matches the candidate's model-authored slug instead of its `Name`; the full
account belongs with the Rule 1 entries above, not here. Either way the subject SURFACE is not the input that went
wrong, so no surface filter at any position can help. By contrast `U.S.` -> `U` (2,470) and `Wall Street` -> `IEP`
(243) are entirely `hybrid_subject`, i.e. genuinely subject-driven and in scope for a seam.
TWO CANDIDATE SEAMS, not chosen — measure before picking. **Seam A**: hoist `Classify` out of
`TryGeminiResolveAsync` up to `ResolveAsync` entry (~`DeterministicResolver.cs:46`). `_surfaceFilter` is already
injected and `ResolveAsync` has one production caller (`V2ExtractionPipeline.cs:77`), so the change is small — but
it puts every false-positive in caveat (2) directly on the resolution path. **Seam B**:
`DslToMergedExtractionAdapter.cs:499`, where `SubjectEntity` is born, which is where GIGO says to clean and which
covers SecMaster, Gemini, `extracted_observations.source_entity` and the matrix in one edit (the D-15 precedent) —
but it is upstream of the candidate list, so it cannot address caveat (3) either.
Whichever seam wins, land a counter for rows attaching on a subject the filter would reject; today that number is
obtainable only by replaying the classifier over the DB in SQL, which is how this entry was written.

**gemini-resolver runs at 100% of its daily cap while its gate rejects ~1 call in 3,000.** Measured 2026-08-14:
`gemini_resolver_live_calls_24h` 1500 against `gemini_resolver_daily_cap` 1500, `gemini_resolver_gated_24h` = 1 of
3,076 total calls, and 877 of SecMaster's 3,425 dispatches/24h refused as `cap_exhausted`. Refusal is
first-come-first-served, so genuine resolutions are dropped at random once the window is spent. `_company_gate`
(gemini_resolver/server.py) is purely syntactic — it rejects money, markup, code-slugs and 13 abbreviations, and
cannot reject a well-formed noun phrase that is not a tradeable issuer, which is what the junk is
("Birmingham Legion", "Hellenic Shipping News World", "Focus On Inflation"). Not a matrix-corruption event as of
this measurement: recent Gemini self-seeds are legitimate issuers. Re-check with `curl :9300/health`.

**The gemini-resolver daily-cap refusal is a fourth outcome that increments nothing, so refused demand has no
metric at all.** `try_reserve_live` (`gemini-resolver-mcp/gemini_resolver/server.py:732-747`) gates dispatch on
`if live >= daily_cap: return False` (`:744-745`). The refusal branch (`:1040-1054`) logs a WARNING (`:1050-1053`)
and raises `HTTPException` 429 (`:1054`) — and increments nothing. `server.py` defines exactly three
`prometheus_client` Counters: `c_ledger_live` (`:883`), `c_dispatch_rejected` (`:900`, incremented only at `:1091`
for Google-side 401/403/429) and `c_breaker_refused` (`:905`, incremented only at `:1024` for an open breaker). A
cap refusal increments none of them, and does not call `record_gated()` (`:952`) — that is reserved for the
company-name gate, a different rejection class.
MEASURED 2026-08-20: **35** cap refusals in 30 minutes (**754** in 24h), in bursts of up to **14** within a single
second (24h to 2026-08-20T18:05Z; top per-second counts 14, 14, 12, 10, 10, and a third second also hit 10), while
`gemini_resolver_gated_24h`, `gemini_resolver_dispatch_rejected_total` and
`gemini_resolver_breaker_refused_total` **all read 0**. The quantity that says how much real resolution work is
being dropped exists only as a log line.
THE PROCESS IS BURSTY — A RE-CHECK WILL NOT LAND NEAR THESE NUMBERS. Three measurements across ~an hour gave
**35 / 67 / 69** for the 30-minute count and **754 / 792 / 818** for the 24h count: the short window swings ~2x,
the 24h window drifts under 10%, so compare against the 24h figure and treat the 30-minute one as an order of
magnitude only. Re-check:
`sudo journalctl -u gemini-resolver-mcp --since "30 min ago" | grep -c "daily call cap"` against
`curl -s http://localhost:9300/metrics | grep -E "gated_24h|dispatch_rejected_total|breaker_refused_total"`.
RUN BOTH HALVES. `grep -c` prints `0` and exits 1 if the log wording ever drifts, which is indistinguishable from
"no refusals" — the `/metrics` half is what separates the two, because a real fix moves one of those counters OFF
zero while a wording drift leaves all three at zero. The grep alone can read "fixed" when it merely stopped
matching. (EVERY LINE CARRIES TWO TIMESTAMPS FOUR HOURS APART, AND `--utc` FIXES ONLY ONE OF THEM. `journalctl`
prints LOCAL time here — mercury is `America/New_York` — so pass `--utc`; but the Python logger writes its own
NAIVE LOCAL timestamp into the message body, which `--utc` does not touch. Measured 2026-08-20 with `--utc` in
force: `Aug 20 18:05:28 mercury python[284493]: 2026-08-20 14:05:28,178 WARNING ... daily call cap 1500 reached`.
The journal stamp on the LEFT is the UTC one; the in-message stamp on the right is still local. Quote the left.)

THE CAP ITSELF IS HOLDING — record this so nobody re-raises it. Ledger file birth **2026-08-06T16:20:51Z**
(`/opt/ai-inference/gemini-resolver-ledger.db`; `stat` reports it as `2026-08-06 12:20:51 -0400`, and mercury is
`America/New_York`, so read that field as local and convert). `ledger_meta.lifetime_live_calls` = **18,586** over
**14.04 days** = **1,324/day**; every measured restart-to-restart interval is at or below 1500/day; the full UTC day
2026-08-19 is **exactly 1500** live, against `gemini_resolver_daily_cap` 1500. The gauge is bounded by real
enforcement, not by a display clamp.
INTENT VIOLATION. The alert's own annotation
(`deployment/artifacts/monitoring/alerts/gemini-resolver.yml:320`) states "A true last resort is dozens/day".
Measured sustained volume is ~1,324/day — **15-100x the documented design intent**. This is the CLAUDE.md
INTENT_FIDELITY worked example recurring in the service it was written about: the frontier last-resort operating as
a primary path.
WHAT IS BEING SENT — **a FAILURE-BIASED SAMPLE**, because only failed or truncated calls log their subject at
production log level. It is qualitative and cannot support a fraction. Distinct `subject=` values over 3 days
include: tech sector, euro zone, Brazilian currency, Dollar Index, New Orleans, government agencies, Reserve Bank of
Australia, Donald Trump, Large Fries, Hershey's Zero Sugar, Coke Zero, Health Care and Social Assistance, and a
headline fragment carrying an embedded newline (`'Data Center Growth Remains a Key Driver\nFabrinet'`).
`_company_gate` (`gemini-resolver-mcp/gemini_resolver/server.py:412-433`, with its regexes and abbreviation set at
`:403-409`) is a SHAPE filter — money/number strings, markup, hyphenated code slugs, a fixed 13-entry abbreviation
list, and >10-word boilerplate — not a semantic company classifier, so sectors, cities, currencies, people and
product names pass it **by design**.
`gated_24h` near zero is therefore consistent with the gate working as specified, not with it being broken.
RE-CHECK CAVEAT. The ledger's `call_events` table is pruned to 48h (`PRUNE_RETENTION_WINDOWS = 2` at
`gemini-resolver-mcp/gemini_resolver/ledger.py:39`, applied at `:187` as
`ts < reference - PRUNE_RETENTION_WINDOWS * window_seconds`), so it **cannot answer any question spanning more than
two days** — measured span 2026-08-18T17:03:59Z to 2026-08-20T17:04:04Z, 6,569 rows. For longer windows use
`ledger_meta.lifetime_live_calls` plus restart checkpoints from `journalctl`. Open the file read-only
(`sqlite3.connect("file:...?mode=ro", uri=True)`); the service is writing it.

**METHOD NOTE — a Prometheus Counter's `_created` is the exporting PROCESS's age, not the data's.** It is stamped at
object instantiation. Dividing a persisted lifetime total by an age decoded from `_created` produced "**2,803**
calls/day against a 1,500/day cap" and a false conclusion that the cap was not enforcing. Measured 2026-08-20:
`gemini_resolver_ledger_live_calls_created`, `..._dispatch_rejected_created` and `..._breaker_refused_created` all
decode to **2026-08-14T02:02:47.98Z**, which is exactly `systemctl show gemini-resolver-mcp -p
ActiveEnterTimestamp` (`Thu 2026-08-13 22:02:47 EDT`) — the last restart, and nothing to do with the ledger. The
error is stable, not a one-off arithmetic slip: re-running it the same way at 17:27Z gives 18,586 over a 6.64-day
process age = 2,799/day, while the ledger's true 14.04-day age gives 1,324/day and the cap holds. Use the data
store's own age, never a Counter's `_created`, and cross-check against a second source.

**The merge gate reads a shell redirect as a PR number.** `gh pr merge <N> --squash 2>&1` denies with "names more
than one PR number: 2 <N>". Same redirect-parsing class the push guard already fixed; a third guard still carries it.
Workaround until fixed: drop the redirect.

**`whole_act_git` still picks its subcommand the naive way, so a git global option hides the two acts that name no
path.** `ansible-gate-guard.sh`'s operand dispatch now finds the subcommand by SHAPE, which is what opened `git -C
<dir> show <rev>:<gate>` and closed `git -C /opt/ai-inference checkout -- compose.yaml`. `whole_act_git` was not
changed and takes the first non-dash token as the subcommand, which under `-C <dir>` or `-c <k>=<v>` is the option's
VALUE. Measured 2026-08-17 on ddbaff89 and on the fix alike: `git -C /tmp/r clean -fdx` and `git -c user.name=x clean
-fdx` ALLOW, while the bare `git clean -fdx` denies; same for `git -C /tmp/r update-index --cacheinfo …` vs the bare
spelling. A pathspec-less `clean -x` removes IGNORED files, and `.claude/settings.local.json` — the wiring for every
hook in this layer — is ignored globally, so this is the most complete unwiring available and it is one flag away.
Fix is the same two-line shape test already used in the operand dispatch (`[[ "$w" =~ ^[a-z][a-z0-9-]*$ ]]` when
selecting `sub`). Re-check: `git -C /tmp/r clean -fdx` must deny.

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

**A quarantined row no longer retires its SYMBOL — the index is partial (CLOSED 2026-08-15).**
`idx_instruments_symbol` was a FULL (non-partial) unique index, so a soft-deleted row owned its symbol exactly as
hard as an active one and no row could ever take that symbol again. Migrations `QuarantineGeminiJunkInstruments`
and `QuarantineGeminiEquityEtfJunk` (2026-07-18) soft-deleted 91 rows to retire a junk NER-surface NAME —
`HON`="TD Cowen", `MELI`="Amy Legate-Wolfe", `AEM`="Toronto Stock Exchange", `ANGX`="Minions & Monsters" — and
the second migration's own header says every one of its 82 symbols "is a real, tradeable ticker", so the catalog
became permanently unable to hold Honeywell, MercadoLibre, Agnico Eagle, Diageo, AB InBev or Mesoblast.
FIXED by migration `ScopeSymbolUniquenessToActiveRows`: uniqueness is now scoped to `is_active = true`, so at most
one ACTIVE row holds a symbol while any number of quarantined rows may share it. Pre-flight on prod 2026-08-15,
both pure SELECTs: ZERO symbols have two or more active rows (so the partial index is satisfiable without moving a
row), and ZERO of the 91 quarantined symbols is also held by an active row (so all 82 tickers are genuinely free
rather than merely unblocked at the index). No row was repaired — this frees the SYMBOL only.
91 IS THE QUARANTINE COUNT, NOT THE TICKER COUNT, and this entry once headlined 91 while its own body cited the
migration's 82. Re-measured 2026-08-15, `SELECT asset_class, count(*) FROM instruments WHERE is_active=false
GROUP BY asset_class` = Equity 74 / ETF 8 / fred_series 8 / Economic Indicator 1. So 9 of the 91 are not
tickers at all: 8 `asset_class='fred_series'` (`NAPMII`="Asian share markets", `CONCCONF`="US",
`WCSPOIL`="U.S. Strategic Petroleum Reserve", `MCRFPC1`="Justin Trudeau", …) plus 1 `'Economic Indicator'`
(`GSV.NE`="2025 full year GSV"). Those 9 are the D-4 macro-junk class — a hallucinated macro label, which is a
different defect from a retired equity ticker and wants a different disposition. Every count below is therefore
stated over the population it actually applies to, 82 or 91, never both at once.

**The quarantine refusals are now a POLICY nobody has decided, not a constraint.** This is what the partial index
turned from impossible into optional, and it is the live question. While the index was FULL, `CatalogService.cs`
and `EntityResolutionService`'s self-seed had no choice: the INSERT behind a quarantined row was a guaranteed
23505, so dropping the item / skipping the seed was the only reachable outcome. Both now REFUSE deliberately —
the insert would succeed — because re-acquiring a retired ticker is a catalog-repair decision, and resolution
time, per candidate, silently, is the worst place to take it. The code and its tests say so at both sites
(`quarantined_skip`, and the discovery-cache `continue`). CONSEQUENCE while it stands: `CatalogService.cs` drops a
quarantined discovery item with a bare `continue` and no `results.Add`, so `EntityResolutionService.cs:850`
`result.Results.FirstOrDefault()` is null and a CompanyName candidate loses its ticker PROPOSAL — the surface
enters the confirm cascade with nothing proposed, and every news mention of one of the 82 pays the full cascade
(per D-1, up to the paid Gemini leg). It is also reachable from the `search_catalog` MCP tool, i.e. outside entity
resolution entirely, where the item simply vanishes from an operator's catalog search. Deciding this means
choosing per path, not globally: an operator-curated config (`InstrumentConfigurationWatcher`, which reactivates
by design) and a collector registration are authoritative in a way a news-surface self-seed is not.
Re-check: the `is_active=false` count above, plus
`sum(secmaster_entity_resolution_self_seed_total{result="quarantined_skip"})` — which
is a FLOOR on the wall-hits, NOT "the live rate" this entry first called it. That tag is emitted at ONE of four
self-seed skip paths (`EntityResolutionService.cs:1038`); silent are `:877` (unconfirmed), `:901`
(contextFactor<=0), `:1002` (EnableSelfSeed=false), and the `CatalogService.cs:206` drop above, which carries a
LogWarning but no metric at all. Left as a floor rather than closed by emitting at `CatalogService.cs:206`,
deliberately: that would still leave three silent paths, so the qualification is needed either way, whereas the
emission is a new untested signal in a PR whose thesis is the pre-insert read. Measured 2026-08-15, prod carries
three series — `idempotent_skip`=1492, `inserted`=67, `error`=22 (the 23505s this PR stops) — and NO
`quarantined_skip`, because the emitting code is unmerged. `EntityResolutionSelfSeed` is also absent from
`MetricWarmupHostedService`, so even after deploy an absent series cannot be read as zero. The tag VALUE is
pinned by `EntityResolutionServiceTests.should_record_the_quarantined_skip_result_tag_when_the_symbol_is_held_by_a_quarantined_row`;
without it this PromQL could be renamed or typo'd and ship green (the pre-fix proxy,
`{service_name="SecMaster"} |= "Self-seed persistence failed"`, is VARIABLE, not a fixed rate: 30 lines/24h with
ANGX 21 of them in the window ending 2026-08-14T12:00Z, 7 with ANGX 6 ending 2026-08-15T13:00Z, so a low
post-deploy reading is that variance and not a regression). The two candidate repairs this entry once posed have
been RESOLVED IN FAVOUR OF THE SECOND: not "reactivate with `name = symbol` and let the D-2 enrichment fill-gaps
re-name them", but "make the unique index partial on `is_active` and let a fresh row take the symbol". The
reactivate option was rejected on this entry's own evidence — it strands the 9 non-equity rows permanently at
Name=Symbol (see the pool-membership split below) and reinstates the junk names the quarantine existed to remove.
The partial index touches no row, so it carries neither hazard.
TWO NUMBERS, NOT ONE — this entry previously welded them into a single conjunct ("all 91 satisfy Figi=null AND
Country=null … so all 91 are in BOTH enrichment pools"), and only the first half was true. Figi=null AND
Country=null does hold for 91 of 91 (measured 2026-08-15). Pool MEMBERSHIP is 82 of 91, because both candidate
queries also require an equity-shaped class, `EquityAssetClasses {Equity,ETF,Stock}` —
`OpenFigiEnrichmentBackgroundService.cs:112` (IsActive AND class AND Figi==null) and
`CatalogEnrichmentBackgroundService.cs:96` (IsActive AND class AND (AtlasSectorCode==null OR Exchange==null OR
Country==null)); same SELECT plus `asset_class IN ('Equity','ETF','Stock')` = 82. CONSEQUENCE, and it is the
reason the conflation mattered: the reactivate-with-`name = symbol` repair would strand those 9 PERMANENTLY at
Name=Symbol. The class filter keeps them out of BOTH candidate queries, so no timer ever selects them and the
fill-gap that was supposed to re-name them never runs — the row goes active, un-named, and stays that way. The 9
need their own disposition decided BEFORE that migration runs, not discovered after it.
A THIRD SITE shared the same predicate mismatch and is fixed BY THE INDEX rather than at the call site.
`RegistrationService.cs:309` is another active-only pre-insert read — `GetBySymbolAsync`, whose `IsActive` filter
is itself load-bearing for the FRED-pollution recovery path, so it could never simply be swapped — feeding
`AddAsync` at `:391`. A collector registering any of the 82 real tickers used to hit a guaranteed 23505 there;
under the partial index a quarantined row cannot raise one at all, so that path now INSERTS, which is the desired
outcome for a collector (an authoritative source, unlike a news surface). This is the payoff of fixing the
constraint at the SOURCE rather than gating each caller: the site needed no patch.
THAT HOLDS BY POPULATION, NOT BY CONSTRUCTION, and the distinction is what makes it re-checkable. Only
`idx_instruments_symbol` was scoped to `is_active`; `idx_source_mappings_collector_source` is still UNIQUE on
`(collector, source_id)` GLOBALLY, with no `is_active` predicate, so a quarantined row's source mapping still
reserves its `(collector, source_id)` pair exactly as hard as a live one. Register the same pair again and the
insert at `RegistrationService.cs:391` raises a 23505 the index change does not touch. It does not fire today
because the quarantined rows almost never carry a mapping — measured 2026-08-15, `SELECT asset_class, count(*),
count(*) FILTER (WHERE EXISTS (SELECT 1 FROM source_mappings m WHERE m.instrument_id = i.id)) FROM instruments i
WHERE NOT i.is_active GROUP BY 1`: 1 of the 91 quarantined rows carries one (`GSV.NE`, SentinelCollector, the
`Economic Indicator` singleton), and 0 of the 82 real tickers (Equity 74 / ETF 8) do. Re-run that query before
relying on the claim: it turns false the moment a quarantined row acquires a mapping, and nothing alerts on it.
Its
`DuplicateInstrumentException` (`: Exception`, so it passes `RegisterWithRetryAsync`'s `DbUpdateException`-only
catch at `:216`) was caught by `catch (Exception ex)` at `RegistrationService.cs:135` and surfaced as
`Success=false`, never escaping. Re-check that it stays quiet: `{service_name="SecMaster"} |= "Failed to register"`
read 0 over 7d before the change (measured 2026-08-15T13:00Z; the same selector unfiltered carries 12,444 lines in
that window, so the zero is a real zero and not a dead query) — it was not firing then either, so a post-deploy
zero confirms nothing on its own.
Dead code at the same site, worth deleting
whenever that policy lands: `RegistrationService.cs:313` `if (!instrument.IsActive)` sits AFTER the
`IsActive`-filtered read, so its operator-facing "registering against INACTIVE instrument" warning is
UNREACHABLE. Its sibling at `:258` IS live and must not be removed with it — that one reaches the instrument
through `existingMapping.Instrument`, a different lookup with no active-only filter.

**`PatternDataSeverelyOverdue{pattern_id="buffett-indicator"}` has fired unbroken for 14 days and the feed is
healthy — the threshold is wrong for this series' release cadence.** PENDING since **2026-07-31T18:48:11Z**
(`ALERTS_FOR_STATE` = 1785523691 — that gauge is the PENDING start, not the firing start), and the rule carries
`for: 24h`, so FIRING since **2026-08-01T18:48:11Z**: **14.04 days** to the 2026-08-15T19:52Z anchor.
Prometheus scrapes every **15s** (`count_over_time(up{job="otel-collector"}[1h])` = 240, measured 2026-08-15T21:05Z),
so `count_over_time(ALERTS{...,alertstate="firing"}[15d])` = 80,895 samples IS those 14.04 days
(80,895 x 15s = 1,213,425 s) — sample count and timestamp arithmetic agree independently, which is what says
continuous rather than flapping. Both figures are needed to re-check this and neither is derivable from the other
without the 15s interval and the 24h `for` offset, so both are recorded here.
Rule: `deployment/artifacts/monitoring/alerts/thresholdengine.yml:100-108`, `overdue_days > 120 for 24h` — a
HARDCODED 120. Live gauges: `age_days` 226, `overdue_days` 136, `severe_overdue_threshold_days` **270**.
**Collection is NOT broken.** `buffett-indicator` requires `NCBEILQ027S` + `GDP`; both were collected
2026-08-15T18:00Z. The metric tracks NCBEILQ027S, whose newest observation is 2026-01-01 — exactly 226 days old —
because FRED Z.1 simply has no newer point. Our own vintages give the real release lag: 2026-01-01 first seen
2026-06-12 (162d), 2025-10-01 -> 2026-03-22 (172d), 2025-07-01 -> 2026-01-16 (199d). A quarterly point stays newest
until its successor is RELEASED, i.e. until `date + 90 + lag`, so age peaks at **252-289 days** and — since
`overdue_days = age_days - publicationFrequencyDays` and the effective frequency is the SecMaster-derived 90 —
**peak `overdue_days` EQUALS the release lag: 162-199 every quarter in perfect health.** Even the SMALLEST observed
lag exceeds the rule's `> 120` by 42 days, so this alert is guaranteed to fire every quarter on a healthy feed.
136 is INSIDE the normal wait; Q1-2026 Z.1 is due ~2026-09.
**Decision: re-threshold, do not retire and do not touch the feed.** Two independent defects, and the alert rule was
deliberately NOT changed in the PR that recorded this. **(2) is now CLOSED and (1) is OPEN but no longer
alert-affecting** — see the two notes appended below.
(1) The pattern's authored `"publicationFrequencyDays": 120` is DEAD CONFIG —
`ThresholdEngine/src/Configuration/PatternConfigurationLoader.cs:320-322` overwrites it unconditionally with
`PublicationFrequencyDaysOverride ?? RequiredSeries.Max(frequencies)`, so only the OVERRIDE is honoured and the
authored 120 is discarded in favour of the SecMaster-derived 90. Same class as WM2NS/#898-#899: publication cadence
!= data frequency. buffett-indicator has NO override. Since peak `overdue = 90 + lag - override`, holding `overdue <= 120` needs
`override >= lag - 30`, so the WORST observed lag (199d) puts the floor at **>= 169**. An earlier revision of this
entry recommended **>= 142**, which is `lag - 30` for the 172d lag and fails against the 199d lag printed three
lines above it — corrected 2026-08-15. **180** clears the floor with headroom and also moves
`severe_overdue_threshold_days` to `max(3*override, 14)`. Re-derive the floor if a longer lag is ever observed.
(2) The rule ignores the per-pattern gauge that exists FOR it. `PatternEvaluationService.cs:74-79` says
`thresholdengine_pattern_severe_overdue_threshold_days` was emitted "so an alert can fire per-pattern ... without
duplicating the threshold formula in PromQL (single source: PatternDataHealthEvaluator)". The rule hardcodes 120
anyway, so for 14 days the alert has said SEVERELY OVERDUE while the service's own health evaluator does not
(136 < 270). Fixing only (1) leaves the two surfaces free to disagree again.

**(2) CLOSED 2026-08-17.** The rule now joins `thresholdengine_pattern_data_overdue_days` against
`thresholdengine_pattern_severe_overdue_threshold_days` on `pattern_id`, so the two surfaces read one number.
Measured over all **71** patterns publishing both gauges at 2026-08-17T11:17:04Z: `buffett-indicator` 138 vs 270
stops firing, and **zero** patterns lose coverage (`count(overdue unless on(pattern_id) threshold)` = 0). A pattern
whose threshold is NOT published falls back to the old literal 120 rather than dropping out of the join — the
alternative silently un-covers it — and carries `threshold_source="fallback_literal_120"` so the degraded mode is
visible. Guards: `deployment/tests/alerts/thresholdengine_test.yml`, 14 assertions; reverting the rule to `> 120`
turns 5 of them RED, the buffett negative among them. Re-check: `promtool test rules ./thresholdengine_test.yml`
after restoring the literal must FAIL.
**(1) stays OPEN, but its alert consequence is gone.** With the join in place, the peak `overdue` of 162-199 sits
under the derived threshold of 270, so the authored-vs-derived frequency mismatch no longer produces a false page and
the `>= 169` override floor above is no longer needed to stop one. What remains is the trap itself: a pattern author
writes `publicationFrequencyDays` and `PatternConfigurationLoader.cs:320-322` discards it without a word. Re-check:
author any value in a pattern JSON with no `PublicationFrequencyDaysOverride` and confirm
`thresholdengine_pattern_severe_overdue_threshold_days` for it still reads `max(3 * SecMaster-derived freq, 14)`.
The SAME `Max`-over-inputs rule has a second, distinct consequence recorded in the dead-feed entry below: on a
MIXED-cadence pattern it lets a stalled MONTHLY input mask a dead DAILY one (`truflation-vs-cpi` judged at 90 days
where its Daily series implies 14). Same line, two defects — fix one and re-read the other before closing either.

**Four more patterns are within 4 days of the same crossing**, all climbing +1/day: `challenger-layoff-surge`,
`challenger-vs-payroll`, `sentinel-challenger-divergence` and `truflation-vs-cpi` all read **86 against a threshold
of 90** at 2026-08-17T11:17:04Z, so they cross ~2026-08-21 unless their sources publish — but the ALERT fires
~2026-08-23, not 08-21: the strict `>` plus `for: 24h` add two days, derived in the dead-feed entry below, so seeing
nothing on 08-21 is the rule working, not a broken join. Recorded so that a burst of
new alerts a few days after this deploy is recognised as the pre-existing feed lag it is, and not read as the join
misfiring. Re-check: the same gauge for those four `pattern_id`s.

**The `BrokenCircuitException` classification gap is OPEN. Recovering the 55 `raw_content` rows it already orphaned
is CLOSED as won't-do — alpha decay.** Two opposite verdicts on one finding; revised 2026-08-17, superseding the
2026-08-15 revision that framed the whole thing as pending data loss. Every measurement below still reproduces
(re-measured 2026-08-17) — the VERDICT moved, not the evidence.

**Won't-do: the 55 rows. Letting them prune is the DECISION, not an oversight.** Financial news alpha-decays — its
predictive value falls off sharply with age — so re-extracting content collected 2026-07-19 -> 2026-07-24 buys close
to nothing now that it is ~4 weeks old (measured 2026-08-17: oldest row 28d 17h, all 55 still present, all 55 still
childless). The one-off `POST /admin/reprocess` that would recover them is deliberately NOT worth making, and the
prune described below is the EXPECTED OUTCOME rather than a loss. **A later re-check returning 0 is this decision
landing on schedule** — not fabrication, not an unexplained count, and not grounds to re-raise recovery.

**Open, and this is the whole of the remaining value: the classification gap itself.** It is not scoped to these 55 —
the NEXT breaker-open event orphans whatever is in flight at the time, which is FRESH content, and fresh is exactly
where the predictive value lives. The alpha-decay argument does not reduce this defect's importance; it INVERTS where
that importance sits: all of it is now in preventing future orphaning, none of it in backfilling past rows. A vLLM or
llama-server outage long enough to trip the breaker (3 consecutive failures, `DependencyInjection.cs:342`, `:359`)
silently abandons that day's articles with no retry and no queue presence.
**The precedent for the fix is already in-repo** — three other SentinelCollector call sites name
`BrokenCircuitException` explicitly (`DigestNarrativeGenerator.cs:331`, `FinnhubLookupClient.cs:51`,
`VllmClient` count-tokens per `VllmClientCountTokensTests.cs:59`); `ExtractionProcessor`'s classifier is the one that
does not. Fix = add it to the transient set at `:967-973`, with a guard test asserting `RetryCount` increments on a
circuit-open failure instead of taking the permanent branch.

**The measurement that re-derives all of the above** — every number below is unchanged from the 2026-08-15 revision;
the "438 orphans" figure that originally surfaced this is a mixed predicate, so quote the 55, not the 438.
`sentinel.raw_content` (94,799 rows) carries **440** rows with a terminal `processing_error`: 377 `age_cutoff`,
**55** `The circuit is now open and is not allowing calls.`, 7 HttpClient-timeout, 1
`v2_pipeline_failed: prompt_too_large_after_chunking`. The 438 was `age_cutoff` (all 377) PLUS the never-extracted
error rows, which is why it does not reconcile against any single predicate.
**All 377 `age_cutoff` rows DO have extraction rows** — they are a deliberate too-old refusal, not orphans. Of the 7
timeouts, 2 were retried and succeeded. So the genuinely-abandoned set is **61**: the 55 circuit-open, 5
timeout-exhausted, 1 prompt-too-large. (11,576 rows have no extraction at all, but 11,515 of those have
`processed_at` set with `processing_error IS NULL` — processed cleanly, zero observations, which is normal.)
The 55 are the finding: collected **2026-07-19T17:14:25Z -> 2026-07-24T11:00:06Z**, `retry_count = 0` for all 55,
`processed_at IS NULL` for all 55. They were never retried ONCE. Cause is a classification gap, not exhaustion:
`SentinelCollector/src/Workers/ExtractionProcessor.cs:967-973` treats only `TimeoutException`,
`TaskCanceledException` wrapping one, and `HttpRequestException` with null/5xx/429/408 as transient. Polly's
`BrokenCircuitException` matches none, so a circuit-open failure took the PERMANENT branch on the FIRST attempt
(`:989`) without incrementing `RetryCount`. Contrast the 5 timeout orphans, all at `retry_count = 3` (`MaxRetries`) —
those genuinely exhausted.
**Nothing will ever pick them up.** Every queue predicate is `ProcessedAt == null && ProcessingError == null`
(`RawContentRepository.cs:40`, `:66`, `:186`, `IRawContentRepository.cs:14`, and the queue-depth gauge at
`SentinelMeter.cs:423-431`); `ExtractionProcessor.cs:508` states it outright — setting `ProcessingError` exits the
row from the queue. The only clearing path is `POST /admin/reprocess`
(`SentinelCollector/src/Endpoints/AdminEndpoints.cs:189`), which requires an
explicit `rawContentIds` array. No sweeper, no retry service. This is why the classification fix alone does NOT
recover the 55 — nothing re-queues them retroactively, the reprocess call would have to be made by hand, and per the
verdict above it will not be.
**THE 55 ARE ON A DELETION CLOCK — that is now the intended ending, and a later re-check returning 0 is the decision
executing, not a missing measurement.**
`StaleContentPrunerService` runs its passes once per host start and pass 1 calls
`repo.DeleteChildlessOlderThanAsync(cutoff, ...)` (`StaleContentPrunerService.cs:83`, pass 2 at `:100`) with
`cutoff = UtcNow - MaxArticleAgeDays` (30 days, the
banner at `:73` says "full 30-day raw_content retention", `ExtractionOptions.cs:588` default, overridden nowhere in
`/opt/ai-inference/compose.yaml`). The predicate is `CollectedAt < cutoff && !r.Observations.Any()`
(`RawContentRepository.cs:106-111`) — these 55 have no extraction children, so they qualify: collected
2026-07-19 -> 2026-07-24, they become deletable **2026-08-18 -> 2026-08-23** and vanish at the first
SentinelCollector restart after that. Because the pruner only runs in `StartAsync`, a long-lived container can hold
them well past the window; 55 on a re-check means "no restart yet", not "decision reversed".
Re-check (psql is SELECT-only):
`SELECT count(*), min(collected_at), max(collected_at), count(*) FILTER (WHERE retry_count=0) FROM
sentinel.raw_content WHERE processing_error LIKE '%circuit is now open%';` — returned
`55 | 2026-07-19T17:14:25Z | 2026-07-24T11:00:06Z | 55` on 2026-08-17. After the prune window an empty result means
pruned as decided, not absent. **What stays open is the classifier, and only the classifier.**

**10 of SentinelCollector's 16 hosted workers log their startup banner at `LogInformation`, so prod has no record
those started — while 2 siblings already log theirs at Warning, on purpose.** Prod log level defaults to Warning,
which makes an Information banner invisible — the opposite of CLAUDE.md OBSERVABILITY ("startup banners STAY
Warning # boot-loop visibility") and of `feedback_warn_vs_info_by_trigger`.
**The precedent is already in-repo, so this is finishing a conversion, not proposing one.**
`AutoApproveDrainWorker.cs:74-76` and `NoResolutionSweepWorker.cs:66-70` both log theirs with `LogWarning`, each
carrying the comment "Startup banner at Warning so a restart loop is visible under the prod WARN log floor (Info is
filtered in prod)" — landed 2026-07-05 in #852 (`d91f1a50`). An earlier revision of this entry said "every startup
banner is LogInformation" and "prod has no record any worker started"; both were false, and the second would have
sent the next reader looking for a precedent that was two files away. Corrected 2026-08-15.
The 10 still at `LogInformation` (measured over the 16 classes deriving from `BackgroundService`/`IHostedService`):
`ReExtractBackgroundService.cs:119-122`, `ExtractionProcessor.cs:50`, `MirrorSearchWorker.cs:79`,
`ResolutionWorker.cs:51`, `StaleContentPrunerService.cs:73`, `RssFeedCollectorWorker.cs:31`, `EdgeSyncWorker.cs:27`,
`SearxngCollectionScheduler.cs:60`, `ValidationEventConsumerWorker.cs:35`, `ValidationQueryExecutorWorker.cs:27`
— plus ReExtract's disabled-by-flag banner (`:93-95`) and its stop banner (`:183`). The remaining 4 emit no startup
banner at all, which is the same blind spot wearing a different shape and should get one.
ReExtract is the one that prompted this: it logs Mode, Cohort, **MinRowAgeDays**, RowsPerMinute, BatchSize and
BackpressureThreshold at Information, so prod cannot confirm the worker started or which `MinRowAgeDays` it read —
the parameter whose default is the live-traffic guard D-21.
Re-check: `grep -rn "class .*: *\(BackgroundService\|IHostedService\)" SentinelCollector/src/Workers` for the
denominator, then the banner level per file. One-line change per worker; belongs in a SentinelCollector PR,
deliberately not bundled with the tooling work that measured it (2026-08-15).

**The merge gate refuses a cross-repository merge on the `gh pr merge` route and permits the SAME merge on the REST
route, so the two routes now disagree about one threat.** `gh api -X PUT repos/<owner>/<repo>/pulls/<N>/merge`
resolves against THIS checkout's verdict marker whatever owner/repo the path names: `git-push-guard.sh:2149` takes
the number with `grep -oE '/pulls/[0-9]+/merge'` and reads neither owner nor repo. The refusal that WOULD catch it is
at `:2305`, and the scans filling `MERGE_FOREIGN` no longer depend on `MERGE_NUMS` being empty — `5911c7a0` made the
third of them unconditional, which is why the `--hostname` FLAG is now caught on this route. What the REST path still
evades is different: it puts the redirect in the URL PATH, and no scan reads a path.
Measured 2026-08-15 by driving the hook the way `.claude/hooks/test/` does — the command as JSON on stdin, an
isolated `ATLAS_MARKER_DIR` carrying an approved-at-head verdict for #99901 and none for #99904, `gh` stubbed, a
temp repo whose origin is `git@github.com:jpansarasa/ATLAS.git`. No merge was attempted and no repository was
contacted. Decisions, at base `64cfe334` / `7a25d23e` / branch tip `9f6e0663`:
`gh api -X PUT repos/attacker/evil/pulls/99901/merge` -> **allow / allow / allow**; the identical call for the
unreviewed #99904 -> deny / deny / deny, which is what proves the number IS read and the marker consulted IS this
repo's; `curl -X PUT https://api.github.com/repos/attacker/evil/pulls/99901/merge` -> allow / allow / allow. So the
hole is PRE-EXISTING, on both the `gh api` and `curl` spellings, and this branch neither opened nor closed it.
The REST route reads neither owner, repo NOR **host**, so a foreign-HOST spelling of THIS repository's own path is
the third shape of the same hole. Re-measured 2026-08-15 at base `64cfe334` and tip
`4f4a35d7`: `curl -X PUT https://evil.example.com/api/v3/repos/jpansarasa/ATLAS/pulls/99901/merge` -> **allow /
allow**, and `gh api --hostname evil.example.com -X PUT repos/jpansarasa/ATLAS/pulls/99901/merge` -> **allow /
allow**. It is the exact counterpart of the `--hostname` redirect the subcommand route DOES refuse, which makes the
route disagreement one shape wider than the owner/repo case alone.
**HALF of that host shape is now CLOSED, as a side effect and not a fix.** The uncut-source scan added at `5911c7a0`
tokenizes the WHOLE command, so a `--hostname` FLAG is seen on any route, verb or no verb. Re-measured 2026-08-15 at
`7bf200ef` / `5911c7a0`, same fixture: `gh api --hostname evil.example.com -X PUT
repos/jpansarasa/ATLAS/pulls/99901/merge` -> **allow / deny**. The `curl` spelling is untouched — **allow / allow** —
because curl has no `--hostname` flag and carries the host inside the URL, which nothing on this path reads. The
owner/repo shapes are untouched too: `gh api -X PUT repos/attacker/evil/pulls/99901/merge` -> allow / allow and the
`curl` equivalent -> allow / allow, with the unreviewed #99904 denying on both, which is what still proves the number
IS read and the marker consulted is this repo's. So the entry NARROWS rather than closes: what remains is every REST
shape whose redirect lives in the PATH rather than in a flag. Closing that is the same fixture job costed below.
The subcommand route at the tip refuses five equivalents — `gh pr merge https://github.com/attacker/evil/pull/99901`,
`-R attacker/evil`, `--repo=attacker/evil`, a pre-verb `gh -R attacker/evil pr merge`, and `--hostname
evil.example.com` -> **deny**, reason "names a repository other than the one this checkout tracks" — where the base
allowed four of the five (the URL positional denied there too, but on the unrelated "cannot read" cause) and
`7a25d23e` allowed all five. The inconsistency is the risk: a merge refused as foreign on one spelling succeeds when
written as the other.
A SIXTH equivalent used to ALLOW, one space from a denied one, and is CLOSED as of this PR — recorded because the
measurement is what keeps the remaining REST hole re-checkable, not as open work. `merge_scan_redirects`
(`git-push-guard.sh`) enumerated `-R=`, `--repo=`, `--hostname=` and the spaced forms but not gh's ATTACHED
shorthand `-Rowner/repo`, which the identity loop then discarded as an unknown boolean flag. Measured 2026-08-15 at
tip `4f4a35d7`, same fixture as above: `gh pr merge 99901 --squash -R attacker/evil` -> **deny** but
`gh pr merge 99901 --squash -Rattacker/evil` -> **allow**, as did `-R"attacker/evil"` (shell_split delivers it in the
same shape) and the host-qualified `-Rgithub.evil.com/jpansarasa/ATLAS`. The PRE-VERB placement leaked too —
`gh -Rattacker/evil pr merge 99901 --squash` -> **allow** against a spaced pre-verb **deny** — so the gap was in both
placements, not only after the verb. gh really does bind the attached value in both: `gh pr list -Rfoo/bar --help`
exits 0, an unknown attached short flag `-Zfoo/bar` exits 1, and `gh -Rfoo/bar pr list --help` exits 0 while a bogus
pre-verb `-Q` exits 1. A generic `-R?*` arm closed all four, per the guard-change ETHOS "gate the ACT, not a
spelling"; the four now DENY and the four same-repo decoys still ALLOW. Rows landed as
`run-pr-verdict-smoke.sh` 56h-56n; deleting the arm moves **10** assertions (56h-56l, 58c, 58m, 59b, 60b, 60h). It
was written as 5, and was 8 by the time the sentence was next read: every later block added its own attached-`-R` row
and none of them re-measured this number. Re-measure it whenever an attached-`-R` row lands — a count quoted in one
file and grown in another drifts on commits that never touch the sentence.
A SEVENTH equivalent — a redirect PAST the span's cut — is also CLOSED as of this PR, and recorded for the same
reason. The span stops at the first `|`, `;`, `&` or newline and the other scan read only the PRE-verb prefix, so a
post-verb `-R` beyond the cut was in neither text; the truncation refusal that would have caught it fires only when
NO PR was named, which made naming the PR the way to switch the check off. Measured 2026-08-15 at `7bf200ef` /
`5911c7a0`, same fixture: `gh pr merge 99901 --squash --subject "a|b" -R attacker/evil` -> **allow / deny**, and 17
shapes in total (every redirect spelling past every cut character, plus the REST route) -> **17 allow / 0 allow**,
with 14 legitimate-merge rows allowing throughout. 13 of the 17 allowed at base `64cfe334` as well; the other four
denied there for reasons unrelated to the redirect, and RE-DERIVED 2026-08-15 against main `67396749` that mechanism
splits two ways rather than one: `2>&1 -R attacker/evil` (row 58n) denies with "names more than one PR number", the
"2>-is-a-PR-number" false positive `8f547bbe` removed — but rows 58f, 58g and 58k deny with "names its PR with
something this gate cannot read", the quote-stripping unreadable-token rule, a DIFFERENT removed check. The count of
four was right; the single mechanism was wrong for three of them. Only 58n is therefore a REGRESSION this branch
introduced and this PR closes: `&>/tmp/out.log -R attacker/evil` (row 58o) measures **allow on main**, so it was
always this hole and was mislabelled a regression in both this entry and the suite. Rows `58-58q`; deleting the scan
moves **40** assertions (28 at `e45ecaf5` before rows `60-60u` joined it, 20 before `59-59g`, 18 before `58y-58z`).
**That seventh fix OVER-CORRECTED, and this PR closes that too.** The scan tokenized the ENTIRE command line, so an
`-R`-shaped token belonging to a DIFFERENT command in the same invocation was attributed to the merge. Measured
2026-08-15, same fixture, nine shapes ALLOW at `7bf200ef` and on main `67396749` and DENY at `0b28bdc0`:
`&& grep -Rn "TODO" src/`, `&& cp -R build /tmp/out`, `&& ls -R docs`, `&& chmod -R 755 scripts`,
`&& sort -R list.txt`, `&& rsync -R a b`, the same before the verb, `&& gh pr list --repo jpansarasa/other`, and
`&& echo "-R attacker/evil"`. The refusal read "Names another repository: -Rn" — a flag the merge does not carry,
with a remedy (run it from a checkout of that repository) that cannot apply to a grep. `merge_bound_to_act` cuts the
token list at STANDALONE control-operator tokens and keeps the segment holding the act; all 17 post-cut rows survive
because in each the cut character is quoted or attached to a word. Rows `58v-58z`; deleting the bounding moved 4 when
those rows were written, **7** on the tree that closed the substitution hole below (`59h`, `59i`, `59j` join them) and
**11** once the depth fix added `60p`, `60q`, `60s`, `60t`.
**TWO SHAPES OF THAT OVER-CORRECTION WERE STILL OPEN, NOT ONE, AND `|&` IS NOW CLOSED.** Bash has a control operator
the cut set was missing: `gh pr merge 99901 --squash |& grep -Rn TODO src/` -> **deny** at `26dcbadf`, reason "Names
another repository: -Rn", while the same command spelled `|` allowed and both allow on main `67396749`. The set had
been written by enumerating operators from memory, so the tenth spelling of the nine shapes above shipped refused.
`'|&'` added to the case in `merge_bound_to_act`; row `59j`, and dropping it from the case turns that row red.
THE REMAINING SHAPE IS AN UNQUOTED NEWLINE separating the two commands. A `gh pr merge
99901 --squash` followed by a newline and `grep -Rn TODO src/` -> **deny** at `0b28bdc0` AND after this fix, reason
"Names another repository: -Rn". `shell_split` treats a newline as an ordinary character, so `--squash<NL>grep` is
ONE token and no standalone operator exists to cut at. Deliberately not fixed: quotes are consumed during
tokenization, so a token carrying an unquoted newline cannot be told from one carrying a QUOTED newline (row 58k),
and cutting there would drop tokens from the merge's OWN segment — a fail-OPEN, the wrong direction to trade for a
wrong-cause deny. Re-check by putting that one command through the hook.

**THE NARROWING THAT FIXED THE OVER-CORRECTION REOPENED THE ATTACK, and this PR closes that too.** A control operator
INSIDE a command substitution is a standalone token to `shell_split` and a command boundary to nothing: the `;` in
`$(mktemp ; true)` separates two commands inside the substitution and separates nothing in the line holding it.
`merge_bound_to_act` cut there anyway, so the merge's OWN `-R` fell into a segment carrying no merge and was never
scanned. Measured 2026-08-15, same fixture (#99901 approved at head, `gh` stubbed, isolated `ATLAS_MARKER_DIR`,
`CLAUDE_PROJECT_DIR` at an empty directory, nothing merged): **twelve** shapes ALLOW at `7bf200ef`, on main
`67396749` and at `26dcbadf`, and DENY at `0b28bdc0` where the scan was still unbounded — the narrowing lost exactly
what the wide scan had. `gh pr merge 99901 --body-file $(mktemp ; true) -R attacker/evil` is the shortest of them;
the axes are the four `$(…)` operators (`;`, `&`, `|`, `&&`, plus `||`), the backtick spelling, both carriers of a
substituted value (`--body-file`, `-t`) and all five redirect spellings.
Fix (SUPERSEDED by the depth-desync entry two paragraphs down, which replaced the `$(` counter with a `(` counter and
added an escape arm — read both): `shell_split` records a per-token substitution depth (`SPLIT_DEPTH`, a `$(` counter plus a backtick toggle, read
at the token's FIRST character) and `merge_bound_to_act` cuts only at depth 0. Rows `59-59g`; deleting either the
depth test or the tokenizer's appends moves 8 at `e45ecaf5` and **20** once rows `60-60l` joined them. The two
mutations measure the SAME number because an unset `SPLIT_DEPTH` element reads as 0 — a genuine equivalence,
re-measured rather than carried forward.
**The cheaper fix was measured and DECLINED, and rows `59h`/`59i` are what keep it declined.** Abandoning the
narrowing whenever ANY token carries a substitution closes all twelve and re-breaks `&& grep -Rn TODO $(pwd)` and
`&& cp -R $(pwd)/docs /tmp/x` — ordinary chained commands whose substitution belongs to the CHAINED command, with the
same wrong-cause "Names another repository: -Rn". Measured: under that fix `59-59g` pass and `59h`/`59i` go red.

**THAT DEPTH COUNTER WAS A PARTIAL MODEL OF BASH'S NESTING AND DESYNCHRONISED IN BOTH DIRECTIONS — CLOSED here.** It
rose only on the two-character `$(`, so `<(`, `>(` and a plain `( … )` raised nothing; and it fell on every unquoted
`)`, including one closing an arithmetic `$(( … ))` or sitting behind a backslash. Either way an operator bash reads
as NESTED reported depth 0, `merge_bound_to_act` cut there, and the merge's OWN `-R` landed in a segment holding no
merge. Measured 2026-08-15, same fixture (#99901 approved at head, `gh` stubbed, isolated `ATLAS_MARKER_DIR`,
`CLAUDE_PROJECT_DIR` at an empty directory, nothing merged): the three shortest are
`gh pr merge 99901 --squash 2> >(cat ; true) -R attacker/evil`,
`… --body-file $(mktemp --suffix=$((1)) ; true) -R attacker/evil` and
`… --body-file $(mktemp --suffix=x\) ; true) -R attacker/evil` — **all three ALLOW at `e45ecaf5` and DENY at
`0b28bdc0`**, where the scan was still unbounded. Across the four operators (`;`, `&&`, `|`, `&`) and all five
redirect spellings, **27 of 29 shapes measured ALLOW** at `e45ecaf5`; the other 2 denied through the
unreadable-token rule, not the redirect, so they are not clean rows and are not asserted as such.
**It defeated the REST route too**, which is row 58p's protection:
`gh api -X PUT repos/o/r/pulls/99901/merge --input $(mktemp --suffix=$((1)) ; true) --hostname evil.example.com` and
four siblings all **ALLOW** at `e45ecaf5`. The oracle is bash itself — a stub `gh` dumping its argv receives
`[-R] [attacker/evil]` in every one of them.
Fix: count EVERY unquoted, unescaped `(` as an opener (so `<(`, `>(`, `$((` and a plain subshell balance under one
rule) and read a backslash as making the next character literal, rather than enumerating a fourth substitution
spelling. Rows `60-60u`; the `(` line moves **9** assertions and the escape arm **6**, measured separately, with
`60m-60o` and `60u` the controls that keep the hits from passing on their syntax rather than on the repository.
TWO MORE CONSTRUCTS LOWER THAT COUNTER WHERE BASH DOES NOT NEST, AND ARE DELIBERATELY NOT FIXED — recorded because
the comment at the counter claimed counting every `(` "balances all four constructs under ONE rule" and that
enumerating a fourth spelling "would only have waited for a fifth". These are the fifth and sixth, and the comment
is corrected in the same PR. A `${x:-)}` default value and a `case` pattern `y)` each carry a `)` bash does not
nest, so each falls a depth it never raised; the `)` is also a token boundary here, so `${x:-)}` splits into
`${x:-` and `}` where bash produces one word (measured 2026-08-16, same fixture as the entries above — #99901
approved at head, `gh` stubbed, isolated `ATLAS_MARKER_DIR`, `CLAUDE_PROJECT_DIR` at an empty directory, nothing
merged). CAPABILITY DELTA AGAINST MAIN IS ZERO on the subcommand route, which is why this is a record and not a
fix: put either construct inside a real `$( … )` so the underflow can actually cut a span early, and all four
shapes — `$(mktemp --suffix=${x:-)} ; true)` and `$(case $v in y) true;; esac ; true)` carrying `-R attacker/evil`,
`--repo=`, an attached `-R` and `--hostname` — measure **deny / deny / deny** at `7c1261e1`, after the ANSI-C fix
and on main `67396749` alike, every one through MERGE_OPAQUE ("names its PR with something this gate cannot read"):
the same split that loses the depth leaves loose tokens no identity rule can read, and an unreadable token is
already a refusal. Their no-redirect controls deny too, so these rows would be denying for their syntax and are NOT
clean redirect rows. The only route that would be exposed is REST, whose own `repos/<o>/<r>/pulls/…` path is read
for the NUMBER and never for owner or repo on ANY tree — re-measured here at `7c1261e1`, after the fix and on main:
`gh api -X PUT repos/attacker/evil/pulls/99901/merge` -> **allow / allow / allow**, and with `--hostname github.com`
appended -> allow / allow / allow, while the same call for the unreviewed #99904 denies on all three. That is the
pre-existing REST gap recorded above, not a new one. Re-check: put the four `$( … )` shapes through the hook and
read the refusal CAUSE, not just the decision — a deny through MERGE_OPAQUE is not the redirect rule firing.
**A SECOND, OLDER HOLE FELL OUT OF THE SAME ESCAPE ARM.** An escaped `"` inside a double-quoted value toggled the
quote state back OPEN, so the whole remainder of the line — the redirect included — was delivered as ONE quoted word
that no scan reads: `gh pr merge 99901 --squash --subject "a\"b" -R attacker/evil` -> **allow on main `67396749`, at
`0b28bdc0` and at `e45ecaf5`**, while gh really receives `[--subject] [a"b] [-R] [attacker/evil]`. Not this branch's
regression; closed here because the escape arm is what fixes it. Row `60k`.
**AND THE ESCAPED-BACKTICK OVER-CORRECTION IT INTRODUCED, one commit old, is undone.** The backtick state is a
TOGGLE, so a single escaped backtick left every later token reporting depth 1 and no boundary was ever found:
`gh pr merge 99901 --squash --subject a\`b && grep -Rn TODO src/` -> **deny** at `e45ecaf5` with the wrong-cause
"Names another repository: -Rn", the eleventh spelling of the nine over-corrections above. It measures **allow on
main `67396749` and at `26dcbadf`** — the toggle arrived with `e17bb4e1`, so `26dcbadf` predates it and the
regression is exactly one commit wide. Rows `60s`/`60t`.
**AND `$'…'` DEFEATED THE SAME ESCAPE ARM FROM THE OTHER SIDE — the gate approved one PR while `gh` merged another.
CLOSED here.** The arm is skipped inside single quotes because bash has no escapes there, which is true of `'…'` and
false of ANSI-C `$'…'`, where bash DOES honour `\'`. Reading them alike closed the quote at the escaped quote and
reopened it at the string's real terminator, so the POSITIONAL was swallowed into the value:
`gh pr merge --squash --subject $'a\'b' 99904` tokenized to `<--subject> <$a\b 99904>`. Nothing had been CUT (the
span holds no `|;&`, so that refusal cannot fire) and no token was left unreadable, so control reached the
`gh pr view` fallback and resolved the CURRENT BRANCH's PR — stderr reading "✓ PR #99901 … approved" while a stubbed
`gh` received `[99904]`. Measured 2026-08-16, same fixture: **ALLOW** at `7c1261e1`, `0b28bdc0`, `26dcbadf`,
`e45ecaf5` and `8f547bbe`, **DENY** at `54e0fac5`/`3bad23ad` and on main `67396749`. Bisected: `8f547bbe` opened it,
the commit that replaced main's whole-word-integer scrape with the positional tokenizer, so main refuses it only by
accident of the scrape that commit removed — and merging this PR takes a shape main blocks. Four spellings behaved
identically: that one, the same with `--delete-branch` appended, `-t $'x\'y' 99904`, and the same with a quoted
`"#99904"`. Fix: an `_ansi` flag set from the character PRECEDING the opening quote, cleared on every open and
close, admitting the escape arm inside `$'…'` only. Rows `61-61i`; reverting the flag moves **8** assertions
(61, 61a-61d, 61g, 61h flip to allow, 61f to deny) with the two controls `61e`/`61i` staying green.
The CHEAPER route — deny whenever the tokenizer ends inside a quote — was measured and DECLINED. Re-derived here by
replaying the guard's own span cut and quote tracking over each row: on the PRE-FIX tokenizer it does catch `61` and
`61b`, but it ALSO ends mid-quote on `57k`, `58s` and `58u`, three ALLOW rows whose spans are cut mid-quote BY
DESIGN — a `--subject "a|b"` is cut at the `|` inside the quotes — so the rule cannot tell them from the hole.
`57j`, whose span is cut at a REAL pipe, terminates cleanly and is the control showing the flag tracks the quote
rather than the cut.
TWO `-R`-hiding siblings closed with it (`61g`/`61h`), both **allow on main** too, so that half is a net improvement
over main. And ONE ORDINARY COMMAND that head AND main both refuse today starts allowing:
`gh pr merge 99901 --squash --subject $'it\'s fine' && grep -Rn TODO src/` -> **deny / deny** before, allow after —
the twelfth spelling of the nine over-corrections above, pinned by row `61f` so the direction cannot regress.
THE UNQUOTED-NEWLINE over-correction recorded above is UNCHANGED by this fix, re-measured here: that command still
**denies** at `0b28bdc0`, `26dcbadf`, `e45ecaf5` and after the fix, and **allows** on main. The escape arm does not
touch it, which is the intent — reversing it would fail OPEN.

**`ansible-gate-guard.sh` reports a write to a path it invented — sibling guard, NOT fixed here.** Observed live
2026-08-15 four times in one session: `git show <rev>:.claude/hooks/git-push-guard.sh > /tmp/<scratch>/guard.sh`
drew "This command writes to deployment/CI gate file(s)
(`/home/james/ATLAS/.claude/worktrees/agent-<id>/67396749:.claude/hooks/git-push-guard.sh`)". The guard took the
`git show` REVISION SPEC for a path and resolved it against the repo root, naming a file that does not exist and was
never written; the actual redirect target was in `/tmp` and touched no gate file at all. A second sighting of the same
class was reported separately, an ABSOLUTE `$C/...` argument resolved as repo-relative — so the defect is the path
CONSTRUCTION, not the matching, and one fix covers both. Cost is a false positive that consumes a bypass
justification: the operator is told a gate file is being
written, so the bypass looks required when nothing gated was touched. Re-check by running any
`git show <rev>:<tracked-path>` redirect to a `/tmp` file and reading the path the guard names.
TWO FURTHER SPELLINGS, measured 2026-08-15 with `CLAUDE_PROJECT_DIR` at an EMPTY directory so the live bypass is out
of the measurement. Both DENY.
(1) An ABSOLUTE `/tmp/...` path is matched as though it were repo-relative, which BLOCKS a copy with **both ends in
`/tmp`**: `cp /tmp/claude-1000/scratch/sandbox/.claude/hooks/git-push-guard.sh /tmp/claude-1000/scratch/probe/copy.sh`
-> **deny**, naming `/tmp/claude-1000/scratch/sandbox/.claude/hooks/git-push-guard.sh` — the cp's SOURCE, under
`/tmp`, in the repo's gate layer only by substring. Nothing in the repo is read or written. This is worse than the
`git show` sighting: that one only wasted a bypass justification, this one refuses work that touches no gate file at
all, and the only remedy the message offers is a bypass.
(2) RUNNING a gate-path script is reported as WRITING to it whenever a PREFIX WORD displaces the runner — which the
guard's own deny message contradicts ("RUNNING one of these files is NOT blocked — only writing to it … redirecting
its output to a log is fine"). `bash .claude/hooks/test/run-pr-verdict-smoke.sh > /tmp/out.log` -> **allow**, but
`time bash …` -> **deny**, naming the SCRIPT, not the redirect target. Nine prefixes measured deny — `time`, `nice`,
`timeout 60`, `stdbuf -oL`, `command`, `exec`, `ionice`, `nohup`, `xargs` — and only `env` still allows, so the
exemption is keyed to the runner being the segment's FIRST word rather than to the act. The redirect is load-bearing:
`time bash <script>` with no `>` allows, and so does `time bash <script> 2>&1 | tail -3`.
Re-check: run each of the ten prefixes above with `> /tmp/out.log` and read the path the guard names.
(3) A THIRD spelling, and the worst of the three because it makes the guard's OWN scoping mechanism unreachable:
the gate reads gate-layer paths out of a command's CONTENT, so WRITING A SCOPE into `.ansible-gate-confirmed` is
refused by the gate the scope exists to narrow. Observed live 2026-08-16:
`printf '%s\n' '.claude/hooks/git-push-guard.sh' … > /home/james/ATLAS/.claude/.ansible-gate-confirmed` -> **deny**,
naming `.claude/hooks/git-push-guard.sh` — a path that is the DATA being written, never the target; the target is
`.ansible-gate-confirmed`, which `ansible-gate-guard.sh` deliberately does not gate. The documented spelling at its
own comment (`printf '%s\n' .claude/hooks/ansible-gate-guard.sh > .claude/.ansible-gate-confirmed`) therefore
cannot be run. Only the all-or-nothing `touch` survives, which is the widest bypass — the exact failure the 2026-08-07
scoping change was added to prevent, since an unreadable or empty scope authorises the entire gate layer. Workaround
used: write the fragments to a file elsewhere and `cp` it in, so no gate path appears in the command. Same root as
(1) and (2) — path CONSTRUCTION over command text rather than over the resolved write target — so one fix covers all
three. Re-check: try the documented `printf … > .claude/.ansible-gate-confirmed` and read the path the guard names.
A FOURTH sighting of the `git show <rev>:<path>` spelling recorded above also landed 2026-08-16, on a worktree-scoped
agent, so that one is not rare.
(5) A FIFTH spelling, measured 2026-08-17 against the merged #974 guard (`ee1a957c`) — NOT a regression of it, and
the first of the five to refuse a plainly READ-ONLY command. The read-verb exemption is keyed to the verb being the
segment's FIRST WORD, so anything that displaces it — a group opener `(` or `{`, or one of the prefix words already
recorded at (2) — lets a stdout redirect anywhere in that segment turn the command's gate-path OPERAND into the
reported write target. TWELVE shapes DENY: `(grep -n marker git-push-guard.sh > /tmp/o.txt)` and the same read with
`cat`, `wc -l`, `head -20`, `sed -n`, `awk`; a two-`grep` batch; a `{ …; }` brace group; a `time`-prefixed and a
`nice`-prefixed `grep`; and the `.claude/hooks/…` and absolute spellings of the operand. NINE of twelve controls ALLOW — the same reads
with the group opener removed, with the redirect removed, piped instead of redirected, sent to `2>` instead of
stdout, or reading a non-gate file. (The three controls that still deny are a different shape: `sed` and `awk` are
write-capable verbs at segment head too, and `>>` behaves as the twelve do.) With a BARE BASENAME the invented path
is repo-root-relative — `/home/james/ATLAS/git-push-guard.sh`, which DOES NOT EXIST, the file being
`.claude/hooks/git-push-guard.sh` — so a command merely MENTIONING a gate file's basename is refused as a write to a
path that is not there, while nothing outside `/tmp` is read or written. Cost measured on this round: the harness
mirror had to be re-sited twice and every probe run from a file, because naming a gate path on the command line is
itself refused. Same root as (1)-(4) — path CONSTRUCTION over command text rather than over the resolved write
target — so one fix still covers all five. Re-check, from the repo root; the command is itself not refused, and the
trailing pipe keeps it valid when copied across the line break:
`jq -n '{tool_input:{command:"(grep -n x git-push-guard.sh > /tmp/o)"}}' | bash .claude/hooks/ansible-gate-guard.sh |
jq -r .hookSpecificOutput.permissionDecision` prints `deny` today. Fixed, it prints NOTHING — an allowing guard
emits no JSON at all, so there is no `permissionDecision` to read and jq exits 0 with empty output.

**A nested `sh -c` hides the redirect from every scan, and this is the one placement still open.** The outer
tokenizer delivers a quoted `-c` argument as ONE token, so `-R` is never a token to test; the span-based scan would
still catch it, but only while the span reaches that far. Put a cut character INSIDE the `-c` string and neither
sees it. Measured 2026-08-15 at `7bf200ef` / `5911c7a0`, fixture as above (#99901 approved at head, `gh` stubbed,
isolated `ATLAS_MARKER_DIR`, nothing merged): all five spellings -> **allow / allow** —
`sh -c "gh pr merge 99901 --squash --subject 'a|b' -R attacker/evil"`, the same with `'a;b'`, `bash -c` with `'a&b'`
and `--repo=`, the quote-swapped `sh -c 'gh pr merge 99901 --squash --subject "a|b" -R attacker/evil'`, and the
`--hostname evil.example.com` spelling. The control that keeps this re-checkable:
`sh -c "gh pr merge 99901 --squash -R attacker/evil"` — no cut inside the string — -> **deny / deny**, so it is the
CUT and not the nesting that does the hiding. Deliberately not fixed here: reaching inside `-c` means tokenizing a
nested shell, which is a different job from scanning one more text, and the round that measured it was scoped to the
placement. Re-check by running those six commands through the hook and reading the decisions.
**A number PIPED into the merge is read as no number at all — pre-existing, and documented nowhere until now.**
`echo 99904 | xargs gh pr merge --squash` -> **allow** on main `67396749`, at `0b28bdc0`, and after this PR's fix
(measured 2026-08-15, same fixture: #99901 approved at head, #99904 carrying no verdict, `gh` stubbed, isolated
`ATLAS_MARKER_DIR`, nothing merged). The span holds NO number and nothing is CUT, so neither the unreadable-positional
rule nor the truncation refusal has anything to fire on; the gate falls through to the `gh pr view` fallback and
answers with the CURRENT BRANCH's approved PR while `xargs` hands gh the piped one. Same approve-one-merge-another
shape the comment and `$N` rules close, arriving by a route that leaves the span EMPTY rather than unreadable — which
is why the fallback's "provably names NO identity" precondition holds and lets it run. Re-check with that one command.
**A test currently encodes the hole as expected behaviour.** `run-pr-verdict-smoke.sh:533` row 2 asserts ALLOW for
`gh api -X PUT repos/o/r/pulls/$PR_A/merge`, and `o/r` is not this checkout. Re-derive:
`bash .claude/hooks/test/run-pr-verdict-smoke.sh` -> rc 0, 164 PASS / 0 FAIL (104 when this was first
measured, then 126, then 148, then 153; the merge-gate coverage rows added 22, the cut-redirect rows a further 22,
the over-correction decoys 5 and the substitution-depth rows 11, none of them touching this row), including
`PASS: 2. single REST merge, approved at head -> allow` (2026-08-15). That is the same shape as the row this branch
flipped to a deny (`row allow "3. DECOY gh -R org2/repo7 …"`, `:473` in the base file): the subcommand spelling was
corrected and the REST spelling was left.
Why it was deferred, and the size of the deferred job: `repos/o/r/pulls` appears on 17 fixture lines across two
suites — 7 in `run-pr-verdict-smoke.sh` (`:534`, `:547`, `:549`, `:559`, `:574`, `:583`, `:909`) and 10 in
`run-entry-shape-smoke.sh`. The last of the seven is row 58p, added by this PR, and it needs no re-pointing: it
already DENIES, on the `--hostname` flag rather than on the path. None of those slugs names this checkout, so the
rule cannot bind on the REST route
without re-deciding every row that carries one: row 2 flips to deny outright, and the chained / no-verdict deny rows
keep denying on the NEW rule, so they stop exercising the property they were written for. The fixtures must be
re-pointed at this checkout's slug in the same change as the rule, or the suite goes green while testing nothing.

**An attempted BLOCK that is refused leaves a prior APPROVE standing, and the merge gate honours it.** Narrow, and
NOT the thing it first looks like. THE MOVED-HEAD CASE IS ALREADY COVERED — do not re-open it:
`.claude/hooks/git-push-guard.sh:2667` compares the marker's sha against the PR's live `headRefOid` and calls
`deny` (`:770`, which emits a deny decision and exits) when they differ, and between the marker read at `:2617` and
that comparison every branch is a deny — no marker, unreadable verdict, `blocked` verdict, unreadable head. The one
`allow` sits after the comparison passes. Hook wired at `.claude/settings.json:87`. So a verdict recorded against a
superseded head cannot unblock anything, and a read-side fix is already shipped.
What IS reachable is a write-side gap. `scripts/claude-pr-verdict` calls `warn_surviving_marker` (`:147`) before
every refusal, leaving any earlier `pr-reviewed-<N>` on disk. When the refusal is the head-mismatch one (`:195`)
that is harmless: the surviving marker cannot match the current head either, so the guard denies. But the other
refusals — missing pending record (`:166`), malformed pending (`:179`), unreadable `gh` (`:192`), invoke-then-stamp
too fast (`:220`, the `MIN_REVIEW_SECONDS` guard, NOT a reason-length check) — can fire while a prior approve sits
at the CURRENT head. That approve stays valid, the guard correctly honours it, and the merge proceeds even though
the reviewer's last action was an attempted BLOCK. The auto-unlink is declined on purpose (silently deleting a prior
verdict on an unrelated refusal destroys a legitimate record), so the fix is to invalidate or DOWNGRADE the prior
approve when a block is attempted and refused — write side, not read side. Related in shape only to the historical
defect where `review-pr` wrote a passing marker at INVOCATION.
**THE SCRIPT'S OWN WARNING IS WHERE THIS DEFECT GETS MIS-DIAGNOSED, and it is armed right now.**
`warn_surviving_marker` reads only `v verdict rest` off the marker and branches on `verdict == "approved"`; the
recorded sha sits unparsed in `rest` and is never compared to anything. So its message body — lines 155-156 of
`scripts/claude-pr-verdict`, written here in non-citation form for the reason the entry below gives — prints
"an APPROVED verdict ... is still on disk and still unblocks the merge" UNCONDITIONALLY, including after the
moved-head refusal where `git-push-guard.sh:2667` will in fact deny. The function's own header comment, lines
144-145, states the same thing as fact. That sentence is how this entry came to assert the opposite of the code in its first
revision — a reviewer runs the script, reads the warning, and believes it. Fix: compare the marker sha to the head
and stay silent when they differ, or weaken the wording to "may still unblock the merge — check the head".
Scope note: this is the MERGE gate (`pr-reviewed-<N>`) alone, ON THE BASH PATH. The PUSH gate is a different gate
keyed on a TREE hash and never reads this marker. And "the read-side fix already ships" must not be read as "merges
are gated": `.claude/hooks/git-push-guard.sh:128-145` records a KNOWN GAP, dated 2026-08-06 and deliberately left
open, that the hook is wired at PreToolUse with matcher `Bash`; PreToolUse DOES fire for MCP tools, but matchers are
compared exactly, so `Bash` matches no MCP tool name, and the comment states no hooks block in any settings file
carries an `mcp__` matcher. It names three tools that reach gated outcomes with no marker consulted at all —
`mcp__plugin_github_github__merge_pull_request` (merges with no verdict marker), and `push_files` and
`create_or_update_file` (write main with no PR and no marker) — all of which sit in a dispatched agent's tool set.
The recorded fix shape is a second PreToolUse entry matching `mcp__.*`, routed to a sibling hook that reads the
structured `.tool_input` rather than `.tool_input.command`. So the verdict gate holds for `gh pr merge` run through
Bash and is ABSENT on the MCP merge path.
**BUT THAT ENTRY'S STATED BLOCKER IS DISCHARGED, AND ITS FIX SHAPE DOES NOT COVER MERGE.** The "deny may not bind"
rationale is a 2026-08-06 record, and the guard closes it with "Confirm the deny binds empirically, then build" —
which was done the NEXT DAY. Measured 2026-08-07 on this host (Claude Code 2.1.224, isolated `claude -p` runs
against a purpose-built probe MCP server, nothing live mutated): PreToolUse matchers DO fire for MCP tools and
**deny BINDS** — `server_calls=0`, proven by the probe server's own log showing non-execution rather than by a
transcript claim. The matcher is an unanchored REGEX when it contains a metacharacter and an exact full-name
comparison otherwise, which is why a plain `mcp__plugin_github_github` matcher ships and gates NOTHING silently.
The residual is narrow: confirmation against the LIVE `plugin_github_github` server, which needs one log-only
`mcp__.*` hook in live settings plus one read-only call.
The real obstacle is elsewhere, and it is why this must not be built by reflex: **the recorded fix shape misses the
very tool the entry exists for.** A sibling hook reading `owner/repo/pullNumber/branch` cannot gate
`merge_pull_request`, whose input carries only `owner`, `repo`, `pullNumber` (plus optional commit title/message and
merge method) and **no `branch` at all** — verified against the live tool schema. A branch-reading gate would cover
`push_files` and `create_or_update_file` and silently no-op on merge. Gating merge needs `pullNumber -> base.ref`
resolved INSIDE the hook's ~5s budget, or a verdict marker keyed by PR number rather than by branch.
Both repo artifacts still carry the superseded rationale and neither is touched here (gate-layer, outside this PR's
blast radius): `.claude/hooks/README.md:404-406` says the fix is unbuilt because deny-binds "has not been exercised
on this host", while `.claude/hooks/README.md:377-380` — twenty-six lines earlier in the SAME document — already
says the regex matcher "works" and notes it caught its own earlier wrong revision; `git-push-guard.sh:141-145`
carries the same superseded reason. Re-check for that pair: both must either cite the 2026-08-07 measurement or
state the `branch`-field obstacle; today neither does. Re-check, static and safe — it needs no PR and writes no marker:
`grep -n warn_surviving_marker scripts/claude-pr-verdict` must list refusal sites OTHER than the head-mismatch one
at `:195` (those are the reachable ones), AND `git-push-guard.sh:2667` must still compare `MARKER_COMMIT` against
`PR_HEAD_COMMIT`. If the second ever stops being true the moved-head case re-opens and this entry is wrong.

**`sudo` and `env` set `PFX_SKIP` even when the wrapper span is ABANDONED.** `ansible-gate-guard.sh:prefix_span`
assigns `PFX_SKIP=$n` from the `sudo`/`env`/`VAR=` arm before it is known whether the span will be accepted, and the
abandonment line at the end clears only `PFX_CHECK`. An abandoned span therefore still tells the token walk to skip
tokens, and the executable slot — the one slot exempt from being read as a write target — lands on the wrong word.
Same root as the wrapper-allowlist regression closed in #974, and the allowlist does NOT close it: the `sudo` arm
sits above the wrapper branch and never consults `WRAPPER_RE`. Measured 2026-08-17:
`nice -n /opt/ai-inference/compose.yaml sudo rm /tmp/x` ALLOWS both before and after the #974 fix, and DENIES at
ddbaff89. No spelling of it that actually writes a guarded path has been constructed, which is why it is recorded
rather than fixed — the desynchronised walk is real, the reachable act is not yet. Re-check: feed that command
string to the guard as INPUT (never execute it); it must deny.

**A write verb ABUTTING an opening quote is never walked at all.** `WRITE_RE` anchors every verb on
`(^|[[:space:]])`, so a verb sitting against a quote character matches nothing, the segment is skipped before any
operand rule runs, and no amount of widening the operand classes can reach it. Measured 2026-08-17, ALLOW at
ddbaff89 AND after the #974 fix: `printf '%s' 'rm .claude/settings.local.json' | bash`,
`echo 'rm .claude/settings.local.json' | bash`, `bash -c 'rm .claude/settings.local.json'` and the double-quoted
spelling of the last. The space-anchored counterparts DENY after the fix
(`printf '%s\n' rm .claude/settings.local.json | bash`, `echo rm .claude/settings.local.json | bash`), which is what
pins the cause to the anchor rather than to the pipe. `bash -c` deserves its own line: it has no unquoted spelling
at all, so that shape is unwalked unconditionally and cannot be repaired by respelling a fixture. This bit #974's
test authoring twice — a new row passed for this reason and only the mutation battery exposed it — so it costs
test-authoring time, not merely coverage. Re-check: feed both spellings of any one pair to the guard as input; they
must agree.

**A write-capable command outside `_WRITE_VERBS` is not walked unless something else in the segment matches, and
closing it needs a DECISION rather than an edit.** Inherited, and named in the guard's own comment as a known gap;
#974 measured the pipeline shape of it. Measured 2026-08-17, ALLOW at ddbaff89 AND after the fix:
`echo shred /opt/ai-inference/compose.yaml | bash` and `echo .claude/settings.local.json | xargs rm` — neither
upstream segment carries a verb `WRITE_RE` knows, so no operand rule ever runs on it. The same acts spelled with a
known verb DENY after the fix (`echo rm -f /opt/ai-inference/compose.yaml | sh`,
`echo rm .claude/settings.local.json | xargs -n1 rm`). Closing it means widening `_WRITE_VERBS` or widening the
walk gate, and the walk gate was measured to cost real ordinary work: `ls .claude/hooks | awk '{print}'` and
`find .claude/hooks | xargs grep x` both ALLOW today and would start denying. Widening `_WRITE_VERBS` also widens
`NEVER_WRAPPER_RE`, which is coupled to it by construction, so the blast radius is not one list. Someone must
choose which cost to pay; a one-line edit here would be choosing silently. Re-check: the four command strings
above as input — the first two must deny, and the two ordinary-work rows must still allow.

**The Bash path and the Edit/Write path do NOT apply the same rules, despite the header saying they do.**
`ansible-gate-guard.sh:183-184` claims "ONE definition, consulted by the Edit/Write path AND the Bash path, so the
two can never drift apart". That is true of `is_gate_path` and `is_deployed_path` and false of `GATE_BASENAMES`,
which is read at exactly two sites (`check_token` and `prefix_span`), both on the Bash path. Measured 2026-08-17 at
ddbaff89 AND after the #974 fix, with the guard sited beside the hook set so `GATE_BASENAMES` is populated: Bash
`cp /tmp/a /tmp/scratch/ansible-gate-guard.sh` DENIES, while an Edit naming `/tmp/scratch/ansible-gate-guard.sh`
ALLOWS. The direction is safe — Bash is the stricter path — but the header overstates the coupling, and the
asymmetry is load-bearing for anyone reasoning about the bare-basename rule, which is the only rule that follows a
guard's NAME out of the gate layer. A docstring claiming coverage it does not have is the defect moved into the
tool. Re-check: the two inputs above through one guard copy sited beside the hook set; they must agree, or the
header must stop claiming they do.

**`run-wiring-smoke.sh` has been RED for nine days, and the red is the suite's own stale list.** Measured
2026-08-17 at `8d04fd33`: `bash .claude/hooks/test/run-wiring-smoke.sh` -> rc 1, 48 `PASS:` lines and exactly ONE
`FAIL:`, reading `registered set drifted:5a6 > dream-pending-notice.sh`, ending `WIRING SMOKE: FAIL`. READ THE
DIFF DIRECTION BEFORE ACTING ON IT — the failure text invites the opposite conclusion. The check diffs
`EXPECTED_WIRED` against `ACTUAL_WIRED` in that order (`:72`), so a `>` line is present in ACTUAL and missing from
EXPECTED: the hook IS registered in tracked `.claude/settings.json` (15 distinct basenames), and it is the suite's
hardcoded list (`:63-66`, 14 names, whose pass message still says "exactly the expected 14 hooks") that never
learned about it. The registration landed 2026-08-08 in #936; the list was last touched 2026-08-07 in #918, so the
suite has failed on every run since. The check is doing precisely the job its comment claims — proving nothing was
"added unnoticed" — and nobody noticed the notice.

The cost is not the one red row. The suite exits 1, so its other 48 assertions — including the marker writers must
be 100755 IN THE INDEX rows, which exist because a 100644 shipped once and broke the merge gate — sit behind a
failing summary, and anything gating on rc reads the whole suite as broken rather than as one stale line. A suite
that is permanently red teaches its readers to skip it, which is the failure mode that lets the NEXT drift through.

Pre-existing and independent of PR #974: all three inputs (`.claude/settings.json`, `run-wiring-smoke.sh`,
`dream-pending-notice.sh`) are unmodified at `8d04fd33`, and the drift check reads neither file that PR touches.
Decide the direction rather than silencing the row: either the dream notice is a wired participant and belongs in
`EXPECTED_WIRED`, or it should not be registered at all. Re-check:
`bash .claude/hooks/test/run-wiring-smoke.sh; echo rc=$?` — rc must be 0 and the summary `WIRING SMOKE: PASS`.

**A write in one tool call and its execution in the NEXT are invisible to any command-string guard. ACCEPTED LIMIT,
not an open bug — nothing in a future round can close it.** `ansible-gate-guard.sh` is a `PreToolUse` hook: it is
handed ONE `tool_input.command` and must decide on that string alone. `echo cp /tmp/evil
/opt/ai-inference/compose.yaml > /tmp/run.sh` in call 1 and `bash /tmp/run.sh` in call 2 are two strings, neither of
which contains the other's half, and no amount of parsing reaches across them. Recorded because the shape looks like
a defect to every reviewer who meets it, and three rounds of #974 were spent on designs that implicitly promised to
cover it.

WHAT COVERS IT: denying call 1. Since #974 the guard checks echo/printf operands whenever the segment's stdout lands
in a file, whatever happens to that file afterwards — which is `ddbaff89`'s rule, restored deliberately after a
narrower one was measured open. Measured 2026-08-17, all three DENY on the current guard and at `ddbaff89`, all
three ALLOWED at `6c276949`: the bare write with no run anywhere, the same write followed only by `ls -l`, and the
gate-layer spelling naming `.claude/settings.local.json`.

THAT INVARIANT IS BOUNDED BY HOW A REDIRECT IS RECOGNISED, and the bound is written down because the sentence above
overstated it for one round. `>&N` was excluded as "a descriptor is not a file" until #974 round 5. It is not: `>&N`
means stdout goes wherever descriptor N goes, and N is aimed at a file by a `3>/tmp/run.sh` in the same segment or an
`exec 3>/tmp/run.sh` before it — six spellings that DO put echo's text into a script were answering "no redirect",
and `echo cp /tmp/evil /opt/ai-inference/compose.yaml 3>/tmp/run.sh >&3` needs no `exec` at all. That exclusion is
deleted and all six deny, matching `ddbaff89`. What remains open is ONE property, measured 2026-08-17: an operator is
recognised only where it STARTS a token. So `echo cp /tmp/evil /opt/ai-inference/compose.yaml>/tmp/run.sh` — operator
welded to the preceding WORD — still ALLOWS, and so do the `exec` spellings `exec>/tmp/run.sh`, `\exec > /tmp/run.sh`
and `>| /tmp/f exec` (below).
TWO OF THE FIVE THIS ENTRY LISTED ARE NOW CLOSED and are struck rather than left standing: `> /tmp/run.sh exec`, the
operator SPACED from its target, was round 6's blocker and now DENIES, and `eval "exec > /tmp/run.sh"` denies with it —
the walk already strips one quote layer, so `"exec` reaches the test as `exec` (double-, single- and welded-target eval
spellings all deny).

`>| /tmp/f exec` IS DISCLOSED OPEN, AND ITS ROOT CAUSE IS UPSTREAM OF THE PREDICATE — which is why it is listed here
rather than fixed in the round that found it. `>|` is the noclobber-override redirect and it BINDS: measured
2026-08-17 on bash 5.2.21 against a benign target, both `>| f exec` and `>|f exec` put the following segment's echo
text in the file and leave stdout empty. It ALLOWS at this branch's head and DENIES at `ddbaff89`. It is NOT a
regression from the exec-walk rewrite — measured at `2bddf6de` as well, where it also ALLOWS — so no round of this PR
introduced it. `segment_opens_exec_redirect` never sees it: `split_segments` treats the `|` of `>|` as a PIPE and cuts
the operator in half, so `>| /tmp/f exec; echo hi` tokenises to segments `[|>]`, `[; /tmp/f exec]`, `[; echo hi]` and
the operator never reaches the predicate as one token. Note the scope is narrow — the plain `echo cp /tmp/evil
/opt/ai-inference/compose.yaml >| /tmp/run.sh` still DENIES; only the exec-binding form escapes.
NOT CLOSED, DELIBERATELY: the fix is an edit to `split_segments`, the tokenizer every other mechanism in this guard
depends on, and four of this PR's five preceding rounds introduced a FRESH defect in exactly that class of change —
a predicate rewritten to cover one shape and walked past by another. The risk of the fix exceeds the risk of the gap,
which is a deliberate-evasion spelling nobody writes by accident, so it is disclosed with its measurement instead.
Re-check (INPUT only, never execute — it writes the path it names): `>| /tmp/f exec; echo cp /tmp/evil
/opt/ai-inference/compose.yaml; bash /tmp/f` allows today and must deny if `split_segments` is ever taught `>|`.

CLOSING THE REST DOES NOT MEAN RESTORING MAIN'S RULE, and the sentence that stood here implied it did by calling the
cost "unmeasured". It is measured. `segment_redirects_to_file` can recognise an operator welded INSIDE a word (a second
arm, `[[ "$tok" =~ [^0-9<>&]\> ]]`); applied alone it leaves `run-advisory-guards-smoke.sh` at ZERO FAILURES and
flips both the welded write and `exec>/tmp/run.sh` to deny. The figure is keyed to the FAILURE COUNT, not to the row
total, because the total moves whenever a row is added and this measurement has already rotted once that way: it read
"426 rows" until the round that added a control row, and a re-check whose denominator no longer matches what a reader
counts invites the conclusion that the measurement was never taken. It buys nothing for the VERBLESS prose spelling, because
WRITE_RE's redirect arm anchors at `(^|[[:space:]])` so a welded operator never opens the walk gate; widening that arm
TOO costs exactly ONE new denial — `echo we should review /opt/ai-inference/compose.yaml>/tmp/run.sh`, which ALLOWS at
`ddbaff89` AND at this branch's head — and also moves zero suite rows. One new cost row, not blanket denial. NOT DONE
HERE: the decision to leave it open stands, and what changed is that the entry now states the price instead of pleading
ignorance of it. Re-check (INPUT only, never execute — it writes the path it names):
`echo cp /tmp/evil /opt/ai-inference/compose.yaml>/tmp/run.sh` allows today and must deny if either widening lands.

`ddbaff89` DENIES THE SPELLINGS THAT REMAIN OPEN, AND THAT IS NOT THE ENDORSEMENT IT LOOKS LIKE — recorded because the
sentence above reads as "pre-existing, nobody covers it" and the next agent would otherwise close it by reverting to
main's rule. Measured 2026-08-17, guard sited beside a full hook set so GATE_BASENAMES seeds: the welded-operator
write, `exec>/tmp/run.sh` and `\exec > /tmp/run.sh` all ALLOW here and DENY at `ddbaff89`. But `ddbaff89` also denies
`echo cp /tmp/evil /opt/ai-inference/compose.yaml` with NO redirect anywhere, and `echo we should patch
/opt/ai-inference/compose.yaml` with no write verb either, emitting the SAME reason string for all of them. It does
not read redirects on this path at all — it denies any command string naming a guarded path. So its deny on the three
is the blanket prose deny this branch removes ON PURPOSE, not coverage this branch lost, and restoring it is not the
fix: it re-instates the false denials the branch exists to remove. Live proof, not a fixture — the installed hook at
`/home/james/ATLAS/.claude/hooks/` is byte-identical to `ddbaff89` (verified with `cmp` 2026-08-17) and refused
`time bash <a gate path> > /tmp/out 2>&1`, i.e. running a guard suite and logging it, which both this branch and its
parent allow.

"MAIN DENIES, HEAD ALLOWS" IS NOT BY ITSELF A LOOSENING, and that is written down because a review of this PR read it
as one. Main denies essentially ANY command string naming a guarded path — the control in the paragraph above proves
it: no redirect, no write verb, still denied — so the comparison holds for a large class of strings and cannot
distinguish a real gap from a false denial this branch removed on purpose. "Did THIS change loosen X" has exactly one
correct baseline: the commit BEFORE the change, not main. Worked example, measured 2026-08-17 —
`1> exec; echo we should patch .claude/hooks/git-push-guard.sh` and its `2>` spelling both DENY at `ddbaff89` and
ALLOW at this branch's head, which reads as a regression and is not one: both also ALLOW at `2bddf6de`, so no round of
this PR moved them. Neither binds anything (verified on bash 5.2.21: the file named `exec` is created EMPTY and the
prose stays on stdout), so allowing them is correct rather than merely harmless. Name the pre-change commit, not main,
whenever a verdict is called a loosening.

AND "`ddbaff89` COVERS THE WELDED FORM" IS ONLY HALF TRUE, which the sentence above asserted flatly. It holds for a
`/opt`-prefixed path and fails for a gate-layer one, because `is_deployed_path` anchors `/opt/*` at the FRONT while
every gate glob anchors a SUFFIX — and the welded `>` moves the end of the token into the REDIRECT TARGET. Measured
2026-08-17 at `ddbaff89`: `echo rm -f .claude/settings.local.json>/tmp/run.sh` ALLOWS, while the same string with the
target spelled `/tmp/run.json` DENIES, `*.claude/settings*.json` matching the target's extension rather than the gate
file's. The hooks spelling splits on the same accident — `…git-push-guard.sh>/tmp/run.sh` denies via `*.claude/hooks/*.sh`
and `…git-push-guard.sh>/tmp/run.txt` allows. Main's coverage of this shape is an artefact of what the redirect target
happens to be called, so it is not a standard this branch is failing to meet.
Re-check (INPUT only, never execute — each writes the path it names): the three allow here and deny at `ddbaff89`;
`echo cp /tmp/evil /opt/ai-inference/compose.yaml` with no redirect at all must ALSO deny at `ddbaff89`, and that is
the control proving the deny is redirect-blind rather than a redirect this branch stopped seeing; and the
`/tmp/run.sh` vs `/tmp/run.json` pair must keep its SPLIT verdict at `ddbaff89`, which is what makes the coverage
claim conditional rather than general.

WHAT WOULD NOT COVER IT: any design keyed on "the target is later executed". `6c276949` carried one, and it leaked
three ways in the SAME command string before the two-call case was even reached — the target compared as raw text
while every other site resolves with `realpath -m` (7 spellings), the runner tested by an enumeration defaulting to
allow (12 spellings, including `dash`, `python3` and every wrapped form, which the PIPE route denies because it asks
the inverted question), and a redirect opened by an earlier `exec >` (4 spellings). Do not reintroduce one: the cost
of the broad rule is a single shape, `echo <prose naming a guarded path> > <file>`, pinned by its own COST fixtures
in `run-advisory-guards-smoke.sh`.

Re-check (feed as INPUT to the guard, never execute — each writes the path it names):
`echo cp /tmp/evil /opt/ai-inference/compose.yaml > /tmp/run.sh` must deny on its own, with no second segment.

**Four of the six series Sentinel is PRIMARY SOURCE for have not published since April 2026, and FOUR of the six
emit no freshness gauge — three have no pattern (`ADP_EMPLOYMENT`, `INDEED_POSTINGS`, `REDBOOK_SALES`) and
`BDIY`'s pattern file is disabled — so no overdue alarm, so a death there is silent. Two of the four already died
that way; the other two are not dead, and the two dead series that WERE noticed are the two that are gauged.**
Measured 2026-08-19 across the D-18 owned set
(`SentinelCollector/AGENT_README.md` D-18) — last publish, pattern status:
- `CHALLENGER_JOB_CUTS` **2026-04-23 05:01:12** — 3 live patterns (`challenger-layoff-surge`,
  `challenger-vs-payroll`, `sentinel-challenger-divergence`) — dead, visible.
- `TRUFLATION_CPI` **2026-04-23 04:43:57** — 1 live pattern (`truflation-vs-cpi`) — dead, visible.
- `INDEED_POSTINGS` **2026-04-16 13:35:19** and `REDBOOK_SALES` **2026-04-23 08:26:30** — no pattern — dead, invisible.
- `ADP_EMPLOYMENT` **2026-07-10 12:24:50** — no pattern — **NOT dead**: 11 rows published after 2026-05-01, 40 days
  silent at measurement. Never carry the April cohort's "four months" onto it.
- `BDIY` **2026-08-19 13:19:43**, the measurement day — pattern FILE exists carrying `"enabled": false`
  (`ThresholdEngine/config/patterns/recession/baltic-freight-recession.json:24`) — **ALIVE**, equally ungauged.
Re-derive (SELECT only): `SELECT coalesce("Symbol","OriginalSymbol") AS sym, max(published_at), count(published_at)
FROM sentinel.extracted_observations WHERE coalesce("Symbol","OriginalSymbol") IN ('ADP_EMPLOYMENT','BDIY',
'CHALLENGER_JOB_CUTS','INDEED_POSTINGS','REDBOOK_SALES','TRUFLATION_CPI') GROUP BY 1;`

**READ BEFORE SELECTING ANY POPULATION: a NULL `instrument_id` today does NOT mean resolution failed at extraction
time.** `ApplyReExtraction` (`SentinelCollector/src/Entities/ExtractedObservation.cs:247`) snapshots the prior
resolution into the `Original*` columns **only when all three are still null** (`:260-268`, preserving the EARLIEST
snapshot across repeat runs — guarded by
`SentinelCollector.UnitTests/Workers/ReExtractBackgroundServiceTests.cs:519`
`should_preserve_earliest_audit_snapshot_on_second_re_extract`), then overwrites `InstrumentId`/`Symbol` with the
new result **including NULL**. `Quarantine()` (`:220`) and `QuarantineInPlace()` (`:331`) write the same three
columns, so a quarantine can be the snapshot event a later re-extract then declines to overwrite.
Only two of the five call sites can null a resolved row:
`SentinelCollector/src/Workers/ReExtractBackgroundService.cs:487` (full re-extract) and `:663` (resolve-only, **the
leg prod runs**) pass the shim's result through, and that result may be null. The other three — `:369` (null
`RawContent`), `:438` (zero extractions), `:573` (empty `Description`) — pass `observation.InstrumentId`/
`observation.Symbol` straight back in: watermark-only stamps that cannot change a row's instrument or symbol — they
still run the full `ApplyReExtraction` body, which recomputes `ResolutionState` from the passed-back instrument
(`SentinelCollector/src/Entities/ExtractedObservation.cs:287`, flipping a Resolved-with-null-instrument row to
`NoResolution`) and can clear `QuarantinedAt` (`:304`). So a sweep turns resolved rows into `instrument_id IS NULL,
resolution_state='NoResolution'` while `published_at` still stands, but
only via `:487`/`:663`. Measured 2026-08-19: **329 of 329** Challenger rows carry `re_extracted_at` and only
**4** still hold an instrument — and those same 4 now carry a DIFFERENT `Symbol` than the key they were filed under
(ids **10714**, **10716**, **17490** = `UNRATE`, id **12528** = `BLK`; OPEN LEAD, unexplained), which is also why
`WHERE "OriginalSymbol"='CHALLENGER_JOB_CUTS'` returns **329** rows while the `coalesce("Symbol","OriginalSymbol")`
form returns **325**. A 2026-05-16 sweep re-extracted **286** Challenger rows, **all 286** carrying a non-null
`"OriginalInstrumentId"` — but for **275** of them that snapshot was taken **22 days earlier**, by the 2026-04-24
quarantine, not by the sweep (re-verified 2026-08-19: all 275 carry `"QuarantinedAt"='2026-04-24 00:18:52.867997+00'`;
only the remaining **11** were first snapshotted at the sweep itself). ADP rows **466076**/**466077** published
2026-07-10 12:24:50.729671 and were re-extracted at
12:25:10.108552 / 12:25:10.266989 — **twenty seconds later** — leaving `instrument_id` NULL on published rows.
**Any population keyed on current `instrument_id` mixes "never resolved" with "resolved, then re-extracted to null";
no conclusion drawn that way is safe.** Separate them: `SELECT re_extracted_at IS NOT NULL, "OriginalInstrumentId"
IS NOT NULL, count(*) FROM sentinel.extracted_observations WHERE instrument_id IS NULL AND extracted_at >= '<from>'
GROUP BY 1,2;` Read the second axis precisely: because the snapshot is first-writer-wins across `Quarantine()`,
`QuarantineInPlace()` and `ApplyReExtraction()`, `"OriginalInstrumentId" IS NOT NULL` means **"held an instrument at
the EARLIEST snapshot event"** — NOT "at extraction", and NOT "immediately before the latest re-extract". On a row
quarantined first, the column reports the pre-quarantine state and says nothing about what the re-extract found.
CONFOUND, not cause — **the April trigger remains unestablished.**

**THE DETECTION BLIND SPOT, and the crossing it explains.** No pattern -> no `RequiredSeries` entry -> no
`PatternDataHealthEvaluator` freshness row -> no `thresholdengine_pattern_severe_overdue_threshold_days` -> nothing
to alert on. That gauge reads **90** for each of the four Challenger/Truflation patterns and returns nothing at all
for `baltic-freight-recession`; `grep -rl` over `ThresholdEngine/config/patterns/` returns **zero** files for
`ADP_EMPLOYMENT`, `INDEED_POSTINGS` or `REDBOOK_SALES` (their only mention in the service is
`ThresholdEngine/AGENT_README.md`). Those three are outside its coverage and BDIY would die the same way: a
PATTERN-level instrument used as a FEED-level one, with nothing enumerating the gap. Neither the pattern FILES nor
the live registry is an authority alone — `ThresholdEngine/src/Configuration/PatternConfigurationLoader.cs:143`
`continue`s on `!pattern.Enabled` at load, so the registry cannot contain a disabled pattern and "all N enabled" is a
tautology, not a cross-check.
The four gauged patterns read **88** against threshold **90** on 2026-08-19 (`..._pattern_data_overdue_days`=88,
`..._pattern_severe_overdue_threshold_days`=90, `..._pattern_data_age_days`=118). `PatternDataSeverelyOverdue`
compares with a strict `>` (`deployment/artifacts/monitoring/alerts/thresholdengine.yml:145`) under `for: 24h`
(`:157`): from 88, equality falls 2026-08-21 and `90 > 90` is false, the condition first holds 08-22, so it fires
**~2026-08-23**. The climb is not a source waiting to publish, so the crossing is unavoidable and its alerts true.

**The publish gate, and why a plausible symbol does not survive it.** The gate is
`o.InstrumentId.HasValue && o.ResolutionConfidence >= 0.8f && o.Certainty is Definite or Expected`
(`SentinelCollector/src/Workers/ExtractionProcessor.cs:848` v1, `:1972` v2). **`Symbol` is not in the predicate**, so
a row carrying a plausible symbol and no instrument is dropped without a trace on the symbol axis. Nor is
`"InstrumentId": null` inside `candidate_symbols_json` the defect: **0 of 3,650,818** candidates all-time carry a
non-null value there, including every candidate on every row that published successfully. That field is the
pre-resolution proposal; the result lands on the ROW's `instrument_id`.

**THE LAST PUBLISHED VALUE ON BOTH VISIBLE FEEDS WAS JUNK**, so restoring resolution without auditing what gets
published resumes publishing junk into a recession detector and an inflation-divergence detector. Remediation must
gate on value plausibility, not merely on whether rows resolve again.
- `CHALLENGER_JOB_CUTS` published **600** (id **27753**, 2026-04-23 05:01:12), quote *"Electrolux ... will close its
  Jaszbereny factory in Hungary ... affecting around 600 employees"* — a single-company non-US layoff under a US
  NATIONAL monthly key, where a real print is tens of thousands. `challenger-layoff-surge`'s trigger is
  `cuts.HasValue && cuts.Value > 100000m` (`expression`, no default — an absent series simply does not fire), while
  the `?? 30000m` default lives ONLY in `signalExpression`; frozen at 600 the trigger can NEVER fire — and the stale
  value is worse than absence: the default yields signal 0, the stale 600 a confident +0.98.
- `TRUFLATION_CPI` published **3.3** (id **27425**, 2026-04-23 04:43:57), description `U.K. inflation`, quote *"U.K.
  inflation rose to 3.3% in March ... Office for National Statistics"* — a UK ONS print under the US
  `Truflation Daily Inflation Index` key (ids 24914/24913 before it are UK `transport inflation`, 2.4/4.7).
  `truflation-vs-cpi` returns Signal **-0.0019280253531531587**, Triggered **false**; `signalExpression` is
  `divergence = truflation - cpi(YoY)` then `signal = ±divergence/2`, so the DIVERGENCE is `3.3 - 3.3039` =
  **-0.0039** and the halved signal **-0.00195** — three decimals of agreement, NOT the five-decimal reproduction an
  earlier revision claimed. It establishes the served value **3.3000** rather than the `?? 2.5m` default (-0.40).
  **The UK print of 3.3 is what the matrix reads for TRUFLATION_CPI**, and sitting ~0.004 from US CPI YoY it reads as
  "no divergence": a dead feed on a foreign country's number presenting as a healthy null, the corpse-detector shape
  at its worst — no anomalous reading to notice. Re-checking the CPI leg REQUIRES a latest-vintage filter, since
  `CPIAUCSL` 2025-07-01 also carries an older vintage **322.132** (AsOf 2025-08-12) yielding YoY **3.3157**.
Both rows carry `"OriginalInstrumentId"` set with `instrument_id` now null. `SELECT id, description, value,
published_at, text_quote FROM sentinel.extracted_observations WHERE "OriginalSymbol"='<SERIES>' AND published_at
IS NOT NULL ORDER BY published_at DESC LIMIT 3;`

**A SECOND independent defect: `TRUFLATION_CPI` is declared Daily but judged at its monthly companion's cadence.**
`PublicationFrequencyDays` is `PublicationFrequencyDaysOverride ?? RequiredSeries.Max(...)`
(`ThresholdEngine/src/Configuration/PatternConfigurationLoader.cs:320-322` — the SAME unconditional overwrite
recorded for `buffett-indicator` earlier in this file; neither pattern carries an override), so `truflation-vs-cpi`
takes `Max(TRUFLATION_CPI=1, CPIAUCSL=30) = 30` and severe becomes `Math.Max(pubFreq * 3, 14)`
(`ThresholdEngine/src/HealthChecks/PatternDataHealthEvaluator.cs:32-33`) = **90**, where a TRUFLATION_CPI-only
pattern gets `Math.Max(3, 14)` = **14**. Overdue measures against pubFreq (118 - 30 = 88), so severe lands
~2026-08-23 instead of ~2026-05-08. Under a `Max` rule a stalled MONTHLY series always masks a dead DAILY one.

**D-18 RE-CHECK, recorded here and deliberately NOT applied to the card.** D-18 cites `challenger-layoff-surge 0.98`
(2026-08-12) as evidence that blanket namespacing would sever live feeds. `evaluate_pattern` returned Signal **0.98**
/ Triggered **false** / `DaysSinceLatestData` **118** on 2026-08-19, and under signal `-(cuts - 30000)/30000` that
0.98 inverts to `cuts = 600` exactly — the frozen value, confirmed without a `GetLatest` handle. The EXEMPLAR
therefore shows a frozen feed, not a live one. D-18's DECISION is not in question: Sentinel IS the primary source for
these six, so namespacing severs their only feed either way. No supersession.

**The bulk quarantine is NOT the forward-blocking mechanism.** A quarantine stamped at exactly
`2026-04-24 00:18:52.867997+00` hit **15,894** rows across **1,054** distinct `"OriginalSymbol"` values (re-verified
2026-08-19): **275** CHALLENGER_JOB_CUTS, **66** TRUFLATION_CPI, **35** ADP_EMPLOYMENT, **25** BDIY, **20**
REDBOOK_SALES, **17** INDEED_POSTINGS — it hit BDIY too, and BDIY recovered. **272 of the 275** Challenger rows had
ALREADY published before being quarantined, so there it is retroactive on already-sent data; the three exceptions
are ids **11152**, **12344**, **12815** (`resolution_state='Resolved'`, `published_at IS NULL`), which do not make it
a forward block either. `SELECT "OriginalSymbol", count(*), count(published_at) FROM sentinel.extracted_observations
WHERE "QuarantinedAt"='2026-04-24 00:18:52.867997+00' GROUP BY 1 ORDER BY 2 DESC;`
Its origin is INFERRED — do not repeat this search expecting to close it. `ExtractedObservation.Quarantine()`
(`SentinelCollector/src/Entities/ExtractedObservation.cs:218`) has never had a production call site in git history,
and `QuarantineInPlace()` (`:326`) is called only from `SentinelCollector/src/Endpoints/AdminEndpoints.cs:1589`,
added in `1040861f` on 2026-05-15 — three weeks AFTER the event. Likely a manual script or interactive session.

**SecMaster is not the defect.** Controlled 2026-08-19, both tools against both strings, the answer depends only on
the tool: `hybrid_resolve` returns `ExactSql` -> InstrumentId `ee98373d-cf04-4b16-bc7b-a3e6cf3ae57f` for BOTH
`"job cuts announced"` and `"challenger job cuts"`, `search_catalog` returns 0 for BOTH, and both are correct.
`search_catalog` matches symbol+name; `hybrid_resolve`'s first stage hits the ALIAS table, and `SELECT a.alias FROM
aliases a JOIN instruments i ON i.id=a.instrument_id WHERE i.symbol='CHALLENGER_JOB_CUTS'` returns **12** aliases
including the literal rows `job cuts announced` and `challenger job cuts` — neither a substring of the Name
`Challenger Job Cut Announcements` ("cuts" vs "cut"), which is why the name search misses and the alias resolve hits.
One live GIGO datapoint, and the reason it is NOT a hazard — do not re-raise it without re-reading this: **CFIGY**
(`CHALLENGER LTD-UNS ADR`, an Australian annuities firm) is a real instrument created **2026-08-18** by
`discovery_source='entity_resolution:gemini'`. Tempting, and refuted: resolving the vendor's own name proposes CFIGY
on NEITHER route (measured 2026-08-19, `q=Challenger, Gray & Christmas`) — the string appears nowhere in either
response body. Both routes retrieve the SAME five neighbours at the SAME scores, all FRED credit-card and
expected-inflation series (`RCCCBBALREV`, `EXPINF21YR`, `RCCCBACTDPD60P`, `EXPINF12YR`, `RCMFLBACTDPDPCT90POCC1`,
0.678-0.680), and differ only in method and in whether they resolve — so a resolution quoted without NAMING its
endpoint is unattributable. `/api/semantic/resolve-local` returns method **RagSynthesis**, `hypothesis` **MATCH**,
`instrumentId`/`symbol`/`confidence` null; it has NO `resolution` and NO `answer` field to report — its shape is
`{method, instrumentId, symbol, hypothesis, confidence, candidates}`
(`SecMaster/src/Endpoints/SemanticSearchEndpoints.cs:341`). `/api/semantic/resolve` returns method
**UpstreamDiscovery** and resolves to the CORRECT instrument, `CHALLENGER_JOB_CUTS` /
`ee98373d-cf04-4b16-bc7b-a3e6cf3ae57f`, its `ragResponse.answer` `NO_MATCH` over those same five. `sudo nerdctl exec
threshold-engine curl -s -G 'http://secmaster:8080/api/semantic/resolve-local' --data-urlencode 'q=<query>'` — the
param is `q`, not `query` (`query` is HTTP 400 on both routes); `secmaster` itself DOES have `curl` (`/usr/bin/curl`
8.5.0), `secmaster-mcp` is the container without it.

**METHOD NOTE, because it manufactured a false finding twice.** In `public.macro_observations`, `source_id` has the
form `{raw_content_id}:sig:{signal_identity_id}` for **39,689 of 42,716** rows (93%) — the rest are OFR/FRED feeds
keyed on a bare symbol, so it is a majority convention, never a schema guarantee. The numeric prefix is
`sentinel.raw_content.id`, NOT `sentinel.extracted_observations.id`; joining it to the latter lands on unrelated rows
and yields a convincing "systemic identity mismapping" that does not exist. **No order-of-magnitude separation
protects you from that join**: measured 2026-08-19 the id spaces were `raw_content` 1..**150,090** and
`extracted_observations` 10,604..**714,614** — a ratio of **4.8x**, OVERLAPPING across [10,604 .. 150,090], which is
precisely why the bad join looked convincing.
**EVERY absolute count in this entry is a moving snapshot of an append-only table — compare RATIOS AND SHAPES, never
integers.** Re-measured the SAME day, hours later, every figure this entry records moved: `NoResolution`
**620,351** where the `"OriginalSymbol"` paragraph below records 620,174; candidates **3,651,864** where the publish
gate above records 3,650,818; `macro_observations` **42,739**/**39,712** against the 42,716/39,689 in the METHOD NOTE
above; id bounds 1..**150,145** and 10,604..**714,827**. Every ratio and conclusion held — 93%, 4.8x, still
overlapping. The
two figures that did NOT move are the ones that are structurally exact rather than a running total: **358,398**, and
the **0** in "0 of N". A re-check returning different integers is this note working, not a contradiction; a re-check
changing a RATIO, or turning that 0 non-zero, is a real finding.
THREE THINGS ARE CALLED "Symbol": (1) the COLUMN `sentinel.extracted_observations."Symbol"` — quoted, PascalCase,
the resolved catalog symbol, NULL until resolution succeeds; (2) the KEY `"Symbol"` INSIDE `candidate_symbols_json`
— an LLM-minted slug of the proposed entity's name (`Challenger_Gray_Christmas`), not a catalog symbol and usually
absent from SecMaster; (3) the AXIS — any query or index keyed on (1). `"OriginalSymbol"` is NOT a fallback identity
for (1): it is written only by `Quarantine()` (`SentinelCollector/src/Entities/ExtractedObservation.cs:220`),
`ApplyReExtraction()` (`:265`) and `QuarantineInPlace()` (`:331`), each as `OriginalSymbol = Symbol`, making it a
PRE-REMEDIATION AUDIT SNAPSHOT — keying on it selects rows that were quarantined or re-extracted, NOT "the feed"
(**620,174** rows are `NoResolution` and **358,398** of those carry a NON-NULL `"OriginalSymbol"`). Rows with neither
column populated are invisible to every symbol-keyed query; reach them through the CANDIDATE or the description, and
use `LEFT JOIN LATERAL ... ON true` because a plain `,`-LATERAL drops every row whose `candidate_symbols_json` is
NULL or empty — those are dead rows too:
`SELECT o.id, o.description, o.value, o.resolution_state, o.resolution_method, c->>'Name' AS candidate
FROM sentinel.extracted_observations o LEFT JOIN LATERAL jsonb_array_elements(o.candidate_symbols_json) c ON true
WHERE o.extracted_at >= '<from>' AND (c->>'Name' ILIKE '%<vendor>%' OR o.description ILIKE '%<surface>%');`
No remediation is recorded, and one route is closed on principle: re-keying historical rows is a WRITE to production
data and is not on the table.

**Four of the five Grafana-managed rule groups expire out of Alertmanager between pushes, so a long-lived alert
emits a false `resolved` on every cycle.** Grafana forwards its native rules through a `prometheus-alertmanager`
contact point, which POSTs only when Grafana's OWN notification policy fires, and each posted alert carries
`endsAt = last_eval + 4x the RULE GROUP's interval`. Where that lifetime does not outlast the gap between pushes the
alert expires inside Alertmanager, which fires a `send_resolved: true` webhook to AlertService -> ntfy claiming a
recovery that never happened and then re-admits the same alert as new on the next push. Measured 2026-08-19 on the
live instance: `updatedAt` 17:12:10.004Z against `endsAt` 19:12:10.000Z on a 30m group -- exactly 4x -- while the rule
had been firing continuously in Grafana since 2026-08-03T01:42:10Z with 4 instances and Alertmanager's active list
(`/api/v2/alerts?active=true`) held none of them. Standing today against the root policy's 4h `repeat_interval`:

The scored inequality is `3x interval > push gap`, not 4x: `endsAt` is stamped at the EVALUATION, Alertmanager's
hold starts at the POST, and the POST fires on the notification-policy cadence up to one whole interval later. So
what a push GUARANTEES is `4i - i = 3i`; the 4x is the best case, true only when a push lands right after an
evaluation.

| rule group | interval | endsAt (4x) | held from push (3x) | push gap | standing |
|---|---|---|---|---|---|
| `loki-warning-rate` | 5m | 20m | 15m | 4h | expires 3h45m before the next push |
| `threshold-engine-projector` | 10m | 40m | 30m | 4h | expires 3h30m before the next push |
| `threshold-engine-regime` | 15m | 1h | 45m | 4h | expires 3h15m before the next push |
| `ofr-derived-cell-age` | 1h | 4h | 3h | 4h | expires 1h before the next push |
| `threshold-engine-pattern-data` | 12h | 48h | 36h | 24h | fixed in #981 -- 12h of margin |

`ofr-derived-cell-age` is a latent instance in its own right, and it is why #981 did not simply take the smallest
interval that clears. It was recorded here as ON THE BOUNDARY on the 4x reading (4h endsAt == 4h gap); on the 3x
bound it is not a boundary case at all but underwater by an hour, which is what the optimistic bound was hiding.
The other three have not yet been observed firing long enough to emit the false resolve, which is why all four are
recorded rather than fixed. The fix per group is to raise its `interval` until `3x interval > push gap` -- for
`ofr-derived-cell-age` that is past 1h20m, not the 1h that 4x would have accepted -- paid for in that much
detection latency; `interval` is the ONLY lever that moves
`endsAt`. TWO LEVERS THAT LOOK RIGHT AND ARE NOT: `keep_firing_for` extends how long GRAFANA holds the rule in a
firing state, not the `endsAt` it stamps on the push, so Alertmanager still expires it on schedule; and
`disableResolveMessage: true` on the `atlas-alertmanager` contact point gates only the resolve GRAFANA sends, while
this false resolve is generated by ALERTMANAGER itself when `endsAt` lapses and delivered by the `warning` receiver's
own `send_resolved: true` -- suppressing Grafana's leaves it fully intact.
The inequality is now guarded: `deployment/tests/alerts/check-routing.py` check (d) fails on any group that goes
underwater without being listed in its `KNOWN_UNDERWATER` set, and equally on a listed group that becomes healthy,
so closing one of these rows is what deletes it from the list.
Re-check for one alert (Alertmanager publishes NO host port -- a host-side `curl :9093` exits 7, which is
indistinguishable from "not held" -- and the image carries no `curl`; `amtool` and `wget` are present):
`sudo nerdctl exec alertmanager amtool --alertmanager.url=http://localhost:9093 --output=json alert query --active 'alertname="<rule title>"'`
`[]` means Alertmanager is NOT holding it (still broken); a non-empty JSON array means it is; a non-zero exit means
the measurement did not happen, which is a third answer and not the first. Verified 2026-08-19: `[]` for
`ThresholdEngine pattern approaching severe-overdue`, one element for `GeminiResolverApproachingFreeGroundingCap`.
Do NOT substitute `grep -c severe-overdue` over the JSON -- it also matches the Prometheus-sourced
`PatternDataSeverelyOverdue`, whose description carries that literal string, and which stood at 88 days against its
90-day threshold on these same four `pattern_id`s on 2026-08-19; it begins firing **2026-08-23**, after which a
substring check reads "fixed" whatever the truth is. NOT 08-22, and an earlier round "corrected" it to 08-22 the
wrong way: the gauge steps at UTC midnight (measured, 87 -> 88 between 2026-08-19T00:00:00Z and 00:05:00Z), so it
reads 91 on 08-22, which is the first day the strict `>` against 90 holds, and `for: 24h` puts the FIRING a day
after that. Re-check: `thresholdengine_pattern_data_overdue_days{pattern_id="truflation-vs-cpi"}` at 300s step
across a midnight.

## MEASUREMENT DEBT [instruments that cannot report their own dullness]

**The Alertmanager `warning` route's `repeat_interval` is not what paces a Grafana-managed alert, and reading it
as such overstates the notification volume.** Alertmanager delivered **687** webhook notifications total across
ALL alerts in 18.86 days of uptime (`alertmanager_notifications_total{integration="webhook"}`, container up since
2026-07-31T20:38:54Z) — about **36/day** for the whole stack, not per alert. Alertmanager is NOT a Prometheus
scrape target, so this counter is reachable only by curling `:9093/metrics` directly and is invisible to PromQL and
to every dashboard. Separately, Loki carried **72** `severity_text="Warning"` lines in the 24h to 2026-08-19T17:15Z
across all services, and `alert-service` logged none of them (prod level is Warning and a healthy container is
silent). Any "N warnings per day" figure should name which of the three populations it counts — webhook
notifications, ntfy messages, or Loki lines — because they differ by more than an order of magnitude.

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

**Three figures the PR-verdict decision check leaves un-re-checkable.** All measured 2026-08-17 while fixing that
check; none is a defect in it, and each is a number that will drift with nothing going red.

The POPULATION of the over-denial sweep is unpinned while its RESULT is pinned. `run-pr-verdict-smoke.sh` BD18
sweeps every `#NNN` in this file and asserts the refusing set is exactly {729, 935}, so a new decision entry turns
that row red — but NOTHING asserts how many numbers were swept, in either store that quotes the figure. Both now
carry a revision stamp instead (`112449be`, where it read 29), which keeps them true but leaves the quantity
uncheckable: the count moves whenever an entry here names a PR, and THIS entry moved it to 30. A stamped figure and
an asserted one are not the same guarantee, and only the second survives a reader who re-quotes it without the
stamp. Re-check: `grep -oE '#[0-9]+' docs/BACKLOG.md | sort -u | wc -l`. Fix is a BD row asserting the population
beside the set, which costs one line and makes the drift visible where the other numbers already are.

Rounds and recorded verdicts are different quantities and only one is mechanical. PR #978's body says seven review
rounds on #974; `~/.claude/atlas-pr-verdict.log` holds SIX verdict records for it (four block, two approve,
14:01-18:24 on 2026-08-17). Neither falsifies the other — a round ending without a recorded verdict leaves no trace
in that log, which is the log's designed scope — but "rounds" is a human count with no store behind it while the
six is the only mechanical record, so a claim citing rounds cannot be checked against anything. Re-check:
`grep -c 'PR#974' ~/.claude/atlas-pr-verdict.log`.

Which PUSH deny answers which shape is asserted by no test, and the one carrying the useful remedy is reachable
only by a shape prose cannot produce. `git-push-guard.sh`'s two-pushes-merged-into-one-span deny is the only push
refusal naming the pair `git commit -F <file>` / `gh pr create --body-file <file>`; every prose shape reaches an
EARLIER deny — the span/prefix mismatch ("Run the pushes as separate commands"), direct-to-main ("use a feature
branch"), or unknown-branch ("check 'git branch --list'") — because both push-count derivations read the same
`git … push` text, so a second mention raises the prefix count too and the mismatch fires first. It is NOT dead
text, which an earlier draft of this entry guessed it might be: measured 2026-08-17, a newline inside a quoted
option value reaches it — `git -c "user.name=A<newline>B" push origin main` yields raw count 1 and isolated count
0, so the prefix comparison passes on 0 == 0 and the backstop fires. Recorded because
`supervisor-mode/LESSONS.md` ALREADY_ENCODED now cites which denies say what while nothing pins that mapping, so
reordering a rule would silently change the message a blocked reviewer reads. Re-check both halves: put
`git push origin <a real branch> # git push origin main` and the newline shape through the hook and read which
deny answers each.

**A Sentinel unit test fails spuriously, and it fails in the shape of a real regression.** Measured 2026-08-15: over
5 consecutive full runs of the Sentinel suite on identical trees, `ExtractionProcessorArticleEmbeddingTests`
`.should_tag_skipped_empty_when_text_blank` went red ONCE — `Expected capture.Results to contain a single item, but
found {"failed", "skipped_empty"}` — and the same committed tree then passed 2218/2218 on the immediately following
run. Root cause located, not guessed: exactly two test classes touch
`sentinel_news_article_embedding_total`, and only ONE of them is collected. `ExtractionProcessorArticleEmbeddingTests`
carries `[Collection("SentinelMeterStatic")]`; `ExtractionProcessorTests` — which reaches the same
`TryCaptureArticleEmbeddingAsync` through the real pipeline, with no embedder configured, i.e. emitting exactly the
leaked `"failed"` — carries NO collection attribute, so it sits in its own implicit collection and runs IN PARALLEL
with the listening class. The capture is a global `MeterListener` filtered by meter+instrument name but NOT by test,
so that concurrent measurement is indistinguishable from the one under assertion.
Two candidate fixes, neither verified here: put `ExtractionProcessorTests` in the same collection (one line, but
serialises a large class), or scope the capture per-test — a tag the test alone sets, or an assertion on the DELTA
rather than the absolute set. Never a retry. NOT fixed in the PR that found it because flake reproduction is
stochastic (2 in 7 runs), so a green suite would not have demonstrated the fix worked.
Cost is not the flake, it is the DIAGNOSIS: the failure looks exactly like a real cross-test regression, and it lands
on whoever is running the suite for an unrelated reason — here a re-extract change that touches none of this code,
including on a DOCS-ONLY commit whose predecessor tree had just passed 2218/2218, which is what proves it
content-independent.

**16 citations in tracked `.md` cannot land, and rc 1 is therefore the corpus's steady state.** Reproduce:
`git ls-files -z '*.md' | xargs -0 python3 scripts/verify-citations.py --quiet` — 296 checked, 17 cannot land at
`68b655df` (PR #967 head), 16 once that PR's own regression in the architecture-cards exemplar card is repaired;
292 checked / 16 cannot land at its base `eb2835e8`. So a green run is not the bar today and nobody should chase one
as a merge gate: judge a PR on whether its cannot-land SET is a subset of its base's. Composition of the 16, because
they are three different jobs and only the first two are defects: 11 AMBIGUOUS basenames a `--scope` would resolve
(`server.py` eight times, ambiguous across `gemini-resolver-mcp` and `SentinelCollector`, plus `Program.cs`,
`DependencyInjection.cs`, `AdminEndpoints.cs`); 3 UNRESOLVED "no file named", of which ONE — the `src/Removed.cs`
cite in the architecture-cards `weak-card` test fixture — is a DELIBERATE broken citation and must stay broken,
while `QuarantineGeminiEquityEtfJunk.cs` and `new-guard.sh` are genuinely gone; 2 BLANK landings, in
`regime-news-staleness-redesign.md` and `ReviewUiEndpoints.cs`. THIS ENTRY DELIBERATELY CARRIES NO `file:line`
FORMS — the tool parses its own backlog entry, so spelling the findings out verbatim ADDS four findings to the
count it is describing (measured, not feared). Re-run the command rather than trusting these counts: the figure
moves with every doc edit, and the tool cannot see content drift at all (see its module docstring), so it is a
FLOOR on rot, never a census of it.

**2,138 assertions that no suite-level count can see.** `run-push-config-destination-smoke.sh` prints no
per-assertion `PASS:` line — its `pass()` (`:44`) increments a counter silently while `fail()` (`:45`) does print
`FAIL: <label>` — so its only positive output is the summary at `:656`. Measured 2026-08-15:
`bash .claude/hooks/test/run-push-config-destination-smoke.sh` -> rc 0, 22 lines of output, **0** of them matching
`PASS: `, ending `cells: 1020 + 14 targeted + 33 config-injection` / `PASS=2138 FAIL=0`. It is the only one of the
ten suites in `.claude/hooks/test/` that emits none — `grep -L 'PASS: ' .claude/hooks/test/run-*.sh` names it and
nothing else. A sweep that counts `PASS:` lines therefore scores this suite 0 and reads its coverage as ABSENT
rather than passing; the failure direction is visible, the coverage direction is not. Either print per-assertion
lines or teach the sweep to read `PASS=<n>` — until then, count 2,138 here by hand.

**FredCollector writes to an UNBOUNDED channel nobody reads — the same orphan that wedged Finnhub, failing the
other way.** `FredCollector/src/Events/ObservationChannel.cs` extends `EventChannel<ObservationCollectedEvent>`
(`Channel.CreateUnbounded`). Two live writers — `DataCollectionService.cs:231` and `BackfillService.cs:147`, both
`PublishAsync` — and ZERO readers: `EventChannel.ReadAllAsync` (`:35`) is DEFINED and never called, measured
2026-08-17 by `grep -rn "ReadAllAsync" FredCollector/src`, whose only hits are the definition itself. Because it is
unbounded it grows instead of blocking, so it leaks rather than halting; the card already says so
(`ObservationChannel no reader=memory-growth`) but there is no gauge and no measured growth rate, so nobody knows
whether it matters. This is the SAME dead scaffold FinnhubCollector carried (channel + a never-implemented
`IEventPublisher`, both from the service's first commit), and the same reasoning applies: FredCollector's gRPC
`EventRepository` serves `FredObservations` straight from the DB, so the channel carries nothing to anyone. NOT
fixed here — deliberately out of this change's blast radius. `FredCollector/README.md:24` actively asserts the
opposite ("published over the `ObservationChannel` to gRPC subscribers"), which `:146` of the same file then
correctly denies ("polls the DB events table — it does NOT emit from an in-process channel"); that first line is the
reason a reader can believe this channel is load-bearing, and it should go in whatever PR fixes the channel.
Re-check: `grep -rn "ReadAllAsync\|\.Reader"
FredCollector/src` must show a real consumer, or the writers must be gone. Related and smaller: FinnhubCollector's
`IFinnhubRepository.GetObservationsSinceAsync` (`src/Data/FinnhubRepository.cs:530`) is likewise a zero-caller
survivor of that scaffold — harmless (a read, it cannot block) and left in place.

**ThresholdEngine builds a channel per event type that has neither a reader NOR a writer.**
`ThresholdEngine/src/Events/ChannelEventBus.cs:190` — `GetOrCreateChannel<TEvent>` creates a
`Channel.CreateUnbounded<TEvent>` for every event type published, and nothing ever enqueues to it or drains it:
`PublishAsync` (`:58`) invokes the subscribed handlers directly through `Task.Run` and BYPASSES the channel
entirely. Measured 2026-08-17: `grep -n "\.Writer\|\.Reader\|ReadAllAsync\|WriteAsync"
ThresholdEngine/src/Events/ChannelEventBus.cs` returns exactly ONE hit, `Channel.Writer.Complete()` in `Dispose`
(`:257`). So this is the third variant of the same dead scaffold and the only HARMLESS one: with no writer it
cannot grow (unlike FredCollector's above) and with no bounded capacity it cannot block (unlike the Finnhub
channel that wedged prod for 16 days). Recorded because PR #975 enumerated the repo's remaining channels and this
one is absent from that table, which reads as "there is no third channel" rather than "the third one is inert" —
and because the class NAME is the trap: an event bus called ChannelEventBus that dispatches without touching its
channel is exactly the thing the next agent reasons about wrongly. NOT fixed here: deleting it is a
ThresholdEngine change with no defect driving it, and this round's blast radius is FinnhubCollector. Re-check: the
grep above must still return only the `Dispose` hit; more than that means the class has grown a real channel path
and stops being inert.

**Alert rules and the metrics they read ship on different schedules, and rule-first pages a healthy system.**
`FinnhubCollectorQuoteCollectionStalled` carries an `absent()` leg and ships via `--tags monitoring`, while the gauge
it watches ships inside the container image. Deploy the rule first and `absent()` is true from the moment
Prometheus loads it, so a healthy collector pages 15 minutes later; deploy the image first and the worst case is a
few minutes of an unwatched gauge. Sequenced image-first BY HAND for this PR, which is exactly the kind of
knowledge that does not survive. NOT fixed here: the durable fix is ordering (or gating) inside `deploy.yml`.
Re-check: `deployment/ansible/playbooks/deploy.yml` must either template rule files after the service image is
running, or refuse a rule whose metric is absent from `/api/v1/label/__name__/values`.

**A test class that forgets `[Collection(StalenessGaugeCollection.Name)]` silently re-opens a data race, and the
suite stays green.** `FinnhubCollector/tests/Workers/StalenessGaugeProbe.cs` declares the collection that serialises
every class touching FinnhubMeter's process-global staleness origins; membership is enforced by PROSE in that
docstring and by nothing else. Measured 2026-08-17 on the pre-existing pair: as shipped the two classes are strictly
disjoint (6.3s wall, serialised); split into two collections they run concurrently (3.0s) and 2 of 4 unsynchronised
appends were lost — and the split control still ran GREEN 4/4, which is the whole problem. The blast radius grew with
this PR: the origins are now a dictionary that `TrackActiveSymbols` PRUNES, so an unserialised sibling can delete the
symbol another test is reading, not merely move a timestamp. NOT fixed here. Re-check: `grep -L
"StalenessGaugeCollection.Name" FinnhubCollector/tests/Workers/*Tests.cs` must return nothing.

**`QuoteStalenessSeeder` resolves its repository OUTSIDE the try, so a DI failure is fatal to startup.**
`FinnhubCollector/src/Workers/QuoteStalenessSeeder.cs` calls `GetRequiredService<IFinnhubRepository>()` before the
`try`, so a resolution failure throws out of `StartAsync` and takes the host down — where every other failure in this
seeder is deliberately Warning-and-continue, because collection does not depend on the seed. Theoretical today:
`FinnhubRepository`'s constructor takes only a `FinnhubDbContext`, and `FinnhubCollector/src/Program.cs:144` would already have thrown on
a dead database. It stops being theoretical the moment that constructor grows a dependency. NOT fixed here: moving it
inside the try changes the startup failure mode of a service that is currently wedged in prod, and this PR's job is
to make the wedge visible. Re-check: the `GetRequiredService` call must sit inside the `try` block, or
`FinnhubRepository`'s constructor must still take exactly one parameter.

**The two remaining collector dead-man alerts may be as blind as Finnhub's was, and nobody has checked.**
`FinnhubCollectorScheduledCollectionMissed` watched `sum(increase(finnhub_api_requests_total[3d])) == 0` and could not
fire during a 16-day total collection stall, because SecMaster catalog-enrichment and SentinelCollector resolution
call FinnhubCollector's live-passthrough endpoints on their own schedules: measured 2026-08-17,
`sum(increase(finnhub_api_requests_total[3d]))` = **104,135** with quote collection dead since
2026-07-31T21:34:45Z. That rule is now replaced by a work-path gauge. `OfrCollectorScheduledCollectionMissed` and
`AlphaVantageCollectorScheduledCollectionMissed` still use the counter form, and the question that invalidated the
Finnhub one — "can anything other than this collector's own scheduler move this counter?" — has not been asked of
either. OFR at ~9 req/business-day has almost no margin for a confounder to hide in; AlphaVantage at ~870/day has
plenty. NOT fixed here. Re-check: for each, enumerate every caller that can reach the collector's upstream, then
confirm the counter goes flat when its scheduler alone is stopped.

**The staleness-origin fallback degrades the dead-man SILENTLY — no log line, no metric, nothing.**
`FinnhubMeter.ProcessStartTicksOrNow` (`FinnhubCollector/src/Telemetry/FinnhubMeter.cs:89`) catches a failed
`Process.StartTime` read and returns `DateTime.UtcNow.Ticks`. The catch is correct — an escaping exception becomes a
CLR-cached `TypeInitializationException` that kills every meter in the process — but the fallback then measures
staleness from METER INIT rather than process start, understating it by the whole boot duration, and nothing
anywhere says so. That understatement runs in the direction that makes `FinnhubCollectorQuoteCollectionStalled`
LESS likely to fire, and boot duration is unbounded here because no `lock_timeout` is set on the migration
connection (see DEFERRED WORK). So the degraded state is indistinguishable from the healthy one at every surface an
operator can read. The comment previously justified the silence by claiming "no logger exists this early", which was
FALSE and is corrected as of 2026-08-17: `FinnhubCollector/src/Program.cs:38` sets `Log.Logger` before `:63`'s
`AddMeter` and long before `:144`'s migration, so Serilog's static `Log` is available at this point. NOT fixed here: emitting the signal is a
behaviour change. Re-check: force the catch (make `readProcessStart` throw) and confirm that no log line is written
and no metric distinguishes the meter-init origin from a process-start one.

**A conflicted path in the index makes the alerts selftest report a permissions defect that does not exist.**
`deployment/tests/alerts/selftest.sh:457` reads a file's mode with `git ls-files -s -- "$f" | cut -d' ' -f1`, which
assumes ONE row per path. During an unresolved merge `git ls-files --stage` returns stages 1/2/3, so the mode
variable becomes the mangled `100755\n100755\n100755`, fails the `= "100755"` comparison, and the control prints
`a #! file is not executable in the index` — naming a permissions failure for a file whose permissions are fine.
Measured 2026-08-17 during the #975/#973 merge: the suite scored **41/42** with the conflict unresolved and
**42/42** the moment the resolutions were staged, with no file mode touched in between. It is a conflict-state
artifact misreporting as a permissions defect, and it misleads in the expensive direction — an agent mid-merge is
told to run `git update-index --chmod=+x` on a file that needs nothing. Fix: take the last field, or filter to
stage 0. Re-check: run `selftest.sh` with any conflicted path in the index and read the FAIL line.

**Loki's `service_name` for this service is `finnhub-collector-service`, and the bare name matches NO stream.**
Measured 2026-08-17: a health query filtered on `service_name="finnhub-collector"` returned zero Warning entries and
was nearly reported as clean health — it had matched no stream at all. This is the worst possible failure shape
here, because prod log level defaults to Warning and a HEALTHY container therefore emits NOTHING: an empty result
from a wrong label value is byte-identical to an empty result from a healthy service. The value comes from
`FinnhubCollector/src/Program.cs:35` (`["service.name"] = "finnhub-collector-service"`). Whether every ATLAS service carries the
`-service` suffix is NOT established — `list_loki_label_values` for `service_name` over the last 24h returned
`SecMaster`, `sentinel-collector`, `finnhub-collector-service`, `threshold-engine-service`, `reports-daily-host`,
`reports-weekly-host`, so the suffix is demonstrably NOT uniform, and that listing covers only services that logged
in the window rather than the full roster. Do not infer a service's label from its container or tag name. Re-check:
`list_loki_label_values` for `service_name`, or assert that a known-noisy window returns rows before trusting an
empty one.

**`verify-citations.py` reports GREEN on a citation that has drifted onto the WRONG line — it only catches the ones
that land on a blank.** The tool resolves a `file.cs:NN` reference and confirms line NN exists; it does not and
cannot confirm NN is still the line the prose meant. So a green run is evidence that a citation points at a line
that EXISTS, never that it points at the RIGHT one. Measured 2026-08-17: a 6-line comment edit inside
`FinnhubCollector/src/Telemetry/FinnhubMeter.cs` shifted five D-2 GUARD citations in `FinnhubCollector/AGENT_README.md`
down by six lines each (`TrackActiveSymbols` 144->150, `SeedLastQuoteCollected` 172->178, `MarkQuoteOriginsDurable`
212->218, `QuoteCollectionStaleness` 246->252, `QuoteStalenessOriginDurable` 286->292). The tool flagged exactly
ONE of the five — `:212`, and only because six lines further on happened to be blank. The other four had drifted
onto real-but-wrong lines and read GREEN. Worse, a bare exit-code check catches none of it: this repo's docs already
carry 2 unrelated unresolvable citations, so rc is 1 both before and after, and "still rc 1" looks like no change.
Only the COUNT moved: base `21b02a58` measured **91 citations checked / 2 cannot land**; the same two docs with the
comment edit applied and the drift NOT yet repaired measured **94 / 3**. The third cannot-land is the `:212`
citation and ONLY that one — six lines on from its old target happened to be blank. The other four drifted
citations are counted inside the 94 and reported as landing, which is the entire point: the arithmetic reconciles
as 2 standing unresolvables + 1 blank = 3, so a sweep can never have flagged more than one of the five.
CONSEQUENCE: any edit that shifts line numbers in a cited file requires a BASELINE COMPARISON, not a green run —
and after repairing, re-derive each cited line's content by hand, because the tool will pass whatever you write.
Re-check: run the tool on the touched docs from a pristine checkout of the merge-base and again from the branch, and
compare the `N citation(s) checked, M cannot land` line from each; M must not rise, and any rise in N must be
accounted for by citations you deliberately added.

**`verify-citations.py` silently skips a citation whose file has an extension outside a 10-item allowlist, and
skips a bare `:NN` continuation entirely unless `--bare` is passed.** Two separate gates, both read from the code
2026-08-17. The first is `scripts/verify-citations.py:121-122`:
`_EXTS = "py|cs|yml|yaml|md|sh|json|csproj|ts|sql"` feeding
`CITATION = re.compile(rf"(?P<file>[\w./-]+\.(?:{_EXTS})):(?P<start>\d+)(?:-(?P<end>\d+))?\b")` — the `\.` is
mandatory, so an extensionless path never matches, and neither does a real extension that is not on the list. This
is DELIBERATE and the rationale is at `:119-120`: a permissive `\w+\.\w+:\d+` also swallows version strings and
`host:port` URLs. So the gap is a precision/recall tradeoff already priced in, NOT a bug to widen on sight —
widening it changes what the whole repo's sweep reports and needs its own before/after counts. The second gate is
`:127` `BARE`, which parses a continuation like ``(`:147`)`` only when `--bare` is passed; the docstring at `:103`
says to leave it off by default, so in a default run those references are not checked at all.
MEASURED, 190 tracked docs swept on base `21b02a58`: **ZERO** citations anywhere name an extensionless file, so the
`scripts/` executables (`claude-pr-verdict`, `claude-mark-verified`) carry no `file:NN` citation in any doc — that
exposure is latent, not live. What IS live is the allowlist: **5** citations resolve to a real file and are
invisible — one in `deployment/artifacts/compose.yaml.j2`, one in `.gitignore`, one in
`SentinelCollector/src/cod-prompts/cod-dsl-v2.3.gbnf`, and two `/opt/ai-inference/prompts/cod/*.txt` host paths.
The `.j2` is the one that bites, because ansible templates are gate-layer files whose line numbers move.
Probe confirming the mechanism rather than inferring it from a count: a file citing line 183 of the extensionless
`scripts/claude-pr-verdict` alongside line 122 of `scripts/verify-citations.py` reports `1 citation(s) checked` —
the `.py` one. NOTE the file:line pairs above and in that probe are deliberately written in a NON-citation form
(bare filename, line number in prose). Spelled the normal way they would be invisible citations pointing at real
lines, so this entry would silently rot while documenting exactly that failure — and it would make its own
"zero extensionless" measurement false. Regenerate them from the re-check instead of maintaining them here.
The entry above on `scripts/claude-pr-verdict` is this file's own worked example of the second gate: its references
are bare continuations, so none of them is checked by any default invocation. Re-check: sweep every tracked `.md`
for `path:NN` tokens that resolve to a real file but do not match `CITATION`, and confirm the count and the
extension breakdown before trusting a sweep that reports "every one lands".

**Applying a patch here drops the executable bit, `core.fileMode=false` hides that from every normal read, and a
disarmed hook then fails SILENTLY.** Four mechanisms compose into a defect with no symptom, and the composition is
the point — each one alone is survivable. (a) A patch applied to this tree has twice landed its touched files
non-executable on disk, both times on 2026-08-17; the second occurrence left two LIVE hooks disarmed until they were
repaired by hand. (b) `core.fileMode=false` is set in this repo, so git ignores the on-disk mode entirely: `git status`
is CLEAN across the whole hazard and `git diff` emits no `old mode`/`new mode` line. Nothing in the ordinary
pre-commit read can show it — the index must be inspected explicitly. (c) `.claude/settings.json` wires every hook as
a BARE PATH — `$CLAUDE_PROJECT_DIR/.claude/hooks/<name>.sh`, **18** entries, and **0** of them prefixed `bash` — so a
non-executable hook is an exec failure the harness swallows: no notice, no stderr in the transcript, the guard simply
never speaks again. A gate that has stopped gating is indistinguishable from a gate with nothing to say, which is why
this belongs here and not under KNOWN DEFECTS. (d) `git add` records a NEW file as **100644 regardless of disk mode**
under `core.fileMode=false`, so a hook authored executable and staged the normal way ships disarmed on its first
commit. Probed in a throwaway repo 2026-08-17: disk 755, plain `git add` -> 100644, `git add --chmod=+x` -> 100755.
Note `git update-index --chmod=+x` — the form most references reach for — is DENIED by `ansible-gate-guard.sh` for
ANY path, not merely a gate-layer one (probed directly against the guard, deny; `git add --chmod=+x` allows), so it
is not an available repair here.
MEASURED: SIX files carried the hazard in that one day — `commit-marker-staleness.sh` and
`lessons-uncommitted-notice.sh`, both NEW at `1f813af9` and therefore mechanism (d); the three suites
`test/run-advisory-guards-smoke.sh`, `test/run-wiring-smoke.sh` and `test/run-pr-verdict-smoke.sh`; and
`scripts/claude-pr-verdict`, modified at `d4c5b244`. All six read 100755 in HEAD, 100755 in the index and 775 on disk
as of this entry, so what is recorded here is the HAZARD and its re-check, not an open break.
SCOPE, so the re-check is not widened on sight: the bit matters only where something execs the file DIRECTLY, which
here is `.claude/hooks/**` plus the two extensionless `scripts/claude-*` executables. An interpreter-invoked script
does not need it, and **13** tracked files under `scripts/` carry a shebang while sitting 100644 in the index —
`verify-citations.py`, `devcontainer-owner.sh`, `agent-stall-watchdog.sh`, the four `gemini-spend-calibration`
modules, the four `sentinel-quality-check` modules and the two `test-devcontainer-*` scripts. Every one is invoked as
`python3 …` or `bash …` and is CORRECT as it stands; the 13 `compile.sh` recorded in CLAUDE.md are the same story. A
shebang-based sweep flags all of them and teaches the next reader to ignore the check entirely.
Re-check — it must read the INDEX, because the disk is the part already repaired by hand twice and the index is what
ships. Silence is the pass:
`git ls-files -s -- .claude/hooks scripts/claude-pr-verdict scripts/claude-mark-verified | awk '$4 !~ /\.md$/ && $1 != "100755" { print "NOT 100755 IN THE INDEX:", $1, $4 }'`
CONTROL, because a sweep whose only output is silence is an opinion: pipe one fabricated
`100644 <sha> 0<TAB>.claude/hooks/git-push-guard.sh` line into that same `awk` and it must name that file back.
Verified both ways 2026-08-17. Repair is `git add --chmod=+x -- <path>`; a `chmod` on disk fixes nothing that ships.

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
reproducible by a later reviewer. PARTIALLY SUPERSEDED 2026-08-16: the classification-layer salvage landed on
main's guard — 15 of those write shapes closed, and the object-store carve-out deliberately NOT opened.
#935 itself stays blocked and unmergeable; what it still holds over the salvage is the lexer, worth 3 more shapes
(see the correction two entries below).

CORRECTION 2026-08-16, and it is the SAME correction #935 needed three times: the salvage's "LOOSENED=0 against
`67396749` on an 83-row regression corpus" was FALSE, and false for the reason that number has always been false
here — **the corpus did not contain the shapes**. Re-measured on a 342-row corpus (312 write rows, 30 read rows),
the landed salvage LOOSENED **60 write shapes** against `67396749`, in two classes it had no fixture for:
  OBJECT STORE, reopening the exact route #935 was blocked for. `git_operand_class` read `POS[1] == "--"` as proof
    the content came from the index, but a source-selecting FLAG never enters `POS` — the walk consumes dash
    tokens — so `--` slid into that slot. `git restore --source=<rev> -- <gate>`, `-s<rev>` glued, `--worktree
    --source=`, `--staged --source=`, `git checkout --ours --` and `--theirs --` all ALLOWED. Proven end to end:
    `hash-object -w` -> `mktree` -> `commit-tree` -> `git restore --source=<commit> -- <path>`, every step allowed,
    tracked file contents replaced. The three existing OBJECT STORE fixtures stayed green because every one of
    them is spelled WITHOUT the `--`.
  BUNDLED DESTINATION FLAGS. `dest_flag` tested `-t` and `--target-directory` by exact string, so `-rt`, `-at`,
    `-vt`, `-ft` and `install -Dt` never set it and `dest_last` then trusted `POS[-1]`, which under `-t` is the
    SOURCE. `cp -rt .claude/hooks /tmp/evil/.` overwrites every hook in the layer and was ALLOWED.
FIXED in the same PR by one rule at two sites — an unrecognised dash token makes the class fail closed, rather
than a list of flag spellings. That round claimed "LOOSENED=0 write rows on the 342-row corpus"; **that number was
false for the fourth time, and for the same reason every time — the corpus did not contain the shapes.** Chasing
zero is what produced four rounds; the bar was changed to NET BETTER with a strict zero only on HARD_STOP paths.

MEASURED 2026-08-16 on an independently built 498-row corpus (447 write, 16 read, 35 ordinary), guard versions run
IN their own hook directories, against `170be75a` (main; guard blob `c5ac440a`, unchanged by #970):
  writes  **tightened 110, loosened 4, NET +106**. All 4 loosened rows are the MANDATED seam
    (`git checkout|restore -- <gate>`), which the acceptance bar requires to allow. Unmandated write loosenings: 0.
  **"HARD_STOP loosened: 0" was FALSE, for the fifth time and by the mechanism named three lines above — the
    corpus did not contain the shape.** The 4 mandated-seam rows were counted as gate-only, but `reader` checks
    NOTHING, so the same class skipped the DEPLOYED rule too: 16 rows spelling `git checkout|restore -- <deployed
    path>` BARE deny on `c5ac440a` and were allowed here, `/opt/ai-inference/compose.yaml` among them (measured
    2026-08-16; the flag-carrying form was covered, the bare form was not). Closed in the round below, which
    re-applies the deployed rule — and only that rule — to a `reader` `checkout`/`restore`'s operands.
  reads 16 rows: 3 loosened, counted separately so the reader-class price cannot bury a write regression. One of
    the three names a HARD_STOP path — `cp /opt/ai-inference/compose.yaml /tmp/compose.bak` — and it is a READ:
    the destination is `/tmp`, and `dest_last` is sound for `cp` now that the destination FLAG is parsed.
  ordinary work 35 rows: **0 new false denials**, against main and against the branch head.
The corpus is only as good as itself: 498 rows chosen by one author, and each previous round's zero was true of its
own corpus too. Corroborating evidence that does not depend on the corpus: every one-change mutant kills assertions
in the repo suite, and the harness carries a known-bad control that must show a planted loosening before any zero
is reported. Per-fix mutation counts, suite assertions killed / corpus rows flipped to allow (suite counts net of
the 2 bare-basename artefacts recorded two entries below):
  destination-flag prefix match, reverted to the exact canonical name   4 / 124
  glued short-bundle value, reverted to the unpinned greedy prefix      2 /  27
  `WRITE_RE` git arm, reverted to requiring the subcommand to abut git  6 /  74
  ambiguity signal, reverted to requiring a positional first            4 /  60
  known-bad control (short-bundle `dest_flag` disarmed)                 3 /  36
No mutation measured zero. The last two are the SAME family: neither closes the global-option shape alone.
Suite: **172/0 at `170be75a` -> 215/0 at the branch head -> 234/0 -> 239/0 with the reader/deployed round**, all
measured, not counted. That round's mutation counts (suite assertions killed): drop the deployed re-check 3,
widen it to the gate rule as well 3, drop its `checkout`/`restore` condition 1. No mutation measured zero.

**RECORDED, NOT CHASED — what the 498-row corpus surfaced beyond the two families it was scoped to fix.**
None touches a HARD_STOP path, which is the only reason each was left open (2026-08-16):
  `cp <gate file> /tmp/x` and `cat <gate file> > /tmp/x` ALLOW here and DENY on main. This is the reader/`dest_last`
    class working as designed — the gate file is the SOURCE — but it is also **the obstruction that stopped the
    previous review measuring deletion counts**, and it disappears the day this merges. CONFIRMED by measurement,
    since the entry is what the next reviewer will plan around: the INSTALLED guard (`/home/james/ATLAS/.claude/
    hooks/ansible-gate-guard.sh`, blob `c5ac440a`, identical to main) DENIES `cp <hook> /tmp/backup.sh`, so mutants
    cannot be staged out of the layer without a bypass. Re-check by feeding that command to the installed hook.
  `cp -T <gate file> /tmp/x` and `cp --no-target-directory <gate file> /tmp/x` ALLOW, same class, same reason.
    `--no-target-directory` is deliberately NOT a prefix of `target-directory`, so it keeps `dest_last`.
  Every finding above is a READ of a gate path. No write shape against the gate layer was left open.

**`run-wiring-smoke.sh` has been RED on main since #940, and its whole job is to notice a hook being
unwired.** Measured 2026-08-16: `WIRING SMOKE: FAIL`, one assertion, `registered set drifted: >
dream-pending-notice.sh`. `.claude/settings.json:160` wires that hook (added by #940, merged) but the
suite's `EXPECTED_WIRED` list at `run-wiring-smoke.sh:63-67` still names 14 hooks and its pass message
says "exactly the expected 14". So the tool that would report "a guard got unwired" has been reporting
FAIL for an unrelated reason, which is how that signal gets ignored. Not fixed here: bumping the list to
15 asserts the hook SHOULD be wired, which is a judgement about #940 and not this round's to make.
Re-check with `bash .claude/hooks/test/run-wiring-smoke.sh`; it is red on main, not on this branch's
changes. Nothing in this repo runs these ten suites automatically (already recorded in hooks/README).

**A harness that copies the guard into a scratch directory cannot see the bare-basename rule, and it fails GREEN.**
`GATE_BASENAMES` is built from `$_hookdir/*.sh`, i.e. the guard's OWN directory, so a copy in a private scratch dir
knows only the basenames of whatever sits beside it. Measured 2026-08-16: pointing `ATLAS_GATE_HOOK` at a scratch
copy of the CURRENT guard turns 2 of the suite's 234 assertions red ("unwiring by bare basename after cd",
"deleting a hook by bare basename") with no code defect present — and the same 2 rows are silently absent from any
corpus measured that way. The mutation counts in this file are quoted net of that constant. Re-check by running
`run-advisory-guards-smoke.sh` twice, once in place and once with `ATLAS_GATE_HOOK` at a copy, and diffing the
FAIL lists. Not fixed: the honest fix is a hook-dir fixture, and creating files named after the other guards is
itself denied by the installed guard.

**A SECOND siting axis exists, it has no self-check at all, and it surfaces as one ordinary red row.**
`design-intent-dispatch-guard.sh` resolves `REPO=$(cd "$_dir/../.." && pwd)` and globs `"$REPO"/*/AGENT_README.md`.
With no service cards under that root it logs `ANOMALY: no service card ... failing open` and returns `none`, which
lands in `run-advisory-guards-smoke.sh` as `design-intent-dispatch-guard returned 'none', expected 'deny'` — a row
that names the GUARD, reads exactly like a guard regression, and says nothing about the harness. Measured 2026-08-17
on a scratch tree carrying every `.claude/hooks/**` file but no cards: **418/1**; siting the 12 cards at `$REPO` and
changing nothing else: **419/0**, matching the in-place count. The suite's `HARNESS MISCONFIGURED` check covers only
the OTHER axis — it tests `$(dirname "$ATLAS_GATE_HOOK")/git-push-guard.sh` so `GATE_BASENAMES` is populated — and is
silent about cards, so an agent who mirrors the hook set without also being told to mirror the cards spends the round
chasing a phantom guard regression. That happened on this PR; the brief's "mirror the cards" line is the only reason
it was caught. Re-check: copy `.claude/hooks/**` into a scratch tree with no `*/AGENT_README.md` beside it and run the
suite — the row above must be the ONLY red one, and must go green when the cards are added.
NOT A PRODUCTION HOLE, stated so nobody reads it as one: failing open with no cards to enforce against is that
guard's deliberate behaviour, and in the repo the cards always exist. This is harness siting, not a live gap in
dispatch gating.

**The gate refuses its own maintenance, and until 2026-08-16 the documented escape could not be typed.**
`ansible-gate-guard.sh` denies every Edit, Write and Bash write to `.claude/hooks/**` — correct, and the reason
the layer holds. The problem is the remedy. The README's own scoping example,
`printf '%s\n' .claude/hooks/ansible-gate-guard.sh > .claude/.ansible-gate-confirmed`, is itself DENIED (measured
on `67396749` and on the current guard): the guard reads gate paths out of a command's CONTENT, so naming the file
you want to authorise is a gate act. That left only `touch .claude/.ansible-gate-confirmed`, a four-hour bypass of
the WHOLE layer, so "edit one guard" became "switch off every guard" — the same false-denial-forces-a-global-bypass
shape #925 was reverted for, aimed at the guard itself. It cost a full round of work before anyone tried a shorter
fragment. Matching is SUBSTRING, so a STEM (`ansible-gate-guard`) scopes identically and contains no gate path;
the README example is now stems and is verified executable. STILL OPEN: nothing makes the guard's own maintenance
a first-class path, and nothing tests that the README's bypass instructions can be run. Re-check by feeding each
fenced command in the bypass section to the hook on stdin and asserting it is not denied.

**4 write shapes still reach the gate layer and are ALLOWED**, measured on `67396749` and on the landed guard
(2026-08-16). Each needs a real lexer rather than a token walk, which is what made #935 unmergeable:
  `echo x >| <path>` — `split_segments` splits on `|`, so the redirect operand lands in the next segment.
  `git stash push -u` with NO pathspec — the `stash` arm fires only on a named operand.
  `awk '{print > "<gate file>"}' f` — the target is a string literal inside the program text.
  `bash -c 'sed -i … <gate file>'` — `WRITE_RE` anchors verbs to `(^|[[:space:]])`, so a verb abutting a quote is
    invisible. Pre-existing and conceded in the guard's own header.
CORRECTION, and it changes the trade: an earlier revision of this entry said all four were ALLOW on `1d098006`
too, "so there is nothing to port". Three of them DENY there — `>|` (both the gate and deployed spellings), the
awk form, and the quote-abutting form — each naming the exact gate path in its refusal. Only pathspec-less
`git stash push -u` allows on both. So #935's lexer bought three real shapes and dropping them is a real cost,
not a free simplification. Re-check with a synthetic Bash payload per shape against
`git show 1d098006:.claude/hooks/ansible-gate-guard.sh`.

**16 read-only shapes are now ALLOWED that main denied**, beyond the 8 the suite names, and they are the price of
the reader classes. Every one is a READ of a gate or deployed path in a segment that happens to carry a write verb
or a redirect elsewhere: `head`/`wc`/`md5sum`/`diff`/`stat`/`readlink`/`cat`/`jq` of a gate or deployed file with
the output redirected to `/tmp`, `grep -c rm <gate file>`, `cp -r .claude/hooks /tmp/backup`,
`install <gate file> /tmp/x`, `git show HEAD:<gate file> > /tmp/x`, `git grep rm -- <gate file>`,
`git ls-files --stage .claude/hooks`, `git blame <gate file> > /tmp/b`, and `git checkout -- .claude/hooks`.
They are listed as loosenings rather than folded into the regression corpus, because burying them there is exactly
how #935 reported a clean zero three times.

**The `cp A B` row in the guard suite has gone decorative.** `test/run-advisory-guards-smoke.sh`, section "a
command is a SET of acts": the row asserts "two guarded tokens in ONE segment" and still passes, but `dest_last`
makes the source operand a read, so it now carries ONE finding and would no longer go RED if the decide-once loop
were reverted to `refuse`-on-first. Measured through the bypass announcement, the only place the finding SET is
observable: both guards named on `67396749`, one on the landed guard. Needs a single-segment, two-destination
re-spelling (`tee <gate1> <gate2> < /tmp/x` is the shape) — not attempted here, because the landing round was
scoped to the measured patch.

**The gate's corpora and harness are still not in the repo.** The salvage landed with 3 corpora (31 target rows,
83 regression rows, 16 class-reach rows), a per-rule mutation battery and a main-vs-`1d098006` inversion control,
all built in an agent scratch directory and all thrown away. The 5,005-cell sweep from the #935 rounds went the
same way. Nothing here is reproducible by a later reviewer without rebuilding it, which is why each round has had
to re-measure from scratch — and why measuring against a previous head instead of against main went unnoticed for
three rounds.

**A recorded do-not-merge DECISION on PR #935 went unread through four review rounds** (2026-08-15). The entry
titled "#935 ansible-gate — BLOCKED. Do not merge and do not patch-round it." landed on main in `f235e79d`
(2026-08-14), written because THREE rounds had already each claimed "loosened writes = 0" and each been falsified
by hand-probing outside the template sweep — and it PREDICTED that shape in those words. FOUR further patch-rounds
then ran against it and reproduced it exactly, with the entry byte-identical on main and on the PR head. The split
is three-before / four-after: #935's commit dates cannot establish it (the branch was rebased, so they all read
08-15/16); the evidence is the entry's landing commit plus the round count in the session's working memory.
WHY IT SURVIVED: every dispatch asked agents to verify CLAIMS — do the numbers reproduce, do
the citations resolve, is the guard real — and nobody was ever asked whether a DECISION already existed. Agents read
this file repeatedly and correctly answered the question they were given. The discriminator: a claim is falsifiable
by measurement, a decision is only reversible by a human, so finding one means STOP and escalate, never "measure
harder until it goes away". PRE-FLIGHT, now step 0 of `SKILL.md` MERGE_GATE SEQUENCE: before reviewing a PR's code,
grep this file for its number and for the decision vocabulary (`BLOCKED`, `do not merge`, `do not patch-round`,
`superseded`, `rejected`) — one command, run before a review round rather than after the fourth one to miss it. FIRST
occurrence, recorded here so a second is recognisable; a second sends it to `LESSONS.md`. Re-check:
`grep -n -e '#935' -e 'do not merge' docs/BACKLOG.md` returns the blocking entry today, and a review dispatch either
carries that grep as its first step or it does not.

**The zero-loosened SERIES for #935 and its salvage — the figures that lived only in `STATE.md`.** Each round
measured its own corpus honestly and each zero was falsified by the next, bigger corpus (2026-08-15/16): 83-row
corpus -> claimed 0, actually **14** loosened shapes · 181-row -> found those 14, missed 46 · 342-row -> **60**
(detailed in the CORRECTION entry above) · 968-row -> **88** write shapes, 52 of them landing in `/opt` or `/etc`.
**Each ~3x corpus finds ~2-6x more. NOT converging.** Separately, #935 measured "loosened = 0" against its own
PREVIOUS HEAD — true each time — while against MAIN the branch opened **41 shapes, 32 of them executing a real
write**, sandbox-proved; the sweep that found them was agent-scratch and is not in the repo, so its row count is
not recorded. Every zero claimed on this work — at 83 rows, at 342 rows, and the HARD_STOP zero at 498 rows — was
later falsified by a larger corpus. Recorded here because 41, 32 and 14 existed nowhere but `STATE.md`, which is
gitignored and wiped at every epic boundary, while still being cited in briefs as settled. ONE COPY OF 41 IS ALREADY
LOOSE and it is the copy a guard author meets first: `.claude/hooks/README.md` says "which is what #935 did, drifting
41 shapes" — no corpus, no baseline, exactly the bare figure this entry legislates against. Left as written on
purpose: that file is gate layer and the guard refuses writes to it, so the fix is this entry carrying the
provenance the sentence lacks. Re-check: the corpora themselves were thrown away (see "The gate's corpora and
harness are still not in the repo"), so what these numbers buy is the standing rule — any future "loosened = 0" must
name its corpus SIZE and the baseline it was measured against, and a claim carrying neither is not evidence.

**Dependency debt — the 10.0.8 pin WAS the NU1903 fix and has become the NU1903 exposure.** PR #703 (`eb81d03b`)
pinned `System.Security.Cryptography.Xml` to 10.0.8 specifically to clear NU1903, and it pinned exactly the SIX
TEST projects. It did NOT touch MacroSubstrate: that project's 10.0.8 predates #703 and arrived with #604
(`869d9054`). Recorded because "#703 pinned it" invites the fix to be scoped to #703's files, which is precisely
the one PRODUCTION project it would miss. Corrected 2026-08-15.
Five advisories now stand against that exact version, all `[10.0.0, 10.0.9]`, all HIGH: GHSA-g8r8-53c2-pm3f, GHSA-8q5v-6pqq-x66h, GHSA-23rf-6693-g89p,
GHSA-cvvh-rhrc-wg4q, GHSA-mmjf-rqrv-855v. (The three the pin DID clear are `[10.0.0, 10.0.5]`: GHSA-37gx-xxp4-5rgx,
GHSA-w3x6-4m5h-cxqf, GHSA-6588-8gv4-xfgh.) Fixed version is **10.0.10**, and it is already in-tree —
`Reports.Hosting` and `Reports.Substrate` sit at 10.0.10 — so the bump target is known-good.
Seven projects still pin 10.0.8, only ONE of them production: `MacroSubstrate/src/MacroSubstrate/MacroSubstrate.csproj`
(the other six are SecMaster / FinnhubCollector / AlphaVantageCollector unit+integration test projects).
Sixth advisory, unchanged and test-only: `SQLitePCLRaw.lib.e_sqlite3` 2.1.11 (GHSA-2m69-gcr7-jv3q, HIGH, range
`(, 2.1.11]` — upper bound INCLUSIVE, so 2.1.11 is affected). SQLite is the unit-test DB provider, prod is
TimescaleDB; transitive only, no `.csproj` names it.
There is **no `Directory.Packages.props`** — the version is restated in nine files independently, which is why #703's
fix rotted unevenly and why Reports could move while the rest did not. Central package management is the durable fix;
bumping seven files is the cheap one.
Re-check: `curl -s --compressed https://api.nuget.org/v3/vulnerabilities/index.json` then the base+update pages
(`--compressed` is required; without it the response is gzip and unreadable). Measured 2026-08-15.
NasdaqCollector's gRPC-Swagger chain is old but clean for CVE-2026-49451, and Nasdaq is DISABLED in prod anyway.

**FinnhubCollector's quote-staleness gauges are bounded by the DATA, not by the code.** Two series per
active Quote symbol (`finnhub_quote_collection_staleness_seconds` + `finnhub_quote_staleness_origin_durable`,
both keyed `symbol`), 36 in prod today against CLAUDE.md's <100 bounded-cardinality rule. Nothing enforces it:
`POST /api/admin/series` -> `SeriesManagementService.cs:54` sets `IsActive = true` UNCONDITIONALLY, with no
ceiling and no warning, and the MCP `add_series` tool exposes that path. The consequence that matters is not
Prometheus load: `FinnhubCollectorQuoteCollectionStalled` pages per symbol, so a universe grown past what anyone
watches degrades it into chronic noise and it gets MUTED — which returns coverage to zero by a second route,
after PR #975 closed the first. Fix: refuse or loudly warn past a configured ceiling at the add boundary, and
add the ceiling to D-2 as a scaling PRECOND with that guard (a PRECOND with no guard is the decorative-guard
antipattern, which is why #975 did not add one). Re-check:
`sudo nerdctl exec timescaledb psql -U ai_inference -d atlas_data -c "SELECT count(*) FROM finnhub_series WHERE is_active AND series_type = 'Quote';"`
(18 on 2026-08-17; the column is `series_type` and holds the enum NAME, not an ordinal) — if it has grown past ~50, this is due.

**The stamps table's single-writer invariant is one negative test plus convention.** `finnhub_quote_collection_stamps`
is the staleness ORIGIN and D-2 INV stamps-single-writer requires the collection loop to be its only writer — but
`UpsertQuoteCollectionStampsAsync` sits on the shared `IFinnhubRepository` that `SeriesManagementService` already
injects for other reasons, and `FinnhubDbContext.cs:14` exposes a public `DbSet<QuoteCollectionStamp>` that
bypasses the repository entirely. What actually holds the line is one test
(`SeriesManagementServiceTests.TriggerCollectionAsync_DoesNotWriteTheDurableStalenessOrigin`) that names one
caller: a second writer added anywhere else compiles, passes, and silently disarms the dead-man across the next
restart. Structural fix is a refactor, not a patch: a narrow writer interface the cycle alone takes, and a
non-public `DbSet`. Re-check:
`grep -rn "QuoteCollectionStamps\|UpsertQuoteCollectionStampsAsync" FinnhubCollector/src --include=*.cs` — 2026-08-17
it returns the DbSet declaration, the repository implementation, the interface, and exactly one caller
(`QuoteCollectionWorker.PersistCollectionStampsAsync`). A fifth non-test hit means a second writer exists.

**The staleness stamp measures a successful FETCH, not an advancing quote — needs a design call, not a reflex fix.**
`QuoteCollectionWorker.cs:114-120` upserts and stamps on any non-null quote and never consults `quote.Timestamp`.
A halted or delisted symbol whose upstream keeps serving a frozen non-zero `t` therefore advances the stamp every
cycle with no exception raised: staleness stays ~60s, the durable gauge stays 1, no error counter moves, and the
matrix takes a price frozen at the halt date. Every leg of both PR #975 rules is false throughout. The reflex fix
(stamp only when `t` advances) is WRONG as stated: a closed market legitimately freezes `t` across a weekend and a
holiday, which is why the sibling collectors' dead-men use 3-4 DAY windows while this one uses 6h. Any fix needs a
market-calendar-aware window or a separate slower gauge, so it is a design decision. Re-check:
`sudo nerdctl exec timescaledb psql -U ai_inference -d atlas_data -c "SELECT symbol, max(timestamp) FROM finnhub_quotes GROUP BY symbol ORDER BY 2;"`
— a symbol whose max(timestamp) is days old while its row in `finnhub_quote_collection_stamps` is minutes old is
this defect, live. That table does NOT exist in prod until PR #975's migration deploys, so a `relation does not
exist` here means the fix has not shipped yet, not that the check passed.

**A failed Prometheus reload is a GREEN deploy with the old ruleset still evaluating.** `deploy.yml:649` runs
`nerdctl exec prometheus kill -HUP 1` with `failed_when: false`, and nothing afterwards asserts the rules actually
landed. Two failure modes, neither of which can fail the play: the HUP does not land at all, or it lands and
Prometheus REJECTS the file — in both cases the rule sits on disk, ansible reports success, and the PREVIOUS rule
set keeps evaluating. NOT theoretical: the 2026-08-17 deploy shipped THREE rules through this task
(`FinnhubCollectorQuoteCollectionStalled`, `FinnhubCollectorQuoteErrorsSustained`,
`FinnhubCollectorQuoteSymbolCoverageDropped`), and because the playbook cannot tell a landed reload from a silently
failed one, all three had to be verified against the Prometheus API BY HAND before the deploy could be called done.
A silently-failed reload leaves a service believed to be watched and watched by nothing — the exact state the 16-day
stall was found in. Fix: a post-reload assertion task that greps `/api/v1/rules` for the rule names the run just
copied. Re-check, after any `--tags monitoring` run — every rule in `deployment/artifacts/monitoring/alerts/*.yml`
must appear:
`sudo nerdctl exec prometheus wget -qO- http://localhost:9090/api/v1/rules | python3 -c "import json,sys; print(sorted(r['name'] for g in json.load(sys.stdin)['data']['groups'] for r in g['rules']))"`
It must be `nerdctl exec`: prometheus publishes NO host port, so a host-side `curl localhost:9090` exits 7 and reads
as a failed reload rather than as an unreachable probe. Verified 2026-08-17 — 86 rules, the three PR #975 rules
among them.

**A Postgres lock wait inside `MigrateAsync` defeats the 3-minute retry budget — SEVEN services.**
`MigrateWithRetryAsync` (`Events/src/Events.EntityFrameworkCore/DatabaseMigrationExtensions.cs`) retries on
EXCEPTIONS, and a lock wait is not one: `MigrateAsync` blocks indefinitely inside the DDL transaction, so the retry
loop never engages and the budget never starts. It presents as the `Applying database migrations...` startup
Warning followed by silence, with no timeout to end it — no `lock_timeout` or `statement_timeout` is set on that
connection anywhere. Callers: `Program.cs` of ThresholdEngine, FinnhubCollector, SecMaster, FredCollector,
OfrCollector, SentinelCollector, CalendarService (7, verified 2026-08-17). Pre-existing, found during PR #975's
migration review. Fix: set `lock_timeout` on the migration connection so a wait becomes a retryable exception.
Re-check: `grep -rn "lock_timeout\|statement_timeout" Events/src/Events.EntityFrameworkCore/ */src/Program.cs` —
zero hits today.

**`__EFMigrationsHistory` is one shared table for every ATLAS service in `atlas_data`.** 58 rows, measured
2026-08-17 (`SELECT count(*) FROM "__EFMigrationsHistory";`). Safe TODAY — EF filters by the migrations assembly
and the IDs are timestamp-prefixed, so collisions need two services to generate the same `yyyyMMddHHmmss_Name` —
but it is a namespace the services compete in rather than an isolation boundary, and nothing enforces the naming
that keeps them apart. Recorded as a known shape, not an action: separate schemas per service would be the durable
answer and it is a migration of the migration table. Re-check:
`sudo nerdctl exec timescaledb psql -U ai_inference -d atlas_data -c "SELECT \"ProductVersion\", count(*) FROM \"__EFMigrationsHistory\" GROUP BY 1;"`
— a duplicate `MigrationId` is the failure mode, and it would surface as a service silently skipping a migration.

**`finnhub_quote_staleness_origin_durable` LATCHES at 1 and nothing ever lowers it.** Measured 2026-08-17 in
`FinnhubCollector/src/Telemetry/FinnhubMeter.cs`: `Durable` is set true in exactly two places (the accepted seed in
`SeedLastQuoteCollected`, and `MarkQuoteOriginsDurable`), `MarkQuoteCollected` CARRIES it forward through
`current with { Ticks = ..., Stamped = true }`, `TrackActiveSymbols` re-adds with `GetOrAdd` so an already-tracked
symbol keeps whatever it had, and no path sets it false on an existing entry. So a symbol armed at boot by an
accepted seed whose stamp write then fails for ten days publishes `durable=1` for the whole process life, and the
dead-man's third leg — which exists to report precisely that state — stays silent through it.
`CollectAllQuotesAsync_StampWriteFails_LeavesTheDeadManDisarmed` exercises only the NEVER-armed half. NOT blocking,
and the mitigations are why: `FinnhubCollectorQuoteErrorsSustained` fires at 1h on
`operation="persist_quote_stamps"`, and the next restart seeds from the now-stale row so the 6h staleness leg pages.
The fix is not a one-liner — clearing `Durable` on a failed write needs the failure attributed PER SYMBOL, and
`PersistCollectionStampsAsync` currently fails the batch. Re-check: arm a symbol via an accepted seed, make its
stamp write fail, then read `finnhub_quote_staleness_origin_durable{symbol="<it>"}` — it should be 0 and is 1.

**The coverage rule's 18 must be re-pinned in two files plus a monitoring deploy, and until it is the alert fires
continuously.** `FinnhubCollectorQuoteSymbolCoverageDropped` is `count(finnhub_quote_collection_staleness_seconds) < 18`
(`deployment/artifacts/monitoring/alerts/collectors-deadman.yml:145`) and the same number is pinned as
`ProdActiveQuoteSeries` in `FinnhubCollector/tests/Workers/QuoteCollectionWorkerTests.cs:160`. The hardcoding is
deliberate — a self-referential form goes blind to a one-a-week drip — but the cost is unpriced: after ONE
legitimate deactivation the rule fires every 30m forever until someone edits both files AND redeploys monitoring
(`--tags monitoring --skip-tags always`, which does not reload Grafana or assert the rules landed — see the
Prometheus-reload entry above). A permanently-firing alert is a muted alert, which returns coverage to zero by the
same route the 16-day stall took. Cheapest de-risk: source the number from one place both consumers read, so
re-pinning is a single edit. Re-check:
`grep -n "< 18" deployment/artifacts/monitoring/alerts/collectors-deadman.yml && grep -n "ProdActiveQuoteSeries = " FinnhubCollector/tests/Workers/QuoteCollectionWorkerTests.cs`
— two hits, two files, both must move together.

**That coverage rule is ONE-SIDED: it sees the universe shrink and cannot see it grow, which is how the constant
rots.** `count(...) < 18` fires on 17 and is silent on 19. Nothing anywhere compares the published series count
against the DB's active-Quote count, so a symbol added and never reflected in the rule leaves a constant that is
quietly wrong in the direction that WEAKENS it: at a true universe of 25, coverage can fall to 18 — seven symbols
dark — with every leg of every rule false. Measured 2026-08-17: the DB says 18 and the rule says 18, so it is
correct today and nothing would tell us when it stops being. Fix is the same one-source-of-truth as the entry above,
or a rule that alerts on ANY divergence rather than on a floor. Re-check:
`sudo nerdctl exec timescaledb psql -U ai_inference -d atlas_data -t -c "SELECT count(*) FROM finnhub_series WHERE is_active AND series_type = 'Quote';"`
against the `< 18` in `collectors-deadman.yml` — a DB count ABOVE the constant is this defect, live, and silent.

**A symbol added at runtime inherits the PROCESS-START origin, so its first staleness reading is the container's
age.** `TrackActiveSymbols` does `GetOrAdd(symbol, ProcessStartOrigin)`, and that origin is fixed once per process,
so a symbol added two minutes into a long-lived container publishes staleness measured from the container's start,
not from its own. Measured 2026-08-17: `finnhub-collector` was created 2026-07-31T20:38:51Z and is still up, so a
symbol added now would publish ~1.45 million seconds — 67x the 6h threshold — on its first scrape. The DIRECTION is
safe (it over-reports, never under-reports) and it self-corrects on the first cycle that collects the symbol, well
inside the 15m dwell; PR #975's move from `DateTime.UtcNow` at type-init to the real process start made the number
larger without changing that. What it costs is a per-symbol value that lies to whoever reads the gauge by symbol —
which the alert's own runbook instructs — and a symbol that never collects (permanent 403) shows an age that
implies a stall far older than the symbol. Whether a runtime-added symbol should instead start at `UtcNow` is a
design question: it trades this for a symbol that is genuinely uncollectable from birth taking 6h longer to page.
Re-check: `sudo nerdctl container inspect finnhub-collector --format '{{.Created}}'` against
`finnhub_quote_collection_staleness_seconds{symbol="<a symbol added since that time and not yet collected>"}` — the
gauge reports the container's age, not the symbol's.

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

**Scoped-deploy collateral is CONDITIONAL — record what actually moved, do not derive a rule.** Two runs on
2026-08-15: the `secmaster` scoped deploy ALSO recreated `llama-cpu-rag` and `llama-cpu-embed`; the
`sentinel-collector` one recreated nothing else. Two runs is not enough to say which services drag which
neighbours, and a general rule written off this sample would be wrong in one direction or the other — so no rule
goes in CLAUDE.md. The operative practice is the one that survives either explanation: after any scoped deploy,
enumerate what actually restarted (`nerdctl container inspect` — NOT bare `inspect`, which resolves the IMAGE and
hands back the BUILD time) and record it with the run. If a third and fourth observation agree, that is when a rule
is earned.
