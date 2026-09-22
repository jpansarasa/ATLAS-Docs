# Deleting wrongly-mapped SecMaster instruments

**Status:** PROPOSAL — merging this document is not approving it.
**Measured:** 2026-09-21 against atlas_secmaster and atlas_data, SELECT-only throughout. Authored at main `97475644`; every figure re-derived at `9f0f0ce8` on **2026-09-21T17:26Z** after a blocking review, and again on **2026-09-21T18:07-18:09Z** after a second one. Live counters moved between the two rounds, so every figure carries its own stamp AT THE POINT IT IS USED, never once per section.
**Authored:** by a read-only planning agent; nothing in it was executed.

## Verdict in one sentence

**Delete leg A (82 GeminiFallback equity/ETF rows) and leg C (DX, KC) in two separately-sequenced migrations; DO-NOT-BUILD leg B (9 rows)** — but the case for deletion is weaker than the brief assumes, because re-acquisition is *already working on the news-surface path* without it (3 of the 8 quarantined ETFs; **0 of the 74** quarantined equities), and the case against leg B is neither the supervisor's reason nor the one this document first gave.

---

## 0. Corrections to inherited figures

Every number in the brief was treated as a hypothesis. Seven needed correction.

| # | Inherited | Re-derived (2026-09-21) | Consequence |
|---|---|---|---|
| 1 | "9 invented FRED series ids, rejected by the live FRED API" | **8** invented FRED ids (`discovery_source='GeminiFallback'`, class `fred_series`) **+ 1 unrelated row**: `GSV.NE` "2025 full year GSV", class `Economic Indicator`, `discovery_source` **NULL**, created 2026-05-01, a legacy SentinelCollector row | Leg B is two provenances, not one. `GSV.NE` is a dotted ticker, not a FRED id; the FRED-API discriminator never applied to it |
| 2 | "the QUARANTINES broke 82 re-registrations" | **Partly refuted — and NOT by the path the first draft credited. 3 of the 82 tickers have been re-acquired, correctly, with the quarantined row still in place**: `INDA`→ISHARES MSCI INDIA ETF (2026-09-11), `IWS`→ISHARES RUSSELL MID-CAP VALU (2026-09-13), `UUP`→INVESCO DB US DOLLAR INDEX B (2026-08-20). All three active rows carry `discovery_source='GeminiFallback'` — the **news surface** (`POST /api/instruments`), not a collector registration. All three are ETFs: **3 of the 8** quarantined ETFs, against **0 of the 74** quarantined equities in the 65 days since the last quarantine write (2026-07-18) | The premise is partly false, but only on the path the catalog treats as LEAST authoritative. The demonstrated rate does not generalise to equities. See §2 |
| 3 | `~2,658` rows on `OriginalInstrumentId` | **2,678** (A 1,402 / B 47 / C 1,229) at 2026-09-21T17:26Z | Grew 20 since the brief, and 2 more between this document's two drafts — the pipeline is live. Every figure on this column needs an as-of stamp, including this one |
| 4 | "455+79 such orphans already exist" | **455 orphaned distinct `OriginalInstrumentId` values spanning 26,505 rows; ZERO orphaned `instrument_id` values** (5,019 of 5,019 distinct refs resolve) | The "79" is not reproducible. `instrument_id` referential integrity is currently perfect — deletion would be the *first* break |
| 5 | Two affected columns | **Three.** `sentinel.extracted_observations` also has `CorrectedInstrumentId` (today 0 hits for our population) | The paired migration must cover all three or a future correction re-orphans |
| 6 | "six guards read the state as a decision" | **Eight sites.** The brief's six plus `RegistrationService.cs:338` and `:402` | See §3 — and `:402` is dead code |
| 7 | "`quarantined_skip` … each triggering a paid-Gemini confirm cascade" | **192.0/30d confirmed exactly** (192.001 at 2026-09-21T18:08Z). The first draft's denominator, "54,276 confirmations", was the **`unconfirmed` arm** — a population that structurally CANNOT contain a `quarantined_skip`, because the self-seed returns before reaching the quarantine branch unless `confirmation.Confirmed` (`EntityResolutionService.cs:1029-1032`). `sum by (result) (increase(secmaster_entity_resolution_confirmation_total[30d]))` at 2026-09-21T18:08Z is **confirmed 59,539 / unconfirmed 54,387** (unit: extrapolated counter events over 30d). So 192 is **0.32% of the 59,539 confirmed** or **0.17% of all 113,926 cascade events** — never 0.35%, which divided neither arm. Same-instant neighbours: `secmaster_gemini_resolver_calls_total{outcome="success"}` **57,233**, `cap_exhausted` **12,050** | The count is right; the *cost* framing is not, and a denominator on this metric must NAME ITS ARM. This is not a material saving and must not be the justification |

**What these samples structurally cannot contain:** the 93-row population is *only* rows some migration already chose to quarantine — it cannot contain wrong mappings nobody has yet noticed, and it cannot contain wrong mappings on rows that stayed **active**, the far larger GIGO surface. That surface is sized in `SecMaster/AGENT_README.md`'s `✗ scope-search-by-source-mapping` GOTCHA (line 94 at `9f0f0ce8`), not in `docs/BACKLOG.md`, and it is TWO figures in different units: **3,118 of the 4,432 instruments** Sentinel has ever attached to carry no source mapping, spanning **21,441 unmapped rows**. ("3,118" appears nowhere in `docs/BACKLOG.md` at either revision; the first draft cited the wrong file and read an instrument count as a row count, understating the row surface ~7x.) The `quarantined_skip` counter has no symbol label, so the 192/30d cannot be attributed per row or per cause from Prometheus — the per-symbol view in §2 comes from Loki instead, bounded by the 7 days queried, so a symbol that declined only outside that window is invisible to it. The re-acquisition evidence (correction 2) is **ETF-only AND news-surface-only**: it establishes nothing about equities (0 of 74 in 65 days) and nothing about the operator-config or collector-registration paths, neither of which was observed acquiring anything at all.

---

## 1. GRANULARITY — delete the `instruments` row; the `source_mappings` row is a no-op

**Decision: delete the `instruments` row only. Do not write a `source_mappings` DELETE.**

The user's words name the MAPPING. The database says that row almost never exists:

```
cause                      instruments  source_mappings  aliases  embeddings  sector_overrides
A gemini equity/etf                 82                0        0           0                 0
B fred-rejected + legacy             9                1        0           0                 0
C futures roots                      2                0        0           0                 0
```

**92 of 93 quarantined instruments own zero source mappings.** The single exception is `GSV.NE` → `(SentinelCollector, GSV.NE)`, `is_primary=true`. So a `source_mappings` DELETE would be a no-op for 98.9% of the population, and for the one row it does reach it is subsumed by cascade (below).

### The index evidence

Both indexes, read from the live catalog:

```
idx_instruments_symbol                UNIQUE, btree (symbol) WHERE is_active = true     -- PARTIAL
idx_source_mappings_collector_source  UNIQUE, btree (collector, source_id)              -- NOT partial
```

and every child FK cascades:

```
aliases               FK ... ON DELETE CASCADE
instrument_embeddings FK ... ON DELETE CASCADE
source_mappings       FK ... ON DELETE CASCADE
```

**Which index a re-registration collides with — proven, not claimed:**

- **For legs A and C (84 rows): neither.** The quarantined row is `is_active=false`, so it is *not in* `idx_instruments_symbol` and reserves nothing. It holds no mapping, so it occupies no `(collector, source_id)` slot. The blocker is **entirely application code**. The live proof is correction 2: three of these exact symbols now have an active row sitting beside the quarantined one. That state is only reachable if the partial index permits it, and the catalog-wide count agrees — exactly **3** symbols have one active plus ≥1 inactive row, and they are `INDA`, `IWS`, `UUP`.
- **For `GSV.NE` alone: the mapping is the blocker, and it is the *non-partial* index that binds.** `(SentinelCollector, GSV.NE)` is globally unique regardless of `is_active` on either table, so a re-register hits `RegistrationService`'s existing-mapping early return (`:328`) and re-binds to the corpse. This is the one row where the user's word "mapping" is literally correct — and it is in the leg I recommend **not** building.

