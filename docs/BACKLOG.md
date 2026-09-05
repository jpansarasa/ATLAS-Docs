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
an agent's (LESSONS.md L16), so the only honest alternative is to stop.

Do NOT close this by teaching it the `bash -c` and `python3` spellings. That is the
converging-on-a-reimplementation-of-bash path L14 prices out, and the prose case shows the grammar is
unbounded. Close it by gating the ACT -- the tool call's resolved write target -- rather than the
spelling of the command line.

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
BLOCKED 2026-09-04: that confirmation cannot be produced by the current harness -- pointed at the
production prompt+schema, BOTH KV arms score `aggregate_f1: null`. See MEASUREMENT DEBT, "`run_model.py
--prompt-file` cannot score production's CoD path".

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
**144 of 3,598 (4.00%)**. That catches only the case where the article's own subject STRINGS
differ; the worked case below has a constant `subject_entity` and is still a mis-resolution, so
4.00% is a floor of a floor and NOT a refutation of the 19%.

THE WORKED CASE, and it names the mechanism. `raw_content_id=153978`, one Mexican market-summary
article. Instrument `35253636-94b1-4b88-848c-30982e2de7f2` is **`EWW`, the iShares MSCI Mexico ETF**
— SecMaster `name` is the bare word **"Mexico"**, `asset_class` ETF. `subject_entity` on all four
rows is "Mexico", `resolution_method` is `hybrid_subject`, and the four values filed under EWW are:
  S&P/BMV IPC daily return    0.06 PCT
  number of rising stocks      125 COUNT
  number of declining stocks   107 COUNT
  number of unchanged stocks    18 COUNT
None of those is a measurement of EWW. An index level and an exchange's advance/decline breadth
were attached to a tradeable ETF because the resolver matched a COUNTRY word to an instrument NAMED
for the country. The same article resolves its actual equities correctly (KOFUBL.MX, FEMSA, GAP-B),
so this is not a broken resolver — it is a resolver with no way to say "this subject is a place, not
a security."

THE LEVER IS #988 (resolver retries with the description), NOT R2/S4's key. And note the direction:
#988 RAISES resolution success, which admits MORE rows into both this defect and the collision one.
Do not read a rising resolution rate as progress here without splitting these two populations.

A THIRD CONFOUND, small but real: exact-duplicate rows inflate the same groups —
**63 of 3,598 (1.75%)** colliding groups are pure duplicates (identical description, value and
unit). `raw_content_id=153978` carries one: instrument `750b06ec` holds each of its three
observations twice.

Re-check (SELECT-only):
  `SELECT count(*) FILTER (WHERE c>1 AND ds>1) AS multi_subject_groups,
     count(*) FILTER (WHERE c>1) AS colliding_groups
   FROM (SELECT raw_content_id, instrument_id, count(*) c,
           count(DISTINCT coalesce(nullif(btrim(subject_entity),''),'(null)')) ds
         FROM sentinel.extracted_observations
         WHERE published_at IS NOT NULL AND instrument_id IS NOT NULL GROUP BY 1,2) g;`
  `SELECT instrument_id, subject_entity, resolution_method, value, unit, left(description,50)
   FROM sentinel.extracted_observations WHERE raw_content_id=153978 AND published_at IS NOT NULL
   ORDER BY instrument_id, id;`
  and in `atlas_secmaster`:
  `SELECT id, symbol, name, asset_class FROM instruments
   WHERE id='35253636-94b1-4b88-848c-30982e2de7f2';`
THE RATIOS ARE THE DURABLE CLAIM, not the absolutes — the pipeline keeps publishing. The ~19%
sampled share is a FLOOR and the 4.00% proxy is a floor beneath it; neither is an upper bound, and
nothing here has measured how much of the 74% collision figure is really mis-resolution.

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
  `SentinelCollector/src/Services/DeterministicResolver.cs:485` (`TryExactCandidateMatchAsync` success,
  `llm_candidate_exact`). The other four are refusals or non-resolutions — `:253` (`LiftSector`; ONE site whose
  status is a ternary over `no_subject_match` / `matched_no_sector`, always `resolution_state="no_sector"`), `:437`
  (`TryExactCandidateMatchAsync` co-mention rejection, `exact_rejected_name`), `:521` and `:538`
  (`TryHybridResolveAsync` guard rejections).
- `ExtractionProcessor.cs` never calls `DeterministicResolver` — **zero** grep hits — and its own two
  `status="resolved"` emissions (`SentinelCollector/src/Workers/ExtractionProcessor.cs:1376` `ticker_in_quote`,
  `:1419` `cove_*`) have not fired in prod for 30 days. Those two cite the `status` label line, one BELOW their
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
`DeterministicResolver.cs:360-364`), so that column cannot answer what Rule 1 received, and querying it is circular.
It is observable, just not in the DB: PR #963 added `sentinel_resolver_rule1_input_confidence` ("ResolutionConfidence
as received by Rule 1", `SentinelMeter.cs:1625-1627`, recorded at `DeterministicResolver.cs:382`), which is the only
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
`DeterministicResolver.cs:541`) and attaches it. `DIA` is the catalog's quote stub, literally named "DIA (Quote)";
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
already invoked at `DeterministicResolver.cs:466` and `:509`. "Never on this branch" was the wrong compression and
is corrected here to match D-22 in the card: `:430` is D-8's leg, which Rule 1 never reaches. `:509` IS reachable
from Rule 1 — it sits on the RagSynthesis hypothesis-materialisation branch inside `TryHybridResolveAsync`, which
the id-less Rule 1 leg calls — but a DTO already carrying an instrument id bypasses it, and the whole guard is
behind `Extraction__GuardsEnabled=false` (`/opt/ai-inference/compose.yaml:1155`), so it is inert on both counts.
The 77 `DIA` rows above went straight through it: `SharedTokenCount("Dow Jones", "DIA (Quote)")` is 0, so with the
flag on they would have been refused. Adding a THIRD call behind that same disabled flag would read as protection
that does not exist. Deciding the flag's fate is the prerequisite, and it is its own entry's worth of work.
TWO GAPS #969 LEFT OPEN, recorded rather than fixed because both need a decision this PR is not the place for.
(a) The `!candidate.InstrumentId.HasValue &&` exemption on the blank-Name refusal (`DeterministicResolver.cs:397`)
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
(`EntityResolutionPrepass.cs:404`), Rule 2.5's paid-Gemini leg (`DeterministicResolver.cs:640`, D-6) and its V1
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

**The `BrokenCircuitException` classification gap is CLOSED (PR "fix: a circuit-open failure does not orphan an
article", SentinelCollector D-27). Recovering the 55 `raw_content` rows it orphaned in 2026-07 stays CLOSED as
won't-do — alpha decay; they pruned as decided.** The 223 rows it orphaned on 2026-09-04 were recovered by hand
before the fix landed. Kept rather than deleted because the re-check below is the standing detector for a NEW
orphaning event, and because the fix PROPOSED here (revised 2026-08-17) was the wrong one — see the trap below.

**What the fix actually does, and why the obvious version was wrong.** This entry used to say "Fix = add it to the
transient set, with a guard test asserting `RetryCount` increments on a circuit-open failure". That would not have
fixed it. `willRetry = isTransient && RetryCount < MaxRetries` with `MaxRetries = 3`: an open breaker short-circuits
with NO HTTP call, and the frontier re-serves the same rows immediately, so a multi-hour outage burns all three
attempts in seconds and the rows still end permanently failed — the same data loss at 3x the log volume. What
shipped instead classifies dependency-unavailable as its own case: the row is left exactly as fetched
(`ProcessingError` null, `RetryCount` UNCHANGED — a dependency being down is not the article's defect and may not
spend its retry budget), the worker slot pauses the breaker's own 30s break window so the requeue cannot hot-spin,
and the loop is terminated by the pre-existing `MaxArticleAgeDays` age gate if the dependency never returns. The
full reasoning, both bounds, and the alert that was riding on the bug are in `SentinelCollector/AGENT_README.md`
D-27; do not restate them here.

**Why it took a second cohort to fix: the incident's only signal was an accident of the bug.** Orphaning fired
`sentinel_extraction_error_total{reason="other"}` and tripped `SentinelHighExtractionErrorRate`. The row-level
signals said nothing — `sentinel_llm_error_total` stayed ~0 (an open breaker short-circuits BEFORE the HTTP call),
and `sentinel_extraction_queue_depth` / `sentinel_extraction_frontier_lag_seconds` did NOT rise, because the rows
left the queue the instant they were orphaned. The backlog was invisible by construction, and the ROOT cause had no
alert of its own either — `up{job="vllm"}` returned no data during the incident (see the last paragraph). Only the
downstream symptom alerted, and only because the bug itself was writing the errors it counted.

**The 2026-09-04 cohort: 223 rows, same predicate, all fresh.** Collected 00:06:38Z -> 16:46:56Z in two discrete
windows (00:00-02:04Z and 13:33-16:46Z), each bracketing to the minute a window in which `vllm-server` was STOPPED
to free the GPU for a vLLM 0.28.0 evaluation (the `vllm-028` container; the same evaluation recorded in CLAUDE.md
VLLM_UPGRADE). Loki carried `BrokenCircuitException ---> HttpRequestException: No route to host (vllm-server:8000)`
— the container was ABSENT, not slow. All 223 childless, `processed_at IS NULL`, `retry_count = 0`; ids
162986..163791; sources rss 204, rss-fallback 10, rss-mirror 8, searxng-content 1. Recovered same-day by
`POST /admin/reprocess` with the explicit id list. Rows collected 17:00Z onward processed cleanly.

**Re-check (psql is SELECT-only) — read BY COHORT. The predicate matches EVERY breaker-open event ever recorded, so
a bare total answers nothing and a bare zero is ambiguous between "pruned as decided" and "never happened".**

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
dependency outage.** `TryDispatchQualitativeAsync`'s extract-stage catch (`SentinelCollector/src/Workers/ExtractionProcessor.cs:2748`)
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

**The ROOT cause had no alert of its own during the incident, and now does — verified, not assumed.** #1001's vLLM
scrape job and rules were in the repo but NOT on the box while the 223 were being orphaned, so `vllm-server` being
absent for ~3.5h was alertable only through the downstream Sentinel symptom. They have since been deployed:
measured on the fix branch, live `/opt/ai-inference/monitoring/prometheus.yml` carries 8 jobs INCLUDING `vllm`,
`/opt/ai-inference/monitoring/alerts/vllm.yml` is present, and `up{job="vllm"}` returns
`{instance="vllm-server:8000"} = 1`. Recorded because a read-only diagnostic taken at 2026-09-04T22:30Z reported
the opposite and was already stale by the time the fix was written — re-run the three checks above rather than
quoting either state.

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

**`RawLayerOptions.RetentionDays: 90` IN `appsettings.json` NAMES A RETENTION POLICY THAT NOTHING
ENFORCES.** Measured 2026-08-26 while re-verifying the 30-day-rolling-window finding below (a same-day
pass had proposed reversing it to "raw retention is ungoverned" — it is not; that finding reproduces
exactly against a fresh DB query and the physical files on disk, see the re-check there): `RawLayerOptions`
(`SentinelCollector/src/Configuration/RawLayerOptions.cs`) has exactly one consumer, `RawContentService`,
and it reads only `.BasePath` — `.RetentionDays` has zero readers anywhere in `SentinelCollector/src`. The
retention that actually runs is a **different, appsettings-invisible** setting: `ExtractionOptions.MaxArticleAgeDays`
(code default 30, absent from every `appsettings*.json` — there is no operator-visible knob for it at all),
consumed by `StaleContentPrunerService` (deletes childless `raw_content` rows outright; nulls `raw_file_path`
and calls `IFileSystem.DeleteFile` on the HTML + `.meta.json` for rows that still have `extracted_observations`
children) and by `ExtractionProcessor`'s per-article age-cutoff skip. An operator reading `RawLayer.RetentionDays:
90` reasonably concludes raw content lives three months; it lives one, via a knob they cannot see. Note for
whoever re-checks this: a grep for the literal string `File.Delete` also misses the live mechanism — the delete
runs through the injected `IFileSystem` abstraction (`DeleteOne` in `SentinelCollector/src/Data/Repositories/RawContentRepository.cs`),
not the literal call; a pass that grepped only the literal concluded no pruner exists at all, which is false.
Fix: wire `RawLayerOptions.RetentionDays` to actually drive the prune cutoff (retiring the invisible
`MaxArticleAgeDays` default), or delete the dead field so `appsettings.json` stops asserting a policy nothing
enforces. Storage is not why 30 days was chosen, if anyone weighs extending it: `sata-bulk/raw-data` reports
5.8G live (df / zfs `REFER`) against 32.1G physical `used` (the gap is `zfs-auto-snapshot`-retained deleted
data) at 3.67x lz4 compression (109G logical), out of 5.11T pool-available. At the lifetime average since
collection began (`RawContentService` landed 2025-12-26, ~8 months ago) of roughly 4 GB physical/month, a
decade of uncapped retention would land under 500 GB — under 10% of the pool.
Re-check: `grep -n "_options\." SentinelCollector/src/Services/RawContentService.cs` (only `.BasePath`
appears); `grep -rn "MaxArticleAgeDays" --include=*.json .` (zero hits outside its own C# default);
`zfs get -Hp used,logicalused,compressratio,available sata-bulk/raw-data`.

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

## MEASUREMENT DEBT [instruments that cannot report their own dullness]

### `run_model.py --prompt-file` cannot score production's CoD path -- the flag exists, the measurement does not [2026-09-04]
`--prompt-file` / `--schema-file` were added (a2877cdb) so the harness could score "production's
cod_json_v1.txt" -- its own help text says exactly that. Pointed at production's real prompt+schema they
produce a scorecard whose **`aggregate_f1` is `null`**, because four independent things do not line up.
The model is fine; the instrument cannot read it.

MEASURED 2026-09-04, vLLM 0.19.0, production container as-is (`--kv-cache-dtype fp8_e5m2`),
Qwen2.5-32B-AWQ rev `5c7cb76a268fc6cfbb9c4777eb24ba6e27f9ee6c`, 3 records through
`run_model.py --endpoint-mode completions` with production's `cod_json_v1.txt` + `cod_json_schema_v1.json`:
`records: 3  errors: 0  schema_invalid: 3`, every record `n_extractions = 0`; scored `measurable: 4/18`,
`aggregate_f1: null ("no data")`, `json_valid: 0.0`. The same call by hand returns `finish_reason: stop`
and **30 populated `numbers[]` objects** on record 1 -- nothing is wrong with the extraction.

1. OUTPUT SHAPE. The CoD stage emits ONE object with four list-valued keys (`entities`, `numbers`,
   `events`, `claims`). `run_model.parse_extractions` accepts a dict only when exactly ONE value is a list,
   so it returns `([], False)` -- correctly refusing to guess, which makes every production response a
   parse failure.
2. GOLD SHAPE. `grep -c 'period\|certainty' cod_json_schema_v1.json` returns **0**. The scorer's contract
   is `NUMERIC_REQUIRED = (text_quote, value, period)` and it grades `certainty`; production's `numbers[]`
   carries `(source_text, value, unit, context, source_entity)`. `period` and `certainty` are not
   under-emitted, they are absent from the schema by design -- CoD is stage 1 of 4, and those fields are
   supplied downstream by dsl-parser-mcp `/parse_json` + the verifier + the adapter
   (`GpuJsonExtractionService`). `text_quote` (a gold SENTENCE) and `source_text` (a verbatim numeric
   literal) are not the same field either.
3. PROMPT ASSEMBLY. `cod_json_v1.txt` is a TEMPLATE (`{{source_id}}`, `{{article_text}}`, inside a
   `<<<ARTICLE ... ARTICLE` block). `--prompt-file` concatenates `f"{instruction}\n\n{content}"`, so the
   literal `{{article_text}}` is sent to the model and the article lands AFTER the prompt's closing
   "Output ONLY the JSON object" line. The harness does not send production's prompt TEXT, before any
   question of scoring it.

4. NO CHAT TEMPLATE [closed in #1002; the measurement above predates the fix, so it was taken with this
   divergence live]. `--chat-template` was optional and defaulted to `None`, while `--endpoint-mode
   completions` advertised that it "reproduces ATLAS production (client-side template +
   /v1/completions)". `/v1/completions` applies no template server-side, so the run above sent a raw
   continuation with no `<|im_start|>` turn -- production's ChatML wrapper was on neither side of the
   wire. This is the one divergence with NO tell in the output: the model answers, the scorecard fills
   in, the prompt was simply not the one production sends. `run_model.validate_request_shape` now
   REFUSES completions mode without the flag rather than defaulting it, and `--chat-template '{0}'` is
   how an operator asks for an untemplated continuation on purpose.

Same class, minor: production also sends `repetition_penalty=1.1` (`CpuCod__JsonRepetitionPenalty`); the
runner omits it unless `--repetition-penalty` is passed.

CONSEQUENCE. The first KNOWN DEFECT in this file is blocked on "confirm the fp8 penalty on the production
prompt", and that confirmation cannot be produced by this harness at ANY KV dtype -- both arms score
`null`. A two-arm run costs a production stop, two ~4min GPU reloads and ~1h of inference to yield two
empty scorecards.

CLOSE THIS by deciding what "production's path" means for scoring, which is a design choice and not a
patch: either (a) put dsl-parser-mcp `/parse_json` in the harness loop so the thing scored is the
`DocumentAst` production actually consumes, or (b) build gold labels in CoD shape and score stage 1 on its
own terms. Do NOT close it by hand-mapping `numbers[]` onto the gold fields -- `period` and `certainty`
have no source in that payload, so the mapping would inject a constant penalty larger than the ~0.05
effect being measured, and the adapter's own design choices would dominate the result.
Re-check: run `run_model.py --endpoint-mode completions --prompt-file
SentinelCollector/src/cod-prompts/cod_json_v1.txt --schema-file
SentinelCollector/src/cod-prompts/cod_json_schema_v1.json --chat-template
$'<|im_start|>user\n{0}<|im_end|>\n<|im_start|>assistant\n' --limit 3` and score it; if this entry is
still true, `aggregate_f1` is still `null` and `schema_invalid` still equals the record count. The
`--chat-template` argument is mandatory in this mode now (cause 4); the other three causes are untouched
by that fix and are what keep the entry open.

### `run_model.py --schema-file` silently bypasses `SCHEMA_REQUIRED`, on the model-acceptance path [2026-09-04]
`build_payload` reads `schema = load_schema(args) or extraction_json_schema()`
(`LlmBenchmark/scripts/run_model.py:296`), and `load_schema` (`:346-351`) returns whatever JSON the
operator handed `--schema-file`, verbatim. Nothing between there and the wire checks that the supplied
schema's `required` covers `SCHEMA_REQUIRED`. The derived schema is the only one carrying that coverage
guarantee, and `--schema-file` is the flag that discards it -- without a word in the output.

WHY THIS FLAG. `--schema-file` is not exotic: `CLAUDE.md` §MODEL_ACCEPTANCE names it in the exact
invocation a model swap must produce a scorecard from (`--endpoint-mode completions --prompt-file
cod_json_v1.txt --schema-file cod_json_schema_v1.json --chat-template ...`). The bypass therefore sits on
the one path that decides whether a candidate model replaces the incumbent.

WHAT IT COSTS is already measured on this harness, at `run_model.py:83-99`: `certainty` was optional in
the request schema, so the model emitted it on 0 of 2,213 extractions while gold carries it on 5,111 of
5,111; `eval_harness` counts every omission a miss (`certainty_accuracy` at `eval_harness.py:320`), and
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

CLOSE THIS by making `validate_request_shape` (`run_model.py:398`) refuse a `--schema-file` whose
`required` does not cover `SCHEMA_REQUIRED`, in the same fail-closed-rather-than-default shape it already
applies to `--chat-template-kwargs` in completions mode -- with the override spelled explicitly so the
choice lands in provenance. Do NOT close it by merging the supplied schema into the derived one: that
sends a schema the operator did not write, and then misreports it as production's.

TEST THAT WOULD PIN IT: `test_run_model.should_exit_two_when_a_schema_file_omits_a_graded_field`, a
sibling of `should_exit_two_when_completions_mode_has_no_chat_template` (`test_run_model.py:516`),
driving `main()` rather than a helper, with its paired positive (a covering schema runs). Note that the
existing `should_send_the_supplied_schema_when_a_schema_file_is_given` (`test_run_model.py:362`) asserts
the bypass verbatim -- it pins the current behaviour in place and must be read as the contract it is,
never as coverage of this hole.

Re-check: run the snippet above. If this entry is still true it still prints
`['certainty', 'period', 'text_quote']`, and `grep -n SCHEMA_REQUIRED LlmBenchmark/scripts/run_model.py`
still shows no reference inside `load_schema` or `validate_request_shape`.

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

**`claim_kind` SHIPS WITH ITS COVERAGE UNMEASURED — the columns exist, the subject join is proven in
unit tests, and NOTHING says what share of production rows will actually carry a value.** Measured
2026-08-27: 0 of 744,631 `sentinel.extracted_observations` rows carry a claim kind, because the
column did not exist until the S3 claim-kind persistence PR (branch
`fix/sentinel-claimkind-persistence`). The share that WILL carry one cannot be measured before
deploy and is not a matter of trying harder: CoD CLAIM blocks are never persisted anywhere, so
there is no stored corpus to replay the join over, and the golden corpus is drawn from
`extracted_observations` and therefore inherits the same blindness. The failure mode is silent by
construction — a column that is always NULL reads exactly like a column nobody queries, which is
the shape of defect the whole identity epic is about. Fix: read the counter after the first full
collection cycle post-deploy and record the split here.
Re-check:
  `sum by (outcome) (increase(sentinel_dsl_adapter_claim_atom_total[24h]))` — three outcomes,
  `attached|ambiguous|none`, one increment per emitted NUM row, so the sum is directly comparable
  to `sentinel_dsl_adapter_row_decision_total`. Anchor the eval instant to an actual `date -u`.
  # `attached` at or near 0 over a day of v2 traffic (1,179.2 articles/day at the 2026-08-12
  #   volume anchor) means the subject join finds nothing and the columns are decorative
  # a high `ambiguous` share is a finding about the DATA, not a reason to loosen the join: it says
  #   articles routinely make several kinds of claim about one issuer, and per-SUBJECT scoping is
  #   what should be reconsidered — first-wins would only hide it
  # `none` dominating on a source whose articles do carry claims points at the join key, the raw
  #   entity name both producers copy, drifting between the two arrays -- but read it WITH
  #   `sum by (reason) (increase(sentinel_dsl_adapter_claim_atom_refused_total[24h]))` first: a
  #   claim_kind too wide for the 40-char column is refused at the adapter and its rows land in
  #   `none` like any other, so producer drift and join drift are the same reading on the first
  #   counter alone. Non-zero `kind_too_long` is a PRODUCER finding, not a join one

**THE SENTINEL RAW-DATA STORE IS A 30-DAY ROLLING WINDOW, NOT AN ARCHIVE — so any corpus drawn
from it is a snapshot that cannot be re-drawn, and a plan that treats it as a queryable history is
planning against something that does not exist.** `extraction-identity-implementation.md` §1 sizes
it as "59,634 files, 5.7 GB, retained since 2025-01-01" and builds story S0 on that. Measured
2026-08-26: `StaleContentPrunerService` enforces a 30-day `raw_content` retention (its own summary
line says so), and the oldest retained raw file on EVERY source is 2026-07-27, exactly 30 days back.
48% of published observations (48,211 of 100,383) have a raw file at all. The consequence is
structural, not cosmetic: every article in the golden corpus stops being regenerable within a month
of selection, which is why those fixtures are committed rather than queried, and widening the corpus
means selecting RECENT articles rather than reaching further back. The number that would make this
re-checkable at a glance does not exist anywhere — there is no metric or dashboard for retained-file
age or fixturable share, so the next person to size the store will read the directory and guess
again.
Re-check:
  `SELECT source, min(collected_at)::date, max(collected_at)::date, count(*) FROM sentinel.raw_content
     WHERE raw_file_path IS NOT NULL GROUP BY 1 ORDER BY 4 DESC;`
  # every source's min must be ~today-30; 2026-08-26 returned 2026-07-27 on all 14

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
Re-check: run the tool on the touched docs from a pristine checkout of the merge-base and again from the branch, and
compare the `N citation(s) checked, M cannot land` line from each; M must not rise, and any rise in N must be
accounted for by citations you deliberately added.

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
not a card-entry pointer, and the baseline comparison is still the only way to see one move.

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
`git ls-files -z | xargs -0 grep -nE '\w+\.cs:[0-9]' | grep -v '\.md:'` -- currently 4 sites, one of which
(`dream/index_hooks.py:43`) is an EXAMPLE inside a regex doc comment and must never be "repaired", which is why
this is a corpus question with a judgement in it rather than a flag to flip. Closing it means either extending
the documented invocation and pricing in that example class, or a `--scope`-style opt-in; both change what every
future sweep reports and need their own before/after counts, exactly as the `_EXTS` entry below says.

**Nine real-but-wrong citations stand in `SentinelCollector/AGENT_README.md`, all GREEN, none of them any PR's
debt.** The live instance of the drift class above, found by hand in review of PR #1004 and deliberately NOT
repaired there: they sit in the `DeterministicResolver` / `ExtractionProcessor` D-entries as a -35 cluster from an
old un-followed insertion plus -74, -30 and -4, and one is a NAMING error rather than a number --
a GUARD reads `ResolveAsync @ src/Services/DeterministicResolver.cs:626` while `ResolveAsync` is the thin
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
- PER-REQUEST COST IS DISCARDED. deepinfra returns `usage` carrying `estimated_cost`, and `grep -c usage`
  is **0 in BOTH `run_model.py` and `eval_harness.py`** -- the field is read by nothing, so a labelling run
  cannot report what it spent from its own output. Same shape as the Foundry ledger whose `cost_est` had to
  be corrected 3x after the largest run: a run that does not record its OBSERVED cost leaves only list-price
  arithmetic behind.

WHAT THIS DOES NOT CLOSE, AND THE DISTINCTION TO PRESERVE. The router is CHAT-only, so this route cannot
close the MEASUREMENT DEBT entry "`run_model.py --prompt-file` cannot score production's CoD path" -- and
does not need to. LABELLING (produce gold) and SCORING (grade production's own `/v1/completions` path
against that gold) are two jobs on two endpoints. A hosted route buys the first; the second still runs
against vLLM, and CLAUDE.md §MODEL_ACCEPTANCE stays blocked until that entry is closed on its own terms.

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
  columns and found the join absent -- that PR's own coverage debt is recorded above (`claim_kind`
  SHIPS WITH ITS COVERAGE UNMEASURED).
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
