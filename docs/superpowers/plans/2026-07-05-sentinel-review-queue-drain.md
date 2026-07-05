# Sentinel Review-Queue Drain — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Stop the `sentinel.extracted_observations` review queue (321K Pending, +~1,650/day, zero drain) from growing unbounded by (1) giving terminally-unresolvable `NoResolution` rows a non-Pending disposition, (2) recovering the ~27% of `NoResolution` rows that are real instruments the fuzzy confirm missed, and (3) restoring the auto-approve drain for high-confidence `Resolved` rows.

**Architecture:** Three staged, independently-shippable fixes against the **V2 production path** (`Extraction__UseV2Pipeline=true`, sources rss / fed-* / searxng-content / challenger-rss / rss-mirror — confirmed in `deployment/artifacts/compose.yaml.j2:1127-1140`). Stage 1 adds an age-gated sweep that flips terminal `NoResolution` rows to a new terminal `ReviewStatus`. Stage 2 adds an EXACT catalog-symbol recovery leg to `DeterministicResolver` (confirmation-gated, never fuzzy). Stage 3 propagates the real resolution confidence through `ResolverOutcome` (3a) and adds an automated downstream drainer that runs `AutoApprovePolicy` over `Pending` rows on a schedule (3b, DD-4=b) — the V2 hot path is left untouched, preserving the "downstream owns approval" cutover.

**Tech Stack:** .NET 10 / C# 14, EF Core (Npgsql / TimescaleDB), xUnit, OpenTelemetry (`SentinelMeter`), Serilog. Compile via `SentinelCollector/.devcontainer/compile.sh` (invoke with `bash` — file has no +x bit; capture full output, check real `$?`, never `| grep | tail`). Filter-run tests via `nerdctl compose exec -T sentinel-collector-dev dotnet test --filter 'Name~<Test>'`.

## Global Constraints

- **No hand-written EF migrations** (CLAUDE.md HARD_STOP). If any stage needs a schema change use `nerdctl compose exec -T sentinel-collector-dev dotnet ef migrations add {Name} --project src/Data`. Migration analysis per stage below: **all three stages are expected to need ZERO migrations** (see Stage 1 Task 1 verification gate — `review_status` / `resolution_state` are `HasConversion<string>()` varchar columns with no DB CHECK constraint, so new enum values are additive with no snapshot change; `resolution_method` / `candidate_symbols_json` already exist).
- **`extracted_observations` is NOT read by the matrix or the digest news-momentum path.** The matrix feed is `macro_observations` `:sig:` rows via the ThresholdEngine projector (SentinelCollector/AGENT_README.md DISTINCTIONS). Confirmed: no reader of `ExtractedObservations` in `ThresholdEngine/src` or `MacroSubstrate/src`. The ONLY in-repo readers of `extracted_observations` are `DigestQueryService`, `ReviewUiEndpoints`, `AdminEndpoints`, and the repositories.
- **The digest DOES read `extracted_observations`** (`DigestQueryService.GetObservationsAsync`, `src/Services/DigestQueryService.cs:86-91`) and excludes `ReviewStatus == Rejected`. **Any terminal disposition for `NoResolution` MUST NOT be `Rejected`** or those rows (many are legitimate macro/qualitative digest content — CPI, foreign GDP, official statements) vanish from the digest. This is the load-bearing constraint on Stage 1's mechanism choice.
- **INTENT_FIDELITY:** every earned-exception / scarce-resource path gets a D-entry in `SentinelCollector/AGENT_README.md` DECISIONS block (format: `.claude/skills/architecture-cards/CARD_TEMPLATE.md`), an `// INTENT(D-n):` comment at the guard site, guard code, and a guard test — as one atomic set. Stages 2 and 3 add D-entries (see per-stage blocks). Stage 1 does not (it is not an exception path — it is a terminal-state janitor).
- **Auto-approve single-source-of-truth:** `AutoApprovePolicy` is the ONE place thresholds live (`src/Services/AutoApprovePolicy.cs`). All callers (hot path, ReExtract worker, backfill endpoint, new V2 gate) MUST route through it. Do not inline threshold logic.

---

## Grounded Diagnosis (established by two supervisor-verified read-only investigations; re-grounded here against live code — do NOT re-derive)

Load-bearing code facts this plan depends on (each verified 2026-07-05):

1. **The V2 production path never calls `AutoApprovePolicy`.** `ExtractionProcessor.RunV2ProductionAsync` (`src/Workers/ExtractionProcessor.cs:1629-1803`) writes all observations with the default `ReviewStatus.Pending` and only counts them (`V2ObservationsPendingCounter`, :1775-1783). The comment at :1772-1774 ("Sentinel no longer auto-approves locally — downstream owns approval") and `V2ExtractionPipeline` class-doc (:160-167, "the v1 path's local auto-approve … is intentionally not reproduced here — that decision moves upstream to ThresholdEngine") record a **cutover intent that was never realized**: no downstream consumer approves these rows (ThresholdEngine does not read `extracted_observations` at all). This is why hot-path drain is structurally zero. **Stage 3 honors this cutover (DD-4=b): the V2 path stays Pending-only and an automated DOWNSTREAM drainer becomes the "downstream" that approves — see the D-ENTRY CONFLICT block below.**

2. **`ResolverOutcome` carries no confidence field.** `IDeterministicResolver.cs:47-55` — `(Guid? InstrumentId, string? Symbol, string ResolutionMethod, string? AtlasSectorCode)`. The resolver KNOWS the real resolution confidence internally (`LocalResolutionDto.Confidence` for hybrid, `dto.Confidence` for Gemini, the LLM `ResolutionConfidence` for a pick) but **discards it**. `V2ExtractionPipeline.BuildObservationFromV2Result` (`src/Services/V2ExtractionPipeline.cs:206`) then writes `res_conf = (float?)extraction.ResolutionConfidence` — the **LLM's self-reported** number, which is `null` for `hybrid_subject` / `gemini_fallback` rows (the LLM did not pick) and only populated (`>= ResolverCandidatePickThreshold = 0.7`, e.g. ~0.756) for `llm_candidate_pick` rows. This is the root of "most Resolved rows carry NULL res_conf; a fraction ~0.756." Even the populated 0.756 fails the `>= 0.8` auto-approve gate.

3. **Auto-reject floor is inert post-#607.** `git show 2e5b038a` — CoD confidence is now `PASS -> 0.9`, `SOFT_FAIL/null -> 0.6`, hard-fails dropped upstream. Nothing scores below 0.5, so `AutoApprovePolicy`'s `< 0.5` reject branch never fires. The 0.9 PASS rows DO satisfy the `>= 0.9` extraction leg — the auto-approve blocker is purely the resolution-confidence leg (fact 2).

4. **`NoResolution` rows are terminal by construction and cannot reach a disposition.** Set at `V2ExtractionPipeline.cs:214-217` (`InstrumentId` null -> `NoResolution`) and at the V1 CoVe/cascade sinks (`ExtractionProcessor.cs:1277`, `:1323`). They have `InstrumentId == null` -> can never auto-approve (needs non-null instrument), score `>= 0.5` -> never auto-reject -> stay `Pending` forever. ~73% are legitimately non-instrument (countries/people/agencies/macros/foreign indices); ~27% (~1,670/day) are real instruments missed by the surface-form-sensitive fuzzy confirm.

---

## OPEN DESIGN DECISIONS (need the user's call before/at implementation)

