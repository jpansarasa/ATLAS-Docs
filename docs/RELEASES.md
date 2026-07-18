# ATLAS Releases

Tagged phase-completion snapshots. Pull the tag to recover the full set of
plans / recon docs / iteration history that was in `docs/plans/` at that
moment:

    git fetch origin --tags
    git checkout <tag> -- docs/plans/

Or inspect a single file at a tag without checkout:

    git show <tag>:docs/plans/<filename>

## Tags

### `secmaster-assetclass-taxonomy-gigo-guards-done` (@ 06e8e44b)
**asset_class canonicalization + register-ingress GIGO guards + GeminiFallback junk quarantine** (2026-07-18). A morning documentation review surfaced 10 mergeable PRs; reviewed → fixed → merged → deployed as one train. Two through-lines: **(1)** one canonical spelling per `asset_class`, enforced at every write boundary; **(2)** GIGO — reject junk at the SOURCE (extraction / register ingress) instead of backfill-cleaning downstream. The per-PR review caught **two bugs that would have shipped**: #875's quarantine migration was a **silent no-op** (lowercase `WHERE asset_class IN ('equity','etf')` vs #865's canonicalization that runs first — Postgres `=` is case-sensitive → 0 rows), and #873's new allowlist would have **rejected legitimate AlphaVantage FX** registrations.
- **asset_class canonical-at-write (D-5)** — **#865** case-fold (`equity→Equity`, `etf→ETF`, `commodity→Commodity`) at the `SaveChanges` persist choke + **reject blank/null LOUDLY** (a NOT-NULL-but-empty class is invisible to every `asset_class='X'` consumer) + `NormalizeAssetClassCase` migration + drift counter `secmaster_asset_class_canonicalized_total`; **#866** dropped SentinelCollector's `.ToLowerInvariant()` on the register payload (fix at the source, not the SecMaster receiver); **#868** CLEAR-synonym merge `fx→Currency`, `Interest Rates→Rate` (+ `ConsolidateAssetClassTaxonomy`, each mapping verified against the actual rows); **#876** `treasury→Rate` (+ `AddTreasuryAssetClassSynonym`).
- **register-ingress GIGO guards** — **#873 (D-4)** flipped `RegistrationService.EvaluateGuard` from a 2-item Economic **blocklist** to an equity-shaped **allowlist** `{Equity,ETF,Index,Currency,Crypto,Commodity}` (closed-by-default: any macro-ish label a future model invents is rejected without a code change), reason `macro_class_from_untrusted`; review found this would reject legit AlphaVantage `Forex`, fixed **at the source** (AV `GetCategory(Forex)→"Currency"`, the canonical allowlisted class) rather than widening the gate (which would reopen the fx-pollution vector D-4 blocks). **#874 (D-12)** Gemini confirm-before-self-seed: an unconfirmed subject or zero subject-token overlap refuses the self-seed.
- **GeminiFallback junk quarantine (GIGO cleanup)** — **#872** deactivated fred_series junk (surface whose symbol doesn't resolve in FRED); **#875** deactivated 82 equity/etf wrong-ticker rows (74 Equity + 8 ETF, each corroborated per-row via Finnhub/OpenFIGI — a news co-mention or hallucination stapled to a real ticker; 20 legit issuers / single-country ETFs KEPT).
- **other** — **#867 (D-1)** FRED frequency normalize-at-source (qualified strings like `"Weekly, Ending Saturday"` silently defaulted to Monthly) + `FixSeriesConfigFrequency` + GUARD-citation card fix; **#869** serialized the meter-cardinality tests (shared-meter flake).

Deployed + COMPLETION_GATE-verified (2026-07-18): 4 images built `--no-cache` (freshness proven by grepping new-only symbols — `QuarantineGeminiEquityEtfJunk`, `GeminiSelfSeedGate`, `FixSeriesConfigFrequency` — inside the runtime DLLs), scoped ansible deploy (`ok=117 changed=12 failed=0`), all 6 migrations applied. **#875 quarantine verified LIVE: exactly 82 rows flipped to `is_active=false` in Title-Case (the pre-fix bug would have flipped 0).** Smoke all-green: 4/4 services + vllm Healthy (HTTP 200), **0 Loki Error/Fatal in 10 min** (selector validated against Warning startup banners), `resolve_batch(UNRATE,SOFR,DGS10)` 3/3 + `search_instruments(Apple)` returning canonical-cased `Equity`. Injection note: the #875 fix subagent spuriously no-op'd on injected adversarial content (the recurring pattern) — the fix was completed by the supervisor directly; anti-injection preambles remain standard on dispatches.