**Therefore:** deleting the `instruments` row releases everything, via cascade, in one statement. This matches `PurgeLoRAEraQuarantinedInstruments` exactly, whose comment already recorded the mechanism: *"The source_mappings FK is ON DELETE CASCADE — the dead mappings drop with the instruments."*

---

## 2. POPULATION — delete A and C (84); keep B (9), and the supervisor's reason largely holds

### The supervisor's standing recommendation is right in outcome, and closer to right in reasoning than the first draft allowed

The claim was: *keep B because #874 fixed NAMES not INVENTION, so the quarantine IS the guard there.* The first draft answered **"the quarantine is not the guard — D-4's allowlist is"**. Half of that is correct and the other half is the half that matters. D-4's allowlist is what stopped the INVENTION arriving, and the data shows it working:

```
asset_class  count  first        last
Equity          92  2026-06-25   2026-09-20
Economic        64  2026-06-25   2026-07-18   <- stopped
ETF             48  2026-06-25   2026-09-21
Currency         9  2026-06-26   2026-06-30   <- stopped
Commodity        9  2026-06-25   2026-07-27   <- stopped
fred_series      8  2026-06-29   2026-07-09   <- stopped
Index            3  2026-06-25   2026-07-14   <- stopped
Rate             2  2026-06-25   2026-07-04   <- stopped
```

Every macro-shaped class from `GeminiFallback` stopped in July 2026; only `Equity` and `ETF` continue. `fred_series` invention has been **zero for 74 days**.

**These rows do NOT arrive through `RegistrationService`. That attribution is WITHDRAWN here, not only under *The honest weakness in the case for deleting A* below.** Measured 2026-09-21T18:08Z: `source_mappings` holds **ZERO** rows with `collector='GeminiFallback'` — the whole table is FredCollector 5,081 / SentinelCollector 2,230 / OfrCollector 130 / FinnhubCollector 18 — against **235** `GeminiFallback` instruments (143 active). `RegisterCoreAsync` writes a `SourceMappingEntity` on BOTH branches (`:422`→`:431` attach, `:484`→`:494` new row) and stamps `DiscoverySource = collector` (`:473`), so any row that came through it would carry a `GeminiFallback` mapping. None does, so none came through it. The route is `POST /api/instruments`. *What this sample cannot contain:* a mapping deleted after its instrument was written — but §1 measures the same population from the other side, and 92 of the 93 quarantined rows own zero mappings today.

**D-4 stops the invention regardless, because the REST route calls the SAME predicate.** `EvaluateGuard` is one `internal static` implementation (`RegistrationService.cs:189`) and `InstrumentEndpoints.cs:106` calls it before the entity is built. It rejects `macro_class_from_untrusted` — `GeminiFallback` is absent from `TrustedMacroCollectors` {FredCollector, FRED, BLS, OFR, OfrCollector} and `fred_series` is absent from `EquityShapedAssetClasses` {Equity, ETF, Index, Currency, Crypto, Commodity}. That shared predicate is why the macro-shaped classes stopped, and the supervisor's sentence rests on it.

**On that path the quarantined rows do no guard work — MORE completely than the first draft claimed.** Its reason was `GetBySymbolAsync`'s `IsActive` filter; the REST route performs no symbol read at all to filter. Between `EvaluateGuard` (`:106`) and `repository.AddAsync` (`:149`) there is no `GetBySymbolAsync`, no `GetBySymbolIncludingInactiveAsync` and no quarantine read. The first draft over-generalised it into "the quarantined rows do no guard work", full stop, which its own §3 site 4 contradicts and live measurement refutes: the SecMaster-internal self-seed reads the quarantined row *deliberately* (`GetBySymbolIncludingInactiveAsync`) and declines on it **50 times in 7 days**, 8 of them on leg B. D-4 does not cover that path either, and this is checkable rather than asserted: `EvaluateGuard` has exactly **two** call sites in the service — `RegistrationService.cs:111` (gRPC register) and `InstrumentEndpoints.cs:106` (REST) — while the self-seed inserts straight through `ctx.InstrumentRepository.AddAsync` (`EntityResolutionService.cs:1102`), calling neither. And it self-seeds macro-shaped classes today (`RRSFS` Economic, `DEXUSNZ` Currency, both 2026-09-18). The quarantine is the only thing declining leg B. **So the supervisor's sentence stands and the first draft's correction of it does not:** D-4 stops the invention ARRIVING through either register ingress, because `EvaluateGuard` is shared by both; the quarantine is what declines it RECURRING through the self-seed, which reaches no ingress guard at all.

### The real reason to keep B: deleting it would RE-MINT the junk

> "Then if the instrument shows up again, we get *another* chance to map it correctly."

Leg B's symbols are **invented** — `BX.KLT.DINV.CD.WD`, `MCRFPC1`, `USSLD90`, `NAPMNOI`. There is no instrument behind them, so the "second chance" the user wants has nothing to buy. That much stands.

**What does NOT stand is the first draft's "it is inert", because its evidence was zero BY CONSTRUCTION.** It cited *"leg B has 0 rows in `sentinel.extracted_observations.instrument_id`"* — still 0 at 2026-09-21T17:26Z, and still worthless as proof. The self-seed refuses at §3 site 4, so no ACTIVE leg-B row exists, so nothing can attach to one. Deletion is the only act that could ever populate the measure being used to prove deletion pointless. A zero that only deletion could move is not evidence about deletion.

**Measured instead, per row, leg B is the most active leg of the three.** `{service_name="SecMaster"} |= "Self-seed declined for confirmed"` over the 7 days ending 2026-09-21T17:26Z (events span 2026-09-14T19:09Z → 2026-09-21T12:22Z): **50 declines across 21 distinct symbols**. Mapped back to the legs: **19 leg-A symbols / 42 events**, **2 leg-B symbols / 8 events** (`USCOSPRE` 6, `CONCCONF` 2, both `fred_series`/`GeminiFallback`), **leg C zero**. Per row per day that is **0.127 for leg B's 9 rows against 0.073 for leg A's 82** — B recurs at ~1.7x A's rate, ≈34 events/30d. *What this sample cannot contain:* symbols whose articles did not arrive inside those 7 days, and the confirming SOURCE of each event — neither the log line nor the `self_seed` counter (labelled `result` only) carries it.

**So the reason to keep B is the opposite of inertness: deleting it would re-mint exactly the junk §5 says can no longer be minted.** Every gate upstream of the quarantine branch has *already passed* on those 6 `USCOSPRE` events — the decline at `EntityResolutionService.cs:1079` sits after `EnableSelfSeed`, after `Confirmed`, and after D-16's `ConfirmedAssetClass` refusal (`:1036-1038`). Remove the row and that same traffic falls through to the insert at `:1090`, where `Name = SelfSeedName(confirmation)` resolves the name from the confirming source — and only `OpenFigi` and `Finnhub` assert one. Every other source falls to `_ => null` and stores the **TICKER**: a fresh, active, embedded row named `USCOSPRE`, whose name is its own symbol, for a symbol that is not an instrument. That shape is live at scale on this exact writer: of the **1,660** rows self-seeded since 2026-08-01 (`discovery_source LIKE 'entity_resolution:%'`, the sole writer being `EntityResolutionService.cs:1098`), **174 have `name = symbol`** — 173 of 1,464 Gemini-confirmed (**11.8%**) and 1 of 196 OpenFIGI-confirmed. `CatalogNameRepairService.cs:42` lists `entity_resolution:gemini` as an unauthoritatively-named source for precisely this reason.

**DO-NOT-BUILD leg B** — not because deletion has no effect to produce, but because the effect it produces is the defect.

### Legs A and C are genuinely wrongly mapped

Every one is a real, tradeable ticker bound to an unrelated NER surface (`NSRGY`="Nomura" but NSRGY is Nestlé; `DX`="Japan" but DX is Dynex Capital; `KC`="Colombia" but KC is Kingsoft Cloud). These are exactly what the user described, and the second chance is real — `COLO`→GLOBAL X MSCI COLOMBIA ETF was seeded correctly on 2026-09-18 from the same "Colombia" surface family that produced `KC`.