Each has a recommendation; the implementer proceeds on the recommendation unless the user overrides.

- **DD-1 (Stage 1 mechanism):** How to give `NoResolution` a terminal disposition?
  Options: (a) auto-**Reject** after N days; (b) **new terminal `ReviewStatus`** value (not Rejected, not Pending); (c) never enqueue as Pending.
  **RECOMMENDATION: (b)** — add `ReviewStatus.AutoClosed`. Rejecting (a) would strip legit macro/qualitative rows from the digest (`!= Rejected` filter). Not-enqueueing (c) is premature at write time — Stage 2 recovery must run first, and we cannot cheaply tell "legit non-instrument" from "missed instrument" at persist. A new terminal status exits the human queue (`ReviewUiEndpoints` filters `== Pending`) while the digest keeps it (`!= Rejected`). Zero migration (string column, no CHECK constraint).

- **DD-2 (Stage 1 criteria + backlog):** N (grace age) and how to treat the existing ~277K backlog.
  **RECOMMENDATION:** age-gated **sweep worker** closing `ReviewStatus == Pending AND ResolutionState == NoResolution AND ExtractedAt < now - N days`, **N = 7** (tunable via config). The grace window decouples deploy ordering: once Stage 2 ships, newly-missed-but-recoverable instruments resolve at extraction and never age into the sweep. The 277K backlog (all > 7 days old) is drained by the SAME sweep in rate-limited batches — no separate one-time backfill needed; add a manual admin trigger for on-demand acceleration. **No data deleted** — only `Pending -> AutoClosed`; digest unaffected.

- **DD-3 (Stage 2 gate):** Exactly what makes the LLM's candidate trustworthy without reintroducing wrong-ticker risk?
  **RECOMMENDATION:** resolve ONLY when the candidate's `Symbol` (uppercased) is an **EXACT** primary-symbol catalog hit via `GetInstrumentBySymbolAsync` **AND** the returned instrument's `Name` shares >= 1 content token with the candidate `Name` / `SubjectEntity` (`SubjectNameNormalizer.SharedTokenCount >= 1`). Never fuzzy, never similarity-scored. Candidate is sourced from the LLM-selected SecMaster shortlist (`candidate_symbols_json`), so this is re-linking an id the shortlist already vouched for, not trusting a free guess. Slots into `DeterministicResolver.ResolveAsync` as a new leg BEFORE Rule 3 / the `hybrid_subject`-null-id epilogue.

- **DD-4 (Stage 3 drain) — DECIDED = option (b):** drain DOWNSTREAM via an AUTOMATED/scheduled run of the existing `AutoApprovePolicy` (the `backfill-auto-approve` logic, non-dry-run), NOT via a V2 hot-path gate. The V2 path keeps writing everything `Pending` — this **PRESERVES the cutover's "downstream owns approval" intent**, so there is **NO cutover reversal and NO D-9 supersession**. A background drainer (or systemd timer) runs `AutoApprovePolicy.Evaluate` over `Pending` rows in batches, non-dry-run, moving them to Approved/Rejected. If it warrants a record, it is a NEW D-entry (automated drain gated to the same thresholds), not a supersession.

- **DD-5 (Stage 3 threshold):** Keep the `>= 0.8` res_conf gate, or lower it, or add an exact-match bypass?
  **RECOMMENDATION:** keep `>= 0.8` AND treat Stage 2 exact-match rows as res_conf = 1.0 (auto-approvable). Leave `hybrid_subject` / `llm_candidate_pick` rows in the 0.6-0.8 band as `Pending` **by design** — they are genuinely less certain and belong in human review. Do NOT lower the floor to chase drain volume; the volume lever is Stage 1 (NoResolution), not loosening approval. Leave the inert `< 0.5` auto-reject branch in place as harmless defense-in-depth; do not rely on it as a drain.

---

## D-ENTRY CONFLICT ANALYSIS (required before implementing Stages 2 and 3)

**Stage 2 vs SecMaster D-2 (`fuzzy-proposes-authoritative-confirms`):** D-2's intent is anti-hallucination — "discovery top-match = PROPOSAL only; resolves ONLY when an authoritative source grounds it in a real FIGI/ticker; unconfirmed -> null, never floored-score" (wrong-neighbour WULF->USAF corrupts catalog + matrix). **No contradiction, therefore NO supersession — a NEW SentinelCollector D-entry is the correct mechanism.** Rationale to record in the plan-of-record and the PR: Stage 2 does **exact primary-symbol catalog match + name-token confirmation**, which IS authoritative confirmation (the catalog is the authority; exact symbol existence + name overlap = confirmed), not a fuzzy proposal. It introduces **no similarity scoring**, so the wrong-neighbour failure mode D-2 guards is not reintroduced. The candidate is an LLM SELECTION from a SecMaster-provided shortlist, not a free-form guess. Because the new leg lives in `SentinelCollector/DeterministicResolver` (not SecMaster), it earns its own D-entry there (proposed **D-8**, below), and Stage 2's guard test constructs the D-2 failure mode (an exact-but-wrong-company symbol with zero name overlap) and asserts refusal — RED if the gate is ever weakened to fuzzy or the name check dropped. **Per the CONFLICT HARD_STOP rule, this was checked explicitly and found non-conflicting; the plan does NOT route around D-2 — it satisfies D-2's ethic.**

**Stage 3 vs the V2 cutover intent (`ExtractionProcessor.cs:1772-1774` + `V2ExtractionPipeline.cs:160-167`) — DD-4 = (b), NO CONFLICT:** the user chose option (b), so the V2 path KEEPS writing everything `Pending` and the drain is a DOWNSTREAM automated consumer. This **preserves** the "downstream owns approval" intent literally — the scheduled drainer IS the downstream approver — so there is **no contradiction, no cutover reversal, and NO D-9 supersession.** The two cutover comments stay as-is (not rewritten). The automated drainer is a new **scarce-resource-adjacent quality gate** (batch approval into the digest-visible Approved set), so it earns a **new** SentinelCollector D-entry **D-9 `automated-review-drain`** (a plain new entry, NOT a supersession) recording that the `AutoApprovePolicy` runs on a schedule, non-dry-run, gated to the same thresholds, over `Pending` rows only. INTENT_FIDELITY is satisfied without touching the cutover ethic.

---

## Stage Ordering & Rationale

**Order: Stage 1 (Fix 1) -> Stage 2 (Fix 2) -> Stage 3 (Fix 3).** Matches the "stop the bleeding / fix the root cause / restore the drain" framing and is safe:

- **Stage 1 first = biggest lever, lowest risk.** It attacks the 321K directly and cannot lose matrix signal (`NoResolution` rows have `InstrumentId == null` and never publish `SeriesCollectedEvent` — event predicate requires `InstrumentId.HasValue`, `ExtractionProcessor.cs:1787-1790`, so they feed neither the event stream nor the matrix regardless). The 7-day grace (DD-2) protects the ~27% recoverable rows until Stage 2 lands.
- **Stage 2 second = the real root cause.** It reduces `NoResolution` INFLOW so Stage 1's sweep closes fewer real instruments over time, and it feeds Stage 3 (exact-match rows are the cleanest auto-approve candidates).
- **Stage 3 last = restores the `Resolved`-row drain** once res_conf is trustworthy and exact-match rows exist to approve — via a downstream automated drainer (DD-4=b), leaving the V2 hot path untouched.

