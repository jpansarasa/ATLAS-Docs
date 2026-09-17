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

Ordering, every section: by IMPACT class first, newest measurement first within a class. The class is the wrong action a
reader takes if the entry is missing or false --
  A: production data correctness (a wrong value or series reaches ThresholdEngine or the matrix)
  B: silent loss or an observability blind spot
  C: cost ($, GPU, a paid API)
  D: tooling or a harness that fails toward success
  E: prose, citation or doc drift
Each section opens with its table of contents. An entry is claim + measurement (dated) + consequence + re-check; the
history of how it was found lives in git, not here. Triaged 2026-09-16 from 133 top-level entries as inventoried (bold leads and ### headings, before
splitting), closed and expired ones removed, the rest trimmed to what re-checks them; the 26 sub-points that had lost
their headings and the survivors of removed headings got their own, which is why the tables below index more rows
than that.

## KNOWN DEFECTS

Defects with a measurement that makes them re-checkable.

| impact | measured | status | entry |
|---|---|---|---|
| A | 2026-09-17 | OPEN | 107 of 141 active GeminiFallback instruments are named by the query surface, not a title |
| A | 2026-09-16 | OPEN | Corrected source_entity prompt puts the COUNTRY in the owner slot; catalog matches it |
| A | 2026-09-16 | AWAITING-DECISION | Observation identity is the entity, not the measurement: N datapoints collapse to one key |
| A | 2026-09-16 | OPEN | Apparent identity collisions are partly mis-resolutions; proxy 6.6% and rising |
| A | 2026-09-16 | AWAITING-DECISION | D-18 matrix damage bounded: 240,353 -> 1,111 -> 242 signature floor; disposition open |
| A | 2026-09-16 | OPEN | ReExtract watermark-only legs can lift a quarantine without a re-resolve (population 0) |
| A | 2026-09-16 | OPEN | Surface filter sits after Rule 2: 95.7% of attaching rows never meet it; seams unchosen |
| A | 2026-09-16 | OPEN | Three of six Sentinel-owned series dead since April, four ungauged (Challenger revived) |
| A | 2026-09-16 | OPEN | Production's CoD prompt carries two defects no labeller can work around |
| A | 2026-09-07 | OPEN | A DELTA AND A LEVEL ARE THE SAME ROW: numbers[] cannot express dropped 2% vs is 2% |
| A | 2026-08-15 | OPEN | Rule 1 slug substitution fixed (#969); open: INTC regression, 2 untested gaps, guards flag |
| B | 2026-09-17 | OPEN | Merged SecMaster PRs sat undeployed 10 days; D-13's deploy shipped them unannounced |
| B | 2026-09-16 | OPEN | Scoped secmaster deploy also recreates llama-cpu-rag and the shared llama-cpu-embed |
| B | 2026-09-16 | OPEN | NameAppearsInContext demands the catalog NAME verbatim in the context; good hits return NONE |
| B | 2026-09-16 | OPEN | Production CoD loses ~74 gold entities per run to its loop guard (repetition_penalty 1.1) |
| B | 2026-09-16 | OPEN | matrix_cells provenance columns are written on 0 rows; no cell traces to its observations |
| B | 2026-09-16 | OPEN | SecMasterDiscoveryTimeoutsElevated cannot fire: per-candidate deadline double-increments |
| B | 2026-09-16 | OPEN | No resolution-rate alert: SentinelLowResolutionRate retired 2026-09-16, replacement owed |
| B | 2026-09-16 | OPEN | ReExtract erasure (D-31 fixed): pre-2026-09-16 instrument_id readings understate; 3 traps |
| B | 2026-09-16 | OPEN | SentinelExtractionDead inhibits every sentinel warning (equal: service), undocumented |
| B | 2026-09-16 | OPEN | request-log guard gap (#886): DiagnosticContext re-registration has no INTENT tag or test |
| B | 2026-09-16 | OPEN | BrokenCircuit orphan tripwire: 3 rss rows dated 2026-09-06 after D-27, image unconfirmed |
| B | 2026-09-16 | OPEN | 10 of 16 SentinelCollector workers log the startup banner at Information (invisible) |
| B | 2026-09-16 | OPEN | Four Grafana rule groups expire out of Alertmanager between pushes: false resolves |
| B | 2026-09-16 | OPEN | 1,168 files under rss/2026/04/23 have no raw_content row: orphaned on write, unreclaimable |
| B | 2026-09-16 | OPEN | A raw_content row with any extracted_observations child is never pruned, at any retention |
| B | 2026-09-16 | OPEN | period is extracted only by two non-publishing v1 sources; every v2/DSL source is at zero |
| B | 2026-09-16 | AWAITING-DECISION | Three raw_content rows orphaned by the pre-D-27 build on 2026-09-06 await a reprocess call |
| B | 2026-09-13 | OPEN | No alert keys on CoD truncation: truncated_salvaged counted, dashboarded by nobody |
| B | 2026-09-13 | OPEN | GPU path has no per-stage CoD token series: VllmClient tags everything classifier_llm |
| B | 2026-09-13 | OPEN | News-signal feed narrowed under Gemma 4 (36% -> 25% :sig:): noise leaving, 1 id confusion |
| B | 2026-09-07 | OPEN | fp8 -> unquantized KV unmeasured on Gemma 4 @ 0.28.0 CoD path (Qwen +0.051 stale) |
| B | 2026-09-07 | OPEN | vLLM 0.28.0 fp8_e5m2 fault isolation table (CLOSED by e4m3; re-check owed on next release) |
| B | 2026-09-05 | OPEN | Qualitative dispatch catch still orphans validation-content rows on a breaker outage |
| B | 2026-09-05 | OPEN | ResolutionWorker catches HttpRequestException only: BrokenCircuit abandons batch silently |
| B | 2026-08-26 | OPEN | tsa-checkpoint has published nothing since 2026-02-07 while still extracting |
| B | 2026-08-17 | OPEN | Pattern publicationFrequencyDays is dead config: silently overwritten by Max(series freq) |
| B | 2026-08-15 | OPEN | Rule 1 outcome erased downstream (D-31 fixed): read Original*; input confidence constant |
| B | 2026-08-15 | OPEN | sentinel_chunk_extraction_dedup_ratio keeps SDK default buckets a [0,1] value cannot use |
| C | 2026-09-16 | OPEN | gemini-resolver saturates its 1500/day cap; intent says dozens/day (INTENT_FIDELITY) |
| C | 2026-09-16 | AWAITING-DECISION | Quarantined-ticker re-acquisition is an undecided policy: Gemini cost + un-alerted 23505 |
| D | 2026-09-16 | OPEN | Gemma 4 swap residuals: one-directional coordinate sweep, hermes parser, promtool fixtures |
| D | 2026-09-16 | OPEN | D-23 thin-draw gate cannot deny: Bind() appends to the Engines default (inert until wired) |
| D | 2026-09-16 | OPEN | Static-meter flake: two ExtractionProcessor test classes still outside SentinelMeterStatic |
| D | 2026-09-16 | OPEN | Merge gate: REST route reads only the PR number; foreign-repo merge allowed (row 2) |
| D | 2026-09-16 | AWAITING-DECISION | Merge gate: two cheaper fixes measured and declined; newline/depth-slip shapes recorded |
| D | 2026-09-16 | OPEN | ansible-gate-guard invents a write path: scope-line write and displaced read verb deny |
| D | 2026-09-16 | OPEN | Refused BLOCK leaves a prior APPROVE valid (write side); MCP merge_pull_request is ungated |
| D | 2026-09-16 | OPEN | Bash and Edit/Write paths do not apply the same rules; the guard header says they do |
| D | 2026-09-16 | OPEN | run-wiring-smoke.sh RED since 2026-08-08; the red is its own stale EXPECTED_WIRED list |
| D | 2026-09-16 | AWAITING-DECISION | Write-then-execute across two tool calls is invisible to a command-string guard (accepted limit) |
| D | 2026-09-07 | OPEN | Push gate keys to a TREE: two PRs green apart can land a red main (first occurrence) |
| D | 2026-09-07 | OPEN | Three unfixed edges on the coordinate sweep (docstring, allowlist advice, prefix) |
| D | 2026-09-07 | OPEN | PR #1035 review: nine scoped-out gate-tooling defects (a..i) still unfixed |
| D | 2026-09-07 | OPEN | GOLD DEFECT: 55% of entity_ticker_accuracy cases are labeller world-knowledge |
| D | 2026-09-07 | OPEN | Worktree agent cannot do gate-layer work: ansible-gate-guard project_dir ignores worktrees |
| D | 2026-09-04 | OPEN | ansible-gate-guard denies READS, RUNS and PROSE about a gate file, permits a python WRITE |
| D | 2026-09-04 | OPEN | check_staleness.py: two endpoints under one engine name collapse last-wins, verdict flips |
| D | 2026-08-17 | OPEN | whole_act_git: a git global option (-C, -c) hides clean -x and update-index from the guard |
| D | 2026-08-17 | OPEN | sudo and env set PFX_SKIP even when the wrapper span is abandoned (ansible-gate-guard) |
| D | 2026-08-17 | OPEN | A write verb abutting an opening quote is never walked at all (ansible-gate-guard) |
| D | 2026-08-17 | AWAITING-DECISION | Write-capable commands outside _WRITE_VERBS are not walked; closing it needs a decision |
| D | 2026-08-15 | OPEN | Human-placed bypass scoped to a guard's basename excludes that guard's tests |
| D | 2026-08-15 | OPEN | Merge gate: a cut character inside a nested sh -c string hides -R from every scan |
| D | 2026-08-15 | OPEN | Merge gate: a PR number piped via xargs leaves the span empty; fallback merges wrong PR |
| E | 2026-09-16 | OPEN | Three live sites still teach the retired MODEL_SIZE >= 30B floor, one in production code |
| E | 2026-09-16 | OPEN | docs/BACKLOG.md has no out-flow that runs: 3,849 -> 4,187 -> 6,470 lines in eleven days |
| E | 2026-09-16 | OPEN | Metric prefix inconsistency: sentinel_candidate_surface_* vs sentinelcollector_semantic_* |
| E | 2026-09-16 | OPEN | Two SecMaster comments still call a Finnhub 403 transient (permanent, arrives as NULL) |
| E | 2026-09-16 | OPEN | SentinelCollector card is 5.3x over its D-entry gate and its line count hides it |
| E | 2026-09-16 | AWAITING-DECISION | Six deployed directories have no card and sit outside the SERVICES roster (HARD_STOP gap) |

**107 OF THE 141 ACTIVE `GeminiFallback` INSTRUMENTS ARE NAMED BY THE QUERY SURFACE THAT FOUND THEM, NOT BY A TITLE.
THE REPAIR IS DECIDED AND WRITTEN AS MIGRATIONS (#1053), NOT YET DEPLOYED; GC'S NAME AND THE OBSERVATIONS REMAIN.**
Mechanism, per-class authority and every fetched value: `SecMaster/AGENT_README.md` D-14 and the headers of
`SecMaster/src/Data/Migrations/20260917011551_RepairGeminiFallbackSurfaceNames.cs` and
`20260917013629_DisposeGeminiFallbackFuturesRoots.cs`. `A824RE1A156NBEA` (FRED: national defense as a share of GDP)
is still named `Mark Rutte` in production until that deploy. Measured 2026-09-17 on `atlas_secmaster`.

DECIDED 2026-09-17. Descriptions are nulled on the pre-#874 rows only. The 28 post-#874 rows that carry a correct
title and a pre-#1031 rationale description (`EGO`, `UUP`, `IUSG`, ...) are left as they are. The observations already
attached under the junk names are re-resolved (story 5, being built).

WHAT THE MIGRATIONS DO. 82 FRED series and 21 US listings get their title and lose the rationale description. `DX`
and `KC` are quarantined: their tickers belong to Dynex Capital and Kingsoft Cloud. `CT` is curated to
`Cotton No. 2 Futures`. Every write is compare-and-swapped on the name recorded at authoring.

WHAT REMAINS.
- `GC` (named `gold`, 2,133 observations in the 30 days to 2026-09-16T23:12Z) is untouched and awaits the user's
  choice of name. The literal-name gates (CoVe full-name grounding, SecMaster NameAppearsInContext) would stop gold
  news attaching under an exchange-style name such as the CFTC's `GOLD - COMMODITY EXCHANGE INC.`.
- Story 5: re-resolving the attached observations. In the 30 days to 2026-09-16T23:12Z, 3,888
  `sentinel.extracted_observations` rows attached to 75 of the 107, and a rename re-resolves none of them.
  `/admin/reprocess` is not the route: it has no dry-run, it deletes Pending rows and it can reach Gemini.
- The executable bar: `SecMaster/scripts/junk-name-audit.sh` (PR #1054), which checks names against their title
  source rather than this entry's date split.
- Until the deploy, the resolution-regression probe row `defense spending | 3.5%` still resolves `A824RE1A156NBEA`
  (`SentinelCollector/scripts/resolution-regression/corpus.tsv`, KNOWN PROBE FAIL). Close this entry, and rewrite that
  comment, once the plan's AFTER measurement has run against the deployed migrations.

Re-check (psql is SELECT-only; `atlas_secmaster`). Deployed: `SELECT "MigrationId" FROM "__EFMigrationsHistory"
WHERE "MigrationId" LIKE '%GeminiFallback%';` -> 2 rows. Rows: `SELECT count(*), count(*) FILTER (WHERE name =
'Mark Rutte'), count(*) FILTER (WHERE description IS NOT NULL AND symbol NOT IN ('DB', 'EVR', 'NMR', 'RILY', 'PALL',
'GC')) FROM instruments WHERE discovery_source = 'GeminiFallback' AND is_active AND created_at < TIMESTAMPTZ
'2026-07-19';` -> `107 | 1 | 101` on 2026-09-17T01:52Z, `105 | 0 | 0` after the deploy. `DX` and `KC` leave the
active population. The five listings are excluded because the EDGAR classification backfill writes their SIC
description into the NULL at the same startup, and `GC` keeps its description until its name is decided.
Observations: the ids come from `SELECT string_agg(quote_literal(id::text), ',') FROM instruments WHERE
discovery_source = 'GeminiFallback' AND created_at < TIMESTAMPTZ '2026-07-19' AND (is_active OR symbol IN ('DX',
'KC'));` on `atlas_secmaster` (107 ids; DX and KC are the only rows of their symbols), then `SELECT count(*),
count(DISTINCT instrument_id) FROM sentinel.extracted_observations WHERE instrument_id IN (<ids>) AND extracted_at >=
TIMESTAMPTZ '2026-08-17 23:12Z' AND extracted_at < TIMESTAMPTZ '2026-09-16 23:12Z';` on `atlas_data` -> `3888 | 75`.
`instrument_id` can change after extraction, so a different figure is not by itself a refutation.

**THE CORRECTED `source_entity` PROMPT IS IN PRODUCTION, IT STOPPED THE BLANKING, AND ON ARTICLES THAT NAME
NO SERIES IT PUT THE COUNTRY IN ITS PLACE. 37 rows anchored on a country, 10 of them resolved to an
instrument, and all 10 are wrong on the FIGURE-to-INSTRUMENT fit** -- an inflation rate is not a quantity of
an equity ETF. [2026-09-06; catalog co-cause re-verified 2026-09-16 -- catalog SCOPE is now `SecMaster/AGENT_README.md` D-13, which retires the DISCONTINUED rows and leaves the country-named rows below in place] That is a judgement made by reading each row
against its article, not a measured label, and it is FOUR articles' worth of evidence (two Turkish macro, one Chinese
gold, one European budget, all naming no series): 10 wrong of 10 could be four articles' worth of bad luck, and it
needs a day of articles before the rate means anything. #1017's prompt reached prod 2026-09-06T10:26:43Z; window is
`extracted_at >= 2026-09-06 10:26:43Z`, snapshot 10:59Z: 72 observations, 8 documents. The country is the one answer
the prompt forbids -- `SentinelCollector/src/cod-prompts/cod_json_v1.txt:57-58` reads "the country is NOT the owner".

**THIS IS NOT AN ARGUMENT FOR A `country -> reject` RULE.** The entry "The candidate surface filter gates 4.3% of the
rows that attach instruments" measures the opposite half: country subjects ALSO produce defensible resolutions --
`Brazil` -> `EWZ` 446, `Germany` -> `DAX` 345, `China` -> `GXC` 190. That entry judges country -> instrument, this
one judges number -> instrument. The fix this points at is the prompt's contradictory bullet and the catalog naming
below (scope for the catalog as a whole landed as `SecMaster/AGENT_README.md` D-13; the country-named rows are outside
it), NEVER a reject rule that entry already refutes in three caveats.

THE MECHANISM IS A HYPOTHESIS, recorded as one. `cod_json_v1.txt:56-58` tells the model to emit the series "under
the name THE ARTICLE gives it" and that `""` is NOT the answer; `:60-63` says to emit `""` when the article "never
NAMES" the series. On a Turkish inflation story that names no series those point opposite ways, and the model
appears to take the third option `:58` forbids. The gold already contains this case: article 183's six payroll rows
sit on `US` (the macro-owner entry under MEASUREMENT DEBT), a careful human labeller reaching for the country on an
article that named no series; 16 of the gold's 22 open anchors are the same class (a series the article DESCRIBES
without NAMING), and closing them needs the rule the prompt currently states twice, differently.

`source_entity` AND `subject_entity` ARE THE SAME FIELD ON THIS PATH. `DslToMergedExtractionAdapter.cs:399` sets
`perRowSubject = sourceEntity` whenever the slot is non-blank; of 20,640 rows since 2026-09-01 where BOTH columns
are non-blank, ZERO differ. So a country in `source_entity` IS the string `DeterministicResolver` Rule 2 queries.
Re-check, and silence on the second column is the pass:
`SELECT count(*), count(*) FILTER (WHERE source_entity IS DISTINCT FROM subject_entity) FROM
sentinel.extracted_observations WHERE extracted_at >= TIMESTAMPTZ '2026-09-01' AND
coalesce(trim(source_entity),'')<>'' AND coalesce(trim(subject_entity),'')<>'';`

DOWNSTREAM. A country ENT is excluded from the Rule 1 shortlist by `NonInstrumentEntTypes`
(`DslToMergedExtractionAdapter.cs:108-125`, `"country"` at `:115`), so the row falls to Rule 2, which hands the RAW
`SubjectEntity` to hybrid resolve and consults NO surface filter. D-1 already counts that leg -- 7,184
instrument-attaching rows carrying a `gpe_country` subject, 3,060 landing on `U` (Unity Software) -- over ONE
31-day window (`extracted_at` [2026-07-15, 2026-08-15)) and as a FLOOR (exact-match sets only); both caveats are
mandatory. The rows DO reach Rule 2.5 and are refused at its gate before any paid call:
`sentinel_gemini_resolver_calls_total{outcome="surface_filtered"}` read 29 by 10:58Z against `secmaster_match` 6,
and none of the 10 attachments came from `gemini_fallback` -- the counter, not the result table, is what separates
reached-and-rejected from never-reached. What is NEW is the live table:

| figure | value | anchor | leg | attached to |
|---|---|---|---|---|
| Turkey annual inflation | 28.4 PCT | Turkey | `hybrid_subject` | `TUR` |
| Turkey annual inflation | 21.0 PCT | Turkey | `hybrid_subject` | `TUR` |
| Turkey national income | $2.2T | Turkey | `hybrid_subject` | `TUR` |
| Turkey export value | $450B | Turkey | `hybrid_subject` | `TUR` |
| Turkey GDP growth projection | 5.0 PCT | Turkey | `hybrid_subject` | `TUR` |
| unemployment rate [Turkish, a 2029 projection of "below 8%"] | 8.0 PCT | Turkey | `hybrid_subject_description` | `UNRATE` |
| China's gold purchases in 2023 | 225,000 | China | `hybrid_subject` | `NGDPXDCCNA` |
| China's gold purchases in July 2026 | 20,000 | China | `hybrid_subject` | `AIRYY` |
| UK defence spending target | 3.0 PCT | UK | `hybrid_subject_description` | `UKNGDP` |
| budget deficit projection for France and Germany | 5.5 PCT | Germany | `hybrid_subject` | `EWG` |

`UNRATE` and `UKNGDP` arrived on Rule 2b (`hybrid_subject_description`), the leg added to RESCUE macro series, which
off a country anchor reaches the wrong COUNTRY's series. The same France/Germany figure was emitted twice, once per
country, and only the `Germany` copy attached.

**THE CATALOG IS A CO-CAUSE, AND A PROMPT FIX WILL NOT REMOVE IT.** These are not fuzzy near-misses: SecMaster holds
instruments whose `name` is a bare country string, so a country anchor EXACT-matches one. `TUR` is named "Turkey",
`NGDPXDCCNA` "China", `UKNGDP` "UK", `EWG` "Germany" -- four for four on the wrong attachments above, all still
present 2026-09-16. Three instruments are named `UK` (`.LON`, `IMPUK`, `UKNGDP`) and eight `US`, so WHICH wins is a
ranking artifact; `UUP` (the Invesco DB US Dollar Index fund) is named "Italy", a catalog defect in its own right.
Fixing the prompt stops the anchor being produced; it does not stop the catalog answering to it, and any OTHER path
that reaches hybrid resolve with a country string still lands here. Re-check, `atlas_secmaster`:
`SELECT symbol, name FROM instruments WHERE name IN ('Turkey','China','UK','Germany','France','Spain','Italy','US');`

BLANK IS NOT SAFE EITHER. A blank slot falls back to the DOCUMENT-LEDE ENT (`DslToMergedExtractionAdapter.cs:399-401`):
on the five blank rows at 10:46Z that lede was `China` (x3), `Wall Street Breakfast` and `John Healey` -- a country, a
PUBLICATION and a PERSON, all excluded from ownership by the prompt (`cod_json_v1.txt:65-66`) -- yet four LATER blank
rows attached correctly to `LITE` off a single-company article. Blanking is safe when the document has ONE subject
and dangerous when it has several or none, the same condition that produces the country anchor; a positional
fallback cannot tell those apart, and that is the defect.

THE ISSUER ARM IS NOT A CLEAN CONTROL EITHER. At 10:46Z it was 12 rows / 9 attachments, all through Rule 1's
shortlist as `llm_candidate_hybrid`, all checked by hand and correct (`BA` x7, `AKR` x2). It then grew, and the growth
is not clean: three rows anchored on `European government bond yields` -- a named surface, so this entry's own rule
bins it as issuer -- attached to **`DGS10`, the US 10-Year Treasury**, the identical "right series kind, wrong
country" failure as `UNRATE`. So the narrow claim survives (an issuer-anchored row reaches Rule 1's shortlist and
resolves through the leg designed for it) and the wide one does NOT.

CONTAINMENT, partial. Every wrong row sits `review_status='Pending'` on a method NOT in `AutoApproveMethods`
(`ExtractionOptions.cs:436` -- only `ticker_in_quote`, `llm_candidate_exact` and, since D-32, `source_keyed`), but
`AutoApprovePolicy` default-accepts a held row after `ReviewGraceDays` = 3 (`ExtractionOptions.cs:449`, metered as
`GraceDefaultAccept`). The XML doc at `ExtractionOptions.cs:446` claims `extracted_observations` feed only the
qualitative digest and NOT the WS3 matrix -- the CODE's claim, unverified here and in tension with CLAUDE.md GIGO;
assume neither. Nothing here argues for reverting #1017 (its gain, `numbers_f1` 0.3446 -> 0.5152, never claimed
full compliance).

Any query here re-run after a deploy measures a DIFFERENT binary, and a changed figure is not by itself a refutation
of this entry: the figures above were taken on the container that carried the corrected PROMPT (a host-mount sync)
and NEITHER of the resolver fixes that merged after it (#1029, #1031), and the container was rebuilt 2026-09-16 18:04Z
(D-32), so the fixed window now spans two or more resolver binaries.

RE-CHECKS, all psql SELECT-only, window `extracted_at >= TIMESTAMPTZ '2026-09-06 10:26:43Z'`. Report
`count(DISTINCT raw_content_id)` and the per-document surfaces beside every row count: on the no-instrument-ENT side
(`candidate_symbols_json` NULL or empty) one 246-row `TSA` document (164489) turned 22-of-22 country at 10:59Z into a
row-weighted 24 of 272 at 17:32Z while it was still 3 of 5 documents (164424 `Turkey` 12, 164425 `Turkey` 10, 164507
`Italy` 2), so a row-weighted rate measures document SIZE, not model behaviour. Classify anchors with the FULL
country list below -- a 7-name shortlist binned `Germany`/`France`/`Spain`/`Italy` as named-issuer and reported 33/9
where the full list gives 37/10:
- anchor-class split, the headline: `SELECT CASE WHEN coalesce(trim(source_entity),'')='' THEN 'blank' WHEN
  source_entity IN ('United States','USA','US','U.S.','America','United Kingdom','UK','U.K.','Britain','China','Japan',
  'Germany','France','India','Russia','Brazil','Canada','Mexico','Italy','Spain','Australia','South Korea','Korea',
  'Saudi Arabia','UAE','United Arab Emirates','Israel','Iran','Turkey','Switzerland','Netherlands','Sweden','Singapore',
  'Hong Kong','Taiwan','Indonesia','Thailand','Vietnam','Nigeria','South Africa','Egypt','Argentina','Poland','Ireland',
  'Norway','Denmark','Finland','Belgium','Austria','Portugal','Greece','European Union','EU','Eurozone','Europe')
  THEN 'country' ELSE 'named-issuer' END AS anchor, count(*), count(*) FILTER (WHERE instrument_id IS NOT NULL),
  count(DISTINCT raw_content_id) FROM sentinel.extracted_observations
  WHERE extracted_at >= TIMESTAMPTZ '2026-09-06 10:26:43Z' GROUP BY 1;`
- the attachments: same `WHERE`, plus `AND instrument_id IS NOT NULL`, selecting `description, value, source_entity,
  resolution_method, "Symbol"`.
- has ReExtract reached the window: same `WHERE`, `count(*) FILTER (WHERE re_extracted_at IS NOT NULL)`. The table
  above read the LIVE columns, earned only because 0 of 72 rows had been re-extracted; non-zero means it is no
  longer reproducible (the sibling entry reads `OriginalInstrumentId`/`OriginalResolutionMethod` for this reason).
- Rule 2.5 arrivals: `sentinel_gemini_resolver_calls_total{outcome="surface_filtered"}` in Prometheus.

**AN OBSERVATION'S IDENTITY IS THE ENTITY MENTIONED, NOT THE MEASUREMENT TAKEN -- so N datapoints from one article
collapse onto ONE series key, and roughly three quarters of everything ever published is in such a group.**
[2026-08-26; group ratio re-measured 2026-09-16] The extractor is doing its job: an article legitimately carries 0..n
datapoints and it mines them all. The defect is downstream of that -- resolution answers "which instrument is this
about", the publish path uses that answer AS the series id, and ThresholdEngine keys its ObservationCache by SeriesId
keeping the newest write. So a series holds whichever of its n claimants landed last.

MEASURED 2026-08-26 on `sentinel.extracted_observations`, published rows only: 20,822 published observations, 2,727
instruments, 15,344 distinct descriptions; 15,397 of 20,822 (74%) sit in a (raw_content_id, instrument_id) group of
size > 1 (3,591 such groups against 5,425 sole claimants); 1,559 of 2,727 instruments (57%) carry MIXED UNITS. On
2026-09-16 the first query reads 5,966 of 14,476 groups with size > 1 (41%, was 40%). THE DURABLE CLAIMS ARE THE
RATIOS -- roughly three quarters of published rows in a collision, more than half of instruments carrying mixed
units, ~5.6 distinct descriptions per instrument; read a changed absolute as the corpus growing. Re-check both:
  `SELECT COUNT(*) FILTER (WHERE c>1), COUNT(*) FROM (SELECT raw_content_id, instrument_id,
     COUNT(*) c FROM sentinel.extracted_observations WHERE published_at IS NOT NULL
     AND instrument_id IS NOT NULL GROUP BY 1,2) g;`
  `SELECT COUNT(*) FILTER (WHERE u>1), COUNT(*) FROM (SELECT instrument_id, COUNT(DISTINCT unit) u
     FROM sentinel.extracted_observations WHERE published_at IS NOT NULL
     AND instrument_id IS NOT NULL GROUP BY 1) x;`

THE SHAPE, from a real published group. One Procter & Gamble article, four rows, all Symbol=PG:
  tariffs on American goods         25 PCT
  tariffs on American goods         50 PCT
  imported toilet paper from Canada 328,000,000 USD
  global tissue consumption         20 PCT
None of those is "the value of PG". They are facts MENTIONED NEAR Procter & Gamble, and TE computes on whichever
wrote last; the 5.6 descriptions-per-instrument ratio says this is the norm, not an outlier.

WHY IT SURFACED AS A CHALLENGER BUG. One Challenger release yields four datapoints -- headline job cuts 33,429, an
AI-attributed subset 10,970, a sector-and-YTD figure 149,023, and planned HIRES 107,500, which has the opposite sign.
All four resolve to the same instrument and similarity cannot separate them: 0.8196 / 0.8125 / 0.8507 / 0.7994, so
the WRONG answer scores highest, and `confidence` is a flat 0.85 across all four; no threshold keeps the headline and
drops the rest. Publishing planned hires as job cuts drives challenger-layoff-surge's `-(cuts - 30000)/30000` to
-2.58, a maximal false recession signal from a number meaning the opposite. Everywhere else the same collision is
silent: nothing alerts on a wrong value, only on a missing one.

LATENT ON THE `SENTINEL:NUM:` KEYS. Measured 2026-08-26: zero of TE's 71 loaded patterns reference a `SENTINEL:NUM:`
key, which is where 47,366 of 47,402 published observations landed over 30 days; the 3 patterns touching a Sentinel
key use the `SENTINEL:SECTOR:` prefix through `GetSeriesCount` / `GetSectorBreadth`
(`ThresholdEngine/src/Entities/PatternEvaluationContext.cs:312`, `:362`), which read `SeriesId` and `LatestDate` and
NEVER `Value`. The matrix path is keyed `{raw_content_id}:sig:{slug}` in `public.macro_observations`, so it does not
collapse across articles. The projector DOES evaluate `signalExpression` over the ObservationCache
(`ThresholdEngine/src/Workers/ObservationCellProjector.cs:810-821`), so "the cache is unread" is FALSE; the explanation
is that no loaded expression names the key. Re-check:
  `grep -rl 'SENTINEL:NUM:' ThresholdEngine/config/patterns --include='*.json' | wc -l`   # expect 0
The one worked example on a bare `OwnedSeries` key: `BDIY` publishes several datapoints a day under one key, so the
LAST write of the day wins; on the day measured that was the LOW (290), and `baltic-freight-recession` would compute
`bdiy < 700` TRUE and clamp its signal to **-3** -- but that pattern is `"enabled": false` and is the one repo pattern
file of 72 absent from TE's loaded set. Loaded gun, not fired; whoever flips `enabled: true` walks into the clamp.
  confirm `baltic-freight-recession` is absent from `list_patterns(enabled_only=false)`.
NO LONGER LATENT ON THE BARE `SentinelSeriesKey.OwnedSeries` KEYS. This entry once read "Sentinel publishes NEITHER
`CHALLENGER_JOB_CUTS` nor `TRUFLATION_CPI`"; D-32 (#1044-#1046) inverted that: Sentinel now publishes
`CHALLENGER_JOB_CUTS` from provenance (52,881 for 2026-08-31 confirmed end to end, #1046), and that key has 3 loaded
readers. The live question is whether the AI-subset, YTD and planned-hires datapoints of the same release still key
onto it -- the collision the -2.58 figure above depends on -- and whether `TRUFLATION_CPI` (1 reader) is published; re-derive
both against D-32 before trusting anything here about the Challenger feed.

STILL OPEN, AND IT IS A DESIGN QUESTION, NOT A BUG FIX. Identity probably needs to be (instrument, measurement)
rather than instrument alone, which is a schema change and touches D-18's ownership rules -- so it needs a human
decision, not a patch. An article-level guard refusing every claimant when n>1 ("ambiguity denies") was written,
tested and deliberately not committed: fail-closed and WRONG AT THIS SCALE, it would refuse ~3/4 of the pipeline to
prevent corruption that has already happened, and must be rewritten against whatever identity model wins. Also open:
whether #988 (resolver retries with the description) is net-negative, since it raises resolution success and admits
MORE rows into a broken identity model.

**A SHARE OF APPARENT IDENTITY COLLISIONS ARE MIS-RESOLUTIONS, NOT IDENTITY COLLAPSE — DIFFERENT
ENTITIES LANDING ON ONE INSTRUMENT ID. THE KEY CANNOT FIX THESE; THE RESOLVER IS THE LEVER.**
The entry above counts a `(raw_content_id, instrument_id)` group of size > 1 as a collision. Some of
those groups are not one entity measured n ways — they are n DIFFERENT things the resolver filed
under one instrument. Re-keying on (instrument, claim_kind, unit) leaves every one of them wrong,
and worse: it splits the wrong attribution across two "series" by unit, so it acquires structure.

SAMPLED 2026-08-26 by adversarial review of the remediation plan: **6 of 32 groups (~19%)**.
**That is a FLOOR** -- the sample was drawn from PUBLISHED rows carrying an instrument, so it excludes everything
unpublished and everything the resolver failed on outright.
THE MACHINE-CHECKABLE PROXY measures something narrower -- colliding groups whose members DISAGREE on `subject_entity`:
**394 of 5,966 (6.6%)** re-measured 2026-09-16, against 148 of 4,110 (3.60%) on 2026-09-05 and 144 of 3,598 (4.00%)
on 2026-08-26 -- the proxy nearly doubled in 11 days. It misses the case where the subject string is CONSTANT and the
resolution is still wrong (worked case, re-selected 2026-09-05: `raw_content_id=160553`, 15 published rows all
`subject_entity` "Germany", filed under `DAX` GLOBAL X DAX GERMANY ETF via `llm_candidate_exact`, one of them "Brent
oil closing price per barrel"; siblings `DEXINUS` "India" and `DEXCAUS` "Canada", both asset_class Currency, absorb
their countries' subjects), so the proxy is a floor beneath the 19%, not a refutation of it.
THE LEVER IS #988 (resolver retries with the description), NOT R2/S4's key -- and #988 RAISES resolution success,
admitting MORE rows into both this defect and the collision one, so never read a rising resolution rate as progress
here without splitting the two populations. Exact-duplicate rows are a small third confound inflating the same groups.
Re-check (SELECT-only):
  `SELECT count(*) FILTER (WHERE c>1 AND ds>1) AS multi_subject_groups,
     count(*) FILTER (WHERE c>1) AS colliding_groups
   FROM (SELECT raw_content_id, instrument_id, count(*) c,
           count(DISTINCT coalesce(nullif(btrim(subject_entity),''),'(null)')) ds
         FROM sentinel.extracted_observations
         WHERE published_at IS NOT NULL AND instrument_id IS NOT NULL GROUP BY 1,2) g;`
  # 2026-09-05 148 | 4110; 2026-09-16 394 | 5966
  and RE-SELECT an exhibit rather than trusting the one above, which can be re-extracted away too:
  `SELECT raw_content_id, instrument_id, subject_entity, count(*) c, count(DISTINCT unit) u
   FROM sentinel.extracted_observations
   WHERE published_at IS NOT NULL AND instrument_id IS NOT NULL
     AND subject_entity IN ('Mexico','Brazil','Japan','China','India','Germany','France','Canada','Australia')
   GROUP BY 1,2,3 HAVING count(*)>1 ORDER BY c DESC LIMIT 8;`
  then in `atlas_secmaster`: `SELECT id, symbol, name, asset_class FROM instruments WHERE id='<uuid>';`
THE RATIOS ARE THE DURABLE CLAIM, not the absolutes -- the pipeline keeps publishing. The ~19% sampled share is a
FLOOR and the 6.6% proxy is a floor beneath it; neither is an upper bound, and nothing here has measured how much of
the collision figure is really mis-resolution.

**S5 (`docs/proposals/extraction-identity-implementation.md` §5) is DONE, and the 240,353 bound DOES
tighten: D-18's mechanism can only reach 1,111 pre-fix cells, and 242 of them carry a
positively-identified corruption signature.** Measured 2026-08-26 on `public.matrix_cells` (D-18 fixed by
`3bae398a`, 2026-08-13 00:56 UTC); the coarse bound, the hard-data path and the 242 floor reproduce exactly on
2026-09-16. Split on `created_at`, never `evaluated_at`: 1,166 cells were evaluated FOR a pre-fix date but computed
after the fix on clean data.
MECHANISM, structural not statistical: D-18 corrupted the `ObservationCache`, and only the hard-data arm of
`ObservationCellProjector` (`EvaluateHardMagnitudeAsync`, `ThresholdEngine/src/Workers/ObservationCellProjector.cs:683-688`,
code comment "News groups NEVER use SignalExpression") reads it. A news-path cell cannot consume a polluted value.

| population | pre-fix cells | what it is |
|---|---|---|
| whole table | **240,353** | every cell that existed while the bug was live (coarse upper bound) |
| hard-data path | **5,929** | cells whose magnitude came from `signalExpression` + `ObservationCache` |
| reads a polluted mnemonic | **1,111** | of those, the patterns whose expression reads a series Sentinel actually published under |
| carries the +/-3 signature | **242** | of those, the identified floor: `abs(signal) = 3.0` EXACTLY (`SignalUtilities.ClampSignal`, `ThresholdEngine/src/Services/PatternEvaluationService.cs:344`), 9 `:fred` patterns, `created_at` 2026-06-25..2026-08-12 |

242 is cells MATCHING A SIGNATURE, not cells PROVEN damaged; true damage is in [242, 1111] on the mechanism's own path
and inside [0, 240353] for the table. It is attributable because the published values are read directly from
`sentinel.events`, not inferred from a before/after shift: DGS10 was published as 125000000000 against a real ~4.4-4.7%
yield, DTWEXBGS's maximum published value is `20230729`, a DATE, and `ust-10y-yield` clamps at +3 only when DGS10 >= 5.5,
so its clamp burst 2026-07-23..2026-08-12 is exactly the junk-DGS10 span; the same arithmetic-impossibility test passes
for the other eight.
READING RULES: detect with `= 3.0`, never `>= 2.9` (the raw UNCLAMPED mean fallback in `EvaluateHardMagnitudeAsync`
puts unclamped values above 2.9); the superseded reading's "242 rows" was a DIFFERENT set (overlap 88), not
corroboration; D-18's card originally named six polluted mnemonics but the measured set is 37 (the full list is now in
the card) and 5 of the 9 floor patterns read one outside the six, so a truncated list FALSELY refutes the mechanism;
no pattern config or evaluator code changed in `ThresholdEngine/**` between 2026-08-05 and 2026-08-20 (`3bae398a`
plus one devcontainer and one docs commit).
FLOOR OF A FLOOR: damage that never reached the clamp, cells since recomputed (26,205 rows show a recompute lag),
raw-mean fallback cells and collision loss are all invisible to this detector.
DO NOT RETRY: (1) mean-signal-shift attribution across the fix boundary -- the clean control `natural-gas-price`
moves +0.694 while `oil-price` moves +1.437, and every pattern it tested was news-path; (2) a before/after COUNT on
any clamp-like threshold -- post-fix eras are 1-3 batches for 10 of 12 patterns, no power. Re-deriving
240,353 -> 1,111 is settled work; the open question is disposition, a human decision
(`extraction-identity-implementation.md` section 6.2). No backfill or recompute recommendation follows from this entry.
Re-check (SELECT-only, `atlas_data`):
  the three nested populations —
  `SELECT count(*) FILTER (WHERE created_at < '2026-08-13') AS coarse_bound,
     count(*) FILTER (WHERE created_at < '2026-08-13'
       AND source_collector IN ('fred','ofr')) AS hard_data_path
   FROM public.matrix_cells;`
  the 242-cell floor —
  `SELECT pattern_id, signal, count(*), min(created_at)::date, max(created_at)::date
   FROM public.matrix_cells
   WHERE abs(signal) = 3.0 AND created_at < '2026-08-13'
     AND pattern_id IN ('obs:ust-10y-yield:fred','obs:oil-price:fred','obs:dxy-dollar-index:fred',
       'obs:cpi-headline-yoy:fred','obs:cpi-core-yoy:fred','obs:pce-headline-yoy:fred',
       'obs:industrial-production:fred','obs:nonfarm-payrolls:fred','obs:fed-funds-rate:fred')
   GROUP BY 1,2 ORDER BY 3 DESC;`
  the excluded both-era clampers, which must STAY non-zero post-fix for the exclusion to hold —
  `SELECT pattern_id, count(*) FILTER (WHERE created_at >= '2026-08-13') AS post_fix_clamps
   FROM public.matrix_cells WHERE abs(signal) = 3.0
     AND pattern_id IN ('obs:repo-liquidity-stress:fred','obs:um-consumer-sentiment:fred',
       'obs:gdp-real:fred') GROUP BY 1;`
  # 2026-09-16: coarse_bound 240353 | hard_data_path 5929; floor 242 (2026-06-25..2026-08-12);
  #   post-fix clamps gdp-real 11, repo-liquidity-stress 66, um-consumer-sentiment 11
  what Sentinel actually published under first-party keys (the direct evidence) -- the first-party side is DERIVED,
  never a frozen IN-list (the earlier hardcoded 15 mnemonics missed 22 of the 37 keys in the window). The series key
  is nested at `payload->'seriesCollected'->>'seriesId'`; `payload->>'seriesId'` matches NOTHING and reads as a
  clean zero --
  `WITH first_party AS (
     SELECT DISTINCT "SeriesId" AS k FROM public.fred_observations
     UNION SELECT DISTINCT "SeriesId" FROM public.series_configs
     UNION SELECT DISTINCT series_id FROM public.alphavantage_series
     UNION SELECT DISTINCT series_id FROM public.nasdaq_series
     UNION SELECT DISTINCT mnemonic FROM public.ofr_hfm_series
     UNION SELECT DISTINCT mnemonic FROM public.ofr_stfm_series
     UNION SELECT DISTINCT series_id FROM public.finnhub_series)
   SELECT e.payload->'seriesCollected'->>'seriesId' AS mnemonic, count(*),
     min((dp->>'value')::numeric) AS min_v, max((dp->>'value')::numeric) AS max_v,
     max(e.occurred_at)::date AS last_pub
   FROM sentinel.events e
   JOIN first_party fp ON fp.k = e.payload->'seriesCollected'->>'seriesId'
   CROSS JOIN LATERAL jsonb_array_elements(e.payload->'seriesCollected'->'dataPoints') dp
   GROUP BY 1 ORDER BY 2 DESC;`
Closes only on disposition. `last_pub` must stay <= 2026-08-12 for every mnemonic — a later date means
the D-18 egress guard has regressed and the corruption is live again.

**`ReExtractBackgroundService`'s three watermark-only legs can lift a quarantine without a re-resolve.** The null-`RawContent`,
zero-extractions and empty-`Description` legs (`SentinelCollector/src/Workers/ReExtractBackgroundService.cs:384`, `:456`,
`:601`) pass the row's OWN `InstrumentId`/`Symbol` back into `ApplyReExtraction` as a watermark-only stamp. On a QUARANTINED
row that still holds both, that input is `grounded`, so the `recovered` branch fires: `QuarantinedAt` is cleared and a
`[re-extract] recovered` note appended by a leg that did no resolution work, while the metric records `error`. Identical
on main (the predicate was logically the same before D-31, which changed only the retain side); found by the round-4
reviewer of PR 1042 (2026-09-16) and left unfixed there because the population is empty: 45,386 quarantined rows,
**0** holding an instrument. It sits against D-31's framing that clearing needs a positive signal. Re-check before
relying on it: `SELECT count(*) FROM sentinel.extracted_observations WHERE "QuarantinedAt" IS NOT NULL AND instrument_id
IS NOT NULL AND re_extracted_at IS NULL;` (2026-09-16: 0) -- non-zero means the `:601` leg (documented as the path for legacy
quarantines with cleared text) can now reach it. Fix shape: the watermark-only legs should stamp the watermark
without evaluating recovery, i.e. `recovered` must require a re-resolve to have happened, not merely a grounded input.

**The candidate surface filter gates 4.3% of the rows that attach instruments; 95.7% resolve without ever meeting
it.** `EntityResolutionPrepass.ApplySurfaceFilter` is unconditional (`const string mode = "enforce"`,
`EntityResolutionPrepass.cs:396`, no flag) and live, but POSITIONED after Rule 2: `Classify` runs on the NER-candidate
prepass and on the paid-Gemini legs (`DeterministicResolver.cs:802`, `GeminiSymbolFallbackService.cs:85`), while the
LLM-extracted `SubjectEntity` reaches Rule 1 and Rule 2 unfiltered. Measured over `extracted_at` [2026-07-15,
2026-08-15) via the `Original*` columns: **45,831 of 47,891 instrument-attaching rows (95.7%) took an unfiltered leg**
(only `gemini_fallback`'s 2,060 met the filter), and **7,957 (16.6%, a floor -- exact-match sets only)** carry a
subject the filter already rejects (gpe_country 7,184 over 38 surfaces). Consequence in the window: `U.S.` -> `U`
(Unity Software) 2,470, `Wall Street` -> `IEP` 243 -- 3,060 country-subject rows land on `U` alone, none via the
filtered leg. Same-article control, the cheapest re-check: `raw_content_id=146707`, 12 rows all
`subject_entity='U.S.'`; nine attached `U` via `hybrid_subject` while Loki carries three `decision=rejected
reason=gpe_country` lines for the same id -- the three are Rule 2's misses, the only rows the filter ever sees.
THREE CAVEATS, because the obvious fix ("hoist the filter, country subjects are junk") is wrong on all three:
(1) country subjects also resolve DEFENSIBLY (`Brazil` -> `EWZ` 446, `Germany` -> `DAX` 345, `India` -> `DEXINUS` 123),
so the discriminator is country -> single-issuer EQUITY, never country -> anything; (2) the filter false-positives on
live issuers today (`Zions Bancorporation, National Association` and `Flagstar Bank, National Association` classify
`institution` via the `GenericLastWords` arm, both ACTIVE catalog issuers `ZION`/`FLG`), so hoisting promotes "skips a
paid call" to "silently drops a real resolution"; (3) `Sensex` / `S&P 500` / `yen` arrive on `llm_candidate_pick` with
a CORRECT pick (30,575 of 30,575 rows) and a wrong substitution AFTER it (the Rule 1 entries below), so no surface
filter at any position can reach them.
TWO CANDIDATE SEAMS, not chosen -- measure before picking. Seam A: hoist `Classify` to `ResolveAsync` entry
(`_surfaceFilter` is already injected; one production caller, `V2ExtractionPipeline.cs:99`), which puts every
caveat-(2) false positive on the resolution path. Seam B: `DslToMergedExtractionAdapter.cs:499`, where `SubjectEntity`
is born, which cleans SecMaster, Gemini, `source_entity` and the matrix in one edit (the D-15 precedent) but cannot
reach caveat (3). Whichever wins, land a counter for rows attaching on a subject the filter would reject; today that
number exists only by replaying the classifier over the DB in SQL. Positioning re-verified unchanged 2026-09-16
(`:396` and `:495` still `enforce`, no hoist into `DeterministicResolver.ResolveAsync`).

**Three of the six series Sentinel is PRIMARY SOURCE for (`SentinelCollector/AGENT_README.md` D-18) have not
published since April 2026, and FOUR of the six emit no freshness gauge -- three have no pattern (`ADP_EMPLOYMENT`,
`INDEED_POSTINGS`, `REDBOOK_SALES`) and `BDIY`'s pattern file is disabled -- so no overdue alarm, so a death there is
silent. Two of the four already died that way; the other two are alive. CHALLENGER_JOB_CUTS is live again since
2026-09-16 under D-32 (#1044/#1045), which covers only a series with a dedicated feed and touches none of the other
five.** Last publish per series, measured 2026-09-16 with the SELECT below:
- `CHALLENGER_JOB_CUTS` 2026-09-16 18:05:55Z: 52,881 for `period_end` 2026-08-31 (frozen at 2026-04-23 until then)
  -- 3 live patterns (`challenger-layoff-surge`, `challenger-vs-payroll`, `sentinel-challenger-divergence`) -- LIVE.
  `challenger-layoff-surge` evaluates Signal -0.7627 = -(52881-30000)/30000 on real data. Do NOT reprocess
  raw_content 170999 again: row 912393 is review-Pending with `published_at` set and would be deleted the way 886616
  was. The first ORGANIC arrival, and the first two-document batch (the feed re-serves the prior month's report
  beside the current one; before D-32 the last writer, not the latest month, would have reached `GetLatest`), is
  the early-October report. The three-run confirmation transcript is PR #1045's body. One junk point survived the
  cleanup: `sentinel.events` 152022, `SENTINEL:NUM:BA` = 41 (2026-09-14), sits in TE's cache until retention;
  latent, since no pattern reads a `SENTINEL:NUM:` key.
- `TRUFLATION_CPI` 2026-04-23 04:43:57 -- 1 live pattern (`truflation-vs-cpi`) -- dead, visible.
- `INDEED_POSTINGS` 2026-04-16 and `REDBOOK_SALES` 2026-04-23 -- no pattern -- dead, invisible.
- `ADP_EMPLOYMENT` 2026-09-05 (count 47 on the SELECT below) -- no pattern -- ALIVE, ungauged. Never carry the April
  cohort's silence onto it.
- `BDIY` 2026-09-16 -- pattern FILE exists carrying `"enabled": false`
  (`ThresholdEngine/config/patterns/recession/baltic-freight-recession.json:24`) -- ALIVE, equally ungauged.
Re-derive (SELECT only): `SELECT coalesce("Symbol","OriginalSymbol") AS sym, max(published_at), count(published_at)
FROM sentinel.extracted_observations WHERE coalesce("Symbol","OriginalSymbol") IN ('ADP_EMPLOYMENT','BDIY',
'CHALLENGER_JOB_CUTS','INDEED_POSTINGS','REDBOOK_SALES','TRUFLATION_CPI') GROUP BY 1;`
D-18 re-check (2026-08-19), recorded here and deliberately NOT applied to the card: D-18 cites
`challenger-layoff-surge 0.98` (2026-08-12) as evidence that blanket namespacing would sever a live feed; under
signal `-(cuts - 30000)/30000` that 0.98 inverts to `cuts = 600`, the frozen value, so the exemplar showed a frozen
feed, not a live one. D-18's DECISION stands (Sentinel IS the primary source for these six), no supersession; the card
still cites the 0.98 exemplar once (2026-09-16) and Challenger now publishes 52,881, so the citation is stale either
way -- a card fix, not a defect.

**THE DETECTION BLIND SPOT.** No pattern -> no `RequiredSeries` entry -> no `PatternDataHealthEvaluator` freshness
row -> no `thresholdengine_pattern_severe_overdue_threshold_days` -> nothing to alert on. `grep -rl` over
`ThresholdEngine/config/patterns/` returns zero files for `ADP_EMPLOYMENT`, `INDEED_POSTINGS` or `REDBOOK_SALES`
(re-checked 2026-09-16; their only mention in the service is `ThresholdEngine/AGENT_README.md`), and BDIY's only
pattern is disabled: a PATTERN-level instrument used as a FEED-level one, with nothing enumerating the gap. Neither
the pattern FILES nor the live registry is an authority alone --
`ThresholdEngine/src/Configuration/PatternConfigurationLoader.cs:139` `continue`s on `!pattern.Enabled` at load, so
the registry cannot contain a disabled pattern and "all N enabled" is a tautology, not a cross-check.
Re-check: the `grep -rl` above must return 0 files.

**The publish gate, and why a plausible symbol does not survive it.** The gate is
`o.InstrumentId.HasValue && o.ResolutionConfidence >= 0.8f && o.Certainty is Definite or Expected`
(`SentinelCollector/src/Workers/ExtractionProcessor.cs:908/:918/:934` v1, `:2321/:2331/:2348` v2). **`Symbol` is not in the predicate**, so
a row carrying a plausible symbol and no instrument is dropped without a trace on the symbol axis. Nor is
`"InstrumentId": null` inside `candidate_symbols_json` the defect: **0 of 3,650,818** candidates all-time carry a
non-null value there, including every candidate on every row that published successfully. That field is the
pre-resolution proposal; the result lands on the ROW's `instrument_id`.

**THE LAST PUBLISHED VALUE ON TRUFLATION_CPI IS JUNK**, so restoring resolution without auditing what gets published
resumes publishing junk into an inflation-divergence detector. Remediation must gate on VALUE plausibility, not
merely on whether rows resolve again. (The Challenger half of this finding -- 600 from a single-company Hungarian
layoff, id 27753, frozen so the `> 100000m` trigger could never fire while the stale value read a confident +0.98 --
is closed: Challenger published 52,881 on 2026-09-16 under D-32, whose plausibility bound is the gate this paragraph
asked for; `challenger-layoff-surge` still carries the `> 100000m` trigger and the `?? 30000m` default only in
`signalExpression`.)
`TRUFLATION_CPI` last published 3.3 (id 27425, 2026-04-23 04:43:57), description `U.K. inflation`, quote "U.K.
inflation rose to 3.3% in March ... Office for National Statistics" -- a UK ONS print under the US `Truflation Daily
Inflation Index` key (ids 24914/24913 before it are UK `transport inflation`). `truflation-vs-cpi` computes
divergence `3.3 - 3.3039` = -0.0039 and halved signal -0.00195, Triggered false: the UK print IS what the matrix reads
for TRUFLATION_CPI, and sitting ~0.004 from US CPI YoY it reads as "no divergence" -- a dead feed on a foreign
country's number presenting as a healthy null, the corpse-detector shape at its worst. Re-checking the CPI leg
REQUIRES a latest-vintage filter (`CPIAUCSL` 2025-07-01 also carries an older vintage 322.132 yielding YoY 3.3157).
Re-check: `SELECT id, description, value, published_at, text_quote FROM sentinel.extracted_observations WHERE
"OriginalSymbol"='TRUFLATION_CPI' AND published_at IS NOT NULL ORDER BY published_at DESC LIMIT 3;`

**A SECOND independent defect: `TRUFLATION_CPI` is declared Daily but judged at its monthly companion's cadence.**
`PublicationFrequencyDays` is `PublicationFrequencyDaysOverride ?? RequiredSeries.Max(...)`
(`ThresholdEngine/src/Configuration/PatternConfigurationLoader.cs:320-322` — the SAME unconditional overwrite
recorded for `buffett-indicator` earlier in this file; neither pattern carries an override), so `truflation-vs-cpi`
takes `Max(TRUFLATION_CPI=1, CPIAUCSL=30) = 30` and severe becomes `Math.Max(pubFreq * 3, 14)`
(`ThresholdEngine/src/HealthChecks/PatternDataHealthEvaluator.cs:32-33`) = **90**, where a TRUFLATION_CPI-only
pattern gets `Math.Max(3, 14)` = **14**. Overdue measures against pubFreq (118 - 30 = 88), so severe lands
~2026-08-23 instead of ~2026-05-08. Under a `Max` rule a stalled MONTHLY series always masks a dead DAILY one.

**READ BEFORE SELECTING ANY POPULATION: a NULL `instrument_id` today does NOT mean resolution failed at extraction
time.** Until D-31 (#1042, merged 2026-09-16) a re-extract overwrote `InstrumentId`/`Symbol` with the new result
INCLUDING NULL on two of its five legs (`SentinelCollector/src/Workers/ReExtractBackgroundService.cs:512` full
re-extract, `:698` resolve-only, the leg prod runs); the other three pass the row's own instrument back but still
recompute `ResolutionState` and can clear `QuarantinedAt`. So historical rows sit at `instrument_id IS NULL,
resolution_state='NoResolution'` with `published_at` still standing. Measured 2026-08-19: 329 of 329 Challenger rows
carry `re_extracted_at` and only 4 still hold an instrument -- and those 4 carry a DIFFERENT `Symbol` than the key
they were filed under (ids 10714, 10716, 17490 = `UNRATE`, id 12528 = `BLK`; OPEN LEAD, unexplained); ADP rows
466076/466077 were re-extracted to NULL twenty seconds after publishing on 2026-07-10. Any population keyed on
current `instrument_id` mixes "never resolved" with "resolved, then re-extracted to null"; no conclusion drawn that
way is safe. Separate them: `SELECT re_extracted_at IS NOT NULL, "OriginalInstrumentId" IS NOT NULL, count(*) FROM
sentinel.extracted_observations WHERE instrument_id IS NULL AND extracted_at >= '<from>' GROUP BY 1,2;`
Read the second axis precisely: `ApplyReExtraction()` snapshots the `Original*` columns only when all three are
still null (preserving the EARLIEST snapshot; guarded by `ReExtractBackgroundServiceTests.cs`
`should_preserve_earliest_audit_snapshot_on_second_re_extract`), but `Quarantine()`
(`SentinelCollector/src/Entities/ExtractedObservation.cs:308`) and `QuarantineInPlace()` (`:475`) assign them
UNCONDITIONALLY, so `"OriginalInstrumentId" IS NOT NULL` means "held an instrument at the LATEST quarantine, or at
the first re-extract if never quarantined after it" -- NOT "at extraction". CONFOUND, not cause -- the April trigger
remains unestablished.

**The bulk quarantine is NOT the forward-blocking mechanism.** A quarantine stamped at exactly
`2026-04-24 00:18:52.867997+00` hit 15,894 rows across 1,054 distinct `"OriginalSymbol"` values (re-verified
2026-08-19) -- it hit BDIY too, and BDIY recovered. 272 of the 275 Challenger rows had ALREADY published before being
quarantined, so it is retroactive on already-sent data, not a forward block. Its origin is INFERRED -- do not repeat
this search expecting to close it: `ExtractedObservation.Quarantine()` (`:308`) has no production call site in git
history, and `QuarantineInPlace()` (`:475`) is called only from `SentinelCollector/src/Endpoints/AdminEndpoints.cs:1589`,
added 2026-05-15, three weeks AFTER the event (re-checked 2026-09-16). Likely a manual script or interactive session.
Re-check: `SELECT "OriginalSymbol", count(*), count(published_at) FROM sentinel.extracted_observations WHERE
"QuarantinedAt"='2026-04-24 00:18:52.867997+00' GROUP BY 1 ORDER BY 2 DESC;`

**SecMaster is not the defect.** Controlled 2026-08-19, both tools against both strings: `hybrid_resolve` returns
`ExactSql` -> `CHALLENGER_JOB_CUTS` (`ee98373d-cf04-4b16-bc7b-a3e6cf3ae57f`) for BOTH `"job cuts announced"` and
`"challenger job cuts"` via the ALIAS table (12 aliases on the instrument, re-counted 2026-09-16), and
`search_catalog` returns 0 for BOTH by design (symbol+name only, and "cuts" is not a substring of `Challenger Job Cut
Announcements`). Both routes retrieve the SAME five neighbours at the same scores and differ only in method and in
whether they resolve -- so a resolution quoted without NAMING its endpoint is unattributable. One GIGO datapoint,
refuted; do not re-raise it without re-reading this: CFIGY (`CHALLENGER LTD-UNS ADR`, created 2026-08-18 by
`discovery_source='entity_resolution:gemini'`) is proposed on NEITHER route for `q=Challenger, Gray & Christmas`.
Mechanics: the semantic endpoints take `q`, not `query` (HTTP 400 on both); `/api/semantic/resolve-local` has no
`resolution` or `answer` field (`SecMaster/src/Endpoints/SemanticSearchEndpoints.cs:341`); `secmaster` has `curl`,
`secmaster-mcp` does not. Re-check (atlas_secmaster): `SELECT a.alias FROM aliases a JOIN instruments i ON
i.id=a.instrument_id WHERE i.symbol='CHALLENGER_JOB_CUTS'` -> 12 rows.

**METHOD NOTE, because it manufactured a false finding twice.** (1) In `public.macro_observations`, `source_id` has
the form `{raw_content_id}:sig:{signal_identity_id}` on 93% of rows (the rest are OFR/FRED feeds keyed on a bare
symbol -- a majority convention, never a schema guarantee). The numeric prefix is `sentinel.raw_content.id`, NOT
`sentinel.extracted_observations.id`, and the two id spaces OVERLAP (measured 2026-08-19: 1..150,090 against
10,604..714,614), so the wrong join lands on unrelated rows and yields a convincing "systemic identity mismapping"
that does not exist. (2) Every absolute count in this entry is a moving snapshot of an append-only table -- compare
RATIOS AND SHAPES, never integers; the only figure that cannot move is the 0 in "0 of N". A re-check returning
different integers is this note working; a re-check changing a RATIO is a real finding. Under that rule the share of
`NoResolution` rows carrying a non-null `"OriginalSymbol"` is one: 57.8% (2026-08-19) -> 54.5% (2026-09-05) -> 51.9%
(2026-09-16, 412,951 of 796,197) -- the quarantine/re-extract share of the dead population is falling.
(3) THREE THINGS ARE CALLED "Symbol": the COLUMN `"Symbol"` (the resolved catalog symbol, NULL until resolution
succeeds); the KEY `"Symbol"` inside `candidate_symbols_json` (an LLM-minted slug such as `Challenger_Gray_Christmas`,
usually absent from SecMaster); and the AXIS. `"OriginalSymbol"` is NOT a fallback identity for the column: it is
written only by `Quarantine()` (`SentinelCollector/src/Entities/ExtractedObservation.cs:310`), `ApplyReExtraction()`
(`:394`) and `QuarantineInPlace()` (`:480`), each as `OriginalSymbol = Symbol` -- a PRE-REMEDIATION AUDIT SNAPSHOT,
so keying on it selects rows that were quarantined or re-extracted, NOT "the feed". Rows with neither column
populated are reachable only through the candidate or the description, and only with `LEFT JOIN LATERAL
jsonb_array_elements(o.candidate_symbols_json) c ON true` (a plain `,`-LATERAL drops every row whose json is NULL or
empty -- those are dead rows too). No remediation is recorded, and one route is closed on principle: re-keying
historical rows is a WRITE to production data and is not on the table.
Re-check (SELECT-only): `SELECT count(*) AS noresolution, count(*) FILTER (WHERE "OriginalSymbol" IS NOT NULL) AS
with_original FROM sentinel.extracted_observations WHERE resolution_state='NoResolution';` -> `796197 | 412951` on
2026-09-16.

**Production's CoD prompt carries two defects no labeller can work around.** In
`SentinelCollector/src/cod-prompts/cod_json_v1.txt`, which is what production runs: (a) `:36` "the
normalized numeric as a string" conflicts with `:68` "Emit each distinct numeric value ONCE" when one
magnitude appears with opposite signs -- dedup is keyed on the VALUE, not on (value, context,
source_entity); (b) nothing excludes clock times, quarter ordinals or bare years, and all three
bake-off labellers extracted at least one, so this is the prompt's SILENCE, not one model's judgement.
A prompt change is a quality change, so MEASURE it rather than fixing blind: the harness scores
production's CoD path end to end now -- take the 40-article scorecard before and after any edit here.
See MEASUREMENT DEBT, "The CoD gold cannot yet back a model swap".
Re-check:
```
P=SentinelCollector/src/cod-prompts/cod_json_v1.txt
grep -n 'distinct numeric value ONCE\|normalized numeric as a string' $P
grep -c -iE 'clock|ordinal|bare year|exclude|not a fact' $P
```
2026-09-16 -> lines `36` and `68` match (`59` on 2026-09-04; the prompt grew under D-32's addendum);
exclusion-word count `0`. A NON-ZERO second figure means an
exclusion rule landed and (b) is closed.

**A DELTA AND A LEVEL ARE THE SAME ROW: `numbers[]` cannot express "dropped 2%" vs "is 2%"**
[raised 2026-09-07 by the user; IMMATERIAL TO MODEL SCORING and parked on that basis -- every
candidate faces the identical schema, so this discriminates between no two of them. A data-model
defect, not a benchmark one.]

CoD stage-1 `numbers[]` has exactly five fields -- `source_text, value, unit, context,
source_entity` -- and none carries sign, direction, or level-vs-change. `"dropped 2%"` and
`"is 2%"` both emit `value: "2", unit: "PCT"`.

  - The distinction survives ONLY as free text inside `context`. Real gold, same article:
    `"total nonfarm payroll employment change"` (a delta) sits beside `"unemployment rate in
    December"` (a level). Nothing parses that string.
  - `eval_harness.number_value_accuracy` compares 2 to 2, so a model reading a delta as a level
    scores a PERFECT HIT. That is why the benchmark cannot surface this.
  - A column EXISTS: `sentinel.extracted_observations.is_comparison`, required. Production holds
    **789,386 `false` / 18,657 `true`** (2.3%); the `true` rows stop at **2026-09-04** while
    `false` continues.
  - `MergedExtractionService.cs:382` hardcodes `IsComparison: false`. Production does not route
    there -- `Extraction__Backend=VllmJson` routes to `GpuJsonExtractionService`, which **never
    mentions `IsComparison` at all**. Neither derives it from the model, because the model is
    never asked for it.
  - **Nothing branches on it.** Every reference across SentinelCollector, ThresholdEngine and
    MacroSubstrate is display or persistence: a review-UI field, admin endpoints echoing it, the
    parser, the shadow writer. No consumer filters, weights or routes on it.

UNVERIFIED, and it is the question that sizes this: does a delta actually land as a level in
`matrix_cells`, or does something upstream drop it by accident? Until that is answered this is a
schema smell; if it is the former it is live data corruption, since `"unemployment dropped 0.2"`
and `"unemployment is 0.2"` are currently one row.

TWO OPTIONS, different sizes: add a `fact_kind` enum (`level | change | forecast | prior`) to
`numbers[]` -- prompt + schema change plus a gold relabel; or join `events[]` to the number it
describes, since `events[].trigger` already captures `"rose by 256,000"` and is otherwise orphaned.
The second also answers where a direction lives.

**Rule 1's pick was never the wrong row -- the resolver SUBSTITUTED a different instrument after it, and #969 fixed
that.** On the fixed window `extracted_at` [2026-07-15, 2026-08-15), read through the `Original*` columns, **29,140 of
31,392 instrument-attaching Rule 1 rows (92.83%)** persisted a symbol matching NEITHER surface the pick named
(the figure also falls MONOTONICALLY BY CONSTRUCTION: the 817 rows the re-extract path attached enter the
denominator through `coalesce("OriginalInstrumentId", instrument_id)` while their `"OriginalSymbol"` is still the
picked slug, so they can never enter the numerator -- every re-extract pass drags the percentage down without
anything changing at Rule 1; a lower re-derived figure is not Rule 1 improving)
(`S&P 500` -> `S` SentinelOne 683 and `SP500` 528 off the SAME slug; `Sensex` -> `SNSE` 675; `yen` -> `U` 234), because
`DeterministicResolver` fuzzy-resolved the picked candidate's model-authored slug instead of its `Name`. The pick
itself matches `subject_entity` in 31,390 of 31,392 rows. #969 (`64cfe334`) resolves the Name; 12,227 rows (38.95%)
have Name == slug, so `Sensex` and `yen` are NOT addressed by it. Do not re-search the candidate list; the defect lived
between the pick and the write. #969's aggregate blast-radius count is STRUCK (no script, no output, one of its two
regressions contradicted on re-derivation) -- do not reinstate it from the PR body.
STILL OPEN, three items:
(1) Regression introduced by the fix: `Intel Corporation` -> `INL.DEX` (`FuzzySql` 0.9, the German line) where the
    slug `INTC` -> `INTC` (`ExactSql` 1.0); in-window exposure is **24 attaching rows**, not the 155 the raw `Symbol`
    column suggests (131 of those attached nothing and carry the slug only because unfixed code stores it on
    non-resolution). An exact-symbol-first leg at Rule 1 would keep `INTC` but is a new ungated exact path needing
    D-8's subject-overlap companion, its own PR. Re-check against live SecMaster:
    `nerdctl exec secmaster curl -s "http://localhost:8080/api/semantic/resolve-local?q=<surface>&enableRag=true&limit=5"`
    (`Dow Jones` is the counter-case: the slug produced `DIA`, `DOW` or nothing nondeterministically, 91 attaching
    rows; the Name resolves to `DJIA` deterministically, so that pair IMPROVES by removing a wrong answer).
(2) No test on the `!candidate.InstrumentId.HasValue &&` exemption (`DeterministicResolver.cs:488`; dead today, 0
    non-null candidate ids across 503,446 rows, and it activates the day SecMaster's search endpoint returns ids -- a
    server-side change with no compile-time signal here) nor on the hybrid leg's `ResolutionConfidence` contract
    (`IDeterministicResolver.cs:67-83`: a null-instrument `llm_candidate_hybrid` row still carries the LLM's pick
    confidence, and the >= 0.8f event-publish predicate reads that field).
(3) `SubjectNameNormalizer.SharedTokenCount` scores 0 for all four bad pairs above and is reachable from Rule 1 on
    the RagSynthesis materialisation branch (`DeterministicResolver.cs:707`), but the whole guard sits behind
    `Extraction__GuardsEnabled=false` (`/opt/ai-inference/compose.yaml:1269`), so it is inert; deciding that flag's
    fate is the prerequisite, and a third call behind the same disabled flag would read as protection that does not
    exist.

**Merged SecMaster PRs sat undeployed for 10 days, and a later deploy shipped them unannounced, so the D-13
post-deploy acceptance first charged #1030's latency cost to D-13.** First occurrence. #1029, #1031 and #1030
merged 2026-09-06 (15:56Z-17:53Z). The `secmaster` image running until 2026-09-16 was built 2026-08-25T00:53Z, one
minute after `8bac67a7` (#987) merged, so all three first reached production in the D-13 (#1049) deploy at
2026-09-16T23:01Z. Neither #1049's PR body nor #1051's diff names #1029-#1031 or ef_search. #1030 raised `hnsw.ef_search` from 40 to 400. The acceptance saw
vector-SQL latency rise and first read it as the new D-13 join. A live SELECT-only A/B then put it on ef_search: p50
1.28 / 1.38 ms without / with the join at 40, and 6.91 / 7.02 ms at 400. The figures and the open keep-or-tune
decision are in `docs/RELEASES.md` `d13-retired-scope-done`. The harness readings from that deploy mix the same two
changes. Re-tested 2026-09-17T00:02Z, with no retired embeddings left, `CHALLENGER_JOB_CUTS` is outside the top 5
for the probe query `Challenger, Gray & Christmas` at ef_search 40 and rank 1 at 400. Its return is #1030's, and only
the DISCONTINUED count is D-13's.
Nothing records merged-but-undeployed. The images carry no revision label (`org.opencontainers.image.version` only),
so the gap can only be rebuilt from build times, and that rebuild is blind in both directions. An image built from a
PR branch shows that PR's own merge as "after the build" (calendar-service below). An image built AFTER a merge from
an older checkout hides the merge.
Drift measured 2026-09-17T00:05Z, and re-checked at 00:24Z with the corrected check below, for every running compose
service with a `<Project>/.devcontainer/build.sh`. `NasdaqCollector` and `edge/sentinel-edge` also have a `build.sh`
but no running compose service, so they are out. On a container, `.Image` is the TAG (`docker.io/library/<image>:latest`),
not an image id. `image inspect <tag>` therefore returns the build the tag names NOW, which is the running build
only if that build is older than the container. On all 12 rows the tag's `.Created` is older than the container's
`.Created`, so the build time in the table is taken as the running one. A commit counts when it is on `origin/main`,
touches `<Project>/`, and was committed after that build. Shared paths such as `Events/` and `deployment/` are NOT
counted.

| service | image built (UTC) | last merge on `<Project>/` | after build: all / non-`.md` | what the non-`.md` commits touch |
|---|---|---|---|---|
| alert-service | 2026-07-22T03:14Z | 2026-09-07T15:37Z | 4 / 2 | `.devcontainer/` only (#920, #1035) |
| alphavantage-collector | 2026-07-22T01:52Z | 2026-09-07T15:37Z | 4 / 2 | `.devcontainer/` only (#920, #1035) |
| calendar-service | 2026-07-31T20:34Z | 2026-09-07T15:37Z | 4 / 3 | `.devcontainer/` (#920, #1035); `src/Containerfile` (#905), merged 83 s AFTER this build |
| finbert-sidecar | 2026-06-17T23:36Z | 2026-05-28T00:37Z | 0 / 0 | none |
| finnhub-collector | 2026-08-17T15:53Z | 2026-09-07T15:37Z | 2 / 2 | `.devcontainer/` (#1035); a comment-only edit to `src/Telemetry/FinnhubMeter.cs` (#976) |
| fred-collector | 2026-07-31T20:32Z | 2026-09-07T15:37Z | 4 / 3 | `.devcontainer/` (#920, #1035); `.cursorrules` (#1005) |
| migrate-macro-substrate | 2026-07-31T20:32Z | 2026-09-07T15:37Z | 4 / 2 | `.devcontainer/` only (#920, #1035) |
| ofr-collector | 2026-07-31T20:37Z | 2026-09-07T15:37Z | 4 / 2 | `.devcontainer/` only (#920, #1035) |
| secmaster | 2026-09-16T22:59Z | 2026-09-16T23:19Z | 1 / 1 | a comment-only edit to `src/Data/Entities/InstrumentEntity.cs` (#1050) |
| sentinel-collector | 2026-09-16T18:03Z | 2026-09-16T23:39Z | 3 / 3 | `scripts/resolution-regression/` (#1048, #1051); a comment-only edit to `src/Workers/ExtractionProcessor.cs` (#1050) |
| threshold-engine | 2026-07-31T20:37Z | 2026-09-07T15:37Z | 4 / 3 | `.devcontainer/` (#920, #1035); `PatternMnemonicFormatValidator` (#953), whose only callers are in `tests/` |
| whisper-service | 2026-06-17T23:35Z | 2026-09-07T15:37Z | 3 / 2 | `.devcontainer/` only (#920, #1035) |

11 of 12 services have a merged commit newer than their running image: 37 service-commit pairs (shared commits such
as #1035 count once per service), 25 of them touching something other than `.md`. Read diff by diff, none of the 25
is an undeployed runtime change. One cannot be settled from
timestamps: calendar-service's `Containerfile` fix merged after its image was built, which fits a build from the PR
branch but was not verified against the image. So there is no drift in behaviour today, but the SecMaster gap
went unseen for ten days, and nothing would show the next one. The consequence is misattribution: a deploy's
acceptance charges whatever it measures to the PR its brief names. Class B, not D, because the blind spot is in
what production runs, not in a harness.
Re-check, per service (fields verified on nerdctl 1.7.7):
(1) `sudo nerdctl container inspect <svc> --format '{{.Image}} {{.Created}}'` gives the tag and the container's
creation time. `container` is load-bearing: bare `inspect` returns the IMAGE.
(2) `sudo nerdctl image inspect <tag> --format '{{.Created}}'` gives the build the tag names now.
(3) If that image is NEWER than the container, the tag was rebuilt after the container started. The running build
is UNKNOWN: report the service as NOT deployed and stop, because reading on would count merged code as running.
This happens after any `build.sh` without a deploy, and inside every deploy's own window. The D-13 deploy had it
from 22:59:56Z, when the new `secmaster:latest` was built, to 23:00:40Z, while the 11:00:31Z container still ran the
2026-08-25 build.
(4) Otherwise, `git log --oneline --since=<image Created> origin/main -- <Project>/`. Any output is a merge newer
than the running build; read its diff before calling it undeployed behaviour.
Still blind: a tag re-pointed to an OLDER build after the container started (a rollback retag) passes step (3).

**The CLAUDE.md "SCOPED" `secmaster` deploy also recreates `llama-cpu-rag` and `llama-cpu-embed`, and
`llama-cpu-embed` is shared with SentinelCollector.** This comes from the tags, not from chance: the blocks
`Bring up llama-cpu-rag (SecMaster RAG generation runner)` and `Bring up llama-cpu-embed (SecMaster embedding runner)`
in `deployment/ansible/playbooks/deploy.yml` carry `tags: [llama-cpu-rag, secmaster]` and
`tags: [llama-cpu-embed, secmaster, models]`. Their `Start/recreate llama-cpu-rag container` and
`Start/recreate llama-cpu-embed container` tasks run `nerdctl compose up -d <svc>` with no `when:`, and
`-e "scoped_restart=true scoped_services=secmaster"` scopes only the compose-file restart, not these tasks. CLAUDE.md
DEPLOYMENT lists `secmaster` among the tags that carry non-build tasks but never says which; `deployment/README.md`
names `secmaster` as an alias of `llama-cpu-rag` and leaves it off `llama-cpu-embed`.
Measured 2026-09-16 with `ansible-playbook playbooks/deploy.yml --tags secmaster --skip-tags build -e
"scoped_restart=true scoped_services=secmaster"`: `nerdctl container inspect` Created = `secmaster` 23:00:40Z,
`llama-cpu-rag` 23:01:00Z, `llama-cpu-embed` 23:01:12Z. Tempo has two error spans on
`http://llama-cpu-embed:8080/v1/embeddings`, both `service.name=SecMaster` in trace `f27d0cf14bb139b7723a2b4805650f3e`:
23:01:10Z (no response) and 23:01:14Z (503 while loading). In that trace the last 200 before them is at 23:01:06Z and
the next at 23:01:18Z, so SecMaster's failure window was about 8s (23:01:10-23:01:18Z). SentinelCollector was NOT hit
on this run, but only because of timing: it calls `llama-cpu-embed` in bursts (1,090 spans in 811 traces, 22:04-23:11Z).
Its last call before the recreate was at 22:48:22Z, closing a burst of 17 calls from 22:48:10Z (2 of them errors,
13 minutes before the recreate), and its next was at 23:04:23Z. If a burst overlaps the recreate, its embedding calls
fail. That is exposure, not an observed loss. This is the SECOND occurrence: the 2026-08-15 `secmaster` scoped deploy
recreated the same two containers (PRACTICE NOTES, "Scoped-deploy collateral"). That `compose up -d` recreates an
UNCHANGED service does not rest on those two runs: the playbook records it for nerdctl 1.7.7, the version running here,
at `deployment/ansible/playbooks/deploy.yml:1217-1218` ("recreates unconditionally here") and
`deployment/ansible/playbooks/deploy.yml:1509-1510` ("recreates every transitive dep unconditionally (no config-hash
skip)", the cascade incident). The two runs agree with it. Class B, not D: the harm is a shared dependency restarted
without notice, with its failures landing in other services' traces.
Re-check: `grep -nE 'tags: \[([^]]*, )?secmaster(,|\])' deployment/ansible/playbooks/deploy.yml` lists both
`llama-cpu-*` blocks while this holds. After a scoped `secmaster` deploy,
`sudo nerdctl container inspect secmaster llama-cpu-rag llama-cpu-embed --format '{{.Name}} {{.Created}}'` shows all
three within a minute (`container` is load-bearing: bare `inspect` returns the IMAGE).

**`HybridResolutionService.NameAppearsInContext` is a literal substring test of the catalog NAME in the context string,
and it gates every live hybrid tier -- so a good catalog hit returns NONE unless the instrument's catalog name appears
verbatim in the quote.** First occurrence, recorded so the next NONE-with-a-good-catalog-hit is recognisable; NOT a
defect this epic fixes. `SecMaster/src/Services/HybridResolutionService.cs:564-571` is
`context.Contains(name, StringComparison.OrdinalIgnoreCase)` (`:571`), applied in `ResolveLocalAsync` to the ExactSql
result (`:76`), the FuzzySql top hit (`:115`) and every Vector candidate (`:184`); an empty context passes everything
through, a non-empty one demands the NAME, and the refusal is one Information line (invisible at the Warning prod
level). Measured 2026-09-16 with the resolution-regression harness in `--live` mode (`q=<subject>`,
`context=<description>`, `minScore=0.75`, the Rule 2 leg): corpus row 1, `Challenger, Gray & Christmas` /
`job cuts in July` (the July 2026 headline print in the August report), returns NONE for exactly that reason -- the
catalog name `Challenger Job Cut Announcements` does not occur in that description. Moot for `challenger-rss`, which D-32 (`SentinelCollector/AGENT_README.md`)
keys from provenance and never sends through this leg; live for every other subject whose description names the
publisher but not the series. Re-check: `SentinelCollector/scripts/resolution-regression/run.sh --live` row 1 (branch
`tooling/resolution-regression-harness`, PR #1048, until it merges).

**Production's CoD extraction loses ~74 gold entities per run to its own loop guard, TODAY.**
`entities_recall` 0.5529 -> 0.4348 on the prompt production actually runs (the #1017 prompt, on the host mount since
2026-09-06 10:26Z; the pre-#1017 prompt read 0.5545 -> 0.3809, ~108), measured 2026-09-06 and
attributable to `CpuCod__JsonRepetitionPenalty` 1.1 rather than the token cap. The full entry, its
re-check lives under §MEASUREMENT DEBT ("repetition_penalty 1.1 (not the cap) costs entities_recall 0.5529 -> 0.4348
on the production prompt"; its four-cell Qwen table was retired at the 2026-09-16 triage)
because that is where the measurement that found it sits,
but the DEFECT is a live production one and belongs in a reader's scan of this section. Nothing in
production measures it; the cheapest next step is a 1.02 / 1.05 / 1.1 penalty sweep, scored on coined
names as well as recall.

**THE MATRIX PROVENANCE CHAIN IS EMPTY: NO `matrix_cells` ROW CAN BE TRACED TO THE OBSERVATIONS
THAT PRODUCED IT.** The schema carries `contributing_observation_refs` (jsonb) and `source_provenance` (jsonb) for
exactly this, and neither is written with anything usable. Measured 2026-08-26 on 287,763 rows: refs populated on
**0**, `source_provenance` on 223 (a dead phase-4.5 experiment, `rawContentId` null on all 223). Re-measured
2026-09-16 on 361,342 rows: `refs_populated` **0**, `prov_reaches_article` **0** -- both columns still decorative.
CONSEQUENCE: every question that starts "which cells were affected" (D-18 damage, identity-collision damage, any
wrong-signal audit, any selective recompute) is unanswerable; the only substitute is a split on `evaluated_at` at a
fix date, a coarse upper bound over the whole table and not an attribution.
NOT A DATA-REPAIR JOB: the links were never written, so there is nothing to backfill. The fix is at the WS3
projector's write seam -- populate `contributing_observation_refs` when a cell is computed.
Re-check (SELECT-only, no deploy):
  `SELECT count(*) AS total,
     count(*) FILTER (WHERE contributing_observation_refs IS NOT NULL
       AND contributing_observation_refs::text NOT IN ('null','[]','{}')) AS refs_populated,
     count(*) FILTER (WHERE source_provenance IS NOT NULL
       AND source_provenance::text NOT IN ('null','[]','{}')) AS prov_populated,
     count(*) FILTER (WHERE source_provenance->>'rawContentId' IS NOT NULL) AS prov_reaches_article
   FROM public.matrix_cells;`
Closes when `refs_populated` is a material share of `total` on cells written after the fix. `total` grows
continuously -- read `refs_populated` and `prov_reaches_article` as the claim, not the absolute row count.

**`SecMasterDiscoveryTimeoutsElevated` structurally cannot fire.** The rule is `timeout/total > 0.5`, but a
per-candidate deadline propagates through the `finally` (emitting `not_found`) and THEN emits `timeout` — one
candidate, two increments, ratio pinned at exactly 0.5. Only the pre-discovery semaphore arm can exceed it.
Fix the double-count, not the threshold; an alert tuned around a miscount hides the miscount.
Metric gotcha: OTEL appends `_total`, so alert on `secmaster_fred_search_skipped_total`, not the bare name.
Re-verified 2026-09-16: rule unchanged at `secmaster.yml:57-58` (`timeout/total > 0.5` for 30m); `outcome=timeout`
emitted at `EntityResolutionService.cs:356`/`:419` with the `finally` at `:289`.

**Sentinel has NO resolution-rate alert: `SentinelLowResolutionRate` was RETIRED 2026-09-16 because it measured
nothing, and the replacement it needs is still unbuilt.** The rule divided
`sum(rate(sentinel_secmaster_resolution_total{status="resolved"}[5m]))` by the same counter unfiltered. That counter
is failure-biased: `DeterministicResolver` (the live leg, `SentinelCollector/src/Services/V2ExtractionPipeline.cs:99`)
meters NO success outcome, so only `llm_candidate_exact` ever carried `status="resolved"` (14 of 10,040 increments in
the 24h to 2026-09-16) while 70-78% of the denominator is `sector_grounding`, which by construction can never resolve.
The counter and its dashboard panels stay; removing the rule demoted no signal because it carried none.
STILL OWED, unchanged since 2026-08-20: a per-observation OUTCOME counter at the persist boundary (one increment per
row, `resolved|unresolved` x method x source feed), landed WITH a rule carrying a minimum-volume guard on the
denominator (no NaN flapping on idle windows) and a per-feed label so a dead source is nameable -- today a source
dying 100% moves no metric. The notification must carry current rate, `gemini_resolver_cap_refused_total` increase
and SecMaster `up`: the two failure modes of 2026-09-16 (Gemini cap reached 06:03Z; host reboot 11:00Z) were
invisible to the retired rule and would be to any bare ratio.
GROUND TRUTH the replacement must agree with, 7 days to 2026-08-20T17:27Z: DB **36.1%** (16,030 resolved / 44,410,
erasure-corrected; the naive `instrument_id IS NOT NULL` read is 30.1%) against a counter-side ratio of **0.13%**
(49 of 39,523). Daily buckets 20.3-40.8%, chronically below the retired 50% threshold and trending UP while the
metric sat near zero.
```sql
SELECT count(*) AS total,
       count(*) FILTER (WHERE CASE WHEN re_extracted_at IS NULL
                        THEN instrument_id ELSE "OriginalInstrumentId" END IS NOT NULL) AS resolved_corrected,
       count(*) FILTER (WHERE instrument_id IS NOT NULL)                                AS resolved_naive
FROM sentinel.extracted_observations
WHERE extracted_at >= timestamptz '2026-08-20T17:27:00Z' - interval '7 days'
  AND extracted_at <  timestamptz '2026-08-20T17:27:00Z';
```
ABSOLUTE bounds on purpose: a `now()`-relative window drifts within minutes. Run `SHOW timezone` before any
`date_trunc('day', ...)`: the 2-arg form takes its day boundary from the session TimeZone, and the psql session in
the `timescaledb` container is UTC while mercury's host TZ is `America/New_York`. Never mix the naive
`resolution_method` column with the `Original*`-corrected one in a single breakdown (the naive column is the only
place `cove_VectorSearch` / `ticker_in_quote` / `cove_FuzzySql` appear -- what the re-extract adapter wrote, not what
resolved the row).
READING TRAP: `resolution_state='Resolved'` overstated the rate until #854 (2026-07-05) -- 28,298 April rows were
Resolved with a NULL instrument -- so any dashboard on that column shows a June "collapse" that is the FIX, not a
regression; compare `instrument_id IS NOT NULL` in the same query before concluding anything moved.
Re-check that nothing fills the gap: `grep -rn resolution_rate deployment/artifacts/monitoring/alerts/` returns 0
(re-run 2026-09-16: the retirement comment at `sentinel.yml:823` spells `SentinelLowResolutionRate`, no underscore, so
it does not match); a hit is the replacement landing.

**`ReExtractBackgroundService` POST-erasure caveat for the `instrument_id` readings cited above.** The sweep wrote a
miss back as NULL over held instruments until D-31 (SentinelCollector/AGENT_README.md, #1042) made a miss retain, so
`instrument_id` readings taken before 2026-09-16 understate the real resolution rate by an unknown margin (2026-08-15:
49,616 of 84,531 genuinely-aged rows had lost one against 213 gained; 2026-09-16, the 24h before D-31 deployed, on
`sentinel_reextract_rows_processed_total`: 3,829 `instrument_lost` against 80 `recovered`, most of which were replacements). Rows already erased stay erased -- nothing
backfills them. Three traps for anyone re-measuring the sweep:
(1) `ClassifyOutcome` emits `outcome="recovered"` only when the SYMBOL CHANGES, so the counter undercounts recoveries
    by construction and a cumulative read is worthless for hours after any container restart. Read `retained` (a miss
    on a held row), `quarantine_cleared` (a quarantined row's intended clear) and `replaced` (a held attachment
    swapped by a grounded re-resolve, the D-28 hazard's own counter) instead, and treat `instrument_lost` > 0 as the
    tripwire that the guard was bypassed. A SecMaster outage reads as retained plus still-null with nothing grounded
    (alerted by SentinelReExtractGroundingNothing).
(2) `NoResolutionSweepWorker` is EXONERATED and should not be re-suspected: it only calls `SetReviewStatus`.
(3) `"OriginalInstrumentId" IS NOT NULL` does NOT mean "held an instrument at extraction": `ApplyReExtraction` writes
    the `Original*` columns only when all three are still null (earliest re-extract wins), but `Quarantine()` /
    `QuarantineInPlace()` assign them UNCONDITIONALLY, so the column reports the state at the LATEST quarantine (or at
    the first re-extract if the row was never quarantined).

**`SentinelExtractionDead` inhibits every sentinel warning if it fires** (`equal: ['service']`), including the
collapse alert. 0 inhibited to date, but the coupling is undocumented anywhere else.
Re-verified 2026-09-16: `SentinelExtractionDead` is severity critical (`sentinel.yml:55-66`); `alertmanager.yml:91`
inhibits critical -> warning on `equal: ['service']`.

**request-log regression-guard gap (#886).** The DiagnosticContext re-registration in AlertService, SecMaster and
CalendarService — which preserves `UseSerilogRequestLogging` after `Host.UseSerilog` was dropped — has no
`// INTENT` tag and no test, so a future edit deleting it silently breaks request logging at request time.
Re-verified 2026-09-16: registration present with a prose comment and no `// INTENT` tag at
`AlertService/src/Program.cs:44-51`, SecMaster `:48`, CalendarService `:49-55`; `grep DiagnosticContext` over the three `tests/` dirs
returns 0 files.

**TRIPWIRE, green by design: a NEW `BrokenCircuitException` orphaning cohort.** The classification gap itself is
closed (SentinelCollector D-27, `SentinelCollector/AGENT_README.md:151`) and both known cohorts are disposed of: 55
rows orphaned 2026-07-19..07-24, left as won't-do on alpha decay; 223 rows orphaned 2026-09-04 while `vllm-server`
was stopped for a vLLM 0.28.0 evaluation, recovered same-day by `POST /admin/reprocess`. The query is kept because
it is the standing detector for a THIRD cohort, and because the predicate matches EVERY breaker-open event ever
recorded — so a bare total answers nothing and a bare zero is ambiguous between "pruned or recovered as decided"
and "never happened". Read it BY COHORT (psql is SELECT-only).

```sql
SELECT date_trunc('day', collected_at) AS day, count(*),
       min(collected_at), max(collected_at),
       count(*) FILTER (WHERE retry_count = 0) AS never_retried
FROM sentinel.raw_content
WHERE processing_error LIKE '%circuit is now open%'
GROUP BY 1 ORDER BY 1;
```

- rows dated **2026-07-19..07-24** -> the original 55 survived to the 180-day window. Won't-do cohort; leave them,
  they prune 2027-01-15..01-20 at the first SentinelCollector restart after that date.
- rows dated **2026-09-04** -> the 223 above. They were recovered by `POST /admin/reprocess`, which CLEARS
  `processing_error`, so they should NOT appear; if they do, the reprocess did not take.
- **no rows in either range** -> both cohorts pruned or recovered as decided. Expected ending, not a missing
  measurement, and not grounds to re-raise recovery.
- rows dated **2026-09-06** -> the three pre-D-27 rows (164565-164567), orphaned by the 2026-08-27 image during the
  A/B outage; awaiting a reprocess decision in the entry named below, not a regression.
- rows on **ANY OTHER date** -> a REGRESSION on the numeric path, WITH one known exception named below. D-27 makes
  it impossible there: whether the breaker is refused before the model call (row requeued) or after it (article
  finished), neither branch writes `processing_error`. A row here means the guard was removed, bypassed, or a second
  code path writes the breaker's message. Correlate the date against `vllm-server` availability (`journalctl -t
  atlas-stack-watchdog`; host clock is EDT, not UTC) and read BOTH reasons —
  `sentinel_extraction_error_total{reason="dependency_unavailable"}` and `{reason="dependency_unavailable_after_extraction"}`
  — for the same window before concluding anything.
- the KNOWN exception: rows whose `source` is `validation-content` (or `validation-content:sector:*`). Those take
  the qualitative dispatch leg, whose catch writes `processing_error` for ANY exception before the article catch can
  see it — see the entry "DEFECT, pre-existing and now the ONLY leg outside D-27's gate: the qualitative dispatch path
  still orphans". Bucket them out with `AND source NOT LIKE 'validation-content%'` before
  reading the query above as a regression signal.

Measured 2026-08-17: `55 | 2026-07-19T17:14:25Z | 2026-07-24T11:00:06Z | 55`.
Measured 2026-09-04T22:27Z, before the recovery: ZERO from the 2026-07 cohort (the old 30-day clock pruned them
first) and `223 | 2026-09-04T00:06:38Z | 2026-09-04T16:46:56Z | 223` from a same-day cohort — both halves of this
entry landing on the same day, which is what made a bare total unreadable and is why the query above buckets.
Measured 2026-09-04, after the recovery and on the fix branch: **0 rows**, which under the reading above is the
expected steady state and NOT evidence the fix works — the fix is evidenced by
`ExtractionProcessorCircuitOpenRequeueTests`, not by this query.
RE-MEASURED 2026-09-16: 3 rss rows (raw_content 164565-164567) carry processing_error 'The circuit is now open' dated
2026-09-06T18:23-18:43Z, retry_count 0, after D-27 (#1004) merged 2026-09-05 and the 10:26Z scoped restart that day;
`vllm-server` exited 18:12Z that day; the rows are NOT `validation-content`, so the known exception (the qualitative
dispatch leg, next bullet) does not cover them. ESTABLISHED, same day: the container running at 18:23Z was the
2026-08-27 image with every D-27 symbol absent (binary probe), and the guard entered the running container only at
the 2026-09-16 18:04Z rebuild -- see the entry "Three `raw_content` rows orphaned on 2026-09-06 by the pre-D-27
build". These are the three pre-D-27 rows, not a regression.

**10 of SentinelCollector's 16 hosted workers log their startup banner at `LogInformation`, so prod has no record
those started -- while 2 siblings already log theirs at Warning, on purpose.** Prod log level defaults to Warning, so
an Information banner is invisible -- the opposite of CLAUDE.md OBSERVABILITY ("startup banners STAY Warning #
boot-loop visibility"). The precedent is in-repo: `AutoApproveDrainWorker.cs:74-76` and `NoResolutionSweepWorker.cs:66-70`
log at `LogWarning` with the comment "Startup banner at Warning so a restart loop is visible under the prod WARN log
floor" (#852, 2026-07-05), so this is finishing a conversion, not proposing one.
The 10 at `LogInformation` (measured 2026-08-15; ReExtract, MirrorSearch and ResolutionWorker re-read unchanged
2026-09-16): `ReExtractBackgroundService.cs:120-123` (plus its disabled-by-flag banner `:94-96` and stop banner
`:184`), `ExtractionProcessor.cs:96`, `MirrorSearchWorker.cs:79`, `ResolutionWorker.cs:51`,
`StaleContentPrunerService.cs:85`, `RssFeedCollectorWorker.cs:31`, `EdgeSyncWorker.cs:27`,
`SearxngCollectionScheduler.cs:60`, `ValidationEventConsumerWorker.cs:35`, `ValidationQueryExecutorWorker.cs:27`. The
remaining 4 emit no startup banner at all -- the same blind spot in a different shape -- and should get one. ReExtract
matters most: its banner carries `MinRowAgeDays`, the live-traffic guard D-21, so prod cannot confirm which value it
read.
Re-check: `grep -rn "class .*: *\(BackgroundService\|IHostedService\)" SentinelCollector/src/Workers` for the
denominator, then the banner level per file. One-line change per worker, in a SentinelCollector PR.

**Four of the five Grafana-managed rule groups expire out of Alertmanager between pushes, so a long-lived alert
emits a false `resolved` on every cycle.** Grafana forwards its native rules through a `prometheus-alertmanager`
contact point, which POSTs only when Grafana's OWN notification policy fires, and each posted alert carries
`endsAt = last_eval + 4x the RULE GROUP's interval`. Where that lifetime does not outlast the gap between pushes the
alert expires inside Alertmanager, which fires a `send_resolved: true` webhook to AlertService -> ntfy claiming a
recovery that never happened and then re-admits the same alert as new on the next push. Measured 2026-08-19 on the
live instance: `endsAt` exactly 4x the interval after `updatedAt` on a 30m group, while the rule had been firing
continuously since 2026-08-03T01:42:10Z with 4 instances and Alertmanager's active list held none of them.
What a push GUARANTEES is `3x interval` (the POST fires up to one interval after the evaluation), so a group needs
`3x interval > push gap` against the root policy's 4h `repeat_interval`: `loki-warning-rate` (5m),
`threshold-engine-projector` (10m), `threshold-engine-regime` (15m) and `ofr-derived-cell-age` (1h -- needs past
1h20m, not the 1h a 4x reading would accept) are underwater; `threshold-engine-pattern-data` (12h) was fixed in #981.
`interval` is the ONLY lever that moves `endsAt`, paid for in detection latency: `keep_firing_for` extends GRAFANA's
firing state, not the stamped `endsAt`, and `disableResolveMessage: true` on the contact point gates only the resolve
GRAFANA sends, while this one is generated by ALERTMANAGER itself. Guarded: `deployment/tests/alerts/check-routing.py`
check (d) fails on any group underwater without being listed in `KNOWN_UNDERWATER` (`:94-99`; all four still listed
2026-09-16, intervals unchanged) and on a listed group that becomes healthy, so closing a row is what deletes it.
Re-check for one alert (Alertmanager publishes NO host port and its image has no `curl`; `amtool` is present):
`sudo nerdctl exec alertmanager amtool --alertmanager.url=http://localhost:9093 --output=json alert query --active 'alertname="<rule title>"'`
`[]` means Alertmanager is NOT holding it (still broken); a non-empty JSON array means it is; a non-zero exit means
the measurement did not happen, which is a third answer and not the first.

**1,168 FILES UNDER `rss/2026/04/23/` (PLUS 34 MORE THE SAME DAY ACROSS `searxng-content` /
`tsa-checkpoint` / `validation-content`, AND A FEW DOZEN STRAYS FROM 2025-12-31 THROUGH 2026-07-03) HAVE
NO MATCHING `sentinel.raw_content` ROW AT ALL — orphaned on write, not pruned after write.** Measured
2026-08-26: `SELECT count(*) FROM sentinel.raw_content WHERE raw_file_path LIKE '%rss/2026/04/23%'`
returns **0** against a directory holding 1,168 `.html` / `.meta.json` pairs on disk.
`StaleContentPrunerService` cannot be the cause — both its passes select rows FROM `raw_content` and
delete the file that row names, so a file the table never referenced was never a pruning candidate.
`RawContentService.StoreContentAsync` writes the file (and its sidecar) BEFORE `_repository.AddAsync`
inserts the row, and its own duplicate check ("DB unique constraint provides final protection" against
`IX_raw_content_source_content_hash`) is check-then-write, not atomic under concurrent collection —
consistent with, but not confirmed as, the mechanism: a lost race on that unique index would leave
exactly this shape (file on disk, no row). Not chased further this round.
Re-check: `sudo find /opt/ai-inference/raw-data/sentinel/rss/2026/04/23 -type f | wc -l` (1168) against
`SELECT count(*) FROM sentinel.raw_content WHERE raw_file_path LIKE '%rss/2026/04/23%'` (0).
Re-checked 2026-09-16: 1168 files, 0 rows, unchanged. Confirmed not a pruning artifact (2026-08-27): 292
`raw_content` rows exist for that date with `source='rss'` (322 total that day; the other 30 are
`searxng-content`), all already `raw_file_path IS NULL` from pass 2 under the PRIOR 30-day `RawRetentionDays` --
so these files were never referenced by any row, not referenced-then-nulled, and no pruner setting will ever reclaim
them (the pruner mechanism is the entry below).

**A `raw_content` ROW WITH EVEN ONE `extracted_observations` CHILD CAN NEVER BE PRUNED, AT ANY
`RawRetentionDays` SETTING -- the cohort is unreachable by construction, not by policy, and raising or
lowering the setting cannot fix it.** `StaleContentPrunerService` runs two FK-safe passes: pass 1
(`StaleContentPrunerService.cs:95`) deletes rows past the cutoff only `WHERE CollectedAt < cutoff AND
!r.Observations.Any()` (`RawContentRepository.cs:108`); pass 2 (`StaleContentPrunerService.cs:112`) nulls
`raw_file_path` on the rows pass 1 skipped -- `CollectedAt < cutoff AND RawFilePath != null AND r.Observations.Any()`
(`RawContentRepository.cs:146`) -- keeping the row and its children intact. Once pass 2 has run once on a row there
is nothing left for EITHER pass to do to it; moving the cutoff changes which rows are OLD ENOUGH, never which rows
HAVE CHILDREN. Scope: true of the pruner, not of the whole system -- `/admin/reprocess` (`SentinelCollector/src/Endpoints/AdminEndpoints.cs:228`)
deletes a row's children scoped to `ReviewStatus.Pending AND QuarantinedAt IS NULL`, and deleting a row's last
Pending child makes it childless and unlocks it for pass 1 on the next host start; measured 2026-08-27 the cohort's
children were Approved 36,570 / AutoClosed 18,567 / Rejected 55 / Skipped 3, zero Pending, so the exception exists
but does not apply here.
Measured 2026-08-27, raising `RawRetentionDays` 30 -> 180 and restarting the container: 0 rows deleted. Rows past
the 180-day cutoff 4,980; childless among them 0; children-bearing with `raw_file_path` still set 0 -- all 4,980 have
children and already carry `raw_file_path IS NULL` (their HTML was freed on an earlier host start). Re-measured
2026-09-16: 5,008 past the cutoff, 0 childless -- the same shape.
- FIXTURE NOTE (from the retired plan-phase entry): with the 180-day window (`Extraction__RawRetentionDays=180`,
  `/opt/ai-inference/compose.yaml:1235`; code default 180) "select only RECENT articles" is no longer the right
  fixture instruction: anything back to 2026-07-27 is fixturable and the first post-raise prune is due ~2027-01-23.
  Measured 2026-09-05: fixturable share 56.9% (68,826 of 120,998 published observations carry a raw file);
  re-verified 2026-09-16: `min(collected_at)` 2026-07-27, `max` 2026-09-16, 47,281 `raw_content` rows with a raw
  file. Re-check: `SELECT source, min(collected_at)::date, max(collected_at)::date, count(*) FROM sentinel.raw_content
  WHERE raw_file_path IS NOT NULL GROUP BY 1 ORDER BY 4 DESC;` -- every min must be >= today-180; a min that starts
  ADVANCING means the first post-raise prune ran; a min at today-30 means the setting was reverted. Pass 1 is demonstrably active on childless rows
elsewhere: a 30-day RANGE query on `sentinel_extraction_error_total{source="prune"}` steps up to 3405, while
`StaleContentPrunerService.cs:98` emits the counter only when the CURRENT run deleted something, so an empty INSTANT
read after a restart says nothing about the pass.
Severity: not urgent, and this should not read as a fire. The bytes that matter are already reclaimed -- pass 2 freed
the on-disk HTML, which is the bulk of the storage; what accumulates is `raw_content` ROWS plus their
`extracted_observations` children, narrow rows with no blobs, at whatever rate articles get extracted and then age
past 180 days. Size a prune's expected deletions with the childless query below, never with a past-cutoff count: the
deploy brief that predicted ~315 deletions from this restart counted rows past the cutoff (4,980 against 0
childless). The separate 377 rows with `processing_error LIKE 'age_cutoff:%'` (still exactly 377 on 2026-09-16) are
`ExtractionProcessor`'s per-article `MaxArticleAgeDays` (30d) skip marker -- a different column, setting and service
-- and not a prune-candidate count.
Re-check (psql is SELECT-only):
  `SELECT count(*) FROM sentinel.raw_content WHERE collected_at < now() - interval '180 days';`
  `SELECT count(*) FROM sentinel.raw_content r WHERE r.collected_at < now() - interval '180 days' AND
    NOT EXISTS (SELECT 1 FROM sentinel.extracted_observations o WHERE o.raw_content_id = r.id);`
  `SELECT count(*) FROM sentinel.raw_content r WHERE r.collected_at < now() - interval '180 days' AND
    r.raw_file_path IS NOT NULL AND EXISTS (SELECT 1 FROM sentinel.extracted_observations o WHERE
    o.raw_content_id = r.id);`

**`period` IS EXTRACTED BY ONLY TWO SOURCES, AND NEITHER OF THEM PUBLISHES ANYTHING -- EVERY
LIVE-PUBLISHING SOURCE IS AT ZERO.** The v2/DSL extraction path carries no `period` on any source; the only two
sources still carrying one (`tsa-checkpoint`, `rss-fallback`) are precisely the two NOT listed in
`Extraction__V2EnabledSources` (`/opt/ai-inference/compose.yaml`, READ ONLY, never edit it) and the only two whose
rows carry no `dsl_block_id`, and neither publishes. Two earlier statements of this finding are RETIRED: "`period` is
not extracted" (wrong only about the table total -- 16.8% of all rows carry one) and "`period` is extracted at volume
and LOST between extraction and publish" (that volume is one source that publishes nothing, so there is no clearing to
find and no nulling write to hunt). Not a lost value: a MISSING FIELD on the CoD path.
Measured 2026-08-28, re-measured 2026-09-16, `sentinel.extracted_observations` last 30 days (rows / with `period` /
published): `rss` 166,200 / 0 / 58,317; `tsa-checkpoint` 16,789 / 16,789 / 0; `rss-fallback` 19 / 17 / 0; every v2
source 0 with `period` (`fed-press-*` now has 5 rows, 0 `period`). Strongest form, one source across the cutover:
`rss` alone by month went from 56.6% populated (2026-04, 0 DSL-tagged) to 0 (2026-06 onward, every row DSL-tagged) --
one collector, one publish path, before and after. Since 2026-06-01 DSL-tagged rows carry `period` on 0 of 534,655
and non-DSL rows on 48,519 of 48,634 (99.8%). The last `extracted_at` on any published, period-bearing row is
2026-06-30 14:40:17+00 (unchanged 2026-09-16): the cutover, not a gate closing. `tsa-checkpoint` publishes NOTHING at
all (last `published_at` 2026-02-07; KNOWN DEFECTS, below), so there is no publish-side effect to find.
THE LEAD, and one grep already kills its naive form: `SentinelCollector/src/cod-prompts/cod_json_schema_v1.json`
contains no `period` field (`"additionalProperties": false` on the item schemas and the envelope), yet
`SentinelCollector/src/Services/V2ExtractionPipeline.cs:271` DOES assign `Period = extraction.Period`, so "the v2
adapter forgot to map the field" is false. What settles it is a CODE READ, not another query: read
`V2ExtractionPipeline.cs` and `GpuJsonExtractionService.cs` against the v1 site `Workers/ExtractionProcessor.cs:750`
(the only other `Period =` site in the service) and establish what fills `extraction.Period` on the CoD path. NOBODY
HAS READ THAT CODE; do not assert a mechanism from this entry. The read decides whether restoring `period` for R2/S4
(PARKED EPICS, below) is a bug fix or a schema-plus-prompt change. Severity: a lead, not a fire -- no published VALUE
is touched, R2/S4 is parked, and nothing reads `period` on the live path.
Re-check:
  `SELECT source, count(*) total, count(*) FILTER (WHERE nullif(btrim(period),'') IS NOT NULL) with_period,
     count(*) FILTER (WHERE published_at IS NOT NULL) published,
     count(*) FILTER (WHERE metadata ? 'dsl_block_id') dsl_tagged
   FROM sentinel.extracted_observations WHERE extracted_at >= now() - interval '30 days' GROUP BY 1 ORDER BY 2 DESC;`
  `SELECT max(extracted_at) FROM sentinel.extracted_observations
   WHERE nullif(btrim(period),'') IS NOT NULL AND published_at IS NOT NULL;`

**Three `raw_content` rows orphaned on 2026-09-06 by the pre-D-27 build during the A/B vLLM outage are still
permanently failed; whether to reprocess them is the user's call.** Taking the GPU for the A/B in MEASUREMENT DEBT,
"The candidate BEATS the incumbent on production's CoD path", stopped production's vLLM for 32m56s (incumbent healthy
`18:12:19Z`, candidate serving `18:15:50Z`, incumbent restored `18:45:15Z`). The container running then was a
pre-#1004 image and permanently failed three rows inside that window in EXACTLY the pre-spend orphan shape D-27 says
can no longer occur (`SentinelCollector/AGENT_README.md`): ids 164565, 164566, 164567, source `rss`, collected
18:23-18:43Z, `retry_count 0`, `processed_at` null, `processing_error` "The circuit is now open and is not allowing
calls." The window census (17:50-19:00Z, 14 rows) bounds the blast radius to those three; the other 11 processed
cleanly. D-27's guard has been in the running container since the 2026-09-16 18:04Z rebuild (binary probe:
`ArticleExtractionSpend` 1, `DependencyOutage` 1, `IsCircuitOpen` 1, control `MaxArticleAgeDays` 1), so the mechanism
is closed and only the residue remains. NOT REPAIRED HERE -- psql is SELECT-only.
Re-check (SELECT only): `SELECT id, source, retry_count, processed_at, processing_error FROM sentinel.raw_content
WHERE processing_error LIKE '%circuit is now open%' ORDER BY id;` -> 2026-09-16: the three rows, `processed_at` null.
A GROWING count means an outage orphaned rows despite the guard, a real regression; an EMPTY result means they were
reprocessed, which closes this entry. To ask whether a symbol is in the RUNNING container (the method that settled
this): `sudo nerdctl exec sentinel-collector grep -a -c <TypeOrMember> /app/SentinelCollector.dll` with
`MaxArticleAgeDays` as the positive control; string LITERALS live in the UTF-16 `#US` heap and are invisible to it,
so probe type and member names only.

**No alert keys on CoD truncation: `truncated_salvaged` is counted, dashboarded by nobody, and the only re-check is a
hand-run PromQL** [2026-09-13]

Zero references to `truncated_salvaged` anywhere under `deployment/` -- no rule, no dashboard -- while the outcome was
2.3-3.2% of weekday articles for a week without anyone seeing it (measurement: SentinelCollector D-30 and
`LlmBenchmark/BENCHMARKS.md` *Completion budget 8192*). Raising the cap to 8,192 moves the
cliff from ~80 to ~160 objects; a further verbosity move (a model swap, a prompt asking for longer quotes) walks off it
again in silence. Proposed rule, P3 notify, same shape as `NewsSignalSubFloorDropHigh` in
`deployment/artifacts/monitoring/alerts/sentinel.yml` (volume floor + ratio, clamp_min epsilon):
```
sum(increase(sentinel_extraction_outcome_total{outcome="truncated_salvaged"}[6h]))
  / clamp_min(sum(increase(sentinel_extraction_outcome_total{outcome=~"success|verify_failure"}[6h])), 1e-9) > 0.02
and sum(increase(sentinel_extraction_outcome_total{outcome=~"success|verify_failure"}[6h])) >= 50
```
plus the promtool fixture in `deployment/tests/alerts/` and a `selftest.sh` control that breaks it by name.
The counter series has carried no sample since 2026-09-14T00Z (last value 0, after the cap raise), so the rule must
tolerate an absent series rather than read silence as health.
Re-check: `grep -rn truncated_salvaged deployment/artifacts/monitoring/alerts/` returns a rule, and the fixture fires it.

**The GPU path has no per-stage CoD token series: `VllmClient` tags every completion
`stage="classifier_llm"`** [2026-09-13]

`VllmClient.RecordTokensPerSecond` (`SentinelCollector/src/Services/VllmClient.cs`) records
`sentinel_llm_completion_tokens` and `sentinel_llm_prompt_tokens` under `stage="classifier_llm"` from BOTH
completion cores, CoD included; its comment still says the client "serves the NewsSignalClassifier",
which the GPU-JSON role flip made false. `cod_llm` reaches the token histograms only from
`LlamaServerClient`, the CPU rollback arm. Measured 2026-09-13: the only `stage` value on
`sentinel_llm_completion_tokens_count` over 7 days is `classifier_llm`, 11,639 samples, and per day the
sample count is ~ GPU-path articles + classifier requests (about 0.9x each other), so the histogram is a
blend of ~half tiny classifier completions and its mean (450 -> 650 per call across the Gemma swap) is
not a CoD figure. Until the stage is split, tokens per object cannot be read from the histogram; the Loki
`|= "did not close"` ratio method that once stood here was retired with the "FIXED 2026-09-13 by the PR that raised JsonMaxCompletionTokens" entry (its own close
condition met, 2026-09-16) and SentinelCollector D-30 is the record of the truncation rate. Re-check:
```
count by (stage) (sentinel_llm_completion_tokens_count)   # one value = still blended
```

**The news-signal feed narrowed under Gemma 4 -- 36% -> 25% of articles carry a `:sig:` row -- and a
labelled check says that is noise leaving, not recall lost. One id confusion is the only defect**
[2026-09-13]

Volume, so the next reader does not file the drop as a regression. `:sig:` rows per processed article
(psql: `public.macro_observations` with `source_collector='sentinel'`, joined to `sentinel.raw_content` by UTC day
of `processed_at`): Qwen weekdays 35.6-37.0% of articles carried at least one signal, Gemma weekdays 20.7-29.8%;
distinct signals fed 78 -> 59 over a 4-day window; `sentinel_news_signal_macro_write_total` ~700/day -> ~500/day;
digest momentum signals 16/day -> 7-12/day. The classifier returns empty arrays (`outcome="empty"` 61% -> 75% of
requests) and `sentinel_news_signal_classifier_dropped_total{reason="sub_floor"}` FELL (17-103/day -> 10-15/day),
so the signals are not being filtered out.

THE LABELLED CHECK, 2026-09-13. 30 articles: per period 8 with a `:sig:` row and 7 without, `n_obs >= 3`,
weekdays 2026-09-01..04 (Qwen) and 2026-09-08..11 (Gemma), ordered by `md5('seed-2026-09-13-' || id::text)`
within each stratum so the draw is reproducible from psql. Text rebuilt through the trafilatura sidecar
(`/extract-file`) and cut at `NewsSignalExcerptMaxChars` = 12,000, i.e. what the classifier saw. Two independent
blind labellers tagged catalog ids under the production prompt's own rules
(`SentinelCollector/src/prompts/news_signal_classify.md`; Jaccard 0.98 on the Qwen set, 1.00 on the Gemma set),
scored against the union (lenient) and the intersection (strict) of the two label sets:

| | Qwen 2.5 | Gemma 4 |
|---|---|---|
| signals emitted on the 15 articles | 18 | 12 |
| precision, strict..lenient | 0.39..0.44 | 0.58 |
| precision counting same-sign `weak` labels | 0.44 | 0.75 |
| recall, lenient..strict | 0.80..0.88 | 0.88 |
| tilt sign agreement, matches where both tilts are non-zero | 5/6 | 7/7 |
| signal-bearing articles that genuinely carried one | 3/8 | 7/8 |
| labelled signals across the 7 no-signal articles | 0 | 0 |

READING: weighting the strata back to population, Qwen delivered ~13% of articles with a genuine signal and ~22%
with a spurious one; Gemma ~22% genuine and ~3% spurious. Qwen's false positives were wrong-country series and a
boilerplate page; Gemma's strict misses were borderline-confidence calls the labellers put in `weak` with the same
sign.

WHAT IS OPEN:
  1. ID CONFUSION, n=1 of 12: `inflation-expectations` -> `cpi-headline-yoy` when an article's inflation
     content is expectations rather than a print. A prompt-clarification candidate; not measured beyond
     one case.
  2. ZERO-TILT EMISSIONS (1/18 Qwen, 1/12 Gemma) write `value_numeric = 0` rows: harmless to the decay
     sum, but they count as signal rows in every volume figure above.
  3. VOLUME FIGURES NOW MISLEAD: `sentinel_news_signal_macro_write_total`, digest momentum signals, and
     any panel on `:sig:` counts read the swap as a ~30% degradation. No rule alerts on those counts
     today (`deployment/artifacts/monitoring/alerts/sentinel.yml` keys on classifier `request_total` by
     failure outcome and on `dropped_total` by reason, and deliberately excludes `outcome="empty"`);
     re-baseline before adding one.
  4. POWER: n=8 per model in the with-signal stratum; the 3/8 vs 7/8 split is Fisher two-sided ~0.12.
     The load-bearing numbers are the signal-level precision (18 vs 12 emissions) and 0 misses across
     14 no-signal articles. The labellers are frontier-model judges, not humans.

Re-check (psql is SELECT-only):
```sql
-- articles with at least one :sig: row, per UTC day; Gemma weekdays sat at 21-30%
WITH s AS (SELECT (ingestion_time AT TIME ZONE 'UTC')::date d, count(DISTINCT split_part(source_id, ':sig:', 1)) a
           FROM public.macro_observations WHERE source_collector = 'sentinel' AND source_id LIKE '%:sig:%'
             AND ingestion_time >= now() - interval '7 days' GROUP BY 1),
     r AS (SELECT (processed_at AT TIME ZONE 'UTC')::date d, count(*) n FROM sentinel.raw_content
           WHERE processed_at >= now() - interval '7 days' AND processing_error IS NULL GROUP BY 1)
SELECT r.d, r.n, s.a, round(100.0 * s.a / r.n, 1) AS pct FROM r LEFT JOIN s USING (d) ORDER BY 1;
```
Redraw the sample with the seed above (8 `has_sig` and 7 without per period, `n_obs >= 3`) to re-label.

**fp8 -> UNQUANTIZED KV is unmeasured on Gemma 4 @ vLLM 0.28.0 on production's CoD path.** [measured 2026-09-04,
re-scoped 2026-09-07] On Qwen2.5-32B-AWQ @ vLLM 0.19.0, KV dtype the ONLY variable, full 597-record substrate,
unquantized KV was +0.051 aggregate_f1 (0.443 -> 0.494), all in recall (text_quote_recall +0.058, selectivity_recall
+0.058), for +34% latency (10.0 -> 13.4 s/doc) against ~20x measured headroom (1,945 req/h capacity vs ~96 req/h
actual). That number does not transfer: neither the model nor the engine is served any more, and the 2026-09-07
swap moved `fp8_e5m2` -> `fp8_e4m3` (the crash fix, +0.0103 single-axis, null), NOT fp8 -> unquantized. Re-earn it
with `--task cod` against the 40-article gold, not `aggregate_f1` (a CoVe metric the CoD scorecard does not carry).
The dtype is a literal in the vllm-server `command:` in `deployment/artifacts/compose.yaml.j2` pinned by
`ExtractionModelCoordinateTests`, so moving it is a re-score, not an edit; cost a production stop plus two ~4min
GPU reloads. Re-check: `grep -i unquant LlmBenchmark/BENCHMARKS.md` -- no unquantized-KV arm on Gemma 4 yet.

**vLLM 0.28.0 was BLOCKED by our `fp8_e5m2` KV cache on sm_120; `fp8_e4m3` is the one-flag fix. CLOSED
2026-09-07** by the Gemma 4 swap, which pins `vllm_image` to 0.28.0 and `--kv-cache-dtype fp8_e4m3` in
`deployment/artifacts/compose.yaml.j2`, and pins the dtype in CI
(`ExtractionModelCoordinateTests.should_serve_the_kv_cache_dtype_the_engine_does_not_fault_on`). The isolation table
is NOT retired with the entry: CLAUDE.md VLLM_UPGRADE points at it, and the re-check at its foot is owed on the next
release. Measured 2026-09-04 on the RTX 5090 (sm_120) with production's exact nine flags: 0.28.0 STARTS fine and
serves single requests, then faults under concurrent decode with `torch.AcceleratorError: CUDA error: an illegal
memory access` and stays 503. Isolated to one variable:

| vLLM | KV dtype | ctx | conc | result |
|---|---|---|---|---|
| 0.19.0 | fp8_e5m2 | 32K | 6 | 597/597 clean, twice (production until the 2026-09-07 swap) |
| 0.28.0 | fp8_e5m2 | 32K | 1 | OK |
| 0.28.0 | fp8_e5m2 | 32K | 2 / 4 / 6 | crash |
| 0.28.0 | fp8_e5m2 | 16K | 6 | crash, 17/18 |
| 0.28.0 | fp16 | 16K | 6 | 0 errors, healthy |
| 0.28.0 | **fp8_e4m3** | 32K | 6 | **0 errors, healthy** |

Only the KV dtype differs between the crash row and the two healthy rows at matched context and concurrency, so it
is the e5m2 FORMAT -- not sm_120 generally, structured output, context length or CUDA graphs (`--enforce-eager`
still crashes), and `--attention-backend TRITON_ATTN` does not help (it fails to start on a torch.compile error
inside the 0.28.0 image). The lever is a flag, not a version.

RE-CHECK: run the levers again on the next vLLM release. Correctness of the e4m3 path was spot-checked at n=18
against the known-good 0.19.0 output (aggregate_f1 0.517 -> 0.526, all deltas small and mixed-sign) -- that is "no
evidence of the failure", NOT proven equivalence. `period_accuracy` moved -0.056 and is the one to watch on a full
597-record confirmation.

**DEFECT, pre-existing and now the ONLY leg outside D-27's gate: the qualitative dispatch path still orphans on a
dependency outage.** `TryDispatchQualitativeAsync`'s extract-stage catch (`SentinelCollector/src/Workers/ExtractionProcessor.cs:2897`)
calls `MarkRawContentProcessedAsync(..., ex.Message, ...)` for EVERY exception, so a `BrokenCircuitException` writes
`processing_error` and the row leaves the queue with nothing re-driving it — the original D-27 failure mode, on this
one leg. It is not reachable by the gate BY CONSTRUCTION: the gate lives in the article catch, and this catch runs
inside the `try`, so it decides before the gate is ever consulted. Found by review of PR #1004 (R2, finding M1) and
left alone there deliberately: it changes qualitative-path behaviour, which that PR does not otherwise touch, and it
has no test. Reachable only for `validation-content` sources, so the blast radius is the validation-query worker's
rows, not the news frontier.
Fix, when it is picked up: rethrow from that catch when `DependencyOutage.IsCircuitOpen(ex)` so the single gate
decides, rather than adding a second place that answers the same question — the whole point of D-27's current shape.
Re-check (psql is SELECT-only):
```sql
SELECT count(*), min(collected_at), max(collected_at)
FROM sentinel.raw_content
WHERE processing_error LIKE '%circuit is now open%'
  AND source LIKE 'validation-content%';
```
Measured 2026-09-05: **0 rows**, against **0** for the same predicate with the source filter removed — so the whole
`processing_error LIKE '%circuit is now open%'` population is currently empty and this figure does NOT distinguish
"never hit" from "hit and pruned by the 180-day retention". Read it as a BASELINE for the re-check, not as evidence
the leg is unreachable. A non-zero count is the defect firing, and closes the "is it worth fixing" question with a
number.

**DEFECT, pre-existing and MORE reachable after D-27: `ResolutionWorker.ResolveOneAsync` wraps `secMaster.ResolveAsync`
in `catch (HttpRequestException)` only.** During the same shared-SecMaster-breaker outage that D-27 now routes rows
into as `resolution_state=Pending`, a `BrokenCircuitException` escapes that catch and the per-row try (which catches
only `OperationCanceledException`), abandoning the rest of the batch each tick with NO
`processed_total{outcome="error"}` increment — so `SentinelResolutionWorkerErrors` cannot fire and the stall is
invisible. Rows stay Pending, which is safe and self-healing, so this is a VISIBILITY defect rather than a data one.
Found by review of PR #1004 (R2, finding M2). Re-check: during any SecMaster outage, compare
`sum(increase(sentinel_resolution_worker_processed_total[15m]))` against the Pending row count —
```sql
SELECT count(*) FROM sentinel.extracted_observations WHERE resolution_state = 'Pending';
```
— a Pending count that climbs while the worker counter is FLAT is this defect. Measured 2026-09-05: Pending = **0**,
which is the healthy baseline and says nothing either way about the defect; it has NOT been observed during an
outage, the mechanism is read from the code, and the first real SecMaster break window is what would confirm it.

**tsa-checkpoint HAS PUBLISHED NOTHING SINCE 2026-02-07, WHILE STILL EXTRACTING — 83,160
observations mined, 729 ever published (0.88%), and the last of those is six and a half months
old.** Measured 2026-08-26 while selecting the golden corpus. It matters beyond the feed itself:
`extraction-identity-implementation.md` §1 names TSA among "the outliers that work today ... these
must stay green throughout", and a story that treats a six-month-dead publish path as a working
control is measuring nothing. TSA is also un-fixturable for the same reason — its 70 retained raw
files carry 237 observations with ZERO published between them, so there is no published row to
assert on. Unknown, and worth establishing before anyone calls this a regression: whether the stop
is a publish-gate change, a resolution change, or a deliberate retirement.
Re-check:
  `SELECT max(published_at)::date, count(*) FILTER (WHERE published_at IS NOT NULL), count(*)
     FROM sentinel.extracted_observations WHERE source='tsa-checkpoint';`
  # 2026-08-26 returned 2026-02-07 | 729 | 83160
  # 2026-09-16 returned 2026-02-07 | 729 | 95521

**A pattern author's `publicationFrequencyDays` is DEAD CONFIG, discarded without a word.**
`ThresholdEngine/src/Configuration/PatternConfigurationLoader.cs:320-322` overwrites the authored value
unconditionally with `PublicationFrequencyDaysOverride ?? RequiredSeries.Max(frequencies)`, so only the OVERRIDE is
honoured. Same class as WM2NS (#898/#899): publication cadence != data frequency. The ALERT consequence is gone —
`thresholdengine.yml` now joins `thresholdengine_pattern_data_overdue_days` against
`thresholdengine_pattern_severe_overdue_threshold_days` on `pattern_id` (landed 2026-08-17), so
`buffett-indicator`'s healthy 162-199 peak sits under its derived 270 and pages nobody. What is left is the silent
discard, and a SECOND consequence of the same `Max`: on a MIXED-cadence pattern a stalled MONTHLY input masks a dead
DAILY one (`truflation-vs-cpi` judged at 90 days where its Daily series implies 14). One line, two defects — read
both before closing either.
Re-check: author any `publicationFrequencyDays` in a pattern JSON carrying no `PublicationFrequencyDaysOverride`,
reload, and confirm `thresholdengine_pattern_severe_overdue_threshold_days` for it still reads
`max(3 * SecMaster-derived freq, 14)` — the authored number must not appear anywhere.

**Rule 1's outcome is ERASED downstream, not never produced -- and the column that says otherwise cannot tell the
difference.** Measured 2026-08-15 over 24h: 6,099 rows carry `OriginalResolutionMethod='llm_candidate_pick'` and
**1,421 of them (23%) had lost an instrument they already held**; the erasure hit EVERY `DeterministicResolver` leg
(`hybrid_subject` 549, `gemini_fallback` 128, `llm_candidate_exact` 1 in the same window -- the composition
reproduces, the absolutes move with the window). The writer was `ReExtractBackgroundService`, fixed going forward by
D-31 (#1042); rows erased before it stay erased.
THE TRAP, which cost hours of wrong root-cause search: any query over `resolution_method` / `instrument_id` reads
POST-erasure state and cannot distinguish "never set" from "overwritten" (this entry once concluded "no
`DeterministicResolver` outcome has ever been persisted" from 0 hits in 658,167 rows -- true of the column, false
about the resolver). For any row re-extracted before 2026-09-16 read `OriginalResolutionMethod` /
`OriginalInstrumentId` alongside the live columns, always.
A SECOND circular column: `extracted_observations.resolution_confidence` holds the resolver OUTCOME's value
(`DeterministicResolver.cs:416-420`), not what Rule 1 received; the input is visible only in
`sentinel_resolver_rule1_input_confidence` (`SentinelMeter.cs:1739`, from #963). Every observation of it sits at
exactly 0.850 = `DslPreselectionConfidence`, a hardcoded constant, so the `< 0.7` gate can never trip: an absent
`below_threshold` series on `sentinel_resolver_rule1_decision_total` (`SentinelMeter.cs:1720`) is a property of the
constant, not evidence about the data, and `bucket{le="0.7"}` reads 0 indefinitely.
Not to be re-derived: the `ExtractionSchemaV2 required[]` hypothesis was DISPROVEN by probing vLLM with the shipped
schema, which emitted `resolution_confidence` non-null 5/5.

**A third histogram still carries the SDK default buckets a [0,1] value cannot use.**
`sentinel_chunk_extraction_dedup_ratio` (`SentinelCollector/src/Telemetry/SentinelMeter.cs:299`, unit `{ratio}`) has
no `AddView`, so it keeps the SDK boundaries `[0, 5, 10, 25, ...]` and every observation of a `1 - post/pre` fraction
would land in `le=5.0` — the identical collapse #963 fixed on `sentinel_dsl_adapter_resolution_confidence` and
`sentinel_resolver_rule1_input_confidence`. Nothing is misled TODAY: measured 2026-08-15 UTC, the metric has NO series
in prod (`count({__name__=~"sentinel_chunk_extraction_dedup_ratio.*"})` empty, against the sibling confidence
histogram returning all 16 default-bucket series in the same query shape), because the v2 chunked path is not
emitting. The entry exists so that the first time it does, the collapse is already known rather than rediscovered.
Fix: add the name to the same `AddView` list in `SentinelCollector/src/Program.cs` that already applies
`confidenceBuckets` — the [0,1] boundaries suit a ratio unchanged.

**gemini-resolver runs at 100% of its daily cap while its gate rejects ~1 call in 3,000.** Measured 2026-08-14:
`gemini_resolver_live_calls_24h` 1500 against `gemini_resolver_daily_cap` 1500, `gemini_resolver_gated_24h` 1 of 3,076
calls, and 877 of SecMaster's 3,425 dispatches/24h refused as `cap_exhausted`; refusal is first-come-first-served, so
genuine resolutions are dropped at random once the window is spent. Re-measured 2026-09-16: `live_calls_24h`
**1500/1500**, `gated_24h` 0, `gemini_resolver_cap_refused_total{reason="at_cap"}` **837** since the 11:00Z restart,
`{reason="ledger_unavailable"}` 0.
THE CAP HOLDS -- record this so nobody re-raises it: `ledger_meta.lifetime_live_calls` 38,476 over 29.8 days =
~1,292/day (2026-09-05; 1,324/day on 2026-08-20, so the rate is flat), every restart-to-restart interval at or below
1500, and the full UTC day 2026-08-19 exactly 1500. The gauge is bounded by real enforcement, not a display clamp.
INTENT VIOLATION: the alert's own annotation (`deployment/artifacts/monitoring/alerts/gemini-resolver.yml:328`) says
"A true last resort is dozens/day"; sustained ~1,292/day is **15-100x the documented design intent** -- the CLAUDE.md
INTENT_FIDELITY worked example recurring in the service it was written about. `_company_gate`
(`gemini-resolver-mcp/gemini_resolver/server.py:412-433`) is a SHAPE filter by design (money, markup, code slugs, a
13-entry abbreviation list, >10-word boilerplate), so sectors, cities, currencies, people and product names pass it;
`gated_24h` near zero is consistent with the gate working as specified, NOT evidence it is broken. What is sent (a
failure-biased log sample, qualitative only) includes "tech sector", "Dollar Index", "Donald Trump", "Large Fries".
Re-check -- metric half first; the log half only detects wording drift:
  `curl -s http://localhost:9300/metrics | grep -E 'cap_refused|gated_24h|live_calls_24h'`
  `sudo journalctl -u gemini-resolver-mcp --since "24 hours ago" --utc | grep -c "live reservation refused"`
  # if the log count disagrees with `increase(gemini_resolver_cap_refused_total[24h])`, the WORDING drifted again,
  #   not the refusals (the previous wording "daily call cap" returned 0 for three weeks while 1,852 were counted)
CAVEATS: a Prometheus Counter's `_created` is the exporting PROCESS's age, not the data's (2026-09-16:
`cap_refused_created` decodes to 2026-09-16T11:00:22Z = the unit's `ActiveEnterTimestamp`), so divide lifetime totals by
the ledger file's own age, never by `_created` (that error produced a false "2,803/day, cap not enforcing" once); the
ledger's `call_events` prunes at 48h (`gemini_resolver/ledger.py:39`), so use `ledger_meta.lifetime_live_calls` plus
restart checkpoints for longer windows; open `/opt/ai-inference/gemini-resolver-ledger.db` read-only (`mode=ro`) --
the service is writing it. `journalctl` prints LOCAL time (mercury is `America/New_York`): pass `--utc` and read the
LEFT-hand stamp; the Python logger's own naive local timestamp in the message body is untouched by `--utc`.

**The quarantine refusals are a POLICY nobody has decided, not a constraint.** The partial index turned re-acquiring a
retired ticker from impossible into optional, and `CatalogService` and `EntityResolutionService`'s self-seed now
REFUSE deliberately, because resolution time, per candidate, silently, is the worst place to take a catalog-repair
decision. Decide per PATH whether a quarantined (`is_active=false`) ticker may be re-acquired: an operator-curated
config and a collector registration are authoritative in a way a news-surface self-seed is not.
CONSEQUENCE while undecided: `CatalogService.cs:207` drops a quarantined discovery item with a bare `continue` (a
LogWarning, no metric), so a CompanyName candidate loses its ticker PROPOSAL and every news mention of one of the 82
real quarantined tickers pays the full confirm cascade up to the paid Gemini leg (D-1); also reachable from the
`search_catalog` MCP tool. `quarantined_skip` is a FLOOR on wall-hits, emitted at one of four self-seed skip paths
(`EntityResolutionService.cs:1038`); the other three and the `:206` drop are silent.
POPULATION: 91 quarantined rows = 82 real tickers + 9 macro-junk (the D-4 class) that sit in NEITHER enrichment pool
and need their own disposition (`SELECT asset_class, count(*) FROM instruments WHERE is_active=false GROUP BY 1;` in
`atlas_secmaster` -> Equity 74 / ETF 8 / fred_series 8 / Economic Indicator 1, 2026-09-05).
HAZARD, un-alerted: `idx_instruments_symbol` is `UNIQUE ... WHERE (is_active = true)` but
`idx_source_mappings_collector_source` is STILL UNIQUE on `(collector, source_id)` GLOBALLY with no predicate
(unchanged 2026-09-16), so `RegistrationService.cs:391` raises a 23505 the moment a quarantined row carries a mapping
-- 1 of 91 does (`GSV.NE`) -- and nothing alerts when that stops being harmless.
Re-check: `sum by (result)(secmaster_entity_resolution_self_seed_total)` (2026-09-05: `idempotent_skip` 4625,
`inserted` 126, `quarantined_skip` 32, and no `error` series -- the 23505s are gone) and
  `SELECT indexname, indexdef FROM pg_indexes WHERE indexname LIKE 'idx_instruments_symbol%'
     OR indexname LIKE 'idx_source_mappings_collector_source%';`

**Gemma 4 residuals the swap PR found and did not fix (DEPLOYED 2026-09-07 evening; 1b, 2 and 3 still open)**
[2026-09-07, title corrected 2026-09-13, file count re-measured 2026-09-16]

The Gemma 4 coordinate swap (SentinelCollector/AGENT_README.md D-29) was deployed the evening it merged (first
Gemma request ~22:48Z on 2026-09-07). Item 1 CLOSED 2026-09-07: the weights are staged and `HF_HUB_OFFLINE=1`
shipped in `deployment/artifacts/compose.yaml.j2` (the serving condition all three acceptance boots ran under,
pinned by `ExtractionModelCoordinateTests.should_boot_the_engine_under_the_hub_resolution_the_acceptance_run_used`);
`--revision` is deliberately NOT added (no acceptance boot passed it, so adding it would ADD an axis). Re-check:
  `cat /opt/ai-inference/models/huggingface-cache/hub/models--google--gemma-4-31B-it-qat-w4a16-ct/refs/main`   # 52f3f65bc7a02d555763bc923bd1d9094898219d
  `grep -c '\-\-revision' deployment/artifacts/compose.yaml.j2`      # 0, by decision
  `grep -c 'HF_HUB_OFFLINE=1' deployment/artifacts/compose.yaml.j2`  # 2 since 2026-09-07: one real at :362, one comment at :354

  1b. THE COORDINATE SWEEP IS ONE-DIRECTIONAL. `ExtractionModelCoordinateTests` hunts the INCUMBENT's literals, so
     nothing looks at the sites hardcoding the NEW id: 43 lines across 30 tracked files on 2026-09-07, 40 files on
     2026-09-16 (`git grep -c gemma-4-31B-it-qat-w4a16-ct -- ':!.claude/worktrees' | wc -l`; drop `-c` for lines).
     FOUR are read by a run as a default and are therefore live at the NEXT swap:
     `scripts/sentinel-quality-check/weekly_quality_check.sh:65` (SENTINEL_QC_MODEL),
     `scripts/sentinel-quality-check/compare_base_vs_resolved.py:67` (DEFAULT_MODEL),
     `SentinelCollector/tools/gpu-json-shadow/score_shadow_recall.py:251` (JUDGE_MODEL) and
     `SentinelCollector/agent/agent.py:34` (VLLM_MODEL). All four are env-overridable, which bounds the damage to a
     run nobody overrode and does not remove it. CLOSE by single-sourcing those four from `vllm_base_model`.

  2. `--tool-call-parser hermes` IS QWEN-SHAPED AND SURVIVED THE SWAP (`compose.yaml.j2`). `hermes` parses
     Hermes/Qwen `<tool_call>` output, which Gemma 4 does not emit. Deliberately kept (D-29: extraction sends no
     tools, so it cannot reach the scored path), but the agent REPL's tool-calling leg is UNMEASURED on this model
     and 0 requests have exercised it. Re-check by driving one tool-call through `ToolAugmentedChatClient` against
     the live engine and reading whether `tool_calls` comes back populated or the text arrives unparsed.

  3. THE PROMTOOL FIXTURES CARRY THE OLD MODEL ID ON PURPOSE. 30 sample series in
     `deployment/tests/alerts/vllm_test.yml` label `model_name="Qwen/Qwen2.5-32B-Instruct-AWQ"`; NO rule selects on
     `model_name`, so the fixtures passing unchanged is the evidence that a model swap cannot silence an engine
     alert. They become wrong only if a rule ever adds a `model_name` selector. Re-check (the file's two hits are
     comment prose, so a bare count answers the wrong question):
       `grep -v '^\s*#' deployment/artifacts/monitoring/alerts/vllm.yml | grep -c model_name`   # must stay 0

**D-23's thin-draw gate can never deny, because `.Bind()` APPENDS to a non-empty list default.**
`SearxngIssuerProbeOptions.Engines` initialises to `["duckduckgo", "bing"]`
(`SentinelCollector/src/Configuration/SearxngIssuerProbeOptions.cs:72`) and `SentinelCollector/src/appsettings.json:74` configures the
byte-identical list; .NET's options binder APPENDS to an existing `List<T>`, so the bound list is four duplicates and
`RespondingPinnedEngines` (`IssuerProbeScorer.cs:189`) = 4 - 2 = **2** = `MinRespondingPinnedEngines`'s default (`:85`).
The floor whose purpose is "never judge on a thin draw" sits at its own minimum and passes with ZERO real engines
answering; the guard's own code is correct, the denominator is inflated outside it by configuration. A SECOND inflation
route into D-23, needing no SearXNG involvement. Inert today: `IssuerProbePinVerifier` is registered
(`SentinelCollector/src/DependencyInjection.cs:108`) but no consumer reads a probe verdict; it goes live the moment the probe is wired.
Fix: drop the property initialiser or clear the list before binding -- never raise `MinRespondingPinnedEngines`.
Re-check: a unit test that binds the shipped `appsettings.json` section and asserts `options.Engines.Count == 2` goes
RED today; no test asserts it. Recorded from the PR #947 review 2026-08-15; re-verified on main 2026-08-23 and
2026-09-16 (`:72`, `:85`, `SentinelCollector/src/appsettings.json:74` unchanged).

**THE STATIC-METER FLAKE'S ROOT CAUSE IS FIXED; what is left is two classes that never joined the collection.**
Mechanism: a global `MeterListener` filtered by meter+instrument but NOT by test, plus any class outside
`[Collection("SentinelMeterStatic")]` running in parallel with the listening ones (measured 2026-08-15 as
`Failed: 2, Passed: 2222` then `Failed: 0, Passed: 2224` on an identical tree). #947 added the attribute to the classes
this entry first named. STILL OPEN: `ExtractionProcessorV1SkipOutcomeTests` and
`ExtractionProcessorThinContentOutcomeTests` carry no attribute, so the race is intact for whatever they emit. The fix
is the attribute (or a capture scoped per test), never a re-run -- a re-run goes green ~half the time either way and
sends someone holding a REAL meter regression to press re-run.
Re-check (2026-09-05, reproduced 2026-09-16) from `SentinelCollector/tests/SentinelCollector.UnitTests/Workers`:
  `grep -L 'Collection("SentinelMeterStatic")' ExtractionProcessor*Tests.cs`
  # prints exactly ExtractionProcessorV1SkipOutcomeTests.cs and ExtractionProcessorThinContentOutcomeTests.cs;
  # a THIRD name is a new class that skipped the collection, and it is invisible to a suite run.

**The merge gate refuses a cross-repository merge on the `gh pr merge` route and permits the SAME merge on the REST
route, so the two routes disagree about one threat.** `gh api -X PUT repos/<owner>/<repo>/pulls/<N>/merge` resolves
against THIS checkout's verdict marker whatever owner, repo or HOST the path names. Eleven closed sub-shapes were
deleted from this entry on 2026-09-05; what stays open is one thing — a redirect that lives in the URL PATH reaches
no scan at all. Measured 2026-08-15 on a fixture with #99901 approved at head, #99904 unreviewed, `gh` stubbed and
an isolated `ATLAS_MARKER_DIR`, nothing merged: `gh api -X PUT repos/attacker/evil/pulls/99901/merge` -> **allow**,
the `curl` spelling -> **allow**, `curl -X PUT https://evil.example.com/api/v3/repos/jpansarasa/ATLAS/pulls/99901/merge`
-> **allow**, while #99904 denies on every one — which is what proves the NUMBER is read and the marker consulted is
this repo's. The subcommand route denies all five equivalents, including `--hostname`.
Re-check WITHOUT the fixture, because the mechanism is readable and the decisions are not (run 2026-09-05, re-verified
2026-09-16): the REST patterns `GH_API_MERGE_RE` / `CURL_MERGE_RE` (`.claude/hooks/git-push-guard.sh:369` / `:379`,
located by `grep -n '/pulls/\[0-9\]\+/merge' .claude/hooks/git-push-guard.sh`) scrape the number and nothing else;
`merge_scan_redirects` (`:2131`) inspects only the `-R` / `--repo` / `--hostname` FLAGS; the refusal that WOULD catch a
foreign slug is `MERGE_FOREIGN` (`:2500-2501`). No owner, repo or host is read on the REST route on any tree.
A test encodes the hole as expected behaviour: `run-pr-verdict-smoke.sh` row 2 (`:550`) asserts ALLOW for
`gh api -X PUT repos/o/r/pulls/$PR_A/merge`, and `o/r` is not this checkout --
`bash .claude/hooks/test/run-pr-verdict-smoke.sh` -> rc 0, **239 PASS / 0 FAIL** (2026-09-16), including
`PASS: 2. single REST merge, approved at head -> allow`. The subcommand spelling of the same shape was flipped to deny
(`row allow "3. DECOY gh -R org2/repo7 ..."`); the REST spelling was left.
Why it is still open, and the size of the job: `repos/o/r/pulls` appears on 17 fixture lines across two suites, and
binding the rule on the REST route flips `run-pr-verdict-smoke.sh` row 2 to deny while the chained / no-verdict rows
stop exercising what they were written for. The fixtures must be re-pointed at this checkout's slug in the SAME
change as the rule, or the suite goes green while testing nothing.
Those 17 are 7 in `run-pr-verdict-smoke.sh` and 10 in `run-entry-shape-smoke.sh`; row 58p among them already DENIES,
on the `--hostname` flag rather than the path, and needs no re-pointing.

**TWO CHEAPER FIXES WERE MEASURED AND DECLINED; the smoke rows that keep them declined ARE the record.**
(i) Abandoning the substitution narrowing whenever ANY token carries a substitution closes the twelve `$(... ; ...)`
shapes but re-breaks ordinary chained commands whose substitution belongs to the CHAINED command (`&& grep -Rn TODO
$(pwd)`, `&& cp -R $(pwd)/docs /tmp/x`), refused with the same wrong cause: rows `59h`/`59i` go red. (ii) Denying
whenever the tokenizer ends inside a quote catches rows `61`/`61b` but also ends mid-quote on `57k`, `58s` and `58u` --
ALLOW rows whose spans are cut mid-quote BY DESIGN (`--subject "a|b"`); `57j` is the control showing the flag tracks
the quote rather than the cut.
Recorded, not fixed, zero capability delta against main: an UNQUOTED NEWLINE in a merge command is one token run with
no standalone operator to cut at, so a chained `grep -Rn` is attributed to the merge and denied for the WRONG cause
("names a repository other than the one this checkout tracks" instead of "no recorded review verdict") -- cutting there
cannot be told from a QUOTED newline and would drop tokens from the merge's own segment, a fail-OPEN, so it stays; and
`${x:-)}` / `case y)` each lower the substitution-depth counter a level they never raised, but the redirect still
cannot get through (denied via MERGE_OPAQUE, exactly as their no-redirect control is). Read the refusal CAUSE, never
the decision, when re-checking either.
Re-check: `bash .claude/hooks/test/run-pr-verdict-smoke.sh` -> rc 0, 239 PASS / 0 FAIL (2026-09-05, reproduced
2026-09-16). Either declined fix turns the named rows red, which is the only reason they are named here.

**`ansible-gate-guard.sh` reports a write to a path it INVENTED -- two spellings stand.** The root is path
CONSTRUCTION over command TEXT rather than over the resolved write target, so one fix covers both. Probed 2026-09-05
against the ARMED installed guard, each command fed as JSON on the hook's stdin, none executed (on 2026-09-16 only
the grep-redirect form was re-probed):
STANDS (1): writing a scope line into the bypass file is refused by the gate the scope exists to narrow --
  `printf '%s\n' '.claude/hooks/git-push-guard.sh' > /home/james/ATLAS/.claude/.ansible-gate-confirmed` -> **deny**,
  naming the DATA being written as the target, so only the all-or-nothing `touch` survives, the WIDEST bypass and the
  exact failure the 2026-08-07 scoping change was added to prevent. Workaround: write the fragments elsewhere and `cp`
  them in, so no gate path appears in the command.
STANDS (2): a read verb displaced from segment head turns its gate-path OPERAND into the reported write target
  whenever a stdout redirect sits anywhere in the segment -- `(grep -n x git-push-guard.sh > /tmp/o)` -> **deny**, as
  do the `.claude/hooks/...` and absolute spellings, a `{ ...; }` brace group, a `time`-prefixed grep, and
  `cp <gate file> /tmp/x` under any of the nine `WRAPPER_RE` prefixes (naming the SOURCE; bare `cp` allows). With a
  bare basename the invented path is `/home/james/ATLAS/git-push-guard.sh`, which DOES NOT EXIST; a heredoc body
  carrying that basename plus a redirect is denied the same way (it bit the 2026-09-16 triage itself). Controls:
  group opener removed -> allow, redirect removed -> allow, non-gate file -> allow.
Re-check, from the repo root (the command itself is not refused; the trailing pipe keeps it valid across the break):
`jq -n '{tool_input:{command:"(grep -n x git-push-guard.sh > /tmp/o)"}}' | bash .claude/hooks/ansible-gate-guard.sh |
jq -r .hookSpecificOutput.permissionDecision` prints `deny` today. Fixed, it prints NOTHING -- an allowing guard emits
no JSON, so jq exits 0 with empty output.

**An attempted BLOCK that is refused leaves a prior APPROVE standing, and the merge gate honours it.** Narrow, and
NOT the moved-head case: `.claude/hooks/git-push-guard.sh:2667` compares the marker's sha against the PR's live
`headRefOid` and denies on mismatch, and every branch between the marker read and that comparison is a deny, so a
verdict against a superseded head cannot unblock anything -- the read side is shipped.
WRITE SIDE, reachable: `scripts/claude-pr-verdict` calls `warn_surviving_marker` before every refusal (call sites
`:342`, `:362`, `:483`, `:497`, `:510`, `:523`, `:533`, `:551`, re-verified 2026-09-16) and leaves any earlier
`pr-reviewed-<N>` on disk. Harmless for the head-mismatch refusal, whose surviving marker cannot match the current head
either; but the other refusals -- missing pending record, malformed pending, unreadable `gh`, unparseable timestamp,
and the `MIN_REVIEW_SECONDS` invoke-then-stamp guard -- can fire while a prior approve sits at the CURRENT head. That
approve stays valid, the guard correctly honours it, and the merge proceeds although the reviewer's last action was a
BLOCK. Fix: invalidate or DOWNGRADE the prior approve when a block is attempted and refused; auto-unlink is declined on
purpose (deleting a prior verdict on an unrelated refusal destroys a legitimate record). The script's own warning now
MEASURES the surviving marker's effect instead of claiming it (fixed 2026-08-17); that mis-diagnosis is closed.
MCP PATH IS UNGATED, and the recorded fix shape cannot gate it: the hook is wired at PreToolUse matcher `Bash`, which
matches no MCP tool name, so `mcp__plugin_github_github__merge_pull_request` merges with no verdict consulted (and
`push_files` / `create_or_update_file` write main with no PR) -- all in a dispatched agent's tool set. Deny DOES bind
for `mcp__` matchers (measured 2026-08-07, Claude Code 2.1.224, probe server `server_calls=0`; a metacharacter-free
matcher is compared exactly, so a plain `mcp__plugin_github_github` matcher ships and gates NOTHING silently), which
discharges the 2026-08-06 "deny may not bind" blocker; the residual is one log-only `mcp__.*` hook against the LIVE
server. But the recorded sibling-hook shape reads `owner/repo/pullNumber/branch`, and `merge_pull_request`'s input
carries `owner`, `repo`, `pullNumber` and merge method with NO `branch` (verified against the live tool schema), so it
would cover the file writes and silently no-op on merge. Gating merge needs `pullNumber -> base.ref` resolved inside
the hook's ~5s budget, or a verdict marker keyed by PR number. `.claude/hooks/README.md:404-406` and
`git-push-guard.sh:141-145` still state the superseded rationale (2026-09-16), twenty-six lines after `.claude/hooks/README.md:377-380`
says the regex matcher works; both must cite the 2026-08-07 measurement or state the `branch`-field obstacle.
Scope: the MERGE gate (`pr-reviewed-<N>`) on the Bash path; the PUSH gate is keyed on a TREE hash and never reads
this marker.
Re-check, static and safe: `grep -n warn_surviving_marker scripts/claude-pr-verdict` must list refusal sites OTHER
than the head-mismatch one, AND `git-push-guard.sh:2667` must still compare `MARKER_COMMIT` against `PR_HEAD_COMMIT`.
If the second ever stops being true the moved-head case re-opens and this entry is wrong.

**The Bash path and the Edit/Write path do NOT apply the same rules, despite the header saying they do.**
`ansible-gate-guard.sh:189-190` claims "ONE definition, consulted by the Edit/Write path AND the Bash path, so the
two can never drift apart". That is true of `is_gate_path` and `is_deployed_path` and false of `GATE_BASENAMES`,
which is read at exactly two sites (`:570` and `:1006`, in `check_token` and `prefix_span`), both on the Bash path.
The direction is safe -- Bash is the stricter path -- but the bare-basename rule is the only rule that follows a
guard's NAME out of the gate layer, and a docstring claiming coverage it does not have is the defect moved into the
tool. The fixture recorded 2026-08-17 (`cp /tmp/a /tmp/scratch/ansible-gate-guard.sh`) no longer discriminates:
since #1035 an ABSOLUTE path is excluded from the bare-basename rule (`:567-568` sets `bare=0`), so it ALLOWS on both
paths. Measured 2026-09-16 with a RELATIVE spelling, guard sited beside the hook set so `GATE_BASENAMES` is
populated: Bash `cp /tmp/a scratch/ansible-gate-guard.sh` DENIES while an Edit naming
`scratch/ansible-gate-guard.sh` ALLOWS. Re-check: those two inputs through one guard copy sited beside the hook set;
they must agree, or the header must stop claiming they do.

**`run-wiring-smoke.sh` is RED and has been since 2026-08-08, and the red is the suite's OWN stale list.**
Re-run 2026-09-16: rc 1, 52 PASS, exactly one FAIL reading `registered set drifted:6a7 > dream-pending-notice.sh`,
ending `WIRING SMOKE: FAIL` (identical to 2026-09-05). READ THE DIFF DIRECTION BEFORE ACTING ON IT: the check diffs
`EXPECTED_WIRED` against `ACTUAL_WIRED` in that order, so a `>` line is present in ACTUAL and missing from EXPECTED --
the hook IS registered in tracked `.claude/settings.json:170`, and it is the suite's hardcoded list
(`run-wiring-smoke.sh:65`) that never learned about it. Registration landed 2026-08-08 in #936.
The cost is not the one red row: the suite exits 1, so its other 52 assertions sit behind a failing summary, and
anything gating on rc reads the whole suite as broken rather than as one stale line. A permanently red suite teaches
its readers to skip it, which is what lets the NEXT drift through. Decide the direction rather than silencing the
row: either the dream notice is a wired participant and belongs in `EXPECTED_WIRED`, or it should not be registered.
Re-check: `.claude/hooks/test/run-wiring-smoke.sh; echo rc=$?` -- rc must be 0 and the summary `WIRING SMOKE: PASS`.

**A write in one tool call and its execution in the NEXT are invisible to any command-string guard. ACCEPTED LIMIT,
not an open bug -- nothing in a future round can close it.** `ansible-gate-guard.sh` is a `PreToolUse` hook handed
ONE `tool_input.command`: `echo cp /tmp/evil /opt/ai-inference/compose.yaml > /tmp/run.sh` in call 1 and
`bash /tmp/run.sh` in call 2 are two strings, neither containing the other's half. Recorded because the shape looks
like a defect to every reviewer who meets it, and three rounds of #974 were spent on designs that promised to cover it.
WHAT COVERS IT: denying call 1. Since #974 the guard checks echo/printf operands whenever the segment's stdout lands
in a file, whatever happens to that file afterwards (`ddbaff89`'s rule, restored deliberately). Re-checked
2026-09-16: `echo cp /tmp/evil /opt/ai-inference/compose.yaml > /tmp/run.sh` DENIES on its own, no second segment.
DISCLOSED OPEN, DELIBERATELY (2026-08-17, still ALLOW 2026-09-16): an operator is recognised only where it STARTS a
token, so the redirect WELDED to a word (`echo cp /tmp/evil /opt/ai-inference/compose.yaml>/tmp/run.sh`) and the
noclobber exec-binding form (`>| /tmp/f exec; echo cp /tmp/evil /opt/ai-inference/compose.yaml; bash /tmp/f`, where
`split_segments` cuts `>|` in half as a pipe) still allow. The fix is an edit to `split_segments`, the tokenizer
every other mechanism depends on, and that edit's risk exceeds the gap, a deliberate-evasion spelling nobody writes
by accident. The welded-operator widening is priced: one new denial (`echo we should review
/opt/ai-inference/compose.yaml>/tmp/run.sh`) and zero suite failures.
Do NOT close it by reintroducing a target-is-later-executed design (`6c276949` carried one and leaked three ways in a
single string), and do NOT revert to `ddbaff89`'s rule: its deny on these shapes is a redirect-blind blanket deny of
any string naming a guarded path (it denies `echo cp /tmp/evil /opt/ai-inference/compose.yaml` with no redirect at
all), which is the false denial this branch exists to remove. "Main denies, head allows" is not by itself a
loosening; the baseline for a loosening is the commit BEFORE the change, never main.
Re-check (feed as INPUT to the guard, never execute -- each writes the path it names): the covering spelling must
deny; the two disclosed-open spellings (`exec>/tmp/run.sh`, operator welded to the preceding word, and
`\exec > /tmp/run.sh`, spaced, escaped by the backslash) allow today. The welded-operator arm flips only the
welded write and `exec>/tmp/run.sh`; `\exec` needs its own arm, and `>|` belongs to a third spelling
(`>| /tmp/f exec`).

**Two PRs green apart, `main` red together: the push gate keys to a TREE, so a cross-PR interaction
is structurally invisible to it** [2026-09-07]

`.claude/hooks/git-push-guard.sh` matches `v2 tree <hash>` against the PUSHED BRANCH's tree. A squash-merge
result is a tree no branch ever had and no `compile.sh` ever ran on it, so N PRs green in parallel can land a red
`main`, and the first to notice is the next agent the repo-wide push gate blocks. FIRST measured occurrence:
`ExtractionModelCoordinateTests` shipped RED on `main` at 56e6344d (19/20, the chat-template sweep naming 9 tracked
files). #1037 (665ba78e) added 12 `LlmBenchmark/eval-substrate/commoncoord-*.scorecard.json` sidecars, 9 hitting the
chat-template needle (a SUBSTRING match, so any template extending ChatML matches); #1038 (56e6344d) added the sweep
from a branch cut before #1037 merged, so the tree `compile.sh` attested held the sweep and no scorecards. Neither
PR was wrong alone. The instance is fixed (#1039, allowlist symmetry). THE GAP IS OPEN, and this is its FIRST
measured occurrence: a SECOND promotes it to `.claude/skills/supervisor-mode/LESSONS.md` per CLAUDE.md
WHERE_WORK_LANDS.

Re-check -- the interaction reproduces from git alone, and the gap's own measurement is a zero:
  `git log --diff-filter=A --format=%h -- 'LlmBenchmark/eval-substrate/commoncoord-*'`   # 665ba78e
  `git log --diff-filter=A --format=%h -- 'SentinelCollector/tests/SentinelCollector.UnitTests/Configuration/ExtractionModelCoordinateTests.cs'`   # 56e6344d
  two DIFFERENT introducing commits = no pre-merge tree could have run the sweep over the scorecards.
  `grep -l dotnet .github/workflows/*.yml | wc -l`   # 0 -- nothing re-runs the .NET suite on `main` after a merge
A fix is either a post-merge run of the touched projects' `compile.sh` on `main`, or a merge queue that attests
the MERGE RESULT's tree instead of the branch's; both are unbuilt.

**Three unfixed edges on the coordinate sweep, none of them the allowlist bug** [2026-09-07]

Found reviewing PR #1039 and deliberately left out of it. All three are PRE-EXISTING, none is a live hole today,
and each fails toward looking fine.

  1. A DOCSTRING CLAIMING COVERAGE IT DOES NOT HAVE. The comment above `PRODUCTION_CHAT_TEMPLATE_HINT`
     (`LlmBenchmark/scripts/run_model.py`) says the hint "is swept against the shipped value by SentinelCollector's
     `ExtractionModelCoordinateTests`". Nothing sweeps it: the constant holds a Gemma 4 value, so NEITHER incumbent
     needle matches it, and no case in that class reads `run_model.py` at all; if it drifts, a post-deploy run
     scores an arm nobody measured. CLOSE by adding a case that reads the literal out of `run_model.py` and asserts
     it against `ExtractionOptions.ChatTemplate`, or by deleting the sentence. Re-check:
     `grep -rln PRODUCTION_CHAT_TEMPLATE_HINT --include='*.cs' SentinelCollector/`   # 0 = still open

  2. BOTH SWEEP FAILURE MESSAGES SELL "ADD IT TO THE ALLOWLIST" AS UNCONDITIONAL. Each closes with "add the file
     to `Incumbent{Id,Template}IsHistoryIn` with the reason it is evidence", never saying the entry must be EARNED:
     the file has to still CONTAIN that list's own needle or
     `.should_still_find_the_incumbent_in_every_path_the_allowlist_exempts` turns one RED case into two. CLOSE
     with one clause in each message. Re-check, against
     `SentinelCollector/tests/SentinelCollector.UnitTests/Configuration/ExtractionModelCoordinateTests.cs`:
     `grep -c 'with the reason it is evidence' <that file>`   # 2, neither qualified = still open

  3. THE `LlmBenchmark/eval-substrate/` EXEMPTION IS A DIRECTORY PREFIX WIDER THAN ITS STATED REASON ("scorecards
     ARE the measurement artefact"): the prefix also covers `cod-stage1.criteria.json` -- `DEFAULT_COD_CRITERIA_PATH`
     (`LlmBenchmark/scripts/eval_harness.py`, lines 102-103), the LIVE scoring thresholds -- so a future coordinate
     literal there is exempt SILENTLY. THE TRAP: narrowing to the `commoncoord-` prefix is measurably WRONG --
     `qwen25-32b-awq-vllm-20260903` and `qwen25-32b-awq-unquantkv-vllm019-20260904` are scorecards carrying 2 id
     hits each and would stop being exempt, taking the id sweep RED. The wanted predicate is the `.scorecard.json`
     SUFFIX, which `Sweep`'s `StartsWith` cannot express, so CLOSE means giving an allowlist entry a suffix or
     exclusion predicate beside its prefix, not editing the string. Re-check -- count each needle (the two
     constants declared in the test class) against `cod-stage1.criteria.json`: `0` and `0` today; either going
     non-zero means the prefix has started hiding a live value.

**PR #1035's review found more than the three defects that were fixed; the rest were scoped out**
[2026-09-07]

#1035 landed (23a43af5, under a human bypass scoped to the two files) three fixes to
`.claude/hooks/ansible-gate-guard.sh` with their rows in `.claude/hooks/test/run-advisory-guards-smoke.sh`: operand
CONCRETENESS on the `update-index|checkout-index` and `clean -x` arms (opaque operands had dissolved to deny -> ALLOW),
apostrophes in the jq deny program routed through `--arg`, and the deny message naming `run-entry-shape-smoke.sh`
(the only suite that reads `PUSH_GUARD_HOOK`).
RE-CHECK: `bash .claude/hooks/test/run-advisory-guards-smoke.sh` -> 482 assertions, rc 0.

SCOPED OUT of that round, each re-derived in this session against HEAD, none fixed:
  a. `test-audit.sh` cannot see the LIVE roster SHRINK -- the original bug's own class. Deleting
     `AlphaVantageCollector` from `CLAUDE.md` `## SERVICES` gives rc 0, 73 PASS, 0 FAIL, and prints
     `PASS [live roster parses: 10 services]` (was 11) asserting that count against nothing. Shape
     that would work: assert the live set is a SUPERSET of the 11 the fixtures declare.
  b. `enumerate-services.sh:43` `if (seg ~ /^(mcp|shared):/) continue` skips the WHOLE segment.
     `mcp: OfrCollector` -> 10 services, rc 0, EMPTY stderr, OfrCollector silently gone. Fail closed.
  c. `enumerate-services.sh:44-46` a bare identifier with no role prefix is emitted AS a service:
     `Reports` on its own line -> 12 services, rc 0, `Reports` in the output. Fail closed.
  d. `enumerate-services.sh:65` `^[A-Za-z0-9]+$` forbids hyphens, so a hyphenated service dir could
     never join the roster. Undecided, not a defect yet.
  e. `build-deploy-hint-selftest.sh` case 6 uses `grep -q -- 'ansible-playbook.*--tags'` over the
     WHOLE FILE, so the header COMMENT satisfies it: delete both `echo` hint lines from
     `AlertService/.devcontainer/build.sh` and 0 hint echoes remain while the predicate still matches
     1 line -> 6/6 pass. Reuse `scan()`'s own line predicate plus a not-a-comment test.
  f. `build-deploy-hint-selftest.sh:105-138` `f1..f4="$(mkfixture ...)"` under `set -uo pipefail`
     with no `-e`: mkfixture's fail-closed `exit 1` kills only the subshell, the var goes empty and
     `scan ""` reads nothing and returns clean. Three "must NOT be flagged" cases can pass on zero
     bytes. Needs `|| exit 1` and a pass-count floor.
  g. Two guard mutations survive `run-advisory-guards-smoke.sh` at 470/470 GREEN, so no row covers
     either: dropping the `..` exclusion from the `bare` test (`cp /tmp/evil
     /tmp/scratch/hooks/../git-push-guard.sh` -- HEAD deny, mutant none) and dropping `--all` from
     the `--index-info|--stdin|--all` arm (`git checkout-index --all -- foo` -- HEAD deny, mutant
     none).
  h. `scripts/tests/build-deploy-hint-selftest.sh` is committed `100644` while its sibling
     `new-epic-selftest.sh` is `100755` -- and recording exactly that bit is what the `--chmod`
     narrowing in item 1 exists to permit.
  i. The PR body claims `ports.yml` "is now reconciled with a set-equality control". There is none:
     `grep -rln 'ports.yml' scripts/ deployment/tests/ .github/` is empty.
  Also unfixed and NOT in that review: `run-advisory-guards-smoke.sh` has no assertion-count floor,
  `pin()`/`json_ok()` ignore `ATLAS_GATE_HOOK` (so six assertions describe the repo's guard under the
  workflow the deny message prescribes), and `json_ok` green-lights EMPTY output on rows labelled
  `(deny)` -- empty output being the exact signature of item 2.

MEASURED AND WORTH KEEPING: the apostrophe defect in item 2 has TWO forms and only one is visible to
anything dynamic. Restoring the exact shipped spelling (`'ask'`, no space in the interruption) leaves
the suite 482 PASS / 0 FAIL while the emitted message silently loses the quotes -- source and output
already differed on HEAD. One keystroke further (`'ask decision'`) the word splits, jq fails to
compile, stdout is ZERO BYTES, no decision, ALLOW: 157 assertions RED. A source-level control (no
single-quoted jq program may contain a bare apostrophe) catches both and was built and proven RED on
HEAD in this session, then dropped by a scope cut; it is not in the patch.

**GOLD DEFECT: 55% of `entity_ticker_accuracy`'s scored cases are labeller world-knowledge, not
extraction** [2026-09-07]

Of the 53 entities in `cod_stage1_gold_v1.json` carrying a `ticker`, **29 have a ticker that appears
nowhere in their article** -- the labeller supplied `Nvidia -> NVDA`, `Microsoft -> MSFT`,
`Delta Air Lines -> DAL`, `JPMorgan Chase -> JPM` from world knowledge. Only 24 are tickers a model
could read off the page.

The CoD prompt asks for transcription: *"include ONLY if the ticker is stated in or directly
resolvable from the article; otherwise omit."* Entity resolution is SecMaster's job and resolves
from the NAME, so a model omitting `MSFT` on an article that never printed it is obeying the prompt
at no cost to the pipeline. For the majority of scored cases the metric therefore ranks PRETRAINING
RECALL of ticker symbols.

The ordering is the tell: the worst extractor measured (EXAONE, `numbers_f1` 0.2218) has the BEST
ticker score at 0.6369, and the best extractor (Gemma 4, 0.7570) is fourth from bottom at 0.4101.

  ✗ barred as a swap criterion -> `CLAUDE.md` §MODEL_ACCEPTANCE
  ✗ removed from `LlmBenchmark/BENCHMARKS.md` -- it is a gold defect, not a benchmark result
  FIX: score only the 24 article-stated cases, which would be a real extraction metric. Not done,
    so the current number cannot be read in either direction.

Values as measured, kept only so the fix has a before: EXAONE 0.6369 | incumbent 0.5504 |
Gemma 3 0.5455 | Command-R 0.4640 | Gemma 4 0.4101 | Qwen3.8 0.3725 | Mistral 0.3311 | GLM 0.1667.
Where a model does emit a ticker it is not wrong, only absent -- across 15 runs on Qwen3.8 the
wrong-ticker count is 0.

**A worktree-isolated agent CANNOT do gate-layer work at all: the two guards deadlock.** [2026-08-15; the
develop-elsewhere route re-measured 2026-09-07] `ansible-gate-guard.sh` refuses every write under `.claude/hooks/**`
and names one sanctioned escape, a bypass file at `$CLAUDE_PROJECT_DIR/.claude/.ansible-gate-confirmed`, ideally
scoped by path fragment. But `project_dir` is `"${CLAUDE_PROJECT_DIR:-$PWD}"` (`.claude/hooks/ansible-gate-guard.sh:104`)
with NO worktree handling, so for an agent in `.claude/worktrees/<id>/` it resolves to the SHARED checkout -- which
worktree isolation then refuses to let that agent write ("Edit the worktree copy of this file instead") -- and the
worktree copy is never read. Measured 2026-08-15: creating `<worktree>/.claude/.ansible-gate-confirmed` activates the
bypass when the guard is invoked by hand from the worktree and does nothing for the real hook, which reported the
shared path in its deny message on every attempt. Net effect: PR #970's own review findings (three guard fixes in
`git-push-guard.sh` plus rows in two suites) could be measured but not landed. Options, cheapest first: honour a
worktree-local bypass by resolving `project_dir` through `git rev-parse --show-toplevel` before falling back to
`CLAUDE_PROJECT_DIR`; or have the supervisor dispatch gate-layer work non-isolated.
THE DEVELOP-AND-VERIFY-ELSEWHERE ROUTE IS OPEN. The gate matches a path FRAGMENT, so it closes only scratch paths that
themselves contain `.claude/hooks`. Measured 2026-09-07 from a worktree with no bypass file, scratch root
`/tmp/claude-1000/<session>/scratchpad`: `cp -r .claude/hooks <scratch>/hookstree` ALLOWED and `python3 <script>
<scratch>/hookstree` writing the copies ALLOWED, so a full patched tree can be built and its suites RUN (the guard's
own deny text now teaches these steps). Two over-denials remain: `cp .claude/hooks/git-push-guard.sh
<scratch>/git-push-guard.sh` DENIES on the DESTINATION basename (a `.txt` destination is allowed), and
`python3 <script> .claude/hooks/git-push-guard.sh <b>` DENIES although every path after the script name is read-only
argv. Neither is worth a rule change alone; what they cost is the operator's belief that the route is shut.
Re-check: from a worktree, attempt any edit to `.claude/hooks/git-push-guard.sh` after creating a scoped
`<worktree>/.claude/.ansible-gate-confirmed` -- it must be allowed. And with no bypass, `cp -r` the hooks dir to a
scratch path containing no `.claude/hooks` fragment and run one of its suites -- both must succeed.

**`ansible-gate-guard` denies READS, RUNS and even PROSE about a gate-layer file, while permitting an
actual WRITE to one.** Its deny text states the contract exactly -- "RUNNING one of these files is NOT
blocked -- only writing to it ... redirecting its output to a log is fine" -- and both halves are
inverted in practice, because it matches the command TEXT rather than the act. Second instance of the
class in `.claude/skills/supervisor-mode/LESSONS.md` L14; first one recorded against THIS guard.

MEASURED 2026-09-04 during #1002's compile.sh work, target
`SentinelCollector/.devcontainer/compile.sh`, one Bash tool call per form:

- DENIED, a READ: `sed -n '34,63p' <target> > /tmp/.../control-block.sh`. The redirect target is a
  scratchpad file; the gated path appears only as the input.
- DENIED, a RUN logged the way the deny message itself recommends:
  `nohup bash -c 'bash <target> > /tmp/.../full-compile.log 2>&1; ...' &`. The unnested
  `bash <target> --no-test > /tmp/.../build-nostest.log 2>&1` is ALLOWED, so what it refuses is the
  `bash -c '...'` nesting, not the act.
- DENIED, PROSE: the Bash call adding THIS ENTRY to `docs/BACKLOG.md` was refused, because the entry
  text quotes the forms above. Already-encoded remedy (LESSONS.md ALREADY_ENCODED, the quoted-push
  line): pass long text by path, which is how this entry landed.
- ALLOWED, a real WRITE: `python3 /tmp/.../gate-probe.py`, whose body opens the gated path and calls
  `write_text` on it. It succeeded, undenied. Every compile.sh edit in #1002 went in that way.

Re-check: run those four forms. If this entry is still true, the read, the nested run and the prose
are refused and the python write succeeds. The probe writes byte-identical content, so a re-check
leaves no diff.

CONSEQUENCE, both directions. The write path means the gate does not enforce what it claims: a
gate-layer change lands without the deliberate, visible confirm-file step that IS the mechanism. The
read/run/prose path costs working forms an agent then has to route around -- which is how a guard
teaches people to reach for the bypass, and creating the confirm file is the USER's decision, never
an agent's (`.claude/skills/guard-change/SKILL.md` item 14), so the only honest alternative is to stop.

Do NOT close this by teaching it the `bash -c` and `python3` spellings. That is the
converging-on-a-reimplementation-of-bash approximation that `guard-change` item 9 tells you to
PRICE IT OUT LOUD rather than ship, and the prose case shows the grammar is unbounded. Close it by
gating the ACT -- the tool call's resolved write target -- rather than the spelling of the command
line.

**`check_staleness.py`: two endpoints reporting the SAME engine name collapse last-wins, so the
verdict flips on argument order.** `main()` builds `live_versions[engine] = version` in a loop, and
a second endpoint answering the same engine name silently overwrites the first. One ordering is
fail-OPEN, and nothing warns.

MEASURED 2026-09-04 against two local stubs both identifying as `vllm`, one serving 0.19.0 and one
0.28.0, over a scorecard recorded at vLLM 0.19.0:

```
--endpoint <0.19.0> --endpoint <0.28.0>   ENGINE_MOVED  "vllm 0.19.0 -> 0.28.0"   0/1 current  rc 1
--endpoint <0.28.0> --endpoint <0.19.0>   CURRENT       "matches live; 0d old"    1/1 current  rc 0
```

Today's documented topology (vLLM :8000 + llama.cpp :8080) reports two DIFFERENT engine names and
does not trigger it, which is why the tool's own `--endpoint ... --endpoint ...` example is safe.
It triggers the moment two vLLM endpoints are compared -- exactly what an upgrade evaluation does,
and the version under evaluation is the one an operator naturally passes second.

Re-check: run two stubs answering `/version` with different values, point both `--endpoint` flags at
them in each order, and compare the verdicts. If this is still true the two orders disagree.

Do not close it by picking a winner. Two live builds under one engine name is an AMBIGUITY, and the
rule this file's own tooling now follows in three places is that ambiguity DENIES -- the honest
verdict is a refusal naming both endpoints, not a coin flip.

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

**THREE live sites outside CLAUDE.md still teach the retired `MODEL_SIZE >= 30B` floor, and one of them is
PRODUCTION CODE.** [2026-09-06, re-counted 2026-09-16] The ROOT `README.md` asserts ">=30B-parameter models" for
Sentinel extraction and links the CLAUDE.md section that RETIRED the floor; `docs/SENTINEL-RLM.md` carries a
`### Model Size (30B+)` heading, a table cell and a `## Do Not` bullet; `SentinelCollector/src/Services/ExtractionService.cs:35`
carries `// CoVe (extraction) always runs on GPU - requires >=30B model`. This MISLEADS rather than merely lagging:
the floor was retired because it is a PROXY that would have BLOCKED a real upgrade (a 27B model beats the 32B
incumbent in the MODEL BASELINES table), so a reader who lands on any of these declines the upgrade the measurement
favours. Close by replacing each with MODEL_ACCEPTANCE's actual bar -- a scorecard on production's prompt path that
beats the incumbent's -- not by deleting the numbers. THE `.cs` EDIT IS A SEPARATE PR: a comment change in a service
pulls in a full SentinelCollector compile, and a docs PR may not carry it.
Re-check: `grep -c 30B README.md docs/SENTINEL-RLM.md SentinelCollector/src/Services/ExtractionService.cs` -> 1, 4, 1
on 2026-09-06; 1, 3, 1 on 2026-09-16, all three sites still open.

**`docs/BACKLOG.md` has no out-flow that RUNS, and it grows.** `scripts/new-epic.sh` audits STATE.md and LESSONS.md
at the epic boundary and refuses the reset on a finding; this file has no such audit -- its out-flow is a human
closing an entry in the PR that fixes it, which is the mechanism that produced the stale count. Measured: 3,849 lines
with 21 entries reading as live on 2026-09-05 (at `c32354f7`), 4,187 on 2026-09-06, 6,470 on 2026-09-16. The 21 is an
AUDIT count, NOT a grep -- nothing here is labelled stale, which is precisely what "reading as live" means -- so
re-deriving it means re-reading the entries. Re-check: `wc -l docs/BACKLOG.md`, then re-read.

**Metric prefix inconsistency from a single service:** `sentinel_candidate_surface_*` vs
`sentinelcollector_semantic_signal_*`.
Re-verified 2026-09-16: both prefixes still present in `SentinelMeter.cs` (6 hits); a rename breaks dashboards and
alerts, so it needs a deliberate PR.

**TWO uncorrected copies of the Finnhub transient-only claim, not four.**
`SecMaster/src/Configuration/EnrichmentOptions.cs:14` and `SecMaster/src/Services/CatalogEnrichmentBackgroundService.cs:187`
both call "Finnhub 403 for foreign tickers" a TRANSIENT enrichment failure. A 403 is plan-uncovered and permanent, and
it arrives as a NULL profile rather than an exception -- which `SecMasterMeter.cs:243` and
`SecMaster/src/Services/IFinnhubCollectorClient.cs:33` already say correctly, so this is a finished conversion with
two comment fixes left.
Re-check (2026-09-05, reproduced 2026-09-16):
  `grep -rn '403' --include=*.cs SecMaster/src FinnhubCollector/src | grep -i transient`
  -> four hits today: the two above are the debt; `FinnhubApiClient.cs:29` and `:406` state the permanent/transient
  split CORRECTLY and are the controls, not the debt.

**THE SENTINELCOLLECTOR CARD IS 5.3x OVER ITS OWN D-ENTRY GATE, AND ITS LINE COUNT HIDES IT.**
`SentinelCollector/AGENT_README.md` measured 2026-09-16: 275,956 bytes, 136 non-blank lines, 32 D-entries (207,010 /
101 / 28 on 2026-09-07 -- a third larger in nine days). `CARD_TEMPLATE.md` sets the gate at "card <= ~1 page / ~55
non-blank lines" and ">~6 entries = smell (scope creep dilutes the signal)". The LINE ratio is the lying one: the
longest single line is 32,788 characters (top three 32,788 / 22,393 / 22,389), so a density gate counted in LINES
cannot see it -- a card can be driven arbitrarily far past "~1 page" without moving the metric, just by not pressing
Enter. Any future check must be on BYTES. NOT A SILENT GAP -- a documented one:
`.claude/skills/architecture-cards/scripts/audit.sh:13-14` says the smell is "deliberately LLM-scope (intent-review /
human judgment), not script-checked", so the audit passes this card and always will. The reading agent is the only
enforcement, and it is the party the card was supposed to serve: at 276KB the card has become a second codebase, and
whether agents given it read, skim or truncate it is unmeasured.
  FIX (not done, and a judgement call, not a mechanical split): decide which D-entries are still exception paths /
    scarce-resource boundaries / non-obvious preconditions and which are ordinary mechanism that accreted, then move
    the catalog to `README.md` §Reference per the template's own escape hatch. Superseded entries are rewritten in
    place, never tombstoned.
  Re-check:
```
f=SentinelCollector/AGENT_README.md
echo "bytes=$(wc -c < $f) nonblank=$(grep -c '[^[:space:]]' $f) D=$(grep -c '^  D-' $f)"
awk '{print length($0)}' $f | sort -rn | head -3
```
2026-09-16 -> `bytes=275956 nonblank=136 D=32`, longest lines `32788 22393 22389`.

**SIX DEPLOYED DIRECTORIES HAVE NO CARD AND SIT OUTSIDE THE ROSTER, SO THE SERVICE_ARCHITECTURE HARD_STOP CANNOT
REACH THEM** [NEEDS A HUMAN DECISION -- recorded, not decided]
`CLAUDE.md` §SERVICE_ARCHITECTURE is a HARD_STOP: read `{Service}/AGENT_README.md` before reasoning about a service.
Its reach is the §SERVICES roster. (The roster claim "every service in SERVICES has one at that exact path" was FALSE
on 2026-09-07 because an `mcp:` row named six card-less paths; `e5447167` removed that row the same day and the claim
is true again as of 2026-09-16 -- the sole off-roster card is `gemini-resolver-mcp/AGENT_README.md`.)
Six directories that are DEPLOYED are not in the roster at all, so the HARD_STOP cannot reach them even in principle
(re-checked 2026-09-16: none has an `AGENT_README.md`):
| directory | compose services it ships |
|---|---|
| `WhisperService/` | `whisper-service`, `whisper-service-mcp` |
| `Reports/` | `reports-daily`, `reports-weekly`, `reports-monthly` |
| `FinBertSidecar/` | `finbert-sidecar` |
| `markitdownMCP/` | `markitdown-mcp` |
| `SentinelCollector/dsl-parser-mcp/` | `dsl-parser-mcp` |
| `edge/sentinel-edge/` | none -- Cloudflare Worker, deployed by the `sentinel-edge` ansible tag |
All six have a plain `README.md` except `edge/sentinel-edge/`, which has neither. That is 8 of the 31 services in
`/opt/ai-inference/compose.yaml`. THE DECISION IS NOT MINE AND NOT AN AGENT'S. Three coherent answers, and the
roster's current silence is none of them:
  (a) they join §SERVICES and owe cards -- the most work, the widest HARD_STOP coverage;
  (b) the roster's scope is stated explicitly (e.g. "services with their own data model; sidecars and MCP servers
      inherit their parent's card");
  (c) MCP sidecars are declared card-exempt by the same reasoning §OBSERVABILITY already uses for them ("MCP
      sidecars deliberately rely on parent-service telemetry"), leaving only `WhisperService/`, `Reports/`,
      `FinBertSidecar/` and `edge/sentinel-edge/` owing cards.
Until one is chosen, the HARD_STOP reads as covering the whole deployment and does not, which is the failure mode a
HARD_STOP is least able to survive.
  Re-check: `ls */AGENT_README.md` against the §SERVICES roster and against the service list in
    `/opt/ai-inference/compose.yaml`.

## MEASUREMENT DEBT [instruments that cannot report their own dullness]

Harnesses, golds and scorecards whose blind spots are known and unfixed.

| impact | measured | status | entry |
|---|---|---|---|
| A | 2026-09-06 | OPEN | Three watermark-only ReExtract legs re-assert the row's tier; nothing pins that they do |
| B | 2026-09-16 | OPEN | Alert rules and their metrics ship on different schedules; rule-first pages a healthy system |
| B | 2026-09-16 | OPEN | QuoteStalenessSeeder resolves its repository OUTSIDE the try: DI failure kills startup |
| B | 2026-09-16 | OPEN | The staleness-origin fallback degrades the dead-man SILENTLY -- no log line, no metric |
| B | 2026-09-16 | OPEN | repetition_penalty 1.1 (not the cap) costs entities_recall 0.5529 -> 0.4348 on the production prompt |
| B | 2026-08-30 | OPEN | containerd exporter never reports an OOM kill (worked around, not fixed) |
| B | 2026-08-17 | OPEN | FredCollector writes to an UNBOUNDED channel nobody reads |
| B | 2026-08-17 | OPEN | The two remaining collector dead-man alerts may be as blind as Finnhub's was |
| C | 2026-09-16 | OPEN | The `source_entity` divergence guard: three things it does not pin |
| C | 2026-09-16 | OPEN | The Azure oracle ledger overstates spend -- exactly 3.0x on the largest run, 2.79x overall |
| D | 2026-09-16 | OPEN | run_model.py --schema-file silently bypasses SCHEMA_REQUIRED on the model-acceptance path |
| D | 2026-09-16 | OPEN | run_model.py diverges from production sampling: --repetition-penalty, --stop, --min-p |
| D | 2026-09-16 | OPEN | Empty-but-valid results grade CORRECT in the sentinel quality check |
| D | 2026-09-16 | OPEN | "stub" is an unpinned cross-file contract that fails OPEN |
| D | 2026-09-16 | BLOCKED | CI is advisory, not blocking |
| D | 2026-09-16 | OPEN | Three figures the PR-verdict decision check leaves un-re-checkable |
| D | 2026-09-16 | OPEN | Citations in tracked .md that cannot land are the corpus's steady state, and so is rc 1 |
| D | 2026-09-16 | OPEN | A conflicted index path makes the alerts selftest report a nonexistent permissions defect |
| D | 2026-09-16 | OPEN | verify-citations.py reports GREEN on a citation that has drifted onto the WRONG line |
| D | 2026-09-16 | AWAITING-DECISION | The documented citation sweep is .md-ONLY, so a line shift rots citations it cannot see |
| D | 2026-09-16 | OPEN | verify-citations.py skips off-allowlist extensions, and bare :NN continuations sans --bare |
| D | 2026-09-16 | OPEN | Patches drop the exec bit, core.fileMode=false hides it, a disarmed hook fails SILENTLY |
| D | 2026-09-16 | OPEN | Stage 2 has a gold now, and it prices three things a comparison must clear |
| D | 2026-09-16 | OPEN | The CoD gold cannot yet back a model swap: macro-owner DECIDED, the key's swing is not |
| D | 2026-09-07 | OPEN | The replicate sd is a within-serve-session statistic; the between-session term is unpriced |
| D | 2026-09-06 | OPEN | llama.cpp /v1/completions silently IGNORES response_format: check content, never status |
| D | 2026-09-06 | OPEN | production_prompt_path is blind on prompt content and on sampling; the card cites it bare |
| D | 2026-09-06 | OPEN | The anti-invention clause FAILS, referential integrity is blind to it, the guard hides it |
| D | 2026-09-05 | OPEN | Nothing checks whether the SHIPPED gold still states the `source_entity` convention |
| D | 2026-09-05 | OPEN | A well-formed 2xx envelope whose content is not an answer still passes silently |
| D | 2026-09-04 | AWAITING-DECISION | check_staleness.py never recurses: a nested scorecard is unopened, not stale |
| D | 2026-08-27 | OPEN | Golden corpus: its controls mutate the FIXTURE, none mutates the publisher SUT |
| D | 2026-08-17 | OPEN | A test class missing [Collection(StalenessGaugeCollection.Name)] re-opens a data race |
| D | 2026-08-15 | OPEN | 2,138 assertions that no suite-level count can see |
| E | 2026-09-16 | OPEN | Alertmanager's warning-route repeat_interval does not pace a Grafana-managed alert |
| E | 2026-09-16 | OPEN | 3 memory citations cannot land, all of one irreparable class |
| E | 2026-09-16 | OPEN | ThresholdEngine builds a channel per event type that has neither a reader NOR a writer |
| E | 2026-09-16 | OPEN | Nine real-but-wrong citations stand in SentinelCollector/AGENT_README.md, all GREEN |
| E | 2026-09-16 | OPEN | ~35 file:line citations in THIS FILE point at real-but-wrong content; sweep reads GREEN |
| E | 2026-09-07 | OPEN | MODEL BASELINES, aggregate_f1 on the v6.2 substrate (six of seven rows name no sampling) |
| E | 2026-09-06 | OPEN | colibri cannot produce an admissible scorecard (DO-NOT-BUILD, permanent): seed refused |
| E | 2026-09-06 | OPEN | Precision ladder on gemma-3-27b: Q6_K LOSES to Q4_K_M, and vLLM has no rung above 4-bit |


### The three watermark-only ReExtract legs re-assert the row's tier, and nothing pins that they do [2026-09-06]
`ExtractedObservation.ApplyReExtraction`'s `newSecMasterMethod` became REQUIRED in #1030, so the next
caller must CHOOSE a value. It cannot make them choose the right one. The three watermark-only skip
legs -- null `RawContent`, zero extractions, empty `Description`, at
`SentinelCollector/src/Workers/ReExtractBackgroundService.cs:384`, `:456` and `:601` -- pass
`observation.SecMasterMethod` to re-assert the row's OWN tier, and that VALUE CHOICE is the thing
standing between a no-op re-extract and silent erasure of the column. Measured 2026-09-06: zero
assertions on `observation.SecMasterMethod` anywhere in `ReExtractBackgroundServiceTests.cs`, and no
test drives any of the three skip paths for tier preservation. Someone clearing a compile error at
`:369` by typing `newSecMasterMethod: null` reinstates exactly the bug the required parameter exists
to prevent, on rows the worker touches routinely, with all 2,436 SentinelCollector unit tests green.
The comment at the call site is currently the only thing holding that line.
NOT closed in #1030 deliberately -- that round's brief authorised no new guard tests, and a test
written to satisfy a review round is the shape this repo has already measured as decorative. The fix
is one unit test on the cheapest of the three legs (empty `Description`, `:583`): a row carrying
`SecMasterMethod = "VectorSearch"` and a blank description, run the resolve-only pass, assert the
tier SURVIVES and `ReExtractedAt` is stamped. It is green today and REDs when the argument is
swapped to `null` -- name that mutation in the test, because the axis is the VALUE, not the arity.
Re-check (no database, no engine): `grep -c 'SecMasterMethod\.Should()' SentinelCollector/tests/SentinelCollector.UnitTests/Workers/ReExtractBackgroundServiceTests.cs`
-- 0 means still open. Count ASSERTIONS, not mentions: a bare `grep -c SecMasterMethod` on that file
already returns 1 today, matching the `newSecMasterMethod: null` ARGUMENT this entry's own PR added
at `:606`, so the obvious predicate reads CLOSED while the hole is open. That draft shipped in this
entry for one revision.

### Alert rules and their metrics ship on different schedules; rule-first pages a healthy system [2026-09-16]
**Alert rules and the metrics they read ship on different schedules, and rule-first pages a healthy system.**
`FinnhubCollectorQuoteCollectionStalled` carries an `absent()` leg and ships via `--tags monitoring`, while the gauge
it watches ships inside the container image. Deploy the rule first and `absent()` is true from the moment
Prometheus loads it, so a healthy collector pages 15 minutes later; deploy the image first and the worst case is a
few minutes of an unwatched gauge. Sequenced image-first BY HAND for this PR, which is exactly the kind of
knowledge that does not survive. NOT fixed here: the durable fix is ordering (or gating) inside `deploy.yml`.
Re-check: `deployment/ansible/playbooks/deploy.yml` must either template rule files after the service image is
running, or refuse a rule whose metric is absent from `/api/v1/label/__name__/values`.
Re-verified 2026-09-16: `deploy.yml` carries no `__name__/values` or `absent()` gate; the monitoring-tag tasks
template rule files independently of the image.

### `QuoteStalenessSeeder` resolves its repository OUTSIDE the try, so a DI failure is fatal to startup [2026-09-16]
**`QuoteStalenessSeeder` resolves its repository OUTSIDE the try, so a DI failure is fatal to startup.**
`FinnhubCollector/src/Workers/QuoteStalenessSeeder.cs` calls `GetRequiredService<IFinnhubRepository>()` before the
`try`, so a resolution failure throws out of `StartAsync` and takes the host down — where every other failure in this
seeder is deliberately Warning-and-continue, because collection does not depend on the seed. Theoretical when written (2026-08-17:
`FinnhubRepository`'s constructor took only a `FinnhubDbContext`), and `FinnhubCollector/src/Program.cs:144` would already have thrown on
a dead database. It stops being theoretical the moment that constructor grows a dependency. NOT fixed here: moving it
inside the try changes the startup failure mode of a service that is currently wedged in prod, and this PR's job is
to make the wedge visible. Re-check: the `GetRequiredService` call must sit inside the `try` block, or
`FinnhubRepository`'s constructor must still take exactly one parameter.
RE-CHECK TRIPPED 2026-09-16: `FinnhubRepository`'s constructor now takes TWO parameters (`FinnhubDbContext`,
`ILogger<FinnhubRepository>`) and the `GetRequiredService` call still precedes the `try`, so the entry's own
close condition has fired. The added dependency is a trivially-resolvable logger, so the failure stays
theoretical in practice, but the arity guard the re-check leaned on is gone. Re-check is now the first half
only: the `GetRequiredService` call must sit inside the `try` block.

### The staleness-origin fallback degrades the dead-man SILENTLY -- no log line, no metric [2026-08-17]
`FinnhubMeter.ProcessStartTicksOrNow` (`FinnhubCollector/src/Telemetry/FinnhubMeter.cs`) catches a failed
`Process.StartTime` read and returns `DateTime.UtcNow.Ticks`. The catch is correct (an escaping exception
becomes a CLR-cached `TypeInitializationException` that kills every meter in the process), but the fallback
then measures staleness from METER INIT rather than process start, understating it by the whole boot --
unbounded here, since no `lock_timeout` is set on the migration connection -- in the direction that makes
`FinnhubCollectorQuoteCollectionStalled` LESS likely to fire, with no log line or metric distinguishing the
degraded origin from a healthy one. `Log.Logger` IS available at that point (`Program.cs` sets it before
`AddMeter`), so the silence is a choice, not a constraint. Measured 2026-08-17; re-verified 2026-09-16: the
`catch (Exception)` body still contains no `Log.` call.
Re-check: force the catch (make `readProcessStart` throw) and confirm nothing is emitted and no metric
distinguishes the meter-init origin from a process-start one.

### repetition_penalty 1.1 (not the cap) costs entities_recall 0.5529 -> 0.4348 on the production prompt [2026-09-06]
OPEN DEFECT. Restated for a reader's scan at KNOWN DEFECTS ("Production's CoD extraction loses ~74
gold entities per run to its own loop guard, TODAY"); the measurement and the next step live here. The
loop guard is not free and nothing in production measures its price. Measured 2026-09-06 on the
then-incumbent (Qwen2.5), the 40 gold articles, three runs per cell, the SAME prompt either side so
the only axis is sampling: 1.1 against no penalty costs `entities_recall` **0.5545 -> 0.3809** on the
pre-#1017 prompt and **0.5529 -> 0.4348** on the prompt production runs today (`git hash-object
/opt/ai-inference/prompts/cod/cod_json_v1.txt` = `85187c39` = the repo blob, verified 2026-09-16) -- roughly 74 of
the gold's 624 entities per run that production does not extract -- for +0.06 precision. #1017 partially RECOVERS
the loss: at matched production sampling it buys +0.0540 recall and +0.0478 `entities_f1` for -0.0196 precision, so
deploying it made the cost smaller, not larger. A prompt effect's SIGN can depend on the sampling regime (the
`entities_f1` delta was -0.0337 at harness sampling and +0.0478 at production's), so compare prompts at production
sampling only. UNMEASURED ON GEMMA 4, which is what production serves now.
IT IS THE PENALTY, NOT THE CAP, on those 40: every guarded record finished `stop`, `truncated` 0, and
the largest completion was 3148 tokens. The cap still earns its place on the other side (unguarded,
nine records over eight runs exceeded 4096), and the guard cannot simply be removed: `repetition_penalty
1.0` at the same cap still saturates article 183's `events[]` at `maxItems` with `finish_reason: stop`
(the two-sided wire control in the anti-invention entry), and 2.0 collapses the output entirely. Only
1.0, 1.1 and 2.0 were ever probed; nothing between 1.0 and 1.1 has been measured.
THE KNOB IS UNCHANGED: `/opt/ai-inference/compose.yaml` carries `CpuCod__JsonRepetitionPenalty=1.1`
beside `CpuCod__JsonMaxCompletionTokens=8192` -- D-30 raised the cap on 2026-09-13 and did not revisit
the penalty.
THE NEXT MEASUREMENT, never run: a 1.02 / 1.05 / 1.1 sweep on the 40-article harness (~2 minutes a
run, three runs a point, ~18 minutes for all three points), scored on the SATURATION and COINED-NAME
observables of the anti-invention entry as well as on recall. Lowering 1.1 toward 1.0 walks back
toward the arm where article 35 emits 60 entities and ~50 coined names, so a point that maximises
`entities_recall` and reopens the runaway is a regression the scorecard would have no number for.
FOLLOW-UP: `CpuCodOptions.JsonRepetitionPenalty` is load-bearing for PROMPT CORRECTNESS and nothing at
the code says so -- `SentinelCollector/src/Configuration/CpuCodOptions.cs` carries `INTENT(D-30)` on the
CAP only, and the penalty's XML doc justifies the value on loop-breaking and latency alone. Per
CLAUDE.md INTENT_FIDELITY it needs a D-entry on the SentinelCollector card with an `// INTENT(D-n):`
at the option, so an agent tuning it for recall meets the constraint at the code.
Re-check (the config half; the 2026-09-06 scorecards under `/tmp` are gone, so the numbers need 12
runs across four cells, ~25 minutes at concurrency 6, written where the tree can reach them):
```
grep -n 'JsonRepetitionPenalty\|JsonMaxCompletionTokens' /opt/ai-inference/compose.yaml   # 1.1 / 8192
grep -n 'INTENT(D-' SentinelCollector/src/Configuration/CpuCodOptions.cs                     # cap only, until the D-entry lands
```

### containerd exporter never reports an OOM kill (worked around, not fixed) [2026-08-30]
`container_memory_oom_total` is exported as a GAUGE read from the live cgroup's `memory.events`,
so a restart tears the cgroup down and the value returns to 0. It is therefore incapable of
reporting the event it is named for.

MEASUREMENT, re-checkable: `max_over_time(container_memory_oom_total[7d])` is 0 for EVERY series,
across two real spacy-ner OOM kills (2026-08-24, 2026-08-28) and one induced deliberately at
2026-08-30T13:24:15Z, at a 15s scrape interval with a 20m lookback. If this entry is still true,
that query still returns 0 everywhere after a kill you can see in `dmesg -T | grep -i "out of
memory"`.

WORKED AROUND in #1000 by `ContainerRestarted`, which detects the CPU-counter reset a restarted
cgroup produces (`resets(container_cpu_usage_usec_microseconds[15m]) > 0`). That covers the
restart, NOT the distinction between an OOM kill and any other in-place restart -- an operator
still has to read dmesg to tell them apart, and a container with `restart: no` that stays dead
produces no reset at all and so is invisible to both rules.

CLOSE THIS by finding a signal that reports the kill itself: a newer containerd exporter that
emits a monotonic counter, a node-exporter textfile collector fed from the kernel log, or a small
systemd unit tailing `dmesg`. Do not close it by pointing at `ContainerRestarted`.

### FredCollector writes to an UNBOUNDED channel nobody reads [2026-08-17]
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

### The two remaining collector dead-man alerts may be as blind as Finnhub's was [2026-08-17]
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

### The `source_entity` divergence guard: three things it does not pin [2026-09-06]
`build_cod_gold.py`'s gate is what stands between a diverged labelling instruction and a paid rebuild of
the gold; three gaps remain, each run against the shipped file on 2026-09-06 and each re-checkable below
with no corpus and no key.

**1. Clause TEXTS are unpinned -- only the NAMES are.** `EXPECTED_CLAUSE_NAMES` is the independent
witness that stops `SOURCE_ENTITY_CLAUSES` being emptied, and what it compares is names. Shrink a
clause's SENTENCE to any substring the three sites still contain -- the single word `issuer` -- and
all eleven controls stay green while the guard has stopped watching the rule. It then accepts an
INVERTED producer: an adjudication instruction reading "the country owns it" draws 1 complaint under
the shipped tuple and 0 under the gutted one, so the paid call proceeds on an instruction that
teaches the opposite of the prompt. Gutting a clause to `""` IS caught -- the mutation then removes
nothing and its three site controls fail, `selftest: 8/11` at rc 1 -- so the hole is precisely a
surviving SUBSTRING of the real sentence, not any edit to the tuple.
```
python3 - <<'PY'
import sys; sys.path.insert(0, 'LlmBenchmark/scripts')
import build_cod_gold as b
rule = b.extraction_prompt_source_entity_rule()
inverted = rule.replace("the SERIES owns it", "the country owns it")
sites = {n: (inverted if n == "adjudication_instruction" else rule)
         for n in b.EXPECTED_SITE_NAMES}   # this ORDER is pinned too -- reorder and the shape check fires
print("shipped clause tuple:", len(b.source_entity_rule_divergence(sites)), "complaint(s)")
b.SOURCE_ENTITY_CLAUSES = (("macro_series_is_the_owner", "issuer"),) + b.SOURCE_ENTITY_CLAUSES[1:]
print("clause gutted to 'issuer':", len(b.source_entity_rule_divergence(sites)), "complaint(s)")
print("selftest rc:", b.selftest_source_entity_rule())
PY
```
2026-09-06 -> `shipped clause tuple: 1 complaint(s)`, `clause gutted to 'issuer': 0 complaint(s)`,
`selftest: 11/11 controls behaved as required` and `selftest rc: 0`. A `1` on the second line closes it.

**2. Site PROVENANCE is unpinned.** The guard asks whether the three sites AGREE, never whether each
is still the text its name claims. Replace `source_entity_rule_sites()` with three echoes of
production's own bullet -- no `adjudication_prompt()`, no `alignability()` output -- and the selftest
prints `11/11` at rc 0. The nine mutations are cut from the prompt and replicated under each site
name by design, so they cannot tell an echo from a producer, and the live check compares each site's
text against the clause list rather than against what produced it. A producer that stopped being
called at all therefore reads as a full pass.
```
python3 - <<'PY'
import sys; sys.path.insert(0, 'LlmBenchmark/scripts')
import build_cod_gold as b
rule = b.extraction_prompt_source_entity_rule()          # every site an echo of the prompt
b.source_entity_rule_sites = lambda: {n: rule for n in b.EXPECTED_SITE_NAMES}
print("selftest rc:", b.selftest_source_entity_rule())
PY
```
2026-09-06 -> `selftest: 11/11 controls behaved as required`, `selftest rc: 0`. A non-zero rc, or a
control naming the echo, closes it.

**3. No control asserts that the `main()` wiring exists.** The gate that refuses the paid call is
seven lines in `main()` -- `divergence = source_entity_rule_divergence()` and its refusal -- and
`--selftest` returns from `main()` before reaching them, so deleting the block leaves every control green
(measured 2026-09-06: `gate deleted: 7 lines`, then `selftest: 11/11 controls behaved as required`, rc 0).
Nothing else in the repo names the function: `grep -rln source_entity_rule_divergence --include='*.py'
--include='*.sh' --include='*.yml' --include='*.cs' .` -> `build_cod_gold.py` alone (unchanged 2026-09-16).
Close with a control that costs nothing: the divergence check runs BEFORE `args.work.mkdir` and
`load_corpus`, so driving `main()` with a diverged site, `--corpus /nonexistent/corpus.json` and a work dir
that does not exist returns `rc = 2` with `REFUSED: the labelling instruction and production's prompt
disagree about source_entity.` on stderr and creates nothing -- measured 2026-09-06, no key and no request.

### The Azure oracle ledger overstates spend -- exactly 3.0x on the largest run, 2.79x overall [2026-09-04]
`SentinelCollector/scripts/azure_oracle_client.py` shipped with Anthropic LIST prices (15/75) and was
corrected to Foundry's (5/25) at `e38f0c56`, 2026-05-02 -- a week AFTER the largest run. Rows before it are
exactly 3x high and rows after are correct, so the ledger is a MIXTURE and no blanket divisor repairs it;
the per-record key is `cost_est`; `cost_usd` belongs to `v7-labeled-opus47-*.jsonl`, where 9,715 of 9,720 rows are
off by exactly 3.0000 ($1,288.73 recorded vs $429.72 actual) -- rescale before quoting either. The ledger is FROZEN (mtime 2026-07-02; Foundry access revoked
2026-09-04), so the defect cannot grow, but it is the only record of past spend and supervisor-mode
estimates still anchor to it -- rescale with the snippet before quoting any figure from it.
Re-check (read-only, no DB):
```
python3 - <<'PY'
import json,collections
F={"claude-opus-4-7":(5,25),"claude-opus-4-6":(5,25),"claude-sonnet-4-6":(1,5),"claude-haiku-4-5":(.25,1.25)}
r=[json.loads(l) for l in open("/opt/ai-inference/training-data/azure-oracle-ledger.jsonl") if l.strip()]
a=lambda x:x["input_tokens"]/1e6*F[x["model"]][0]+x["output_tokens"]/1e6*F[x["model"]][1]
print("rows",len(r),"recorded",round(sum(float(x["cost_est"]) for x in r),2),"actual",round(sum(a(x) for x in r),2))
print(collections.Counter(round(float(x["cost_est"])/a(x),4) for x in r if a(x)).most_common(4))
PY
```
2026-09-04 and again 2026-09-16 (identical): `rows 22026 recorded 1346.53 actual 483.49` and
`[(1.0, 12292), (3.0, 9730), (3.1828, 3), (2.8061, 1)]`. The 1.0 bucket IS the mixture; the two small
buckets are accounted for, not defects (3 haiku rows at rounded prices; one aggregate row carrying `calls: 70`).

### `run_model.py --schema-file` silently bypasses `SCHEMA_REQUIRED`, on the model-acceptance path [2026-09-04]
`build_payload` reads `schema = load_schema(args) or extraction_json_schema()` and `load_schema` returns
whatever JSON the operator handed `--schema-file`, verbatim; nothing between there and the wire checks that
the supplied schema's `required` covers `SCHEMA_REQUIRED`. The flag is not exotic: the SentinelCollector card's
MODEL_ACCEPTANCE (`SentinelCollector/AGENT_README.md:12`, the `production_prompt_path: true` conjunct) requires a
named schema file in the invocation a model swap must produce a scorecard from, so the bypass sits
on the path that decides whether a candidate replaces the incumbent. The cost is already measured on this
harness (comment block above `SCHEMA_REQUIRED`): `certainty` optional in the request schema -> emitted on
0 of 2,213 extractions while gold carries it on 5,111 of 5,111, `certainty_accuracy` -0.85 against
threshold, read as "the model is bad at certainty" when it was the request schema.
MEASURED 2026-09-04 (unchanged 2026-09-16) against the schema file that conjunct requires (`cod_json_schema_v1.json`):
```
python3 - <<'PY'
import json, sys; sys.path.insert(0, 'LlmBenchmark/scripts')
import run_model as rm
req = set()
def walk(n):
    if isinstance(n, dict):
        req.update(n.get('required', []))
        for v in n.values(): walk(v)
    elif isinstance(n, list):
        for v in n: walk(v)
walk(json.load(open('SentinelCollector/src/cod-prompts/cod_json_schema_v1.json')))
print(sorted(set(rm.SCHEMA_REQUIRED) - req))
PY
-> ['certainty', 'period', 'text_quote']
```
CLOSE by making `validate_request_shape` refuse a `--schema-file` whose `required` does not cover
`SCHEMA_REQUIRED`, in the fail-closed shape it already applies to `--chat-template-kwargs` in completions
mode; never by merging the supplied schema into the derived one. Pin it with
`test_run_model.should_exit_two_when_a_schema_file_omits_a_graded_field` driving `main()`, paired with a
covering-schema positive; `should_send_the_supplied_schema_when_a_schema_file_is_given` asserts the bypass
verbatim and is a contract, not coverage.
Re-check: the snippet still prints `['certainty', 'period', 'text_quote']`, and `grep -n SCHEMA_REQUIRED
LlmBenchmark/scripts/run_model.py` shows no reference inside `load_schema` or `validate_request_shape`.

### `run_model.py` still diverges from production's sampling on `--repetition-penalty`, `--stop` and `--min-p` [2026-09-04]
Three divergences from production remain on the harness that scores model acceptance (`--seed` landed
2026-09-05; the eight scorecards in `LlmBenchmark/eval-substrate/` that predate it carry no `seed` key and
nothing can retro-fit it). `--repetition-penalty` defaults to `None` while production always sends
`CpuCodOptions.JsonRepetitionPenalty` = 1.1, worth -0.17 `entities_recall` on the prompt production runs
(the "Production's `repetition_penalty` 1.1" entry). `--stop` exists but defaults to none while production
always sends `ExtractionOptions.StopTokens`. `--min-p` forwards `min_p` into the vLLM payload though
`VllmCompletionRequest` has no such field (`ExtractionOptions.MinP` is documented "llama.cpp min_p" and only
`LlamaServerClient` sends it), so a `--min-p` run records a knob the engine never read. `--max-tokens` does
NOT belong on this list: default and production are both 8192 (D-30).
Re-check (grep the symbols, never a line number):
```
python3 -c "import json;print(sorted(json.load(open('LlmBenchmark/eval-substrate/qwen25-32b-awq-vllm-20260903.scorecard.json'))['adapter_metadata']['sampling']))"
grep -n 'ap.add_argument("--stop"' -A 3 LlmBenchmark/scripts/run_model.py
grep -rn -B 7 'public float MinP' SentinelCollector/src/Configuration/ExtractionOptions.cs
```
2026-09-16 -> `--repetition-penalty` default `None`; `--stop` still `action="append", default=None`; `min_p`
still forwarded; `MinP` still documented llama.cpp-only. A `--stop` default carrying production's tokens, or
`min_p` dropped from the vLLM payload, closes the remaining halves.

### Empty-but-valid results grade CORRECT in the sentinel quality check [2026-09-16]
**Empty-but-valid results grade CORRECT.** A schema-valid `qualitative_result` with `sentiment_polarity:
"unspecified"`, zero sectors, zero regimes and confidence 0.1 skips EVERY rubric rule on full-length content and is
graded CORRECT as a GRADED row — a full draw then reports `0.0% MAJOR, SHIP_FULL`, the best possible reading.
Already 2.1% of graded rows (3 of 146). Root cause: the rubric is a violation detector, so emptiness is
indistinguishable from perfection. Rubric design, not plumbing.
Re-verified 2026-09-16 in `scripts/sentinel-quality-check/ab_scorecard.py`: the default verdict is CORRECT and
every rule gates on sentiment not in (None, 'unspecified'), so an empty `qualitative_result` still skips them all.

### "stub" is an unpinned cross-file contract that fails OPEN [2026-09-16]
**`"stub"` is an unpinned cross-file contract that fails OPEN.** Writer `scripts/sentinel-quality-check/compare_base_vs_resolved.py:781` ->
reader `scripts/sentinel-quality-check/ab_scorecard.py:129` (citations re-verified 2026-09-16; `"stub"`
appears in neither test file). A writer-side rename returns `usable_nonstub` to 6 on a dead run with zero tests
going red — the same shape as the false-green it was introduced to close.

### CI is advisory, not blocking [2026-09-16]
**CI is advisory, not blocking** — branch protection 403s on this GitHub plan, so a red run does not stop a merge.
Re-verified 2026-09-16: `gh api repos/{owner}/{repo}/branches/main/protection` -> HTTP 403 "Upgrade to GitHub
Pro or make this repository public". This is why every merge gate in this repo is a hook, not branch protection.

### Three figures the PR-verdict decision check leaves un-re-checkable [2026-08-17]
Measured 2026-08-17 while fixing that check; none is a defect in it, and each drifts with nothing going red.
(1) POPULATION unpinned, RESULT pinned: `run-pr-verdict-smoke.sh` BD18 sweeps every `#NNN` in this file and
asserts the refusing set is exactly {729, 935}, but nothing asserts how many numbers were swept -- 30 when
stamped, 55 on the pre-triage file (`40d372eb`) and 51 on this file after the 2026-09-16 triage (b08ccdef), nothing red in
between. Fix: a BD row asserting the population beside the set.
Re-check: `grep -oE '#[0-9]+' docs/BACKLOG.md | sort -u | wc -l`.
(2) Rounds and recorded verdicts are different quantities and only one is mechanical: PR #978's body says
seven review rounds on #974; `~/.claude/atlas-pr-verdict.log` holds six verdict records for it (still 6 on
2026-09-16). "rounds" has no store behind it. Re-check: `grep -c 'PR#974' ~/.claude/atlas-pr-verdict.log`.
(3) Which `git-push-guard.sh` deny answers which shape is asserted by no test. Every prose shape reaches an
EARLIER deny (span/prefix mismatch, direct-to-main, unknown-branch) because both push-count derivations read
the same `git ... push` text; the two-pushes-merged deny naming the `git commit -F` / `gh pr create
--body-file` pair is reachable only via a newline inside a quoted option value (measured 2026-08-17).
`supervisor-mode/LESSONS.md` ALREADY_ENCODED cites which denies say what, so reordering a rule silently
changes the message a blocked reviewer reads. Re-check: put `git push origin <a real branch> # git push
origin main` and the newline shape through the hook and read which deny answers each.

### Citations in tracked `.md` that cannot land are the corpus's steady state, and so is rc 1 [2026-09-06]
Reproduce: `mapfile -d '' F < <(git ls-files -z '*.md'); python3 scripts/verify-citations.py --quiet "${F[@]}"`
-- 198 files / 497 citations / 28 cannot land at `5d42ce9f` (2026-09-06), rc 1; 202 / 572 / 28 at `40d372eb`
(2026-09-16). The floor is composed of ambiguous basenames a `--scope` would resolve, deliberately-broken
fixtures (the architecture-cards `weak-card` test's citation must stay broken) and blank landings; it moves
with every doc edit and the tool cannot see content drift, so it is a FLOOR on rot, never a census. Never pipe
it through `xargs -0`: xargs remaps the child rc to 123, so the rc above is unobservable through that form.
Judge a PR by whether its cannot-land SET is a subset of its base's and diff the LANDING TEXT; equal counts
are not a pass (502 and 502 on both sides of #1026 while eleven citations moved), and stamp any figure with
the MERGED sha, never a branch. Each instrument has a disjoint blind spot: a diff-computed handover sees only
citations whose target MOVED; a cannot-land check sees only those hitting NOTHING; the residue -- already
wrong, never moved, lands on something -- is visible only to a reader comparing prose against target.
THIS ENTRY DELIBERATELY CARRIES NO `file:line` FORMS -- the tool parses its own backlog entry.

### A conflicted index path makes the alerts selftest report a permissions defect that does not exist [2026-08-17]
**A conflicted path in the index makes the alerts selftest report a permissions defect that does not exist.**
`deployment/tests/alerts/selftest.sh:519` (re-verified 2026-09-16) reads a file's mode with `git ls-files -s -- "$f" | cut -d' ' -f1`, which
assumes ONE row per path. During an unresolved merge `git ls-files --stage` returns stages 1/2/3, so the mode
variable becomes the mangled `100755\n100755\n100755`, fails the `= "100755"` comparison, and the control prints
`a #! file is not executable in the index` — naming a permissions failure for a file whose permissions are fine.
Measured 2026-08-17 during the #975/#973 merge: the suite scored **41/42** with the conflict unresolved and
**42/42** the moment the resolutions were staged, with no file mode touched in between. It is a conflict-state
artifact misreporting as a permissions defect, and it misleads in the expensive direction — an agent mid-merge is
told to run `git update-index --chmod=+x` on a file that needs nothing. Fix: take the last field, or filter to
stage 0. Re-check: run `selftest.sh` with any conflicted path in the index and read the FAIL line.

### `verify-citations.py` reports GREEN on a citation that has drifted onto the WRONG line [2026-09-06]
The tool resolves `file.cs:NN` and confirms line NN exists; it cannot confirm NN is still the line the prose
meant, so a green run is evidence a citation points at a line that EXISTS, never the RIGHT one. Measured
2026-08-17: a 6-line comment edit in `FinnhubCollector/src/Telemetry/FinnhubMeter.cs` shifted five D-2 GUARD
citations in `FinnhubCollector/AGENT_README.md` by six lines each; the tool flagged exactly ONE (the one that
landed on a blank), the count moved only 91/2 -> 94/3, and rc was 1 on both sides.
NARROWED, NOT CLOSED [2026-09-04, #1002]: a citation whose prose names `D-n` within 8 lines and which LANDS on
a line beginning `D-m` is reported `WRONG-D-ENTRY`, counted apart from cannot-land (`grep -c WRONG-D-ENTRY
scripts/verify-citations.py` -> 2 on 2026-09-16). The FinnhubCollector case is STILL LIVE AND GREEN: `.cs`
GUARD citations land on method declarations where no `D-n` appears, and demanding one would condemn every
GUARD citation in this repo's cards. CONSEQUENCE (CLAUDE.md TOOL_UPKEEP, LESSONS.md L8): any edit shifting
line numbers in a cited file requires a comparison of LANDING TEXT against a pristine merge-base checkout;
equality of the `N checked, M cannot land` line is NOT a pass (488/27 matched its base while four drifted;
`a8a0ed5d` matched 195/496/28 exactly while three did).

### The documented citation sweep is `.md`-ONLY, so a line shift rots citations it cannot see [2026-09-05]
**The documented citation sweep is `.md`-ONLY, so a line shift rots citations it structurally cannot see.**
Same shape as the `--memory` corpus gap above -- the resolver is fine, the CORPUS is wrong -- and this one is
inside the tracked repo, so nothing signals it. The documented invocation is
`mapfile -d '' F < <(git ls-files -z '*.md')`, and citations also live in `.py` and `.cs` COMMENTS, where the
tool's own `_EXTS` list would happily resolve them if a sweep ever handed them over. Measured 2026-09-05 on
PR #1004, which shifted `SentinelCollector/src/Workers/ExtractionProcessor.cs` by +10 lines above the
dependency-outage guard and +35 below it: the `.md` sweep reported 488 checked / 27 cannot land, a set identical
to its base's, while FOUR citations in two non-`.md` files had silently drifted onto real-but-wrong lines --
`SentinelCollector/scripts/build_golden_corpus.py:184` (`:875-877` and `:2138-2140`) and `:195`, and
`SentinelCollector/tests/.../GoldenCorpus/GoldenCorpusFixture.cs:172-173` (`:901-903` and `:2163-2165`), every
one of them a two-cite pair spanning the v1 and v2 sector/instrument gates. All four were repaired in that PR and
verified by content, not by rc. Reproduce the blind spot:
`git ls-files -z | xargs -0 grep -nE '\w+\.cs:[0-9]' | grep -v '\.md:'` -- **53 sites across 20 files**,
re-run 2026-09-05; **58** on 2026-09-16, still with nothing red. This entry said "currently 4": a 13x understatement that grew with nothing going red, and
the missed set includes `deployment/artifacts/compose.yaml.j2`, the gate-layer template this entry itself
calls the one that bites. One of the 53
(`dream/index_hooks.py:43`) is an EXAMPLE inside a regex doc comment and must never be "repaired", which is why
this is a corpus question with a judgement in it rather than a flag to flip. Closing it means either extending
the documented invocation and pricing in that example class, or a `--scope`-style opt-in; both change what every
future sweep reports and need their own before/after counts, exactly as the `_EXTS` entry below says.

### `verify-citations.py` skips citations outside a 10-item extension allowlist, and bare `:NN` unless `--bare` [2026-08-17]
Two gates in `scripts/verify-citations.py`, anchored to symbols because every line number this entry once
carried had rotted. (1) `_EXTS = "py|cs|yml|yaml|md|sh|json|csproj|ts|sql"` feeds `CITATION`; the `\.` is
mandatory, so an extensionless path or an off-list extension never matches. DELIBERATE (the comment above
`_EXTS`: a permissive `\w+\.\w+:\d+` also swallows version strings and `host:port`), so a precision/recall
tradeoff already priced in -- widening it changes what every sweep reports and needs its own before/after
counts. (2) The `BARE` regex parses a continuation like ``(`:147`)`` only under `--bare`; the docstring says
leave it off by default, so a default run checks none of them. Live cost, measured 2026-08-17 on base
`21b02a58` (190 tracked docs): ZERO citations name an extensionless file (the `scripts/claude-*` exposure is
latent), but 5 citations resolve to a real file and are invisible -- one in
`deployment/artifacts/compose.yaml.j2`, one in `.gitignore`, one `.gbnf`, two `/opt/ai-inference/prompts/cod/*.txt`.
The `.j2` bites: gate-layer templates whose line numbers move. `_EXTS` unchanged on 2026-09-16. Pairs here are
deliberately in NON-citation form so this entry cannot itself rot invisibly.
Re-check: sweep every tracked `.md` for `path:NN` tokens that resolve to a real file but do not match
`CITATION`, and confirm the count and extension breakdown before trusting a sweep that reports "every one lands".

### Patches drop the exec bit, `core.fileMode=false` hides it, and a disarmed hook fails SILENTLY [2026-08-17]
Four mechanisms compose into a defect with no symptom; each alone is survivable. (a) A patch applied to this
tree has twice landed its touched files non-executable on disk (both 2026-08-17; the second left two LIVE
hooks disarmed until repaired by hand). (b) `core.fileMode=false` is set, so `git status` is CLEAN and
`git diff` emits no mode line -- the index must be inspected explicitly. (c) `.claude/settings.json` execs
every hook as a BARE PATH (18 entries, 0 prefixed `bash`), so a 644 hook is an exec failure the harness
swallows: the guard simply never speaks again. (d) plain `git add` records a NEW file as 100644 regardless of
disk mode (probed 2026-08-17: disk 755, `git add` -> 100644, `git add --chmod=+x` -> 100755). SCOPE: only
`.claude/hooks/**` and the two extensionless `scripts/claude-*` executables need the bit; interpreter-invoked
scripts sitting 100644 are CORRECT, and a shebang-based sweep teaches the reader to ignore the check. Repair
is `git add --chmod=+x -- <path>` (`git update-index --chmod=+x` is DENIED by `ansible-gate-guard.sh` for ANY
path); `chmod` on disk fixes nothing that ships.
Re-check reads the INDEX; silence is the pass (silent on 2026-09-16, `core.fileMode` still false):
`git ls-files -s -- .claude/hooks scripts/claude-pr-verdict scripts/claude-mark-verified | awk '$4 !~ /\.md$/ && $1 != "100755" { print "NOT 100755 IN THE INDEX:", $1, $4 }'`
CONTROL: pipe one fabricated `100644 <sha> 0<TAB>.claude/hooks/git-push-guard.sh` line into that same `awk`;
it must name that file back. Verified both ways 2026-08-17.

### Stage 2 has a gold now, and it prices three things a comparison must clear [2026-09-07]
`--task cove` is the metric every `aggregate_f1` in this repo is quoted in. `LlmBenchmark/cove-gold/`
is its gold on the 40-article corpus: **420 items over 40 articles, 8 with no figures**, labelled on
production's stage-2 prompt path (`initial_extraction.txt` + `ExtractionSchema.cs`) at production's
stage-2 decoding (16,384 completion tokens, no `repetition_penalty`, temperature 0 -- stage 1's cap and
1.1 penalty are a different call and are NOT carried across). Two arms, different families, neither the
then-incumbent's nor the swap candidate's; $0.5099 over 101 requests. Inter-arm item-set f1 **0.785**
(P 0.754 / R 0.819) on 272 matched pairs. `description` agrees at **0.357** -- NOISE, same class as
CoD's free-form `event_kind`/`claim_kind`; do not score it. 4 of 271 `confirmed` stamps (1.5%) name a
figure the cross-check did not produce at that span: the floor of a tie-break-only repair, and closing
them needs the scorer to stop pairing on quote alone (point 1).

THREE NUMBERS A STAGE-2 COMPARISON MUST CLEAR, all re-derivable from the artifact's own `controls` block:
1. `eval_harness._align_one` keys on `text_quote` overlap ALONE and pairs same-quote items by INDEX, and
   282 of 420 gold items (67%) share a quote with a sibling. Reversing every same-quote group moves
   `value_accuracy` 1.000 -> 0.367 and `period_accuracy` 1.000 -> 0.781 while `aggregate_f1` stays at
   1.000. A stage-2 comparison goes on the headline, never on `value_accuracy`. Still open in the
   scorer, deliberately: `_align_one` grades every published scorecard, so changing it re-prices history.
2. An absolute `aggregate_f1` on this gold is NOT comparable to a published one on the v6.2 substrate:
   the two golds align at item f1 0.654 (P 0.764 / R 0.571) on the same 40 articles, and every
   baseline quoted in `aggregate_f1` sits on the legacy side of that gap.
3. Gold construction owns 0.0690 of recall: the primary arm scores recall 0.788 against the gold it
   helped build, the cross-check 0.857. Common-mode across two models on the same gold, but it is the
   size of this repo's cross-family effects, so a candidate resembling one arm inherits part of it.

A SCORECARD ON THIS GOLD STAMPS `production_prompt_path: false`, and that is a runner limitation, not a
prompt one: `run_model.py` `render_prompt` substitutes `{{article_text}}`, `{{source_id}}` and
`{{published_at}}` only, while production's CoVe prompt uses `{{source}}`, `{{content_type}}` and
`{{content}}`, so a `--prompt-file` run would send literal braces in the header line. The substrate's
per-record `instruction` carries the correctly rendered prompt instead (byte-identical to what
`GetInitialExtractionPrompt` builds); until the runner renders those placeholders, compare the recorded
`stage.prompt.sha256` against the mount by hand. And `eval_harness._is_schema_valid` treats any key
outside its 11 as invalid, while `ExtractionSchema.cs` carries `period_end` and `release_date` -- a
model handed production's real schema scores `json_valid` 0 on a request-shape fact, not a model defect.

UNSCORED. No model has been scored on this gold, so its ability to SEPARATE two models is inferred from
inter-arm agreement, not demonstrated. And the path it measures is near-dormant: V1
`ChainOfVerification` (sources NOT in `Extraction__V2EnabledSources`) produced **6 observations in 14
days** against rss's 69,827, and `QualitativeVerificationService` is gated on `validation-content`,
which has produced **no row since 2026-04-17**. The gold is still the right instrument -- it is what
every `aggregate_f1` baseline means -- but a swap decision weighted by production impact belongs on
stage 1.
Re-check (the `--chat-template` is a coordinate of the served model, SentinelCollector D-29, not a
constant of this command):
```
python3 LlmBenchmark/scripts/run_model.py --task cove --endpoint-mode completions \
  --chat-template $'<|im_start|>user\n{0}<|im_end|>\n<|im_start|>assistant\n' \
  --max-tokens 16384 --substrate LlmBenchmark/cove-gold/cove_stage2_substrate_v1.json \
  --endpoint http://localhost:8000 --model <candidate> --out /tmp/cove-preds.jsonl
python3 LlmBenchmark/scripts/eval_harness.py --task cove \
  --substrate LlmBenchmark/cove-gold/cove_stage2_substrate_v1.json \
  --predictions /tmp/cove-preds.jsonl --adapter-meta /tmp/cove-preds.jsonl.provenance.json \
  --out /tmp/cove-scorecard.json
python3 LlmBenchmark/scripts/verify_cove_gold.py \
  --gold LlmBenchmark/cove-gold/cove_stage2_gold_v1.json \
  --corpus LlmBenchmark/cod-gold/cod_stage1_corpus_v1.json --selftest
```
`--selftest` is the re-check that matters for the ARTIFACT: 15 controls, 12 mutations caught by name
and 3 negative controls that must stay silent -- **15/15** at the 2026-09-16 re-check. It also FAILS if
`initial_extraction.txt` or `ExtractionSchema.cs` moves under the gold: the labels answer a specific
prompt, and a derived schema that outlives its source still parses.

### The CoD gold cannot yet back a model swap: macro-owner DECIDED, the key's swing is not [2026-09-05]
Production's CoD path is scoreable END TO END (chat template #1002, scorer #1011, runner #1013, gold
#1012, outage-vs-null gate #1014). What still stops a scorecard from DECIDING a swap is the instrument:
`LlmBenchmark/eval-substrate/cod-stage1.criteria.json` sets `numbers_f1.min_initial` **0.4** with
`basis: carried` (never measured on this task) and `ratified_by: null`, and at seed 42 / temperature 0
the pass count flips run to run WITHIN one arm -- the then-incumbent's two concurrency-1 runs, identical
in everything this repo records, scored 0.3865 and 0.4422. Any bar inside the run-to-run band flips
without the model changing, and moving the bar only moves the band. The swing has to come out of the
instrument before a THRESHOLD verdict means anything. It does not have to for a COMPARISON: n>=3 an
arm with disjoint runs separates two models (that is how every later A/B was read), so what this entry
forbids is reading a `pass` as a verdict, not comparing two candidates.

THE GOLD. `LlmBenchmark/cod-gold/` holds 1,744 gold facts in CoD's own shape over 40 chosen articles,
every object validating against `cod_json_schema_v1.json`, every fact naming its labeller
(`build_cod_gold.py`; $3.40 over 96 requests). What it covers, and does not:
- numbers (518) and entities (624) are usable gold, 1,142 of 1,744 objects; events are usable on
  `subject`+`trigger` only; `claims` measure labeller vocabulary. Inter-labeller agreement by array:
  numbers 0.886, entities 0.737, events 0.302, claims 0.138; rescored on `subject` alone events recover
  to 0.708 and claims only to 0.356. A scorer weighting the four arrays equally reports vocabulary as
  model quality.
- `guidance` and `regulatory` have ZERO gold members out of 40; a per-class figure there is an empty
  cell, not a score. Three `numbers[].value` fields hold RANGES ("3-4", "4-5", "50-100"). The recall
  denominator is 89 hand-counted facts across 5 of the 40 articles.
- 144 `entities.ticker` blanks are an ENCODING DEVIATION -- the prompt says OMIT -- so a field-level
  ticker comparison marks a correctly-omitting model wrong on all 144. A scorer must treat `""` as a
  value that aligns with `""`, never as an absent identity, or it zeroes the 22 blank anchors below.
- the alignment key is `(context, source_entity)` with independent floors: never `source_text` (a 1-3
  token literal, so two unrelated `"$15"` align perfectly) and never `value` (anything in the key reads
  1.0 by construction, and value accuracy is the number a swap turns on). Align events and claims on
  the wide key (`trigger` / `object` added), which takes every array to 0 collisions.

MACRO OWNER: DECIDED 2026-09-05 BY THE USER. The SERIES owns its own print, the country does not, and
`""` is not the answer; #1017 rewrote the prompt's worked example and 20 anchors were repaired. The 518
anchors partition **490 conform + 6 knowingly non-conformant + 22 blank**. The 6: article 183's payroll
rows stay on `US` because that article never says "payroll" or "nonfarm", so `""` was the conformant
answer -- disclosed and kept deliberately. The 22: 6 blank BY DESIGN (a figure spread across entities,
a bare count) and 16 the article never NAMES -- article 1's eleven Conference Board survey shares and
article 48's five Challenger job-cut figures, the latter now keyed from provenance by D-32 (#1044) in production;
the gold's five stay blank.
Closing the rest needs a rule for a series a story describes without naming, not a re-decision.
149 of 518 anchors (28.8%) name a `macro_indicator`. WHAT THE DECISION DID NOT BUY: `NonInstrumentEntTypes`
(`SentinelCollector/src/Extraction/DslToMergedExtractionAdapter.cs`) excludes `macro_indicator`
alongside `country`, `industry`, `sector` and `concept`, so **194 of 518** gold anchors pre-select
nothing at Rule 1 under EITHER convention and fall through to Rule 2, which runs no surface filter. It
bought a RESOLVABLE surface: in `atlas_secmaster` (SELECT, 2026-09-05) `United States` exact-matches
ONE instrument, the CO2 series `EMISSCO2TOTVTTTOUSA`, while `Unemployment Rate` -> `UNRATE` and
`All Employees, Total Nonfarm` -> `PAYEMS`. The country convention names a carbon series.

THE SCHEMA'S STRING CAPS SHAPED THE GOLD. `cod_json_schema_v1.json` caps `numbers.context` at 80 and
`claims.object` at 120, and `build_cod_gold.py` DISCARDS a cross-check correction whose string exceeds a
cap (`not applied: would break schema`) -- length silently beating correctness. On the first build 19
of 355 claims sat at exactly 120, every one severed mid-phrase, one with corrupt text. Instances
repaired: 0 at 120 now, max 119; a cap-shaped spike in the length histogram is truncation. OPEN and not
fixable in the gold: the caps are production's, so whether 120 costs real claim content at EXTRACTION
time is unmeasured, and whoever raises them needs that number first.

RUNNER FOOTGUN: `run_model.py --substrate` reads a JSON list and `--limit` takes a PREFIX, so
`--limit 40` runs the wrong 40 and exits 0. Build the 40-article subset from the committed corpus and
write it OUTSIDE the tree:
```
python3 -c 'import json; c=json.load(open("LlmBenchmark/cod-gold/cod_stage1_corpus_v1.json"));
json.dump([{"input":{"content":a["content"]},"source_file":a["source_file"],
"source_index":a["source_index"]} for a in c["articles"]], open("/tmp/g40.json","w"))'
```
And read `truncated` and `finish_reasons` in the provenance before concluding anything from
`schema_invalid`: a run budgeted too low lands truncated responses there and reproduces the
`schema_invalid == record count` symptom this entry once led with, which is CLOSED. The cap production
sends is 8192 since D-30.

Re-checks (no engine; each pinned):
```
python3 LlmBenchmark/scripts/verify_cod_gold.py --gold LlmBenchmark/cod-gold/cod_stage1_gold_v1.json \
  --corpus LlmBenchmark/cod-gold/cod_stage1_corpus_v1.json --selftest   # 10/10 controls behaved as required
python3 LlmBenchmark/scripts/rescore_alignment_keys.py --selftest         # selftest: 14/14 controls behaved as required
grep ratified_by LlmBenchmark/eval-substrate/cod-stage1.criteria.json     # null until a human ratifies
python3 -c "
import json
d=json.load(open('LlmBenchmark/cod-gold/cod_stage1_gold_v1.json'))
n=[(a['id'],(x.get('source_entity') or '').strip()) for a in d['articles'] for x in a['gold']['numbers']]
b=sum(1 for _,s in n if not s); k=sum(1 for i,s in n if i.endswith(':183') and s=='US')
print(len(n)-b-k, k, b, len(n))"                                           # 490 6 22 518
python3 -c "import json;
d=json.load(open('LlmBenchmark/cod-gold/cod_stage1_gold_v1.json'));print(sum(1 for a in
d['articles'] for n in a['gold']['numbers'] if {e['name']:e['ent_type'] for e in
a['gold']['entities']}.get(n['source_entity'])=='macro_indicator'))"      # 149; a 129 means the 20 repairs were reverted
python3 - <<'PY'
import collections, json
NON = {"metric","macro_indicator","exchange","currency","person","country","region",
       "location","sector","industry","object","event","concept","analyst_firm"}
d = json.load(open('LlmBenchmark/cod-gold/cod_stage1_gold_v1.json'))
c = collections.Counter()
for a in d['articles']:
    t = {e['name']: e['ent_type'] for e in a['gold']['entities']}
    for n in a['gold']['numbers']:
        c[t.get(n['source_entity'], '<blank>' if n['source_entity'] == '' else '<undeclared>')] += 1
print(sum(v for k, v in c.items() if k in NON), sum(c.values()), c.most_common())
PY
```
The census prints `194 518` with `equity 187, macro_indicator 149, instrument 68, org 26, <blank> 22,
index 21, industry 15, country 13, concept 10, person 3, event 2, location 1, region 1` and ZERO
`<undeclared>`; a non-zero `<undeclared>` means a gold anchor stopped naming an entity its article
declares. The partition's first three MUST sum to the fourth, and any conformance figure quoted MUST
be one of the four (an earlier revision's `496` was none of them). `rescore_alignment_keys.py
--selftest` pins fourteen mutations by name, three at n>1 because the shuffled-gold verdict as first
written averaged the floor ACROSS runs (a 0.4570 breach diluted to 0.0457 and printed `FLOOR_OK`); a
review mutated the tool eighteen ways and four controls close five of the six escapes -- the SIXTH is
still open, so a re-mutation sweep should expect one escape these controls do not see. All three
selftests held at the 2026-09-16 re-check (10/10, 14/14, `490 6 22 518`).

### The replicate sd is a within-serve-session statistic; the between-session term is unpriced [2026-09-07]
Measured 2026-09-07, opened by #1037. Gemma 4 was re-run at the coordinate of the 0.7570 row -- same
engine image, revision `52f3f65b`, serve flags, template `7a8a39d2`, gold, substrate, scorer and
`usage.prompt_tokens` 86,633, with both boots logging an IDENTICAL vLLM engine-config line and the
same 68,892-token KV -- and scored **0.7346**, -0.0224 or 3.5x that arm's replicate sd. NO AXIS
DIFFERS; the sd is what misleads. Each triple runs back-to-back against ONE boot, sharing a warm
prefix cache, so replicates agree on 9-13 / 40 records (pinned) and 13-18 / 40 (the re-run) while
ACROSS the two sessions the same pairs agree on only **4-11 / 40**. The 0.0009 was three unstable
output sets scoring alike, not determinism. The then-incumbent's (Qwen2.5) own cross-session pair at
that coordinate moved +0.0060 (0.4545 -> 0.4605, 0.29 sd), so the term is there for both models and
merely hides inside a larger within-session sd.
Pooled Gemma 4 at that coordinate, n=6: **mean 0.7458, sd 0.0130, range 0.7280-0.7577**.
Nothing already published flips on it: -0.0224 is a third of that arm's article-level CI half-width
and an order below its +0.2214 over production. `LlmBenchmark/BENCHMARKS.md` footnotes the row ("one
serve session's replicate sd, not a reproducibility figure") and D-29 quotes replicate sd 0.0065 with a
cluster-bootstrap CI; no scorecard set under `LlmBenchmark/eval-substrate/` has >=3 boots per arm.
RE-CHECKABLE: two sessions per model shows the term EXISTS and cannot size it. Sizing it needs >=3
boots per arm with the triples kept separate, which no sweep here has run. Until then quote a
replicate sd as decoding jitter within one session and NEVER as reproducibility. The raw artifacts
were untracked under `/tmp/sentinel-remediation/` and are gone.

### llama.cpp /v1/completions silently IGNORES response_format: check content, never status [2026-09-06]
A verification failed toward success inside the task about that. "llama.cpp honours `json_schema` on
`/v1/completions` -- verified" was FALSE: the check read HTTP status and `finish_reason`, never the
content. `/v1/completions` ignores `response_format` and emits the JSON inside a triple-backtick json
fence -- the first Q6 arm returned 40/40 with 0 call errors, `finish_reason` stop, and `schema_invalid`
40. Measured across all three endpoints: `/v1/completions` ignores it, `/v1/chat/completions` honours
it, `/completion` with a top-level `json_schema` honours it. A llama.cpp arm is therefore either off
production's prompt path (chat mode, `production_prompt_path: false` -- what the precision-ladder arms
did) or unconstrained. Separately, reading a provenance file WHILE the runner was writing it reported
`call_errors 40, wall 1.9s` for a run whose own log said `records 40, errors 0, wall 291.8s` -- a
spurious engine failure, caught only by cross-checking an independently written artifact.
THINKING DEFAULTS ARE AN UNCONTROLLED AXIS ACROSS VENDORS: GLM's vendor chat template defaults thinking
ON (it ends on an open `<think>`), so the 4,096-token budget goes to reasoning and the JSON never
closes; Gemma 4 and EXAONE default OFF. Derive one thinking-OFF template per arm so the axis is held
constant across arms.
This gotcha has no durable home outside this entry -- `grep -rn 'ignores.*response_format'
LlmBenchmark --include=*.md --include=*.py` finds nothing -- and it will bite the next llama.cpp arm.

### production_prompt_path is blind on prompt content and on sampling; the card cites it bare [2026-09-06]
`eval_harness` stamps `production_prompt_path: true` on a four-way conjunction -- `endpoint_mode ==
"completions"`, a `prompt_file` NAMED, a `schema_file` named, and a chat template that wraps the prompt
-- and NOT ONE conjunct inspects prompt CONTENT or reads a decoding knob. Measured 2026-09-06: all
TWELVE scorecards of a two-prompt x two-sampling grid stamped `true`, including the six that ran
`repetition_penalty: null, max_tokens: 8192` where the service sent 1.1 / 4096 and the six on the
prompt production was not running. The flag is sound on WIRE SHAPE, which is what it was built for,
and certifies the arm that is false of production exactly as loudly as the one that is true of it.
Since `d139f954` (#1026) `request_shape` carries `prompt_file_sha256` / `schema_file_sha256` /
`chat_template_sha256` and a per-knob `request_sampling` block sits beside it, under the invariant
that a digest is of bytes that REACHED a request or is null. That RECORDS; it does not close: the
conjuncts are unchanged and the comment above them says so. The scorer records and a human
ADJUDICATES, comparing the digests against the prompts mount and `request_sampling` against
`CpuCodOptions`.
FOLLOW-UP STILL OPEN: `SentinelCollector/AGENT_README.md` MODEL_ACCEPTANCE cites a
`production_prompt_path: true` scorecard as evidence the bar runs end to end, with no clause saying
the flag does not discriminate content or sampling; an agent reading only that will believe it does.
Either the flag consumes the digests, or the card gains the clause.
Re-check: `grep -n production_prompt_path SentinelCollector/AGENT_README.md` -- open while the hit
carries no wire-shape-only qualifier.

### The anti-invention clause FAILS, referential integrity is blind to it, the guard hides it [2026-09-06]
THE CLAIM. `SentinelCollector/src/cod-prompts/cod_json_v1.txt` says of `source_entity` that a series the
article never NAMES is "a blank, not licence to coin one" and closes with "NEVER invent a name to fill
this field" (cited by verbatim text: `scripts/verify-citations.py` `_EXTS` has no `txt`, so a
`.txt:<line>` citation can never be swept). Measured 2026-09-06 on the then-incumbent (Qwen2.5) at
HARNESS sampling (`repetition_penalty: null, max_tokens: 8192`), the model coins names anyway. On
`sentinel-v6.2-cove.json` index 35 it emitted **60 entities in every run where the gold holds 15**, of
which 50 / 50 / 49 were `macro_indicator` nominalisations of one sentence (`U.S. holiday sales growth
rate`, `... pace`, `... momentum`, ...) and NOT ONE occurs in the article; on index 1 it coined six
`concept` anchors from clauses the article only describes, in 1 of 3 runs. Indices 48 and 183 held
clean in all three runs. Hand-checked on four of the eight articles the criterion selects: a sample,
not a sweep.
WHY IT IS DEBT, NOT A BUG REPORT. `source_entity_referential_integrity` asks whether an anchor appears
in the model's OWN `entities[]` -- never whether it appears in the ARTICLE. Every coinage was duly
declared in `entities[]`, so the metric read **0.9603 / 0.9760 / 0.9900** across the three runs and
the run carrying hand-verified coined anchors on BOTH articles scored the MIDDLE value: it cannot even
RANK runs by coinage. A metric that cannot fail on the failure mode its clause exists to prevent is a
harness that fails toward success. The article text is ALREADY BOUND in the same scorer loop
(`content = _norm_ws(p.source_content)`, read by `number_source_text_verbatim_rate` a few lines on),
so the missing check needs no new data. STILL ABSENT on main:
`grep source_entity_article_grounding LlmBenchmark/scripts/eval_harness.py` finds nothing.
THE NORMALISER IS THE PRICE. A plain substring test marks correct answers wrong wherever the source
text is mangled: index 429's analyst table holds `Commerzb\nank`, `Bank of\nAmerica`,
`Standard\nChartere\nd`, and the model REASSEMBLED them correctly. Any coinage rate needs a normaliser
that survives an intra-word newline before it means anything; the corpus-wide rates produced during
this work are not recorded for that reason.
THE CONSEQUENCE, AND THE COUPLING. At production's sampling (1.1 / 4096 then; 1.1 / 8192 since D-30)
the runaway does NOT manifest: index 35 emits 6 / 4 / 8 entities with ZERO coined names, and no
article-run saturates a schema array (0 / 120, against 6 / 120 unguarded on the same prompt). But the
two-sided wire control on article 183 shows `repetition_penalty 1.0` at the same cap STILL saturating
`events[]` at `maxItems` 60 with `finish_reason: stop` (4076 completion tokens), and 2.0 collapses the
output (`finish_reason: length`, `schema_invalid 1`) -- so `max_tokens` alone does not close the loop,
the penalty is the knob, and `CpuCodOptions.JsonRepetitionPenalty` is LOAD-BEARING FOR PROMPT
CORRECTNESS, not only for latency. If the guard is loosened, this prompt change bites. Whether the
`macro_indicator` clause CAUSED the index-35 runaway or merely uncovered it is unmeasured (one prompt
edit and one re-run settles it; the guard hides it). THE PROMPT IS WHAT MAKES ARTICLE 35 LOOP: both arms of the
original comparison ran with the penalty unset, so a CONSTANT cannot explain a difference that appears in ONE arm;
the corrected prompt is the only axis that moved, and it remains the cause -- this entry does not exonerate it.
Control: the OLD prompt emits 9, 10 and 11 entities on the same article, zero `macro_indicator` and zero coined.
Every figure here is the retired incumbent's: NONE has been measured on Gemma 4.
CLOSING IT is a scorer change (a gold-free `source_entity_article_grounding` beside the integrity
metric -- same loop, same `p.source_content`, the normaliser above) or a prompt clause the model
actually obeys. Quoting integrity cannot close it: 0.9603-0.9900 is the number this failure mode produces.
Re-check (no engine; the 2026-09-06 predictions under `/tmp` are gone, so it needs a fresh predictions
file at harness sampling and the 40-article subset from the CoD gold entry's one-liner):
```
python3 - <<'PY'
import json, re
sub = {r['source_index']: r for r in json.load(open('<the 40-article subset>.json'))}
art = re.sub(r'[^a-z0-9]+', ' ', sub[35]['input']['content'].lower()).strip()
for line in open('<a predictions file>.jsonl'):
    r = json.loads(line)
    if r['source_index'] != 35:
        continue
    e = r['prediction']['entities']
    coined = [x['name'] for x in e if re.sub(r'[^a-z0-9]+', ' ', x['name'].lower()).strip() not in art]
    print(len(e), 'entities,', len(coined), 'not in the article:', coined[:6])
PY
```
2026-09-06 at harness sampling -> `60 entities, 50 not in the article` (49 on the third run); at
production's sampling -> `6 entities, 0 not in the article` (4 and 8 on the other two runs). The second
line is the one that decides whether the defect reaches production. Re-deriving the full four-cell
control costs 12 runs (two prompts x two samplings x three, ~2 minutes a run at concurrency 6) plus
`git show 45d02cd8bd3e360529267c9fec3868fd8554af4a` for the pre-#1017 prompt.

### Nothing checks whether the SHIPPED gold still states the `source_entity` convention [2026-09-05]
`build_cod_gold.py`'s divergence gate is a PRODUCER gate: it compares production's prompt against
the adjudication instruction it is about to send and against the text `alignability()` would write
into the next artifact. It never opens `LlmBenchmark/cod-gold/cod_stage1_gold_v1.json`. The README
and the docstring both claimed it did -- corrected in the same PR rather than the gate widened,
because opening the committed artifact would DEADLOCK the tool: after a deliberate rule change the
shipped artifact necessarily still states the old rule, so the gate that must pass for
`--stage assemble` to rewrite it would be the one refusing. That leaves a real hole. The artifact's
`controls.control_5_alignability.blank_by_design.why` is what a reader ACTS ON when deciding how to
treat a blank anchor, and it is prose a build may hand-tune, so it can drift from the prompt with
nothing complaining. It has: it paraphrases all three pinned clauses and states none verbatim.
```
python3 -c "
import json,sys; sys.path.insert(0,'LlmBenchmark/scripts')
import build_cod_gold as b
w=json.load(open('LlmBenchmark/cod-gold/cod_stage1_gold_v1.json'))['controls']['control_5_alignability']['blank_by_design']['why']
print(sum(1 for _,c in b.SOURCE_ENTITY_CLAUSES if b._rule_text(c) in b._rule_text(w)), 'of', len(b.SOURCE_ENTITY_CLAUSES))"
```
2026-09-05 -> `0 of 3`. Closing this belongs in `verify_cod_gold.py`, which grades a FINISHED
artifact and so cannot deadlock a build -- either it requires the three clauses verbatim in that
field, or the field is generated and the hand-tuned prose moves to a sibling key. `3 of 3`, or a
recorded decision that a paraphrase is enough and what checks the paraphrase, closes it.

### A well-formed 2xx envelope whose content is not an answer still passes silently [2026-09-05]
`choice_content` now refuses an envelope with no usable choice, so a provider outage reaches
`call_errors` and the fail-closed gate. Three shapes still do not, each landing in `schema_invalid`
at rc 0: the provider's error text delivered AS the completion; `finish_reason: content_filter`
(counted in `finish_reasons`, gated on by nothing); and a `stop` carrying
`completion_tokens: 0`. Class: an outage indistinguishable from a null result once it wears a
valid envelope. The last two close by gating on those fields; the first cannot without
pattern-matching content, which is the salvage-parse this harness refuses.
```
python3 - <<'PY'
import sys, argparse; sys.path.insert(0, 'LlmBenchmark/scripts'); import run_model as rm
A = argparse.Namespace(model='m', temperature=0.0, seed=42, top_p=None, top_k=None, min_p=None,
    presence_penalty=None, repetition_penalty=None, max_tokens=64, chat_template_kwargs=None,
    no_structured_output=False, endpoint='http://x', timeout=5.0, endpoint_mode='chat',
    task='cod', prompt_file=None, schema_file=None, chat_template=None, stop=None)
R = {'source_file': 'f', 'source_index': 0, 'instruction': 'S', 'input': {'content': 'C'}}
for name, content, fr in (('error_as_text', 'Error: rate limited (429)', 'stop'),
                          ('content_filter', '', 'content_filter'),
                          ('zero_tokens', '', 'stop')):
    rm._http_json = lambda *a, _c=content, _f=fr, **k: {
        'choices': [{'message': {'content': _c}, 'finish_reason': _f}],
        'usage': {'completion_tokens': 0}}
    print(name, 'recorded as a call error:', 'error' in rm.run_one(R, A))
PY
-> all three print False while this entry is true
```

### `check_staleness.py` never looks inside a subdirectory, so a nested scorecard is unopened, not stale [2026-09-04]
The grading half of "a file skipped in silence takes the denominator with it" is closed twice over:
an unparseable file and a shapeless one both reach the tally as `UNEXAMINED`, and after #1002 that
holds whatever the file is CALLED (the previous version rescued only `*.scorecard.json`, so
`--out scorecard.json` -- the name `LlmBenchmark/scripts/README.md` itself writes -- fell through to
a silent skip). The ASSEMBLY half is still open: `_iter_scorecards` uses a non-recursive
`p.glob("*.json")`, so a card one directory down is not skipped, not counted and not named.

MEASURED 2026-09-04, stdlib stub answering `/version` with `{"version": "0.19.0"}`, two real
scorecards on disk:
```
<dir>/good.scorecard.json          engine vllm 0.19.0, generated today
<dir>/sub/moved.scorecard.json     engine vllm 0.28.0  -> would verdict ENGINE_MOVED
python3 LlmBenchmark/scripts/check_staleness.py --scorecards <dir> --endpoint <stub>
  -> "1/1 current; 0 stale"   exit 0   stderr EMPTY
```
That is character-for-character the output the `UNEXAMINED` verdict exists to prevent, one level
down, and neither the module docstring's control nor `_corpus_control` can see it: both are handed
the corpus this function assembles, so a file it never looks for is invisible to every check below.

NOT PATCHED IN #1002 deliberately. `rglob` is a one-word change and a semantic one -- it decides
which files this tool claims to have an opinion about, and `--scorecards` takes any path an operator
names, including trees holding `obj/`, sibling checkouts and archived runs. The committed corpus
(`LlmBenchmark/eval-substrate`) is flat, so the change is a no-op there and cannot be validated by
running it. CLOSE THIS by deciding the corpus rule first -- recurse, or refuse a directory that
CONTAINS subdirectories holding `*.json` and name them -- then add the matching row to
`_CORPUS_CONTROL`, which is where a partition claim is enforced on every run.
Re-check: reproduce the two-file layout above; if this entry is still true it still prints
`1/1 current; 0 stale` at exit 0.

### Golden corpus: its controls mutate the FIXTURE, none mutates the publisher SUT [2026-08-27]
**THE GOLDEN CORPUS PROVES ITS ASSERTIONS READ THE FIXTURE; NOTHING PROVES THEY WOULD CATCH A
CHANGE TO THE PUBLISHER.** Those are different guarantees and the corpus only measures one of them.
Measured 2026-08-26 on `GoldenCorpusIdentityTests` (68 cases under `DisplayName~GoldenCorpus`; 70 from
2026-08-27, the claim-atom census and its control): the
two committed mutation controls break a FIXTURE -- give every row of `collision-151480-walmart` its
own symbol, force every row of `separating-150183-slug-fallback` onto one -- and require the
matching verdict to complain by name. That is fixture mutation. No control mutates
`EventPublisher.CreateSeriesCollectedEvent` or `SentinelSeriesKey`, so the claim "this corpus would
catch an identity regression" rests on reading the assertions, not on having seen one caught. The
original 5-of-96 kill figure was a one-off manual run of the same fixture-only kind and was never
reproducible; the two controls replace it with a number the suite re-derives, on the same axis.
Fix: mutate the SUT once and record what dies -- and if a plausible mutation survives, the corpus
has a hole the fixture axis cannot see.
Re-check (run it, it is two minutes, and revert the edit afterwards):
  in `SentinelCollector/src/Publishers/EventPublisher.cs`, replace the identity derivation in
  `CreateSeriesCollectedEvent` with a constant (`var identity = "MUTANT";`), then
  `nerdctl compose exec -T sentinel-collector-dev sh -c "cd /workspace/SentinelCollector/tests/SentinelCollector.UnitTests && dotnet test --filter 'DisplayName~GoldenCorpus'"`
  # expected: every separating case and every baseline case REDs; record the survivors, they are
  # the assertions that do not depend on the publisher at all

### A test class missing `[Collection(StalenessGaugeCollection.Name)]` silently re-opens a data race [2026-08-17]
**A test class that forgets `[Collection(StalenessGaugeCollection.Name)]` silently re-opens a data race, and the
suite stays green.** `FinnhubCollector/tests/Workers/StalenessGaugeProbe.cs` declares the collection that serialises
every class touching FinnhubMeter's process-global staleness origins; membership is enforced by PROSE in that
docstring and by nothing else. Measured 2026-08-17 on the pre-existing pair: as shipped the two classes are strictly
disjoint (6.3s wall, serialised); split into two collections they run concurrently (3.0s) and 2 of 4 unsynchronised
appends were lost — and the split control still ran GREEN 4/4, which is the whole problem. The blast radius grew with
this PR: the origins are now a dictionary that `TrackActiveSymbols` PRUNES, so an unserialised sibling can delete the
symbol another test is reading, not merely move a timestamp. NOT fixed here. Re-check: `grep -L
"StalenessGaugeCollection.Name" FinnhubCollector/tests/Workers/*Tests.cs` must return nothing.

### 2,138 assertions that no suite-level count can see [2026-08-15]
**2,138 assertions that no suite-level count can see.** `run-push-config-destination-smoke.sh` prints no
per-assertion `PASS:` line — its `pass()` (`:44`) increments a counter silently while `fail()` (`:45`) does print
`FAIL: <label>` — so its only positive output is the summary at `:656`. Measured 2026-08-15:
`bash .claude/hooks/test/run-push-config-destination-smoke.sh` -> rc 0, 22 lines of output, **0** of them matching
`PASS: `, ending `cells: 1020 + 14 targeted + 33 config-injection` / `PASS=2138 FAIL=0`. It is the only one of the
ten suites in `.claude/hooks/test/` that emits none — `grep -L 'PASS: ' .claude/hooks/test/run-*.sh` names it and
nothing else. A sweep that counts `PASS:` lines therefore scores this suite 0 and reads its coverage as ABSENT
rather than passing; the failure direction is visible, the coverage direction is not. Either print per-assertion
lines or teach the sweep to read `PASS=<n>` — until then, count 2,138 here by hand.

### Alertmanager's `warning` route `repeat_interval` does not pace a Grafana-managed alert [2026-08-19]
Reading it as the pacing overstates notification volume. Alertmanager is NOT a Prometheus scrape target
(`deployment/artifacts/monitoring/prometheus.yml` lists it only under `alertmanagers:`, no `job_name`;
re-verified 2026-09-16), so `alertmanager_notifications_total` is reachable only by curling `:9093/metrics`
directly and is invisible to PromQL and every dashboard. One-off window, 2026-08-19: 687 webhook
notifications across ALL alerts in 18.86 days of uptime (about 36/day for the whole stack, not per alert)
against 72 Loki `severity_text="Warning"` lines in 24h. Any "N warnings per day" figure must name which
population it counts -- webhook notifications, ntfy messages, or Loki lines -- they differ by more than 10x.
Re-check: `grep -n job_name deployment/artifacts/monitoring/prometheus.yml` still names no alertmanager job.

### 3 memory citations cannot land, all of one irreparable class [2026-08-24]
The memory corpus lives outside the repo, so `git ls-files` cannot name a memory file; `verify-citations.py
--memory` assembles it from `$DREAM_MEMORY_ROOT`. Reproduce: `python3 scripts/verify-citations.py --quiet
--memory` -> 35 checked / 3 cannot land (2026-08-24); 111 files / 36 citations / 3 cannot land (2026-09-16).
The figure moves whenever a citation is added to or retired from the corpus; the command is the durable half.
THE 3 ARE A PERMANENT EXEMPTION AND MUST STAY BROKEN: each sits inside a VERBATIM dream `ev:` provenance
quote (session id, turn, timestamp), none ever resolved, and re-pointing one would fabricate a quote. This is
a class, not three incidents -- any provenance quote may contain a citation-shaped string -- so expect the
floor to sit above zero permanently and judge a change by whether its cannot-land SET is a subset of its
base's. Deliberately NOT skipped in the tool: 0 of 411 tracked-repo citations sat inside an HTML comment and
6 of 35 memory ones did, all in provenance quotes, so a skip would stop covering the one place provenance lives.
Hazard documented in the module docstring and NOT guarded: a memory file's own directory is searched first
for a bare basename, so a bare `.md` cite colliding with a sibling memory filename binds to the memory copy
silently (0 occurrences measured). THIS ENTRY CARRIES NO `file` plus line-number FORMS -- the tool parses
its own backlog.

### ThresholdEngine builds a channel per event type that has neither a reader NOR a writer [2026-08-17]
`ThresholdEngine/src/Events/ChannelEventBus.cs`: `GetOrCreateChannel<TEvent>` creates a
`Channel.CreateUnbounded<TEvent>` per event type and `PublishAsync` dispatches handlers directly through
`Task.Run`, never touching it; only `Dispose` completes the writer. Measured 2026-08-17 and again 2026-09-16:
`grep -n "\.Writer\|\.Reader\|ReadAllAsync\|WriteAsync" ThresholdEngine/src/Events/ChannelEventBus.cs`
returns exactly ONE hit, `Channel.Writer.Complete()` in `Dispose`. Inert (no writer, so it cannot grow; no
bound, so it cannot block) -- the class NAME is the trap: an event bus called ChannelEventBus that dispatches
without its channel is what the next agent reasons about wrongly, and PR #975's channel table omits it, which
reads as "no third channel" rather than "the third is inert".
Re-check: the grep must still return only the `Dispose` hit; more means the class has grown a real channel path.

### Nine real-but-wrong citations stand in `SentinelCollector/AGENT_README.md`, all GREEN [2026-09-05]
**Nine real-but-wrong citations stand in `SentinelCollector/AGENT_README.md`, all GREEN, none of them any PR's
debt.** The live instance of the drift class above, found by hand in review of PR #1004 and deliberately NOT
repaired there: they sit in the `DeterministicResolver` / `ExtractionProcessor` D-entries as a -35 cluster from an
old un-followed insertion plus -74, -30 and -4, and one is a NAMING error rather than a number --
a GUARD reads `ResolveAsync @ src/Services/DeterministicResolver.cs:753` while `ResolveAsync` is the thin
wrapper at `:47` and the enclosing method at the cited leg is `ResolveCoreAsync` (`:82`). Present at base
`eb5aa3e3` with the identical offsets, and six of the nine were among the 21 AGENT_README citations that PR
"repaired" -- i.e. shifted while already 35 lines short, which is what makes a repair sweep no evidence at all.
LEFT ALONE ON PURPOSE: the tool reads GREEN on every one of them (they land on real lines), so a repair is a
content-level hand audit whose result nothing can check, landing in the one file that PR was already rewriting
heavily -- the trade CLAUDE.md TOOL_UPKEEP declines, cosmetic fixes bought with a new unverifiable claim.
Re-check: they are invisible to `verify-citations.py` by construction; the measurement that finds them is reading
each D-entry's GUARD prose against the symbol at the cited line. A symbol-anchored checker would decide this
class mechanically -- `Symbol.Method @ path:line` where the landing window must mention `Method` -- but only 3
of this card's citations use that exact form today, so the check would need the card's other citation forms
normalised first.
RE-MEASURED 2026-09-16: the class is live (D-6 GUARD `TryGeminiResolveAsync` repaired to `:753`, method now
at `:788`, +35 drift again); the count nine and the `ResolveAsync` naming are stale; a re-count is a hand audit.

### ~35 `file:line` citations in THIS FILE point at real-but-wrong content, and the sweep reads GREEN [2026-09-05]
The live instance of the drift class above, inside `docs/BACKLOG.md`. Measured 2026-09-05 by a full-file
audit opening each citation: 125 resolve and every one is IN BOUNDS, while roughly 35 land on a comment, an
XML doc, a blank line or a different symbol; the largest cluster is a uniform +35-line drift across the
`DeterministicResolver.cs` set. Left on purpose: a repo-wide repair sweep is no evidence of anything (six of
the nine AGENT_README citations above were "repaired" while already 35 lines short). The 2026-09-16 triage
found fresh drift in three more entries (compose.yaml 1127 -> 1235, compare_base_vs_resolved.py 771 -> 781,
selftest.sh 479 -> 519), repaired in place -- written in non-citation form on purpose.
Re-check, and note what it CANNOT do: `python3 scripts/verify-citations.py docs/BACKLOG.md` -> 135 checked /
4 cannot land (2026-09-05, all four ambiguous-basename unresolveds, not one a drifted line); 174 checked /
7 cannot land (2026-09-16 on the pre-triage file `40d372eb`, composition not re-taken); 131 checked / 1 cannot land on
this file after the triage (b08ccdef; the one is the pre-existing ambiguous-basename cite of `SeriesManagementService.cs` at its line 54). The count would not move if every one of the 35 were
repaired, nor if thirty more drifted: opening each citation and reading the line is the only finder, so
treat a fall in `cannot land` as no signal at all.

**MODEL BASELINES, aggregate_f1 on the v6.2 substrate.** Moved out of CLAUDE.md MODEL_ACCEPTANCE 2026-09-06;
the SentinelCollector card's MODEL_ACCEPTANCE line points here for coordinates + controls + sample detail, and
scorecards are in `LlmBenchmark/eval-substrate/*.json`. NONE OF THIS IS ACCEPTANCE EVIDENCE: it is scored on the
substrate's own 16 instruction blocks, a task production does not run, for models no longer served. A swap needs a
CoD scorecard on production's prompt path AND sampling -- `production_prompt_path` reads no decoding knob and stamps
`true` on a run that omitted production's loop guard -- and its figures are NOT these (`numbers_f1` on production's
CoD path is a different task and metric from `aggregate_f1` here).

| model | aggregate_f1 | engine / config |
|---|---|---|
| Qwen2.5-32B-AWQ | 0.443 | vllm-0.19.0, fp8_e5m2 KV (production until the 2026-09-07 swap) |
| Qwen2.5-32B-AWQ | 0.494 | vllm-0.19.0, unquantized KV |
| Qwen3.8-27B-AWQ-INT4 | 0.764 | vllm-0.19.0 |
| Qwen3.8-27B-AWQ-INT4 | 0.7634 | vllm-0.28.0 [BLOCKED engine at the time; 0.28.0 + fp8_e4m3 is production since 2026-09-07, D-29] |
| Qwen3.8-27B-AWQ-INT4 | 0.7629 | vllm-0.28.0 + MTP speculative decode -- F1 unchanged, wall clock -30% |
| Qwen3.8-27B-AWQ-INT4 | 0.744 | NVFP4 @ vllm-0.28.0 |
| Qwen3.8-27B-AWQ-INT4 | 0.694 | vllm-0.19.0, the model card's own sampling |

FOUR digits on the 0.7634 and 0.7629 rows deliberately -- both round to 0.763, so a three-digit row does not say
WHICH scorecard it came from. Do not round them.
AND THE ENGINE IS ONLY HALF THE IDENTITY: SIX OF THESE SEVEN ROWS DO NOT NAME THEIR SAMPLING [2026-09-06].
The last row is the proof, sitting inside the table it indicts -- the ONLY axis separating 0.694 from 0.764
is the decoding, a **0.070** swing, larger than the 0.051 fp8 KV effect this file devotes an entry to and
larger than most gaps the table is used to argue. So "EVERY FIGURE NAMES ITS ENGINE" is necessary and not
sufficient: a row carrying an engine and no sampling still does not identify a run. Whoever next edits this table
should add a sampling column rather than trusting the engine column to carry it.
The 30B floor was retired because a 27B model beat the 32B incumbent here: the proxy would have BLOCKED an upgrade
on a number that was never the point.

### colibri cannot produce an admissible scorecard: seed refused, no constrained decoding [2026-09-06]
DO-NOT-BUILD, on two structural blockers, both measured on v1.10.2 (commit `fd93c41a`). Build and serve
were not the problem -- 1.96 s to a 180 KB zero-dependency binary, weights loaded in 6.5 s, all OpenAI
endpoints correct, `coli doctor` 11 ok / 1 warn / 1 skip:
1. `seed` is REFUSED -- `HTTP 400 "Per-request seeds are not supported yet."` `run_model.py` sends
   `seed` on every request because production does and has no off switch, so the unmodified harness
   reports `2/2 calls failed; that is an outage, not a run`. Reproduces with and without
   `--no-structured-output`, so the two blockers are independent.
2. NO CONSTRAINED DECODING AT ALL -- every `response_format` form on both wire shapes returns
   `400 "response_format grammars are not supported by the qwen36 engine yet."`
   `FamilyCapabilities.grammar_payload` is true for 1 of 8 families (the 372 GB GLM-5.2/5.3), and even
   there it is a SPECULATIVE DRAFT SOURCE, not a constraint: `pick_tok` ranges over the full vocab
   unmasked and the draft is kept only `if(next==draft[j])`. Scoring would require exactly the
   free-generate-and-salvage mode that produced the nine false eliminations in `BENCHMARKS.md`.
Latency was measured and is confounded, so it is not quoted as the engine's: 1.91 tok/s and one gold
article at 2,853 s with no JSON, but the engine was disk-streaming under a concurrent GPU sweep, and
production's template does not suppress Qwen3.6's thinking (D-26 leaves `ThinkingSuppressionSuffix`
empty). The region above 32B was priced for the first time -- Qwen3.8-Flash-Next 125B fits this box's
RAM at 62.5 GB int4 -- and every candidate there hits both blockers before size is the question.
Download nothing.
THE SCREEN THIS PRODUCED lives at `LlmBenchmark/MEASUREMENT_SPACE.md:163` ("The screen, and it is
cheap"), NOT in `CLAUDE.md` as an earlier revision of this entry claimed: an engine must accept `seed`
AND perform REAL constrained decoding -- a grammar that MASKS sampling, not one that only drafts
speculatively -- or no admissible scorecard can exist on it. SGLang and ktransformers pass it;
ExLlamaV2/V3 is the only GPU route to the Q6-as-floor question; MLC buys nothing on one fixed NVIDIA
card. The `/tmp/sentinel-remediation/colibri/` artifacts are gone.

### Precision ladder on gemma-3-27b: Q6_K LOSES to Q4_K_M, and vLLM has no rung above 4-bit [2026-09-06]
The only record of this measurement (`LlmBenchmark/BENCHMARKS.md` carries no Q6_K row). Single axis:
gemma-3-27b-it, unsloth GGUF, llama.cpp server-cuda b10820, only `general.file_type` differing (15 vs
18, read from the GGUF header, not the filename). `numbers_f1` Q4_K_M **0.6263** sd 0.0039 vs Q6_K
**0.5985** sd 0.0003 -- d = -0.0278 at 12.2x se, run ranges DISJOINT; recall -0.0257, precision
-0.0297, `entities_f1` flat at -0.0019 (0.5x se). Q6 is also 14% slower (43.58 vs 38.22 s/doc) for
5.6 GB more VRAM. Limits: one model, one task, one publisher, and k-quants are not strictly
bpw-ordered, so this is two artifacts rather than a precision dial -- the card's MODEL_ACCEPTANCE says
the same: a quantization result on one model says nothing about another.
NOT COMPARABLE TO THE vLLM ROWS: both ladder arms ran in CHAT mode at 8,192 per slot, so they carry
`production_prompt_path: false` (the llama.cpp `/v1/completions` entry says why), against vLLM's
completions path at 32,768. Gemma 3 lands at 0.6263 here and 0.6220 on vLLM w4a16 -- close across
three moved axes, suggestive and not evidence.
vLLM HAS NO USABLE RUNG ABOVE 4-BIT AT 27B, measured: w8a16 weights of 27.26 GiB leave a 4,176-token
KV pool, below this eval's own 7,617-token worst case, and throughput collapses 419.6 -> 92.8 tok/s
with 2 of 6 requests resident. That is why the ladder moved engines. And llama.cpp CUDA is **2.54x**
slower wall than vLLM on the same 40 articles at concurrency 6 (260.9 s vs 102.7 s; per-request
38.22 s vs 14.28 s = 2.68x) -- not the 15x an earlier revision carried, which divided per-request
latency by a throughput figure.

## DEFERRED WORK

Work decided and not yet scheduled, with the decision that deferred it.

| impact | measured | status | entry |
|---|---|---|---|
| A | 2026-09-16 | OPEN | FRED name-drift propagation into retired_at: a rename after the D-13 migration stays proposable |
| A | 2026-09-16 | AWAITING-DECISION | Extraction__GuardsEnabled=false -- awaiting an owner decision |
| A | 2026-09-16 | OPEN | The staleness stamp measures a successful FETCH, not an advancing quote (design call) |
| A | 2026-09-04 | OPEN | Labeller quality at n=5 through production's CoD prompt+schema, 2026-09-04 |
| B | 2026-09-16 | OPEN | Alert-continuity acceptance (sentinel-resolution-signal) re-measured: still NOT met |
| B | 2026-09-16 | OPEN | Dependency debt: the Cryptography.Xml 10.0.8 pin is now the NU1903 exposure (7 csproj) |
| B | 2026-09-16 | OPEN | The stamps table's single-writer invariant is one negative test plus convention |
| B | 2026-09-16 | OPEN | A failed Prometheus reload is a GREEN deploy with the old ruleset still evaluating |
| B | 2026-09-16 | OPEN | The coverage rule's 18 is pinned in two files plus a deploy, and the rule is one-sided |
| B | 2026-09-16 | OPEN | A runtime-added symbol inherits the PROCESS-START origin (first reading = container age) |
| B | 2026-09-04 | OPEN | ExtractionOptionsValidator has no rule for Thinking=Enabled with a non-empty suffix |
| B | 2026-08-17 | OPEN | FinnhubCollector quote-staleness gauge cardinality is bounded by the DATA, not the code |
| B | 2026-08-17 | OPEN | A Postgres lock wait inside MigrateAsync defeats the 3-minute retry budget (7 services) |
| B | 2026-08-17 | OPEN | finnhub_quote_staleness_origin_durable LATCHES at 1 and nothing ever lowers it |
| C | 2026-09-16 | OPEN | Hosted labelling routes: json_schema enforcement is per-PROVIDER (routes + ZZZ pre-flight) |
| D | 2026-09-16 | OPEN | The cp A B row in the guard suite has gone decorative |
| D | 2026-09-16 | OPEN | The gate's corpora and harness are still not in the repo |
| D | 2026-09-07 | OPEN | The gate refuses its own maintenance; README bypass commands are untested |
| D | 2026-09-05 | DO-NOT-MERGE | #935 ansible-gate -- BLOCKED, do not merge (closed unmerged; headline is a gate fixture) |
| D | 2026-09-05 | OPEN | Recorded, not chased: gate-path READS the corpus surfaced (incl. 16 read-only loosenings) |
| D | 2026-09-04 | OPEN | probe_engine cannot tell no-/version from nothing-listening; check_staleness reports it |
| D | 2026-08-17 | OPEN | Second siting axis (service cards) has no harness self-check; reads as a guard regression |
| D | 2026-08-16 | OPEN | A guard copied to a scratch dir cannot see the bare-basename rule; the harness fails GREEN |
| D | 2026-08-16 | OPEN | 4 write shapes still reach the gate layer and are ALLOWED (3 of them DENY on #935's lexer) |
| E | 2026-09-16 | OPEN | Accepted risks, do not re-flag: plaintext DB password and tracked OfrCollector/.env |
| E | 2026-08-17 | OPEN | __EFMigrationsHistory is one shared table for every ATLAS service in atlas_data |
| E | 2026-08-16 | OPEN | The README's bare '41 shapes' for #935: its provenance (series lives in LESSONS L15) |
| E | 2026-08-15 | OPEN | A do-not-merge DECISION on #935 went unread through four rounds (first occurrence) |

**FRED name-drift propagation into `retired_at` -- deferred, not wired.** FRED discontinues a series by renaming it in
place (the name gains "(DISCONTINUED)"), so a row registered BEFORE the rename keeps its live name and stays inside
`InstrumentSearchScope.Proposable` (`SecMaster/AGENT_README.md` D-13) until that renamed name is next persisted here,
where `InstrumentRetirementStamp` stamps it at the SaveChanges boundary. No collector SIGNAL retires a row at runtime:
FredCollector's `/api/series` payload carries no discontinued flag, and its 90-day `IsDiscontinued` heuristic was
measured to flag every annual series, so it is NOT wired as a writer. Deferred by D-13's own "what this does NOT do"
clause until FRED exposes the state or the collector carries the name through. Measured 2026-09-16 (`atlas_secmaster`,
psql SELECT-only): 1,308 of 28,732 instruments carry the marker, and 1,182 of those still hold an ACTIVE
`FredCollector` source mapping (plus 11 `SentinelCollector`) -- the mapping is untouched by D-13, so exact lookups
and `ResolveBatch` still resolve them (D-11); only the PROPOSING tiers stop seeing them. Re-check after deploy:
`SELECT count(*) FROM instruments WHERE name LIKE '%(DISCONTINUED)%' AND retired_at IS NULL;` -- expected 0, and any
growth is a rename that arrived through a path the persist-boundary stamp does not see. Mapping count:
`SELECT count(DISTINCT i.id) FROM instruments i JOIN source_mappings sm ON sm.instrument_id = i.id WHERE i.name LIKE
'%(DISCONTINUED)%' AND sm.collector = 'FredCollector' AND sm.is_active;` -> 1,182.

**`Extraction__GuardsEnabled=false` — AWAITING AN OWNER DECISION.** A deliberate experiment, not an accident:
`/opt/ai-inference/compose.yaml:1269` carries it dated 2026-05-03 (re-read 2026-09-16, still `false`; the line
moved from :1161), on the rationale that the Phase 4.3
client-side guards (token-overlap + asset-class) were defensive bandaids that had become precision sinks, with
the PR-3 prose template plus cosine expected to disambiguate without literal-token rules. The flag is still
false and the decision is still open; this entry prejudges nothing.
THE EXHIBIT IS GONE, and the entry no longer claims it. It cited a probe of 2026-08-15T00:46Z returning
`symbol=BBAI` (BigBear.ai) WITH an instrument id at confidence 0.85 for `q=OpenAI` — a private company
resolving to an unrelated public issuer. Re-run 2026-09-05 through the SecMaster MCP
(`hybrid_resolve("OpenAI")`): `method=RagSynthesis`, `resolution=null`, the RAG answering `NO_MATCH`, and five
70%-similar Business-Formations vector matches with no instrument attached. So the COST side of this tradeoff
currently has no worked example at all — find a fresh false accept before re-arguing it in either direction,
and do not quote the BBAI case as if it were live.
Re-check: `hybrid_resolve` on a private-company surface (`OpenAI`, `Anthropic`, `SpaceX`) and read whether any
returns a non-null `resolution`. `SubjectNameNormalizer.SharedTokenCount("OpenAI", "BigBear.ai Holdings")`
scores 0 and WOULD reject that pair; the guard simply does not run. It needs
no deploy.

**The staleness stamp measures a successful FETCH, not an advancing quote — needs a design call, not a reflex fix.**
`QuoteCollectionWorker.cs:179` upserts and stamps on any non-null quote and never consults `quote.Timestamp`.
A halted or delisted symbol whose upstream keeps serving a frozen non-zero `t` therefore advances the stamp every
cycle with no exception raised: staleness stays ~60s, the durable gauge stays 1, no error counter moves, and the
matrix takes a price frozen at the halt date. Every leg of both PR #975 rules is false throughout. The reflex fix
(stamp only when `t` advances) is WRONG as stated: a closed market legitimately freezes `t` across a weekend and a
holiday, which is why the sibling collectors' dead-men use 3-4 DAY windows while this one uses 6h. Any fix needs a
market-calendar-aware window or a separate slower gauge, so it is a design decision. Re-check:
`sudo nerdctl exec timescaledb psql -U ai_inference -d atlas_data -c "SELECT symbol, max(timestamp) FROM finnhub_quotes GROUP BY symbol ORDER BY 2;"`
— a symbol whose max(timestamp) is days old while its row in `finnhub_quote_collection_stamps` is minutes old is
this defect, live. Migration `20260817134302_QuoteCollectionStamps` has shipped, so the table exists in prod and a
`relation does not exist` here is no longer an expected outcome (line ref and caveat corrected 2026-09-16).

**Labeller quality at n=5 through production's CoD prompt+schema, 2026-09-04.** Routes/enforcement:
"hosted labelling routes" below. DeepSeek-V4-Pro@deepinfra found ZERO facts the other two did not, but
missed 6 incl. a headline `$90/oz`: under-extraction, zero junk. Kimi-K3 MANUFACTURES facts ('third
quarter'->3, '20:49 ET'->"20:49"), half its barren-article extractions fabricated; 4/5 non-streamed
calls 504'd (120.1s); `reasoning_tokens: 0` is false (23,150 chars in `delta.reasoning_content`);
non-deterministic at temp 0. Opus 5 degrades `source_entity`, the resolution anchor, to a metric label
at 33%. The least-bad route still errs in kind: `$35.69` bound to January where the article says December -- wrong period,
silent series corruption. CAVEATS: n=5 is not a ranking; the recall denominator is a regex proxy
over-counting clock times and years; 4/5 were macro/market-wrap, so re-measure the 33% on
earnings/analyst-action first.
Re-check: the artifacts live only in `/tmp` (non-durable -- expect them gone). Re-running is ~$1.32 /
32 requests (5 substrate articles x 3 labellers) after the ZZZ pre-flight below; until then these
figures ARE the record.

**The alert-continuity acceptance from the sentinel-resolution-signal epic was never re-measured, and
re-measuring it now says it is NOT met.** Evicted from STATE.md 2026-08-26; the criterion was written
before that epic's first dispatch and left `UNMEASURED since the deploys`, so it would have been lost
at the reset. Criterion: no alert fires continuously for 24h unless it has an open entry in this file
naming the condition, or is retired. Baseline recorded at epic start (2026-08-14): 3 continuously
firing (2x gemini-resolver, 1x TE approaching-severe) plus 2 flapping.

Measured 2026-08-26T15:23Z, Prometheus up 620h so the window is not truncated by a restart
(`max by (alertname, alertstate) (count_over_time(ALERTS[24h]))`; 15s scrape, so 5,760 samples = the
full 24h):

- `PatternDataSeverelyOverdue` — **5,760 firing, the entire window.** Has entries here.
- `GeminiResolverCapRefusingDemand` — 3,180 firing (~13.2h). **No entry in this file names it.**
- `GeminiResolverApproachingFreeGroundingCap` — 2,310 firing (~9.6h). One entry mentions it.
- `GeminiResolverBillableCallRateHigh` — 1,313 firing (~5.5h). **No entry in this file names it.**

So the count did not improve against baseline; it moved sideways, and two of the four now-sustained
alerts have no entry naming their condition, which is the specific thing the criterion forbids.
**Beware the query shape:** `count by (alertname) (count_over_time(ALERTS[24h]))` counts SERIES, not
samples, and returns 1-4 for a continuously-firing alert — it reads as "nothing is firing" and is how
this was first mis-called on 2026-08-26. Use `max by (...)`, and anchor `time=` to actual `date -u`.

Re-measured 2026-09-16T18:30Z, same query and window: `PatternDataSeverelyOverdue` 5,745 firing;
`GeminiResolverCapRefusingDemand` **5,205 firing, up from 3,180**; `HighMemoryUsage` 1,758 firing (not in the
2026-08-26 list; no entry names it, see the grep below); `GeminiResolverBillableCallRateHigh` 1,343 firing.
The two alertnames flagged above still occur in this file only inside this entry (both `grep -n` hits for each are
this entry's two measurement lines); `HighMemoryUsage` `grep -c` = 2, both inside this entry, so no other entry names it either.
Criterion still unmet.

Disposition per alert (entry-or-retire) is unchosen. This is measurement debt, not a defect in any one
rule.

**Dependency debt — the 10.0.8 pin WAS the NU1903 fix and has become the NU1903 exposure.** Seven `.csproj` pin
`System.Security.Cryptography.Xml` 10.0.8, against which five HIGH advisories stand, all range `[10.0.0, 10.0.9]`
(GHSA-g8r8-53c2-pm3f, GHSA-8q5v-6pqq-x66h, GHSA-23rf-6693-g89p, GHSA-cvvh-rhrc-wg4q, GHSA-mmjf-rqrv-855v). Fixed
version **10.0.10** is already in-tree (`Reports.Hosting`, `Reports.Substrate`), so the bump target is known-good.
Only ONE of the seven is production: `MacroSubstrate/src/MacroSubstrate/MacroSubstrate.csproj`, pinned by #604
(`869d9054`), NOT by #703 (`eb81d03b`, which pinned the six SecMaster / FinnhubCollector / AlphaVantageCollector
test projects) -- a fix scoped to #703's files misses exactly the production project. There is **no
`Directory.Packages.props`**: the version is restated per file, which is why it rotted unevenly. Central package
management is the durable fix; bumping seven files is the cheap one. Test-only and unchanged:
`SQLitePCLRaw.lib.e_sqlite3` 2.1.11 (GHSA-2m69-gcr7-jv3q, HIGH, range `(, 2.1.11]` -- upper bound INCLUSIVE;
transitive, SQLite is the unit-test provider, prod is TimescaleDB).
Measured 2026-08-15; re-verified 2026-09-16 (7 files at 10.0.8, 2 at 10.0.10, no `Directory.Packages.props`).
Re-check: `grep -rn Cryptography.Xml --include=*.csproj .`, and for the advisory set
`curl -s --compressed https://api.nuget.org/v3/vulnerabilities/index.json` then the base+update pages
(`--compressed` is required; without it the response is gzip and unreadable).

**The stamps table's single-writer invariant is one negative test plus convention.** `finnhub_quote_collection_stamps`
is the staleness ORIGIN and D-2 INV stamps-single-writer requires the collection loop to be its only writer -- but
`UpsertQuoteCollectionStampsAsync` sits on the shared `IFinnhubRepository` that `SeriesManagementService` already
injects for other reasons, and `FinnhubDbContext.cs:14` exposes a public `DbSet<QuoteCollectionStamp>` that
bypasses the repository entirely. What actually holds the line is one test
(`SeriesManagementServiceTests.TriggerCollectionAsync_DoesNotWriteTheDurableStalenessOrigin`) that names one
caller: a second writer added anywhere else compiles, passes, and silently disarms the dead-man across the next
restart. Structural fix is a refactor, not a patch: a narrow writer interface the cycle alone takes, and a
non-public `DbSet`.
Re-check (re-baselined 2026-09-16: the earlier `QuoteCollectionStamps\|Upsert...` pattern now returns 15 src hits
across the `Get*` reader, migrations and DbContext config, so its "fifth hit" rule fails toward a false alarm):
`grep -rn UpsertQuoteCollectionStampsAsync FinnhubCollector/src --include=*.cs` -> 3 hits, the interface, the
`FinnhubRepository` implementation, and the one caller `QuoteCollectionWorker.cs:179`. A fourth hit is a second writer.

**A failed Prometheus reload is a GREEN deploy with the old ruleset still evaluating.** `deploy.yml:650` runs
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
It is also ONE-SIDED: `count(...) < 18` fires on 17 and is silent on 19, so a symbol added and never reflected in
the rule leaves the constant quietly wrong in the direction that WEAKENS it (a true universe of 25 can fall to 18
-- seven symbols dark -- with every leg of every rule false). Same one-source fix, or a rule on ANY divergence
rather than a floor. Re-check: the DB active-Quote count (`get_all_series_admin`, or SELECT-only
`SELECT count(*) FROM finnhub_series WHERE is_active AND series_type = 'Quote';`) against the `< 18` -- a DB count
ABOVE the constant is this defect, live and silent. 2026-08-17 and 2026-09-16: DB 18 == rule 18 == published 18.

**A symbol added at runtime inherits the PROCESS-START origin, so its first staleness reading is the container's
age.** `TrackActiveSymbols` does `GetOrAdd(symbol, ProcessStartOrigin)` (`FinnhubMeter.cs:156`), and that origin is
fixed once per process, so a symbol added into a long-lived container publishes staleness measured from the
container's start, not its own -- the container's age in seconds on its first scrape. The DIRECTION is safe (it
over-reports, never under-reports) and it self-corrects on the first cycle that collects the symbol, well inside
the 15m dwell. What it costs: a per-symbol value that lies to whoever reads the gauge by symbol (which the alert's
own runbook instructs), and a symbol that never collects (permanent 403) shows an age implying a stall far older
than the symbol. Open design question: start runtime-added symbols at `UtcNow` instead -- it trades this for a
symbol that is genuinely uncollectable from birth taking 6h longer to page. Mechanism unchanged 2026-09-16.
Re-check: `sudo nerdctl container inspect finnhub-collector --format '{{.Created}}'` against
`finnhub_quote_collection_staleness_seconds{symbol="<a symbol added since that time and not yet collected>"}` -- the
gauge reports the container's age, not the symbol's.

**`ExtractionOptionsValidator` has no rule for `Thinking=Enabled` with a non-empty
`ThinkingSuppressionSuffix`, so a suffix an operator explicitly set is accepted at boot and discarded on
every request.** `ThinkingControl.FormatPrompt` (`SentinelCollector/src/Services/ThinkingControl.cs:32-34`) appends
the suffix on the `Disabled` branch only -- correct, and D-26's paired negative test asserts exactly that. The gap is
one layer up: two settings that cannot both be honoured are accepted in silence, the "configured and inert" shape
D-26 exists to prevent. NOT A SUPERSESSION of D-26: its guard, precondition and tests are untouched.
MEASURED 2026-09-04: `grep -c Thinking SentinelCollector/src/Configuration/ExtractionOptionsValidator.cs` -> **0**
(nine `failures.Add` rules, none reads `Thinking` or `ThinkingSuppressionSuffix`). Remedy unchosen: reject at boot
(`failures.Add` on `Thinking != ThinkingMode.Disabled && !string.IsNullOrEmpty(ThinkingSuppressionSuffix)`, the
file's existing shape) or warn once at startup. Pin either way with
`ExtractionOptionsValidatorTests.Validate_ThinkingEnabledWithSuppressionSuffix_Fails` plus the paired
`Validate_ThinkingEnabledWithoutSuffix_Succeeds`, so the rule is not a refuse-everything.
Re-check: the grep above. `0` means this entry is still true; nonzero means it has been closed.

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

**Hosted labelling routes -- measured 2026-09-04. STRICT `json_schema` enforcement is a per-PROVIDER
property, not a per-model one.** `.claude/skills/supervisor-mode/SKILL.md` ORACLE_ROUTING routes labelling / gold /
bulk-oracle work to a hosted OpenAI-compatible API (user directive 2026-09-04; Azure Foundry access revoked) and
points here by the literal string "hosted labelling routes". This entry is that target.
THE FINDING, WHICH CARRIES ITS OWN NEGATIVE CONTROL: `zai-org/GLM-5.2` served by **deepinfra** honours an
`enum: ["alpha","beta"]`; the SAME model served by **zai-org's own endpoint** ignores the schema, invents `company` /
`event_type` / `financial_metrics`, and returns HTTP 200 while doing it. The route key is (model, provider), never
the model alone; that pair is also the proof the probe discriminates rather than flattering everything it touches.
MEASURED ROUTES, HF router `https://router.huggingface.co` (NO `/v1` suffix on the base; the client appends it):
```
ENFORCED      Qwen/Qwen3.8-2.4T-A95B             @ deepinfra
ENFORCED      deepseek-ai/DeepSeek-V4-Pro-0813   @ deepinfra
ENFORCED      moonshotai/Kimi-K3                 @ deepinfra
ENFORCED      zai-org/GLM-5.2                    @ deepinfra
ENFORCED      moonshotai/Kimi-K3                 @ fireworks-ai  -- but now answers HTTP 402
BEST-EFFORT   zai-org/GLM-5.2                    @ zai-org       -- 200 + invented fields, SILENT
FAILS CLOSED  deepseek-ai/DeepSeek-V4-Pro-0813   @ novita        -- HTTP 400, honest refusal
NO ACCESS     together | featherless-ai                          -- HTTP 403 for this token
```
Four model families enforce on deepinfra, so the property tracks the PROVIDER. CHOSEN: **DeepSeek-V4-Pro @
deepinfra** -- zero reasoning tokens, **~$0.00013/call OBSERVED**; whole probe **$0.045 across 29 requests,
OBSERVED**. Those are the only BILLED prices here; the pre-flight's ~$0.0003 is derived from the first. Never average
a pricing-page LIST price in with any of them.
PRE-FLIGHT BEFORE ANY LABELLING BATCH (~$0.0003 for two calls; never skipped, never replaced by `GET /v1/models`
`providers[].supports_structured_output`, which predicted all 7 outcomes but is a CLAIM -- zai-org advertises `false`
and serves 200 rather than refusing, so an advertised `true` is equally unverified): POST `/v1/chat/completions` with
`model: "<id>:<provider>"`, `max_tokens` >= 1024, and a `response_format` `json_schema` whose `required` includes a
`ticker` of `pattern: "^Z{3}$"`, against an article naming no such ticker. Twice -- one sample can look enforced by
luck. `ZZZ` both times = enforced; a plausible invented ticker, a missing field, or one of each = best-effort, and
that provider does not label. An enum probe alone is weak: a competent model obeys a two-value enum on semantics.
BUDGET TRAP: reasoning models burn 600-3000 chars of hidden thinking before any answer; GLM returned
`finish_reason: length` with EMPTY content at `max_tokens` 80 AND at 512. Use >= 1024 on any route here. A cut-off
response lands in `truncated` AND `schema_invalid`, so a lowered budget reads as a MODEL defect in the scorecard.
`run_model.py` ON THIS ROUTE: `--allow-unidentified-engine` is MANDATORY (the router answers neither `/version` nor
`/props`, so `probe_engine` returns `engine: None` and `main` exits 2; do NOT loosen the gate, the flag stamps the
scorecard `engine_identified: false`, which is the point). Always `--model <id>:<provider>`: `--model` is forwarded
verbatim and never refused, so a bare id lets the router pick a provider whose enforcement was never established.
`strict: true` is NOT required (deepinfra enforced without it). The `Authorization` header and per-request `usage`
capture (`aggregate_usage`, `estimated_cost`, `cost_reported_by`) are WIRED since #1009 (`8dc8fca7`);
`eval_harness.py` has zero `usage` readers by design -- it scores an existing scorecard and makes no calls.
ACCESS: `~/.hf-inference` holds the router token (present since 2026-09-04); the 2026-09-04 finding "not wired on
this box" is closed. Per ORACLE_ROUTING's own rule, never read ACCESS from ARTIFACTS -- confirm the file is present
before pricing work that assumes it.
THE DISTINCTION TO PRESERVE: the router is CHAT-only. LABELLING (produce gold) and SCORING (grade production's own
`/v1/completions` path against that gold) are two jobs on two endpoints; the second runs against a LOCAL engine and
always will. `eval_harness._acceptance_evidence` stamps `production_prompt_path: false` on any non-completions-mode
run, so a hosted scorecard is never acceptance evidence however good its numbers look. Nothing here measures label
CORRECTNESS -- only enforcement, failure mode and cost; do not read `ENFORCED` as "good labels" (n=5 entry above).
Re-check (free, local; re-baselined 2026-09-16 -- `Authorization` and `usage` are NON-ZERO since #1009, so a zero
in either now means the file moved or the command is broken, not that a gap reopened):
```
for p in Authorization usage '"strict"' Content-Type allow_unidentified_engine; do
  printf '%-26s %s\n' "$p" "$(grep -c -- "$p" LlmBenchmark/scripts/run_model.py)"; done
test -s ~/.hf-inference && echo access-present
```
2026-09-16 -> `Authorization 3` / `usage 10` / `"strict" 0` / `Content-Type 1` / `allow_unidentified_engine 1` /
`access-present`. Only `"strict"` is 0 BY DESIGN; every other row is a positive control.

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

**The gate refuses its own maintenance, and until 2026-08-16 the documented escape could not be typed.**
`ansible-gate-guard.sh` denies every Edit, Write and Bash write to `.claude/hooks/**` -- correct, and the reason
the layer holds. But the guard reads gate paths out of a command's CONTENT, so naming the file you want to
authorise in the bypass-scoping command is itself a gate act: the README's own example
(`printf '%s\n' .claude/hooks/ansible-gate-guard.sh > .claude/.ansible-gate-confirmed`) was DENIED, leaving only
`touch .claude/.ansible-gate-confirmed`, a four-hour bypass of the WHOLE layer -- the false-denial-forces-a-global-bypass
shape #925 was reverted for, aimed at the guard itself. Matching is SUBSTRING, so a STEM (`ansible-gate-guard`)
scopes identically and contains no gate path; the README example is now stems (`.claude/hooks/README.md:1100`) and
the guard's deny message documents a no-bypass scratch-copy maintenance workflow (measured 2026-09-07).
STILL OPEN: nothing tests that the README's bypass-section commands run.
Re-check: feed each fenced command in that section to the hook on stdin and assert it is not denied.

**#935 ansible-gate — BLOCKED. Do not merge and do not patch-round it.** CLOSED unmerged 2026-09-05. The headline stays
verbatim: `run-pr-verdict-smoke.sh` BD17/BD18 read it to refuse an approve of exactly {729, 935}, so re-wording it is
a gate change with a test to update in the same PR. Three successive rounds each claimed "loosened writes = 0" and
each was falsified by hand-probing outside the template sweep; the unsound carve-out was demonstrated end to end
(`hash-object -w` -> `mktree` -> `commit-tree` -> `git checkout <sha> -- <file>`, every step ungated).
The classification-layer salvage landed on main's guard 2026-08-16, object-store carve-out deliberately NOT opened.
MEASURED 2026-08-16, 498-row corpus (447 write, 16 read, 35 ordinary), guards run IN their own hook directories,
against `170be75a` (guard blob `c5ac440a`): writes **tightened 110, loosened 4, NET +106** -- all 4 the MANDATED
`git checkout|restore -- <gate>` seam; reads 3 loosened, counted separately; ordinary work 0 new false denials.
Suite 172/0 at `170be75a` -> 239/0 with the reader/deployed round; no one-change mutant measured zero.
What #935 still held over the salvage is the lexer, worth 3 shapes (`>|`, awk string-literal target, quote-abutting
verb -- the 4-write-shapes entry below). Every "loosened = 0" claimed on this work was falsified by a bigger corpus;
the rule (name corpus SIZE and baseline) is `.claude/skills/supervisor-mode/LESSONS.md` L15.

**RECORDED, NOT CHASED — what the 498-row corpus surfaced beyond the two families it was scoped to fix.**
None is a write against a HARD_STOP path, which is the only reason each was left open (2026-08-16). Every finding is
a READ of a gate or deployed path -- the reader/`dest_last` class working as designed, the gate file is the SOURCE:
`cp <gate file> /tmp/x`, `cp -T` / `cp --no-target-directory <gate file> /tmp/x`, `cat <gate file> > /tmp/x`, and
16 read-only shapes main denied beyond the 8 the suite names (`head`/`wc`/`md5sum`/`diff`/`stat`/`readlink`/`cat`/`jq`
of a gate or deployed file redirected to `/tmp`, `grep -c rm <gate file>`, `cp -r .claude/hooks /tmp/backup`,
`install <gate file> /tmp/x`, `git show HEAD:<gate file> > /tmp/x`, `git grep rm -- <gate file>`,
`git ls-files --stage .claude/hooks`, `git blame <gate file> > /tmp/b`, `git checkout -- .claude/hooks`). Listed as
loosenings rather than folded into the regression corpus, because burying them there is how #935 reported a clean
zero three times. No write shape against the gate layer was left open.
CORRECTED 2026-09-05, and it INVERTS what a reviewer would plan around: against the armed installed guard (blob
`31baafbb`) bare `cp <hook> /tmp/backup.sh` **ALLOWS**, as do `cp -T`, `cp --no-target-directory` and
`cat <gate file> > /tmp/x` -- plan a mutation round without a bypass. What still obstructs is the PREFIXED spelling:
any of the nine `WRAPPER_RE` words (`time nice timeout stdbuf command exec ionice nohup xargs`) in front of the same
cp **DENIES**, naming the SOURCE -- the verb-displacement class recorded as spelling (2) of the sibling-guard entry
in KNOWN DEFECTS, not a separate finding. Write the cp bare.
Re-check: feed each shape to `.claude/hooks/ansible-gate-guard.sh` as
`{"tool_name":"Bash","tool_input":{"command":"..."}}` on stdin and read `.hookSpecificOutput.permissionDecision`;
an allow emits no JSON at all, so empty output IS the allow.

**`probe_engine` cannot tell "this endpoint has no `/version`" from "nothing is listening", and the
checker that reads it now REPORTS that ambiguity rather than resolving it.** `probe_engine` in
`LlmBenchmark/scripts/run_model.py` wraps each of `/version`, `/props`, `/v1/models` in its own bare
`except Exception:` and returns `engine: None` both for a dead port and for a live OpenAI-compatible server that
does not implement `/version` or `/props` (SGLang, a proxy, a future engine). `check_staleness.py` no longer reads
unknown as current (`live: UNIDENTIFIED`, scorecards `DRIFT_UNCHECKED`, non-zero exit), but its one message covers
two remedies -- restart it, or `run_model.py --allow-unidentified-engine` -- and cannot say which it means.
MEASURED 2026-09-04, a local `HTTPServer` stub answering only `/v1/models` against closed port `127.0.0.1:9`: both
return `engine None, engine_version None`; `served_models` is `['some-model']` live and `[]` dead. That
discriminator already exists in the returned dict and no caller reads it:
`grep -c served_models LlmBenchmark/scripts/check_staleness.py` -> **0**.
Remedy unchosen: return reachability as a field distinct from identity (non-empty `served_models` with
`engine: None` IS "reachable but unidentified") and have `check_staleness.py` name the right remedy for each. Pin
with two sibling stubs in `test_check_staleness.py`, one serving `/v1/models` only and one refusing connections,
asserting DIFFERENT operator-facing text -- assert on the difference, since a checker that says the same thing
about everything passes any single-case test.
Probe (2026-09-04):
```
python3 - <<'PY'
import sys, json, threading
from http.server import BaseHTTPRequestHandler, HTTPServer
sys.path.insert(0, 'LlmBenchmark/scripts')
from run_model import probe_engine
class H(BaseHTTPRequestHandler):
    def do_GET(self):
        if self.path != "/v1/models":
            self.send_response(404); self.end_headers(); return
        b = json.dumps({"data": [{"id": "some-model"}]}).encode()
        self.send_response(200); self.send_header("Content-Length", str(len(b)))
        self.end_headers(); self.wfile.write(b)
    def log_message(self, *a): pass
srv = HTTPServer(("127.0.0.1", 0), H)
threading.Thread(target=srv.serve_forever, daemon=True).start()
up   = probe_engine(f"http://127.0.0.1:{srv.server_address[1]}")
down = probe_engine("http://127.0.0.1:9")
print(up["engine"], up["engine_version"], up["served_models"])
print(down["engine"], down["engine_version"], down["served_models"])
srv.shutdown()
PY
-> None None ['some-model']
-> None None []
```
Re-check: the two-stub probe above still prints `None None` twice with `served_models` the only difference, and the
`served_models` grep is still `0`.

**A SECOND siting axis exists, it has no self-check at all, and it surfaces as one ordinary red row.**
`design-intent-dispatch-guard.sh` resolves `REPO=$(cd "$_dir/../.." && pwd)` and globs `"$REPO"/*/AGENT_README.md`;
with no cards under that root it logs `ANOMALY: no service card ... failing open` and returns `none`, which the
suite reports as `design-intent-dispatch-guard returned 'none', expected 'deny'` -- a row that names the GUARD and
reads exactly like a guard regression. The suite's `HARNESS MISCONFIGURED` check covers only the `GATE_BASENAMES`
axis (`$(dirname "$ATLAS_GATE_HOOK")/git-push-guard.sh`) and is silent about cards.
Measured 2026-08-17 on a scratch tree carrying every `.claude/hooks/**` file but no cards: **418/1**; siting the 12
cards at `$REPO` and changing nothing else: **419/0**, matching the in-place count.
NOT A PRODUCTION HOLE: failing open with no cards to enforce against is that guard's deliberate behaviour, and in
the repo the cards always exist. This is harness siting, not a live gap in dispatch gating.
Re-check: copy `.claude/hooks/**` into a scratch tree with no `*/AGENT_README.md` beside it and run the suite --
that row must be the ONLY red one, and must go green when the cards are added.

**A harness that copies the guard into a scratch directory cannot see the bare-basename rule, and it fails GREEN.**
`GATE_BASENAMES` is built from `$_hookdir/*.sh`, i.e. the guard's OWN directory, so a copy in a private scratch dir
knows only the basenames of whatever sits beside it. Measured 2026-08-16: pointing `ATLAS_GATE_HOOK` at a scratch
copy of the CURRENT guard turns 2 of the suite's 234 assertions red ("unwiring by bare basename after cd",
"deleting a hook by bare basename") with no code defect present — and the same 2 rows are silently absent from any
corpus measured that way. The mutation counts in this file are quoted net of that constant. Re-check by running
`run-advisory-guards-smoke.sh` twice, once in place and once with `ATLAS_GATE_HOOK` at a copy, and diffing the
FAIL lists. Not fixed: the honest fix is a hook-dir fixture, and creating files named after the other guards is
itself denied by the installed guard.

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

**Accepted risks, do not re-flag.** Plaintext DB password `atlas_secure_password_2025` in 10+ tracked files, and
`OfrCollector/.env` tracked with `DB_PASSWORD` / `SMTP_PASSWORD` / `FRED_API_KEY`. The user accepted both
explicitly: private repo, LAN-only, public and public-derived data. Rotating touches the DB user and every consumer.

**`__EFMigrationsHistory` is one shared table for every ATLAS service in `atlas_data`.** 58 rows, measured
2026-08-17 (`SELECT count(*) FROM "__EFMigrationsHistory";`). Safe TODAY — EF filters by the migrations assembly
and the IDs are timestamp-prefixed, so collisions need two services to generate the same `yyyyMMddHHmmss_Name` —
but it is a namespace the services compete in rather than an isolation boundary, and nothing enforces the naming
that keeps them apart. Recorded as a known shape, not an action: separate schemas per service would be the durable
answer and it is a migration of the migration table. Re-check:
`sudo nerdctl exec timescaledb psql -U ai_inference -d atlas_data -c "SELECT \"ProductVersion\", count(*) FROM \"__EFMigrationsHistory\" GROUP BY 1;"`
— a duplicate `MigrationId` is the failure mode, and it would surface as a service silently skipping a migration.

**The zero-loosened SERIES for #935 and its salvage — the figures that lived only in `STATE.md`.** The series
itself and the standing rule (any "loosened = 0" must name its corpus SIZE and the baseline it was measured
against; a claim carrying neither is not evidence) now live in `.claude/skills/supervisor-mode/LESSONS.md` L15.
What this entry still holds is the provenance of one bare figure: `.claude/hooks/README.md:1154` says #935 was
"drifting 41 shapes", with no corpus and no baseline. That 41 is #935 measured against MAIN -- not against its own
previous head, where "loosened = 0" was true each time: 41 shapes opened, 32 of them executing a real write,
sandbox-proved, by an agent-scratch sweep whose row count was never recorded. Left as written on purpose: the
README is gate layer and the guard refuses writes to it, so this entry carries the provenance the sentence lacks.

**A recorded do-not-merge DECISION on PR #935 went unread through four review rounds** (2026-08-15). The entry
titled "#935 ansible-gate — BLOCKED. Do not merge and do not patch-round it." landed on main in `f235e79d`
(2026-08-14), predicting in those words the "loosened writes = 0" shape three rounds had already produced; FOUR
further patch-rounds then ran against it with the entry byte-identical on main and on the PR head.
WHY IT SURVIVED: every dispatch asked agents to verify CLAIMS -- do the numbers reproduce, do the citations resolve
-- and nobody was asked whether a DECISION already existed. The discriminator: a claim is falsifiable by
measurement, a DECISION is only reversible by a human, so finding one means STOP and escalate, never "measure
harder until it goes away". PRE-FLIGHT, now step 0 of `SKILL.md` MERGE_GATE SEQUENCE: before reviewing a PR's code,
grep this file for its number and for the decision vocabulary (`BLOCKED`, `do not merge`, `do not patch-round`,
`superseded`, `rejected`). FIRST occurrence, recorded so a second is recognisable; a second sends it to `LESSONS.md`.
Re-check: `grep -n -e '#935' -e 'do not merge' docs/BACKLOG.md` returns the blocking entry today, and a review
dispatch either carries that grep as its first step or it does not.

## PARKED EPICS

A park is a decision; it stands until reversed or the epic ships.

| impact | measured | status | entry |
|---|---|---|---|
| A | 2026-09-16 | AWAITING-DECISION | R2 / S4 observation identity key redesign -- PARKED; trip condition 3 fired 2026-09-16 |
| A | 2026-09-05 | PARKED | #729 regime news-as-staleness redesign -- parked; blockers refuted, un-park is the user's |
| A | 2026-08-26 | PARKED | Candidate-pool epic -- PARKED, asked of the user twice, never started |
| E | 2026-09-16 | PARKED | Claude Code function hooks -- PARKED, not adopted |

**R2 / S4 -- observation identity key redesign -- PARKED on the user's decision, 2026-08-27.** The
thesis holds, and it is not what parked the work: an observation's identity is the entity MENTIONED,
not the measurement TAKEN. Re-measured 2026-08-27, **15,959 of 21,569 published observations carrying
an instrument (74.0%) sit in a colliding `(raw_content_id, instrument_id)` group** -- 3,660 colliding
groups of 9,270. `ObservationCache.AddObservation`
(`ThresholdEngine/src/Data/ObservationCache.cs:47`) truncates both `date` and `asOf` to `.Date`, so
same-day claimants collapse onto one exact key and the `BinarySearch` hit overwrites in place at
`:59`; the survivor is deterministically the LAST published, not an arbitrary one. What parked it is
that the proposed key cannot separate them and the harm is not yet live. Anyone re-proposing
`(instrument, claim_kind, unit)` meets these numbers first.

EVERY AXIS OF THE PROPOSED KEY FAILED A MEASUREMENT.
- `claim_kind` cannot discriminate within one article about one issuer. `cod_json_schema_v1.json`
  emits `numbers[]` (`source_text`, `value`, `unit`, `context`, `source_entity`) and `claims[]`
  (`claim_kind`, `subject`, `predicate`, `object`, `polarity`) as SIBLING arrays with no reference
  between them, and `"additionalProperties": false` on both item schemas (`:56`, `:103`) and on the
  envelope (`:3`) makes adding one structurally impossible without a schema change. The only join
  either producer offers is the subject name. Confirmed independently by PR #997, which shipped the
  columns and found the join absent. That PR's coverage debt is now DISCHARGED rather than recorded:
  6,140 of 802,405 rows carry a `claim_kind`, 2,878 of 25,599 (11.2%) since 2026-09-02, and
  `increase(sentinel_dsl_adapter_claim_atom_total[24h])` splits attached 547 / ambiguous 869 / none 2,935
  (2026-09-05). A high `ambiguous` share is a finding about the DATA -- articles routinely make several kinds
  of claim about one issuer -- not a reason to loosen the per-SUBJECT join.
- `period` is populated on **582 of those 21,569 published-with-instrument rows (2.70%)**, and not
  one has been extracted since **2026-06-30**. THE REASON IS THE V2/DSL CUTOVER, and this bullet has
  carried two retired explanations before this one. It is NOT "the extractor does not do periods"
  (125,351 of 748,317 rows overall carry one, 16.8%) and it is NOT "extracted then lost before
  publish" -- re-measured 2026-08-28, only TWO sources extract a `period` at all in the last 30 days,
  `tsa-checkpoint` (15,001) and `rss-fallback` (11), and those are exactly the two sources absent
  from `Extraction__V2EnabledSources`. Neither publishes anything -- `tsa-checkpoint` has published
  nothing since 2026-02-07 (entry above). Every live-publishing source (`rss`, `rss-mirror`,
  `searxng-content`, `challenger-rss`) is on the v2 path and is at **zero**. `rss` itself went from
  21,132 of 37,367 rows (56.6%) in April to 0 from June, on the month `dsl_block_id` took the source.
  So the discriminator is not merely rare in the colliding population -- the pipeline that FEEDS that
  population does not carry it at all. Full breakdown, the retirement of both earlier framings, and
  the code read that would settle the mechanism: KNOWN DEFECTS, above.
    `SELECT source, count(*) FILTER (WHERE nullif(btrim(period),'') IS NOT NULL) with_period,
       count(*) FILTER (WHERE published_at IS NOT NULL) published
       FROM sentinel.extracted_observations
      WHERE extracted_at > now() - interval '30 days' GROUP BY 1 ORDER BY 2 DESC;`
    # 2026-08-28: tsa-checkpoint 15001/0, rss-fallback 11/0, all four v2 sources 0/publishing
- `unit` is unnormalised, so keying on the raw string FALSE-separates one measurement into several
  series. Over published rows: `PCT` 31,467 vs `percent` 2,605, `COUNT` 9,310 vs `count` 2,209,
  `INDEX` 525 vs `index` 256, `USD` 32,351 alongside `million_USD` / `billion_USD` / `thousand_USD`.
  Normalising costs separation rather than buying it: colliding groups sharing one unit rise from 2,545 / 3,660 (69.5%) raw to
  2,596 / 3,660 (70.9%) once case and magnitude prefixes are collapsed.
- **THE KEY'S MEASURED SURVIVAL RATE IS 75.7%, AND THE "~45% INSUFFICIENT" FIGURE IS RETIRED.** Adding
  the normalised unit to `(raw_content_id, instrument_id)` still leaves 3,500 colliding sub-groups,
  holding **12,084 of the 15,959 colliding rows (75.7%)** -- one colliding group can split into two
  colliding sub-groups, so count the ROWS, not the groups. The ~45% figure appears only as a
  back-reference in PR #997's body; no query in either proposal doc derives it, and it understates the
  insufficiency by a factor of ~1.7. Quote 75.7% with the query below, or re-measure -- do not requote
  45%.

THE HARM WAS LATENT WHEN PARKED -- which is what made parking legitimate rather than a deferral of live
corruption -- AND ONE TRIP CONDITION HAS SINCE FIRED. RE-MEASURED 2026-09-16: trip condition 3 has FIRED --
CHALLENGER_JOB_CUTS published 52,881 for 2026-08-31 at 18:05:55Z under D-32 (#1044-#1046), an OwnedSeries key
with 3 loaded readers; one row so far, so the two-same-day-claimants harm is not yet demonstrated; re-run the
trip-3 SELECT before the next Challenger arrival. It tripped through D-32's provenance keying (a known-publisher
feed keys its owned series from provenance, #1044), not through the resolver outcome condition 3 anticipated.
The park is the user's decision and stands until the user re-decides; a trip condition it named has fired, so its
status here is AWAITING-DECISION, not a quiet park. As verified 2026-08-27:
- **zero of ThresholdEngine's 71 loaded patterns read a `SENTINEL:NUM:` key** (`list_patterns`
  `enabled_only=false` returns 71 of 71 enabled). Three of the 72 repo pattern files name any Sentinel
  key at all, and all three use `SENTINEL:SECTOR:` through value-independent counts.
- **the matrix path cannot collide across articles**: 16,540 `public.macro_observations` rows in 30
  days carry the `{raw_content_id}:sig:{slug}` infix and all 16,540 `source_id` values are distinct.
- **the live exception is BDIY, and its trigger is still unpulled**: 163 rows, 160 published, still
  publishing 2026-08-27 under a bare `OwnedSeries` key; its only reader `baltic-freight-recession` is
  `"enabled": false` and is the one repo pattern file of 72 absent from TE's loaded set. Of the two
  `OwnedSeries` keys that DO have loaded readers, `CHALLENGER_JOB_CUTS` had never carried a Sentinel row on
  2026-08-27 and carries one as of 2026-09-16 (above); `TRUFLATION_CPI` returned 0 rows on 2026-09-16. `ADP_EMPLOYMENT` has 9 rows, last extracted 2026-05-18.
- **a share of these collisions is mis-resolution, which no key can fix** -- different entities filed
  under one instrument id, sampled at ~19% (6 of 32 groups) with the machine proxy re-measuring
  136 / 3,660 (3.72%) today. See the mis-resolution entry above; the lever there is the resolver
  (#988), not the key.

THE REVIVAL PATH IS NOT A KEY REDESIGN. It is **make CoD emit a per-measurement claim reference** so
`numbers[]` and `claims[]` can be joined -- a model / prompt / schema change, not a storage one.
Without that discriminator there is nothing to key ON, and any key shipped in the meantime reads as a
fix while separating 24% of the colliding rows, which is the seam-reports-success failure the whole identity plan
exists to stop. D-18 (`SentinelCollector/AGENT_README.md`) still governs the guard site at
`SentinelCollector/src/Publishers/EventPublisher.cs:105-112`; a revived story names `supersedes D-18`
or stops.

WHAT UN-PARKS IT -- four trip conditions, each with the command that reads it. None was tripped as of
2026-08-27; condition 3 fired 2026-09-16 (above), and the park still stands until the user says otherwise.
1. A LOADED pattern names a `SENTINEL:NUM:` key.
   `grep -rl 'SENTINEL:NUM:' ThresholdEngine/config/patterns --include='*.json' | wc -l`   # expect 0
   AND `list_patterns(enabled_only=false)` -- check BOTH: the grep misses a pattern loaded from
   outside the repo dir, and the loaded set misses a file that is present but not loaded.
2. `baltic-freight-recession` becomes enabled, arming the one bare key Sentinel actually publishes.
   `python3 -c "import json;print(json.load(open('ThresholdEngine/config/patterns/recession/baltic-freight-recession.json'))['enabled'])"`   # expect False
   AND confirm it is still absent from `list_patterns(enabled_only=false)` -- the file's flag and TE's
   loaded set are two different facts, and it is absence from the loaded set that keeps it inert.
3. The resolver starts minting an `OwnedSeries` key that a loaded pattern reads. This is a resolver
   OUTCOME, not a design decision, and #988 raising resolution success is exactly what could trip it.
   `SELECT "Symbol", count(*), count(*) FILTER (WHERE published_at IS NOT NULL) AS published,
      max(extracted_at)::date FROM sentinel.extracted_observations
    WHERE "Symbol" IN ('CHALLENGER_JOB_CUTS','TRUFLATION_CPI') GROUP BY 1;`
   # expect 0 rows returned; either symbol appearing converts this defect from latent to live
   # 2026-08-27: 0 rows. 2026-09-16: CHALLENGER_JOB_CUTS appears, one row -- the key is live (the trip); a SECOND
   # same-day claimant on it would be the collision harm itself, which each re-run also looks for
4. The golden corpus tripwires move. The 15 collision cases assert the key count production produces
   TODAY, so the suite is GREEN while the defect holds still and REDs when the shape changes. THE
   DOTNET RUN IS THE TRIPWIRE; `--check` certifies the corpus that run reads, and trips nothing.
   `nerdctl compose exec -T sentinel-collector-dev sh -c "cd /workspace/SentinelCollector/tests/SentinelCollector.UnitTests && dotnet test --filter 'DisplayName~GoldenCorpus'"`
   # expect green, 70 tests, with all 15 collision cases asserted STILL collapsing. A FAILING
   # `should_still_collapse_distinct_measurements_while_the_identity_defect_is_open` IS this condition
   # tripping: the fix has landed, regenerate the fixture and drop ExpectedRedCount. `DisplayName~` is
   # required -- `Name~` matches zero tests and still exits 0 (CLAUDE.md DEPLOYMENT).
   `python3 SentinelCollector/scripts/build_golden_corpus.py --check`   # expect "corpus intact", exit 0
   # Two named verdicts, and only the first is a defect. CORPUS DRIFT (exit 1) = the committed corpus
   # disagrees with itself, someone hand-edited a fixture, the spec or MANIFEST.md, and the dotnet run
   # above is asserting against a corpus nobody vouches for. PRODUCTION MOVED (exit 0) is EXPECTED and
   # is NOT this trip condition: the resolver re-resolves retroactively, so the live rows behind a
   # fixture move on their own. Measured 2026-08-28, one day after this line last read clean, 7 of 30
   # fixtures had moved and 5 no longer collapse live -- 208 rows NULLed to `NoResolution` with the old
   # ids kept in `OriginalInstrumentId`. NEVER rebuild to quiet it: a rebuild overwrites the very key
   # counts these tripwires assert with today's.

Re-check the population and the key's survival (SELECT-only):
  `WITH pub AS (SELECT raw_content_id, instrument_id,
                 regexp_replace(lower(coalesce(nullif(btrim(unit),''),'(null)')),
                                '^(million|billion|thousand|trillion)_','') AS unit_class
               FROM sentinel.extracted_observations
               WHERE published_at IS NOT NULL AND instrument_id IS NOT NULL),
        grp AS (SELECT raw_content_id, instrument_id, count(*) c FROM pub GROUP BY 1,2),
        keyed AS (SELECT raw_content_id, instrument_id, unit_class, count(*) c FROM pub GROUP BY 1,2,3)
   SELECT (SELECT count(*) FROM pub) AS pub_rows,
          (SELECT sum(c) FROM grp WHERE c>1) AS colliding_rows,
          (SELECT count(*) FROM grp WHERE c>1) AS colliding_groups,
          (SELECT sum(c) FROM keyed WHERE c>1) AS rows_surviving_the_unit_key;`
  # 2026-08-27 returned 21569 | 15959 | 3660 | 12084
  `SELECT count(*) FILTER (WHERE nullif(btrim(period),'') IS NOT NULL) AS with_period, count(*)
     FROM sentinel.extracted_observations WHERE published_at IS NOT NULL AND instrument_id IS NOT NULL;`
  # 2026-08-27 returned 582 | 21569

**#729 — regime news-as-staleness redesign.** Spec is PR #729, DO-NOT-MERGE. Intent: FRED/OFR benchmark is the slow
grounding anchor; Sentinel news is a fast-decaying coincident perturbation weighted by benchmark STALENESS, so the
system measures economic significance rather than coverage volume. Phases 1 / 2a / 2b are built and deployed
(2b in shadow), #730-#736.
- **BOTH STATED BLOCKERS ARE REFUTED, AND THIS ENTRY DOES NOT UNPARK THE EPIC [2026-09-05].** The Phase 2c hold
  rested on two numbers: embedding coverage "still ramping" (`missing_embedding` ~0.57) AND shadow
  `magnitude_ratio` ~2.5, "meaning dedup AMPLIFIES or sign-flips the net rather than gently compressing it".
  Measured over 7d: **0.0000479** and **1.0434**. Coverage finished ramping and dedup now compresses gently,
  which is precisely the condition the hold was waiting for. The entry carried NO re-check, which is why it sat
  refuted for months. Whether 2c, Phase 3 and the backtest harness resume is the USER'S decision, not an
  agent's, and nothing here makes it.
- **THE HEADLINE STAYS WORDED AS IT IS, DELIBERATELY.** PR #729 itself MERGED 2026-06-16T12:27:10Z — it was the
  review surface, not a change. But `run-pr-verdict-smoke.sh` BD17/BD18 read this file's first line and assert
  that the live backlog refuses an approve of exactly {729, 935}. Re-wording it is a GATE change with a test to
  update in the same PR, not a docs edit — do not tidy it in passing.
- **Phase 3** — staleness crossfade `news_weight = g(benchmark_age/cadence)` plus benchmark-anchored aggregation.
  This is the principled fix for the structural news-overweight (news MACRO cells run ~2-4x FRED magnitude and
  net-negative, which washes out sector differentiation into an over-neutral regime). Gated on 2c.
- **Backtest harness** is the validation layer that unblocks all of this — shadow has no ground truth. Outcome
  data located: `finnhub_quotes` XLE/XLF/XLV/XLK ~6.5mo, SPY/QQQ, Yahoo EOD backfill for all 11, AV
  WTI/BRENT/NATGAS to 1986. First gate is establishing a realized-sector-return series.
- **Classifier sign-fix is parked** — do not ship it off the energy anecdote; an n=6 peek showed news
  directionally right, but it needs a real backtest N. `commercial-paper-stress` de-flagged in the interim
  (#733).
Re-check, anchoring the eval instant to an actual `date -u` (metric prefix
`thresholdengine_observation_projector_news_clustering_`):
  `sum(increase(<prefix>missing_embedding_total[7d])) / sum(increase(<prefix>magnitude_ratio_count[7d]))`
  # 2026-09-05T13:50Z -> 0.0000479, against the ~0.57 this entry held the cutover on
  `sum(increase(<prefix>magnitude_ratio_sum[7d])) / sum(increase(<prefix>magnitude_ratio_count[7d]))`
  # 2026-09-05T13:50Z -> 1.0434, against ~2.5. Lifetime totals 4 and 49,030, so both series are alive and the
  #   near-zero is a real zero, not an absent series.
  # A ratio back above ~2, or a missing-embedding share back near 0.5, RE-ARMS the original blocker. That is
  #   the drift this entry had no way to detect, and it is why the numbers now travel with the command.

**Candidate-pool epic — PARKED, asked of the user twice, never started.** Evicted from STATE.md
2026-08-26 at the epic boundary; it was the last durable thing in that file and would have been lost.
Intent: the candidate surface that feeds extraction admits entities on a filter that gates 4.3% of the
rows attaching instruments (see the candidate-surface entry above), so the pool is the real fix behind
the feeds that resolve to nothing. Cost of getting it wrong: it re-admits the wrong-ticker GIGO class
that was paid for twice already (FRED surfaces, then the paid resolver), and the free wrong-ticker
resolutions were worse than the bill. Wants shadowing before any cutover.
- **Volume it would touch, measured 2026-08-26:** 210,000 (Jun) / 187,442 (Jul) / 167,270 (Aug-to-26)
  observations per month. The "~115k rows/month" figure carried in STATE.md since 2026-08-14 was stale
  and is retired — re-measure, never re-quote it.
- **NOW OVERLAPS the extraction-identity work.** `docs/proposals/extraction-identity-remediation.md` R2
  re-keys observation identity on (instrument, claim kind, unit). That addresses part of the same
  surface from the other end. **Decide whether this epic is absorbed by R2 or stays distinct BEFORE
  either starts** — running both independently re-opens the GIGO class from two directions at once.
- Re-check the volume: `SELECT to_char(date_trunc('month', extracted_at),'YYYY-MM'), count(*) FROM
  sentinel.extracted_observations GROUP BY 1 ORDER BY 1;`

**Claude Code function hooks — PARKED, not adopted [2026-09-05].** TypeScript in-process plugin
middleware: typed events, `next()` composition, `deny`, argument rewriting, `$.ui.ask`,
`$.model.classify`, `$.store` (cross-session). Unreleased — proposal #91870, flag
`CLAUDE_CODE_ENABLE_FUNCTION_HOOKS=1`, runtime in build 2.1.260; this box ran 2.1.261 when parked and 2.1.273 on 2026-09-16 (whether the flag shipped was not
checked). Parked
because composition is strictly serial (their figure: 8x300ms, ~640ms parallel vs ~2.4s chained)
and our 18 scripts / 36 registrations, mostly PreToolUse, fire on every tool call. Revisit for the
JUDGEMENT half of memory hygiene (`$.model.classify`), which shell cannot do.
Source: https://claudefa.st/blog/tools/hooks/function-hooks
Re-check: `claude --version` past 2.1.261 with the flag shipped; `ls .claude/hooks/*.sh|wc -l` (18),
`grep -c '"command"' .claude/settings.json` (36).

## PRACTICE NOTES [cheap checks that were skipped three times or more]

Checks that keep being skipped.

| impact | measured | status | entry |
|---|---|---|---|
| E | 2026-09-16 | OPEN | A defect found in a branch is not a defect on main |
| E | 2026-09-16 | OPEN | Scoped-deploy collateral: read the tag's blocks first, then record what actually moved |
| E | 2026-08-27 | OPEN | An empty INSTANT query on a cumulative counter is not evidence the counter never fired |

**A defect found in a branch is not a defect on main.** Three fail-opens in one session were reported as live on
main and were not. The settling check is one command — `git branch --contains` on the commit that introduced the
line — and it was never run until a reviewer ran it.

**Scoped-deploy collateral: read the tag's blocks first, then record what actually moved.** On 2026-08-15 this note
called collateral CONDITIONAL because two runs disagreed: the `secmaster` scoped deploy ALSO recreated
`llama-cpu-rag` and `llama-cpu-embed`, while the `sentinel-collector` one recreated nothing else. The playbook
explains the difference. Both `llama-cpu-*` blocks carry the `secmaster` tag, and no neighbour's block carries
`sentinel-collector` (KNOWN DEFECTS, "Scoped secmaster deploy also recreates llama-cpu-rag and the shared
llama-cpu-embed"; it recurred 2026-09-16). What still holds: before a scoped deploy, grep `deploy.yml` for blocks
tagged with that tag; after it, list what actually restarted and record it WITH the run. That `compose up -d`
recreates even an unchanged service is recorded in the playbook for nerdctl 1.7.7
(`deployment/ansible/playbooks/deploy.yml:1217-1218` and `deployment/ansible/playbooks/deploy.yml:1509-1510`, the
cascade incident), and both runs agree with it.
Re-check: `sudo nerdctl container inspect <svc> --format '{{.Created}}'` per service — `container` is
load-bearing (CLAUDE.md VERIFY_TRAP: bare `inspect` resolves the IMAGE and hands back the BUILD time).

**An empty INSTANT query on a cumulative counter is not evidence the counter never fired.** Range-query it (or
`increase()` over the window) before concluding absence; the trap was hit on the pruner counter (2026-08-27, #995).