### The honest weakness in the case for deleting A

Correction 2 shows re-acquisition happening without deletion — but **on the news surface, and only for ETFs**, and the first draft got the mechanism wrong. All three active rows carry `discovery_source='GeminiFallback'`, which `InstrumentEndpoints.cs:142` stamps from the request's `Collector`; the senders are SentinelCollector's `DeterministicResolver.cs:1080` and `Workers/ExtractionProcessor.cs:1752`, and the client posts to `/api/instruments` (`SecMasterClient.cs:367`). That route runs `EvaluateGuard` and then `repository.AddAsync` (`:106`, `:149`) — **no `GetBySymbolAsync`, no symbol pre-read, no quarantine read anywhere**. It succeeds only because `idx_instruments_symbol` is partial, so a second active row may sit beside the quarantined one. The first draft credited this to the collector-registration path and to `GetBySymbolAsync`'s `IsActive` filter; that attribution is **withdrawn**. `GetBySymbolAsync`'s comment does say soft-deleted rows must not shadow a re-registration, but no collector registration was observed re-acquiring any of these 82 symbols, so the comment is a design intent here, not an observation.

The population is the other half of the honesty. **3 of the 8 quarantined ETFs re-acquired; 0 of the 74 quarantined equities did, in 65 days.** D-4 permits `Equity`, so the path is reachable by equities *in principle* — the demonstrated RATE is ETF-only and must not be extrapolated.

So deletion's remaining benefit is narrow, and it is two SecMaster-internal sites, not one: the discovery drop (§3 site 1), whose refusals carry **no counter at all** and are visible only as Warning log lines, and the self-seed (§3 site 4), worth **~192 skips/30d across all 93 rows — 0.32% of the 59,539 CONFIRMED cascade results over the same 30d, or 0.17% of all 113,926** (2026-09-21T18:08Z; §0 correction 7 — the denominator is the `confirmed` arm, and the `unconfirmed` arm cannot contain a `quarantined_skip`). The 192 is the self-seed's number only; nothing sizes the discovery drop. That is a real but small win. Leg A is worth building because it is cheap, low-risk and matches explicit user intent; it is **not** worth building on a cost argument, and the PR body must not make one.

---

## 3. THE GUARD SITES — eight, not six, and one is already dead

