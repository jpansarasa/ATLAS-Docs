# Extraction identity + event-contract: implementation plan

**Status:** PLAN / for review -- NOT approved, NOTHING deploys from this doc.
**Companion:** `extraction-identity-remediation.md` (the WHY + evidence). This is the HOW.
**Audience:** a supervisor-mode session dispatching agents. Stories are dispatch-ready.

---

## 0. How to run this

- Templates: `.claude/skills/supervisor-mode/templates/`. Pick per story below, never hand-roll.
- **Compiles parallelise.** Each `compile.sh` owns a compose project keyed to its worktree
  (`scripts/devcontainer-owner.sh`). N stories in N worktrees compile simultaneously.
  Do not sequence agents behind each other. (Both templates were stale on this; corrected 2026-08-26.)
- Agents never push, never open PRs, never deploy, never restart a service. Supervisor owns the remote.
- `STATE.md`: `scripts/new-epic.sh extraction-identity` before the first dispatch.
- Long agent output -> `/tmp/sentinel-remediation/{slug}/`, not the report.

### Dependency graph

```
S0 golden corpus  ─┐
S1 parity metric  ─┼─ parallel, three worktrees, day one
S5 matrix damage  ─┘

S1 ──> S2  port sector publish to v2        (S1's metric is S2's proof)
S0 ──> S3  carry ClaimKind + Polarity       (corpus is S3's proof)
S3 ──> S4  identity key redesign            (BLOCKED: key insufficient + scope decision, §6.1)

S2 ⇄ S3   SAME FILE: SentinelCollector/src/Workers/ExtractionProcessor.cs
```

**S4 is LAST, and its priority was overstated.** It was ranked as the largest blast radius because
it is the largest change. Measured 2026-08-26, its live exposure is the smallest of the four:
**zero of TE's 71 loaded patterns read a `SENTINEL:NUM:` key**, the three that touch a Sentinel key
use `SENTINEL:SECTOR:` through value-independent counts, and the matrix path is keyed
`{raw_content_id}:sig:{slug}` per article. See `extraction-identity-remediation.md` §2.2.1 for the
derivation, and §6.1 below for the consequence. S1/S2 (an event kind lost in production for 102
days) and S0/S3 (an atom discarded on every article) are the live losses; S4 buys down a latent one.

**S4 does not start without a human decision.** See §6.1 -- and note that the decision is no longer
the cutover question this plan originally escalated, which was withdrawn on measurement.

**S2 and S3 edit the same file and the dependency arrows do not say so.** They have no logical
dependency -- S2 is gated on S1, S3 on S0 -- so the graph reads as "run them in parallel", and
they will collide on merge. Verified 2026-08-26:

| story | region of `ExtractionProcessor.cs` | also touches |
|---|---|---|
| S2 | `RunV2ProductionAsync` `:1746` + its XML doc `:1741-1745` | -- |
| S3 | v1 `new ExtractedObservation` block `:687` | `V2ExtractionPipeline.cs:180`, `ReExtractBackgroundService.cs:464` and `:600`, the entity class, the adapter |

**How to handle it: do NOT sequence the agents.** The edits are ~1,050 lines apart and touch
different members, so this is a textual collision, not a semantic one -- git merges it cleanly in
the common case. Compiles parallelise per worktree (§0), and serialising two independent stories
to dodge a merge that git handles is a real cost paid against an unlikely one.

What the supervisor owes instead, at merge time:

1. **Merge order is whichever lands first; the SECOND one rebases onto main.** Neither story is
   privileged.
2. **The rebased story re-runs `bash SentinelCollector/.devcontainer/compile.sh` before it pushes,
   with no `--no-test`.** This is not optional politeness: the push-guard marker is keyed to
   `HEAD^{tree}`, and a rebase remaps the root tree, so the pre-rebase marker does not carry over
   and the guard will refuse the push. Re-running the suite is the only way through, and it is
   also the only thing that proves the two changes compile together.
3. **If the rebase conflicts, the S3 agent resolves it** -- S3 is the larger, multi-file change and
   its author has the context for both construction sites. Do not resolve it in the supervisor
   session.
4. **Line citations in the S2 D-entry are resolved after the rebase, not before.** S3 inserting
   columns into the entity shifts nothing at `:1746` today, but a rebase that moves either edit
   invalidates a `:line` captured earlier -- see §3's placeholder note.

---

## 1. S0 -- golden corpus from landed raw data  [do first, parallel]

Origin: the user's question -- *"Why not take some of the raw data we land and write tests to
validate the fault is fixed? We keep everything we collect."* Correct, and it is the foundation
for every story after it. Without it these fixes are verified by assertion, not by real data.

**What we actually have** (measured 2026-08-26):

