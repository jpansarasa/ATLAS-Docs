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
128 `gemini_fallback` and 1 `llm_candidate_exact` row, which with Rule 1's 1,421 sum to an ALL-METHODS total of
2,099 for that window (identical under `extracted_at` and `re_extracted_at` windows). The window is what makes that
number move: the same measurement re-run 2026-08-15T15:00Z read 1,046 all-methods (726 / 256 / 63 / 1) — the RATIO
and the per-method COMPOSITION reproduce, the absolutes do not, so cite the composition.
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
(`EntityResolutionPrepass.cs:404`), Rule 2.5's paid-Gemini leg (`DeterministicResolver.cs:560`, D-6) and its V1
mirror (`GeminiSymbolFallbackService.cs:85`, D-12) — while the LLM-extracted `SubjectEntity` reaches
`DeterministicResolver` through Rule 1 (`:60`) and Rule 2 (`:96`, raw `SubjectEntity` straight to hybrid resolve),
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

**Quarantining a SecMaster instrument row retires its SYMBOL, and 82 of them are real tradeable tickers.**
`idx_instruments_symbol` is a FULL (non-partial) unique index, so a soft-deleted row owns its symbol exactly as
hard as an active one and no row can ever take that symbol again. Migrations `QuarantineGeminiJunkInstruments`
and `QuarantineGeminiEquityEtfJunk` (2026-07-18) soft-deleted 91 rows to retire a junk NER-surface NAME —
`HON`="TD Cowen", `MELI`="Amy Legate-Wolfe", `AEM`="Toronto Stock Exchange", `ANGX`="Minions & Monsters" — and
the second migration's own header says every one of its 82 symbols "is a real, tradeable ticker". So the catalog
is now permanently unable to hold Honeywell, MercadoLibre, Agnico Eagle, Diageo, AB InBev or Mesoblast, and every
news mention of one pays the full confirmation cascade (per D-1, up to the paid Gemini leg) and self-seeds
nothing. Measured 2026-08-14: `SELECT count(*) FROM instruments WHERE is_active=false` = 91, all but one
`discovery_source='GeminiFallback'`, all `updated_at` in the two migration transactions; no active row exists for
`HON` (only the foreign lines `HON.NE`/`HON.VI`/`HON.MX`/`HON.TO`), and none at all for MELI/AEM/MESO.
91 IS THE QUARANTINE COUNT, NOT THE TICKER COUNT, and this entry headlined 91 while its own body cited the
migration's 82. Re-measured 2026-08-15, `SELECT asset_class, count(*) FROM instruments WHERE is_active=false
GROUP BY asset_class` = Equity 74 / ETF 8 / fred_series 8 / Economic Indicator 1. So 9 of the 91 are not
tickers at all: 8 `asset_class='fred_series'` (`NAPMII`="Asian share markets", `CONCCONF`="US",
`WCSPOIL`="U.S. Strategic Petroleum Reserve", `MCRFPC1`="Justin Trudeau", …) plus 1 `'Economic Indicator'`
(`GSV.NE`="2025 full year GSV"). Those 9 are the D-4 macro-junk class — a hallucinated macro label, which is a
different defect from a retired equity ticker and wants a different disposition. Every count below is therefore
stated over the population it actually applies to, 82 or 91, never both at once.
A SECOND LOSS SURVIVES THE FIX, deliberately, and is recorded here because a known data loss with no entry and
no counter is how one becomes permanent. `CatalogService.cs:197-208` drops a quarantined discovery item with a
bare `continue` and no `results.Add`, so `EntityResolutionService.cs:850` `result.Results.FirstOrDefault()` is
null and a CompanyName candidate loses its ticker PROPOSAL — the surface goes into the confirm cascade with
nothing proposed. The drop is CORRECT and must stay: surfacing an inactive row would undo the quarantine, and
`AddAsync` has no reactivation branch, so the alternatives are both worse. But correct is not the same as free,
and this one is invisible — it is also reachable from the `search_catalog` MCP tool, i.e. outside entity
resolution entirely, where the item simply vanishes from an operator's catalog search with no indication the
symbol exists. Closing it means deciding the reactivation policy above; until then the loss is real and known.
Re-check: the `is_active=false` count above, plus
`sum(secmaster_entity_resolution_self_seed_total{result="quarantined_skip"})` — which
is a FLOOR on the wall-hits, NOT "the live rate" this entry first called it. That tag is emitted at ONE of four
self-seed skip paths (`EntityResolutionService.cs:1032`); silent are `:877` (unconfirmed), `:901`
(contextFactor<=0), `:1002` (EnableSelfSeed=false), and the `CatalogService.cs:197` drop above, which carries a
LogWarning but no metric at all. Left as a floor rather than closed by emitting at `CatalogService.cs:197`,
deliberately: that would still leave three silent paths, so the qualification is needed either way, whereas the
emission is a new untested signal in a PR whose thesis is the pre-insert read. Measured 2026-08-15, prod carries
three series — `idempotent_skip`=1492, `inserted`=67, `error`=22 (the 23505s this PR stops) — and NO
`quarantined_skip`, because the emitting code is unmerged. `EntityResolutionSelfSeed` is also absent from
`MetricWarmupHostedService`, so even after deploy an absent series cannot be read as zero. The tag VALUE is
pinned by `EntityResolutionServiceTests.should_record_the_quarantined_skip_result_tag_when_the_symbol_is_held_by_a_quarantined_row`;
without it this PromQL could be renamed or typo'd and ship green (the pre-fix proxy,
`{service_name="SecMaster"} |= "Self-seed persistence failed"`, is VARIABLE, not a fixed rate: 30 lines/24h with
ANGX 21 of them in the window ending 2026-08-14T12:00Z, 7 with ANGX 6 ending 2026-08-15T13:00Z, so a low
post-deploy reading is that variance and not a regression). NOT fixed by the PR that added this entry — that one
only stopped the guaranteed-to-fail INSERT and its ~3KB stack. The repair is a policy call and needs a migration: either reactivate with `name = symbol` and let the
D-2 enrichment fill-gaps re-name them,
or make the unique index partial on `is_active` and let a fresh row take the symbol. Reactivating as-is is the
one option that is definitely wrong: it reinstates the junk names the quarantine existed to remove.
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
A THIRD SITE carries the same predicate mismatch and is deliberately NOT fixed here. `RegistrationService.cs:309`
is another active-only pre-insert read — `GetBySymbolAsync`, whose `IsActive` filter is itself load-bearing for
the FRED-pollution recovery path, so it cannot simply be swapped — feeding `AddAsync` at `:391`. A collector
registering any of the 82 real tickers hits the same guaranteed 23505 the fix above stops on the self-seed path,
and `DuplicateInstrumentException` (`: Exception`, so it passes `RegisterWithRetryAsync`'s `DbUpdateException`-only
catch at `:216`) is caught by `catch (Exception ex)` at `RegistrationService.cs:135`. It does NOT escape: it is
surfaced as a failed registration response — `Success=false, Message="Registration failed:
DuplicateInstrumentException: …"` — which the collector re-logs as a Warning. Pre-existing, not a regression, and
out of scope of the PR that recorded it — the fix is the same reactivation policy call as the repair above, not a
second local patch. No counter covers the path (`InstrumentsCreated` bumps only after `AddAsync` returns), but the
`LogError` at `:137` is a working re-check: `{service_name="SecMaster"} |= "Failed to register"` reads 0 over 7d
(measured 2026-08-15T13:00Z; the same selector unfiltered carries 12,444 lines in that window, so the zero is a
real zero and not a dead query). It is NOT firing today — re-run that before treating it as a live burn.
Dead code at the same site, worth deleting
whenever that policy lands: `RegistrationService.cs:313` `if (!instrument.IsActive)` sits AFTER the
`IsActive`-filtered read, so its operator-facing "registering against INACTIVE instrument" warning is
UNREACHABLE. Its sibling at `:258` IS live and must not be removed with it — that one reaches the instrument
through `existingMapping.Instrument`, a different lookup with no active-only filter.

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
