# Implementation Plan — News Pipeline Remediation

**Status:** PLAN / for review — NOT approved, NOTHING deploys from this doc.
**Origin:** user ↔ supervisor, 2026-08-06, after the measurement pass that produced the two defects below.
**Author:** Claude (planning agent). Every number carries a provenance tag (§0).
**Baseline / rollback point:** tag `pre-spec-20260806`; ZFS `sata-bulk/backups@pre-spec-20260806`,
`nvme-fast/timeseries@pre-spec-20260806`; artifact manifest
`/opt/ai-inference/backups/pre-spec-20260806/MANIFEST.md`. **Read that manifest's §5 before assuming
the baseline covers a step — it is not re-derived here.**
**Scope:** two defects with fixes already written (both on *local, unpushed* branches — §2.6), plus the
data repair that neither of them performs.

> Curation note: this is a *build plan*, and it merges. Per `docs/README.md` §Curation policy, a live
> plan or spec sits on `main` while it is being worked and is `git rm`'d at completion with a
> `git show <tag>:<path>` recovery pointer in `docs/RELEASES.md`. **Merging is not approval** —
> **Status** above governs that, and nothing deploys from this doc until the user approves it.
> Retirement trigger for *this* doc: A2, B1 and C2 all merged with their acceptance blocks green.
>
> *(An earlier draft of this note said the branch was "the review surface only … not merged", inside a
> PR that merges it. That note is a copy of the one still standing at
> `regime-news-staleness-redesign.md:7`, which has been merged on `main` since `7f65bd58` — the same
> self-contradiction, unnoticed for weeks. The policy it cited was itself contradictory; that is fixed
> in this PR, see `docs/README.md`.)*

---

## 0. Evidence conventions

The user's constraint for this work: *"We just should not make the same mistake twice."* Two of the
recurring mistakes are about **evidence**, not code — a null sample read as a healthy one, and a
population that structurally cannot contain the thing being looked for. So provenance is a first-class
part of every claim here, not a footnote.

| Tag | Meaning |
| --- | --- |
| `[M]` | Measured by me during this planning pass, 2026-08-06T20:50–21:10Z. Query given or reproducible from the cited path. |
| `[I]` | **Inherited** from the brief / earlier work. Not re-measured. Treat as a hypothesis. |
| `[I→M]` | Inherited *and* re-measured here. Where the value moved, both are shown. |
| `[U]` | Could not verify — listed in §8. |

**Population statement.** Every measurement below names what its population *structurally cannot
contain*. This is mechanism NT-2 (§3), applied to this document itself.

---

## 1. What is broken

### 1.1 Defect A — the numeric extraction path writes nothing, and has not for over a month

A producer/consumer contract break. `dsl-parser-mcp /parse_json` emits NUM slots
`{source_text, value, unit, context}` (+ `source_words`); the C# adapter reads slot names that no
producer emits.

- Producer `SentinelCollector/dsl-parser-mcp/json_cod.py:267-280` (`_num`) emits `value`, `unit`,
  `context`, `source_words`; `source_text` becomes `NUM.raw`, not a slot. The schema
  `SentinelCollector/src/cod-prompts/cod_json_schema_v1.json:57-79` is
  `additionalProperties: false` over exactly `source_text|value|unit|context`, all four required. `[M]`
- Consumer `SentinelCollector/src/Extraction/DslToMergedExtractionAdapter.cs:257-264` reads
  `description` → `label` → **`num.Raw`**, plus `period_start`, `period_end`, `source_entity`. None of
  those five slot names is emitted by any producer on either path today. The `?? num.Raw` fallback at
  `:259` is what silently substitutes the bare numeral. `[M]`
- The `context` slot — the one thing that would work — is produced correctly and discarded. Prompt
  `/opt/ai-inference/prompts/cod/cod_json_v1.txt:42-45` (byte-identical to the in-repo copy) defines it
  as *"the entity or metric this number describes … Use a short surface phrase."* `[M]`
- `MacroObservationRouter.TryPlanMacroWrite` (`SentinelCollector/src/Services/MacroObservationRouter.cs:80-94`)
  keys on `observation.Description` via an **exact** alias-dictionary lookup
  (`MacroSignalIdentityCatalog.cs:111-126`). A bare numeral can never be an alias, so it returns `false`
  at `:93` — **with no log and no counter on that branch.** The row falls back to
  `extracted_observations`. `[M]`

**Measured impact.** `sentinel.extracted_observations`, monthly, whole table: `[M]`

| Month | Rows | Mean `description` length | % starting with a digit | % resolved |
| --- | --- | --- | --- | --- |
| 2026-03 | 2,114 | 23.7 | 0.1% | 1.135% |
| 2026-04 | 44,783 | 23.7 | 1.4% | 4.214% |
| 2026-05 | 119,393 | 20.8 | 8.1% | 3.087% |
| **2026-06** | 210,000 | **9.8** | **77.9%** | 0.792% |
| 2026-07 | 187,442 | 9.4 | 75.8% | 0.336% |
| 2026-08 (partial) | 39,938 | 9.4 | 76.9% | **0.095%** |

The discontinuity lands in June. The GPU-JSON cutover is `f29d3396` (2026-06-09) — it changed
`deployment/ansible/group_vars/all.yml` and the compose template and **did not touch the adapter**. `[M]`

`subject_source` is `doc_lede` on **100%** of rows: `sentinel_dsl_adapter_row_decision_total` over 24h
has three series (`hit` 5,611 / `miss` 2,442 / `no_candidates` 155) and **every one is
`subject_source="doc_lede"` — the `per_num_slot` series does not exist at all.** `[I→M]` (inherited as
45,748/45,748 on a 7d window; same conclusion, different window.)

Extraction quality is *not* the problem — inherited as 99.8% verbatim-in-source and 98.8% values
correct `[I]`, not re-measured here.

> **Population note.** The monthly table is the whole `extracted_observations` table, so it *can*
> contain healthy rows — the April row proves the measurement is capable of showing a good value. That
> is what makes it admissible evidence rather than a one-sided sample.

### 1.2 Defect B — SecMaster self-seeds NER surfaces as instrument names; the faucet is open

`IdentifierConfirmationService.ConfirmAsync` has three confirmation legs, all ending
`?? proposedName ?? surface`: `:78` (OpenFIGI), `:129` (Finnhub), `:178` (Gemini — chain *starts* at the
proposal; there is no source term). `[M]` `GeminiResolverResult`
(`SecMaster/src/Services/GeminiResolverResult.cs:10-19`) carries `Symbol, Exchange, AssetClass,
InstrumentType, Confidence, SourceUrl, Rationale, Cached, CostUsd` — **no name field exists**, so a
Gemini-path name was never anything but the echoed surface. `[M]`

