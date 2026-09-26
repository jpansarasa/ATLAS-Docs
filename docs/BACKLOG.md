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
| C | 2026-09-20 | OPEN | Test 5's roster proves HANDOVER, not EXECUTION: a `return` after the `processed+=` append records a script as walked and checks nothing |
| D | 2026-09-20 | OPEN | Test 6's interactive-port exemption keys the FILENAME, not the declared path; a file of that name anywhere is exempt |
| D | 2026-09-20 | OPEN | Test 6's per-owner nuget check greps FILE-WIDE; one owned and one hardcoded volume in the same file reads as owned |
| D | 2026-09-20 | OPEN | Known-bad control B pins the audited population as a DECLARED literal (`AUDITED 10`), not a derived one |
| C | 2026-09-20 | OPEN | Parallel-compile gap check is nerdctl-only; a docker-first verification script reads container-less and is never audited |
| C | 2026-09-20 | OPEN | Parallel-compile ownership audit is static text; a call behind an early `exit`, or written early and CALLED late, reads as owned and in-order |
| D | 2026-09-20 | OPEN | Test 5's sibling double-trap sweep word-splits its file list; a spaced or globbed path is silently skipped |
| C | 2026-09-20 | OPEN | Test 6's three population counts are FLOORS, not rosters; tight today, and the first compose file added makes a narrowed glob silent |
| A | 2026-09-21 | OPEN | D-19 passes 16 exchange spellings / 339 active rows through; 2 (FRED 175, Crypto 4) are deliberate non-venues, 14 / 160 await a reviewer's venue call |
| B | 2026-09-21 | OPEN | D-19 changed the embedding's venue prose without a template bump: 7,165 active rows keep "listed on CT" until touched |
| A | 2026-09-21 | OPEN | 6 `.SG` rows carry a wrong-venue FIGI, 5 of them ANOTHER COMPANY's; the suffix is now null (bleeding stopped) but no row is repaired, and 194 `.SG` + 95 `.MC` + 87 `.SI` rows have no venue at all |
| B | 2026-09-21 | OPEN | 5 of the 6 SecMaster migration test classes drive SQL CONSTANTS, never `Up`/`Down`: emptying `Up()` leaves each green |
| B | 2026-09-21 | OPEN | D-19's Down CAS is CODE-granular, not WRITE-granular: 20 of 42 map entries resolve to US (93.3% of the population), so a later write of a DIFFERENT US spelling is invisible to it and to SkippedRestoresSql |
| D | 2026-09-22 | OPEN | Four bare `nerdctl inspect` sites outside the freshness gate get whichever object resolves first; one decides a `stop && rm` of vllm-server |
| B | 2026-09-22 | OPEN | `--tags alerting` and `--tags dashboards` copy the ups/gpu exporters' build source but never rebuild them, and a later `monitoring` run then reads the copy unchanged and skips the rebuild too; not live today |
| D | 2026-09-21 | OPEN | Frozen attach-candidate lists hold 2,457 records (742 instruments) whose exchange the D-19 heal rewrites and re-embeds; the only guard is a 24 h clock over two FIXED timestamps, comparing no content |
| D | 2026-09-21 | OPEN | D-19 falsifier fixtures span 4 of 13 venue tokens, 1 of 14 `idx`, 1 of 2 ticker shapes: the `idx` mutant crosses both bars (153 of 2,551, 12:30:11Z) at `controls_passed 11 of 11` |
| D | 2026-09-21 | OPEN | The D-19 falsifier SELECT spells the US venue vocabulary THREE times and guards ONE: 6 of 20 registry US spellings classify FALSE under Title Case (0 as stored, which cannot expose it), the direction the entry forbids (exposure 0 today) |
| D | 2026-09-21 | OPEN | 7 test/selftest harnesses under `scripts/` are run by no workflow; 2 sit inside `scripts/tests/`, which CI triggers on and sweeps with a pytest that collects only `test_*.py` |
| A | 2026-09-21 | OPEN | GIGO broken a THIRD time: the pipe-paste is cleaned at SecMaster while its SOURCE, gemini_client.py:270's prose schema, still hands the model a literal pipe enumeration; 2 live rows echo it verbatim |
| E | 2026-09-21 | OPEN | "ALL read ListingVenueRegistry" is wider than the sweep: 4 of the 6 venue-vocabulary copies are unguarded (the FILE grep counts 4 FILES; D-19's falsifier SELECT alone holds 3 of the 6 copies, 2 of them unguarded), one already wrong on 541 rows today |
| B | 2026-09-21 | OPEN | Deactivating a source mapping is a ONE-WAY DOOR: 14 dead keys, no writer revives one, and the register lookup reads the corpse as a hit and answers SUCCESS |
| B | 2026-09-21 | OPEN | `RegistrationService.cs:402`'s inactive-instrument branch is UNREACHABLE, and so is the `:443` message arm reading the same flag: the subject comes from an `IsActive`-filtered read |
| B | 2026-09-21 | OPEN | FredCollector alone both discards the SecMaster register response AND carries no failure metric; its span stays status-UNSET, so the health instrument reads a rejected registration as fine |
| A | 2026-09-17 | OPEN | D-17's clear names rows read at 12:52:35Z; pre-S2 code keeps stamping until deploy, and those stay |
| A | 2026-09-20 | OPEN | D-17 cleared the STAMP, not the DESCRIPTION: 8 foreign rows keep an EDGAR SIC line, 4 another company's |
| A | 2026-09-20 | OPEN | BTC-USD duplicates the curated CRYPTO:BTC row; 1,101 observations to 0 (T3 items 1 and 2) |
| A | 2026-09-17 | OPEN | 107 of 141 active GeminiFallback instruments are named by the query surface, not a title |
| A | 2026-09-17 | OPEN | D-18 reclassifies 82 mislabelled rows (T3 item 3); 16 unconfirmed FRED ids, WEAT, ^TNX, EURUSD remain |
| A | 2026-09-17 | OPEN | LLM breaker open -> resolve-local emits hypothesis "RAG" from SecMaster's own fallback text |
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
| D | 2026-09-20 | OPEN | SecMaster meter-capture test helpers listen PROCESS-WIDE; a slow sibling test double-counts them |
| B | 2026-09-20 | OPEN | After both EDGAR rules, per-row staleness drift survives: no gauge over edgar_filers.fetched_at |
| B | 2026-09-20 | OPEN | Four collectors mint a random Ulid EventId; TE's processed_events cannot dedupe them or be anti-joined |
| B | 2026-09-20 | OPEN | `public.events` lost its last writer in D-3 and never had a reader: 15,919 rows / 23 MB to retire |
| B | 2026-09-20 | OPEN | Fred's stream still loses ~12 rows/week to cursor inversions; no sort column on the table is commit-ordered |
| B | 2026-09-20 | OPEN | atlas-home id8 keeps `or vector(0)`: a dead fredcollector events counter would read as a healthy 0 |
| B | 2026-09-20 | OPEN | Devcontainer test runs publish `fredcollector_collection_failures_total` to PROD Prometheus; panel 16's control reads WIRED undeployed |
| B | 2026-09-20 | OPEN | `eventTypes` is advertised on ObservationEventStream and ignored by 4 of the 5 collectors |
| B | 2026-09-20 | OPEN | FredCollector's "FredCollector.DataCollection" meter is AddSource'd but never AddMeter'd: 2 instruments unexported |
| B | 2026-09-20 | OPEN | AlphaVantage streams only the LATEST row per series; 9 of its 12 rows in 7d share an instant |
| B | 2026-09-17 | OPEN | SecMaster D-16 drops Gemini fred_series answers for the 16 FRED ids D-18 leaves Equity (10 in 7d) |
| B | 2026-09-20 | OPEN | 7 active catalog rows have a blank name; 71 attach-pool slots were labelled with "-" for a name |
| B | 2026-09-20 | OPEN | selfseed_class_skip (D-34) has no alert: the 7-day production rate is unmeasured |
| A | 2026-09-21 | OPEN | The CoD prompt has no unit for a non-enum currency, so the model writes USD: 1,087 of D-35's 1,284 weekly unit rejections are rupees |
| B | 2026-09-21 | OPEN | D-35's value check still drops ~110 correct facts a week (57-200), in classes too small to have been derived yet |
| B | 2026-09-21 | OPEN | Two more CoVe symbol gates still ground on context_summary, null on 5,155 of 5,155 v2 articles: the re-extract sweep's and quarantine-hallucinated's |
| C | 2026-09-21 | OPEN | 0.66% of v2 rows carry a SubjectEntity the article never spells; no tier-1 check covers it (measured over raw HTML) |
| B | 2026-09-21 | OPEN | D-35's zero-init is registered on ApplicationStarted and nothing pins the registration; the silence alerts depend on it |
| A | 2026-09-22 | OPEN | D-35's value leg passes a SUBUNIT price stored in the main unit ("515p" as 515 GBP): 18 of 70 subunit-priced rows in 7 days |
| B | 2026-09-22 | OPEN | D-35's value-check ERROR path: no alert below the 40% reject share, one Warning per block on a systematic throw, and a 27+-digit overflow counted as an error |
| B | 2026-09-22 | OPEN | D-35's symbol check reads a raw file whose path is set but missing (an outage, never a prune) as a rejection; 0 of 51,822 set paths missing today |
| A | 2026-09-22 | AWAITING-DECISION | D-36 tier 2 KEEPS a figure about what a fund or future tracks when the verdict is tracks_underlying (~187/day) and CLEARS it when the verdict is about_other (~2.2/day on the census labels, ~2.8/day by round 4's second labeller); the policy is the user's call. 2 of 9 kept proxy verdicts were consensus-WRONG |
| A | 2026-09-22 | AWAITING-DECISION | D-36 has no §7.3 HUMAN spot-check (design: agreement >= 85% on n=50); every label on it is a model's. An 18-row sheet of the first cut's hard cases waits on the user |
| A | 2026-09-22 | AWAITING-DECISION | D-36 clears CORRECT figures when its window cuts off the article's own naming of the subject: POWW 13 of 13 clears correct (one letter, one event) |
| A | 2026-09-22 | OPEN | D-36 tier 2's RECALL is unmeasured: every clear of the census is labelled, 30 of its 10,038 kept rows were drawn at random (1 wrong, 1 ambiguous) |
| B | 2026-09-22 | OPEN | text_quote anchors on the raw's first ORDINAL occurrence: fragment-first ("5%" inside "0.45%") on 298 of 14,997 eligible rows (2.0%) |
| C | 2026-09-22 | OPEN | D-36's FigureAnchor counts a number inside a time or date ("42" in "06:42") as a sentence stating the figure: 42 of 14,592 rows (0.29%); ':' and '/' cannot simply become number-internal |
| B | 2026-09-22 | OPEN | D-36 reads a renamed company as another company (MSTR "Strategy Inc" vs "MicroStrategy Inc"; IPAX vs LUNR): 2 correct clears of 5,042 (census) |
| C | 2026-09-22 | OPEN | SentinelCoveAttachmentCheckAbsent forgets tier 2 after 7 days absent; drop its lookback leg once tier 2 has been deployed a week |
| A | 2026-09-22 | OPEN | SecMaster catalog names that identify nothing -- 80 series codes named by their own code, 41 places (DX "Japan", KC "Colombia"), 7 blank -- leave ~16 eligible rows a day unjudged by D-36; loose names (NAQ.DEX "Nasdaq", 18/day) still pass wrong figures |
| A | 2026-09-22 | OPEN | Resolver symbol collisions D-36 clears daily: crude on Colgate (CL, 286 of 286 clears wrong), Japan on AT&T (T), Saudi Arabia on Spire (SR), Indian banks on BSE, Porsche on Deere (DE); 1-2 letter symbols are wrong on 54.8% of judged rows. Tier 2 can only strip them; 4,926 labelled wrong clears are a resolver-regression gold set once each names its right instrument |
| B | 2026-09-22 | OPEN | Attachments ReExtract writes after extraction get no D-36 check: ~57 a day, ~6.6 of them wrong, reach the digest, dedup and the auto-approver |
| B | 2026-09-22 | OPEN | D-36 tier 2 reads an ADR and its home listing as different securities: the ONE consensus false clear of 86, one split clear, and AMKBY/Maersk A (round 3) |
| B | 2026-09-22 | OPEN | D-36's two share alerts take thresholds from a 40-row-per-window replay; re-derive from 7d of live counters |
| C | 2026-09-22 | OPEN | D-36 cannot see 236 of 22,650 weekly instrument-bearing v2 rows (no value), nor SecMaster dedup's subject_entity match on a cleared row |
| C | 2026-09-22 | OPEN | A post-model TRANSIENT failure re-runs the GPU extraction up to MaxRetries times, then parks the article: the retry branch never asks D-27's spend ledger (20 extra extractions, 4 parked on 2026-09-22) |
| C | 2026-09-22 | OPEN | A stalled Gemini resolver now holds an extraction worker 30s per eligible observation: p90 article 10 calls (5 min), max >= 100; no per-article budget |
| B | 2026-09-22 | OPEN | The OCE filter that failed the Gemini leg still sits on 4 fail-soft sites unreachable in the deployed configuration, each one config value from live |
| C | 2026-09-22 | OPEN | A SecMaster promote timeout pins the ResolutionWorker to its oldest row (attempts cap bypassed), a timed-out v2 row is an untagged NoResolution, and a hang can hold one v2 observation for 4 + N x 180s (code-derived); none live (0 calls >= 60s in 7d) |
| B | 2026-09-17 | AWAITING-DECISION | Test databases on the shared timescaledb: 9 fixed-name orphans, per-worktree leaks on kill, each holds a TimescaleDB worker slot |
| B | 2026-09-17 | OPEN | SecMaster EmbeddingCache keys on lower-cased text: "NASDAQ" can search with "Nasdaq"'s vector |
| B | 2026-09-17 | OPEN | backfill_unresolved_rate_high never detected a fault: constant on main, crossed by growth on D-17 |
| B | 2026-09-17 | OPEN | The D-10 name repair still rewrites the whole metadata column from a load older than its API calls |
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
| C | 2026-09-17 | OPEN | Finnhub catalog enrichment re-enriches and re-embeds ~240 rows/hour; its cooldowns never persist |
| C | 2026-09-16 | OPEN | gemini-resolver saturates its 1500/day cap; intent says dozens/day (INTENT_FIDELITY) |
| C | 2026-09-16 | AWAITING-DECISION | Quarantined-ticker re-acquisition is an undecided policy: Gemini cost + un-alerted 23505 |
| D | 2026-09-20 | OPEN | FredCollector compile.sh dies in a fresh worktree: its compose env_file names a gitignored .env (Ofr's is tracked) |
| D | 2026-09-20 | OPEN | 5 of the 12 D-4 composite-cursor sites are unpinned; the whole `Between` path is one of them |
| D | 2026-09-20 | OPEN | Integration suites never reach the push marker and nothing re-runs them between PRs (all six now green) |
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
| E | 2026-09-17 | OPEN | gemini-resolver-mcp cache.py docstring cites SecMaster lines D-16 moved (:173, :166) |
| E | 2026-09-16 | OPEN | Three live sites still teach the retired MODEL_SIZE >= 30B floor, one in production code |
| E | 2026-09-16 | OPEN | docs/BACKLOG.md has no out-flow that runs: 3,849 -> 4,187 -> 6,470 lines in eleven days |
| E | 2026-09-16 | OPEN | Metric prefix inconsistency: sentinel_candidate_surface_* vs sentinelcollector_semantic_* |
| E | 2026-09-16 | OPEN | Two SecMaster comments still call a Finnhub 403 transient (permanent, arrives as NULL) |
| E | 2026-09-16 | OPEN | SentinelCollector card is 5.3x over its D-entry gate and its line count hides it |
| E | 2026-09-16 | AWAITING-DECISION | Six deployed directories have no card and sit outside the SERVICES roster (HARD_STOP gap) |

**Four bare `nerdctl inspect` sites outside the freshness gate each get whichever object resolves first.** On
nerdctl 1.7.7 a bare `inspect <name>` returns the IMAGE when an image of that name exists, and the container only
otherwise. MEASURED 2026-09-22 over the 31 compose services: 23 resolve to the image, 8 to the container. Found by
`grep -rnI inspect deployment scripts .claude`, then each hit read. The freshness gate was the fifth site and is
fixed. Not fixed here, because none is on the gate's path:
  - `deployment/ansible/playbooks/deploy.yml` task "Remove legacy standalone vllm-server container": `nerdctl
    inspect vllm-server` means the CONTAINER, and resolves to it only because no image is named `vllm-server`
    (the image is `vllm/vllm-openai@sha256:…`). If one ever were, the image would carry no compose label, and
    the else branch runs `nerdctl stop vllm-server && nerdctl rm vllm-server` on every unscoped deploy. One-word
    fix: `nerdctl container inspect`.
  - `deployment/artifacts/compose.yaml.j2` (the `SecMaster__TimeoutSeconds` comment) says the value is "visible
    in `nerdctl inspect sentinel-collector`". That resolves to the IMAGE, which lacks it, and so does
    `nerdctl container inspect`, whose Config holds only AttachStdin, Hostname and Labels. Only `nerdctl
    container inspect --mode=native` `.Spec.process.env` shows `SecMaster__TimeoutSeconds=180`. The comment
    renders into `/opt/ai-inference/compose.yaml`, so correcting it changes the rendered file. Land it with a
    deploy that restarts the stack anyway.
  - `deployment/artifacts/monitoring/alerts/sentinel.yml` (`autofix_hint` of the candidate-surface-filter alert)
    says to confirm "the deployed sentinel-collector image is the intended build (nerdctl inspect .Created)".
    Bare, that is the `:latest` image's BUILD time, and it says nothing about what the running container runs
    (CLAUDE.md VERIFY_TRAP). The answer is `deployment/ansible/scripts/freshness-gate.sh`.
  - `deployment/artifacts/scripts/autofix.sh` allows `Bash(sudo nerdctl inspect:*)`, pinned by
    `deployment/tests/autofix/run.sh`. That prefix admits the bare form and `--type container|image`, and NOT
    `nerdctl container inspect` or `nerdctl image inspect`, the spellings CLAUDE.md prescribes.
  - No action: `deployment/artifacts/vllm-rollback.md` quotes a 2026-06-11 `sudo nerdctl inspect vllm-server`,
    which resolved the container then and does now. `.claude/skills/supervisor-mode/templates/deploy.md` already
    uses `--type image` and warns against the bare form.
RE-CHECK: re-run the grep. For each name, `sudo nerdctl inspect <name> | jq '.[0] | has("RepoTags")'` is `true`
exactly when it resolves to an image.

**`--tags alerting` and `--tags dashboards` copy the exporters' build source but never rebuild them, and a later
`monitoring` run then skips the rebuild too.** In `deployment/ansible/playbooks/deploy.yml`, "Deploy monitoring
directory" carries `monitoring`, `dashboards`, `alerting` and `otel`; "Rebuild OTEL exporters built from the monitoring
directory" and "Recreate OTEL exporters onto the rebuilt images" carry only `monitoring` and `otel`, and both are gated
on `monitoring_dir.changed`: whether THAT run's copy changed a file. ups-exporter and gpu-exporter build from
`{{ deployment_base }}/monitoring/{ups,gpu}-exporter` (`deployment/artifacts/compose.otel.yaml.j2`). So an exporter
source edit shipped by an `alerting` or `dashboards` run lands on disk unbuilt, and the next `monitoring` or `otel` run
copies nothing new, reads `changed: false` and skips both tasks: the exporters serve the old image until some other
file under monitoring/ changes in a run that selects the rebuild. MEASURED 2026-09-22 with `--list-tasks` from
`deployment/ansible/`: `--tags alerting --skip-tags always` and `--tags dashboards --skip-tags always` each select the
copy and neither exporter task; `--tags monitoring --skip-tags always` selects all three. NOT LIVE: `sudo nerdctl image
inspect` dates `otel-ups-exporter:latest` and `otel-gpu-exporter:latest` 2026-09-04T22:34Z, later than the newest
exporter source file on the host (`ups-exporter.py`, 2026-07-31T20:10Z) and the last commit touching either source
(b3972b84). The copy task's own comment records the `otel` half of this tag-group mismatch biting once. Fix
direction: gate both tasks on a comparison every run can make, as the prometheus.yml and alertmanager.yml checksums
beside them are, not on `.changed`.
RE-CHECK, from `deployment/ansible/` (prints 0 while this is open):
`ansible-playbook playbooks/deploy.yml --tags alerting --skip-tags always --list-tasks | grep -c 'Rebuild OTEL exporters'`
Then compare the two images' `.Created` with the newest mtime under `/opt/ai-inference/monitoring/{ups,gpu}-exporter/`.

**The parallel-compile gap check decides "starts a container" on `nerdctl` alone, so a docker-first verification
script would read `container-less` and be exempted rather than audited.** The predicate is
`devcontainer_gap_verdict` in `scripts/test-devcontainer-owner.sh` (test 5), matching
`grep -qE 'nerdctl|devcontainer_compose'`; the documented pipeline in
`.claude/skills/supervisor-mode/references/parallel-dispatch.md` uses the same two tokens. That contradicts
CLAUDE.md PROJECT_CONVENTIONS, which mandates runtime-agnostic tooling (`nerdctl|docker|podman`), and the fallback
is already written here: `edge/sentinel-edge/.devcontainer/build.sh` runs `nerdctl compose build` when nerdctl is
on PATH and `docker compose build` otherwise. Measured 2026-09-20, two populations, which an earlier wording
conflated: **0 of the 15** files the selector enumerates mention `docker`
(`git ls-files | grep -E '\.devcontainer/(compile|typecheck|dev)\.sh$' | tr '\n' '\0' | xargs -0r grep -lE docker | wc -l`),
and **1 of the 29** tracked `.devcontainer/*.sh` files does
(`git ls-files | grep -E '\.devcontainer/.*\.sh$' | tr '\n' '\0' | xargs -0r grep -lE docker`) — that one being
`edge/sentinel-edge/.devcontainer/build.sh`, outside the selector. So the exposure is latent, not live: 0 audited
scripts are docker-only today. Direction is UNSAFE: such a script would silently be treated as having nothing to
collide over, and known-bad control B cannot see it, because B proves the exemption keys on the container tokens
without proving those tokens are the complete set.

The half that USED to hide the first migration is now closed, and is recorded here because the entry omitted it:
the `audited >= 12` floor carried slack (14 audited over a floor of 12), so the first audited script to stop
matching the container predicate would have been exempted, dropped to 13, and still passed. The floor is gone —
test 5 now asserts the population the loop WALKED against `verify_script_roster` by name, and the exemptions it
TOOK against `container_less_exempt` for equality, so a script that stops driving containers is named rather than
absorbed. Re-check: rerun both counts above and
`bash scripts/test-devcontainer-owner.sh` (expect rc 0, `passed=137 failed=0`, and "walked exactly the 15
rostered verification scripts"). The entry closes when
the predicate matches `docker` and `podman` too and a control plants a docker-only gap. Fix is one regex in
`devcontainer_gap_verdict` plus the documented pipeline, but both copies must move together or they disagree.

**The parallel-compile ownership audit is STATIC TEXT matching, so an ownership call that is present but
UNREACHABLE reads as `owned` and the script passes.** `devcontainer_gap_verdict` in
`scripts/test-devcontainer-owner.sh` decides by `grep -qE '^[^#]*devcontainer_own'`; it never runs the script, so
it cannot distinguish a call site from a dead line. Every known-bad control is itself a text mutant cut from a real
script, which is exactly why no control can reach this. Measured 2026-09-20: copy
`AlertService/.devcontainer/compile.sh`, insert `exit 0` on the line immediately before its `devcontainer_own`
call, stage it in a throwaway repo and run the real `verify_audit_pipeline` over that repo — it prints
`PASS zeta/.devcontainer/compile.sh owns before touching containers` and `audited=1 exempted=<nothing>`.

ORDER is lexical for the same reason, and that half survives the fixtures added 2026-09-20 (`kappa`, `lambda`)
because they too are text mutants. Measured the same day: a 9-line compile.sh defining `take_ownership() {
devcontainer_own … }` at line 6, starting a container at line 8 and CALLING `take_ownership` at line 9 is reported
`PASS mu/.devcontainer/compile.sh owns before touching containers` — the audit compares the line the call is
WRITTEN on to the line the container starts on, never the order they RUN in. Same instrument closes both halves,
so the re-check below exercises BOTH; a re-check naming only the `exit 0` copy would close the entry with the
lexical-order half still open. Impact is
C, not A: it needs someone to write an early `exit`/dead branch above the ownership call, and **0 of the 15**
selector files carry an unconditional `exit` above their `devcontainer_own` line today (compare the first
`^[[:space:]]*exit ` line number to the first `^[^#]*devcontainer_own ` line number in each). Closing it means
EXECUTING each script with `devcontainer_own` stubbed to record, under a
harness that stops it before it touches a container — a different instrument from the text audit, not a bigger
regex. Re-check, BOTH copies, through the real `verify_audit_pipeline`: plant the `exit 0` copy AND the late-call
copy into the control tree in `scripts/test-devcontainer-owner.sh` (the `plant` calls in test 5) and read what the
pipeline says about each. Re-measured 2026-09-20 at `passed=137`, still open in both directions — `exit 0` above
the ownership call yields `OK nu/.devcontainer/compile.sh owns before touching containers`, and a 9-line script
defining the wrapper on line 6, starting a container on line 8 and calling the wrapper on line 9 yields
`OK mu/.devcontainer/compile.sh owns before touching containers`. The entry closes when BOTH are reported as
gaps; closing only the `exit 0` half leaves the lexical-order half live and is not a close.

**Test 6 pins its three populations with FLOORS — `${#compose_files[@]} >= 14`, `named >= 13`,
`owned_nuget >= 10` — which is the `audited >= 12` shape test 5 retired in #1079, one section out.** All three are
TIGHT today (14/14, 13/13, 10/10 — slack 0), which is why they still catch a narrowing: measured 2026-09-20,
appending `| grep -v ThresholdEngine` to the compose discovery pipeline gives rc 1, `passed=134 failed=3`, naming
all three. The defect is what happens on the next legitimate addition. Modelling slack 1 by holding that same
one-file narrowing and setting the floors to 13/12/9, the suite returns **rc 0, `passed=137 failed=0`** and
ThresholdEngine's compose file drops out of the port, name and nuget checks silently — the PASS lines read 13, 12
and 9 and nothing compares them to anything. A count with slack is not an assertion. Impact C: it needs a compose
file to be added without the floors being bumped, which is an ordinary service addition and has happened before
(#1079's floor carried 14 audited over a floor of 12 and absorbed a real path-scoped exclusion at rc 0 / 128).
Fix is the shape already in test 5 — a NAME roster asserted for equality against what the loop walked, like
`verify_script_roster` and `container_less_exempt` — not a bigger floor. Re-check: run both mutations above; the
entry closes when narrowing the glob by one file fails regardless of what the floors say.

**`scripts/test-devcontainer-owner.sh`'s double-trap sweep enumerates with `for f in $(git ls-files | grep …)`, so
a path containing a space is split and one containing a glob metacharacter is expanded — either way that file is
never checked and the sweep still reports clean.** Same defect class as the documented pipeline in
`.claude/skills/supervisor-mode/references/parallel-dispatch.md`, which was fixed 2026-09-20; this copy was left
UNCHANGED on purpose, as scoped residue. Pre-existing since `8235457f` (#920). Latent, not live, measured
2026-09-20: **0 of the 29** tracked `.devcontainer/*.sh` paths contain a space or a glob metacharacter
(`git ls-files | grep -E '\.devcontainer/.*\.sh$' | grep -cE '[ *?[]'`). Impact D because the sweep it weakens
only catches a doubly-registered teardown trap, and `git ls-files` output is the one input that would have to
change. Re-check: rerun that count; the entry closes when the loop reads `while read -r f; do … done < <(…)` and
a spaced fixture proves the difference.

**Test 5's name roster proves the loop was HANDED each script, never that it CHECKED one.**
`scripts/test-devcontainer-owner.sh:940` appends to `processed` as the FIRST statement of
`audit_verify_script`, deliberately — "recorded before any branch can return" — so that a path-scoped `continue`
in the enumerating loop cannot hide a file. The cost of recording that early is that everything AFTER the append
is unpinned by the roster: a `return` on the next line still yields `the audit walked exactly the 15 rostered
verification scripts`, while that script's ownership, /workspace and marker checks never run, and the suite stays
rc 0 (recorded at `passed=134` by the #1081 review that found it). A reader takes the PASS as proof that fifteen
scripts were audited, dispatches parallel compiles, and one of them was never checked. NOT the same defect as the
static-text entry above: that one is about a call the audit cannot EXECUTE, this one is about a script the audit
never REACHES the checks for. Re-check, two commands, the first static and the second the reproduction:
`sed -n '938,946p' scripts/test-devcontainer-owner.sh` shows the append above `verdict=$(...)`; then insert
`return 0` immediately after line 940, run `bash scripts/test-devcontainer-owner.sh`, and read the roster line —
it must still claim fifteen walked. The entry closes when a script that reached no verdict cannot be counted as
walked — a COMPLETED marker appended after the checks, asserted alongside the handover one, not a bigger roster.

**Three smaller residues in the same suite, all measured 2026-09-20 and all recorded rather than fixed**, because
each wants the same instrument decision as the entry above and none is a regression. Named separately so a fix to
one cannot be read as covering the others.
1. *The interactive-port exemption keys a BASENAME, not the declared path.* CLAUDE.md declares exactly two
   host-port exceptions, `edge/sentinel-edge/.devcontainer/compose.ports.yaml` (8787) and
   `WhisperService/.devcontainer/compose.dev.yaml` (8090), and the suite plants both at those paths
   (`:1369-1370`). The guards that spend the exemption are globs on the name alone:
   `[[ "$c" != *compose.ports.yaml && "$c" != WhisperService/* ]]` at `:1281` and `:1290`. Any compose file
   called `compose.ports.yaml`, anywhere in the tree, therefore publishes any host port with no complaint, and
   the exemption widens without an edit. Re-check: `grep -c '\*compose\.ports\.yaml' scripts/test-devcontainer-owner.sh`
   -> 2 on 2026-09-20, both basename globs (`:1281`, `:1290`); it closes when both compare against the declared path.
2. *The per-owner nuget check is a FILE-WIDE grep on both sides.* `:1300` asks `grep -q 'nuget' "$root/$c"` and
   `:1301` asks `grep -qE 'name: \$\{ATLAS_DEV_NUGET_VOLUME:-' "$root/$c"`. Neither is scoped to a volume block,
   so a file carrying ONE owned volume and ONE hardcoded one satisfies the second grep and is counted
   `owned_nuget`, which is the opposite of what the check exists to say. Re-check: `sed -n '1298,1306p'
   scripts/test-devcontainer-owner.sh`; it closes when the two greps read the same volume entry rather than the
   same file.
3. *Known-bad control B's audited population is a DECLARED literal.* `:1180` asserts
   `"$ctl_out" == *"AUDITED 10"*`. Ten is a number a human keeps in step, so dropping a plant together with its
   required line, its expectation and that literal leaves the control green over a smaller tree — the
   `ctl_covered_elsewhere` disease the comment at `:1150` says it replaced, surviving one level out. Re-check:
   `grep -n 'AUDITED 10' scripts/test-devcontainer-owner.sh` -> 2 lines on 2026-09-20, the assertion at `:1180` and
   its failure message at `:1182`, so the literal must be hand-edited in two places; it closes when the expected
   count is DERIVED from the names `plant()` was called with, the way `ctl_planted_names` already is at `:1168`.

**D-19's `Down` COMPARE-AND-SWAP IS CODE-GRANULAR, NOT WRITE-GRANULAR, AND THE CARD CLAIMS MORE THAN THE
PREDICATE DELIVERS.** Measured 2026-09-21. `RestoreSpellingsSql` swaps on the CANONICAL CODE recomputed from the
pre-image, so it refuses to restore only when the row's exchange no longer equals that code. 20 of the migration's 42
map entries resolve to `US`, and those 20 spellings cover 4,273 of the 4,580 mapped rows -- 93.3% of the population.
So for 93.3% of rows a later LEGITIMATE write of a *different* US spelling (`NYSE` over a row healed from
`NASDAQ NMS - GLOBAL MARKET`) is canonicalized by the persist boundary back to `US`, the CAS sees no change, `Down`
restores the OLD pre-image over it, and `SkippedRestoresSql` does not list the row -- the operator is told the
rollback was complete. The 7 non-US many-to-one codes (US aside, `TT`, `HK`, `CT`, `LN`, `FP`, `NA`, `GR` ... 23
distinct codes for 42 entries) carry the same defect over the remaining 6.7%. The CAS still does what it was added
for: it refuses a row rewritten to a DIFFERENT venue, which is the case that loses data. Closing it needs the
pre-image compared against the resulting value AND the write instant -- e.g. stamping the heal instant beside the
pre-image and refusing any row whose `updated_at` has moved since. Re-check:
`grep -c "'US')" SecMaster/src/Data/Migrations/20260921084921_CanonicalizeExchangeVocabulary.cs` -> 20 of 42 pairs on
2026-09-21, and the 93.3% is `count(*) FILTER (WHERE upper(btrim(exchange)) = ANY(<20 US spellings>))` over
`... ANY(<all 42>)`. Closes when `Down` cannot silently overwrite a post-`Up` write, or when the entry says plainly
that it can.

**GIGO BROKEN A THIRD TIME: D-19 CLEANS THE PIPE-PASTE AT THE DESTINATION WHILE ITS SOURCE IS UNFIXED AND, UNTIL NOW,
UNFILED.** `CLAUDE.md` GIGO is a HARD_STOP -- "clean at the SOURCE where garbage is BORN; never gate each
destination" -- and it names two prior breaks, both destination gates (#818 FRED series-search, #823 paid resolver).
This is the third, and it is in the PR that quotes the rule. Measured 2026-09-21.
`gemini-resolver-mcp/gemini_resolver/gemini_client.py:270` hands the model a PROSE schema whose exchange value is a
literal pipe enumeration -- `"exchange": "NYSE | NASDAQ | AMEX | OTC | FRED | null"` -- so `|` is simultaneously the
type-alternation separator and a character of the example value, and the model cannot tell "choose one of these" from
"the value is this string". Two live `atlas_secmaster` rows hold the echo, both `discovery_source =
entity_resolution:gemini`: `OWLT` (Owlet Inc) stores `NYSE | NASDAQ | AMEX | OTC | FRED | null`, byte-identical to
that line, and `SCHV` stores the truncated `NYSE | NASDAQ`. D-19 nulls both at the persist boundary, which is a
destination gate -- the next such row is born the same way. The same prose schema does the same thing to
`asset_class` and `instrument_type` two and three lines down, exposing D-5's column too; measured today, 0 rows carry
a pipe in either, so only `exchange` has actually been hit. THE SOURCE FIX IS NOT A PROMPT TWEAK: the resolver should
constrain the producer (`response_format` / a JSON schema), which is `CLAUDE.md` DATA_ML_CONTEXT's rule and
[[feedback_constrain_llm_output_not_salvage_parse]]. Re-check:
`SELECT count(*) FROM instruments WHERE exchange LIKE '%|%' OR asset_class LIKE '%|%' OR instrument_type LIKE '%|%'`
-> 2 on 2026-09-21; and `grep -n '|' gemini-resolver-mcp/gemini_resolver/gemini_client.py` at the schema block.
Closes when the resolver constrains its output and the prose enumeration is gone.

**D-19's "ALL READ `ListingVenueRegistry`" IS WIDER THAN THE SWEEP BACKING IT: FOUR UNGUARDED COPIES OF THE VENUE
VOCABULARY SURVIVE, OF SIX.** A pointer gap, not a live regression -- all four were measured, and none breaks *because of*
D-19. Measured 2026-09-21. The entry's claim is true of the five consumers it enumerates (the persisted value, the
OpenFIGI suffix lookup, the country map, the embedding prose, D-17's non-US set) and false as a statement about the
repository.
- `SentinelCollector/scripts/reresolve-by-provenance.sh:116-117` carries its own US-exchange classifier as two regex
  literals -- `exchange ~* '^(US|FRED|ICE|COMEX|CBOT|NYMEX|CRYPTO)$'` OR
  `exchange ~* '(NYSE|NASDAQ|OTC|AMEX|ARCA|BATS|CBOE)' AND exchange !~* '(EURONEXT|OMX)'` -- and no test compares it
  to the registry. **AND IT IS ALREADY WRONG TODAY, INDEPENDENTLY OF D-19**: neither arm matches
  `NEW YORK STOCK EXCHANGE, INC.` (the string contains no `NYSE`), so of the 8,411 active bare rows the script's A10
  check reads, 541 real NYSE listings are classified NON-US right now. D-19's heal turns those into `US`, which the
  first arm does match, so the heal SILENTLY REPAIRS the script -- and it LOOSENS the gate, it does not tighten it:
  the suspect set falls 1,070 -> 529 and `suspect_landings` stops refusing `--live` over those rows. Re-derived
  2026-09-21 by simulating the post-heal column SELECT-only, both sides scored by the script's own predicate: 541
  classifications change, ALL suspect -> clean, and ZERO change the other way. NOT 543 -- an earlier revision of this
  line added the 2 pipe rows, which leave the arm's population when they go NULL but were never suspect (their text
  matches `NASDAQ`), so they change no classification. Nothing else changes.
  (This CORRECTS a relayed claim that the copy was measured not to break: it does not break, but only because it was
  already broken in the safe direction.)
  RE-DERIVED IN ONE SELECT 2026-09-21T13:49:31Z on `atlas_secmaster`, through the script's OWN predicates rather
  than flagged: of the A10 bare-row population (`is_active AND exchange IS NOT NULL AND symbol NOT LIKE '%.%'`,
  8,419 rows today, against the 8,411 read earlier the same day) 541 carry `NEW YORK STOCK EXCHANGE, INC.`, and
  all 541 classify NON-US under `NOT (exchange ~* '^(US|FRED|ICE|COMEX|CBOT|NYMEX|CRYPTO)$' OR (exchange ~*
  '(NYSE|NASDAQ|OTC|AMEX|ARCA|BATS|CBOE)' AND exchange !~* '(EURONEXT|OMX)'))`. 556 active rows carry the spelling
  in all, so the 15-row gap is dotted symbols the A10 population excludes. The heal has not run.
- `SecMaster/src/Data/Migrations/20260917110542_ClearOutOfScopeUsAuthorityStamps.cs` `UnclearedRowsSql` freezes 26
  venue literals inside an operator SELECT. Measured on the simulated post-heal column it returns 0 rows before and 0
  rows after, so it is inert today; it is a frozen artifact and correctly so, but it is a second copy nothing points
  at the registry from.
- TWO MORE COPIES EXIST AND ARE GUARDED, so they are not part of this entry -- they are here because the count
  above is "four UNGUARDED", never "four in the repository", and a reader checking it should find all six.
  `CanonicalizeExchangeVocabulary.SpellingValues` is checked pair-by-pair against
  `ListingVenueRegistry.TryResolveCode` by `the_spelling_map_agrees_with_the_registry`. The OTHER guarded one,
  added 2026-09-21, is the US venue-token alternation inside `SecMaster/DECISIONS.md` §D-19's falsifier SELECT --
  predicate (2)'s, and the only guarded one of the THREE copies that SELECT holds -- a
  REDUCTION of the registry's US spellings to their words -- checked by
  `scripts/tests/test_d19_falsifier_vocabulary.py`, which parses both sides and goes RED when a registry US
  spelling shares no token with the alternation or the alternation names a token no spelling contains
  (`ListingVenueRegistry.cs` is in the python-tests workflow's path list for that reason). RED proved both ways
  2026-09-21 by adding `PINK SHEETS` to the registry and `IEX` to the alternation, one at a time. What that test
  CANNOT see is in its own docstring -- word-boundary placement, an alternation that is too WIDE (the `ARCA`
  case), and the other four predicates. That is the shape the two copies above lack.
- THE COUNT WAS FILE-GRANULAR AND UNDERCOUNTED BY TWO. Corrected 2026-09-21: the grep below counts FILES, and
  `SecMaster/DECISIONS.md` holds THREE separate spellings of the US venue vocabulary inside the ONE falsifier
  SELECT -- predicate (2)'s `\m(...)\M` alternation plus its `NEW YORK STOCK EXCHANGE` arm (guarded), predicate
  (4)'s 11-token `head_is_venue` list, and predicate (5)'s 13-token residual-strip list. So the repository holds
  SIX copies, not four, and FOUR are unguarded. What the two new ones cost, and the exposure that makes them debt
  rather than a live defect, is its own entry ("spells the US venue vocabulary THREE times and guards ONE").
THE RE-CHECK BELOW IS A CLOSED FOUR-FILE GREP: it can re-confirm the six and CANNOT discover a seventh, which is
the same defect class this branch filed twice elsewhere -- stated because a reader must not read it as a sweep.
Swept by hand 2026-09-21 for others: the one further hand-kept exchange alternation is
`SentinelCollector/src/Extraction/TickerExtractor.cs:23` (`NYSEArca|NASDAQ|NYSE|AMEX|OTC|TSXV|TSX|LSE`), which
reads no registry (0 references to `ListingVenueRegistry` anywhere in `SentinelCollector/`). It is deliberately
OUTSIDE this set: it spells exchange PREFIXES in parenthesised ARTICLE notation (`(NYSE:DT)`, `(NYSEArca:SPY)`),
not the `instruments.exchange` COLUMN vocabulary -- it carries non-US venues the column table never maps here
(`TSX`, `TSXV`, `LSE`) and omits the column's own spellings (`NEW YORK STOCK EXCHANGE, INC.`, `OTCQX`, `BATS`,
`CBOE`), so a registry change should NOT move it. The boundary was considered, not missed.
Re-check, over ALL SIX copies so the count of UNGUARDED ones is derived rather than recalled:
`grep -rln "NYSE\|NASDAQ" SentinelCollector/scripts/reresolve-by-provenance.sh
SecMaster/src/Data/Migrations/20260917110542_*.cs SecMaster/src/Data/Migrations/20260921084921_*.cs
SecMaster/DECISIONS.md` -> 4 FILES on 2026-09-21, of which the last two carry a registry-derived test
(`the_spelling_map_agrees_with_the_registry`, `scripts/tests/test_d19_falsifier_vocabulary.py`) and the first
two carry none; then count the venue-vocabulary EXPRESSIONS inside the D-19 falsifier fence, which the file grep
cannot see --
`python3 -c "import re;F=re.compile(r'^\`\`\`[^\n]*\n(.*?)^\`\`\`',re.S|re.M);b=[x for x in F.findall(open('SecMaster/DECISIONS.md').read().split('## D-19')[1]) if x.lstrip().upper().startswith('WITH')][0];print(len(re.findall('(?i)new york stock exchange',b)))"`
-- `findall` counts OCCURRENCES, which is the claim's unit; a line count agrees today only because the three sit on
three separate lines -> 3 occurrences on 2026-09-21 (predicate (2)'s spelled-out arm, predicate (4)'s `head_is_venue` list, predicate
(5)'s strip list), of which only the alternation is parsed by a test. CLOSES WHEN each of those four DERIVES from
`ListingVenueRegistry`, or carries a test that fails when the registry moves AND is not a substring-containment
test. The qualifier is load-bearing, not pedantry: applying the repo's only such gate
(`test_d19_falsifier_vocabulary.py`'s shape) to the two in-query lists was run on 2026-09-21 and returns
`missed=[]` on BOTH -- fully green on `head_is_venue`, and on the strip list green on that leg while going red on
the OTHER leg for `nyse ?arca` / `nyse ?american`, which are regex fragments rather than spellings and have
nothing to do with the blindness. A maintainer would rewrite those two tokens and ship a green gate over an
unfixed defect. AMBIGUITY DENIES: if it is unclear whether a proposed test would fire, it does not close this.

**THE D-19 FALSIFIER'S 11 IN-QUERY FIXTURES CONTROL THE PIPELINE, NOT THE PREDICATES: A ONE-TOKEN-SET EDIT MOVES
THE REAL COUNT PAST BOTH BARS WHILE EVERY CONTROL STAYS GREEN.** The falsifier SELECT in `SecMaster/DECISIONS.md`
§D-19 UNIONs 11 `(surface, want)` fixtures into its own `input` CTE and reports `controls_passed 11 of 11`, which
the entry reads as the licence to believe the number. It is a control over the PIPELINE -- that the corpus and the
fixtures meet the SAME predicate rather than two copies -- and it is not coverage of the five predicates.
PER-AXIS COVERAGE, derived 2026-09-21T12:30:44Z by scoring each alternation member against the 11 fixture surfaces
through Postgres' own regex engine, never by reading the SQL: predicate (2)'s venue alternation 4 of 13 members
(`NASDAQ`, `NYSE`, `OTC`, `CBOE`; the other nine, `NYSEARCA` `NYSEAMERICAN` `AMEX` `BATS` `OTCPK` `OTCQX` `OTCQB`
`OTCMKTS` and the `NEW YORK STOCK EXCHANGE` arm, appear in no fixture); predicate (4)'s `idx` 1 of 14 tokens
(`composite`); predicate (3)'s ticker 1 of 2 shapes (the parenthesised `(NASDAQ:QLYS)`; no fixture carries a BARE
`EXCH:TICKER`, which is the `Cboe:CBOE` shape the entry's prose says predicate (3) exists to catch);
`head_is_venue` 2 of 11 tokens; the residual-strip list 4 of 13 tokens. THAT COUNT IS A CEILING ON STRADDLING, NEVER A MEASURE OF IT -- straddled implies matched, so the matched count can only OVERSTATE how many members are straddled. Read as a floor it would license the cheat below, where 13 of 13 are matched and ZERO are straddled. The closure below is the measure.
TWO MUTANTS, BUILT AND RUN, not reasoned about. Baseline, read 2026-09-21T12:30:11Z on `atlas_data` from the
committed fence extracted verbatim: population 432,636 rows with a non-blank `source_entity`, `venue_token` 2,551,
`falsifier` 14, `controls_passed 11 of 11`.
- Predicate (3) narrowed to require a parenthesis before the `EXCH:TICKER` (`e ~ '\([A-Za-z]...`): `falsifier` 26
  (1.02% of 2,551), `controls_passed 11 of 11`. The count nearly doubles and no control notices.
- Predicate (4)'s `idx` reduced to `composite` alone: `falsifier` 153 (5.998% of 2,551), `controls_passed
  11 of 11`; re-read 158 of 2,560 at 13:01:10Z and again at 13:26:48Z, so the COUNT drifts with the corpus while
  the VERDICT does not. THAT CROSSES BOTH BARS -- at least 100 rows AND more than 5% of the surfaces naming a US venue -- so
  the entry reads FALSIFIED, the accepted display-name merge re-opens, and the trigger was an edit to a token list
  rather than anything production emitted. The bar was chosen to be unreachable by the 14 rows the rule already
  miscounts; it is reachable by a diff nobody would call a behaviour change.
CLOSURE IS A PROPERTY, NOT A FIXTURE COUNT: every axis a predicate reads carries a fixture PAIR that STRADDLES it --
for each alternation member, one surface that the member makes `want true` and one that it makes `want false` -- so
that deleting or narrowing that member turns a CONTROL red before it moves `venue_token` or `falsifier`. A larger `VALUES` list
that still leaves nine venue tokens and thirteen `idx` tokens unstraddled closes nothing.
Re-check (SELECT-only, ~5s), and re-derive the denominator with the numerator, because the observation table does
not grow at a RATE -- it grows in BURSTS. Derived 2026-09-21T13:00:39Z over the 24 h to that instant: 7,353 rows
arrived in 65 of the 96 complete 15-minute buckets BY `extracted_at`, median 84 rows per non-empty bucket, min 1,
max 422 rows; the last 6 h alone mean 200 rows per bucket. Separately, and on a DIFFERENT clock -- the falsifier's
own `population` count, which is table arrival rather than `extracted_at` -- this entry's first and last reads
(12:29:41Z and 12:42:28Z) are 316 rows apart, i.e. 371 rows per 15 minutes. The two are not the same measurement
and are not summed. Any single figure carried forward is wrong within the hour, in either direction.
Extract the single fenced `WITH`-block from §D-19 and run it; then run it twice more
with `e ~ '[A-Za-z][A-Za-z.]{1,9}` changed to `e ~ '\([A-Za-z][A-Za-z.]{1,9}` and, separately, with the `idx`
alternation reduced to `'\m(composite)\M'`.
CLOSES ON AXIS COVERAGE, NOT ON THOSE TWO MUTANTS, and the difference is measured rather than argued: adding just
`('NASDAQ:AAPL', false)` and `('Nasdaq 100 index', false)` to the `VALUES` list turns BOTH named mutants red
(`controls_passed 12 of 13` with a `CONTROL FAILED` row each, `falsifier` unmoved at 14, read
2026-09-21T13:23:36Z) while coverage moves only to venue 4 of 13, `idx` 3 of 14, `head_is_venue` 2 of 11, strip
4 of 13 -- four of the five axes still unstraddled, only ticker complete at 2 of 2 (13:23:59Z). A criterion those
two fixtures satisfy is a criterion the next `idx` or venue edit walks straight through.
AND FULL MEMBERSHIP IS NOT THE EXIT EITHER -- measured, because an earlier revision of this entry named it and it
is cheatable by the shape the query ALREADY SHIPS. NINE `('<VENUE>', false)` fixtures -- the same shape as the
committed `('Nasdaq', false)`, exactly one per UNCOVERED venue token -- drive the per-axis count to venue 13 of 13,
`head_is_venue` 11 of 11 and strip 13 of 13, FULL MEMBERSHIP on three of the five axes, at `controls_passed
20 of 20` (2026-09-21T14:14:24Z). Thirteen, adding four already-covered spellings, reads the same at 24 of 24
(13:48:21Z / 13:48:44Z); twelve does NOT reach full membership, because the spelled-out `NEW YORK STOCK EXCHANGE`
arm sits OUTSIDE the `\m(...)\M` alternation and needs its own fixture.
Then deleting `NYSEARCA` from predicate (2) MOVES `venue_token` -- 2,564 -> 2,561 -- and STILL reads 24 of 24
green. Every member those fixtures cover is
unstraddled, because a fixture that wants FALSE goes on wanting FALSE once the token that matched it is gone: it
cannot discriminate in either direction. (Deleting `OTCQX` moves nothing on either side, so it is silent rather
than wrong -- no production surface names it at a word boundary.)
SO THE CRITERION IS THE PROPERTY, AND THE PROPERTY IS PER MEMBER: for EACH member of each of the five axes,
deleting or narrowing that member must turn a CONTROL red, AND must still do so on a FIXTURES-ONLY run -- the same
query with its corpus arm (the `SELECT ... FROM sentinel.extracted_observations` leg of `input`) removed. The
second half is the attribution: a fixtures-only run has no corpus, so a control that reddens there reddens BECAUSE
of the fixtures.
AN EARLIER REVISION OF THIS LINE SAID "while the corpus count is UNMOVED", AND THAT IS UNREACHABLE BY
CONSTRUCTION -- recorded because it is the same class as everything else in this entry. The three corpus
aggregates are `population`, `venue_token` and `falsifier`, and all three read `WHERE want IS NULL` (the committed
fence, lines 35-37), while every fixture carries a non-null `want`. So no fixture can move any of them, ever, and
the clause was a condition on PRODUCTION rather than on the fixture set. Measured 2026-09-21T14:13:41Z on a
2,564-surface `venue_token`: deleting `NASDAQ` from predicate (2) moves it by -2,071 whatever the fixtures are,
and 7 of the 13 venue members move it at all (NASDAQ 2,071, NYSE 320, CBOE 137, NEW YORK STOCK EXCHANGE 26, OTC 4,
NYSEARCA 3, BATS 1; the other six move nothing). Those 7 could never satisfy it. "The corpus count" was also
ambiguous between `falsifier` and `venue_token` and this entry used it both ways -- named explicitly now, which is
this entry's own AMBIGUITY DENIES applied to its own sentence.
AMBIGUITY DENIES: a fixture set that cannot be shown, member by member, to redden a control on the fixtures-only
run does not close this. The two mutants are a TRIPWIRE for this
entry, never the exit condition.

**THE D-19 FALSIFIER SELECT SPELLS THE US VENUE VOCABULARY THREE TIMES AND GUARDS ONE, AND THE TWO UNGUARDED COPIES
ALREADY MISCLASSIFY REAL REGISTRY SPELLINGS -- IN THE DIRECTION THE ENTRY FORBIDS.** Predicate (2)'s alternation is
checked against `ListingVenueRegistry` by `scripts/tests/test_d19_falsifier_vocabulary.py`. Predicate (4)'s 11-token
`head_is_venue` list and predicate (5)'s 13-token residual-strip list are separate hand-kept copies that no test
reads. This is D-19's own INTENT sentence -- "a listing venue was named in three hand-kept places and they
disagreed" -- reproduced inside the query written to falsify it.
THE DELETION IS SILENT AND THE DELETED TOKEN IS LOAD-BEARING -- both measured 2026-09-21, and it takes both to
make this a defect rather than a tidy-up. SILENT: deleting `new york stock exchange|` from predicate (5)'s strip
list leaves the full query reading `falsifier` 14 and `controls_passed 11 of 11` (12:31:49Z) and leaves
`python -m pytest scripts/tests` at 161 passed (the roster at `20d1540d`; it was 160 before #1092 added a
sixth `scripts/tests/test_*.py`). Nothing in the repository goes red. LOAD-BEARING: the same deletion
flips `the New York Stock Exchange-listed REIT` -- the exact shape this falsifier exists for -- from TRUE to FALSE
(13:00:10Z, driven through the committed `c`/`d` CTEs). So the token is doing real work, and its removal is
invisible.
THE CASING IS THE MECHANISM, AND THE COUNT IS MEANINGLESS WITHOUT A NAMED RENDERING. Predicate (5) excludes a row
when the residual carries a MIXED-CASE token (`\m[A-Z][a-z]`, which no acronym matches). Of the 20 US `Spellings`,
8 are multi-token; 2 of those (`NYSE ARCA`, `NYSE AMERICAN`) ARE spelled whole in the strip list (`nyse ?arca`,
`nyse ?american`) and survive; the other 6 are not, so stripping only the short token leaves a remnant -- and
whether that remnant is mixed case is decided by how the surface is WRITTEN, not by the registry. Derived
2026-09-21T13:00:10Z, one `the <spelling>-listed pharmaceutical company` probe per registry US spelling, pushed
through the committed CTEs, `NOT falsifier` counted:
- AS STORED (the array's own UPPER CASE): 0 of 20. An all-caps remnant matches no mixed-case token, so this
  rendering CANNOT expose the defect and a zero here means nothing.
- TITLE CASE (`str.title()`): 6 of 20.
- ACRONYM-PRESERVED TITLE CASE (title-case every token except a predicate-(2) alternation member): 6 of 20.
The SAME six spellings under both non-stored renderings: `NEW YORK STOCK EXCHANGE, INC.`, `NASDAQ NMS - GLOBAL
MARKET`, `OTC MARKETS`, `NYSE MKT LLC`, `CBOE BZX`, `BATS EXCHANGE`. The vocabulary gate passes all six because its
test is SUBSTRING CONTAINMENT -- `OTC` is in `OTC MARKETS`, so the spelling counts as named. These are FALSE
NEGATIVES, and D-19's own standard for this predicate is "THIS DETECTOR MAY ONLY ERR THE OTHER WAY".
THE STRIP MUTANT MOVES NONE OF THE 20, under any of the three renderings (6 -> 6, 0 -> 0): the only registry
spelling containing `new york stock exchange` is already excluded on the COMMITTED query by its `Inc.` remnant. The
mutant is visible only on prose that names the venue without the corporate suffix -- the REIT probe above. The two
findings are independent and neither substitutes for the other.
THE EXPOSURE IS ZERO TODAY, which is what makes this measurement debt and not a live defect, and it is the figure
that must be re-read before anyone decides the priority. Read 2026-09-21T12:32:37Z on `atlas_data`: of 432,828 rows
with a non-blank `source_entity`, 0 carry any `<multi-word US venue>-listed/-quoted` shape; 0 carry `-quoted` at
all; 12 rows (8 distinct surfaces) carry `-listed` and NONE names a US venue (`BSE-listed companies`, `US-listed
spot Bitcoin ETFs`, `North American-listed funds`, `five major mainland-listed insurers` ...); 34 rows name a
multi-word US venue token anywhere at all. An extraction or prompt change that starts emitting the multi-word
spelling moves that 0, and the falsifier will not see it.
RE-CHECK (SELECT-only), AND IT IS INVERTED ON PURPOSE: this entry documents a LIVE blindness, so the check must
return a NON-ZERO count today and CLOSES AT ZERO. Build one `the <spelling>-listed pharmaceutical company` probe per
entry of `ListingVenueRegistry`'s US `Spellings` array, render each in TITLE CASE (`str.title()`), push them through
the committed `c`/`d` CTEs and count `NOT falsifier` -> 6 of 20 on 2026-09-21. Do NOT run it with the spellings AS
STORED: that returns 0 of 20 by construction, which reads exactly like a fix. Exposure leg, same session:
`SELECT count(*) FROM sentinel.extracted_observations WHERE btrim(source_entity) ~* '(new york stock exchange|nasdaq nms|otc markets|nyse mkt llc|cboe bzx|bats exchange)[ ,.A-Za-z-]{0,30}-(listed|quoted)\M'`
-> 0 of 433,325 on 2026-09-21T13:24:29Z. THAT PATTERN IS THE WIDENED ONE, AND THE FIRST ONE SHIPPED HERE WAS BLIND:
its `[- ]?[A-Za-z]*-` span crossed neither `, Inc.-` nor ` - Global Market-`, so it could not emit a
counter-example for 2 of the 6 spellings it bounds. Controlled on the fixture set where the two disagree
(13:24:29Z): the shipped pattern scores `the New York Stock Exchange, Inc.-listed ...` and
`the NASDAQ Nms - Global Market-listed ...` FALSE and the widened one TRUE, with the other four spellings TRUE
under both. The FIGURE was unaffected -- the widened pattern also returns 0 -- so this was the instrument, not the
conclusion. CLOSES WHEN THE TITLE CASE COUNT REACHES 0, and on nothing else. "Carries a test" does NOT
close it: applying the shape of the repo's only such test to these two lists returns `missed=[]` on both
(2026-09-21) while this count is still 6 of 20, because that gate decides naming by SUBSTRING CONTAINMENT -- the
exact mechanism this entry blames for hiding the defect (`OTC` is in `OTC MARKETS`, so the spelling reads as
named). AMBIGUITY DENIES: a proposed test that cannot be shown to go red on today's 6 does not close this.

**SEVEN TEST AND SELFTEST HARNESSES UNDER `scripts/` ARE RUN BY NO WORKFLOW, AND TWO OF THEM SIT INSIDE THE
DIRECTORY CI SWEEPS.** Derived 2026-09-21 by reading every step of the four files in `.github/workflows/`
(alert-rules, hosted-service-pins, python-tests, sync-docs) and matching it against every harness under `scripts/`:
`audit-catch-spans.py` (which carries a `--selftest`), `test-devcontainer-owner.sh`,
`test-devcontainer-simultaneity.sh`, `tests/new-epic-selftest.sh`, `tests/build-deploy-hint-selftest.sh`,
`gemini-spend-calibration/test_probe_replay.py` and `gemini-spend-calibration/mutation-check.py`. The other 8 of the 15
are run, all of them by python-tests.yml's two `pytest <dir>` steps -- the six `scripts/tests/test_*.py` and the two
`scripts/sentinel-quality-check/test_*.py`. (Re-derived 2026-09-21T13:49:08Z at `20d1540d`, which added a sixth
`scripts/tests/test_*.py`; the UNRUN set is unchanged at 7, same files.) (`scripts/verify-hosted-service-pins.py` has its own workflow but is a
verifier, not one of these 14.)
TWO DISTINCT SHAPES, and the second is the one a reader gets wrong. `scripts/tests/**` is in python-tests.yml's
trigger path list AND the job runs `python -m pytest scripts/tests`, so both `.sh` selftests sit inside the swept
directory, trigger the workflow on every edit, and are never executed -- pytest collects `test_*.py` only. A PR
that breaks `new-epic-selftest.sh` turns the workflow GREEN on a run that touched it.
FIVE OF THE SEVEN HAVE NO AUTOMATED INVOKER ANYWHERE IN THE REPOSITORY -- each DOES carry a documented hand-run
command (`scripts/README.md`, `scripts/tests/README.md`, `scripts/gemini-spend-calibration/README.md`,
`.claude/skills/deploy/SKILL.md`), which is precisely what makes it a suite that runs by memory (`audit-catch-spans.py`,
`test-devcontainer-owner.sh`, `tests/new-epic-selftest.sh`, `tests/build-deploy-hint-selftest.sh`,
`mutation-check.py`); the other two are chained from one of those five (`test-devcontainer-owner.sh
--with-containers` runs the simultaneity proof, `mutation-check.py` drives `test_probe_replay.py`), which is one
unrun root away from the same thing.
THE ROSTER IS SCOPED TO `scripts/` AND THE RE-CHECK BELOW IS NOT: an earlier revision of this entry claimed
`gemini-resolver-mcp/**` was a trigger path with no step running its 8 tests, which is FALSE --
python-tests.yml:438-443 runs `python -m pytest gemini-resolver-mcp/tests -v` unconditionally, after its editable
install at `python-tests.yml:436`, with no `if:` anywhere in the file (line numbers re-derived by `grep -n` at
`20d1540d` 2026-09-21T14:12:50Z -- #1092 added 96 lines to that file and moved the step from :357). It slipped because the check written beside it extracted only
`(scripts|deployment|LlmBenchmark)/` paths and so could not emit the counter-example. Leg (1) is now path-agnostic
for exactly that reason -- a check whose output population cannot contain the counter-example is not a check.
This EXTENDS the `run-wiring-smoke.sh` entry above, which already recorded that no workflow runs the hook suites
and that `new-epic-selftest.sh` sits in the same position; this is the derived ROSTER for `scripts/`, not a second
copy of that finding. FILED, NOT FIXED: wiring a container-driving suite (`test-devcontainer-simultaneity.sh`,
~2 min, real containers) into CI is a decision about runner cost, not a docs edit.
Re-check, in TWO legs, because a path in a workflow's `paths:` trigger is not a step that runs it and a
basename grep over the workflow files cannot tell the two apart (it reports the swept `scripts/tests/test_*.py` as
unrun): (1) the harness paths a step actually EXECUTES --
`grep -rhE '^[[:space:]]+' .github/workflows/*.yml | grep -vE "^[[:space:]]*(-[[:space:]]*'|#)" | grep -oE '\b[A-Za-z][A-Za-z0-9._-]*(/[A-Za-z0-9._*-]+)+' | sort -u`
-- it matches ANY path, never a hardcoded prefix list, and it drops the `- '...'` trigger entries and comment
lines that are not steps -> 34 paths on 2026-09-21T13:49:08Z, of which 3 are under `scripts/` (`scripts/tests`,
`scripts/sentinel-quality-check`, `scripts/verify-hosted-service-pins.py`) and one is `gemini-resolver-mcp/tests`.
Cross-checked the same day against a run-block parser that reads only `run:` bodies: identical but for
`actions/checkout`, `actions/setup-python` and `jpansarasa/ATLAS-Docs`, which come from `uses:`/`repository:` and
are not repo harness paths; (2) the harnesses --
`git ls-files scripts/ | grep -E '(test[-_][^/]*|[^/]*-selftest)\.(sh|py)$|audit-catch-spans\.py|mutation-check\.py'`
-> 15. A harness counts as RUN only when leg (1) names its path, or names a directory a `pytest <dir>` step
sweeps AND the harness is a `test_*.py` (that is the 8 under `scripts/tests/` and `scripts/sentinel-quality-check/`);
everything else is UNRUN -> 7. Closes when each of the 7 is either invoked by a
workflow or carries a recorded decision not to be.

**THE FROZEN ATTACH-CANDIDATE LISTS ARE COUPLED TO THE D-19 HEAL, AND THE ONLY THING GUARDING THEM IS A 24 h CLOCK
THAT COMPARES NO CONTENT.** Measured 2026-09-21. `LlmBenchmark/attach-gold/frozen/attach_candidates_g1_k20.json` and
`..._g2_k20.json` were frozen by calling `GET /api/semantic/candidates` against the catalog at `catalog_snapshot`
2026-09-20T12:35:13Z and 12:36:49Z, so every candidate row in them carries the PRE-heal `exchange` string AND was
ranked by a vector tier reading PRE-heal embeddings. The two files hold 13,802 candidate records over 4,152 distinct
instruments. 2,457 records (17.8%), naming 742 distinct instruments (17.9%), carry an exchange spelling
`CanonicalizeExchangeVocabulary` rewrites -- derived twice and agreeing at 742, once from the frozen values and once by
joining the 4,152 ids to the live catalog. Those 742 rows also take an `updated_at` bump from the heal and therefore
RE-EMBED, so after the deploy the same query does not return the same neighbours the file holds.
THE GUARD CANNOT SEE ANY OF THAT. `eval_harness.check_catalog_drift` holds the gold's `catalog_at` against the file's
`catalog_snapshot` and refuses past `ATTACH_CATALOG_DRIFT_TOLERANCE` (24 h) -- both are FIXED stamps written at freeze
time, so the comparison returns the same verdict forever and a catalog rewritten underneath the freeze passes it
unchanged. The staleness attestation is not a second witness: it re-checks that each ACCEPTED id is still active and
proposable, and a re-embed and an exchange rewrite both leave that true. Consequence: scoring stays REPRODUCIBLE,
which is precisely the hazard -- a post-deploy score describes a candidate list production no longer produces, and
nothing in the run says so. NOT A PROPOSAL TO WIDEN THE WINDOW OR TO DIFF EXCHANGES HERE: what the guard ought to
compare is a separate decision, already escalated; this entry exists so the coupling is measured rather than
remembered. Re-check, from the repo root -- the spelling map is read out of the migration, so the count moves when the
vocabulary does rather than agreeing with a copy of itself:

```
python3 - <<'EOF'
import json
M = {l.split("'")[1].upper() for l in open(
  "SecMaster/src/Data/Migrations/20260921084921_CanonicalizeExchangeVocabulary.cs")
  if l.strip().startswith("('")}
n = hit = 0; ids = set()
for g in ("g1", "g2"):
    d = json.load(open(f"LlmBenchmark/attach-gold/frozen/attach_candidates_{g}_k20.json"))
    for a in d["articles"]:
        for o in a["owners"]:
            for c in o["candidates"]:
                n += 1
                if (c.get("exchange") or "").strip().upper() in M:
                    hit += 1; ids.add(c["id"])
print(f"records={n} rewritten={hit} distinct_instruments={len(ids)}")
EOF
```

-> `records=13802 rewritten=2457 distinct_instruments=742` on 2026-09-21, and `grep -n
'ATTACH_CATALOG_DRIFT_TOLERANCE =' LlmBenchmark/scripts/eval_harness.py` -> one `timedelta` held against two stored
stamps. Closes when a scoring run against a post-heal catalog either re-freezes the lists or refuses to score.

**TWO D-19 VENUE SUFFIXES NAMED THE WRONG VENUE, AND 6 CATALOG ROWS CARRY A FIGI OBTAINED THAT WAY -- 5 OF THEM
ANOTHER COMPANY'S. THE SUFFIXES ARE NULLED (#1091), SO NOTHING NEW IS POISONED; NO ROW IS REPAIRED.** Both entries
came verbatim from the hand-kept client map on `main`, so this is a LIVE DATA DEFECT, not a regression. This is
CLAUDE.md GIGO's named harm and the same class D-19 exists to fix: a wrong resolution at the SOURCE corrupts identity,
and the free wrong-ticker resolutions cost more than a bill would. Measured 2026-09-21 on `atlas_secmaster`,
SELECT-only, plus a READ-ONLY `POST /v3/mapping` that resolved each stored FIGI back to the issuer it names.

`.SG` WAS DECLARED AS SINGAPORE'S (venue `SP`) AND IS BOERSE STUTTGART'S. 194 active `.SG` rows sit beside the German
regional suffixes (`.F` 582, `.DU` 537, `.MU` 308, `.HM` 267, `.BE` 3, `.HA` 8) and carry cross-listing names --
`NVR INC`, `ANGLO AMERICAN PLC-SPONS ADR`, `HEIWA CORP`, `ROBINSON PLC`, `JBT MAREL CORP`. The 87 genuine Singapore
rows are on `.SI` (`1D3.SI`, `40B.SI`, ...), a suffix the table has never held. 6 of the 194 obtained a FIGI under
exchCode `SP`; every one of the 6 also carries `exchange = 'SP'` and `country = 'SG'`, which is wrong for a German
listing, and `SP` is held by no other row in the catalog. Resolving each stored FIGI back:

| symbol | stored name | stored FIGI | the FIGI's issuer |
|---|---|---|---|
| `CAO.SG` | CONAGRA BRANDS INC | BBG000BG7SH4 | CHINA AVIATION OIL SINGAPORE |
| `CMS.SG` | COMMERCIAL METALS CO | BBG01VVVG367 | CHINA MEDICAL SYSTEM HOLDING |
| `SCG.SG` | SPORTING CLUBE DE PORTUGAL | BBG0035NY5Y1 | SKYLINK HOLDINGS LTD |
| `SHS.SG` | SHENZHEN INVESTMENT LTD | BBG000BZB5L2 | SHS HOLDINGS LTD |
| `VCM.SG` | VECIMA NETWORKS INC | BBG000C4P3P6 | VICOM LTD |
| `UOB.SG` | UNITED OVERSEAS BANK LTD | BBG000BFDWJ8 | UNITED OVERSEAS BANK LTD |

Five of six name a different company. `UOB.SG` matches only because the Stuttgart line and the Singapore primary share
both ticker and issuer -- a coincidence, not a working lookup.

`.MA` WAS DECLARED AS MADRID'S (venue `SM`) AND NAMES ZERO ROWS. The 95 real Madrid rows sit on `.MC` (`BBVA.MC`,
`AENA.MC`, `ACS.MC`, ...), which is Casablanca's CODE in the same table, deliberately suffix-less; the 39 `.CS` rows
are the Moroccan ones (33 spelled `CASABLANCA STOCK EXCHANGE`). `SM` is held by no catalog row at all. So the
declaration bought nothing and would have routed a future Moroccan `.MA` row to Spain.

FIXED IN #1091, AND WHAT IT DOES NOT FIX. Both `YahooSuffix` values are now `null`, which is fail-closed: `SuffixToCode`
has exactly one production consumer (`OpenFigiClient.BuildJob`) and an absent suffix builds no job, so those symbols
are never asked. Pinned by `ListingVenueRegistryTests.no_suffix_is_registered_for_the_two_that_were_measured_to_name_another_venue`
and, on the outbound side, `OpenFigiClientTests.should_never_send_a_symbol_whose_suffix_names_another_venue`. NOT
fixed: the 6 poisoned rows keep their wrong FIGI, exchange and country, and 194 `.SG` + 95 `.MC` + 87 `.SI` rows now
have no venue route at all. Deciding what a Stuttgart cross-listing, a Madrid line and a Singapore line SHOULD resolve
to is a venue decision with its own migration. Re-check:
`SELECT symbol, name, figi, exchange, country FROM instruments WHERE symbol LIKE '%.SG' AND figi IS NOT NULL;`
-> 6 rows on 2026-09-21; the entry closes when those rows are repaired or quarantined and the three suffix families
have a decided venue.

**5 OF THE 6 SECMASTER MIGRATION TEST CLASSES DRIVE THE SQL CONSTANTS, NEVER `Up` OR `Down`.** Measured 2026-09-21:
`grep -c 'UpOperations\|DownOperations'` returns 7 for `CanonicalizeExchangeVocabularyMigrationTests` (fixed in #1091)
and 0 for `ClearOutOfScopeUsAuthorityStampsMigrationTests`, `DisposeGeminiFallbackFuturesRootsMigrationTests`,
`ReclassifyFredSeriesLabelledEquityMigrationTests`, `RepairGeminiFallbackSurfaceNamesMigrationTests` and
`RetireDiscontinuedSeriesMigrationTests`; `grep -rn 'UpOperations\|DownOperations' SecMaster/src` returns nothing, so
nothing outside that one test file drives a migration's operations. Consequence, and it is the trap D-17's own entry
documents: emptying `Up()` -- or deleting any single `migrationBuilder.Sql(...)` call from it -- leaves each class
GREEN, while the shipped migration records itself APPLIED in `__EFMigrationsHistory`, heals zero rows and can never
re-run. The exposure is 8 `Up` statements and 11 `Down` statements across the five
(`AddInstrumentRetiredAtAndRetireDiscontinuedSeries` 3/1, `RepairGeminiFallbackSurfaceNames` 1/1,
`ReclassifyFredSeriesLabelledEquity` 1/4, `DisposeGeminiFallbackFuturesRoots` 2/2,
`ClearOutOfScopeUsAuthorityStamps` 1/3). All five are already DEPLOYED, so the risk is a future edit to a spent
migration, not a heal pending today -- which is why this is filed rather than swept. The fix per class is the helper in
`CanonicalizeExchangeVocabularyMigrationTests` (`OperationSql` / `RunOperationsAsync`, ~8 lines). Re-check: the two
greps above; the entry closes when all six read non-zero.

**DEACTIVATING A `source_mappings` ROW IS A ONE-WAY DOOR, AND THE LOOKUP THAT WOULD CREATE A REPLACEMENT CANNOT SEE
THE CORPSE AS DEAD, SO THE REGISTRATION ANSWERS SUCCESS AND INSERTS NOTHING.** Code facts at `97475644`;
`atlas_secmaster` read SELECT-only at 2026-09-21T15:44Z, the round-2 re-derivations at 16:26Z; Prometheus
instants 2026-09-21T15:40:00Z and 16:05:00Z.

ONE WRITER CLEARS THE FLAG AND NONE RESTORES IT. `FredCatalogReconciliationService.cs:183` (`ReactivateAsync`) sets
`mapping.IsActive = false` on every non-`FredCollector` mapping of an instrument it reactivates. That is the ONLY
assignment to an EXISTING `SourceMappingEntity.IsActive` in the tree, and no raw SQL writes the column either:
`grep -rn "UPDATE source_mappings" --include='*.cs' --include='*.sql' --include='*.py' --include='*.sh' .` returns 7
hits, all in `20260731011758_HealFredSourceMappingFrequencyDrift.cs` and all writing `frequency`. The writer is LIVE,
not historical -- `SecMaster/src/DependencyInjection.cs:144` registers `FredCatalogReconciliationHostedService` and
`SecMaster/src/Endpoints/AdminEndpoints.cs:77` calls `ReconcileAsync` on demand.

CORRECTION TO THE REPORT THIS ENTRY WAS RAISED FROM: `CatalogService.cs:392` DOES assign `IsActive = true` on a
`SourceMappingEntity`, so "the only `IsActive = true` assignments are on instruments and sector overrides" is FALSE,
and so is "neither create path sets it explicitly". It is an object initializer on a NEW entity in
`PromoteToCollectionAsync` (`CatalogService.cs:341`), never a revival of a stored row, so the one-way-door claim
itself stands. The two register INSERTs (`RegistrationService.cs:422` and `:484`) are the ones relying on the defaults --
`SourceMappingEntity.cs:19` (`= true`) and `SourceMappingConfiguration.cs:52-54` (`HasDefaultValue(true)`).

THE LOOKUP IS BLIND, AND ITS OWN NEIGHBOUR IS NOT.
`InstrumentRepository.GetSourceMappingByCollectorIdAsync` (`SecMaster/src/Data/Repositories/InstrumentRepository.cs:202-207`)
matches `m.Collector == collector && m.SourceId == sourceId` and nothing else, while `GetSourceMappingsAsync` nine
lines above it (`:193-199`) does filter `m.IsActive`. `RegisterCoreAsync` calls the blind one first
(`RegistrationService.cs:326`), finds the dead row, and returns
`Success = true`, `Confidence = ExactMatch`, `Message = "Source mapping already exists"` (`:375-384`) without reaching
either INSERT. The branch carries THREE `LogWarning` sites (`:340` instrument inactive, `:363` frequency
corrected, `:370` blank frequency preserved) and NONE of them keys the MAPPING's state: `:340` tests the INSTRUMENT
(`existingInactive = existingMapping.Instrument is { IsActive: false }`, `:337`) and all 14 dead rows point at ACTIVE
instruments, while `:363`/`:370` key the request's frequency. So a re-registration of one of the 14 is silent at
Warning ONLY when its frequency is non-blank and equals the stored value; otherwise it logs about frequency and still
says nothing about the corpse. Second consumer of the same
blindness: `ResolutionService.ResolveByCollectorIdAsync` (`:151-163`) would serve a dead mapping as a live
`SourceResolution`; dormant today (`SecMaster/AGENT_README.md` DISTINCTIONS: `LookupSource` is integration-tests-only).

THE COLLISION IS MASKED, NOT ABSENT. `idx_source_mappings_collector_source` is UNIQUE on `(collector, source_id)` with
NO predicate: `SecMasterDbContextModelSnapshot.cs:892-894` carries `.IsUnique()` and no `HasFilter`, and the catalog
agrees -- `CREATE UNIQUE INDEX idx_source_mappings_collector_source ON public.source_mappings USING btree (collector,
source_id)`. An INSERT on a dead key WOULD raise 23505; the blind lookup short-circuits before it.

POPULATION. UNIT: rows in `atlas_secmaster.public.source_mappings`. POPULATION: all 7,459 rows, 4 distinct collectors.
READ TIME 2026-09-21T15:44Z. 14 rows are `is_active = false`; every one is `collector = SentinelCollector`; every
`updated_at` falls inside a 399.754 ms window on 2026-06-06 (12:10:45.436621+00 to 12:10:45.836375+00), which is one
pass of `ReactivateAsync`; every `source_id` is a FRED series id (GDP, GDPC1, HOUST, MSPUS, PCE, PI, RSXFS, TCU,
T5YIE, CCSA, DGS10, ICSA, IPMAN, RSAFS), created 2026-01-23 to 2026-02-07.

WHAT BOUNDS THE HARM TODAY. (1) NO SERIES IS DARK, in the STRONG form: for each of the 14, a live `is_primary`
`FredCollector` mapping exists on the IDENTICAL `source_id`, count exactly 1, single distinct value across all 14
(re-derived 2026-09-21T16:26Z; re-check (a) is that predicate, not a weaker "some live primary"). `ResolveBatch` still
resolves all 14. (2) THE BLOCKED KEYS BELONG TO A COLLECTOR NAME NOTHING WRITES ANY MORE: both Sentinel self-seed
sites send `Collector: "GeminiFallback"` (`SentinelCollector/src/Services/DeterministicResolver.cs:1080`,
`SentinelCollector/src/Workers/ExtractionProcessor.cs:1752`), no `"SentinelCollector"` literal is sent as a collector
anywhere under any `src/`, and the newest `SentinelCollector` mapping was created 2026-05-16 -- so the false-success
path cannot currently be ENTERED for these 14.

AND THE FORWARD POOL MINTS CORPSES, NOT LIVE-COLLECTOR BLOCKS -- a severity claim an earlier revision of this entry
got wrong and which is worth keeping as the correction. 74 instruments carry both a live `FredCollector` mapping and a
live non-Fred one, but GROUPED BY that non-Fred collector the query returns a SINGLE row: `SentinelCollector` 74, i.e.
100% of the pool is the same dead name as bound (2), so a future reactivation over it mints another corpse and blocks
NOBODY. The axis that would actually matter -- an instrument carrying a live `FredCollector` mapping AND a live
mapping from a collector that is neither Fred nor Sentinel -- measures **0**, and re-check (d) is re-scoped to it.
For scale: 147 instruments carry a live `OfrCollector` or `FinnhubCollector` mapping, and not one of them also carries
a live `FredCollector` mapping, which is why `ReactivateAsync` cannot strip a live collector's mapping today at all.
(All four figures: `atlas_secmaster`, read 2026-09-21T16:26Z.)

WHY THIS IS FILED AND NOT FIXED -- DO NOT TAKE THE BAIT. Adding the missing `IsActive` filter at `:206` is one line and
is NOT the fix: the lookup then MISSES the dead row, the caller falls through to the INSERT at
`RegistrationService.cs:422`, and that INSERT collides on the non-partial unique index -- a silent false success
traded for an un-alerted 23505.

ONE GUARD STANDS BETWEEN THE TWO, AND IT COVERS ONLY A THIRD OF THE ADMISSIBLE CLASSES, SO DO NOT STATE THE TRAP
UNCONDITIONALLY AND DO NOT READ IT AS CLOSED EITHER. `IdentityConflict.ClassFamiliesDiffer`
(`SecMaster/src/Services/IdentityConflict.cs:48`) runs between the missed lookup and `:422`. For the 14 the existing
rows are `asset_class = Economic` -> NON-LISTED (`ListedClasses` = Equity/ETF/Stock, `IdentityConflict.cs:25-30`), and
`SentinelCollector` is not in `TrustedMacroCollectors` (`RegistrationService.cs:54-61`), so D-4 admits only
`EquityShapedAssetClasses` from it -- but that set is {Equity, ETF, Index, Currency, Crypto, Commodity}
(`RegistrationService.cs:36-44`) and FOUR of those six are NON-LISTED under D-16. So an `equity` or `etf`
re-registration is refused before `:422` and raises no 23505, while `index`, `currency`, `crypto` or `commodity`
agrees on family, passes D-16 and DOES reach the INSERT. 2 of 6 blocked, 4 of 6 collide. A review round asserted
"only a Listed class is admitted, so there is no 23505 for any of the 14"; the admitted set above is why that does not
hold, and this entry uses the derivation rather than the assertion.

AND THE 23505 DOES NOT STOP AT ONE -- BUT IT STOPS INSIDE THE RPC. The retry absorber at
`RegistrationService.cs:288-315` re-looks-up through the SAME method, so once it is filtered the absorber misses too;
the `// Not found` arm retries the full registration, attempt 1 raises the same 23505, and the
`when (IsUniqueViolation(ex) && attempt == 0)` filter no longer matches -- so a raw `DbUpdateException` leaves
`RegisterWithRetryAsync`. Not even the `InvalidOperationException` at `:316`, which the loop can never reach. It does
NOT leave the RPC: that call sits inside `RegisterAsync`'s `try` (`:128-130`), and the catch-all `catch (Exception ex)`
at `:142` takes it -- the narrower `catch (IdentityConflictException)` at `:135` cannot, that type being its own
direct `Exception` subclass (`IInstrumentRepository.cs:80`) and no base of `DbUpdateException` -- and it does not
rethrow. So the one-line fix trades a silent false success for a LOUD failure, observable on three channels read at
`97475644`: the `Registration.Register` span is set `ActivityStatusCode.Error` with the exception attached
(`:145-146`), which is the health instrument CLAUDE.md §OBSERVABILITY names; an `Error` log line carries
`{Collector}:{SourceId}` (`:144`), ABOVE prod's Warning floor (SecMaster D-6) and so visible without raising the
level; and the caller receives `Success = false` with `Message = "Registration failed: DbUpdateException: ..."`
(`:150-154`), which the collector-side Warning quotes verbatim
(`FinnhubCollector/src/Services/SeriesManagementService.cs:162`). Nothing is unhandled and no collector sees a throw;
UN-ALERTED, as the paragraph above says, is not unobserved.

The real fix is a decision between three candidates -- REVIVE the 14 rows, DELETE them, or SCOPE the index to
`WHERE is_active` (the shape `idx_instruments_symbol` already has) -- and this entry names all three without choosing.
That decision is the same one already parked as "Quarantined-ticker re-acquisition is an undecided policy: Gemini cost
+ un-alerted 23505" (AWAITING-DECISION, 2026-09-16, this file), whose HAZARD paragraph already names this exact index
as the un-alerted 23505 risk and whose question -- may a dead identity be re-acquired, and by which path -- is this
question one level up. Decide them together or neither; scoping the index closes the 23505 half of both.

NO SIGNAL DISCRIMINATES THIS DEFECT, AND THE REGISTRATION COUNTER THAT LOOKS LIKE ONE IS MERELY UNWARMED -- WHICH IS
A DIFFERENT DEFECT WITH A ONE-LINE FIX. `SecMasterMeter.RegistrationRequests` (`SecMasterMeter.cs:392`) is incremented
UNCONDITIONALLY on every registration at `RegistrationService.cs:108`, before the D-4 guard at `:112`, and the meter
is exported at `SecMaster/src/Program.cs:99`. It is named with DOTS (`secmaster.registration.requests`) and reaches
Prometheus with UNDERSCORES, which is why a grep for `secmaster_registration_requests_total` over the source finds
nothing; grep the OTLP spelling. Measured at 2026-09-21T16:05:00Z: `count_over_time(...[120d])` and `[365d]` return
the SAME 1,768 samples across 3 series tagged by collector (FinnhubCollector 1,484, FredCollector 248, OfrCollector
36) -- so retention exceeds 90 d and nothing predates 120 d -- with a first sample at 2026-06-06T12:11:58Z, 73 s after
the last deactivation write in the population above, and a LAST sample at 2026-06-12T22:50:58Z. An instant query
returns NOTHING. The cause is that `MetricWarmupHostedService` zero-initialises `AliasEnsureOutcomes` (`:51`) and
`RegistrationRejected` (`:57`) and does NOT list `RegistrationRequests`, so an unwarmed cumulative counter is simply
absent until the running process registers something -- both warmed counters read 0 at that instant, which is the same
fact made visible. THE FIX IS TO ADD IT TO THAT SERVICE, in the pattern its own docstring (`:9-18`) describes -- but
it is not a free line: `collector` is this counter's ONLY tag, and the same file's docstring (`:20-23`) refuses to
fabricate collector values for the rejection counter because the requesting caller is not a bounded set. Seeding this
one therefore needs a declared collector roster or a tagless seed, and that choice is the work.

WHAT SURVIVES THAT CORRECTION IS THE POINT OF THIS PARAGRAPH: even warmed, `RegistrationRequests` increments
IDENTICALLY for a registration that inserts and one that early-returns on a corpse, so it cannot report this defect at
any value. The only counter on the early-return branch at all is `SourceMappingFrequencyCorrected`
(`SecMasterMeter.cs:425`), which fires only when the frequency actually CHANGED, and which has zero samples over
`[365d]` -- unwarmed AND never incremented. The log line on the branch is `Information`
(`RegistrationService.cs:330`), below prod's Warning floor (SecMaster D-6). And per-mapping attribution is impossible
ON A METRIC by design: `SecMasterMeter.cs:423` says "Never tag source_id (unbounded)" and CLAUDE.md §OBSERVABILITY
caps a metric tag at under 100 distinct values, while `source_id` has 7,301 (read 2026-09-21T15:44Z). It survives only
on the LOG lines, which is why it does not close this gap: the `Information` of this branch, below the floor, and the
`Error` of the post-filter failure above.

A CONTROL MUST BE AIMED AT THE ACT UNDER CLAIM, AND THIS ENTRY IS ITS OWN WORKED EXAMPLE. Round 1 read
"172,772 samples on `rejected` and `alias_ensure`, 0 on `registration_requests`" as evidence that the third counter
was never wired. It is not: those 172,772 are a PER-SERIES sample count, and the two counters carrying them are
zero-initialised at process start while the third is not -- a warmed counter compared against an unwarmed one, with
the difference read as absence of instrumentation. The same round grepped the Prometheus spelling of a dotted
instrument name and read the empty result as "zero emitter anywhere in the repo". Both failed toward the reassuring
answer. The aimed control is the one above: query the metric over a window longer than its last activity and look for
SAMPLES, and grep the source for the spelling the SOURCE uses.

RE-CHECK, four parts; each can CONTRADICT the claim rather than confirm it by construction.
(a) the population and its bound. The sub-select carries the FULL predicate the bound claims -- same collector, same
`source_id` -- because "some live primary" is a weaker statement than the one this entry makes and would stay true
while the bound had already failed --
`sudo nerdctl exec timescaledb psql -U ai_inference -d atlas_secmaster -c "SELECT d.collector, d.source_id, d.updated_at, i.is_active AS instr_active, (SELECT count(*) FROM source_mappings a WHERE a.instrument_id = d.instrument_id AND a.is_active AND a.is_primary AND a.collector = 'FredCollector' AND a.source_id = d.source_id) AS live_fred_primary_same_id FROM source_mappings d JOIN instruments i ON i.id = d.instrument_id WHERE NOT d.is_active ORDER BY d.updated_at;"`
STILL BROKEN, HARM BOUNDED = 14 rows, every `updated_at` on 2026-06-06 and every `live_fred_primary_same_id` = 1
(today, single distinct value). STILL BROKEN, HARM GROWING = any row at 0 -- that series is now dark AND cannot be
re-registered -- or any `updated_at` after 2026-06-06, meaning a second deactivation happened or a writer now touches
the flag; re-derive the writer set before reusing this entry's. FIXED OR MOOT = 0 rows.
(b) the blindness itself. Anchor on the METHOD NAME, never on the signature text: an anchor like
`(string collector` prints nothing when a parameter is renamed, at rc 1, which maps to NO stated outcome and is the
fail-toward-success shape this entry is about --
`grep -n -A5 'GetSourceMappingByCollectorIdAsync' SecMaster/src/Data/Repositories/InstrumentRepository.cs`
STILL BLIND = the `FirstOrDefaultAsync` predicate names only `m.Collector` and `m.SourceId`. CHANGED = an `IsActive`
term appears -- then run (c) in the same breath, because the filter ALONE converts this defect into a 23505 and, per
the absorber above, into a `DbUpdateException` the RPC's catch-all returns as a failed response on an Error span.
EMPTY = the method was renamed or deleted; this entry's code half is stale and must be re-derived before any of it
is quoted.
(c) whether the collision is still masked --
`sudo nerdctl exec timescaledb psql -U ai_inference -d atlas_secmaster -c "SELECT indexdef FROM pg_indexes WHERE indexname='idx_source_mappings_collector_source';"`
MASKED = no `WHERE` clause (today). SCOPED = `WHERE (is_active = true)`, which is candidate fix 3 and closes the
23505 half of the AWAITING-DECISION entry too.
(d) the forward exposure, re-scoped to the axis that would actually matter. The earlier form filtered
`collector <> 'FredCollector'`, which cannot tell a corpse-mint from a live-collector block -- 100% of what it counts
is `SentinelCollector`. This one excludes that dead name, so a non-zero result IS a live collector losing a key --
`sudo nerdctl exec timescaledb psql -U ai_inference -d atlas_secmaster -c "SELECT count(*) FROM (SELECT m.instrument_id FROM source_mappings m WHERE m.is_active GROUP BY 1 HAVING count(*) FILTER (WHERE m.collector='FredCollector') > 0 AND count(*) FILTER (WHERE m.collector NOT IN ('FredCollector','SentinelCollector')) > 0) x;"`
0 = no instrument pairs a live FredCollector mapping with a live third-party one, so a reactivation cannot strip a
live collector (today). >0 = that many instruments are one quarantine-then-reactivation away from a permanently-dead
key for a collector that is still writing; fix before the next pass runs.

**`RegistrationService.cs:402`'s `if (!instrument.IsActive)` BRANCH IS UNREACHABLE, AND SO IS THE `:443` MESSAGE ARM
THAT READS THE SAME FLAG -- THE SUBJECT COMES BACK FROM AN `IsActive`-FILTERED READ.** Code facts at `76cd8d1c`,
derived 2026-09-21T19:47Z. There is no runtime measurement here on purpose: the claim IS that the branch has no
runtime, so a production figure would be a corpse-detector for it. Everything below is decided by reading the one
call path.

THE CALL PATH, AND THE ONE ASSIGNMENT A READER MUST CHECK AND DISMISS. `RegistrationService.cs:387` binds
`instrument` from `_repository.GetBySymbolAsync(symbol, ct)`, whose implementation
(`SecMaster/src/Data/Repositories/InstrumentRepository.cs:26-36`) ends in
`FirstOrDefaultAsync(i => i.Symbol == symbol && i.IsActive, ct)` at `:35` -- so any non-null row it returns is
ACTIVE by construction. Between the bind and the branch the only call is
`IdentityConflict.ClassFamiliesDiffer` (`SecMaster/src/Services/IdentityConflict.cs:48-49`), which compares two
class-family strings and touches no entity. `instrument` IS assigned a second time -- `RegistrationService.cs:461`,
`instrument = new InstrumentEntity { ... }` -- and that assignment is the reason this is worth writing down rather
than eyeballing: it sits inside the execution-strategy lambda on the `instrument == null` path, which runs only
after the `if (instrument != null)` block opened at `:389` has returned unconditionally at `:437-446`. It cannot
reach `:402`. There is no third write to the variable and none anywhere to the flag.

TWO DEAD CONSUMERS, NOT ONE, which is what a removal gets wrong if it reads only the branch. The first is
`:402-412` (a `LogWarning` at `:408-410` plus `Activity.Current?.SetTag("registration.instrument_inactive", true)`
at `:411`). The second is the ternary at `:443-445`, whose false arm -- the string ending "WARNING: instrument is
INACTIVE; resolution will not return it until reactivated" at `:445` -- is unreachable text on the same flag.

NO TEST DRIVES IT, so deleting it REDs nothing: `grep -rn 'IsActive' SecMaster/tests/Services/RegistrationService*.cs`
returns 6 hits at `76cd8d1c` and every one is `IsActive = true`. Note what that means about the INSTRUMENT: the
repository is mocked in those suites, so a test COULD hand the branch an inactive row that the real
`GetBySymbolAsync` can never return. None does.

WHAT REMOVING IT WOULD **NOT** CHANGE, and this is the half most likely to be misread as a bonus cleanup. `:411`
is the SECOND of two `registration.instrument_inactive` tag sites; the first is `:343`, on the existing-mapping
early return, whose subject is `existingMapping.Instrument` (`:337`) -- a navigation property off a row found by
`GetSourceMappingByCollectorIdAsync`, which carries no `IsActive` predicate at all. That site is REACHABLE: the
sibling ONE-WAY DOOR entry above measures it, and guard-site row 7 of `docs/proposals/delete-wrong-mappings.md`
records it as reachable via `GSV.NE` today. The `existingInactive` message ternary at `:381-383` is likewise the
reachable twin of the dead one at `:443`. So the tag, its cardinality and any consumer of it all survive a
deletion, and "this retires an unused tag" would be false.

WHAT IT WOULD CHANGE: nothing observable. Both arms are unreachable, so production behaviour, the emitted telemetry
and the test suite are all identical either side.

CONSEQUENCE IF THIS ENTRY IS MISSING OR FALSE, in two opposite directions. (1) A reader takes the branch as
evidence that registering a new source against a quarantined instrument is a handled, warned case at this site; on
this path it cannot arise at all. (2) The reverse -- someone sweeps it as obvious dead code without noticing it is
LATENT rather than merely dead. `InstrumentRepository.cs:28-30` calls that `IsActive` predicate "load-bearing" for
the FRED-pollution recovery path; narrow or drop it and this branch becomes the only thing telling an operator the
symbol stays dark. The disposition is therefore a decision -- delete it, or keep it with a comment saying what it
guards against -- not an automatic removal, and that is why it is filed with the alternatives named rather than
chosen. Impact B, not A: nothing is lost today, because nothing reaches it.

WHY IT IS FILED RATHER THAN REPAIRED. Found while planning the leg-A deletion work.
`docs/proposals/delete-wrong-mappings.md` step 2 already edits this file's comment set (its section 7 enumerates
three comment edits), and folding a code deletion on a guard-adjacent method into that PR widens a doc-and-comment
change into a behaviour-neutral code change nobody scoped it to review.

RE-CHECK, three parts, each able to CONTRADICT the claim rather than confirm it by construction. Anchor on method
names, never on signature text, so a rename reads as EMPTY rather than as a silent pass.
  `grep -n -A10 'GetBySymbolAsync(string symbol' SecMaster/src/Data/Repositories/InstrumentRepository.cs`
  STILL DEAD = the `FirstOrDefaultAsync` predicate still carries `&& i.IsActive`. ALIVE = that term is gone, the
    branch now reaches production, and this entry closes as MOOT rather than fixed -- re-read `:445`'s arm before
    trusting anything downstream of it. EMPTY = renamed; re-derive the whole path.
  `grep -n 'instrument.IsActive\|var instrument = \|instrument = new InstrumentEntity' SecMaster/src/Services/RegistrationService.cs`
  STILL TWO DEAD CONSUMERS = exactly two reads of `instrument.IsActive` and exactly two writes to `instrument`, the
    second of them below the unconditional return. A THIRD write, or a second one ABOVE that return, falsifies the
    unreachability outright and the entry must be re-derived, not patched.
  `grep -rn 'IsActive' SecMaster/tests/Services/RegistrationService*.cs`
  STILL UNPINNED = every hit is `IsActive = true`. A `false` appears = a test now drives the dead branch through the
    MOCK, asserting on a path production does not have; that test is itself the finding.

**FredCollector IS THE ONLY ONE OF THE FIVE COLLECTORS THAT BOTH DISCARDS THE SecMaster REGISTER RESPONSE AND
CARRIES NO FAILURE METRIC, AND ITS SPAN STAYS STATUS-UNSET -- SO THE HEALTH INSTRUMENT READS A REJECTED
REGISTRATION AS FINE.** Code facts at `76cd8d1c`, all five call sites read 2026-09-21T19:53Z. This is a CODE
reading, not a rate: no figure here says how often any collector's registration is rejected in production.

THE LOAD-BEARING FACT IS IN THE SHARED CLIENT, NOT IN ANY COLLECTOR, and it is why "it is inside a try/catch" is
not an answer for ANY of the five. `Events/src/Events.Client/SecMasterRegistryClient.cs:122` returns the response
for BOTH arms of the `if (response.Success)` at `:107`, so a `Success == false` is RETURNED, never thrown.
Transport failures return `null` instead: `:124` and `:135` catch Unavailable and DeadlineExceeded and RETRY, and
retry exhaustion falls through to `:156-162`; any other exception returns null at `:149-153`. The only exception
that escapes the method is `OperationCanceledException`, rethrown bare at `:145-148`. So no collector-side `catch`
can fire on a failed registration of any kind.

THE FIVE SITES -- one physical `RegisterSeriesAsync` call per collector, with every admin and backfill entry point
funnelling through it (`grep -rn 'RegisterSeriesAsync' --include=*.cs` over the five `src/` trees returns 6 hits,
the sixth being a comment at `AlphaVantageCollector/src/Services/SeriesManagementService.cs:170`):

| collector | call site | binds + tests the response | failure metric | span status on a rejection |
|---|---|---|---|---|
| Fred | `FredCollector/src/Services/SeriesManagementService.cs:309` | **no** -- return value discarded | **no** | **UNSET** |
| AlphaVantage | `AlphaVantageCollector/src/Services/SeriesManagementService.cs:84` | yes, tested `:102`, Warning `:104-106` | no | unset |
| Nasdaq | `NasdaqCollector/src/Services/SeriesManagementService.cs:94` | yes, tested `:113`, Warning `:115-117` | no | unset |
| Finnhub | `FinnhubCollector/src/Services/SeriesManagementService.cs:147` | yes, tested `:159`, Warning `:162-163` | no | unset |
| OFR | `OfrCollector/src/Services/SeriesManagementService.cs:556` | yes, tested `:567`, Warning `:571-573` | **yes**, `:569` -- but UNOBSERVED, below | **Error**, `:574` |

The split on the response is 4 / 1, and the four are not equivalent to each other: three produce a Loki Warning
only, and OFR alone also increments a counter and marks the span. Of the three right-hand columns only the last
TWO are the HEALTH instrument -- `failure metric` is Prometheus, `span status` is Tempo. The `binds + tests` column
and the Warning recorded inside it are Loki, which CLAUDE.md §OBSERVABILITY makes CONTENT after something is known
wrong and never the thing that knows it. So binding and testing the response is a real improvement on Fred's site
and is NOT what closes this entry -- see the closure condition under THE FIX, which the RE-CHECK below is bound to.

WHAT FredCollector's SITE DOES. `:309` calls `RegisterSeriesAsync` with no binding and no test. Its `try` opens at
`:307` and its `catch (Exception ex)` at `:323` sets `ActivityStatusCode.Error` (`:325`), adds the exception
(`:326`) and logs a Warning (`:327`) -- and per the paragraph above that catch cannot fire on a registration
failure. Because `:325` is the ONLY `SetStatus` on the span started at `:304`, and there is no `SetStatus(Ok)` on
the success path, a rejected registration leaves a status-UNSET span, which reads as non-error. CLAUDE.md
OBSERVABILITY makes Tempo span status plus Prometheus counters the health instrument, so this failure is invisible
to BOTH halves of it. (OFR is the contrast: `:578` sets `Ok` explicitly, so its spans discriminate.)

THE MITIGATING FACT, STATED SO THIS IS NOT OVERSTATED INTO "UNOBSERVED" -- which is what keeps it at B.
`SecMasterRegistryClient.cs:116-120` logs, at **Warning**, `"SecMaster registration rejected for {SeriesId}:
{Message}"` quoting the response message, whenever a response arrives with `Success == false`, regardless of what
the caller does with the return value. FredCollector's rejections DO reach Loki through that line. What
FredCollector lacks is its OWN signal -- a collector-scoped line naming the series and category it was registering
-- and a metric. Three conditions leave even the shared line silent, and all three are the `null` paths rather than
the rejection path: no response at all (`:151` and `:156-160` log different text and quote no message), a cancelled
call (`:145-148` rethrows with no log), and `RegisterBatchAsync` (`:168-195`), which never inspects `Success` and
has zero callers in the repo.

THE FIX IS NOT A NEW DESIGN, WHICH IS WHY THIS IS FILED SMALL -- BUT THE SHAPE ON OFFER IS ITSELF NOT YET
OBSERVED, SO COPYING IT ALONE WOULD NOT CLOSE THIS. `OfrMeter.SecMasterRegistrationFailures`
(`OfrCollector/src/Telemetry/OfrMeter.cs:175-178`) is `ofrcollector.secmaster.registration.failures.total`, unit
`{failures}`, tagged `type` at three bounded values, and OFR pairs it with an explicit span status on BOTH arms
(`Error` `:574`, `Ok` `:578`) -- that pairing is the part worth copying, because it is what makes a rejection
legible to Tempo and to Prometheus rather than only to Loki. THE COUNTER ITSELF IS UNOBSERVED, checked
2026-09-21T20:22Z at `76cd8d1c`: grepping the tracked tree for both the metric name
`ofrcollector.secmaster.registration.failures.total` (and its Prometheus spelling
`ofrcollector_secmaster_registration_failures_total`) and the symbol `SecMasterRegistrationFailures` returns hits
ONLY inside `OfrCollector/src`, plus `OfrCollector/AGENT_README.md:28` and this file -- no dashboard panel under
`deployment/artifacts/monitoring/dashboards/`, no Prometheus rule under `.../alerts/`, nothing under
`.../provisioning/` or `.../loki-rules/`. Per CLAUDE.md §OBSERVABILITY -- whose ban on a signal with no WIRED
alert is this repo's form of the user-level `observed` test (machine-local, untracked, and deliberately NOT
pointed at here because a `~/` path is outside every sweep) -- an unobserved metric is not a health signal but
waste plus complexity, so the fix for any collector is counter AND panel-or-alert, never
the counter alone -- and OFR's own missing panel is part of what this entry leaves open rather than a solved case
it points at. (Prometheus also returns no series for that metric at the same instant. That does NOT discriminate
never-incremented from never-exported by a Counter that has recorded nothing, so it is offered as neither a
rejection count nor a rate.) The open question is unchanged: whether the counter belongs per-collector or once in
`Events.Client` where the `Success` test already happens.

THE CLOSURE CONDITION, STATED ONCE HERE AND BINDING ON THE RE-CHECK BELOW. This entry's headline is that a
rejection is invisible to BOTH halves of the health instrument, so it closes only when that instrument can SEE
one: (a) the span started at `FredCollector/src/Services/SeriesManagementService.cs:304` carries a non-UNSET
status on the REAL path -- `Error` where a `Success == false` response is tested, AND `Ok` on the success path,
since without the second an UNSET span is ambiguous rather than good -- AND (b) a failure counter increments on
that same arm AND at least one dashboard panel or alert rule reads it. A bound-and-tested response with a
collector-scoped Warning satisfies NEITHER and does not close this: it is exactly the shape the other three
already have, and this entry exists because that shape leaves Tempo and Prometheus silent.

ONE ADJACENT CLAIM THIS ENTRY DOES **NOT** ESTABLISH. `NasdaqCollector/src/Services/SeriesManagementService.cs:110-112`
carries an in-code claim that SecMaster's D-4 allowlist rejects Nasdaq's default `"General"` asset class, i.e. that
every Nasdaq registration lands in the unsuccessful branch. That comment was READ, not verified against D-4 and not
against a live rate. If it is true, Nasdaq's Warning is firing continuously and that is a separate entry.

RE-CHECK, three parts, each able to contradict:
  `grep -n -A3 'if (response.Success)' Events/src/Events.Client/SecMasterRegistryClient.cs`
  STILL SWALLOWED = one `return response;` serves both arms. A `throw` on the false arm = every collector's catch
    comes alive, and this entry is re-derived from scratch rather than amended.
  `grep -rn 'RegisterSeriesAsync' --include=*.cs FredCollector/src AlphaVantageCollector/src NasdaqCollector/src FinnhubCollector/src OfrCollector/src`
  Expect 5 call sites plus 1 comment. A 6th call is a new site to classify; a missing one means a collector stopped
    registering, which is a louder finding than this entry.
  `grep -n -A25 'private async Task RegisterWithSecMasterAsync' FredCollector/src/Services/SeriesManagementService.cs`
  STILL OPEN = the `RegisterSeriesAsync` call is assigned to nothing, the body contains no counter `.Add(`, and the
    only `SetStatus` is inside the catch. CLOSED is the closure condition above and nothing weaker -- BOTH a
    non-UNSET span status on the real path (`Error` on the tested `Success == false` arm AND `Ok` on the success
    path) AND a failure counter on that arm that some panel or alert rule reads. Check the counter's OBSERVATION,
    not merely its existence: `grep -rn '<metric name>' deployment/artifacts/monitoring/` must return a hit. A
    bound response plus a Warning and nothing else = STILL OPEN, and reading it as CLOSED is the specific wrong
    action this entry is filed to prevent, because Loki is not the health instrument.

**D-17's `ClearOutOfScopeUsAuthorityStamps` names its rows by id as read at 2026-09-17T12:52:35Z, and production keeps
writing out-of-scope stamps until that migration deploys; a row written in between stays.** Measured 2026-09-17,
atlas_secmaster SELECT-only. The pre-S2 code gave DAN (Danieli, IM) Dana Inc's EDGAR CONS_DISC at 12:27:34Z, after a
first authoring at 10:54Z had read A4b as 33, so the lists were re-read at the migration's `AuthoredAt` (5 macro, 34
foreign, 150 home exchange). Rate: 4 of the listed stamps were written in the 30 days before `AuthoredAt` (FPH, SIG,
SKG, DAN), and the home-exchange list read 150 identical rows at both instants. At `AuthoredAt`,
`ClearOutOfScopeUsAuthorityStamps.UnclearedRowsSql` returned 0 stamps plus the 150 listed home exchanges the migration
has not yet moved; with its instant moved back to 10:54Z it also returned DAN. Consequence: a leftover keeps its wrong
sector, and post-deploy A4a, A4b and A4c read above 2, 0 and 0, which looks like a D-17 guard leak. After the deploy,
run that SELECT (`SecMaster/src/Data/Migrations/20260917110542_ClearOutOfScopeUsAuthorityStamps.cs`): each stamp it
returns classified before the deploy instant, and each home exchange, is a leftover; one classified after it is an A4f
leak, not this entry. It returns a leftover -> dispatch a follow-up EF data migration over those ids with the same
compare-and-swaps (`ClearStampsSql`, `MoveHomeExchangeSql`), never a predicate. It returns none -> close this entry.

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

**D-17's MIGRATION CLEARED THE STAMP AND LEFT THE DESCRIPTION, AND ITS OWN LEFTOVER QUERY CANNOT SEE THAT.**
[2026-09-20] `ClearStampsSql` nulls `description` only `WHEN EXISTS (SELECT 1 FROM edgar_filers f WHERE f.ticker =
i.symbol AND f.sic_description = i.description)`
(`SecMaster/src/Data/Migrations/20260917110542_ClearOutOfScopeUsAuthorityStamps.cs:265`), and `UnclearedRowsSql` @ `:309`
selects on `classification_source` alone. So a foreign row whose stamp WAS cleared keeps its EDGAR SIC description and is
invisible to the leftover query, to A4b, and to every acceptance count D-17 defines. Measured on `atlas_secmaster`
2026-09-20: 8 active bare-symbol rows on a non-US venue code carry a description that is some EDGAR filer's
`sic_description`, all 8 with `classification_source IS NULL`. Four of them hold a DIFFERENT company's line -- ACA on FP
is `CREDIT AGRICOLE SA` carrying Arcosa's `Fabricated Structural Metal Products`, SALM on NO is `SALMAR ASA` carrying
Salem Media's `Radio Broadcasting Stations`, REA on AT is `REA GROUP LTD` carrying Rare Earths Americas' `Metal Mining`,
and NANO on CT is `NANO ONE MATERIALS CORP` carrying ATII Holdings' `Blank Checks`. The other four (BHP, BNT, ERO, LAC)
happen to be right. Consequence: `EmbeddingService` vectorises the description, so a wrong one is a wrong identity
sentence for the vector tier, and `InstrumentClassificationBackfillService.ShouldOverwriteDescription` refuses a non-empty
description, so nothing repairs these without a data migration. Fix is a follow-up data migration over the ids, the same
compare-and-swap shape D-17 used, with the predicate widened to `f.sic_description = i.description` alone.
Re-check (psql is SELECT-only; `atlas_secmaster`): `SELECT count(*) FILTER (WHERE (SELECT count(*) FROM edgar_filers f
WHERE f.ticker = i.symbol AND f.sic_description = i.description) = 0) AS another_companys, count(*) AS rows FROM
instruments i WHERE i.is_active AND i.asset_class IN ('Equity','ETF') AND i.symbol !~ '\.' AND i.exchange ~ '^[A-Z]{2}$'
AND i.exchange <> 'US' AND i.description IS NOT NULL AND EXISTS (SELECT 1 FROM edgar_filers f WHERE f.sic_description =
i.description);` -> `1 | 8` on 2026-09-20 (the FILTER counts only NANO, whose description matches no filer holding its
ticker; ACA, SALM and REA need the names compared, which the row list above does). Close when it returns 0 rows.

**BTC-USD IS A SECOND BITCOIN ROW BESIDE THE CURATED ONE, AND EVERY OBSERVATION IS ON THE CURATED ONE.** [2026-09-20]
D-18 relabelled `BTC-USD` Equity -> Crypto, which was the eighty-second of its 82 and is correct; it did NOT merge the
row. `atlas_secmaster` holds `BTC-USD` (`entity_resolution:gemini`, created 2026-08-20T16:40Z, name `BTC-USD` -- the
surface echoed back, SecMaster D-2) beside `CRYPTO:BTC` (`curated_v1`, name `Bitcoin`, created 2026-05-02). Both active,
both Crypto. Measured 2026-09-20 on `atlas_data`: `CRYPTO:BTC` carries 1,101 observations (latest 2026-09-20T11:07Z) and
`BTC-USD` carries 0. The curated side GROWS (1,037 when the plan was written, 1,101 on 2026-09-20) -- a rising number
here is the healthy row working, not a drifting measurement; the figure that must stay 0 is `BTC-USD`'s.
So the duplicate attracts nothing today and costs nothing today -- it is a resolution hazard, not a
data defect, and the reason to close it is that an exact-symbol or vector hit on `BTC-USD` splits the series the moment
one lands. A third row, `BTC` (`GRAYSCALE BITCOIN MINI ETF`, Equity, FinnhubCollector, 128 observations), is a DIFFERENT
instrument and is not part of this. Disposition belongs to plan T3 items 1 and 2 (dedup/merge), not to a relabel.
Re-check (psql is SELECT-only): `SELECT symbol, asset_class, is_active FROM instruments WHERE symbol IN ('BTC-USD',
'CRYPTO:BTC');` on `atlas_secmaster` -> 2 active Crypto rows; then `SELECT instrument_id, count(*) FROM
sentinel.extracted_observations WHERE instrument_id IN ('a1e8f1a3-fbcd-42bc-a55a-f3fcdc2e15a8',
'b77a1888-9875-4857-9693-338f4f3fb1d3') GROUP BY 1;` on `atlas_data` -> ONE row only,
`b77a1888...`, whose count RISES over time (1,101 on 2026-09-20). A count appearing against `a1e8f1a3...` at all is the
split starting, and is what this entry watches; the curated count itself is not a threshold.

**SEVEN ACTIVE CATALOG ROWS HAVE A BLANK NAME, AND THE ATTACH FREEZE LABELLED 71 POOL SLOTS WITH NO NAME TO READ.**
[2026-09-20] Measured on `atlas_secmaster`: 7 active rows have `name = ''` (empty string, not NULL) -- ANTA.MU, DXLG.MU,
HPE.HM, MAMI.HM, NBC.HM, XMHQ.HM from FinnhubCollector and RJETQ from AlphaVantageCollector, every one a foreign or
delisted listing. Two consequences, both measured. (1) A blank name is invisible to every name-keyed tier: the pool
builder's ILIKE and trigram legs rank on `similarity(name, ...)`, so these rows can enter a pool only by exact
symbol/alias or by vector, and `EmbeddingService`'s identity sentence is built without the one field that identifies the
row. (2) `LlmBenchmark/attach-gold/attach_pools_v1.json` (frozen at catalog_at 2026-09-17T11:12:23Z) puts these ids in 71
pool slots across 43 of its 920 units, and `build_attach_gold.py:666` renders a falsy cell as `-`, so the paid labellers
judged those 71 slots with a symbol and no name. NOT a dropped slot and not a corrupted label -- the gold's accepts were
adjudicated and NOT_IN_POOL re-checked by SQL -- but the freeze cannot be re-derived as name-complete, and any future
pool inherits the same blindness until the rows are named or retired.
Re-check (psql is SELECT-only; `atlas_secmaster`): `SELECT count(*), string_agg(symbol, ', ' ORDER BY symbol) FROM
instruments WHERE is_active AND (name IS NULL OR btrim(name) = '');` -> `7 | ANTA.MU, DXLG.MU, HPE.HM, MAMI.HM, NBC.HM,
RJETQ, XMHQ.HM` on 2026-09-20. Pool side, no database: count those ids in `attach_pools_v1.json` -> 71 slots over 43
units.

**`selfseed_class_skip` (SentinelCollector D-34) HAS NO ALERT, BECAUSE ITS PRODUCTION RATE HAS NEVER BEEN MEASURED.**
[2026-09-20] The D-34 skip counts `sentinel_gemini_resolver_calls_total{outcome="selfseed_class_skip",reason}` (V2) and
`sentinel_gemini_fallback_calls_total{...}` (V1), 4 series zero-initialised at boot. A sustained skip rate means the
resolver's class vocabulary has drifted from the table both services hold, which is silent otherwise -- but no alert is
wired, deliberately: a hold derived from an average pages about ten times a week on a bursty signal, and this signal's
shape is unknown because the outcome did not exist before this PR. Expected rate is near zero: replaying the resolver
cache under the vehicle-first rule, SecMaster D-16 measured 0 of 40,088 symbol-bearing answers at confidence >= 0.6
naming no class (2026-09-17).
FIRST, AND BEFORE ANY RATE: the same query answers whether the zero-init landed at all. The priming runs on
`ApplicationStarted` because a measurement taken before OTel's MeterProvider subscribes is dropped; the unit test pins
the method's content and CANNOT see that ordering. So an EMPTY result for `{outcome="selfseed_class_skip"}` within
minutes of a deploy -- before anything can have skipped -- means the priming is still landing too early, not that the
rate is zero.
Re-check: `sum by (reason) (increase(sentinel_gemini_resolver_calls_total{outcome="selfseed_class_skip"}[7d]))` and the
same over `sentinel_gemini_fallback_calls_total`, both anchored to an instant from `date -u`, after 7 days of production
on a build carrying D-34. Close by replaying that 7-day distribution at 1-5m resolution and either wiring a rule with a
`for:` taken from the distribution, or recording that the rate is 0 and no rule is warranted.

**THE CoD PROMPT HAS NO UNIT FOR A NON-ENUM CURRENCY, SO THE MODEL WRITES USD: 1,087 OF D-35'S 1,284 WEEKLY UNIT
REJECTIONS ARE RUPEES.** [2026-09-21] `cod_json_schema_v1.json` enumerates USD, EUR, GBP, JPY, CNY and OTHER; the prompt
never says a currency outside the five is OTHER, and the model answers "Rs 2,714 crore" with `unit: USD`. D-35's unit half
now DROPS those rows (a value stored as 27.1 billion USD is off by ~83x), so the cost moved from wrong facts to lost
facts. Measured through the final `CoveValueCheck` over the 56,170 v2 rows extracted 2026-09-14..21: 1,284 unit
rejections, 1,087 of them naming rupees (the rest A$, C$, S$, CAD, CHF, £); every one of 40 unit rejections adjudicated
against its source text was a wrong currency. The fix is one prompt sentence, and the prompt is a scored coordinate
(D-29/D-30): it ships through a re-score, not a deploy. Re-check after deploy:
`sum(increase(sentinel_cove_check_total{check="value",outcome="fail",field="unit"}[7d]))`, instant from `date -u`.

**D-35'S VALUE CHECK STILL DROPS ~110 CORRECT FACTS A WEEK, IN CLASSES TOO SMALL TO HAVE BEEN DERIVED YET.** [2026-09-21]
The value leg drops only what it measures wrong, and a drop reaches every consumer as a missing fact (D-35). Adjudicated
against source text, 8 of 60 randomly sampled value-leg drops of the final rule were correct facts (Wilson 95% 6.9%-24.2%)
-- ~110 of the 826 a week, 57-200 -- against ~965 a week before round 2. The eight: a zero said in words ("no growth"),
"a hundred trillion" whose article is outside the copy slot, a compound number word ("twenty-four thousand six hundred"),
a 50/50 split read as 0.5, a typo in the article ("$1.1 ,illion"), a table value whose FIRST occurrence in the article is a
different cell (the text quote, and so the check, anchors on the first), a decimal comma before a percent ("4,959%"), and
highway chainage ("Km 154+500"). Development samples also found spoken ranges written with a comma ("maybe 50, 60
vessels") and "1/100th of 1%". Each is derivable; none is common. Re-check: re-run the adjudication on a fresh random
sample of `{check="value",outcome="fail",field="value"}` drops (Tempo `cove.rejected` events carry raw, value and unit).

**TWO MORE CoVe SYMBOL GATES STILL GROUND ON context_summary, WHICH IS NULL ON EVERY v2 ARTICLE.** [2026-09-21] D-35
moved the ResolutionWorker and AlphaVantage-sweep gates onto the article; two gates reading the same pair were left
alone on purpose, because each is governed elsewhere. (1) The re-extract sweep's resolve-only cascade
(`ReExtractResolutionAdapter`, fed `RawContent.ContextSummary` from `ReExtractBackgroundService`) -- D-21, D-28 and D-31
constrain what it may attach relative to live, so widening its grounding needs its own decision. (2) `POST
/admin/review/quarantine-hallucinated` (`AdminEndpoints.cs`) quarantines an Approved row whose symbol is absent from
`text_quote` or `context_summary`, so on a v2 row it would quarantine every symbol the one number's sentence does not
spell. Measured: 5,155 of 5,155 raw_content rows processed 2026-09-14..21 that hold a v2 observation have
`context_summary IS NULL` (SELECT-only). Re-check: the same count; close each gate by grounding it through
`CoveSymbolGate` or by recording why it must not.

**0.66% OF v2 ROWS CARRY A SubjectEntity THE ARTICLE NEVER SPELLS, AND NO TIER-1 CHECK COVERS IT.** [2026-09-21] D-15
grounds `source_entity` against the DOCUMENT's declared ENTs, not against the article, and the adapter does not consume
the verifier's ENT byte verdicts. Measured over the raw HTML files (tag-stripped, entity-decoded, case-insensitive;
boilerplate makes this a SUPERSET of the normalized text, so it UNDER-counts) for the 56,170 v2 rows of 2026-09-14..21:
369 subjects absent (0.66%), 179 of them Resolved. The sample is dominated by legal-name expansions of entities the
article names ("Crocs, Inc.", "JPMorgan Chase & Co.", "CrowdStrike Holdings, Inc.") that resolve correctly, so a
verbatim gate would mostly drop right attributions -- it was not shipped with D-35. Re-check by re-running the
measurement; decide on the Resolved subset whether the expansions come from the prompt's prepass grounding.

**D-35'S ZERO-INIT IS REGISTERED ON ApplicationStarted AND NOTHING PINS THE REGISTRATION.** [2026-09-21] The three tier-1
silence alerts read `sentinel_cove_check_total` and the two async outcome counters with `increase()`, which cannot see a
series born at 1; `SentinelMeter.PrimeCoveCheckSeries` exports all 12 tier-1 series, D-36's 8 tier-2 series and 4
input series at zero.
`CoveCheckZeroInitTests` pins the method's content and CANNOT see the Program.cs line that registers it, nor its
ordering after the MeterProvider subscribes (the D-34 priming has the same gap, entry above). Re-check within minutes
of a deploy, before extraction has run: `count(sentinel_cove_check_total)` must be 20 (12 tier-1 series + D-36's 8
`check="attachment"` ones, whose EXISTENCE SentinelCoveAttachmentCheckSilent requires) and
`count(sentinel_resolution_worker_processed_total{outcome=~"resolved|cove_rejected"})` 2; an empty result means the
priming is landing too early or is gone. The same gap covers `SentinelMeter.PrimeGeminiResolverTimeoutSeries` [2026-09-22]:
`GeminiResolverTimeoutFailSoftTests` pins its content, and nothing pins its Program.cs line. SentinelGeminiResolverTimingOut
reads it with `increase()`, so after a deploy `count(sentinel_gemini_resolver_timeouts_total)` must be 1. So does
`SentinelMeter.PrimeSecMasterTimeoutSeries` [2026-09-22]: `SecMasterClientTests` pins its content, SentinelSecMasterTimingOut
reads it with `increase()`, and after a deploy `count(sentinel_secmaster_timeouts_total)` must be 1.

**D-35'S VALUE LEG PASSES A SUBUNIT PRICE STORED IN THE MAIN UNIT: "515p" AS 515 GBP.** [2026-09-22] The value check
admits the raw's own digits whatever the unit, and derives cents, pence and paise as the main unit; so a raw naming a
subunit passes BOTH scaled (5.15 GBP, right) and unscaled (515 GBP, 100x wrong). #1103's review estimated ~12 a week.
Measured (SELECT-only) over every row extracted 2026-09-15T00:00Z..09-22T00:00Z, all pipelines, whose unit is GBP, USD
or EUR and whose `num_raw` is a number followed by p, pence, cent(s) or paise: 70 rows, 52 scaled, 18 stored the raw's
digits unscaled -- 14 pence as GBP (Diageo's 1,734p target as 1,734 GBP, NBP gas at 200.10 pence as 200.1 GBP), 2
"5 cents" as 5 USD, 2 "$2.16 cents" whose raw is itself contradictory. The fix: a raw that names a subunit next to a
main-currency unit does not admit its unscaled digits -- a D-35 rule change, so it ships with a replay of the value
check over the week, not as an edit. Re-check: the same SELECT, comparing `value` to the raw's digits.

**D-35'S VALUE-CHECK ERROR PATH IS WATCHED ONLY THROUGH THE 40% REJECT SHARE.** [2026-09-22] #1103's review residue
(S2, S8, S9), filed together because each is the case where `CoveValueCheck` THROWS on a block (`value/error`: the
block is dropped, the span goes Error). (1) No rule reads `error` alone: SentinelCoveValueRejectShareHigh counts it
with `fail`, so a defect throwing on, say, 15% of blocks beside the healthy ~4% of fails reads 19% and pages nobody.
(2) The adapter logs one Warning per thrown block, so the same systematic throw is also a log flood; demoting it waits
on (1) -- a visible signal is not demoted without a wired alert (CLAUDE.md OBSERVABILITY). (3) The review found a raw of
27+ digits overflows `decimal` in the check's arithmetic outside `TryMidpoint`'s OverflowException catch, so an
unrepresentable number reads as the check failing to run rather than as a fail; not reproduced here. Measured:
`sum(increase(sentinel_cove_check_total{check="value",outcome="error"}[8h]))` = 0 against ~4,240 checked facts at
2026-09-22T12:12Z, the series first exported by D-35's deploy that day. Close by an error-share rule whose threshold
comes from 7 days of that series, then decide the Warning.

**D-35'S SYMBOL CHECK READS A RAW FILE THAT IS MISSING WHILE ITS PATH IS SET AS A REJECTION.** [2026-09-22] #1103's
review residue S1. `ExtractionProcessor.LoadNormalizedContentAsync` returns empty text for a set path whose file is
gone, and `CoveSymbolGate.CheckAsync` then grounds on the quote alone and, on a v2 row, rejects: the row lands
NoResolution, which D-9 never approves. The gate's comment calls that a pruned file, but the pruner deletes the file
and then NULLS the path (`RawContentRepository.NullFilePathWithChildrenOlderThanAsync`), so a set path with no file
is an outage -- an unmounted volume, a restore -- or a crash between those two statements: an absent dependency,
which D-27 says is no verdict. Measured 2026-09-22: 0 of the 51,822 `sentinel.raw_content` rows with a
`raw_file_path` point at a missing file (SELECT-only, each path stat'ed on the host), so today's exposure is zero and
this is a latent misclassification. The fix is to read that case as `unavailable` (the row stays Pending and retries;
the pruner nulling the path at RawRetentionDays bounds the retry). Re-check: the same SELECT and stat.

**D-36 TIER 2 KEEPS A FIGURE ABOUT WHAT A FUND OR FUTURE TRACKS WHEN ITS VERDICT SAYS SO; WHETHER TO CLEAR IT IS THE USER'S CALL.** [2026-09-22]
AWAITING-DECISION. The attachment check has a verdict for a commodity, index, currency or market figure held on the
fund, ETF, trust or future that tracks it (`tracks_underlying`); by default it is KEPT and counted as `proxy`, because
clearing is final and the attachment may be wanted (D-36). Replayed on 1,120 publishable rows of 2026-09-15..22: 5.9%
of verdicts, ~117 rows a day; under the round-3 anchor rule (a proxy verdict on ANY stating sentence keeps the row)
1,008 of the 15,080 eligible rows of 2026-09-17T00:12Z..09-22T09:15Z, ~187 a day (census, SentinelCollector/DECISIONS.md
§D-36). THE DEFAULT HOLDS AT THE VERDICT, NOT THE FIGURE: a fund-tracks figure the model calls `about_other` is cleared
like any other -- 12 of that window's 5,042 clears, from 10 articles (~2.2 a day: INDY 7, THYP 2, VOE, XLY, SOXX), so
"proxies are kept" is true of the verdict only. Round 4's blind second labeller puts it at 15 (~2.8 a day, INDY 10): it
reads three INDY Nifty target and stop-loss clears (942559, 942560, 942573) as figures about the index INDY tracks,
where the census labelled them ambiguous (a target on the series, not a reading of it) -- a split, so the decision
sees 12 to 15 a day's worth, and whether a target ON the tracked index counts as fund-tracks is part of the policy
call. Labels: /opt/ai-inference/training-data/cove-tier2-census-20260922/census-labels.jsonl (`label` P, or
`labels.r4-reviewer-blind` P). On the 200 labelled rows the check said it 9 times: 2 consensus proxies, 2 consensus
WRONG (a bitcoin-ETF category allocation held on BITB, Dow futures held on the DJIA index), 5 labeller splits -- so
clearing proxies would remove 2 wrong attachments and 2 wanted ones per 200. The decision and 6 worked rows are on the
spot-check sheet (next entry). Close by the user's answer; flipping is one line in `CoveAttachmentGate.Disposition` plus its series
in `CoveChecks.Series` (both pinned). Re-check: `sum(increase(sentinel_cove_check_total{check="attachment",outcome="proxy"}[7d]))`.

**D-36 HAS NO §7.3 HUMAN SPOT-CHECK; EVERY LABEL ON TIER 2 IS A MODEL'S.** [2026-09-22] AWAITING-DECISION. The design's
acceptance line for the semantic tier, verbatim in SentinelCollector/DECISIONS.md §D-36: "Claim verifier agrees with
human spot-check on ≥85% of n=50 sample." None of #1105's four review rounds did one: the 200-row measurement used two
HF models, and every one of the census's 5,042 clear labels and round 4's 352 blind labels came from review agents
reading the article -- the persisted label file below carries no human label. What exists for the user is an 18-row
sheet of the FIRST cut's hard cases (the clears in question, the proxy decision with 6 worked rows, misses and splits),
written before the name rule -- its DX "Japan" row is now refused unjudged and its ~117-a-day proxy rate is the
round-1 replay's, the census reads ~187 -- so it is judgement on the hard cases, not the n=50 the design asks for.
The supervisor reports offering the user 10 examples on 2026-09-22; that offer is recorded nowhere in the repo. What
the design's check needs: 50 rows drawn from the census file (clears and keeps, so agreement covers both directions),
the user's right/wrong per row, and agreement with the shipped disposition, >= 85% on the point estimate (a 50-row
Wilson lower bound at 85% is ~73%). Close when D-36 records that agreement, or records the user's waiver. Data:
/opt/ai-inference/training-data/cove-tier2-census-20260922/ -- `census-labels.jsonl` to draw from,
`tier2-spotcheck-sheet-2026-09-22.md` the sheet as written.

**D-36 TIER 2'S RECALL IS UNMEASURED: EVERY CLEAR IS LABELLED, ALMOST NO KEPT ROW IS.** [2026-09-22] MEASUREMENT DEBT.
D-36's precision is a census (4,926 of 5,042 clears wrong attachments), but how many wrong attachments it KEEPS -- and
publishes to ThresholdEngine, the digest and the matrix -- has no estimate on the population it runs on. The census
window (15,080 eligible rows, 2026-09-17T00:12Z..09-22T09:15Z) kept 10,038: `pass` 8,191, `proxy` 1,008,
`value_mismatch` 524, `unanchored_figure` 163, `unidentifying_name` 86, `insufficient` 66. 364 of them carry a label,
and only 30 were drawn at RANDOM from these kept rows (round 4's blind control, `pass` rows only): 28 correct, 1 wrong
(Reliance Industries' bonds held on Reliance Worldwide, RLLWF), 1 ambiguous (an option premium held on the stock,
SDGR) -- 1 or 2 of 30 (Wilson upper bound 16.7% / 21.3%). The other 334 were labelled because some rule cleared them,
or in earlier rounds' draws under earlier rules, so they cannot give this rule's rate. Two signals from OTHER
populations: round 2's draw A, 3 of 40 random kept rows wrong
(7.5%, Wilson 2.6-19.9) under b303cfc8's rule; and the 2026-09-21 two-model validation, 57 of 150 uniform eligible
rows consensus-wrong (38.0%, Wilson 30.6-46.0; 27 split) over 2026-09-15..22 -- a window that includes the DX/U
volume the census excludes, so its gap to the census's 33.6% clear share sits inside its own interval and is not a
recall measurement. Scale: each 1% of the kept `pass` rows is ~82 rows, ~15 a day, of wrong attachments published.
PROPOSED MEASUREMENT (spend is the user's call; he is open to HF spend "if it improves outcomes"): a stratified draw
of kept rows by keep reason -- e.g. `pass` 400, `proxy` 150, `value_mismatch` 100, 50 each of the three small strata
-- labelled by two families at deepinfra (the standing pair, `.claude/skills/supervisor-mode/SKILL.md`
ORACLE_ROUTING, whose BUDGET block governs: bill printed before the first call, fail-closed cap, pre-flight probe,
proven on 2 known rows) from the same wider window D-36's two-model measurement used, recall population-weighted
over the strata. COST, from that measurement's own ledger ($1.089 for 200 rows by both models): DeepSeek $0.00123 and
Qwen $0.00417 a row, $0.0054 a row for the pair -- so 1,000 rows is ~$5.40, OVER a $5 cap, and the 800 above ~$4.32
fits it. Data: /opt/ai-inference/training-data/cove-tier2-census-20260922/census-labels.jsonl -- kept rows are
`outcome` other than `about_other`; the random 30 are those whose `r4_groups` holds `control`; the earlier validation's
150 rows are `tier2val-150-2026-09-21.jsonl` beside it. Close when D-36 carries a population-weighted recall with its
interval, or the user declines the spend.

**SECMASTER CATALOG NAMES THAT IDENTIFY NOTHING: D-36 NOW REFUSES TO JUDGE THEM, THE SOURCE FIX IS SECMASTER'S.**
[2026-09-22] Tier 2 compares a figure against the held instrument's `instruments.name`. On a name that identifies
nothing every correct figure read "about another entity" and was cleared, for good: the review measured DX (named
"Japan") 72 of 82 held-out rows cleared, 68 of them correct US Dollar Index futures figures, and GASDESW (named
"GASDESW") 15 of 18 correct diesel prices. D-36 now makes no comparison on such a name (`CatalogNameCheck`; kept,
counted `unidentifying_name`). THE TRADE BY CLASS, re-derived on today's population (census of the 109 rows the rule
refused 2026-09-17..22, judged without it, SentinelCollector/DECISIONS.md §D-36): on SERIES codes 16 of 29 would-be
clears were CORRECT (~3.0 a day kept from destruction) against 11 wrong kept (~2.1 a day); on LISTED self-seeds
(Equity/ETF) 0 of 11 correct and 10 wrong -- the rule shielded only wrong attachments there, so D-36 no longer applies
its symbol leg to them. Round 1's "~13.8 avoided against ~2.0 given up" was the DX era and is retired. Measured over all 29,127
instruments (SELECT-only): 259 names are the symbol verbatim (255 `entity_resolution:gemini` and 4 openfigi self-seeds
wrote the code as the name), 41 are a bare place (40 GeminiFallback rows minted from a country surface, all
is_active=false, and .LON "UK"), 7 are blank (see the blank-name entry above). Since 2026-09-17 the refusal hits ~20
eligible rows a day on 26 instruments (DFEDTARL 6.6/day, GASDESW 3.0/day), and DX and KC attach nothing (0 rows
since 2026-09-16). THE FIX IS AT THE SOURCE: the self-seed must write the series title or the company name, and the
place-named rows need their real names. DX's and KC's are NOT this entry's to change: their identity -- the ICE
futures row against the equity tickers of the same symbol -- is leg C of `docs/proposals/delete-wrong-mappings.md`,
parked on the user's decision; renaming either first would move the catalog under that decision. NAMES THAT MISLEAD
WITHOUT BEING EMPTY are not caught and pass wrong figures as `about_instrument`: NAQ.DEX is named "Nasdaq" (18.4
eligible rows a day since 09-17, and the review's random 170 held 3 Nasdaq INDEX figures kept on it), and FMCC held
Freddie Mac's PMMS mortgage-rate survey (0.6/day). Re-check: `SELECT count(*) FILTER (WHERE btrim(name) = btrim(symbol)),
count(*) FILTER (WHERE btrim(name) = '') FROM instruments` in atlas_secmaster (259 and 7), the place count by
`CatalogNameCheck`'s list, and after deploy `sum(increase(sentinel_cove_check_total{check="attachment",
outcome="unidentifying_name"}[7d]))` (~110 expected: ~16 a day on series codes and the place).

**RESOLVER SYMBOL COLLISIONS THAT D-36 CLEARS EVERY DAY.** [2026-09-22] Tier 2 removes these correctly, so no wrong
figure reaches a consumer, but each is a wrong attachment born upstream at a rate that makes it a source defect, not
residue. On the round-2 replay (400 uniform + 145 rows on the most-cleared instruments, one model labeller): WTI crude
held on Colgate-Palmolive (CL; 20 of 20 cleared, all wrong -- the futures root collides with the equity ticker), "U.S."
held on Unity (U, 24/24), Saudi Arabia on Spire (SR, 17/17), Indian banks on BSE Ltd (18/20), Porsche on Deere (DE,
17/17), foreign inflation on MICH (17/17), Madrid on the Spain ETF (EWP), Coca-Cola on its bottler (COKE). Eligible
rows a day since 2026-09-17 (SELECT-only): CL 52.0, BSE 19.4, SR 16.4, DE 13.6, MICH 12.6; U attaches nothing since
2026-09-16. Consequence while unfixed: every one costs a verdict call and leaves a NoResolution fact where a correct
attachment may have been available. Re-check after deploy: `SELECT "OriginalSymbol", count(*) FROM
sentinel.extracted_observations WHERE resolution_method = 'cove_about_other' AND extracted_at > now() - interval '7
days' GROUP BY 1 ORDER BY 2 DESC LIMIT 15`.
ON THE CENSUS (every eligible row 2026-09-17T00:12Z..09-22T09:15Z, every clear labelled; data below): CL 286 clears
of 287 eligible rows, all 286 wrong attachments, 253 naming crude, oil, WTI, Brent or a barrel in the description or
quote (~53 a day); T (AT&T) 83 of 83 cleared, all wrong, from 37 articles, 50 naming Japan or a Japanese company in the
stored text (trading houses -- Sumitomo, Mitsubishi, Mitsui, Itochu, Marubeni --, Japanese banks' returns, the Topix);
SI (Shoulder Innovations) 111 of 112; BSE 132 of 140; SR 92 of 95; DE 64 of 68; MICH 59 of 68. SHORT SYMBOLS carry
the class: over the 14,994 judged rows, the share cleared as a wrong attachment is 54.8% when the held symbol is 1-2
letters (1,117 of 2,038), 36.1% at 3 letters, 30.0% at 4 and 15.6% for longer, dotted or numeric symbols. That is the
measured correlation; the resolver path that attaches a 1-2 letter ticker the prose never names as one is untraced.
THE SIGNAL IS LOST, NOT CORRECTED: a clear leaves the fact NoResolution, so a WTI price cleared off Colgate reaches
neither Colgate nor any crude series. The fix is at the SOURCE, the resolver, and the census is its free gold set: the
4,926 clears labelled W (not the 89 A, 15 C and 12 P), each with its article, figure, quote and refuted instrument, are
a resolver-regression corpus once each carries its CORRECT answer -- an instrument proposed by an HF labeller and
verified against SecMaster by id, or "none" where the figure describes nothing catalogued (a correct NoResolution is a
passing case too: the resolver must still not pick CL). Cost basis, D-36's own ledger: $0.00123 a row for DeepSeek
alone, $0.0054 with the Qwen cross-check (~$6 or ~$27 over 4,926 rows). Data:
/opt/ai-inference/training-data/cove-tier2-census-20260922/census-labels.jsonl, `outcome` `about_other` with `label`
`W`; `symbol` and `instrument_name` are the refuted instrument.

**ATTACHMENTS REEXTRACT WRITES AFTER EXTRACTION GET NO D-36 CHECK.** [2026-09-22] Tier 2 judges attachments at
extraction; the ReExtract sweep (resolve-only, live) writes new ones later -- 397 eligible rows in the 7 days to
2026-09-22T07:00Z (~57 a day; its daily count 09-09..09-21 median 82, range 8-150), 132 recovered and 265 replaced. On
60 of them the check said `about_other` 7 times and all 7 were wrong (CAD/USD holding tariff figures, SP500 a strategy's
edge, BTC a Capital B purchase, GC Dakota Gold's margin): ~6.6 wrong attachments a day (3.3-12.6) reach the digest,
SecMaster's dedup and the auto-approver unchecked. They never reach ThresholdEngine or the sector roll-up: the only
publish seams are in `ExtractionProcessor`. Not gated in the D-36 fix round, deliberately: the candidate has to be
judged BEFORE `ApplyReExtraction`, so that a refusal reads as D-31's retain-on-miss rather than a clear of the held
instrument, which means a refused-candidate outcome in ReExtract's taxonomy and in SentinelReExtractGroundingNothing,
and an article read the resolve-only leg does not make today -- on a population whose tier-2 precision rests on these
7 rows. Re-check: the 7-day count of rows with `re_extracted_at` in the window, an instrument, a value and a quote,
whose instrument differs from `"OriginalInstrumentId"` or has none.

**D-36 TIER 2 READS AN ADR AND ITS HOME LISTING AS DIFFERENT SECURITIES.** [2026-09-22] The one consensus false clear
of 86 was a NOK 125 put strike on TGS held on TGSGY, TGS NOPEC's depositary receipt; a split clear was Carl Zeiss
Meditec's -4.22% (ETR:AFXG) held on CZMWY, its unsponsored ADR; review round 3's tail held one more, Maersk's
A-share move (CSE:MAERSKa) cleared from AMKBY, its B-share ADR (adjudicated ambiguous). The prompt's "an attribute of this instrument or its
issuing company" does not tell the model a receipt shares its issuer. A sentence would, and the prompt is measured
(`CoveAttachmentGateTests` and `CoveTier2PrecisionTests` pin it byte for byte), so the fix ships through a re-measurement
on a fresh labelled sample, not an edit. Re-check: `cove.cleared` events whose `cove.instrument` contains "ADR".

**D-36 CLEARS CORRECT FIGURES WHEN ITS WINDOW CUTS OFF THE ARTICLE'S OWN NAMING OF THE SUBJECT.** [2026-09-22]
AWAITING-DECISION. Review round 3's census of POWW (all 51 eligible rows 2026-09-17..22) found 13 clears and all 13
CORRECT -- ONE EVENT, never a rate: one SRK Capital semi-annual letter, published whole (raw_content 177236) and as an
excerpt (177363), whose heading "Outdoor Holding Company (POWW)" sits just before the 1,000-character evidence window,
while the paragraph the figures come from says "GunBroker" and "the company"; the model reads them as another
company's. They are 13 of the 15 correct clears in the census of all 5,042 clears of 2026-09-17..22.
The verdict here measures the model's knowledge of brand ownership, not the attachment. Candidate guard, measured
and NOT shipped: refuse a clear when the article tags the held ticker in a ticker context ("(POWW)", "NASDAQ:POWW")
outside every window the model saw -- on 1,360 labelled clears (strata, tail, census) it refuses 13 correct, 4 wrong
and 2 ambiguous, and the whole correct side is that ONE letter, so it is the user's call, not a fix-round edit; the
alternative is a window that always carries the article's naming of the ticker (a prompt change: re-measure).
Re-check: after deploy, `cove.cleared` events whose article span names the cleared symbol in parentheses; the census
query is `SELECT count(*) FROM sentinel.extracted_observations WHERE resolution_method = 'cove_about_other' AND
"OriginalSymbol" = 'POWW'` against POWW's eligible rows.

**TEXT_QUOTE ANCHORS ON THE RAW'S FIRST ORDINAL OCCURRENCE, WHICH CAN BE THE TAIL OF ANOTHER NUMBER.** [2026-09-22]
`DslToMergedExtractionAdapter.BuildNumTextQuote` snaps the persisted quote around the first `IndexOf(raw)`, so raw
"5%" lands inside "0.45%", "2 cents" inside "22 cents", "7%" inside "1.07%": 298 of the 14,997 eligible rows of
2026-09-17T00:12Z..09-22T08:18Z (2.0%) carry a quote in which the raw stands only as a fragment. D-36 no longer
reads the quote to judge (it anchors on the raw itself, `FigureAnchor`), but the quote is D-35's value-check sentence
and every `text_quote` reader's (digest, review UI, keyword consumers). The fix is at the source: anchor on the
first STANDALONE occurrence -- a D-35 input change, so it ships with a replay of the value check over the affected
rows, not inside a D-36 round. Re-check: count eligible rows whose `metadata->>'num_raw'` occurs in `text_quote` only
as a fragment (the rule is `FigureAnchor.StandaloneAt`).

**D-36'S FIGUREANCHOR COUNTS A NUMBER INSIDE A TIME OR A DATE AS A STATEMENT OF THE FIGURE.** [2026-09-22]
`FigureAnchor.StandaloneAt` refuses a raw that another digit, or a '.' or ',' and a digit, continues -- but ':' and
'/' end a number there, so "42" in "06:42", "10" in "10/11" and "7" in "7:00 AM" each count as a sentence stating the
figure. Measured by a port of `StandaloneAt` and the sentence snap over the 14,592 eligible rows of
2026-09-17T00:12Z..09-22T09:15Z whose article review round 3 cached, calling an occurrence a fragment when a digit
stands on the far side of the ':' or '/': 42 rows (0.29%) count at least one. On 32 no count crosses a threshold, on 9
the fragments push the figure past the cap of 8 (news-bullet pages stamped "09/19 06:42"), on 1 they are its only
statements. Of the 25 the round-3 reviewer's replay can re-decide, 17 keep their outcome, 2 keep under a different
verdict, 4 cannot be decided (more than 30 sentences), 1 would be judged and cleared (Prince Harry's age, held on CMG)
and 1 would lose its clear -- so the defect errs toward keeping. THE FIX IS NOT "':' and '/' are number-internal":
that 1 row is "Rs 1,24,664/8 grams", a gold price per 8 grams, which states its figure, as "(0.81/0.85)" (a ratio of
DHR's beta) and "2028/2029" (a range) state theirs in rows the same rule would rewrite. It has to recognise the TIME
(h:mm) and short-DATE (m/d) shapes, and it moves the prompt's sentence set, so it ships with a replay. Re-check: count
eligible rows whose `metadata->>'num_raw'` stands in the article between a digit and ':' or '/' followed by a digit.

**D-36 READS A RENAMED COMPANY AS ANOTHER COMPANY.** [2026-09-22] Review round 3 found one correct clear: "MicroStrategy
Inc (MSTR) +14.54%" held on MSTR, which the catalog names Strategy Inc. The census of all 5,042 clears of
2026-09-17..22 found one more of the class: Intuitive Machines' revenue cleared from IPAX, the catalog's ticker for
Intuitive Machines Inc, where the article says LUNR. The verdict compares the article's name and ticker against the
catalog's, and a rename the model does not know reads as a different company. Small (2 of 5,042) and bounded by how
many catalog rows lag a rename; the source fix is SecMaster keeping former names as
aliases the gate can show. Re-check: `cove.cleared` events whose `cove.instrument` differs from a former name in the
article.

**SentinelCoveAttachmentCheckAbsent FORGETS TIER 2 AFTER 7 DAYS ABSENT.** [2026-09-22] The rule pages when the
attachment series vanish after tier 2 was deployed; it knows tier 2 WAS deployed only through
`last_over_time(...[7d])`, so after a week of absence it resolves. The leg exists only so a monitoring deploy that
ships the rule ahead of the image stays silent. The step: once tier 2 has run in production for 7 days, drop the
lookback leg in a follow-up PR (absence then pages for as long as it lasts) and move its promtool group from
"forgotten after 7 days" to "still paging". Re-check: `count(sentinel_cove_check_total{check="attachment"})` is 8
in production for 7 consecutive days.

**D-36'S TWO SHARE ALERTS TAKE THEIR THRESHOLDS FROM A 40-ROW-PER-WINDOW REPLAY.** [2026-09-22] MEASUREMENT DEBT.
`SentinelCoveAttachmentClearShareHigh` (> 70%) was set over 28 six-hour windows of 40 replayed rows each: min 10%, p50
40%, max 60%, where 40 rows alone carry ~7.7pp of sampling noise. The shipped rule on the D-36 census (every eligible row
2026-09-17..22, this rule's own quantity, [6h] at 5m, the 200 floor) reads 34.0% overall and min 19.3% / p50 35.9% /
max 58.4% over 1,161 windows, so 70% holds on the population too; the census is a replay, so the live re-derivation
below is still owed. `SentinelCoveAttachmentCheckErrors` (> 20%) borrows the
news-signal classifier's error distribution (max 2.3% over 30d). Neither counter has live history. Re-derive after 7 days
of production on D-36, instant from `date -u`, 5m resolution:
`max_over_time((sum(increase(sentinel_cove_check_total{check="attachment",outcome="about_other"}[6h])) / sum(increase(sentinel_cove_check_total{check="attachment",outcome!~"error|unidentifying_name|unanchored_figure"}[6h])))[7d:5m])`
and the same for `outcome="error"` over [1h]; move each threshold to the distribution or record why not.

**D-36 CANNOT SEE TWO THINGS A WRONG ATTACHMENT STILL REACHES.** [2026-09-22] (1) Rows resolved onto an instrument with
NO value are not judged (the check is about a figure): 236 of 22,650 instrument-bearing v2 rows 2026-09-15..22 (1.0%,
SELECT-only); 0 lacked a quote and 1 was provenance-keyed (exempt, D-32). (2) SecMaster's dedup reader matches
`"Symbol"` OR `subject_entity` (`SentinelObservationSourceProvider`); a clear empties the symbol, but the model's
`subject_entity` stays as written, so a cleared row whose subject names the refuted company still joins that company's
alias. Re-check (2) after deploy: `SELECT count(*) FROM sentinel.extracted_observations WHERE resolution_method =
'cove_about_other' AND subject_entity IS NOT NULL` over 7 days, then sample whether those subjects are the refuted
instrument's aliases.

**A POST-MODEL TRANSIENT FAILURE RE-RUNS THE GPU EXTRACTION UP TO MaxRetries TIMES, THEN PARKS THE ARTICLE.**
[2026-09-22] In `ExtractionProcessor.ProcessSingleArticleAsync`'s article catch only `DependencyOutage.IsCircuitOpen`
reaches D-27's `ArticleExtractionSpend` gate. Every other `isTransient` failure (TimeoutException, a
TaskCanceledException over a TimeoutException, an HttpRequestException with no status, 5xx, 429 or 408) is requeued
with `RetryCount++` whatever the article has already spent, and at `MaxRetries` (3) it writes `ProcessingError` with
`ProcessedAt` null. D-27 verbatim: INTENT "A dependency being ABSENT is no verdict on the ARTICLE: such a failure takes
neither the permanent branch nor a retry-budget slot"; PRECOND "Requeue is free ONLY pre-spend". Its DETAIL scopes
"which failures are outages" to `IsCircuitOpen`, and D-32's comment at `DeterministicResolver.ResolveAsync` accepts the
transient re-extraction for SecMaster transport failures at "~11 documents a month". So this contradicts D-27's INTENT and
PRECOND while staying inside the scope its GUARD chose. Widening the gate is a D-27 amendment for a human, not an edit.
MEASURED 2026-09-22 (Tempo search by `span.raw_content_id`, SELECT-only): 11 articles hit the gemini-resolver client's
30s timeout after the model call. Every failed attempt carries exactly one `/v1/completions` HTTP span before the
timeout ended it, so that is 20 extra GPU extractions: 4 articles x 3 retries, parked with "The request was canceled due to
the configured HttpClient.Timeout of 30 seconds elapsing." (178397, 178412, 178414, 178415), plus 7 articles that
completed after 1-2 retries. Over the 30 days of `sentinel.raw_content` collected 2026-08-23..09-22T14:00Z (25,642 rows),
13 rows retried (23 retry attempts), 9 then completed and 4 parked, and all 4 parked are that incident's. The population
cannot contain a row reset by `POST /admin/reprocess`, which zeroes `retry_count`. The fix that filed this entry makes
the Gemini timeout and four SecMaster fail-soft timeouts a miss. Two post-spend TRANSIENT sources are still known:
`DslParserClient` rethrows its own TaskCanceledException from `/parse_json` and `/verify` (both called after the vLLM
call), and D-32's Rule 0 rethrows SecMaster transport failures. RECOVERY for the 4 parked rows: `POST /admin/reprocess`
with their ids once the fix is deployed. RE-CHECK: `SELECT count(*) FILTER (WHERE retry_count > 0), count(*) FILTER
(WHERE retry_count >= 3 AND processed_at IS NULL AND processing_error IS NOT NULL) FROM sentinel.raw_content WHERE
collected_at > now() - interval '30 days'`.

**A STALLED GEMINI RESOLVER NOW HOLDS AN EXTRACTION WORKER 30s PER ELIGIBLE OBSERVATION.** [2026-09-22] This is the
cost of making a resolver timeout a miss. Before, the first timeout ended the attempt (and bought a GPU re-extraction,
entry above). Now `V2ExtractionPipeline` resolves an article's observations one after another, and each Rule 2.5 call
waits out the client's 30s. The client has no breaker, by design (`DependencyInjection.cs`: "the MCP itself rate-limits
+ caches"). MEASURED from Tempo: of the 2,478 article attempts with at least one `GeminiResolver.Resolve` span between
2026-09-15T13:13Z and 09-22T12:48Z, the calls per attempt were p50 3, p90 10 and p99 28, with a max of at least 100 (the
search API returns at most 100 spans per span set, and one trace hit that cap). During a stall that holds a worker for
1.5, 5 or 14 minutes, and 50+ at the max. It is visible to SentinelGeminiResolverTimingOut and to
`sentinel_extraction_queue_depth` and `sentinel_extraction_frontier_lag_seconds`. Nothing bounds it. The options, none
chosen: a per-article Gemini time budget, or a client breaker opening on consecutive timeouts. A breaker is safe here
because the client's catch absorbs a BrokenCircuitException as a miss, so D-27's circuit-open branch never sees it.
RE-CHECK: count GeminiResolver.Resolve spans per trace over Tempo's 7 days.

**THE OCE FILTER THAT FAILED THE GEMINI LEG STILL SITS ON 4 FAIL-SOFT SITES THAT ARE UNREACHABLE IN THE DEPLOYED
CONFIGURATION.** [2026-09-22] Each of these catches is fail-soft by intent, but `when (ex is not OperationCanceledException)`
lets an HttpClient timeout, which is a TaskCanceledException, escape it:
`ToolAugmentedChatClient`'s tool-dispatch catch, `ChainOfDensity.ExtractEpistemicMarkersAsync`'s fallback to the
text path, `ExtractionProcessor.RunV2ShadowAsync`, and `ReExtractBackgroundService`'s loop guard. The first two are
registered only under `Extraction:Backend=VllmServer`, which is the appsettings DEFAULT while prod runs `VllmJson`, and
the second also needs `UseToolAugmentedEpistemicMarkers=true`. The shadow runs only on the v1 path, and prod's
`UseV2Pipeline=true` sends every `V2EnabledSources` source to v2 before it is reached. The loop guard is reached only
under `ReExtract:Mode=re-extract` (the appsettings default mode; prod sets `resolve-only`), where
`ReExtractResolutionAdapter`'s ticker leg calls `SecMasterClient.ResolveAsync` under a catch for HttpRequestException
alone and `ProcessRowAsync` does not wrap the adapter, so a SecMaster timeout would end the worker. Prod's resolve-only
leg wraps the adapter in a catch-all. Found by the sweep in the fix that filed this entry: 32 bare-filter sites in
`SentinelCollector/src`, 6 fixed and pinned, these 4 filed, and 22 out of the class because they rethrow, are reached by
no own-timeout OCE (the client beneath swallows it), or guard DB or gRPC work. The INVERSE shape was seen but
not measured: `SecMasterClient.SearchInstrumentsAsync`, `TrafilaturaClient.ExtractArticleAsync` and
`NtfyDigestNotifier` catch the CALLER's cancellation as well. FIX when a path goes live: the
`|| !cancellationToken.IsCancellationRequested` clause, with a test that drives HttpClient's real timeout.
RE-CHECK: `grep -rn "is not OperationCanceledException)" SentinelCollector/src --include=*.cs`.

**A SECMASTER TIMEOUT IS NOW A QUIET MISS: A PROMOTE TIMEOUT PINS THE RESOLUTION WORKER TO ITS OLDEST ROW, A
TIMED-OUT v2 ROW IS AN UNTAGGED NoResolution, AND A HANG CAN HOLD ONE v2 OBSERVATION FOR ~5 TIMEOUTS.** [2026-09-22]
The cost of making a SecMaster client timeout
(`SecMaster__TimeoutSeconds`, 180 in prod) a miss instead of a failed article or a stopped host. What sees it is
SentinelSecMasterTimingOut over `sentinel_secmaster_timeouts_total`; none of the three residues is fixed.
(1) HEAD OF QUEUE. `ResolutionWorker.ResolveOneAsync` counts a promote timeout (`secMaster.ResolveAsync`) and rethrows;
the loop guard aborts the whole batch and retries after `PollIntervalSeconds` (15). `GetPendingResolutionsAsync` orders
by `ExtractedAt`, and `IncrementResolutionAttempts` runs only on the null-instrument branch the exception skips, so
`MaxResolutionAttempts` (5) never applies: while that symbol's promote hangs, the same oldest row is looked up on
Finnhub and promoted again every ~195s and every row behind it waits. SentinelResolutionWorkerErrors cannot see it,
because no per-row outcome is recorded. NOT LIVE, measured 2026-09-22T13:43Z: `max(sentinel_resolution_worker_queue_depth)`
is 0, and `sentinel_resolution_worker_processed_total` rose by 0 over 7 days. An unresolved v2 row is NoResolution unless
D-27's `dependency_unavailable` leaves it Pending. None of Tempo's 458 sentinel-collector -> secmaster spans of 20s or more over
those 7 days was a promote (`/api/semantic/resolve`). Options, none chosen: count the attempt before the promote call,
or catch the timeout in `ResolveOneAsync` as the row's `error` outcome.
(2) UNTAGGED NoResolution. A timeout on the hybrid, exact-candidate, by-symbol, register or confirm leg leaves the v2
row NoResolution, under the same method as a real miss, and the ResolutionWorker never selects it. D-27's INTENT comment
at `V2ExtractionPipeline` keeps a row "the resolver could not even ASK about" Pending, but its guard scopes that to an
open breaker, and a hang never opens the SecMaster breaker: it counts HttpRequestException, 5xx and 408, not a timeout.
Widening `dependency_unavailable` to a timeout is a D-27 amendment for a human. The row IS revisited, though not soon
and not on the same cascade: the ReExtract resolve-only sweep (prod `Cohort=all`, `MinRowAgeDays=7`) re-resolves a
never-held row once, 7 or more days later, with no Gemini leg. Over the 7 days to 2026-09-22T13:43Z,
`sentinel_reextract_rows_processed_total` rose 194 `recovered` against 39,175 `still_null` (`increase()`: anything
counted after a restart's last scrape is lost, so both are floors). The AlphaVantage sweep adds 25 lookups a day, spread
over every NoResolution description. NOT LIVE: none of the ~291k sentinel-collector -> secmaster calls in Tempo's 7 days
reached 60s (max 50.04s).
(3) THROUGHPUT. During a SecMaster hang, `DeterministicResolver.ResolveCoreAsync` (Live) waits out the full 180s on
each SecMaster call it makes, one after another, and a timeout never opens the breaker ((2)), so nothing cuts the
chain short: Rule 2's subject hybrid call, Rule 2b's subject + description hybrid call (non-empty Description), and,
when Rule 2.5's Gemini call (a different service) returns a symbol, the by-symbol lookup and then the confirm call
(prod sets `GeminiAutoRegisterNewInstruments=true`; a timed-out confirm refuses the self-seed, so register is not
reached); then the exact-candidate leg's by-symbol lookup, once per id-less candidate carrying a symbol. That is 4 + N
timeouts, ~5 (15 min) with one candidate, and `V2ExtractionPipeline` resolves an article's observations one after
another. DERIVED FROM CODE, NOT MEASURED: the #1108 reviewer read it off the code and it is re-read here at
`90e1ebbe`; no hang has happened to measure (the NOT LIVE figures above).
ACCEPTED RESIDUE, left unfixed on #1108's review: `ResolutionWorker.ResolveOneAsync`'s
`catch (OperationCanceledException) when (!cancellationToken.IsCancellationRequested)` around the promote is what keeps a
shutdown's cancellation out of `sentinel_secmaster_timeouts_total`, and no test drives the caller's cancellation
through it: the one `ResolutionWorkerTests` case reading `SecMasterTimeoutCapture` asserts the timeout direction only.
Losing the filter costs at most one spurious count per shutdown, and SentinelSecMasterTimingOut needs 3 in 30m.
RE-CHECK: `sum(increase(sentinel_secmaster_timeouts_total[7d]))` after deploy. Above 0 means (1) or (2) has happened;
the Error spans at the timeout's duration name the endpoint, and `/api/semantic/resolve` is (1). For (3), count those
spans per trace.

**D-18 RECLASSIFIES 82 MISLABELLED ROWS BY AUTHORITY, WHICH COMPLETES T3 ITEM 3 AS WRITTEN; NOT YET DEPLOYED. 19 ROWS
NO AUTHORITY SETTLES REMAIN MISLABELLED.** Mechanism, authority and every id: `SecMaster/AGENT_README.md` D-18 and the
header of `SecMaster/src/Data/Migrations/20260917124624_ReclassifyFredSeriesLabelledEquity.cs`. Measured 2026-09-17 on
`atlas_secmaster`. The plan of record named 5 FredCollector primaries labelled Equity, and GSG and PALL -> ETF; all 7
are in the migration. The larger population is SecMaster's own Gemini self-seed, which labelled fred_series answers
Equity: 91 active LISTED rows claim exchange FRED, and FRED search confirms 74 of the ids as series. BTC-USD (Equity ->
Crypto, every OpenFIGI line Curncy/Crypto) is the eighty-second. Deploy it with the D-16 class-family refusal (#1061):
replaying that check over the gemini-resolver cache copied 2026-09-17T12:58Z, answers meeting the other family on these
82 rows fall from 123 in the prior 7 days to 0, and from 375 older ones to 1.

WHAT REMAINS.
- 16 ids for which FredCollector search returns no series at all: CPIW, GOLDPMGBD228NLBM, HHDEBT, MPMIUSSA, NAHB, NAPM,
  NATURALR, PCEPILFE_PC1, PRS85000001, RSTAR, SPRSTOC, US30Y, USMCE, USQCEWEMP, WCESTUS1, WCSSTUS1. Each is still Equity,
  and its disposition is undecided (quarantine as unconfirmed, or a class from another authority). Gemini still answers
  three of them as fred_series (NAHB 6, NAPM 3, WCSSTUS1 1 in the 7 days to 12:40Z), so D-16 refuses answers for ids
  FRED does not serve.
- WEAT (`TEUCRIUM WHEAT FUND`, Commodity, 18 cached ETF answers): no EDGAR filer holds its ticker, and the only Teucrium
  filer is Teucrium Commodity Trust (CIK 0001471824) under ticker BTCK, so the plan's authority does not hold. OpenFIGI
  TICKER WEAT/US answers securityType ETP, which would settle it if that authority is accepted.
- `^TNX` (Equity, exchange CBOE): OpenFIGI has no line for `^TNX` or `TNX` as an Index; TICKER TNX is TENX PROTOCOLS INC
  and TENAX INTERNATIONAL SPA. `EURUSD` (Equity, no exchange): no Curncy or US line; TICKER EURUSD is EUROPEAN LITHIUM
  LTD (exchange ER). Both wait for an authority that names the index or the pair.
- OWLT is Owlet Inc and correctly Equity; its exchange `NYSE | NASDAQ | AMEX | OTC | FRED | null` is why it matches the
  FRED query below.
- A1 does not move: 280 before and 280 after, because the SentinelCollector mapping each FredCollector primary also holds
  enters it as the FredCollector mapping leaves (T3 item 4).

Re-check (psql is SELECT-only; `atlas_secmaster`). Deployed: `SELECT "MigrationId" FROM "__EFMigrationsHistory" WHERE
"MigrationId" LIKE '%ReclassifyFredSeries%';` -> 1 row. Rows: `SELECT count(*) FROM instruments WHERE is_active AND
lower(asset_class) IN ('equity','etf','stock') AND exchange ILIKE '%FRED%';` -> 91 on 2026-09-17T12:49Z, 17 after the
deploy (the 16 and OWLT); `SELECT count(DISTINCT i.id) FROM instruments i JOIN source_mappings sm ON sm.instrument_id =
i.id WHERE i.is_active AND lower(i.asset_class) IN ('equity','etf','stock') AND sm.collector = 'FredCollector' AND
sm.is_active;` -> 5, then 0; `SELECT symbol, asset_class FROM instruments WHERE is_active AND symbol IN ('GSG', 'PALL',
'WEAT', 'BTC-USD', '^TNX', 'EURUSD') ORDER BY 1;` -> Commodity x3 and Equity x3 on 2026-09-17T12:57Z, then ETF for GSG
and PALL and Crypto for BTC-USD. Close this entry when the 16, WEAT, ^TNX and EURUSD each have a disposition.

**WHILE THE LLM CIRCUIT IS OPEN, RESOLVE-LOCAL RETURNS HYPOTHESIS `RAG`, READ OUT OF SECMASTER'S OWN FALLBACK ANSWER.**
`RagService.QueryAsync`'s `BrokenCircuitException` arm answers `"RAG temporarily unavailable."`, and
`HybridResolutionService` hands that answer to `TickerHypothesisExtractor.Extract`, whose regex takes `RAG` as the first
ticker-shaped token (not a stop word). Measured in `ResolveLocalDegradedTests.should_flag_rag_degraded_when_the_llm_circuit_is_open`
(2026-09-17): the body is `{"method":"RagSynthesis","instrumentId":null,"symbol":null,"hypothesis":"RAG",...,"degraded":true}`.
LATENT, not harmless: `SELECT count(*) FROM instruments WHERE upper(symbol) = 'RAG';` on `atlas_secmaster` -> 0. And no
circuit opened in the week to 2026-09-17T02:44Z: `sum(count_over_time(secmaster_rag_abstain_total[7d]))` returns no
series for ANY reason, while `sum(count_over_time(secmaster_rag_degraded_total{reason="timeout"}[7d]))` returns 7,551
samples whose earliest is 2026-09-10T02:44Z, so the scrape covered the whole window and the absence is real, not a gap
(the abstain counter is not zero-initialised, so it has no series until its first increment). The consequence arrives with either a catalog row named `RAG`
(SentinelCollector's hypothesis leg looks the symbol up and attaches) or the Pending path confirming `RAG` upstream.
Since D-15 the same body carries `degraded: true`, so a caller that refuses degraded bodies never reads it; one that
reads `hypothesis` first does. Fix at the source (an answer the extractor cannot read, or null), not with a stop word.

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

`source_entity` AND `subject_entity` ARE THE SAME FIELD ON THIS PATH. `DslToMergedExtractionAdapter.cs:457` sets
`perRowSubject = sourceEntity` whenever the slot is non-blank; of 20,640 rows since 2026-09-01 where BOTH columns
are non-blank, ZERO differ. So a country in `source_entity` IS the string `DeterministicResolver` Rule 2 queries.
Re-check, and silence on the second column is the pass:
`SELECT count(*), count(*) FILTER (WHERE source_entity IS DISTINCT FROM subject_entity) FROM
sentinel.extracted_observations WHERE extracted_at >= TIMESTAMPTZ '2026-09-01' AND
coalesce(trim(source_entity),'')<>'' AND coalesce(trim(subject_entity),'')<>'';`

DOWNSTREAM. A country ENT is excluded from the Rule 1 shortlist by `NonInstrumentEntTypes`
(`DslToMergedExtractionAdapter.cs:110-127`, `"country"` at `:115`), so the row falls to Rule 2, which hands the RAW
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

BLANK IS NOT SAFE EITHER. A blank slot falls back to the DOCUMENT-LEDE ENT (`DslToMergedExtractionAdapter.cs:457-459`):
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
prepass and on the paid-Gemini legs (`DeterministicResolver.cs:897`, `GeminiSymbolFallbackService.cs:85`), while the
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
caveat-(2) false positive on the resolution path. Seam B: `DslToMergedExtractionAdapter.cs:557`, where `SubjectEntity`
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
(`SentinelCollector/src/Workers/ExtractionProcessor.cs:890/:918/:934` v1, `:2321/:2331/:2348` v2). **`Symbol` is not in the predicate**, so
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
(`SentinelCollector/src/Entities/ExtractedObservation.cs:309`) and `QuarantineInPlace()` (`:465`) assign them
UNCONDITIONALLY, so `"OriginalInstrumentId" IS NOT NULL` means "held an instrument at the LATEST quarantine, or at
the first re-extract if never quarantined after it" -- NOT "at extraction". CONFOUND, not cause -- the April trigger
remains unestablished.

**The bulk quarantine is NOT the forward-blocking mechanism.** A quarantine stamped at exactly
`2026-04-24 00:18:52.867997+00` hit 15,894 rows across 1,054 distinct `"OriginalSymbol"` values (re-verified
2026-08-19) -- it hit BDIY too, and BDIY recovered. 272 of the 275 Challenger rows had ALREADY published before being
quarantined, so it is retroactive on already-sent data, not a forward block. Its origin is INFERRED -- do not repeat
this search expecting to close it: `ExtractedObservation.Quarantine()` (`:308`) has no production call site in git
history, and `QuarantineInPlace()` (`:475`) is called only from `SentinelCollector/src/Endpoints/AdminEndpoints.cs:1585`,
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
`resolution` or `answer` field (`SecMaster/src/Endpoints/SemanticSearchEndpoints.cs:423`); `secmaster` has `curl`,
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
written only by `Quarantine()` (`SentinelCollector/src/Entities/ExtractedObservation.cs:311`), `ApplyReExtraction()`
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
(2) No test on the `!candidate.InstrumentId.HasValue &&` exemption (`DeterministicResolver.cs:520`; dead today, 0
    non-null candidate ids across 503,446 rows, and it activates the day SecMaster's search endpoint returns ids -- a
    server-side change with no compile-time signal here) nor on the hybrid leg's `ResolutionConfidence` contract
    (`IDeterministicResolver.cs:100-116`: a null-instrument `llm_candidate_hybrid` row still carries the LLM's pick
    confidence, and the >= 0.8f event-publish predicate reads that field).
(3) `SubjectNameNormalizer.SharedTokenCount` scores 0 for all four bad pairs above and is reachable from Rule 1 on
    the RagSynthesis materialisation branch (`DeterministicResolver.cs:788`), but the whole guard sits behind
    `Extraction__GuardsEnabled=false` (`/opt/ai-inference/compose.yaml:1269`), so it is inert; deciding that flag's
    fate is the prerequisite, and a third call behind the same disabled flag would read as protection that does not
    exist.

**SECMASTER D-16 DROPS GEMINI'S `fred_series` ANSWERS FOR THE 16 FRED-LABELLED IDS D-18 LEAVES `Equity`: 10 IN 7 DAYS.**
D-18 relabels 74 of the 91 LISTED rows claiming exchange FRED and leaves 16 ids FRED search does not return (the list
and their undecided disposition: the D-18 entry above) plus OWLT, which is Owlet Inc and correctly Equity. Until an id
has a disposition, a Gemini `fred_series` answer naming it meets a LISTED row and D-16 drops it as
`class_conflict_skip`; whether that drop is right is the disposition's question. Measured by replaying D-16's
self-seed decision over a read-only copy of `/opt/ai-inference/gemini-resolver-cache.db` taken 2026-09-17T13:18Z: 10 of
the 3,385 answers written in the 7 days to 13:43Z (NAHB 6, NAPM 3, WCSSTUS1 1), 78 of all 40,148. An upper bound: the
self-seed meets a Gemini answer only after every cheaper tier missed. Close in the PR that disposes of the last id.
Re-check (psql is SELECT-only; `atlas_secmaster`): `SELECT count(*) FROM instruments WHERE is_active AND
lower(asset_class) IN ('equity', 'etf', 'stock') AND symbol IN ('CPIW', 'GOLDPMGBD228NLBM', 'HHDEBT', 'MPMIUSSA',
'NAHB', 'NAPM', 'NATURALR', 'PCEPILFE_PC1', 'PRS85000001', 'RSTAR', 'SPRSTOC', 'US30Y', 'USMCE', 'USQCEWEMP',
'WCESTUS1', 'WCSSTUS1');` -> 16 on 2026-09-17, before and after the D-18 deploy, one fewer per disposition. It names the
ids because the exchange-FRED filter also matches OWLT.

**Integration test databases on the SHARED timescaledb outlive their runs, and each one holds a TimescaleDB
background-worker slot.** [2026-09-17] Integration fixtures now use per-worktree names
(`<base>_<ATLAS_WORKTREE_ID>_<service>`, `IntegrationDatabaseName`) and drop their database on dispose. Three
things that change does not cover:
- A KILLED RUN LEAKS ITS DATABASES. A TERM to compile.sh's process group skips xUnit disposal. Measured at
  13:19:01Z with FredCollector `--integration`: one `atlas_integration_test_a9f3f277dc97_fred_collector` was left,
  and PR #1065's review saw three `*_53d9b29d969f_fred_collector` after a TERM. A run in the same worktree reclaims
  them: the bases drop an existing name before creating it, and the per-test fixtures reuse the name and drop it
  after each test. The measured leak was gone after one rerun (per-worktree count 0 at 13:24Z). A TERM sent after
  the Database collection had already finished left nothing, so the size of the leak depends on when the kill
  lands. `scripts/devcontainer-owner.sh`'s reaper removes containers, networks, volumes and images, never
  databases (`grep -c 'DROP DATABASE\|psql\|pg_database' scripts/devcontainer-owner.sh` = 0). So a killed run in a
  worktree that is then deleted leaks for good.
- THE NINE PRE-FIX FIXED-NAME DATABASES ARE STILL THERE, and no code references them any more:
  atlas_integration_test, atlas_api_integration_test, atlas_grpc_integration_test, atlas_macro_idempotency_test,
  atlas_matrix_idempotency_test, atlas_latest_cells_query_test, atlas_secmaster_integration_test,
  finnhub_integration_test, ofr_integration_test (9 present, 2026-09-17T13:11Z).
- EACH DATABASE HOLDS A WORKER SLOT. Every database with the extension gets its own TimescaleDB scheduler. At rest
  (2026-09-17T13:11Z) `timescaledb.max_background_workers` was 16, with 14 schedulers running, 9 of them on those
  orphans. All 127 "out of background workers" warnings in the 24h to 13:11Z were `failed to launch job 3 "Job
  History Log Retention Policy [3]"`. There were 64 at the four 6-hourly boundaries (18:00, 00:00, 06:00, 12:00Z),
  when every database's job 3 fires at once, and 63 between 12:06 and 13:03Z, while integration runs were live.
  A run adds one scheduler per test database it holds open. None of the four production databases checked carries
  a user policy job (job_id >= 1000: 0 in atlas_data, atlas_secmaster, calendar_data and financial_news), so job 3
  is the only job refused so far. A compression or retention policy added later would compete for the same two
  free slots.
DECISION NEEDED, and no agent may make it: nothing here may be dropped by hand. psql is SELECT-only (CLAUDE.md
DATABASE), and a pattern reaper was ruled out when the per-worktree names shipped. Freeing the slots needs either a
human decision to drop the orphans, or an app-owned path, such as a fixture-side sweep of its own worktree's names
at start, or raising `max_background_workers` through IaC.
Re-check (SELECT only):
  `SELECT count(*) FROM pg_database WHERE datname ~ '_[0-9a-f]{12}_[a-z_]+$';`   -- per-worktree names (0 at 13:11Z)
  `SELECT count(*) FROM pg_database WHERE datname IN ('atlas_integration_test', 'atlas_api_integration_test',
     'atlas_grpc_integration_test', 'atlas_macro_idempotency_test', 'atlas_matrix_idempotency_test',
     'atlas_latest_cells_query_test', 'atlas_secmaster_integration_test', 'finnhub_integration_test',
     'ofr_integration_test');`                                                     -- fixed names (9)
  `SELECT (SELECT setting FROM pg_settings WHERE name = 'timescaledb.max_background_workers') AS max_workers,
     count(*) FILTER (WHERE backend_type = 'TimescaleDB Background Worker Scheduler') AS schedulers,
     count(*) FILTER (WHERE backend_type = 'TimescaleDB Background Worker Scheduler'
       AND datname ~ '(_integration_test|_idempotency_test|_query_test|_[0-9a-f]{12}_[a-z_]+)$') AS test_db_schedulers
   FROM pg_stat_activity;`                                                         -- 16 | 14 | 9 at rest
  `SELECT job_id, proc_name, schedule_interval FROM timescaledb_information.jobs;` per database
  `sudo nerdctl logs --since 24h timescaledb 2>&1 | grep -c 'out of background workers'`   -- 127

**SecMaster's `EmbeddingCache` keys on `Trim().ToLowerInvariant()`, but bge-m3 is case-sensitive, so a query's
vector neighbours depend on which spelling of it the process met first.** First occurrence. `EmbeddingCache.Normalize`
(`SecMaster/src/Services/EmbeddingCache.cs`) serves the cached vector of whichever of `NASDAQ` / `Nasdaq` arrived
first to both. Measured 2026-09-17T11:12Z on llama-cpu-embed: cosine("Nasdaq", "NASDAQ") = 0.7685 (control:
cosine("Nasdaq", "Nasdaq") = 1.0). The recall sample's 50 articles alone carry six such pairs (`Core CPI` / `core CPI`,
`S&P 500 Index` / `S&P 500 index`, ...). Observed consequence: GET /api/semantic/candidates run in a fresh process and
the same lists rebuilt from production's long-lived process agreed on 298 of 300 lists; both differences are one row,
at position 16 of its two k=20 lists that read the article's names, on an article carrying `Nasdaq` while an earlier
article carried `NASDAQ`. The mechanism is shown;
that this pair caused that row is inferred, not isolated. Impact on attachments is unmeasured. Re-check: embed both
spellings as above; the defect is closed when the cache key preserves case or the embedder input is case-folded too.

**The MeterListener capture helpers in the SecMaster unit tests filter on instrument NAME only, so they
aggregate measurements from every test running concurrently in the process, and a test that takes long enough to
overlap a sibling makes that sibling fail on a doubled count.** Measured 2026-09-20: adding one test to
`EdgarClientTests` that slept ~2s (a 429 path missing `RetryAfterFirst = TimeSpan.Zero`) turned
`OpenFigiClientTests.should_record_ok_counter_on_success` RED with "Expected capture.SumFor(\"ok\") to be 1L, but
found 2L" — a test that had passed on the two immediately preceding runs of the same suite. The slow test was fixed,
so the suite is green, and THE RACE IS STILL THERE: nothing scopes a capture to its own test. `MeterCapture` and
`EdgarHistogramCapture` both key solely on `instrument.Name == instrumentName`
(SecMaster/tests/Services/OpenFigiClientTests.cs, EdgarClientTests.cs). Consequence: any future test that emits on an
already-captured instrument, or merely runs slowly beside one, produces a failure whose message points at the
INNOCENT test — and the reflex fix is to re-run until green, which is how a real regression gets waved through.
The blast radius is wider than the one pair that failed: the same name-only filter is used by capture helpers in
EmbeddingServiceTests, EntityResolutionServiceTests, RagServiceTests, RegistrationServiceFrequencyTests and
MetricWarmupHostedServiceTests too. The fix is a per-test discriminator on the captured measurements (a unique tag
value, or an xUnit collection that serialises the classes sharing an instrument).
Re-check: `git grep -c 'instrument.Name == instrumentName' -- SecMaster/tests` — 10 hits across 7 files on
2026-09-20; the entry is closed when each capture also filters on something only its own test emits.

**Two EDGAR rules now ship, and the blind spot that survives BOTH is per-row staleness drift: cycles that run at full
volume while individual CIKs quietly stop resolving.** Measured 2026-09-20. `SecMasterEdgarOutage` (error ratio over
1h) observes only while a cycle runs, which is 89 of the 10,080 minutes in a week — a 0.9% duty cycle — and self-clears
~1h after a cycle even if EDGAR is still down. `SecMasterEdgarCycleNotRefreshing` (filers ingested over 8d < 1000)
closes the two shapes that leaves open, a bootstrap-only cycle (20 filers, zero errors) and a cycle that never ran,
and is the rule that actually bounds detection latency at ~8 days. Neither sees the third shape: 121 of the 4,914
`edgar_filers` rows are already older than 7 days, and that number can grow without either rule moving, because a cycle
refreshing 4,790 of 4,914 rows clears the 1000 floor comfortably. Consequence: classification serves the last good
SIC->NAICS for a drifting tail of instruments and nothing erupts. The fix is the gauge neither rule could use —
`count(*) where fetched_at > now() - interval '8 days'` exported at end of cycle, alerting when the FRESH fraction
falls rather than when the absolute count does; Prometheus cannot reach Postgres here, so it has to be emitted by
SecMaster. Note the coupling before touching those 121 rows: `ShouldSkipCycleAsync` skips only when NO row is stale, so
they are what guarantees the cycle never skips; purging them activates the skip path, allowing a ~14d gap that would
false-fire the 8d rule. Re-check: `select count(*) filter (where fetched_at < now() - interval '7 days'), count(*) from
edgar_filers;` — 121 of 4,914 on 2026-09-20; a rising first number with both alerts green is this entry coming true.

**`backfill_unresolved_rate_high` has never detected a fault: on main it fired on every run, and D-17's in-scope
rate stays under its 0.50 threshold through an EDGAR outage, then crosses it on catalog growth alone.** Measured
2026-09-17 (Loki `{service_name="SecMaster"} |= "backfill_unresolved_rate_high"` over 30d; atlas_secmaster SELECT-only).
Main's form (unresolved over every active row, threshold 0.20) fired on all 7 runs in 30 days at 84.06-84.20%; with
every EDGAR lookup empty it would read 84.7% (24,299 of 28,702), so it was a constant. D-17's form (unresolved over
EDGAR-owned rows, `InstrumentClassificationBackfillOptions.UnresolvedAlertThreshold` 0.50): steady state 3,768 of 8,180
(46.1%); every EDGAR lookup empty 3,938 of 8,180 (48.1%); `ListingScope`'s venue arm deleted 4,253 of 8,699 (48.9%).
Neither fault crosses 0.50, and no threshold separates either from the steady state. Growth does: in-scope rows by
creation month are unresolved Jul 749 of 1,355, Aug 932 of 1,613, Sep (to the 17th) 434 of 650, 58.5% pooled, so about
3,800 more rows at that rate cross 0.50, some 80 days at the 46 rows/day those months averaged. A cohort's rate falls as
its rows pick up a Finnhub sector (May 381 of 1,364, Jun 977 of 2,176), so that is an estimate, not a date. Consequence:
the first firing reads as an incident and is growth. Kept, not deleted (CLAUDE.md OBSERVABILITY: nobody decided to
demote it); the fix is a signal an outage moves, such as the share of in-scope rows whose filer lookup hits. PARTIAL,
and it does not close this entry: `SecMasterEdgarOutage` (2026-09-20) now watches the EDGAR HTTP boundary directly,
which is the outage detector this entry says is missing -- but it observes only while the weekly cycle is running, and
`backfill_unresolved_rate_high` is still a constant that growth will cross. Re-check,
per creation month (drop the GROUP BY for the whole-population rate):
  `select to_char(created_at,'YYYY-MM'), count(*) filter (where classification_source is null and not exists (select 1 from instrument_sector_overrides o where o.instrument_id = i.id and o.is_active) and not exists (select 1 from edgar_filers f where upper(f.ticker) = upper(trim(i.symbol)) and coalesce(f.derived_naics_code,'') <> '')), count(*) from instruments i where is_active and asset_class in ('Equity','ETF','Stock') and (strpos(symbol,'.') = 0 or symbol ~ '^[A-Za-z]+\.[ABC]$') and (exchange is null or exchange !~ '^[A-Z]{2}$' or exchange = 'US') and not exists (select 1 from source_mappings m where m.instrument_id = i.id and m.is_active and m.is_primary and m.collector in ('FredCollector','FRED','BLS','OFR','OfrCollector')) group by 1 order by 1;`

**The D-10 name repair still rewrites the whole `instruments.metadata` column from a copy loaded before its OpenFIGI
and Finnhub calls, so an applied run erases any metadata an enrichment cycle saved meanwhile.** Measured 2026-09-17,
atlas_secmaster SELECT-only: 81 active rows in its population lack `name_repair_completed_at`, and 80 of them also sit
in the OpenFIGI pool (listed, `figi` NULL). `CatalogNameRepairService.Stamp` copies `Metadata` and reassigns it, and the
pass saves once after every API call. D-17 moved both enrichers to `InstrumentMetadataMerge`, which closes the race
between those two and not this one. Not a one-line swap: the pass is one save behind a dry-run gate, and a merge is its
own statement. Consequence: on those 80 rows
an OpenFIGI enrichment landing during an applied run loses its composite and share-class FIGI for good, because the
`figi` column survives. A second whole-column writer, also unfixed: `InstrumentRepository.UpdateAsync` calls
`Instruments.Update`, which marks every column modified, metadata included. Its window is one request, not a pass:
each caller loads the row and saves it with no external call between (`PUT /api/instruments/{id}`, `RegistrationService`'s
sector dual-write onto a sectorless row, `FredCatalogReconciliationService.ReactivateAsync` on an inactive row, which
neither enrichment pool selects), so a key is lost only when an enrichment commits inside those milliseconds. Measured
2026-09-17 over 7 days (Prometheus): 0 `PUT /api/instruments/{id}` requests (`http_server_request_duration_seconds_count`),
and no `secmaster_sector_name_mapping_total{source="registration"}` series. Re-check:
  `select count(*) filter (where not metadata ? 'name_repair_completed_at' and asset_class in ('Equity','ETF','Stock') and figi is null) from instruments where is_active and lower(discovery_source) = 'entity_resolution:gemini' and created_at < '2026-08-07T12:00:00Z';`

**Merged SecMaster PRs sat undeployed for 10 days, and a later deploy shipped them unannounced, so the D-13
post-deploy acceptance first charged #1030's latency cost to D-13.** First occurrence. #1029, #1031 and #1030
merged 2026-09-06 (15:56Z-17:53Z). The `secmaster` image running until 2026-09-16 was built 2026-08-25T00:53Z, one
minute after `8bac67a7` (#987) merged, so all three first reached production in the D-13 (#1049) deploy at
2026-09-16T23:01Z. Neither #1049's PR body nor #1051's diff names #1029-#1031 or ef_search. #1030 raised `hnsw.ef_search` from 40 to 400. The acceptance saw
vector-SQL latency rise and first read it as the new D-13 join. A live SELECT-only A/B then put it on ef_search: p50
1.28 / 1.38 ms without / with the join at 40, and 6.91 / 7.02 ms at 400. The figures are in `docs/RELEASES.md`
`d13-retired-scope-done`; the keep-or-tune decision it left open was settled at 200 on 2026-09-17
(`SemanticSearchOptions.HnswEfSearch`). The harness readings from that deploy mix the same two
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
`deployment/ansible/playbooks/deploy.yml:1513-1514` ("recreates every transitive dep unconditionally (no config-hash
skip)", the cascade incident). The two runs agree with it. Class B, not D: the harm is a shared dependency restarted
without notice, with its failures landing in other services' traces.
Re-check: `grep -nE 'tags: \[([^]]*, )?secmaster(,|\])' deployment/ansible/playbooks/deploy.yml` lists both
`llama-cpu-*` blocks while this holds. After a scoped `secmaster` deploy,
`sudo nerdctl container inspect secmaster llama-cpu-rag llama-cpu-embed --format '{{.Name}} {{.Created}}'` shows all
three within a minute (`container` is load-bearing: bare `inspect` returns the IMAGE).

**`HybridResolutionService.NameAppearsInContext` is a literal substring test of the catalog NAME in the context string,
and it gates every live hybrid tier -- so a good catalog hit returns NONE unless the instrument's catalog name appears
verbatim in the quote.** First occurrence, recorded so the next NONE-with-a-good-catalog-hit is recognisable; NOT a
defect this epic fixes. `SecMaster/src/Services/HybridResolutionService.cs:584-593` is
`context.Contains(name, StringComparison.OrdinalIgnoreCase)` (`:592`), applied in `ResolveLocalAsync` to the ExactSql
result (`:76`), the FuzzySql top hit (`:115`) and every Vector candidate (`:197`); an empty context passes everything
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
closed (SentinelCollector D-27, `SentinelCollector/AGENT_README.md` §D-27) and both known cohorts are disposed of: 55
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
dependency outage.** `TryDispatchQualitativeAsync`'s extract-stage catch (`SentinelCollector/src/Workers/ExtractionProcessor.cs:2811`)
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
(`DeterministicResolver.cs:446-450`), not what Rule 1 received; the input is visible only in
`sentinel_resolver_rule1_input_confidence` (`SentinelMeter.cs:1805`, from #963). Every observation of it sits at
exactly 0.850 = `DslPreselectionConfidence`, a hardcoded constant, so the `< 0.7` gate can never trip: an absent
`below_threshold` series on `sentinel_resolver_rule1_decision_total` (`SentinelMeter.cs:1786`) is a property of the
constant, not evidence about the data, and `bucket{le="0.7"}` reads 0 indefinitely.
Not to be re-derived: the `ExtractionSchemaV2 required[]` hypothesis was DISPROVEN by probing vLLM with the shipped
schema, which emitted `resolution_confidence` non-null 5/5.

**A third histogram still carries the SDK default buckets a [0,1] value cannot use.**
`sentinel_chunk_extraction_dedup_ratio` (`SentinelCollector/src/Telemetry/SentinelMeter.cs:347`, unit `{ratio}`) has
no `AddView`, so it keeps the SDK boundaries `[0, 5, 10, 25, ...]` and every observation of a `1 - post/pre` fraction
would land in `le=5.0` — the identical collapse #963 fixed on `sentinel_dsl_adapter_resolution_confidence` and
`sentinel_resolver_rule1_input_confidence`. Nothing is misled TODAY: measured 2026-08-15 UTC, the metric has NO series
in prod (`count({__name__=~"sentinel_chunk_extraction_dedup_ratio.*"})` empty, against the sibling confidence
histogram returning all 16 default-bucket series in the same query shape), because the v2 chunked path is not
emitting. The entry exists so that the first time it does, the collapse is already known rather than rediscovered.
Fix: add the name to the same `AddView` list in `SentinelCollector/src/Program.cs` that already applies
`confidenceBuckets` — the [0,1] boundaries suit a ratio unchanged.

**Finnhub catalog enrichment re-enriches and re-embeds the same rows every cycle, and none of its cooldown state
persists.** Measured 2026-09-17T10:57Z (atlas_secmaster SELECT-only, Prometheus): `sum by (result)
(increase(secmaster_catalog_enrichment_total[24h]))` = enriched 34,286, no_data 558, error 540, against a candidate pool
of 18,002 rows. 241 pool rows had `updated_at` in the last hour, all with `atlas_sector_code` NULL, and the head of the
pool is the rows the previous cycle just enriched. Two mechanisms, both in `CatalogEnrichmentBackgroundService`:
- The pool selects a NULL sector and orders by `updated_at` DESC, and an enriched row gets `updated_at` bumped. A row
  whose profile gives no mappable industry therefore returns at the head of the next cycle: one Finnhub call and one
  re-embed every interval, for good.
- The no_data cooldown (`finnhub_enrichment_attempted_at`) and the failure back-off (`finnhub_enrichment_fail_count`,
  `finnhub_enrichment_next_retry_at`) are written into the jsonb `Metadata` dictionary IN PLACE, which EF never saves
  (no value comparer; `OpenFigiEnrichmentMetadataPersistenceTests` pins the same trap for OpenFIGI). 0 active
  Equity/ETF rows carried any `finnhub_*` key. `RetryAfterDays` and the back-off options are inert.
D-17 (`SecMaster/AGENT_README.md`) made `ApplyProfile`'s own keys persist as a jsonb merge, and left the
no_data and back-off writes alone on purpose: fixing them moves the Finnhub call rate that D-17's acceptance reads as
unchanged. The fix writes those keys with `InstrumentMetadataMerge.MergeInstrumentMetadataAsync`
(`SecMaster/src/Data/InstrumentMetadataMerge.cs`), as the success path does, NEVER by reassigning `Metadata`: that
persists, but as a whole-column write from a load older than the Finnhub call, erasing the OpenFIGI keys saved
meanwhile, and `EnrichmentMetadataRaceTests.should_keep_the_openfigi_keys_saved_while_a_finnhub_cycle_got_no_profile`
and `..._failed` go RED on it. Cost: Finnhub quota and llama-cpu-embed CPU on the same rows all day.
Re-check:
  `select count(*) from instruments where is_active and asset_class in ('Equity','ETF') and updated_at > now() - interval '1 hour' and atlas_sector_code is null;`
  `select count(*) from instruments where metadata ? 'finnhub_enrichment_attempted_at' or metadata ? 'finnhub_enrichment_fail_count';`

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
(`EntityResolutionService.cs:1080`); the other three and the `:206` drop are silent.
POPULATION: 91 quarantined rows = 82 real tickers + 9 macro-junk (the D-4 class) that sit in NEITHER enrichment pool
and need their own disposition (`SELECT asset_class, count(*) FROM instruments WHERE is_active=false GROUP BY 1;` in
`atlas_secmaster` -> Equity 74 / ETF 8 / fred_series 8 / Economic Indicator 1, 2026-09-05).
HAZARD, un-alerted: `idx_instruments_symbol` is `UNIQUE ... WHERE (is_active = true)` but
`idx_source_mappings_collector_source` is STILL UNIQUE on `(collector, source_id)` GLOBALLY with no predicate
(unchanged 2026-09-16), so the source-mapping insert in `RegistrationService.RegisterCoreAsync` raises a 23505 the moment a quarantined row carries a mapping
-- 1 of 91 does (`GSV.NE`) -- and nothing alerts when that stops being harmless.
Re-check: `sum by (result)(secmaster_entity_resolution_self_seed_total)` (2026-09-05: `idempotent_skip` 4625,
`inserted` 126, `quarantined_skip` 32, and no `error` series -- the 23505s are gone; every result is zero-initialised
  at boot since SecMaster D-16, so read `error` by its VALUE, never by the series being present) and
  `SELECT indexname, indexdef FROM pg_indexes WHERE indexname LIKE 'idx_instruments_symbol%'
     OR indexname LIKE 'idx_source_mappings_collector_source%';`

**Six services keep their integration suite behind `compile.sh --integration`, so the push marker never sees it,
and every one of the six is now green.** [2026-09-17, re-measured 2026-09-20] The six are ThresholdEngine,
FredCollector, OfrCollector, NasdaqCollector, AlphaVantageCollector and FinnhubCollector; SecMaster moved its suite
into the gate after #900. All four formerly-red suites are green: OfrCollector 64 of 64, AlphaVantageCollector
45 of 45, FinnhubCollector 38 of 38 (#1067), FredCollector 127 of 127 (was 61/64, 6/45, 6/38, 113/123).
ThresholdEngine (67) and NasdaqCollector (28) were already green. FredCollector's last 7, the
`EventStreamIntegrationTests` that seeded the `events` table while the code serves `fred_observations`, were the
gRPC design conflict this entry held for arbitration; it was decided in favour of the code and the card corrected
(FredCollector D-3), so that block is closed here rather than tombstoned. The consequence stands: a regression
only an integration suite can catch ships behind a valid push marker, and nothing re-runs these six between PRs.
#900 found that class on SecMaster: four tests silently red since #231.
Re-check: `bash <Service>/.devcontainer/compile.sh --integration`, then read the `Failed!` line of the
`*.IntegrationTests.dll`.

**FredCollector's `compile.sh` cannot run in a fresh worktree at all: its devcontainer compose file `env_file:`s a
gitignored `.env` that only the main checkout has.** First occurrence, 2026-09-20.
`FredCollector/.devcontainer/compose.yaml:11-12` carries `env_file: [../../FredCollector/.env]`, and
`FredCollector/.gitignore:44` ignores `.env`, so a fresh worktree has only `.env.example` and its compile.sh --
default or `--integration` -- dies at compose load with `fatal ... Failed to load .../FredCollector/.env: no such
file or directory` before a single test runs (re-derived 2026-09-20 in this worktree: `config` exits 1 with exactly
that message).
OFRCOLLECTOR IS NOT AFFECTED, though its compose file names `.env` the same way: `OfrCollector/.env` is TRACKED
(`c7b937aa`, 2025-12-07), so it is present in every worktree and
`sudo nerdctl compose -f OfrCollector/.devcontainer/compose.yaml config` returns rc 0 with no error there
(re-derived 2026-09-20 in a fresh worktree, where the Ofr integration suite also ran green). That it is tracked is
an accepted risk in its own right -- see "Accepted risks, do not re-flag" below; do not untrack it to make the two
services symmetric, which would give Ofr this defect.
The compose-spec fix (`env_file: [{path: ..., required: false}]`) is NOT available here: nerdctl 1.7.7 rejects it
with `validating ...: services.fred-collector-dev.env_file.0 must be a string` (re-measured 2026-09-20).
DROPPING THE `env_file:` KEY IS NOT A FREE FIX: the `environment:` block below it covers only the DATABASE
variables. It sets `DB_HOST/PORT/USER/PASSWORD` and wins over `env_file`, but it names no FRED variable at all
(`grep -c FRED_API_KEY FredCollector/.devcontainer/compose.yaml` -> 0), so `.env` is the devcontainer's only source
of `FRED_API_KEY`. `FredCollector/src/DependencyInjection.cs:41` gates the WHOLE FRED API client registration on
that key being non-empty, and `:212` reads it again inside that registration with `:213` throwing when it is null.
The throw is therefore not what a missing `.env` produces: `:41` simply skips `AddFredApiClient`, and the five
consumers that take `IFredApiClient` -- `DataCollectionService`, `BackfillService`, `AlfredBackfillService`,
`SeriesManagementService`, `SeriesSearchService` -- are left with no registration to resolve, which is the quieter
failure of the two. A committed non-secret default `.env`, or passing the key through `environment:`, are the open
options; dropping `env_file:` alone is not one.
WORKED AROUND, NOT FIXED, while landing the cursor fix: `cp /home/james/ATLAS/FredCollector/.env
<worktree>/FredCollector/.env` before the first compile. The copy is gitignored so it never reaches a commit, and
it has to be repeated in every new worktree -- which is the defect, not a remedy.
Re-check: `ls <worktree>/FredCollector/.env` (absent in a fresh worktree; present in /home/james/ATLAS), then
`sudo nerdctl compose -f FredCollector/.devcontainer/compose.yaml config` from a worktree without it (rc 1) and the
same command for `OfrCollector` (rc 0), and `grep -n FRED_API_KEY FredCollector/.devcontainer/compose.yaml` (0 hits).

**FOUR OF THE FIVE STREAMING COLLECTORS MINT `EventId = Ulid.NewUlid()` PER SYNTHESISED EVENT, SO
`processed_events` CANNOT DEDUPE THEM AND NO ANTI-JOIN CAN MEASURE WHAT THEY LOST.** [2026-09-20] Only
FredCollector uses a row-derived id (`obs-{Id}`). `SELECT source_collector, count(*) AS total,
count(*) FILTER (WHERE event_id LIKE 'obs-%') AS obs_prefixed FROM public.processed_events WHERE
event_occurred_at > now() - interval '7 days' GROUP BY 1` returns, as total / obs_prefixed:
FredCollector 2046 / 2046, FinnhubCollector 179,206 / **0**, SentinelCollector 11,736 / **0**,
OfrCollector 333 / **0**, AlphaVantageCollector 3 / **0**. Every one of those four is zero — the
totals are what they processed, not what was joinable. THIS IS THE PRECONDITION FredCollector D-4
DEPENDS ON: its cursor re-serves a whole instant on purpose because a repeat is a no-op against a
stable id. Ported to a Ulid collector unchanged, every re-serve becomes a fresh INSERT.
The cost is already being paid without the port: FinnhubCollector holds 1,075,092 `processed_events`
rows against 209,283 `finnhub_quotes` across 18 distinct symbols, and in the last 7 days 179,206
processed rows against 34,513 new quotes — a 5.2x redelivery multiplier, 25,601 rows a day, into a
table that is 513 MB, is NOT a hypertable and has zero retention jobs (1,916,012 rows since
2025-11-21). Cursor shapes, read 2026-09-20: Finnhub `CollectedAt > from` ordered by CollectedAt only,
Ofr `> since`, Nasdaq `>= from` (no skip, but every poll redelivers the boundary instant into a dedupe
that cannot match), AlphaVantage polls the latest row per series. Tie exposure, same 7 days: Finnhub 0
of 34,513 rows share an instant, Ofr 0 of 293, Nasdaq 0 of 0 (commented out of compose), AlphaVantage 9
of 12 — so Finnhub and Ofr carry the same defective cursor SHAPE with no measured loss, while
AlphaVantage's exposure cannot be quantified. Fixing this means deriving the id from the row, per
collector, BEFORE porting D-4. Re-check: the two queries above, plus
`SELECT count(*), count(*)-count(DISTINCT collected_at) FROM public.<table> WHERE collected_at >
now()-interval '7 days'` per collector table, and
`SELECT count(*) FROM timescaledb_information.jobs WHERE hypertable_name='processed_events';` (0).

**`public.events` LOST ITS LAST WRITER IN D-3 AND NEVER HAD A READER; THE TABLE IS STILL THERE.** [2026-09-20]
15,919 rows, 23 MB total (20 MB heap + 3 MB across 4 indexes), 78 rows/day from 2025-11-21 to the D-3 cutover,
written only by FredCollector (15,715 SeriesCollected + 204 CollectionFailed). No view, no FK, no script, no
dashboard and no ansible artifact queried it; `grep -rn '_dbContext.Events' FredCollector/src/` returned only the
two EventPublisher writes, now deleted. The `DbSet<EventEntity>` mapping is retained deliberately so the EF model
still matches the live schema — removing it makes the next `dotnet ef migrations add` emit a DROP TABLE for those
rows. Retiring it is: drop the DbSet and `EventEntity`, generate the migration, decide whether the 204
CollectionFailed rows are worth exporting first. Re-check:
`SELECT count(*), max("OccurredAt"), pg_size_pretty(pg_total_relation_size('public.events')) FROM public.events;`
— a `max` that has not moved since the D-3 deploy confirms the writer is gone.

**DEVCONTAINER TEST RUNS PUBLISH D-5's COUNTER INTO PRODUCTION PROMETHEUS, SO PANEL 16's "IS IT WIRED"
CONTROL READS WIRED WITH NOTHING DEPLOYED.** [2026-09-20] `fredcollector_collection_failures_total` is
live in production Prometheus right now, at the full D-5 product and all at 0:
`count(fredcollector_collection_failures_total) by (exported_job, job, instance)` returns 8 under
`{exported_job="fred-collector", job="otel-collector", instance="otel-collector:8889"}`, first sample
2026-09-20T19:06:32Z and peaking at 16 when two runs overlapped. Nothing is deployed: the running
`fred-collector` container was created 2026-09-16T11:00:32Z from an image built 2026-07-31T16:32:48-04:00
(`nerdctl container inspect fred-collector` then `nerdctl image inspect fred-collector:latest` -- bare
`inspect` returns the IMAGE), which predates this counter entirely. The emitter is the test host:
`ApiIntegrationTestBase` starts a real `WebApplicationFactory<Program>`, so `AddApplication()` runs and
`MetricWarmupHostedService` seeds all 8 series; `FredCollector/.devcontainer/compose.yaml` joins the
production `ai-inference` network and `FredCollector/.env` sets `OTEL_ENDPOINT=http://otel-collector:4317`,
the production collector. The series are labelled exactly as the real service's would be, so nothing
downstream can tell them apart. CONSEQUENCE: panel 16 (`Collection Failures (24h)`) documents "B absent
means the counter is not wired and A cannot be trusted", and its target B is a bare
`sum(fredcollector_collection_failures_total)` with no `exported_job` or `instance` filter -- so B is
PRESENT today because of a test run, and a deployer reading "A=0 with B present" concludes "no failures"
from a host that is not production. This PR's own verification runs created them. Not fixed here: the
honest fix is at the SOURCE (the devcontainer should not export to the production collector), not a
per-panel label filter, which would leave every other consumer to re-learn it.
RE-CHECK, AND IT MUST BE A RANGE QUERY, NOT THE INSTANT ONE ABOVE: the series are created by a test run
and go stale when it ends, so an instant query reads EMPTY -- CLOSED -- at any moment no integration run
is in flight, which is almost always. That is this file's own corpse-detector shape: the check would pass
precisely because the evidence had expired. Ask instead whether the series existed AT ANY POINT over a
window that could contain a run:
`max_over_time(count(fredcollector_collection_failures_total)[24h:1m])` over the last 24h -- any result
at all, while `nerdctl image inspect fred-collector:latest` still predates the D-5 merge, is this defect.
The entry closes when that returns no data across a day in which integration runs DID happen (confirm
that separately, or the empty result is the corpse-detector again). MEASURED SIDE BY SIDE 2026-09-20
~21:40Z, with no run in flight: the instant query returns EMPTY -- it would have reported this entry
CLOSED -- and the range query returns 16 on the same Prometheus at the same moment.

**FIVE OF THE TWELVE SITES OF THE D-4 COMPOSITE CURSOR ARE UNPINNED; THE WHOLE `Between` PATH IS ONE OF
THEM.** [2026-09-20] Three review rounds each found unpinned sites of this one change -- one, then two,
then three -- because each sweep enumerated by reading. This is the enumeration done by MUTATION: every
site was derived mechanically from `grep -rn 'ObservationCursor\|CursoredEvent\|item.Cursor\|sinceId\|fromId\|ThenBy(o => o.Id)'`
over `FredCollector/src`, then mutated ONE AT A TIME against the full 321-unit + 128-integration suite.
PINNED (mutant dies, with the tests that kill it):
  - `StreamEventsSinceAsync` composite predicate -- 4 tests
  - `ToCursoredEvent`'s `new ObservationCursor(obs.CollectedAt, obs.Id)` -- 2 tests
  - `ObservationCursor.Before`'s `0` sentinel -- 2 tests
  - `SubscribeToEvents` opening on `Before(startFrom)` -- SubscribeToEvents_OpenedAtATiedInstant_DeliversTheWholeGroup
  - `SubscribeToEvents` CATCH-UP advance (`cursor = item.Cursor`, 16-space indent) -- SubscribeToEvents_DeliversSiblingsCommittedAfterTheirInstantWasServed
  - `GetEventsSince` opening on `Before(from)` -- GetEventsSince_AtACheckpointInstant_ServesTheWholeTiedGroup
UNPINNED (mutant survives the FULL suite, 321/128 green):
  - `StreamEventsSinceAsync` `.ThenBy(o => o.Id)` -- costs intra-instant ordering determinism only; no row lost
  - `StreamEventsBetweenAsync` composite predicate reverted to scalar -- the SAME defect this PR fixes,
    still live on that method
  - `StreamEventsBetweenAsync` `.ThenBy(o => o.Id)`
  - `SubscribeToEvents` POLL-LOOP advance (20-space indent) -- and that is the loop ThresholdEngine lives
    in. Deleting it does not lose rows: the cursor goes stale and the poll re-serves from it, so the cost
    is unbounded re-delivery, which `processed_events` absorbs because `obs-{Id}` is stable per row
  - `GetEventsBetween` opening on `Before(from)`
STRUCTURAL, not mutatable in isolation: the `CursoredEvent(Event, ObservationCursor)` record shape -- any
edit is a compile error, and its behaviour is covered by the `ToCursoredEvent` mutant above.
WHY THE `Between` PATH IS NOT FIXED HERE RATHER THAN MERELY UNTESTED: `grep -rn GetEventsBetweenAsync`
outside `Events.Client/ObservationEventClient.cs` returns ZERO callers anywhere in the repo, so the method
is an advertised proto surface with no consumer. It carries the identical tie-skipping defect, latent.
Re-check, one mutation at a time: make the edit, run `FredCollector/.devcontainer/compile.sh --integration`
(the `--integration` flag is REQUIRED -- three of the five live in repository SQL that only the integration
suite runs against a real database, so the default unit-only run reports a false GREEN), and confirm the
suite is still green. A RED run means someone has pinned that site and it drops off this list.

**ATLAS-HOME PANEL id8 PAINTS A DEAD FRED EVENT STREAM AS A HEALTHY 0.** [2026-09-20] The Events (1h)
stat was repointed off the retired `fredcollector_events_published_total` onto
`sum(increase(fredcollector_grpc_events_streamed_total[1h])) or vector(0)`, and the `or vector(0)` came
along with it. That counter is created on the first streamed event, so it is ABSENT — not zero — when the
stream has never run, and `or vector(0)` renders both cases as a purple 0. This is the same blindness
`fredcollector.json`'s D-5 failures panel explicitly refuses ("No target carries `or vector(0)`"), so the
home dashboard and the service dashboard now disagree about what absent means. Not fixed here because the
repoint was the scope. Re-check:
`grep -n 'fredcollector_grpc_events_streamed_total' "deployment/artifacts/monitoring/dashboards/Overview/atlas-home.json"`
— the entry closes when that line no longer carries `or vector(0)`.

**FREDCOLLECTOR'S STREAM LOSES ~12 ROWS A WEEK TO CURSOR INVERSIONS, AND NO CHOICE OF SORT COLUMN CAN
FIX IT.** [2026-09-20] D-4's composite (CollectedAt, Id) keyset recovers 204 of the 216 rows TE missed in
7 days; the other 12 are INVERSIONS — a row that became visible AFTER one carrying a strictly later
`CollectedAt`, so the cursor had already advanced past its instant and `>= sinceAt` can never return.
62 rows in 7 days sit in that class (median gap 8.0 ms, max 68.9 ms), of which 12 were actually lost.
The cause is that `CollectedAt` is assigned in app code at cycle start, ahead of the INSERT and of the
COMMIT, so the stream's sort key is not monotone in VISIBILITY order; `Id` is allocated at INSERT and is
equally free to commit out of order, and the table records commit order nowhere. So this is NOT a
tiebreak bug and re-sorting on `Id` would both fail to fix it and silently change the stream's meaning
from collection time to arrival. The two real remedies: (a) re-read a trailing lag window on each poll,
which must be priced against the tie sizes — 730 rows for a 24-month daily backfill and 100,000 once on
2026-06-02 would be re-served once per poll for the width of the lag; (b) give the table a
commit-ordered column and sort on that. Neither is free, and the loss is transient: TE's
DataWarmupService re-reads a fresh window on every restart and heals it.
Re-check, SELECT-only:
```sql
WITH w AS (SELECT o."Id", o."CollectedAt",
       max(o."CollectedAt") OVER (ORDER BY o."Id" ROWS BETWEEN UNBOUNDED PRECEDING AND 1 PRECEDING) AS max_prev,
       (pe.event_id IS NULL) AS missed
  FROM public.fred_observations o
  LEFT JOIN public.processed_events pe ON pe.event_id='obs-'||o."Id" AND pe.source_collector='FredCollector'
  WHERE o."CollectedAt" > now()-interval '7 days' AND o."Value" IS NOT NULL)
SELECT count(*) FILTER (WHERE missed) AS missed,
       count(*) FILTER (WHERE missed AND max_prev > "CollectedAt") AS inversion_residual FROM w;
```
2026-09-20, pre-deploy: 216 / 12 (re-derived 19:20:51Z: 2,271 streamable / 220 / 11). Post-deploy the
left column should fall to the right one as the 7-day window rolls past the deploy -- which takes SEVEN
DAYS to mean anything. For a verdict on the deploy itself, scope the same query to rows collected after
the deploy instant and settled (`< now() - interval '15 minutes'`), and GUARD IT ON A NON-EMPTY WINDOW:
FredCollector collects in ONE burst a day, 18:00:0xZ for 61-95s (measured over 8 days), so a scoped
window that does not span an 18:00Z burst holds ZERO rows and yields `missed = inversion_residual = 0`,
which reads PASS while having measured nothing. Report `count(*) = 0` as NO VERDICT, never PASS, and run
the check only after the first 18:00Z burst following the deploy. Worked, 2026-09-20: empty window
0/0/0 NO VERDICT; trailing 24h 309/30/0 FAIL; the 18:00:60-18:01:10Z slice 36/0/0 PASS. The full form is
in PR #1073's body.

**`eventTypes` IS ADVERTISED ON THE `ObservationEventStream` PROTO AND IGNORED BY FOUR OF THE FIVE COLLECTORS.**
[2026-09-20] Every collector that synthesises events from an observations table can emit only `SeriesCollected`,
so a client asking for `CollectionFailed` gets `SeriesCollected` rows rather than none. Harmless today because
ThresholdEngine requests exactly `new[] { "SeriesCollected" }` on both its phases
(`MultiCollectorEventConsumerWorker`), and NasdaqCollector's card already documents "eventTypes-ignore" as
correct — so this is a contract to either honour or delete from the proto, not a per-collector bug. FredCollector
pins the current behaviour in `EventStreamIntegrationTests.GetEventsSince_EventTypesFilter_IsAcceptedAndIgnored`.
Re-check: `grep -rn 'eventTypes' */src/Grpc/Repositories/EventRepository.cs` — a parameter that appears only in
the signature is an ignored one.

**FREDCOLLECTOR REGISTERS `"FredCollector.DataCollection"` WITH `AddSource` AND NOT WITH `AddMeter`, SO TWO
INSTRUMENTS HAVE NEVER REACHED PROMETHEUS.** [2026-09-20] `Program.cs` lists it under `WithTracing` but its
`WithMetrics` block names only "FredCollector", "FredCollector.EventStream" and the substrate meter. The orphans
are `fredcollector.series.collected.total{series_id,success}` and `fredcollector.data_collection.duration_ms` —
the first is the counter nearest the collection outcome and is absent from the live metric-name enumeration, which
is why D-5 had to add a new counter on the exported meter rather than lean on it. Fixing it is one line, but it
also exports an 80-series_id histogram that duplicates `fredcollector_data_collection_duration_milliseconds`, so
the honest repair is to move the counter, not register the meter. Re-check:
`list_prometheus_metric_names regex=fredcollector_series_collected.*` returns nothing today.

**ALPHAVANTAGE'S STREAM SERVES ONLY THE LATEST ROW PER SERIES, AND 9 OF ITS 12 ROWS IN 7 DAYS SHARE AN INSTANT.**
[2026-09-20] `ObservationEventStreamService` polls `latest` per series and advances `lastEventTime` only when
`latest.CollectedAt > lastEventTime`, so it is not a keyset stream at all: anything behind the latest row is never
offered. TE processed 3 AlphaVantage events in that window against 12 collected rows, but the Ulid EventId (entry
above) makes the anti-join that would settle it impossible. Re-check: the per-table tie query above for
`alphavantage_observations`, against
`SELECT count(*) FROM processed_events WHERE source_collector='AlphaVantageCollector' AND event_occurred_at >
now() - interval '7 days';`
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
(`SentinelCollector/src/DependencyInjection.cs:119`) but no consumer reads a probe verdict; it goes live the moment the probe is wired.
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
Re-run 2026-09-20: UNCHANGED -- rc 1, 52 PASS, the same single FAIL `registered set drifted:6a7 >
dream-pending-notice.sh`, 43 days red. AND THE COST IS BIGGER THAN "READERS SKIP IT": no CI workflow runs the hook
suites at all (`.github/workflows/` holds alert-rules, python-tests, sync-docs and nothing else), so this suite is
the ONLY thing asserting that a hook is REGISTERED -- `wired` covers 12 of them, `git-push-guard.sh`,
`pr-review-marker.sh` and `commit-marker-staleness.sh` among them. Unwiring any one of them changes the summary
from FAIL to FAIL, on a suite nothing runs: a guard can be removed today and NOTHING in the repo notices. That is
the same defect class as the PR the measurement came from (a guard whose enabling line nobody tests), one level up.
`scripts/tests/new-epic-selftest.sh` sits in the same position — nothing runs it either (`git grep -l
new-epic-selftest -- .github` -> 0 on 2026-09-20) — with one mitigation the hook suites lack: `scripts/new-epic.sh`
carries three known-bad controls INSIDE itself, so an operator running it gets the dullness report even when no
suite ran. That is the shape to copy here, not a reason to leave the suites unwired.
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

  ✗ barred as a swap criterion -> `SentinelCollector/AGENT_README.md` §MODEL_ACCEPTANCE
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

**`gemini-resolver-mcp/gemini_resolver/cache.py`'s `make_cache_key` docstring cites SecMaster lines that SecMaster
D-16 (PR 1061) moved.** At `cache.py:173` the docstring points at line 289 of `IdentifierConfirmationService.cs` for the
`$`-leading surface gate, which now sits at line 290, so it lands on the comment above the gate. At `cache.py:166` it
points at line 999 of `EntityResolutionService.cs` for `PersistConfirmedInstrumentAsync`, now at line 1026. (Written
without the colon form on purpose, so a citation sweep does not read these stale numbers as this entry's own claims.)
PR 1061 changed only SecMaster; fixing the docstring there would have meant running the gemini-resolver-mcp suite
too. The citation sweep cannot see it: it reads `*.md` files only. Re-check: `grep -n "s\[0\] == '\$'"
SecMaster/src/Services/IdentifierConfirmationService.cs` and `grep -n "Task<bool> PersistConfirmedInstrumentAsync"
SecMaster/src/Services/EntityResolutionService.cs` against the two numbers in `cache.py`. Close by citing the construct
(`ShouldResolveViaGemini`, `PersistConfirmedInstrumentAsync`) instead of a line.

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
`SecMaster/src/Configuration/EnrichmentOptions.cs:14` and `SecMaster/src/Services/CatalogEnrichmentBackgroundService.cs:183`
both call "Finnhub 403 for foreign tickers" a TRANSIENT enrichment failure. A 403 is plan-uncovered and permanent, and
it arrives as a NULL profile rather than an exception -- which `SecMasterMeter.cs:310` and
`SecMaster/src/Services/IFinnhubCollectorClient.cs:33` already say correctly, so this is a finished conversion with
two comment fixes left.
Re-check (2026-09-05, reproduced 2026-09-16):
  `grep -rn '403' --include=*.cs SecMaster/src FinnhubCollector/src | grep -i transient`
  -> four hits today: the two above are the debt; `FinnhubApiClient.cs:29` and `:406` state the permanent/transient
  split CORRECTLY and are the controls, not the debt.

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
| B | 2026-09-22 | OPEN | gemini-resolver connect fallbacks are counted but unalerted: an unanswering address family is otherwise invisible |
| B | 2026-09-20 | OPEN | 50 of 53 live `AddHostedService<T>` registrations are UNPINNED: the line that makes the component RUN can be deleted with every suite green |
| D | 2026-09-20 | OPEN | The hosted-service pin gate cannot see 5 live factory-overload registrations, and Reports has NO keyable registration at all |
| D | 2026-09-20 | OPEN | A mutation harness that rewrites ONE `.py` path scores a FALSE SURVIVOR: CPython keys bytecode on (mtime-second, size), and `mutation-check.py` already holds an equal-length pair |
| D | 2026-09-20 | OPEN | The pin sweep's zero-test rule binds the CONTROL run only; a MUTATED run that ran nothing files UNPINNED quoting the control's count |
| D | 2026-09-20 | OPEN | A `/*` inside a C# string literal opens a block comment for the pin scanner; an unterminated one blanks the rest of the file (0 live sites) |
| D | 2026-09-20 | OPEN | The pin control's ALREADY-RED arm has no test: deleting it keeps 154/154 green and prints a mutation-phrased sentence about an unmutated run |
| B | 2026-09-20 | OPEN | `verify-card-companion.py` ECHO/STATUS is VOCABULARY-BOUND and cannot be complete: two review rounds found two kinds (negative rules, then wiring status), each added to the word list AFTER a human found it. It also CANNOT fail the build on an existing finding -- the freeze gates GROWTH only, so a card-only rule present before 2026-09-20 stays advisory forever. Repro: `python3 scripts/verify-card-companion.py` -> 16 advisory, 0 gating |
| C | 2026-09-20 | OPEN | Five `verify-card-companion.py` mutations pass GREEN, each a real hole. (1) INVERTED echo: card says "must" where the companion says "must never" -- tokens match so ECHO passes; repro: negate a card ALSO clause, rerun, 0 gating. (2) Card entry degraded to a BARE POINTER (`D-n slug: DETAIL DECISIONS.md §D-n`) -- PARITY and GUARD both pass because neither requires the card to say anything; repro: truncate one entry to its pointer. (3) REORDERED companion sections -- `read_companion` returns an order list that `check()` never reads; repro: swap two `## D-n` blocks, 0 gating. (4) DUPLICATE id: a second `## D-n` silently overwrites the first in the dict; repro: copy a section, 0 gating. (5) DUPLICATE slug across two ids -- never compared; repro: rename one slug to match another |
| C | 2026-09-20 | OPEN | `audit.sh` W10 counts CHARACTERS, not bytes (`${#var}` is character length in bash) and the cards carry non-ASCII, so a flagged entry's byte count runs slightly higher -- FinnhubCollector D-2 is 10,040 characters / 10,070 bytes. Never changes a verdict at the 4,000 threshold, which is itself PICKED (~5x the longest rule line this split produces), not derived. Repro: `bash .claude/skills/architecture-cards/scripts/audit.sh FinnhubCollector` |
| B | 2026-09-20 | OPEN | FinnhubCollector D-2 is 10,040 characters on one card line -- **10,070 BYTES of the card's 23,733 BYTES, 42.4%**, one unit on both sides of the ratio. Two earlier revisions each mixed the units and each got a different answer: 42.5% is chars/chars (10,040 of 23,615) stated against a BYTE denominator, and 42.3% is chars/bytes, which is not a ratio of anything. Like-for-like in characters is 42.5% of 23,615, and that is the pair to quote wherever the CHARACTER threshold is the subject, since W10 counts characters -- the only W10 finding in the repo, unfixed. Same disease as the 303KB SentinelCollector card at smaller scale; the fix is the same split (rule on the card, evidence to a new `FinnhubCollector/DECISIONS.md`, with the card entry ending in a DETAIL pointer at that file's D-2 section -- written as a literal anchor only once the file exists, since the pointer gate resolves a bare `DECISIONS.md` against two tracked files and refuses the ambiguity). Repro: `bash .claude/skills/architecture-cards/scripts/audit.sh FinnhubCollector` |
| A | 2026-09-20 | OPEN | Catch blocks that swallow a REAL fault without marking the span leave it invisible to Tempo (147/673 across 3 services, a FLOOR; excludes 63 cancellation-only, which are NOT defects) |
| B | 2026-09-20 | OPEN | `verify-pointers.py` has no BODY check and no FLOOR, so a deletion test against it proves the ANCHOR is load-bearing, never the RULE — two reproductions below, plus two coverage gaps: bare-label navigation and a template body that is one fence |
| B | 2026-09-20 | OPEN | The smoke test cannot see the OTEL stack, so a green run is consistent with loki/tempo/prometheus being down |
| B | 2026-09-21 | OPEN | 14 of 176 deletion units the attach-scorer PR adds are DELETABLE with all four Python suites green, 6 more constructs only measurable as whole statements -- including the `--attestation` argparse flag the feature hangs off, and two refusal conditions of the staleness gate. Supersedes the 3-of-8 table, one row of which was a parse failure read as a pin |
| C | 2026-09-21 | OPEN | One scorecard carries two different `schema_invalid` populations under one name: `diagnostics.schema` EXCLUDES failed calls, `acceptance_evidence.run_integrity` COUNTS them. Measured on one 2-record arm: 0 of 1 against 1 of 2 |
| C | 2026-09-21 | OPEN | `attach_schema_valid`'s recorded TRUE overrides a shape the fixed schema rejects (a malformed pick scores CORRECT); only the FALSE branch is tested, the stated rationale is false, and two mutants of the `isinstance(..., bool)` check survive the whole suite |
| D | 2026-09-21 | OPEN | The `--task cod` and `--task cove` readers coerce `schema_valid` through `bool(...)`, so the tri-state `null` collapses to False -- NO VERDICT read as INVALID, and the fallback beside each coercion never runs. Deferred because the failure is CONSERVATIVE and the cod reader is reachable only by scoring a pick predictions file under `--task cod`, which mis-scores regardless; the cove reader raises `KeyError` on such a row |
| B | 2026-09-21 | OPEN | #1090's approve reason says three BLINDNESS findings were "routed to the backlog with re-checkable measurements"; none were written. Filed now -- 2 of the 3 closed by the PR that files them, and the remainder is that `stage()` is reachable from no CLI path, so its control cannot be driven end to end |
| C | 2026-09-21 | OPEN | `stage_and_verify.py` has ZERO refusing sites on `attach-pick/pick_prompt_clean.txt` (0 of 6) and `attach-pick/pick_schema.json` (0 of 6): deleting or renaming either keeps `--verify` at `-- OK` rc 0 and only moves coverage, 35 -> 29 -> 23 of 77. CORRUPTION is caught (one byte -> rc 2, 6 MISMATCH). `test_run_model.py` catches the absence (rc 1) and not the corruption (186/186, rc 0) -- neither channel covers both |
| C | 2026-09-21 | OPEN | EIGHT behaviour-preserving mutants of `stage_and_verify.py` survive both channels (selftest rc 0/23 legs, pytest rc 0/161 passed) with `--verify` stdout byte-identical on the committed set: the refusal predicate's key, root ORDER, the `sha256:` strip, `glob`/`rglob`, `stage()`'s non-json branch, the empty-sibling guard, and both sibling-preference orders. N1's two rules disagree on 6 COMMITTED sites and no fixture uses that shape |
| D | 2026-09-21 | OPEN | The paired outputs' `.a` and `.b` are the 2 of 5 redacted path fields with NO sibling digest -- 6 sites naming the two arms gates 1 and 2 are stated on, outside `--verify`'s numerator. Each paired file holds one digest-valued string, `replicates_sha256`, which attests neither arm. The round README sentence claiming all five sit beside a digest is corrected in the PR that files this |
| D | 2026-09-21 | OPEN | THREE unanchored figures in this file are false at `20d1540d` and only one is a test count: `pytest scripts/tests` stated as 154/155 where it is 161 (two entries, four copies), `.github/workflows/` said to hold three files where it has held four since #1087, and a `grep -l dotnet` one-liner reading 0 where it reads 1 -- off a comment saying `no dotnet`, so it now fails toward CLOSED. Every finding holds. The discriminator is ANCHORED vs unanchored, NOT absolute vs delta: the file's 55/51 `#NNN` figures look stale against 80 and are exact at the two commits they name |
| D | 2026-09-21 | OPEN | The attach-round checker's pin chain terminates at three deletions nothing turns red on: a control removed together with its `EXPECTED_LEGS` entry (27 changed lines at the smallest of nine), the out-of-band test file removed outright (96 deletions; 161 -> 160 passed, rc 0 at `20d1540d`), and the two workflow `run:` lines (4 physical lines) -- the last pre-existing and universal, since NO test in this repo asserts on any workflow's `run:` commands, across a population of ELEVEN tracked files mentioning `.github/workflows`, not the five first named |
| D | 2026-09-21 | OPEN | The frozen attach candidates' `generator_version` (10 copied sites in the reduced round) matched the generator at `f37e658b`/`3c4dbb44` and stopped 19 min later at `a9831b59`, when it gained a `control()` and the candidates were not re-frozen. Re-attest from that revision -- `git log` cannot see it across a squash, `git rev-list --all` can. Counted UNREAD by the reader, not red |
| C | 2026-09-21 | OPEN | `stage_and_verify.py --verify` now reads **35 of 77** digest-valued strings in the round directory (was 13), 0 mismatches, no writer change and no committed byte altered -- a non-refusing sibling rule, with the `path` contract still refusing. **42 remain unread** and are counted and named on a `coverage:` line; 10 of those are `substrate_sha256`, whose artifact is deliberately uncommitted |
| B | 2026-09-20 | OPEN | The smoke test asserts nothing non-running, never that everything expected is present |
| B | 2026-09-20 | OPEN | Smoke test's `exited` + `ExitCode 0` exemption has no recency term; its data source carries no timestamp |
| C | 2026-09-20 | OPEN | A smoke-test run executing ZERO tasks exits 0; the class is unbounded in spelling, not closeable inside the playbook |
| B | 2026-09-20 | OPEN | Three smoke-test rows still say PASS on reachability alone: database, vllm-server, container Health |
| A | 2026-09-06 | OPEN | Three watermark-only ReExtract legs re-assert the row's tier; nothing pins that they do |
| B | 2026-09-20 | OPEN | The <=5 ms share is the only detector for an unapplied `hnsw.ef_search`, and nothing watches it |
| B | 2026-09-17 | OPEN | SecMaster D-16 self-seed and discovery refusals are counted but unalerted; their rate is unmeasured |
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
| D | 2026-09-20 | OPEN | new-epic.sh's graduation-shape control: its CANNOT-RUN arm survives three mutations at 59/59 green |
| D | 2026-09-20 | OPEN | verify-pointers.py resolves on the filesystem, not the git index: untracked target, `../` escape, symlink and duplicate construct all pass (0 of 38 affected today) |
| D | 2026-09-16 | OPEN | Three figures the PR-verdict decision check leaves un-re-checkable |
| D | 2026-09-16 | OPEN | Citations in tracked .md that cannot land are the corpus's steady state, and so is rc 1 |
| D | 2026-09-16 | OPEN | A conflicted index path makes the alerts selftest report a nonexistent permissions defect |
| D | 2026-09-16 | OPEN | verify-citations.py reports GREEN on a citation that has drifted onto the WRONG line |
| D | 2026-09-16 | AWAITING-DECISION | The documented citation sweep is .md-ONLY, so a line shift rots citations it cannot see |
| D | 2026-09-16 | OPEN | verify-citations.py skips off-allowlist extensions, and bare :NN continuations sans --bare |
| D | 2026-09-21 | OPEN | A prose-form reference is not a checked citation at all: cross-file line references in tracked `.md` sit outside every sweep -- **15 at `9f0f0ce8`, 16 at `76cd8d1c`** (a floor; the no-number class is unbounded) |
| D | 2026-09-16 | OPEN | Patches drop the exec bit, core.fileMode=false hides it, a disarmed hook fails SILENTLY |
| D | 2026-09-21 | OPEN | The attach-gold drift audit watches `accept` ids ONLY: 61 baseline-only ids are unwatched, and the 2 already deactivated were counted as an anonymous integer while the list that names movement stayed empty |
| D | 2026-09-17 | OPEN | Attachment gold residue: slash terms unsearchable by symbol, 6-term cap, yen rows, CI, IXIC |
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


### gemini-resolver connect fallbacks are counted but unalerted: an unanswering address family is otherwise invisible [2026-09-22]
`gemini_resolver/connect.py` bounds the connect to ONE deadline across every DNS address and races the families
250ms apart, so a family whose path stops answering now costs a stagger per new connection instead of 8 x 25s.
That removes the only trace such an outage left. Before the bound, the resolver's journal showed NO call completing
for 200.0s after 12:17:37.853Z on 2026-09-22 while 11 started, and the first 11 calls after that point completed,
FIFO-paired, 201.9-210.8s after they started. That journal is not in Loki: `list_loki_label_values service_name`
over 24h to 12:49Z returned six values, none of them the resolver. The only new signals are
`gemini_resolver_connect_fallbacks_total{first_family}` and one WARNING per hour in the journal. No rule, recording
rule or dashboard reads the counter.
Why no rule yet: the counter's healthy baseline is unmeasured. It should be near 0, because a fallback needs the
first-ranked address to miss a 250ms window against a ~20ms RTT. But nobody has counted it.
To close: after the resolver restarts on this code, take the 7d distribution of
`sum by (first_family) (increase(gemini_resolver_connect_fallbacks_total[30m]))` at 1-5m, pick threshold and `for:`
from it, then add an info-severity rule to `deployment/artifacts/monitoring/alerts/gemini-resolver.yml` with a promtool
firing case on burst-shaped input, and delete this entry in that PR.
Re-check: `grep -cE '^\s*expr:.*gemini_resolver_connect_fallbacks' deployment/artifacts/monitoring/alerts/gemini-resolver.yml`
returns 0 while this entry is open.

### 50 of 53 hosted-service registrations are unpinned: deleting the line that makes a component run keeps every suite green [2026-09-20]
A component's logic can be correct, tested and green while the single `AddHostedService<T>()` line that makes
the host START it is absent. Measured by removing each registration and running that service's own
`compile.sh`, reading the verdict from dotnet's own summary line rather than the script's exit code, and
only after an unmutated CONTROL run proved that service's suite green (a service that cannot reach green
yields NO VERDICT, because a mutation's RED would be about the pre-existing failure):
**3 PINNED, 50 UNPINNED across 53 registrations in 10 services.** The 3 pinned are `FinnhubCollector QuoteCollectionWorker`, `FinnhubCollector QuoteStalenessSeeder`, `FredCollector MetricWarmupHostedService` — all of them
covered by two test files written on 2026-09-20, and nothing else in the repo pins anything.

This is what cost PR #1073 a round: deleting `AddHostedService<MetricWarmupHostedService>()` from FredCollector
left 321 unit tests green and silently unpinned D-5's "seeded at startup" precondition.

| | |
|---|---|
| re-check the split | `awk '$1!~/^#/ && NF>=4 {print $3}' scripts/hosted-service-pins.baseline \| sort \| uniq -c` |
| re-measure one service | `python3 scripts/verify-hosted-service-pins.py --sweep <Service>`, then `--freeze` |
| what the sweep cost | 10 services, ~20 min wall — a service whose suite stays green with ALL of its registrations removed is settled in ONE run, so only a reacting service pays per-site |

**Not a blanket remedy, and the sharpest one named.** A registration is worth pinning where something
DOWNSTREAM depends on it RUNNING -- a seeded metric, a start ordering -- not per line. The sharpest
unpinned one is `SecMaster/src/DependencyInjection.cs:146`, the same
`AddHostedService<MetricWarmupHostedService>()` carrying the same "the series exist at 0 from process
start" argument that FredCollector D-5 leans on to justify a counter replacing a durable row; every test
in `SecMaster/tests/Telemetry/MetricWarmupHostedServiceTests.cs` calls `new MetricWarmupHostedService()`
directly, so deleting the registration is invisible there, and the sweep scores it UNPINNED.

This supersedes the "52 OF 54" entry retired from KNOWN DEFECTS in the same PR, which counted a
commented-out registration and the test FILES rather than the registrations they pin. It was already the
SECOND occurrence of this failure mode when it was written.

**Not gated, deliberately.** `scripts/verify-hosted-service-pins.py` freezes this set and reports GROWTH only — a
NEW registration, one whose statement or recorded verdict CHANGED, one that VANISHED, or one frozen UNMEASURED
(so `--freeze` alone cannot close a finding with a row that has no measurement behind it). Reddening CI on day one
over 50 findings is how a check gets switched off (#1083). Closing this entry means pinning registrations and
re-freezing, which moves rows from UNPINNED to PINNED; the debt is the UNPINNED count, never the check's silence.
The check is ADVISORY — branch protection 403s on this plan, so it reports and cannot block.

### 14 of 176 deletion units in the attach-scorer PR are DELETABLE line-by-line and 6 more constructs are, and its first table read a parse failure as a pin [2026-09-21]
Same class as the hosted-service entry above, in Python rather than C#. **The first measurement of this PR
(round 2) is superseded, not amended**: it claimed **3 DELETABLE and 5 pinned over 8 hand-picked candidate
lines** -- a sample with no denominator, quoted in that commit message and in the entry this replaces. Of
its 5 "pinned", four re-derive as pinned and one does not: `build_attach_gold.py`'s unmeasured-gold stderr
warning is the lone statement inside `if unmeasured:`, so deleting THAT LINE leaves an empty block, the
file stops PARSING, every suite goes red, and a MEASUREMENT FAILURE was recorded as a pin. Deleted as a
whole statement it is DELETABLE. The old table's other two deletable rows re-derive as deletable; its
third was a docstring line, which is prose and outside the population below.

**The method, corrected.** The unit is the smallest WHOLE syntactic element covering each added line -- a
leaf statement, one element of a dict/list/call display, one keyword argument -- never a fragment. The
harness `ast.parse`s the mutated file BEFORE running any suite, and a deletion that does not parse yields
**NO VERDICT**, naming the exception. Three known-bad controls run first and the harness refuses to print a
table unless all three behave: a FRAGMENT deletion (one line cut from a multi-line statement) must come
back NO VERDICT with a SyntaxError; a line a control asserts must come back PINNED; an injected junk
statement must come back DELETABLE. Without the third, a harness that reported PINNED for everything --
because the suites were already red -- would read as a clean result.

**Measured at `c521b83b`**, the three measured files unchanged since (only test assertion MESSAGES moved
after it), over `eval_harness.py`, `run_model.py` and `build_attach_gold.py` (test files
excluded -- deleting a test line measures nothing): 392 added lines, of which 185 are comment or docstring
prose, excluded because no test can pin prose. The remaining 207 code lines form **176 deletion units**:

| pass | PINNED | DELETABLE | NO VERDICT |
|---|---|---|---|
| line-level (the unit itself) | 72 | 14 | 90 |
| the 90, re-run as the smallest enclosing STATEMENT that parses (45 distinct) | 79 | 11 | 0 |

A statement-level PINNED is weaker than it reads: it says the enclosing CONSTRUCT is load-bearing, not
that the line is. Deleting `try:` also deletes the assignment inside it.

**The 14 line-level DELETABLE units**, which is the debt:

| unit | what a reader or operator loses |
|---|---|
| `eval_harness.py:2193` `"schema_invalid": meta.get(...)` and `:2194` `"schema_rows_judged": meta.get(...)` | the runner's own per-run count AND the population that says whether its null means "not recorded" or "nothing parsed". Both reporting surface, both unpinned; `:2194` was added by the round-3 fix and is recorded here rather than pinned, like its sibling |
| `eval_harness.py:1250` `entry = matched[0]` | with two attestation entries for one digest, the counts would be reconciled against the LAST rather than the FIRST. The loop above it is pinned; this choice is not |
| `eval_harness.py:1233`, `:1240`, `:1241` | three lines of REFUSAL MESSAGE text. The tests match a surviving substring, so the gate keeps denying while its diagnosis silently loses the path it examined and the counts it read |
| `build_attach_gold.py:1822`, `:1824`, `:1826` | the stage's JSON report to stdout, the attestation's parent `mkdir`, and the `attestation:` path line the operator must then pass to `--staleness` |
| `build_attach_gold.py:2087`, `:2090` | the selftest's own scaffolding: the stale-attestation `unlink` between fixtures, and the stdout/stderr redirect |
| `build_attach_gold.py:2111`, `:2119`, `:2120` | three CONJUNCTS of the new staleness controls' assertions. `SELFTEST_CONTROLS` compares the NUMBER of `expect()` calls, so a control that asserts less still reports `66 controls ran, 66 expected` |

**The 6 distinct statement-level DELETABLE constructs**, which the line pass cannot see:

| construct | why it matters |
|---|---|
| `build_attach_gold.py:2162-2165` `ap.add_argument("--attestation", ...)` | the CLI flag the whole feature hangs off. The selftest drives `stage_staleness` with a hand-built `Namespace`, so nothing exercises the argparse WIRING -- the Python twin of the hosted-service finding above |
| `build_attach_gold.py:1828-1830` `if unmeasured: print(ERROR ...)` | the row the old table called pinned. The selftest DOES execute it (stdlib `trace` confirms) into a redirected `StringIO` nothing reads, and asserts only the exit code |
| `eval_harness.py:1214-1216` `if checked is None: raise ...` and `:1227-1228` `if not isinstance(entries, list): raise ...` | two refusal conditions of the staleness gate. Never executed by any suite, and both deletable -- a gate whose point is refusing, with two conditions no control names |
| `build_attach_gold.py:1818-1819` per-gold `print(...)`, `:2096-2097` the selftest's `if not att.exists()` guard | reporting surface, and the selftest's own "no attestation written" diagnosis |

| | |
|---|---|
| re-measure | enumerate units by AST over `git diff main...HEAD -U0` added lines; per unit delete its whole line span, `ast.parse` the result (fail -> NO VERDICT), then run `test_eval_harness.py`, `test_run_model.py`, `test_check_staleness.py` and `build_attach_gold.py selftest`, each `$?` read bare; restore. Run the three known-bad controls first. Harness not committed |
| cost | 176 units x 4 suites, ~2.5 min wall (the four suites total 0.72 s) |
| closing it | assert the field, the printed line or the refusal message in an existing test; for the argparse flag, drive `main()` rather than a hand-built `Namespace` |
| what it CANNOT see | a unit whose deletion changes behaviour no suite exercises reads DELETABLE exactly like dead surface; "PINNED" means some test reacted, never that the test asserts anything useful; and prose is excluded by construction |

### 10 of the attach-round converters' 31 `raise`s stay unpinned: 8 are WRITER-bug detectors no input reaches, 2 are invariants the CLI gates ahead of them [2026-09-21]
The three converters of PR #1090 (`build_attach_substrate.py`, `build_a0_predictions.py`,
`paired_attach_diff.py`). **Round 2: 4 of 27 `raise`s went red when deleted. Round 3: 18 of 28. Now: 21 of
31** -- the three new ones are the input shapes that used to die with a bare `KeyError` at rc 1 (a gold or
corpus that is a JSON object with no `articles` key), and the split is not a judgement call: it is a
property of where each guard sits.

**Pinned, 21.** Every refusal reachable from an INPUT now has exactly one fixture that reaches it and
asserts the message, driven through the CLI in a subprocess: 11 of 16 in `build_attach_substrate.py` and 10
of 14 in `build_a0_predictions.py`. Each of the 21 is killed by exactly ONE named control, which is what
says the control under test is doing the killing rather than a refusal one layer down.

**Filed unpinned, 10, in three classes:**

1. **`build_attach_substrate.verify_round_trip`, all 5.** It re-reads the WRITTEN file, and `build_records`
   has already refused every input that could make any of them fire -- a duplicate join key, a key-set
   disagreement either direction, a content digest that does not match. So no gold, corpus or candidate
   file reaches them; only a bug in the WRITE step does. Pinning them needs a poison hook in shipped code
   (an env var that corrupts the write), which is a documented way to run the tool with its verifier
   defeated. Deleting the whole `verify_round_trip` CALL also leaves every control green.
2. **`build_a0_predictions.verify_scorer_reads_it`, all 3** (`articles_scored`, `extraction_miss`,
   `schema_invalid`-must-be-None). Same shape: `build_rows` emits one row per gold article carrying every
   one of that article's owner surfaces, and A0 rows are id-shaped by construction. The CALL itself IS
   pinned -- the `duplicate_join_key` control fires because the call's own `load_attach_gold` refuses a
   gold with two articles on one join key, which nothing in `build_rows` checks -- but the three raises
   inside it are not.
3. **Two internal invariants.** `reduce_baseline`'s `rule not in REDUCTIONS` is unreachable from the CLI
   (argparse `choices` gates it); it guards a direct API caller. `paired_bootstrap`'s "neither arm scored
   an owner" needs BOTH arms empty -- with one arm empty the union of article keys is non-empty and it
   never fires, which is why the CLI now refuses an empty arm itself, by name, at rc 2. That refusal is
   pinned (`cli_empty_arm`); without it the tool REPORTED a comparison against an arm that scored nothing,
   at rc 0.

**CLOSED 2026-09-21, and the entry that claimed it could not be closed was wrong.** An earlier revision of
this entry read that `_cluster_sums`'s zero-fill ("an arm missing an article gets a zero row rather than a
shifted one") could not be separated from a per-arm keying, because a separating fixture needs the two
arms' article sets to differ by a SYMMETRIC difference while "b a strict subset" was "the only shape either
arm can actually take against one gold". That premise is FALSE. Both arms are subsets of the GOLD's article
set, which does not make either a subset of the OTHER: a predictions file with no row for an article is
skipped by `resolve_attach_owners` and counted in `coverage.articles_skipped_missing_input`, so two arms
each missing a DIFFERENT article differ in both directions. Built from committed inputs (arm A0 `rows_sum`
against A0 `any_non_null` over `attach_gold_g2_v1.json`, each file with one article's row deleted, 120 rows
apiece, 105 clusters in the union) and run under both rules:

| arm coverage (105 clusters either way) | shipped, union-keyed | per-arm keyed | the 6 controls then shipping |
|---|---|---|---|
| both arms cover all 121 gold articles | `wrong_attachment_rate` ci95 [+0.0118, +0.0407] | the same output, byte for byte | OK under both |
| each arm missing a DIFFERENT article | ci95 [+0.0116, +0.0405], excludes 0 | ci95 [-0.0269, +0.0766], STRADDLES 0 | OK under both |

So the mutation flips gate 1's verdict ("the paired 95% CI of the difference excludes 0") on this input and
every control reported OK -- and the first row is the control for the control: with no coverage asymmetry
the two rules are the same numbers, so it is the FIXTURE that separates them, not the mutation.
`cluster_alignment` (`_alignment_fixture`, 4 articles, arm a covering 1-3 and arm b 2-4) now pins it
hermetically, and it is the only control that kills the per-arm keying, the percentile re-cut and the
2000-to-20 resample cut -- `identity` and `known_sign` assert DEGENERATE intervals ([0,0] and [1,1]), which
no cut can move.

**What stays unmeasured here**, precisely: every fixture in this suite is 4 articles or fewer, so no
control can see a mutant that misbehaves only at the committed gold's cardinality (121 articles, 526
owners). The table above is that check, run by hand, once -- it is not continuous, and re-running it is the
measurement: rebuild the two arms with `build_a0_predictions.py --reduction`, delete a different article's
row from each, and compare `paired_attach_diff.py`'s `metrics` under the shipped `_cluster_sums` call
against a copy whose `rows_a`/`rows_b` derive their own article order. Also unreached: the band path under
more than one `--a-alt` (`cli_sensitivity_band` uses one), and the in-process controls over a REAL arm --
CI only ever runs them over the synthetic owner set.

**Re-derive the pin counts, from the repo root.** `PYTHONDONTWRITEBYTECODE=1`, then for each of the three
files run its own `--selftest` with each `raise` neutered in turn and count the rc-0 runs. Replace the
whole `raise` STATEMENT with `pass` (an AST node span, not a text edit): `raise E(` -> `_ = E(`, the round-3
method, cannot express `raise E(...) from None` and leaves an EMPTY block wherever the raise is the only
statement in one, which the interpreter then refuses -- every suite goes red and reads as PINNED, which is
LESSONS L20 instance 2. py_compile each mutant first and report one that does not parse as NO VERDICT, not
as a pin. A NULL pass with no mutation must come back rc 0 first, or no red below it is readable (L20).

### `check_staleness.py` cannot grade a `--task attach` scorecard at all: 4 of 4 come back UNEXAMINED [2026-09-21]
The tool reads a `summary` block to find a scorecard's metrics. `score_attach` does not write one -- an
attach scorecard carries `metrics`, `diagnostics`, `controls` and `acceptance_evidence` and no `summary` --
so every attach card falls into the UNEXAMINED branch whose reason reads "parses, but carries no
summary/metrics block: either it is not a scorecard or the scorecard shape has drifted away from what this
tool reads". That reason names the right two possibilities and the answer is the SECOND one, for a whole
task.

**Measured 2026-09-21**, staging the round's cards into a scratch directory and running the tool at it:
`0/2 current; 2 stale`, both UNEXAMINED. Committing this round's four cards flat beside the cap8192 ones
would have taken the shared corpus from 27 files to 29 with the two non-scorecard `*.json` alone (the
paired outputs and the attestation, which are neither scorecards nor on the `SIDECAR_SUFFIXES` allow-list),
and added four more UNEXAMINED on top. They went into
`LlmBenchmark/eval-substrate/attach-reduced-round-2026-09-21/` instead, which the tool's NON-RECURSIVE
`glob("*.json")` never reaches -- correct for files it cannot grade, and it is why that directory's
README.md records the choice rather than leaving it to look like filing by taste.

**Why it matters rather than being cosmetic:** the attach round is the one whose numbers a pass-or-stop
decision was reported off, and it is the one task whose scorecards the staleness instrument is blind to. A
card that is never graded cannot go stale, which reads exactly like a card that is current.

**Closing it** is a shape decision, not a patch: either `score_attach` writes the `summary` block the tool
reads, or the tool learns the attach shape. Whichever, the re-check is the same -- put an attach scorecard
in a directory, run `check_staleness.py --scorecards <dir>`, and require a verdict that is not UNEXAMINED.
Widening the `glob` is NOT the fix and is deliberately out of scope here; that tool's own docstring says
widening it changes which files it claims to have checked.

### One attach scorecard carries two `schema_invalid` fields with different populations [2026-09-21]
Both are spelled `schema_invalid`, both are written by the same `--task attach` run, and they count
different things. A reader comparing arms, or counting pass gate 5 ("every arm: 0 schema_invalid"), has no
way to tell from the artifact which one they are holding.

| path in the scorecard | who computes it | does a FAILED CALL count? |
|---|---|---|
| `diagnostics.schema.schema_invalid` | `eval_harness._attach_schema_report`, off `resolve_attach_owners` | **No.** A row with `error` is counted in `rows_call_failed` and returns before any schema question |
| `acceptance_evidence.run_integrity.schema_invalid` | `eval_harness._run_integrity`, copied from `run_model`'s provenance | **Yes.** `run_one` sets `schema_valid = parsed_ok and ...` and `parsed_ok` is False whenever `error` is set |

Measured 2026-09-21 on ONE two-record arm, 1 HTTP 503 + 1 conforming pick, driven through the real
`run_model.main` with the HTTP boundary stubbed and scored through the real `resolve_attach_owners`:
**`run_integrity.schema_invalid` 1 of 2 rows judged; `diagnostics.schema.schema_invalid` 0 of 1 row
judged, `rows_call_failed` 1.** Re-derive by driving `run_model.main` with an engine that raises
`HTTPError(503)` for one of two records (the shape is in `test_run_model.should_count_only_the_parsed_rows_when_a_run_mixes_calls_and_no_calls`)
and reading both fields out of the resulting scorecard. Closing it means naming the two apart --
`schema_invalid_incl_failed_calls`, or dropping the runner's copy now that `schema_rows_judged` states its
population -- not silently redefining either.

### `attach_schema_valid`'s TRUE overrides a malformed shape, only the FALSE branch is tested, and its rationale is false [2026-09-21]
`return recorded if isinstance(recorded, bool) else attach_candidates.is_pick_shape_valid(prediction)`.
A row whose recorded flag is `true` is never re-judged, so a response the FIXED schema rejects is scored
valid and its picks are scored as attachments. Measured 2026-09-21 through `resolve_attach_owners`:
a row carrying `{"picks": [{"owner": "E1", "pick": "C1", "why": "x"}]}` (an extra key on a pick, which
`is_pick_shape_valid` rejects) with `schema_valid: true` scores **`rows_schema_valid: 1`,
`schema_invalid: 0`, and the pick CORRECT** -- the same input with no flag is refused. A second shape
gets a scorecard that contradicts itself in one block: `{"picks": "C1"}` with the flag true reports
**`schema_invalid: 0` beside `refusals.schema_invalid: 1`**.

`should_refuse_a_pick_the_runner_marked_schema_invalid_however_well_it_parsed` covers the FALSE branch
only. **Surviving mutants, measured**: replacing the `isinstance(recorded, bool)` check with `bool(recorded)`
or with `recorded is not None` leaves `test_eval_harness` 236/236 and `test_run_model` 186/186 green,
although the docstring states that a non-bool flag falls back rather than collapsing to False.

The docstring's rationale for the flag winning -- "it saw the raw text -- this scorer only ever sees the
parsed object" -- **is false**. `run_one` computes the flag as
`parsed_ok and attach_candidates.is_pick_shape_valid(obj)` where `obj` is the output of `parse_cod(content)`:
the runner judges the SAME parsed object, with the SAME shared function. The only thing it knows that the
scorer cannot recompute is whether `json.loads` succeeded at all, and a failed parse is already visible as
`prediction: null`. Closing it means either testing the TRUE-over-malformed branch as intended behaviour and
rewriting the rationale, or re-judging the shape here and keeping the flag as provenance.

### Three BLINDNESS findings from #1090 were recorded as filed with measurements, and were not filed at all [2026-09-21]
**The process defect is the entry, and this PR does not close it.** #1090's approve reason states the three
BLINDNESS findings accepted at merge were "routed to the backlog with re-checkable measurements". No such
entries were written, and the claim was not checked before it was recorded, so a false statement about the
state of this file is permanent in that PR's audit log. The three are written out below with the
measurement each should have carried; two are closed by the PR that adds this entry and one is partly open.
They are one entry rather than three because their shared provenance -- a verdict that asserted a filing
nobody performed -- is the thing worth being able to find again.

**1. The honest-check guard was unpinned. CLOSED by this PR.** `verify`'s `target is None` branch is the
only thing separating "cannot check" from "checks out", and deleting its reporting left every control green
at rc 0. Re-checkable on the committed set by running the tool with the WRONG repo root, which is the
operator error the branch exists for:

```bash
D=LlmBenchmark/eval-substrate/attach-reduced-round-2026-09-21
python3 $D/stage_and_verify.py --verify $D --repo-root .    # shipped: 9 unresolvable, FAILED, rc 2
```
With the reporting deleted the same command prints `4 recorded digest(s) resolved … 0 unresolvable … OK`
at **rc 0** -- 4 resolved either way, and the 9 pairs it cannot check simply stop being mentioned. Now
pinned by `control cli_missing_reported`, which drives the CLI and asserts rc 2 and the named file.

**2. Every control asserted an internal function's return. PARTLY CLOSED.** At `629a8133` the file had 3
control functions printing 5 legs, none of which spawned the CLI -- against a workflow comment that states
the opposite standard for its three sibling scripts. It now prints **23 legs from 12 leg-printing control
functions (plus the `cli_control` aggregator, which prints none), of which 18 legs from 9 functions drive
the CLI end to end** and assert its exit code and stdout. Re-check:
`python3 $D/stage_and_verify.py --selftest | grep -c '^control '` -> 23, and `| grep -c '^control cli_'`
-> 18; at `629a8133` the same two commands give 5 and 0.

**Those four figures were WRONG here until 2026-09-21, and this entry's own thesis is why that matters.**
They were recorded as 18/13 over 9/6 at the commit that wrote them and never re-run at the head that
shipped them; the next commit on this branch added controls and falsified all four, while handing the
reader the two commands that refute them. An entry titled for claims recorded without checking is the last
place that may happen. RE-RUN EVERY COMMAND IN AN ENTRY AT HEAD, not at the commit that wrote it.
**What remains: the 5 in-process legs, and they cannot be converted.** `stage()` -- the transform the file
exists to record -- is reachable from `_stage_all` only, which is reachable from `control_noop_bytes` and
`control_digest_resolves` only; **no CLI path reaches it**, because there is deliberately no `--stage`
(the pre-staging originals are not committed). So `control_noop_bytes` is structurally un-drivable end to
end, and the byte-preservation rule it guards is asserted only in process. Re-check:
`grep -n 'stage(' $D/stage_and_verify.py` -> the only call site outside its own definition is inside
`_stage_all`. Closing it means either exposing a staging CLI with no input, or accepting the limit --
this entry exists so the choice is made rather than defaulted.

**3. The verifier was wired into no gate. CLOSED by this PR**, and recorded here only because the verdict
claimed it had been filed: both `--selftest` and `--verify` now run in the `Attachment round tooling
selftests` step of `.github/workflows/python-tests.yml`, and breaking one committed artifact's bytes takes
that step to rc 2 under `bash -e`. Not re-opened.

### Where the attach-round checker's pin chain terminates: three deletions nothing turns red on [2026-09-21]
Not the advisory-CI point (CLAUDE.md VERIFY already states that a red check can be seen and cannot block on
this plan). These are three specific deletions that leave EVERY channel at rc 0, measured after the
round-2 and round-3 pins were in place, and RE-DERIVED at `20d1540d` -- unit = one test collected by
`python -m pytest scripts/tests` in a venv, population = every file under `scripts/tests`, read
2026-09-21T13:55Z. The counts below were first taken at the branch base `6a77daf3`, which collects 154 and
becomes 155 once this PR's own out-of-band test file is added; they are not those numbers at head, which
is its own entry further down.

`scripts/tests/test_attach_round_selftest_wiring.py` holds a LITERAL roster, `EXPECTED_LEGS`, compared with
set equality against what `stage_and_verify.py --selftest` prints. That is what kills a control deleted on
its own, and what a derived roster could not. It does not survive the roster and the control being deleted
together, and it does not survive itself being deleted.

| deletion | `--selftest` | `pytest scripts/tests` | what moved | diff a reviewer sees |
|---|---|---|---|---|
| none (control) | rc 0 | rc 0, 161 passed | 23 legs | -- |
| a control **and** its `EXPECTED_LEGS` entries, together | rc 0 | rc 0, **161 passed** | 23 legs -> 22, 21 or 20 | **27 to 57 changed lines**, all nine measured |
| the whole out-of-band test FILE | rc 0 | rc 0, **160 passed** | 23 legs, one fewer test | **96 deletions** |
| the two `run:` lines in `.github/workflows/python-tests.yml` | rc 0 | rc 0, **161 passed** | nothing | **4 physical lines** |

The second row is the general shape: a pin whose expectation is a literal is only as strong as review of
the diff that changes the literal. The third is pytest's contract -- collecting one fewer file is not an
error.

**The fourth is PRE-EXISTING AND UNIVERSAL, and this PR cannot close it.** Deleting the two commands that
invoke the round's checker leaves every suite green, because **no test in this repo asserts on any
workflow's `run:` commands at all**. THE CONCLUSION SURVIVES RE-DERIVATION AND THE POPULATION DOES NOT.
The tracked files whose CONTENT matches `.github/workflows` outside `.github/` itself are **eleven**, not
five -- fourteen counting the three workflow files themselves -- unit = one tracked file, read 2026-09-21
at `20d1540d`. The five named before are exactly the eleven filtered to a `tests/` directory, a filter that
was applied without being stated, and the entry shipped with no command to re-run:

```bash
git grep -lI --fixed-strings '.github/workflows' -- . ':(exclude).github/*'   # 11 -- the real population
git grep -lI --fixed-strings '.github/workflows' -- '*tests/*' ':(exclude).github/*'   # 5 -- the named set
```

The six the filter dropped are prose: `.claude/hooks/README.md`, this file, `scripts/README.md`,
`scripts/sentinel-quality-check/README.md`, `LlmBenchmark/scripts/README.md`, and the round's own
`LlmBenchmark/eval-substrate/attach-reduced-round-2026-09-21/README.md`. All eleven were re-read at
`20d1540d`: every mention sits in Markdown prose, a `#` comment or a docstring, and none opens or parses a
workflow file. Widening to any mention of `.github` at all adds four more files and still no parser --
three `api.github.com` URLs in hook smoke tests, one `docs.github.com` URL in a cached search fixture.
Every CI step in this repo is in that position, not only these two. Re-check by deleting the two lines and
running `python -m pytest scripts/tests` in a venv.

Re-check any row by making the deletion and running both commands; the figures above are the whole test.

**Why it is filed and not closed.** A test-count floor would put the expected count under test in its own
control, which is the failure this repo has already measured twice (see the MEASUREMENT DEBT entries on
harnesses that fail toward success). The remedy left is that all three deletions are VISIBLE in review --
but NONE of them is a one-line diff. Measured at `20d1540d`, unit = changed diff lines (`git diff
--numstat`, insertions plus deletions), population = all nine `cli_*` controls for the first row, each
deleted with its `cli_control()` entry and its `EXPECTED_LEGS` lines and each reverted:

| row | changed lines | population |
|---|---|---|
| a control with its `EXPECTED_LEGS` entries | **27 to 57**, median 42 | nine `cli_*` controls; 27 `cli_unparseable_refused` (1 leg), 57 `cli_duplicate_sites` (3 legs); the five two-leg controls that give the 23 -> 21 move span 40 to 54 |
| the out-of-band test FILE | **96 deletions** | the file is 96 lines |
| the two workflow `run:` lines | **4 physical lines** | two commands, the second wrapped over three |

All nine control deletions survive BOTH channels -- `--selftest` rc 0, `pytest scripts/tests` rc 0 at 161
passed -- so the leg count is the only thing that moves and nothing reads it. Every one of the three
deletions is therefore MORE visible than "one-line" claimed: the error ran in the SAFE direction, which is
why it was residue and not a blocker. Naming them here is what makes a reviewer look.

### The pick prompt and the pick schema have ZERO refusing sites, so deleting either leaves `--verify` OK at rc 0 [2026-09-21]
**ABSENCE, not a wrong digest.** `verify` refuses an absent file only when the sibling key it resolved
through is literally `path`; every other spelling is counted UNREAD and the run stays green. Of the 35 READ
sites, unit = one digest-valued JSON string in
`LlmBenchmark/eval-substrate/attach-reduced-round-2026-09-21/*.json`, measured at `20d1540d`:

| named artifact | READ sites | of which REFUSING (`path`-sited) | sibling key used | digest key |
|---|---|---|---|---|
| `attach-g2-20260921.staleness.json` | 4 | **4** | `path` | `sha256` |
| `attach-gold/attach_gold_g2_v1.json` | 9 | **5** | `path` x5, `gold_path` x4 | `sha256`, `gold_sha256` |
| `attach-gold/frozen/attach_candidates_g2_k20.json` | 10 | **4** | `path` x4, `file` x6 | `sha256` |
| `attach-pick/pick_prompt_clean.txt` | 6 | **0** | `prompt_file` | `prompt_file_sha256` |
| `attach-pick/pick_schema.json` | 6 | **0** | `schema_file` | `schema_file_sha256` |

22 of the 35 sites cannot refuse; what makes 12 of those 22 worth an entry is that they are the ONLY two
artifacts with no refusing site at all -- the other three keep 4, 5 and 4 -- and they are the two files
`.github/workflows/python-tests.yml` names `LlmBenchmark/attach-pick/**` in its path filter FOR, so the
edit that is meant to make this job run is the one the job's own evidence-chain checker cannot see removed.
Measured, each reverted:

| what was done to `attach-pick/` | `--verify` verdict | rc | coverage line |
|---|---|---|---|
| nothing | `-- OK` | 0 | 35 of 77, 42 unread |
| `pick_prompt_clean.txt` deleted | `-- OK` | **0** | **29** of 77, 48 unread |
| `pick_prompt_clean.txt` renamed | `-- OK` | **0** | **29** of 77, 48 unread |
| both deleted | `-- OK` | **0** | **23** of 77, 54 unread |
| one byte flipped in `pick_prompt_clean.txt` | `-- FAILED` | **2** | 35 of 77, **6 MISMATCH lines** |

**CORRUPTION IS CAUGHT; only ABSENCE is not.** The gap is that an absent file is reclassified as
"no sibling I can resolve" and joins a counted total, which is the same words a deliberately-uncommitted
artifact earns.

**WHAT DOES CATCH THE ABSENCE, so this is not filed as invisible.** `LlmBenchmark/scripts/test_run_model.py`
reads both files and turns rc 1, unit = one `FAIL` line, population = its 186 tests, read 2026-09-21 at
`20d1540d`: **14** FAILs with both deleted (`passed: 172/186`), **13** with `pick_prompt_clean.txt` renamed
OR deleted on its own (`passed: 173/186`), against 0 FAILs and `passed: 186/186` rc 0 on the untouched tree.
But it does NOT see the corrupted byte -- with byte 0 of the prompt flipped it reports 0 FAILs and
`passed: 186/186` at rc 0. The two channels are complementary and NEITHER covers both: the harness sees the
file go away and not the edit, the checker sees the edit and not the file going away. Count the `FAIL`
lines, never read them off a `tail` -- the first derivation of this row did, and reported 4 and 3.

**Closing it** means giving the two `attach-pick` sites the round's own attestation spelling -- a
`{path, sha256}` pair, or `prompt_file`/`schema_file` promoted into the refusing branch -- which is a writer
change on the next round, not this one. Re-check:
```bash
D=LlmBenchmark/eval-substrate/attach-reduced-round-2026-09-21
mv LlmBenchmark/attach-pick/pick_prompt_clean.txt /tmp/held
python3 $D/stage_and_verify.py --verify $D --repo-root LlmBenchmark | tail -2   # still OK, rc 0, 29 of 77
mv /tmp/held LlmBenchmark/attach-pick/pick_prompt_clean.txt
```

### Eight behaviour-preserving mutants of `stage_and_verify.py` survive BOTH channels, because no fixture separates the candidate rules [2026-09-21]
Not the mutation battery in `.github/workflows/python-tests.yml`, whose 21 mutants are each recorded there
as killed by a named control. These eight change which RULE the tool implements while leaving `--verify`'s stdout **byte
identical** on the committed set, so nothing in the repo can state which rule shipped. Unit = one distinct
source edit, each `py_compile`d first; scored at `20d1540d` against a baseline of `--selftest` rc 0 / 23
legs and `pytest scripts/tests` rc 0 / 161 passed, every rc read from the process itself.

| # | mutant | why no fixture separates it |
|---|---|---|
| N1 | refusal predicate keyed on the DIGEST KEY (`sha256`) instead of on the sibling being `path` | the three fixtures pairing an ABSENT file with a sibling use `path` (`cli_missing_reported`, `cli_nonrefusing_split_path_refuses`) or a `<stem>_sha256` key (`cli_nonrefusing_split_other_counts`); **none uses `{file, sha256}` with an absent file**, which is the one shape where the two rules disagree -- and the committed set has **6 such sites**, all naming the frozen candidates |
| N2 | root ORDER reversed, repo root consulted before the round directory | 0 names resolve to a DIFFERENT file under both roots; `cli_repo_root`'s fixture puts its gold under one root only |
| N3 | the `sha256:` prefix strip dropped from the comparison | 0 of the 35 READ sites carry a `sha256:`-prefixed digest -- the 10 that do are `generator_version`, UNREAD by construction, which is exactly what `cli_coverage_reported`'s fixture exercises |
| N4 | `glob` -> `rglob` over the round directory | 0 `*.json` under any subdirectory of the round dir; `_card_dir` builds flat fixtures |
| N5 | `stage()`'s non-`.json` copy branch deleted | `_fixture` writes three `.json` files and nothing else, so the branch has no input -- and `stage()` is reachable from no CLI path at all (its own entry above) |
| N6 | the empty-string sibling guard dropped from `_digest_sites` | 0 conventional siblings hold `""` in the committed set, and no fixture writes one |
| N7 | bare-`sha256` sibling preference reversed, `file` before `path` | 0 objects carry BOTH `path` and `file`; `cli_sibling_conventions` gives each shape its own document key with one sibling each |
| N8 | `<stem>` sibling preference reversed, `<stem>_path` before `<stem>` | 0 objects carry BOTH `<stem>` and `<stem>_path`, same fixture shape |

All eight: `--selftest` rc 0 at 23 legs, `pytest scripts/tests` rc 0 at 161 passed, `--verify` rc 0 with
stdout byte-identical to the unmutated baseline. **The runner reports its own dullness** -- three known-bad
controls run through the same harness: `verify()` returning `False` unconditionally comes back KILLED
(selftest rc 3, pytest rc 1, 160 passed), the M1-shaped deletion of the honest-check reporting comes back
KILLED (rc 3 / rc 1), and an unparseable edit comes back NO VERDICT rather than SURVIVED. 3 of 3 behaved.

**N1 is the one worth closing first** and it is cheap: the repo's own rule is to build the fixture where
the two candidate rules DISAGREE (CLAUDE.md TOOL_UPKEEP), and the disagreeing input is already committed --
a `{file: <absent>, sha256: ...}` document would refuse under one rule and count UNREAD under the other.
Adding it would also decide, rather than default, whether the frozen candidates' 6 `file`-sited digests are
part of the round's attestation contract. The other seven are judgement calls about scope; what is filed is
that today nothing in the repo records which way any of them was decided.

**Re-check.** Take the baseline, apply ONE mutant from the table, and re-run all three: stdout identical
and both channels green means it survived. Revert before the next one.
```bash
D=LlmBenchmark/eval-substrate/attach-reduced-round-2026-09-21
python3 $D/stage_and_verify.py --selftest | grep -c '^control '            # 23
python3 $D/stage_and_verify.py --verify $D --repo-root LlmBenchmark > /tmp/attach-verify-base.txt
python -m pytest scripts/tests -q | tail -1                                # 161 passed, in a venv
```
The disagreement fixture N1 is missing is one document: `{"gold": {"file": "absent.json",
"sha256": "<any 64 hex>"}}` in a directory `--verify` is pointed at. Shipped, it counts UNREAD at rc 0;
under N1 it refuses at rc 2. No committed or fixture document has that shape today.

### The paired outputs' `.a` and `.b` are the 2 of 5 redacted fields with NO sibling digest [2026-09-21]
The round README's REDACTION section is corrected in the PR that files this, so what remains OPEN is the
gap the sentence was hiding, not the sentence. Measured at `20d1540d`, unit = one redacted path field:

| redacted field | documents | digest keys in the SAME object |
|---|---|---|
| `.substrate` (provenance sidecars) | 3 | `substrate_sha256` |
| `.adapter_metadata.substrate` (scorecards) | 3 | `substrate_sha256` |
| `.staleness.path` (scorecards) | 4 | `gold_sha256`, `sha256` |
| `.a` (paired outputs) | 3 | **NONE** |
| `.b` (paired outputs) | 3 | **NONE** |

Each paired output carries exactly ONE digest-valued string anywhere in it, `.bootstrap.replicates_sha256`,
an in-memory bootstrap draw that attests neither arm. So 6 sites naming the two arms of the comparison that
gates 1 and 2 are stated on sit outside `--verify`'s numerator entirely. `.a` names `a0_g2.jsonl`, which is
tracked nowhere in the repo (its digest is recorded in the round README's not-committed table); `.b` names a
committed predictions file whose digest is recorded only in `LlmBenchmark/scripts/README.md`, in another
directory. **Closing it** means the writer emitting `a`/`a_sha256` and `b`/`b_sha256`, which brings both
inside the reader's existing `<stem>_sha256` convention at no cost to the reader -- a next-round change.
Re-check:
```bash
D=LlmBenchmark/eval-substrate/attach-reduced-round-2026-09-21
grep -ohE '"[a-z0-9_]+": "(sha256:)?[0-9a-f]{64}"' $D/attachpaired-*.json \
  | cut -d'"' -f2 | sort | uniq -c
# -> `3 replicates_sha256` and no other digest key: nothing sits beside .a or .b
```

### Three UNANCHORED figures in this file are false at head, and only one of them is a test count [2026-09-21]
**Recurrence, not a first occurrence.** The entry titled for #1090's unfiled BLINDNESS findings already
records four figures that were right at the commit that wrote them and wrong at the head that shipped them,
and closes with `RE-RUN EVERY COMMAND IN AN ENTRY AT HEAD, not at the commit that wrote it`. The pin-chain
entry beside it, written in the same PR, shipped with four more. Derived 2026-09-21 at `20d1540d`, unit as
stated per row:

| site | states | measured at `20d1540d` | the finding |
|---|---|---|---|
| the pin-chain entry's four table cells and its summary row (this section) | 155 / 155 / 154 / 155 collected by `pytest scripts/tests` | **161 / 161 / 160 / 161** | holds; corrected in this PR |
| `The pin control's ALREADY-RED arm has no test` (this section), stated twice -- in prose and as its re-check's expected output | `pytest scripts/tests` "at 154 passed" | **161 passed** | holds; but a reader running the re-check gets 161 and cannot tell a moved denominator from a failed re-check |
| KNOWN DEFECTS, the hook-registry block: "`.github/workflows/` holds alert-rules, python-tests, sync-docs **and nothing else**" | three workflow files | **four** -- `hosted-service-pins.yml` landed at `0970bc09` (#1087, 2026-09-20) | holds; no workflow runs the hook suites |
| KNOWN DEFECTS, the merged-tree block: `grep -l dotnet .github/workflows/*.yml \| wc -l   # 0` | 0 | **1**, and the one match is `hosted-service-pins.yml`'s comment `no dotnet, no container` | holds; but the one-liner now fails toward CLOSED -- a reader gets 1 and reads the defect as fixed, off a grep that drifted onto a comment |

Three sites, four copies. The two in KNOWN DEFECTS share one cause -- a fourth workflow landed -- and
NEITHER is a test count.

**THE DISCRIMINATOR IS ANCHORED VS UNANCHORED, not absolute vs delta.** The file's most alarming-looking
absolute is not stale at all: `Three figures the PR-verdict decision check leaves un-re-checkable` says its
`#NNN` sweep saw 55 and 51 against 80 in this file at `20d1540d`, and it is EXACT at both commits it names
-- 55 at `40d372eb`, 51 at `b08ccdef`, re-derived 2026-09-21. An absolute stamped with its commit never
rots; an unanchored delta would, and a delta-and-rc rule would have saved neither KNOWN DEFECTS row above.
**Closing it** means applying the rule this file already states in `Citations in tracked .md that cannot
land are the corpus's steady state` -- *stamp any figure with the MERGED sha, never a branch* -- to the four
copies above, one edit each, which is what every entry this PR adds practises. Re-check:
```bash
python -m pytest scripts/tests -q | tail -1                  # in a venv; pytest is not on this host's python
ls .github/workflows/ | wc -l ; grep -l dotnet .github/workflows/*.yml | wc -l
git show 40d372eb:docs/BACKLOG.md | grep -oE '#[0-9]+' | sort -u | wc -l   # 55, and b08ccdef -> 51
```

### 42 of the round's 77 digest-valued strings are still unidentified, after the reader went from 13 to 35 [2026-09-21]
**What changed, and why the first version of this entry was wrong.** `_digest_pairs` keyed on ONE spelling
-- a `sha256` string beside a `path` string -- and read 13 of the 77 digest-valued strings in the round
directory. This entry previously argued the only fix was for `run_model.py` and `eval_harness.py` to emit
one shape, citing CLAUDE.md GIGO. **That was a misreading of GIGO**, which is about rejecting junk where it
is born rather than gating each destination, and whose stated cost is that each new consumer re-learns the
rule. Here nothing is junk -- the producers emit correct digests under conventional, schema-specific names
-- and there is exactly ONE consumer. A reader that understands more field names is a reader, not a
destination gate.

The argument also rested on an assumption never stated: that the reader must REFUSE what it cannot
resolve, which is what forced a four-item exclusion list and made the change look expensive. Drop that
assumption and the exclusions disappear.

**THE RULE NOW SHIPPED, non-refusing.** Sibling of `<stem>_sha256` is `<stem>`, then `<stem>_path`; bare
`sha256` tries `path`, then `file`. If a sibling names a file that is there, the digest is compared. If no
sibling exists, or it names nothing on disk, the site is counted UNREAD on the `coverage:` line -- never
refused. The ONE exception is the round's own attestation contract: a `path`-sited pair whose file is
absent still REFUSES at rc 2, because that is the honest-check guard this tooling exists for. An escape
(`..` or a symlink out of the declared roots) refuses under both.

| measured 2026-09-21, unit = one digest-valued JSON string in `attach-reduced-round-2026-09-21/*.json` | before | after |
|---|---|---|
| READ (digest compared) | 13 | **35** |
| unresolvable (refused) | 0 | 0 |
| UNREAD (counted and named) | 64 | **42** |
| mismatches among the read | 0 | **0** |
| distinct (name, digest) pairs read | 3 | 5 |
| committed bytes altered / writer changes | — | **none** |

The 22 newly-read sites name two artifacts that were previously invisible: **12 of them**
`attach-pick/pick_prompt_clean.txt` (6) and `attach-pick/pick_schema.json` (6) -- the files the workflow's
own path filter lists so that an edit there runs this job. Before this change the job fired, `--verify`
ran, and the digests those sites record for that very file were not read.

**What is STILL unread, and why each is not a defect of the reader:**

| spelling | sites | why |
|---|---|---|
| `substrate_sha256` | 10 | `substrate_g2.json` is deliberately NOT committed -- regenerable from committed inputs (round README). Non-refusing is what keeps this a counted absence instead of a red run |
| `generator_version` | 10 | key is not `_sha256`-suffixed; its value is `sha256:`-prefixed instead -- its own entry below |
| `chat_template_sha256` | 9 | the sibling is the template STRING, not a path -- there is no file |
| `replicates_sha256` | 7 | an in-memory bootstrap draw; no sibling, no file |
| `prompt_file_sha256` | 3 | the 3 sites under `acceptance_evidence.request_bytes` carry no sibling naming the file |
| `schema_file_sha256` | 3 | same three sites |

**Re-check**, and this is the number to watch rather than any figure in this entry:
```bash
D=LlmBenchmark/eval-substrate/attach-reduced-round-2026-09-21
python3 $D/stage_and_verify.py --verify $D --repo-root LlmBenchmark | tail -1
```
**Closing the remainder** means a writer change after all, but a much smaller one than the first version of
this entry claimed: give the three `acceptance_evidence.request_bytes` sites a sibling, and spell
`generator_version` as `generator_sha256`. Both are on artifacts that are already frozen, so they land on
the next round, not this one. Nothing else in the table can be closed by a reader or a writer -- those
digests are of things that are not files.

### The frozen candidates' `generator_version` matched its generator when the freeze ran, and stopped 19 minutes later [2026-09-21]
**The value, and what is actually wrong with it.** `freeze_candidates.py:203` writes
`"generator_version": "sha256:" + sha256(own bytes)`, and its sibling `generator` names that file.

| measured 2026-09-21 | value |
|---|---|
| `sha256:`-prefixed values under `LlmBenchmark/` | 12 occurrences, **1 distinct value** |
| ...in `eval-substrate/attach-reduced-round-2026-09-21/` | 10: 3 provenance sidecars x1, the 3 B-clean scorecards x2, the A0 scorecard x1 |
| ...in `attach-gold/frozen/attach_candidates_g{1,2}_k20.json` | 2, one each -- the SOURCE the other 10 are copied from |
| recorded value | `0b394c08a064784a3fdb0ec9bfa1b10d388dc8f61051fbe3e1235fee0f1c46d8` |
| `sha256sum` of the generator in the tree today | `81bdbeee4960d77161c351a89ace7d32e8a1720ad4c24f50ba1bf3a4acc6e122` |

**IT WAS TRUE WHEN IT WAS WRITTEN, and the command that shows it is not the obvious one.** `git log --
<path>` walks only what is reachable from HEAD, and on a squash-merging repo the pre-squash blobs are not,
so it reports ONE commit and invites the conclusion that the digest never matched. `git rev-list --all`
returns four, and two of them carry a generator hashing to exactly the recorded value:

```bash
P=LlmBenchmark/attach-gold/frozen/freeze_candidates.py
git rev-list --all -- $P | while read c; do
  printf '%s %s %s\n' "$(git rev-parse --short $c)" \
    "$(git show $c:$P | sha256sum | cut -c1-16)" "$(git log -1 --format='%ad %s' --date=iso $c)"
done
```

| revision | generator sha256 | when |
|---|---|---|
| `c64d5593` (#1074, the squash on main) | `81bdbeee…` | 2026-09-20 09:19:41 |
| `a9831b59` (`benchmark/attach-freeze-v2`) | `81bdbeee…` | 2026-09-20 09:07:06 |
| `f37e658b` (`benchmark/attach-freeze-v2`) | **`0b394c08…`** | 2026-09-20 08:48:05 |
| `3c4dbb44` (`origin/benchmark/attach-freeze`) | **`0b394c08…`** | 2026-09-17 09:17:35 |

`f37e658b` is the freeze that produced the committed file: its `attach_candidates_g2_k20.json` is
byte-identical to main's (`4ee067db697a…`, the value the round's provenance records as `candidates.sha256`).
**19 minutes and 1 second later** `a9831b59` added a `control()` function to the generator -- a known-bad
control on the degraded-owner refusal -- and the candidates were never re-frozen, so the squash carried a
NEW generator with the OLD self-digest. `git diff f37e658b a9831b59 -- $P` shows exactly that addition.
Nothing in the reduced round computed the field: `generator_version` is one of
`attach_candidates.AXES`, `attach_candidates.load` returns it under `axes`, and `run_model.py`'s
`candidates_provenance` spreads `**frozen["axes"]` into the provenance the scorer then copies.

**So the prescription is cheap, and re-freezing is NOT it.** The field can be re-attested from an
identified revision with no run and no live catalog: record the generator's revision beside the digest --
`f37e658b`, or `3c4dbb44`, which carries the identical generator blob and is the one of the two that is
PUSHED (`origin/benchmark/attach-freeze`); `f37e658b` is reachable only from a local branch and dies with
it. Re-freezing against a live SecMaster would replace two frozen inputs three committed scorecards are
scored against, and dropping the field would discard provenance that is correct. Spelling the key
`generator_sha256` on a future round would also bring it inside the reader's sibling convention at no
cost. Today it is counted UNREAD on the `coverage:` line -- 10 sites -- which is why `--verify` stays rc 0
on it rather than either hiding it or going red forever.

**PRE-EXISTING, verified before writing:** the value is byte-identical in `git show
c64d5593:LlmBenchmark/attach-gold/frozen/attach_candidates_g2_k20.json`, and `git rev-list --all` shows no
commit to that file after `c64d5593`.

### The `--task cod` and `--task cove` readers coerce the tri-state `schema_valid` to False, so NO VERDICT reads as INVALID [2026-09-21]
`AttachOwner.schema_valid` is `bool | None` (`LlmBenchmark/scripts/eval_harness.py:1066`) because a row that
made no call has no schema verdict. The two older readers are not, and `None` is falsy:

| reader | `--task` | the coercion |
|---|---|---|
| `_build_cod_predictions`, def at `LlmBenchmark/scripts/eval_harness.py:2455` | `cod` | `LlmBenchmark/scripts/eval_harness.py:2487` -- `else bool(pred_row.get("schema_valid",` |
| `_load_predictions_jsonl`, def at `LlmBenchmark/scripts/eval_harness.py:2861` | `cove` | `LlmBenchmark/scripts/eval_harness.py:2893` -- `schema_valid=bool(pred_obj.get("schema_valid", _is_schema_valid(pred_obj["predicted_extractions"]))),` |

**The fallback written beside each coercion does not save it.** `dict.get(key, default)` returns the STORED
`None` when the key is present and null -- the default expression is used only for an ABSENT key -- so
`_is_cod_schema_valid` / `_is_schema_valid` never run on a null flag. Measured 2026-09-21 by handing the
cove reader a cove-shaped row carrying `schema_valid: null` and `predicted_extractions: []`: **it returns
`schema_valid False`, while `_is_schema_valid([])` returns `True`.** On that input the coercion does not
merely lose the tri-state, it contradicts the judgement it is written beside.

Carrying the tri-state here is not a one-line edit. `RecordPrediction.schema_valid` and
`CodRecordPrediction.schema_valid` are both declared `bool`
(`LlmBenchmark/scripts/eval_harness.py:281`, `LlmBenchmark/scripts/eval_harness.py:717`), and both scorers
divide by counts taken off that field.

**Why it is deferred rather than fixed.** Two reasons, either sufficient:

1. **The failure is CONSERVATIVE.** A no-verdict row reads as schema-INVALID, never as valid, so it cannot
   turn a malformed arm into a passing one -- the direction pass gate 5 is counted in. CLAUDE.md
   §TOOL_UPKEEP ("SHARP ENOUGH, NOT RAZOR": judge a remaining defect by whether it MISLEADS or is merely
   IMPERFECT) puts this on the IMPERFECT side: it understates quality, it does not flatter it.
2. **Nothing routes a null flag there today.** The only row-level `no_call` write is
   `LlmBenchmark/scripts/run_model.py:1296`, inside the `args.task == TASK_PICK` guard at
   `LlmBenchmark/scripts/run_model.py:1281`, and a pick predictions file is the `--task attach` input.

**The reachability is asymmetric between the two readers**, so "no `no_call` row exists on those tasks" is
exact for the PRODUCER and not for the CLI:

- **cod -- reachable by misdirection, and meaningless when it happens.** Nothing refuses
  `--task cod --predictions <pick file>`: the attach path detects pick-shaped rows and demands
  `--candidates` (`LlmBenchmark/scripts/eval_harness.py:2563`), the cod path at
  `LlmBenchmark/scripts/eval_harness.py:2786` has no equivalent check. But such a run mis-scores whatever
  this coercion does -- `_cod_object` hands the CoD scorer `{"picks": []}`, which carries none of the five
  `_COD_KEYS` (`LlmBenchmark/scripts/eval_harness.py:2393`), and `_is_cod_schema_valid({"picks": []})` is
  False on its own terms. Fixing the coercion alone would make an already-meaningless scorecard honest in
  one field.
- **cove -- unreachable by any producer.** A `no_call` row carries `prediction`, not
  `predicted_extractions`, so `_load_predictions_jsonl` raises `KeyError: 'predicted_extractions'` before
  the coercion evaluates. That coercion is latent, waiting on a future producer that emits a null flag on a
  cove-shaped row.

**Re-check**, from the repo root:

```
python3 - <<'EOF'
import sys; sys.path.insert(0, "LlmBenchmark/scripts")
import eval_harness as eh
sub = [{"source_file": "f", "source_index": 0, "input": {"content": "t"},
        "output": [], "is_negative": False}]
row = {"source_file": "f", "source_index": 0, "prediction": {"picks": []},
       "schema_valid": None, "no_call": "no_owners"}
print(eh._build_cod_predictions(sub, None, {("f", 0): row}, mock=False)[0][0].schema_valid)
EOF
```

**2026-09-21 it prints `False`** -- 1 of 1 rows, a recorded `None` arriving as the boolean `False`. It is
closed when that prints `None`. Two static halves on the same date: the coercion count,
`grep -c 'bool(pred_row.get("schema_valid"\|bool(pred_obj.get("schema_valid"' LlmBenchmark/scripts/eval_harness.py`
is **2, which is 2 of 2 reader sites**; and the reachability,
`grep -n no_call LlmBenchmark/scripts/run_model.py` is **6 hits, exactly 1 of them a row-level write**
(`:1296`, pick-gated), the other 5 being 2 reads (`:1748`, `:1829`) and 3 comments -- a second write
outside that guard would retire the deferral above.

Closing it means giving both readers the `bool | None` the attach path already carries, widening the two
record types with it, and counting a no-verdict row into a third population rather than into the complement
of the counts at `LlmBenchmark/scripts/eval_harness.py:297` (divided by `n_total` at
`LlmBenchmark/scripts/eval_harness.py:375`) and `LlmBenchmark/scripts/eval_harness.py:866`.
**Do not copy `_pick_owners`' fix across.** That call site was correctly
moved from `bool(schema_valid)` to `schema_valid is not False`, because there the flag GATES a refusal and a
no-verdict row must not be refused. Here the same field is the NUMERATOR of a validity rate, so `is not
False` would count the row as VALID -- the identical fabricated measurement, inverted.

### The hosted-service pin gate is blind to the factory overload, and to all of Reports [2026-09-20]
`scripts/verify-hosted-service-pins.py` keys `AddHostedService<T>` and nothing else. Counted from the data side
rather than from the instrument — every `AddHostedService` token minus the ones the tool keys — **5 live sites
use `AddHostedService(sp => sp.GetRequiredService<T>())`**: all three Reports hosts (`Reports.DailyHost`,
`Reports.WeeklyHost`, `Reports.MonthlyHost`), SecMaster's `EdgarIngestionBackgroundService` and
FinnhubCollector's `BackgroundCollectionQueue`. **Reports has no keyable registration at all, so that service is
entirely outside the gate.**

Keying that one spelling was rejected rather than deferred: it would read as coverage of the construct while
`AddHostedService(sp => new Foo())` and every other factory shape stayed silent, and an enumeration always grows
another member (`.claude/skills/guard-change/SKILL.md` item 9). The count prints under `NOT COVERED` on every
run, a clean one included, so the blind spot carries a live number instead of a paragraph.

Also unwatched and deliberately NOT counted here, because they are different constructs rather than the same one
spelled differently: `AddSingleton<IHostedService, T>` (zero instances today, checked), Quartz job registration,
`AddMeter` / `AddSource`, middleware, and anything registered by reflection or assembly scanning.

Re-check: `python3 scripts/verify-hosted-service-pins.py` — the `NOT COVERED` line; `--list` names each site.

### A mutation harness that rewrites ONE `.py` path scores a FALSE SURVIVOR on the bytecode cache [2026-09-20]
**CPython validates a cached `__pycache__/*.pyc` against the source's mtime TRUNCATED TO WHOLE SECONDS and its
size — nothing else. Two revisions of one file with EQUAL BYTE LENGTH written inside the same second are
indistinguishable to that check, so the second import silently runs the FIRST one's bytecode.** A mutant that
was never actually executed then reads as SURVIVED, and every later verdict in that run rests on a tree the
interpreter did not read. Measured on PR #1087's sweep, which is where the false survivor appeared.

SCOPE, because it decides which harnesses are exposed: only an IMPORTED module is cached. A file run as
`python3 mutant.py` is `__main__` and writes no `.pyc` at all — controlled 2026-09-20, two runs across an
equal-length same-second rewrite print the OLD then the NEW value and leave no `__pycache__` behind — and that
is why most of this repo's mutation harnesses are clear. `importlib.util.spec_from_file_location` is NOT an
escape: its loader is a `SourceFileLoader`, so it both reads and writes the cache.

Reproduction, 2026-09-20 on Python 3.12.3, both remedies in the same run (write `mod.py` as `VALUE = 'AAAAA'`,
import it by file location in a subprocess, rewrite it as `VALUE = 'BBBBB'` — same length, same second — and
import again): run 2 prints **AAAAA**. `rm -rf __pycache__` before it prints BBBBB; `PYTHONDONTWRITEBYTECODE=1`
prints BBBBB. So the remedy is to wipe the `__pycache__` beside the mutated file, or set
`PYTHONDONTWRITEBYTECODE=1` (equivalently `python3 -B`), BEFORE EACH MUTANT — never once per run.

Nothing in the repo does either: `git grep -n 'PYTHONDONTWRITEBYTECODE\|python3 -B' -- scripts deployment .github`
returns no hits, and `__pycache__/` is gitignored, so no checkout or ordinary clean ever clears it.

**THE ONE SHIPPED HARNESS THAT SHARES THE EXPOSURE**, swept 2026-09-20 over `scripts/` and `deployment/tests/`:
`scripts/gemini-spend-calibration/mutation-check.py`. It copies `probe-replay.py` to ONE fixed path in a temp
workdir, rewrites that path for each of 12 mutants, and re-runs the suite there; `test_probe_replay.py` loads the
mutated file with `spec_from_file_location`, so `workdir/__pycache__` is written and re-read across mutants. It
clears nothing. Confirmed rather than inferred, 2026-09-20: a `--keep` run (PASS, 11/12 killed, 1 declared
equivalent) leaves `__pycache__/probe-replay.cpython-312.pyc` beside the last mutant in its workdir.
**Mutants 2 and 4 ALREADY produce byte-identical 53,880-byte files.** Today mutant 3 (53,920 B) runs between
them and re-validates the cache, so the collision is inert — re-ordering the list, deleting mutant 3, or adding
any equal-length mutant next to either makes the second one a false survivor while the run still prints PASS.

NOT EXPOSED, each checked rather than assumed, so the next reader need not re-sweep. THE DISCRIMINATOR IS
EXECUTION MODE, never the language a harness mutates or how many `.py` files it writes: a file only run as
`__main__`, grepped, or listed by name is never cached, and any of these joins the exposed list the moment it
gains a per-mutant rewrite of a file it IMPORTS.
- `scripts/tests/test_verify_citations.py::test_the_tool_aborts_at_rc_3_when_its_own_control_stops_detecting`
  writes `mutant.py` into a per-test `tmp_path` and runs it as `__main__` via subprocess — no `.pyc`, and one
  mutant per directory.
- `deployment/tests/alerts/` (`mutate.py`, `run-mutant.sh`, `selftest.sh`) mutates YAML rule and test files; the
  checkers themselves are never rewritten. `selftest.sh` writes THREE `.py` files, and execution mode is why
  each is safe: `$WORK/stdlib-shadow.py` (`:475`) is invoked as `python3 <path>` (`:493`, `:500`), so it runs as
  `__main__` and nothing caches it; `$M/tests/alerts/select.py` (`:498`) is created EMPTY and only LISTED by
  name, that case being deliberately a name check rather than an import attempt; and `$WORK/noyaml/yaml.py`
  (`:44`), the one that IS imported, is written once into a fresh `mktemp -d` and never revised.
- `scripts/tests/new-epic-selftest.sh` mutates a BASH script, and separately writes ONE `.py` — a planted
  `scripts/verify-citations.py` holding the literal `WRONG-D-ENTRY` (`:717`). Safe because the predicate
  `grep`s that file and nothing imports or executes it, not because the harness is mostly bash.
- `scripts/audit-catch-spans.py`'s control writes C# fixtures into a fresh `TemporaryDirectory` and calls
  `audit()` in-process; nothing written is imported.
- `scripts/tests/test_verify_{pointers,card_companion,hosted_service_pins}.py` import the tool ONCE from the
  checkout and never rewrite it; their fixtures are `.md` and `.cs`.

Re-check (fires without running any harness):
`python3 -c "import pathlib,collections;p=pathlib.Path('scripts/gemini-spend-calibration/mutation-check.py');ns={'__name__':'x','__file__':str(p)};exec(compile(p.read_text(),str(p),'exec'),ns);b=pathlib.Path('scripts/gemini-spend-calibration/probe-replay.py').read_text();c=collections.Counter(len(b.replace(a,r).encode()) for _,_,a,r,_ in ns['MUTANTS']);print(sorted((s,n) for s,n in c.items() if n>1))"`
-> `[(53880, 2)]` today. Closed when `mutation-check.py` clears the cache or sets the env var per mutant; the
equal-length group is then harmless and this re-check may return anything.

### The pin sweep's zero-test rule binds the CONTROL run, not the mutated run it licenses [2026-09-20]
`scripts/verify-hosted-service-pins.py`'s `control_run` reads the control log's `Passed:` counts and refuses a
green run that ran ZERO tests — the harness-scoring-itself-sharp case. **`classify`, which produces the MUTATED
run's verdict, never reads a count.** It decides on the presence of `Passed!`/`Failed!` summary lines alone, so a
mutated run whose projects each printed `Passed!  - Failed: 0, Passed: 0` (a filter that matched nothing, a
project that collected nothing once the line was removed) is scored UNPINNED, and the row it writes quotes the
CONTROL's count as its evidence — `control 321 test(s) green before mutation`, true of a different run.

Direction, which is why this is debt and not a defect: it over-files UNPINNED and can never produce a false
PINNED, so it costs a re-sweep rather than a wrong clean bill of health.

Re-check: `grep -n TEST_COUNTS scripts/verify-hosted-service-pins.py` -> two hits today, the definition and ONE
use inside `control_run`. Closed when a mutated run with a zero pass count yields no verdict either.

### A `/*` inside a C# string literal opens a block comment for the pin scanner [2026-09-20]
`scan_text` tracks block comments with a running `in_block` flag and does not parse string literals, so a `/*`
inside a string OPENS a comment. **With no `*/` after it, `in_block` stays true to end of file and every later
line is blanked** — those registrations vanish from the keyed set AND from the `unkeyed` blind-spot census, so
both numbers the tool prints are short and neither says so. The docstring states only the opposite direction ("a
registration inside a string literal is scored as real"), which reads as the whole story.

Measured 2026-09-20 over the tool's own corpus (`<Service>/src/**.cs`, `obj`/`bin` pruned): **1,150 `.cs` files,
0 `/*` inside a string literal, 0 files the scanner leaves inside a block comment at EOF**, 53 registrations
enumerated. Live-site-free today, and one C# string literal away from silent.

Re-check: `git grep -nE '"[^"]*/\*' -- '*/src/*.cs' '*/src/**/*.cs'` -> no hits today; any hit is a candidate
site to read. Pair it with `python3 scripts/verify-hosted-service-pins.py`, which must still report 53.

### The pin control's ALREADY-RED arm has no test, and deleting it misdescribes the run [2026-09-20]
`control_run`'s first arm rewrites classify's PINNED into "the suite is ALREADY RED with nothing removed",
because classify's wording is mutation-phrased and would be false of an unmutated run. **Deleting that arm
outright leaves `python -m pytest scripts/tests` at 154 passed** — the DECISION survives, since the next arm
(`verdict != "UNPINNED"`) still returns not-green, and `test_a_red_control_blocks_UNPINNED_too_not_only_PINNED`
asserts on `sweep(...) == {}` rather than on the sentence. What changes is only what the agent reads: measured
2026-09-20 with the arm deleted, `control_run` returns `the unmutated run reached no green suite (PINNED): 1 of
1 test project(s) went RED with the registration removed` — about a run where nothing was removed, sending the
reader to look for a registration nobody touched.

Re-check: delete the three-line `if verdict == "PINNED":` arm from `control_run`, run
`python -m pytest scripts/tests -q` (154 passed, i.e. unpinned), restore. Closed when a test asserts the
ALREADY-RED sentence itself.

### verify-pointers.py: a deletion test against it proves the ANCHOR, never the RULE [2026-09-20]
The gate `scripts/tests/test_verify_pointers.py::test_tracked_corpus_resolves` is a NAME resolver by its
own docstring, and PR #1084 leaned on it to certify seventeen moved rules. That certification is narrower
than it reads, in two measured ways. Both reproduced at `5e3c751b` against the full tracked corpus
(`mapfile -d '' F < <(git ls-files -z '*.md'); python3 scripts/verify-pointers.py "${F[@]}"`), baseline
`202 file(s) swept, 78 anchor pointer(s) checked, 0 cannot resolve` at rc 0.

| blind spot | reproduction | measured |
|---|---|---|
| no BODY check | delete every line between a kept heading and the next heading, one destination at a time | rc **0**, "0 cannot resolve", 4 of 4: `scripts/README.md` POINTER_SWEEP, `deployment/README.md` SCORED_NOT_CONFIG, `docs/OBSERVABILITY.md` VLLM_METRICS, `LlmBenchmark/MEASUREMENT_SPACE.md` ENGINE_POLICY |
| no FLOOR | `git rm --cached docs/OBSERVABILITY.md`, re-run | rc **0**, `201 file(s) swept, 76 anchor pointer(s) checked` — a destination leaving the corpus takes its pointers' checks with it, silently |

So "N/N LOAD-BEARING" from a delete-the-destination test is a claim about the ANCHOR only: it proves the
citing prose would go RED if the construct were RENAMED or REMOVED, and proves nothing about whether the
rule the pointer promises is still written there. The count is not a detector either — it falls under
both reproductions while rc stays 0, which is the `judge a sweep by its COUNT or its rc` trap in
`CLAUDE.md` §TOOL_UPKEEP, now measured on a second tool.

**A third blind spot is already recorded and is the reason this one matters**: bare label navigation.
`deployment/ansible/group_vars/all.yml:205`, `deployment/artifacts/compose.yaml.j2:257`,
`LlmBenchmark/scripts/run_model.py:78`, `SentinelCollector/AGENT_README.md` D-29 and
`SentinelCollector/tests/SentinelCollector.UnitTests/Configuration/ExtractionModelCoordinateTests.cs:390`
all navigate to `CLAUDE.md TRACK LATEST, ROLL BACK ON FAULT` with NO section sign, from files that are not
Markdown. Neither sweep parses those, so retiring the label would have been green in both. Caught in review
on #1084, then re-derived by matching `CLAUDE.md` followed by an upper-case label across every tracked file
and checking each label still leads a line of `CLAUDE.md` — 40 labels navigated that way, one broken.

**A fourth, measured 2026-09-20 while encoding the dispatch rules**: no construct inside either
code-dispatching template can be pointed at AT ALL. `constructs()` skips fenced lines -- deliberately, for
the mermaid case -- and the entire prompt body of `implementation-fix.md` and `story-implementation.md` is
one fenced block, so their line-leading `TRAJECTORY` and `SELF_ATTACK` tokens resolve to nothing. Repro:
put a scratch `.md` in the worktree citing implementation-fix.md with a section sign and `TRAJECTORY`, run
`python3 scripts/verify-pointers.py` on it -> `no line of ... begins with TRAJECTORY`, rc 1, while
`grep -n '^TRAJECTORY$'` on the target prints a hit. Consequence: every reference INTO those two templates
is ungated prose (`CONSTRAINTS`, `Notes for the supervisor`, `TRAJECTORY step 2`, and the two added by this
PR), so renaming a template block rots them silently -- the class this gate exists to catch, unreachable
for the two artifacts the fix-round loop reads most. Fix shape: let the gate treat a file whose body is a
single outermost fence as unfenced, which keeps the mermaid case (an inner fence) intact.

Fix shape, not yet built: a body-presence assertion (a construct's section must be non-empty), a floor on
the checked count, and the label sweep above folded into the same gate so the three run together.

### new-epic.sh's graduation-shape control: the CANNOT-RUN arm is unpinned, and the recorded mutant table was irreproducible [2026-09-20]
Three mutations of `graduation_shape_case` / `run_graduation_shape_control` in `scripts/new-epic.sh`
leave `scripts/tests/new-epic-selftest.sh` fully green. Each was measured by applying it and running
`bash scripts/tests/new-epic-selftest.sh` from the repo root. EVERY COUNT BELOW CARRIES THE HEAD IT
WAS TAKEN AT, because this suite's total moves whenever a case is added and a bare `N/58` stops
being re-checkable the moment one is -- which is the defect the second half of this entry corrects,
repeated once already by the entry's own first revision.

| mutation | edit | measured |
|---|---|---|
| fixture-built proof deleted | drop `grep -qxF "$proof" "$tmp" \|\| { rm -f "$tmp"; return 2; }` (one site) | green at f17f5efc (58/58) |
| mktemp fallback returns success | `tmp="$(mktemp)" \|\| return 2` -> `\|\| return 0` **in `graduation_shape_case` only** -- that literal matches TWO sites, the other being `run_lessons_control`, whose arm IS reachable | green at f17f5efc (58/58) |
| cannot-run sentinel swallowed | the FIVE `rc=$?; [ "$rc" -eq 2 ] && return 2` lines -> `&& return 0` (four at f17f5efc; the fifth fixture added one) | green at f17f5efc (58/58) |

All three disable the same arm: the control's report that it COULD NOT RUN. Nothing in the suite
forces a cannot-run, and it cannot be forced from outside -- `TMPDIR` pointed at an unwritable path
fails `run_control`'s mktemp FIRST, so the script dies on the first control's message and a case
asserting the graduation-shape one is never reached. A reviewer looked for another external lever on
2026-09-20 and found none. Closing this needs an injection point the selftest can aim at that arm
specifically, which is test-only machinery in production code
(`.claude/skills/guard-change/SKILL.md` item 10: a shared test override binds every participant and
must be asserted in BOTH directions). Not built; the cost was judged above the exposure, which is a
control that stays silent when its environment breaks rather than one that mis-scores.
RE-CHECK: apply any row above at the current head, run the selftest, and expect it to stay GREEN at
whatever the suite's full count is then. While it does, this entry is still true.

THE RECORD THE COMMIT MESSAGE CARRIES IS WRONG, and cannot be amended post-merge, so the correction
lives here. Commit `e35439df` ("fix(new-epic): one selftest case per condition, and a control that
reaches them") states that deleting each audit condition gives 56 of 58 with a different case red.
Blinding each condition of `audit_lessons`' empty-index branch in turn and running the selftest:

| blinded condition | e35439df claims | at f17f5efc (58 cases) | at this PR's head (59 cases) |
|---|---|---|---|
| 1, heading present | 56/58 | **11/58** | **12/59** |
| 2, section empty | 56/58 | **12/58** | **13/59** |
| 3, ALREADY_ENCODED populated | 56/58 | **12/58** | **12/59** |

THE HEAD-INDEPENDENT STATEMENT, because none of those numbers survives the next added case: blinding
any one condition turns MOST OF THE SUITE red -- roughly 46 or 47 cases -- and every one of those
failures carries the CONTROL's message, not its own. 56/58, a suite nearly green with one case red,
reproduces only with `run_graduation_shape_control` UNWIRED.

Wired, it does its job: it detects the blinded condition, `die`s at rc 2 before any audit runs, and
every case that invokes the script then fails for the control's reason. That is correct fail-closed
behaviour and it makes per-case discrimination unobservable while the control is live -- so a reader
re-running the recorded table sees ~46 failures, reads it as a regression, and the tempting repair is
to soften the control's die. It is not a regression. The class is
`.claude/skills/intent-review/SKILL.md` §AIMED AT THE ACT, last paragraph.

### verify-pointers.py resolves on the FILESYSTEM, not the git index, and does not mind a construct defined twice [2026-09-20]
`scripts/verify-pointers.py` answers "does this file exist" with `os.path.isfile` under the repo
root, and "does this construct exist" by scanning lines. Four shapes therefore pass GREEN that a
reader would call rot:
  an UNTRACKED or GITIGNORED target -- present on the author's disk, absent for everyone else, and
    absent in CI, where the gate would then report the pointer as unresolvable rather than as the
    green it was locally;
  a `../` escape or a SYMLINK leaving the checkout -- resolved against whatever is there;
  a construct DEFINED TWICE in one file -- the pointer is satisfied by either, so deleting the one
    the prose means leaves it green on the other.
MEASURED 2026-09-20 at this PR's head: **zero of the 38 tracked anchor pointers is affected by any
of the four** -- every target is tracked, no pointer contains `..`, no target is a symlink, and no
resolved construct is duplicated in its file. So this is a hole in the checker, not a live defect.
RE-CHECK: re-run the sweep and compare its pointer count against `git ls-files`; the day a pointer's
target is untracked, CI and a local run will DISAGREE, which is the observable.

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

### Catch blocks that swallow a REAL fault without marking the span leave it invisible to Tempo [2026-09-20]
CLAUDE.md OBSERVABILITY makes Tempo span status the health instrument, and docs/OBSERVABILITY.md gives
`SetStatus(ActivityStatusCode.Error, ex.Message)` + `AddException(ex)` as the house catch-block pattern.
It is a convention, not an enforced one. A catch that logs and continues without either call, and
without rethrowing, leaves its span GREEN -- so the fault is invisible to the instrument we monitor by.
MEASURED 2026-09-20. 210 of 673 catch blocks are silent, but that number SPLITS and ONLY ONE HALF IS A
DEFECT. Never drive the combined figure to zero:
  **REPAIRABLE: 147 of 673 (21.8%)** -- silent over a real fault. THIS is the debt.
  cancellation-only: 63 of 673 (9.4%) -- catches ONLY `OperationCanceledException` /
    `TaskCanceledException`. Graceful shutdown, NOT a fault, and NOT a defect. A cancellation catch
    that sets `SetStatus(Ok)` is CORRECT and must NOT be "fixed": SentinelCollector's ResolutionWorker
    does precisely that on purpose. Adding `SetStatus(Error)` to these 63 would redden Tempo on every
    clean shutdown and inflate the very error ratio that code protects.
  FinnhubCollector/src alone: 21 of 42 silent (50.0%).
COVERAGE -- 147 is a FLOOR, not a total. The count covers exactly three services' production trees:
`FinnhubCollector/src`, `SecMaster/src`, `SentinelCollector/src`. The other eight services on the
CLAUDE.md SERVICES roster were NOT audited, so the repo-wide figure is necessarily higher.
COMMAND (the classifier brace-matches each catch body and masks string literals and comments first, so
a `SetStatus` mentioned in a comment does not count as instrumentation; it reports the two populations
separately, so the number and this entry's re-check cannot drift apart):
  `python3 scripts/audit-catch-spans.py FinnhubCollector/src SecMaster/src SentinelCollector/src`
CONTROL: `--selftest` drives a fixture THROUGH `audit()` end to end -- 11 catch blocks across three
files, 6 silent, of which 4 repairable and 2 cancellation-only -- and asserts the totals, both
populations, the per-site line numbers and the exit code. Two files sit under `Workers/` and
`Services/` so that a poisoned skip list changes the headline and trips the control. Verified against
two mutations: poisoning SKIP_DIRS with `/Services/,/Workers/` (headline falls to 172/49) and replacing
the total accumulator (headline becomes 210/210, 100%) BOTH leave the selftest FAILING by name.
RE-CHECK: rerun the command; closed when **REPAIRABLE reaches 0** for the audited trees -- NOT when the
silent total reaches 0, which would require damaging the 63 cancellation paths. The tool's exit code
already keys on REPAIRABLE alone. Repair is at the catch block (add the two calls, or rethrow) --
NEVER by counting log lines or raising prod to INFO.
KNOWN LIMIT: classification reads the catch's exception TYPE, not its `when` filter. Two blocks
(`EventStreamService.cs` in FinnhubCollector and SentinelCollector, `catch (RpcException ex) when
(ex.StatusCode == StatusCode.Cancelled)`) are gRPC client-disconnect paths counted as REPAIRABLE, so
147 is over-stated by 2. Filter-reading was tried and rejected as more error-prone than the gap: the
common filter here is `when (ex is not OperationCanceledException)`, a NEGATION that a naive filter
match reads backwards, turning 12+ definitively-repairable blocks into false cancellation hits.
WORKED EXAMPLES, one of each, in the SAME file:
  DEFECT: the stamps-persistence catch in `FinnhubCollector/src/Workers/QuoteCollectionWorker.cs` logs a
    Warning and bumps `FinnhubMeter.CollectionErrors`; the span stays green.
  CORRECT: the quote-collection catch higher in that file sets status + AddException and THEN logs at
    Warning, with a comment explaining that only the log level drops. Warning-level logging and a red
    span are not in tension; that block is the pattern to copy.
NOT FIXED HERE: found while rewriting the `deploy` skill's health story (a docs PR). The skill now names
this as blind spot 4 and tells readers a green Tempo window means "nothing painted a span red", not
"nothing failed" -- which removes the wrong action but leaves the instrument incomplete.

### The smoke test cannot see the OTEL stack, so a green run is consistent with loki/tempo/prometheus being down [2026-09-20]
`deployment/ansible/playbooks/smoke-test.yml` lists containers with `nerdctl compose ps -a` from
`deployment_base` (`/opt/ai-inference`). Its `infrastructure_services` var names seven services, but
only `timescaledb` and `alertmanager` are in that compose project. `prometheus`, `grafana`, `loki`,
`tempo` and `otel-collector` are a SEPARATE project — `/opt/otel/compose.otel.yaml`, run by
`otel.service` — and cannot appear in that task's output however the selector is written. So the five
that step 4's health method actually depends on are the five the smoke test cannot check.
MEASURED 2026-09-20:
  `nerdctl compose -f /opt/ai-inference/compose.yaml config --services` -> 31 services, of the seven
    named only `timescaledb` and `alertmanager` appear.
  `nerdctl compose -f /opt/otel/compose.otel.yaml config --services` -> 8 services, containing the
    other five plus node-exporter, gpu-exporter, ups-exporter (which nothing checks at all).
ONE OF THE FIVE IS COVERED BY ACCIDENT, and the accident is worth naming because it dies silently:
`grafana` down WOULD be caught, since the "Check internal service health" task shells out through
`nerdctl exec grafana curl` and every one of its ten checks would return non-zero. That is coupling, not
coverage — it disappears the moment someone moves those checks to another container with curl, and it
says nothing about loki, tempo, prometheus or otel-collector.
RE-CHECK: `nerdctl compose -f /opt/otel/compose.otel.yaml ps -a --format json` returns containers that
appear in no smoke-test task. The defect is closed when a failing container in that project turns the
playbook's exit status non-zero; prove it the way the ATLAS-project gate was proved, by running the
playbook with an extra-vars file that empties the expected-exited exclusion.
FIX SHAPE, not done here: a second `command` task against that compose file, merged into `containers`
before `unhealthy_containers` is computed. Left out of the PR that found it because that PR was closing a
cannot-fail gate, and adding an unexercised new task to a gate is how the first one got in.

### The smoke test asserts nothing non-running, never that everything expected is present [2026-09-20]
`deployment/ansible/playbooks/smoke-test.yml`'s container check computes `unhealthy_containers`
by filtering the `compose ps -a` output for `State != running`. That asks "is anything listed
broken?" and never "is everything that should exist here?", so a container that was never
CREATED -- a service dropped from the rendered compose.yaml, a `compose up` that partially
failed -- is invisible rather than failing.
MEASURED 2026-09-20: an extra-vars file setting `containers: []` (ansible `-e` outranks the
`set_fact` that normally populates it, so this exercises the real downstream tasks) runs the
playbook to `Containers: PASS`, `Overall: PASS`, rc 0 -- with no containers at all.
The `-a` fix landed 2026-09-20 closed the adjacent hole (a container that EXISTS and is not
running); this one is the complement and is older than that fix.
EXPOSURE, measured 2026-09-20: **13 of the 31 compose services have NO check in this playbook
other than that container listing**, so for those 13 a never-created container is invisible
full stop. The other 18 would still be caught by an HTTP or DB probe (10 `internal_services`,
6 `mcp_health_endpoints`, plus vllm-server's own `/health` and timescaledb's `SELECT 1`).
The 13: alertmanager, dsl-parser-mcp, finbert-sidecar, llama-cpu-embed, llama-cpu-rag,
llama-server, markitdown-mcp, migrate-macro-substrate, reports-daily, reports-monthly,
reports-weekly, spacy-ner, trafilatura.
Re-derive the set rather than trusting this list: diff `nerdctl compose config --services`
against the playbook's `internal_services` + `mcp_health_endpoints` + {vllm-server, timescaledb}.
RE-CHECK: `ansible-playbook playbooks/smoke-test.yml -e @<file setting containers to []> --tags
health` returns 0. Closed when it returns non-zero, or when a task compares the observed service
set against `nerdctl compose config --services` (31 as of this date) and fails on a shortfall.

### `exited` + `ExitCode 0` cannot tell a container that ran THIS deploy from a four-day-old corpse [2026-09-20]
`deployment/ansible/playbooks/smoke-test.yml` exempts a one-shot container when
`State == "exited"` and `ExitCode == 0`. Both are true of a container that completed once, days
ago, and has not run since -- the exemption has no RECENCY term, and cannot get one from its
current data source.
MEASURED 2026-09-20: `nerdctl compose ps -a --format json` carries NO timestamp. Its key set is
exactly Command, ExitCode, Health, ID, Image, Name, Project, Publishers, Service, State.
LIVE STATE the same day: `migrate-macro-substrate` finished at 2026-09-16T11:00:33Z (ExitCode 0)
while `llama-cpu-embed`, `llama-cpu-rag` and `secmaster` were created 2026-09-20T14:30Z -- four
days apart, and the bare run reports PASS. A scoped deploy never touches the migrator at all, so
this is the ORDINARY case, not an edge one. Its TOTAL ABSENCE from the listing also passes, by
the complement hole recorded in the empty-container-list entry above.
CONSEQUENCE, stated plainly because the code comment used to overclaim it: the `State` +
`ExitCode` tightening stops a migrator that FAILED or NEVER STARTED in this listing from being
waved through. It does NOT establish that a migration ran, and it composes with the `SELECT 1`
database check (which cannot see an unmigrated schema, entry above) so that "PASS over an
unmigrated database" remains OPEN end to end. Neither entry closes it alone.
DATA-SOURCE SHIFT THAT WOULD CLOSE IT: `nerdctl container inspect <name>` carries `Created`,
`State.StartedAt` and `State.FinishedAt` (verified 2026-09-20 on the migrator: Created
2026-09-16T11:00:31Z, FinishedAt 2026-09-16T11:00:33Z, StartedAt null). A recency term needs
that call per exempt container, plus a decision about what "this deploy" means -- there is no
deploy timestamp in the playbook today.
RESIDUE in the same exemption, measured and deliberately left:
  - `selectattr('ExitCode', 'equalto', 0)` also exempts BOOLEAN `false` and FLOAT `0.0`, because
    Jinja compares by value. Neither occurs in real nerdctl output; both would pass if injected.
  - the State-ABSENT case fails at `Identify unhealthy containers` with "'dict object' has no
    attribute 'State'", NOT at the `selectattr('State', 'defined')` probe in `Identify exempt
    containers` that the comment credits. It fails CLOSED either way (rc 2), but by a different
    task than the one documented.
RE-CHECK: `sudo nerdctl compose ps -a --format json | python3 -c "import sys,json;
print(sorted(json.loads(sys.stdin.read())[0].keys()))"` -- closed when a timestamp appears there
and the exemption reads it, or when the playbook inspects each exempt container instead.

### A smoke-test run that executes ZERO tasks exits 0; the class is unbounded in spelling and not closeable inside the playbook [2026-09-20]
`deployment/ansible/playbooks/smoke-test.yml` couples each domain's verdict to its own checks,
so any selection that runs at least one task is judged. A selection that runs NO task is not,
and exits 0 -- the rc then means "nothing was looked at", not "nothing is wrong".
THE DEFECT IS THE CLASS, NOT ANY SPELLING: **any combination of `--tags` and `--skip-tags`
whose resolved task set is EMPTY**, recognisable by an empty PLAY RECAP and rc 0. Ansible's
tag algebra is closed under intersection and complement, so the spellings that produce an
empty set are unbounded and cannot be listed. Examples measured 2026-09-20 with a real
container failure injected, ILLUSTRATIVE AND NOT AN ENUMERATION -- do not treat this list as
the defect's extent, and do not extend it: `--skip-tags tagged`, `--skip-tags always,health`,
`--tags always --skip-tags always`, `--tags logs --skip-tags always`. An earlier revision of
this entry claimed the exposed spellings "all pair `always` with a second skip, or skip the
`tagged` meta-tag"; the last example falsifies that -- it is a `--tags` form with neither
property -- which is exactly why the CLASS, not its members, is what this entry records.
NOT A REGRESSION AND NOT THE DOCUMENTED IDIOM. `--skip-tags always` ALONE -- the form CLAUDE.md
§DEPLOYMENT prescribes for a non-service tag -- resolves to a NON-empty set of **25 tasks** (27 in the
play, less the two tagged `always`), of which that injected-failure run printed **21 TASK banners**:
`ok=16` + 4 skipped on a false `when` + the 1 that failed. The play aborts at `Verdict - containers`, so
the four later `Verdict -` tasks are resolved but never reached and print no banner. Exit code 2. THE
NUMBER IS THE BANNER COUNT, NOT THE RESOLVED SET -- those are different measurements and an earlier
revision of this entry gave 18, which is neither and reproduces from no denominator. Re-derive both
STATICALLY, with no playbook run: `python3 -c "import yaml; ts=yaml.safe_load(open('deployment/ansible/playbooks/smoke-test.yml'))[0]['tasks']; r=[t for t in ts if 'always' not in t['tags']]; n=[t['name'] for t in r]; print(len(ts), len(r), n.index('Verdict - containers')+1)"`
-> `27 25 21`. The `ok=16` already in this sentence is the arithmetic check: 16 + 4 + 1 = 21. The mechanism
predates this playbook's current shape: Ansible has no task that cannot be deselected.
✗ DO NOT close this by adding a task or a tag to the playbook # that is the enumeration the
  per-domain selection model deleted after four rounds, each of which added one more member.
✗ DO NOT close it with a blacklist of spellings either # same defect one level up: a wrapper
  refusing four literals leaves the rest of an unbounded family exiting 0, while reading as
  closed. The property to assert is EMPTINESS OF THE RESOLVED TASK SET, which is observable
  only AFTER resolution -- never from the argv.
FIX SHAPE, if it is judged worth it: assert on the RECAP, not on the invocation. A wrapper that
runs the playbook and fails when the PLAY RECAP reports zero tasks (equivalently, parse
`--list-tasks` under the same selection and refuse an empty result) catches every member of the
class including ones nobody has spelled yet. It lives OUTSIDE the playbook, which is the only
place the property is expressible.
RE-CHECK, and note it tests the CLASS via one arbitrary member: `ansible-playbook
playbooks/smoke-test.yml --tags logs --skip-tags always; echo $?` -> 0 with an empty recap.
Closed when a run whose resolved task set is empty cannot report success.

### Three smoke-test rows still say PASS on reachability alone: database, vllm-server, container Health [2026-09-20]
Asked deliberately after three review rounds on `deployment/ansible/playbooks/smoke-test.yml` each
found the same class one level down (an exemption with no evidence of a clean run, a gate
skippable by tag selection, a doc claim that did not hold). These are NAMED, not fixed.
1. `Database: PASS` is `psql -c "SELECT 1"`. That proves the server accepts a connection to
   `atlas_data`; it reads no table and no `__EFMigrationsHistory` row, so an UNMIGRATED or
   half-migrated schema reports PASS. This is the direct blast radius of the one-shot migrator
   whose exemption was tightened twice in this same PR -- the container check now catches the
   migrator failing, and the database check still cannot see the consequence.
   Re-check: the task runs `SELECT 1` and nothing else; closed when it asserts something only a
   migrated schema can answer.
2. `vllm-server: PASS` is a `GET /health` returning 200. It is reachability, not inference: no
   completion is requested. CLAUDE.md §VLLM_UPGRADE already records that the fp8_e5m2 fault
   class needs concurrency >= 2 and that a /health probe plus one sequential 1-token completion
   cannot see it -- this playbook does not even do the completion.
   Re-check: the task is a bare `uri` GET; closed when it exercises a real completion.
3. The container check reads `State` and `ExitCode` and never `Health`, so a container that is
   `running` but wedged reads as PASS. MEASURED 2026-09-20: `Health` is the empty string on
   **31 of 31** containers, because nerdctl 1.7.7 discards healthchecks entirely
   (project_nerdctl_ignores_depends_on). So there is no signal to read TODAY -- but the field is
   in the output, it looks authoritative, and a future runtime that populates it would be
   silently ignored.
   Re-check: `nerdctl compose ps -a --format json | ...` count distinct `Health` values; if any
   is non-empty the playbook should be reading it.
None of the three is a regression; all three predate this PR. They are recorded because "PASS
on reachability" is exactly the shape of defect this playbook has now yielded three times.

### The <=5 ms share is the only detector for an unapplied `hnsw.ef_search`, and nothing watches it [2026-09-20]
At `SemanticSearchOptions.HnswEfSearch=200` the share of vector searches at or under 5 ms on
`secmaster_vector_search_duration_milliseconds` separates ef_search=200 from both ef_search=400 and an
unapplied setting -- BUT ONLY AT A 6h WINDOW, and the statistic must be named with its window or it is
useless. Measured 2026-09-20 across the ef=200 era:
  6h non-overlapping windows, n=12: **0.214 to 0.374**, median 0.261.
  30-minute windows, n=145: 0.108 to 0.607, p5 0.148, median 0.266, p95 0.520, mean 0.280. A 30-minute
    window is NOT a discriminator: its floor of 0.108 brushes the top of the ef=400 era (0.025-0.105 at
    1h windows), so the two populations overlap. An earlier revision of this entry quoted "17-31%" with no
    window named; 34% of 30-minute windows fall outside that range, and anyone sizing an alert hold from it
    would page constantly.
THE UNAPPLIED ERA, at the SAME 6h window, so the two populations are comparable. 25 non-overlapping 6h
windows over 2026-09-10T23:00Z to 2026-09-16T23:00Z, before #1030's value first shipped: **0.837 to 0.925**,
p5 0.864, median 0.892, p95 0.912. An earlier revision of this entry said "0.845-0.876, >=0.85 means
unapplied" -- 80% of those windows sit ABOVE 0.876 and one sits BELOW 0.85, so that band was both too narrow
and mis-thresholded. This is the same defect corrected for the applied-era band one paragraph up, and it is
the reason the threshold below is not set at the edge of a measured range.
The two populations do not overlap and nothing has ever been observed between them: applied tops out at
0.374, unapplied bottoms out at 0.837. **Any threshold in that empty 0.46-wide gap separates them; 0.60 is
used because it sits clear of both edges.** A reading above it means ef_search is NOT being applied and the
session is running pgvector's stock 40 -- what a container serving an image that predates the setting looks
like, and that signature hid #1030 for eleven days. Nothing alerts on it: the only consumer of this histogram
is one p50/p95 panel in
`deployment/artifacts/monitoring/dashboards/Collectors & Services/secmaster-resolve-latency.json`, and a
quantile cannot see the share.
Re-check (6h window, expect 0.21-0.37; >=0.60 unapplied; <=0.11 back at 400):
`sum(increase(secmaster_vector_search_duration_milliseconds_bucket{le="5.0"}[6h])) / sum(increase(secmaster_vector_search_duration_milliseconds_bucket{le="+Inf"}[6h]))`
Separately, that the BUCKET LADDER itself deployed (as opposed to the service running an older image) is one
query, and it is a metric check because prod logs default to Warning and a healthy container says nothing:
`count(secmaster_vector_search_duration_milliseconds_bucket{le="7.0"}) > 0` is TRUE once the explicit
boundaries are live and the result is EMPTY while the SDK defaults are in force -- `le="7.0"` exists in no
default ladder. Compare against 0, never against 1. This histogram is tagged `success` and carries
`success="true"` and NOTHING ELSE, by design and not by luck: `EmbeddingService.VectorSearchAsync` records
it at exactly one site, on the success path, and its catch block deliberately skips it because a SQL-only
duration is undefined when the failure lands before or during the embedding call. The metric that gains
`success="false"` is the OTHER one, `secmaster_vector_search_total_duration_milliseconds`, recorded in that
same catch -- do not search this one for failures, they structurally cannot appear here. The error signal is
the counter `secmaster_vector_search_errors_total{error_type=...}` (`SecMasterMeter.VectorSearchErrors`),
already plotted on the resolve-latency dashboard.
`> 0` is correct either way and survives a future second label value.
The `le="5.0"` series is what makes this re-checkable at all, which is why `5` is a pinned boundary in
`SecMaster/src/Telemetry/VectorSearchLatencyBuckets.cs` and in its test rather than a survivor of the SDK default.
To close: wire a rule on that expression, with the `for:` taken from the distribution and not from the average.
One more thing this histogram cannot separate, same date, no setting changed: `instruments` has NEVER been
autovacuumed -- `autovacuum_count` 0, `last_autovacuum` null, 3,349 dead against 28,998 live (10.4%), on
241,290 updates of which 238,922 (99.0%) are HOT. The default trigger is `50 + 0.2 * n_live_tup` = 5,850 dead,
so it has not fired and on this table's shape will not. The vector-search statement JOINs `instruments`
(the D-13 join), so that dead heap is paid on the hydration leg of every vector search and lands in the same
histogram as ef_search with no way to tell the two apart. Counters are live and reset on `pg_stat_reset`.
Re-check (SELECT-only):
`sudo nerdctl exec timescaledb psql -U ai_inference -d atlas_secmaster -c "SELECT n_live_tup, n_dead_tup, n_tup_upd, n_tup_hot_upd, last_autovacuum, autovacuum_count FROM pg_stat_user_tables WHERE relname='instruments';"`

### SecMaster D-16 self-seed and discovery refusals are counted but unalerted; their rate is unmeasured [2026-09-17]
SecMaster D-16 refuses a class-family conflict at four write sites. The two register sites also count under
`secmaster_registration_rejected_total{reason="class_family_conflict"}` and so ride the wired
`SecMasterRegistrationRejectionsSustained`, asserted firing in `deployment/tests/alerts/secmaster_test.yml`. The
self-seed and discovery sites emit only `secmaster_identity_conflict_total{site="self_seed"|"discovery"}` plus an
Information log, which prod's Warning floor drops, so NOTHING watches them. A regression that makes either site
refuse most confirmations drops those resolutions silently.
Why no rule yet: the conflict share of those streams has never been measured, and a hold chosen without the
distribution paged ~10 times in 7 healthy days on a bursty signal (the PR 1042 round 3 note in
`deployment/tests/alerts/expected-counts.yml`). The pre-image, measured 2026-09-17T10:57:55Z over 7d:
`secmaster_entity_resolution_self_seed_total` idempotent_skip 7,068, inserted 190, quarantined_skip 48; idempotent_skip
per 30m at 5m steps has median 17.1 and max 85.7. class_conflict_skip is carved out of what used to count as
idempotent_skip, so that max is the ceiling for the self-seed site.
The query whose distribution decides the rule:
`sum by (site) (increase(secmaster_identity_conflict_total{site=~"self_seed|discovery"}[30m]))`
Take the distribution only after SecMaster D-18 deploys, which removes the drops on 74 FRED-labelled rows; the drops
it leaves are the KNOWN DEFECTS entry on the 16 FRED ids D-18 leaves Equity. A refused candidate no longer
reaches D-3's review-queue enqueue or its Error span, so it adds nothing to a post-deploy Tempo error rate.
To close: at deploy+24h record that query's per-site value here; after 7d take its distribution at 1-5m
(`max_over_time` and `quantile_over_time` over `[7d:1m]`), choose threshold and `for:` from it, add the rule with a
promtool firing case on burst-shaped input, and delete this entry in that PR.
Re-check: `grep -cE '^\s*expr:.*secmaster_identity_conflict_total' deployment/artifacts/monitoring/alerts/secmaster.yml`
returns 0 while this is open. Match the `expr:` line, not the name: a comment naming the metric would read CLOSED.

### Alert rules and their metrics ship on different schedules; rule-first pages a healthy system [2026-09-16]
**Alert rules and the metrics they read ship on different schedules, and rule-first pages a healthy system.**
`FinnhubCollectorQuoteCollectionStalled` carries an `absent()` leg and ships via `--tags alerting`, while the gauge
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
(`Channel.CreateUnbounded`). Two live writers — `DataCollectionService.cs:224` and `BackfillService.cs:141`, both
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
GUARD citation in this repo's cards. CONSEQUENCE (CLAUDE.md TOOL_UPKEEP, LESSONS.md ALREADY_ENCODED, was L8): any edit shifting
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

### A prose-form reference is not a checked citation at all: cross-file line references in tracked `.md` sit outside every sweep -- 15 at `9f0f0ce8`, 16 at `76cd8d1c` [2026-09-21]
`scripts/verify-citations.py` recognises exactly three spellings, and all three require the COLON: `CITATION`
(`path.ext:N`, extension on the `_EXTS` allowlist), `COMMA` (the `,223-225` tail immediately following one), and
`BARE` (`:N`, opt-in behind `--bare`). A reference written as prose -- a sentence naming a file and a line number
in words -- matches none of them, so it is never parsed, never resolved, never reported, and no run says it was
skipped. DISTINCT from the two entries above, which are about a CORPUS or an EXTENSION the resolver would handle
if it were handed them; here there is nothing for the resolver to be handed. The illustrative form, fenced so it
does not perturb the count below:

```
`ThresholdEngine/src/DependencyInjection.cs` line 186 confirms no active subscription.
```

Probed 2026-09-21 at `9f0f0ce8`, running the tool over a scratch file carrying three prose spellings of one
reference plus a single colon-form control on the same target: `1 file(s) swept, 1 citation(s) checked`. The
checked count is the proof -- three of the four references were invisible, and the report named neither them nor
their absence.

MEASURED, and the number is a FLOOR rather than a census: **15 occurrences in 5 of the 209 tracked `.md` files**
at `9f0f0ce8`. UNIT: occurrences, not files -- one line of this file carries two. POPULATION: the set
`git ls-files -z '*.md'` names, which is exactly the corpus the documented sweep hands the tool. METHOD: outside
fenced blocks, a `line N` / `lines N-M` token on a line that ALSO names a file carrying an `_EXTS` extension,
with every span the shipped `CITATION` and `COMMA` regexes already claim masked out first -- both imported from
`verify-citations.py` rather than re-spelled, so the "not a checked citation" half is decided by the shipped rule
and not by a second copy of it. Breakdown of those 15, still at `9f0f0ce8`: 6 are `~`-hedged approximations in
one plan document, 3 are
DELIBERATE (the `make_cache_key` docstring entry under §KNOWN DEFECTS states in its own text that it avoids the
colon form so a sweep cannot read its stale numbers as live claims), 6 are unhedged assertions into living files.
THE 3 IS ITSELF A FLOOR, of this entry's own neighbouring-line class: a FOURTH prose reference sits in that same
`make_cache_key` entry -- the one recording where the `$`-leading gate now sits -- and the METHOD discards it only
because its filename fell on the previous physical line. It is in the 13-line residual below. Said here rather
than left for the reader to find, because the size of the deliberate class is what the no-fix reason turns on.

**THE METHOD'S `line N` TOKEN MATCH IS CASE-INSENSITIVE, AND EVERY NUMBER ABOVE AND BELOW IS THE CASE-INSENSITIVE
ONE** [added 2026-09-21, residue of the round that wrote this entry]. The METHOD as stated does not say so, and it
is not cosmetic: a re-adjudicator who implements it literally gets a case-SENSITIVE matcher, lands one occurrence
short, and reports a count MOVE that is an artefact of their own regex rather than a change in the corpus.
Re-derived at `76cd8d1c` over the 210 tracked `.md` files then in the corpus, running both flags through ONE
implementation and diffing the SETS rather than the counts: case-SENSITIVE gives **6,817** same-line-less
candidates and a **17-occurrence / 12-distinct-line** residual; case-INSENSITIVE gives **6,818** and **18 / 13**.
The `large-card-*` share is 6,800 either way, and the MATCH set is identical either way -- the flag moves exactly
ONE occurrence: the token `Line 70`, a capital `L` at the start of a sentence, in this file's own
`a rule stated in a skill` hook entry. **FIND IT BY TEXT, NOT BY ADDRESS** --
`grep -n 'Line 70 entire' docs/BACKLOG.md` -- because the address was `:6415` at `76cd8d1c` and is `:6656` here,
having moved TWICE inside this one commit as text was inserted above it. That is not an incidental caveat: the one
occurrence the measurement turns on rotted its own line number inside the very commit that measured it, which is
exactly why this entry says to compare the SET and never the count or the address.

**AND THE MATCH COUNT AT `76cd8d1c` IS 16, NOT THE 15 RECORDED ABOVE AT `9f0f0ce8` -- the corpus grew, the METHOD
did not change** (210 tracked `.md` files against 209). The 16th is
`docs/proposals/delete-wrong-mappings.md:27`, a file that did not exist at `9f0f0ce8` at all -- `git cat-file -e`
on that path at that revision reports it absent -- so this is a NEW unhedged prose reference into a living file,
which is precisely what the RE-CHECK below calls the finding, arriving between one measurement and the next. It is
also the sharpest illustration of blast radius shape (2) available: it reads "`SecMaster/AGENT_README.md`'s
`✗ scope-search-by-source-mapping` GOTCHA (line 94 at `9f0f0ce8`)", and the same PR that writes this paragraph
EDITS that line. It survives only because that edit is in place and does not move line 94; nothing checked that,
and nothing would have reported it had the line moved. One point in its favour, and the only mitigation available
to a prose reference: it carries a SHA, which none of the other 15 do.

WHAT THE COUNT STRUCTURALLY CANNOT FIND, stated because an honest floor is worth more than a confident wrong
number:
  a reference carrying NO NUMBER AT ALL -- naming the WRONG FILE for a figure is this class, and nothing keyed on
    a line number can see any of it, so that class is not bounded by the 15 at `9f0f0ce8`, by the 16 at
    `76cd8d1c`, nor by anything measured here;
  a reference whose filename sits on a NEIGHBOURING line -- the same-line requirement is what does all of the
    narrowing, discarding 6,818 candidates, 6,800 of them the two `large-card-*` fixtures' `padding line NNNN`
    filler. Re-derived 2026-09-21 at `7c94166b` from an independently written second implementation of the METHOD
    (15 / 6,818 / 6,800 all unchanged): the residual is 18 occurrences on 13 DISTINCT LINES, and reading all 13 by
    hand, 9 are genuine cross-file references, so ~24 is the better floor. CRITERION, stated so the next reader
    RE-ADJUDICATES rather than re-guesses: a number aimed at a file OTHER than the citing one and asserting
    something about that file's CURRENT content. The 4 excluded are two prose descriptions of a synthetic
    nine-line fixture that is not a tracked file, one dated record of what a past review found, and one set of
    LOG line numbers;
  anything inside a FENCED block -- the METHOD skips fences, which is what keeps the illustrative form above from
    perturbing its own count, so this is a blind spot the entry RELIES on rather than merely has. `verify-citations.py`
    itself has no fence handling at all, so the two disagree: a colon-form citation inside a fence IS swept;
  a same-line filename carrying NO extension, or one off the `_EXTS` allowlist -- the extensionless
    `scripts/claude-*` wrappers are invisible to the filename half of the METHOD for exactly the reason the
    sibling `_EXTS` entry above calls that exposure latent for the sweep itself. **THAT COUNT WAS "three" AND IS
    TWO** (corrected 2026-09-21 at `76cd8d1c`): `git ls-files 'scripts/claude-*'` names FIVE tracked paths, of
    which exactly two are extensionless FILES -- `scripts/claude-mark-verified` and `scripts/claude-pr-verdict`.
    The third `scripts/claude-*` path component is `scripts/claude-watchdog`, a DIRECTORY, and all three of its
    tracked members carry extensions (`README.md`, `notify.sh`, `scan.py`), every one of them on the `_EXTS`
    allowlist. A glob over a directory prefix counts directories as members, which is how the 3 arose. The blind
    CLASS is unaffected by the correction -- it is the same two files plus every extensionless path anywhere, and
    the class was never bounded by this count;
  positional prose ("the fourth row of the DECISIONS block"), spelled-out numbers, an `L186` spelling, and any
    reference split across a line break.
The method can also produce FALSE POSITIVES -- a `line N` naming a diff, a log excerpt or the citing file's own
body, on a line that happens also to name a file. Zero of them in the `9f0f0ce8` run; all 15 of that run's
occurrences were read by hand. The 16th, which only exists at `76cd8d1c`, was read by hand too.

BLAST RADIUS, two shapes, and the second has no author. (1) A prose reference is written WRONG and a fully green
sweep says nothing, because it was never a claim the tool could test -- the reviewer is the only detector, and a
green sweep actively reassures them. (2) A prose line number into a file OTHER than the citing one rots with
nobody editing either file, the moment a sibling PR inserts a row above the target: the same rot the
`make_cache_key` entry under §KNOWN DEFECTS records happening to a `.py` docstring when PR 1061 moved two
SecMaster constructs. `CLAUDE.md` §TOOL_UPKEEP already carries the asymmetry that makes this bite -- anchor
pointers are gated in CI and `file:line` ones are not the same check -- and a prose line number is one rung BELOW
the ungated `file:line`, being not merely unchecked but unparsed. Not restated here.

NO FIX IS PROPOSED -- and NOT because one is impossible. A PARTIAL fix is WORSE than the gap, which is a
different and stronger reason than the one this entry first gave. MEASURED 2026-09-21 at `7c94166b`: two scratch
copies of this file, identical but for the 3 DELIBERATE references rewritten into colon form on the two physical
lines that carry them, each swept alone with `--repo-root` at the worktree. Verbatim 208 citations checked,
prosified 211; FOUR cannot land on BOTH, rc 1 on BOTH, and the unresolved SET is the same four entries either way
(three `sentinel-v6.2-cove.json` cites and the ambiguous-basename `SeriesManagementService.cs` one) -- ZERO new
findings. So teaching the tool to read prose turns NOTHING red here. The claim it replaces -- that the deliberate
class would turn three correct lines red -- does not reproduce.
IT DOES SOMETHING WORSE. All three RESOLVE, onto real non-blank lines, and are printed as healthy landings: a
`//` comment, a `///` doc comment, and the `PersistConfirmedInstrumentAsync` signature itself. Two of those three
numbers are ones the `make_cache_key` entry above records as STALE, and a prose-reading sweep would report them
as LANDING and thereby bless them. RED is visible; a green blessing is not, and `verify-citations.py` is
content-blind by design (the sibling `verify-citations.py reports GREEN on a citation that has drifted onto the
WRONG line`), so nothing downstream catches it either. Prose parsing ALONE is a regression; prose parsing PLUS
content-awareness is that other, still-unfixed entry. And it would leave untouched the class with no bound at
all -- a reference carrying NO NUMBER, which nothing keyed on a line number reaches. Closing this is a judgement
about the corpus, not a regex.
WHAT THAT MEASUREMENT CANNOT CONTAIN: one file, and the 3 deliberate references only. It says nothing about what
prose parsing would do to the other occurrences -- 12 of the 15 at `9f0f0ce8`, 13 of the 16 at `76cd8d1c` -- in
particular the 6 `~`-hedged plan-document ones, which a parser reading a hedge as an exact claim might well turn
red. Not measured, and not an argument made here.

RE-CHECK: re-run the METHOD above at head -- **case-INSENSITIVELY, or your residual is 17-on-12 and the move you
report is your own regex** -- and compare the occurrence SET against **the 16 recorded at `76cd8d1c`**, which is
the later of this entry's two baselines and therefore the one to diff against; the 15 above are the `9f0f0ce8`
set and differ from it by exactly `docs/proposals/delete-wrong-mappings.md:27`. Compare SETS, never the count,
and never the line addresses, which move whenever any file above them is edited -- diffing a bare count against
a baseline whose sha you did not carry is how a corpus that merely GREW reads as a finding. The 3 deliberate ones
must still be there; a NEW unhedged reference into a living file is the finding, and
`docs/proposals/delete-wrong-mappings.md:27` is one that already arrived this way. EXPECT THIS ENTRY'S OWN PROSE
IN YOUR RESULT AND ADJUDICATE IT, DO NOT COUNT IT: the paragraphs above QUOTE the references they are about, so a
run at any revision containing them surfaces `line N` tokens inside `docs/BACKLOG.md` itself. The CRITERION stated
under WHAT THE COUNT STRUCTURALLY CANNOT FIND already excludes them -- each carries a sha and so asserts nothing
about the target's CURRENT content -- and a reader who skips that step reports the entry as its own finding. For the
no-fix reason: copy this file twice, rewrite ONLY those 3 into colon form in one copy, sweep each copy alone, and
compare the unresolved SETS -- never the counts, never the rc, both of which are identical on the two sides.

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

### The attach-gold drift audit watches `accept` ids only: 61 baseline-only ids are unwatched, and the 2 already deactivated were counted as an integer while the named list stayed empty [2026-09-21]
THE SCAN IS NOT MISSING -- ITS POPULATION IS. `LlmBenchmark/scripts/build_attach_gold.py`'s `stage_drift` does
perform an id-existence re-check: `:1434-1440` builds an `accepted` map and hands its keys to `catalog_rows`
(`:526-532`), which SELECTs `id, symbol, is_active, retired_at IS NOT NULL` from `atlas_secmaster.instruments`, and
`:1473-1479` reports five fields under `accepted_ids` -- `checked`, `missing` (which is exactly the hard-DELETE
bucket), `inactive`, `retired` and `symbol_changed`. A DELETE of an ACCEPTED id would be named. What `:1439` reads
is `o.get("accept")` and `o.get("accept_symbols")`; `o["baseline"]` is never read in `stage_drift` -- nor by the
second liveness check, `stage_staleness`, which scopes to the same key at `:1796` and `:1813`. Two independent
checks, one shared population.

MEASURED 2026-09-21T19:54Z, re-derived by a freshly written reader rather than taken from the report this was
raised from. UNIT: distinct instrument ids. POPULATION: every `owners` and `empty_owner_rows` row of both golds,
`LlmBenchmark/attach-gold/attach_gold_g1_v1.json` and `LlmBenchmark/attach-gold/attach_gold_g2_v1.json`.
  distinct `accept` ids, g1+g2 union: **250** -- which is exactly the `accepted_ids.checked` recorded in the
    committed `LlmBenchmark/attach-gold/catalog_drift_v1.json`, so the population claim is confirmed against the
    ARTIFACT and not only against the code;
  distinct non-null `baseline` ids: **121**, all of them in g2 -- g1's row schema has no `baseline` key at all;
  **baseline ids appearing in no `accept` list anywhere: 61.** All 61 exist in `instruments` today (SELECT-only);
    **2 are `is_active = false`** -- `DX` and `KC`, both `discovery_source='GeminiFallback'`, both `updated_at`
    2026-09-17T14:08:42Z -- and **0** are retired;
  a further **561** baseline entries carry `instrument_id: null`, production having made no pick, so they are
    structurally unwatchable and there is nothing there to watch.

THE FIGURE THIS ENTRY CORRECTS ON ITS WAY IN: the finding was dispatched as "DX and KC". Those two are the ids that
have drifted SO FAR; the unwatched POPULATION is 61. Filed as 2 it would understate the exposure roughly 30x, and
it would hand the next reader a re-check that goes green the day someone re-accepts DX. The re-checkable number is
**61 unwatched, 2 drifted**. AND THE 61 IS A FLOOR BOUNDED BY WHAT THE FROZEN GOLDS REFERENCE: it counts only
baseline ids named in `attach_gold_g1_v1.json` and `attach_gold_g2_v1.json` as committed at `76cd8d1c`, so it
structurally cannot contain a pick recorded in a gold version that does not exist yet, the 561 baseline entries
whose `instrument_id` is null, or any catalog row no gold references at all -- it grows with the gold corpus, not
with the catalog.

THE AGGREGATE AND THE NAMED LIST ALREADY DISAGREE IN THE COMMITTED ARTIFACT, and nothing reconciles them.
`catalog_drift_v1.json` records `counts.deactivated_in_window = 2` beside `accepted_ids.inactive = []`, over
`window_since` 2026-09-17T11:00:00Z to `measured_at` 2026-09-20T12:02:05Z -- a window that CONTAINS the 14:08:42Z
deactivation. Those four `counts` aggregates (`:1441-1448`) are catalog-WIDE `COUNT(*)`s, not scoped to the gold,
so DX and KC were counted as an anonymous 2 while the field whose job is to NAME movement stayed empty. A reader
comparing the two fields can see the disagreement; a reader trusting either one alone cannot.

CONSEQUENCE: the audit exists to describe catalog movement under a FROZEN gold. If one of the 61 is ever DELETED
rather than deactivated, `accepted_ids.missing` stays `[]`, no field of the artifact names it, and the baseline row
goes on citing an id that no longer resolves -- so the gold's record of what production PICKED becomes
unverifiable, silently. Deactivation is the milder shape of the same blindness and has already happened twice.

WHAT THE METHOD STRUCTURALLY CANNOT SEE, beyond the population. `source_mappings` -- the string appears NOWHERE in
the 2,176-line script, and every `FROM` in the drift path is `instruments` or `aliases`, so a mapping deactivated
with its instrument left active is invisible to every stage. And the baseline ids ARE catalog-fetched once at BUILD
time (`:557` collects them, `:567` hands them to `catalog_rows`), but only to populate display names: nothing
asserts on that result and nothing re-reads it afterwards, which is why this is a population gap and not a missing
query.

NO ONE-LINE FIX IS PROPOSED. Widening `:1439` to union the `baseline` ids would put 61 ids under a check whose
`inactive` bucket goes non-empty IMMEDIATELY, and a baseline pick is a RECORD of what production did, not an
assertion that the row should still be live -- so a permanently non-empty bucket would train a reader to ignore it,
which is the failure mode this section is about. The fix is a decision about what a baseline id's liveness MEANS to
the audit -- a third bucket, or a separate report -- and this entry does not make it.

RE-CHECK, three parts, each able to contradict rather than confirm by construction:
  `grep -n 'o.get("accept"\|o.get("baseline"\|\["baseline"\]' LlmBenchmark/scripts/build_attach_gold.py`
  STILL SCOPED = `stage_drift` and `stage_staleness` read only `accept`; the only `baseline` reads are the
    build-time display fetch and the gold writer. CHANGED = a `baseline` read appears in either check, and the
    population half closes.
  Re-derive the three counts with a fresh reader over both gold files -- union the `accept` ids and the non-null
    `baseline` `instrument_id`s across `owners` + `empty_owner_rows`, subtract, then read the remainder's
    `is_active` from `atlas_secmaster`, SELECT-only. Expect 250 / 121 / 61 today. 61 FALLING means ids were
    accepted into a gold that is supposed to be frozen, which should be visible in git; 61 RISING means new
    baseline-only picks. The DRIFTED count rising past 2 is the movement this entry exists to make visible.
  `python3 -c "import json; d=json.load(open('LlmBenchmark/attach-gold/catalog_drift_v1.json')); print(d['accepted_ids'], d['counts'])"`
  STILL DISAGREEING = `deactivated_in_window` non-zero while `accepted_ids.inactive` is empty.

### Attachment gold residue: slash terms unsearchable by symbol, 6-term cap, wrong labels, IXIC, unembedded additions [2026-09-17, item 4 closed and item 5 added 2026-09-20, item 3 widened to the label-quality class 2026-09-20]
`LlmBenchmark/attach-gold/` (built by `LlmBenchmark/scripts/build_attach_gold.py`, PR #1064) carries five
known defects that were measured and left open. Item 3 is the only one that changes a SCORED owner label;
the rest can mislead the next rebuild or a later scorer without moving a number today.
1. SLASH-JOINED TERMS NEVER REACH THE SYMBOL LEG. `sought_terms` stopped splitting on "/" because
   "USD/JPY" became the term "USD" and exact-matched the ProShares ETF for 13 currency owners; the live
   `probe_check` now refuses that regression. The cost of the fix: a term with "/" fails the symbol regex
   in `notinpool_sql` (`^?[A-Za-z0-9.=-]{1,12}`), so a pair or joined symbols ("GBP/USD exchange rate",
   "USDMXN / DEXMXUS") reach only the trigram and vector legs. Measured on
   `labeller-runs/matchcheck.json`: 38 of 244 whole-catalog checks carry 56 slash-joined terms.
   Re-check: `python3 -c "import json;m=json.load(open('LlmBenchmark/attach-gold/labeller-runs/matchcheck.json'));print(sum(any('/' in t for t in c['terms']) for c in m['checks'].values()))"` -> 38.
   Fix: split joined SYMBOLS on "/" only when each side is itself a symbol-shaped token AND not a bare
   ISO currency code.
2. `sought_terms` KEEPS 6 TERMS. 6 of 244 checks had more distinct terms and lost the rest. On
   g2:170614:u8 the 7th was PPIFIS, the right row; its owner went unresolved and is now an override.
   Others cut: AWHALL (g1 ...0:u19), ETH-USD (g2:156161:u4), 2317.TW (g2:164606:u2). Re-check: rebuild
   the uncapped term list from each owner's `provenance` `sought` strings in the committed gold and count
   lists longer than 6. Fix: rank symbol-shaped terms first and raise the cap; each extra term is one more
   trigram scan and 5 vector rows.
3. LABELS THAT NAME NONE_IN_CATALOG, OR ONE TWIN, WHILE THE ROW IS THERE. Three instances, one of them
   SCORED. The shared mechanism is that a floor or a rule decided the verdict and nobody re-read the row.
   a. TWO EMPTY-OWNER USD/JPY ROWS ARE NONE_IN_CATALOG WHILE DEXJPUS EXISTS. g2:156597:u14 ("the dollar was
      higher at JPY159.37") and g2:158337:u4 ("strengthened 0.45% to 160.11") give yen per dollar, which is
      DEXJPUS (Japanese Yen to U.S. Dollar Spot Exchange Rate, active). The labellers never saw it. It is
      unscored today because the scorer reads only `owners`; it becomes a wrong gold row when empty-owner
      rows are scored. Re-check: `SELECT symbol, is_active, retired_at FROM instruments WHERE symbol='DEXJPUS'`
      (psql is SELECT-only), plus both units' `verdict` in `attach_gold_g2_v1.json`.
   b. g2:161179:u4 "Japan" IS NONE_IN_CATALOG AND IT IS SCORED -- unlike (a), this row is in `owners`, so
      every arm is graded against a label the catalog contradicts. The quote is "Japan's 10-year bond yield
      just climbed above 3% for the first time since 1996"; IRLTLT01JPM156N ("Interest Rates: Long-Term
      Government Bond Yields: 10-Year: Main (Including Benchmark) for Japan") has been active since
      2026-06-02T13:39:43Z. The whole-catalog check DID surface it, at vector 0.6177 with `strong: false`
      -- under the 0.8 cosine floor, so it never reached a round-3 adjudication and the rationale reads "no
      candidate is a Japanese 10-year yield series". The same gold accepts the Korean equivalent
      IRLTLT01KRM156N at g2:166564:u1, so the verdict is inconsistent within one build. Re-check:
      `python3 -c "import json;print(json.load(open('LlmBenchmark/attach-gold/labeller-runs/matchcheck.json'))['checks']['g2|g2:161179:u4']['rows'][0])"`
      -> id b7653e24-6f51-4b37-8c7d-934ecce0b61a, vector 0.6177, strong false; plus that unit's `verdict`
      in `attach_gold_g2_v1.json` and `SELECT symbol, is_active, created_at FROM instruments WHERE
      symbol IN ('IRLTLT01JPM156N','IRLTLT01KRM156N')` (psql is SELECT-only). Relabelling it INSTRUMENT
      costs candidate recall rather than paying it: the row is NOT in that owner's k=20 list in the
      2026-09-20 freeze (it appears once in `frozen/attach_candidates_g2_k20.json`, under a DIFFERENT
      owner, 'Japanese yen' @ 170614), so G2 candidate recall would go 206/237 = 0.8692 -> 206/238 = 0.8655.
   c. g1:sentinel-v6.2-cove.json:183:u1 ACCEPTS PAYEMS ALONE WHILE ITS TWIN CARRIES THE IDENTICAL CATALOG
      NAME. PAYEMS and PAYNSA are both active and both named "All Employees, Total Nonfarm" -- the
      seasonally adjusted and unadjusted forms of one measure. g1:sentinel-v6.2-cove.json:0:u1 accepts
      BOTH, which is the rule the build states for PPIFIS/PPIFID in `overrides_v1.json` ("the two are the
      same measure stored under two symbols and the verdict's set rule takes both"). A re-derivation found
      a SECOND instance the review did not name: g1:sentinel-v6.2-cove.json:4:u3 ("payroll gains") also
      accepts PAYEMS alone, so the class is 2 of the 4 G1 units that accept either symbol. Re-check:
      `SELECT symbol, name, is_active FROM instruments WHERE symbol IN ('PAYEMS','PAYNSA')` (psql is
      SELECT-only) -> two active rows, one name; then the four units' `accept` sets in
      `attach_gold_g1_v1.json`.
   PROVENANCE OF (b) AND (c), AND WHY THERE IS NO RATE HERE: both came from an independent re-read of 25
   owners against the catalog, i.e. 2 disagreements in 25. THE RATIO IS NOT RE-RUNNABLE AND IS NOT QUOTED
   AS A MEASUREMENT: the 25 were not committed -- no unit-id list, no seed, no draw rule -- so nobody can
   say whether they were drawn uniformly over the 691 labelled owners or over some stratum, and a
   denominator that cannot be re-derived is the one figure in this entry that fails toward the reassuring
   answer. The three findings below it each carry their own re-check and stand on their own. To turn it
   into a rate: draw a seeded sample of unit ids from `owners` across both golds, commit the list beside
   `labeller-runs/`, and re-judge it -- then the number can be compared with the next rebuild's.
   Fix: give the whole-catalog check a second pass for owners whose best vector row lands in a band below
   the floor (0.6177 sits there), and make the twin rule a builder step rather than an override written by
   hand per pair -- same catalog name plus same exchange is the predicate the PPIFIS/PPIFID rationale
   already uses.
4. THE CATALOG-DRIFT SCAN HAS TWO HOLES ON THE OWNER SIDE AND ONE ON THE ADDITION SIDE, AND THE OWNER-SIDE
   ONES ARE THE BIG ONES. `build_attach_gold.py drift` scans every row created inside its window on three
   legs, and `assemble --restamp-catalog-at` re-runs the same scan before it will re-date anything -- but a
   leg that searched nothing still returns nothing, which reads exactly like a clean scan.
   Measured 2026-09-20 over 213 additions in (2026-09-17T11:00Z, 2026-09-20T12:02:05Z], 870 owners:
   - **116 owners get no usable vector query** (`vector_leg_fell_back_to_the_unit_key`): no term of theirs
     is 4 characters or longer, so the query text falls back to the unit key and the vector leg is
     meaningless for them. 111 are NO_SINGLE_OWNER; the other 5 are INSTRUMENT owners whose accepted
     symbols the EXACT leg did search.
   - **98 of those carry no term at all** (`owners_with_no_term_at_all`), so all three legs searched
     nothing. Every one is an `empty_owner_rows` row with a BLANK owner surface and verdict
     NO_SINGLE_OWNER. The argument that this is harmless -- no catalog row can own "no single ownable
     thing", so no addition can change that verdict -- is an argument, not a guarantee, and it is written
     here rather than left implied.
   - **an addition with no stored bge-m3 row is out of the vector leg's reach** (`vector_leg_blind_to`),
     reached by the exact and trigram legs only. It was 2 of 212 (WBND, WBNEF) on the 11:17Z measurement
     and 0 of 213 an hour later, so this one self-heals as SecMaster embeds.
   Re-check: those three fields of `LlmBenchmark/attach-gold/catalog_drift_v1.json`, which the stage writes
   on every run. Fix: rank symbol-shaped terms into the vector query and fall back to the owner's QUOTES
   rather than to the unit key; for the blank-surface rows, either exclude them from the scan explicitly
   (and say so) or give them their article's quote as the query.
5. A LABEL CAN FOLLOW A MIS-COPIED QUOTE INSTEAD OF ITS OWNER, AND ONLY ONE INSTANCE HAS BEEN JUDGED.
   The extraction sometimes attaches ONE quote span to several owners; the owner surface and the number's
   unit description then carry the real subject while the quote does not. The worked example is closed:
   g2:168826:u12 ("gold", number description "commodity return") accepted IXIC and NASDAQCOM off the quote
   "Dow and S&P 500 -0.6 per cent, Nasdaq -1 per cent."; it now accepts GC and GC=F, its sibling
   g2:168826:u25 ("Utilities", "sector return") is NONE_IN_CATALOG under rule 5, and g2:168826:u15
   ("Nasdaq", "index return") keeps IXIC and NASDAQCOM -- each decided from the article, which states all
   three phrases verbatim, as rule 1 directs. THE CLASS IS OPEN: **239 owners across 53 articles share a
   quote with a differently-named owner** and none of the other 236 has been re-judged. Re-check: group
   each article's owners by quote string, keep groups whose owner surfaces differ after casefolding, and
   count the owners in them. Fix: label per (owner, quote), or make the builder flag a shared-quote group
   for judgement instead of labelling each member from the same text.

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
| B | 2026-09-20 | AWAITING-DECISION | Tracked-secret inventory, complete: what publishing this repo would expose |
| B | 2026-09-16 | OPEN | Alert-continuity acceptance (sentinel-resolution-signal) re-measured: still NOT met |
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
| E | 2026-08-16 | OPEN | The README's bare '41 shapes' for #935: its provenance (series now in this entry) |
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

**Tracked-secret inventory, complete -- what publishing this repo would expose. AWAITING A DECISION THAT HAS NOT
BEEN MADE (open-sourcing).** Measured 2026-09-20 at `c13a49b6` (= `origin/main`), 2,964 tracked files. This entry
does NOT re-flag the accepted risk recorded below ("Accepted risks, do not re-flag" -- private repo, LAN-only,
public-derived data); the owner's acceptance stands and nothing here asks for it to be revisited. What it adds is
the inventory that acceptance is conditional on, in the owner's words: "The repo is private. If we ever want to
make this open source, then I would be concerned." Publishing exposes the working tree AND all history, so the
inventory is made once, now, while it is cheap. NO VALUE IS QUOTED HERE OR ANYWHERE IN THIS FILE BY THIS ENTRY --
each item is named by path and kind only.

| # | kind | real? | tracked paths | how it reaches production today |
|---|---|---|---|---|
| A | ATLAS app DB password (`atlas_user`, TimescaleDB `atlas_data`) | REAL, and it MATCHES the live deployed value | 24 | ansible-vault `atlas_db_password` renders `compose.yaml.j2`; the 24 tracked copies are dev/devcontainer conveniences that happen to carry the same value |
| B | FRED API key (free tier) | REAL, MATCHES the live deployed value | 2 (`OfrCollector/.env`, `SentinelCollector/scripts/load_secmaster_fred.py`) | ansible-vault `fred_api_key` |
| C | PostgreSQL SUPERUSER password | REAL (it is the live value) and WEAK -- a `change...`-shaped default never changed | 2 (`deployment/ansible/scripts/validate-rendered-template.sh`, `docs/devcontainer-db-bridge.md`) | ansible-vault `postgres_password` |
| D | SMTP password, `OfrCollector/.env` | NO -- placeholder-shaped | 1 | ansible-vault `email_password` |
| E | `.env.example` values (AlphaVantage, Fred) | NO -- placeholders | 2 | n/a |
| F | `ApiKey` in `appsettings*.json` (AlphaVantage, Nasdaq) | NO -- `${ENV}` interpolation, not a literal | 2 | ansible-vault, via compose env |
| G | test/fixture literals: `test-key`, `not-needed`, an `hf_TES...` fixture token, empty-string `ApiKey`/`Password` | NO | ~14 sites | n/a |

A's 24 split 11 shell-default (`${DB_PASSWORD:-<value>}`, the devcontainer compose files) and 13 BARE literal
(2 `appsettings.json` connection strings, 5 SentinelCollector scripts, `init-db.sh`, a quality-check script,
`validate-rendered-template.sh`, 2 docs, and `OfrCollector/.env`). A shell default is normally not a leak, but
these defaults carry the REAL production value, so the syntax does not change the exposure. `docs/BACKLOG.md`
itself is one of the 24 -- the accepted-risks entry below quotes the value in full.

NOT FREE-TIER, and the reason this is class B rather than E: A and C are production DATABASE passwords, and C is
additionally the superuser account at a weak never-rotated default. The owner may want C handled on its own
schedule rather than at open-sourcing, since its weakness is independent of who can read the repo.

CLEAN, measured not assumed -- zero hits for: AWS, GCP/`AIza`, Anthropic, OpenAI, GitHub/GitLab, Slack, Discord,
Stripe, SendGrid, Twilio, npm tokens, JWTs, private-key blocks, certificates, SSH/PGP keys, and URLs with embedded
credentials. The Finnhub, AlphaVantage, Nasdaq, Grafana-admin, Grafana-Google-OAuth-client-secret and ntfy
credentials are vault-only: the live Finnhub key appears in NO tracked file and in NO commit (`git log -S`, 0).

VAULT CHECK, clean: `group_vars/vault.yml` is the ONLY ansible-vault file, and every blob EVER committed at either
of its two historical paths (`ansible/`, then `deployment/ansible/`) is `$ANSIBLE_VAULT;1.1;AES256` -- 9 distinct
blobs, 10 commits, 2025-11-15 to 2026-05-03. None was ever committed in plaintext.

HISTORY, coarse: NOTHING exists only in history. A first entered at `c7b937aa` (2025-12-07, `OfrCollector/.env`,
which is also the only non-`.example` `.env` ever tracked) and spread through the devcontainer and script paths
2025-11-28 to 2026-08-14; B entered at the same commit and reached its second path at `e38f0c56` (2026-05-02).
Both are still in the tree, so neither is history-exclusive, and no removed file carries a secret the tree lacks.
History cleanup, if it is ever wanted, must also cover the pre-rename paths (`ansible/**`, `infrastructure/**`,
`Deployment/compose.yaml`), which carry A, B and C in their old blobs.

TRIGGER -- act on this entry BEFORE any of: making the repository public, transferring it, adding an outside
collaborator, or publishing any mirror, archive or fork outside the owner. Not before. While the repo stays
private the accepted risk governs.

THAT TRIGGER IS A NOTE, NOT A GATE -- NOTHING FIRES, and nobody will be stopped or warned. Measured 2026-09-20:
no hook, no workflow and no CODEOWNERS file exists that watches for it; GitHub's `security_and_analysis` on this
repo is `null` (secret scanning and push protection are OFF, and on a private repo they are a paid feature), and
`/rulesets` answers 403 "Upgrade to GitHub Pro", so there is no ruleset to hang one on either. The Claude settings
DO deny `Bash(gh repo delete:*)` -- in both `.claude/settings.local.json` and `~/.claude/settings.json` -- but NO
rule matches `gh repo edit --visibility`, so the single command that would publish this repo and everything
inventoried above is ungated. Acting on this entry depends entirely on a human REMEMBERING it at the moment they
go public. That may be the right trade while the repo is private; it is recorded here so it is a CHOICE and not a
surprise.
OPTION, deliberately NOT implemented in this round and needing no rotation and no new infrastructure: one line,
`Bash(gh repo edit:*)`, added to the deny list beside the existing `gh repo delete` entry, would turn an AGENT
flipping visibility into a refusal instead of a silent success. It does NOT cover a human doing it in the GitHub
web UI, which stays unguarded whatever we do here.

REMEDY when the trigger fires, per item: A and C -- rotate the DB roles and re-render from vault, then remove the
24 and 2 tracked copies in favour of `${DB_PASSWORD:?}`-style required env (a bare `:-` default silently restores
the leak), and purge history across the pre-rename paths; the rotation touches the DB users and EVERY consumer,
which is precisely why it was deferred. B -- rotate the FRED key (free, self-service) and read it from env in both
paths. C additionally deserves rotation on its own merits, trigger or not. D through G -- no action, they are
placeholders; keep them placeholder-shaped so a future sweep does not re-raise them.

Re-check -- a DRIFT CHECK on the three values enumerated above, NOT a secret scanner and NOT a substitute for one
(none is installed: `command -v gitleaks trufflehog` returns nothing).
It ASSERTS: every count is compared against its expectation, a mismatch names the check and both values, and the
block exits non-zero. It is a subshell, so paste it anywhere and read `$?` -- a failure will not close your shell.
```bash
( set -uo pipefail
  cd "$(git rev-parse --show-toplevel)" || exit 2
  fail=0
  chk() { if [ "$2" = "$3" ]; then echo "ok    $1 = $3"
          else echo "FAIL  $1: expected $2, got $3"; fail=1; fi; }
  # Values are read from their sources, never typed and never printed.
  ENVP=OfrCollector/.env; COMPOSE=/opt/ai-inference/compose.yaml
  PW=$(git show "HEAD:$ENVP" 2>/dev/null | sed -n 's/^DB_PASSWORD=//p')
  KEY=$(git show "HEAD:$ENVP" 2>/dev/null | sed -n 's/^FRED_API_KEY=//p')
  # C is checked by VALUE, not by variable name: the two carriers spell it DB_PASSWORD / PGPASSWORD,
  # so grepping for POSTGRES_PASSWORD returns 0 and reads exactly like "no exposure".
  PG=$(grep -hoE 'POSTGRES_PASSWORD=[^[:space:]]+' "$COMPOSE" 2>/dev/null | head -1 | cut -d= -f2-)
  # THE GUARD THAT MATTERS: a renamed or missing source leaves the variable EMPTY, and `grep -F ""`
  # matches EVERY tracked file -- a 2,959-of-2,964 result that would otherwise read as a clean pass.
  for n in PW KEY PG; do
    [ -n "${!n}" ] || { echo "FAIL  extraction: \$$n is EMPTY (source missing or renamed)"; exit 2; }
  done
  n_pw=$(git ls-files -z | xargs -0 grep -lIF -- "$PW"  | wc -l)
  n_key=$(git ls-files -z | xargs -0 grep -lIF -- "$KEY" | wc -l)
  n_pg=$(git ls-files -z | xargs -0 grep -lIF -- "$PG"  | wc -l)
  chk A_app_db_password   24 "$n_pw"
  chk B_fred_api_key       2 "$n_key"
  chk C_superuser_password 2 "$n_pg"
  # vault: EVERY blob ever at a vault.yml path must be encrypted. Name both historical paths -- a
  # '*group_vars/vault.yml' glob matches none of them -- and filter on $2, because --objects also
  # emits tag and tree names that would otherwise be counted as blobs.
  V=$(git rev-list --all --objects -- ansible/group_vars/vault.yml deployment/ansible/group_vars/vault.yml \
      | awk '$2 ~ /vault\.yml$/ {print $1}' | sort -u \
      | while read -r b; do git cat-file -p "$b" | head -1 | grep -q '^\$ANSIBLE_VAULT;' \
          && echo ENCRYPTED || echo PLAINTEXT; done)
  chk vault_blobs     9 "$(printf '%s\n' "$V" | grep -c .          || true)"
  chk vault_encrypted 9 "$(printf '%s\n' "$V" | grep -c ENCRYPTED  || true)"
  chk vault_plaintext 0 "$(printf '%s\n' "$V" | grep -c PLAINTEXT  || true)"
  [ "$fail" -eq 0 ] && echo "ALL COUNTS MATCH (inventory unchanged)" || echo "RE-CHECK FAILED"
  exit "$fail" )
```
A count that RISES means one of those THREE enumerated values reached one more tracked file; a count that FALLS
without a rotation means a copy moved, not that a secret went away. Either way the block exits non-zero and names
the count, so drift IS a failure and not a wrong number a reader has to notice.
WHAT IT CANNOT SEE -- both directions run 2026-09-20: spreading A to one more tracked file fails correctly
(`FAIL A_app_db_password: expected 24, got 25`), but planting a credential of a kind it does not enumerate -- an
AWS-shaped key pair, a Stripe-shaped key, a `postgres://user:pass@host/db` URL -- in a NEWLY tracked file leaves it
at rc 0, `ALL COUNTS MATCH`. It knows three values and no others; it cannot recognise a credential by shape, so any
secret of a new KIND, or in a location no enumerated value already occupies, is invisible to it. A green run
therefore means THIS INVENTORY HAS NOT DRIFTED -- it is NOT evidence the repo is clean, and it says nothing about a
secret added after 2026-09-20. Before the trigger above fires, re-run the pattern sweep described next, or install
a real scanner; do NOT read green here as clearance to publish.
The sweep that produced the table was pattern-based over all tracked non-binary files: vendor formats
(the "CLEAN" list above), `Password=`/`Pwd=` in connection strings, `api_key`/`token`/`secret`/`password` assigned
to a quoted literal of 8 or more characters, credential-bearing URLs (`scheme://user:pass@`), and high-entropy
standalone tokens -- hex of 24 or more characters (891 hits) and base64 of 24 or more (21,684 hits), both of which
were entirely dashboard UIDs, git SHAs and benchmark payloads with no credential among them.

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
copied. Re-check, after any `--tags alerting` run — every rule in `deployment/artifacts/monitoring/alerts/*.yml`
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
legitimate deactivation the rule fires every 30m forever until someone edits both files AND redeploys the rule
(`--tags alerting --skip-tags always`, which does not reload Grafana or assert the rules landed — see the
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
the rule (name corpus SIZE and baseline) is the `DIFFERENT MIND than the fix` bullet under CONSTRAINTS in
`.claude/skills/supervisor-mode/templates/implementation-fix.md`, which carries the falsified-zero series too;
the spellings half is `.claude/skills/guard-change/SKILL.md` item 1. It was LESSONS.md L15 until 2026-09-20.

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

**Accepted risks, do not re-flag -- WHILE THE REPOSITORY IS PRIVATE.** Plaintext DB password
`atlas_secure_password_2025` in 10+ tracked files, and `OfrCollector/.env` tracked with `DB_PASSWORD` /
`SMTP_PASSWORD` / `FRED_API_KEY` -- tracked since 2025-12-07 (`c7b937aa`), so it is present in every worktree and in
every clone. The user accepted both explicitly: private repo, LAN-only, public and public-derived data, and the FRED
key a free one (re-affirmed 2026-09-20). Rotating the DB password touches the DB user and every consumer.
THE ACCEPTANCE HAS A TRIGGER, and it is not a re-flag: it holds only while the repo is private. Before ATLAS is ever
open-sourced or shared outside its owner this MUST be resolved, because the key and its whole history travel with
every clone and `git rm` alone does not remove either. Remedy for that day: rotate the key, replace the tracked file
with a non-secret default (`OfrCollector/.env.example`, as FredCollector and AlphaVantageCollector already do), and
supply the real value out of band the way those services do. Until then, its being tracked is also what keeps
OfrCollector off the `env_file` defect above -- do not untrack it as tidy-up.

**`__EFMigrationsHistory` is one shared table for every ATLAS service in `atlas_data`.** 58 rows, measured
2026-08-17 (`SELECT count(*) FROM "__EFMigrationsHistory";`). Safe TODAY — EF filters by the migrations assembly
and the IDs are timestamp-prefixed, so collisions need two services to generate the same `yyyyMMddHHmmss_Name` —
but it is a namespace the services compete in rather than an isolation boundary, and nothing enforces the naming
that keeps them apart. Recorded as a known shape, not an action: separate schemas per service would be the durable
answer and it is a migration of the migration table. Re-check:
`sudo nerdctl exec timescaledb psql -U ai_inference -d atlas_data -c "SELECT \"ProductVersion\", count(*) FROM \"__EFMigrationsHistory\" GROUP BY 1;"`
— a duplicate `MigrationId` is the failure mode, and it would surface as a service silently skipping a migration.

**The zero-loosened SERIES for #935 and its salvage — the figures that lived only in `STATE.md`.** The series is
83 rows -> 14 loosened shapes, 181 -> 60, 342 -> 60, 968 -> 88: every zero honest, every one falsified by the next,
bigger corpus, and not converging. It is written out here because L15 carried it until 2026-09-20 and an entry
that graduates takes its evidence with it. The standing rule it bought (any "loosened = 0" must name its corpus
SIZE and the baseline it was measured against; a claim carrying neither is not evidence) now lives in
`.claude/skills/supervisor-mode/templates/implementation-fix.md` CONSTRAINTS, with the spellings half at
`guard-change` item 1.
What this entry still holds is the provenance of one bare figure: `.claude/hooks/README.md:1154` says #935 was
"drifting 41 shapes", with no corpus and no baseline. That 41 is #935 measured against MAIN -- not against its own
previous head, where "loosened = 0" was true each time: 41 shapes opened, 32 of them executing a real write,
sandbox-proved, by an agent-scratch sweep whose row count was never recorded. This entry carries the provenance
the sentence lacks. It used to say that was FORCED -- "the README is gate layer and the guard refuses writes to
it" -- and that is FALSE, corrected 2026-09-20: `ansible-gate-guard.sh` gates `*.claude/hooks/*.sh`, not every
file in the directory, so `.claude/hooks/README.md` is writable by an ordinary agent (scope and the measured
probe: the service-decisions-context entry below). Repairing that bare 41 in the README is permitted work that
was parked as impossible; it is still open, and it is a README edit, not a bypass.

**One conflict-stop site still names only D-entries, and it is the one an agent reads while editing.**
`.claude/hooks/service-decisions-context.sh` injects "If your brief contradicts a D-entry without a named
supersession: STOP and report" as PreToolUse context on every edit to a service with a DECISIONS block. On
2026-09-20 the stop was widened everywhere else in the class — a brief may not contradict a rule stated in a skill,
a template or CLAUDE.md either — but that file is GATE LAYER and the write was refused, correctly: it is a hook
change and wants the guard-change workflow, not a line slipped into a docs round. That workflow has SINCE been
run in full (below) and the write is refused just the same, so what is left is not work but a human-placed
bypass. Until then, an implementing agent's most immediate copy of the stop is the narrow one. Re-check, 1 while
this is open and 0 once it is closed:
`git grep -c 'contradicts a D-entry without a named supersession' -- .claude/hooks/` -> 1 on 2026-09-20.
The rest of the class is closed.
THE FIX IS WRITTEN, VERIFIED AND STILL NOT LANDED -- the second round to be refused, so the blocker is the
gate, not the work. Attempted 2026-09-20 from `.claude/worktrees/agent-abd7329d07fe289ec`: `ansible-gate-guard.sh`
denied the Edit ("this WRITES TO a file in the GATE LAYER"). ITS SCOPE IS NARROWER THAN "everything under
`.claude/hooks/**`", and saying otherwise parks permitted work as impossible: `is_gate_path`
(`.claude/hooks/ansible-gate-guard.sh:196-203`) matches `*.devcontainer/compile.sh`, the bare `.claude/hooks`
directory as a write target, `*.claude/hooks/*.sh` (the `*` spans `/`, so a suite at
`.claude/hooks/test/run-*.sh` is covered too), `*.claude/settings*.json`, and the two marker scripts
`*/scripts/claude-mark-verified` and `*/scripts/claude-pr-verdict`. A NON-`.sh` file under `.claude/hooks/` is
NOT a gate path: measured 2026-09-20 from this worktree with no bypass file, a `Write` to
`.claude/hooks/.gate-scope-probe.tmp` was ALLOWED, so `.claude/hooks/README.md`, the guard tests' `.md` notes and
`mark-verified.log` are all editable by an ordinary agent. What IS refused is every `.sh` in the layer, its own
repair included (the deadlock entry above). Developed and verified instead on the route that deny text
itself prescribes -- `cp -r .claude/hooks <scratch>/hookstree`, edit the copy, run the copy's suite -- and the
patch applies clean (`git apply --check`, rc 0).
THE PASTE TARGET, stated so the paste cannot go wrong: `.claude/hooks/service-decisions-context.sh:70` is the
FIRST of two physical lines of one `jq -n --arg ctx "<two-line string>"` command; line 71 is `$BLOCK" \` and
line 72 is the `'{hookSpecificOutput:...}'` filter, and NEITHER changes. Replace line 70 in full, including the
`jq -n --arg ctx "` that opens the string -- pasting the prose alone leaves the command unopened, the hook exits
NEUTRAL and injects NOTHING, which is a silent TOTAL loss of the stop and strictly worse than the narrow wording.
The line must stay UNWRAPPED or the `a rule stated in a skill` pin stops matching it. Line 70 entire:
`jq -n --arg ctx "SERVICE DECISIONS ($svc_name/AGENT_README.md) — design decisions governing this code. If this brief contradicts a D-entry, or a rule stated in a skill, a template or CLAUDE.md, without a named supersession -> STOP and report, NAMING the rule and the contradiction; never route-around, never obey the stale entry, and never silently obey a written rule you believe is stale.`
Two rows for `run-intent-fidelity-smoke.sh` go with it, and THEY ARE NOT IN THIS REPOSITORY: `.claude/hooks/test/`
is gated the same way, so they exist only on the scratch tree they were verified on and the in-repo suite is
byte-identical to main. One asserts `a rule stated in a skill` is present in the DELIVERED context; one asserts
the narrow wording is absent; both sit immediately after the "injection stops at next heading" row. Teeth proved
by swapping, not asserted: patched hook + patched suite -> both rows PASS; PRISTINE hook + patched suite -> both
go RED and the suite's failure count moves 43 -> 45. (43 is the floor a copied-out tree carries on BOTH sides --
`design-intent-dispatch-guard.sh` resolves the project root from its own location, so every one of its rows fails
in scratch; the pristine tree IN the repo is rc 0 / 132 passed. The `service-decisions-context` section is
location-independent and is clean in all three runs.)
DELIVERY PATH, measured the same day and the reason this site is the one that matters: the hook is wired in the
tracked `.claude/settings.json` under `PreToolUse` matcher `Edit|Write`, and it fires for a DISPATCHED SUBAGENT,
not only an interactive session -- a subagent's `Write` to `AlertService/src/<probe>` returned the injected
`SERVICE DECISIONS (AlertService/AGENT_README.md) ...` block, in its narrow form, as `additionalContext`.
That is the one delivery path that reaches an editing agent whether or not it read CLAUDE.md, a skill or its brief.
The wording is deliberately greppable as ONE short phrase that survives
line-wrapping: `git grep -l 'a rule stated in a skill' -- ':!docs/BACKLOG.md' | wc -l` -> 7 on 2026-09-20 (CLAUDE.md,
supervisor-mode `SKILL.md`, its `implementation-fix.md`, `story-implementation.md` and `spec-plan-authoring.md`,
`intent-review/SKILL.md`, `architecture-cards/CARD_TEMPLATE.md`). A longer phrase measured 4 of the 7 because three
copies wrap mid-sentence -- which is why the pin is short.

**Four tools that three graduated lessons named as their exit condition, and that still do not exist.** L8, L11 and
L15 were deleted from `.claude/skills/supervisor-mode/LESSONS.md` on 2026-09-20 because their RULES had landed in
the dispatch templates, which is what GRADUATION_RULE asks for. Their GRADUATES clauses named something else — a
TOOL that would make the rule mechanical rather than remembered — and deleting the entries deleted the only
re-runnable record that those tools are missing. Each is measured here so the gap stays checkable, and none is
scheduled. The rules themselves are live and are not what this entry tracks.
1. *No hook runs `scripts/verify-citations.py` on changed files.* The `D-n` half shipped (2026-09-04, the
   `WRONG-D-ENTRY` class), but nothing invokes the sweep, so a citation going wrong is caught only when somebody
   remembers to look — and CLAUDE.md TOOL_UPKEEP ANTI says a falling cannot-land count can itself be the drift.
   Re-check: `find .claude/hooks -type f -perm -u+x -exec grep -l '^[^#]*verify-citations' {} + | wc -l` -> 0 on
   2026-09-20, and `grep -c WRONG-D-ENTRY scripts/verify-citations.py` -> 2. A wired hook makes the first non-zero.
2. *No shared mutation helper asserts the built artifact CHANGED.* Every round hand-rolls the mutate/rebuild/restore
   dance, and the failure mode is silent: a `mv` restore once put content below the mutant's DLL mtime and every
   later run scored the FIRST mutant's binary. Re-check: `git ls-files | grep -c mutate-verify` -> 0 on 2026-09-20.
   The named artifact is `scripts/mutate-verify.sh`, asserting the artifact differs by `cmp`/hash, not by test result.
3. *No alert-rules step fails a rule file holding an `alert:` with no positive assertion naming it.*
   `check-assertion-counts.py` is close and is NOT this: it counts positive and negative assertions per FILE against
   a committed manifest, which a file can satisfy while one of its rules has no positive case at all. Re-check:
   `grep -c 'alertstate\|per-alert' deployment/tests/alerts/check-assertion-counts.py` -> 0 on 2026-09-20, against
   `grep -rlE '^\s*- alert:' deployment --include='*.yml' | wc -l` -> 12 rule files in scope. The cost, measured:
   a rule oscillating pending -> inactive through a real ~3% resolution rate, 24 pending cycles and 0 fires in 24h,
   in a file whose promtool suite asserted only silence.
4. *No adversarial corpus is GENERATED from a guard's own rule table.* Hand-written corpora are scoped to their
   author's imagination, which is why each bigger one falsified the last. Re-check:
   `find .claude/hooks/test -type f -perm -u+x -iname '*corpus*' | wc -l` -> 0 on 2026-09-20. Name it `*corpus*`
   when it is built; the check keys on the name because the tool has no other observable.

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
| A | 2026-09-21 | PARKED | D-33 provenance replay -- PARKED by the user "for good (measured net harmful)"; the 70% control bar is already FAILED at 0 of 47 |
| A | 2026-09-16 | AWAITING-DECISION | R2 / S4 observation identity key redesign -- PARKED; trip condition 3 fired 2026-09-16 |
| A | 2026-09-05 | PARKED | #729 regime news-as-staleness redesign -- parked; blockers refuted, un-park is the user's |
| A | 2026-08-26 | PARKED | Candidate-pool epic -- PARKED, asked of the user twice, never started |
| E | 2026-09-16 | PARKED | Claude Code function hooks -- PARKED, not adopted |

**D-33 provenance replay — PARKED "for good (measured net harmful)" on the user's decision, 2026-09-16/17.**
**The DECISION is the user's; the RECORD is the supervisor's.** The supervisor wrote the user's decision
into `STATE.md`'s `STANDING DIRECTIVES (user 2026-09-16/17, still live)` -- supervisor-owned disposable
working memory, not a file the user types into -- where it reads verbatim: *"D-33 replay PARKED for good
(measured net harmful). Sweep ON since 2026-09-17."* Keeping those two apart is the whole reason for this
entry: a standing USER decision was surviving only as one SUPERVISOR-written line in a file that is
designed to be thrown away. The park is **unqualified** -- it carves out no narrowed, targeted or
named-id-list form. Evicted to this file 2026-09-21 because it lived nowhere durable: at `1e1c9125`, "net
harmful" appeared nowhere under `docs/`, `SentinelCollector/` or `.claude/`, and "D-33" appeared nowhere in
this file, so the only copy of it sat in untracked, gitignored working memory. That is not a filing detail
-- the park was in **no durable artifact any review round could read**, so no round on #1098 could have
seen it, and `docs/proposals/delete-wrong-mappings.md` §9 as merged at `1e1c9125` consequently prescribed
the parked act as a **hard prerequisite** for deleting leg A. What those rounds did or did not reason about
is not established here and this entry does not claim it; what is established is that the park was not
available to them.

**THE MEASUREMENT THAT BACKS THE PARK — one completed production dry run, and both rates re-derived at
2026-09-21 from its frozen journal.** The run: 195 batches, every one HTTP 200, mode `dry_run` throughout,
`T_clean` = 2026-09-17T05:10:56.634102Z, completed 2026-09-17T10:01:07.900Z. **It wrote nothing.**
- **Known-good control cohort: 0 of 47 rows stayed — 0.0% against the driver's 70% bar** (recorded by the
  run's own analysis at 2026-09-17T10:01:20.653Z; outcomes `cleared` 44, `replaced` 3). **Re-derived
  independently 2026-09-21 as 0 of 46 = 0.0%** (`cleared` 43, `replaced` 3) by re-running the driver's
  cohort definition against the live catalog and intersecting it with the frozen journal. The cohort is 47
  then and 46 now because exactly one row has been re-extracted past `T_clean` since. **48 rows match the
  cohort join today, and the two rows outside the boundary are not the same kind of row:** `id` 856131 was
  in the 47 and LEFT it (`re_extracted_at` 2026-09-17T11:57:09Z), while `id` 936578 was never in the 47 at
  all -- its own `extracted_at` is 2026-09-19T07:42:56Z, AFTER `T_clean`, so it arrived after the run had
  finished. **46 + 1 = 47 then; 46 + 1 + 1 = 48 in the join now.** (All four counts derived 2026-09-21 from
  the cohort join at the driver's `control_ids()`, whose boundary is `coalesce(re_extracted_at,
  extracted_at) < T_clean`.) **Both derivations agree on the rate (0.0%) and the verdict (FAIL).**
  *Unit:* observation rows in `sentinel.extracted_observations`, counted once each. *Population:* rows whose
  live `instrument_id` is one of the three `GeminiFallback` pre-2026-07-19 control instruments `DB`
  (`c851f27b`), `EVR` (`f8e2689c`), `NMR` (`659223cf`), whose `subject_entity` equals the OLD name the #1053
  rename migration recorded for that symbol ("Deutsche Bank", "Evercore ISI", "Nomura"), attached before
  `T_clean`. These are rows whose ATTACHMENT is known correct and whose NAME was the thing repaired, so a
  stay is the designed outcome for every one of them. *"Stay"* = outcome in (`unchanged`,
  `retained_corroborated`).
- **Overall: 2,142 of 4,843 rows stayed — 44.2%.** *Unit:* distinct observation rows scanned by the dry run
  (4,843 distinct `observationId`, equal to the summed per-batch `scanned`, so no row is double-counted).
  *Population:* rows attached before `T_clean` to any of the 197 instruments D-33 admits at all. Full
  outcome split: `unchanged` 2,086 / `replaced` 934 / `retained_corroborated` 56 / `cleared` 1,767 /
  `unavailable` 0 / `refused_known_publisher` 0.

**Failing the control bar is not a tuning result — it is the D-21 signature the replay's own guard says must
stop the run.** D-21: the resolve-only cascade is strictly NARROWER than the live one, so it strips
attachments it cannot reproduce. `control_gate` in `SentinelCollector/scripts/reresolve-by-provenance.sh`
carries that reading in an `INTENT(D-33)` comment — *"below the bar the replay is narrower than live and
would destroy genuine issuer attachments (the D-21 signature): refuse --live, never lower the bar."* At 0.0%
the replay cleared or replaced **every** attachment a correct live resolution had made on the control
family. That is the "net harmful".

**WHAT THIS SAMPLE STRUCTURALLY CANNOT CONTAIN.** It is a DRY RUN, so it contains no downstream effect of a
clear on any consumer — the harm is inferred from the proposed outcomes, never observed. `unavailable` is
**0 of 4,843**, so the outage branch D-33's PRECOND turns on ("an outage is never a miss") was never
exercised: the sample says nothing about behaviour when a free leg is down. It covers only rows attached
BEFORE `T_clean`; rows attached after are absent by construction, and one control row has already left that
way. It covers only instruments inside the 197 — every wrong mapping on an ACTIVE non-`GeminiFallback` row,
the far larger surface, is outside it. And the control cohort is **three symbols**: it establishes that the
replay destroys correct attachments on `DB`/`EVR`/`NMR`, not the rate at which it does so elsewhere.

**THE FOUR BLOCKERS, and they are not the same kind of thing.**
1. **THE DECISION [a human arbitrates; nothing else lifts it].** The park above. It is a recorded user
   decision, so it is not falsifiable by measurement and no re-measurement below reopens it. Per the
   standing autonomy grant, overriding a recorded decision is explicitly outside what an agent may do.
2. **WORLD CONDITION — the sweep is ON.** The running `sentinel-collector` carries
   `ReExtract__Enabled=true`, read 2026-09-21T18:45Z at the driver's own read path (`nerdctl exec
   sentinel-collector cat /proc/1/environ`; host PID 3870571, started 2026-09-21T04:12:59Z) and confirmed
   doing work — `sum(increase(sentinel_reextract_rows_processed_total[6h]))` = **3,269.27** at
   2026-09-21T18:45:26Z (unit: extrapolated counter events over 6h). `sweep_off()` fails closed (an absent
   variable reads as ON), and `preflight_refusal` tests it **before** its `if not live: return None`, so
   this refuses the **dry run** as well as `--live`. Separately, the service itself refuses a live run with
   409 on the same option. This condition could change on its own, with a deploy nobody thinks of as
   touching D-33.
3. **WORLD / STRUCTURAL — the gates cannot be satisfied by a narrowed id list, which is the form §9
   proposed.** The control cohort is a FIXED query over `DB`/`EVR`/`NMR`; none of those three is among the
   12 leg-A ids or the 82 (verified 2026-09-21), and `control_gate` refuses outright when a control row is
   absent from the dry run. So a 12-id run cannot pass the bar — it cannot even be **scored** against it.
   A live run additionally needs a COMPLETE dry run for that exact `T_clean` **and** id set, and the only
   recorded completion marker carries the 197-id hash (`61e8b728…`). A fresh `T_clean` is not a way out:
   `record_t_clean` refuses while the sweep is on, and refuses to move one that already exists.
4. **THE MEASUREMENT — the 70% bar is already failed** (above). This is the evidence BEHIND blocker 1
   rather than a fourth independent fact, and it is listed separately because §9 named the bar as a thing
   to get *past*, as though it had not yet been tried.

**TWO FIGURES CORRECTED while filing this, both relayed wrong into the dispatch that produced this entry —
do not re-inherit them.** (a) The control result is **0 of 47 = 0.0%**, not "1 of 71 = 1.4%"; no artifact
under the run tree carries a 71. (b) **`A2b − A2` is 0, not non-zero, over the 12 leg-A ids** at the
recorded `T_clean` (A2 = 26, A2b = 26, derived 2026-09-21), so the escaped-rows check would PASS for step
0's population and is NOT a blocker there. It is **1,006** over the full 197 (A2 3,846 / A2b 4,852) — the
claim was true of the parked population and was carried across to the narrowed one. That check is also
live-only: on a dry run the driver only WARNS.

**THE 1,006 CARRIED NO TIMESTAMP AND NO LONGER REPRODUCES — IT IS 1,009, AND THE DRIFT IS IN THE SAFE
DIRECTION** [re-derived 2026-09-21T19:51:46Z, `atlas_data` SELECT-only, both counts written fresh from the
definitions at `reresolve-by-provenance.sh` `count_a2` and `count_a2b` rather than by invoking the driver,
which is parked in every mode]. **A2 = 3,843, A2b = 4,852, delta 1,009** over the 197-id population at the
recorded `T_clean` 2026-09-17T05:10:56.634102Z. *Unit:* rows in `sentinel.extracted_observations`, counted
once each. *Population:* rows whose live `instrument_id` is one of the 197 (re-confirmed 197 the same
minute). Only A2 moved: it fell by 3 as the re-extract sweep pushed rows past `T_clean`, which is what
`coalesce(re_extracted_at, extracted_at) < T` does as `re_extracted_at` lands. **A2b cannot move in that
direction at all** — it keys on `extracted_at`, which is immutable once written, and its
`review_notes NOT LIKE '%[re-resolve D-33%'` term can only be narrowed by a LIVE replay, which has never
run. So the delta is monotonically NON-DECREASING while the sweep is on. The check refuses `--live` when
the delta is `> 0`, so growth makes it refuse HARDER, never softer: this figure drifting is a stale NUMBER,
never a weakening GUARD, and re-deriving it can only ever confirm the refusal.

**AND IT IS STILL NOT THE BLOCKER FOR STEP 0**, whose population is the 12 leg-A ids, not the 197 — that is
the whole point of the correction above and it survives the re-derivation. Re-derived 2026-09-21T19:52:28Z
over the 82 leg-A ids (`NOT is_active AND discovery_source='GeminiFallback' AND asset_class IN
('Equity','ETF')`, which returns exactly 82, the ids holding no rows contributing nothing): **12 distinct
`instrument_id`s, 26 rows, A2 = 26, A2b = 26, delta 0**, unchanged. The 93 quarantined rows partition
82 / 9 / 2 across legs A / B / C, which is the same 93 the SecMaster card now carries.

**§9's ACT-GATE PREAMBLE DOES NOT DESCRIBE STEP 4's OWN ACT GATE, AND NOTHING TURNS ON IT — SAID HERE SO THE
NEXT READER DOES NOT RE-FIND IT AS A DEFECT.** `docs/proposals/delete-wrong-mappings.md` §9 defines the two
gate kinds as **D (decision)** = "a question only a human answers" and **A (act)** = "a completed,
verifiable act or an elapsed window, **which the implementing agent satisfies**". Step 4's gate reads
"**A** — step 3's 30d window elapsed AND the live scoring epic closed". The 30d window fits the definition;
the scoring epic closing does not — no implementing agent satisfies another epic's closure, which makes it
a world condition of the same kind as blocker 2 above (the sweep being on) rather than an act. It is a
MIS-CLASSIFICATION IN THE PREAMBLE'S WORDING, not a wrong gate: the condition itself is correct and the
plan's own §6 gives its reason (drift-audit blindness plus 322 of 348 attachments). Nothing turns on it
because step 4 ALSO carries a **D** on the same row — the 26-row question answered again for leg C — so a
human is already required before step 4 runs, and no agent can reach it by satisfying acts alone. Filed
rather than fixed for that reason; if the preamble is ever tightened, the honest third kind is "a world
condition nobody here controls", which blockers 2 and 3 are too.

**WHAT WOULD HAVE TO BE TRUE TO UN-PARK IT.** All four, and only the first is a decision:
1. **The user reverses the park explicitly.** Nothing below substitutes for this, and none of 2-4 is
   evidence that it should happen.
2. `sentinel-collector` is deployed with `sentinel_reextract_enabled=false` and the running process shows
   `ReExtract__Enabled=false`.
3. A control cohort that the intended run population **actually contains**, passing at ≥ 70% — which for
   any narrowed id list means the cohort definition itself has to change first, and that change is a D-33
   contract question, not a driver tweak.
4. A fresh `T_clean` recorded after a clean catalog audit, and a complete dry run for that `T_clean` and
   that exact id set.

**Consequence if this entry is missing or false:** an agent reads merged plan §9, runs the replay over a
named id list believing it sanctioned, and the narrower cascade strips correct instrument attachments from
production observation rows — the class that reaches `public.matrix_cells`. That is precisely the path
#1098 left open.

**Provenance, and its own fragility:** journal at `/tmp/sentinel-remediation/junk-names/deploy/run/`
(`responses.jsonl`, `dry_run_complete.json`, `t_clean`) with the run's analysis at
`/tmp/sentinel-remediation/junk-names/dryrun/03-analysis.txt`. `/tmp` here is ext4, not tmpfs, so it
survives a reboot — but it is outside the repo and inside the tree agents are authorised to `rm -rf`. Every
figure above is stated here so the entry survives the journal.

Re-check — **none of these runs the replay, and none of them may be used to un-park it; they establish only
whether blockers 2 and 3 still stand.** Anchor every metric query to an actual `date -u`.
  `sudo nerdctl exec sentinel-collector cat /proc/1/environ > /tmp/e.raw; tr '\0' '\n' < /tmp/e.raw | grep ReExtract__Enabled`
  # 2026-09-21T18:45Z -> ReExtract__Enabled=true. This is the exact read `sweep_off()` performs.
  # `nerdctl container inspect` is useless here: its `.Config.Env` is empty on nerdctl 1.7.7.
  `sum(increase(sentinel_reextract_rows_processed_total[6h])) or vector(0)`
  # 2026-09-21T18:45:26Z -> 3269.27. A zero here means the sweep is idle, NOT that it is disabled — read the
  #   environ for that; the two can disagree.
  Both rates, from the frozen journal: sum `body.outcomes` across `responses.jsonl` for the overall rate;
  for the control rate, re-run the cohort definition at `reresolve-by-provenance.sh` `control_ids()` and
  intersect the ids with the journal's `body.rows[].observationId`. Expect the cohort to keep SHRINKING as
  the sweep pushes rows past `T_clean` (47 on 2026-09-17, 46 on 2026-09-21); a shrinking cohort is drift in
  the sample, never an improvement in the rate.
  Containment, which is what makes the park cover §9's narrowed list:
  `SELECT count(*) FROM instruments WHERE discovery_source='GeminiFallback' AND created_at < TIMESTAMPTZ '2026-07-19T00:00:00Z';`
  # 2026-09-21 -> 197, exactly the id set the parked run covered. All 82 leg-A ids, all 12 that still hold
  #   live attachments, and both leg-C ids are inside it, so no subset of legs A or C escapes the park.
  The escaped-rows delta, WITHOUT invoking the driver -- read `count_a2` and `count_a2b` in
  `SentinelCollector/scripts/reresolve-by-provenance.sh`, write the two SELECTs yourself against
  `atlas_data`, and run them over the 197 ids and the recorded `T_clean`. Running the driver for this is
  NOT an option: it is parked in every mode, dry run included.
  # 2026-09-21T19:51:46Z -> A2 3,843 / A2b 4,852 / delta 1,009 over the 197; 26 / 26 / 0 over the 82 leg-A
  #   ids at 19:52:28Z. EXPECTED = the 197 delta at or ABOVE 1,009 and the leg-A delta at 0. A 197 delta
  #   that has FALLEN means A2b shrank, i.e. rows were deleted or a live replay wrote `[re-resolve D-33`
  #   notes -- neither should have happened under the park, and either one is the finding, not the delta.

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
(`deployment/ansible/playbooks/deploy.yml:1217-1218` and `deployment/ansible/playbooks/deploy.yml:1513-1514`, the
cascade incident), and both runs agree with it.
Re-check: `sudo nerdctl container inspect <svc> --format '{{.Created}}'` per service — `container` is
load-bearing (CLAUDE.md VERIFY_TRAP: bare `inspect` resolves the IMAGE and hands back the BUILD time).

**An empty INSTANT query on a cumulative counter is not evidence the counter never fired.** Range-query it (or
`increase()` over the window) before concluding absence; the trap was hit on the pruner counter (2026-08-27, #995).
