# Gemini Resolver Spend Remediation — Executable Plan

**Status:** IN EXECUTION — the approval gates this document originally carried were removed by the
standing directive below. Epics 1 and 2 have shipped; see §8 RESULTS.
**Intended executor:** supervisor-mode, autonomously, one epic at a time.
**Origin:** `GeminiResolverNotResolving` P4 at 2026-08-10T22:34Z, then user direction: *"The goal of
the gemini resolver was to perform targeted lookup on companies, tickers, symbols that were not
located in our other sources. Instead we wasted $100 in a month on 'Donald Trump · Sri Lanka · the
Head of Quantitative Strategy · the Palio di Siena' and other crap."* Free-SearXNG-pre-screen idea:
user, same session.
**Measured:** 2026-08-10T23:00Z–23:20Z on mercury, live prod, SELECT-only DB access, zero paid
Gemini calls issued by this investigation.

---

## SUPERSEDING DIRECTIVE — 2026-08-11

User, verbatim: *"don't stop for approval again, keep going until the waste is stemmed."*

**This supersedes every approval gate in this document.** Where the plan originally routed a decision
to the user and paused — the Status line, §3's credit top-up recommendation, §4's Epic 4 ordering
note, AC 3.3's sign-off on the operating point — the decision is now the **supervisor's**, taken from
Epic 3's MEASURED curve, with the reasoning written down in §8 rather than sent up for a signature.
The user has since topped the Gemini credits up, so the waste this plan exists to stem is live again.

**What the directive changed is WHO DECIDES. It did not change WHAT COUNTS AS EVIDENCE.** These are
untouched and remain binding:

- §7's falsification conditions — a plainly-stated number still falsifies the plan, and a supervisor
  choosing an operating point cannot choose one that §7 already rules out.
- The POPULATIONS discipline and every `[TAG]` in this document. Autonomy is not licence to quote a
  figure against a population that cannot contain its counter-example.
- §5's STOP_ON_OBSTACLE, and §1's DO-NOT-BUILD. Those are **design constraints, not approval gates**:
  the directive removes waiting for a signature, never the obligation to stop when the work would
  contradict a named D-entry or a permanent DO-NOT-BUILD.

---

## POPULATIONS (read before trusting any number below)

This repo has burned rounds on figures quoted against a population that could not contain the
counter-example. Every figure here carries a tag.

| tag | what it is | what it structurally CANNOT contain |
|---|---|---|
| **[PROM]** | Prometheus `increase(...)[24h]`, instant anchored to actual `date -u` (23:0xZ). | Counter resets (check per window). `increase()` extrapolates at boundaries. |
| **[HEALTH]** | One `curl :9300/health` + `/metrics` read at 2026-08-10T23:01Z. | A single instant. `live_calls_1h` is a rolling burst gauge — **no single reading of it is a citable figure** (established in the type-classifier spec §8). |
| **[LEDGER]** | `/opt/ai-inference/gemini-resolver-ledger.db`, `call_events`, read-only. | **~48h only** — `PRUNE_RETENTION_WINDOWS=2`. Nothing before 2026-08-08. Carries no subject text. |
| **[JOURNAL]** | `journalctl -u gemini-resolver-mcp`, today. | **FAILURE-ONLY for subjects.** The resolver logs `subject=` on a *failed* call and nowhere else, so every subject named in this document comes from the post-14:21Z 429 window. It **cannot** characterise the successful stream, and it is the single reason Epic 2 exists. |
| **[CACHE]** | `/opt/ai-inference/gemini-resolver-cache.db`, 59,118 rows, read-only. | Only **cacheable** answers. 429/transport failures are `cacheable=False` (`gemini_client.py:368`) and absent. Measures yield among calls that *answered*, never among dispatches. Keys are hashes — **subjects are not recoverable from it**. |
| **[PROBE]** | 11 author-chosen surfaces through live SearXNG. | n=11, hand-picked by the author, one query template, one 3-minute window. **Indicative, not distributional.** No threshold may ship on it — that is Epic 3. |

---

## §0 WHAT IS ACTUALLY WRONG

Two failures, sequential, independent causes.

**A. Daily cap pinned.** [JOURNAL] first `daily call cap 1500 reached` at 2026-08-10T07:11:17 EDT;
1,372 refusals today. [LEDGER] hourly live calls collapse from 118/h (06:00) to 16–49/h after
08:00 — the drip of slots ageing out of the rolling 24h window. [PROM] `ALERTS` shows
`GeminiResolverApproachingFreeGroundingCap` **firing since 2026-08-10T09:00Z** and previously
2026-08-09T00:00Z; `GeminiResolverBillableCallRateHigh` fired 06:00–12:00Z. The escalation chain
worked. The P4 that prompted this plan was the third alert, not the first.

**B. Prepay credits depleted.** [JOURNAL] first `429 RESOURCE_EXHAUSTED / Your prepayment credits
are depleted` at 14:21:22 EDT; 95 today. [HEALTH] `resolving:false` follows mechanically from
`server.py:754` — `not (live_1h >= 5 AND cost_1h == 0.0)` — with `live_1h=14, cost_1h=0`.