Two chains the brief did not name, both in `EntityResolutionService.cs`: `:893` (wire/display name,
feeds `ContextFactor`) and `:986` — **the actual persist site**,
`Name = confirmation.CanonicalName ?? confirmation.Ticker!`. The surface enters upstream at `:857`. `[M]`

**Measured population** — `discovery_source IN ('entity_resolution:gemini','entity_resolution:openfigi','GeminiFallback')`: `[I→M]`

| Metric | Inherited | Measured 2026-08-06T20:52Z |
| --- | --- | --- |
| Self-seeded rows | 1,688 | **1,694** |
| — `entity_resolution:gemini` | 1,133 | 1,139 |
| — `entity_resolution:openfigi` | — | 352 |
| — `GeminiFallback` | — | 203 |
| Carrying `atlas_sector_code` | 513 (of 1,133) | **573 of 1,694** (518 of the gemini leg) |
| Carrying a FIGI | 1,386 | 1,392 |
| Already `is_active=false` | — | 90 |
| Catalog share | 6.9% | 6.90% (of 24,543) |

The inherited totals reconcile exactly (1,133+352+203 = 1,688), which confirms the definition — and the
+6 drift in ~1 day confirms **the faucet is open**. Newest self-seed at query time:
2026-08-06T20:47:43Z, twelve minutes earlier. `[M]`

Live examples, with what OpenFIGI's local cache already asserts: `[M]`

| Symbol | Stored `name` | Sector | `openfigi_lookup_cache.canonical_name` |
| --- | --- | --- | --- |
| KOF | `Research Division\nHenrique Morello - Morgan Stanley` | CONS_STAPLES | COCA-COLA FEMSA SAB-SP ADR |
| JAN | `Second Quarter 2026 Conference Call` | REAL_ESTATE | JANUS LIVING INC CL-A-1 |
| OLCLY | `DisneySea` | — | ORIENTAL LAND CO LTD-ADR |
| TEM | `Doudna` | INFOTECH | TEMPUS AI INC-CL A |
| YXT | `Univest Securities` | INFOTECH | *(not cached)* |
| PUK | `the National Financial Regulatory Administration` | — | *(not cached)* |
| ELME | `The Beitel Group` | REAL_ESTATE | *(not cached)* |

Sector-tagged Sentinel observations do reach the matrix: 291 rows across all 11 sectors in the last
24h. `[M]` So a corrupt name attached to a sector code is not inert.

### 1.3 Not built — the data repair

The 1,694 rows do not self-heal. Enrichment is fill-gap only, at **two** sites with an identical
predicate — `OpenFigiEnrichmentBackgroundService.cs:280-286` and
`CatalogEnrichmentBackgroundService.cs:357-363`:

```csharp
if (mapping.Name is not null &&
    (string.IsNullOrEmpty(instrument.Name) ||
     string.Equals(instrument.Name, instrument.Symbol, StringComparison.OrdinalIgnoreCase)))
```

A junk name is neither empty nor equal to the symbol, so it is skipped. `[M]`
Only **7 of 1,694** rows have `name == symbol`, i.e. fill-gap is eligible on 0.4% of the population. `[M]`

**Worse than "declines to fire" — it becomes unreachable.** Neither batch predicate mentions `Name`.
OpenFIGI's pool is `IsActive && EquityAssetClass && Figi == null` (`:110-116`); once a FIGI lands the
row leaves the pool permanently, and **1,392 of 1,694 already have a FIGI**. Catalog's pool is
`IsActive && (AtlasSectorCode == null || Exchange == null || Country == null)` (`:95-99`); once those
three fill, the row is gone too. `[M]`

**Repair material available locally, at zero API cost:** 487 self-seeded rows have a cached OpenFIGI
`canonical_name`, and **444 of those differ from the stored name**. `[I→M]` (inherited as 368; the
inherited figure is lower, and I could not reconstruct its exact predicate — §8.)

**Embeddings.** `instrument_embeddings` coverage of the self-seeded set is **1,694 / 1,694 = 100%**,
and the embedded text is name-bearing (`EmbeddingService.cs:468`:
`instrument.Name + " (" + instrument.Symbol + ")"`). `[M]` They self-heal after a name repair:
`EmbeddingBackgroundService.cs:103-113` marks stale on `i.UpdatedAt > e.CreatedAt` — **cross-table**
(instrument's `UpdatedAt` vs the embedding row's `CreatedAt`), not one row's two columns as inherited.
`[I→M]` The `instrument_embeddings` table has no `updated_at` column at all, which is why it must be
cross-table. `[M]`

---

## 2. Corrections to the brief

Every round of this work has produced corrections; these are this round's. Listed before the plan
because two of them change what the plan must do.

**2.1 "0 substrate rows" is true only for the numeric path — and the shared counter hides it.**
`sentinel` has written to `macro_observations` continuously, all `value_numeric`. `[M]`
Splitting by the `:sig:` infix in `source_id` separates the two producers:

| Path | Rate | Last write | Lifetime |
| --- | --- | --- | --- |
| news-signal (`…:sig:…`) | **~550/day** (7d mean, 2026-07-31…08-06). Strongly bimodal: weekdays 610–826/day (mean 705), weekends 145–186/day (mean 166). | 2026-08-07 (live) | — |
| **numeric / macro (non-`:sig:`)** | **0/day** for 37 days | **2026-06-30T14:40:38Z** | **81 rows, ever** |

> Corrects an earlier reading of this same table ("~700/day; 555–826/day over 12 days"). **~700 is the
> weekday rate, not the all-days rate**, and the 555 floor was a partial current day rather than an
> observed daily minimum. Four of the trailing thirteen days — every Saturday and Sunday in the window —
> sit at **139–186/day**, well below that floor, so the old band did not bracket its own window. Over
> 13 days (2026-07-25…08-06) the all-days mean is **560/day**. Re-measured 2026-08-07T01:36Z. `[M]`
> **Population:** `macro_observations WHERE source_collector='sentinel' AND source_id LIKE '%:sig:%'`,
> bucketed on `observation_time`. It cannot contain writes the router declined to make, nor re-emissions
> collapsed by the `ux_macro_obs_idem` unique index — so it is a floor on production, never a count of
> attempts. It also cannot separate "quiet news weekend" from "pipeline stalled at the weekend": the
> bimodality is a *shape*, not a diagnosis, and nothing here licenses calling the weekend trough healthy.

This is the single most important correction, because of what follows from it (2.2).

**2.2 The proposed acceptance metric cannot fail today.** `sentinelcollector_macro_observations_written_total`
reads **586 over 24h** `[M]` while the numeric path's DB rows are **zero for 37 days**. There is exactly
one increment site — `MacroObservationRouter.cs:338` — and the counter is declared untagged and is
documented as such (`SentinelMeter.cs:673-677`: *"Bounded — no tags, scalar counters only"*). Both
producers pass through it. `[M]` **An alert or acceptance check written against that counter would be
green right now.** No step in §4 may use it unlabelled.