Each stage is independently shippable and independently valuable. Stages do not share code changes (Stage 2 touches the resolver; Stage 3a touches `ResolverOutcome` + the resolver's confidence returns + `BuildObservationFromV2Result`; Stage 3b adds a standalone drainer that reuses `AutoApprovePolicy`. Stage 3 depends on Stage 2's method existing but not vice-versa; sequence them).

---

## Stage 1 — Terminal disposition for `NoResolution` (stop unbounded growth)

**Goal:** `NoResolution` rows that have been `Pending` longer than the grace age flip to a new terminal `ReviewStatus.AutoClosed`, exiting the human review queue while remaining visible to the digest. A batched, rate-limited sweep worker drains both the go-forward trickle and the ~277K backlog. No data deleted; no matrix/digest regression.

**Files:**
- Modify: `src/Entities/ReviewStatus.cs` (add `AutoClosed`)
- Modify: `src/Services/AutoApprovePolicy.cs` (add close decision + note constants)
- Create: `src/Workers/NoResolutionSweepWorker.cs`
- Modify: `src/Configuration/ExtractionOptions.cs` (sweep config)
- Modify: `src/Endpoints/AdminEndpoints.cs` (manual trigger; reuse batch logic)
- Modify: `src/Services/DigestQueryService.cs` — **verify only** (confirm `!= Rejected` already keeps `AutoClosed`; no change expected)
- Modify: `src/Endpoints/ReviewUiEndpoints.cs` — **verify only** (confirm `== Pending` already excludes `AutoClosed`; no change expected)
- Modify: `src/Telemetry/SentinelMeter.cs` (sweep counter)
- Modify: `SentinelCollector/AGENT_README.md` (note the new terminal status in DATA MODEL; no D-entry — not an exception path)
- Test: `tests/SentinelCollector.UnitTests/Services/AutoApprovePolicyTests.cs`
- Test: `tests/SentinelCollector.UnitTests/Workers/NoResolutionSweepWorkerTests.cs`
- Test: `tests/SentinelCollector.UnitTests/Services/DigestQueryServiceAutoClosedTests.cs`

**Interfaces:**
- Produces: `ReviewStatus.AutoClosed` (enum value); `AutoApprovePolicy.EvaluateClose(ExtractedObservation, DateTimeOffset nowUtc, int graceDays) -> bool` (true = eligible to close); `AutoApprovePolicy.AutoClosedNote` const.

### Task 1.1: Verify persistence + consumers (gate — no code yet)

- [ ] **Step 1: Confirm no DB CHECK constraint on `review_status`** (SELECT-only, per DATABASE DEBUG_ONLY rule):

Run: `sudo nerdctl exec timescaledb psql -U ai_inference -d atlas_data -c "\d+ sentinel.extracted_observations" | grep -i "check\|review_status"`
Expected: `review_status` is a text/varchar column, **no** `CHECK` constraint listing enum values. If a CHECK constraint IS present, a migration is required — STOP and use `dotnet ef migrations add AddAutoClosedReviewStatus` (never hand-write) and note it in the plan-of-record.

- [ ] **Step 2: Confirm digest keeps and queue excludes the new status** by reading the two consumers:
  - `src/Services/DigestQueryService.cs:90` is `o.ReviewStatus != ReviewStatus.Rejected` (keeps AutoClosed). No change needed.
  - `src/Endpoints/ReviewUiEndpoints.cs:31` is `o.ReviewStatus == ReviewStatus.Pending` (excludes AutoClosed). No change needed.

Expected: both confirmed. Record findings inline in the PR description.

### Task 1.2: Add the `AutoClosed` enum value (TDD)

- [ ] **Step 1: Write the failing test** in `AutoApprovePolicyTests.cs`:

```csharp
[Fact]
public void EvaluateClose_returns_true_for_aged_noresolution_pending_row()
{
    var obs = TestObs(reviewStatus: ReviewStatus.Pending,
                      resolutionState: ResolutionState.NoResolution,
                      extractedAt: DateTimeOffset.UtcNow.AddDays(-8));
    Assert.True(AutoApprovePolicy.EvaluateClose(obs, DateTimeOffset.UtcNow, graceDays: 7));
}

[Fact]
public void EvaluateClose_false_for_fresh_or_resolved_or_nonpending()
{
    var fresh = TestObs(ReviewStatus.Pending, ResolutionState.NoResolution, DateTimeOffset.UtcNow.AddDays(-1));
    var resolved = TestObs(ReviewStatus.Pending, ResolutionState.Resolved, DateTimeOffset.UtcNow.AddDays(-30));
    var approved = TestObs(ReviewStatus.Approved, ResolutionState.NoResolution, DateTimeOffset.UtcNow.AddDays(-30));
    var now = DateTimeOffset.UtcNow;
    Assert.False(AutoApprovePolicy.EvaluateClose(fresh, now, 7));   // grace not elapsed
    Assert.False(AutoApprovePolicy.EvaluateClose(resolved, now, 7)); // not NoResolution
    Assert.False(AutoApprovePolicy.EvaluateClose(approved, now, 7)); // already terminal
}
```

- [ ] **Step 2: Run — expect FAIL** (`EvaluateClose` / `AutoClosed` not defined):
Run: `nerdctl compose exec -T sentinel-collector-dev dotnet test --filter 'Name~EvaluateClose'`
Expected: compile error / FAIL.

- [ ] **Step 3: Add `AutoClosed` to `ReviewStatus.cs`** (append — additive, keeps existing ordinals):

```csharp
    /// <summary>Skipped - reviewer chose not to evaluate this observation</summary>
    Skipped,

    /// <summary>Terminally closed by the NoResolution sweep — no instrument to
    /// resolve to (SecMaster tried everything). NOT Rejected: the row stays
    /// valid digest content; it only leaves the human review queue. Distinct
    /// from Skipped (human action) — AutoClosed is system-set + audited.</summary>
    AutoClosed
```

- [ ] **Step 4: Add the close decision + note to `AutoApprovePolicy.cs`:**

```csharp
    public const string AutoClosedNote = "Auto-closed: NoResolution and no instrument to resolve to (aged out of review)";

    /// <summary>
    /// True when a Pending, NoResolution row has aged past the grace window and
    /// should be terminally closed. Kept in this policy so the sweep worker and
    /// the manual admin trigger share one definition. NoResolution rows have
    /// InstrumentId == null by construction — they can never auto-approve and
    /// never auto-reject, so without this they stay Pending forever.
    /// </summary>
    public static bool EvaluateClose(ExtractedObservation o, DateTimeOffset nowUtc, int graceDays) =>
        o.ReviewStatus == ReviewStatus.Pending
        && o.ResolutionState == ResolutionState.NoResolution
        && o.ExtractedAt < nowUtc.AddDays(-graceDays);
```

- [ ] **Step 5: Run — expect PASS.** Run the same filter. Expected: PASS.

- [ ] **Step 6: Commit.** `git add src/Entities/ReviewStatus.cs src/Services/AutoApprovePolicy.cs tests/SentinelCollector.UnitTests/Services/AutoApprovePolicyTests.cs && git commit -m "feat(sentinel): add AutoClosed terminal review status + close policy (Fix 1)"`

### Task 1.3: Digest-preservation business test (RED-on-unfixed)

- [ ] **Step 1: Write the failing test** in `DigestQueryServiceAutoClosedTests.cs` — an `AutoClosed` row within the window is still returned by the digest, a `Rejected` row is not:

```csharp
[Fact]
public async Task GetObservations_keeps_AutoClosed_but_excludes_Rejected()
{
    await using var db = NewInMemoryContext();
    var now = DateTimeOffset.UtcNow;
    db.ExtractedObservations.AddRange(
        SeedObs(id: 1, status: ReviewStatus.AutoClosed, extractedAt: now.AddHours(-1), confidence: 0.9f),
        SeedObs(id: 2, status: ReviewStatus.Rejected,   extractedAt: now.AddHours(-1), confidence: 0.9f));
    await db.SaveChangesAsync();
    var rows = await new DigestQueryService(db).GetObservationsAsync(now.AddDays(-1), now.AddDays(1), 0.5f);
    Assert.Contains(rows, r => r.Id == 1);      // AutoClosed KEPT (digest content)
    Assert.DoesNotContain(rows, r => r.Id == 2); // Rejected excluded
}
```

- [ ] **Step 2: Run — expect PASS immediately** (the `!= Rejected` filter already yields this behaviour; the test LOCKS it so a future "close via Reject" refactor REDs). Run: `dotnet test --filter 'Name~GetObservations_keeps_AutoClosed'`. Expected: PASS.
- [ ] **Step 3: Commit.** `git add tests/.../DigestQueryServiceAutoClosedTests.cs && git commit -m "test(sentinel): lock digest keeps AutoClosed, excludes Rejected (Fix 1)"`

### Task 1.4: Sweep worker (TDD)

- [ ] **Step 1: Add config to `ExtractionOptions.cs`:**

```csharp
    /// <summary>NoResolution sweep: rows Pending longer than this are AutoClosed. 0 disables the sweep.</summary>
    public int NoResolutionCloseGraceDays { get; set; } = 7;
    /// <summary>Sweep batch size (bounded to keep the change-tracker + a single UPDATE small).</summary>
    public int NoResolutionSweepBatchSize { get; set; } = 500;
    /// <summary>Interval between sweep passes.</summary>
    public TimeSpan NoResolutionSweepInterval { get; set; } = TimeSpan.FromMinutes(10);
```

- [ ] **Step 2: Write the failing worker test** in `NoResolutionSweepWorkerTests.cs` — one batch flips aged NoResolution Pending rows to AutoClosed, leaves fresh / Resolved / already-terminal rows untouched, and increments the counter:

```csharp
[Fact]
public async Task Sweep_closes_only_aged_noresolution_pending_rows()
{
    await using var db = NewInMemoryContext();
    var now = DateTimeOffset.UtcNow;
    db.ExtractedObservations.AddRange(
        Seed(1, ReviewStatus.Pending, ResolutionState.NoResolution, now.AddDays(-10)), // close
        Seed(2, ReviewStatus.Pending, ResolutionState.NoResolution, now.AddDays(-1)),  // fresh -> keep
        Seed(3, ReviewStatus.Pending, ResolutionState.Resolved,     now.AddDays(-10)), // resolved -> keep
        Seed(4, ReviewStatus.Approved, ResolutionState.NoResolution, now.AddDays(-10))); // terminal -> keep
    await db.SaveChangesAsync();

    var closed = await NoResolutionSweepWorker.RunOneBatchAsync(db, graceDays: 7, batchSize: 500, now, CancellationToken.None);

    Assert.Equal(1, closed);
    Assert.Equal(ReviewStatus.AutoClosed, (await db.ExtractedObservations.FindAsync(1L))!.ReviewStatus);
    Assert.Equal(ReviewStatus.Pending,    (await db.ExtractedObservations.FindAsync(2L))!.ReviewStatus);
    Assert.Equal(ReviewStatus.Pending,    (await db.ExtractedObservations.FindAsync(3L))!.ReviewStatus);
}
```

- [ ] **Step 3: Run — expect FAIL** (worker not defined). Run: `dotnet test --filter 'Name~Sweep_closes_only_aged'`. Expected: FAIL.

- [ ] **Step 4: Implement `NoResolutionSweepWorker`** — a `BackgroundService` whose testable core is a static `RunOneBatchAsync` (cursor-by-Id, `SetReviewStatus(AutoClosed, AutoClosedNote)` per row via `EvaluateClose`, `SaveChangesAsync`, `SentinelMeter.NoResolutionSweepClosedCounter.Add(n)`, `LogInformation` with the batch count). The hosted loop: skip when `NoResolutionCloseGraceDays == 0`; else loop `RunOneBatchAsync` until a batch returns 0, sleep `NoResolutionSweepInterval`, repeat. Register in `Program.cs` via `AddHostedService`. Follow LOG_RULES (Info on batch summary, Warn on transient DB error + continue). Use bounded-cardinality tags only.

- [ ] **Step 5: Run — expect PASS.** Run the filter. Expected: PASS.
- [ ] **Step 6: Full compile + tests.** Run: `bash SentinelCollector/.devcontainer/compile.sh` (capture full output, check `$?` == 0, 0 warnings/errors, all tests pass).
- [ ] **Step 7: Commit.** `git add src/Workers/NoResolutionSweepWorker.cs src/Configuration/ExtractionOptions.cs src/Telemetry/SentinelMeter.cs src/Program.cs tests/.../NoResolutionSweepWorkerTests.cs && git commit -m "feat(sentinel): NoResolution age-gated sweep worker (Fix 1)"`

### Task 1.5: Manual admin trigger + card note

- [ ] **Step 1: Add `POST /admin/review/close-noresolution`** to `AdminEndpoints.cs` mirroring the `backfill-auto-approve` shape (dry-run default true, batch bounds 1-5000, `mode` tag) but calling `AutoApprovePolicy.EvaluateClose` + `SetReviewStatus(AutoClosed, AutoClosedNote)`. This drains the 277K backlog on demand faster than the 10-min loop. Reuse the cursor-by-Id + `ChangeTracker.Clear()` dry-run pattern verbatim.
- [ ] **Step 2: Note the new terminal status** in `SentinelCollector/AGENT_README.md` DATA MODEL block (one line: "review_status AutoClosed = terminal NoResolution disposition; digest keeps (!= Rejected), queue excludes (== Pending); set by NoResolutionSweepWorker"). No D-entry (janitor, not an exception path).
- [ ] **Step 3: Compile + commit.** `bash SentinelCollector/.devcontainer/compile.sh` then `git add src/Endpoints/AdminEndpoints.cs SentinelCollector/AGENT_README.md && git commit -m "feat(sentinel): manual close-noresolution admin trigger + card note (Fix 1)"`

**Stage 1 D-entry / INTENT_FIDELITY:** none required — a terminal-state janitor is not an earned-exception / scarce-resource path. (The card DATA-MODEL note documents the new status.)

**Stage 1 guard test:** N/A (no guard). Business-outcome tests: Task 1.3 (digest keeps AutoClosed) + Task 1.4 (sweep closes only eligible rows).

**Stage 1 migration:** none expected (Task 1.1 gate confirms; string column + no CHECK constraint = additive enum value, no snapshot change). If Task 1.1 finds a CHECK constraint -> `dotnet ef migrations add AddAutoClosedReviewStatus`.

**Stage 1 out-of-scope:** re-resolving backlog rows (that is re-extraction, a separate worker); changing the digest confidence floor; touching `macro_observations` / the matrix.

---

## Stage 2 — Recover the ~27% missed real instruments (exact catalog-symbol match)