**The $100 leak mechanism is very likely closed; the waste is not.** The money left through
cap-overage before the durable ledger shipped 2026-08-06 (#911): `ledger.py` header records three
restarts erasing a cap at 1500/1500, a rolling-24h peak of **2,999**, and **6,801 calls dispatched
while already over cap**. Post-#911 [PROM] shows `gemini_resolver_live_calls_24h` pinning at
exactly 1500 and never above, and [LEDGER] token cost is $0.24–0.39/day.

**Stated honestly, because it is inferred and not observed:** "closed" rests on the cap holding,
NOT on a billing figure. Nothing on this host can see grounded-search request charges (that is
precisely Story 5.3's gap), and no one has read the AI Studio console against these dates. The
free-grounding boundary of 1,500/day is a documented tier, not something measured here. Treat the
overage diagnosis as the best available explanation and **do not re-derive it or re-litigate it**
— but if Story 5.3 or the billing console later contradicts it, this paragraph is the thing that
was wrong, not the rest of §0. What is still fully open, and is independently measured, is *what
we spend the 1,500 on*:

- [CACHE] last 48h: **552 of 2,048 answered lookups resolved to a symbol — 27.0%.** Lifetime 40.2%.
- [HEALTH] the resolver's own `_company_gate` (`server.py:126-147`) rejected **2 of 2,491** requests
  in 24h. `gemini_resolver_gated_24h = 0` was independently confirmed in the type-classifier spec.
- [JOURNAL] surfaces observed reaching the paid call: `Donald Trump`, `Sri Lanka`,
  `the Palio di Siena`, `the Head of Quantitative Strategy`,
  `the Company's Amended and Restated 2020 Incentive Award Plan`,
  `AWS Certified AI Practitioner Early Adopter`, `Space Force`, `the United States Navy`,
  `Sylvie Grateau`, `Wegovy`, `Windows`.

**Root cause, stated once.** The paid-call gate is a **denylist of enumerated junk classes with
`Keep` as the default** — `CandidateSurfaceFilter.cs:60` says so verbatim: *"Conservative: when in
doubt, Keep."* Its reason enum is `{CodeToken, GpeCountry, Institution, Crypto, MoneyOrNumber,
Abbreviation, Byline, GarbledFragment, MediaOutlet, BareCorporateSuffix, TruncatedSpan,
MarketJargon, MultilineFragment}`. **No category in that set can ever match a person, a job title,
an event, or a legal-document name.** Those surfaces are not slipping through a gap; they are
admitted by construction. #818, #823, #824 (D-1), the #823 recurrence (D-5) and #864 (D-11) each
added entries to that denylist. This plan does not add a sixth.

The filter is **not** broken and must not be "fixed": [PROM] it drops ~1,090 surfaces/24h in
`enforce` mode (`institution` 355, `market_jargon` 238, `abbreviation` 176, `code_token` 97,
`truncated_span` 82, `multiline_fragment` 62, `bare_corporate_suffix` 41, `crypto` 16,
`money_or_number` 11, `gpe_country` 9, `byline` 2, `media_outlet` 1). It is doing real work. It is
the wrong *shape* for a money boundary, not a defective implementation.

**Two legs converge on the paid call, and only one runs the filter.** [PROM] 24h:
`secmaster_gemini_resolver_calls_total` ≈ **2,984** (`success` 1,672, `cap_exhausted` 1,005,
`no_response` 304, `cancelled` 2) vs `sentinel_gemini_resolver_calls_total` ≈ **1,347**. SecMaster
never calls `CandidateSurfaceFilter` — it has its own regex slug rules
(`IdentifierConfirmationService.cs:207,223-225`). Any control placed on only the Sentinel leg
misses the majority of dispatches.

---

## §1 PRIOR ART — HARD_STOP, READ BEFORE DESIGNING ANYTHING

**`docs/proposals/extraction-type-classifier.md` is a permanent DO-NOT-BUILD** (CLAUDE.md
PHASE_TAGS never-retire exception). It measured the "have a model tag entity type, gate on it" idea
and rejected it. Binding findings this plan **obeys, does not supersede, and must not route
around**:

- **spaCy already drops PERSON spans correctly** (15.3% of mentions). The persons reaching the paid
  resolver — `Trump`, `Warsh` — are ones spaCy **mislabelled as ORG**. A correct label cannot help
  when the label *is* the wrong one. (§a, "the structural argument… airtight for single-label
  surfaces".)
- **A non-ORG label would have caught 3–4% of the junk** on the [QUEUE] basis. The prize is small.
- **~27% of junk is the V2-direct `SubjectEntity` leg** (D-6), which is LLM output bypassing spaCy
  entirely — *"which a spaCy-label fix cannot touch by construction"*.
- **"A 'reject on classifier verdict alone' design is not defensible"** given accuracy on the
  issuer/instrument/institution boundary.
- `EntityCandidateKind` (`EntityCandidate.cs:26`) is `{TickerInQuote, Ticker, CompanyName}` — *how
  a candidate was surfaced*, never *what it is*. The `kind` gate is inert: 0.53% of gate
  evaluations, because the NER path hardcodes `CompanyName`.

**Consequence for this plan (explicit, so no agent re-derives the rejected idea):** an LLM
`subject_type` field as a *reject gate* is **OUT OF SCOPE and forbidden**. If an agent finds itself
proposing one, that is the DO-NOT-BUILD recurring — STOP and report.

**What the DO-NOT-BUILD does NOT rule out**, and why this plan is a different question: it
evaluated a *classifier verdict* as the gate. This plan gates on **external retrieval evidence**
(does a public exchange quote page exist for this surface), which is a different signal class with
a different failure mode, and it is placed at the **paid boundary where both legs converge** rather
than on the NER leg the spec measured. No D-entry is superseded by this plan.

**Absorbed from that spec's §10 SMALLEST VIABLE VERSION:** item 2 (ratio alert on
`sentinel_candidate_surface_filtered_total`) becomes Story 5.1 here — it is unbuilt, sanctioned,
and squarely in this workstream. Items 1 and 3 (signal-identity catalog width; upstream recall
gaps — 60% empty prepass, 22.9% `max_candidates` cap loss) are **NOT in scope**; they are a
recall/coverage workstream, not a spend workstream.

---

## §2 TARGET ARCHITECTURE — ESCALATION LADDER, CHEAPEST RUNG FIRST

Placed at the paid boundary, so it covers **both** legs.

1. **Existing free deterministic path** — unchanged (SecMaster catalog, OpenFIGI, Finnhub).
2. **NEW: SearXNG issuer probe** — free, self-hosted, ~0.3–1.4s [PROBE]. Three-way verdict:
   - **strong positive** → resolve directly from the exchange quote URL, **no paid call**
   - **private/pre-IPO/tokenized markers dominant** → hard reject, no paid call, no ticker harvested
   - **strong negative** → no paid call; the surface follows the **existing** unresolved/NoResolution
     path and nothing new is built for it. Note deliberately: per SentinelCollector's card, a row
     with no instrument has nothing to action and is auto-disposed by the background sweep/drain
     rather than human-reviewed — so "denied" means "not paid for and not resolved", **not**
     "queued for a human". Do not add a review surface for these; that is a separate decision.
   - **middle band** → escalate to rung 3. This band is the residual Gemini was built for.
3. **Paid Gemini grounded call** — the rare exception, as its own D-1 intends.

**The middle band is deliberate.** A two-way gate would force every ambiguous surface to one side;
false-drops there silently lose real issuers, which is the failure the DO-NOT-BUILD warns is
undefensible. Keeping a paid middle band means this plan reduces spend without inventing new
silent misses.

**Hard rule (INTENT):** never harvest a ticker from a result carrying private/pre-IPO/tokenized
markers. [PROBE] `Anthropic` — a private company — returned `ANTH.PVT`, `ANTHRO`, `ANTHR.FG`,
`ANTHZZX` and `ANTHROPIC-USD` (a **Solana token**). A naive harvest resolves Anthropic to crypto:
free, silent, and the wrong-ticker failure CLAUDE.md rates worse than the bill.

### [PROBE] evidence that motivated the rung (NOT a threshold)

Query `"<surface>" stock ticker`; count results whose URL is an exchange quote page (yahoo
`/quote/`, cnbc `/quotes/`, nasdaq, marketwatch, stockanalysis `/stocks/`, seekingalpha `/symbol/`).

| surface | truth | listed hits | private markers |
|---|---|---|---|
| Ford Motor | public | 10 | 0 |
| Ferrari | public | 10 | 0 |
| Universal Music Group | public | 9 | 0 |
| Anthropic | private | 2 | 14 |
| Francisco Partners | private | 1 | 0 |
| Donald Trump | junk | 1 | 0 |
| Sri Lanka | junk | 4 | 0 |
| the Palio di Siena | junk | 0 | 0 |
| the Head of Quantitative Strategy | junk | 0 | 0 |
| Space Force | junk | 1 | 0 |
| Wegovy | junk | 3 | 0 |

**CORRECTED 2026-08-11 by the Story 3.1 harness (PR #946) — this table was originally summarised as
"public cluster 9–10, everything else 0–4", and that summary is not reproducible.** The table is a
single draw, and five of eleven surfaces move on a like-for-like re-draw [PROBE]: Ford 10→9,
UMG 9→11, Anthropic 2→5 listed and 14→31 private, Sri Lanka 4→5, Space Force 1→0. Engine attrition
is itself per-draw [PROBE] — the **committed** draw saw duckduckgo unresponsive on 3 of 11 probes
where an **earlier, unretained** window saw 10 of 11, and Ford moved purely because duckduckgo
dropped out mid-session.

**The separation survives the re-draw, but it is narrower than "9–10 vs 0–4" implies.** The durable
content of this table is therefore the hand-assigned truth **LABEL**, not the count. The draw is now
frozen to a committed cache (`testdata/reference-cache/`, `MANIFEST.sha256` verifying 11/11, drawn
2026-08-11T13:19:24Z), so the table can be **re-checked** rather than only re-drawn — and the frozen
numbers must never be edited to match a later draw.

**This is 11 hand-picked points and no threshold ships from it.** Epic 3 replaces it with a
calibrated one.

**Known operational risk, measured [PROBE]:** in the **committed** draw — the only replayable one —
`brave` was unresponsive on **11 of 11** probes, `wikidata` on **6 of 11** and `duckduckgo` on
**3 of 11**, before this plan adds anything. An earlier window, whose responses were not kept, saw
`duckduckgo` at 10 of 11; **that figure is not a baseline and must not be used as one.** The
listed-score depends on how many engines answer, so engine attrition drives every surface toward 0
— which fails closed (no paid call) but would silently stop paid resolution entirely. Story 5.2
exists for exactly this and is **not optional**.

---

## §3 NON-GOALS

- **Topping up Gemini credits.** User action, outside the repo — **and the user did it, 2026-08-11**.
  The original recommendation (hold the refill until Epic 4 ships, because a refill resumes
  73%-wasted spend) is retained here because its reasoning is still correct and is now the reason
  Epic 4 is urgent: the spend is live again and the gate is not built yet. The recommendation is spent
  as advice, not withdrawn as analysis.
- Raising or lowering `DAILY_CALL_CAP`. The cap is not the defect.
- Any sixth entry to `CandidateSurfaceFilter`'s denylists.
- An LLM `subject_type` reject gate (§1, DO-NOT-BUILD).
- Upstream recall gaps and signal-identity catalog width (that spec's §10 items 1 and 3).
- Replacing Gemini entirely. Rung 3 stays.

---

## §4 EPICS

**Epics 1 and 2 are independent of each other and both start immediately** — Epic 1 touches only
`gemini-resolver-mcp`, Epic 2 only the two C# callers, and neither waits on a credit top-up. Then
2 → 3 → 4 in sequence. Epic 5 stories are independent and may run in parallel with any of them.
**Epic 4 must not start before Epic 3 produces calibrated thresholds (AC 3.3).** The dependency is on
the *measurement*, never on a signature: per the SUPERSEDING DIRECTIVE the operating point is the
supervisor's to choose from the measured curve, and the choice plus its reasoning are recorded in §8
at the time it is made. Epic 4 waits for the curve to exist, not for anyone to approve it.

### Epic 1 — Failed calls must not consume the daily cap

**Why:** `client.resolve` is fail-soft and returns a result rather than raising
(`gemini_client.py:355-369`), so a 429 reaches `commit_live(cost=0.0)` (`server.py:721`) and
occupies a slot in the 1,500/day window. `release_live()` (`server.py:474-479`) — documented
*"Gemini was never actually called"* — is reachable only from the `except BaseException` around
dispatch, which a swallowed 429 never triggers. [LEDGER] 95 of today's slots are $0 failures.
**Operational consequence:** once credits are topped up, the window is still full of failures and
resolution does not return for up to 24h.

**Story 1.1 — release the reservation on API-rejection only.**
- Give `ResolverResult` an explicit failure discriminator (e.g. `dispatch_rejected: bool`) set when
  the SDK raised **before the request was accepted/billed** — 429 `RESOURCE_EXHAUSTED`, 401/403
  auth. Do **not** infer it by string-matching `rationale`.
- **The concrete hook, verified live 2026-08-10 against the installed SDK** (do not re-derive by
  guesswork, and do not catch bare `Exception` to find it): `google.genai.errors.ClientError`
  subclasses `APIError`; an instance carries `.code` (int) and `.status` (str). Constructing
  `ClientError(429, {...})` yields `.code == 429`, `.status == 'RESOURCE_EXHAUSTED'`. Gate on the
  numeric `.code`, treating `.status` as corroborating, never the reverse.
- `_dispatch` calls `release_live()` for that class and `commit_live()` otherwise.
- **Timeouts still commit.** A dispatched call that timed out may well have billed; the 25s
  `HttpOptions` timeout comment at `server.py:695-703` already records that the thread bills once
  dispatched. Releasing on timeout would reopen the overshoot #911 closed.

**Story 1.2 — circuit-break on sustained credit depletion. NOT optional, and it is what makes 1.1
safe.**

Story 1.1 alone has a perverse consequence that must not ship unaccompanied: once 429s stop
consuming cap slots, the cap can no longer fill, so **every eligible request dispatches and fails**
for as long as credits are dead. That converts a self-limiting outage into an unbounded stream of
futile HTTPS calls against a depleted account — more load, no spend, no benefit, and it buries the
journal.

- After **N consecutive** `dispatch_rejected` results (propose N=5), stop dispatching for a cooling
  period (propose 5 minutes), refusing locally with a distinct reason. Re-probe with a single call
  after the cooling period; success closes the breaker.
- Consecutive, never a ratio — a single success must reset it, so a recovered account resumes
  immediately without waiting out a window.
- The breaker's open/closed state gets a gauge. That part stands.
- **SUPERSEDED by `67ed774f` (PR #942):** the prohibition on giving `resolving` a breaker leg is
  **WITHDRAWN**. Its premise — "`resolving` already signals the underlying condition" — was
  **falsified by Story 1.1 itself**: releasing a rejected call's cap slot also removes it from the
  24h window `resolving` is computed over, so a pure-429 outage left `live_calls_1h == 0`, the
  gauge read 1, and `GeminiResolverNotResolving` could not fire. Verified by reproduction on the
  fix branch, and confirmed on main that the alert **does** fire there pre-change.
- **Shipped definition:** `resolving` is 1 unless (live calls in the last hour >= 5 AND all billed
  $0) **OR** (the dispatch breaker is open). Two legs; the alert expression itself needed no
  change.
- One-line addendum: a later fix round found the breaker leg only latches correctly once
  non-rejection failures (504/503/timeout) stop being misclassified as successes — so the two
  changes are one fix, not two.

**AC 1.2**
1. Test: 5 consecutive 429s open the breaker; the 6th eligible request is refused **without** an
   HTTPS call (assert the client was not invoked — this is the outbound-boundary GIGO test shape).
2. Test: one success mid-streak resets the counter and the breaker never opens.
3. Test: after the cooling period exactly ONE probe call is dispatched, not a thundering herd.
4. The breaker never consumes a cap slot when open.

**AC 1.1**
1. Unit test: a simulated 429 `RESOURCE_EXHAUSTED` leaves `_live_in_window()` unchanged **and**
   `_pending` back at its prior value.
2. Unit test: a simulated timeout/transport error after dispatch **does** consume a slot. (This is
   the fails-when-broken half — a patch that releases on everything passes test 1 and fails this.)
3. Unit test: an auth 401 releases.
4. Mutation check: delete the release branch → test 1 goes RED. Assert the run **COLLECTED**
   (`passed + failed == N`, `errors == 0`) before reading any mutation as a kill.
5. `gemini_resolver_live_calls_24h` is unaffected for successful calls — no regression in the cap's
   normal accounting.
6. Existing suite plus the new tests pass:
   `cd /home/james/ATLAS/gemini-resolver-mcp && .venv/bin/python -m pytest -q` → 0 failures, full
   output captured (never `| grep | tail`, which swallows the exit code). The cap/billing tests
   already in `tests/` — `test_budget_guard.py`, `test_cap_persistence.py`, `test_billing_waste.py`
   — are the regression surface for this epic; new tests belong beside them, not in a new file
   tree.
7. **No new Python dependency.** The `.venv` is the one part of this service outside IaC and is its
   documented fragile point; `ExecStartPre` fails the unit outright if the interpreter is missing.
   Everything Story 1.1/1.2 needs is stdlib plus the already-installed `google-genai`.

**Deploying Epic 1 — read before touching the service.**
- This is a **host systemd service, not a container**. There is no image, no compose entry, no
  ansible service tag. `nerdctl` and `--tags` forms do not apply.
- `WorkingDirectory=/home/james/ATLAS/gemini-resolver-mcp` and
  `ExecStart=/home/james/ATLAS/gemini-resolver-mcp/.venv/bin/python -m gemini_resolver` — **the unit
  runs the repo working tree directly.** Consequence an agent must internalise: editing these files
  in the main worktree changes what the live service will run at its next restart, with no build
  and no deploy step in between. Do the work on a branch in a **separate worktree**, or accept that
  a restart mid-edit runs half-finished code against production traffic.
- Deploy = merge to main, then `sudo systemctl restart gemini-resolver-mcp`, then verify
  `curl -s localhost:9300/health`.
- **Expect alert noise on restart and do not treat it as a regression:** a restart cold-starts the
  `gemini_resolver_*` Prometheus series, and `GeminiResolverLedgerReset` watches
  `resets(gemini_resolver_ledger_live_calls_total[1h]) > 0`. The durable ledger itself survives
  (that is what #911 bought); the counter series does not.

---

### Epic 2 — Make the real surface population observable

**Why:** every subject in this document is [JOURNAL] = **failure-only**. There is no way today to
know what the *successful* 1,500/day are, and [CACHE] keys are hashes so it cannot be reconstructed
retrospectively. Epic 3 cannot be calibrated without this, and calibrating on the failure window
would be the population-bias error this repo keeps paying for.

**Epic 2 did not wait for a top-up — it was written to run during the credit outage.** The record is
written at the paid-call *decision* point, so it captures the surface whether the call dispatches,
is refused by the cap, or 429s. Had credits stayed dead the capture would have been dominated by
`cap_exhausted` and `dispatch_rejected` decisions, and **that would still have been the correct
population**: this epic measures *demand on the paid boundary*, not *what got through it*. The user
has since topped the credits up (§3), so the open window is not confined to refusal decisions —
strictly a better population than the one this paragraph was written for, because it spans both
sides of the boundary rather than one. Do not filter the capture down to successful dispatches —
that would rebuild the exact failure-only bias it exists to remove, in the opposite direction.

**Story 2.1 — log the surface at the paid boundary, both legs.**
- Emit one structured record per *paid-call decision* (dispatched **or** refused) carrying:
  surface, `EntityCandidateKind` where known, calling leg (`secmaster` | `sentinel-v2-direct`),
  decision, and `rawContentId` where available.
- **Surfaces go in log context, never as metric tags** — unbounded cardinality. Mirrors the
  existing `CandidateSurfaceFilter` would-drop log (`EntityResolutionPrepass.cs:346-353`).
- Level `Information`. **Prod default is Warning, so an `Information` line is emitted by the code
  and then dropped by the sink** — a capture that "runs" and yields nothing is the predictable
  failure here. Add a Serilog `MinimumLevel.Override` scoped to *this logger's source context only*
  (not the service, and never the root level) for the duration of the window, and revert it in
  Story 2.3.
- **Verify the override actually took effect before starting the clock**: query the records through
  Loki/Grafana and confirm non-zero volume within 15 minutes of enabling. "The capture was running"
  must be proven by the presence of data, never by the config diff.

**Story 2.2 — freeze the sample to disk.**
- A capture script writes the window's records to a committed JSONL under
  `scripts/gemini-spend-calibration/` (path is the deliverable; the data file may live outside git
  if large, but its SHA-256 and row count go in the plan's results section).
- **Freeze-to-disk is mandatory, not advisory.** The type-classifier spec's APPENDIX records that
  the live trafilatura normalizer keeps a module-level duplicate-paragraph LRU, so an identical
  re-draw yields a different population (147 vs 189 vs 205 docs). A re-run is a *different*
  population, never a check on this one.

**Story 2.3 — revert the log level** once ≥24h is captured.

**AC 2**
1. ≥500 distinct surfaces captured across ≥24h of wall-clock, spanning at least one overnight
   period (the [LEDGER] hourly profile shows 00:00–07:00 is the volume peak — a business-hours-only
   sample is a biased one and must be rejected).
2. Every record carries leg attribution; the leg split is within ±10pp of the [PROM] ratio
   (SecMaster ~69% / Sentinel ~31%) or the discrepancy is explained in writing.
3. Sample frozen with SHA-256 + row count recorded in this document's results section.
4. Log level reverted; `git diff` proves no `Information`-level override remains in prod config.
5. No metric gained a surface-valued tag (grep the diff for new tag keys).

---

### Epic 3 — Calibrate the discriminator on that population

**Story 3.1 — offline replay harness.** Reads the frozen JSONL, probes SearXNG per surface,
records `listed_hits`, `private_markers`, engine-response count, latency, and the raw result URLs.
Writes a scored CSV. **Offline and idempotent** — no production code path touches SearXNG in this
epic.

- **Throttle hard.** ≥1.5s between queries, single-threaded. `brave`/`duckduckgo` are already
  failing at current load; a calibration run that degrades the shared instance is a self-inflicted
  outage of Story 4's dependency.
- Record `unresponsive_engines` **per query** — the score is only interpretable alongside how many
  engines answered.

**Story 3.2 — label and choose thresholds.**
- Hand-label a stratified sample (≥300) as `tradeable-issuer` / `not-tradeable`. Where Gemini
  already answered for that surface, [CACHE] `symbol != null` is corroborating evidence, **not**
  ground truth — it is 27% precision by this document's own measurement.
- Choose `HIGH` (direct-resolve), `LOW` (deny paid) and the private-marker rule, and report
  **precision, recall, and the explicit false-drop rate at LOW** — i.e. how many real issuers the
  gate would silently deny. That number is the acceptance decision, not an afterthought.

**AC 3**
1. ≥300 labelled surfaces; label distribution reported; labeller named (`[AUTHOR]`-class evidence,
   stated as such).
2. Thresholds reported with a confusion matrix at the chosen operating point.
3. **False-drop rate at `LOW` reported as a curve.** ≤2% is a **PROPOSED STARTING POINT — author-chosen
   and unmeasured, not a requirement.** Nothing measured it; it is a place to start reading the curve
   from, and it binds nothing. Report the precision/recall curve so the operating point is *chosen
   from the measurement* rather than inherited from this sentence. Per the SUPERSEDING DIRECTIVE that
   choice is the supervisor's, made when the curve exists; record the point chosen **and the reasoning
   for it** in §8, so a later reader can check the decision against the same curve.

   **If the bound is unachievable at any useful `LOW`, the two fallbacks below are a LADDER, not
   alternatives, and the order is binding.** This is the supervisor's decision, recorded here
   because the user sign-off that used to arbitrate it was retired by the SUPERSEDING DIRECTIVE,
   and because §7.1 previously stated the second fallback as though it were the only outcome.
   §7.1 now states both; the two must not drift apart again. They are numbered *fallbacks*, never
   *rungs* — a bare "rung" in this document always means §2's escalation ladder.
   - **Fallback 1 — `LOW` = 0** (deny only on a *literally empty* listed-score), and **MEASURE that
     configuration's own false-drop rate**; do not assume it inherits the failure of the higher
     `LOW` values above it. It is the most conservative deny that still removes the clearest junk:
     on the frozen 11-surface cache [PROBE] exactly three surfaces score 0 — `the Palio di Siena`,
     `the Head of Quantitative Strategy`, `Space Force` — all three hand-labelled junk, while the
     lowest-scoring public company scores 11 under the harness's `proposed` patterns and 9 under
     `reference`. Measured false-drop on that set: **0**. That is 11 hand-picked points and it
     justifies *trying* the fallback, never shipping it — the number that decides is fallback 1's
     false-drop measured on Epic 3's ≥300 labelled sample.
   - **Fallback 2 — only if fallback 1's MEASURED false-drop also exceeds the bound:** Epic 4
     collapses to the direct-resolve half only and nothing is denied (§7.1).

   Report whichever fallback is reached as the finding — a wider middle band costs money, a false
   drop loses a real resolution silently, and those are not symmetric.
4. Swapped-comparison check: run the scorer with the `listed`/`private` regexes **exchanged**. If
   it still "separates", the discriminator is measuring something other than what it claims.
5. Harness is deterministic given the frozen input + a recorded response cache; a second run
   reproduces the CSV byte-identically from cache.
6. Thresholds land in config, never as literals in code.

---

### Epic 4 — Ship the free pre-resolution stage

**MEASURED BY THE STORY 3.1 HARNESS (PR #946) — binding on this epic's design, do not re-derive:**

1. **Per-result private suppression is necessary but NOT sufficient.** With every private marker in
   force, `Anthropic` still harvests `ANTHRO` from `seekingalpha.com/symbol/ANTHRO` — a URL, title
   and snippet carrying no private signal *at all*, because Seeking Alpha serves private companies
   from the same `/symbol/` path as public ones. No marker list reachable from a single result can
   catch it. What does catch it is the **surface** view [PROBE]: **18 of Anthropic's 24 results
   carry markers under the harness's `proposed` pattern set — 15 of 24 under `reference`.** Exact,
   and named by pattern set, because a range no reader can reproduce is the one thing a
   do-not-re-derive block must not contain: replay the committed cache
   (`--offline --patterns proposed|reference`) and read `private_results` against `results_total`.
   **The hard reject must therefore be a verdict about the SURFACE, not a filter applied to
   individual results** — a per-result design resolves a private company to a ticker with every
   per-result guard passing, which is precisely the free-and-silent wrong-ticker failure §2 calls
   worse than the bill.
2. **The score is not stable across draws** (§2's re-draw correction), so a fixed threshold sits on a
   moving denominator. **The gate must normalise by answering-engine count, or require a minimum
   answering-engine count before it is allowed to deny.** This makes Story 5.2's engine-attrition
   alert load-bearing rather than hygiene: without it, attrition silently walks the operating point.

**Story 4.1 — `SearxngIssuerProbe`.** New service in SentinelCollector reusing the existing
`ISearxngClient` (`SentinelCollector/src/Services/SearxngClient.cs`). Returns the three-way verdict
plus, on strong-positive, the harvested exchange:symbol from the quote URL.

**Story 4.2 — wire at the paid boundary on BOTH legs.** Sentinel V2-direct
(`DeterministicResolver.TryGeminiResolveAsync`) and the SecMaster leg
(`IdentifierConfirmationService`). A change that covers only Sentinel misses ~69% of dispatches and
does not satisfy this epic.
- **Not inline in the confirmation timeout.** SecMaster's `GeminiConfirmTimeout` is 4s and 37 of
  106 dispatches already hit it (`server.py:705-710`); a ~1s probe consumes a quarter of that
  budget. Place the probe where the surface is known ahead of confirmation, or raise the timeout
  with measured justification — do not silently spend the existing budget.
- **No feature flag.** Per house rule, no default-OFF flags. Validation happened offline in Epic 3;
  this ships enforcing.
- Probe failure (SearXNG unreachable, zero engines answered) → **escalate to paid**, not deny. A
  dependency outage must not silently stop resolution; that failure mode gets Story 5.2's alert.

**Story 4.3 — D-entries and guard tests.** **All three touched services carry a card**:
SentinelCollector, SecMaster, and — since `ddbe014f` (#942), the same commit §8 records as Epic 1's
landing — `gemini-resolver-mcp/AGENT_README.md`. That card's DECISIONS block already holds **D-1
`fail-closed-durable-cap`, D-2 `accepted-not-merely-unrejected`, D-3
`rejected-call-releases-its-slot`, D-4 `wrong-ticker-worse-than-the-bill`**; D-2 **is** the "later
fix round" the Story 1.2 addendum describes, written up where its guard lives. Read all three
DECISIONS blocks before designing this story — CLAUDE.md's INTENT_FIDELITY CONFLICT rule and §5's
STOP_ON_OBSTACLE both key on knowing which entries exist, and Story 5.3 modifies this same service
again. Add D-entries in the touched services' DECISIONS blocks per CLAUDE.md INTENT_FIDELITY
MECHANICS, with `INTENT(D-n):` comments at each guard site.
- One entry for the private-marker hard reject (INTENT: never harvest a ticker from a pre-IPO or
  tokenized listing).
- One for the paid-escalation precondition (INTENT: paid call earned only by an unresolved middle
  band).
- Each needs a GUARD_TEST that constructs the violation and asserts refusal **at the boundary
  through the real flow**, RED if the guard is deleted.

**AC 4**

> **Gate on credits before evaluating AC 1 and AC 2.** Both are trivially satisfiable by the
> outage: with credits depleted and Story 1.1 landed, 429s release their slots, so
> `live_calls_24h` sits near zero and yield is undefined — a green reading that proves nothing.
> Neither may be evaluated until credits are restored **and** `gemini_resolver_resolving == 1` has
> held for ≥1h. If Epic 4 completes while credits are still dead, it ships marked
> **PENDING-VERIFICATION** with AC 1/2 explicitly unmeasured, and STATE.md says so.

1. `gemini_resolver_live_calls_24h` falls below the 1,500 cap for 7 consecutive days without the
   cap being lowered.
2. [CACHE]-equivalent resolution yield among *paid* calls rises above 27.0% (report the new figure
   against a stated window; a rate, not a single reading).
3. A replay of the Epic 2 frozen sample through the shipped code path denies the paid call for
   `Donald Trump`, `Sri Lanka`, `the Palio di Siena`, `the Head of Quantitative Strategy`,
   `the Company's Amended and Restated 2020 Incentive Award Plan` — and **still escalates** at
   least one genuinely-hard real issuer from the labelled set.
4. Anthropic-shaped input (private markers present) yields no ticker and no paid call; test asserts
   `ANTHROPIC-USD` is never returned.
5. Both legs covered — a test exercises the SecMaster path, not only Sentinel.
6. Probe-unreachable → paid escalation, proven by test.
7. `bash {Project}/.devcontainer/compile.sh` for every modified project: **0 errors, 0 warnings, all
   tests pass**, full output captured.
8. D-entries + `INTENT(D-n):` comments + guard tests land in the **same** PR as the guard code.

---

### Epic 5 — Close the observability gaps (parallelisable)

**Story 5.1 — `SentinelCandidateSurfaceFilterCollapsed`.** Absorbed verbatim from the
type-classifier spec §10 item 2: nothing watches
`sentinel_candidate_surface_filtered_total` today (re-verified: zero hits across `deployment/`).
Predicate is a **RATIO against dispatch volume, never a raw rate** — a bare `rate(...) == 0` fires
on every quiet ingress period. Mirror `NewsSignalClassifierFailingHard`
(`deployment/artifacts/monitoring/alerts/sentinel.yml:305-313`): same shape, same `clamp_min`
denominator, same volume floor.

**Story 5.2 — SearXNG engine-attrition alert.** Fire when the mean answering-engine count per probe
falls below a floor, or probe-derived denials spike. Without this, upstream engines throttling
degrades the gate to "deny everything" silently.

**Derive the floor from a STATED window, never a remembered one.** Attrition is a property of the
moment you drew, so a floor carried over from a window nobody can replay is a floor nobody can
check. The only replayable baseline is the committed draw (2026-08-11T13:19Z, n=11 [PROBE]): brave
unresponsive 11/11, wikidata 6/11, duckduckgo 3/11; answering-engine count 3 on eight probes, 2 on
two, 1 on one — mean 2.6. The earlier unretained window's duckduckgo figure differs by ~3x on the
same engine (10 of 11 unresponsive against 3 of 11), and the direction is the dangerous one: a
floor derived from that window assumes duckduckgo is essentially always absent, so it sits **below**
the band actually observed and would not fire on a real collapse. State the window the floor came
from in the rule's annotation, and re-derive it when the probe's own attrition series exists.

**Story 5.3 — a real spend signal.** `gemini_resolver_total_cost_usd_24h` is **token cost only**
($0.464884 for 1,500 calls [HEALTH]). Grounded-search request charges appear in no gauge.

**Scope this honestly — we cannot read Google's meter.** What is buildable here is a count of *our
own* grounded requests against the known free-tier boundary and published overage price, i.e. a
modelled projection. The authoritative balance lives only in the AI Studio billing console, and
nothing in this plan changes that. Ship the projection, label it a projection, and alert on
projected depletion **before** it happens. Per CLAUDE.md INTENT_FIDELITY: never ship a
`calls>0 AND cost=$0` check as the burn alert — that is a corpse-detector, and `resolving` at
`server.py:754` is already exactly that shape (it is a correct *death* detector; it is not a burn
alert and must not be treated as one).

**AC 5**
1. Each alert has a test in `deployment/tests/alerts/` (`gemini-resolver_test.yml` is the in-repo
   worked example) that goes **rc≠0 when the alert is broken**, not merely green when it is present.
   Runner: `bash deployment/tests/alerts/run.sh`. Prove the test has teeth by breaking the rule
   deliberately once and observing rc≠0 — a suite that has never gone red has not been tested.
2. Alerts deploy via `ansible-playbook playbooks/deploy.yml --tags alerting --skip-tags always`
   **then** `sudo nerdctl restart grafana` — alerting provisioning is startup-loaded and grafana
   lives in the separate OTEL stack.
3. Each new rule is confirmed present via the Grafana rule API and its PromQL returns data.

---

## §5 CONSTRAINTS FOR AUTONOMOUS EXECUTION (HARD_STOP)

- **`psql` is SELECT-only.** No `INSERT`/`UPDATE`/`DELETE`/`ALTER` against prod to "fix" state, ever
  — including to clean junk instruments. Fix in code. This is restated verbatim because an agent
  that already had the rule still ALTERed prod.
- **No interactive commands** in any dispatch: no `-it`, no `--ask-vault-pass`, no y/n prompts.
  `sudo` is passwordless.
- **Never edit** `/opt/ai-inference/**` directly, and never edit prompts on the host or in a
  container — the repo path is the source and deploy overwrites host edits.
- **Verify before commit**: `bash {Project}/.devcontainer/compile.sh` (note `bash` — the scripts are
  100644 in git). Capture **full output and the real `$?`**; never `| grep | tail`, which hides
  Permission-denied and swallows the exit code.
- **Never `git push` without a full test run**; the push guard enforces it. Never chain commit+push
  in one bash invocation.
- **Deploy scoped only.** Bare `--tags <x>` is an unconditional full-stack restart including a
  ~4min vLLM GPU reload.
- **SearXNG is a shared, already-degraded dependency.** Throttle every harness. Do not parallelise
  probes.
- **STOP_ON_OBSTACLE**: if any story requires contradicting the §1 DO-NOT-BUILD or a named D-entry,
  stop and report to the user. Do not route around it, do not obey a stale entry.

---

## §6 ROLLBACK

- Epic 1: revert the commit. Accounting returns to committing every dispatch — the pre-plan
  behaviour, which over-counts but never under-counts the cap.
- Epic 4: revert the wiring commit. Both legs return to calling Gemini directly; the probe service
  becomes dead code and can be deleted separately. **No data migration, no schema change, nothing
  to un-write** — this plan writes no new persistent state beyond log lines and one calibration
  file.
- Epic 5: alert rules are additive; delete the rule files and re-run the alerting deploy.
- Snapshot before Epic 4 deploys, per the repo's usual pre-deploy discipline.

---

## §7 WHAT WOULD FALSIFY THIS PLAN

State these plainly so a later reader can check rather than assume:

1. **If Epic 3's false-drop rate at any usable `LOW` exceeds 2% *and* `LOW` = 0 — deny only on a
   literally empty listed-score — is MEASURED to exceed it too**, the probe is not a safe deny
   signal, and Epic 4 collapses to the direct-resolve half only (spend falls, nothing is denied).
   The `LOW` = 0 fallback must be *measured*, never assumed to fail with the rest of the curve:
   AC 3.3 fixes the order, and this condition falsifies the plan only once that fallback has been
   read off.
2. **If the leg split in Epic 2 contradicts [PROM]'s 69/31**, the dispatch accounting is
   mis-measured and §0's attribution needs redoing before Epic 4 is designed.
3. **If the captured population shows the successful stream is *not* junk-heavy** — i.e. the [CACHE]
   27% yield is driven by genuinely hard real issuers rather than people and places — then the
   waste framing is wrong, the cap is simply too small for legitimate demand, and this plan should
   be replaced by a cap/budget conversation.
4. **If SearXNG engine attrition worsens to the point where fewer than ~2 engines answer reliably**,
   rung 2 is not viable at all and Epic 4 must not ship.

---

## §8 RESULTS (filled in during execution — leave a row empty until it is measured)

| epic | status | evidence |
|---|---|---|
| 1 | **DONE** | Merged `ddbe014f` (#942), deployed, verified. Stories 1.1 and 1.2 shipped together; the Story 1.2 supersession above (breaker leg on `resolving`) is part of this landing. |
| 2 | **IN PROGRESS** — capture running | Merged `afffa0c6` (#944), deployed. Capture window **OPEN from 2026-08-11T13:45:29Z**, proven on **both legs** by a probe returning rows — presence of data, not a config diff, per Story 2.1. Stories 2.2 (freeze + SHA-256 + row count) and 2.3 (revert the level) fall due when the ≥24h window closes; AC 2 is unevaluated until then. |
| 3 | Story 3.1 built | PR #946 — offline replay harness plus a frozen reference cache (`testdata/reference-cache/`, `MANIFEST.sha256` 11/11, drawn 2026-08-11T13:19:24Z). Its two design-binding findings are recorded in §2 (the table is not reproducible) and in Epic 4 (surface-level reject; normalise by answering-engine count). Story 3.2 awaits Epic 2's frozen sample. |
| 4 | not started | |
| 5 | not started | |