### `loki-warnings-autofix-remediation-done` (@ 73a7a25f)
**Loki warnings/errors cleanup + autofix warning-flood escalation** (2026-07-07). Triggered by "look at 24h Loki errors/warnings — autofix didn't fire; a well-running system produces none, so something's wrong or miscategorized." Diagnosis (24h): 0 Fatal; 9 Error (a transient Finnhub-503 burst, retried + recovered); ~2,693 Warning dominated by one cascade + log-spam. **Root cascade:** `llama-cpu-rag` is healthy-but-CPU-slow, so SecMaster's RAG circuit breaker was counting our OWN 25s-budget `TaskCanceledException`s as faults → flapped open 60s → RAG degraded → load spilled to the Gemini frontier → daily-cap **429** flood. Autofix was correctly idle: every alert in 24h was `severity=warning`, and `AutoFixChannel` gates on `[critical,error]`. The warning-flood detector existed (`loki-warning-rate.yaml`, Grafana-native) but was never wired to autofix. Fixed across 5 PRs (each: DESIGN-INTENT-stanza dispatch, review loop that found+fixed a real issue, guard tests, honors the observability scar-tissue rule = dedup-to-visible, never silent-Info):
- **#863 SecMaster** — breaker counts server faults only (dropped `.Or<TaskCanceledException>()`; our budget-cancels are bounded by the RAG CTS, not the breaker) + RAG budget 25→50s / min 12→18s + demote the redundant per-request circuit-open log (the once-per-open `onBreak` Warning stays visible) + Gemini 429 log deduped to once/hour (Interlocked-CAS). Review caught that the change **blinds** the existing `SecMasterRagCircuitChronicallyOpen` alert for a hung-but-alive backend → added compensating `SecMasterRagBudgetTimeoutsSustained` (`rag_degraded{reason="timeout"}>0.05/s for 30m`, promtool-validated).
- **#861 SentinelCollector** — dedup the high-volume Warnings (Gemini 429, spaCy transport, ContentNormalizer) to once-per-window via the ThresholdEngine CAS idiom, and **add the missing `sentinel_normalizer_outcome_total` counter** for the ~700/day "no visible text" path (was unmetered — a real observability gap, not just noise).
- **#864 SentinelCollector (D-11)** — reject non-tradeable **media-outlet** surfaces at the D-5/D-6 ingress filter (GIGO — clean at source). Review + a live SecMaster-catalog check **dropped the originally-proposed MarketIndex class**: US indices resolve to real tracked instruments (`S&P 500`→SP500, `DJIA`, `Nasdaq Composite`→IXIC, `Russell 2000`→RUT) and even bare "Nifty" fuzzy-collides with a real ticker — rejecting them would silently drop legitimate resolutions (the exact D-5 PRECOND violation). Media-outlet list verified non-tradeable (Axios/Politico/TechCrunch/The Verge all absent from the catalog).
- **#862 AlertService + monitoring** — the user's ask: a SUSTAINED per-service warning-FLOOD now self-escalates. New Loki-ruler rule `ServiceLogWarningsElevated` (`>500/15m for 30m`) fires at `severity=error` → Alertmanager `error` route → AlertService `error→[ntfy,autofix]` → autofix. Individual warnings still never auto-fix (honors #757); only the escalated flood does. New AlertService D-1 narrowly supersedes #757's absolute "warning never reaches autofix". Fixed the `EnabledSeverities` doc-drift + the stale `sentinel.yml` "KEPT DOWN" comment.
- **#860 FredCollector** — demote expected-retry / no-result logs Warn→Info (#784 pattern); terminals, circuit-breaker callbacks, startup banners, and the trace-less worker-loop backstops stay at Warn (scar-tissue).

