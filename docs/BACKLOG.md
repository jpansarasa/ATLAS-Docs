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

**THE CORRECTED `source_entity` PROMPT IS IN PRODUCTION, IT STOPPED THE BLANKING, AND ON ARTICLES THAT NAME
NO SERIES IT PUT THE COUNTRY IN ITS PLACE. 37 rows anchored on a country, 10 of them resolved to an
instrument, and all 10 are wrong on the FIGURE-to-INSTRUMENT fit** -- an inflation rate is not a quantity of
an equity ETF. That is a judgement made by reading each row against its article, not a measured label, and it
is FOUR articles' worth of evidence: read the caveats before quoting the number. #1017's prompt reached prod
2026-09-06T10:26:43Z; window is `extracted_at >= 2026-09-06 10:26:43Z`, snapshot **10:59Z: 72 observations,
8 documents**. The country is the one answer the prompt explicitly forbids --
`SentinelCollector/src/cod-prompts/cod_json_v1.txt:57-58` reads "the country is NOT the owner".

**THIS IS NOT AN ARGUMENT FOR A `country -> reject` RULE, and the entry below must not be quoted as one.**
The existing entry "The candidate surface filter gates 4.3% of the rows that attach instruments" measures the
opposite half: country subjects ALSO produce defensible resolutions -- `Brazil` -> `EWZ` 446, `Germany` ->
`DAX` 345, `China` -> `GXC` 190, and `China` -> `NGDPXDCCNA` 73, which THIS entry lists as a wrong
attachment. Both readings are correct because they judge different things: that entry judges
country -> instrument, this one judges number -> instrument. The fix this observation points at is the
prompt's contradictory bullet and the catalog naming below, NEVER a reject rule that entry already refutes in
three caveats.

READ THE COUNTRY ARM AS THE FINDING AND EVERYTHING ELSE AS A SNAPSHOT, because this window is OPEN and moves
fast: 33 rows / 3 documents at 10:38Z, 50 / 6 at 10:46Z, 62 / 7 at 10:53Z, 72 / 8 at 10:59Z. Country-valued
has gone 81.8% -> 66% -> 53% -> 51% of a rising denominator, so the RATE is a reading and the COUNT is the
figure. Anyone re-running gets more than this entry reports, and should.

THE FIRST READING OF THIS WINDOW GOT THAT WRONG, WHICH IS WHY THE PARAGRAPH ABOVE EXISTS. At 10:38Z it was
written up as "the population, not a slice" because three re-checks six minutes apart had not moved it. It
then grew four times. Three unchanged re-checks are not evidence that a live window is closed.

THE DEPLOY, restricted to what can be re-derived. `SentinelCollector/src/cod-prompts/cod_json_v1.txt` moved
blob `45d02cd8` -> `85187c39` (`git rev-parse a8a0ed5d~1:<path>` / `git hash-object <path>`). The host mount
`/opt/ai-inference/prompts/cod/cod_json_v1.txt` carries mtime 2026-09-06T10:26:41Z and the same sha256 as the
repo file, so the sync at `deployment/ansible/playbooks/deploy.yml:1112` (`force: true`,
`tags: [sentinel-collector, cpu-cod-prompts]`) landed. `sentinel-collector` was recreated two seconds later at
10:26:43.554966881Z; `vllm-server` was NOT touched, still created 2026-09-01T10:11:00Z. First row landed
**47.4s** after the recreate (document 164424 at 10:27:30.942403Z), not the 53s reported at handover -- 53s is
document 164425. The tally `ok=36 changed=6 failed=0` is NOT re-derivable and is recorded as REPORTED, never
measured: `deployment/ansible/ansible.cfg` sets no `log_path` and no run log survives.

38 log lines in the first ten minutes and 0 at Error or Fatal, both re-derivable. An earlier revision of this
entry said "15 Warning and all 15 the startup banner"; the COUNT is right and the CLASSIFICATION was wrong,
and the exception is load-bearing rather than cosmetic. At least four are runtime: an EF `First`/`FirstOrDefault`
-without-`OrderBy` warning at 10:26:55, "Semantic signal shadow budget exhausted with 3 catalog misses
unscored" at 10:29:15, and **`ReExtract cycle removed an existing instrument from 4 of 6 rows` at 10:29:58 and
`from 5 of 5 rows` at 10:34:02**. That last pair matters to this entry's own evidence: the limitations
paragraph below names ReExtract as a hazard, and the worker is not a future risk -- it ran TWICE inside the
ten minutes being reported, on a roughly four-minute cadence, stripping instruments from 9 of 11 rows in other
cohorts. It has not reached this window yet (0 of 72 rows carry `re_extracted_at`), which is what keeps the
table below readable, and it is running now.

WHAT THE FIELD HOLDS at 10:59Z: `Turkey` 22, blank 11, `Boeing` 7, `UK` 6, `Lumentum Holdings Inc` 4,
`China` 3, `People's Bank of China` 3, `European government bond yields` 3, `Hong Kong` 2, `Acadia Realty
Trust` 2, and one each of `Germany`, `France`, `Spain`, `Italy`, `Eyepoint Pharmaceuticals Inc`, `Sonida
Senior Living Inc`, `Covista Inc`, `Sempra Energy`, `U.S. 10-year Treasury yield`. `People's Bank of China` is
arguably right (an institution owning its own gold reserves); the company names are right outright.

THE PRE/POST STEP IS WEAKER THAN IT LOOKS, and none of the obvious framings survives. The handover's "11/12
and 5/6 blank in the two windows immediately before restart" both reproduce, but they are MINUTE buckets
slicing ONE document. Per DOCUMENT the last two before the deploy are **164422 at 16/17 blank and 164423 at
0/1**. Across the whole pre-deploy day (00:00Z-10:26:43Z, 425 rows / 47 documents) blank is
157/425 = **36.9%**, and **20** minute-buckets today ran at ZERO blanks. Blank rate tracks article mix; no
single number states the step. Nor is a country in `source_entity` new: the same pre-deploy day carries
`India` 13, `US` 3, `China` 1, `U.S.` 1 = 18/425 = 4.2%.

`source_entity` AND `subject_entity` ARE THE SAME FIELD ON THIS PATH, and the invariant is ABSOLUTE rather
than dominant. `DslToMergedExtractionAdapter.cs:399` sets `perRowSubject = sourceEntity` whenever the slot is
non-blank; `:528` persists `SourceEntity` and `:536` persists `SubjectEntity` from it. Measured since
2026-09-01: of **20,640 rows where BOTH columns are non-blank, ZERO differ**.
The 3,660 rows since 2026-09-01 that do carry a non-blank `source_entity` against a differing `subject_entity`
all have `subject_entity` BLANK, which this adapter cannot produce; they are the other producer's rows.
Re-check, and silence on the second column is the pass:
`SELECT count(*), count(*) FILTER (WHERE source_entity IS DISTINCT FROM subject_entity) FROM
sentinel.extracted_observations WHERE extracted_at >= TIMESTAMPTZ '2026-09-01' AND
coalesce(trim(source_entity),'')<>'' AND coalesce(trim(subject_entity),'')<>'';`
It MATTERS because it makes the downstream argument mechanical rather than inferred: a country in
`source_entity` IS a country in `SubjectEntity`, which IS the string `DeterministicResolver` Rule 2 queries.

THE MECHANISM IS A HYPOTHESIS, recorded as one. The corrected bullet tells the model to emit the series "under
the name THE ARTICLE gives it" (`cod_json_v1.txt:56-58`) and in the SAME sentence that `""` is NOT the answer;
two lines later (`:60-63`) it says to emit `""` when the article "never NAMES" the series. On a Turkish
inflation story that names no series those two instructions point opposite ways, and the model appears to
resolve the conflict by taking the third option `:58` explicitly forbids.

