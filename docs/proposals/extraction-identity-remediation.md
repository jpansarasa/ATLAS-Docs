# Extraction identity + event-contract remediation

**Status:** PLAN / for review -- NOT approved, NOTHING deploys from this doc.
**Author:** supervisor session 2026-08-25/26
**Scope:** SentinelCollector extraction -> event publish -> ThresholdEngine matrix
**Companion:** docs/BACKLOG.md (PR #989) carries the raw identity-collision measurement

---

## 1. The common shape

Four defects found in one session. All four are the same failure, not four failures:

> A component produces the right thing. The next component has nowhere to put it.
> The seam fails silently, and nothing measures the seam.

| # | Producer works | Consumer drops it | Silent because |
|---|---|---|---|
| R1 | v1 publishes sector events | v2 path never calls `CreateSectorEvent` | no metric counts event KINDS |
| R2 | CoD yields (subject, topic) pairs | `ExtractedObservation` keys on entity only | collisions look like volume |
| R3 | DSL carries `ClaimKind` + polarity | no column exists past the adapter | field simply absent, no error |
| R4 | embeddings computed | 59% stored un-normalised (FIXED #987) | HNSW returns A neighbour, not THE neighbour |

This is why the pipeline reads healthy while producing wrong output. Every stage
reports success. The loss happens between stages, where nothing is watching.

---

## 2. Evidence

Measured 2026-08-26 unless noted. Re-run each query to re-check.

### 2.1 Sector events (R1)

```
Extraction__UseV2Pipeline=true        # flipped 2026-05-16 (PR #340)
81da1ed4 2026-05-16 ops(sentinel): ... broaden V2 sources to all (#341)
```

- `CreateSectorEvent` has exactly ONE production call site: `ExtractionProcessor.cs:889`
- that site is inside `ProcessSingleArticleAsync`, AFTER the v2 early return at `:599`
- `RunV2ProductionAsync` (`:1760`) never calls it
- every August source with published rows is in `V2EnabledSources`:
  rss, searxng-content, challenger-rss, rss-mirror, fed-* (12 entries, indices 0-11)
- the only non-v2 August sources are `tsa-checkpoint` (0 published) and `rss-fallback` (0 published)

=> **100% of published extraction has run the v2 path since 2026-05-16. Sector events: zero for 102 days.**

August volume that reached publish and therefore reached the v2 return:

| source | observations | published |
|---|---|---|
| rss | 136,813 | 37,363 |
| rss-mirror | 8,473 | 2,280 |
| searxng-content | 8,264 | 1,437 |
| challenger-rss | 460 | 12 |

The v2 method's own XML doc asserts the parity that is broken:

> "publishes SeriesCollectedEvent identically to the v1 branch (plan D8: contract unchanged)"

v1 publishes TWO event kinds. v2 publishes one. The comment records the intent;
the code lost half of it. This settles port-vs-retire: **port**, not retire.

**Measurable after all, and better than the alternative** (corrected 2026-08-26 by
adversarial claim-check; both errors below were the supervisor's):

- `AtlasSectorCode` **is** a persisted column, added by migration
  `20260509234900_AddAtlasSectorCodeToObservation` on 2026-05-09 and populated even on the
  v2 path (`V2ExtractionPipeline.cs:215`). The original claim came from a column enumeration
  piped through `head -40` against a 47-column table -- a truncated probe read as complete.
  **Consequence: the loss is directly recoverable.** 4,324 rows since the 81da1ed4 cutover
  satisfy the sector gate as the code writes it. That is the backlog of lost sector events and
  an exact verification target for the port, rather than an unknown. (The claim-check that
  caught this offered 1,987; that figure did not reproduce, and 2,084 is the count when the
  REGULAR-publish gate's terms are wrongly borrowed. A correction is a claim too.)
- 16 Prometheus metrics DO match `sentinel.*(sector|event|publish)`
  (`sentinel_matrix_update_published_total`, `sentinel_atlas_sector_parse_outcome_total`,
  `sentinel_sector_tagger_request_total`, ...). The original "zero matches" came from a grep
  against an **empty response**: `localhost:9090` is unreachable from the host (rc=000);
  Prometheus answers on the container IP. A failed search is not a search that found nothing.

The finding survives in its precise form, and it is narrower and more useful: **no metric
distinguishes event KIND at the publish seam.** Sector *tagging* and *parsing* are observed
upstream; sector *publishing* is not observed anywhere, which is why 102 days of silence read
as health. Check what an existing metric already covers before adding one (S1).

### 2.2 Identity collision (R2)

```sql
WITH pub AS (SELECT raw_content_id, instrument_id FROM sentinel.extracted_observations
             WHERE published_at IS NOT NULL AND instrument_id IS NOT NULL),
     grp AS (SELECT raw_content_id, instrument_id, count(*) c FROM pub GROUP BY 1,2)
SELECT (SELECT count(*) FROM pub), (SELECT sum(c) FROM grp WHERE c>1);
```

- 20,770 published observations carry an instrument
- 15,331 of them (**73.8%**) sit in a `(raw_content_id, instrument_id)` group of size > 1
- 15,317 distinct descriptions across 2,733 instruments
- **1,422 of 2,735 instruments carry genuinely mixed units** -- more than one unit CLASS
  under one identity. The raw `count(DISTINCT unit) > 1` form gives 1,562, but ~140 of those
  are pure spelling (`PCT` and `percent`, `COUNT` and `count`, both live in the column).
  The strict bucket this doc used to describe here -- percent AND count AND dollars together
  under one identity -- is **289 instruments**, and the S0 corpus draws from those 289.
  An earlier revision attached the strict prose to the loose 1,577 figure, overstating the
  strict bucket 5.5x; see `extraction-identity-implementation.md` S0 for the query.

Root cause: an observation's identity is the entity **mentioned**, not the
measurement **taken**. `src/Publishers/EventPublisher.cs:105-112` keys the series on
Symbol / InstrumentId / source-slug. One article naming MSFT five times about
five different quantities produces five events with one identity, and
ThresholdEngine's `ObservationCache` is newest-write-wins -- so four are lost
and the fifth is **deterministically the last published**, not an arbitrary pick.
`ObservationCache.AddObservation` (`ThresholdEngine/src/Data/ObservationCache.cs:44`)
truncates both `date` and `asOf` to `.Date` at `:46-47`, so same-day claimants collapse onto
one exact key and the `BinarySearch` hit at `:54` overwrites in place at `:59`. The distinction
matters for the fix: "arbitrary" invites a tie-break rule, and there is no tie to break --
the survivor is whichever event the publish loop emitted last.

**That block is D-18's guard site, and R2/S4 cannot touch it without saying so.** It is
`CreateSeriesCollectedEvent`, it carries `// INTENT(D-18):` at `:99-104`, and `:112` --
`SeriesId = SentinelSeriesKey.ForNumeric(identity)` -- is the line D-18's own GUARD citation
names. See R2 below before designing anything against it.

This is the user's framing, and the measurement supports it:
> "each paragraph has either one topic with 1..n subjects, or one subject with 1..n topics"

The pipeline models neither. It models one subject, one value, last writer wins.

#### 2.2.1 The collision is LATENT, not actively harming output

Measured 2026-08-26. The collision above is mechanically real and reproduces every time.
**What does not reproduce is the harm**: today essentially nothing READS the clobbered keys.
This was the hole in the founding claim -- the loss was measured at the write seam and never
traced to a consumer.

*No pattern names a `SENTINEL:NUM:` key.* ThresholdEngine has **71 loaded patterns**
(`list_patterns` with `enabled_only=false` returns 71/71 enabled). Exactly **3** touch a Sentinel
key at all -- `sentinel-layoff-breadth`, `sentinel-challenger-divergence`,
`sentinel-financial-insider-breadth` -- and all three use the `SENTINEL:SECTOR:` prefix through
`GetSeriesCount` (`ThresholdEngine/src/Entities/PatternEvaluationContext.cs:312`) or
`GetSectorBreadth` (`:362`). **Both read `SeriesId` and `LatestDate` only and never touch `Value`**,
so a clobbered value cannot change their answer. **Zero patterns reference `SENTINEL:NUM:`** --
which is where 99.9% of Sentinel's numeric output lands: over the 30 days to 2026-08-26, 47,366 of
47,402 published observations mint a `SENTINEL:NUM:` key and 36 mint a bare owned-series key
(`SentinelSeriesKey.ForNumeric` namespaces everything outside `OwnedSeries`).

*The matrix path does not read them either, and the reason is broader than the projector's input
table.* Sentinel reaches `matrix_cells` through `public.macro_observations`, keyed
`{raw_content_id}:sig:{signal-slug}` -- 16,647 of 16,694 sentinel rows in 30 days carry the `:sig:`
infix and 16,690 of 16,694 `source_id` values are distinct. That key embeds the article id, so it
does not collapse across articles or across signals; the residual is two observations from ONE
article mapping to the SAME signal at the same `observation_time`, which the `ux_macro_obs_idem`
index resolves by upsert. But the projector ALSO evaluates a pattern's `signalExpression` over the
`ObservationCache` for its hard-data magnitude
(`ThresholdEngine/src/Workers/ObservationCellProjector.cs:810-821`), so "the cache is unread" is
**false** and must not be written down as the reason. The reason is the one above: no loaded
expression names a `SENTINEL:NUM:` key.

*The only key being minted bare today is BDIY, and it is a textbook loaded gun.* `BDIY` is in
`SentinelSeriesKey.OwnedSeries`, so it publishes under a BARE key that a pattern could read.
One article yields four rows. On 2026-08-26: level **3056**, increase **130**, highest **11793**,
lowest **290** -- and 290 is the last write. `baltic-freight-recession` computes
`bdiy.HasValue && bdiy.Value < 700m` -> **true**, and its signal `(290 - 1500) / 250` clamps to
**-3**, a maximal false recession signal, daily. **But that pattern is `"enabled": false` and is
absent from TE's loaded set entirely** -- it is the one repo pattern file of 72 that TE does not
load. The gun is loaded and pointed; nothing has pulled the trigger.

*The EXPOSED surface, though, is the whole `OwnedSeries` set -- and BDIY is its least dangerous
member.* Re-derived 2026-08-26: of the six bare keys Sentinel may mint, **two have LOADED, ENABLED
pattern readers today**, and they are safe only because the resolver is not currently producing
those symbols:

| owned key | loaded patterns reading it | Sentinel published rows |
|---|---|---|
| `CHALLENGER_JOB_CUTS` | `challenger-layoff-surge`, `challenger-vs-payroll`, `sentinel-challenger-divergence` | **0** (no row has ever carried this `Symbol`) |
| `TRUFLATION_CPI` | `truflation-vs-cpi` | **0** |
| `BDIY` | `baltic-freight-recession` (`enabled: false`, not loaded) | 156 published, 4/day currently |
| `ADP_EMPLOYMENT` | none | 7 published, last 2026-05-18 |
| `INDEED_POSTINGS` | none | 0 |
| `REDBOOK_SALES` | none | 0 |

`docs/BACKLOG.md` records the Challenger incident behind that first row: four datapoints from one
release (headline cuts, an AI-attributed subset, a YTD figure, and planned HIRES, which has the
opposite sign) resolving to one identity, with `challenger-layoff-surge` reaching a maximal false
**-2.58**. That entry now carries a flag: whether -2.58 was OBSERVED on a live evaluation or derived
as what would happen is unresolved, since no Sentinel row has ever carried that `Symbol`. Either way
the direction is the point -- **#988 raises resolution success**, which is exactly the change that
could start minting `CHALLENGER_JOB_CUTS` again. The latch here is a resolver outcome, not a design
decision, and nothing watches it.

**Consequence for the plan: R2/S4 is a LATENT-RISK fix, not an output-correctness emergency.**
It stops being latent the moment anyone enables `baltic-freight-recession`, adds a pattern naming
a `SENTINEL:NUM:` key, or adds a series to `OwnedSeries`. Each of those is a one-line config
change by someone who has no reason to know this. That is what R2 is buying down, and pricing it
honestly is what lets it be ranked against R1 and R3 rather than assumed to outrank them.

### 2.3 The atom exists and is thrown away (R3)

- `DslAst.cs:90` -- `DslClaim(ClaimKind, Subject, SubjectRefs, ClaimText, Slots, ...)`
- `DslToMergedExtractionAdapter.cs:640` -- passes `ClaimKind` into `DslClaimInput`
- **no further consumer.** `ExtractedObservation` has no ClaimKind and no Polarity column

CoD already distils what the design asked for. The schema cannot hold it.

### 2.4 Already fixed this session

| PR | Defect | Measurement |
|---|---|---|
| #987 | 59% of SecMaster embeddings stored un-normalised; HNSW cosine returned wrong neighbour | template v5 -> v6, `NormalizeToUnitLength` at the write seam |
| #988 | resolver matched publisher name over measurement subject | Rule 2b `hybrid_subject_description`, retry-on-miss only |
| #986 | two alerts fired on a boundary already crossed; cap refusals uncounted | promtool suite + ratchet |
| #983/#984/#985 | dream: 99.2% replay, citation corpus blind spot, index-hook rot | tooling, not pipeline |

D-18 (shipped 2026-08-13) stopped the first-party mnemonic corruption:
bare keys 1,878 -> 45 that day, namespaced 0 -> 1,742.

---

## 3. Remediation

Ordered by dependency, not by size. **R4 goes first**: without it we cannot prove
R1 worked, and the whole point of this plan is that unmeasured seams rot.

**Order corrected 2026-08-26.** The sections below still read R4, R1, R2, R3, R5, which is the
order they were drafted in. The order to WORK them is **R4 -> R1 -> R3 -> R2 -> R5**: R2 is
sequenced behind R3 (it needs the atom in the key), and behind R1 as well, because R1 and R3 are
losing data in production today while R2 protects keys no loaded pattern reads (2.2.1). R2 also
does not yet have a key that works (see R2). R5 is read-only and can run at any time.

### R4 -- measure event-kind parity  [do first]

- add a counter on the publish path, labelled by event kind and pipeline version:
  `sentinel_events_published_total{kind, pipeline}` (bounded cardinality: 2 x 2)
- add the known-bad control: a test asserting the counter is NON-ZERO for
  `kind="sector"` when a sector-coded observation passes the gate
- wire an alert on `kind="sector"` staying at zero across a full collection cycle
- **acceptance:** the metric reads ZERO on today's prod before R1 lands, and
  non-zero after. A metric that cannot show the bug is not a control.

Rationale: CLAUDE.md OBSERVABILITY -- "a signal riding on a bug dies with the fix."
This signal never existed, which is worse: 102 days of silence read as health.

### R1 -- port sector publish to v2

- call `CreateSectorEvent` from `RunV2ProductionAsync` on the same gate v1 uses
  (`AtlasSectorCode.HasValue && ResolutionConfidence >= 0.8 && Certainty in {Definite, Expected}`)
- keep the isolating try/catch, but **log at Warning and count failures** --
  the current block is silent by design and that is half the reason this went unseen
- persist `AtlasSectorCode` on `ExtractedObservation` (EF migration) so the loss is
  measurable next time; today it is inferable only from live traffic
- **D-entry:** `D-24 sector-event-pipeline-parity: INTENT both pipelines publish the
  same event KINDS / PRECOND any new pipeline path must publish every kind v1 does /
  GUARD ExtractionProcessor.RunV2ProductionAsync @ src/Workers/ExtractionProcessor.cs:<line> /
  TEST ExtractionProcessorV2SectorEventTests.V2_publishes_sector_events_like_v1`
  (path service-root-relative per `CARD_TEMPLATE.md:74`; `<line>` resolved by the agent once the
  call exists -- a `GUARD` with no `:line` is HIGH `D_entry_no_citation`. See the implementation
  plan's S2 for the full form.)
- **guard test contract:** delete the new call -> test goes RED. Assert through the
  real publish flow with a fake `IEventPublisher`, not by asserting the method exists
- correct the false XML doc at `:1741-1745` in the same PR

Effort: small. Risk: low. This is a missed port with a written intent.

### R2 -- identity = measurement, not entity  [PARKED 2026-08-27 on the user's decision]

**PARKED, and the thesis is not what parked it.** 5.1 (replacement) question 2 was put to the user --
expand the story to cover extraction, or park R2 with the measurement recorded -- and the answer was
**park**. The identity claim still holds and still reproduces (74.0% of published rows carrying an
instrument sit in a colliding group, re-measured 2026-08-27). What is missing is a discriminator: the
proposed key separates 24% of the colliding ROWS, and `claim_kind` -- now a real column, PR #997 -- turns
out not to discriminate inside one article about one issuer, because `cod_json_schema_v1.json` emits
`numbers[]` and `claims[]` as sibling arrays with no reference between them and
`"additionalProperties": false` on both. The revival path is therefore a CoD change (emit a
per-measurement claim reference), not a key redesign. `docs/BACKLOG.md` PARKED EPICS carries the full
measurement set and the four trip conditions that would un-park it, each with its re-check command.
Everything below is retained as the evidence a re-proposal must meet.

Not a patch. Sequenced behind R3 because the key must contain the atom.

**Rank, corrected 2026-08-26: R2 goes LAST of the fixes, not first.** It was carried as the
headline defect because it is the largest CHANGE. Blast radius is the opposite: R1 is losing a
whole event kind in production right now, R3 is discarding an atom the model already produces on
every article, and R2 is protecting a key nothing currently reads. Re-ranked by live exposure the
order is R4 (measure) -> R1 (live loss) -> R3 (live loss) -> R2 (latent), and R2 is additionally
under-specified (its key does not work, below).

**Framing, corrected 2026-08-26 by adversarial review and re-derived here: this buys down a LATENT
risk, it does not stop live corruption.** Zero of TE's 71 loaded patterns read a `SENTINEL:NUM:`
key, the three that touch `SENTINEL:SECTOR:` are value-independent, and the matrix path is keyed
per-article -- see 2.2.1 for the measurements. The only bare-key surface is BDIY, whose sole reader
is `enabled: false` and not loaded. R2 is worth doing because enabling one pattern or adding one
`OwnedSeries` entry converts the latent case to the live one silently; it is NOT worth doing on the
theory that today's matrix numbers are wrong because of it.

- key the series on (instrument, claim kind, unit), not instrument alone
- **THE PROPOSED KEY IS INSUFFICIENT AND MUST NOT BE DESIGNED AS IF IT WERE.** Measured
  2026-08-26 over published rows carrying an instrument (20,800 rows, 3,598 colliding groups):
  - **2,517 of 3,598 groups (70.0%) contain two or more members sharing one `unit`.** For those,
    adding `unit` to the key separates nothing, and `claim_kind` is not yet a column so it cannot
    be measured -- but see the worked case below, where it would not separate them either
  - **`unit` is not a normalised vocabulary**, so keying on the raw string FALSE-SEPARATES: `PCT`
    30,090 vs `percent` 2,605, `COUNT` 8,959 vs `count` 2,209, `INDEX` 521 vs `index` 256, `USD`
    alongside `million_USD` / `billion_USD` / `thousand_USD`. Collapsing to unit CLASSES first is
    mandatory, and doing so makes the key WORSE at its job, not better: shared-unit groups rise
    to 2,571 of 3,598 (**71.5%**). Normalisation and separation pull in opposite directions
  - **the missing axis is `period`, and it is populated on 2.80% of the population**
    (582 of 20,800 non-blank; 2.37% inside colliding groups). The information exists in the
    prose -- it is the `FY2024` / `Q3` / `FY27 Q2` in the description -- and is not extracted
    into the column
  - worked case, `raw_content_id=151480` (Walmart), ONE instrument, 20+ published rows, **every
    one `unit=PCT`**: gross margin FY2024 / FY2025 / FY2026, net margin FY2024 / FY2025 / FY2026,
    dividend yield, EPS surprise, revenue surprise, price reaction, Q3 sales guidance, full-year
    operating income guidance. Same instrument, same unit, and largely the same claim kind. The
    axes that actually separate them are the metric and the period
  - and `raw_content_id=146762`: gold futures settlement 4437.30 USD vs spot gold 4379.95 USD,
    one instrument, one unit, one article
  - this is not duplicate-row inflation: only 63 of 3,598 groups (1.75%) are pure exact duplicates
  - **so any S4 design story must solve period-or-equivalent FIRST.** Shipping
    (instrument, claim_kind, unit) alone leaves ~70% of collisions intact while reading as a
    completed fix, which is the failure mode this whole plan is about: a seam that reports success
- **THIS LANDS ON D-18's GUARD SITE. The design story MUST carry `supersedes D-18` or STOP.**
  The target block is `src/Publishers/EventPublisher.cs:105-112` -- `CreateSeriesCollectedEvent`,
  `// INTENT(D-18):` at `:99-104`, and `:112` is D-18's cited GUARD line
  (`GUARD SentinelSeriesKey.ForNumeric applied @ src/Publishers/EventPublisher.cs:112`).
  D-18 opens, verbatim: *"D-18 sentinel-series-key-ownership: INTENT a series key belongs to
  whoever MEASURED the number, and Sentinel is on BOTH sides of that line."* The full entry is
  ~2,000 words at `SentinelCollector/AGENT_README.md:93` -- **read it in full before designing**;
  it is heavily guard-tested and it shipped 2026-08-13 on measured production breakage.
  Per CLAUDE.md INTENT_FIDELITY: a brief that contradicts a D-entry without a NAMED supersession
  is a **STOP-and-escalate**, not an agent decision -- never route around it, never obey a stale
  entry, a human arbitrates. So either the S4 brief names "supersedes D-18" and rewrites the entry
  in the same PR (no tombstones), or the agent stops and reports the conflict
- **do not** ship the collision-refusal guard drafted this session: measured, it
  would have refused 74% of live traffic. Patch retained at
  `/tmp/sentinel-remediation/collision-guard/`, deliberately uncommitted.
  A guard that denies three quarters of production is an outage, not a gate
- migration path: new key alongside old, shadow-compare, cut over on evidence
- **open question for human decision, see 5.1 (replacement)** -- scope, not cutover; the cutover
  question originally posed here was withdrawn on measurement

### R3 -- carry ClaimKind and Polarity to storage  [prerequisite for R2]

- add `claim_kind` and `polarity` columns to `sentinel.extracted_observations` (EF migration)
- thread from `DslClaimInput` through the adapter to the entity
- `matrix_cells.polarity` already exists and is nullable -- the consumer is waiting
- **acceptance:** distinct-description count per (raw_content_id, instrument_id)
  becomes explainable by claim kind rather than unexplained collision

### R5 -- bound historical damage BY TIME  [measurement only, no fix]

- 287,763 matrix cells exist (2026-08-26). Unknown how many were computed from
  pre-2026-08-13 corrupted values or from collision-lossy caches
- **NOT a query against `contributing_observation_refs` + `source_provenance`.** Measured
  2026-08-26: refs populated on 0 rows, provenance on 223 (0.08%) with `rawContentId` null
  on all 223. There is no populated join path from a cell to its observations, and that
  empty chain is a separate and larger defect -- `docs/BACKLOG.md` §KNOWN DEFECTS
- what is available instead is a split on `evaluated_at` at the D-18 fix date:
  241,508 before / 46,255 after. **A coarse upper bound, not an attribution** -- it cannot
  distinguish a cell that consumed a corrupted value from one that did not. See the
  implementation plan's S5 for the method and its stated limits
- **do not backfill-to-green.** Report the number, decide separately.
  Card invariant: backfilling to make a dashboard look right is prohibited

---

## 4. Non-goals

**RL / LoRA retraining -- deferred, not rejected.**

The user raised this and it deserves a straight answer: the evidence says the
model is not the bottleneck.

- CoD extraction quality measured 0.78-0.87 (project record: low yield = wiring, not extraction)
- CoD already emits `ClaimKind` and subject refs correctly -- see 2.3
- the loss is downstream, in a schema with no column for what the model produced

Retraining a model whose output is discarded cannot improve output. Revisit
**after R3** lands, when the schema carries the atom and per-claim accuracy is
measurable end to end. Then a LoRA question has a metric to move.
Prior attempt: v6 MLP precision regression; base beat LoRA 4% vs 20% MAJOR.

---

## 5. Open questions -- human decision required

**5.1 R2 cutover -- WITHDRAWN AS POSED, 2026-08-26.** This question read: *"(instrument,
claim_kind, unit) ... changes the ThresholdEngine cache key and every signalExpression that reads
a Sentinel series. That is a `:sig:`-class string-contract change: all-or-none. Needs a decision on
cutover strategy before design."* **It escalated a measured-near-zero risk to a human.** There is no
`signalExpression` that reads a Sentinel numeric series: zero of TE's 71 loaded patterns reference
a `SENTINEL:NUM:` key, and the three touching `SENTINEL:SECTOR:` never read a value (2.2.1). The
cutover blast radius of a `SENTINEL:NUM:` re-key is **one config file** -- and it is
`baltic-freight-recession`, which is `enabled: false` and not loaded. Nothing here is `:sig:`-class
and nothing here is all-or-none. Asking a human to arbitrate it spent the scarcest thing in the plan
on the cheapest decision in it.

**5.1 (replacement) -- what R2's evidence actually asks.** Two questions, neither about cutover:

1. **Do we buy the cheap latch now instead of the redesign?** The whole live exposure is the bare
   owned-series keys (`SentinelSeriesKey.OwnedSeries`), where an extracted number can still occupy
   a key a pattern reads -- BDIY today, and any series added to that set tomorrow. A guard at that
   surface alone is small, testable, and independent of the identity model. R2 is the correct fix
   for the general case; this is the question of whether the general case needs to be fixed NOW.
2. **Does S4's scope expand to extraction, or does R2 park until it can be solved whole?**
   **ANSWERED 2026-08-27: PARK.** The user's decision, taken against the measurement below.
   The proposed key leaves ~70% of colliding GROUPS intact -- re-measured at the row level,
   12,084 of 15,959 colliding rows (75.7%) survive it -- because `period` is populated on 2.70% of
   the published-with-instrument population and `claim_kind` does not discriminate within one
   article about one issuer (PR #997: `numbers[]` and `claims[]` are sibling arrays with no
   reference between them). Solving it properly means making CoD emit a per-measurement claim
   reference, which is extraction work this plan does not contain. Shipping the partial key was
   never a third option -- every acceptance check would have read green.
   The park is recorded in `docs/BACKLOG.md` PARKED EPICS with four trip conditions and the command
   that reads each: a loaded pattern naming a `SENTINEL:NUM:` key, `baltic-freight-recession` being
   enabled, the resolver minting `CHALLENGER_JOB_CUTS` or `TRUFLATION_CPI`, or the golden corpus
   tripwires moving. None is tripped as of 2026-08-27.

The D-18 supersession requirement in R2 is UNCHANGED by this withdrawal: it is a design-intent
gate, not a cutover-risk one, and it still stands whatever is decided above.

**5.2 R5 disposition.** If a material share of 287k cells is provably corrupt, do we
mark them, recompute them, or leave them and start clean? Recompute is expensive
and re-runs the same lossy path unless R2 lands first.

**5.3 Scope of this plan vs PR #989.** #989 is the BACKLOG record. This is the plan.
Confirm both land, or fold.

---

## 6. How we know it worked

Re-run, expect movement:

| check | today | target |
|---|---|---|
| `sentinel_events_published_total{kind="sector"}` | metric absent | non-zero |
| colliding share of published observations | 73.8% | falls after R2 |
| instruments with mixed units | 1,577 / 2,733 | falls after R3 |
| sector events in a collection cycle | 0 (102 days) | > 0 |
| loaded patterns reading a `SENTINEL:NUM:` key | 0 of 71 | **re-check before R2 is scheduled** |
| colliding groups whose members share one `unit` | 70.0% | falls after R2, or R2 shipped partial |

**R2 is PARKED (2026-08-27), so its rows here are standing measurements, not pending targets** --
re-measured that day, 74.0% colliding share and 69.5% shared-unit groups. The latent-risk controls
below are now trip conditions: see `docs/BACKLOG.md` PARKED EPICS for all four and their commands.

The last two rows are the latent-risk controls. The first goes non-zero the moment someone adds a
pattern naming a Sentinel numeric key or enables `baltic-freight-recession`, which is what converts
R2 from latent to live -- re-run it rather than inheriting this table's zero. The second is the test
that R2 actually separated the collisions instead of only re-shaping the key:

```
grep -rl 'SENTINEL:NUM:' ThresholdEngine/config/patterns --include='*.json' | wc -l   # expect 0
```

A green run is not proof. Each item above ships with a control that fails when
the fix is removed.