| # | Site | Reads quarantine as | After deletion |
|---|---|---|---|
| 1 | `CatalogService.cs:199` | Discovery drop — "re-acquiring a retired ticker is a deferred decision" | **Flips to accept.** Row absent → `existing is null` → discovery caches a fresh active row. *This is the point for A and C.* |
| 2 | `CatalogService.cs:218` | D-13 `IsProposable()` drop | **No change.** Unreachable for our population — site 1 `continue`s first. Fires only on `retired_at` (the disjoint 1,308) |
| 3 | `CatalogService.cs:289` | D-13 re-check of the just-saved row | **No change.** Judges the saved row, not ours |
| 4 | `EntityResolutionService.cs:1079` | `quarantined_skip`, self-seed declined | **Flips to insert.** The 192/30d stop; the next mention seeds a fresh active row, named by the confirming source and falling back to the TICKER (§5) |
| 5 | `FredCatalogReconciliationService.cs:125` | Skips a quarantined row lacking a FredCollector `is_primary` mapping (`QualifiesForReactivation`, `:62-70`; the skip branch is `:130`) | **Becomes unreachable for these ids.** Harmless: it already skips all 93 (none carries a FredCollector mapping — the only mapping is SentinelCollector's on `GSV.NE`, in the leg we keep) |
| 6 | `InstrumentConfigurationWatcher.cs:198` | Deliberately unfiltered (`:192-196`, the second paragraph of the block headed INTENT(D-11) at `:185`) so operator config *reactivates* rather than inserting beside | **Flips to insert.** An operator config naming a deleted symbol now creates a clean new row instead of reviving a junk one — an improvement, but it silently changes the watcher's semantics for those symbols |
| 7 | `RegistrationService.cs:338` | Existing-mapping early return warns `registration.instrument_inactive` | **Reachable only via `GSV.NE` today.** Unchanged if B is kept |
| 8 | `RegistrationService.cs:402` | `if (!instrument.IsActive)` warn-and-continue (`:404` is its comment) | **Dead code, before and after.** Its subject comes from `GetBySymbolAsync`, which filters `IsActive` — it can never return an inactive row. Worth filing separately; do not "fix" it in this PR |

Sites 1, 4 and 6 are the behaviour change. Sites 2, 3, 5, 7, 8 are no-ops for this population — the brief's framing that "deleting flips each from refuse to accept" holds for three of eight, not six of six.

---

## 4. CROSS-DATABASE ORPHANS — the paired migration, and which edge is load-bearing

A SecMaster migration cannot reach `atlas_data`; there is no FK between the databases.

**Blast radius, measured 2026-09-21T17:26Z** (the `OriginalInstrumentId` row moves; the others were static across both drafts)**:**

| Column | Leg A | Leg B | Leg C | Total |
|---|---|---|---|---|
| `instrument_id` | 26 rows / 12 ids | **0** | **322** | 348 |
| `OriginalInstrumentId` | 1,402 | 47 | 1,229 | 2,678 |
| `CorrectedInstrumentId` | 0 | 0 | 0 | 0 |

Two facts bound the risk:

- **All 348 attached rows are terminal** — 338 `Approved`/`Resolved`, 10 `AutoClosed`/`NoResolution`, **zero Pending**. The review UI's actionable query (`Pending AND InstrumentId!=null AND ResolutionState==Resolved`) selects none of them, so the human queue is untouched.
- **`OriginalInstrumentId` HAS a production reader, and the safety comes from a MEASUREMENT, not from an absence.** It is written at `ExtractedObservation.cs:312/471/535` — the only three `OriginalInstrumentId = InstrumentId;` writes in the service — and it is **READ at `:531`**, four lines above the third write, by `SnapshotOriginalResolutionIfAbsent`'s presence probe:

  ```csharp
  if (OriginalSymbol != null || OriginalInstrumentId != null || OriginalResolutionMethod != null)
      return;
  ```

  Presence of ANY of the three counts as "a snapshot exists", deliberately (`:508-512`: `OriginalSymbol` is legitimately null on a row snapshotted with `Symbol=null`, so no single column is a safe probe). The paired migration NULLs a column that probe reads. **It is harmless TODAY, and the number is the reason — not the reader's non-existence.** At 2026-09-21T18:07Z, of the **240,237** rows carrying a non-null `OriginalInstrumentId`, **0** carry it as the ONLY non-null of the three; of the **2,678** rows in the NULLing population, likewise **0**. Clearing `OriginalInstrumentId` alone therefore cannot flip the probe on any row that exists. *What this sample cannot contain:* rows written after the query instant, and rows with all three columns already null — which the probe reads as "no snapshot yet", the state it exists to fill.

  **CONSEQUENCE FOR THE IMPLEMENTER — do NOT widen the NULLing to `OriginalSymbol` or `OriginalResolutionMethod` "for tidiness".** The three together ARE the probe. Clearing all three re-arms the snapshot, and the next `ApplyReExtraction` or resolution write on those 2,678 rows overwrites the provenance §8 makes load-bearing with the post-deletion state. Re-run the 0-of-240,237 / 0-of-2,678 count at deploy time; if either is non-zero, the ordering argument below needs re-deriving before the migration ships.

  Separately, 455 ids / 26,505 rows are *already* orphaned on this column — the residue of `PurgeLoRAEraQuarantinedInstruments` — with no recorded breakage and no metric (the only `orphan` counter is `sentinelcollector_macro_observations_orphan_signal_identity_total`, a different column).

### The paired migration

`SentinelCollector`, EF-generated in the devcontainer per CLAUDE.md §MIGRATIONS (never hand-written):

```
nerdctl compose exec -T sentinel-collector-dev sh -c \
  "cd /workspace/SentinelCollector/src && dotnet ef migrations add NullWrongMappingInstrumentRefs --output-dir Data/Migrations"
```

Body: an enumerated-id `UPDATE` setting `instrument_id`, `OriginalInstrumentId` and `CorrectedInstrumentId` to NULL where each matches the leg's id list, as a compare-and-swap on the recorded id (the D-14/D-19 shape). Enumerate ids, never a predicate — `atlas_data` cannot see `is_active`, so there is no predicate to write. `Down` restores by id from the same frozen list. SecMaster's side, separately:

```
nerdctl compose exec -T secmaster-dev sh -c \
  "cd /workspace/SecMaster/src && dotnet ef migrations add DeleteWrongMappingInstruments --output-dir Data/Migrations"
```

Body: `DELETE FROM instruments WHERE id IN (<enumerated>) AND symbol = <recorded> AND name = <recorded> AND NOT is_active;` — a four-way compare-and-swap so a row anyone reactivated or renamed since authoring is left alone. `Down` is a documented irreversible, as in `PurgeLoRAEraQuarantinedInstruments`.

### Deploy order — Sentinel FIRST, and which edge is load-bearing

**Deploy SentinelCollector's NULLing migration, verify, then deploy SecMaster's DELETE.**

The load-bearing edge is **Sentinel-before-SecMaster on `instrument_id`**, and only on `instrument_id`:

- **If SecMaster deletes first**, the window between deploys leaves 348 rows pointing at ids that no longer exist. Any path that resolves an attachment by id — `ReExtractResolutionAdapter`, D-33's replay, `/ui/review` rendering — sees "SecMaster has no such instrument" and cannot tell that from a SecMaster outage. D-27's whole point is that a dependency being absent is no verdict on the row; a deleted id manufactures a *permanent* false absence.
- **If Sentinel NULLs first**, the intervening state is a row with no attachment and its provenance preserved in `OriginalInstrumentId` — which is precisely what D-33 already writes 1,385 times on this exact population. The system is known to tolerate it.

The reverse edge does not bind for `OriginalInstrumentId`: its only reader is the presence probe above, which the 0-of-240,237 measurement shows cannot flip while just that one column is cleared, and 455 ids are already orphaned there — so its ordering is free. State the edge as `instrument_id`-scoped, not as a blanket ordering.

---

## 5. THE #874 CLAIM — verified in code; the faucet is open, and "clean" only in the sense #874 fixed

**The write path can no longer persist a junk NER SURFACE as a name — it stores the TICKER instead, deliberately. Verified at `EntityResolutionService.SelfSeedName` (`:1189`):**

```csharp
var sourceName = confirmation.Source switch {
    ConfirmationSource.OpenFigi => confirmation.CanonicalName,
    ConfirmationSource.Finnhub  => confirmation.CanonicalName,
    _ => null,
};
return string.IsNullOrWhiteSpace(sourceName) ? confirmation.Ticker! : sourceName;
```

The raw NER surface is never persisted as `Name` — a Gemini-confirmed seed falls to `_ => null` and stores the **ticker**. The surface survives only as the wire/display name at `:914` (`CanonicalName ?? proposedName ?? surface`), which feeds ContextFactor and must stay the surface (D-2 ALSO).

**Corroborated in the data — but the first draft corroborated it against the WRONG WRITER, and the correction matters to §2.** The `GeminiFallback` faucet is open: **32 rows since 2026-08-01** (21 in August, 11 in September, most recently `ECH`→ISHARES MSCI CHILE ETF on 2026-09-21), and **0 of those 32 have `name = symbol`** — all read as correct canonical names. That figure is exact and re-derived. What it measures is `POST /api/instruments`, whose `Name` is the caller's `request.Name` (`InstrumentEndpoints.cs:127`) — **not** `SelfSeedName`. The path `SelfSeedName` actually governs stamps `discovery_source='entity_resolution:<source>'` (`:1098`), and over the same window **174 of its 1,660 rows have `name = symbol`**: 173 of 1,464 `entity_resolution:gemini` (**11.8%**) and 1 of 196 `entity_resolution:openfigi`.

Both figures are correct and they say different things. Deletion does **not** re-accumulate junk NER surfaces — that is what #874 closed, and the code above is the proof. It **does** re-open a ticker-named seed, which is by design (`:1179-1183`: an `== Symbol` name is precisely what the two enrichment fill-gaps repair, and a fabricated one is what they cannot). For legs A and C, whose symbols are real tradeable tickers, that repair machinery works and the plan does not change. For leg B, whose symbols are invented, nothing will ever repair the name — which is §2's argument, and it is why this paragraph could not be left pointing at the `GeminiFallback` population.

**The hazard #874 did not close, which the PR must state plainly:** #874 fixed the *symptom*, not the *disease*. Leg A's failure was a wrong SYMBOL; the junk name was merely its visible tell. Post-#874 the same wrong mapping would store `NSRGY` → "NESTLE SA-SPONS ADR" — correctly named and pointing at the wrong company for a Nomura article, with nothing to notice. **Deleting leg A destroys the only surviving record of the NER surfaces that produced those 82 wrong mappings.** This is the exact failure mode `spec-plan-authoring.md` names ("a recommendation that would destroy the evidence it rests on"). It is the principal argument for the pre-image in §8 (the first draft pointed at §7, which is the backlog entry), and it is why the pre-image must carry `name`, not just `id` and `symbol`.

---

## 6. BENCHMARK COUPLING — sequence C after the epic, but correct the reason

**Verified:** `DX` (`37e023d8-…`) and `KC` (`202d9ca8-…`) appear in `LlmBenchmark/attach-gold/attach_gold_g2_v1.json` at exactly one JSON path each — `.articles[].owners[].baseline[].instrument_id`. Neither appears in any `accept[]`, and neither appears in `attach_gold_g1_v1.json`.

**Correction to the inherited hazard:** "a deleted `baseline` id passes SILENTLY" is true, and that is **not a defect here**. `build_a0_predictions.py` makes **zero** database calls — `reduce_baseline` reads `baseline[j]["instrument_id"]` straight from the frozen gold. The A0 arm's job is to reproduce *what production attached at label time*, a historical fact the file records. A deleted id does not corrupt it; the id string is still there and still means what it meant. **Scoring does not break.**

**The real coupling is the drift audit, and it is a blind spot — though not the one the first draft named.** `build_attach_gold.py:1441-1448` measures catalog movement as:

```sql
'retired_in_window',     count(*) ... WHERE retired_at > … 
'deactivated_in_window', count(*) ... WHERE NOT is_active AND updated_at > …
```

A **DELETE is invisible to both** — no row, no `updated_at`, no `retired_at`. So deleting DX/KC moves the catalog underneath the gold's `catalog_at` stamps while the audit record that exists to describe exactly that movement reports nothing. `eval_harness`'s 24h `ATTACH_CATALOG_DRIFT_TOLERANCE` would be silently describing a catalog that changed in a way it cannot represent.

**Recommendation: sequence leg C after the live scoring epic closes** — for the drift-audit blindness, not for a scoring failure, and reinforced by leg C carrying **322 of the 348** live attachments (92.5%), the largest cross-DB change in the plan. Do not let the epic measure attachment quality while the largest attachment population moves under it.

**The follow-up the first draft "owed" ALREADY EXISTS — do not build it twice.** `build_attach_gold.py:1440` runs `live = catalog_rows(sorted(accepted))` and reports the result at `:1473-1479` as `accepted_ids.missing` — exactly the id-set existence re-check the first draft proposed writing, and the comment at `:1232` names deletion as the case it covers (*"an accepted row retired, deactivated or deleted -> `stale_ids`, re-run live at assemble"*).

**The real gap is its POPULATION.** `:1439` builds `accepted` from `o.get("accept", [])` only, and DX and KC are **baseline-only**: one occurrence each, `.articles[68].owners[4].baseline[0]` (KC) and `.articles[68].owners[11].baseline[0]` (DX), both in owners whose `accept` is `[]`, and neither id appears anywhere in `attach_gold_g1_v1.json`. So the existing scan would never look them up. **Owed follow-up (file, do not build here):** widen that scan's id set to cover `baseline` ids as well as accepted ones — a one-population change to an existing check, not a second check beside it. The conclusion stands unchanged: the audit as it ships is blind to DX/KC being deleted.

---

## 7. The `Quarantined-ticker re-acquisition` backlog entry — it dissolves, and that is acceptable on a NARROWER ground than the first draft gave

**Find the entry by its TEXT, never by a line number.** It was `docs/BACKLOG.md:118` at this document's base `97475644` and is `:119` at `9f0f0ce8`, having moved down one row when #1097 landed above it — a one-line drift in four days, which is why the quoted text is the anchor:

> `| C | 2026-09-16 | AWAITING-DECISION | Quarantined-ticker re-acquisition is an undecided policy: Gemini cost + un-alerted 23505 |`

It is cited from four places: `SecMaster/AGENT_README.md`'s `✗ read a quarantine refusal as an impossibility` GOTCHA (`:91` at `9f0f0ce8`), `CatalogService.cs:206`, `EntityResolutionService.cs:1076`, and `DisposeGeminiFallbackFuturesRoots`'s header.

**Deleting does dissolve it, and the brief's concern is correct**: with no row to refuse, every site that reads the row becomes "yes" at once, including the news-surface self-seed that the entry — and the card GOTCHA above it — single out as *not authoritative in the way an operator config or a collector registration is*.

**The first draft argued that away on a ground that is FALSE, and the ground is WITHDRAWN.** It claimed "two of the three paths were already 'yes' … only the self-seed was ever 'no'". Neither half survives:

- **The demonstrated re-acquisitions are the NEWS SURFACE, not a collector registration** (§0 correction 2, §2): all three active rows carry `discovery_source='GeminiFallback'`, written by `POST /api/instruments`, which reads no quarantine row and no symbol at all before inserting. The operator-config and collector-registration paths were never observed acquiring anything. So the first draft credited the "already yes" to the two paths the card calls authoritative, when the only observed yes is on the one it calls least authoritative.
- **More than one site was "no".** §3's own table lists the discovery drop (`CatalogService.cs:199`) as a second refusal that flips, alongside the self-seed (`EntityResolutionService.cs:1079`). For legs A and C **two** sites flip from refuse to accept, not one.

**What survives as the case for accepting the dissolution — three grounds, each measured:**

1. **The refusal is not protecting the distinction it appears to protect.** The only acquisition path observed reaching these symbols is the news surface, and that path never consults the quarantine row. So the quarantine is already not what stands between a news surface and a catalog row; deletion changes which SITE says yes, not whether the least-authoritative path can acquire.
2. **The 23505 risk named in the entry is gone.** `idx_instruments_symbol` is partial and the row is deleted, so there is no collision left to raise.
3. **The cost risk inverts.** A successful acquisition means future mentions resolve locally at **zero** external cost, which is why the refusal costs ~192 cascades/30d rather than saving them.

**This document does NOT supersede `SecMaster/AGENT_README.md`'s `✗ read a quarantine refusal as an impossibility` GOTCHA, and nothing here may be read as superseding it.** The card says an operator config and a collector registration are authoritative in a way a news surface is not. §0 correction 2 does not contradict that — it makes it the live concern, because the only observed re-acquisitions are on the non-authoritative path. It was the first draft's *attribution* that contradicted the card, and that attribution is retracted rather than superseded: no card line needs to change for this document to be true.

**What the PR must do to the entry — do not simply delete it.** Close the row *and* replace it with the narrower question deletion leaves standing, since the closing PR is the only place the distinction survives:

- **Close** the row matched by the quoted text above (NOT by line number) in the same PR that lands the migrations, with the outcome and the measurement: `re-acquisition demonstrated 3x, NEWS-SURFACE path only (discovery_source='GeminiFallback'), 2026-08-20/09-11/09-13, INDA/IWS/UUP, all ETFs; 0 of 74 quarantined equities in 65 days` — per CLAUDE.md §WHERE_WORK_LANDS "no tombstones".
- **Open** one successor row: *"A news surface can acquire any ticker with no catalog row, and nothing distinguishes that from an operator-authoritative acquisition after the fact — measured 3x on ETFs, 0x on 74 equities, 65d."* — with `discovery_source` as the re-checkable handle. This preserves the one real question the entry contained, and it is the question the card GOTCHA already states.
- **Rewrite** `SecMaster/AGENT_README.md`'s "✗ read a quarantine refusal as an impossibility" GOTCHA in the same PR, since its BACKLOG pointer goes stale.

**THE COMMENT SET, ENUMERATED — "the four `INTENT` comments at the guard sites" was a population that does not exist, and an implementer who searched for it would have edited the WRONG FOUR.** Read across all five guard-site files at `9f0f0ce8`:

*The comments that DESCRIBE the quarantine refusal — SIX, and not one of them carries an `INTENT(D-n)` marker:*

| §3 site | Guard | The comment, at its line range | Carries the `docs/BACKLOG.md` pointer | Owed by this PR |
|---|---|---|---|---|
| 1 | `CatalogService.cs:199` | `:201-206` "Quarantined: the row was soft-deleted to retire a junk name … a deferred catalog-repair decision" | **yes, at `:206`** | **REPOINT** — the pointer goes stale; the refusal itself keeps firing |
| 4 | `EntityResolutionService.cs:1079` | `:1067-1078` "THIS REFUSAL IS NOW A POLICY, NOT A CONSTRAINT" | **yes, at `:1076`** | **REPOINT** — same |
| 5 | `FredCatalogReconciliationService.cs:125` | `:132-135` "Quarantined but lacks the FredCollector trust signal" | no | **nothing** — it already skips all 93, and the sole FRED-relevant mapping is `GSV.NE`'s, in the leg we keep |
| 6 | `InstrumentConfigurationWatcher.cs:198` | `:192-196` "Deliberately NOT filtered to IsActive … authoritative over quarantine and REACTIVATES the row it names" | no | **AMEND** — a deleted symbol has no row to reactivate, so the sentence stops holding for leg A's 82 |
| 7 | `RegistrationService.cs:338` | `:334-336` "Same inactive-instrument deception guard" | no | **nothing** — reachable only via `GSV.NE`, leg B, kept |
| 8 | `RegistrationService.cs:402` | `:404-407` "Registration against a quarantined/inactive instrument still commits the mapping" | no | **nothing** — dead code before and after (§3 site 8) |

*The four `INTENT(D-n)` markers that actually sit at those eight sites — **none** of which describes the quarantine, and **none** of which this PR may touch:* `CatalogService.cs:214` (D-13, guards site 2 at `:218`), `CatalogService.cs:286` (D-13, guards site 3 at `:289`), `RegistrationService.cs:391` (D-16, guards the class-family refusal at `:399`, immediately above site 8) and `InstrumentConfigurationWatcher.cs:185` (D-11, heads the block whose *second* paragraph is row 6 above — edit the paragraph, never the marker). Deleting the rows orphans none of these four: D-13, D-16 and D-11 are untouched by this plan.

**So the same-PR obligation is NARROWER and more exact than "update four INTENT comments", and CLAUDE.md §INTENT_FIDELITY's ATOMIC_SET is NOT what binds it** — ATOMIC_SET binds a `D-n` entry to its `INTENT(D-n)` marker, its guard and its guard test, and no D-entry governs the quarantine refusal, so there is no atomic set to keep whole here. What binds is the ordinary stale-comment discipline (`~/.claude/CLAUDE.md` §DOC, MAINTAIN — machine-local and untracked, cited the way CLAUDE.md §GIGO cites the same file): every edit reviews the nearby comments and updates or deletes the stale ones. What the PR owes:

1. **Repoint the two `docs/BACKLOG.md` citations** at `CatalogService.cs:206` and `EntityResolutionService.cs:1076` to the successor row this section opens. These are two of the entry's four citation sites; the other two are the card GOTCHA (`SecMaster/AGENT_README.md:91`) and `SecMaster/src/Data/Migrations/20260917013629_DisposeGeminiFallbackFuturesRoots.cs:29`, and all four move together.
2. **Amend row 6's paragraph** (`InstrumentConfigurationWatcher.cs:192-196`) for leg A's symbols, per §3 site 6.
3. **Leave sites 5, 7 and 8 alone** — site 7's subject is `GSV.NE` in leg B (DO-NOT-BUILD), site 8 is dead code, and site 5's skip is unchanged in outcome. Touching them is scope, not ATOMIC_SET.

**NONE of the six is orphaned by step 2, and that is measurable, not a judgement.** `is_active=false` is EXACTLY the 93 quarantined rows (the 1,308 retired rows are `is_active=TRUE` with `retired_at` set, measured 2026-09-21T18:09Z), so after step 2 the refusal still has **11** rows to fire on — leg B's 9 plus leg C's 2 — and §2 measures it firing on leg B 8 times in 7 days. Do NOT delete these comments as describing "a decision no longer being taken"; the decision is still taken, on a population that shrinks from 93 to 11.

---

## 8. ROLLBACK — and no, it is not real

**Say it plainly: this is a one-way door.** `Down()` cannot restore a deleted row's `id`, and the `id` is the only thing the 348 + 2,678 `atlas_data` references mean. Restoring a row under a fresh `id` would silently re-point nothing.

**ZFS is a disaster floor, not an undo.** Measured: `nvme-fast/timeseries` → `/opt/ai-inference/timeseries` is **one dataset holding both `atlas_secmaster` and `atlas_data`**, and that single fact is what makes the conclusion hold. The first draft then enumerated only the three snapshot classes that support its own recommendation; the full set at 2026-09-21T17:26Z is **8 frequent, 24 hourly, 7 daily (2026-09-15 → 2026-09-21), 4 weekly, 6 monthly, 358 `pre-deploy-*` (oldest 2025-11-15, newest `pre-deploy-20260921T041251Z`) and 7 one-off `pre-*` manual snapshots** — 414 in all, rotating, so re-derive rather than quote. The omitted classes are the ones a migration would actually reach for: a `pre-deploy-*` is taken immediately before each deploy, i.e. **before an EF migration runs**, which is a far better pre-image than "daily back to 2026-09-18" suggests. State it anyway, because **the one-dataset fact is unaffected**: rolling any of those 414 back to undo this migration also rolls back every news observation, matrix cell and macro row written since it. Nobody will choose that to recover 84 junk rows — and the omission ran in the direction that flattered this document's own recommendation.

**A per-row pre-image is REQUIRED.** It is what §5 makes load-bearing: deletion destroys the junk names, and those names are the only surviving record of the NER surfaces that produced the wrong mappings.

**Where it lives — in the migration file, not in the database.** `PurgeLoRAEraQuarantinedInstruments` used a *predicate* (`is_active=false AND metadata='{}'`) and recorded nothing; that is why the 455 orphaned ids it left behind can no longer be explained. Follow `DisposeGeminiFallbackFuturesRoots` instead: an `internal const string` VALUES block carrying `(id, symbol, name, asset_class, discovery_source, created_at)` for every deleted row, which is simultaneously the compare-and-swap predicate, the pre-image, the code-review artifact and the git-permanent record. It costs nothing at runtime and survives every rollback that matters. **Do not** stage a pre-image into a database table — CLAUDE.md §DATABASE forbids the `CREATE`, and a table is exactly the artifact a future ZFS rollback would lose.

---

## 9. SEQUENCING, with the load-bearing edges named

**GATING MODEL -- every gate in the table is one of two kinds, and the column names WHO or WHAT lifts it.**
**D (decision)** = a question only a human answers; no act by any agent satisfies it and it does not expire
on its own. **A (act)** = a completed, verifiable act or an elapsed window, which the implementing agent
satisfies. A step is runnable once every gate on its row is lifted. Step 0's gate is a **D that has already
been answered NO**, and step 0 is therefore deliberately **not** an edge into steps 1-4: the question those
steps wait on is a different one, stated under the table.

| Step | Work | Unblocked by | Why the edge is load-bearing |
|---|---|---|---|
| **0** | **BLOCKED — DO NOT RUN, in any mode, including a dry run.** This step prescribes the D-33 provenance replay. A recorded user decision of **2026-09-16/17** PARKED that replay **“for good (measured net harmful)”**, and the **70% control-group stay bar this row names as the thing to get past (`SentinelCollector/AGENT_README.md:168`) has already been measured and FAILED** on a completed production dry run whose population contains all 12 of these ids: **0 of 47 known-good control rows stayed — 0.0% against a 70% bar** (recorded 2026-09-17T10:01:20Z; re-derived 2026-09-21 as **0 of 46**, the cohort having lost one row to the sweep since). Failing that bar IS the D-21 signature the replay's own guard says must STOP AND REPORT. **Route this to a human; an implementing agent may not run it, and may not narrow, re-scope or re-time it to get past the block.** Decision, measurement, blockers and un-park conditions: `docs/BACKLOG.md` §PARKED EPICS, “D-33 provenance replay” | **D -- the user, reversing the park** (conditions behind this row's own pointer). No act lifts it and no agent may attempt one: not a re-measurement, not a narrowed id list, not waiting for the sweep to go off | **The premises this row gives about the PROOF are sound and stand; what is wrong is that the ACT is parked and the bar is already failed.** `ProvenanceReResolveService` (`:208-218`) proves provenance by a SecMaster by-id lookup and refuses the **whole request** with 400 “(SecMaster has no such instrument)” for any absent id — per REQUEST, before any row is read, and only on a **live** run (the check sits inside `if (!run.DryRun)` at `:100-115`). So deleting first really would permanently disable the designed clearing path for those rows, and that consequence is why the block matters rather than being a technicality. That half of the proof passes today for all 12: a by-id lookup returns 200 carrying `discoverySource='GeminiFallback'` and a `createdAt` for every one, with `created_at` spanning **2026-06-25T13:32:30Z → 2026-07-09T16:31:39Z** (verified 2026-09-21T18:46Z, all 12), every one before the `2026-07-19` cutoff at `:29`. **The “→ 2026-07-16” this row carried until now was a population mix-up:** 2026-07-16 is the newest `created_at` across all **82** leg-A rows, not across the **12** that still hold live attachments. **Two populations, which the first draft mixed:** of the **1,402** rows whose `OriginalInstrumentId` is a leg-A id, 1,385 are already cleared and **17** are still attached (8 of those to a leg-A id) — the remainder of the 1,402 is **17, not 26**. The **26** are the rows whose LIVE `instrument_id` is a leg-A id (re-confirmed 2026-09-21: 12 distinct ids, 26 rows): 8 overlap the 1,402, 11 carry no `OriginalInstrumentId` at all, 7 carry one that is not a leg-A id. So **18 of the 26 have never been through D-33's cause-A replay** — which is what makes their disposition an open question rather than a mop-up |
| **1** | Sentinel `NullWrongMappingInstrumentRefs`, **leg A ids only** | **D -- a human answering the 26-row question below the table.** NOT step 0 completing: the migration is runnable today, and what is undecided is whether running it is WANTED | Deploy + verify before any DELETE, per §4 |
| **2** | SecMaster `DeleteWrongMappingInstruments`, **leg A only**; close the `Quarantined-ticker re-acquisition` backlog row (match by its TEXT, §7 — its line number has already moved once), open its successor, rewrite the card GOTCHA, and make the THREE comment edits §7 enumerates — **not** "4 INTENT comments", which names the wrong four | **A -- step 1 deployed and verified** (§4). No decision of its own: the answer given at step 1 admitted the whole sequence | The `instrument_id` edge (§4). Same PR for the doc/comment changes — §7 shows which comments, and why ATOMIC_SET is not the rule that binds them |
| **3** | Observe 30d: `secmaster_entity_resolution_self_seed_total{result}` | **A -- step 2 merged and deployed** | Leg C must not move while the effect of leg A is being measured, or neither is attributable |
| **4** | Legs C: repeat **steps 1-2** for DX/KC. Its step-0 equivalent is parked on the same decision -- both leg-C ids sit inside the same parked replay population (verified 2026-09-21) -- so leg C is not waiting on a replay either, and "repeat step 0" must never be read back in as a prerequisite | **D -- the 26-row question below, answered again for leg C's 322 attached rows** (§4), **and A -- step 3's 30d window elapsed AND the live scoring epic closed** | §6 — drift-audit blindness plus 322 of 348 attachments |
| — | **Leg B** | n/a -- never built | **DO-NOT-BUILD** (§2) |