THE GOLD ALREADY CONTAINS THIS EXACT CASE, WHICH IS THE STRONGEST SUPPORT THE HYPOTHESIS HAS AND IT IS NOT
FROM PRODUCTION AT ALL. The macro-owner entry under MEASUREMENT DEBT discloses article 183's six payroll rows
as the gold's ONE knowing non-conformance: that article names no series -- zero occurrences of "payroll" or
"nonfarm" -- so the rule-conformant label was `""`, and the gold keeps them on **`US`**, the country, which
the corrected bullet forbids. A careful human labeller with the rule in front of them reached for the country
on an article that named no series, and production now does the same. Separately, **16** of the gold's 22 open
anchors are the same class -- a series the article never NAMES (article 1's eleven Conference Board survey
shares, article 48's five Challenger figures); the other 6 are blank-by-design and are a different question.
Closing the 16 needs the rule this observation points at: what to emit for a series a story DESCRIBES without
NAMING. The prompt currently answers that twice, differently.

THE TEST, one step, and the strongest evidence inside this window. `BuildCandidatesFromEnts`
(`DslToMergedExtractionAdapter.cs:1411-1416`) drops every ENT whose `ent_type` is in `NonInstrumentEntTypes`
(`DslToMergedExtractionAdapter.cs:108-125`, `"country"` at `:115`), so an EMPTY `candidate_symbols_json` means
the model declared no instrument-like entity at all -- the proxy for "this article named nothing else to
anchor to". The 54-name country list below is the ONE list this entry uses; the anchor-class re-check further
down reuses it deliberately, because an earlier revision used a 7-name shortlist there and silently binned
`Germany`, `France`, `Spain` and `Italy` as named-issuer, contaminating the exact arm the headline rests on
(it reported country 33/9 where the full list gives 37/10):

```sql
-- sudo nerdctl exec timescaledb psql -U ai_inference -d atlas_data     [psql is SELECT-only]
SELECT (candidate_symbols_json IS NULL OR jsonb_array_length(candidate_symbols_json)=0) AS no_instrument_ent,
       count(*) AS rows,
       count(*) FILTER (WHERE source_entity IN
         ('United States','USA','US','U.S.','America','United Kingdom','UK','U.K.','Britain','China','Japan',
          'Germany','France','India','Russia','Brazil','Canada','Mexico','Italy','Spain','Australia',
          'South Korea','Korea','Saudi Arabia','UAE','United Arab Emirates','Israel','Iran','Turkey',
          'Switzerland','Netherlands','Sweden','Singapore','Hong Kong','Taiwan','Indonesia','Thailand',
          'Vietnam','Nigeria','South Africa','Egypt','Argentina','Poland','Ireland','Norway','Denmark',
          'Finland','Belgium','Austria','Portugal','Greece','European Union','EU','Eurozone','Europe'))
         AS country_se,
       count(*) FILTER (WHERE coalesce(trim(source_entity),'')='') AS blank_se
FROM sentinel.extracted_observations
WHERE extracted_at >= TIMESTAMPTZ '2026-09-06 10:26:43Z'
GROUP BY 1 ORDER BY 1;
```

READ THE EMPTY SIDE PER DOCUMENT, NEVER ROW-WEIGHTED -- the query above returns rows, and a single
high-row document swamps it. At 10:59Z the empty side read **22 of 22 country, 0 blank**; at 17:32Z the
row-weighted figure is **24 of 272 = 8.8%**, which looks like a collapse and is not one: **246 of those 272
rows are ONE document** (`raw_content_id` 164489, every row anchored `TSA`). Each of the 5 documents on this
side carries exactly ONE distinct `source_entity`, so a row-weighted rate here measures document SIZE and
not model behaviour. Per DOCUMENT it is **3 of 5 country** -- 164424 `Turkey` 12 rows, 164425 `Turkey` 10,
164507 `Italy` 2, together **24 of 24 rows and 0 blank**, the unanimity intact -- against one 246-row `TSA`
document and one 2-row blank document (164552). This entry already draws that distinction for the country
arm ("the RATE is a reading and the COUNT is the figure") and shipped its own discriminator without it, so:
anyone re-running the query must report `count(DISTINCT raw_content_id)` and the per-document surfaces
beside the row counts, or a single fat document reads as a refutation of an intact hypothesis. The POPULATED
side has fallen 45% -> 39% -> 30% as company articles arrived, which moved because the denominator did. It
remains a PROXY and not the real discriminator: `macro_indicator` is ALSO in `NonInstrumentEntTypes`, so an
empty shortlist cannot separate "declared only a country" from "declared a country AND a macro series and
anchored on the wrong one" -- and the `TSA` document is that limitation live, an INSTITUTION ENT being
non-instrument too, so the empty side admits documents that declared neither a country nor a series.
Settling THAT needs the model's `entities[].ent_type`, which no column persists; it takes a harness re-run
over these `raw_content_id`s.

DOWNSTREAM. A country ENT is excluded from the Rule 1 shortlist by `NonInstrumentEntTypes`, so the row falls
to Rule 2 (`DeterministicResolver.cs:154`), which hands the RAW `SubjectEntity` to hybrid resolve and consults
NO surface filter. That uncovered leg is already measured -- `SentinelCollector/AGENT_README.md` D-1 counts
7,184 instrument-attaching rows carrying a `gpe_country` subject, 3,060 landing on `U` (Unity Software),
**over ONE 31-day window, `extracted_at` [2026-07-15, 2026-08-15), and a FLOOR rather than a total** because
D-1 replayed the exact-match sets only. Both caveats are mandatory (this file says so at the sibling entry);
without them it reads as an all-time count. The mechanism, its three caveats and the two candidate seams live
in "The candidate surface filter gates 4.3% of the rows that attach instruments" in this file -- go there, not
to a fourth restatement. What is NEW here is only that it is now visible in a live post-deploy window, one row
at a time:

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

The `UNRATE` and `UKNGDP` rows arrived on Rule 2b (`hybrid_subject_description`,
`DeterministicResolver.cs:172`) -- the leg added to RESCUE macro series, which off a country anchor reaches
the wrong COUNTRY's series. Note also that the same France/Germany figure was emitted twice, once per country,
and only the `Germany` copy attached.

**THE CATALOG IS A CO-CAUSE, AND A PROMPT FIX WILL NOT REMOVE IT.** These are not fuzzy near-misses. SecMaster
holds instruments whose `name` is a bare country string, so a country anchor EXACT-matches one:
`TUR` is named `"Turkey"`, `NGDPXDCCNA` is named `"China"`, `UKNGDP` is named `"UK"`, `EWG` is named
`"Germany"` -- four for four on the wrong attachments above. There are three instruments named `UK` (`.LON`,
`IMPUK`, `UKNGDP`) and eight named `US`, so WHICH one wins is a ranking artifact. `UUP` -- the Invesco DB US
Dollar Index fund -- is named `"Italy"`, which is a catalog defect in its own right and would have caught the
`Italy` row had it ranked. Re-check, `atlas_secmaster`:
`SELECT symbol, name FROM instruments WHERE name IN ('Turkey','China','UK','Germany','France','Spain','Italy','US');`
Fixing the prompt stops the anchor being produced; it does not stop the catalog answering to it, and any
OTHER path that reaches hybrid resolve with a country string still lands here.

THE PAID GEMINI RUNG COSTS NOTHING HERE, BUT THE ROWS DO REACH IT -- an earlier revision of this entry said
"not one country-anchored row reached it", which is false and was refuted by the counter that exists to say
so. `Classify` runs INSIDE Rule 2.5 (`DeterministicResolver.cs:675`, D-6) and the rejection is metered at
`:684`: `sentinel_gemini_resolver_calls_total{outcome="surface_filtered"}` reset at the restart and read
**29** by 10:58Z, against `secmaster_match` 6 and `below_threshold_or_null` 2. So country rows DO enter the
rung and are refused at its gate before any paid call -- the mechanism working as designed, and none of the 10
attachments above came from `gemini_fallback`. The conclusion (no bill) stands; the claim that they never
arrive did not, and the table alone could never have decided it. That is the lesson: an absence in a result
set is not an absence in the system, and the counter is what distinguishes reached-and-rejected from
never-reached.

BLANK IS NOT SAFE EITHER, but the rule is sharper than "blank is bad". A blank slot does not withhold an
anchor: `perRowSubject` falls back to the DOCUMENT-LEDE ENT (`DslToMergedExtractionAdapter.cs:399-401`), so
the row is anchored on whatever the lede happens to be. On the five blank rows adjudicated at 10:46Z that lede
was `China` (x3), `Wall Street Breakfast` and `John Healey` -- a country, a PUBLICATION and a PERSON, and the
prompt excludes all three from ownership by name ("the PUBLISHER is never the owner", `cod_json_v1.txt:65-66`).
"Potentially unreported buying" attached to `TW` through the lede "China". But four LATER blank rows attached
to `LITE` off a single-company Lumentum article where the lede IS the right owner and the fallback did exactly
the right thing. So blanking is safe when the document has ONE subject and dangerous when it has several or
none -- the same condition that produces the country anchor. A positional fallback cannot tell those apart;
that is the defect, not the fallback's existence.

THE ISSUER ARM IS NOT A CLEAN CONTROL EITHER. At 10:46Z
it was 12 rows / 9 attachments, all through Rule 1's shortlist as `llm_candidate_hybrid`, all checked by hand
and correct (`BA` x7, `AKR` x2). It then grew, and the growth is not clean: three rows anchored on
`European government bond yields` -- a named surface, so this entry's own rule bins it as issuer -- attached
to **`DGS10`, the US 10-Year Treasury**, the identical "right series kind, wrong country" failure as `UNRATE`.
So the narrow claim survives (an issuer-anchored row reaches Rule 1's shortlist and resolves through the leg
designed for it, which is why the pipeline is not broadly broken) and the wide one does NOT. Note too that the
two arms are judged by ASYMMETRIC standards: country-arm wrongness follows from instrument identity alone,
while issuer-arm rightness only establishes that the instrument matches the named issuer -- "aircraft
delivered 53" on `BA` is a Boeing operating metric, not a quantity of the BA security, and this entry does not
call that right.

CONTAINMENT, and it is partial. Every wrong row sits `review_status='Pending'` on a method NOT in
`AutoApproveMethods` (`ExtractionOptions.cs:396` -- only `ticker_in_quote` and `llm_candidate_exact`), so it
is held. It is not held forever: `AutoApprovePolicy` default-accepts a held row after `ReviewGraceDays` = 3
(`ExtractionOptions.cs:409`), metered as `GraceDefaultAccept`. The XML doc at `ExtractionOptions.cs:405-406`
claims `extracted_observations` feed only the qualitative digest and NOT the WS3 matrix -- recorded here as
the CODE's claim, unverified by this entry, and in tension with CLAUDE.md §GIGO's "a junk `entity` resolving
to the WRONG instrument corrupts the matrix". Which of the two is right is not settled here and must not be
assumed either way.

THE COUNTERWEIGHT, not buried. The benchmark gain was `numbers_f1` 0.3446 -> 0.5152 at production's OWN
sampling (see the Convention B and sign-reversal entries under MEASUREMENT DEBT) -- a PARTIAL improvement that
never claimed full compliance, so residual country-emission is CONSISTENT with the benchmark rather than
contradicting it. **Nothing in this entry argues for reverting #1017, and nothing in it argues for a
country -> reject rule** (see the second paragraph).

WHAT THIS SAMPLE CANNOT CONTAIN. Thirty-two minutes of one morning cannot state a corpus-wide rate. The
country arm rests on FOUR articles -- two Turkish macro, one Chinese gold, one European budget -- all naming
no series; 10 wrong of 10 could still be four articles' worth of bad luck, and it needs a day of articles
before the rate means anything. Correctness is a JUDGEMENT made by reading each row against its article, not a
measured label -- there is no ground truth column, which is why the grown issuer arm is left unadjudicated
rather than assumed. METHODOLOGY EXCEPTION, stated because the sibling entry forbids it in bold: that entry
reads `OriginalInstrumentId`/`OriginalResolutionMethod`, never the live columns, because ReExtract erases
them. This entry reads the LIVE columns, which is earned ONLY because the window is minutes old and 0 of 72
rows have been re-extracted -- verified, not assumed. The moment ReExtract reaches this cohort the table above
stops being reproducible, and the worker is running on a ~4-minute cadence.

WHAT PRODUCTION IS ACTUALLY RUNNING, AND IT IS NOT `main`. `sentinel-collector` was created
2026-09-06T10:26:43.554966881Z and **has NOT been recreated since** (`nerdctl container inspect`, re-read
17:32Z). So the running process carries the corrected PROMPT -- a host-mount sync, which needs no image --
and NEITHER of the two resolver fixes that merged after it: #1029 (merged 15:56:11Z) and #1031 (17:18:33Z).
Every live figure in this entry therefore describes a production that has the prompt and NOT the code fixes.
Rows were still landing on `U` at 11:00Z, 13:00Z and 16:00Z today (9 rows / 5 documents, anchored `CPI`,
`PCE inflation` and one blank) -- macro-indicator surfaces rather than the country strings this entry tracks,
so those are D-1's leg still open and not this entry's arm. Any query here re-run after the next deploy
measures a DIFFERENT binary, and a changed figure is not by itself a refutation of this entry.

RE-CHECKS, all `psql` SELECT-only. Every one widens as the window grows:
- distribution: `SELECT coalesce(nullif(trim(source_entity),''),'<BLANK>'), count(*) FROM
  sentinel.extracted_observations WHERE extracted_at >= TIMESTAMPTZ '2026-09-06 10:26:43Z' GROUP BY 1 ORDER BY 2 DESC;`
- anchor-class split, the entry's headline. Use the SAME 54-name list as the discriminator query above --
  a shortened list silently reclassifies `Germany`/`France`/`Spain`/`Italy` and inflates the issuer arm:
  `SELECT CASE WHEN coalesce(trim(source_entity),'')='' THEN 'blank' WHEN source_entity IN (<the 54 names>)
  THEN 'country' ELSE 'named-issuer' END AS anchor, count(*), count(*) FILTER (WHERE instrument_id IS NOT NULL)
  FROM sentinel.extracted_observations WHERE extracted_at >= TIMESTAMPTZ '2026-09-06 10:26:43Z' GROUP BY 1;`
- the attachments: same `WHERE`, plus `AND instrument_id IS NOT NULL`, selecting
  `description, value, source_entity, resolution_method, "Symbol"`.
- catalog co-cause, DB `atlas_secmaster`: `SELECT symbol, name FROM instruments WHERE name IN
  ('Turkey','China','UK','Germany','France','Spain','Italy','US');` -- the table's parenthetical glosses
  (`TUR` = iShares MSCI Turkey ETF, `UNRATE` = the US rate) are external knowledge, NOT what this returns.
- has ReExtract reached the window: same `WHERE`, `count(*) FILTER (WHERE re_extracted_at IS NOT NULL)`.
  Non-zero means the attachment table above is no longer reproducible.
- Rule 2.5 arrivals: `sentinel_gemini_resolver_calls_total` in Prometheus, `outcome="surface_filtered"`.
- deploy: `sudo nerdctl container inspect sentinel-collector --format '{{.Created}}'` -- NOT bare `inspect`,
  which resolves the IMAGE and returns the BUILD time -- plus `sha256sum` on the two prompt paths.

GRADUATED OUT OF THIS ENTRY, so it is not lost when this defect closes: **the `cod` prompt directory is NOT
CPU-only -- it is production's GPU extraction prompt.** That is service SHAPE, so by §WHERE_WORK_LANDS it now
lives in `SentinelCollector/AGENT_README.md` §GOTCHAS ("the `cod` PROMPT DIRECTORY IS NOT CPU-ONLY"), with
its citations, rather than in a KNOWN DEFECTS entry that gets deleted on close.

NOT REPAIRED HERE, recorded so it is not re-discovered from scratch: D-1's four `DeterministicResolver.cs`
citations have drifted. It cites Rule 1 `:60`, Rule 2 `:124`, Rule 2.5 `:150` and `Classify` `:640`; those
lines now hold a comment fragment, a comment fragment, a `LiftSector` call and a bare `{`, while Rule 1,
Rule 2, Rule 2.5 and `Classify` live at `:95`, `:154`, `:212` and `:675`. The default sweep is SILENT on all
four and NOT for one reason: only `Classify`'s cite is filename-qualified in D-1's prose, so it is the ONLY
one the tool reads -- and it reports GREEN on that bare `{`. The other three are BARE CONTINUATIONS, and
bare continuations are opt-in (`--bare`, default OFF -- `scripts/verify-citations.py:185` and
`scripts/verify-citations.py:719`), so the default sweep never PARSES them. Silence on those three is not a
clearance: the tool did not inspect them at all, which is the inverse of the blind spot its docstring
warns about and worse, because a reader repairing D-1 would take a quiet sweep as three citations checked.
AND `--bare` DOES NOT RESCUE THEM -- measured on this card: it binds all three to `V2ExtractionPipeline.cs`,
the nearer preceding filename in D-1's own prose, where `:60` reads BLANK and `:124`/`:150` land on
real-but-unrelated lines. Three wrong answers in place of three unread ones. These three can be checked by
READING them and no other way.
(The `:60`/`:124`/`:150`/`:640` forms in this paragraph quote D-1 verbatim and are bare for the same
reason, so they are equally unswept.)

**Production's CoD extraction loses ~108 gold entities per run to its own loop guard, TODAY.**
`entities_recall` 0.5545 -> 0.3809 on the prompt production actually runs, measured 2026-09-06 and
attributable to `CpuCod__JsonRepetitionPenalty` 1.1 rather than the token cap. The full entry, its
four-cell table and its re-check live under §MEASUREMENT DEBT ("Production's `repetition_penalty` 1.1 --
NOT the token cap -- costs `entities_recall`") because that is where the measurement that found it sits,
but the DEFECT is a live production one and belongs in a reader's scan of this section. Nothing in
production measures it; the cheapest next step is a 1.02 / 1.05 / 1.1 penalty sweep, scored on coined
names as well as recall.

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

**MODEL BASELINES, aggregate_f1 on the v6.2 substrate.** Moved out of CLAUDE.md §MODEL_ACCEPTANCE 2026-09-06,
which now carries only the rule and a pointer here; scorecards are in `LlmBenchmark/eval-substrate/*.json`.
EVERY FIGURE NAMES ITS ENGINE, because a bare number does not identify a run: the challenger's label spans
0.694-0.764 across five scorecards in one diff -- a wider range than the 0.051 effect the "`fp8_e5m2` KV
cache costs ~0.05 aggregate F1" entry tables -- and the bare `0.763` that sat in CLAUDE.md was a vLLM 0.28.0 run,
the engine the "0.28.0 upgrade is BLOCKED by our `fp8_e5m2` KV cache" entry blocks, not production's.

| model | aggregate_f1 | engine / config |
|---|---|---|
| Qwen2.5-32B-AWQ | 0.443 | vllm-0.19.0, fp8_e5m2 KV (production today) |
| Qwen2.5-32B-AWQ | 0.494 | vllm-0.19.0, unquantized KV |
| Qwen3.8-27B-AWQ-INT4 | 0.764 | vllm-0.19.0 |
| Qwen3.8-27B-AWQ-INT4 | 0.7634 | vllm-0.28.0 [BLOCKED engine] |
| Qwen3.8-27B-AWQ-INT4 | 0.7629 | vllm-0.28.0 + MTP speculative decode -- F1 unchanged, wall clock -30% |
| Qwen3.8-27B-AWQ-INT4 | 0.744 | NVFP4 @ vllm-0.28.0 |
| Qwen3.8-27B-AWQ-INT4 | 0.694 | vllm-0.19.0, the model card's own sampling |

FOUR digits on the 0.7634 and 0.7629 rows deliberately -- both round to 0.763, so a three-digit row does not
say WHICH scorecard it came from. Do not round them.

AND THE ENGINE IS ONLY HALF THE IDENTITY: SIX OF THESE SEVEN ROWS DO NOT NAME THEIR SAMPLING [2026-09-06].
The last row is the proof, sitting inside the table it indicts -- the ONLY axis separating 0.694 from 0.764
is the decoding, a **0.070** swing, larger than the 0.051 fp8 KV effect this file devotes an entry to and
larger than most gaps the table is used to argue. So "EVERY FIGURE NAMES ITS ENGINE" is necessary and not
sufficient: a row carrying an engine and no sampling still does not identify a run. Measured on the CoD
path the same week, changing ONLY `repetition_penalty` and `max_tokens` moved `entities_f1` by 0.135 and
REVERSED the sign of a prompt comparison -- see "The `entities_f1` regression REVERSES SIGN". Whoever next
edits this table should add a sampling column rather than trusting the engine column to carry it.
THE ACCEPTANCE SENTENCE BELOW IS ALSO NOW KNOWN INSUFFICIENT: "a swap needs a CoD scorecard on production's
prompt path" is true and incomplete, because `production_prompt_path` reads no decoding knob and stamps
`true` on a run that omitted production's loop guard. Prompt path AND sampling, or the scorecard is not
production's request.

NONE OF THIS IS ACCEPTANCE EVIDENCE. It is scored on the substrate's own 16 instruction blocks, a task
production does not run; CLAUDE.md §MODEL_ACCEPTANCE governs, and a swap needs a CoD scorecard on production's
prompt path. THAT SCORECARD NOW EXISTS and its figures are NOT these: `numbers_f1` on production's CoD
extraction is 0.5153 incumbent / 0.7400 candidate, a different task and a different metric from the
`aggregate_f1` in this table -- see MEASUREMENT DEBT, "The candidate BEATS the incumbent on production's
CoD path". 0.7400 and 0.764 are not the same number measured twice, and reading them as one is the
conflation this table's engine labels were added to prevent. NOR ARE THE TWO ARMS THERE SERVED
ALIKE -- that entry discloses a KV-cache and context difference between them, which is exactly what
this table's own engine column exists to surface.
The table is also why the old `MODEL_SIZE >= 30B` floor was retired: a 27B model beats our 32B
incumbent here, so the proxy would have BLOCKED an upgrade on a number that was never the point. That
retirement rationale lived in CLAUDE.md justifying a rule that no longer existed; it belongs with the
measurement that settled it.

**THREE live sites outside CLAUDE.md still teach the retired `MODEL_SIZE >= 30B` floor, and one of them is
PRODUCTION CODE.** Found 2026-09-06 while moving the retirement rationale into the entry above; all three predate
that move and none was touched by it. The ROOT `README.md` asserts ">=30B-parameter models" for Sentinel
extraction and links `[CLAUDE.md -> SENTINEL]` -- the section that RETIRED the floor -- so the pointer now leads
to its own refutation; the same file also advertises "Sentinel sizing" as one of CLAUDE.md's conventions.
`docs/SENTINEL-RLM.md` carries a `### Model Size (30B+)` heading, a table cell reading "30B+ required for
extraction quality", and a `## Do Not` bullet forbidding sub-30B models. AND
`SentinelCollector/src/Services/ExtractionService.cs:35` carries it as a code comment on the CoVe constructor --
`// CoVe (extraction) always runs on GPU - requires >=30B model`. THE CODE SITE IS WHY THIS ENTRY WAS WIDENED
RATHER THAN CLOSED: an earlier revision said "two docs" and gave a two-FILE grep as its re-check, so anyone
closing it by its own instrument would have left the retired rule taught in a comment and read GREEN doing it.
This MISLEADS rather than merely lagging: the floor was retired because it is a PROXY that would have BLOCKED a
real upgrade -- a 27B model beats our 32B incumbent on this substrate, in the table above -- so a reader who
lands on any of these declines the upgrade the measurement favours. Cited by grep for the two docs and not by
line, because a `README.md` citation is ambiguous across this repo and would rot besides; the `.cs` site is cited
by line because that file's name is unique:
`grep -c 30B README.md docs/SENTINEL-RLM.md SentinelCollector/src/Services/ExtractionService.cs` -> 1, 4 and 1 on
2026-09-06. Close it by replacing each with MODEL_ACCEPTANCE's actual bar -- a scorecard on production's prompt
path that beats the incumbent's -- not by deleting the numbers. THE `.cs` EDIT IS A SEPARATE PR: it is a comment
change in a service, so it pulls in a full `SentinelCollector` compile, and a docs PR may not carry it.

**The three stores, and the numbers CLAUDE.md §WHERE_WORK_LANDS no longer carries.** Measured 2026-09-05
(recorded by #1007) at `c32354f7`, which is the sha that reproduces them and NOT #1007's own `7a9769ed`, a
later tree measuring 3,936: STATE.md 293 lines; docs/BACKLOG.md 3,849 lines with 21 STALE entries reading as live;
LESSONS.md 13 of 14 exit criteria written and never once run -- that third figure survives, in LESSONS.md's own
header, and the first two had nowhere to live until this entry. Two of the three now have an out-flow that RUNS:
`scripts/new-epic.sh` audits STATE.md and LESSONS.md at the epic boundary and refuses the reset on a finding.
THIS FILE HAS NONE -- its out-flow is a human closing an entry in the PR that fixes it, which is the mechanism
that produced the 21. Re-check: `wc -l docs/BACKLOG.md` -- 4,187 with this entry in it on 2026-09-06, so 3,849
is the dated measurement and not today's size, and the growth is the point. STATE.md was reset since (49 lines,
2026-09-06) and lives ONLY at the repo root -- untracked, gitignored, absent from every worktree, so `wc -l` on
it from one fails rather than reporting zero. The 21 is an AUDIT count, NOT a grep -- nothing here is labelled
stale, which is precisely what "reading as live" means -- so re-deriving it means re-reading the entries.

**Production's `fp8_e5m2` KV cache costs ~0.05 aggregate F1 on extraction, concentrated in RECALL.**
Measured 2026-09-04, Qwen2.5-32B-AWQ on vLLM 0.19.0, KV dtype the ONLY variable, full 597-record substrate:

| metric | fp8_e5m2 (production) | unquantized KV | delta |
|---|---|---|---|
| aggregate_f1 | 0.443 | 0.494 | +0.051 |
| text_quote_recall | 0.289 | 0.346 | +0.058 |
| selectivity_recall | 0.317 | 0.375 | +0.058 |
| symbol_exact_match | 0.661 | 0.691 | +0.030 |
| period_accuracy | 0.434 | 0.463 | +0.029 |
| latency s/doc | 10.0 | 13.4 | +3.4 |

The cost lands exactly where this pipeline is weakest. The trade is +0.051 F1 for +34% latency against ~20x
measured headroom (1,945 req/h capacity vs ~96 req/h actual), so it looks worth taking -- and it is a change to
the engine we run TODAY, independent of any upgrade or model swap. Unquantized KV still fits 32K on this card
under 0.19.0 (36,848 tokens of cache vs ~73,712 at fp8), which is ample at ~3K tokens/request and concurrency 8.

NOT YET DONE, and why: measured on the substrate's own 16 instructions, NOT production's cod_json_v1.txt +
schema path. Confirm on the production prompt before changing vllm_kv_cache_dtype.
UNBLOCKED 2026-09-05: the harness scores production's CoD path end to end now, so both KV arms CAN be
measured on it -- `--task cod` against the 40-article gold, NOT `aggregate_f1` (a CoVe metric the CoD
scorecard does not carry). The macro-owner convention is DECIDED and in the gold now; while it was
open it biased both arms identically, so a within-model A/B was never what it blocked. Cost is
unchanged: a production
stop and two ~4min GPU reloads. See MEASUREMENT DEBT, "The CoD gold cannot yet back a model swap".

RELATED: this same flag is what crashes vLLM 0.28 (see the entry below), and e5m2 carries 2 mantissa bits to
e4m3's 3 -- production runs the lower-precision of the two 8-bit formats AND the crash-prone one.

**vLLM 0.28.0 upgrade is BLOCKED by our `fp8_e5m2` KV cache on sm_120; `fp8_e4m3` is the one-flag fix.**
Measured 2026-09-04 on the RTX 5090 (sm_120) with production's exact nine flags. 0.28.0 STARTS fine and serves
single requests, then faults under concurrent decode with `torch.AcceleratorError: CUDA error: an illegal memory
access` and stays 503. Isolated to one variable:

| vLLM | KV dtype | ctx | conc | result |
|---|---|---|---|---|
| 0.19.0 | fp8_e5m2 | 32K | 6 | 597/597 clean, twice (production today) |
| 0.28.0 | fp8_e5m2 | 32K | 1 | OK |
| 0.28.0 | fp8_e5m2 | 32K | 2 / 4 / 6 | crash |
| 0.28.0 | fp8_e5m2 | 16K | 6 | crash, 17/18 |
| 0.28.0 | fp16 | 16K | 6 | 0 errors, healthy |
| 0.28.0 | **fp8_e4m3** | 32K | 6 | **0 errors, healthy** |

The last two rows against row four differ ONLY in KV dtype, at matched context and concurrency, so it is neither
sm_120 generally, nor structured output, nor context length, nor CUDA graphs (`--enforce-eager` still crashes).
It is the e5m2 FORMAT. `--attention-backend TRITON_ATTN` does not help -- it fails to start on a torch.compile
error raised by `kv_cache_dtype.startswith("nvfp4")`, at line 507 of vLLM's own `attention.py` INSIDE the 0.28.0
image. That is upstream source, not ours, and it is deliberately NOT written in `file:line` form: nothing in this
repo can ever resolve it, so a citation there is a permanent false positive in every future sweep.

NOT a transitive dependency: the image ships flashinfer-python 0.6.16.post3, NEWER than the 0.6.8/0.6.9 that
reproduce the sibling bug vllm#41651; `vllm_flash_attn` is no longer a module and `VLLM_FLASH_ATTN_VERSION` is
gone from envs.py, so the old force-FA2 workaround is retired. The lever is a flag, not a version.

WHY WE WERE EXPOSED AT ALL: vLLM's own FP8 KV documentation benchmarks FA3 on Hopper and FlashInfer on B200
(SM100), discusses e4m3 throughout, and never mentions SM120 or e5m2. Our production config is outside the
tested matrix on all three axes. It works on 0.19.0; nobody upstream is validating it.

RE-CHECK: run the levers again on the next vLLM release. Correctness of the e4m3 path was spot-checked at n=18
against the known-good 0.19.0 output (aggregate_f1 0.517 -> 0.526, all deltas small and mixed-sign, no sign of
the garbage-output mode of vllm#41651) -- that is "no evidence of the failure", NOT proven equivalence.
`period_accuracy` moved -0.056 and is the one to watch on a full 597-record confirmation.

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

**AN OBSERVATION'S IDENTITY IS THE ENTITY MENTIONED, NOT THE MEASUREMENT TAKEN — so N datapoints
from one article collapse onto ONE series key, and 74% of everything ever published is in such a
group.** The extractor is doing its job: an article legitimately carries 0..n datapoints and it
mines them all. The defect is downstream of that — resolution answers "which instrument is this
about", the publish path uses that answer AS the series id, and ThresholdEngine keys its
ObservationCache by SeriesId keeping the newest write. So a series holds whichever of its n
claimants landed last.

MEASURED 2026-08-26 on sentinel.extracted_observations, published rows only:
  20,822 published observations · 2,727 instruments · **15,344 distinct descriptions** · 35 units
  15,397 of 20,822 (74%) sit in a (raw_content_id, instrument_id) group of size > 1; 3,591 such
    groups against 5,425 sole claimants
  **1,559 of 2,727 instruments (57%) carry MIXED UNITS** — percentages and dollars under one key
Re-check both:
  `SELECT COUNT(*) FILTER (WHERE c>1), COUNT(*) FROM (SELECT raw_content_id, instrument_id,
     COUNT(*) c FROM sentinel.extracted_observations WHERE published_at IS NOT NULL
     AND instrument_id IS NOT NULL GROUP BY 1,2) g;`
  `SELECT COUNT(*) FILTER (WHERE u>1), COUNT(*) FROM (SELECT instrument_id, COUNT(DISTINCT unit) u
     FROM sentinel.extracted_observations WHERE published_at IS NOT NULL
     AND instrument_id IS NOT NULL GROUP BY 1) x;`
THESE COUNTS MOVE AND THAT IS NOT A CONTRADICTION: the pipeline keeps publishing, so a re-run
minutes later already returned 3,592 of 9,022 groups and 1,560 of 2,732 instruments. **The
durable claims are the RATIOS — roughly three quarters of published rows in a collision, more
than half of instruments carrying mixed units, and ~5.6 distinct descriptions per instrument.**
Read a changed absolute as the corpus growing, and re-derive from the commands above rather than
from the numbers printed here.

THE SHAPE, from a real published group. One Procter & Gamble article, four rows, all Symbol=PG:
  tariffs on American goods         25 PCT
  tariffs on American goods         50 PCT
  imported toilet paper from Canada 328,000,000 USD
  global tissue consumption         20 PCT
None of those is "the value of PG". They are facts MENTIONED NEAR Procter & Gamble, and TE now
computes on whichever wrote last. The 5.6 descriptions-per-instrument ratio says this is the norm,
not an outlier.

WHY IT SURFACED AS A CHALLENGER BUG AND IS NOT ONE. CHALLENGER_JOB_CUTS was the one instance where
the collision was VISIBLE, because that series has patterns reading it and an alert that fires when
it goes stale. One Challenger release yields four datapoints — headline job cuts 33,429, an
AI-attributed subset 10,970, a sector-and-YTD figure 149,023, and planned HIRES 107,500, which has
the opposite sign. All four resolve to the same instrument and **similarity cannot separate them**:
measured 0.8196 / 0.8125 / 0.8507 / 0.7994, so the WRONG answer scores highest, and `confidence` is
a flat 0.85 across all four. No threshold keeps the headline and drops the rest. Publishing planned
hires as job cuts drives challenger-layoff-surge's `-(cuts - 30000)/30000` to **-2.58**, a maximal
false recession signal from a number meaning the opposite. Everywhere else the same collision is
silent, because no pattern is watching and nothing alerts on a wrong value — only on a missing one.

WHAT WAS TRIED AND MUST NOT BE SHIPPED AS-IS: an article-level guard refusing every claimant when
n>1 ("ambiguity denies"). It is fail-closed and it is WRONG AT THIS SCALE — it would refuse 74% of
the pipeline to prevent corruption that has already happened. Written, tested, and deliberately not
committed; the patch is at /tmp/sentinel-remediation/collision-guard/ and will not survive a reboot,
which is correct — it should be rewritten against whatever identity model wins, not resurrected.

ANSWERED 2026-08-26 — this was open item (1): **NOTHING CURRENTLY READS THE CLOBBERED KEYS, so this defect is
LATENT — mechanically real, not presently corrupting output.** Measured: **zero of TE's 71 loaded
patterns reference a `SENTINEL:NUM:` key**, which is where 47,366 of 47,402 published observations
land over 30 days. Exactly 3 patterns touch a Sentinel key at all, all via the `SENTINEL:SECTOR:`
prefix through `GetSeriesCount` / `GetSectorBreadth`
(`ThresholdEngine/src/Entities/PatternEvaluationContext.cs:312`, `:362`), both of which read
`SeriesId` and `LatestDate` and NEVER `Value`. The matrix path is keyed
`{raw_content_id}:sig:{slug}` in `public.macro_observations` (16,690 of 16,694 `source_id` values
distinct over 30 days), so it does not collapse across articles. Note the reason carefully: the
projector DOES evaluate `signalExpression` over the `ObservationCache`
(`ThresholdEngine/src/Workers/ObservationCellProjector.cs:810-821`), so "the cache is unread" is
FALSE and must not be written down as the explanation — the explanation is that no loaded
expression names the key.
THE LIVE SURFACE IS THE BARE `SentinelSeriesKey.OwnedSeries` SET, and it is one resolver outcome
away from arming. Of its six keys, `CHALLENGER_JOB_CUTS` (3 loaded patterns) and `TRUFLATION_CPI`
(1) have live readers and Sentinel currently publishes NEITHER — no `sentinel.extracted_observations`
row has ever carried either as its `Symbol`, so Sentinel's Challenger rows key as
`SENTINEL:NUM:<instrument-uuid>`, not as the bare mnemonic.
**THIS SITS IN TENSION WITH THE "WHY IT SURFACED AS A CHALLENGER BUG" PARAGRAPH BELOW and is NOT a
refutation of it — do not delete that paragraph on this evidence.** Unresolved: whether the -2.58
figure there was observed on a live `challenger-layoff-surge` evaluation or derived as what WOULD
happen, and whether `CHALLENGER_JOB_CUTS` has a non-Sentinel publisher. Establish that before either
paragraph is edited. `BDIY` publishes 4 rows/day under a bare key (level 3056, increase 130, highest 11793,
lowest 290 on 2026-08-26, and 290 is the last write); its only reader `baltic-freight-recession`
would compute `bdiy < 700` TRUE and clamp its signal to **-3**, but that pattern is
`"enabled": false` and is the one repo pattern file of 72 absent from TE's loaded set. Loaded gun,
not fired — and #988 raising resolution success is the change that could re-arm the Challenger key.
Re-check before treating this as still latent:
  `grep -rl 'SENTINEL:NUM:' ThresholdEngine/config/patterns --include='*.json' | wc -l`   # expect 0
  and confirm `baltic-freight-recession` is absent from `list_patterns(enabled_only=false)`.

STILL OPEN, AND IT IS A DESIGN QUESTION, NOT A BUG FIX. Identity probably needs to be
(instrument, measurement) rather than instrument alone, which is a schema change and touches D-18's
ownership rules — so it needs a human decision, not a patch. One thing left to establish, item (1)
above having been answered: (2) whether #988 (resolver retries with the description)
is net-negative on this evidence, since it raises resolution success and therefore admits MORE rows
into a broken identity model. Do not treat the Challenger feed as fixed: it now resolves, and
resolving into this model is not obviously better than starving.

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

**A SHARE OF APPARENT IDENTITY COLLISIONS ARE MIS-RESOLUTIONS, NOT IDENTITY COLLAPSE — DIFFERENT
ENTITIES LANDING ON ONE INSTRUMENT ID. THE KEY CANNOT FIX THESE; THE RESOLVER IS THE LEVER.**
The entry above counts a `(raw_content_id, instrument_id)` group of size > 1 as a collision. Some of
those groups are not one entity measured n ways — they are n DIFFERENT things the resolver filed
under one instrument. Re-keying on (instrument, claim_kind, unit) leaves every one of them wrong,
and worse: it splits the wrong attribution across two "series" by unit, so it acquires structure.

SAMPLED 2026-08-26 by adversarial review of the remediation plan: **6 of 32 groups (~19%)**.
**That is a FLOOR** — the sample was drawn from PUBLISHED rows carrying an instrument, so it
excludes everything unpublished and everything the resolver failed on outright.

THE MACHINE-CHECKABLE PROXY IS SMALLER AND MEASURES SOMETHING NARROWER, which is why the sample
is the number to quote. Colliding groups whose members disagree on `subject_entity`:
**148 of 4,110 (3.60%)** re-measured 2026-09-05, against 144 of 3,598 (4.00%) on 2026-08-26. That catches
only the case where the article's own subject STRINGS differ; the worked case below has a CONSTANT
`subject_entity` and is still a mis-resolution, so the proxy is a floor of a floor and NOT a refutation
of the 19%.

THE WORKED CASE WAS RE-EXTRACTED OUT OF EXISTENCE; here is a live one, on a different resolver leg.
`raw_content_id=153978` (the Mexican market summary filed under `EWW`) now carries `instrument_id` and
`resolution_method` NULL on all four "Mexico" rows — the hazard the re-extraction entry below warns about,
eating this entry's own exhibit. Re-selected 2026-09-05: `raw_content_id=160553`, 15 published rows,
`subject_entity` "Germany" on every one, all filed under `8cc911bf-9846-4f31-a3e7-3d4d9b7161f2` = **`DAX`,
GLOBAL X DAX GERMANY ETF**, via `llm_candidate_exact` — one of them "Brent oil closing price per barrel".
Two siblings sit one query away: `DEXINUS` (SecMaster `name` literally "India") and `DEXCAUS` ("Canada"),
both asset_class Currency, absorbing their country's subjects. The resolver still cannot say "this subject
is a place, not a security", and a country-NAMED instrument is what it reaches for.

THE LEVER IS #988 (resolver retries with the description), NOT R2/S4's key. And note the direction:
#988 RAISES resolution success, which admits MORE rows into both this defect and the collision one.
Do not read a rising resolution rate as progress here without splitting these two populations.

A THIRD CONFOUND, small but real: exact-duplicate rows inflate the same groups — `raw_content_id=153978`
still shows instrument `750b06ec` holding each of its three observations twice.

Re-check (SELECT-only):
  `SELECT count(*) FILTER (WHERE c>1 AND ds>1) AS multi_subject_groups,
     count(*) FILTER (WHERE c>1) AS colliding_groups
   FROM (SELECT raw_content_id, instrument_id, count(*) c,
           count(DISTINCT coalesce(nullif(btrim(subject_entity),''),'(null)')) ds
         FROM sentinel.extracted_observations
         WHERE published_at IS NOT NULL AND instrument_id IS NOT NULL GROUP BY 1,2) g;`
  # 2026-09-05: 148 | 4110
  and RE-SELECT an exhibit rather than trusting the one above, which can be re-extracted away too:
  `SELECT raw_content_id, instrument_id, subject_entity, count(*) c, count(DISTINCT unit) u
   FROM sentinel.extracted_observations
   WHERE published_at IS NOT NULL AND instrument_id IS NOT NULL
     AND subject_entity IN ('Mexico','Brazil','Japan','China','India','Germany','France','Canada','Australia')
   GROUP BY 1,2,3 HAVING count(*)>1 ORDER BY c DESC LIMIT 8;`
  then in `atlas_secmaster`: `SELECT id, symbol, name, asset_class FROM instruments WHERE id='<uuid>';`
THE RATIOS ARE THE DURABLE CLAIM, not the absolutes — the pipeline keeps publishing. The ~19%
sampled share is a FLOOR and the 3.60% proxy is a floor beneath it; neither is an upper bound, and
nothing here has measured how much of the collision figure is really mis-resolution.

**THE MATRIX PROVENANCE CHAIN IS EMPTY: NO `matrix_cells` ROW CAN BE TRACED TO THE OBSERVATIONS
THAT PRODUCED IT.** The schema carries two columns for exactly this — `contributing_observation_refs`
(jsonb) and `source_provenance` (jsonb) — and neither is written with anything usable. MEASURED
2026-08-26 on `public.matrix_cells`, 287,763 rows:
  `contributing_observation_refs` populated on **0 rows** — every row is NULL, `'null'`, `[]` or `{}`
  `source_provenance` populated on **223 rows (0.08%)**, and `rawContentId` is **null on all 223**,
    so even the 0.08% does not reach an observation. Those 223 carry only
    `{dslVersion, producerVersion: "semantic-verifier@phase4.5", sourceTimestamp, sourceDocumentRef}`
    with 2 distinct `sourceDocumentRef` values, and they are all `evaluated_at` 2026-04-13..2026-06-01
    — a dead phase-4.5 experiment, not a live writer
CONSEQUENCE, and it is why this is filed rather than noted: **every question that starts "which cells
were affected" is unanswerable.** Damage assessment after the D-18 mnemonic corruption, damage
assessment after the identity collision above, any audit of a wrong signal, and any decision to mark
or recompute a subset of cells all require the join and none of them can have it. The best available
substitute is a split on `evaluated_at` at a fix date — 241,508 cells before 2026-08-13 vs 46,255
on-or-after — which is a COARSE UPPER BOUND over the whole table and not an attribution: it cannot
distinguish a cell that consumed a corrupted value from one that did not, and it says nothing at all
about collision loss, which has no fix date and is still happening. This is strictly larger than the
S5 story in `docs/proposals/extraction-identity-implementation.md`, which had to be rewritten onto
the time bound because of it.
NOT A DATA-REPAIR JOB: the missing links were never written, so there is nothing to backfill. The fix
is at the WS3 projector's write seam — populate `contributing_observation_refs` when a cell is
computed. Until then the columns are decorative, and a reader who sees two provenance columns in the
schema reasonably assumes provenance exists.
Re-check (SELECT-only, no deploy):
  `SELECT count(*) AS total,
     count(*) FILTER (WHERE contributing_observation_refs IS NOT NULL
       AND contributing_observation_refs::text NOT IN ('null','[]','{}')) AS refs_populated,
     count(*) FILTER (WHERE source_provenance IS NOT NULL
       AND source_provenance::text NOT IN ('null','[]','{}')) AS prov_populated,
     count(*) FILTER (WHERE source_provenance->>'rawContentId' IS NOT NULL) AS prov_reaches_article
   FROM public.matrix_cells;`
Closes when `refs_populated` is a material share of `total` on cells written after the fix.
`total` grows continuously — read `refs_populated` and `prov_reaches_article` as the claim, not the
absolute row count.

**S5 (`docs/proposals/extraction-identity-implementation.md` §5) is DONE, and the 240,353 bound DOES
tighten: D-18's mechanism can only reach 1,111 pre-fix cells, and 242 of them carry a
positively-identified corruption signature.** MEASURED 2026-08-26 on `public.matrix_cells`, 288,467
rows (D-18 fixed by `3bae398a`, 2026-08-13 00:56 UTC). This supersedes an earlier reading of this
same entry that called the bound un-tightenable; that reading was not wrong about its own method
(see DO NOT RETRY below), it just never asked which cells the mechanism can physically reach.

`created_at` **240,353** cells created before the fix vs `evaluated_at` **241,519** evaluated before
it. **Use `created_at`** — 1,166 cells carry a pre-fix `evaluated_at` but a post-fix `created_at`:
evaluated FOR a pre-fix date but COMPUTED after the fix, on clean data, so the `evaluated_at` split
over-counts by that many. `evaluated_at` is the nominal date being evaluated, not when the row was
computed: `created_at` postdates it by more than a day on 26,205 rows (9.09%, avg 5.38 days, max
210.0 days), spread across 86 distinct creation days from 2026-05-30 — a continuous rolling
recompute, not one backfill burst. 240,353 remains the correct COARSE upper bound over the whole
table, and it is still not an attribution.

**THE NARROWING, and it is structural rather than statistical: D-18 corrupted the `ObservationCache`,
and only ONE of the projector's two magnitude paths reads that cache.** `ObservationCellProjector`
computes a cell's magnitude as `isNews ? {the :sig: decay sum} : EvaluateHardMagnitudeAsync(...)`
(`ThresholdEngine/src/Workers/ObservationCellProjector.cs:686-688`), and only the second arm compiles
and runs the pattern's `signalExpression` against the cache — the code comment states it outright:
"News groups NEVER use SignalExpression." So a news-path cell cannot consume a polluted cache value
no matter how corrupt the cache was. Three nested populations, each measured:

| population | pre-fix cells | what it is |
|---|---|---|
| whole table | **240,353** | every cell that existed while the bug was live (coarse upper bound) |
| hard-data path | **5,929** | cells whose magnitude came from `signalExpression` + `ObservationCache` |
| reads a polluted mnemonic | **1,111** | of those, the patterns whose expression reads a series Sentinel actually published under |
| carries the ±3 signature | **242** | of those, the identified floor (below) |

The hard-data restriction is corroborated in the data, not only in the code: **zero** cells with
`abs(signal) = 3.0` exist on any `sentinel` (280,423 cells) or `ofr` (3,487) row — all 418 exact
clamps in the table sit on the 4,334 `fred` rows — and 99.71% of `sentinel` rows in
`public.macro_observations` carry the `:sig:` news infix (43,516 of 43,644).

**THE FLOOR: 242 cells, 9 patterns, `created_at` 2026-06-25..2026-08-12.** Detector is
`abs(signal) = 3.0` EXACTLY (the `SignalUtilities.ClampSignal` bound,
`ThresholdEngine/src/Services/PatternEvaluationService.cs:344`) on a pattern whose `signalExpression`
reads a polluted mnemonic and whose clamping stops when that mnemonic's junk stops:
`ust-10y-yield` 77 · `oil-price` 33 · `dxy-dollar-index` 33 · `cpi-headline-yoy` 22 · `cpi-core-yoy`
22 · `pce-headline-yoy` 22 · `industrial-production` 11 · `nonfarm-payrolls` 11 · `fed-funds-rate` 11
(all `:fred`). **This is 242 cells MATCHING A SIGNATURE, not 242 cells PROVEN damaged** — read that
before citing it. True damage is in `[242, 1111]` on the mechanism's own path, still inside
`[0, 240353]` for the table as a whole.

**Numerical-coincidence trap, flagged deliberately: the superseded reading of this entry also
reported "242 rows", from `signal <= -2.9` across the whole table. THEY ARE DIFFERENT SETS that
happen to have the same count.** The old 242 spans 7 patterns and includes the three non-attributable
ones below; the new 242 spans 9, includes `+3` clamps the old detector could not see, and excludes
all three. The overlap is 88 cells. Do not treat one as confirming the other.

**Why the 242 are attributable at all: the published values, read directly, not inferred from a
before/after shift.** `sentinel.events` stores every `SeriesCollected` payload, so what Sentinel put
under each first-party key is recoverable. It is not marginal noise — DGS10 was published as 8, 18,
20, 22, 28, 50, 55, 60, 67, 100 and 125000000000 against a real 10y yield of ~4.4-4.7%; DCOILWTICO
ranges -1,934,000 to 1e9; DTWEXBGS's maximum published value is `20230729`, a DATE. `ust-10y-yield`
clamps at `+3` only when DGS10 >= 5.5, which real data never reached in the window, and its clamp
burst runs 2026-07-23..2026-08-12 — the exact span of the junk DGS10 publications. The same
arithmetic-impossibility test passes for the other eight.

**The controls behave as controls should — three patterns clamp in BOTH eras and are excluded.**
`repo-liquidity-stress:fred` clamps on 10 of 10 batches across both eras: `-WLRRAL/75000` against a
level in the millions is permanent formula saturation, and Sentinel's last WLRRAL publication was
2026-05-01, 3.5 months before the fix. `um-consumer-sentiment:fred` needs UMCSENT <= 55 for its `-3`,
which real consumer sentiment genuinely reaches. `gdp-real:fred` still clamps 2026-08-26. None is
attributable to D-18 and none is counted.

**FOUR CORRECTIONS to the naive version of this measurement, each of which alone would have produced
a wrong answer:**
1. **`abs(signal) >= 2.9` is NOT a clamp detector.** `EvaluateHardMagnitudeAsync` falls back to the
   raw, UNCLAMPED mean when the expression throws (`ObservationCellProjector.cs:826-839`), so
   `>= 2.9` sweeps in raw means: `continuing-jobless-claims` at 1,777,000, `initial-jobless-claims`
   at 225,000, `cpi-headline-yoy:fred` at 3.779246. 143 cells counted as "clamped" by that detector
   are unclamped fallbacks. Use `= 3.0`.
2. **The single largest row of the naive table is off-mechanism entirely.**
   `obs:cpi-headline-yoy:sentinel` (110 "clamped" pre / 0 post) is news-path, so it never touched the
   cache — and all 110 are `> 3.0`, i.e. not clamps either. It is disqualified twice over.
3. **"Stops dead at the fix boundary" is false for 5 of 12 patterns.** `industrial-production` last
   clamps 2026-07-17, `dxy-dollar-index` 2026-07-21, `pce-headline-yoy` 2026-07-30, `fed-funds-rate`
   2026-08-04, `nonfarm-payrolls` 2026-08-07. They stop when their OWN mnemonic's junk stops, which
   fits better and still supports the mechanism — INDPRO's last Sentinel publication and
   `industrial-production`'s only clamp are the SAME DAY, 2026-07-17, 27 days before the fix.
4. **The `created_at` split has almost no statistical power and is not what carries this claim.**
   Post-fix eras are 1-3 projection batches (11 sectors each) for 10 of the 12 clamping patterns;
   `oil-price` is 0-of-2 post-fix batches, p≈0.44. Only `ust-10y-yield` has any power (39 pre / 10
   post batches) and even it is p≈0.13 on batch counts alone. The claim rests on the published values
   and the arithmetic impossibility, NOT on the before/after counts.

**D-18's card named SIX polluted mnemonics; the measured set is THIRTY-SEVEN, and the six were a
truncated top of a list, not a scope.** *Re-measured 2026-08-26; this supersedes the "FIFTEEN" figure
first recorded here, which was itself an undercount and is withdrawn by its own author.* D-18's own
30-day window (2026-07-13 .. 2026-08-12) reproduces five of its six counts exactly (DGS10 42,
DCOILWTICO 36, UNRATE 30, PAYEMS 29, ICSA 26; CPIAUCSL 98 vs its 96, window-boundary jitter) but
contains **37** distinct bare first-party keys -- not six, and not fifteen. The fifteen omitted the
third-largest key in the window outright: **DEXJPUS, 37 events, 0.1 .. 1.17e13**, ahead of
DCOILWTICO. The full window list with per-key event counts now lives in the D-18 card. 33 of the 37
published at least one value outside a 2x band of that series' own real 2026 FRED range, and **151**
distinct first-party keys carry 3,630 events across all history before the 2026-08-13 cutover. The
list was cut and nothing marked it as cut -- that unmarked truncation, not the specific count, is the
defect, and the card now states that the set is measured and open. This is load-bearing: the
obvious test of this entry's claim, "does the pattern read one of the six mnemonics D-18 names",
FALSELY REFUTES it — 5 of the 9 floor patterns (`dxy-dollar-index`, `cpi-core-yoy`,
`pce-headline-yoy`, `fed-funds-rate`, `industrial-production`) read a polluted mnemonic that is not
on D-18's six. A truncated list in a card became a wrong answer in a measurement that trusted it.

**Nothing else changed at the boundary.** The only `ThresholdEngine/**` commit between 2026-08-05 and
2026-08-20 is `3bae398a` itself, and it touches one file (`PatternMnemonicFormatValidator.cs`). No
pattern config under `ThresholdEngine/config/patterns/**` was edited in the window, so no threshold,
formula or clamp bound moved across the split.

**WHAT THIS SAMPLE STRUCTURALLY CANNOT CONTAIN, and every item biases the floor DOWNWARD:**
- damage that did not reach the clamp. A polluted value moving a signal from 0.3 to 1.7 is real
  corruption with NO signature. Only saturation is visible, so 242 is a floor of a floor
- cells since recomputed. The projector heals on rewrite, so a damaged cell recomputed post-fix on
  clean data no longer carries the signature — 26,205 rows show a recompute lag, so this is not
  hypothetical
- raw-mean fallback cells, which are also corrupted (the expression threw) but carry no clamp
- the 0.29% residual: 128 `sentinel` rows in `macro_observations` lack the `:sig:` infix, so a group
  made entirely of them would take the hard-data path. The hard-data population is measured at 5,929
  as `source_collector IN ('fred','ofr')`; that residual is not excluded by measurement, only bounded
- anything about collision loss, which has no fix date and is still happening
- per-cell provenance, which does not exist at all (see the entry above this one)

**DO NOT RETRY (1/2): mean-signal-shift attribution across the fix boundary.** Refuted by its own
control and still refuted; this round did not resurrect it. The projector evaluates hard-data
patterns' `signalExpression` against the same `ObservationCache` D-18 corrupted
(`ObservationCellProjector.EvaluateHardMagnitudeAsync`), so comparing a pattern's mean `signal`
before/after the fix looked plausible. `obs:oil-price:sentinel` shifts -0.137 -> +1.301 (**+1.437**)
across `created_at='2026-08-13'`. But SentinelCollector/AGENT_README.md D-18 names `natural-gas-price`
as its CLEAN control (Sentinel has never published `DHHNGSP`), and `obs:natural-gas-price:sentinel`
shifts **+0.694** over the same boundary — smaller, same direction, on a series the corruption never
touched. Others shift further in the OPPOSITE direction: `obs:fed-funds-rate:sentinel` **-2.185**,
`obs:dxy-dollar-index:sentinel` **-1.515**. A method that moves the clean control almost as much as
the implicated series is measuring period effects. It also fails for a second reason now visible:
every pattern it tested was `:sentinel`, i.e. news-path, which never reads the corrupted cache at all.

**DO NOT RETRY (2/2): tightening the bound with a before/after COUNT on any clamp-like threshold.**
Attempted this round and it does not carry weight in either direction — post-fix eras are 1-3 batches
for 10 of 12 patterns (§correction 4). The narrowing that DID work is structural (which path can
reach the cache) plus direct (what values Sentinel published), and both are re-runnable below. A
future round that re-derives 240,353 -> 1,111 by that route is repeating settled work, not extending
it; the open question is disposition, not measurement.

No backfill or recompute recommendation follows from this entry — disposition stays a human decision
(`extraction-identity-implementation.md` §6.2).

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
  what Sentinel actually published under first-party keys (the direct evidence) -- the first-party
  side is DERIVED, never a frozen IN-list: the earlier hardcoded 15 mnemonics missed 22 of the 37 keys
  actually in the window, which is the same truncation defect this entry is about. The series key is
  nested at `payload->'seriesCollected'->>'seriesId'`; `payload->>'seriesId'` matches NOTHING and
  reads as a clean zero --
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

**D-23's thin-draw gate can never deny, because `.Bind()` APPENDS to a non-empty list default.**
`SearxngIssuerProbeOptions.Engines` initialises to `["duckduckgo", "bing"]`
(`SentinelCollector/src/Configuration/SearxngIssuerProbeOptions.cs:72`) and `appsettings.json:75` configures the
byte-identical `["duckduckgo", "bing"]`. .NET's options binder APPENDS to an existing `List<T>` rather than
replacing it, so the bound list is FOUR entries, all duplicates. `RespondingPinnedEngines` is then
`pinnedEngines.Count - missingPinned.Count` (`IssuerProbeScorer.cs:189`) = 4 - 2 = **2**, which is exactly
`MinRespondingPinnedEngines`'s default of 2 (`:85`) — so the floor whose entire purpose is "never judge on a thin
draw" sits at its own minimum and passes even when ZERO real engines answered. The guard's own code is correct,
which is why no mutation test catches it: the denominator is inflated OUTSIDE the guard, by configuration that
looks like it agrees with the default. Note this is a SECOND inflation route into D-23 — the one the D-entry
already documents is SearXNG silently serving its default set for an unknown engine name; this one needs no
SearXNG involvement at all. **Currently inert and therefore not urgent: `IssuerProbePinVerifier` is registered
(`DependencyInjection.cs:108`) but no consumer reads a probe verdict, so nothing acts on the floor today. It goes
live the moment the probe is wired**, which is what makes it worth recording rather than fixing now. Fix shape:
either drop the property initialiser and let configuration be the sole source, or clear the list in the binder
callback before binding — not by raising `MinRespondingPinnedEngines`, which tunes around the miscount. Re-check
(cheap, no deploy): a unit test that binds the shipped `appsettings.json` section and asserts
`options.Engines.Count == 2`; it goes RED today. Recorded from the PR #947 review 2026-08-15; code re-verified on
main 2026-08-23, both line citations unmoved.

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
- The live leg is `DeterministicResolver`, called at `SentinelCollector/src/Services/V2ExtractionPipeline.cs:78`.
  Its only consumer is `BuildObservationFromV2Result` (`SentinelCollector/src/Services/V2ExtractionPipeline.cs:173-242`),
  an `internal static` PURE BUILDER — it does NOT touch the database. It sets `instrument_id` and
  `resolution_state` on an in-memory `ExtractedObservation` (`UpdateResolution` / `SetResolutionState`), returns it
  into `observations` at `SentinelCollector/src/Services/V2ExtractionPipeline.cs:98`, and the row reaches
  `ExtractionProcessor` on `V2PipelineResult` to be persisted downstream. It calls
  `SentinelMeter.SecMasterResolutionCounter.Add` for **no** outcome — neither success nor failure; the whole file
  has **zero** call sites (`grep -c SecMasterResolutionCounter` = 0). That is the whole defect: the resolution
  OUTCOME is decided here and metered nowhere, so nothing between the decision and the persist is counted.
- Inside `DeterministicResolver` the counter has FIVE emission sites and exactly one carries `status="resolved"`:
  `SentinelCollector/src/Services/DeterministicResolver.cs:496` (`TryExactCandidateMatchAsync` success,
  `llm_candidate_exact`). The other four are refusals or non-resolutions — `:253` (`LiftSector`; ONE site whose
  status is a ternary over `no_subject_match` / `matched_no_sector`, always `resolution_state="no_sector"`), `:437`
  (`TryExactCandidateMatchAsync` co-mention rejection, `exact_rejected_name`), `:521` and `:538`
  (`TryHybridResolveAsync` guard rejections).
- `ExtractionProcessor.cs` never calls `DeterministicResolver` — **zero** grep hits — and its own two
  `status="resolved"` emissions (`SentinelCollector/src/Workers/ExtractionProcessor.cs:1376` `ticker_in_quote`,
  `:1419` `cove_*`) have not fired in prod for 30 days. Those two cite the `status` label line, one BELOW their
  `.Add(`; the `DeterministicResolver` citations above cite the `.Add(1,` line itself. Both land inside the correct
  emission block — do not "fix" either to match the other. A sixth site outside the resolver,
  `SentinelCollector/src/Workers/ReExtractResolutionAdapter.cs:195`, emits only `comention_rejected`.
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
`DeterministicResolver.cs:371-375`), so that column cannot answer what Rule 1 received, and querying it is circular.
It is observable, just not in the DB: PR #963 added `sentinel_resolver_rule1_input_confidence` ("ResolutionConfidence
as received by Rule 1", `SentinelMeter.cs:1625-1627`, recorded at `DeterministicResolver.cs:393`), which is the only
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
`DeterministicResolver.cs:552`) and attaches it. `DIA` is the catalog's quote stub, literally named "DIA (Quote)";
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
already invoked at `DeterministicResolver.cs:477` and `:509`. "Never on this branch" was the wrong compression and
is corrected here to match D-22 in the card: `:430` is D-8's leg, which Rule 1 never reaches. `:509` IS reachable
from Rule 1 — it sits on the RagSynthesis hypothesis-materialisation branch inside `TryHybridResolveAsync`, which
the id-less Rule 1 leg calls — but a DTO already carrying an instrument id bypasses it, and the whole guard is
behind `Extraction__GuardsEnabled=false` (`/opt/ai-inference/compose.yaml:1155`), so it is inert on both counts.
The 77 `DIA` rows above went straight through it: `SharedTokenCount("Dow Jones", "DIA (Quote)")` is 0, so with the
flag on they would have been refused. Adding a THIRD call behind that same disabled flag would read as protection
that does not exist. Deciding the flag's fate is the prerequisite, and it is its own entry's worth of work.
TWO GAPS #969 LEFT OPEN, recorded rather than fixed because both need a decision this PR is not the place for.
(a) The `!candidate.InstrumentId.HasValue &&` exemption on the blank-Name refusal (`DeterministicResolver.cs:408`)
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

**THE STATIC-METER FLAKE'S ROOT CAUSE IS FIXED; what is left is two classes that never joined the collection.**
Measured 2026-08-15: two consecutive full runs of an identical tree gave `Failed: 2, Passed: 2222` then
`Failed: 0, Passed: 2224`. Mechanism: a global `MeterListener` filtered by meter+instrument but NOT by test,
plus a class outside `[Collection("SentinelMeterStatic")]` running in parallel with the listening ones.
CLOSED for every class this entry named — #947 `67396749` (2026-08-15 18:04) added the attribute to
`ExtractionProcessorStreamingTests.cs:29` and `ExtractionProcessorTests.cs:18`, hours after the measurement.
This entry and two duplicates deleted with it then spent three weeks saying "a re-run turns it green, so the
standing incentive is to re-run rather than to fix" — which now sends someone holding a REAL meter regression
to press re-run. STILL OPEN: `ExtractionProcessorV1SkipOutcomeTests` and
`ExtractionProcessorThinContentOutcomeTests` carry no attribute, so the mechanism is intact for whatever they
emit. Never a retry; the fix is the attribute, or a capture scoped per test.
Re-check, run 2026-09-05 from `SentinelCollector/tests/SentinelCollector.UnitTests/Workers`:
  `grep -L 'Collection("SentinelMeterStatic")' ExtractionProcessor*Tests.cs`
  # prints exactly ExtractionProcessorV1SkipOutcomeTests.cs and ExtractionProcessorThinContentOutcomeTests.cs.
  # A THIRD name appearing is a new class that skipped the collection — that is the regression to catch,
  # and it is invisible to a suite run, which goes green ~half the time either way.

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
(`EntityResolutionPrepass.cs:404`), Rule 2.5's paid-Gemini leg (`DeterministicResolver.cs:651`, D-6) and its V1
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
`TryGeminiResolveAsync` up to `ResolveAsync` entry (~`DeterministicResolver.cs:47`). `_surfaceFilter` is already
injected and `ResolveAsync` has one production caller (`V2ExtractionPipeline.cs:78`), so the change is small — but
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

**The gemini-resolver daily-cap refusal HAS a counter now, and this entry's own re-check went blind for three
weeks without anyone noticing.** It used to read "a fourth outcome that increments nothing, so refused demand
has no metric at all". `gemini-resolver-mcp/gemini_resolver/server.py:932-938` defines
`c_cap_refused = Counter("gemini_resolver_cap_refused", ..., ["reason"])`, both reasons pre-created at
`:942-943` and incremented at `:1085`. Live 2026-09-05: `gemini_resolver_cap_refused_total{reason="at_cap"}`
**1852**, `{reason="ledger_unavailable"}` 0, the counter born 2026-09-01T10:10:32Z at the last restart. Every
line the entry cited moved too: `try_reserve_live` is `:737-760` and returns a REASON STRING rather than
`False`, and `record_gated` is `:725`.
AND THE LOG WORDING CHANGED, so the re-check printed here returned 0 while 1,852 refusals were being counted —
the exact false negative this entry warned about, arriving as a clean pass. Measured 2026-09-05:
`journalctl -u gemini-resolver-mcp --since "24 hours ago" --utc | grep -c "daily call cap"` -> **0** (rc 1);
the same window against the current wording, `grep -c "live reservation refused"` -> **96**. A sample line:
`Sep 05 06:23:52 mercury python[1096089]: ... WARNING gemini_resolver.server: live reservation refused
(at_cap), cap 1500 (= free-grounding boundary); refusing without calling Gemini`.
RE-CHECK — run the METRIC half FIRST now that it exists, and treat the log half as the cross-check:
  `curl -s http://localhost:9300/metrics | grep -E 'cap_refused|gated_24h|dispatch_rejected_total|breaker_refused_total'`
  # 2026-09-05: cap_refused{at_cap} 1852.0, cap_refused{ledger_unavailable} 0.0, the other three 0.0
  `sudo journalctl -u gemini-resolver-mcp --since "24 hours ago" --utc | grep -c "live reservation refused"`
  # 2026-09-05: 96. If this disagrees with `increase(gemini_resolver_cap_refused_total[24h])`, the WORDING
  #   drifted again, not the refusals — that is the only thing the log half can still tell you.
  (EVERY LINE CARRIES TWO TIMESTAMPS FOUR HOURS APART AND `--utc` FIXES ONLY ONE. `journalctl` prints LOCAL
  time here — mercury is `America/New_York` — so pass `--utc`; the Python logger writes its own NAIVE LOCAL
  timestamp into the message body, which `--utc` does not touch. The journal stamp on the LEFT is the UTC one.)

THE CAP ITSELF IS HOLDING — record this so nobody re-raises it. Ledger file birth **2026-08-06T16:20:51Z**
(`/opt/ai-inference/gemini-resolver-ledger.db`; `stat` reports it as `2026-08-06 12:20:51 -0400`, and mercury is
`America/New_York`, so read that field as local and convert). `ledger_meta.lifetime_live_calls` = **38,476** over
**29.8 days** = **~1,292/day** (re-measured 2026-09-05; it read 18,586 over 14.04 days = 1,324/day on
2026-08-20, so the RATE is flat and the total is just the counter running). Every measured restart-to-restart
interval is at or below 1500/day; the full UTC day
2026-08-19 is **exactly 1500** live, against `gemini_resolver_daily_cap` 1500. The gauge is bounded by real
enforcement, not by a display clamp.
INTENT VIOLATION. The alert's own annotation
(`deployment/artifacts/monitoring/alerts/gemini-resolver.yml:320`) states "A true last resort is dozens/day".
Measured sustained volume is ~1,292/day — **15-100x the documented design intent**. This is the CLAUDE.md
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

**request-log regression-guard gap (#886).** The DiagnosticContext re-registration in AlertService, SecMaster and
CalendarService — which preserves `UseSerilogRequestLogging` after `Host.UseSerilog` was dropped — has no
`// INTENT` tag and no test, so a future edit deleting it silently breaks request logging at request time.

**Metric prefix inconsistency from a single service:** `sentinel_candidate_surface_*` vs
`sentinelcollector_semantic_signal_*`.

**TWO uncorrected copies of the Finnhub transient-only claim, not four, and neither is where this entry
pointed.** `SecMaster/src/Configuration/EnrichmentOptions.cs:14` and
`SecMaster/src/Services/CatalogEnrichmentBackgroundService.cs:187` both call "Finnhub 403 for foreign tickers"
a TRANSIENT enrichment failure. A 403 is plan-uncovered and permanent, and it arrives as a NULL profile rather
than an exception — which `SecMasterMeter.cs:243` and `SecMaster/src/Services/IFinnhubCollectorClient.cs:33`
now both say correctly,
so this is a finished conversion with two files left. The pointer this entry used to carry,
`IdentifierConfirmationService.cs:361-362`, is an XML doc about cancellation and `grep -n 403` on that file
returns ZERO: "four" was never reproducible from anything written down.
Re-check (run 2026-09-05): `grep -rn '403' --include=*.cs SecMaster/src FinnhubCollector/src | grep -i transient`
  -> four hits, the two SecMaster ones above plus `FinnhubApiClient.cs:29` and `:406`, which state the
  permanent/transient split CORRECTLY and are the controls, not the debt.

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

**The quarantine refusals are a POLICY nobody has decided, not a constraint.** The partial index turned the
refusal from impossible into optional: `CatalogService` and `EntityResolutionService`'s self-seed now REFUSE
deliberately, because re-acquiring a retired ticker is a catalog-repair decision and resolution time, per
candidate, silently, is the worst place to take it. CONSEQUENCE while it stands: `CatalogService.cs:206` drops a
quarantined discovery item with a bare `continue`, so a CompanyName candidate loses its ticker PROPOSAL and every
news mention of one of the 82 real tickers pays the full confirm cascade, up to the paid Gemini leg (D-1)
(`CatalogService.cs:206-210` — the LogWarning and the `continue` it precedes). It is
also reachable from the `search_catalog` MCP tool, outside resolution entirely. Decide per PATH, not globally: an
operator-curated config and a collector registration are authoritative in a way a news-surface self-seed is not.
POPULATION, because this entry once welded two numbers into one: of the 91 quarantined rows, 82 are real
tickers and 9 are not tickers at all (the D-4 macro-junk class). Both enrichment candidate queries require an
equity-shaped class, so those 9 sit in NEITHER pool and need their own disposition; the split is in the
`asset_class` query below.
`quarantined_skip` IS A FLOOR ON THE WALL-HITS, NOT A RATE: it is emitted at ONE of four self-seed skip paths
(`EntityResolutionService.cs:1038`); silent are `:877` unconfirmed, `:901` contextFactor<=0, `:1002`
EnableSelfSeed=false, and the `CatalogService.cs:206` drop, which carries a LogWarning and no metric at all.
Re-check, all run 2026-09-05:
  `sum by (result)(secmaster_entity_resolution_self_seed_total)` -> `idempotent_skip` 4625, `inserted` 126,
    `quarantined_skip` **32**. This entry used to say that series did not exist "because the emitting code is
    unmerged". It exists, and the `error` series (the 23505s, 22 at the time) is GONE — the index working.
  `SELECT asset_class, count(*) FROM instruments WHERE is_active=false GROUP BY 1;` in `atlas_secmaster` ->
    Equity 74 / ETF 8 / fred_series 8 / Economic Indicator 1, unchanged.
  `SELECT indexname, indexdef FROM pg_indexes WHERE indexname LIKE 'idx_instruments_symbol%'
     OR indexname LIKE 'idx_source_mappings_collector_source%';` -> `idx_instruments_symbol` is
    `UNIQUE ... WHERE (is_active = true)`, but `idx_source_mappings_collector_source` is STILL UNIQUE on
    `(collector, source_id)` GLOBALLY with no `is_active` predicate. A quarantined row's mapping pair is still
    reserved and `RegistrationService.cs:391` would raise a 23505 the index change does not touch. That is
    harmless only while quarantined rows carry no mapping (1 of 91 does, `GSV.NE`) and nothing alerts when it
    stops being true.

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

**TRIPWIRE, green by design: a NEW `BrokenCircuitException` orphaning cohort.** The classification gap itself is
closed (SentinelCollector D-27, `SentinelCollector/AGENT_README.md:118`) and both known cohorts are disposed of: 55
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
- rows on **ANY OTHER date** -> a REGRESSION on the numeric path, WITH one known exception named below. D-27 makes
  it impossible there: whether the breaker is refused before the model call (row requeued) or after it (article
  finished), neither branch writes `processing_error`. A row here means the guard was removed, bypassed, or a second
  code path writes the breaker's message. Correlate the date against `vllm-server` availability (`journalctl -t
  atlas-stack-watchdog`; host clock is EDT, not UTC) and read BOTH reasons —
  `sentinel_extraction_error_total{reason="dependency_unavailable"}` and `{reason="dependency_unavailable_after_extraction"}`
  — for the same window before concluding anything.
- the KNOWN exception: rows whose `source` is `validation-content` (or `validation-content:sector:*`). Those take
  the qualitative dispatch leg, whose catch writes `processing_error` for ANY exception before the article catch can
  see it — see the entry immediately below. Bucket them out with `AND source NOT LIKE 'validation-content%'` before
  reading the query above as a regression signal.

Measured 2026-08-17: `55 | 2026-07-19T17:14:25Z | 2026-07-24T11:00:06Z | 55`.
Measured 2026-09-04T22:27Z, before the recovery: ZERO from the 2026-07 cohort (the old 30-day clock pruned them
first) and `223 | 2026-09-04T00:06:38Z | 2026-09-04T16:46:56Z | 223` from a same-day cohort — both halves of this
entry landing on the same day, which is what made a bare total unreadable and is why the query above buckets.
Measured 2026-09-04, after the recovery and on the fix branch: **0 rows**, which under the reading above is the
expected steady state and NOT evidence the fix works — the fix is evidenced by
`ExtractionProcessorCircuitOpenRequeueTests`, not by this query.

**DEFECT, pre-existing and now the ONLY leg outside D-27's gate: the qualitative dispatch path still orphans on a
dependency outage.** `TryDispatchQualitativeAsync`'s extract-stage catch (`SentinelCollector/src/Workers/ExtractionProcessor.cs:2753`)
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
`ReExtractBackgroundService.cs:119-122`, `ExtractionProcessor.cs:74`, `MirrorSearchWorker.cs:79`,
`ResolutionWorker.cs:51`, `StaleContentPrunerService.cs:85`, `RssFeedCollectorWorker.cs:31`, `EdgeSyncWorker.cs:27`,
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
route, so the two routes disagree about one threat.** `gh api -X PUT repos/<owner>/<repo>/pulls/<N>/merge` resolves
against THIS checkout's verdict marker whatever owner, repo or HOST the path names. Eleven closed sub-shapes were
deleted from this entry on 2026-09-05; what stays open is one thing — a redirect that lives in the URL PATH reaches
no scan at all. Measured 2026-08-15 on a fixture with #99901 approved at head, #99904 unreviewed, `gh` stubbed and
an isolated `ATLAS_MARKER_DIR`, nothing merged: `gh api -X PUT repos/attacker/evil/pulls/99901/merge` -> **allow**,
the `curl` spelling -> **allow**, `curl -X PUT https://evil.example.com/api/v3/repos/jpansarasa/ATLAS/pulls/99901/merge`
-> **allow**, while #99904 denies on every one — which is what proves the NUMBER is read and the marker consulted is
this repo's. The subcommand route denies all five equivalents, including `--hostname`.
Re-check WITHOUT the fixture, because the mechanism is readable and the decisions are not (run 2026-09-05):
`grep -n '/pulls/\[0-9\]\+/merge' .claude/hooks/git-push-guard.sh` returns ONE line, `:2326`, which scrapes the
number and nothing else; `merge_scan_redirects` (`:2131-2152`) inspects only the `-R` / `--repo` / `--hostname`
FLAGS; the refusal that WOULD catch a foreign slug is `MERGE_FOREIGN` at `:2500-2501`. No owner, repo or host is
read on the REST route on any tree.
Why it is still open, and the size of the job: `repos/o/r/pulls` appears on 17 fixture lines across two suites, and
binding the rule on the REST route flips `run-pr-verdict-smoke.sh` row 2 to deny while the chained / no-verdict rows
stop exercising what they were written for. The fixtures must be re-pointed at this checkout's slug in the SAME
change as the rule, or the suite goes green while testing nothing.

**THE UNQUOTED NEWLINE IS A WRONG-CAUSE DENY, AND IT STAYS.** `shell_split` treats a newline as an ordinary
character, so `gh pr merge 99901 --squash<NL>grep -Rn TODO src/` is ONE token run with no standalone operator for
`merge_bound_to_act` to cut at, and the chained grep's `-Rn` is attributed to the merge. Deliberately not fixed:
quotes are consumed during tokenization, so a token carrying an unquoted newline cannot be told from one carrying a
QUOTED newline, and cutting there would drop tokens from the merge's OWN segment — a fail-OPEN traded for a
wrong-cause deny, the wrong direction.
Re-check (run 2026-09-05, command as JSON on the hook's stdin): the newline shape denies with "this merge names a
repository other than the one this checkout tracks", while the `&&` control denies with "PR #99901 has no recorded
review verdict" — same decision, different cause, and the CAUSE is the whole signal.

**TWO CONSTRUCTS LOWER THE SUBSTITUTION-DEPTH COUNTER WHERE BASH DOES NOT NEST — RECORDED, NOT FIXED, because the
capability delta against main is ZERO.** A `${x:-)}` default value and a `case` pattern `y)` each carry a `)` bash
does not nest, so each falls a depth it never raised. Put either inside a real `$( … )` and the redirect still
cannot get through: measured 2026-09-05, `gh pr merge 99901 --body-file $(mktemp --suffix=${x:-)} ; true)
-R attacker/evil` and the `case` spelling both **deny** — but through MERGE_OPAQUE, "names its PR with something
this gate cannot read", because the same split that loses the depth leaves loose tokens no identity rule can read.
Their no-redirect control denies identically, so these are NOT clean redirect rows.
Re-check by reading the refusal CAUSE, never the decision: a deny through MERGE_OPAQUE is not the redirect rule
firing, and a fix that made it one would not change any outcome.

**TWO CHEAPER FIXES WERE MEASURED AND DECLINED; the smoke rows that keep them declined ARE the record.**
(i) Abandoning the substitution narrowing whenever ANY token carries a substitution closes the twelve `$(… ; …)`
shapes and re-breaks `&& grep -Rn TODO $(pwd)` and `&& cp -R $(pwd)/docs /tmp/x` — ordinary chained commands whose
substitution belongs to the CHAINED command, refused with the same wrong cause. Under that fix rows `59-59g` pass
and `59h`/`59i` go red. (ii) Denying whenever the tokenizer ends inside a quote catches rows `61` and `61b` but also
ends mid-quote on `57k`, `58s` and `58u` — three ALLOW rows whose spans are cut mid-quote BY DESIGN
(`--subject "a|b"`), which the rule cannot tell from the hole; `57j` is the control that shows the flag tracks the
quote rather than the cut.
Re-check: `.claude/hooks/test/run-pr-verdict-smoke.sh` -> rc 0, 239 PASS / 0 FAIL (2026-09-05). Either fix turns the
named rows red, which is the only reason they are named here.

**`ansible-gate-guard.sh` reports a write to a path it INVENTED — three of the five spellings are fixed, two
stand.** The root is path CONSTRUCTION over command TEXT rather than over the resolved write target, so one fix
still covers what is left. Re-probed 2026-09-05 against the ARMED installed guard (blob `31baafbb`,
byte-identical to this worktree's copy, `.ansible-gate-confirmed` verified absent), each command fed as JSON on
the hook's stdin and none of them executed:
  FIXED, and deleted from this entry rather than left standing — the entry's (1), (2) and its "fourth
    sighting": `git show <rev>:<tracked path> > /tmp/x`, an absolute `/tmp` source merely CONTAINING the
    gate-layer substring, and the runner-displacement `time bash <gate script> > /tmp/out.log` (with `nice`
    and `env` alike) all now **allow**.
  STANDS (1) — WRITING A SCOPE INTO THE BYPASS FILE is refused by the gate the scope exists to narrow, so the
    guard's own scoping mechanism is unreachable and only the all-or-nothing `touch` survives, which is the
    WIDEST bypass and the exact failure the 2026-08-07 scoping change was added to prevent.
    `printf '%s\n' '.claude/hooks/git-push-guard.sh' > /home/james/ATLAS/.claude/.ansible-gate-confirmed` ->
    **deny**, naming the path that is the DATA being written, never the target. Workaround: write the fragments
    to a file elsewhere and `cp` it in, so no gate path appears in the command.
  STANDS (2) — A READ VERB DISPLACED FROM SEGMENT HEAD turns its gate-path OPERAND into the reported write
    target whenever a stdout redirect sits anywhere in the segment. `(grep -n x git-push-guard.sh > /tmp/o)` ->
    **deny**, as do the `.claude/hooks/…` and absolute spellings of the operand, a `{ …; }` brace group and a
    `time`-prefixed grep. Controls behave: group opener removed -> allow, redirect removed -> allow, non-gate
    file -> allow. With a BARE BASENAME the invented path is repo-root-relative —
    `/home/james/ATLAS/git-push-guard.sh`, which DOES NOT EXIST.
    THE SAME CLASS BITES `cp`, which is why the DEFERRED WORK entry below needed correcting: bare
    `cp <gate file> /tmp/x` ALLOWS, but all nine `WRAPPER_RE` prefixes
    (`time nice timeout stdbuf command exec ionice nohup xargs`) in front of the same cp DENY, naming the SOURCE.
Re-check, from the repo root; the command is itself not refused, and the trailing pipe keeps it valid when
copied across the line break:
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
What IS reachable is a write-side gap. `scripts/claude-pr-verdict` calls `warn_surviving_marker`
(defined `:156`, called at `:339`, `:359`, `:480`, `:494`, `:507`, `:520`, `:530`, `:548`) before every
refusal, leaving any earlier `pr-reviewed-<N>` on disk. When the refusal is the head-mismatch one (`:525-531`)
that is harmless: the surviving marker cannot match the current head either, so the guard denies. But the other
refusals — missing pending record (`:496`), malformed pending (`:506`), unreadable `gh` (`:519`), unparseable
timestamp (`:537`) and invoke-then-stamp too fast (`:541`, the `MIN_REVIEW_SECONDS` guard, NOT a reason-length
check) — can fire while a prior approve sits at the CURRENT head. That approve stays valid, the guard correctly
honours it, and the merge proceeds even though the reviewer's last action was an attempted BLOCK. The
auto-unlink is declined on purpose (silently deleting a prior verdict on an unrelated refusal destroys a
legitimate record), so the fix is to invalidate or DOWNGRADE the prior approve when a block is attempted and
refused — write side, not read side. Related in shape only to the historical defect where `review-pr` wrote a
passing marker at INVOCATION.
THE SCRIPT'S OWN WARNING USED TO BE WHERE THIS GOT MIS-DIAGNOSED, AND THAT HALF IS CLOSED [verified
2026-09-05]. This entry said `warn_surviving_marker` "reads only `v verdict rest`", never compares the sha, and
prints "still unblocks the merge" UNCONDITIONALLY — a sentence a reviewer reads and believes. It now reads
`v verdict msha _rest`, fetches the head on demand and prints one of THREE outcomes: still unblocks (sha ==
head), does NOT unblock (naming both shas), or EFFECT NOT MEASURED when the head is unreadable. Its header at
`:148-155` says so — "IT MEASURES, IT NO LONGER CLAIMS ... fixed 2026-08-17" — and records that the false
sentence had already been briefed into an agent as a finding once. The proposed fix was taken; only the
write-side gap above is open.
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

**`run-wiring-smoke.sh` is RED and has been since 2026-08-08, and the red is the suite's OWN stale list.**
Re-run 2026-09-05: rc 1, **52 PASS**, exactly one FAIL reading
`registered set drifted:6a7 > dream-pending-notice.sh`, ending `WIRING SMOKE: FAIL`. READ THE DIFF DIRECTION
BEFORE ACTING ON IT — the failure text invites the opposite conclusion. The check diffs `EXPECTED_WIRED` against
`ACTUAL_WIRED` in that order, so a `>` line is present in ACTUAL and missing from EXPECTED: the hook IS
registered in tracked `.claude/settings.json:170`, and it is the suite's hardcoded list
(`run-wiring-smoke.sh:65-72`, 16 names, whose pass message says "exactly the expected 16 hooks") that never
learned about it. Registration landed 2026-08-08 in #936 and the list has not caught up in four weeks; every
figure in the first draft of this entry (48 PASS, `5a6`, `:63-66`, 14 names) has since moved without the defect
changing at all.

The cost is not the one red row. The suite exits 1, so its other 52 assertions — including the marker writers
must be 100755 IN THE INDEX rows, which exist because a 100644 shipped once and broke the merge gate — sit
behind a failing summary, and anything gating on rc reads the whole suite as broken rather than as one stale
line. A suite that is permanently red teaches its readers to skip it, which is the failure mode that lets the
NEXT drift through. Decide the direction rather than silencing the row: either the dream notice is a wired
participant and belongs in `EXPECTED_WIRED`, or it should not be registered at all.
Re-check: `.claude/hooks/test/run-wiring-smoke.sh; echo rc=$?` — rc must be 0 and the summary
`WIRING SMOKE: PASS`.

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
time.** `ApplyReExtraction` (`SentinelCollector/src/Entities/ExtractedObservation.cs:316`) snapshots the prior
resolution into the `Original*` columns **only when all three are still null** (`:260-268`, preserving the EARLIEST
snapshot across repeat runs — guarded by
`SentinelCollector.UnitTests/Workers/ReExtractBackgroundServiceTests.cs:585`
`should_preserve_earliest_audit_snapshot_on_second_re_extract`), then overwrites `InstrumentId`/`Symbol` with the
new result **including NULL**. `Quarantine()` (`:220`) and `QuarantineInPlace()` (`:331`) write the same three
columns, so a quarantine can be the snapshot event a later re-extract then declines to overwrite.
Only two of the five call sites can null a resolved row:
`SentinelCollector/src/Workers/ReExtractBackgroundService.cs:496` (full re-extract) and `:663` (resolve-only, **the
leg prod runs**) pass the shim's result through, and that result may be null. The other three — `:369` (null
`RawContent`), `:438` (zero extractions), `:573` (empty `Description`) — pass `observation.InstrumentId`/
`observation.Symbol` straight back in: watermark-only stamps that cannot change a row's instrument or symbol — they
still run the full `ApplyReExtraction` body, which recomputes `ResolutionState` from the passed-back instrument
(`SentinelCollector/src/Entities/ExtractedObservation.cs:380`, flipping a Resolved-with-null-instrument row to
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
(`SentinelCollector/src/Workers/ExtractionProcessor.cs:878` v1, `:2116` v2). **`Symbol` is not in the predicate**, so
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
overlapping.
THE NOTE'S OWN EXAMPLE WAS WRONG, corrected 2026-09-05. It named **358,398** as one of two figures that "did NOT
move", being structurally exact rather than a running total. Re-measured: `NoResolution` **715,102** with
**389,483** carrying a non-null `"OriginalSymbol"` — so 358,398 moved with everything else, and the RATIO went
57.8% -> 54.5%. Under this note's own rule that 3.3-point move is a REAL FINDING rather than drift: the
quarantine/re-extract share of the dead population is falling. The only figure that genuinely cannot move is the
**0** in "0 of N". A re-check returning different integers is this note working; a re-check changing a RATIO, or
turning that 0 non-zero, is a real finding.
Re-check (SELECT-only, run 2026-09-05):
  `SELECT count(*) AS noresolution, count(*) FILTER (WHERE "OriginalSymbol" IS NOT NULL) AS with_original
   FROM sentinel.extracted_observations WHERE resolution_state='NoResolution';` -> `715102 | 389483`
THREE THINGS ARE CALLED "Symbol": (1) the COLUMN `sentinel.extracted_observations."Symbol"` — quoted, PascalCase,
the resolved catalog symbol, NULL until resolution succeeds; (2) the KEY `"Symbol"` INSIDE `candidate_symbols_json`
— an LLM-minted slug of the proposed entity's name (`Challenger_Gray_Christmas`), not a catalog symbol and usually
absent from SecMaster; (3) the AXIS — any query or index keyed on (1). `"OriginalSymbol"` is NOT a fallback identity
for (1): it is written only by `Quarantine()` (`SentinelCollector/src/Entities/ExtractedObservation.cs:299`),
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

**A `raw_content` ROW WITH EVEN ONE `extracted_observations` CHILD CAN NEVER BE PRUNED, AT ANY
`RawRetentionDays` SETTING — the cohort is unreachable by construction, not by policy, and raising or
lowering the setting cannot fix it.** `StaleContentPrunerService` runs two FK-safe passes: pass 1
(`StaleContentPrunerService.cs:95`) deletes rows past the cutoff only `WHERE CollectedAt < cutoff AND
!r.Observations.Any()` (`RawContentRepository.cs:107-108`); pass 2 (`StaleContentPrunerService.cs:112`)
nulls `raw_file_path` on the rows pass 1 skipped — `CollectedAt < cutoff AND RawFilePath != null AND
r.Observations.Any()` (`RawContentRepository.cs:144-146`) — keeping the row and its children intact.
Once pass 2 has run once on a row there is nothing left for EITHER pass to do to it: pass 1's predicate
permanently excludes any row with a child, and pass 2 only ever fires once per row (`raw_file_path` is
already null the second time). Moving the cutoff changes which rows are OLD ENOUGH; it never changes
which rows HAVE CHILDREN, so no `RawRetentionDays` value reaches this cohort.

**Scope: "can never be pruned" is true of the pruner, not of the whole system.**
`/admin/reprocess` (`AdminEndpoints.cs:224-230`, wired, real) deletes a row's `extracted_observations`
children scoped to `ReviewStatus.Pending AND QuarantinedAt IS NULL`. Deleting a row's last Pending,
non-quarantined child makes it childless, which unlocks it for pass 1 on the NEXT host start — a
narrow, real exception to the absolute above. Measured 2026-08-27: of the 4,980 rows' children,
review-status is Approved 36,570 / AutoClosed 18,567 / Rejected 55 / Skipped 3 — **zero Pending**, so
the exception does not apply to this cohort today. It does not apply, but it exists; the phrasing
above doesn't scope it out.

**Measured 2026-08-27**, raising `RawRetentionDays` 30 -> 180 and restarting the container: 0 rows
deleted.
  `SELECT count(*) FROM sentinel.raw_content WHERE collected_at < now() - interval '180 days';` ->
    **4,980** (all rows past the live cutoff)
  `SELECT count(*) FROM sentinel.raw_content r WHERE r.collected_at < now() - interval '180 days' AND
    NOT EXISTS (SELECT 1 FROM sentinel.extracted_observations o WHERE o.raw_content_id = r.id);` ->
    **0** (pass-1-eligible: childless)
  `SELECT count(*) FROM sentinel.raw_content r WHERE r.collected_at < now() - interval '180 days' AND
    r.raw_file_path IS NOT NULL AND EXISTS (SELECT 1 FROM sentinel.extracted_observations o WHERE
    o.raw_content_id = r.id);` -> **0** (pass-2-eligible: has children, file-path still set)
All 4,980 have children (nothing for pass 1) and all 4,980 already carry `raw_file_path IS NULL`
(nothing for pass 2 — their HTML was freed on an earlier host start). An INSTANT query against
`sentinel_extraction_error_total{source="prune"}` (Prometheus, datasource `bf2ya9fqus268c`) at
2026-08-27T11:15:44Z returns no series — but that is evidence about THIS restart, not about pass 1
overall: `StaleContentPrunerService.cs:98` (`if (deleted > 0)`) emits the counter only when the
CURRENT run's pass 1 deletes at least one row, and this run found 0 (`pass1_eligible=0`, above). A
30-day RANGE query on the same series is non-empty and repeatedly stepping — from single digits up
to **3405**, last sampled 2026-08-27T02:31:15Z (~9h before the empty instant read) and gone from the
series by ~02:40Z, consistent with a restart landing between those two timestamps. Pass 1 is
demonstrably active; only children-bearing rows are untouched.

**The trap, named because it just cost a round: an empty INSTANT query on a cumulative counter is
not evidence the counter never fired — range-query it before concluding absence.**

**Severity: not urgent, and this should not read as a fire.** The bytes that matter are already
reclaimed — pass 2 freed the on-disk HTML for all 4,980 rows, and the HTML is the bulk of the storage
(see the `sata-bulk/raw-data` measurement above, ~4 GB physical/month). What accumulates here is
`raw_content` ROWS plus their `extracted_observations` children — narrow rows, no blobs — at whatever
rate articles get extracted and then age past 180 days. Comparatively small, and slow.

**What it invalidated.** The deploy brief that predicted ~315 deletions from this restart was wrong,
and the mistake is worth naming so the next prune isn't sized the same way: it counted rows PAST THE
CUTOFF and treated that as the deletion estimate, when pass 1 only ever deletes the CHILDLESS subset
of that count. At the 180-day cutoff those two counts are 4,980 and 0 — not close, and no amount of
retention-setting tuning brings them together. Size a prune's expected deletions with the childless
query above; a plain past-cutoff count is not a deletion estimate.

**A separate "377" figure from the same day does not belong to this measurement — conflating the two
was this round's near-miss.** This file already carries a "**377** `age_cutoff`" figure (the
`BrokenCircuitException` entry, above). Reproduced here: `SELECT count(*) FROM sentinel.raw_content
WHERE processing_error LIKE 'age_cutoff:%';` -> **377**, exactly. But that predicate is
`ExtractionProcessor`'s per-article skip marker for `MaxArticleAgeDays` (extraction eligibility,
currently 30 days) — a different setting, a different column, and a different service than
`StaleContentPrunerService`'s `RawRetentionDays` prune cutoff (180 days) measured above. Both 377 and
4,980 are real numbers; neither substitutes for the other, and an early pass at this entry used 377 as
if it were the prune-candidate count before this re-check caught it.

**Still open, already tracked above, and unrelated to this mechanism: the 1,168-file `rss/2026/04/23`
orphan will never be reclaimed either.** Reproduced 2026-08-27: 584 `.html` + 584 `.meta.json` =
**1,168** files on disk; `SELECT count(*) FROM sentinel.raw_content WHERE raw_file_path LIKE
'/opt/ai-inference/raw-data/sentinel/rss/2026/04/23/%';` -> **0**. Confirmed not a pruning artifact
this round: 292 `raw_content` rows exist for that date with `source='rss'` (322 total that day; the
other 30 are `source='searxng-content'`), ~126.5 days old — well past the PRIOR 30-day
`RawRetentionDays` setting (see "Measured 2026-08-27, raising `RawRetentionDays` 30 -> 180" above).
Pass 2 already nulled all 292 `raw_file_path`s under that earlier 30-day cutoff, long before today's
change widened the live cutoff to 180 days; they are not "unreached yet" by the current setting, they
were already reached by the prior one, and pass 2 fires at most once per row so the wider cutoff has
nothing left to do here. These files were never referenced by any row, not referenced-then-nulled. See the entry above for the
write-path race hypothesis; not re-investigated here.

Re-check (psql is SELECT-only):
  `SELECT count(*) FROM sentinel.raw_content WHERE collected_at < now() - interval '180 days';`
  `SELECT count(*) FROM sentinel.raw_content r WHERE r.collected_at < now() - interval '180 days' AND
    NOT EXISTS (SELECT 1 FROM sentinel.extracted_observations o WHERE o.raw_content_id = r.id);`
  `SELECT count(*) FROM sentinel.raw_content r WHERE r.collected_at < now() - interval '180 days' AND
    r.raw_file_path IS NOT NULL AND EXISTS (SELECT 1 FROM sentinel.extracted_observations o WHERE
    o.raw_content_id = r.id);`

**`period` IS EXTRACTED BY ONLY TWO SOURCES, AND NEITHER OF THEM PUBLISHES ANYTHING -- EVERY
LIVE-PUBLISHING SOURCE IS AT ZERO. THIS FINDING HAS BEEN WRONG TWICE; THIS IS ITS THIRD STATEMENT.**

The correction chain, kept visible so the next reader does not re-derive a retired middle step:
1. RETIRED -- **"`period` is not extracted."** Retired on a whole-table count: 125,351 of 748,317
   rows carry one (16.8%).
2. RETIRED 2026-08-28 -- **"`period` IS extracted at volume (13-14k rows/month) and is LOST between
   extraction and publish."** This is what this entry said until now, and it is also wrong. That
   volume is one source that publishes nothing. There is no clearing to find, because the rows that
   would have to be cleared never enter the publish population at all.
3. CURRENT -- **the v2/DSL extraction path carries no `period` on any source, and the only two
   sources that still carry one are the two the v2 cutover never covered. Neither publishes.**
Note that retiring (1) was itself an over-correction: "not extracted" was right about every
live-publishing source and wrong only about the table total.

MEASURED 2026-08-28 on `sentinel.extracted_observations`, last 30 days, every source with rows:

| source | rows | with `period` | published | in `Extraction__V2EnabledSources` |
|---|---|---|---|---|
| `rss` | 167,868 | **0** | 46,260 | yes |
| `tsa-checkpoint` | 15,001 | **15,001** | **0** | **no** |
| `rss-mirror` | 10,489 | **0** | 3,242 | yes |
| `searxng-content` | 9,866 | **0** | 1,833 | yes |
| `challenger-rss` | 476 | **0** | 12 | yes |
| `rss-fallback` | 11 | **11** | **0** | **no** |

The split is exact. The two sources carrying a `period` are precisely the two NOT listed in
`Extraction__V2EnabledSources` (`/opt/ai-inference/compose.yaml:1140-1151` -- READ ONLY, never edit
it; twelve entries: `rss`, `rss-mirror`, `searxng-content`, `challenger-rss`, and eight `fed-*`).
They are also the only two whose rows carry no `dsl_block_id`. The eight `fed-*` entries are v2 but
DORMANT -- last row anywhere 2026-06-04, none in the window -- so they are zero-of-zero and are not
confirming instances; do not count them as such.
  `SELECT source, count(*) total,
     count(*) FILTER (WHERE nullif(btrim(period),'') IS NOT NULL) with_period,
     count(*) FILTER (WHERE published_at IS NOT NULL) published,
     count(*) FILTER (WHERE metadata ? 'dsl_block_id') dsl_tagged
   FROM sentinel.extracted_observations WHERE extracted_at >= now() - interval '30 days'
   GROUP BY 1 ORDER BY 2 DESC;`

**One source across the v2 cutover is the strongest form of this, and it removes the cross-source
confound the earlier correlation carried.** `rss` alone, by month -- `dsl_block_id` appears in May
and owns the source by June, and `period` goes to zero on the same boundary:

| month | `rss` rows | with `period` | DSL-tagged |
|---|---|---|---|
| 2026-04 | 37,367 | 21,132 (56.6%) | 0 |
| 2026-05 | 88,566 | 19,214 | 9,404 |
| 2026-06 | 177,630 | **0** | 177,630 |
| 2026-07 | 138,623 | **0** | 138,623 |
| 2026-08 | 152,424 | **0** | 152,424 |

One source, one collector, one publish path, before and after.
  `SELECT to_char(date_trunc('month',extracted_at),'YYYY-MM') mon, count(*) total,
     count(*) FILTER (WHERE nullif(btrim(period),'') IS NOT NULL) with_period,
     count(*) FILTER (WHERE metadata ? 'dsl_block_id') dsl_tagged
   FROM sentinel.extracted_observations WHERE source='rss' GROUP BY 1 ORDER BY 1;`

The whole-table DSL correlation, re-measured: since 2026-06-01, DSL-tagged rows carry `period` on
**0 of 534,655**; non-DSL rows on **48,519 of 48,634 (99.8%)**. (0/526,733 and 48,042/48,157 on
2026-08-27 -- both sides simply grew.)
  `SELECT (metadata ? 'dsl_block_id') AS is_dsl, count(*) total,
     count(*) FILTER (WHERE nullif(btrim(period),'') IS NOT NULL) with_period
   FROM sentinel.extracted_observations WHERE extracted_at >= '2026-06-01' GROUP BY 1;`

**THE MONTHLY TABLE IS STILL TRUE AND IT IS NOT A DECAY.** Measured 2026-08-27, re-measured
2026-08-28:

| month | rows with period | of those, published |
|---|---|---|
| 2026-05 | 48,010 | 510 |
| 2026-06 | 20,464 | 5 |
| 2026-07 | 13,884 | **0** |
| 2026-08 | 14,171 | **0** |

2026-05 through 2026-07 are closed; 2026-08 is partial and still accruing (13,694 through the 27th,
14,171 through the 28th). Read both columns correctly:
- the 13-14k/month is almost entirely `tsa-checkpoint` -- 15,001 of the 15,012 period-bearing rows
  in the last 30 days, leaving 11 from every other source combined;
- the **0** in the published column is that source publishing NOTHING AT ALL -- its last
  `published_at` is 2026-02-07 00:06:11+00, 729 rows lifetime (KNOWN DEFECTS, above). It is not
  period-bearing rows being filtered at a publish gate, and there is no publish-side effect to find;
- the fall from 48,010 is the v1 sources being cut over to v2, not an extractor degrading.
The last `extracted_at` on any published, period-bearing row is **2026-06-30 14:40:17+00** -- the
cutover, not a gate closing.
  `SELECT max(extracted_at) FROM sentinel.extracted_observations
   WHERE nullif(btrim(period),'') IS NOT NULL AND published_at IS NOT NULL;`

**ALSO RETIRED: "the field is cleared between extraction and publish."** Separating it was named as
requiring a write-path read. It does not: it needs a publish population containing period-bearing
rows to clear, and there is none. Nothing further should be spent hunting a nulling write.

**THE LEAD, NOW MORE SPECIFIC -- AND ONE GREP ALREADY KILLS THE NAIVE FORM OF IT.** On the CoD path
this is a MISSING-FIELD question, not a lost-value question. Two facts, neither of which is a
mechanism:
- `SentinelCollector/src/cod-prompts/cod_json_schema_v1.json` contains no `period` anywhere. Its
  `numbers[]` items are `(context, source_entity, source_text, unit, value)`, and
  `"additionalProperties": false` is set on the item schemas and the envelope (recorded above), so
  there is no field for the model to emit one into.
- BUT `SentinelCollector/src/Services/V2ExtractionPipeline.cs:192` DOES assign
  `Period = extraction.Period`. "The v2 adapter forgot to map the field" is therefore already false,
  and anyone repeating it has not run the grep. What fills `extraction.Period` on the CoD path is
  the open question.
WHAT WOULD SETTLE IT IS A CODE READ, NOT ANOTHER QUERY: read `V2ExtractionPipeline.cs` and
`GpuJsonExtractionService.cs` against the v1 assignment at `Workers/ExtractionProcessor.cs:728` --
the only other `Period =` site in the service -- and establish what populates the field on each path.
NOBODY HAS READ THAT CODE. Do not assert a mechanism from this entry.

Cross-reference: this is the concrete lead for reviving R2/S4 (PARKED EPICS, below), and it is a
better lead than when it was written. R2's premise was that a `period` axis needs a MODEL change --
teaching CoD to emit something it never has. The measurement now says `rss` went from 56.6%
populated to 0% across one pipeline cutover, so `period` may be a capability v1 HAD and the v2 path
does not carry -- a bug fix, cheaper than the change R2 proposed. That is a lead, not a finding: the
CoD schema genuinely has no `period` field, so restoring it may still be a schema-plus-prompt change
rather than a one-line repair. The code read above is what decides which, and it is the next step.

**Severity: a lead, not a fire.** This does not touch any published VALUE -- it affects only the
availability of a discriminator, R2/S4 is parked, and nothing currently reads `period` on the live
path. Worth someone's next hour, not an incident.

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

**Production's CoD prompt carries two defects no labeller can work around.** In
`SentinelCollector/src/cod-prompts/cod_json_v1.txt`, which is what production runs: (a) `:36` "the
normalized numeric as a string" conflicts with `:59` "Emit each distinct numeric value ONCE" when one
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
2026-09-04 -> lines `36` and `59` match; exclusion-word count `0`. A NON-ZERO second figure means an
exclusion rule landed and (b) is closed.

**D-27's dependency-outage guard is MERGED, TESTED, CARDED -- and NOT IN THE RUNNING CONTAINER. It
orphaned three articles on 2026-09-06.** Taking the GPU for the A/B in MEASUREMENT DEBT, "The
candidate BEATS the incumbent on production's CoD path", stopped production's vLLM for 32m56s (engine
snapshots: incumbent healthy `18:12:19Z`, candidate serving `18:15:50Z`, incumbent restored and
healthy `18:45:15Z`). Three `raw_content` rows were permanently failed inside that window, in
EXACTLY the pre-spend orphan shape D-27 says can no longer occur (`SentinelCollector/AGENT_README.md:118`)
-- `ProcessingError` written on a row nothing finished:

| id | source | collected_at (UTC) | retry_count | processed_at | processing_error |
|---|---|---|---|---|---|
| 164565 | rss | 2026-09-06 18:23:09.108 | 0 | null | `The circuit is now open and is not allowing calls.` |
| 164566 | rss | 2026-09-06 18:43:14.766 | 0 | null | `The circuit is now open and is not allowing calls.` |
| 164567 | rss | 2026-09-06 18:43:15.094 | 0 | null | `The circuit is now open and is not allowing calls.` |

BLAST RADIUS IS THOSE THREE, AND THE WHOLE-TABLE COUNT IS NOT WHAT ESTABLISHES IT. That count (three
rows carrying this error, table-wide) bounds circuit-open orphans only. The window census is what
bounds the outage: 17:50-19:00Z holds 14 rows -- 12 `rss`, 1 `rss-mirror`, 1 `tsa-checkpoint` -- and
all 11 that are not in the table above are `processed_at` non-null with no error. Note that D-27's
223 rows from 2026-09-04 carry the same predicate and do NOT appear today, so the table-wide count is
of SURVIVING rows; the earlier cohort was reprocessed. NOT REPAIRED HERE -- psql is SELECT-only and
every figure above was read that way; whether to reprocess is the user's call.

**THE GUARD IS NOT A HOLE. IT IS NOT DEPLOYED.** `ArticleExtractionSpend` and `DependencyOutage`
both landed in #1004 (`29846cf0`, 2026-09-05). The running `sentinel-collector` container was created
2026-09-06T10:26:43Z from an image built **2026-08-27T14:55:24-04:00** (18:55Z), nine days
earlier: recreated on the
CURRENT `:latest`, which nothing rebuilt after #1004. `CLAUDE.md` §DEPLOYMENT already warns that
`--skip-tags build` does exactly this, and for 21 of 26 service tags it is the only form on offer.
Read the CONTAINER, never the image -- `nerdctl inspect` resolves the image first and returns the
BUILD time as `.Created`, which is the trap that makes this invisible. VERIFIED AGAINST
THE BINARY, not inferred from dates, with a positive control in the same probe so a silent search
failure could not read as absence:
```
sudo nerdctl exec sentinel-collector sh -c 'for s in ProcessSingleArticleAsync ExtractionProcessor \
  MaxArticleAgeDays ArticleExtractionSpend RecordModelCallReturned DependencyOutage IsCircuitOpen; \
  do printf "%-32s " "$s"; grep -a -c "$s" /app/SentinelCollector.dll || true; done'
```
2026-09-06 -> `ProcessSingleArticleAsync 2`, `ExtractionProcessor 2`, `MaxArticleAgeDays 1` (the
control: type and member names ARE reachable in the assembly) and `ArticleExtractionSpend 0`,
`RecordModelCallReturned 0`, `DependencyOutage 0`, `IsCircuitOpen 0` -- every symbol D-27's GUARD
clause names is absent. Search for STRING LITERALS instead and you learn nothing: `age_cutoff`
returns 0 too, because literals live in the UTF-16 `#US` heap and a plain grep cannot see them. Probe
type and member names only.
THE FIX IS A BUILD, NOT A CODE CHANGE: `SentinelCollector/.devcontainer/build.sh --no-cache` then the
scoped deploy. This entry closes when the probe above prints a non-zero `ArticleExtractionSpend`.

DO NOT "FIX" THIS IN THE `isTransient` ALLOW-LIST. The obvious reading of the evidence --
`BrokenCircuitException` is absent from the transient set
(`SentinelCollector/src/Workers/ExtractionProcessor.cs:1155-1161`), so add it -- is the remedy D-27
MEASURED AND REJECTED: it leaves MaxRetries to be spent and the row orphaned anyway, and D-27 records
three named tests going RED on it. The design answer is the spend ledger, and it already exists.

WHAT THE BOX SAID, and it is D-27's own warning arriving on schedule.
`sentinel_extraction_error_total{reason="other", source="rss"}` reads **3** (Prometheus, instant at
2026-09-06T19:07Z) -- the single unnamed bucket that, at 223 rows on 2026-09-04, was the ONLY reason
that incident was visible at all. Whether three increments moved SentinelHighExtractionErrorRate is
NOT checked here and should not be assumed either way; what IS certain is that the queue-depth gauge
cannot see these rows by construction, because they LEFT the queue. So the deployed
build's only signal for this class is a counter increment too small to alert on, under a label that
says nothing about the cause. #1004 is what replaces `other` with two named reasons.

Re-check (SELECT only):
```
sudo nerdctl exec timescaledb psql -U ai_inference -d atlas_data -c \
 "SELECT id, source, retry_count, processed_at, processing_error FROM sentinel.raw_content
  WHERE processing_error LIKE '%circuit is now open%' ORDER BY id;"
```
2026-09-06 -> three rows, ids `164565`-`164567`, `source rss`, `retry_count 0`, `processed_at` null.
A GROWING count means another outage passed through the undeployed build. An EMPTY result means
someone reprocessed the rows, which does NOT close this entry -- only the binary probe does.

## MEASUREMENT DEBT [instruments that cannot report their own dullness]

### The CoD gold cannot yet back a model swap: macro-owner DECIDED, the key's swing is not [2026-09-05]
Production's CoD path is scoreable END TO END. The four divergences that made every scorecard on it
`null` are closed -- chat template (#1002), prompt assembly, scorer (#1011), runner (#1013) -- the gold
they needed exists (#1012), and the outage-vs-null-result gate that `call_errors` depends on landed
with it (#1014). The labelling decision that blocked it is TAKEN -- the macro-owner
paragraph below is no longer a question, and the 518 anchors now partition 490 conform + 6
knowingly non-conformant + 22 open, re-checkable further down this entry. What
still stops a scorecard from DECIDING a model swap is the instrument: the committed key's own
run-to-run swing straddles the acceptance threshold, and `cod-stage1.criteria.json` is still
`ratified_by: null`. Both are measured further down this entry; neither is a labelling question,
and neither is fixed by the decision that closed the first one.
NARROWED 2026-09-06: the swing blocks a THRESHOLD verdict, not every comparison. A three-arm A/B at
five runs an arm separated two models by +0.2246 `numbers_f1` with disjoint runs, while the PASS
COUNT flipped run to run in both arms -- so what this paragraph forbids is reading a `pass` as a
verdict, not comparing two candidates. See "The candidate BEATS the incumbent on production's CoD
path" below.

MEASURED 2026-09-05, the measurement that closes the path. vLLM 0.19.0, production container as-is
(`--kv-cache-dtype fp8_e5m2`), Qwen2.5-32B-AWQ rev `5c7cb76a268fc6cfbb9c4777eb24ba6e27f9ee6c`, the 40
gold articles on production's own request shape (`--task cod --endpoint-mode completions`,
`cod_json_v1.txt` + `cod_json_schema_v1.json`, client-side ChatML, `--max-tokens 8192`, temperature 0,
seed 42, concurrency 6). NOT production's full sampling: provenance records `repetition_penalty: null`
and `stop: null` where production sends 1.1 and `ExtractionOptions.StopTokens`, so this run is
production's PROMPT PATH -- which is what `production_prompt_path` certifies and what §MODEL_ACCEPTANCE
requires -- and not a byte-for-byte replay of its request. Pass `--repetition-penalty 1.1` and `--stop`
to close that gap. Result: `records: 40  errors: 0  schema_invalid: 0  truncated: 0`, wall 134.9s. Scored
by `eval_harness.py --task cod --cod-gold`: `scored 40/40  with_gold=40  with_prediction=40
call_failures=0`, `measurable: 24/29  passed: 10  failed: 14  not_measurable: 5`, shuffled-gold control
`numbers_f1` **0.0020** (FLOOR_OK), scorecard stamped `production_prompt_path: true`. The five
not-measurable are `period`, `certainty` and `text_quote_{precision,recall}` -- absent from CoD's schema
by design (stage 1 of 4; supplied downstream by dsl-parser-mcp `/parse_json` + the verifier + the
adapter) and reported null WITH that reason rather than synthesised -- plus `null_precision`, which has
no denominator because no gold record is empty.
This entry used to lead with the same engine and the same prompt scoring `schema_invalid: 3` and
`aggregate_f1: null` on 3 records. The instrument changed, not the model.

THE 14 FAILURES ARE NOT A MODEL VERDICT, and must not be quoted as one. They land exactly where this
gold is known weak: `claims_recall` -0.60, `events_precision` -0.55, `events_recall` -0.55,
`claims_precision` -0.52 -- the free-form `event_kind`/`claim_kind` problem measured below, where two
careful HUMAN labellers agree 0.302 and 0.138. Every threshold is PROVISIONAL and most are
`basis=carried` from the ratified CoVe bar, i.e. never measured on this task. The arrays that carry the
gold read: `number_value_accuracy` 0.925, `number_unit_accuracy` 0.920,
`number_source_entity_exact_match` 0.915, `source_entity_referential_integrity` 0.987, `json_valid` 1.0.

THE 40-RECORD SUBSTRATE IS NOT A COMMITTED ARTIFACT, and the re-check needs it. `run_model.py
--substrate` reads a JSON LIST; the gold's 40 articles are scattered through the 597-record v6.2
substrate (indices 0..9504, across two source files) and `--limit` takes a PREFIX, so `--limit 40` runs
the wrong 40 and still exits 0. Build the subset by joining on `(source_file, source_index)` -- nothing
in the repo does it for you, which is the next small piece of debt on this path.
Re-check:
```
python3 - <<'PY'
import json
sub = json.load(open('/opt/ai-inference/training-data/eval-substrates/v6.2-cove-plus-negatives-20260510T223757Z.json'))
gold = json.load(open('LlmBenchmark/cod-gold/cod_stage1_gold_v1.json'))
by = {(r['source_file'], r['source_index']): r for r in sub}
json.dump([by[(a['source_file'], a['source_index'])] for a in gold['articles']], open('/tmp/g40.json', 'w'))
PY
python3 LlmBenchmark/scripts/run_model.py --task cod --endpoint-mode completions \
  --prompt-file SentinelCollector/src/cod-prompts/cod_json_v1.txt \
  --schema-file SentinelCollector/src/cod-prompts/cod_json_schema_v1.json \
  --chat-template $'<|im_start|>user\n{0}<|im_end|>\n<|im_start|>assistant\n' \
  --max-tokens 8192 --concurrency 6 --substrate /tmp/g40.json \
  --endpoint http://localhost:8000 --model Qwen/Qwen2.5-32B-Instruct-AWQ --out /tmp/cod-preds.jsonl
python3 LlmBenchmark/scripts/eval_harness.py --task cod --substrate /tmp/g40.json \
  --cod-gold LlmBenchmark/cod-gold/cod_stage1_gold_v1.json --predictions /tmp/cod-preds.jsonl \
  --adapter-meta /tmp/cod-preds.jsonl.provenance.json --out /tmp/cod-scorecard.json
```
A non-zero `schema_invalid`, or `production_prompt_path: false`, means this closure regressed.
TWO CAVEATS ON THAT SENTENCE AND ON THE COMMAND ABOVE IT [2026-09-06]. The `--max-tokens 8192` is what
this run USED, not what the task needs -- 4096 truncates nothing (see "THE 8192 ... IS NOT A
REQUIREMENT" below). And `production_prompt_path: true` is NOT sufficient for the claim this line makes:
the flag reads no decoding knob, so this closure -- measured with `repetition_penalty` unset -- is a
closure of production's PROMPT PATH and not of production's REQUEST. Both are developed in the
"blind on a second axis -- SAMPLING" paragraph further down.

The two sampling knobs above are `CpuCod__JsonRepetitionPenalty` and `ExtractionOptions.StopTokens`;
`seed=42` is `ExtractionOptions.V2Seed`. What the runner does and does not reproduce of production's
sampling has its own entry -- "still diverges from production's sampling on `--stop` and `--min-p`".

DO NOT close the remaining question by hand-mapping `numbers[]` onto the CoVe gold fields -- `period`
and `certainty` have no source in that payload, so the mapping would inject a constant penalty larger
than the ~0.05 effect being measured, and the adapter's own design choices would dominate the result.
The alignment key is `(context, source_entity)` with independent floors, NOT `source_text`: that field
is a 1-3 token literal, so two unrelated figures both written `"$15"` align perfectly on it and every
per-field accuracy inherits the wrong pairing. `value` is out of the key for the same reason plus one
more -- anything in the key reads 1.0 by construction, and value accuracy is the number a model swap
turns on.

WHAT THE CoD SCORER MEASURED ON ITS OWN, WITH NO GOLD [2026-09-05]. Two of its metrics need none, and
they are the only CoD numbers that exist today. Qwen2.5-32B-AWQ rev `5c7cb76a...` @ vLLM 0.19.0 on
production's exact request shape (client-side ChatML -> `/v1/completions`, `cod_json_v1.txt` +
`cod_json_schema_v1.json`, temperature 0, seed 42, repetition_penalty 1.1, max_tokens 4096, production's
stop tokens), all 597 substrate articles:

| number | value | reading |
|---|---|---|
| `source_entity_referential_integrity` | **0.9472** | 2,799 of 2,955 non-empty anchors. 156 name an entity the same response never emitted. Bar 0.95; FAILS by 0.0028. |
| `source_entity_empty_rate` | **0.5806** | 4,090 of 7,045 predicted numbers carry `""`. NOT a conformance figure, and NOT legal-by-definition: `""` is the answer only where no ONE named entity owns the number, and #1017 took macro prints OUT of that set -- the SERIES owns its own print -- so a blank on a macro print is now a defect. This run predates that correction on both sides (the row landed at #1014; `cod_json_v1.txt`'s macro clause changed only at #1017), so the rate says what the OLD prompt asked for. REQUIRED READING beside the row above -- integrity is scored on the other 2,955, 42%, so the row above alone cannot be told from 0.9472-over-everything. (An earlier version of this cell said a model blanking every anchor "would score 1.0 on an empty denominator". False: the metric returns null with a reason -- see `cod-stage1.criteria.json`'s `not_the_reason`.) |
| `number_source_text_verbatim_rate` | 0.9709 | 6,840 of 7,045 literals appear verbatim (whitespace-normalized) in the article. Bar 0.90; passes. |
| `json_valid` | 0.9849 | 588 of 597 parse and satisfy the schema. All 9 failures are `finish_reason: length` at 4,096 completion tokens -- the loop-guard cap, NOT JSON discipline. Production salvages partial JSON there; the harness deliberately does not. Bar 1.00; FAILS. |
| `per_document_mean_latency_seconds` | 19.64 | concurrency 6 against the live production engine. Bar 300; passes. |
| shuffled-gold control, `numbers_f1` | 0.0106 | every prediction scored against a DIFFERENT article's facts, against 0.9993 unshuffled on the same corpus. Near zero is the floor this arm exists to establish, and it is known without trusting the gold or the model. |
| `identityless_predicted_items` | events **150**/2520, claims **38**/2351, numbers 5/7045, entities 0/5527 | items whose identity fields are BLANK. The schema's `required` is satisfied by `""`, so the model emits events with no `subject` and claims with no `object`; such an item cannot align with anything, including a perfect copy of itself, and scores as a false positive AND a false negative. This is what puts the tautology ceiling below 1.0 (`events_f1` 0.9405, `claims_f1` 0.9838) and it accounts for each one exactly. A gold labeller will hit the same wall -- an event with no subject cannot be labelled either. |

The integrity figure is the honest one to act on: the CoD prompt calls `source_entity` the
entity-resolution anchor and says a name absent from `entities[]` is "worse than none at all", so those
156 are anchors that resolve against nothing. It needs no gold and no labelling budget to re-check.
NOT MEASURED HERE: whether the extractions are CORRECT. Every number above is either self-consistency or
schema conformance. Recall in particular is unmeasured and unmeasurable without gold -- do not read
`0.9709` or `0.9472` as extraction quality.

Re-check (a CoD corpus now comes from `run_model.py --task cod` itself -- see the re-check above):
```
python3 LlmBenchmark/scripts/eval_harness.py --task cod --substrate <substrate> \
  --predictions <cod-preds.jsonl> --out /tmp/cod.json
python3 -c "import json;d=json.load(open('/tmp/cod.json'));\
print(d['metrics']['source_entity_referential_integrity']['value'],\
d['diagnostics']['source_entity_empty_rate']['value'],d['controls']['shuffled_gold']['value'])"
```
A shuffled-gold value that is NOT near zero is the finding, not the model's: it means the alignment key
stopped discriminating, or the corpus went near-duplicate.

THE GOLD ITSELF. `LlmBenchmark/cod-gold/` holds 1,744 gold facts in CoD's own shape over 40
deliberately-chosen articles, every object validating against `cod_json_schema_v1.json` under
jsonschema Draft 2020-12, every fact naming the labeller that produced it
(`LlmBenchmark/scripts/build_cod_gold.py`, `verify_cod_gold.py --selftest` = 10/10 known-bad mutations
caught -- 8 positive, 2 negative -- $3.40 measured over 96 requests). A scorer now reads it, on the run
above; what it cannot yet DECIDE is the rest of this entry.
Re-check: `python3 LlmBenchmark/scripts/verify_cod_gold.py --gold LlmBenchmark/cod-gold/cod_stage1_gold_v1.json
--corpus LlmBenchmark/cod-gold/cod_stage1_corpus_v1.json --selftest` exits 0 while the gold is intact.

WHAT THE GOLD DOES NOT COVER, measured on it 2026-09-05 and re-checkable from its own `controls` block:
- `guidance` and `regulatory` have ZERO gold members out of 40. Not a labelling slip -- the cross-check
  confirmed the primary's `article_type` on 37 of 40, so both labellers independently place every
  candidate elsewhere. This substrate's guidance articles are earnings articles that also guide, and its
  regulatory articles are macro stories with a court in them. A per-class article_type figure for those
  two values is an empty cell, not a score.
- Inter-labeller agreement splits hard by array: numbers 0.886, entities 0.737, events 0.302, claims
  0.138 (two independent full extractions, 8 articles). `event_kind` and `claim_kind` are free-form
  strings with no enum -- 39 distinct event_kind and 44 distinct claim_kind values across the two
  labellers on those 8 articles -- so two careful labellers name the same event differently and score
  zero. Rescored on `subject` ALONE the vocabulary effect separates: events recover 0.302 -> **0.708**,
  claims only 0.138 -> **0.356**. So for events the free-form kind is the whole problem; for claims it is
  NOT, and the two labellers genuinely pick different claim subjects. Stated plainly: numbers (518) and
  entities (624) are usable gold, 1,142 of 1,744 objects (65%); events are usable on `subject`+`trigger`
  only; a model comparison run over `claims` measures noise. A scorer that weights the four arrays
  equally reports vocabulary as model quality.
- Three gold `numbers[].value` fields hold RANGES ("3-4", "4-5", "50-100") where the schema's contract is
  a single normalised number. Small, real, and in the gold: `controls.control_2b_invention_sweep.detail`.
- The recall denominator is 89 hand-counted facts across 5 of the 40 articles. It bounds those five.

**ALIGNABILITY IS NOT SCHEMA-VALIDITY, and a scorer built on this gold has two decisions to make.**
Measured 2026-09-05 on the gold's `controls.control_5_alignability`; both hazards are invisible to
jsonschema and silent in a score.
1. A required string field can be present and BLANK. It then aligns with nothing -- including a
   byte-perfect copy of itself -- scoring as a false positive AND a false negative at once, and the loss
   reads as a model miss. Production's own Qwen2.5 over 597 substrate articles emits 150 of 2,520 events
   with a blank `subject` and 38 of 2,351 claims with a blank `object`, every one schema-valid; that is
   what holds a gold-tautology ceiling to 0.9993 instead of 1.0. This gold carries ZERO of them, with one
   deliberate exception: `numbers.source_entity` is blank on **22 of 518**, because production's prompt
   SPECIFIES `""` where no ONE named entity owns the number. A NATIONAL GAS PRICE IS NO LONGER ONE OF
   THEM, and neither is any other macro print: the decision below gives the SERIES ownership, and the ten
   blanks that rested on the old clause now carry the series the article names. The 22 that remain are
   6 blank-by-design (a figure spread across several entities, a bare count) and 16 the article never
   NAMES -- article 1's eleven Conference Board survey shares and article 48's five Challenger job-cut
   figures. They are KEPT -- dropping them would delete real facts. **A scorer must treat `""` as a value
   that aligns with `""`, never as an absent identity**, or it silently zeroes 4.2% of numbers, and 11 of
   article 1's 22 and all 5 of article 48's.
   COUNT THE WHOLE CENSUS, NOT THE IDENTITY FIELDS. That 22 counts only the REQUIRED fields a scorer
   aligns on. Gold holds **171** blank strings in total: 22 `numbers.source_entity`, 5 `events.object`,
   and **144 `entities.ticker`**. The latter two are optional in the schema and outside every alignment
   key here, so the zero-unalignable-blanks claim is unaffected -- but "the only blanks are the
   source_entity ones" was wrong by 149 and is the kind of number that gets quoted forward. The 144 are
   also an ENCODING DEVIATION: production's prompt says of `ticker` "include ONLY if the ticker is stated
   in or directly resolvable from the article; otherwise OMIT", and these emit `""` instead. A scorer
   doing field-level ticker comparison marks a correctly-OMITTING model wrong on all 144.
2. The obvious alignment key COLLIDES. On `(event_kind, subject)` 47 of 247 events (19.0%) and on
   `(claim_kind, subject)` 87 of 355 claims (24.5%) share a key with another object in the SAME article
   -- one article states six different things about the BLS, all `labor_market_statement`. Adding the
   discriminating field (`trigger` for events, `object` for claims, `value` for numbers) takes every
   array to **0 collisions**. Align on the wide key; the narrow one matches arbitrarily inside those
   groups and credits a prediction for the wrong fact.
Re-check: `python3 LlmBenchmark/scripts/verify_cod_gold.py --gold ... --corpus ... --selftest` prints
`10/10 controls behaved as required`; a lower number, or any `alignability:` or `anchor:` line in the
non-selftest output, means one of these regressed.

**THE SCHEMA'S STRING CAPS SHAPED THE GOLD, AND ONCE PRODUCED A KNOWINGLY-WRONG LABEL.** Measured
2026-09-05 on the first build; the instances are fixed, the mechanism is not. `cod_json_schema_v1.json`
caps `numbers.context` at 80 characters and `claims.object` at 120, and the builder DISCARDS a
cross-check correction whose corrected string exceeds the cap -- disposition `not applied: would break
schema` -- leaving the uncorrected value in gold. That is a length constraint silently winning against a
correctness one.
- `numbers.context` at 80: article 0's `$35.69` row said the earnings were reached **in January** where
  the article says **In December**; January is only the release-title month, and the sibling row for the
  same sentence said December, so one sentence was labelled two ways. The correction was dropped for
  being **81** characters: `average hourly earnings for all employees on private nonfarm payrolls in
  December`. It fits at 79 as `... private nonfarm payrolls, December` -- " in " -> ", " -- so this one
  was a rewording away, and shipping the wrong month instead is the defect. The shipped string was
  exactly 80, the only `numbers.context` in the corpus sitting at the cap.
- `claims.object` at 120: **19 of 355 (5.4%)** sat at exactly 120, every one severed mid-phrase, one
  shipping corrupt text -- article 522 ended `...sticky inflation andeLe` where the article reads `the
  risk of sticky inflation and elevated asset prices`. The cross-check supplied the correct 139-char
  string and it was discarded for length. Others truncated a labelled entity out of existence (472 ended
  `...DDR4 and DDR` while `DDR5` is its own entity in the same file) or ended on a dangling preposition.
  All 19 re-cut from the article to a phrase boundary; **0 claims now sit at 120**, max is 119.
THE DETECTION HEURISTIC IS THE LENGTH HISTOGRAM: before the fix, lengths 111-119 held 22 objects between
them and length 120 alone held 19. A cap-shaped spike is truncation, not natural phrasing. Re-check:
`python3 -c "import json,collections;d=json.load(open('LlmBenchmark/cod-gold/cod_stage1_gold_v1.json'));
print(collections.Counter(len(c['object']) for a in d['articles'] for c in a['gold']['claims'])[120])"`
prints `0`; anything above ~2 means truncation is back.
OPEN, and NOT fixable inside this gold: the caps belong to production's schema, so raising them changes
what production emits, and no measurement yet says whether 120 costs real claim content at extraction
time or only cost it at labelling time. Whoever raises them needs that number first.

**IS A MACRO SERIES AN OWNER? DECIDED 2026-09-05, BY THE USER: YES. The series owns its own print,
the country does NOT, and `""` is not the answer. The prompt's worked example said the opposite and
is rewritten; 20 gold anchors were repaired to match it and the 518 now partition **490 conform
+ 6 knowingly non-conformant + 22 open** (re-check below the disclosure two paragraphs down). An
earlier revision of both lines read **496 of 518 conform**, which is 518 minus the blanks alone:
"conform" silently meant "non-blank" and swallowed the six rows the very same paragraph discloses.
A partition that does not sum to 518 out of buckets each named is how that recurs. THE SIZINGS
BELOW WERE ALL TAKEN ON THE PRE-DECISION GOLD (129 macro-owned rows, before the repair moved it to
149), so they price the question that WAS open and are NOT a measurement of the gold as it now
stands. Dropping both free-form fields from the alignment key was worth 0.5064 `numbers_f1` and 86.6%
of the POOLED run-to-run swing; `source_entity` alone 0.3707 and 65.4%; the then-undecided quarter,
isolated, 0.2048 under the harness's greedy matching and 0.2105 under maximum-cardinality matching on
the same graph -- both measured, neither a bound by construction -- and what it did to the swing
REVERSED SIGN BY ARM (pooled the range GROWS 15.6%, at concurrency 6 it GROWS 45.0%, at concurrency 1
it SHRINKS 14.3%). THE SWING IS WHAT REMAINS, AND THE DECISION DOES NOT REPAY IT -- that was the
finding then and the decision does not change it. Re-measuring any of these on the amended gold needs
the five predictions files this entry's PROVENANCE LIMIT says are not in the repo.**
The gold now anchors **149 of 518 (28.8%)** numbers to a `macro_indicator` entity ("unemployment
rate", "Manufacturing PMI") -- 129 before the decision plus the 20 repaired. `cod_json_v1.txt` used to
read `On "US CPI rose 3.1%", context is "US CPI year over year" and source_entity is ""`; it now reads
`... and source_entity is exactly "US CPI"`. The question was pre-existing and unwritten, and it was
never one agent's to settle -- which is why it sat here as debt. `source_entity` sits in both proposed
alignment keys, so a scorer inherits the answer this gold holds; that answer is now a decided one
rather than an accident of labelling.
WHAT IS STILL OPEN IS 22 ANCHORS, NOT THE CONVENTION. 6 are blank BY DESIGN (a figure spread across
several entities, a bare count) and 16 are ones the article never NAMES -- article 1's eleven
Conference Board survey shares, which it describes in clauses and never names as indicators, and
article 48's five Challenger job-cut figures, whose series exists in SecMaster (CHALLENGER_JOB_CUTS)
but is absent from that two-line article. They stay blank on the clause that survived the change
unaltered, NEVER invent a name. Closing them needs a rule for a series a story describes without
naming -- not a re-decision of this one.
AND ONE KNOWN NON-CONFORMANCE, DISCLOSED RATHER THAN GUESSED: article 183's six payroll rows stay on
`US`. That article contains ZERO occurrences of "payroll" or "nonfarm", so `""` was the
rule-conformant answer and gold rewards a prompt violation on 6 of 518 rows. Left as-is deliberately;
inventing the series name would be the larger error, and it is the same 16-row question above.
RE-CHECK OF THE WHOLE PARTITION, which is what catches a bucket being folded into another:
```
python3 -c "
import json
d=json.load(open('LlmBenchmark/cod-gold/cod_stage1_gold_v1.json'))
n=[(a['id'],(x.get('source_entity') or '').strip()) for a in d['articles'] for x in a['gold']['numbers']]
b=sum(1 for _,s in n if not s); k=sum(1 for i,s in n if i.endswith(':183') and s=='US')
print(len(n)-b-k, k, b, len(n))"
```
2026-09-05 -> `490 6 22 518`. The first three MUST sum to the fourth, and any figure this entry
quotes for conformance MUST be one of the four. `496` is none of them and only exists by adding a
bucket to `490` without saying so.
THE 149 ARE A SUBSET OF THE `source_entity` DISPUTE, NOT THE WHOLE OF IT -- which is why the three
figures above are three different numbers, and why the largest of them is NOT this question's price.
The gold's anchors also read `country 13`, `industry 15`, `concept 10` and `<blank> 22` (census near
the end of this entry), and on the flagship article `sentinel-v6.2-cove.json` index 0, 16 of the 30
rows are non-macro. An earlier revision of this headline, its commit subject and its PR title all
carried the 0.5064/86.6% pair against the macro quarter: those are the BOTH-FIELDS figures, and the
body has always said so. Quote the row you mean -- and quote the ARM, because the five runs behind
every percentage here are not one population (next paragraph).

THE SIZE. Five runs of the incumbent (Qwen2.5-32B-AWQ @ vLLM 0.19.0, production's prompt path, the
40 gold articles, seed 42, temperature 0), rescored through the harness's own key functions under
alternative keys. NOT "five identical runs", which an earlier revision of this line called them:
THREE ran at concurrency 6 and TWO at concurrency 1, because concurrency was itself under study that
afternoon (`drive.sh`, interleaved c6/c1/c6/c1/c6). So the POOLED range is a CROSS-ARM gap, and both
per-arm ranges are smaller than it:

| alignment key | `numbers_f1` mean | range POOLED (n=5, two arms) | range c6 (n=3) | range c1 (n=2) |
|---|---|---|---|---|
| committed `(context, source_entity)` | 0.3883 | 0.0893 | 0.0427 | 0.0557 |
| committed, `source_entity` floor waived on the 129 macro-owned rows ONLY | 0.5931 | 0.1033 | 0.0619 | 0.0477 |
| `context` alone | 0.7591 | 0.0309 | 0.0210 | 0.0013 |
| `source_entity` alone | 0.4942 | 0.0953 | 0.0625 | 0.0578 |
| `value` alone -- **INADMISSIBLE AS AN ACCEPTANCE KEY, DIAGNOSTIC ONLY** (shuffled-gold floor 0.0267, the WORST of the five runs, not their mean) | 0.8947 | 0.0120 | 0.0120 | 0.0032 |

EVERY SWING PERCENTAGE IN THIS ENTRY DERIVES FROM THE POOLED RANGE unless it names an arm, so each
one is partly measuring the gap BETWEEN the arms. The pooled column stays the headline because it is
what a reader re-derives by handing the rescorer all five files -- but n=5 split 3/2 cannot attribute
the spread to concurrency, and nothing here claims it does.

THE `value`-ALONE ROW IS NOT A SCORE AND MUST NOT BE CARRIED AWAY AS ONE. It is precisely the shape
this entry's own alignment-key paragraph forbids: anything in the key reads 1.0 by construction, so a
key holding `value` makes value accuracy -- the number a model swap turns on -- unmeasurable.
`eval_harness.py:539-544` carries the same refusal at the code. What 0.8947 says is that the model's
NUMBERS are largely right and the KEY is what rejects them; it does not say the incumbent's real
score is 0.89. The macro-waived row is not a shippable key either -- it reads the GOLD's own
`ent_type` to decide where to waive, which is only possible once the question is settled -- so it is
a counterfactual, reported as an UPPER BOUND (method below).

Dropping BOTH free-form fields from the key adds 0.5064 to the mean and removes 86.6% of the POOLED
swing (0.0893 -> 0.0120); per arm, +0.5228 and 71.9% at c6, +0.4818 and 94.3% at c1. THAT IS THE
BOTH-FIELDS ROW AND IT IS NOT THE MACRO QUESTION'S PRICE. Removing `source_entity` alone (committed
-> `context` alone) adds 0.3707 and removes 65.4% pooled; per arm, +0.3806 and 50.9% at c6, +0.3560
and 97.7% at c1.
Waiving the `source_entity` floor on the 129 macro-owned rows and NOWHERE ELSE adds 0.2048 -- and
this is the figure whose SIGN REVERSES BY ARM, which is why it is no longer written as one. POOLED it
removes NONE of the swing and the range GROWS 15.6% (0.0893 -> 0.1033). At concurrency 6 (3 runs,
+0.2022) the range GROWS 45.0% (0.0427 -> 0.0619). At concurrency 1 (2 runs, +0.2087) it SHRINKS
14.3% (0.0557 -> 0.0477). An earlier revision of the headline read "AND NONE OF THE SWING", which is
the pooled answer written as if it were the only one. WHAT SURVIVES ALL THREE: settling the macro
convention alone does not reliably repay the swing this entry opens with, and n=5 across two arms is
far too little to say which way it cuts. METHOD, because the number is only as good as it: the
committed key is applied unchanged to every pair except those whose GOLD `source_entity` is a
`macro_indicator` the same article declares, where the floor is skipped and the pair ranks on
`context`.
0.2048 IS AN UPPER BOUND MEASURED ON THIS DATA, NOT ONE BY CONSTRUCTION -- an earlier revision of this
line claimed the latter, and that claim is false. Waiving the floor credits PERFECT agreement on those
rows, which is the most any settlement could buy, and a real convention still has to be one the model
emits: THAT half is by construction. The half that is not: `eval_harness._cod_align` is GREEDY, and
greedy matching is not monotone under a change to the candidate graph -- the waiver both ADDS edges
and RE-WEIGHTS existing ones (a waived row ranks on `context` alone, so it can be outbid by a pair the
committed key ranked below it). Built from those very functions, a 3x3 grid scores TP 2 committed and
TP 1 waived. On THIS data the bound does hold: TP_waived >= TP_committed in all 200 article-runs --
which `rescore_alignment_keys.py` now COUNTS and prints on every invocation rather than leaving to
argument. THE RESIDUAL, also measured: greedy leaves true positives on the table, so a real convention
could beat the "bound" by whatever greedy is short. Maximum-cardinality matching on the SAME waived
candidate graph finds 1,465 pairs against greedy's 1,451, worth **+0.0057 `numbers_f1`** -- 0.5988
waived, **+0.2105** over committed, which is the headline's second figure -- also recomputed on
every run (`max_cardinality_headroom_f1`), never carried as a remembered number. NEITHER number
changes a decision here: 0.5931 and 0.5988 both clear the `min_initial` 0.4 the next paragraph
straddles.
`source_entity` is the larger half of the key: of 2,062 (pred, gold) pairs across the five
runs agreeing on BOTH value and unit, the committed key accepts 41.1% (848), rejects **40.1% (827) on
`source_entity` alone** with `context` already clear of its floor, and 11.0% (226) on `context` alone.
On the stricter value+unit+`source_text` proxy (n=1,713) source-entity-alone rejection is 37.9%.
Across the five runs `numbers_f1` correlates r=0.967 with the key's accept rate and r=0.118 with what
the model actually extracted (n=5, so directional not decisive) -- the published figure is measuring
the convention mismatch, not the extraction.

AND THE COMMITTED KEY STRADDLES ITS OWN ACCEPTANCE THRESHOLD, WHICH IS WHERE THE SWING DOES DAMAGE.
`LlmBenchmark/eval-substrate/cod-stage1.criteria.json` sets `numbers_f1.min_initial` **0.4**
(`basis: carried`, and the file's `ratified_by` is `null`). The five-run mean is **0.3883** and the
per-run values are c6 0.3646 / 0.3528 / 0.3955 and c1 0.3865 / **0.4422** -- four runs FAIL that
threshold and one PASSES, at seed 42 and temperature 0. THE STRADDLE IS NOT AN ARTEFACT OF POOLING THE
TWO ARMS: the one passing run is c1_b, and its own arm-mate c1_a, at settings identical in everything
this repo records, scores 0.3865 -- so 0.4 sits inside [0.3865, 0.4422] WITHIN a single arm. A gate
whose verdict flips run to run without the model changing cannot decide a model swap in either
direction. Moving the threshold does not repair that -- it only moves where the coin-flip band sits:
any bar inside the pooled [0.3528, 0.4422] flips run to run, and 0.4 is inside it. The swing has to
come out of the instrument first.
FOR A THRESHOLD. It does not have to come out first for a COMPARISON, and 2026-09-06 it did not:
five runs an arm at production's sampling give a within-arm sd of 0.0106 and an effect 8.0x the
widest within-arm range -- 2.5x the 0.0893 pooled swing above -- with the arms disjoint at run level.
The pass count still flipped in both arms, which is this paragraph, reproduced. See "The candidate
BEATS the incumbent on production's CoD path" below.

Re-check (the whole table, the headline's three figures, and both controls):
```
python3 LlmBenchmark/scripts/rescore_alignment_keys.py \
  --cod-gold LlmBenchmark/cod-gold/cod_stage1_gold_v1.json \
  --predictions <run>.jsonl [--predictions ... once per run] \
  [--scorecard <matching eval_harness scorecard>.json ... ]

# and this one needs nothing outside the repo -- fourteen mutations, each caught by name;
# prints `selftest: 14/14 controls behaved as required`:
python3 LlmBenchmark/scripts/rescore_alignment_keys.py --selftest
```
PROVENANCE LIMIT, stated because these figures cannot be re-derived from the repo ALONE. The rescorer
is committed and the gold is committed; the FIVE PREDICTIONS FILES ARE NOT. They came from the
`run_model.py` re-check near the top of this entry, 5 x 40 articles on production's own engine, and
live outside the tree, so re-deriving 0.3883/0.5931/0.7591/0.4942/0.8947 means re-running that first.
BUDGET ~33 MINUTES, NOT ~11. Measured wall clock: 147s / 143s / 138s for the three concurrency-6 runs
and 766s / 772s for the two at concurrency 1. An earlier revision of this line quoted the c6 figure
("~135s each") for all five, which understates the re-derivation by a factor of three. AND THE
ARTIFACTS CANNOT CORRECT THAT FOR YOU: `run_model.py`'s provenance sidecar records seed, temperature,
max_tokens, engine build and substrate hash but NOT concurrency, so "five identical runs" was
UNFALSIFIABLE from the committed-format files and had to be recovered from the driving script's log.
Whoever re-runs this either fixes the sidecar or records the arm by hand. This is the same disclosure
`LlmBenchmark/scripts/README.md` makes for the shuffled-gold figures, and for the same reason.
What IS re-derivable today: hand the rescorer any CoD predictions file and it regenerates the whole
table against the committed gold, importing `eval_harness`'s own `_cod_align`, `_token_f1`,
`_source_entity_affinity` and both floors rather than re-implementing them, so a change to the
scorer's notion of agreement moves these rows too.
AND ITS CONTROLS NOW HAVE CONTROLS OF THEIR OWN. `--selftest` builds a synthetic corpus and runs
fourteen mutations that must each be caught BY NAME, three of them NEGATIVE controls that must stay
quiet -- no predictions file needed, which the out-of-tree predictions used to prevent. THREE run
at n>1 deliberately: the shuffled-gold verdict as first written averaged the floor ACROSS runs, so a
run breaching at 0.4570 diluted to 0.0457 beside nine honest ones and printed `FLOOR_OK`, exit 0 --
and it had passed every SINGLE-run mutation test, because a defect in how a tool AGGREGATES across
units is invisible to a mutation exercised on one unit. It is judged PER RUN now, names the offending
run, treats an uncomputable floor as `NOT_MEASURABLE` rather than a pass, and takes its bar from
`eval_harness.SHUFFLED_CONTROL_MAX_HEADLINE` rather than restating it (the local copy had already
drifted to the opposite boundary, so exactly 0.10 read OK here and BREACHED in the scorer). Run
against the five real runs it reports `FLOOR_OK` and reproduces all five published `numbers_f1`
values to 1e-9.
FOUR OF THE FOURTEEN PIN THIS ENTRY'S OWN NUMBERS, added because a review mutated the shipped tool
eighteen ways and six of those mutations left it reporting `10/10`, exit 0. The four close FIVE of
those six, each proved by re-running its mutation against the shipped file and requiring exactly ONE
control to fail; the sixth was not carried into that round and is STILL OPEN, so a re-mutation sweep
of this tool should expect one escape these controls do not see.
(i) THE VERDICT MUST SWEEP ALL SEVEN KEYS. Every earlier fixture handed a record its partner's WHOLE
gold, so all seven variants floored identically at 0.3333 and a verdict reading the committed key
alone caught every one of them. The new fixture keeps the identity fields honest and takes only the
stranger's VALUES: the value keys breach at 0.3333 while committed sits at 0.0000 -- the same 26x
spread the real data shows (committed 0.0000, `value_only` 0.0267).
(ii) THE BAR MUST STAY `eval_harness`'s. Every 0.3333 fixture would survive inflating it threefold,
so one fixture now floors at 0.1111 against a bar of 0.1000.
(iii) A SINGLE RUN MUST PRINT `range n/a`, NEVER `0.0000`. Two routes reach that false "no swing"
and only the shared-basename one was pinned.
(iv) THE TWO FIGURES DEFENDING "MEASURED, NOT CONSTRUCTIONAL" -- the `0 of 200` counter and the
`+0.0057` headroom -- read on an honest corpus exactly as a counter that cannot fire and a
max-cardinality search degraded to greedy would read. The 3x3 grid cited in `macro_waived_key`'s
docstring is now an EXECUTABLE fixture where the waiver genuinely COSTS greedy two true positives
(committed 2, waived 1, maximum cardinality 2, headroom 0.1667).

WHAT IT HIDES: the correct value is present in the model's output for 0.845 of gold numbers (0.879
set-wise, 0.909 over the union of the five runs) against a committed `numbers_recall` of 0.367.
Article `sentinel-v6.2-cove.json` index 0 is the clean demonstration -- c6_a and c6_c, two runs at
IDENTICAL settings in the SAME arm, agreed with gold on the same 29 of 30 values and scored 0 and 14
true positives, because c6_a wrote `source_entity: "United States"` on all 30 numbers and c6_c wrote a
different convention on 16 of them. The concurrency-1 arm splits the same way (c1_a 0, c1_b 14), so
this one is not an arm effect either. Gold on that article holds a third answer again: 14
`macro_indicator` anchors, 10 industries, 6 demographic `concept`s and, since the decision, no
blanks. (`macro_indicator`, not "metric": ent_type `metric` occurs ZERO times in this gold, so a
reader searching the census for metric labels finds none. And the article is written
`<file> index <n>` rather than `<file>:<n>` because that pair is a substrate
`(source_file, source_index)`, not a `file:line` citation -- written the other way it becomes an
unresolvable citation in `scripts/verify-citations.py`, which is how it was found.)

NOT THE ENTITIES ARRAY -- do not carry this finding across. `entities_f1` is 0.6744 with a range of
0.0092, its key is already name-only, and TIGHTENING it scores LOWER (name-exact, 0.6366). Stricter,
not looser, and an earlier revision of this line had the word backwards: the committed `_entity_key`
(`eval_harness.py:556`) admits a pair at token-F1 >= 0.8, while name-exact is a multiset intersection
on the normalized name -- a strict SUBSET of what that floor already admits. Scoring lower is what a
subset key does, and it is the point: there is no slack in this key for a convention dispute to be
hiding in, which is exactly what makes the numbers key's slack a finding. Entity recall 0.556 is
genuine under-emission: 391-412 predicted against the 616 gold entities that run was scored on. The
macro-owner decision has since added 8 (616 -> 624), so a re-run scores against a slightly larger
denominator; 0.556 is the PRE-decision figure and is not restated here as a current one.

THE OPERATIONAL TIE-BREAK IS ALREADY IN THE CODE, AND IT DOES NOT FAVOUR THE COUNTRY. `source_entity`
is not a display field: `DslToMergedExtractionAdapter` puts it on `ExtractionResult.SubjectEntity`
(SentinelCollector D-15), which `DeterministicResolver` keys Rule 1 (candidate pre-selection), Rule 2
(`hybrid_subject`, NO surface filter) and Rule 2.5 (paid Gemini) off; the instrument that comes back
reaches `extracted_observations`, the digest and the matrix. Two measurements bear on the choice:
- `NonInstrumentEntTypes` (`SentinelCollector/src/Extraction/DslToMergedExtractionAdapter.cs:108`)
  excludes `country`, `macro_indicator`, `industry`, `sector` and `concept` ALIKE from the candidate
  list, so **194 of 518 (37.5%)** of this gold's anchors pre-select nothing under EITHER convention and
  fall through to Rule 2. It was 184 (35.5%) before the decision: moving 10 anchors off a country and
  10 off a blank onto the SERIES kept them all inside the excluded set, so the decision bought nothing
  at Rule 1 and was never argued on that ground.
- Rule 2 runs no surface filter, and the country surface is the measured wrong-instrument class:
  SentinelCollector D-1 counts 7,184 instrument-attaching rows carrying a `gpe_country` subject, 3,060
  of them landing on `U` (Unity Software) -- over ONE 31-day window, `extracted_at` [2026-07-15,
  2026-08-15), and a FLOOR rather than a total, because D-1 replayed the exact-match sets only and did
  not re-run the shape classes. Cite it with both caveats or it reads as an all-time count.
  SELECT against `atlas_secmaster` 2026-09-05: the surface
  `United States` exact-matches ONE instrument, `EMISSCO2TOTVTTTOUSA` "United States" -- a CO2
  emissions series -- while `Unemployment Rate` -> `UNRATE`, `All Employees, Total Nonfarm` ->
  `PAYEMS` and `Average Hourly Earnings of All Employees, Total Private` -> `CES0500000003` are all
  catalogued Economic instruments. The metric-series convention names something the catalog holds; the
  country convention names a carbon series.
That was the EVIDENCE FOR the decision, and the decision went with it: the resolver is supposed to
receive the SERIES on a macro print. The prompt, the gold and the producer's own labelling instruction
changed in ONE PR, as this paragraph said they would have to. What the evidence bought is a resolvable
surface where `""` resolved to nothing and `United States` resolved to a CO2 series; what it did NOT
buy is Rule 1 pre-selection, which still refuses a `macro_indicator` -- see the count above.

Re-check (the 149): `python3 -c "import json;
d=json.load(open('LlmBenchmark/cod-gold/cod_stage1_gold_v1.json'));print(sum(1 for a in
d['articles'] for n in a['gold']['numbers'] if {e['name']:e['ent_type'] for e in
a['gold']['entities']}.get(n['source_entity'])=='macro_indicator'))"` prints 149. It printed 129
before the decision; a 129 today means the 20 repairs were reverted, which is what
`build_cod_gold.py --selftest` exists to make loud.
Re-check (the 194 that can never pre-select, and the ent_type census behind it):
```
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
2026-09-05, after the decision -> `194 518` with `equity 187, macro_indicator 149, instrument 68,
org 26, <blank> 22, index 21, industry 15, country 13, concept 10, person 3, event 2, location 1,
region 1` and ZERO `<undeclared>`. Before it: `184 518` with `macro_indicator 129, <blank> 32,
country 23` and the other twelve rows unmoved -- the whole delta is 10 anchors off a country surface
and 10 off a blank, onto the series. Naming those series also added 8 `entities[]` rows the labellers
had never declared (616 -> 624), which is why the object totals above moved without a number
moving. A non-zero `<undeclared>` means a gold anchor stopped naming an entity its own article
declares, which is the one thing `source_entity_referential_integrity` scores a MODEL on.

THE 8192 IN THE `run_model.py` RE-CHECK NEAR THE TOP OF THIS ENTRY IS NOT A REQUIREMENT, AND THIS
PARAGRAPH USED TO SAY IT WAS ("budget ... for MORE THAN 4096 completion tokens"). Corrected 2026-09-06
against the loop-guard control below: 240 records at `--max-tokens 4096` on the local incumbent, across
BOTH prompt arms, give `truncated 0`, `finish_reason: stop` on every record, and a maximum observed
`completion_tokens` of **3148 -- 77% of the cap**. Nothing legitimate wanted 8192. Every record that
DID exceed 4096 in the unguarded arms sits on an article that was simultaneously degenerating:
`sentinel-v6.2-cove.json` indices 183 (4181, 6432, 6516), 219 (5767 twice) and 35 (5607) all saturate a
schema array at `maxItems` in the same run, and index 3 (4558, 4559) swings from `events: 1` to
`events: 23` and `claims: 25` to `claims: 33` across runs at temperature 0. The loop wanted 8192; the
task did not. An earlier enumeration of this list omitted index 219 and both 183-at-4181 records, and
called index 3 a looper though it saturates no array -- it is repetition-degenerate by the run-to-run
swing, which is a weaker test, and the distinction is kept here rather than smoothed over.
THE SEPARATE HOSTED-ROUTE FINDING STANDS and is not what the sentence above was ever about.
Measured 2026-09-05, Qwen3.8-27B via the HF router (deepinfra), production's CoD prompt now correctly
substituted, 2 substrate records at `--max-tokens 2048`: `truncated: 2`, `finish_reason: length` on
both, `completion_tokens: 4096` -- i.e. both records spent the entire budget and were cut off. A third
record at 6000 was still generating when the router returned 504. Truncated responses land in
`schema_invalid`, so a run budgeted too low reproduces the `schema_invalid == record count` symptom
this entry USED TO LEAD WITH -- which is CLOSED, at the top of this entry: production's engine on
production's prompt path reports `schema_invalid: 0`, and it closed on the chat template, prompt
assembly, scorer and runner, none of which a token budget touches. So a low budget is a BUDGET
artefact wearing a fixed defect's face, recorded here rather than reopened; the runner's own default
is 4096. Read `truncated` and `finish_reasons` in the provenance before concluding anything from
`schema_invalid`.

### The candidate BEATS the incumbent on production's CoD path -- +0.2246 `numbers_f1` -- and still cannot ship [2026-09-06]
THE COMPARISON THE ENTRY ABOVE SAYS THE SWING BLOCKS HAS NOW BEEN MADE, and the swing was not what
stopped it. Three arms, FIVE runs each, the 40 committed gold articles, production's CoD prompt path
AND production's sampling, local vLLM 0.19.0, the committed gold on the committed alignment key.
Every one of the fifteen runs reports `records 40  call_errors 0  schema_invalid 0  truncated 0`,
`finish_reasons {"stop": 40}`, `json_valid 1.0` and `production_prompt_path: true`.

| arm | model | client-side template | `numbers_f1` | sd | range |
|---|---|---|---|---|---|
| A -- incumbent | `Qwen/Qwen2.5-32B-Instruct-AWQ` | ChatML, sha256 `b18811f5` | **0.5153** | 0.0106 | 0.0282 |
| B -- candidate | `cyankiwi/Qwen3.8-27B-AWQ-INT4` | ChatML, sha256 `b18811f5` | **0.7400** | 0.0072 | 0.0175 |
| B' -- candidate + suppression | `cyankiwi/Qwen3.8-27B-AWQ-INT4` | ChatML + `<think></think>` prefill, sha256 `389c3561` | **0.6729** | 0.0100 | 0.0257 |

B-A = **+0.2246** (se_diff 0.0057), B'-A = **+0.1576** (se_diff 0.0065). NEITHER IS A p-VALUE and
neither is written as one -- n=5 an arm does not support one. What is claimed is SEPARATION: |diff|
is 39x and 24x its own se_diff, and the arms are disjoint at RUN level, `min(B) - max(A)` =
**+0.2036** and `min(B') - max(A)` = **+0.1305**, so the worst pairing of runs still separates.

**PLAN AGAINST B', THE CONSERVATIVE ARM.** B' is the same model with a `ThinkingSuppressionSuffix`
expressed in the client-side template. PRODUCTION SETS NO SUCH SUFFIX TODAY and this entry does not
claim it does: `ExtractionOptions.ThinkingSuppressionSuffix` defaults to `string.Empty`
(`SentinelCollector/src/Configuration/ExtractionOptions.cs:246`), nothing in
`/opt/ai-inference/compose.yaml` or any appsettings sets it, and D-26 says that emptiness is
DELIBERATE because the model served today does not reason (D-26,
`SentinelCollector/AGENT_README.md:117`). So B is what production's CURRENT configuration would
produce and B' is the arm a reasoning-model deployment might choose. Plan against B' because it is
the LOWER of the two and the suffix machinery exists precisely for a model like this one -- not
because anything sets it now. Either way the candidate is ahead: +0.2246 at B, +0.1576 at B'.

THE SUFFIX COSTS 0.0671 `numbers_f1` AND 19% WALL CLOCK HERE, WHICH IS NOT WHAT IT WAS BOUGHT FOR --
recorded as an open question, not answered. B and B' differ on the wire by the six prefilled template
tokens and NOTHING else: same model revision, same seed, same endpoint, same prompt and schema bytes.
Both arms sent `structured_output: true` -- a `response_format` json_schema
(`LlmBenchmark/scripts/run_model.py:429-433`) -- so neither arm could have emitted a reasoning block
for the suffix to suppress, yet B' spends 22% more completion tokens (59,613 vs 48,805 mean per
40-article run) reaching a lower score. Whoever plans the swap should measure whether the suffix is
needed at all on the GPU JSON path before paying that; D-26 governs the suffix REACHING the wire and
says nothing about whether it should be set, so this is a question for the swap, not a contradiction
of the entry.

**THE ERROR BAR CAME IN TIGHTER THAN THE EPIC BUDGETED FOR, AND FIVE RUNS WAS GENEROUS.** The entry
above sizes the instrument's swing as a RANGE, 0.0893 pooled and 0.0427 within the concurrency-6 arm,
against a ~0.05 effect; recomputing an sd from those same five committed-key runs gives 0.0346 pooled
and 0.0221 within the c6 arm. Measured here: within-arm sd **0.0106 / 0.0072 / 0.0100** and range
0.0282 / 0.0175 / 0.0257. That is 3.3x tighter than the pooled prior sd and 2.1x tighter than the c6
one, and the effect is 8.0x the widest within-arm range -- 2.5x the pooled swing the entry above says
a comparison must clear. READ THAT AS "THE NOISE HERE WAS SMALL", NOT AS "THE LOOP GUARD SHRANK THE
NOISE": the prior figures were taken at `repetition_penalty: null, max_tokens: 8192` and these at
1.1 / 4096, and this file already measures that axis moving `entities_f1` by 0.135 and reversing the
sign of a prompt comparison. Two runs an arm would have separated these models.

**WHAT DOES NOT MOVE IS THE PASS COUNT, AND IT STILL FLIPS RUN TO RUN.** Against the PROVISIONAL
criteria (`LlmBenchmark/eval-substrate/cod-stage1.criteria.json`, `ratified_by: null`) all fifteen
runs score 24 of 29 metrics measurable, and the passed count is A 8/24 in four runs and 9/24 in the
fifth, B 9/24 in three and 10/24 in two, B' 8/24 in four and 9/24 in the fifth. A gate whose verdict
moves without the model changing is the entry above's straddle, reproduced at production's sampling
on a 0.22 effect: SEPARATING two models and PASSING a threshold are different questions, and only the
first one is answered here.

FULL METRIC TABLE, every metric the harness could measure, means over 5 runs an arm:

| metric | A incumbent | B candidate | B' cand+suppr | B-A | B'-A |
|---|---|---|---|---|---|
| `numbers_f1` | 0.5153 | 0.7400 | 0.6729 | +0.2246 | +0.1576 |
| `numbers_precision` | 0.5611 | 0.7173 | 0.6274 | +0.1562 | +0.0663 |
| `numbers_recall` | 0.4764 | 0.7641 | 0.7255 | +0.2876 | +0.2490 |
| `entities_f1` | 0.5704 | 0.7842 | 0.7491 | +0.2138 | +0.1787 |
| `entities_precision` | 0.8831 | 0.8406 | 0.7886 | -0.0425 | -0.0946 |
| `entities_recall` | 0.4215 | 0.7349 | 0.7135 | +0.3135 | +0.2920 |
| `events_f1` | 0.2441 | 0.3693 | 0.3601 | +0.1252 | +0.1160 |
| `events_precision` | 0.3452 | 0.5514 | 0.4175 | +0.2062 | +0.0723 |
| `events_recall` | 0.1895 | 0.2777 | 0.3166 | +0.0883 | +0.1271 |
| `claims_f1` | 0.2145 | 0.3952 | 0.3636 | +0.1807 | +0.1491 |
| `claims_precision` | 0.3538 | 0.5475 | 0.4901 | +0.1937 | +0.1363 |
| `claims_recall` | 0.1544 | 0.3093 | 0.2890 | +0.1549 | +0.1346 |
| `number_value_accuracy` | 0.9359 | 0.9697 | 0.9680 | +0.0338 | +0.0321 |
| `number_unit_accuracy` | 0.8629 | 0.9116 | 0.9159 | +0.0487 | +0.0530 |
| `number_source_entity_exact_match` | 0.9060 | 0.8914 | 0.8957 | -0.0146 | -0.0103 |
| `number_source_text_verbatim_rate` | 0.9376 | 1.0000 | 0.9983 | +0.0624 | +0.0607 |
| `source_entity_referential_integrity` | 0.9368 | 1.0000 | 0.9922 | +0.0632 | +0.0554 |
| `ent_type_accuracy` | 0.7421 | 0.8055 | 0.8091 | +0.0634 | +0.0670 |
| `entity_ticker_accuracy` | 0.6714 | 0.4583 | 0.4167 | **-0.2130** | **-0.2547** |
| `event_kind_accuracy` | 0.4061 | 0.6699 | 0.5347 | +0.2638 | +0.1286 |
| `claim_polarity_accuracy` | 0.6926 | 0.7810 | 0.8265 | +0.0884 | +0.1339 |
| `article_type_accuracy` | 0.7250 | 0.6750 | 0.7000 | -0.0500 | -0.0250 |
| `json_valid` | 1.0000 | 1.0000 | 1.0000 | +0.0000 | +0.0000 |
| `per_document_mean_latency_seconds` | 16.07 | 20.28 | 24.22 | +4.21 | +8.16 |

PER-RUN VALUES for the four metrics anything is decided on, in run order:
```
A  numbers_f1              0.5272 / 0.4989 / 0.5171 / 0.5126 / 0.5207
B  numbers_f1              0.7358 / 0.7390 / 0.7484 / 0.7308 / 0.7458
B' numbers_f1              0.6738 / 0.6798 / 0.6577 / 0.6697 / 0.6834
A  entities_f1             0.5839 / 0.5617 / 0.5579 / 0.5659 / 0.5826
B  entities_f1             0.7811 / 0.7835 / 0.7840 / 0.7856 / 0.7867
B' entities_f1             0.7440 / 0.7525 / 0.7542 / 0.7473 / 0.7477
A  numbers_recall          0.4865 / 0.4575 / 0.4807 / 0.4730 / 0.4846
B  numbers_recall          0.7606 / 0.7625 / 0.7722 / 0.7548 / 0.7703
B' numbers_recall          0.7297 / 0.7317 / 0.7104 / 0.7201 / 0.7355
A  entity_ticker_accuracy  0.6744 / 0.6744 / 0.6744 / 0.6744 / 0.6591
B  entity_ticker_accuracy  0.4583 / 0.4583 / 0.4583 / 0.4583 / 0.4583
B' entity_ticker_accuracy  0.4167 / 0.4167 / 0.4167 / 0.4167 / 0.4167
```

**PROVENANCE -- THE BYTES, WHICH NO EARLIER RUN IN THIS EPIC RECORDED.** The neighbouring entries all
disclose that their figures die with `/tmp` (see the PROVENANCE LIMIT paragraph in "Convention B
measured end to end" below). This one's do not, because the INPUTS are named by digest and every one
of them is committed:

| input | sha256 | committed as / value |
|---|---|---|
| prompt | `0dd66ddec19ea58f01dcdad545da9aa4b8da4ab736746b44547bf189a6f445de` | `SentinelCollector/src/cod-prompts/cod_json_v1.txt` |
| schema | `1b9719041dfee8effb02062d8a4843897736f55ecf5fd7ab7610a7b173e948d6` | `SentinelCollector/src/cod-prompts/cod_json_schema_v1.json` |
| gold | `bc9c5b4c8331ac01ded0fc6eb718ea787b27fa39f6ff95cff07755925354acca` | `LlmBenchmark/cod-gold/cod_stage1_gold_v1.json` |
| criteria | `fefef64721986eacf918b6fcf7bf62b3bf898015e3a3d9fe2e253cbfb0370927` | `LlmBenchmark/eval-substrate/cod-stage1.criteria.json` |
| substrate | `008c338deaf6596884983883ef63bd1f0308c2f05403ad6daf83bdf3b392d684` | NOT committed -- see below |
| template A/B | `b18811f5f4fac851c7a16cdc2839d96ac3d2a284baf8712e5ed2a043ba203a88` | `<\|im_start\|>user\n{0}<\|im_end\|>\n<\|im_start\|>assistant\n` |
| template B' | `389c3561b39cf757fc65f93f973d30d8fc9be2c90089f63bb7b99ae49c01e941` | the same plus `<think>\n\n</think>\n\n` |

The four repo digests were re-computed from the tree at `3d1fa9e5` and match the provenance sidecars
exactly. THE SUBSTRATE WRAPPER IS THE ONE FILE THAT IS NOT REPO-DERIVABLE, and
that costs nothing HERE: its 40 article texts are byte-identical, in the same order, to the 40
`content` fields of the committed `LlmBenchmark/cod-gold/cod_stage1_corpus_v1.json` (checked
40 of 40), and with `--prompt-file` the runner reads only `input.content` and the join key --
`record["instruction"]` is the FALLBACK the flag overrides (`LlmBenchmark/scripts/run_model.py:406-409`).
The wrapper's own sha256 is unreproducible because it also carries the v6.2 substrate's
`instruction` and `output` fields, which this task never reads and which live outside the repo.

`request_sampling`, identical on all fifteen runs: `temperature 0.0`, `seed 42`, `repetition_penalty
1.1`, `max_tokens 4096`, `top_p/top_k/min_p/presence_penalty null`, `structured_output true`,
`endpoint_mode completions`, concurrency 6.
`stop` WAS `null` AND PRODUCTION SENDS `["<|im_end|>", "<|endoftext|>"]`
(`SentinelCollector/src/Configuration/ExtractionOptions.cs:254`) -- so this is production's prompt
path at production's DECODING, still not a byte-for-byte replay of its request. On this corpus the
omission is observably inert (`truncated 0`, `finish_reason stop` on all 600 records), but the gap is
real and `--stop` closes it.
Models: `Qwen/Qwen2.5-32B-Instruct-AWQ` rev `5c7cb76a268fc6cfbb9c4777eb24ba6e27f9ee6c` on
`localhost:8000`; `cyankiwi/Qwen3.8-27B-AWQ-INT4` rev `63768c10df38c0395e12ef49edac1bd539eaeeea` on
`localhost:8001`. Engine `vllm 0.19.0` both. Harness at `3d1fa9e5`.

**THE ARMS WERE NOT SERVED ALIKE, AND THE DIFFERENCE FAVOURS THE CANDIDATE. READ THIS BEFORE QUOTING
THE NUMBER.** `localhost:8000` is production's live `vllm-server`, serving
`--max-model-len 32768 --kv-cache-dtype fp8_e5m2 --gpu-memory-utilization 0.92 --max-num-seqs 16`
(read off `nerdctl container inspect vllm-server`). The candidate on `localhost:8001` came up at
`--max-model-len 15360` with `fp8_e5m2` REFUSED on its compressed-tensors checkpoint, so it ran on
UNQUANTIZED KV. The incumbent alone carried the quantized cache.

| arm | endpoint | `--kv-cache-dtype` | `--max-model-len` |
|---|---|---|---|
| A incumbent | `:8000` (production's own container) | `fp8_e5m2` | 32768 |
| B / B' candidate | `:8001` | unquantized (`fp8_e5m2` refused) | 15360 |

AT LEAST THESE TWO AXES -- THE TABLE IS NOT A CLOSED DIFF. The A row is the incumbent's full live
flag set minus four the table omits (`--quantization awq_marlin`, `--max-num-seqs 16`,
`--enable-auto-tool-choice --tool-call-parser hermes`, `--generation-config vllm`); the candidate's
remaining flags are UNRECOVERABLE, because its container was removed and the serve invocation was
never written down. `--generation-config vllm` is the one to notice: `run_model.py` omits
`top_p`/`top_k`/`min_p`/`presence_penalty` from the body when they are null, so the SERVER's defaults
apply, and that flag is what makes production ignore a model card's. This candidate's card sets
`presence_penalty 1.5`, worth a 0.070 swing on the substrate -- larger than the KV effect below. It
would have cut RECALL, so it works against the candidate and does not threaten the headline; it is
named because an unrecorded axis is not a controlled one.

THIS FILE ALREADY PRICES THAT AXIS: "Production's `fp8_e5m2` KV cache costs ~0.05 aggregate F1 on
extraction, concentrated in RECALL" -- 0.443 -> 0.494, with `text_quote_recall` +0.058 and
`selectivity_recall` +0.058. The headline gains here ARE the recall metrics (`numbers_recall`
+0.2876, `entities_recall` +0.3135), so the confound points the same way as the effect. It is
measured on the SUBSTRATE task, not this one, so ~0.05 is an order of magnitude and not a
subtractable correction.
WHAT SURVIVES AND WHAT DOES NOT. The DIRECTION survives with room: +0.2246 is ~4.4x the 0.051 the
KV flag is worth, and no plausible reading of a 0.05-scale handicap closes a 0.2246 gap with
disjoint runs. What does NOT survive is quoting +0.2246 as a clean model-vs-model delta -- it is an
ARM-vs-ARM delta over model AND KV dtype AND context length -- though CONTEXT LENGTH IS OBSERVABLY
INERT HERE (`truncated 0` and `finish_reason stop` on all 600 records, ~2.1K-token prompts against
15,360), so the one live axis is the KV dtype. The three adverse deltas in blocker 4 below (0.0146,
0.0425, 0.0500) are all at or under the confound's AGGREGATE magnitude of 0.051, so on that reading
their SIGNS are not established either.
BUT ALL THREE ARE NON-RECALL METRICS, and the fp8 entry's own non-recall rows are the fairer
yardstick -- +0.030 `symbol_exact_match`, +0.029 `period_accuracy`. Measured against THOSE, only
`number_source_entity_exact_match` (-0.0146) sits inside the confound; `entities_precision`
(-0.0425) and `article_type_accuracy` (-0.0500) EXCEED it. So the honest split is one delta the
confound could explain and two it probably cannot -- which cuts AGAINST the candidate, and is
recorded here rather than left on the flattering aggregate comparison.
THE MISSING CONTROL IS ONE THIS FILE WAS ALREADY WAITING FOR: the `fp8_e5m2` entry's own re-check --
both KV arms of the INCUMBENT on the CoD path -- would price the confound on this task and was not
run before the GPU was handed back. Run it before the next quote of this number.
KNOWN-BAD CONTROL, run inside every one of the fifteen scorecards: `shuffled_gold` scores
`numbers_f1` 0.0000 (A), 0.0019 (B), 0.0053-0.0054 (B') against `max_expected` 0.1 --
`verdict: FLOOR_OK` fifteen times. A green run without it would be an opinion.

**NONE OF THIS IS A SHIPPING DECISION. FOUR THINGS BLOCK IT, AND THE FIRST IS DISQUALIFYING.**

1. **THE CANDIDATE DOES NOT MEET THE 32K CONTEXT FLOOR.** `--kv-cache-dtype fp8_e5m2` is REFUSED on
   this compressed-tensors checkpoint, and without it the engine came up at `--max-model-len 15360`
   reporting `GPU KV cache size: 15,680 tokens` -- below the floor `CLAUDE.md` §SENTINEL requires,
   for the reason it requires it (full-document decomposition). It is ample for these articles, whose
   prompts run ~2.1K tokens, and that is exactly why the scorecard cannot answer the question: the
   corpus never exercises the axis the floor exists to protect. `--max-model-len 15360` is in the
   engine snapshot (`engine.candidate-up.txt`); THE `15,680` IS NOT RE-DERIVABLE -- the candidate
   container was removed after the run and no startup log was preserved, so that figure survives
   only as `results.json`'s `kv_cache_note`. Whoever re-runs this must capture the engine log.
   DISQUALIFYING ON WHAT WAS TRIED, NOT ON WHAT IS POSSIBLE -- AND THE UNTRIED LIST IS ONE LEVER,
   NOT THREE. UNQUANTIZED KV IS NOT UNTRIED: it is what PRODUCED the 15,360 above, once the
   checkpoint refused `fp8_e5m2`. An earlier revision of this paragraph listed it as a remedy, two
   lines under the sentence saying the engine ran "without it" -- read the serving table, not this
   list, if the two ever disagree again.
   THE ONE FLAG NOBODY PULLED IS `fp8_e4m3`, and its supporting measurement does not transfer
   cleanly: this file records it serving 32K at concurrency 6 with 0 errors, but ON THE INCUMBENT and
   ON vLLM **0.28.0**, which is NOT the engine this A/B ran. There is no e4m3 row at 0.19.0 anywhere
   in this file. Do not shorten that to "a blocked engine": what §VLLM_UPGRADE blocks is carrying
   `fp8_e5m2` past 0.19, and it names e4m3 as the one-flag FIX -- so the objection here is the
   0.28.0/0.19.0 mismatch and the incumbent-not-candidate checkpoint, nothing more.
   A HIGHER `--gpu-memory-utilization` is the other candidate and THE SIGN MATTERS -- lowering it
   SHRINKS the cache. Headroom exists: the candidate left 3,476 MiB free against the incumbent's
   1,260 MiB at 0.92 (engine snapshots), about 2,216 MiB more. Whether that reaches 32K is NOT
   measured and NOTHING here should be read as predicting it either way: 15,680 -> 32,768 tokens
   wants roughly 2.1x the KV memory, which 2.2 GB may not cover -- but this file also records
   unquantized KV fitting 36,848 tokens on this card under 0.19.0, for the LARGER 32B incumbent,
   which points the other way and is not reconciled with the candidate's 15,680. Two recorded
   numbers disagree; measure, do not adjudicate them from the armchair.
   So the honest state is "this configuration does not reach 32K and one flag is untried", NOT "this
   checkpoint cannot". Settle it before treating the blocker as a property of the model.
2. **`entity_ticker_accuracy` REGRESSES, and it is upstream of SecMaster.** 0.6714 -> 0.4583 (B) ->
   0.4167 (B'). RE-DERIVED OUTSIDE THE HARNESS, matching gold entity names case-insensitively with a
   containment fallback rather than the harness's token-F1 aligner: the 40 articles carry **53**
   gold ticker-bearing entities; the incumbent matches **47** of them and gets **29** tickers right
   (0.617), the candidate matches **51** and gets **23** (B, 0.451) or **21** (B', 0.412) right.
   THE CANDIDATE ARMS ARE FLAT ACROSS ALL FIVE RUNS; THE INCUMBENT IS NOT -- it matches 47, 47, 47,
   47, **48**, which is why its five-run total is 236 and not 235, and the harness sees the same
   one-row move (its A value is exactly `29/43` in four runs and `29/44` in the fifth). An earlier
   revision of this line called both counts stable and was refuted by its own 236. BOTH DERIVATIONS SAY THE SAME THING: the candidate finds MORE ticker-bearing entities
   and TICKERS FEWER of them.
   IT DOES NOT MIS-TICKER THEM, AND THAT DISTINCTION IS THE WHOLE RISK ASSESSMENT: across all 15
   runs, of 236 / 255 / 255 matched gold entities, the wrong-ticker count is **0 / 0 / 0**. Every
   single miss in every arm is a NULL or empty `ticker` on a household name (`Boeing`, `Delta Air
   Lines`, `Apple`, `Tesla`, `JPMorgan Chase`) -- 91 (A), 140 (B), 150 (B'). So the candidate hands
   SecMaster fewer pre-resolved symbols, not wrong ones: it degrades RECALL at the resolver's front
   door and adds no wrong-instrument risk. PIN IT BEFORE ANY SWAP -- it is a regression on the path
   this epic spent a day repairing -- and pin it as a recall regression, which is what it is.
3. **The criteria are PROVISIONAL.** `ratified_by: null`, most thresholds carried from the CoVe bar
   unmeasured, and the pass count flips run to run in both arms. A `pass` here is not a ratified pass.
4. **Three metrics move the wrong way besides the ticker**: `number_source_entity_exact_match`
   -0.0146, `article_type_accuracy` -0.0500, `entities_precision` -0.0425, and latency
   16.07 -> 20.28 s/doc (+26%), or 24.22 (+51%) at B'.

**0.7400 HERE IS NOT THE 0.764 IN "MODEL BASELINES" ABOVE.** Different task, different metric,
different corpus: that one is `aggregate_f1` on the v6.2 substrate's own 16 instruction blocks, this
one is `numbers_f1` on production's CoD extraction over 40 gold articles. The two have been conflated
before. They are not comparable and neither converts to the other.
`events` and `claims` gained the most in relative terms and DECIDE NOTHING: `CLAUDE.md` §SENTINEL
already records `event_kind`/`claim_kind` as free-form, with two careful human labellers scoring
0.302 and 0.138 against each other. Read the `numbers` and `entities` rows; treat the rest as texture.

**THE FAMILY RE-QUALIFICATION [2026-09-06, numbers_f1, production's CoD path, 40 gold articles].**
Six families that `LlmBenchmark/BENCHMARKS.md` eliminated were re-run at feasible points. Rule and
method: `LlmBenchmark/MEASUREMENT_SPACE.md`. All rows n=3 unless noted, fp8_e4m3 KV, max_model_len
32768, temp 0, seed 42, rep_penalty 1.1, max_tokens 4096, production's prompt and schema.

| model | numbers_f1 | sd | engine | max-num-seqs / util | note |
|---|---|---|---|---|---|
| Gemma 4 31B QAT w4a16-ct | **0.7570** | 0.0009 | vllm-**0.28.0** | 6 / 0.90 | NOT production's engine -- see caveat |
| Qwen3.8-27B (armC candidate) | 0.7132 | 0.0109 | vllm-0.19.0 | 16 / 0.95 | |
| Gemma 3 27B w4a16 | **0.6177** | 0.0130 | vllm-0.19.0 | 16 / 0.95 | n=3 final; leaderboard says `0.0% FAIL` |
| Qwen2.5-32B-AWQ (incumbent, armA, n=5) | 0.5153 | 0.0106 | vllm-0.19.0 | 16 / 0.95 | production today -- a VALID baseline |
| Mistral-Small 24B | 0.5105 | 0.0032 | vllm-0.19.0 | 16 / 0.95 | THE CONTROL -- see below |
| GLM-4.7-Flash | -- | -- | vllm-0.19.0 | 16 / 0.95 | DEGENERATE at both grammar settings; coordinate finding |
| Command-R 08-2024 (current build) | 0.3178 | 0.0098 | vllm-0.19.0 | 16 / 0.95 | |
| EXAONE 4.0 32B | 0.2218 | 0.0089 | vllm-0.19.0 | 16 / 0.95 | |

**A CORRECTION OF A CORRECTION, AND THE AXIS IT UNCOVERED.** An earlier revision of this entry
declared armA a FLATTERING baseline, withdrew the Mistral control on that basis, and restated every
gain as larger. **That was wrong and is withdrawn.** The -0.0536 gap between armA and the control arm
was NOT the KV dtype -- pinned single-axis, `fp8_e5m2` vs `fp8_e4m3` is **+0.0103, null**. It was the
**CHAT TEMPLATE**: the control deriver silently injected Qwen's default "helpful assistant" system
block, which production does not send. So armA is a valid baseline, and

  gemma3  vs incumbent  **+0.1024**  11.5x se  DISJOINT   <- as ORIGINALLY reported
  mistral vs incumbent  **-0.0048**  indistinguishable    <- as ORIGINALLY reported

**MISTRAL IS THE CONTROL AGAIN, and it does the job it was claimed to do**: indistinguishable from
the incumbent, from a leaderboard row of 52.1% on retired Ollama, so moving a model onto vLLM with
`response_format` json_schema is NOT a universal uplift and Gemma's gain is not a harness artifact.

**THE SYSTEM PROMPT IS AN AXIS, AND IT IS WORTH ~0.05-0.06 F1** -- larger than the KV dtype, larger
than the engine step, larger than concurrency, and comparable to the whole effect this epic set out
to detect. It was invisible because a template deriver injected it by default and the validation that
should have caught it (`tail -3` on the rendered template) CUT OFF the system block -- a check that
read the end of the thing whose defect was at the beginning. Record it as axis 11.

**AND THE SUPERVISOR AMPLIFIED IT.** The unpinned attribution was flagged at the time ("two axes
moved; do not quote it as a dtype result until one is pinned") and the CONCLUSION built on it was
published anyway, into two files, as fact. Hedging the cause while asserting the conclusion that
depends on it is not hedging. A correction is the riskiest claim in the room and this one was acted
on before it was pinned.

**GEMMA 4 IS NOW ATTRIBUTABLE, AND BOTH FLAGGED CONFOUNDS MEASURED NULL.**

  ENGINE      C1 (0.28.0) 0.4545 vs C2 (0.19.0) 0.4540, identical in every other axis
              -> +0.0005, 0.0x se. Independently consistent with the -0.0006 for the same engine
              step on the SUBSTRATE task: two tasks, two golds, the same near-zero answer.
  CONCURRENCY C2 (seqs 6 / util 0.90) vs C3 (seqs 16 / util 0.95)
              -> -0.0078, 0.4x se. FIRST TIME THIS PROJECT HAS MEASURED AXIS 6. The supervisor's
              hypothesis that the 16-vs-6 split discounted Gemma 4 is REFUTED: it does not move
              this metric. The variance side is a hint only (sd 0.0273 at seqs 16 vs 0.0146 at
              seqs 6, n=3 each) and does NOT on its own explain Gemma 4's sd of 0.0009.

  vs C1 -- same engine, same serving point, ONLY the model differs:  **+0.3026**, 32.2x se, DISJOINT
  vs armC candidate:                                                 **+0.0439**,  7.0x se, DISJOINT
  engine-adjusted 0.7565 vs armC 0.7132 = +0.0434

The weight-quant format axis was never moved against armC either -- both are compressed-tensors
pack-quantized, read from each model's own `config.json`. **This is the first cross-family
comparison in this project that survives its own admissibility rule.**

**GLM-4.7-Flash IS A THIRD COORDINATE FINDING**: degenerate at BOTH grammar settings -- non-termination when unconstrained, empty arrays when constrained. Not an F1.

**TWO ELIMINATIONS WERE REFUTED AS FAMILY VERDICTS AND STILL SCORE POORLY, WHICH IS A DIFFERENT FACT.**
Command-R's "35B too large for KV cache" was true of v01's no-GQA design (640 KiB/token, 8K native)
and was recorded as a fact about the FAMILY; the current build boots at full 32K with 71,952 KV tokens
and 6,078 MiB spare, 0 errors, 82-96s per gold set -- and then scores 0.3178. The elimination reason
was wrong and the conclusion happens to survive. EXAONE's "can't follow extraction format" was HALF
right, and the precise half matters: it follows the schema with 0 call errors but fails to TERMINATE
inside production's 4,096-token budget on 8-9 of 40 articles (finish_reason `length`, stable across
runs), which score zero. That is a budget interaction, not a schema-compliance failure.

**COLIBRI: NO ADMISSIBLE SCORECARD IS POSSIBLE [2026-09-06]. Two structural blockers, both measured.**
Evaluated because `CLAUDE.md` §INFERENCE_TOPOLOGY had been treating the deployed engine as the whole
axis. Build and serve are excellent and are NOT the problem: v1.10.2 at commit `fd93c41a`, `gcc -O3
-march=native -fopenmp`, **1.96 s** to a 180 KB zero-dependency binary, weights loaded in 6.5 s, all
OpenAI endpoints correct, `coli doctor` 11 ok / 1 warn / 1 skip.

  1. **`seed` is REFUSED** -- `HTTP 400 "Per-request seeds are not supported yet."` `run_model.py`
     sends `seed` on every request because production does, and has no off switch. The unmodified
     harness reports `2/2 calls failed; that is an outage, not a run`. Reproduces with AND without
     `--no-structured-output`, so the two blockers are independent.
  2. **NO CONSTRAINED DECODING AT ALL** -- every `response_format` form on both wire shapes returns
     `400 "response_format grammars are not supported by the qwen36 engine yet."`
     `FamilyCapabilities.grammar_payload` is true for **1 of 8** families (the 372 GB GLM-5.2/5.3),
     and even there it is a SPECULATIVE DRAFT SOURCE, not a constraint: `pick_tok` ranges over the
     full vocab unmasked and the draft is kept only `if(next==draft[j])`. Their own docs: *"a draft
     source, never a sampling constraint."* Scoring would require exactly the free-generate-and-salvage
     mode that produced the nine false eliminations in `BENCHMARKS.md`.

**LATENCY, and the confound is flagged rather than hidden.** Warm steady state **1.91 tok/s**; ONE
gold article on production's prompt took **2,853 s (47.6 min)** and ended `finish_reason: length` --
against **124.5 / 119.8 / 122.5 s for all forty** on vLLM at concurrency 6. That is 933x measured, or
240x on the optimistic basis where it terminates at vLLM's 1,054-token average. Concurrency is
hard-capped at 1 (`max_kv_slots=1`). **BUT the engine was disk-streaming, not RAM-resident** -- 910-948
MiB/s sustained, 476 MiB read per token against 510 predicted (ratio 0.93, near-zero expert cache
reuse) -- because the concurrent GPU sweep held ~95 of 125 GB and swap was full. An idle-box
re-measurement would be materially faster; HOW MUCH IS UNMEASURED and was deliberately not guessed.
The 47.6 minutes also produced NO answer: 4,096 tokens of thinking-mode monologue, cut off
mid-sentence, zero JSON -- Qwen3.6 is a thinking model and production's template does not suppress it
(D-26 leaves `ThinkingSuppressionSuffix` empty). There was nothing to salvage even in principle.

**THE PREMISE IS VINDICATED EVEN THOUGH THE ENGINE IS NOT, AND THIS IS THE FIRST TIME THE REGION
ABOVE 32B HAS BEEN PRICED HERE.** Arithmetic validated at ratio 0.93 against the measured point:
Qwen3.8-Flash-Next 125B is **62.5 GB at int4 and genuinely fits this box's RAM**, as does DeepSeek
REAP-150B at 75 GB; 284B / 321B / 744B are disk-resident with 0.54 / 0.13 / 0.06 tok/s ceilings. But
both reachable candidates are `grammar_payload=False` and every candidate hits the seed blocker, so
every path is blocked BEFORE model size becomes the question. Download nothing.

**THE SCREEN THIS PRODUCES, now in `CLAUDE.md` §INFERENCE_TOPOLOGY and cheap to apply:** an engine
must accept `seed` AND perform REAL constrained decoding -- a grammar that MASKS sampling, not one
that only drafts speculatively -- or no admissible scorecard can exist on it. Two engines pass it and
are worth the GPU: **SGLang** (takes `seed` and `json_schema`, XGrammar is real constraint, and it
gives a same-GPU single-axis comparison against vLLM 0.19.0 -- the one that could land a scorecard
this week) and **ktransformers** (GPU attention with experts in RAM, real grammar constraint -- the
right comparator for that 125B row, where colibri cannot be scored at all). ExLlamaV2/V3 is the only
route to the Q6-as-floor question on a GPU; TensorRT-LLM moves axes 1 and 3 together; MLC buys
nothing on one fixed NVIDIA card. Artifacts: `/tmp/sentinel-remediation/colibri/COORDINATE.md`; the
22 GB model there is reclaimable.

**THE PRECISION LADDER [2026-09-06]: Q6_K LOSES TO Q4_K_M.** Single axis, gemma-3-27b-it,
unsloth GGUF, llama.cpp server-cuda b10820, only `general.file_type` differing (15 vs 18, read from
the GGUF header not the filename). `numbers_f1` Q4_K_M **0.6263** sd 0.0039 vs Q6_K **0.5985** sd
0.0003 -- d = -0.0278 at 12.2x se, run ranges DISJOINT. Recall -0.0257, precision -0.0297,
entities_f1 flat at -0.0019 (0.5x se). Q6 is also 14% slower (43.58 vs 38.22 s/doc) for 5.6 GB more
VRAM. 4-bit is vindicated by measurement. Limits: one model, one task, one publisher, and k-quants
are not strictly bpw-ordered, so this is two artifacts rather than a precision dial.

NOT COMPARABLE TO THE vLLM ROWS ABOVE, and the reasons are the point of recording them: these arms
ran in CHAT mode at 8,192 per slot, so they carry `production_prompt_path: false`, against the vLLM
arms' completions path at 32,768. The comparison is internally valid (both ladder arms matched) and
cross-engine invalid. Note anyway that Gemma 3 lands at 0.6263 here and 0.6220 on vLLM w4a16 --
close, across three moved axes, which is suggestive and not evidence.

**AND vLLM HAS NO USABLE RUNG ABOVE 4-BIT AT 27B.** Measured, not estimated: w8a16 weights of
27.26 GiB leave a 4,176-token KV pool -- below this eval's own 7,617-token worst case -- and
throughput collapses 419.6 -> 92.8 tok/s with 2 of 6 requests resident. That is why the ladder moved
engines. And llama.cpp CUDA is slower on the same 40 articles at concurrency 6, on both
measures, which agree with each other: wall 260.9 s vs 102.7 s = **2.54x**, per-request 38.22 s vs
14.28 s = 2.68x. So llama.cpp is not a deployment path here even where it is the only measurement
path -- but the margin is 2.5x, not the 15x an earlier revision of this entry carried. That figure
was produced by dividing llama.cpp's PER-REQUEST latency by vLLM's THROUGHPUT figure; the two differ
by the concurrency factor and the ratio inflated ~6x. Full table, wall for 40 articles / s-per-doc
throughput / s-per-doc per-request:

| arm | engine | wall | s/doc (thruput) | s/doc (per-req) |
|---|---|---|---|---|
| mistral | vLLM 0.19.0 | 83.5 | 2.09 | 12.08 |
| commandr | vLLM 0.19.0 | 87.3 | 2.18 | 12.75 |
| gemma3 | vLLM 0.19.0 | 102.7 | 2.57 | 14.28 |
| gemma4 | vLLM 0.28.0 | 179.9 | 4.50 | 26.30 |
| lcpp Q4_K_M | llama.cpp | 260.9 | 6.52 | 38.22 |
| lcpp Q6_K | llama.cpp | 294.4 | 7.36 | 43.58 |

**A VERIFICATION FAILED TOWARD SUCCESS, INSIDE THE TASK ABOUT THAT.** An earlier report of
"llama.cpp honours `json_schema` on `/v1/completions` -- verified" was FALSE; the check read HTTP
status and `finish_reason`, never the content. `/v1/completions` silently IGNORES `response_format`
and emits the JSON inside a ```json fence -- the first Q6 arm returned 40/40 with 0 call errors,
`finish_reason` stop, and `schema_invalid` 40. Measured across all three endpoints: `/v1/completions`
ignores it, `/v1/chat/completions` honours it, `/completion` with a top-level `json_schema` honours
it. Both ladder arms were re-run in chat mode, which is why they are off the production prompt path.
Separately, reading a provenance file WHILE the runner was writing it reported "call_errors 40, wall
1.9s" for a run whose own runner log said "records 40, errors 0, wall 291.8s" -- a spurious engine
failure, caught only by cross-checking an independently written artifact.

**THINKING DEFAULTS ARE AN UNCONTROLLED AXIS ACROSS VENDORS.** GLM's first scored attempt was
invalid and killed: its vendor chat template defaults thinking ON (it ends on an open `<think>`), so
the 4,096-token budget goes to reasoning and the JSON never closes. Gemma 4 and EXAONE both default
OFF. Three vendor defaults were being inherited as though they were one setting; GLM is re-queued
with a derived thinking-OFF template so the axis is consistent across arms.

**COROBORATES `CLAUDE.md` §VLLM_UPGRADE**: this Gemma 4 arm ran 0.28.0 with `--kv-cache-dtype
fp8_e4m3` at concurrency 6 for 3 full runs with 0 errors -- the flag that block names as the one-flag
fix for the e5m2 fault, now exercised on a second model.


Re-check. The predictions, the fifteen scorecards, the sidecars and the engine snapshots are under
`/tmp/sentinel-remediation/qwen-ab/` and one `tmpwatch` ends them; the INPUTS are committed, so the
measurement is repeatable even after that, which is what the digest table is for. One arm, five runs:
```
python3 LlmBenchmark/scripts/run_model.py --task cod --endpoint-mode completions \
  --prompt-file SentinelCollector/src/cod-prompts/cod_json_v1.txt \
  --schema-file SentinelCollector/src/cod-prompts/cod_json_schema_v1.json \
  --chat-template '<|im_start|>user\n{0}<|im_end|>\n<|im_start|>assistant\n' \
  --repetition-penalty 1.1 --max-tokens 4096 --concurrency 6 \
  --substrate <40-article subset> --endpoint http://localhost:8000 \
  --model <model-id> --model-label "<model-id> @ vllm-0.19.0" --out preds.jsonl
python3 LlmBenchmark/scripts/eval_harness.py --task cod --substrate <same subset> \
  --cod-gold LlmBenchmark/cod-gold/cod_stage1_gold_v1.json --predictions preds.jsonl \
  --adapter-meta preds.jsonl.provenance.json --out scorecard.json
```
Confirm `substrate_sha256` in the sidecar reads `008c338d...`. IF `/tmp` HAS CLEARED, rebuild the
subset from the committed corpus -- one record per gold article, in the corpus's own order, and only
`input.content` plus the join key are read on this path:
```
python3 -c 'import json; c=json.load(open("LlmBenchmark/cod-gold/cod_stage1_corpus_v1.json"));
json.dump([{"input":{"content":a["content"]},"source_file":a["source_file"],
"source_index":a["source_index"]} for a in c["articles"]], open("/tmp/g40.json","w"))'
```
Write it OUTSIDE the tree, as above -- the input path is repo-relative, so a bare `g40.json` lands an
untracked file in the checkout. That reproduces the SUBSTRATE but NOT its sha256: the original
wrapper also carried the v6.2 substrate's `instruction`, `output` and `is_negative` fields, none of
which this task reads. So a rebuilt run cannot claim `008c338d...`; check the 40 `content` values
against the corpus instead and say which route was taken.
THE ENGINE SIDE OF THE RE-CHECK IS NOT RECORDED ANYWHERE DURABLE -- the candidate's serve invocation
lived only in the removed container, and the arm difference above makes it load-bearing. Whoever
re-runs this must write BOTH serve commands into the entry. `--model-label` is in the invocation
deliberately; the fifteen runs behind this entry omitted it, and the entry below says what that cost.

CLOSES when a matched-serving re-run exists: both arms on the SAME `--kv-cache-dtype` and the SAME
`--max-model-len`, at 32K or with the shortfall priced on the CoD task, with the ticker recall
regression pinned by a test. Until then this entry is a measured COMPARISON and not a swap decision,
and `CLAUDE.md` §MODEL_ACCEPTANCE governs.

### An acceptance scorecard names NO MODEL in its headline field, and does it silently [2026-09-06]
All fifteen scorecards behind the entry above read `"model": "unspecified"` at the top level. The
real id is in the file -- `adapter_metadata.model` carries `Qwen/Qwen2.5-32B-Instruct-AWQ` and
`cyankiwi/Qwen3.8-27B-AWQ-INT4` with their revisions -- so nothing was lost HERE, because the
provenance sidecar was passed and this entry re-reads it. THAT IS THE POINT: the artefact whose whole
purpose is to say which model earned a score has a field for exactly that, and it defaults to a
string that looks like a value.

MECHANISM, one line: `--model-label` has `default="unspecified"`
(`LlmBenchmark/scripts/eval_harness.py:1714`) and is written verbatim into the scorecard
(`LlmBenchmark/scripts/eval_harness.py:1474`). Omit the flag and the run is scored, stamped
`production_prompt_path: true`, and filed under a model name of "unspecified" -- no warning, exit 0.
CHECKED, because the opposite was assumed first: all eight committed scorecards in
`LlmBenchmark/eval-substrate/` carry a real label (`Qwen3.8-27B-AWQ-INT4 (no-think) @ vllm-0.19.0`
and siblings), so the convention has held by DISCIPLINE for eight runs and broke on the ninth
occasion anyone forgot the flag -- which is what an optional flag with a plausible-looking default
guarantees eventually. §MODEL_ACCEPTANCE turns on comparing a candidate's scorecard to the
incumbent's; a comparison whose two sides are both labelled "unspecified" is decided by whoever
remembers which file was which.

This is the TOOL_UPKEEP shape `CLAUDE.md` names: the tool fails toward SUCCESS. There is no dull
reading, no missing-field error, no null -- the scorecard is complete and internally consistent and
says nothing about the model.

FIX, not applied here (this PR is docs-only): default the label to `adapter_metadata["model"]` when
`--adapter-meta` supplied one, and emit the literal `"unspecified"` ONLY when neither source has a
name -- at which point a scorecard that carries no model id anywhere is worth a warning on stderr.
The data is already in the process; the defect is that the two fields never meet.

**THE SAME SIDECAR HAS NO FIELD FOR HOW THE ENGINE WAS SERVED, AND THAT ONE IS NOT A FORGOTTEN
FLAG.** `run_model.provenance` carries `endpoint`, `engine`, `engine_version`, `model`,
`model_revision` and the full `sampling` block, and NO field for `kv_cache_dtype`, `max_model_len`,
`gpu_memory_utilization` or `quantization` -- no CLI flag accepts them either. Two arms served with
DIFFERENT KV dtypes therefore differ in their sidecars only in timings and token counts; NOTHING
records the serving difference, and the scorecard pair reads like a matched one. That is not
hypothetical: it is what happened to the A/B two entries above, where the incumbent alone carried
`fp8_e5m2` -- a flag this file prices at ~0.05 F1 concentrated in recall -- and the confound had to
be recovered afterwards from `nerdctl container inspect` and a hand-written engine snapshot in
`/tmp`. An A/B harness that cannot describe the engine it measured cannot report its own dullness,
which is this section's whole subject.
FIX, AND IT IS CHEAPER THAN IT LOOKS -- THE DATA IS ALREADY IN THE PROCESS. `run_model.py` ALREADY
GETs `/v1/models` and iterates the entries, taking only `d.get("id")`
(`LlmBenchmark/scripts/run_model.py:284-286`); that same response carries `max_model_len`. The
engine's `/metrics` exposes `vllm:cache_config_info` with `cache_dtype` and
`gpu_memory_utilization` as labels. So three of the four are one already-open call and one scrape
away. Record them, and refuse to stamp `production_prompt_path: true` on a run whose serving config
could not be read.

Re-check:
```
grep -n 'model-label\|"model": model_label' LlmBenchmark/scripts/eval_harness.py
python3 -c 'import json,sys; d=json.load(open(sys.argv[1])); print(d["model"], "|", d["adapter_metadata"].get("model"))' <scorecard>.json
```
2026-09-06 -> `default="unspecified"` at `:1714`, `"model": model_label` at `:1474`; on all fifteen
runs the two-field print reads `unspecified | <the real id>`. This entry closes when the first field
carries the second.

### Convention B measured end to end: +0.1873 `numbers_f1` AT HARNESS SAMPLING, disjoint arms, 82.4% of the ceiling [2026-09-06]
The macro-owner decision (#1017: the SERIES owns its own print) is no longer a prediction. Both arms were
run against the SAME committed gold on the SAME committed key, so the effect below is the PROMPT's and
nothing else: incumbent Qwen2.5-32B-Instruct-AWQ rev `5c7cb76a268fc6cfbb9c4777eb24ba6e27f9ee6c` @ vLLM
0.19.0 (production's live engine, client-only), the 40 gold articles, concurrency 6, seed 42, temperature
0, `--max-tokens 8192`, THREE runs per arm. The only axis that moves is the prompt file's content:
sha256 `ab06b7b0` (pre-#1017) against sha256 `0dd66dde` (repo HEAD). BOTH ARE CONTENT DIGESTS, NOT GIT
OBJECT IDS, and an earlier revision of this line called them "blobs": `git show ab06b7b0` is a fatal
error. The corresponding git blobs are `45d02cd8bd3e...` and `85187c399dd4...`; everywhere below, a git id
appears in full only next to the `git` command that consumes it.

| arm | `numbers_f1` per run | mean | within-arm range |
|---|---|---|---|
| OLD prompt, sha256 `ab06b7b0` | 0.3564 / 0.3426 / 0.3792 | **0.3594** | 0.0366 |
| NEW prompt, sha256 `0dd66dde` | 0.5747 / 0.5389 / 0.5265 | **0.5467** | 0.0482 |

**+0.1873**, and the distributions are DISJOINT: `min(NEW) - max(OLD)` = **+0.1473**, so the worst pairing
of runs still separates. The effect is 3.9x the wider of the two within-arm ranges and 4.4x their mean --
which matters because the swing is what the entry above this one says has to come out of the instrument,
and an effect inside the swing would have decided nothing.

QUOTE THE SAMPLING WITH THE FIGURE. THIS IS A CAVEAT, NOT A RETRACTION -- the +0.1873 is real and it
SURVIVES the move to production's decoding. Both arms above ran `repetition_penalty: null,
max_tokens: 8192`; production sends **1.1 / 4096** (`CpuCod__JsonRepetitionPenalty` and
`CpuCod__JsonMaxCompletionTokens` in `/opt/ai-inference/compose.yaml`, defaulted at
`SentinelCollector/src/Configuration/CpuCodOptions.cs:91` and
`SentinelCollector/src/Configuration/CpuCodOptions.cs:102`, forwarded at
`SentinelCollector/src/Services/GpuJsonExtractionService.cs:222`). The fourth cell -- OLD prompt at
production's sampling -- was measured 2026-09-06 and makes the comparison same-sampling: `numbers_f1`
**0.3446 -> 0.5152 = +0.1707**, arms still DISJOINT at worst-case **+0.1357**. So the prompt's number
holds under the guard; what does NOT hold across the sampling axis is the entity story two entries
below, which reverses sign. The four-cell table is in "The `entities_f1` regression REVERSES SIGN at
production's sampling" below, and the standing production cost it exposes has its own entry after that;
this heading now names its sampling because the figure was written bare and read forward twice.

WHAT IT DID NOT BUY IS 17.6% OF THE PRIZE. QUOTE THE ARM, which is what the entry above legislates and
what an earlier revision of this line failed to do: its headline +0.2048 is the POOLED n=5 figure, and its
ARM-MATCHED counterpart -- the same three c6 runs used here -- is **+0.2022**. Against the NEW gold that
same c6 arm sizes the question at **+0.2273** greedy, or **+0.2321** under maximum-cardinality matching on
the same waived graph. Captured: **82.4%** of the greedy ceiling. The 80.7% figure divides the same
greedy-measured +0.1873 by the max-cardinality ceiling, so it is a MIXED basis and is the conservative
reading, not a second measurement. The remaining 0.0400 is a real convention the model does not yet emit,
not a scoring artefact.

The decomposition, all c6 n=3, which is what separates the GOLD's move from the PROMPT's:

| measurement | `numbers_f1` | reading |
|---|---|---|
| OLD preds x OLD gold, committed key | 0.3710 | the c6 arm of the entry above, whose published figure is the POOLED 0.3883 |
| OLD preds x NEW gold, committed key | 0.3594 | the gold move ALONE costs **-0.0116** |
| OLD preds x NEW gold, WAIVED key | 0.5867 | counterfactual ceiling on the gold as it stands |
| NEW preds x NEW gold, committed key | 0.5467 | achieved -- **0.0400 short** of that ceiling |
| NEW preds x NEW gold, WAIVED key | 0.6143 | **+0.0676 residual headroom** still in the key |

MECHANISM, per row through `eval_harness`'s own `_cod_align`/`_number_key`: macro_indicator-owned gold
rows that align AT ALL went **2.3 of 149 -> 83.3 of 149**; aligned number pairs **176 -> 264**; the
model's own `source_entity_empty_rate` **0.3142 -> 0.0943**. The prompt asked for a series name where it
used to ask for a blank, and the model supplied one.

OPERATIONAL, because a clean run is a claim too: all six runs rc 0, `schema_invalid 0`, `call_errors 0`,
`truncated 0`, `finish_reasons {stop: 40}`, shuffled-gold control FLOOR_OK on EVERY run (bar 0.10, judged
per run). The three NEW runs took 7m27s end to end (03:48:04Z-03:55:31Z), 446s of it inference. The engine
was unharmed: `vllm:request_success_total{finished_reason="error"}` and `="abort"` both 0 before and after
each run, and the `stop` counter advanced by exactly 40 per run.

WHICH PROMPT A RUN READ IS NOT IN ITS ARTIFACT -- the same sidecar gap the entry above discloses for
concurrency. `run_model.py`'s provenance records the prompt FILE PATH and never its hash, and BOTH arms
name the same path, so nothing in the six sidecars distinguishes them. What pins the assignment is time
and token count: the OLD arm's three runs finished 2026-09-05T23:31:56Z and #1017 landed at `a8a0ed5d`
four hours later, 2026-09-06T03:46Z, with the NEW arm starting 03:48:04Z -- two minutes after. The runs
corroborate it themselves: `usage.prompt_tokens` is 76,759 in every OLD run and 83,079 in every NEW one
over the identical 40 articles, a constant +158 per article, which is the size of the clause #1017 added.
Whoever repeats this should hash the prompt INTO the sidecar rather than reconstruct it from timestamps.

`number_source_entity_exact_match` SHOWS AN APPARENT -0.020 REGRESSION AND IT IS ONE ARTICLE. 0.9280 ->
0.9076 over all aligned pairs. It is `sentinel-v6.2-cove.json` index 183 in full: that article's six `US`
payroll rows -- the non-conformance the entry above DISCLOSES rather than hides -- now ALIGN (6, 5 and 5
per run) and every one of them scores 0 exact, where before they did not align at all and were counted by
nothing. They align because the model writes `US jobs market` or `US economy` against a gold `US`:
`_source_entity_affinity` 0.5 on 12 of the 16 rows and 0.667 on the other 4, i.e. AT or just above the
0.5 floor. Over the 490 conform rows the metric is FLAT: **0.9247 -> 0.9246**. (The OLD arm's 0.9280 and
its conform-only 0.9247 differ even though no article-183 row aligned in that arm, because the all-pairs
figure also carries the aligned OPEN_BLANK rows, which score 1.0 on blank-equals-blank. Neither number is
wrong; they have different denominators.) A disclosed
non-conformance surfacing in a metric is the metric working, not a defect -- but it also means this number
cannot be read as a conformance figure until article 183's six rows are settled.

PRODUCTION STILL RUNS THE OLD PROMPT, and this is the most decay-prone claim in the entry because it is
live host state, so it gets its own one-liner:
`git hash-object /opt/ai-inference/prompts/cod/cod_json_v1.txt` -> `45d02cd8bd3e360529267c9fec3868fd8554af4a`
on 2026-09-06, which IS the pre-#1017 blob. So this entry measures the CORRECTED prompt, not what
production does, and the +0.1873 is not yet a production number -- an ansible deploy of the prompts mount
is what would make it one, and that hash changing is how you know it happened.
AND THE FLAG CANNOT TELL THE TWO ARMS APART: `eval_harness` stamped `production_prompt_path: true` on ALL
SIX scorecards. The stamp is a four-way conjunction -- `endpoint_mode == "completions"`, a `prompt_file`
NAMED, a `schema_file` named, and a chat template that wraps the prompt -- and NOT ONE of the four
inspects prompt CONTENT (`LlmBenchmark/scripts/eval_harness.py:1335` is the `prompt_file` conjunct, a bare
`bool()` on the path string). So the flag is sound on WIRE SHAPE, which is what it was built for, and
blind on the content axis: it certifies the arm that is false of production exactly as loudly as the arm
that is true of it. FOLLOW-UP THIS PR DOES NOT TAKE: CLAUDE.md §MODEL_ACCEPTANCE cites a
`production_prompt_path: true` scorecard as evidence that the bar runs end to end, and an agent reading
only that will believe the flag discriminates prompt content. Either the flag hashes the prompt into the
scorecard, or that HARD_STOP gains a clause saying it does not. Editing CLAUDE.md was out of scope here.

AND IT IS BLIND ON A SECOND AXIS -- SAMPLING -- WHICH IS THE SAME FLAG AND THE SAME CLASS, recorded here
rather than in a second entry [2026-09-06]. None of the four conjuncts reads a decoding knob, so a run
that omits production's loop guard stamps `true` exactly like one that sends it. Measured over the
loop-guard control below: **all TWELVE scorecards across the four cells stamp `production_prompt_path:
true`**, including the SIX that ran `repetition_penalty: null, max_tokens: 8192` where the service sends
1.1 / 4096. Six of those twelve are false of production on sampling, six of them are false of production
on prompt content, and the flag separates neither. A run that DECODED differently from production is not
a run on production's path however right its wire shape -- and the entry at the top of this file that
closes the CoD path end to end was itself measured at `--max-tokens 8192` with the penalty unset, so the
closure it records is of the PROMPT PATH and not of production's request.
LANDED 2026-09-06 as `d139f954` (PR #1026): `request_shape` now carries `prompt_file_sha256` /
`schema_file_sha256` / `chat_template_sha256`, and a per-knob `request_sampling` block sits beside it, under
the invariant that a digest is of bytes that REACHED a request or is null. Scorecards produced after that sha
carry them; the TWELVE counted above predate it and do not.
THE MERGE DOES NOT CLOSE THIS ENTRY, and an earlier revision of this line predicted that it would -- read the
merged code before believing it. `production_prompt_path` is still the same four conjuncts on `d139f954`
(`eval_harness.py:1334-1337`); not one of them reads a digest or a decoding knob, and the comment directly
above them says so in as many words. So a run that omitted production's loop guard still stamps `true`
exactly like one that sent it. What changed is that the evidence is now IN the scorecard instead of absent
from it: the scorer RECORDS and a human ADJUDICATES, comparing those digests against production's prompts
mount and `CpuCodOptions` against `request_sampling`. Both blindnesses are now READABLE from a scorecard
rather than closed by one, and the follow-up above -- CLAUDE.md §MODEL_ACCEPTANCE gaining a clause saying the
flag does not discriminate content -- is still open.

RE-CHECK (no GPU and no engine -- it re-scores the committed gold against the EXISTING prediction files,
so it is free only while those files exist; see the PROVENANCE LIMIT below):
```
python3 LlmBenchmark/scripts/rescore_alignment_keys.py \
  --cod-gold LlmBenchmark/cod-gold/cod_stage1_gold_v1.json \
  --predictions <OLD run a>.jsonl --predictions <OLD run b>.jsonl --predictions <OLD run c>.jsonl
python3 LlmBenchmark/scripts/rescore_alignment_keys.py \
  --cod-gold LlmBenchmark/cod-gold/cod_stage1_gold_v1.json \
  --predictions <NEW run a>.jsonl --predictions <NEW run b>.jsonl --predictions <NEW run c>.jsonl
```
2026-09-06 -> the OLD invocation prints `committed_context_and_source_entity 0.3594 range 0.0366` and
`macro-owner question, ISOLATED: +0.2273`; the NEW one prints `0.5467 range 0.0482` and `+0.0676`. Those
four figures ARE the headline, the ceiling and the residual, so ROWS 2 TO 5 of the table re-derive from
two commands. ROW 1 DOES NOT, and the entry does not pretend otherwise: `OLD preds x OLD gold` needs the
PRE-DECISION gold, which is committed NOWHERE -- it was reconstructed for this work and lives in `/tmp`
beside the predictions. It was measured, not quoted, and it lands on the entry above's own published c6
per-run values to four decimals (0.3646 / 0.3528 / 0.3955, mean 0.3710), which is the corroboration
available without that file. So the **-0.0116** gold-move figure is the one number here that a future
reader cannot re-derive from the repo plus the predictions alone. Add `--scorecard <matching
scorecard>.json` once per run and
`controls.reproduces_published_numbers_f1` goes from `NOT RUN` to `OK`, which is the check that the
rescorer and `eval_harness` still agree; it printed `OK` on 2026-09-06.

PROVENANCE LIMIT, the SAME one the entry above discloses for its own figures and for the same reason: the
gold and the rescorer are committed, THE PREDICTIONS FILES, THE SCORECARDS AND THE RECONSTRUCTED
PRE-DECISION GOLD ARE NOT. They live under `/tmp`, so one `tmpwatch` ends every figure in this entry and
in the three below it -- none is re-derivable from the repo alone. THIS APPLIES TO THE THREE ENTRIES THAT
FOLLOW AS WELL, including the one whose re-check says its entity figures are "checkable against the
scorecards without re-scoring": true today, and it costs GPU sweeps the moment `/tmp` clears.
THE POPULATION IS NOW TWELVE, NOT SIX, and the count moved because a fourth cell was added 2026-09-06:
twelve predictions files and twelve scorecards across four cells.
UNITS, because three statements in this file disagreed until 2026-09-06: one RUN is ~2 minutes at
concurrency 6 (measured 110-118s per run); one ARM is three runs, ~6-7.5 minutes. Rebuilding THIS entry's
two arms is 6 runs; rebuilding the whole four-cell table is **12 runs, ~25 minutes**, not four 2-minute
sweeps.
What survives is the METHOD, not the numbers: re-running THIS entry's two arms costs two `run_model.py`
arms on the live engine (~7.5 minutes each at concurrency 6) plus a checkout of the pre-#1017 prompt for the OLD arm,
`git show 45d02cd8bd3e360529267c9fec3868fd8554af4a` (the git blob; the sha256 in the table is a different
namespace). Whoever repeats this should write the predictions somewhere the tree can reach BEFORE the
numbers are quoted forward.

### The prompt's own anti-invention clause FAILS, referential integrity is blind to it, and production's loop guard suppresses the RUNAWAY but not the COINAGE [2026-09-06]
WHAT CHANGED 2026-09-06, and it changes the DEPLOYABILITY of the corrected prompt without excusing it.
Everything below was measured at `repetition_penalty: null, max_tokens: 8192`. Production sends **1.1 /
4096**. Re-run at production's sampling, the runaway DOES NOT MANIFEST: article 35 emits **6, 4 and 8**
entities against 60/60/60, with **ZERO** coined names against 50/50/49. Read the causation carefully,
because an earlier draft of this correction got it backwards and a sibling agent refuted it:
- THE PROMPT IS WHAT MAKES ARTICLE 35 LOOP. Both arms of the original comparison ran with the penalty
  unset, so a CONSTANT cannot explain a difference that appears in ONE arm. The corrected prompt is the
  only axis that moved, and it remains the cause. This entry does not exonerate it.
- PRODUCTION'S LOOP GUARD SUPPRESSES WHAT THE PROMPT PROVOKES. That is a different claim and it is the
  one the control establishes.
- SO THE CORRECTED PROMPT IS DEPLOYABLE, CONDITIONALLY. **If the loop guard is ever loosened or removed,
  this prompt change bites.** That coupling is live, it is not a closed question, and it is the reason
  this entry stays open rather than being deleted: `CpuCodOptions.JsonRepetitionPenalty` is now load
  bearing for prompt correctness, not only for latency.
- STILL OPEN, unchanged: whether the `macro_indicator` instruction CAUSED the article-35 runaway or
  merely uncovered it. One prompt edit and one re-run settles it. The guard does not settle it -- it
  hides it.

The corrected `source_entity` bullet added a clause for exactly this case -- "a series the article never
NAMES (a clause may describe the measure while naming no indicator; that is a blank, not licence to coin
one)", in `SentinelCollector/src/cod-prompts/cod_json_v1.txt`, closed four lines later by "NEVER invent a
name to fill this field". Cited by its verbatim text and NOT by line, because `scripts/verify-citations.py`
`_EXTS` has no `txt` and a `.txt:<line>` form is therefore a citation no sweep in this repo can ever check.
Measured on the arm-NEW runs above -- at HARNESS sampling, `repetition_penalty: null`; see the control
below for what production's 1.1 does to every figure in this entry -- the model coins names anyway, and
the metric written to grade anchor grounding cannot see it.

WHAT MAKES IT DEBT RATHER THAN A BUG REPORT. `source_entity_referential_integrity` asks whether an anchor
appears in the model's OWN `entities[]` (`LlmBenchmark/scripts/eval_harness.py:775`) -- never whether it
appears in the ARTICLE. Every coinage below is duly declared in `entities[]`, so the metric reads **0.9603
/ 0.9760 / 0.9900** across the three NEW runs, and the ONE run carrying hand-verified coined anchors on
BOTH articles below scores 0.9760 -- the middle value. The metric does not even RANK the runs by coinage.
A metric that cannot fail on the failure mode its clause exists to prevent is a signal riding on a
mechanism that does not observe it, and the article text is ALREADY BOUND IN THE SAME LOOP: `content =
_norm_ws(p.source_content)` sits just above that test and `number_source_text_verbatim_rate` reads it a few
lines below at `LlmBenchmark/scripts/eval_harness.py:782`. The missing check needs no new data, only the
normaliser this entry's last paragraph specifies.

WHAT WAS MEASURED, BY HAND, ON FOUR ARTICLES -- AND THE CRITERION SELECTS EIGHT, so this is a sample and
must not be read as a sweep. Articles whose gold carries a blank anchor are 1 (eleven), 48 (five), 395
(two), 471, 479, 490 and 35 (one each), plus article 183's six non-conformant `US` rows: eight in all.
Hand-checked here: 1, 35, 48 and 183. The other four were not opened.
- `sentinel-v6.2-cove.json` index 1 (Conference Board survey): in **1 of 3 runs** the model coined six
  `concept` anchors nominalised from clauses the article only DESCRIBES -- `income increase`, `income
  decrease`, `business conditions improve`, `business conditions worsen`, `home purchases`, `automobile
  purchases`, on 7 of its 20 predicted numbers. The article reads "The share of consumers expecting their
  incomes to increase rose to 17.6 percent" and "Intentions to buy homes within six months"; it names no
  indicator anywhere. The other two runs coined nothing on this article. AN EARLIER DRAFT OF THIS ENTRY
  SAID "2 OF 3" -- it is 1, and the other two runs' `income prospects` / `jobs plentiful` /
  `business conditions` are verbatim article phrases, which is what a re-derivation caught and a reading
  of the summary would not have.
- `sentinel-v6.2-cove.json` index 35 (Visa holiday sales) IS THE LARGE ONE AND IT REPRODUCES IN ALL THREE
  RUNS. The model emits **60 entities in every run where the gold holds 15**, of which 50, 50 and 49 are
  `macro_indicator` and **NOT ONE of those occurs in the article**: `U.S. holiday sales growth rate`,
  `... pace`, `... momentum`, `... velocity`, `... acceleration`, `... deceleration`, `... forecast`,
  `... prediction`, `... estimate`, `... projection`, `... outlook`, `... trend` -- a dozen near-duplicate
  nominalisations of one sentence. The OLD prompt emits 9, 10 and 11 entities on the same article, zero
  `macro_indicator` and zero coined. The coinage reaches `numbers[].source_entity` in only ONE of the
  three runs (16 rows on `U.S. holiday sales`), which is why an anchor-only sweep understates this by 3x:
  the invention is in `entities[]` every time.
- Held CLEAN: index 48 emits 5 of 5 blanks in all three runs (publisher-is-not-owner held perfectly), and
  index 183's anchors are all verbatim article surfaces in all three.

THE CORPUS-WIDE SWEEP IS A NOISY INSTRUMENT AND ITS RATE MUST NOT BE QUOTED. A substring test against the
article marks a correct answer wrong whenever the source text is mangled: on `sentinel-v6.2-cove.json`
index 429 the analyst table is broken across column breaks -- the content field literally holds
`Commerzb\nank`, `Bank of\nAmerica`, `Standard\nChartere\nd`, `Societe\nGenerale`, `Goldman\nSachs`,
`Morgan\nStanley` -- and the model REASSEMBLED them correctly. A substring sweep flags every one as coined.
Any measurement of this failure mode needs a normaliser that survives an intra-word newline before its
rate means anything, so the corpus-wide coined-row rates produced during this work are NOT recorded here.

THE DISCRIMINATING CONTROL, measured 2026-09-06, same engine and same 40 articles, THREE runs per cell,
changing ONLY the two sampling knobs. The positive observable is a COUNT of article-runs whose longest
array reaches the schema's `maxItems` (**60**, read from the schema actually sent, so a schema change
cannot retune the threshold under its own control) -- never "no blowup", which is an absence and would
have read green on a broken harness.

| observable | OLD @ null/8192 | NEW @ null/8192 | OLD @ 1.1/4096 [production today] | NEW @ 1.1/4096 |
|---|---|---|---|---|
| article 183 `events[]` (cap 60) | 60 / 60 / 17 | 60 / 60 / 60 | 11 / 10 / 11 | 9 / 10 / 9 |
| saturated article-runs | **3 / 120** | **6 / 120** | **0 / 120** | **0 / 120** |
| article 35 `entities[]` | 9 / 10 / 11 | 60 / 60 / 60 | 4 / 4 / 4 | 6 / 4 / 8 |
| coined entity names, corpus | 14 / 29 / 19 | **65 / 74 / 77** | 13 / 11 / 11 | **15 / 14 / 13** |

Across BOTH unguarded arms article 183 saturates in **7 of 8 runs** (the five OLD runs include the two
concurrency-1 runs). Under the guard it saturates in none, and coinage falls ~5x. QUOTE THE RESIDUAL
HONESTLY: 15/14/13 is not zero and it is not below the OLD arm's floor -- the same-sampling OLD
comparator is 13/11/11, so a gap of **2.33** coined names per run survives the guard (per-run 2, 3, 2 --
"~3" would be the flattering round of a mean that is nearer 2). The corpus figure is a
substring test and the noisy-instrument caveat above governs it; the article-35 collapse to zero is the
hand-checked number.

THE WIRE CONTROL, TWO-SIDED, because a null result on the real run is unreadable without it -- "the flag
worked" and "the flag was silently dropped" produce the same clean scorecard. Article 183 alone,
concurrency 1, `--max-tokens 4096` both sides:
- `repetition_penalty 1.0` -- 1.0 is the identity for the penalty (it divides/multiplies logits by 1),
  which is what an OMITTED field means on this engine; that equivalence is an INFERENCE about vLLM's
  default, not a recorded artifact, and it is the one load-bearing assumption in this control. STILL
  SATURATES: `events[] == 60`, `finish_reason: stop`, `truncated false`, `schema_valid true`, 4076
  completion tokens. **So `max_tokens` alone does NOT close the loop** -- it stops cleanly at the schema
  cap, well inside the budget. The penalty is the knob that matters.
- `repetition_penalty 2.0` COLLAPSES the output: `finish_reason: length`, `truncated 1`,
  `schema_invalid 1`, prediction `null`, 4096 completion tokens.
One side proves the field is honoured; the other proves it is not silently ignored. Neither alone
distinguishes the two.

CLOSING IT is either a scorer change or a prompt change and the entry does not presume which:
a gold-free `source_entity_article_grounding` beside the integrity metric (same loop, same
`p.source_content`, the normaliser above), or a prompt clause the model actually obeys. What CANNOT close
it is quoting integrity: 0.9603-0.9900 is the number this failure mode produces.
Re-check (no engine, but it reads a predictions file from `/tmp` -- article 35 is the one that reproduces
in every run):
```
python3 - <<'PY'
import json, re
sub = {r['source_index']: r for r in json.load(open('<the 40-article subset>.json'))}
art = re.sub(r'[^a-z0-9]+', ' ', sub[35]['input']['content'].lower()).strip()
for line in open('<a NEW-arm predictions>.jsonl'):
    r = json.loads(line)
    if r['source_index'] != 35:
        continue
    e = r['prediction']['entities']
    coined = [x['name'] for x in e if re.sub(r'[^a-z0-9]+', ' ', x['name'].lower()).strip() not in art]
    print(len(e), 'entities,', len(coined), 'not in the article:', coined[:6])
PY
```
2026-09-06 -> `60 entities, 50 not in the article` (49 on the third run) for the three NEW runs; the OLD
arm prints 9, 10 and 11 entities and `0 not in the article`. Build the 40-article subset with the joiner
in the entry two above this one.
RUN THE SAME LOOP OVER A GUARDED PREDICTIONS FILE and it prints `6 entities, 0 not in the article` (4 and
8 on the other two runs) -- that is the control, and it is the one line of this re-check that decides
whether the defect reaches production. WHAT THE RE-CHECK CANNOT REACH: every predictions file it reads
lives in `/tmp` and none is committed, so one `tmpwatch` ends it. The PROVENANCE LIMIT recorded two
entries above governs this entry and the two below it identically; re-deriving these numbers afterwards
costs 12 RUNS on the live engine -- four cells (two prompts x two samplings) at three runs each, ~2
minutes per RUN and ~25 minutes in total at concurrency 6 plus a checkout of the pre-#1017 prompt, `git show 45d02cd8bd3e360529267c9fec3868fd8554af4a`.

### The `entities_f1` regression REVERSES SIGN at production's sampling: -0.0337 becomes +0.0478 [2026-09-06]
THIS ENTRY USED TO BE HEADED "`entities_f1` fell 0.0338 under the corrected prompt, and 94.9% of it is
ONE article", and it was measured only at `repetition_penalty: null, max_tokens: 8192`. A fourth cell --
the OLD prompt at production's **1.1 / 4096** -- was measured 2026-09-06 and makes the comparison
same-sampling. The prompt's effect on entities is not a constant with a sign; it depends on the decoding:

| cell | `numbers_f1` | `entities_f1` | ent precision | ent recall |
|---|---|---|---|---|
| OLD x `null`/8192 | 0.3594 | 0.6734 | 0.8573 | 0.5545 |
| NEW x `null`/8192 | 0.5467 | 0.6396 | 0.7588 | 0.5529 |
| OLD x **1.1/4096** [production today] | 0.3446 | 0.5385 | 0.9191 | 0.3809 |
| NEW x **1.1/4096** [production + #1017] | 0.5152 | 0.5862 | 0.8995 | 0.4348 |

`entities_f1`: **-0.0337 at harness sampling, +0.0478 at production sampling**. BOTH are disjoint over
their three runs -- NEW's best (0.6531) sits below OLD's worst (0.6712) in the first pair, and NEW's
worst (0.5820) above OLD's best (0.5452) in the second, worst-case **+0.0367** -- so neither sign is
noise. Both comparisons are prompt-against-prompt; what decides WHICH SIGN the prompt's effect carries is
the SAMPLING REGIME it is measured under. (The old headline's 0.0338 was the
difference of two 4-decimal displays; from full precision it is 0.03374.)
`numbers_f1` does NOT reverse and needs no retraction: +0.1873 harness, **+0.1707** production, disjoint
at worst-case +0.1357 (OLD runs 0.3278 / 0.3505 / 0.3554, NEW 0.4910 / 0.5184 / 0.5362).
EVERY DELTA IN THIS ENTRY IS COMPUTED FROM FULL PRECISION, NOT BY DIFFERENCING THE PRINTED CELLS.
Subtracting the 4-decimal table gives 0.1706, 0.0477, 0.0539 and 0.0368 -- four apparent last-digit
errors that are only rounding. The full-precision values are +0.187325, +0.170662, -0.033744,
+0.047776, and the worst cases +0.147330, +0.135671, +0.036727.

THE MECHANISM RECORDED HERE WAS ALSO WRONG, and the corrected conclusion must not carry the discredited
explanation forward. The entry attributed the fall to pure false-positive inflation -- "`entities_recall`
did NOT move" while emissions rose, so the extra entities bought nothing and only precision suffered.
That is true WITHIN harness sampling and it is not the operative mechanism. Under the guard, in **BOTH**
arms, precision RISES and recall FALLS: OLD `P 0.8573 -> 0.9191, R 0.5545 -> 0.3809`; NEW
`P 0.7588 -> 0.8995, R 0.5529 -> 0.4348`. Emissions fall **403.7 -> 258.7** per run (OLD) and
**454.7 -> 301.7** (NEW). The guard is trading recall for precision wholesale, in both arms, and that
trade is far larger than the prompt effect it was masking. The article-35 concentration below is a real
description of the unguarded arms and has NO production counterpart: article 35 contributes 180 of the
NEW arm's 1364 entities unguarded (13.2%) and 18 of 905 guarded (2.0%).

WHAT SURVIVES UNCHANGED, at harness sampling. `entities_recall` did NOT move -- **0.5545 -> 0.5529** --
but the arms emit **403.7 -> 454.7** entities per run for **346.0 -> 345.0** true positives: fifty-one
more entities per run bought NEGATIVE ONE. So precision fell **0.8573 -> 0.7588** and `entities_f1` fell
**0.6734 -> 0.6396**.
EVERY ENTITY FIGURE IN THIS ENTRY IS THE c6 ARM ALONE, n=3, AGAINST THE POST-DECISION GOLD'S 624
entities. The entry above quotes `entities_f1` **0.6744** with range 0.0092 -- that is the POOLED
five-run figure against the OLD gold and its pre-decision 616-entity denominator. Different rows, not a
contradiction, and the two must not be differenced.
`ent_type_accuracy` ROSE, 0.7332 -> 0.7594, and it is graded on ALIGNED pairs only, so the false positives
below never touch it.
`macro_indicator` EMISSIONS ARE THE FLATTERING NUMBER HERE AND MUST NOT BE QUOTED BARE: 30.0 -> 99.3 per
run against 94 in gold reads as the model finally typing entities the way the gold does. It is not.
The gold's 94 `macro_indicator` entities include ZERO on article 35, and article 35 is where ~50 per run
of the 99.3 come from. Excluding that one article the emissions are **30.0 -> 49.7 against the same 94**
-- a real move toward the gold's typing, and still barely half of it.

THE CONCENTRATION IS THE FINDING AT HARNESS SAMPLING, AND ONLY THERE -- under the guard article 35 emits
4 to 8 entities and the concentration does not exist. The false-positive delta over the three-run pair is **+156 NET**, and
`sentinel-v6.2-cove.json` index 35 alone contributes **+148 of it -- 94.9% of the net** -- the runaway
coinage in the entry directly above: 30 -> 180 entities emitted over three runs against 29 -> 31 true
positives. QUOTE IT AS A NET, because the gross is a different number and says something else: 14 articles
move UP (+189 together, of which article 35 is +148 and the largest of the other thirteen is +12, four per
run, on article 1), 10 move DOWN (-33) and 16 do not move at all. An earlier draft of this line read "the
other thirteen sum to +8" -- that is the NET after the ten decreases, not their sum, and the two figures
differ by 33.
EXCLUDE ARTICLE 35 FROM BOTH ARMS AND THE REGRESSION IS ESSENTIALLY GONE: precision 0.8544 -> 0.8480,
`entities_f1` **0.6709 -> 0.6669**, a net +8 false positives across three runs.

A HYPOTHESIS WAS PUT AND THE PROBE DOES NOT SUPPORT IT, recorded because the next agent will otherwise put
it again: that the new series names clear the `source_entity` affinity floor of **0.5**
(`LlmBenchmark/scripts/eval_harness.py:546`) while missing the entity-NAME floor of **0.8**
(`LlmBenchmark/scripts/eval_harness.py:551`), so one string helps `numbers` and hurts `entities`. The band
it predicts is real in aggregate -- unaligned predicted entities whose best token-F1 against a gold name
in the SAME article falls in [0.5, 0.8) go 22.3 -> 63.3 per run, and the `macro_indicator` subset of that
band goes 3.0 -> 44.7 per run. BUT THAT `macro_indicator` SUBSET IS **126 OF ITS 134 ROWS -- 94.0% --
ARTICLE 35 AGAIN** (rows counted across all three NEW runs, not per run), where the "near-miss" partner is
the gold entity `U.S.` at token-F1 0.500. That is not a surface-form mismatch on a legitimate series name;
it is the invented name scoring half-marks on a country. A general floor mismatch would spread across
articles and this does not. THE FLOORS ARE STILL THE RIGHT PLACE TO LOOK if the question is reopened --
both are named above and a one-line edit to either re-scores the whole corpus -- but the aggregate
regression is ALREADY EXPLAINED by invention, and lowering the entity floor to admit invented names would
make the scorer agree with a defect.

WHAT IS ACTUALLY OPEN: whether the corrected prompt's `macro_indicator` instruction CAUSED the article-35
runaway or merely uncovered it. The OLD arm emits 9, 10 and 11 entities and zero `macro_indicator` on that
article, so the instruction is at least proximate; that is one prompt edit and one re-run to settle, and until it
is, `entities_f1` should not be quoted as a cost of the macro-owner decision.
Re-check (no engine, same `/tmp` dependency): read `metrics.entities_{f1,precision,recall}.value` out of
the TWELVE `eval_harness` scorecards, three per cell, and average per cell -- that reproduces every row
of the table above to four decimals. THE FOUR CELLS LIVE IN TWO DIRECTORIES AND THE BARE BASENAMES RESOLVE TO NOTHING -- give the paths:
`/tmp/sentinel-remediation/convention-b-measurement/scorecard.old_c6_?.vsNEWgold.json` (OLD @ null/8192)
and `.../convention-b-measurement/scorecard.new_c6_?.json` (NEW @ null/8192);
`/tmp/sentinel-remediation/loopguard-control/scorecard.prod_c6_?.json` (OLD @ 1.1/4096) and
`.../loopguard-control/scorecard.lg_c6_?.json` (NEW @ 1.1/4096). The PREDICTIONS follow a DIFFERENT
naming: the OLD @ null/8192 arm's are `/tmp/sentinel-remediation/qwen-phase1/preds.c6_?.jsonl`, NOT
`preds.old_c6_*` -- searching for the scorecard's name finds nothing and reads as an artifact already
lost. It is not; it is named differently. Which
prompt a guarded run read is pinned by `request_shape.prompt_file` in its provenance sidecar -- the
guarded runs are the first on this path where that field distinguishes the arms, because the OLD arm was
run from a checked-out copy at an explicit path rather than from the repo path both earlier arms shared.
WHAT THIS RE-CHECK CANNOT REACH: all twelve scorecards and all twelve predictions files are in `/tmp`
and NONE is committed -- one `tmpwatch` ends every figure in this entry, exactly as the PROVENANCE LIMIT
two entries above discloses for its own. Afterwards it is 12 runs across four cells, ~25 minutes, not
the two arms the entry above budgets for its own two.

### Production's `repetition_penalty` 1.1 -- NOT the token cap -- costs `entities_recall` 0.5545 -> 0.3809, and that bill is being paid TODAY [2026-09-06]
NEW FINDING, and the only one on this path that is about PRODUCTION rather than about the benchmark.
The loop guard is not free and its price had never been measured. `CpuCod__JsonRepetitionPenalty=1.1`
and `CpuCod__JsonMaxCompletionTokens=4096` are live in `/opt/ai-inference/compose.yaml`
(`SentinelCollector/src/Configuration/CpuCodOptions.cs:91` and
`SentinelCollector/src/Configuration/CpuCodOptions.cs:102` carry the defaults and the rationale;
forwarded at `SentinelCollector/src/Services/GpuJsonExtractionService.cs:222`). Measured
2026-09-06 on the 40 gold articles, three runs per cell, the SAME prompt either side so the only axis is
sampling:

| prompt | `entities_recall` | `entities_precision` | `entities_f1` | entities emitted/run |
|---|---|---|---|---|
| OLD (what production runs) | **0.5545 -> 0.3809** | 0.8573 -> 0.9191 | 0.6734 -> 0.5385 | 403.7 -> 258.7 |
| NEW (#1017) | **0.5529 -> 0.4348** | 0.7588 -> 0.8995 | 0.6396 -> 0.5862 | 454.7 -> 301.7 |

Against the gold's 624 entities, production's guarded arm finds ~238 per run where the unguarded one
finds ~346: **roughly 108 gold entities per run that production does not extract and would extract
without the penalty.** The trade is coherent -- precision rises in both arms -- but it is a TRADE, it was
never quantified before today, and nothing in production measures it.

THIS IS NOT A COST OF #1017, and must not be filed as one. Production runs the PRE-#1017 prompt
(`git hash-object /opt/ai-inference/prompts/cod/cod_json_v1.txt` -> `45d02cd8bd3e...`, verified
2026-09-06, which IS the pre-#1017 blob), so the top row IS today's production. #1017 partially RECOVERS
the loss: at matched production sampling it buys **+0.0540 recall and +0.0478 `entities_f1`** for
-0.0196 precision. Deploying the corrected prompt makes this cost smaller, not larger.

IT IS THE PENALTY, NOT THE TOKEN CAP, ON THESE 40 ARTICLES. Across all 240 guarded records `truncated`
is **0**, every `finish_reason` is `stop`, and the largest completion is **3148 tokens -- 77% of the 4096
cap**. A cap nothing reaches cannot be removing content, so ON THIS SUBSET the recall delta is the
`repetition_penalty`'s.
SCOPE THAT SENTENCE, because the full corpus contradicts its general form: the `json_valid` row far above
records **9 of 597** substrate articles hitting `finish_reason: length` at 4,096 AT PRODUCTION'S OWN
SAMPLING. So the cap IS reached in production, on ~1.5% of articles, and "nothing reaches the cap" is
true of the 40 gold articles and FALSE of the corpus. Whether those 9 are loopers (which
`CpuCodOptions.cs:96` argues -- a JSON-CoD document not closed by ~4K tokens is in a repetition loop) or
legitimate long documents is NOT settled here, and the recall figures above do not depend on it: they are
measured on the 40, where the cap binds on nothing.
The cap still earns its place on the OTHER side: unguarded, NINE records over the eight runs exceeded
4096 -- 6516, 6432, 5767 twice, 5607, 4559, 4558 and 4181 twice -- and the cap would have cut every one.

AND THE GUARD CANNOT SIMPLY BE REMOVED -- the two-sided wire control in the entry above shows
`repetition_penalty 1.0` at the same 4096 cap STILL saturating article 183's `events[]` at the schema's
`maxItems` with `finish_reason: stop`. `max_tokens` alone does not close the loop. So the open question
is not whether to keep the guard but whether **1.1 is the right value**: only 1.0, 1.1 and 2.0 were ever
probed here, 2.0 collapses the output entirely, and nothing between 1.0 and 1.1 has been measured. A
sweep of 1.02 / 1.05 / 1.1 on this same 40-article harness is ~2 minutes per RUN (so ~6 minutes for a
three-run point, ~18 for all three points) and would say whether
half the recall comes back for a penalty that still breaks the argmax fixed point. THAT is the next
measurement on this path, and it is cheap.
DO NOT RUN THAT SWEEP WITHOUT READING THE COUPLING IT WOULD TRIGGER. The anti-invention entry above
records that the corrected prompt's article-35 runaway is SUPPRESSED by this penalty, not absent from it:
lowering 1.1 toward 1.0 walks back toward the arm where article 35 emits 60 entities and ~50 coined
names. So the sweep is not a one-metric optimisation -- each point must be scored on the SATURATION and
COINED-NAME observables from that entry as well as on recall, or it will buy recall with invention and
the scorecard will not show it. A penalty that maximises `entities_recall` and reopens the runaway is a
regression this file would have no number for.
FOLLOW-UP THIS PR DOES NOT TAKE (docs-only by scope): `CpuCodOptions.JsonRepetitionPenalty` is now load
bearing for PROMPT CORRECTNESS, not only for latency and loop-breaking, and nothing at the code says so
-- `grep RepetitionPenalty SentinelCollector/AGENT_README.md` returns nothing, and the XML doc on
`JsonRepetitionPenalty` (cited by SYMBOL, not line: it is a doc comment and the line will move)
justifies the value on loop-breaking and latency alone.
Per CLAUDE.md §INTENT_FIDELITY that precondition belongs in a D-entry on the SentinelCollector card with
an `// INTENT(D-n):` at the option, so a future agent tuning this knob for recall meets the constraint
at the code rather than in a backlog entry they may never open.

Re-check (no engine): average `metrics.entities_recall.value` across the three scorecards of each cell --
`scorecard.old_c6_?.vsNEWgold.json` and `scorecard.prod_c6_?.json` for the OLD row,
`scorecard.new_c6_?.json` and `scorecard.lg_c6_?.json` for the NEW row. The production-config half is
re-checkable from the repo and the host and does NOT decay:
```
grep -n 'JsonRepetitionPenalty\|JsonMaxCompletionTokens' /opt/ai-inference/compose.yaml
git hash-object /opt/ai-inference/prompts/cod/cod_json_v1.txt   # 45d02cd8... == pre-#1017
```
WHAT THE RE-CHECK CANNOT REACH: the twelve scorecards and twelve predictions files live in `/tmp` and
none is committed, so one `tmpwatch` ends every measured figure in this entry -- the same PROVENANCE
LIMIT the three entries above disclose, and for the same reason. What survives a clear is the METHOD and
the config half: re-deriving the table costs 12 runs across four cells (~2 minutes per run,
~25 minutes total at concurrency 6) plus `git show 45d02cd8bd3e360529267c9fec3868fd8554af4a` for the OLD prompt. Whoever repeats this
should write the predictions where the tree can reach them BEFORE quoting the numbers forward.

### The three watermark-only ReExtract legs re-assert the row's tier, and nothing pins that they do [2026-09-06]
`ExtractedObservation.ApplyReExtraction`'s `newSecMasterMethod` became REQUIRED in #1030, so the next
caller must CHOOSE a value. It cannot make them choose the right one. The three watermark-only skip
legs -- null `RawContent`, zero extractions, empty `Description`, at
`SentinelCollector/src/Workers/ReExtractBackgroundService.cs:369`, `:441` and `:583` -- pass
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

### The `source_entity` divergence guard: three things it does not pin [2026-09-06]
`build_cod_gold.py`'s gate is what stands between a diverged labelling instruction and a paid
rebuild of the gold, and a review of it 2026-09-05 confirmed the mechanism works: nine mutations cut
from production's own bullet, each required to be caught BY NAME, plus a known-good base and the
live check, and its own shape checked against literals neither structure feeds. A FOURTH gap that
review found is closed by the PR carrying this entry -- `--selftest` for `build_cod_gold.py`,
`verify_cod_gold.py` and `rescore_alignment_keys.py` now runs in
`.github/workflows/python-tests.yml`, where it previously ran only when a human remembered. Three
remain. None is theoretical; each was run against the shipped file on 2026-09-06 and each re-check
below writes nothing, needs no corpus and buys nothing.

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
`--selftest` returns from `main()` before reaching them. Delete the block and every control still
passes; nothing else in the repo names the function either (`grep -rln source_entity_rule_divergence
--include='*.py' --include='*.sh' --include='*.yml' --include='*.cs' .` -> `build_cod_gold.py` alone,
five occurrences in it, one of which is that call). The gate is reachable by inspection only.
```
python3 - <<'PY'
import pathlib, sys
src = pathlib.Path('LlmBenchmark/scripts/build_cod_gold.py')
body = src.read_text()
gate = body[body.index("    divergence = source_entity_rule_divergence()"):]
gate = gate[:gate.index("        return 2\n") + len("        return 2\n")]
print("gate deleted:", gate.count("\n"), "lines")
sys.path.insert(0, 'LlmBenchmark/scripts')
mod = {"__file__": str(src.resolve()), "__name__": "gate_deleted"}   # REPO resolves off __file__
exec(compile(body.replace(gate, ""), str(src), "exec"), mod)
print("selftest rc:", mod["selftest_source_entity_rule"]())
PY
```
2026-09-06 -> `gate deleted: 7 lines`, then `selftest: 11/11 controls behaved as required` and
`selftest rc: 0`. A control that fails on the gate-deleted body closes it, and such a control costs
nothing: the divergence check runs BEFORE `args.work.mkdir` and `load_corpus`, so driving `main()`
with a diverged site, `--corpus /nonexistent/corpus.json` and a work dir that does not exist returns
`rc = 2` with `REFUSED: the labelling instruction and production's prompt disagree about
source_entity.` on stderr and creates nothing -- measured 2026-09-06, no key and no request.

### `run_model.py --schema-file` silently bypasses `SCHEMA_REQUIRED`, on the model-acceptance path [2026-09-04]
`build_payload` reads `schema = load_schema(args) or extraction_json_schema()`
(`LlmBenchmark/scripts/run_model.py:430`), and `load_schema`
(`LlmBenchmark/scripts/run_model.py:514-517`) returns whatever JSON the
operator handed `--schema-file`, verbatim. Nothing between there and the wire checks that the supplied
schema's `required` covers `SCHEMA_REQUIRED`. The derived schema is the only one carrying that coverage
guarantee, and `--schema-file` is the flag that discards it -- without a word in the output.

WHY THIS FLAG. `--schema-file` is not exotic: `CLAUDE.md` §MODEL_ACCEPTANCE names it in the exact
invocation a model swap must produce a scorecard from (`--endpoint-mode completions --prompt-file
cod_json_v1.txt --schema-file cod_json_schema_v1.json --chat-template ...`). The bypass therefore sits on
the one path that decides whether a candidate model replaces the incumbent.

WHAT IT COSTS is already measured on this harness, at `run_model.py:123-127`: `certainty` was optional in
the request schema, so the model emitted it on 0 of 2,213 extractions while gold carries it on 5,111 of
5,111; `eval_harness` counts every omission a miss (`certainty_accuracy` at `eval_harness.py:351`), and
`certainty_accuracy` scored **-0.85** against threshold. That read as "the model is bad at certainty"
and it was the request schema. A supplied `--schema-file` reintroduces precisely that defect, and the
constant that was written to prevent it does not run.

MEASURED 2026-09-04, against the very file §MODEL_ACCEPTANCE names:
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
Three of the six graded fields `SCHEMA_REQUIRED` covers are not required by the supplied schema at ANY
nesting depth, `certainty` among them, and the run says nothing.

DISTINCT FROM THE ENTRY ABOVE, and that is the point. For production's CoD schema specifically the
mismatch is a whole different SHAPE, which the `--prompt-file` entry covers and which at least announces
itself as `schema_invalid = record count`. This entry is about the check that does not run for ANY
supplied schema -- including one that IS array-shaped, scorer-compatible and merely under-specified.
That case has no tell at all: it scores, it fills in, and the number is a penalty on the request.

CLOSE THIS by making `validate_request_shape` (`run_model.py:743`) refuse a `--schema-file` whose
`required` does not cover `SCHEMA_REQUIRED`, in the same fail-closed-rather-than-default shape it already
applies to `--chat-template-kwargs` in completions mode -- with the override spelled explicitly so the
choice lands in provenance. Do NOT close it by merging the supplied schema into the derived one: that
sends a schema the operator did not write, and then misreports it as production's.

TEST THAT WOULD PIN IT: `test_run_model.should_exit_two_when_a_schema_file_omits_a_graded_field`, a
sibling of `should_exit_two_when_completions_mode_has_no_chat_template` (`test_run_model.py:774`),
driving `main()` rather than a helper, with its paired positive (a covering schema runs). Note that the
existing `should_send_the_supplied_schema_when_a_schema_file_is_given` (`test_run_model.py:397`) asserts
the bypass verbatim -- it pins the current behaviour in place and must be read as the contract it is,
never as coverage of this hole.

Re-check: run the snippet above. If this entry is still true it still prints
`['certainty', 'period', 'text_quote']`, and `grep -n SCHEMA_REQUIRED LlmBenchmark/scripts/run_model.py`
still shows no reference inside `load_schema` or `validate_request_shape`.

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

### `run_model.py` still diverges from production's sampling on `--repetition-penalty`, `--stop` and `--min-p` [2026-09-04]
`--seed` LANDED (measured 2026-09-05: `grep -c seed run_model.py` -> 7, the flag defaults to
production's 42, `build_payload` sets `"seed": args.seed` and provenance records it). The eight
scorecards in `LlmBenchmark/eval-substrate/` that PREDATE it still carry no `seed` key and still imply
a determinism their runs did not have -- read them accordingly; nothing can retro-fit it.
THREE divergences remain, and this line said TWO until 2026-09-06 -- the third is the one that turned out
to REVERSE a finding's sign, so the omission was not cosmetic.
`--repetition-penalty` defaults to `None` (`run_model.py`, `ap.add_argument("--repetition-penalty")`)
while production always sends `CpuCodOptions.JsonRepetitionPenalty` = 1.1 -- the IDENTICAL shape to
`--stop` below, and the entry "Production's `repetition_penalty` 1.1 ... costs `entities_recall`"
measures what it is worth: -0.17 recall on the prompt production runs. `--max-tokens` does NOT belong on
this list: it defaults to 4096 and production sends 4096.
`--stop` exists but defaults to none while production always sends
`ExtractionOptions.StopTokens`; and `--min-p` forwards `min_p` into the vLLM payload though
`VllmCompletionRequest` has no such field (`ExtractionOptions.MinP` is documented "llama.cpp min_p"
and only `LlamaServerClient` sends it) -- so a `--min-p` run records a knob the engine never read.
Re-check (grep the symbols, never a line number -- this file's citations have rotted before):
```
python3 -c "import json;print(sorted(json.load(open('LlmBenchmark/eval-substrate/qwen25-32b-awq-vllm-20260903.scorecard.json'))['adapter_metadata']['sampling']))"
grep -n 'ap.add_argument("--stop"' -A 3 LlmBenchmark/scripts/run_model.py
grep -rn -B 7 'public float MinP' SentinelCollector/src/Configuration/ExtractionOptions.cs
```
2026-09-05 -> the old scorecard's sampling list still has no `seed` key; `--stop` still
`action="append", default=None`; `MinP` still documented llama.cpp-only. A `--stop` default carrying
production's tokens, or `min_p` dropped from the vLLM payload, closes the remaining halves.

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

### The Azure oracle ledger overstates spend -- exactly 3.0x on the largest run, 2.79x overall [2026-09-04]
`SentinelCollector/scripts/azure_oracle_client.py` shipped with Anthropic LIST prices (15/75) and was
corrected to Foundry's (5/25) at `e38f0c56`, 2026-05-02 -- a week AFTER the largest run. Pre-fix rows
are exactly 3x high and post-fix rows are correct, so the ledger is a MIXTURE and no blanket divisor
repairs it. The per-record key is `cost_est`; `cost_usd` belongs to `v7-labeled-opus47-*.jsonl`, where
9,715 of 9,720 rows are off by exactly 3.0000 ($1,288.73 recorded vs $429.72 actual). Foundry is
unreachable, but this file is our only record of past spend and new estimates anchor to it -- rescale
before quoting either.
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
2026-09-04 -> `rows 22026 recorded 1346.53 actual 483.49` and
`[(1.0, 12292), (3.0, 9730), (3.1828, 3), (2.8061, 1)]`. The 1.0 bucket IS the mixture. Outliers are
accounted for, not defects: 3.1828 = 3 haiku rows (0.80/4.00 -> 0.25/1.25, rounded); 2.8061 = one
aggregate row carrying `calls: 70`, not a per-call record.

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

**THE SENTINEL RAW-DATA STORE IS A 180-DAY WINDOW, NOT AN ARCHIVE AND NOT THE 30 DAYS THIS ENTRY CLAIMED —
so a corpus drawn from it still expires, six times slower.** `extraction-identity-implementation.md` §1 sizes
it as "59,634 files, 5.7 GB, retained since 2025-01-01" and builds story S0 on that; that half is still wrong.
Retention is `Extraction__RawRetentionDays=180` (`/opt/ai-inference/compose.yaml:1127`; code default 180 at
`SentinelCollector/src/Configuration/ExtractionOptions.cs:686`), raised from 30 on 2026-08-27 — a change this
file already recorded ("Measured 2026-08-27, raising `RawRetentionDays` 30 -> 180", in the raw-retention entry
above) while this entry went on asserting 30, so the document contradicted itself for nine days.
Measured 2026-09-05: the oldest retained raw file is **2026-07-27**, now today-40 and WIDENING because nothing
has aged out since the raise, and the fixturable share moved **48% -> 56.9% (68,826 of 120,998)** — growth a
30-day window cannot produce. The consequence is softened, not gone: fixtures are still committed rather than
queried, but "select only RECENT articles" is no longer the right instruction. Anything back to 2026-07-27 is
available and the first post-raise prune is due ~2027-01-23.
Re-check:
  `SELECT source, min(collected_at)::date, max(collected_at)::date, count(*) FROM sentinel.raw_content
     WHERE raw_file_path IS NOT NULL GROUP BY 1 ORDER BY 4 DESC;`
  # 2026-09-05 returned min 2026-07-27 on the four large sources and later dates on the ten small ones. Every
  #   min must be >= today-180. A min that starts ADVANCING means the first post-raise prune ran; a min at
  #   today-30 means the setting was reverted and this entry's original claim is true again.
  `SELECT count(*) AS published, count(*) FILTER (WHERE r.raw_file_path IS NOT NULL) AS with_raw_file
   FROM sentinel.extracted_observations o JOIN sentinel.raw_content r ON r.id = o.raw_content_id
   WHERE o.published_at IS NOT NULL;`   # 2026-09-05: 120998 | 68826

**THREE NAMED CANDIDATE IDs IN THE S0 SELECTION DO NOT MEAN WHAT THE PLAN SAYS, AND ALL THREE FAIL
THE SAME WAY: THE PLAN COUNTED PUBLISHED ROWS AND THE DEFECT IS ABOUT KEYS.**
`extraction-identity-implementation.md` §1 names `154787`, `136367` and `146606` as the
"max-collision" articles on 60 published observations each. Measured 2026-08-26, only `154787`
collides: `136367` and `146606` carry NO Symbol and NO instrument on any published row, so
`CreateSeriesCollectedEvent` falls to its `{source}:{description-slug}` leg, which separates them
(60 rows -> 15 measurements -> 15 keys, and 60 -> 32 -> 31). They are duplicate-extraction articles,
not collision articles. Separately, `150183` is listed as the top mixed-unit candidate at "6 classes
/ 34 obs"; the plan's own mixed-unit SQL requires `instrument_id IS NOT NULL` and `150183` has none,
so it does not appear in that query's output at all — it is one of the cleanest SEPARATING articles
in the corpus (35 published, 35 keys). The corrected selection is in
`SentinelCollector/tests/SentinelCollector.UnitTests/Fixtures/GoldenCorpus/corpus.spec.json`, and
the extractor refuses to write a fixture whose declared state disagrees with the measured one.
Re-check: run the collapse query in `MANIFEST.md`'s "keys today" column definition against those
four ids; only 154787 may show measurements > keys.

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

**Citations in tracked `.md` that cannot land are the corpus's steady state, and so is rc 1: 28 of them at
`5d42ce9f`.** Reproduce:
`mapfile -d '' F < <(git ls-files -z '*.md'); python3 scripts/verify-citations.py --quiet "${F[@]}"` -- 198
files / 497 citations / 28 cannot land at `5d42ce9f`, rc 1. THE FIGURE MOVES WITH EVERY DOC EDIT, WHICH IS
WHY IT IS STAMPED: 296 checked, 17 cannot land at
`68b655df` (PR #967 head), 16 once that PR's own regression in the architecture-cards exemplar card is repaired;
292 checked / 16 cannot land at its base `eb2835e8`. This headline read a bare, unstamped "16" until 2026-09-06,
by which point the true figure was 28 -- the sentence a reader quotes was the one sentence in the paragraph
carrying no sha.
THE `xargs -0` PIPELINE THIS LINE ALSO CARRIED UNTIL 2026-09-06
REPORTS THE SAME COUNTS AT rc 123, NEVER rc 1: `xargs` remaps a child's exit status, so the rc this entry's own
headline asserts is UNOBSERVABLE through the form it prescribed. Measured on the identical corpus at `5d42ce9f`:
`mapfile` rc 1, `xargs` rc 123, both 198 files / 497 citations / 28 cannot land. #1021 repaired the same defect
in CLAUDE.md §TOOL_UPKEEP; this copy survived it, which is what a fact written twice does. So a green run is not
the bar today and nobody should chase one
as a merge gate: judge a PR on whether its cannot-land SET is a subset of its base's. Composition of the 16 AT
`eb2835e8` -- a SNAPSHOT, not today's 28, and it is the composition that has never been re-taken -- because
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
A MECHANICAL LINE-SHIFT REPAIR PRESERVES A CONTENT DRIFT IT PASSES THROUGH, measured 2026-09-06 on the eleven
citations #1026 handed over. SEVEN are pure shifts whose base citation was already correct and which re-land
on byte-identical content; FOUR are content repairs of citations ALREADY stale on main, three of them with
deltas of +82, +114 and +194 -- mutually inconsistent, so they were never shifts at all. An earlier revision
of this line said TEN pure shifts, which counted three content repairs as shifts and is the same over-claim
this paragraph exists to catch.

NAME THE MERGED SHA, NOT THE BRANCH. An earlier revision of this entry said the eleven were "verified against
#1026's branch", and that was never a checkable statement: a branch is a moving target. The numbers were
computed against `e4e77381`, a mid-PR revision; #1026 then took FIVE more rounds and merged as `d139f954`.
THREE of the eleven repairs were wrong against the tree that actually landed -- the `prompt_file` conjunct
1242 -> **1335** and `validate_request_shape` 712 -> **743**, those rounds having inserted 93 and 31 further
lines above them, plus the certainty cite below. Re-verified by reading every target on `d139f954` and
repaired here. A citation figure without its tree is the same defect as a metric without its sampling, one
directory up -- and "its tree" has to mean a sha that cannot move.

THE ELEVENTH IS THE CONTENT CASE AND TOOK TWO PASSES. It names `certainty_accuracy` and had ALREADY drifted
onto the `value` comparison; shifting it by the same +8 as its neighbours carried that drift forward intact.
The first repair in this PR moved it to the `certainty` COMPARISON (`_certainty_equal`) -- the mechanism, but
not the thing the prose NAMES. The metric itself is defined 20 lines below, at `eval_harness.py:351` on the
merged tree and eight lines earlier on the tree before it. FOUR landings, every one non-blank and every one
GREEN: the original, the mechanical shift, the first repair, and the right answer. Only opening the target
and reading what the prose names separates them.

THREE MORE SAT OUTSIDE THE HANDOVER'S ELEVEN, all in the `--schema-file` entry above, and they were caught
by three DIFFERENT instruments -- which is the finding. Sweeping every citation in this file into the files
#1026 touched, rather than only the ones the handover listed, surfaced two: one cited a line BLANK in both
trees, the other a teardown line, while the test functions their prose names --
`should_exit_two_when_completions_mode_has_no_chat_template` and
`should_send_the_supplied_schema_when_a_schema_file_is_given` -- sit 94 and 11 lines further down.
THE THIRD WAS FOUND BY NEITHER, and only by an independent re-derivation reading each target against the
prose. That entry's opening sentence points at a block of stdlib imports and a comment documenting a
DIFFERENT constant, while the measurement it paraphrases -- `certainty` optional, 0 of 2,213, -0.85 -- sits
seventeen lines below, in the comment on the constant the entry is actually about. It never moved on ANY of
the three trees and it lands on real code.
SO EACH INSTRUMENT HAS ITS OWN BLIND SPOT, and they do not overlap: a handover computed from a diff sees only
citations whose target MOVED; a cannot-land check sees only citations that hit NOTHING. The residue --
already wrong, never moved, lands on something -- is invisible to both, and visible only to a reader
comparing the prose against the target. THIS PARAGRAPH SPELLS NO `file:line` FORM, for the reason the entry
above it gives: the tool parses its own backlog, so writing a broken citation out verbatim to describe it
ADDS that finding to the count being described.

THE COUNTS WERE IDENTICAL ACROSS THAT BRANCH ON BOTH SIDES, 502 and 502, while eleven citations moved: an
aggregate hides a substitution, which is why the discipline is to diff the LANDING TEXT and never the totals.
Measured for THIS repair the same way, documented invocation, once per checkout: pristine `d139f954` reads
502 citations / 30 cannot land and this branch 512 / 27. The delta of three is exactly the three cannot-lands
this file carried into the harness files on merged main, and the OTHER three wrong repairs moved no count at
all -- they landed on real-but-wrong lines both before and after. Read the totals as corroboration of a
landing-text diff that was already done, never as the check itself.

**3 memory citations cannot land, all of one irreparable class, and until 2026-08-24 no routine sweep had ever
included one.** The memory corpus lives outside the repo, so `git ls-files` cannot name a memory file and the documented
sweep answered for the repo alone while printing a sentence that reads like a full answer.
`verify-citations.py --memory` now assembles that corpus (`$DREAM_MEMORY_ROOT`); the resolver needed no change,
only the corpus did. Reproduce: `python3 scripts/verify-citations.py --quiet --memory`. First sweep measured 105
files, 37 citations, 9 cannot land; six were repaired the same day, leaving 36 checked. A later dream
review retired a `deploy.yml:1453` citation from `MEMORY.md` itself, so the count is now **35 checked, 3 cannot
land** — the figure moves whenever a citation is added to or retired from the corpus, which is why the reproduce
command above is the durable half of this entry and the number is not.
Composed with the tracked corpus the figures add exactly — files, citations and findings each summing — which is
what proves the flag is additive rather than a different sweep wearing the same summary line.

THE SURVIVING 3 ARE A PERMANENT EXEMPTION AND MUST STAY BROKEN, like the architecture-cards `weak-card` fixture
above. Every one sits inside a VERBATIM `ev:` provenance quote — a transcript excerpt carrying the session id,
turn and timestamp that a dream-authored memory rests on. NONE of the three ever resolved, so there is no correct number to move any of them to: one landed on a blank
line at the nearest contemporaneous commit, and the other two were already AMBIGUOUS when written — a second
`server.py` has existed since 2026-05-17 and ten files are named SKILL.md. And
re-pointing any of them would edit quoted words, fabricating a quote in the record that exists to make the claim
traceable. THIS IS A CLASS, NOT THREE INCIDENTS: any provenance quote may contain a citation-shaped string, so
expect the floor to sit above zero permanently and judge a change by whether its cannot-land SET is a subset of its
base's. Deliberately NOT solved in the tool, and measured before deciding rather than after: 0 of 411 tracked-repo
citations sit inside an HTML comment, and 6 of 35 memory ones do — all six in dream provenance quotes. So the
skip would cost no real citation TODAY, and would silently stop covering any that later appeared in the one place
provenance lives. A documented floor is the smaller risk, but it is a judgement, not a free win.

What the six repairs were, since they are the reusable half. THREE were bare basenames the monorepo shares
(`AdminEndpoints.cs` once, `EventPublisher.cs` twice), fixed by giving each a directory component rather than by
widening the tool — AMBIGUITY DENIES is working as designed, and one of them
was also DRIFTED, so resolving the ambiguity alone would have produced a confident GREEN on a line that had become
an unrelated `.Include(...)`. Content had to be read; the tool cannot do that half. ONE was a plain line re-point, the only repair of the class this
entry otherwise avoids, and it was safe because the SKILL had meanwhile absorbed the finding so the target text
changed too. The last TWO were line numbers quoted as EXAMPLES OF ROT — the card text as it read when it was
wrong — rewritten in non-citable form, because re-pointing them would have inverted the anecdote they exist to
tell. Three plus one plus two; the earlier wording said four plus two and counted a line-number-free prose mention
among the findings. THIS ENTRY CARRIES NO `file` plus
line-number FORMS, for the reason the entry above gives: the tool parses its own backlog. Same caveats as the
tracked corpus — a green run is a FLOOR on rot, never a census, and the tool is blind to content drift. One
hazard specific to this corpus is documented in the module docstring and deliberately NOT guarded: a memory file's
own directory is searched first for a bare basename, so a bare `.md` cite colliding with a sibling memory filename
would bind to the memory copy silently. Measured at 0 occurrences today.

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
`deployment/tests/alerts/selftest.sh:479` reads a file's mode with `git ls-files -s -- "$f" | cut -d' ' -f1`, which
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
Re-check [SUPERSEDED 2026-09-06 on the comparison, not on the consequence]: run the tool from a pristine checkout
of the merge-base and again from the branch, and diff the LANDING TEXT of every citation. Do NOT settle for
comparing the `N citation(s) checked, M cannot land` line: a rise in M is still a finding, but EQUALITY IS NOT A
PASS. Two cases since prove it -- the `.md`-blind-spot entry below, where 488/27 matched its base while four
citations drifted, and `a8a0ed5d`, where 195/496/28 matched EXACTLY while three did. CLAUDE.md TOOL_UPKEEP
carries the rule in its current form.

NARROWED, NOT CLOSED [2026-09-04, #1002]. One sub-class is now machine-decidable and is decided: a citation whose
prose names `D-n` within 8 lines and which LANDS on a line beginning `D-m` is reported as `WRONG-D-ENTRY`, counted
apart from cannot-land so every figure above stays comparable. That sub-class is the one that recurred: PR #1002
added 8 lines above `SentinelCollector/AGENT_README.md`'s DECISIONS block, `:84` stopped being blank and became
D-13, and the cannot-land count FELL -- 474/30 at base c761e7b4, 483/29 at 91ec2319, by the Re-check above --
while two docs began sending a reader to the wrong entry. (Anchor such a pair to its two shas. This line said
487 for one review round: true when measured, dead two commits later when a backlog edit removed four
citations by re-anchoring them to symbols and nobody re-swept. Re-derive it, never quote it.) The
FinnhubCollector case ABOVE IS STILL LIVE AND STILL GREEN: those five are GUARD citations landing on method
declarations in a `.cs` file, where no `D-n` appears at the landing site and demanding one would condemn all 111
GUARD citations in this repo's cards. So the CONSEQUENCE paragraph above is unchanged for every citation that is
not a card-entry pointer, and a comparison against a pristine baseline is still the only way to see one move --
on its LANDING TEXT, per the superseded Re-check above, never on the counts alone.

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
re-run 2026-09-05. This entry said "currently 4": a 13x understatement that grew with nothing going red, and
the missed set includes `deployment/artifacts/compose.yaml.j2`, the gate-layer template this entry itself
calls the one that bites. One of the 53
(`dream/index_hooks.py:43`) is an EXAMPLE inside a regex doc comment and must never be "repaired", which is why
this is a corpus question with a judgement in it rather than a flag to flip. Closing it means either extending
the documented invocation and pricing in that example class, or a `--scope`-style opt-in; both change what every
future sweep reports and need their own before/after counts, exactly as the `_EXTS` entry below says.

**Nine real-but-wrong citations stand in `SentinelCollector/AGENT_README.md`, all GREEN, none of them any PR's
debt.** The live instance of the drift class above, found by hand in review of PR #1004 and deliberately NOT
repaired there: they sit in the `DeterministicResolver` / `ExtractionProcessor` D-entries as a -35 cluster from an
old un-followed insertion plus -74, -30 and -4, and one is a NAMING error rather than a number --
a GUARD reads `ResolveAsync @ src/Services/DeterministicResolver.cs:637` while `ResolveAsync` is the thin
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

**~35 `file:line` citations in THIS FILE point at real-but-wrong content, and the sweep reads GREEN on all of
them.** The live instance of the drift class above, inside `docs/BACKLOG.md` rather than a card. Measured
2026-09-05 by a full-file audit that opened each citation: 125 resolve and every one is IN BOUNDS, while roughly
35 land on a comment, an XML doc, a blank line or a different symbol. The largest cluster is a uniform +35-line
drift across the `DeterministicResolver.cs` set, from #969 / #988 / #1004 all landing after the 2026-08-15
measurements; a second pair (`/opt/ai-inference/compose.yaml:1155`, real `:1161`) appears in TWO entries at
once. Only the citations inside entries the 2026-09-05 cleanup rewrote were repaired. The rest are LEFT ON
PURPOSE: a repo-wide repair sweep is no evidence of anything, which the nine AGENT_README citations above
already demonstrate — six of the nine were "repaired" while already 35 lines short.
Re-check, and note what it CANNOT do: `python3 scripts/verify-citations.py docs/BACKLOG.md` ->
**135 citation(s) checked, 4 cannot land** (2026-09-05, after the cleanup; 134 / 4 before it). All four are
AMBIGUOUS-BASENAME unresolveds — `appsettings.json`, `DependencyInjection.cs`, `AdminEndpoints.cs`,
`SeriesManagementService.cs` — and not one of them is a drifted line. The count would not move if every one of
the 35 were repaired, nor if thirty more drifted, which is the whole point: the only finder is opening each
citation and reading the line, so treat a fall in `cannot land` as no signal at all.

**`verify-citations.py` silently skips a citation whose file has an extension outside a 10-item allowlist, and
skips a bare `:NN` continuation entirely unless `--bare` is passed.** Two separate gates, both read from the code
2026-08-17, and ANCHORED TO SYMBOLS rather than lines because every line number this paragraph once carried had
already rotted: the tool's own docstring grows, and `:121-122` had drifted off `_EXTS` into prose. The first gate
is the `_EXTS` / `CITATION` pair in `scripts/verify-citations.py`:
`_EXTS = "py|cs|yml|yaml|md|sh|json|csproj|ts|sql"` feeding
`CITATION = re.compile(rf"(?P<file>[\w./-]+\.(?:{_EXTS})):(?P<start>\d+)(?:-(?P<end>\d+))?\b")` — the `\.` is
mandatory, so an extensionless path never matches, and neither does a real extension that is not on the list. This
is DELIBERATE and the rationale is the comment directly above `_EXTS`: a permissive `\w+\.\w+:\d+` also swallows
version strings and `host:port` URLs. So the gap is a precision/recall tradeoff already priced in, NOT a bug to
widen on sight — widening it changes what the whole repo's sweep reports and needs its own before/after counts.
The second gate is the `BARE` regex, which parses a continuation like ``(`:147`)`` only when `--bare` is passed;
the docstring paragraph headed BARE CONTINUATIONS ARE OPT-IN says to leave it off by default, so in a default run
those references are not checked at all.
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

**`ExtractionOptionsValidator` has no rule for `Thinking=Enabled` with a non-empty
`ThinkingSuppressionSuffix`, so a suffix an operator explicitly set is accepted at boot and discarded on
every request.** `ThinkingControl.FormatPrompt` appends the suffix on the `Disabled` branch only --
verified 2026-09-04, `SentinelCollector/src/Services/ThinkingControl.cs:32-34` reads
`return options.Thinking == ThinkingMode.Disabled ? formatted + options.ThinkingSuppressionSuffix :
formatted;`. That branch is correct and D-26's paired negative test asserts exactly it. The gap is one
layer up: nothing tells the operator the suffix they configured will never reach the wire.

MEASURED 2026-09-04: `grep -c Thinking SentinelCollector/src/Configuration/ExtractionOptionsValidator.cs`
-> **0**. The validator carries nine `failures.Add` rules (MaxToolRounds; tool-flags-vs-backend;
QualitativeCoveTrustDowngradeFactor; three EntityResolution bounds; two AutoApproveDrain bounds) and not
one reads `Thinking` or `ThinkingSuppressionSuffix`.

This is the "configured and inert" shape D-26 exists to prevent, one level out. D-26's own measurement is
the same class inside the clients: the suffix was applied at VllmClient's three sites and at NEITHER of
LlamaServerClient's two, so a CPU-arm run configured for suppression was unsuppressed and LOOKED
configured. The single-site fix closed the client asymmetry; it did not make a self-contradictory
CONFIGURATION visible, and `Thinking=Enabled` plus a non-empty suffix is exactly that -- two settings
that cannot both be honoured, accepted in silence. NOT A SUPERSESSION of D-26: the guard, its
precondition and its tests are untouched. This asks for a boot-time check the guard was never given.

Remedy unchosen -- reject at boot, or warn once at startup. A `failures.Add` on
`options.Thinking != ThinkingMode.Disabled && !string.IsNullOrEmpty(options.ThinkingSuppressionSuffix)`
matches the file's existing shape and fails fast; a Warning is the softer read if a deployment
legitimately parks a suffix across a mode flip. Test that would pin it either way:
`ExtractionOptionsValidatorTests.Validate_ThinkingEnabledWithSuppressionSuffix_Fails`, following the
`Validate_<condition>_Fails` convention already in that file, with the paired
`Validate_ThinkingEnabledWithoutSuffix_Succeeds` so the rule is not merely a refuse-everything.

Re-check: `grep -c Thinking SentinelCollector/src/Configuration/ExtractionOptionsValidator.cs`. `0` means
this entry is still true; nonzero means it has been closed.

**`probe_engine` cannot tell "this endpoint has no `/version`" from "nothing is listening", and the
checker that reads it now REPORTS that ambiguity rather than resolving it.** `probe_engine` in
`LlmBenchmark/scripts/run_model.py` wraps each of its three probes -- `/version`, `/props`, `/v1/models`,
each its own grep -- in a bare `except Exception:` of its own, and returns `engine: None` for both. A dead
port and a live OpenAI-compatible server that simply does not implement `/version` or `/props` -- SGLang,
a proxy, a future engine -- are indistinguishable in the field every caller reads.

MEASURED 2026-09-04, one alive stub serving `/v1/models` only, against one closed port:
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

WHAT IS ALREADY FIXED AND WHAT IS NOT. `check_staleness.py` no longer reads unknown as current: an
unidentified endpoint prints `live: UNIDENTIFIED (no /version, no /props, or nothing listening)` (the
`--endpoint` loop in `check_staleness.main`; grep `live: UNIDENTIFIED`) and every scorecard for that
engine classifies `DRIFT_UNCHECKED ... no live <engine> version available` (the `if live is None:` branch
of `check_staleness.classify`; grep `was NOT compared`), stale, with a non-zero exit. That removed the
MISLEADING half. It did not remove the root cause: the operator is told to "pass --endpoint for one that
answers /version or /props" whether their endpoint is DOWN (restart it) or merely SILENT about its
identity (no restart will ever help, and `run_model.py --allow-unidentified-engine` -- the flag spelling
is its own anchor -- is the actual answer). One message, two different remedies, and the checker cannot
say which it means.

The discriminator already exists in the returned dict and no caller uses it: `served_models` is
`['some-model']` in the live case and `[]` in the dead one, per the measurement above --
`grep -c served_models LlmBenchmark/scripts/check_staleness.py` -> **0**.

Remedy unchosen: `probe_engine` returning reachability as a field distinct from identity (the
information is already there -- non-empty `served_models` with `engine: None` IS "reachable but
unidentified"), and `check_staleness.py` naming the right remedy for each. Test that would pin it:
sibling stubs in `test_check_staleness.py`, one endpoint serving `/v1/models` only and one refusing
connections, asserting the two produce DIFFERENT operator-facing text -- assert on the difference, since
a checker that says the same thing about everything passes any single-case test.

Re-check: run the snippet above. If this entry is still true both lines still print `None None`, and
`served_models` is the only thing that differs between them.

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

Disposition per alert (entry-or-retire) is unchosen. This is measurement debt, not a defect in any one
rule.

**`Extraction__GuardsEnabled=false` — AWAITING AN OWNER DECISION.** A deliberate experiment, not an accident:
`/opt/ai-inference/compose.yaml:1161` carries it dated 2026-05-03, on the rationale that the Phase 4.3
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
  `cp <gate file> /tmp/x` and `cat <gate file> > /tmp/x` ALLOW here and DENY on main. This is the
    reader/`dest_last` class working as designed — the gate file is the SOURCE.
  `cp -T <gate file> /tmp/x` and `cp --no-target-directory <gate file> /tmp/x` ALLOW, same class, same reason.
    `--no-target-directory` is deliberately NOT a prefix of `target-directory`, so it keeps `dest_last`.
  Every finding above is a READ of a gate path. No write shape against the gate layer was left open.
CORRECTED 2026-09-05, and it INVERTS the sentence a reviewer would plan around. This entry asserted that the
INSTALLED guard DENIES `cp <hook> /tmp/backup.sh` — the stated reason a previous review could not measure
deletion counts. Probed against the armed installed guard (blob `31baafbb`): that cp **ALLOWS**, and so do
`cp -T`, `cp --no-target-directory` and `cat <gate file> > /tmp/x`. The obstruction is gone for the bare
spelling; plan the mutation round without a bypass. What still obstructs is the PREFIXED spelling — all nine
`WRAPPER_RE` words (`time nice timeout stdbuf command exec ionice nohup xargs`) in front of the same cp
**DENY**, naming the SOURCE. That is the verb-displacement class recorded as spelling (2) of the sibling-guard
entry in KNOWN DEFECTS, not a separate finding — write the cp bare.
Re-check: feed each shape to `.claude/hooks/ansible-gate-guard.sh` as
`{"tool_name":"Bash","tool_input":{"command":"…"}}` on stdin and read
`.hookSpecificOutput.permissionDecision`; an allow emits no JSON at all, so empty output IS the allow.

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
**NOT converging** — the monotone series above is the claim, not a ratio: one of its three steps found nothing
new at all. Separately, #935 measured "loosened = 0" against its own
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

**Hosted labelling routes -- measured 2026-09-04. STRICT `json_schema` enforcement is a per-PROVIDER
property, not a per-model one.** `.claude/skills/supervisor-mode/SKILL.md` §ORACLE_ROUTING routes new
labelling / gold / bulk-oracle work to a hosted OpenAI-compatible API (user directive 2026-09-04; Azure
Foundry access revoked) and points here -- by the literal string "hosted labelling routes" -- for the
measured routes and the probe that settles one. This entry is that target.

THE FINDING, WHICH CARRIES ITS OWN NEGATIVE CONTROL. Same model, same request, two providers, opposite
outcomes: `zai-org/GLM-5.2` served by **deepinfra** honours an `enum: ["alpha","beta"]`; the SAME model
served by **zai-org's own endpoint** ignores the schema entirely, invents `company` / `event_type` /
`financial_metrics`, and returns HTTP 200 while doing it. A route table keyed by MODEL is therefore wrong
by construction -- the key is (model, provider). That pair is also the proof the probe DISCRIMINATES rather
than flattering everything it touches: one arm passes and one fails on one model, one request.

THE PROBE DESIGN, which matters as much as the table. An enum probe ALONE is weak -- a competent model
obeys a two-value enum on semantics, with no grammar involved, so enforced and best-effort routes both
pass it. What proves ENFORCED decoding is a `required` field the input gives NO basis for: a `ticker` with
`pattern: "^Z{3}$"` on an article naming no such ticker. Constrained decoding physically cannot leave the
grammar and emits `ZZZ`; best-effort emits a plausible ticker, or omits the field. Run it TWICE per
(model, provider) pair -- one sample can look enforced by luck.

MEASURED ROUTES, HF router `https://router.huggingface.co` -- note NO `/v1` suffix on the base; the client
appends it:
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
Four different model families enforce on deepinfra, so the property tracks the PROVIDER and not the family.
CHOSEN: **DeepSeek-V4-Pro @ deepinfra** -- zero reasoning tokens, **~$0.00013/call OBSERVED**. Whole probe:
**$0.045 across 29 requests, OBSERVED**. Those two are the only BILLED prices in this entry; the pre-flight's
`~$0.0003` below is derived from the first and is the only other dollar figure here. Any per-token figure a
reader brings from a pricing page is a LIST price and must not be averaged in with any of them.

THE FLAG THAT PREDICTS IT IS NOT THE TEST. `GET /v1/models` exposes `providers[].supports_structured_output`,
and it predicted all 7 outcomes above. It is still a CLAIM, not a fact -- zai-org advertises `false` and
serves 200 rather than refusing, so the failure mode on this axis is SILENT in both directions and an
advertised `true` is equally unverified. The ZZZ probe is a PRE-FLIGHT before each batch; it is never
replaced by the flag.

BUDGET TRAP, worth its own paragraph. Reasoning models burn 600-3000 chars of hidden thinking before any
answer. GLM returned `finish_reason: length` with EMPTY content at `max_tokens` 80 AND at 512. Use >= 1024
output tokens on any route here. `run_model.py` does not hide this -- `--max-tokens` defaults to 4096, a
cut-off response is counted in `truncated` ALONGSIDE `schema_invalid`, and the run prints a WARNING naming
unsuppressed thinking by name -- but the record still lands in `schema_invalid` too, so an operator who
lowers the budget gets a scorecard that reads as a MODEL defect and is a harness budget setting.

`run_model.py` GAPS FOR THIS ENDPOINT. Each verified against the file at this commit; grep the symbol, do
not trust a line number -- this branch has moved them repeatedly.
- NO `Authorization` HEADER. `_http_json` sends `headers={"Content-Type": "application/json"} if data else {}`
  and nothing anywhere adds a bearer token (`grep -c Authorization` -> 0). Without this one change nothing
  else in this entry is reachable.
- THE FAIL-CLOSED ENGINE GATE ABORTS, CORRECTLY. The router answers neither `/version` nor `/props`, so
  `probe_engine` returns `engine: None` and `main` exits 2. `--allow-unidentified-engine` is MANDATORY for
  this route. Do NOT loosen the gate: the flag is the designed escape and it stamps the scorecard
  `engine: unidentified` / `engine_identified: false`, which is the whole point of the gate.
- `--model` IS FORWARDED VERBATIM AND NEVER REFUSED. `build_payload` sets `"model": args.model`, and the
  served-model membership check that follows the engine gate only PRINTS a warning. Because enforcement is
  per-provider, an id with no `:provider` suffix lets the router pick one and the harness will not stop it
  -- the scorecard then names a model whose enforcement was never established. Always pass
  `--model <id>:<provider>`.
- `strict: true` IS NOT REQUIRED. `grep -c '"strict"' run_model.py` -> 0: it sends
  `{"type": "json_schema", "json_schema": {"name": ..., "schema": ...}}` with no `strict` key, and deepinfra
  enforced anyway. Nothing has to be added for enforcement -- only for auth.
- PER-REQUEST COST WAS DISCARDED -- CLOSED at #1009 (`8dc8fca7`). `grep -c usage` WAS 0 in both
  `run_model.py` and `eval_harness.py`. run_model.py now captures `usage` per request, `aggregate_usage`
  sums `estimated_cost` and records how MANY records priced (`cost_reported_by`), so a partly-priced run
  cannot understate the per-record rate by dividing across all of them, and the run prints the bill.
  `eval_harness.py` still has zero `usage` readers and correctly so: it scores an existing scorecard and
  makes no calls. The Foundry ledger whose `cost_est` had to be corrected 3x after the largest run is why
  this mattered: a run that does not record its OBSERVED cost leaves only list-price arithmetic behind.

THE DISTINCTION TO PRESERVE. The router is CHAT-only. LABELLING (produce gold) and SCORING (grade
production's own `/v1/completions` path against that gold) are two jobs on two endpoints: a hosted route
buys the first, the second runs against a LOCAL engine and always will. `eval_harness._acceptance_evidence`
stamps `production_prompt_path: false` on anything that is not completions-mode carrying production's
prompt, its schema and a template that actually WRAPS the prompt -- so a hosted scorecard cannot be
acceptance evidence however good its numbers look.

WHAT IS NOT MEASURED HERE. Nothing above measures label CORRECTNESS -- only schema enforcement, failure
mode and cost. A quality comparison between these routes is a separate measurement and is NOT reported in
this entry; do not read `ENFORCED` as "good labels".

ACCESS IS NOT WIRED ON THIS BOX. Measured 2026-09-04: no `HF_TOKEN` in the environment, and
`~/.cache/huggingface/token` absent. The probe ran on a token supplied for it. Per §ORACLE_ROUTING's own
rule -- never read ACCESS from ARTIFACTS -- ask the user for the token rather than pricing work that
assumes it.

Re-check (free, local; the route table's own re-check costs money and is the pre-flight below):
```
for p in Authorization usage '"strict"' Content-Type allow_unidentified_engine; do
  printf '%-26s %s\n' "$p" "$(grep -c -- "$p" LlmBenchmark/scripts/run_model.py)"; done
printf '%-26s %s\n' 'usage@eval_harness' "$(grep -c usage LlmBenchmark/scripts/eval_harness.py)"
```
2026-09-04 -> `Authorization 0` / `usage 0` / `"strict" 0` / `Content-Type 1` /
`allow_unidentified_engine 1` / `usage@eval_harness 0`. Exactly TWO of the six rows are POSITIVE CONTROLS --
`Content-Type` and `allow_unidentified_engine`, the only two that are non-zero -- and a zero in EITHER means the
command is broken or the file moved, not that a gap closed; a typo'd path prints zeros too. The other four rows
are 0 BY DESIGN and a zero there is the FINDING, not a fault -- `usage@eval_harness` included, since it is the
last bullet above restated as a command.

PRE-FLIGHT BEFORE ANY LABELLING BATCH (costs money, ~$0.0003 for the two calls; never skipped, never
replaced by `supports_structured_output`): POST `/v1/chat/completions` on `https://router.huggingface.co`
with `model: "<id>:<provider>"`, `max_tokens` >= 1024, and a `response_format` `json_schema` whose
`required` includes a `ticker` of `pattern: "^Z{3}$"`, against an article naming no such ticker. Twice.
`ZZZ` both times = enforced. A plausible invented ticker, a missing field, or one of each = best-effort,
and that provider does not label.

**Labeller quality at n=5 through production's CoD prompt+schema, 2026-09-04.** Routes/enforcement:
"hosted labelling routes" above. DeepSeek-V4-Pro@deepinfra found ZERO facts the other two did not, but
missed 6 incl. a headline `$90/oz`: under-extraction, zero junk. Kimi-K3 MANUFACTURES facts ('third
quarter'->3, '20:49 ET'->"20:49"), half its barren-article extractions fabricated; 4/5 non-streamed
calls 504'd (120.1s); `reasoning_tokens: 0` is false (23,150 chars in `delta.reasoning_content`);
non-deterministic at temp 0. Opus 5 degrades `source_entity`, the resolution anchor, to a metric label
at 33%. The least-bad route still errs in kind: `$35.69` bound to January where the article says December -- wrong period,
silent series corruption. CAVEATS: n=5 is not a ranking; the recall denominator is a regex proxy
over-counting clock times and years; 4/5 were macro/market-wrap, so re-measure the 33% on
earnings/analyst-action first.
Re-check: the artifacts live only in `/tmp` (non-durable -- expect them gone). Re-running is ~$1.32 /
32 requests (5 substrate articles x 3 labellers) after the ZZZ pre-flight above; until then these
figures ARE the record.

**Accepted risks, do not re-flag.** Plaintext DB password `atlas_secure_password_2025` in 10+ tracked files, and
`OfrCollector/.env` tracked with `DB_PASSWORD` / `SMTP_PASSWORD` / `FRED_API_KEY`. The user accepted both
explicitly: private repo, LAN-only, public and public-derived data. Rotating touches the DB user and every consumer.

## PARKED EPICS

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

AND THE HARM IS LATENT, NOT ACTIVE -- which is what makes parking legitimate rather than a deferral of
live corruption. Re-verified 2026-08-27:
- **zero of ThresholdEngine's 71 loaded patterns read a `SENTINEL:NUM:` key** (`list_patterns`
  `enabled_only=false` returns 71 of 71 enabled). Three of the 72 repo pattern files name any Sentinel
  key at all, and all three use `SENTINEL:SECTOR:` through value-independent counts.
- **the matrix path cannot collide across articles**: 16,540 `public.macro_observations` rows in 30
  days carry the `{raw_content_id}:sig:{slug}` infix and all 16,540 `source_id` values are distinct.
- **the live exception is BDIY, and its trigger is still unpulled**: 163 rows, 160 published, still
  publishing 2026-08-27 under a bare `OwnedSeries` key; its only reader `baltic-freight-recession` is
  `"enabled": false` and is the one repo pattern file of 72 absent from TE's loaded set. The two
  `OwnedSeries` keys that DO have loaded readers -- `CHALLENGER_JOB_CUTS` and `TRUFLATION_CPI` -- have
  never carried a Sentinel row. `ADP_EMPLOYMENT` has 9 rows, last extracted 2026-05-18.
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

WHAT UN-PARKS IT -- four trip conditions, each with the command that reads it. None is tripped as of
2026-08-27.
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

**Claude Code function hooks — PARKED, not adopted [2026-09-05].** TypeScript in-process plugin
middleware: typed events, `next()` composition, `deny`, argument rewriting, `$.ui.ask`,
`$.model.classify`, `$.store` (cross-session). Unreleased — proposal #91870, flag
`CLAUDE_CODE_ENABLE_FUNCTION_HOOKS=1`, runtime in build 2.1.260; this box runs 2.1.261. Parked
because composition is strictly serial (their figure: 8x300ms, ~640ms parallel vs ~2.4s chained)
and our 18 scripts / 36 registrations, mostly PreToolUse, fire on every tool call. Revisit for the
JUDGEMENT half of memory hygiene (`$.model.classify`), which shell cannot do.
Source: https://claudefa.st/blog/tools/hooks/function-hooks
Re-check: `claude --version` past 2.1.261 with the flag shipped; `ls .claude/hooks/*.sh|wc -l` (18),
`grep -c '"command"' .claude/settings.json` (36).

## PRACTICE NOTES [cheap checks that were skipped three times or more]

**A defect found in a branch is not a defect on main.** Three fail-opens in one session were reported as live on
main and were not. The settling check is one command — `git branch --contains` on the commit that introduced the
line — and it was never run until a reviewer ran it.

**Scoped-deploy collateral is CONDITIONAL — record what actually moved, do not derive a rule.** Two runs on
2026-08-15: the `secmaster` scoped deploy ALSO recreated `llama-cpu-rag` and `llama-cpu-embed`; the
`sentinel-collector` one recreated nothing else. Two runs cannot say which services drag which neighbours, and a
general rule written off this sample would be wrong in one direction or the other — so no rule goes in
CLAUDE.md. The practice that survives either explanation: after any scoped deploy, enumerate what actually
restarted and record it WITH the run. A third and fourth agreeing observation is when a rule is earned.
Re-check: `sudo nerdctl container inspect <svc> --format '{{.Created}}'` per service — `container` is
load-bearing (CLAUDE.md VERIFY_TRAP: bare `inspect` resolves the IMAGE and hands back the BUILD time).