Deployed + COMPLETION_GATE-verified (2026-07-07): 4 images built `--no-cache` (verified to contain the code), scoped ansible deploy (ok=51 changed=9 failed=0), alert-service recreated (excluded from `scoped_services`), alertmanager restarted (error route live), Prometheus HUP'd + Loki ruler auto-polled the new rules. Smoke all-green: 11/11 containers Up, **0 Loki errors/10min**, SecMaster Healthy, **error→autofix verified via synthetic alert** (`{"queued":1}` → queue file → removed), **429 flood stopped** (0/8min post-restart vs 277/hr), breaker not flapping (0 opens/90m), warnings falling. Deferred (non-blocking): re-check the sustained warning-volume drop + `sentinel_normalizer_outcome_total` populating after ~30-60min, and recalibrate the `ServiceLogWarningsElevated` 500/15m threshold against the new (lower) baseline.

### `sentinel-review-queue-drain-done` (@ 0a2f4a47)
**Sentinel review-queue drain — restored + 321K backlog drained** (2026-07-05). The `extracted_observations` review queue had grown UNBOUNDED to ~321K Pending (+~1,650/day) with no drain: the V2 extraction cutover writes everything Pending and the downstream approval path was never wired. Root causes fixed across 3 stages / 4 PRs (each stage: 3 review rounds, every guard test RED-verified):
- **Stage 1 (#850) — terminal disposition for un-resolvable rows.** New `ReviewStatus.AutoClosed` (digest-safe: kept as digest content, dropped from the human queue — NOT Rejected) + `NoResolutionSweepWorker` (batched, cursor-by-Id, grace 7d) + `POST /admin/review/close-noresolution`. Hardened for the backlog: fresh-scope/batch + `ChangeTracker.Clear` (memory-bounded), `CREATE INDEX CONCURRENTLY` (no deploy write-lock), OCE-safe (command-timeout can't crash the host), throttled, 3-fail→Error escalation, queue-depth gauge. On deploy the sweep drained **~234K aged-NoResolution rows → AutoClosed (Pending 321K→87K in ~5 min)**.
- **Stage 2 (#851) — recover the ~27% missed real instruments (D-8).** New `DeterministicResolver` exact catalog-symbol leg: an LLM-shortlisted candidate whose Symbol is an EXACT primary-symbol hit AND the OBSERVATION SUBJECT shares ≥1 name token (NEVER fuzzy) → authoritative resolve. Review caught + killed a CRITICAL — a tautological `Math.Max` name-overlap gate (`c.Name==hit.Name` for catalog candidates) that would resolve a merely co-mentioned company to the WRONG ticker; fixed to subject-overlap-only. D-8 = new earned exception, NOT a SecMaster D-2 supersession.
- **Stage 3 (#852) — restore the drain (D-9).** (3a) `res_conf` now reflects the RESOLVING leg (hybrid/Gemini/exact=1.0), not the LLM self-report — the value the auto-approve gate reads. (3b) `AutoApproveDrainWorker`: scheduled, non-dry-run, runs `AutoApprovePolicy` over Pending (SAME thresholds as the backfill endpoint), approving high-confidence Resolved rows DOWNSTREAM of the untouched V2 cutover (DD-4=b — no cutover reversal, D-9 not a supersession). Same Stage-1 hardening + also fixed the sibling backfill endpoint's live-loop O(n²) ChangeTracker bug.
- **Enable (#853)** — flag on after smoke; first pass approved **249 / rejected 43** (exact dry-run match), 0 errors.

Outcome (verified 2026-07-05): `review_status` **AutoClosed 234,510 · Pending 86,521 · Approved 55,115 · Rejected 39,585**. Drainer live; go-forward new extractions carry correct res_conf → auto-approve → queue won't regrow. **Deferred follow-up:** a depth-gauge ALERT — raw `sentinel_autoapprove_drain_queue_depth` is confounded by the ~86K stale-res_conf backlog that can't qualify (3a's res_conf fix is go-forward-only; backlog rows keep the old LLM self-report), so a stall alert needs a decision-rate / approvable-Pending design, not a raw-depth threshold. Plan recoverable: `git show sentinel-review-queue-drain-done:docs/superpowers/plans/2026-07-05-sentinel-review-queue-drain.md`.

**Follow-on — the "still 86.5K, some months old" catch (#854, #855).** After the above was tagged, review of the residual queue found the "epic complete" framing had masked a distinct BUG: `ExtractedObservation.ApplyReExtraction` overwrote the instrument fields on re-extraction but **never recomputed `resolution_state`**, so a re-resolution that cleared the instrument left the row stuck at the stale `Resolved` → **42,830** `Resolved`-with-null-instrument **mislabels** the NoResolution sweep couldn't reach (it requires `NoResolution`) and that can never auto-approve (no instrument). **#854** fixed the mechanism (recompute state on re-extract, mirroring the resolving callers) + a dry-run-defaulted `POST /admin/review/reconcile-resolution-state`; the reconcile relabeled 42,830 → NoResolution and the sweep AutoClosed the aged ones. Verified data-safe: these never fed the matrix (WS3 `ObservationCellProjector` reads `macro_observations`, not `extracted_observations`) and stay in the digest (`!= Rejected`). **#855** then filtered the review UI to `ActionablePending` = `Pending AND InstrumentId != null` — the human queue is for approving/rejecting RESOLUTIONS; NoResolution rows have nothing to action and are system-disposed by the sweep. **Final (verified 2026-07-05):** Pending 53,686 total, but `/ui/review` now shows only the **~558 actionable** (instrument-linked) rows — nothing months-old remains, and the root cause won't recur. Gotcha recorded: `/admin/review/*` endpoints bind camelCase `dryRun` (snake `dry_run` silently no-ops as dry-run). Deferred (non-blocking): dry-run cursor-convergence + NoResolution-instrument-linked tests, OCE-at-Error log demotion.

**Follow-on 2 — "558 is still 9 hours" → trust the state, kill the co-mentions (#856, #857).** The ~558 "actionable" rows were still untenable (~9h at a minute each). Inspecting the actual values showed neither population needed a human: ~31% were mundane-correct macro data (`S&P 500 +1.2%`, `unemployment 4.3%`) where the resolution **state** is the quality signal, not the confidence *number*; ~69% were `NoResolution` **co-mention garbage** (a ticker stapled to an unrelated subject — `Gilbert Cisneros → LLY`, `Banxico → EWW`). Since `extracted_observations` feed only the qualitative digest (not the matrix), borderline resolutions are low-stakes. **#856** changed the auto-approve gate to trust the state (`Rejected if extraction<0.5; else Approved if Resolved AND instrument; else Pending` — the `extraction≥0.9`/`res_conf≥0.8` floors were theater; **supersedes DD-5**) and filtered the review UI to `Resolved` instrument-linked rows. Result (verified): the drainer auto-approved the resolved backlog on startup (+217); **`/ui/review` now reads "Pending: 0 — No pending observations to review."** **#857** fixed the co-mention garbage at its source (**D-10**): both `ticker_in_quote` legs (the live `ExtractionProcessor` primary path + the re-extract adapter) attached a quoted ticker on exact-symbol-match WITHOUT checking it matched the subject; a shared `TickerInQuoteGuard.SubjectSharesInstrumentToken` predicate now requires `SharedTokenCount(subject, catalog Name) ≥ 1` (mirroring D-8) at both sites, refusing co-mentions (metered `comention_rejected`). Deployed + guard-verified (RED tests; 0 new NoResolution-with-instrument rows post-deploy); the existing ~378 co-mention rows self-clean as re-extraction reprocesses them. Deferred (non-blocking): fail-closed on a transient SecMaster lookup; the bare-ticker-subject false-negative edge (accepted D-8 tradeoff); an optional reconcile-reject for an immediate digest clean.

**Follow-on 3 — "fix the queue: I'll review ~10/day, not more" → method-aware review (#858, #859).** Driving the queue to 0 (Follow-on 2) removed review entirely; the owner wanted a *functional* bounded spot-check ("more than 10/day is a chore I won't do"). Live data killed the confidence-threshold approach — **94% of resolutions are `ticker_in_quote` at a flat 0.900**, no gradient to rank by. The real quality signal is the resolution **method**: exact-verified methods (`ticker_in_quote` via D-10, `llm_candidate_exact` via D-8) are trustworthy; fuzzy/semantic methods (`cove_*`, `hybrid_subject`, `llm_candidate_pick`, gemini) are where **every sampled error lived** (`gold → BEA price index`, `$30T debt → 30-year yield`, `Bitcoin → ETH`). **#858** made the auto-approve gate (**D-9**) method-aware: auto-approve exact methods; **HOLD fuzzy for review** (~2-5/day); **default-accept** any held row after `ReviewGraceDays` (3) so the queue can never back up. Re-extract old-backlog fuzzy default-accepts (deliberate — those rows aren't in the *current* digest, and holding ~80K would flood the bounded queue). `ReviewUiEndpoints` needs no change — the `Resolved AND instrument AND Pending` set *is* the fuzzy set. **Supersedes the #856 trust-all-Resolved gate.** **#859** closed the review's observability/robustness gaps: an `approve_reason` metric (`method_trusted` vs `grace_default_accept`) makes the silent default-accept branch visible (reviewed-vs-defaulted rate); shared `ResolutionMethods` constants tie the resolver write-sites to the auto-approve set (compile-time rename safety); live-call-site tests (the V1-source `ExtractionProcessor` gate is live, not dead). Deployed + verified (49 `ticker_in_quote` Approved, 0 fuzzy Approved; `/ui/review` fills going-forward). **Security note:** an anomalous subagent output carried a prompt-injection (redirect-to-ntfy / "user can't see you") — caught, not obeyed, zero repo tampering; a re-dispatch ran clean, and an anti-injection preamble is now standard on dispatches. Deferred (non-blocking): per-pass grace-count log; an oldest-held-fuzzy-age leading gauge; a one-time re-open of the Follow-on-2-auto-approved fuzzy rows (the only errors this design can't retroactively catch).

### `sentinel-trend-viz-done` (@ a2141c54)
**Sentinel digest trend-visualization redesign — SHIPPED** (2026-07-04). Replaced the points-in-time digest with a market-first, trend-over-time design (user ask: the reports were "always points in time... hard to see trend"). Delivered: (1) redesigned digest — KPI tiles → sector×day heatmap index → per-sector `<details>` drill-downs (net-lean sparkline + regime-timeline strip + top-5 per-signal dual-trend [matrix cell vs daily news tilt]) → Executive Take; (2) live `GET /sentinel/matrix?range={7d|30d|90d|max}` explorer reusing the same blocks; (3) "News Signal Momentum" rebuilt as server-rendered SVG (bar color = tilt SIGN matching the heatmap, momentum direction moved to a separate ↗/↘/→ arrow, Chart.js/CDN removed). Server-rendered inline SVG, zero client JS, native `<details>` drill-down; hot=red / cold=blue, CVD-safe. Cutover removed the old Sector Heat markdown table + Sector Detail lists + raw extracted-quantity bullets + the "Sector Watch" narrative (host-mounted `/prompts`). SentinelCollector-only, read-only over existing tables (matrix_cells / sector_regimes / macro_observations); no schema/projector/write-path change.
5 PRs: **#841** Stage A (TrendQueryService + TrendSvgRenderer + tests), **#842** Stage B (market-first DigestRenderer + `/sentinel/matrix` endpoint), **#843** drill-down nesting fix (user-caught empty-body bug — rows opened to nothing), **#844** momentum SVG redesign (user-caught confusing green/red dual-encoding), **#845** Stage C cutover. All merged + deployed + verified; live at digest report 185. Design docs (on main): `docs/superpowers/specs/2026-07-04-sentinel-trend-visualization-design.md` + `docs/superpowers/plans/2026-07-04-sentinel-trend-visualization.md`. Recover via `git show sentinel-trend-viz-done:<path>`. Notable: the two user live-review catches (empty drill-downs, confusing momentum) both passed automated tests+smoke — the fix PRs added behavior-level guard tests (nesting-inside-details, glyph-tracks-sign) that go RED on the actual regression.

### `okf-rag-spike-done` (@ d519fcf1, NEGATIVE RESULT — not merged)
**OKF / LLM-wiki RAG feasibility spike — NO-GO as a subsystem** (2026-07-02).
Investigated whether the Karpathy LLM-wiki / Google Open-Knowledge-Format pattern
(LLM curates a compounding folder of cross-linked markdown) should augment or replace
ATLAS RAG. Verdict: no. The wiki is squeezed out from both sides — grounded/structural
retrieval is done better, cheaper, and more context-efficiently by the existing
bge-m3 + pgvector RAG (the 7b arm truncated 8/14 ingests because whole-page YAML reads
exhaust its real 2048-token slot under `--parallel 4` — the context cost a targeted
vector retrieval avoids); and news alpha is fast-decaying and must NOT accumulate into a
compounding store (reinforcement bias) — it belongs in the numeric benchmark-staleness
pipeline (#729). The 32b "GO" only cleared quality bars by burning 32K context, which is
orthogonal to the small-model/context-economy goal. Structural finding worth keeping: the
video's "LLMs mangle markdown at scale" skepticism was refuted for both arms (0 invented
links / 28 ingests) — format was never the problem; fit was. Keepers: the adversarial
eval-gate harness (caught a rigged eval-set pre-inference) + the measured local 2048-token
slot reality. Baseline `pre-okf-rag-investigation` @ c558bb52 was the revert point; main
was never touched, so no revert was needed. Full artifacts recoverable:
`git checkout okf-rag-spike-done`.

### `intent-fidelity-done` (@ 84569aac)
**Intent-fidelity enforcement epic — COMPLETE** (shipped 2026-07-02).
D-entry decision records (AGENT_README `DECISIONS` blocks, atomic with
INTENT(D-n) comments + guards + RED-capable guard tests) for
SentinelCollector, ThresholdEngine, and SecMaster; enforcement hooks
(dispatch-guard BLOCK / decisions-injector ADVISE / plan-retirement ASK)
plus the testing-context outbound-boundary line; `intent-review` skill
wired into REVIEW_FIX_LOOP and the deploy skill; CLAUDE.md MECHANICS.
PRs #826–#830. Spec doc `docs/proposals/intent-fidelity-enforcement.md`
retired 2026-07-02 (human-confirmed through the plan-retirement-guard's
first ASK, after verifying all decisions migrated to the 13 D-entries +
CLAUDE.md MECHANICS + intent-review GUARD_TEST_CONTRACT). Recover the
spec via `git show intent-fidelity-done:docs/proposals/intent-fidelity-enforcement.md`.

### `digest-matrix-redesign-done` (@ b8dcd2a2)
**Matrix-first digest redesign — COMPLETE.** The Sentinel digest is now
organized by the signal x sector matrix instead of the 7-theme taxonomy:
Sector Heat strip + per-sector Detail blocks (net tilt, regime, top signals
cross-referenced with news momentum, cited articles), articles grounded to
sectors via their `:sig:` rows' `atlas_sector_code`, narrative prompt
rewritten (Executive Take / Sector Watch / Cross-Sector Signals /
Noteworthy One-Offs). ThresholdEngine `SectorRegimeProjectionWorker` now
publishes SectorScoreEvents so `sector_regimes` populates (first rows ever,
2026-06-12). ThemeClassifier/DigestTheme deleted; Grafana theme panel ->
sector panel. PRs #693 (matrix read + heat), #695 (sector grounding),
#697 (sector skeleton), #698 (deletion sweep), #694 (regime publisher).
Plan retired from `docs/plans/digest-matrix-redesign.md` — recover via
`git show digest-matrix-redesign-done:docs/plans/digest-matrix-redesign.md`.


### `pre-docs-consolidation-2026-05-26`
Anchor tag preserving the full `docs/plans/` tree as it existed before
consolidation. Includes every iteration-history plan, recon doc, and spike
note that's now removed from main. Use this tag to recover any of the
deleted Phase 2 / Phase 3 / recon documents.

### `dsl-poc-phase4-done` (@ 93fd94d0)
**Phase 4 GPU semantic verifier — PASS** at recalibrated >=80% Foundry
agreement gate. 80.64% via prompt iter-2-axis-C + IsSelfReferential
short-circuit on Qwen2.5-32B-Instruct-AWQ. Recalibration rationale in
`docs/plans/atlas-dsl-poc-phase4-gpu-semantic-verifier.md §11`.

### `dsl-poc-phase5-done` (@ ff72eb8f)
**Phase 5 matrix integration — DONE.** CPU CoD cutover live
(`Extraction__Backend=LlamaServerDsl` default); §17.4 path A
`cells>0` criterion met (2 cells from 1/10 articles via FOMC 29807).
4 review findings closed (CR-IMP-1 prune blast radius narrowed,
CR-IMP-2 timeout-override coverage, OBS-1 DSL adapter dashboard
panels, OBS-2 calculator-skip alert). 1333 unit + 6 integration
tests pass; CPU throughput lifted to 6–13 t/s. Pending Phase 6:
entity_resolution_exhausted=61 / gpu_verifier_none=28 filter drops;
audit provenance `rawContentId=null`; CPU throughput borderline for
synchronous extraction (~50% timeout on long-output articles at
900s budget). Sub-plan
`atlas-dsl-poc-phase5-matrix-integration.md` pruned from main per
PHASE_TAGS; recover via `git show dsl-poc-phase5-done:docs/plans/atlas-dsl-poc-phase5-matrix-integration.md`.

### `gpu-cod-roleflip-2026-06-09` (@ 41e649b4)
**GPU-CoD role-flip — DONE.** Sentinel extraction CoD moved from CPU
llama-server (qwen3-30b-a3b) to GPU vLLM (Qwen2.5-32B-AWQ, JSON-schema
output). ~25x throughput lift (~327 art/hr vs ~20); backlog drains ~20h.
Distributed: GPU handles CoD emission + CoVe verification; CPU handles
classifier + embeddings. Loop-guard added. Recall gate 0.79 (n=80).
PRs #640 #642 #643 #644 #645 #646.

## Docs consolidation 2026-06-11 (`docs/consolidation` branch)

`docs/` re-curated to "current documentation + active plans" (index:
`docs/README.md`). Retired to git history — recover any path via
`git show <removal-commit>^:<path>` (or the tags noted):

- `docs/benchmarks/cod-2026-05-17/` — DSL PoC Phase 1–5 benchmark tree
  (~4,500 files: corpora, prompts, results, training runs, scripts).
  Contained in tags `dsl-poc-phase4-done` / `dsl-poc-phase5-done`; the
  production parser/verifier mirror is `SentinelCollector/dsl-parser-mcp/dsl/`.
- `docs/benchmarks/cod-cove-acceptance/` — CoD->CoVe acceptance harness;
  the CoVe claim verifier it validated was removed in #647.
- `docs/plans/gpu-json-cod-rollout-2026-06-09.md` — rollout DONE; tag
  `gpu-cod-roleflip-2026-06-09` contains the plan.
- `docs/plans/cpu-tuning-throughput-2026-06-08.md` +
  `docs/plans/model-task-hardware-optimization-2026-06-08.md` — CPU-era
  throughput investigations, superseded by the GPU role-flip.
- `docs/plans/cod-truncation-experiment-plan.md` +
  `docs/superpowers/specs/2026-06-07-cod-truncation-value-experiment-design.md`
  — truncation experiment mooted by the role-flip throughput win.
- `docs/plans/symbol-identification-remediation.md` — absorbed: RAG
  similarity floor + `GeminiSymbolFallbackService` shipped.
- `docs/plans/secmaster-parallel-failthrough-resolver.md` — shipped as
  #619; design context lives in SecMaster's service docs.
- `docs/superpowers/` — shipped design specs/plans: digest
  article-grounded narrative #610, news-signal feed #613, news-signal
  decay model #615.
- `docs/FRED_DATA_RESEARCH.md` — Jan-2026 research; conclusions absorbed
  into `docs/FRED_SERIES_REFERENCE.md`.
- `docs/sentinel-extraction-pipeline.md` — described the pre-DSL
  six-stage pipeline as deployed 2026-05-16; superseded by
  `SentinelCollector/README.md` + `docs/ARCHITECTURE.md` after the DSL
  cutover and GPU role-flip.

## Phase summaries

### Phase 2 — llama.cpp llama-server CPU sibling (DONE 2026-05-21, PR #387 @ 25cb33fa)
llama-server on :11437 serving qwen3-30b-a3b via bind-mounted Ollama blob;
v1.1 GBNF wired through ollama's `format` parameter; runner `--backend`
flag for portability. Outcome: working CPU GBNF substrate, foundation for
Phase 3 schema iteration. Full plan at
`pre-docs-consolidation-2026-05-26:docs/plans/atlas-dsl-poc-phase2-llama-server.md`.

### Phase 3 — v2 schema + word-grounding arc (DONE 2026-05-24)
Multi-PR iteration: v2 baseline (#390) -> v2.1 no-offset (#392, #393) ->
ENT-ref cleanup (#394) -> v6 leaner prompt (#400) -> context-aware /
RAG / leaner-prompt sweep (#404–#412) -> v2.2 token-grounding shelved
(#425) -> v2.3 word-grounding (#426–#428) -> v15 compound chunked
(#429, #431, #432). Outcome: word-grounding v2.3.1 + chunked compound
v15 are the leading production candidates; T=0 A/B verdict
DESIGN-SOUND. Deterministic Python verifier shipped as
`docs/benchmarks/cod-2026-05-17/dsl/verifier_v2_3_1.py` (benchmark tree
retired from main 2026-06-11 — production mirror lives at
`SentinelCollector/dsl-parser-mcp/dsl/`; recover the original via
`git show dsl-poc-phase5-done:<path>`). Iteration
history at `pre-docs-consolidation-2026-05-26:docs/plans/`
(`atlas-dsl-poc-phase3-v2-schema.md`, `token-grounding-recon.md`,
`three-hypotheses-scoping.md`, `hyp-b-logits-processor-spike.md`).

### Phase 4 — GPU semantic verifier (DONE 2026-05-26, see `dsl-poc-phase4-done` tag)
Tag covers the milestone above. Canonical record (kept on main):
`docs/plans/atlas-dsl-poc-phase4-gpu-semantic-verifier.md`. The 4.7s SFT
sub-arc (`atlas-dsl-poc-phase4-7-sft-cove-verifier.md`) is now subsumed
by §11 of the canonical record; the full sub-plan is preserved at
`pre-docs-consolidation-2026-05-26:docs/plans/atlas-dsl-poc-phase4-7-sft-cove-verifier.md`.

### Phase 5 — Matrix integration (DONE 2026-05-28, see `dsl-poc-phase5-done` tag)
PR #510 merged at `ff72eb8f`. CPU CoD cutover live; §17.4 path A
`cells>0` met. Iteration history at
`dsl-poc-phase5-done:docs/plans/atlas-dsl-poc-phase5-matrix-integration.md`.
Parent plan `docs/plans/atlas-dsl-poc-plan.md` remains active for
Phase 6.

## Going forward

Per `CLAUDE.md ## PHASE_TAGS`: tag at phase completion
(`<workstream>-phase<N>-done`), add an entry to this file, and delete
phase-specific iteration-history docs from `docs/plans/`. Reference
docs (architecture, specs, runbooks, schemas, grammars) stay on main.

## Docs current-state rebuild (2026-06-11)

The documentation set was rewritten to describe the system **as implemented**
(code-grounded briefs, live config, live DB — see `docs/README.md` for the
curation policy). The following were retired from main in the same change;
recover any of them via `git show 5cf265f7:<path>` (the last commit that
contains them all):

- `docs/atlas-matrix-mvp-plan.md` — matrix MVP plan (superseded by `docs/MATRIX.md`)
- `docs/atlas-matrix-handoff-v2.md` — matrix handoff brief (superseded by `docs/MATRIX.md`)
- `docs/atlas-matrix-realignment-brief.md` — realignment brief (superseded by `docs/MATRIX.md`)
- `docs/atlas-matrix-backtesting-spec.md` — backtesting spec (the `backtest/` tooling it
  specified remains in-repo; its READMEs carry self-contained summaries)
- `docs/f4.6.4-prepass-rollout.md` — prepass rollout working doc (prepass is live;
  current behavior documented in `docs/ARCHITECTURE.md` §5 and `SentinelCollector/README.md`)
- `docs/sentinel-product-spec-v2.md` — vision-level product spec (current behavior in
  `docs/ARCHITECTURE.md` + `SentinelCollector/README.md`)
- `docs/plans/` (entire tree): `atlas-dsl-poc-plan.md`,
  `atlas-dsl-poc-phase4-gpu-semantic-verifier.md`, `atlas-ws3-completion-plan.md`,
  `sentinel-edge-realization.md`, `atlas-matrix/README.md`,
  `atlas-matrix/epic-6-consumption-surfaces.md` — all phases complete; outcomes
  recorded above and tagged per PHASE_TAGS
- `docs/research/dsl-prior-art.md` — research notes (DSL era closed)
- `docs/llm/ACCEPTANCE_CRITERIA.md` — LoRA-era ratified criteria; the pinned copy
  (`SentinelCollector/training/acceptance_criteria.json`, sha-locked) remains the
  machine-readable record
- `docs/grammars/` (cod-dsl v1–v2.3 GBNF): the production grammar mirror lives at
  `SentinelCollector/src/cod-prompts/cod-dsl-v2.3.gbnf` (the CPU rollback backend);
  earlier grammar versions are history
- `docs/README-TEMPLATE*.md` (7 files) — README templates; the canonical copies used by
  tooling live in `.claude/skills/readme-consistency/` (`TEMPLATE.md`,
  `TEMPLATE_CONFIG.md`, `TEMPLATE_SCRIPTS.md`)