**THE OPEN QUESTION, stated so that one answer settles it for steps 1-4.** Step 0 is refused for good, so
leg A's **26 rows on 12 instruments** (§4) will never be re-resolved by the D-33 replay. Steps 1 and 2 are
nonetheless runnable -- nothing about the migrations themselves is blocked -- but running them detaches
those 26 rows with no replay ever having been given the chance, and after step 2 the D-33 path is closed to
them permanently (the by-id proof 400s on a deleted id). So what steps 1-4 wait on is not an act somebody
is part-way through finishing; it is an unanswered question:

> **Delete leg A anyway, accepting that those 26 rows are never cleared -- or leave leg A in place?**

**On "go", steps 1 -> 2 -> 3 run in order on their act gates alone.** The implementing agent does not wait
for step 0, does not run it, and does not re-open it. That "go" is an answer about the 26 rows; it is **not**
a reversal of the park, which only the user can give and only in the terms `docs/BACKLOG.md` §PARKED EPICS
sets out. If the park is ever separately reversed, step 0 becomes runnable and rejoins the sequence ahead of
step 1 -- that is the only route by which it ever does, and nothing in steps 1-4 opens one.

**This document deliberately does not propose a replacement clearing mechanism.** Whether the 26 rows are
cleared another way, left attached, or accepted as collateral of the leg-A deletion is part of the question
above, and inventing a substitute clearing path here would be exactly the "route around the block" the park
forbids.