- `/opt/ai-inference/raw-data/sentinel` -- **60,038 files, 5.8 GB**, and it is a **30-day ROLLING
  WINDOW, not an archive** (corrected 2026-08-26 by S0; this doc said "retained since 2025-01-01").
  `StaleContentPrunerService` enforces a 30-day `raw_content` retention and the oldest retained file
  on every one of the 14 sources is 2026-07-27. **A corpus drawn from it cannot be re-drawn**, which
  is why S0's fixtures are committed; widening it means selecting RECENT articles, never older ones
- `sentinel.raw_content.raw_file_path` joins them to observations; 48,211 of 100,383 published rows
  (48%) carry a path, measured 2026-08-26. This absolute moves with the window in BOTH directions
- worst-collision articles retain their raw HTML, e.g. `raw_content_id=154787` (rss-mirror,
  60 published observations from one article)

**Template:** `implementation-fix` (FRESH). **Branch:** `test/golden-corpus-extraction`.

**Deliverables**

1. A fixture extractor -- given a `raw_content_id`, copy the raw file plus its expected
   observations into `SentinelCollector/tests/SentinelCollector.UnitTests/Fixtures/GoldenCorpus/`.
   Committed fixtures, not a live DB dependency: the test suite must run with no database.
2. A corpus of ~30 articles, each with a manifest row stating *what it proves*:
   - max-collision: **154787 only, of the three this doc named.** Measured 2026-08-26: `136367` and
     `146606` carry no Symbol and no instrument on any published row, so the identity falls to the
     `{source}:{description-slug}` leg and SEPARATES them (60 rows -> 15 measurements -> 15 keys,
     and 60 -> 32 -> 31). They are duplicate-extraction articles, not collision ones. **Counting
     published rows is not counting keys**, and that is the whole defect. Landed replacements:
     154787, 151480, 153630, 152410, 152406, 151362, 146762
   - mixed-unit: instruments carrying percent + count + dollars under one identity
   - the Challenger case (`challenger-rss`, 460 obs / 12 published in August)
   - the outliers that work today: **not TSA, FRED or OFR.** TSA last published 2026-02-07 (729 of
     83,160 observations ever published) so it is not a working control at the publish seam, and its
     70 retained raw files carry zero published rows; FRED and OFR are not Sentinel sources and have
     no `raw_content` at all. S0 uses a `separating-today` bucket instead -- articles measured to give
     every one of their measurements its own key now. See docs/BACKLOG.md for the TSA entry
   - clean single-observation controls
3. A test class driving the real extraction path over each fixture.

**Selection SQL for the two buckets that had no rule.** Named IDs cover max-collision and
Challenger; mixed-unit and clean-control were prose. Both queries below were run 2026-08-26 and
return usable candidates -- 819 mixed-unit groups and 844 clean controls, all with a raw file on
disk. The `raw_content.raw_file_path IS NOT NULL` join is load-bearing: only 29,197 observations
carry a path, and a fixture with no raw file cannot be built.

*Mixed-unit.* **`unit` is NOT a normalised vocabulary** -- `PCT` and `percent` and `COUNT` and
`count` all coexist -- so a bare `count(DISTINCT unit)` counts spelling as unit mixing. Collapse to
classes first:

```sql
WITH cls AS (
  SELECT o.raw_content_id, o.instrument_id,
         CASE WHEN upper(o.unit) IN ('PCT','PERCENT','%')            THEN 'PCT'
              WHEN upper(o.unit) IN ('COUNT','SHARES')               THEN 'COUNT'
              WHEN upper(o.unit) LIKE '%USD%' OR upper(o.unit)='EUR' THEN 'CURRENCY'
              ELSE upper(coalesce(nullif(o.unit,''),'UNKNOWN')) END AS unit_class
  FROM sentinel.extracted_observations o
  JOIN sentinel.raw_content rc ON rc.id = o.raw_content_id
  WHERE o.published_at IS NOT NULL AND o.instrument_id IS NOT NULL
    AND rc.raw_file_path IS NOT NULL)
SELECT raw_content_id, instrument_id, count(DISTINCT unit_class) AS unit_classes, count(*) AS obs
FROM cls GROUP BY 1,2 HAVING count(DISTINCT unit_class) >= 3
ORDER BY 3 DESC, 4 DESC LIMIT 10;
```

Top candidates, re-run 2026-08-26 (815 groups, not 819): `151468` (6 classes / 32 obs), `154938`
(6 / 21), `151480` (5 / 58), `152410` (5 / 49), `152553` (5 / 44). **`150183` is NOT among them** --
it has no published row carrying an instrument, which this query requires, so it cannot appear in
its own output. Take 5-6 from the head of this list.

*Clean single-observation controls* -- one published observation for the whole article.
**Corrected 2026-08-26 by S0: these do NOT detect a false-positive collapse.** With one
published row the measurement count and the key count are both forced to 1, so `keys <
measurements` cannot hold on them whatever the key does. What they DO separate is the
publish GATE from the key -- `control-155505` extracts 58 rows and publishes one, which is
gating, not collapse, and the two must not be conflated:

```sql
SELECT o.raw_content_id, o.source, count(*) AS published_obs
FROM sentinel.extracted_observations o
JOIN sentinel.raw_content rc ON rc.id = o.raw_content_id
WHERE o.published_at IS NOT NULL AND o.instrument_id IS NOT NULL
  AND rc.raw_file_path IS NOT NULL
GROUP BY 1,2 HAVING count(*) = 1
ORDER BY 1 DESC LIMIT 10;
```

Take 5-6, and **spread them across `source`** (the head of that list is `rss`-heavy); a control set
drawn from one source proves the corpus works on one source.

**Two counts here do not mean what §8's row says**, and the S0 agent should not reconcile them by
changing its query. Measured 2026-08-26 over published rows with an instrument: 1,562 of 2,735
instruments have `count(DISTINCT unit) > 1`; collapsing the synonym classes above drops that to
**1,422 of 2,735** -- ~140 were pure spelling. The strict bucket this plan names in prose,
percent AND count AND dollars together under one identity, is **289 instruments**. §8's
`1,577 / 2,733` is the raw-string form measured a day earlier; the corpus is drawn from the 289.
These absolutes move continuously as the pipeline publishes -- see `docs/BACKLOG.md`, the ratios
are the durable claim.

**KNOWN-BAD CONTROL (mandatory).** The corpus must be **RED today** on the collision cases.
A corpus that passes against the current broken pipeline proves nothing and is worse than none --
it converts an open defect into a green dashboard. Record each expected-RED case and its reason
in the manifest.

**Acceptance**

- `nerdctl compose exec -T sentinel-collector-dev dotnet test --filter 'DisplayName~GoldenCorpus'`
  runs N cases with M documented failures.
- **Use `DisplayName~`, never `Name~`.** xUnit exposes `DisplayName` and `FullyQualifiedName`;
  `Name~` matches ZERO tests and still exits 0, so a filtered run that tested nothing reads as
  a pass. Measured: `Name~should_return` = 0 tests, `DisplayName~` = 304.
- Report the N/M split. Those numbers are the baseline every later story moves.

**Watch for:** fixture size. 5.7 GB is the corpus, not the commit. Trim each fixture to the
article body; do not commit 60 MB of HTML.

---

## 2. S1 -- event-kind parity metric  [do first, parallel]

**Template:** `implementation-fix` (FRESH). **Branch:** `fix/sentinel-event-kind-metric`.

The reason R1 went unseen for 102 days is that **no metric distinguishes event KIND at the
publish seam.** 16 sentinel metrics DO match `sentinel.*(sector|event|publish)` -- sector
*tagging* and *parsing* are observed upstream -- but sector *publishing* is observed nowhere.
(An earlier "zero metrics exist" claim in this plan was wrong: the probe hit `localhost:9090`,
which is unreachable; Prometheus answers on the container IP.)
**FIRST STEP, before writing any code: enumerate what the existing 16 already cover.** If one
can carry a `kind` label instead of a new metric being added, take that -- CLAUDE.md
OBSERVABILITY warns against double-counting the same act at two layers.

**Deliverables**

1. A publish-seam counter carrying `{kind, pipeline}` -- either a new
   `sentinel_events_published_total` or a `kind` label on an existing metric, decided by the
   enumeration above. Cardinality is bounded and small: kind in {series, sector},
   pipeline in {v1, v2}.
2. Alert: `kind="sector"` at zero -> P3_NOTIFY. Wire it; an unobserved metric is waste plus
   complexity. **"Across a full collection cycle" is not a `for:` -- the rule needs numbers, and
   these are measured, not guessed.** Sector-eligible observations arrive at **1.76/h** since the
   `81da1ed4` cutover and **7.82/h** over the last 7 days, and the gaps between them are long:
   **p99 inter-arrival 11.76h, max 338.75h**. So:
   - **increase window `[24h]`, `for: 2h`.** 24h clears the p99 gap with headroom; a `[6h]`
     window sits inside it and would flap. Precedent in the same file:
     `SentinelCandidateSurfaceFilterCollapsed` (`deployment/artifacts/monitoring/alerts/sentinel.yml:601`)
     runs `[6h]` + `for: 1h` and carries a written note that a `[1h]` window "legitimately touches
     zero during quiet periods and would flap".
   - **AND a clause requiring the INPUT to be non-zero over the same window**, so a genuinely
     quiet 24h cannot fire. Without it the 338.75h max gap guarantees false pages. The upstream
     sector-tagging metrics are the candidate denominator (`sentinel_atlas_sector_parse_outcome_total`,
     `sentinel_sector_tagger_request_total`) -- the §2 enumeration step picks which, and confirms
     it actually counts eligible observations rather than attempts.
   - a bare `sum(increase(...{kind="sector"}[24h])) == 0` with no input clause is a **can't-fail
     alert in reverse**: it fires on every quiet night and gets muted, which is how it stops
     working. That failure mode is already annotated at `sentinel.yml:474`.