**2.3 The numeric path was already dying before the cutover.** Its 81 lifetime rows run
2026-05-15…2026-05-28 (79 rows, peak 25/day), then one row on 06-29 and one on 06-30. `[M]` The June 9
cutover took a nearly-dead path to exactly zero; it did not kill a healthy one. Last write was
**2026-06-30**, not the inherited 2026-07-08. `[I→M]` The remediation should not be sold on restoring
lost volume — at its best this path wrote 25 rows/day.

**2.4 `source_entity` was real, but on the retired path.** Introduced 2026-05-27 by `71ddd980` as a *v8
DSL prompt* slot (not a GBNF rule — `cod-dsl-v2.3.gbnf:99`'s `generic-slot` already allowed arbitrary
NUM slot names). Still present in the live DSL prompt at
`/opt/ai-inference/prompts/cod/cod_dsl_v8_baseline.txt:165,176,857`. It was never carried into the JSON
schema. `[I→M]` `description`, `label`, `period_start`, `period_end` were never emitted by *either*
path — they are dead reads, not cutover casualties. `[M]`

**2.5 "Wrong symbol → quarantine" would destroy good rows.** `idx_instruments_symbol` is UNIQUE `[M]`,
so there is exactly one KOF row and one TEM row in the catalog. Both are real tickers whose authoritative
names are already in the local cache (§1.2). Quarantining them removes Coca-Cola FEMSA and Tempus AI
from the catalog entirely. The corrupt artifact is the *name* and the *surface→ticker association*, not
the instrument. Absence from the OpenFIGI cache is also not evidence of junk — PUK (Prudential) and ELME
(Elme Communities) are real tickers that simply have no cached entry. `[M]` §5 splits disposition
accordingly.

**2.6 Neither fix is "in review".** Both branches exist **locally only** and are unpushed;
`git ls-remote origin` returns empty for both, and there is no PR. `[M]`
`fix/sentinel-dsl-adapter-slot-contract` = 2 commits, 7 files, +431/-12, and it does add a
producer/consumer contract test (`DslNumSlotContractTests`, 246 lines, replaying a captured wire
payload). `fix/secmaster-no-surface-as-name` = 4 commits, 6 files, +272/-6, HEAD `3b12e724`, and it does
include the D-2 per-field amendment. Both are checked out in other agent worktrees, one of them locked.

**2.7 Quarantine leaves the vector, but search already filters it.** `is_active=false` never deletes an
embedding — there is no delete path outside migrations and the cascade on *hard* delete — and **90
inactive self-seeded rows retain embeddings right now**. `[M]` But vector search filters inactive rows
at hydration (`EmbeddingService.cs:281-283`, with a comment naming exactly this failure mode). So
dropping the vector is hygiene and defence-in-depth, **not** the correctness requirement the brief
implies. `[I→M]`

**2.8 The review queue has never been drained.** 72,544 rows, **all 72,544 `is_open`**, none ever
resolved. `[M]` Top surfaces: `Investing.com` (6,665), `Reuters` (5,991), `InvestingPro` (3,285),
`Research Division` (2,000), `Trump` (1,997), `NYSE` (1,681). `Research Division` at 2,000 occurrences
is the surface that became KOF's name. This is a third instance of NT-5 (a repair path that never
fires) and it is not otherwise in scope — flagged, not planned.

---

## 3. The NOT-TWICE register — mechanisms, not resolutions

This register is the spine of the plan. Each entry names a failure mode observed in **this** work, what
it cost, and the mechanism that prevents recurrence. **Every step in §4 and §5 carries the `NT-n` it
discharges, and §7.1 is a traceability check that no entry is left without a step.** An entry with no
step is an unfixed failure mode, not an aspiration.

A resolution says "be careful next time." A mechanism fails the build, fires a page, or blocks a merge.
Only the second kind is admissible here.

**Three of the five clear that bar. Two do not, and this register counts three.** NT-1, NT-4 and NT-5
ship as artefacts that fail on their own: a contract test that goes red, a mutation-verified alert, a
promtool assertion. **NT-2 and NT-3 are enforced only by the §7.1 reviewer checklist** — a human reading
a table and judging whether a population statement or an adversarial case is adequate. By the bar set
one paragraph above, that is discipline, not a mechanism. They stay in the register because the failure
modes are real and a checklist beats nothing, but they are marked **`[checklist — not a gate]`** and
must not be counted as gates or reported as such. Claiming five mechanisms when three are enforced is
NT-4 applied to the register itself; an honest three is worth more than an inflated five.

### NT-1 — Silent contract break at a cutover
**Observed:** twice. Defect A (JSON cutover `f29d3396`, adapter untouched, two months of well-formed
garbage). Previously, the CoD cutover zeroed the digest the same way — separate consumers, no alert `[I]`.
**Cost:** ~437k rows since June carrying a bare numeral as `description`; the numeric substrate path at
zero for 37 days; nothing failed, nothing alerted.
**Why it survived:** the adapter's own tests fabricate `new DslSlot("description", "Q3 revenue")` in 17
places. `[M]` The fixtures assert the author's belief about the producer, never the producer's output.
**Mechanism (two, both required):**
1. A test that pins the payload contract by **replaying a real captured producer payload** through the
   real deserializer and real adapter — not a fabricated fixture. Symmetric halves on both sides of the
   language boundary, so either side moving alone goes red. → Step A1.
2. A **funnel-ratio alert** on extracted → resolved → written. There is no alert anywhere on this funnel
   today: `deployment/artifacts/monitoring/alerts/sentinel.yml` has 18 rules, and the closest,
   `SentinelLowResolutionRate` (`:197-208`), covers SecMaster instrument resolution, not the substrate
   leg. `grep -rln "sentinelcollector_macro_observations" deployment/` returns **nothing** — no rule, no
   panel. `[M]` → Step A2.

### NT-2 — Evidence population bias `[checklist — not a gate]`
**Observed:** six occurrences `[I]`; three re-confirmed here `[M]` — the review queue holds only
failures (72,544 rows, 100% open, §2.8); SecMaster logs only failure paths; the resolver journal is
failure-only. A census of surfaces that *reached* Gemini structurally cannot show which real names got
dropped before it.
**Cost:** repeated confident conclusions from one-sided samples; at least two rounds of correction.
**Mechanism:** every measurement in a plan, PR description, or acceptance check states **what its
population structurally cannot contain**, and every acceptance criterion carries a **negative control** —
a companion query that would return a different value if the pipeline were healthy. A criterion with no
negative control is not a criterion. → Applied in §0 and in every §4/§5 acceptance block.
**Enforcement is the §7.1 reviewer checklist and nothing else** — no build fails and no page fires if a
population statement is missing or wrong. Making this a real gate would need a lint over acceptance
tables, which is not in this plan's scope; until then, treat NT-2 as a habit the checklist reminds you
of, not a control.