**And the block is not a timing accident that waiting fixes:** the replay's run driver refuses a **dry run**
too while the re-extract sweep is on (`preflight_refusal` tests `sweep_off` before its `if not live` return,
so `--live` is not the only mode gated), and the control gate step 0 names is **structurally unevaluable for
a narrowed id list** -- its cohort is a fixed query over `DB`/`EVR`/`NMR`, none of which is among the 12 or
among the 82, and `control_gate` refuses outright when a control row is absent from the dry run. A 12-id run
cannot pass that bar; it cannot even be scored against it.


---

## 10. Acceptance criteria

Each has a measure, a pass, a **fail**, and a negative control separating "broken" from "no data yet".

**AC-1 — the self-seed refusal stops (leg A).**
Measure `sum by (result) (increase(secmaster_entity_resolution_self_seed_total[30d]))`.
Baseline: `quarantined_skip` = **192.001 at 2026-09-21T18:08Z** (unit: extrapolated counter events over 30d; population: **all 93 quarantined rows**, not leg A alone — the counter carries no symbol label, §11 item 2).

**Step 2 deletes leg A ONLY, so a correct outcome is not zero — and the first draft's pass and fail BOTH claimed the values in between.** Its pass was "`quarantined_skip` falls", its fail "`quarantined_skip` ≥ 150"; an observed **170** satisfied both, with no precedence stated, and 170 is a plausible reading because leg A is only **42 of the 50** declines Loki attributes over 7 days. Derive the residual instead: leg B is kept and leg C is not yet deleted, so the surviving share is **8 of 50 = 16%**, and `192 × 8/50 = 30.7`, i.e. **~31/30d**. The 95% Wilson interval on 8/50 at n=50 is **[0.083, 0.285]**, i.e. **[16, 55]/30d**.