3. `deployment/tests/alerts/*_test.yml` case for the new rule -- both directions: fires when
   sector publishes are zero WHILE input is flowing, and stays silent when both are zero.

**Acceptance**

- On today's prod the metric reads **ZERO** for `kind="sector"`. That zero is the control:
  it is what proves S2 worked.
- `bash deployment/tests/alerts/run.sh` green, and the new rule mutation-verified -- invert it,
  confirm RED, restore.

**Do not** let this story also fix the publish gap. S1 measures; S2 fixes. Landing them together
destroys the before/after reading that makes the fix provable.

---

## 3. S2 -- port sector publish to the v2 path  [after S1]

**Template:** `implementation-fix` (FRESH). **Branch:** `fix/sentinel-v2-sector-events`.

**The defect, verbatim for the brief:**

> `CreateSectorEvent` has exactly one production call site, `ExtractionProcessor.cs:876`, inside
> `ProcessSingleArticleAsync`. The v2 branch returns at `:599`, before reaching it.
> `RunV2ProductionAsync` (`:1746`) never calls it. Every August source with published rows is in
> `V2EnabledSources`, so 100% of published extraction has taken the v2 path since `81da1ed4`
> (2026-05-16). Sector events: zero for 102 days.
>
> `RunV2ProductionAsync`'s XML doc at `:1741-1745` claims it "publishes SeriesCollectedEvent
> identically to the v1 branch (plan D8: contract unchanged)". v1 publishes TWO event kinds;
> v2 publishes one. The comment records the intent the code lost.

**Deliverables**

1. Call `CreateSectorEvent` from `RunV2ProductionAsync` on the identical v1 gate:
   `AtlasSectorCode.HasValue && ResolutionConfidence >= 0.8f && Certainty is Definite or Expected`.
2. Keep the isolating try/catch, but **log at Warning and increment a failure counter**. The
   current block is silent by design, and that is half the reason this went unseen for 102 days.
3. **No migration needed -- `atlas_sector_code` already exists** (`20260509234900`, 2026-05-09)
   and is populated on the v2 path. Use it instead as the port's verification target: **4,324
   rows** since the `81da1ed4` cutover satisfy the sector gate EXACTLY as the code writes it --
   sector code + confidence + certainty, with no `published_at` and no instrument condition
   (the regular-publish gate is a separate filter on the same list; do not borrow its terms).
   Report that count before and after; it is the backlog the port has to clear.
   Re-check:
   ```sql
   SELECT count(*) FROM sentinel.extracted_observations
   WHERE atlas_sector_code IS NOT NULL AND resolution_confidence >= 0.8
     AND certainty IN ('Definite','Expected') AND extracted_at >= '2026-05-16';
   ```
4. Correct the false XML doc in the same commit.

**Migration (HARD_STOP -- restate in the brief):**

```
nerdctl compose exec -T sentinel-collector-dev sh -c \
  "cd /workspace/SentinelCollector/src && dotnet ef migrations add PersistAtlasSectorCode --output-dir Data/Migrations"
```

Never hand-write a migration `.cs`. Missing `Designer.cs` -> EF records it in
`__EFMigrationsHistory` with the schema unchanged. `dotnet tool restore` first if `dotnet-ef`
is missing (local tool manifest, not a global install).

**D-entry (new, `SentinelCollector/AGENT_README.md`):**

```
D-24 sector-event-pipeline-parity: INTENT both pipelines publish the same event KINDS, because a
new pipeline path that silently publishes fewer is indistinguishable from healthy /
PRECOND any path reaching publish must emit every kind v1 emits /
GUARD ExtractionProcessor.RunV2ProductionAsync @ src/Workers/ExtractionProcessor.cs:<line> /
TEST ExtractionProcessorV2SectorEventTests.V2_publishes_sector_events_like_v1
```

**The citation form is not cosmetic -- `audit.sh` fails the entry on both counts.**
`CARD_TEMPLATE.md:74`: *"file:line is relative to the service root (the dir containing
AGENT_README.md)"*, so the path is `src/Workers/ExtractionProcessor.cs`, NEVER
`SentinelCollector/src/...`. Every landed entry in the card confirms it -- D-18's own guard reads
`@ src/Publishers/EventPublisher.cs:112`. And `CARD_TEMPLATE.md:123-126` reports a `GUARD` with no
`:line` as **MISSING** (HIGH `D_entry_no_citation`), not as a lesser malformed signal.

**`<line>` is a placeholder and must not be committed as one.** The guard site does not exist
until S2 adds the `CreateSectorEvent` call inside `RunV2ProductionAsync`, so the number is
unknowable while this plan is being written. The agent resolves it LAST, after the call is
written and the file has stopped moving: take the line of the new call itself (the `// INTENT(D-24):`
comment sits immediately above it), then re-read the file at that line to confirm it landed on the
call and not on a comment or a blank -- `verify-citations.py` is content-blind and reads a citation
that drifted onto a comment as GREEN.

ATOMIC_SET, all-or-none: D-entry + `// INTENT(D-24):` at the call site + the call + the guard test.