### NT-3 — A fix re-opening the hole it closed `[checklist — not a gate]`
**Observed:** three occurrences `[I]` — the cache key traded a re-bill for a wrong instrument; the
quantity strip collapsed tenor families; the transient re-arm handed back the whole cap.
**Cost:** each shipped as a fix and needed a second fix.
**Mechanism:** **adversarial verification of the fix, not the bug.** For each step, before merge, write
down the answer to: *what does this fix now permit that was previously refused?* — then test that case
explicitly. Worked example for this plan: Step B1 stops names being invented, which means rows that
would previously have carried a plausible-looking name now carry the ticker; the adversarial case is a
legitimate OpenFIGI name being **wrongly rejected**, so B1's guard test needs a **negative control**
proving a real authoritative name still lands. → Named per-step in §4 as `ADVERSARIAL:`.
**Enforcement is the §7.1 reviewer checklist**: writing the adversarial case down is a human act, and a
reviewer can accept a weak one. The *tests* that a written-down adversarial case produces are real gates
(they go red), so NT-3 converts into enforcement one step downstream — but the conversion itself is
discretionary, which is why it is not counted as a mechanism.

### NT-4 — Checks that cannot fail
**Observed:** a merge marker written at invocation regardless of verdict; an alert whose numerator can
be absent; a test asserting through constants so a value rename keeps it green `[I]`. **And, found in
this pass:** `sentinelcollector_macro_observations_written_total` = 586/24h while the path it is meant
to evidence has written zero for 37 days (§2.2) `[M]`. I also caught myself producing one: a
`query_type = 'ticker'` filter returned `0 repairable` when the stored value is `'TICKER'` — a false
negative that reads exactly like a real finding.
**Cost:** two months of invisible breakage; a null sample mistaken for a healthy one twice in this work.
**Mechanism:** **mutation-verify every guard and every alert.** Break it deliberately, watch the
expected test/alert go red, restore. An alert expression must additionally be `absent()`-guarded so a
missing series pages instead of resolving green. This is the existing GUARD_TEST contract
(`.claude/skills/intent-review/SKILL.md §GUARD_TEST_CONTRACT`) extended to alert rules. → Steps A2, B1,
C2, and the §7.1 check.

### NT-5 — A repair path that declines to fire
**Observed:** enrichment fetched the right name and refused to apply it for two months (§1.3); and the
review queue, 72,544 rows, never drained (§2.8).
**Cost:** 444 rows repairable from cache at zero cost, sitting unrepaired; the population still growing.
**Mechanism:** **alert when a repair path's no-op rate approaches 100%.** A path whose *whole purpose*
is to change things, that changes nothing, is broken — indistinguishable from healthy without this
alert. Requires the path to count attempts and applications separately. → Step C2.

---

## 4. Sequencing and steps

### 4.0 The dependency graph, and why

```
A1 (adapter slot contract) ──> A2 (funnel alert)        [A2 needs a non-zero numerator to calibrate]
B1 (stop inventing names)  ──> C1 (repair existing rows) ──> C2 (repair no-op alert)
                                     └──> (embeddings self-heal, no step)
```

**B1 blocks C1.** Repairing before the faucet closes re-accumulates corruption. Measured self-seed
rate: **~30 rows/day** over the last 7 days (204 rows / 7d; 65 already today by 21:00Z). `[M]` Applying
the inherited junk fraction (~300–450 of 1,688 ≈ 18–27% `[I]`) gives **~5–8 junk rows/day**, not the
inherited ~19/day — the inherited figure does not reconcile with either the measured total rate or the
inherited fraction. `[I→M]` **The ordering conclusion is unchanged and is the reason for the edge; only
the magnitude moves.** If B1 slips, C1 slips with it.

**C1 blocks the embedding heal.** Embeddings are name-bearing and become stale only on
`i.UpdatedAt > e.CreatedAt` (§1.3), so they re-embed *because* C1 writes the row — no separate step, but
also no heal before C1.

**A1 blocks A2.** A2's threshold is a ratio; calibrating it while the numerator is structurally zero
would bake in the broken state. A2 must be authored against a path that can produce a non-zero value.

**A and B/C are independent** — different services, different tables, no shared code. They can run in
parallel. A is *not* on the critical path for B/C and should not gate it.

---

### Step A1 — Make the adapter read the slots the producer emits
**Discharges:** NT-1(1), NT-3, NT-4.
**Change:** land `fix/sentinel-dsl-adapter-slot-contract` (push, PR, review, merge, deploy). Adapter
description precedence becomes `description ?? label ?? context ?? num.Raw`; producer re-emits
`source_entity`; schema and prompt updated to five required fields; slot names hoisted to shared
constants. Includes `DslNumSlotContractTests` replaying a captured wire payload, plus the symmetric
Python half. `[M]`
**Pre-merge additions required by this plan:**
- Delete or re-label the 17 fabricated `new DslSlot("description", …)` fixtures. Leaving them means the
  suite still passes if the producer changes again — the exact NT-1 mechanism. `[M]`
- `ADVERSARIAL:` the fix adds `context` as a description source. `context` is a *free-text surface
  phrase*, so it can now feed `TryPlanMacroWrite`'s exact alias lookup with arbitrary text. Test that a
  junk `context` does **not** produce a macro write (GIGO at the send boundary), and that
  `MacroObservationsRejectedImplausible` still catches a level-vs-rate value.

**Acceptance.** All three measured ≥24h after deploy, on the same window.