**Goal:** When the fuzzy confirm fails on a colloquial name ("Nike", "Tesla") but the LLM selected a shortlist candidate whose `Symbol` is an EXACT primary symbol in the SecMaster catalog, resolve it — confirmation-gated (exact symbol + name-token overlap), NEVER fuzzy. Reduces `NoResolution` inflow and feeds Stage 3.

**Files:**
- Modify: `src/Services/DeterministicResolver.cs` (new exact-match leg + guard + `INTENT(D-8)` comment)
- Modify: `src/Services/IDeterministicResolver.cs` — only if a new outcome method value is documented; `ResolutionMethod` is free-text so no type change
- Modify: `src/Telemetry/SentinelMeter.cs` (outcome tag values for the new leg)
- Modify: `SentinelCollector/AGENT_README.md` (add **D-8**)
- Test: `tests/SentinelCollector.UnitTests/Services/DeterministicResolverTests.cs`

**Interfaces:**
- Consumes: `ISecMasterClient.GetInstrumentBySymbolAsync(string symbol) -> InstrumentLookupDto?` (exact primary-symbol lookup; returns `Id`, `Symbol`, `Name`); `SubjectNameNormalizer.SharedTokenCount(string?, string?) -> int`; `CandidateRef(Index, InstrumentId, Symbol, Name, AssetClass, Aliases)` from `input.Candidates`.
- Produces: `ResolverOutcome` with `ResolutionMethod = "llm_candidate_exact"` on recovery (Stage 3 stamps its confidence).

### Task 2.1: Guard test FIRST — exact-but-wrong-company is refused (the D-2 failure mode)

- [ ] **Step 1: Write the failing guard test** in `DeterministicResolverTests.cs` (per intent-review GUARD_TEST_CONTRACT — construct the violation, assert refusal through the real flow, RED if the guard is deleted):

```csharp
[Fact]
public async Task exact_match_leg_refuses_symbol_collision_with_zero_name_overlap()
{
    // LLM picked a shortlist candidate whose Symbol happens to exact-match a real
    // catalog instrument of a DIFFERENT company (the wrong-ticker risk D-2 guards).
    var secmaster = new FakeSecMaster()
        .WithExactSymbol("DELT", id: Guid.NewGuid(), name: "Delta Apparel Holdings"); // NOT the airline
    var resolver = NewResolver(secmaster);
    var input = InputWithPickedCandidate(
        subjectEntity: "Delta Air Lines", candidateSymbol: "DELT", candidateName: "Delta Air Lines",
        resolutionConfidence: 0.4 /* below pick threshold so Rule 1 is skipped */);

    var outcome = await resolver.ResolveAsync(input);

    Assert.Null(outcome.InstrumentId);                 // refused: zero name-token overlap
    Assert.NotEqual("llm_candidate_exact", outcome.ResolutionMethod);
}

[Fact]
public async Task exact_match_leg_recovers_colloquial_name_on_exact_symbol_and_name_overlap()
{
    var nkeId = Guid.NewGuid();
    var secmaster = new FakeSecMaster()
        .WithExactSymbol("NKE", id: nkeId, name: "Nike Inc")
        .WithFuzzyMiss("Nike"); // fuzzy confirm fails on the bare colloquial name
    var resolver = NewResolver(secmaster);
    var input = InputWithPickedCandidate(
        subjectEntity: "Nike", candidateSymbol: "NKE", candidateName: "Nike Inc",
        resolutionConfidence: 0.4);

    var outcome = await resolver.ResolveAsync(input);

    Assert.Equal(nkeId, outcome.InstrumentId);         // recovered
    Assert.Equal("llm_candidate_exact", outcome.ResolutionMethod);
}
```

- [ ] **Step 2: Run — expect FAIL** on the recovery case (leg not implemented; `NKE` stays NoResolution). Run: `dotnet test --filter 'Name~exact_match_leg'`. Expected: recovery test FAIL, refusal test may PASS-by-accident (no leg = no resolution) — that is fine; it will guard once the leg exists.

### Task 2.2: Implement the exact-match leg + guard

- [ ] **Step 1: Add the leg to `DeterministicResolver.ResolveAsync`**, slotted AFTER the Gemini leg (Rule 2.5) and BEFORE the Rule 2 `hybrid_subject`-null epilogue (`DeterministicResolver.cs:118`). Iterate the LLM-selected candidate(s) from `input.Candidates` (prefer the `SelectedCandidateIndex` pick; fall back to any candidate the LLM surfaced) whose `InstrumentId` is null; for each, exact-lookup and confirm:

```csharp
        // INTENT(D-8): recover real instruments the surface-form-sensitive fuzzy
        // confirm misses on colloquial names ("Nike","Tesla"). The LLM SELECTED
        // this candidate from a SecMaster shortlist; when its Symbol is an EXACT
        // primary-symbol catalog hit AND the catalog Name shares a content token
        // with the candidate/subject, that is AUTHORITATIVE confirmation (not a
        // fuzzy proposal — no similarity score), so it does not reintroduce the
        // wrong-ticker risk SecMaster D-2 guards. NEVER fuzzy; exact + name-overlap only.
        var exact = await TryExactCandidateMatchAsync(input, activity, ct);
        if (exact is not null)
            return exact;
```

with the helper:

```csharp
    private async Task<ResolverOutcome?> TryExactCandidateMatchAsync(
        ResolverInput input, Activity? activity, CancellationToken ct)
    {
        if (input.Candidates is null || input.Candidates.Count == 0) return null;

        // The LLM's pick first (strongest signal), then any other id-less shortlist row.
        var ordered = OrderPickFirst(input.Candidates, input.SelectedCandidateIndex);
        foreach (var c in ordered)
        {
            if (c.InstrumentId.HasValue) continue;                 // already had an id (Rule 1 path)
            if (string.IsNullOrWhiteSpace(c.Symbol)) continue;
            var symbol = c.Symbol.Trim().ToUpperInvariant();
            var hit = await _secMasterClient.GetInstrumentBySymbolAsync(symbol, ct); // EXACT primary-symbol lookup
            if (hit is null) continue;                              // symbol not a real primary symbol -> refuse
            if (!string.Equals(hit.Symbol, symbol, StringComparison.OrdinalIgnoreCase)) continue; // defense: exact only

            // Name-token confirmation: the catalog row must be the SAME entity the
            // candidate/subject names — blocks a symbol collision with a different company.
            var overlap = Math.Max(
                SubjectNameNormalizer.SharedTokenCount(c.Name, hit.Name),
                SubjectNameNormalizer.SharedTokenCount(input.SubjectEntity, hit.Name));
            if (overlap < 1)
            {
                activity?.SetTag("exact_match.rejected_name_overlap", true);
                SentinelMeter.SecMasterResolutionCounter.Add(1,
                    new KeyValuePair<string, object?>("status", "exact_rejected_name"),
                    new KeyValuePair<string, object?>("resolution_state", "no_resolution"),
                    new KeyValuePair<string, object?>("method", "llm_candidate_exact"));
                continue;
            }

            activity?.SetTag("resolution_method", "llm_candidate_exact");
            activity?.SetTag("exact_match.symbol", hit.Symbol);
            SentinelMeter.SecMasterResolutionCounter.Add(1,
                new KeyValuePair<string, object?>("status", "resolved"),
                new KeyValuePair<string, object?>("resolution_state", "resolved"),
                new KeyValuePair<string, object?>("method", "llm_candidate_exact"));
            return new ResolverOutcome(hit.Id, hit.Symbol, "llm_candidate_exact",
                LiftSector(hit.Symbol, input.SubjectEntity, input, activity));
        }
        return null;
    }
```