**Three DISJOINT and exhaustive bands on `quarantined_skip` at 30d — this is the whole criterion, and every observation lands in exactly one:**

| Band | Verdict | Arithmetic |
|---|---|---|
| **≤ 55** | **PASS**, provided `inserted` is also within 25% of its **838.0/30d** baseline (2026-09-21T18:08Z). If `quarantined_skip` ≤ 55 but `inserted` fell, score INCONCLUSIVE — the site may be unreached for an unrelated reason | `192 × 0.285 = 54.7`, rounded UP to 55 — the Wilson upper bound on the leg-B share |
| **56 – 149** | **INCONCLUSIVE** — not a pass and not a fail. Above the residual the leg split predicts, below the floor at which the deletion plainly missed. Re-derive the leg split from Loki over the POST-deploy window before step 3 → 4 | 56 = 55 + 1; 149 = 150 − 1 |
| **≥ 150** | **FAIL** — 78% of the 192 baseline survives, so the deletion did not reach the decision site | `150 / 192 = 78.1%` |

**Negative control:** `idempotent_skip` (**26,475/30d at 2026-09-21T18:08Z**; it was 26,373 when this criterion was first written and 26,457 at 17:45Z, so re-derive it at deploy time rather than quoting this line) must stay within ±25%. If *every* series drops, resolution is down and AC-1 has **no data** rather than a pass. This control is what makes the criterion falsifiable — the series exists and is non-zero today, so it cannot pass on a dead path.

**AC-2 — re-acquisition produces correct rows, not new junk.**
`SELECT symbol, name, discovery_source FROM instruments WHERE created_at > <deploy> AND symbol = ANY(<82 symbols>)` — deliberately **no `discovery_source` filter**. Deletion unblocks the SELF-SEED (`entity_resolution:<source>`), while the re-acquisitions measured so far are the news surface (`GeminiFallback`); filtering to either one measures a writer the other's junk does not pass through (§5).
**Pass:** no new row's `name` matches a stored pre-image name (§8).
**Fail:** any new row whose `name` equals a pre-image name — the NER surface is being re-minted and #874 did not hold.
**`name = symbol` is NOT a fail on the self-seed writer.** It is that writer's designed fall-through (`:1179-1183`) and its measured base rate is **173 of 1,464 `entity_resolution:gemini` rows created 2026-08-01 → 2026-09-21 = 11.8%**. Report it as its own line and judge it against that baseline, never against zero. On the `GeminiFallback` writer the baseline IS zero (0 of 32, same window).
**Negative control:** zero new rows is **no verdict**, not a pass — the 82 symbols may simply not have recurred (0 of the 74 quarantined equities did in 65 days). Report it as `no_data`.

**AC-3 — referential state is as designed, not accidental.**
Re-run §0 correction 4's comparison.
**Pass:** orphaned distinct `instrument_id` values = **0** (the paired migration NULLed every one). **Fail:** > 0 — the migrations landed out of order or the id lists disagree.
**Second arm — the `OriginalInstrumentId` bound.** Orphaned distinct `OriginalInstrumentId` values move by leg A's footprint **on that column**, which is **79 distinct ids across 1,402 rows** (re-derived 2026-09-21T18:09Z) — *not* the **12 ids across 26 rows** that leg A occupies on `instrument_id`. Different column, different population; the first draft used the `instrument_id` count as the `OriginalInstrumentId` bound and was low by 6.6x, so an observed +79 would have read as out-of-bounds and sent someone hunting a defect that does not exist. Expect **+0** if the paired migration NULLs `OriginalInstrumentId` as specified, **+79** if it does not, and anything strictly between as partial NULLing. Baseline: **455 orphaned ids / 26,505 rows at 2026-09-21T18:07Z** (§0 correction 4).

**Negative control — and the first draft's was BROKEN, because it assigned the SUCCESS signature to "did not run".** It read: *"Unchanged on both columns means the Sentinel migration did not run."* But a fully correct sequence leaves BOTH arms unchanged: `instrument_id` orphans stay at **0** (that is the pass) and `OriginalInstrumentId` orphans stay at **455 / 26,505** (that is the `+0`). "Unchanged on both" is the pass and the no-op reading the SAME observation, so the control separated nothing — and the reader who ran it would either declare a correct run a failure and halt the sequence, or accept a migration that never ran.