**Guard test contract.** Construct a v2 run over an observation carrying an `AtlasSectorCode`,
assert a sector event reaches a fake `IEventPublisher` through the real flow. Mock only the
external client. **Delete the call -> test goes RED.** Asserting the method exists is not a
guard test.

**Acceptance**

- S1's `kind="sector"` counter moves from zero to non-zero after deploy.
- S0's sector fixtures turn green.
- `bash SentinelCollector/.devcontainer/compile.sh` -- 0 errors, 0 warnings, all tests pass.
  Capture the full log and the real `$?`; `| grep | tail` hides Permission-denied and swallows
  the exit code.

**Deploy is the supervisor's, not the agent's:**

```
ansible-playbook playbooks/deploy.yml --tags sentinel-collector --skip-tags build \
  -e "scoped_restart=true scoped_services=sentinel-collector"
```

Build first (`SentinelCollector/.devcontainer/build.sh --no-cache`) -- `--skip-tags build`
deploys the current `:latest`, it does not build. Verify with `nerdctl container inspect`,
never bare `nerdctl inspect` (that returns the IMAGE and its build time, so a deploy "verified"
that way compared the fresh image to itself).

---

## 4. S3 -- carry ClaimKind and Polarity to storage  [after S0]

**Template:** `implementation-fix` (FRESH). **Branch:** `fix/sentinel-claimkind-persistence`.

CoD already produces the atom the design asked for. The schema cannot hold it:

- `DslAst.cs:90` -- `DslClaim(ClaimKind, Subject, SubjectRefs, ClaimText, Slots, SourceSpan, Line)`
- `DslToMergedExtractionAdapter.cs:640` -- passes `ClaimKind` into `DslClaimInput`
- **no further consumer.** `ExtractedObservation` has no ClaimKind and no Polarity column
- `matrix_cells.polarity` already exists and is nullable -- the consumer is waiting

**Deliverables**

1. EF migration adding `claim_kind` and `polarity` to `sentinel.extracted_observations`.
2. Thread both from `DslClaimInput` through the adapter to the entity.
3. Extend the S0 corpus manifest: each fixture's expected claim kinds become assertions.

**Acceptance**

- Distinct-description count per `(raw_content_id, instrument_id)` becomes *explainable by claim
  kind* rather than an unexplained collision. Query for the report:

```sql
SELECT claim_kind, count(*), count(DISTINCT unit)
FROM sentinel.extracted_observations
WHERE published_at IS NOT NULL AND extracted_at >= now() - interval '7 days'
GROUP BY 1 ORDER BY 2 DESC;
```

- S0 corpus: claim-kind assertions green.

---

## 5. S5 -- bound historical matrix damage BY TIME  [read-only, parallel, anytime]  **STATUS: DONE (2026-08-26)**

**Template:** `recon-measurement`. Read-only. **No branch, no code.**

**The provenance method this story originally specified does not exist. Do not attempt it.**
An earlier draft asked for the share of `matrix_cells` whose
`contributing_observation_refs` / `source_provenance` chain reaches a corrupted observation.
Measured 2026-08-26 against `public.matrix_cells` (288,412 rows):

| column | rows populated | share |
|---|---|---|
| `contributing_observation_refs` | **0** | 0% |
| `source_provenance` | 223 | 0.077% |
| `source_provenance->>'rawContentId'` non-null | **0 of those 223** | 0% |

There is no populated join path from a matrix cell back to the observations that produced it.
Dispatched as originally written this story returns a false near-0% "corrupted" share into the
§6.2 human decision, or the agent silently invents a substitute methodology nobody authorised.
The empty chain is a defect in its own right and larger than this story -- it is filed under
`docs/BACKLOG.md` §KNOWN DEFECTS with its own re-check.

**The question S5 was scoped to answer was "how many cells existed before D-18 shipped" -- and
the answer to the question actually worth asking is narrower than that by two orders of
magnitude.** The coarse time bound comes first because it is still the honest ceiling over the
whole table, then the narrowing that survived adversarial review on 2026-08-26.

The first draft split on `evaluated_at`. That was wrong: `evaluated_at` is the nominal date being
evaluated, not when the row was computed. `created_at` is the true write/projection timestamp,
and it disagrees with `evaluated_at` on 1,166 rows (evaluated for a pre-fix date but computed
post-fix, on clean data) and postdates it by more than a day on 26,205 rows (9.09%, avg 5.38
days, max 210 days) -- a continuous rolling recompute since 2026-05-30, not one backfill burst.

```sql
SELECT count(*) AS total,
       count(*) FILTER (WHERE created_at  < '2026-08-13') AS created_before_fix,
       count(*) FILTER (WHERE evaluated_at < '2026-08-13') AS evaluated_before_fix,
       count(*) FILTER (WHERE evaluated_at < '2026-08-13' AND created_at >= '2026-08-13')
         AS eval_pre_created_post
FROM public.matrix_cells;
```

