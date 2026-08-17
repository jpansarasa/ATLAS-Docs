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
deliberately NOT changed in the PR that recorded this:
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

**55 `raw_content` rows were abandoned by a circuit-breaker classification gap — and the "438 orphans" figure that
surfaced them is a mixed predicate.** Corrected 2026-08-15; quote the 55, not the 438.
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
explicit `rawContentIds` array. No sweeper, no retry service. Fix is to classify `BrokenCircuitException` as
transient; the 55 then still need a one-off reprocess call, because nothing re-queues them retroactively.
**THE 55 ARE ON A DELETION CLOCK, so a later re-check will return 0 and must not read that as fabrication.**
`StaleContentPrunerService` runs its passes once per host start and pass 1 calls
`repo.DeleteChildlessOlderThanAsync(cutoff, ...)` (`StaleContentPrunerService.cs:83`, pass 2 at `:100`) with
`cutoff = UtcNow - MaxArticleAgeDays` (30 days, the
banner at `:73` says "full 30-day raw_content retention"). These 55 have no extraction children, so they qualify:
collected 2026-07-19 -> 2026-07-24, they become deletable **2026-08-18 -> 2026-08-23** and vanish at the first
SentinelCollector restart after that. Re-check BEFORE that window against the `processing_error` predicate above;
after it, an empty result means pruned, not absent. The classification fix is still worth landing — it is the
recurrence that matters, not these 55 rows.

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