(`OrderPickFirst` = a small helper putting `Candidates[SelectedCandidateIndex]` first, then the rest in order.)

- [ ] **Step 2: Run — expect BOTH tests PASS** (recovery resolves; collision refused). Run: `dotnet test --filter 'Name~exact_match_leg'`. Expected: PASS.
- [ ] **Step 3: Run the FULL `DeterministicResolverTests`** to confirm no existing behaviour regressed (Rule 1/2/2.5 ordering intact — the new leg only fires on id-less candidates the earlier rules did not resolve). Run: `dotnet test --filter 'FullyQualifiedName~DeterministicResolverTests'`. Expected: all PASS.
- [ ] **Step 4: Commit.** `git add src/Services/DeterministicResolver.cs src/Telemetry/SentinelMeter.cs tests/.../DeterministicResolverTests.cs && git commit -m "feat(sentinel): exact catalog-symbol recovery leg for LLM-selected candidates (Fix 2, D-8 guard)"`

### Task 2.3: Add D-8 to the card (atomic with the guard)

- [ ] **Step 1: Add D-8 to `SentinelCollector/AGENT_README.md` DECISIONS block:**

```
  D-8 exact-candidate-recovery: INTENT recover REAL instruments the surface-form-sensitive fuzzy confirm misses on colloquial/bare names ("Nike"/"Tesla" fall below the ~0.85 fuzzy threshold and are filtered, discarding the LLM's already-correct EXACT-catalog ticker) — the ~27% of NoResolution that are genuine misses / PRECOND the candidate came from the LLM-SELECTED SecMaster shortlist (candidate_symbols_json) AND its Symbol is an EXACT primary-symbol catalog hit (GetInstrumentBySymbolAsync, verbatim symbol equality — NEVER fuzzy, no similarity score) AND the catalog Name shares >=1 content token with the candidate/subject (SubjectNameNormalizer.SharedTokenCount>=1, blocks symbol-collision with a different company — the wrong-ticker risk). Does NOT contradict SecMaster D-2 (fuzzy-proposes-authoritative-confirms): exact symbol existence + name overlap IS authoritative confirmation, not a fuzzy proposal, so the wrong-neighbour failure mode is not reintroduced — new D-entry, NOT a D-2 supersession / GUARD DeterministicResolver.TryExactCandidateMatchAsync @ src/Services/DeterministicResolver.cs:<line> / TEST DeterministicResolverTests.exact_match_leg_refuses_symbol_collision_with_zero_name_overlap
```

- [ ] **Step 2: Update the guard-site `// INTENT(D-8):` comment** to reference the final line number, and confirm the `@ file:line` in D-8 matches. Compile.
- [ ] **Step 3: Commit.** `git add SentinelCollector/AGENT_README.md src/Services/DeterministicResolver.cs && git commit -m "docs(sentinel): D-8 exact-candidate recovery (Fix 2) — atomic w/ guard+test"`

**Stage 2 D-entry / INTENT_FIDELITY:** **D-8** (new earned exception; NOT a SecMaster D-2 supersession — see CONFLICT block). Atomic set: D-entry + `INTENT(D-8)` comment + guard code + guard test all in this stage.

**Stage 2 guard test:** `exact_match_leg_refuses_symbol_collision_with_zero_name_overlap` — constructs the D-2 wrong-ticker failure mode and asserts refusal through the real `ResolveAsync` flow; REDs if the gate is weakened to fuzzy or the name-overlap check is dropped.

**Stage 2 business-outcome test:** `exact_match_leg_recovers_colloquial_name_on_exact_symbol_and_name_overlap` — asserts a real missed instrument now resolves (RED on the unfixed code).

**Stage 2 migration:** none (`resolution_method` is free-text varchar; new value `"llm_candidate_exact"` needs no schema change).

**Stage 2 out-of-scope:** OpenFIGI / Finnhub upstream discovery (async worker owns that); the async `ResolutionWorker` retry path; changing the fuzzy threshold in SecMaster (that would violate D-2 and the RAG-over-gates memory); registering new instruments (this leg only re-links existing catalog rows).

---

## Stage 3 — Restore the drain (real res_conf propagation + automated downstream drainer) [DD-4 = option (b)]

**Goal:** Two parts. **(3a)** Stop discarding the real resolution confidence — carry it on `ResolverOutcome` and write it to `res_conf` (still required so the drain has a trustworthy value to gate on). **(3b)** Add an AUTOMATED/scheduled drainer that runs `AutoApprovePolicy.Evaluate` (the SAME single-source policy the backfill endpoint uses) over `Pending` rows in batches, **non-dry-run**, moving high-confidence `Resolved` rows to `Approved` (and any `< 0.5` extraction rows to `Rejected`). **The V2 hot path is UNCHANGED — it keeps writing everything `Pending`; the drainer IS the "downstream" the cutover intended.** No cutover reversal, no D-9 supersession.

**Files:**
- Modify: `src/Services/IDeterministicResolver.cs` (`ResolverOutcome` gains `ResolutionConfidence`)
- Modify: `src/Services/DeterministicResolver.cs` (populate confidence at each resolving return; exact-match = 1.0)
- Modify: `src/Services/V2ExtractionPipeline.cs` (`BuildObservationFromV2Result` writes `outcome.ResolutionConfidence`)
- Create: `src/Workers/AutoApproveDrainWorker.cs` (scheduled batch drainer; reuses `AutoApprovePolicy`)
- Modify: `src/Configuration/ExtractionOptions.cs` (drain enable flag + batch/interval config)
- Modify: `src/Telemetry/SentinelMeter.cs` (drain counter; reuse `AutoApproveDecisionCounter` where sensible)
- Modify: `SentinelCollector/AGENT_README.md` (add **D-9 `automated-review-drain`** — NEW entry, NOT a supersession)
- **NOT modified:** `src/Workers/ExtractionProcessor.cs` `RunV2ProductionAsync` and the two cutover comments stay as-is (DD-4=b preserves the cutover).
- Test: `tests/SentinelCollector.UnitTests/Services/V2ExtractionPipelineTests.cs` (res_conf source)
- Test: `tests/SentinelCollector.UnitTests/Workers/AutoApproveDrainWorkerTests.cs` (drain approves high-conf Resolved)

**Interfaces:**
- Consumes: `AutoApprovePolicy.Evaluate(extractionConfidence, resolutionState, resolutionConfidence, instrumentId) -> Decision`; `ResolverOutcome` (now with `ResolutionConfidence`).
- Produces: `ResolverOutcome(Guid? InstrumentId, string? Symbol, string ResolutionMethod, string? AtlasSectorCode = null, float? ResolutionConfidence = null)` (trailing-optional so existing 3/4-arg sites compile); persisted `res_conf` sourced from the resolver, not the LLM self-report; `AutoApproveDrainWorker.RunOneBatchAsync(SentinelDbContext, ExtractionOptions, CancellationToken) -> (int approved, int rejected, int skipped)`.

### Task 3.1: Carry the real confidence on `ResolverOutcome` (TDD)

- [ ] **Step 1: Write the failing test** in `V2ExtractionPipelineTests.cs` — a hybrid/exact Resolved row's persisted `res_conf` comes from the OUTCOME, not the LLM's `extraction.ResolutionConfidence`:

```csharp
[Fact]
public void BuildObservation_uses_outcome_res_conf_not_llm_self_report()
{
    var extraction = ExtractionWith(resolutionConfidence: null,   // LLM did NOT self-report
                                    extractionConfidence: 0.9);
    var outcome = new ResolverOutcome(Guid.NewGuid(), "NKE", "llm_candidate_exact",
                                      AtlasSectorCode: null, ResolutionConfidence: 1.0f);
    var obs = V2ExtractionPipeline.BuildObservationFromV2Result(
        1, "rss", extraction, Array.Empty<CandidateRef>(), outcome, NullLogger.Instance);
    Assert.Equal(1.0f, obs.ResolutionConfidence);          // from outcome, not null
    Assert.Equal(ResolutionState.Resolved, obs.ResolutionState);
}
```

- [ ] **Step 2: Run — expect FAIL** (`ResolverOutcome` has no `ResolutionConfidence`; build still writes `(float?)extraction.ResolutionConfidence == null`). Run: `dotnet test --filter 'Name~uses_outcome_res_conf'`. Expected: compile error / FAIL.

- [ ] **Step 3: Add `ResolutionConfidence` to `ResolverOutcome`** (trailing-optional, `IDeterministicResolver.cs:47-55`):

```csharp
public sealed record ResolverOutcome(
    Guid? InstrumentId,
    string? Symbol,
    string ResolutionMethod,
    string? AtlasSectorCode = null,
    // Real resolution confidence from the resolving leg (SecMaster hybrid
    // similarity, Gemini confidence, or 1.0 for an exact catalog match). Distinct
    // from the LLM's self-reported extraction ResolutionConfidence — this is what
    // the auto-approve gate must see. Null only on the NoResolution "none" outcome.
    float? ResolutionConfidence = null);
```