Measured 2026-08-26: **240,353 created before the fix** (`created_at`, the correct discriminator)
vs 241,519 evaluated before it (`evaluated_at`, superseded) / 288,467 total. These move as the
projector keeps writing; re-run rather than quoting them.

**THIS IS A COARSE UPPER BOUND, NOT AN ATTRIBUTION. 240,353 is every cell that EXISTED while the
bug was live, not cells PROVEN to have consumed a bad value.** State it in exactly those words in
any report. A number reported without that sentence attached will be read as an attribution and
will drive the §6.2 decision wrongly. It says nothing at all about collision loss, which has no
fix date and is still happening.

**THE BOUND DOES NARROW, structurally rather than statistically.** D-18 corrupted the
`ObservationCache`, and only one of the projector's two magnitude paths reads that cache:
`isNews ? {the :sig: decay sum} : EvaluateHardMagnitudeAsync(...)`
(`ThresholdEngine/src/Workers/ObservationCellProjector.cs:686-688`, whose own comment reads "News
groups NEVER use SignalExpression"). A news-path cell therefore cannot consume a polluted cache
value however corrupt the cache was. Three nested populations, all measured 2026-08-26:

| population | pre-fix cells | what it is |
|---|---|---|
| whole table | **240,353** | every cell alive while the bug was live (coarse ceiling) |
| hard-data path | **5,929** | magnitude came from `signalExpression` + `ObservationCache` |
| reads a polluted mnemonic | **1,111** | of those, patterns reading a series Sentinel published under |
| carries the ±3 signature | **242** | of those, the identified floor |

Corroborated in data as well as code: **zero** cells at `abs(signal) = 3.0` exist on any
`sentinel` (280,423) or `ofr` (3,487) row -- all 418 exact clamps sit on the 4,334 `fred` rows --
and 99.71% of `sentinel` rows in `public.macro_observations` carry the `:sig:` news infix.

**True damage is in `[242, 1111]` on the mechanism's own path. 242 is cells MATCHING A SIGNATURE,
not cells PROVEN damaged** -- that sentence travels with the number exactly like the one above.
The floor spans 9 patterns, `created_at` 2026-06-25..2026-08-12, detected with `abs(signal) = 3.0`
EXACTLY (the `SignalUtilities.ClampSignal` bound,
`ThresholdEngine/src/Services/PatternEvaluationService.cs:344`). Full enumeration, the direct
published-value evidence that makes the clamps arithmetically impossible from real data, the three
both-era clampers excluded as controls, and every re-check query live in `docs/BACKLOG.md`
§KNOWN DEFECTS (search "S5 (`docs/proposals/...`) is DONE") rather than being duplicated here.

**Four traps this measurement fell into and had to be corrected out of** -- any one of them alone
flips the answer, and all four are recorded in full in the BACKLOG entry:

1. `abs(signal) >= 2.9` is not a clamp detector. The projector falls back to the raw, UNCLAMPED
   mean when an expression throws (`ObservationCellProjector.cs:826-839`), so that threshold
   sweeps in raw means of 1,777,000 and 225,000. Use `= 3.0`.
2. The largest row of the naive result (`obs:cpi-headline-yoy:sentinel`, 110 pre / 0 post) is
   news-path AND all 110 are `> 3.0`. Disqualified twice: wrong path, and not clamps.
3. "Stops dead at the fix boundary" is false for 5 of 12 patterns. They stop when their OWN
   mnemonic's junk stops -- INDPRO's last Sentinel publication and `industrial-production`'s only
   clamp are the same day, 2026-07-17, 27 days before the fix.
4. **D-18's card named six polluted mnemonics; the measured set is thirty-seven.** *Re-measured
   2026-08-26 -- supersedes the "fifteen" recorded in the first pass of this section, which was
   itself an undercount.* D-18's own 30-day window holds 37 distinct bare first-party keys, not six
   and not fifteen; the fifteen missed DEXJPUS (37 events, third-largest in the window, ahead of
   DCOILWTICO). The list was cut with nothing marking it as cut. Testing this story's claim against
   those six FALSELY REFUTES it, because 5 of the 9 floor patterns read a polluted mnemonic outside
   D-18's list. The card now records the window list as MEASURED and OPEN, so the next reader does
   not treat it as a scope.

**What the sample structurally cannot contain**, to be stated before concluding -- every item
biases the floor DOWNWARD:

- no per-cell record of which observations fed it (the table above)
- damage that never reached the clamp: a polluted value moving a signal 0.3 -> 1.7 is real
  corruption with no signature. Only saturation is visible, so 242 is a floor of a floor
- cells since recomputed. The projector heals on rewrite, and 26,205 rows show a recompute lag,
  so a damaged cell recomputed post-fix on clean data no longer carries the signature
- raw-mean fallback cells, also corrupted (the expression threw) but carrying no clamp
- a 0.29% residual: 128 `sentinel` rows lack the `:sig:` infix, so a group made entirely of them
  would take the hard-data path. The 5,929 figure bounds that residual, it does not exclude it
- cells are written by the WS3 projector only -- confirmed 2026-08-26: 100% of `pattern_id`
  values fall under the `obs:` or `sentinel:` prefixes it writes, zero from any third writer
- `evaluated_at` is the nominal date, not the computation time (see the `created_at` correction)

**Two methods are closed; do not re-attempt either.** (1) Mean-signal-shift attribution across the
fix boundary, refuted by its own control -- `obs:natural-gas-price:sentinel`, D-18's named CLEAN
series, shifts +0.694 where the implicated `obs:oil-price:sentinel` shifts +1.437, and unrelated
patterns shift further in the opposite direction (`obs:fed-funds-rate:sentinel` -2.185,
`obs:dxy-dollar-index:sentinel` -1.515). It fails for a second reason now visible: every pattern
it tested was `:sentinel`, i.e. news-path, which never reads the corrupted cache. (2) Tightening
the bound with a before/after COUNT on any clamp-like threshold -- post-fix eras are 1-3
projection batches for 10 of 12 clamping patterns, so the split has almost no power and does not
carry the claim in either direction. What worked was structural (which path reaches the cache)
plus direct (what values Sentinel actually published, recoverable from `sentinel.events`).

**S5 is closed as a measurement.** The remaining question is disposition, not a tighter number,
and disposition is §6.2's human decision.

**Constraints for the brief -- restate verbatim:**

- **psql is SELECT-only.** No INSERT/UPDATE/DELETE/ALTER to fix state; fix the root cause in
  the app. This HARD_STOP alone has proven necessary but not sufficient -- an agent that had
  the rule still ALTERed prod.
- **Do not backfill to green.** Report the number. Disposition is a human decision (§6.2).
- **Do not substitute a different method.** If the time bound is judged insufficient, say so
  and stop; inventing a provenance proxy is out of scope for this story.

---

## 6. Blocked -- human decision required before dispatch

### 6.1 S4, identity key redesign (R2)  [last of the stories, and blocked on scope, not cutover]

The fix is to key the series on **(instrument, claim kind, unit)** rather than instrument alone.

**That key does not do the job, and the design story has to start there.** Measured 2026-08-26
over published rows carrying an instrument -- 20,800 rows, 3,598 colliding groups:

| measurement | value | consequence for the key |
|---|---|---|
| groups with >=2 members sharing one `unit` | **2,517 / 3,598 (70.0%)** | `unit` separates nothing for them |
| same, after collapsing `unit` synonyms to classes | **2,571 / 3,598 (71.5%)** | normalising makes separation WORSE |
| `period` non-blank | **582 / 20,800 (2.80%)** | the axis that would separate them is not extracted |
| `period` non-blank inside colliding groups | 364 / 15,361 (2.37%) | ditto, and slightly worse where it matters |
| groups that are pure exact duplicates | 63 / 3,598 (1.75%) | duplicate rows are NOT the explanation |

`unit` is not a normalised vocabulary -- `PCT` 30,090 vs `percent` 2,605, `COUNT` 8,959 vs
`count` 2,209, `INDEX` 521 vs `index` 256, `USD` alongside `million_USD` / `billion_USD` /
`thousand_USD` -- so keying on the raw string ALSO false-separates one measurement into several
series. Any design that uses `unit` must normalise first, and normalising costs separation.

Worked case for the brief, `raw_content_id=151480` (Walmart): one instrument, 20+ published rows,
**every one `unit=PCT`** -- gross margin FY2024 / FY2025 / FY2026, net margin FY2024 / FY2025 /
FY2026, dividend yield, EPS surprise, revenue surprise, price reaction, Q3 sales guidance,
full-year operating income guidance. Same instrument, same unit, largely the same claim kind.
The axes that separate them are the metric and the PERIOD, and the period is in the prose
(`FY2024`, `Q3`, `FY27 Q2`) while the `period` column is empty. Second case,
`raw_content_id=146762`: gold futures settlement 4437.30 USD vs spot gold 4379.95 USD, one
instrument, one unit, one article.

**So the S4 spec must solve period-or-equivalent BEFORE the key shape is settled.** A story that
ships (instrument, claim_kind, unit) and stops leaves ~70% of collisions intact while every
acceptance check reads green -- the exact seam-reports-success failure this plan exists to fix.
Note that this makes S4 depend on extraction work (populating `period`, or carrying the metric
identity), not only on S3's `claim_kind` column.

**The cutover question this section used to ask is WITHDRAWN, 2026-08-26.** It read: *"That changes
the ThresholdEngine `ObservationCache` key and every `signalExpression` reading a Sentinel series.
It is a `:sig:`-class string-contract change: all-or-none. Needs a cutover decision -- new key
alongside old with a shadow compare, or a hard cut."* Measured, there is no such `signalExpression`:
zero of TE's 71 loaded patterns reference a `SENTINEL:NUM:` key. The re-key's entire consumer-side
blast radius is one pattern file, `baltic-freight-recession`, which is `enabled: false` and is not
in TE's loaded set. **A shadow-compare cutover is ceremony for a change with no live reader**, and
escalating it to a human spent the plan's scarcest resource on its cheapest decision. Record this:
the failure was measuring a defect at the write seam and never asking what READS it.

**What S4 is actually blocked on** is scope, and it is `spec-plan-authoring` until it is answered:
either the story expands to cover extracting `period` (or an equivalent metric identity), or R2 is
parked with the measurement above recorded and the latent risk accepted. Shipping the partial key
is not a third option. See `extraction-identity-remediation.md` §5.1 (replacement) for the paired
question -- whether to instead buy a small guard at the bare `OwnedSeries` key surface, which is
where the whole live exposure sits and which is independent of the identity model.

**S4 LANDS ON D-18's GUARD SITE. The brief MUST carry `supersedes D-18` or the agent STOPS.**
The site is `src/Publishers/EventPublisher.cs:105-112` -- `CreateSeriesCollectedEvent`, with
`// INTENT(D-18):` at `:99-104` and `:112` (`SeriesId = SentinelSeriesKey.ForNumeric(identity)`)
being the line D-18's own GUARD citation names
(`GUARD SentinelSeriesKey.ForNumeric applied @ src/Publishers/EventPublisher.cs:112`).

D-18 opens, verbatim -- and this is the only part to quote, the rest must be read, not
paraphrased into a brief:

> D-18 sentinel-series-key-ownership: INTENT a series key belongs to whoever MEASURED the number,
> and Sentinel is on BOTH sides of that line.

The full entry runs ~2,000 words at `SentinelCollector/AGENT_README.md:83`. **Whoever authors the
S4 spec reads it end to end first.** It shipped 2026-08-13 against measured production breakage,
it is heavily guard-tested, and it turns on a polarity ("is this a series Sentinel OWNS?" rather
than "does this key collide?") that a naive re-key would silently invert.

Per CLAUDE.md INTENT_FIDELITY, CONFLICT is a HARD_STOP: a brief that contradicts a D-entry without
a named supersession -> **STOP and report**. Never route around it; never obey a stale entry; a
human arbitrates, not the implementing agent. So S4 has exactly two legal shapes -- the brief names
"supersedes D-18" and the entry is rewritten in the SAME PR as the code (no tombstones), or the
agent stops and escalates the conflict. Reaching the guard site with neither is the failure mode
this paragraph exists to prevent.

**Do not ship the collision-refusal guard drafted on 2026-08-25.** Measured against live
traffic it would have refused **74%** of it. The patch is retained, deliberately uncommitted, at
`/tmp/sentinel-remediation/collision-guard/`. A guard that denies three quarters of production
is an outage, not a gate.

### 6.2 S5 disposition

If a material share of 287k cells is provably corrupt: mark, recompute, or leave and start
clean? Recompute is expensive and re-runs the same lossy path unless S4 lands first.

---

## 7. Standing constraints for every brief in this plan

Copy verbatim; these are the ones this codebase has been bitten by.

- **psql is SELECT-only.** Fix the root cause in the app, never the row.
- **Never hand-write an EF migration.** `dotnet ef migrations add`, `--output-dir Data/Migrations`.
- **`DisplayName~`, never `Name~`** in a test filter -- `Name~` matches zero and exits 0.
- **`bash {Project}/.devcontainer/compile.sh`**, capture the full log and the real `$?`.
  0 errors AND 0 warnings AND green.
- **`git add -- <paths>`**, never `-A` / `-u` / `.`.
- **Mutation-verify every guard and alert added or moved.** Delete or invert it, confirm RED,
  restore. A test that stays green is the bug, not the proof.
- **Do not push, PR, merge, deploy, or restart a service.** The supervisor owns the remote.
- **Mechanisms in a brief are hypotheses.** Every line number, count, and threshold here is
  unverified unless the story says otherwise. An agent correcting one on evidence is the round's
  value, not a deviation.
- **Check it already exists first.** This system is heavily built out; the capability is usually
  present and merely unused.

---

## 8. What "done" reads like

| check | today | after |
|---|---|---|
| a publish-seam metric carrying `kind="sector"` | no such label exists | non-zero |
| sector events in a collection cycle | 0 (102 days) | > 0 |
| golden corpus collision cases | RED (by design) | green |
| colliding share of published observations | 73.8% (15,331/20,770) | falls after S4 |
| instruments with mixed units | 1,577 / 2,733 | falls after S3 |
| `claim_kind` populated | column absent | populated, queryable |
| loaded patterns reading a `SENTINEL:NUM:` key | 0 of 71 | re-check before S4 is scheduled |
| colliding groups whose members share one `unit` | 70.0% (2,517/3,598) | falls after S4, or S4 shipped partial |

A green run is not proof. Every row above ships with a control that fails when its fix is removed.