**Use AC-4's post-deploy arm as the run-detector instead; it already exists and it genuinely discriminates.** In the same session, run `SELECT review_status, resolution_state, count(*) FROM sentinel.extracted_observations WHERE instrument_id = ANY(<leg A ids>) GROUP BY 1,2`:

| Observation | Sentinel NULLing | SecMaster DELETE | Which arm says so |
|---|---|---|---|
| detector returns **0 rows**, arm 1 = 0, arm 2 = +0 | ran | ran | the full, correct sequence |
| detector returns **26 rows** (16 `Approved`/`Resolved` + 10 `AutoClosed`/`NoResolution`, 2026-09-21T18:09Z), arm 1 = 0, arm 2 = +0 | did **not** run | did **not** run | **only the detector** — both arms read exactly as the success case |
| detector returns **26 rows**, arm 1 = **12**, arm 2 = **+79** | did **not** run | ran | arm 1 FAILs (out of order), and the detector confirms which half is missing |

The detector is what makes AC-3 falsifiable: arms 1 and 2 are both silent on the difference between "everything worked" and "nothing happened", and that difference is what §10's opening sentence promises to separate. *What the detector cannot contain:* rows whose `instrument_id` was NULLed by something other than this migration — nothing else writes that column for these ids, but the pre-deploy run of AC-4 is what establishes the 26 in the first place, so run it.

**AC-4 — no human was mid-decision on a row this migration unattaches.**
**The first draft's version could not fail, and is replaced.** Its query carried no id filter, so it counted the entire actionable queue — **5,958 at 2026-09-21T17:26Z**, nowhere near 348, and moving with ordinary traffic. Its pass was "unchanged across the deploy" on that live global counter; its fail was "any increase attributable to these ids" — but the migration NULLs `instrument_id` on exactly those rows and deletes the instruments, so afterwards **no row can be attributed to them at all**. Both verdicts were unreachable.

Measure the population the migration touches instead, from the frozen id list, at three times (author time, immediately pre-deploy, post-deploy):
`SELECT review_status, resolution_state, count(*) FROM sentinel.extracted_observations WHERE instrument_id = ANY(<leg ids>) GROUP BY 1,2`.
**Pass:** the pre-deploy run returns **zero `review_status='Pending'` rows** — today, at 2026-09-21T17:26Z, leg A returns 16 `Approved`/`Resolved` + 10 `AutoClosed`/`NoResolution` = 26 and leg C returns 322, **0 Pending** across all 348 — *and* the post-deploy run returns **0 rows of any kind** for those ids.
**Fail, and both arms are reachable:** any `Pending` row in the pre-deploy run means a human is mid-decision on a row about to be unattached, and the migration stops; any row surviving post-deploy means the NULLing did not cover the population.
**Negative control:** in the same session, run the global actionable count above. It is **6,109 at 2026-09-21T18:09Z** — against **5,958 at 17:26Z**, 151 higher 43 minutes later, which is itself the evidence that the first draft's version could not fail. Re-derive it at deploy time; the only property this control rests on is that it is never 0 on a live system, so a 0 there says the query shape or the connection is broken and AC-4 has **no data** rather than a pass.

---

## 11. What could not be verified (MANDATORY)

The first draft's list held six items and omitted five load-bearing claims it asserted as measured; a blocking review found them, and a second review added item 13. The bar is: a claim is either derived in this document with its unit and its population, or it is on this list.

1. **The "rc 2 on a deleted `accept` id" claim is FALSE, not merely unverified — resolved, and the first draft's item 1 is withdrawn.** `stage_drift` records a missing id as DATA (`accepted_ids.missing`, `build_attach_gold.py:1475`) and ends `return 0` at `:1491`; its only non-zero exit is a `raise SystemExit` for a missing `--window-since` argument. **There is no rc-2 path, and a deleted id moves no exit code.** This strengthens §6 rather than weakening it: the audit is silent, not loud.

2. **Per-leg attribution of the 192 `quarantined_skip`/30d.** The counter carries only `result`, no symbol, so Prometheus cannot attribute it. Loki can, but only over the 7 days queried: **42 leg A / 8 leg B / 0 leg C across 50 events** (§2). The 30-day 192 stays unattributed, and the two instruments must not be combined arithmetically — one is a counter, the other a Warning-level log line subject to retention.

3. **The confirming SOURCE behind each self-seed decline.** Neither the log line nor the `self_seed` counter carries it. §2's claim that a deleted leg-B row self-seeds a row named `USCOSPRE` therefore holds for every `ConfirmationSource` except `OpenFigi` and `Finnhub` — and if one of those two HAD confirmed it, `USCOSPRE` would be a real instrument and leg B's "invented" premise would be the thing that is wrong. Either way one of the two is false; which, is unmeasured.

4. **Whether 11.8% is a MINT rate or a RESIDUE.** The 173-of-1,464 `name = symbol` figure is a stock measured now, after the two enrichment fill-gaps have had time to repair some rows. It is therefore a **lower bound** on the rate at which the self-seed writes a ticker as a name, not the rate itself.

5. **Whether the 3 re-acquired rows are CORRECT.** `INDA`/`IWS`/`UUP` carry names that read as canonical by inspection; none was checked against OpenFIGI or Finnhub. The claim "re-acquired, correctly" rests on reading, not on an authoritative source.

6. **Why 0 of 74 quarantined equities re-acquired.** The rate is measured (0 in 65 days). Whether that is a mechanism difference or simply that no equity mention reached the path in that window is **not** measured — the two have the same observable.

7. **Whether the 82 cause-A dispositions were correct.** Inherited from the quarantining migration's per-row Finnhub/OpenFIGI corroboration and not re-run. If any of the 82 was a false positive, deletion converts a reversible mistake into an irreversible one. The §8 pre-image is the mitigation; it is not a check.

8. **The "546 rows" in the `PurgeLoRAEraQuarantinedInstruments` precedent.** Unverifiable by construction — the rows are gone and the migration used a predicate, not an enumerated list, so it recorded nothing. What can be shown is its residue (455 orphaned ids / 26,505 rows) coexisting with a working system. "No recorded breakage" is really "nothing was watching": there is no orphan metric or alert for instrument references anywhere in the repo.

9. **The FRED-API rejection behind leg B.** The live FRED API was not re-queried for the 8 invented ids. Leg B is DO-NOT-BUILD, so no recommendation rests on it — but "rejected by the live FRED API" remains inherited, and `GSV.NE` was never a FRED id at all (§0 correction 1).

10. **AC-4's replacement criterion has never been run.** Its pre-deploy numbers here are the author-time run; nothing has been deployed, so the post-deploy arm is specified, not exercised.

11. **The §8 snapshot set rotates.** 414 snapshots at 2026-09-21T17:26Z is a reading, not a constant — `pre-deploy-*` alone grows by one per deploy and is trimmed. Re-derive it before relying on it; the one-dataset fact it supports does not rotate.

12. **Nothing was executed.** No migration was run, no `dotnet ef` invoked, no deploy attempted; every database statement in this document is a proposal, not a transcript. Every SQL statement issued to derive the figures here was a `SELECT`. The guard sites were read at the cited line numbers on `9f0f0ce8` and will drift — §7's backlog entry already drifted one line between this document's two drafts, which is why it is anchored by text.

13. **AC-2's heading overclaims its measure — ACCEPTED, not fixed.** "Re-acquisition produces correct rows, not new junk" is scored against stored pre-image **names** and nothing else. §5 says plainly that leg A's disease was a wrong **SYMBOL** and that post-#874 the same wrong mapping arrives *correctly named*, so a re-acquired `NSRGY` → "NESTLE SA-SPONS ADR" attached to a Nomura surface passes AC-2 cleanly. AC-2 is **deliberately left as it stands**: catching a correctly-named wrong mapping needs an authoritative per-row symbol check, which is exactly item 7's un-rerun corroboration, and this plan has no such criterion. **Read AC-2 as what it measures — "the #874 NAME defect did not return" — never as "the mapping is right."** Whoever runs it must not report a pass as evidence that re-acquisition mapped anything correctly.