| # | Measure | PASS | FAIL | Negative control (NT-2/NT-4) |
| --- | --- | --- | --- | --- |
| A1.1 | `SELECT count(*) FILTER (WHERE description ~ '^[0-9$\-+.]')::float/count(*) FROM sentinel.extracted_observations WHERE extracted_at > now() - interval '24 hours'` | **< 0.20** | **≥ 0.20** | The bands partition — there is no unadjudicated middle. Two FAIL sub-bands share the verdict and differ only in diagnosis: **≥ 0.60** = the fix did not take at all (baseline **0.7901 on n=9,028**, 24h to 2026-08-07T01:40Z `[M]`), while **0.20–0.60** = it took partially, so a second producer of bare numerals survives and must be identified before re-running. An earlier draft left 0.20–0.60 with no verdict; a post-fix 0.45 would have had nothing to compare against. Same query on `extracted_at` in April returns 0.014 — the query is *known* to be capable of a low value, so a low reading is not an artifact. **`count(*)` must be > 5,000; if the denominator is small the result is void, not passing.** |
| A1.2 | `sum by (subject_source) (increase(sentinel_dsl_adapter_row_decision_total[24h]))` | a `per_num_slot` series **exists** and is > 25% of total | no `per_num_slot` series (today's state) | Today the series is *absent*, not zero — so the check must assert **presence**, not `> 0`. A PromQL `> 0` on an absent series returns empty, which is not a failure signal. |
| A1.3 | Numeric-path substrate writes: `SELECT count(*) FROM macro_observations WHERE source_collector='sentinel' AND source_id NOT LIKE '%:sig:%' AND ingestion_time > now() - interval '48 hours'` | **≥ 5** | ≤ 4 | **48h and a count floor, not `> 0` over 24h** — the same burstiness argument B1 already applies to itself, which an earlier draft applied there and not here. `> 0` cannot separate "restored" from "still dying": this path wrote **exactly 1 row on 6 of its 14 ever-active days** (`ingestion_time` buckets, lifetime), so a death-rattle clears `> 0` at 24h and clears it again at 48h with 2 rows. The floor of 5 sits above that tail and below the path's own median throughput (4/day median over active days ⇒ ~8 per 48h; peak was 25/day, §2.3). `[M]` **Must not use `sentinelcollector_macro_observations_written_total`** — see §2.2; it reads 586/24h today with a zero numeric path. The `:sig:` exclusion is the whole check. Companion: the unfiltered count must also be > 0, proving the query reaches live data. |

`exact_rejected_name` (`DeterministicResolver.cs:324`) is a *status tag* on a resolver counter, not a
standalone metric `[M]`. Its parent counter emits no series in Prometheus today `[M]`, so it cannot be a
gating criterion — measure it as an observation only, and treat "no series" as "not measured", never as
"fell to zero". This is NT-4 in miniature. → §8.

**Rollback.** Revert the merge; rebuild and redeploy `sentinel-collector` from the prior image. Pin by
**Image ID / RepoDigest**, not `:latest` — MANIFEST §1 records
`sentinel-collector@sha256:0d4613f2…`, and MANIFEST §5.1 notes image blobs are **not** in the baseline
(they live on `/var`, a third uncovered disk) so the image must be rebuilt from git if it has been
pruned. Prompt/schema changes are host-mounted at `/opt/ai-inference/prompts/cod/`; restore from
`prompts-cod.tar.gz` (`-C /opt/ai-inference/prompts`). No data migration, so no data rollback.

---

### Step A2 — Alert the funnel
**Discharges:** NT-1(2), NT-4.
**Depends on:** A1 merged and producing a non-zero numeric-write rate.
**Change:** add rules to `deployment/artifacts/monitoring/alerts/sentinel.yml`.
1. **Numeric-path write starvation.** Requires a discriminating label first: either add a `path`
   (or `producer`) tag to `MacroObservationsWritten` at `MacroObservationRouter.cs:338`, or split the
   counter. Without that the alert is un-writable (§2.2). Prefer the tag — cardinality is 2, well inside
   the bounded-tag rule.
2. **Funnel ratio.** `resolved / extracted` below a floor over 24h.

**Both rules must be `absent()`-guarded** so a vanished series pages rather than silently passing.

**Acceptance.**
- Rule loads: `mcp__grafana__alerting_manage_rules` get returns each rule, and the PromQL evaluates to a
  **non-empty** result against live data. An empty result at authoring time means the expression is
  wrong — that is exactly how "an alert whose numerator can be absent" ships.
- **Mutation-verify (NT-4), mandatory, and the step is not done without it:** temporarily set the
  threshold so the rule *must* fire, confirm it fires end-to-end (Alertmanager → alert-service → ntfy
  `atlas-alert`), then restore. Per `[[project_grafana_alerting_provisioning_reload.md]]`, ansible copies
  alerting YAML but does **not** reload Grafana — restart `grafana` or hit the admin reload API, then
  verify the rule via a get, not by assuming the copy took.
- FAIL: the rule exists but its expression returns empty, or the forced-fire test produces no
  notification.

**Rollback.** Remove the rules and redeploy; reload Grafana. Monitoring-only, no data or service risk.
The counter-tag change ships with A1's service and rolls back with it.

---

### Step B1 — Stop inventing names (close the faucet)
**Discharges:** NT-3, NT-4. **Blocks:** C1.
**Change:** land `fix/secmaster-no-surface-as-name` (push, PR, review, merge, deploy). All three legs
drop the `?? proposedName ?? surface` fallback; self-seed persist becomes `Name = SelfSeedName(confirmation)`,
a closed-by-default helper returning `CanonicalName` only for `ConfirmationSource.OpenFigi`/`Finnhub`,
else `null`, then `?? confirmation.Ticker!`. The wire/display chain at `:893` is deliberately left intact
(substituting the ticker there would zero `ContextFactor` and break Sentinel's subject match). `[M]`

**D-2 amendment (INTENT_FIDELITY):** the branch rewrites D-2 to record that **authoritative is
PER-FIELD** — a confirmation grounds the TICKER and does not thereby ground the NAME. `[M]` This is a
supersession of the committed D-2, which states only a whole-record `confirmation.Confirmed` precondition
`[M]`. The PR must name it as such. Per CLAUDE.md MECHANICS the atomic set is: D-entry + `// INTENT(D-2):`
comment at each guard site + guard code + guard test — verify all four are present before merge, since
there are now **four** guard sites (three legs + `SelfSeedName`).

**`ADVERSARIAL:` (NT-3)** — the fix refuses names. The hole it could re-open is refusing *legitimate*
ones: a real OpenFIGI/Finnhub name failing to land, leaving the catalog full of ticker-named rows and
starving `ContextFactor`. The guard test needs a **negative control** asserting an authoritative name
still persists unchanged. A test that only proves junk is rejected is half a test.

**Acceptance.** Measured over 48h after deploy (the rate is bursty — 0 rows on 2026-08-02, 66 on 07-31
`[M]` — so 24h is too short to distinguish "fixed" from "quiet day").

| # | Measure | PASS | FAIL |
| --- | --- | --- | --- |
| B1.1 | New self-seeded rows created post-deploy whose `name` is neither the symbol nor a cached OpenFIGI/Finnhub name | **0** | ≥ 1 |
| B1.2 | Self-seeding still functions: rows created post-deploy with `discovery_source LIKE 'entity_resolution%'` | **> 0** | 0 → the fix broke resolution rather than fixing naming; this is the negative control and it is the more likely failure |

B1.2 is not optional. Without it, a change that stops all self-seeding scores a perfect B1.1.

**Rollback.** Revert the merge; redeploy `secmaster` pinned to
`secmaster@sha256:39ed35f3…` (MANIFEST §1), rebuilding from git if the blob is gone (MANIFEST §5.1).
Rows created while the fix was live keep ticker-shaped names — harmless and repairable by C1. Revert the
D-2 amendment in the same PR as the code revert (no tombstones).

---

### Step C1 — Repair the existing rows
See §5 — it needs more room than a table.

### Step C2 — Alert when the repair path no-ops
**Discharges:** NT-5, NT-4.
**Depends on:** C1 deployed.
**Change:** the enrichment services must count **attempts** and **applications** separately (today
neither site emits a counter on the skip branch, which is why a 99.6%-ineligible path looked like a
working one). Add a rule: a repair/enrichment path whose application rate is < 1% of its attempt rate
over 24h, with a non-trivial attempt count, is broken.

**Acceptance — and the NT-4 that was sitting inside the NT-4 mitigation.** The obvious criterion, and
the one an earlier draft of this step carried, was *"backfill the rule against history: it must fire
when evaluated over the pre-C1 window."* **That check cannot be run**, and it is worth naming why
rather than quietly substituting a working one. This step is what *adds* the attempt and application
counters — so **no series exists for any window before this step deploys**. A PromQL expression over an
absent series returns empty, and empty is not "fires". The criterion would have been unrunnable, and an
unrunnable criterion gets recorded as passing by whoever could not run it. That is NT-4 occurring inside
the NT-4 mitigation, written by someone holding the register open. The register does not exempt itself,
and this is the second time in this document that a "no series" reading had to be caught before it was
misread as a healthy zero (cf. A1.1's `exact_rejected_name` note and §8 item 5).

The runnable equivalent needs no history, because it supplies its own: a **promtool synthetic-series
test**.

- Add `deployment/tests/alerts/sentinel_test.yml`. It is picked up automatically by the `./*_test.yml`
  glob in `deployment/tests/alerts/run.sh` (present, executable, promtool-in-container;
  `gemini-resolver_test.yml` is the in-repo worked example — the only test file there today `[M]`).
  House style, per #912 §6.3 and §10.2: `promql_expr_test` against the `ALERTS` series, never
  `alert_rule_test` + `exp_alerts`.
- **All three directions are mandatory.** A test carrying only the first is precisely the can't-fail
  alert this entry exists to prevent:
  1. **Fires on the known-bad shape.** `input_series` reproducing the measured pre-C1 state — 1,694
     attempts against 7 applications (0.41%) over the window — then assert
     `ALERTS{alertname="…",alertstate="firing"}` is **present**.
  2. **Silent when healthy.** Application rate above the floor at the same attempt volume → assert
     absent.
  3. **Silent when idle.** Attempts at or near zero → assert absent. A repair path nobody is asking to
     repair anything is not a broken repair path, and a rule that pages on quiet gets silenced, after
     which it is decoration.
- Then **mutation-verify the live rule** as in A2: move the threshold so it must fire, confirm
  end-to-end delivery, restore.

**FAIL:** any of the three promtool directions missing or red, or the live rule cannot be made to fire
by mutation.
**Rollback.** Remove the rule; the counters are additive and harmless.

---

## 5. Step C1 — the data repair

**Depends on:** B1 (§4.0). **Discharges:** NT-5, and NT-2 in its disposition logic.

### 5.1 Mechanism — EF, never raw DML

CLAUDE.md `## DATABASE` forbids raw INSERT/UPDATE/DELETE to fix state. Note the prior art cuts the other
way: `20260718133628_QuarantineGeminiJunkInstruments.cs` and `20260718152925_QuarantineGeminiEquityEtfJunk.cs`
did exactly this problem with **raw SQL inside EF migrations** `[M]`. That precedent should not be
followed — a migration is a schema-versioning tool, it runs once at startup with no dry-run, no
per-row isolation, and no audit trail.

**Use the existing admin-endpoint shape instead.** `SecMaster/src/Endpoints/AdminEndpoints.cs:152-241`
(`POST /api/admin/catalog/dedupe-fred-hijacks`) is the in-repo precedent and already solves every
sub-problem `[M]`:

- loads candidates via `db.Set<InstrumentEntity>()…Where(…)` — EF, typed, no SQL;
- **per-row `SaveChangesAsync`** for failure isolation, with detach-on-error (`:228`);
- **no `BeginTransactionAsync`** — explicitly warned against at `:196-202`, incompatible with
  `NpgsqlRetryingExecutionStrategy`. This is a real constraint and it is why the rollback in §5.4 is
  per-row rather than transactional;
- idempotency via a `Metadata` sentinel key (`fred_origin`, `:183`);
- provenance written into `Metadata` **before** mutating (`:206-212`);
- soft-delete as `row.IsActive = false; row.UpdatedAt = …` (`:214-215`);
- structured `{Merged, Skipped, Errors, RanAt}` response.

So: **`POST /api/admin/catalog/repair-self-seeded-names`**, modelled on it.
**`dryRun` defaults to `true`.** Per `[[reference_sentinel_admin_dryrun_camelcase.md]]`, snake_case
`dry_run` silently no-ops as a dry run — the request must use `{"dryRun": false}` for a live run **and
the response must echo the mode it actually ran in**, which is then checked before believing any
"0 rows changed" result. A silent dry-run reporting zero changes is NT-4 wearing a different hat.

### 5.2 Disposition — three classes, and the one the brief got wrong

Population: `discovery_source IN ('entity_resolution:gemini','entity_resolution:openfigi','GeminiFallback')`,
1,694 rows `[M]`.

| Class | Predicate | Action | Count `[M]` |
| --- | --- | --- | --- |
| **R — Rename** | a fresh `openfigi_lookup_cache` row (`query = symbol`, `query_type = 'TICKER'`, `is_hit`) whose `canonical_name` differs from the stored name | set `Name = canonical_name`; stamp `Metadata.name_repair_v1` with the pre-image | **444** |
| **L — Leave + flag** | no cached authoritative name | no mutation; record in the response for a later pass | ~1,250 |
| **Q — Quarantine** | the symbol is not a real ticker (no FIGI, no cache hit, and the surface does not correspond to any instrument) | `IsActive = false`, `Metadata.quarantine_reason` | per-row corroborated, **not** predicate-driven |

**Class Q is deliberately small and manual.** Per §2.5, KOF/TEM/JAN/OLCLY are class **R**, not Q —
quarantining them would remove real instruments, since `symbol` is UNIQUE and these are the only rows
for those tickers. And absence from the cache (PUK, ELME) is *not* evidence of junk — that is NT-2, and
it is the single easiest way for this repair to do damage. The existing precedent agrees:
`QuarantineGeminiEquityEtfJunk.cs:46-64` used an **enumerated symbol list**, not a predicate, precisely
because each disposition was per-row corroborated (82 quarantined, 20 kept) `[M]`. C1 keeps that
discipline: **Q takes an explicit symbol list in the request body; it is never inferred.**

Sector codes on repaired rows: 573 self-seeded rows carry an `atlas_sector_code` `[M]` derived while the
name was junk. Renaming does not re-derive it. Clearing `atlas_sector_code` + `classified_at` on class-R
rows lets `CatalogEnrichmentBackgroundService` re-classify from the corrected name (its batch predicate
includes `AtlasSectorCode == null`, so clearing the field re-admits the row to the pool `[M]`). This
should be **in scope for C1** — otherwise the matrix keeps consuming a sector derived from
`"DisneySea"`.

### 5.3 Embeddings

No step. Class R sets `UpdatedAt`, which satisfies `i.UpdatedAt > e.CreatedAt` and re-embeds on the next
`EmbeddingBackgroundService` pass (§1.3). **Acceptance must verify the heal actually happened rather
than assuming it** — see C1.4.

Class Q: the vector persists (§2.7) but is filtered from search at hydration, so it is inert. Deleting it
is optional hygiene; if done, it must go through the EF entity, not raw DML. The 90 already-inactive rows
holding embeddings are the standing evidence that nothing cleans these up.

### 5.4 Acceptance

**C1.0 — take the pre-images BEFORE the live run.** Two readings are only obtainable while the damage is
still present, and both are inputs to criteria below:

1. **The embedding basin (C1.5's baseline).** Run the semantic-search control
   `"Board of Executive Directors"` against live search and **record which rows come back, by id**.
   The inherited "5/5 polluted" is `[I]`, and §8 item 4 previously deferred measuring it to C1.5 —
   but **C1.5 runs after the repair**, so a post-only reading cannot distinguish "the repair drained
   the basin" from "the basin was never there on this population". Deferring the baseline past the
   repair is NT-2 in its purest form: a sample that structurally cannot contain the counter-example.
   This reading costs one query and is unrecoverable once C1 writes.
2. **The dry-run class counts** (§5.1), diffed against §5.2's expected 444 / ~1,250 / explicit-list.

Run the dry run first and diff it against these expected counts; a dry run that reports numbers far from
them means the predicate is wrong, and that is the cheapest place to find out.

| # | Measure | PASS | FAIL |
| --- | --- | --- | --- |
| C1.1 | Response `mode` field | `live` | `dryRun` while a live run was intended — halt, do not read the counts |
| C1.2 | Class-R rows whose `name` still differs from the cached `canonical_name` | **0** | > 0 |
| C1.3 | Rows carrying `Metadata.name_repair_v1` | **= rows changed**, and re-running the endpoint changes **0** further rows (idempotence) | a second run changes rows → the sentinel is not being honoured |
| C1.4 | Repaired rows whose `instrument_embeddings.created_at` < `instruments.updated_at`, 6h after the run | **0** | > 0 → the heal did not fire; do not assume it did |
| C1.5 | Semantic-search control: query `"Board of Executive Directors"`, run **twice** — the C1.0 baseline before the live run, and again after | the post-run result returns **0** of the row ids the C1.0 baseline returned | still returns baseline rows → names repaired but the basin persists, i.e. C1.4 lied. **If C1.0 itself returns 0 polluted rows, C1.5 is VOID, not passing** — there was no basin on this population and the criterion measured nothing. Recording a void C1.5 as green is the failure this row exists to catch. |
| C1.6 | **Negative control (NT-2/NT-4):** total active instrument count | **unchanged ± the class-Q count** | a larger drop → the repair deactivated more than intended |

C1.6 is the check that catches a predicate that matched too much. Without it, a repair that quarantined
half the catalog would pass C1.2 and C1.3 cleanly.

### 5.5 Rollback — and why it is not ZFS

**The pre-image exists nowhere else.** `aliases` holds **0 rows** for the entire self-seeded population
`[M]` — the surface→instrument association was never recorded. So the corrupt `name` is simultaneously
the damage and the only surviving record of the surface that caused it. Overwriting it without capturing
it destroys the evidence for any future analysis of *which* surfaces corrupt the catalog. This is the
concrete reason the brief's "an authoritative source may not be able to restore it" is correct.

Therefore, in order:

1. **Per-row pre-image in `Metadata`, written before the mutation** (the `:206-212` precedent). This is
   the real rollback: a reverse endpoint reads `Metadata.name_repair_v1.previous_name` and restores it,
   per-row, via EF. **Ship the reverse endpoint in the same PR as the forward one** — a repair whose
   undo is "we'll write it if we need it" is not reversible.
2. **A fresh ZFS snapshot of `nvme-fast/timeseries` immediately before the live run.** Note
   `@pre-spec-20260806` predates this work and is *not* a usable undo for C1 alone: the dataset holds
   **all** of `atlas_data` and `atlas_secmaster`, so rolling it back would discard every unrelated
   collector write since 07:00Z that day. MANIFEST §3 also records that the snapshot is taken under a
   live Postgres and is therefore **crash-consistent**, recovering via WAL replay like a power loss.
   Treat ZFS as the disaster floor, not the step rollback.
3. **Not available:** re-deriving the original names from an upstream source. There is no upstream — the
   names were locally invented. This is what makes (1) mandatory rather than a nicety.

---

## 6. What could go wrong with the plan itself

- **A1 raises resolution but the values are wrong.** `context` is a free-text surface phrase feeding an
  exact alias lookup. If it matches aliases loosely, the numeric path could start writing *incorrect*
  macro observations — worse than writing none, because the matrix consumes them silently. The A1
  adversarial test is the guard; if it cannot be made convincing, ship A1 behind a shadow write
  (`extracted_observations_shadow` already exists `[M]`) and compare before enabling the real write.
- **A1 restores little.** The numeric path's best-ever throughput was 25 rows/day (§2.3). If the
  business value is not there, A1 is still worth landing for the contract test (NT-1) but should not
  justify further investment on its own.
- **This plan may not be the highest-value next thing, and a reader deciding what to do needs both
  numbers side by side.** Every step here targets a path with a small measured ceiling: the numeric
  substrate path's best-ever day was **25 rows** (§2.3), and C1's repairable set is capped at
  **444 of 1,694** by local cache coverage (§5.2). PR #912
  (`docs/proposals/extraction-type-classifier.md`, rev-2 — read *after* this plan was drafted, §8 item 8)
  measures a constraint an order of magnitude larger on a **different** path: **~60% of articles ground
  zero entities** (7-day; 43–75% daily), **~74% produce zero macro signals** (single 24h reading, not a
  weekly rate), and a ranking-free `MaxCandidates=10` cap discards **22.9%** of candidates — in
  probe-then-document order, with no ranking — before anything downstream sees them. **Not a
  contradiction:** #912's gap is on the news-signal path that already writes ~550 rows/day (§2.1); this
  plan's defects are on the numeric path that writes zero. They are different paths and both facts are
  true. They do compete for the same attention, and this plan cannot adjudicate that from inside itself.
  What survives the comparison: **A1/A2 earn their place on the NT-1 contract test, not on throughput,
  and B1 earns its place because the faucet is open and is corrupting the catalog now** — neither claim
  rests on volume, so neither is weakened by #912. What does not survive: any argument that this
  workstream is the best available use of effort. #912 §10 items 1–3 are independently actionable and
  cheaper than any step here.
- **C1's disposition is wrong at the margin.** The R/L/Q split rests on the local OpenFIGI cache, which
  covers 487 of 1,694 rows (28.7%) `[M]`. The other 71% get no verdict. That is the honest state, and
  the plan deliberately leaves them alone rather than guessing — but it means C1 does not "finish" the
  problem, and anyone reading a green C1 as "catalog clean" is repeating NT-2.
- **B1 starves `ContextFactor`.** Ticker-shaped names could degrade Sentinel's subject matching. B1.2 is
  a rate check, not a quality check; if downstream matching degrades, that surfaces later and B1 may need
  the display-name path revisited.
- **Both fix branches are unpushed and live in other agents' worktrees, one locked** `[M]`. They can be
  lost or diverge. First action on approval: push both to origin.

**Abandon or re-plan if:**
- A1 lands and A1.1 passes but A1.3 stays at or below 4 → the break is not only the slot contract, and
  the root cause is not yet understood. Stop; do not add a second speculative fix on top
  (PROBLEM_SOLVING).
- C1's dry run reports a class-R count wildly off from 444 → the predicate is wrong. Do not run live.
- B1 causes B1.2 to hit zero → the faucet was closed by breaking resolution. Revert immediately; C1 is
  moot until self-seeding works again.
- The measured junk fraction, once someone actually labels a sample, comes in near zero → the whole B/C
  workstream is mis-premised. Nobody has labelled one; §8.

---

## 7. Traceability and verification

### 7.1 Pre-merge checklist (this is the structural part of NOT-TWICE)

Every NT entry must map to a shipped step. Reviewers check this table, not the prose.

The **Enforced by** column is the honest part. Only `gate` rows fail on their own; `checklist` rows fail
only if a reviewer notices (§3). Do not report this table as five mechanisms.

| Entry | Enforced by | Mechanism | Step | Done when |
| --- | --- | --- | --- | --- |
| NT-1 | **gate** | Captured-payload contract test, both language halves | A1 | Test present **and** fabricated fixtures removed |
| NT-1 | **gate** | Funnel alert, extracted → resolved → written | A2 | Rule loads, expression non-empty, forced-fire verified |
| NT-2 | *checklist* | Population statement + negative control per measurement | §0, all acceptance tables | No criterion lacks a negative control — reviewer judgement, nothing fails automatically |
| NT-3 | *checklist* | Adversarial case written before merge | A1, B1 | `ADVERSARIAL:` answered per step; the tests it produces are gates, the decision to write it is not |
| NT-4 | **gate** | Mutation-verify guards and alerts; `absent()`-guard expressions | A2, B1, C2 | Each broken deliberately, seen red, restored |
| NT-5 | **gate** | No-op-rate alert on repair paths | C2 | promtool test green in all three directions (fires on the pre-C1 shape, silent when healthy, silent when idle) **and** the live rule mutation-verified |

**No step ships without its row green.** A step whose NT row is red is an unfixed failure mode with new
code layered on top — which is how this list got to five entries, three of which are enforced.

### 7.2 Verification of this document
Docs-only change. No compile, no deploy, no data mutation. All DB access during authoring was SELECT-only;
no Gemini calls were made. Verified via `scripts/claude-mark-verified` after the final commit.

---

## 8. What I could not verify

Stated plainly, because a plan that hides its gaps reproduces NT-2.

1. **The junk fraction (~300–450 of 1,688, ≈18–27%) `[I]`.** Not re-derived — it needs a labelled sample
   and I made no Gemini calls. Everything about C1's *value* (not its mechanism) rests on it. The
   inherited "~19 junk rows/day" does not reconcile with the measured ~30 self-seeds/day at that fraction
   (§4.0).
2. **The inherited "368 with an authoritative OpenFIGI name in the local cache."** I measure **487**
   cached and **444** differing. I could not reconstruct a predicate yielding 368 — possibly a freshness
   (TTL) filter I did not apply, since `TryGetFreshAsync` evaluates TTL caller-side `[M]`. C1's dry run
   settles it before any mutation.
3. **Extraction quality (99.8% verbatim, 98.8% values correct) `[I]`.** Not re-measured; requires
   sampling against source text.
4. **The embedding attractor basin ("Board of Executive Directors" → 5/5 polluted) `[I]`.** Not
   re-measured — running it means live semantic search. **It is now C1.0, taken immediately before the
   live repair, not C1.5.** An earlier draft deferred it to C1.5, which runs *after* the repair; a
   post-only reading cannot tell a drained basin from one that never existed, so the criterion would
   have been unfalsifiable in the direction that matters. C1.5 is now a diff against the C1.0 baseline
   and is explicitly **void** if that baseline is empty.
5. **`exact_rejected_name` as a criterion.** It is a status tag at `DeterministicResolver.cs:324` `[M]`,
   and its parent counter emits **no series** in Prometheus over 24h `[M]`. I could not determine whether
   the counter is unwired or merely never incremented. Until that is resolved it cannot gate A1 — and
   "no series" must never be read as "fell to zero".
6. **The inherited "6,648 model calls" `[I]`.** Not re-measured. Adjacent measured values for the same 7d
   window: 7,105 articles collected (all processed), 4,278 distinct articles producing ≥1 extracted row,
   49,258 extracted rows, 49 resolved `[M]`. The inherited 7,098 articles matches 7,105 within window
   drift; the inherited 45,748 rows is the same measure on an earlier window.
7. **The post-cutover census z-test (z=0.96, p≈0.34) `[I]`.** Not recomputed. The directional claim it
   supported — no improvement from the earlier merges — is independently corroborated by the +6 self-seed
   drift and the 20:47Z row (§1.2).
8. **PR #912's spec (`docs/proposals/extraction-type-classifier.md`) — RESOLVED, was: not read.** It has
   since been read at rev-2. It does **not** constrain the adapter or any step here; its verdict is
   DO-NOT-BUILD and it designs nothing that touches the numeric path. What it *does* carry is a much
   larger measured gap on the news-signal path, which changes this plan's priority claim but none of its
   mechanisms — recorded in §6, not here, because it is a live consideration rather than a gap in the
   evidence. Two of its conventions are now adopted directly: the promtool synthetic-series test shape
   (C2, from its §10.2) and the reset-checking discipline (item 9 below).
9. **Counter resets — checked late, and clean for the windows quoted.** #912 records that Sentinel and
   SecMaster counters reset frequently. Re-checked here at 2026-08-07T01:36Z:
   `sentinel_dsl_adapter_row_decision_total` shows **0 resets over 24h** but **8 / 6 / 3 resets over 7d**
   across its `hit` / `miss` / `no_candidates` series `[M]`. Every PromQL figure in this plan uses
   `increase()`, which is reset-correct, so no number moves — but **A1.2 and A2 must keep using
   `increase()` and must never read a raw counter value as a total**, and any 7-day restatement of
   §1.1's decision split has to carry the reset count with it. Un-checked at authoring time; the
   discipline is adopted from #912's evidence appendix.