- [ ] **Step 4: Populate it at each RESOLVING return in `DeterministicResolver.cs`:**
  - Rule 1 pick with id (`:59`): `ResolutionConfidence: (float?)input.ResolutionConfidence` (the LLM pick confidence, already `>= 0.7`).
  - Rule 1 semantic-search materialize (`:73`): `resolved?.Confidence ?? (float?)input.ResolutionConfidence`.
  - Rule 2 `hybrid_subject` resolved (`:91`): `hybridResolved.Confidence` (the `LocalResolutionDto.Confidence`, SecMaster's `.95/.85/.75` tiers).
  - Rule 2.5 Gemini `secmaster_match` / `registered_new` (`:433`, `:465`): `(float?)dto.Confidence`.
  - **Stage 2 exact match** (`TryExactCandidateMatchAsync`): `ResolutionConfidence: 1.0f` (exact catalog identity = maximal confidence).
  - Rule 2 null epilogue (`:127`) and Rule 3 `none` (`:138`): leave null (NoResolution).

- [ ] **Step 5: Update `BuildObservationFromV2Result`** (`V2ExtractionPipeline.cs:206`): replace `var legacyConfidence = (float?)extraction.ResolutionConfidence;` with `var resConfidence = outcome.ResolutionConfidence;` and pass `resConfidence` to `UpdateResolution`. Update the nearby comment (WHY: res_conf must reflect the resolving leg, not the LLM self-report — the auto-approve gate and the `>= 0.8f` event-publish predicate at `ExtractionProcessor.cs:1788` both read it).

- [ ] **Step 6: Run — expect PASS.** Run the filter, then run `FullyQualifiedName~DeterministicResolverTests` and `FullyQualifiedName~V2ExtractionPipelineTests` to confirm no regression. Expected: all PASS.
- [ ] **Step 7: Commit.** `git add src/Services/IDeterministicResolver.cs src/Services/DeterministicResolver.cs src/Services/V2ExtractionPipeline.cs tests/.../V2ExtractionPipelineTests.cs && git commit -m "fix(sentinel): carry real resolution confidence on ResolverOutcome -> res_conf (Fix 3a)"`

### Task 3.2: Automated downstream drainer (TDD; DD-4 = b)

- [ ] **Step 1: Add config to `ExtractionOptions.cs`:**

```csharp
    /// <summary>Automated review-queue drainer: runs AutoApprovePolicy over Pending rows non-dry-run. Off disables it.</summary>
    public bool AutoApproveDrainEnabled { get; set; } = false;
    /// <summary>Rows per drain batch (bounded — one buffered UPDATE per batch).</summary>
    public int AutoApproveDrainBatchSize { get; set; } = 500;
    /// <summary>Interval between drain passes (operational cadence; ~every 15 min clears the ~1,650/day inflow with headroom).</summary>
    public TimeSpan AutoApproveDrainInterval { get; set; } = TimeSpan.FromMinutes(15);
```

- [ ] **Step 2: Write the failing worker test** in `AutoApproveDrainWorkerTests.cs` — one batch Approves a high-conf Resolved row, leaves a borderline (< 0.8 res_conf) Resolved row Pending, and leaves a NoResolution row Pending (Stage 1's sweep owns it). This is the business-outcome test — RED on the unfixed code (no drainer today):

```csharp
[Fact]
public async Task Drain_approves_high_conf_resolved_only()
{
    await using var db = NewInMemoryContext();
    db.ExtractedObservations.AddRange(
        Seed(1, ReviewStatus.Pending, ResolutionState.Resolved, instrument: Guid.NewGuid(), confidence: 0.9f, resConf: 1.0f),  // approve
        Seed(2, ReviewStatus.Pending, ResolutionState.Resolved, instrument: Guid.NewGuid(), confidence: 0.9f, resConf: 0.75f), // keep Pending (<0.8, by design)
        Seed(3, ReviewStatus.Pending, ResolutionState.NoResolution, instrument: null,        confidence: 0.9f, resConf: null)); // keep Pending (Stage 1 owns)
    await db.SaveChangesAsync();
    var opts = new ExtractionOptions { AutoApproveDrainEnabled = true, AutoApproveDrainBatchSize = 500 };

    var (approved, rejected, skipped) = await AutoApproveDrainWorker.RunOneBatchAsync(db, opts, CancellationToken.None);

    Assert.Equal(1, approved);
    Assert.Equal(0, rejected);
    Assert.Equal(ReviewStatus.Approved, (await db.ExtractedObservations.FindAsync(1L))!.ReviewStatus);
    Assert.Equal(ReviewStatus.Pending,  (await db.ExtractedObservations.FindAsync(2L))!.ReviewStatus);
    Assert.Equal(ReviewStatus.Pending,  (await db.ExtractedObservations.FindAsync(3L))!.ReviewStatus);
}
```

- [ ] **Step 3: Run — expect FAIL** (worker not defined). Run: `dotnet test --filter 'Name~Drain_approves_high_conf'`. Expected: FAIL.

- [ ] **Step 4: Implement `AutoApproveDrainWorker`** — a `BackgroundService` whose testable core is `static RunOneBatchAsync(SentinelDbContext, ExtractionOptions, CancellationToken)`: query `ReviewStatus == Pending` cursor-by-Id (batch-sized), call `AutoApprovePolicy.Evaluate(observation)` per row (the SAME single-source overload the backfill endpoint uses at `AdminEndpoints.cs:1185`), apply `SetReviewStatus(Approved, ApprovedNote)` / `SetReviewStatus(Rejected, RejectedNote)` for Approved/Rejected, leave Pending untouched, `SaveChangesAsync`, return `(approved, rejected, skipped)`. The hosted loop: skip when `!AutoApproveDrainEnabled`; else loop `RunOneBatchAsync` until a pass approves+rejects zero (queue converged for this pass), sleep `AutoApproveDrainInterval`, repeat. Emit `SentinelMeter.AutoApproveDecisionCounter` per decision (SAME counter as the V1 hot path, so the existing near-total-rejection alarm covers this drainer too) plus a bounded `mode="drain"` tag. Register in `Program.cs` via `AddHostedService`. Follow LOG_RULES (Info batch summary, Warn on transient DB error + continue). **Do NOT touch `RunV2ProductionAsync` — the V2 path stays Pending-only.**

- [ ] **Step 5: Run — expect PASS.** Run the filter. Expected: PASS.
- [ ] **Step 6: Full compile + tests.** `bash SentinelCollector/.devcontainer/compile.sh` (full output, `$?`==0, 0/0, all pass).
- [ ] **Step 7: Commit.** `git add src/Workers/AutoApproveDrainWorker.cs src/Configuration/ExtractionOptions.cs src/Telemetry/SentinelMeter.cs src/Program.cs tests/.../AutoApproveDrainWorkerTests.cs && git commit -m "feat(sentinel): automated downstream review-queue drainer (Fix 3b, DD-4=b)"`

### Task 3.3: Add D-9 (new entry, NOT a supersession) to the card

- [ ] **Step 1: Add D-9 to `SentinelCollector/AGENT_README.md` DECISIONS block as a NEW entry (no "supersedes"):**

```
  D-9 automated-review-drain: INTENT drain the Pending review queue DOWNSTREAM of the V2 write (the V2 path keeps writing everything Pending — DD-4=b PRESERVES the 2026-05 cutover "downstream owns approval": this scheduled drainer IS that downstream approver, so NO cutover reversal / NOT a supersession). AutoApprovePolicy.Evaluate runs on a schedule, NON-dry-run, over review_status=Pending rows only, batched + interval-bounded, gated to the SAME thresholds as the backfill endpoint (single source of truth) / PRECOND res_conf reflects the RESOLVING leg (Fix 3a — ResolverOutcome.ResolutionConfidence), NOT the LLM self-report; gate = extraction>=0.9 AND Resolved AND res_conf>=0.8 AND instrument non-null; exact-match rows (D-8, res_conf 1.0) approve, 0.6-0.8 rows stay Pending BY DESIGN (DD-5); the <0.5 auto-reject branch is inert post-#607 (harmless) / GUARD AutoApproveDrainWorker.RunOneBatchAsync @ src/Workers/AutoApproveDrainWorker.cs:<line> / TEST AutoApproveDrainWorkerTests.Drain_approves_high_conf_resolved_only
```

- [ ] **Step 2: Confirm `@ file:line` and the `INTENT(D-9)` comment agree; compile; commit.** `git add SentinelCollector/AGENT_README.md src/Workers/AutoApproveDrainWorker.cs && git commit -m "docs(sentinel): D-9 automated review-queue drain (Fix 3b) — atomic w/ worker+test"`

**Stage 3 D-entry / INTENT_FIDELITY:** **D-9 `automated-review-drain`** — a NEW entry, NOT a supersession. The V2 cutover comments are left untouched (DD-4=b preserves "downstream owns approval"; the drainer IS the downstream). Fix 3a's res_conf-source change is the precondition captured in D-9.

**Stage 3 guard/business-outcome tests:** Task 3.1 (`uses_outcome_res_conf_not_llm_self_report` — res_conf sourced from the resolving leg, RED if it reverts to the LLM self-report) + Task 3.2 (`Drain_approves_high_conf_resolved_only` — the drainer Approves high-conf Resolved rows and leaves borderline/NoResolution Pending; doubles as the D-9 guard, RED if the drainer is removed or the threshold gate is bypassed). Both RED on the unfixed code.

**Stage 3 operational cadence:** drainer runs every `AutoApproveDrainInterval` (default 15 min); at 500/batch it clears the ~1,650/day inflow with large headroom and burns down the existing Resolved-but-Pending fraction over the first few passes. Enable via `Extraction__AutoApproveDrainEnabled=true` in `compose.yaml.j2` after smoke-testing the batch counts.

**Stage 3 migration:** none (`res_conf` column exists; `ResolverOutcome` is a C# record; the drainer adds no schema).

**Stage 3 out-of-scope:** any change to `RunV2ProductionAsync` / the V2 hot path (DD-4=b keeps it Pending-only); lowering the `>= 0.8` gate (DD-5 keeps it); re-basing the inert `< 0.5` auto-reject floor; any ThresholdEngine change; the async `ResolutionWorker`.

---

## Post-Implementation Verification (all stages — COMPLETION_GATE)

Per CLAUDE.md COMPLETION_GATE, after each stage's deploy (`ansible-playbook playbooks/deploy.yml --tags sentinel-collector`):

- [ ] Smoke: `POST /admin/review/backfill-auto-approve {"dry_run":true}` and (Stage 1) `POST /admin/review/close-noresolution {"dry_run":true}` — report processed/approved/rejected/closed counts (dry-run, no writes). Confirm non-zero close-eligible and approve-eligible counts.
- [ ] Loki (last 10 min, anchored to `date -u`): `severity_text` error scan for `service=sentinel-collector` — report findings.
- [ ] Prometheus: `sentinel_noresolution_sweep_closed_total` (Stage 1) climbing; `sentinel_auto_approve_decisions_total{decision="approved"}` (Stage 3) climbing; queue depth `sentinel.extracted_observations WHERE review_status='Pending'` trending DOWN (SELECT-only psql).
- [ ] Confirm the digest still emits (Stage 1 did not empty it): trigger `DigestScheduler` path or inspect the latest ntfy digest — AutoClosed macro rows still present.

---

## Self-Review (spec coverage / placeholders / type consistency)

- **Coverage:** Fix 1 -> Stage 1 (DD-1/DD-2, digest-safe terminal status, sweep + backlog). Fix 2 -> Stage 2 (DD-3, exact-match gate, D-8, D-2 conflict resolved as non-conflict). Fix 3 -> Stage 3 (DD-4=b/DD-5, res_conf root cause propagated in 3a, automated downstream drainer in 3b, D-9 as a NEW entry — NO cutover reversal / NO supersession, auto-reject-floor decision). All four "for each stage" asks (goal/files/D-entry/guard-test/backfill+migration/business-test/out-of-scope) present per stage. Ordering + rationale present. Migration analysis: zero expected, Stage 1 gated on a verification step.
- **Type consistency:** `EvaluateClose` / `AutoClosed` / `AutoClosedNote` (Stage 1); `TryExactCandidateMatchAsync` / `"llm_candidate_exact"` / `SharedTokenCount` / `GetInstrumentBySymbolAsync -> InstrumentLookupDto(Id,Symbol,Name)` (Stage 2); `ResolverOutcome.ResolutionConfidence` / `AutoApproveDrainWorker.RunOneBatchAsync` / `AutoApprovePolicy.Evaluate` (Stage 3) — names consistent across tasks and against the grounded code. The V2 cutover comments are deliberately left untouched (DD-4=b).
- **Placeholders:** `@ file:line` in D-8/D-9 and the `INTENT(D-n)` comments are the ONLY deferred values — resolved in the same task that writes the guard (the atomic-set rule requires the final line number). No TBD/TODO in committed code.
