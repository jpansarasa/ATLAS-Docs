# ATLAS WS3 Matrix-Realignment Completion — Supervisor Plan

**Status:** active execution driver (supersedes `docs/atlas-matrix-realignment-brief.md` + `~/.claude/plans/harmonic-wobbling-manatee.md`, which are now *reference*). Created 2026-06-02.

**Mission:** finish WS3 — turn the *rendering* matrix into the *disagreement* tool the brief's §1 thesis promises. Today the matrix renders (206 cells, 178 non-zero) but every cell is written by the legacy per-article Path-2 and carries **no `signal_identity_id`**, so FRED-vs-news disagreement is not magnitude-comparable in a cell. This plan builds the Option-A cutover (A), an independent backtesting/calibration harness (B), and the Phase-4 presentation (C).

## Autonomy envelope [USER-SET 2026-06-02]
**FULL AUTONOMY.** Supervisor dispatches all impl/review/fix/merge, **and deploys to prod and deletes legacy Path-2 autonomously.** NTFY (`atlas-claude-ask`) ONLY on: (1) a genuine blocker the supervisor can't resolve, (2) an architectural fork not anticipated here, (3) sub-project completion (A done / B done / C done) as a milestone. Do NOT NTFY-and-wait before deploy or before the Path-2 deletion. End every supervisor turn with `▶ continuing autonomously` or `🛑 DECISION NEEDED` per [[feedback_signal_autonomous_vs_pause]].

## Decomposition + order
```
decisions #2,#3 (surface-and-proceed) ──┐
                                         ▼
              A (Option-A cutover, spine) ──► C (Phase-4 presentation)
                                         ▲
B (backtesting harness) ─────────────────┘  independent; runs in PARALLEL; informs decisions #4,#5 + re-validates D1 weights
```
**Order: A → C sequential; B in parallel from the start.** A and B run concurrently in **separate worktrees** ([[feedback_parallel_agents_use_worktrees]]); B mostly adds a new `backtest/` area + a FinnhubCollector change + Python, with only read-only reuse of TE libs, so merge collisions with A are unlikely — but the supervisor sequences merges and rebases the trailing branch if TE files overlap.

---

## STANDARD PER-PR PROCEDURE (every task below follows this)
1. **Dispatch impl subagent** (background, **worktree-isolated**) with the task's goal + acceptance criteria + guardrails. It implements, runs `{Project}/.devcontainer/compile.sh` (0-err/0-warn + tests), opens a PR.
2. **Review:** supervisor **invokes the skill `/pr-review-toolkit:review-pr <N>`** — this sets the merge-gate marker AND runs the reviewer agents (code / silent-failure / tests as applicable). Marker is keyed to the PR head; it goes stale on every push, so re-invoke after fixes ([[project_pr_review_marker_staleness]], merge-guard lesson in STATE).
3. **Fix loop:** if reviews surface Critical/Important findings → dispatch a **fix subagent** → re-invoke `/pr-review-toolkit:review-pr <N>` on the new head. Repeat until 0 critical / 0 important.
4. **Merge** (squash) → **ff local main** (`git pull --ff-only`) → append a TIGHT one-liner to STATE only if it changes current-truth (merged PRs are git-log history, NOT STATE diary — [[feedback_state_md_not_session_diary]]).
5. **Deploy** where the task is runtime-affecting: scoped `nerdctl compose up -d <svc>` (NOT ansible full restart — it cycles the GPU + starts alert-service); **alert-service STAYS DOWN**; verify the running digest via `inspect .Created` ([[ansible_deploy_stale_image]]); smoke-check.

### Subagent dispatch guardrails (put in every brief)
- **Worktree isolation** for any agents that may run concurrently.
- **No interactive commands** (`-it`, `--ask-vault-pass`, y/n prompts); sudo is passwordless ([[feedback_background_dispatches_no_interactive]]).
- **DB: SELECT only**, never DML; schema changes via **EF migrations only** (`dotnet ef migrations add` — never hand-author migration files) ([[feedback_agent_no_raw_db_fixes]], CLAUDE.md MIGRATIONS).
- **Do NOT touch STATE.md / this plan / CLAUDE.md** (supervisor-owned); restore tracked files cleanly, don't stash-leak.
- **Long waits:** foreground blocking poll loops ≤9 min, never Monitor/PID-poll ([[feedback_agent_long_wait_pattern]]).
- Selective `git add -- <paths>`; never `-A`/`-u`/`.`.

---

## SUB-PROJECT A — Option-A cutover (the spine)
Goal: TE scores **every** source from `macro_observations` via a new `ObservationCellProjector`; cells become keyed by `signal_identity_id` + `source_collector`; legacy Path-2 is deleted. Reference: harmonic plan Part 2 + Part 4 (Phases 2–3).

### A1 — `ObservationCellProjector` in shadow mode
**Build:** a TE service `ThresholdEngine/src/Services/ObservationCellProjector.cs`, sibling of `CellProjector`, that reads `macro_observations`, groups by `(signal_identity_id, sector_code)`, alpha-normalizes the contributions (`Σoᵢ/n`), applies `sourceTrust × freshness × temporal × confidence`, clamps to ±3, and writes cells. **Reuse** `CellProjector.CalculateFreshness` (static, `CellProjector.cs:60`) + `GetTemporalMultiplier(TemporalType)` (`:90`). Relocate the per-source-collector trust table from `SentinelCollector/src/Semantic/SourceTrustConfig.cs` into TE. Idempotency tuple: `(signal_identity_id, sector_code, evaluated_at, source_collector)`.
- **Shadow:** behind a config flag `Matrix__ObservationProjectorMode = shadow|off|authoritative` (default **shadow**). In shadow it writes projector cells **tagged/distinguishable** from legacy cells (e.g. a `source_collector`-bearing row) WITHOUT removing or overwriting the legacy `MatrixCellSentinelWorker` output, so the two are comparable side-by-side.
- **First, confirm the pattern↔signal-identity linkage** the projector keys on: `PatternConfiguration` has no `SignalIdentityId` C# field; the mapping was reconciled into the pattern JSON / `SignalIdentityEntity` catalog in #584. A1 establishes exactly where the projector reads `signal_identity_id` from and wires it; if `PatternConfiguration` needs the field surfaced, add it.
- **Also characterize `macro_observations` coverage** (today: ~79 obs / 12 signals) and report it — sparse input ⇒ sparse projector cells; if FRED/OFR dual-write coverage is thin, that's a finding to NTFY (it bounds the cutover's yield).
- **Acceptance:**
  1. `compile.sh` 0-err / 0-warn; all TE unit + integration tests pass; new unit tests for the projector (alpha-normalization, trust×decay×temporal math, ±3 clamp, idempotency-tuple dedup) pass.
  2. Deployed in **shadow**; projector emits cells from `macro_observations` (DB: shadow cells with non-null `signal_identity_id` > 0).
  3. **Legacy path provably untouched** — legacy cell count/values unchanged after deploy (the authoritative matrix is unaffected).
  4. Coverage report on `macro_observations` produced (obs count, distinct signals, distinct sources, date span).

### A2 — Shadow validation + surface decisions #2/#3
**Run** shadow over live traffic for a soak window; **compare** projector cells vs legacy cells for the same `(signal, sector, day)`.
- **Acceptance (the gate to A3):**
  1. On overlapping `(signal, sector, day)` cells: **sign agreement ≥ 90%**, and **median |Δcell| ≤ 0.5** on the ±3 scale, over **≥ 50** overlapping cells (if < 50 overlaps exist because `macro_observations` is sparse, NTFY that as a coverage blocker rather than forcing a weak gate).
  2. A committed comparison report (`docs/benchmarks/ws3-cutover-shadow/REPORT.md`) with the per-(signal,sector) sign/magnitude deltas and the discrepancy diagnosis for any sign-flips.
  3. **Decisions #2 + #3 surfaced** via NTFY with the recommended defaults (below); supervisor proceeds on defaults if no redirect by the time A3 is ready.

### A3 — Cutover: flip authoritative + delete Path-2
**Flip** `Matrix__ObservationProjectorMode = authoritative`; **delete** the legacy path: `SignalMagnitudeCalculator`'s weight/scoring math (KEEP its polarity classifier — that's classification, emitted as raw observation metadata), the `MatrixUpdateStream` gRPC bridge (`Events/src/Events/Protos/matrix_updates.proto`), `ThresholdEngine/src/Workers/MatrixCellSentinelWorker.cs`, and the two now-dead collector configs. **Backfill** historical cells by re-projecting `macro_observations`.
- **Acceptance:**
  1. `compile.sh` 0-err / 0-warn; all tests green; the proto/worker/calculator deletions don't break the build (dangling refs removed).
  2. **Deployed to prod** (full autonomy); DB verifies: `matrix_cells` with `signal_identity_id IS NOT NULL` > 0 AND `count(DISTINCT source_collector) ≥ 2`; **a signal observed by both FRED and Sentinel has both contributions coexisting in one cell**.
  3. **Zero** `thresholdengine_matrix_sentinel_*` / gRPC-bridge traffic post-cutover (Prometheus, reset-safe `increase`).
  4. Path-2 source files gone from the tree; flag remains for reversibility (revert = flip flag + git-revert + redeploy).
  5. Milestone NTFY: A complete, with the keyed-cell + disagreement evidence.

---

## SUB-PROJECT B — Backtesting / calibration harness (parallel, independent)
Goal: build the harness from the merged design at `docs/atlas-matrix-backtesting-spec.md`. Runs offline; **no dependency on A**. New `backtest/` area.

### B1 — Free-source 12-ETF adjusted-daily history  [RE-SOURCED 2026-06-02]
Finnhub `/stock/candle` is premium-403 and paid candle data isn't justified for an offline backtest (PR #590 closed). Instead, in the offline `backtest/` Python harness (venv), fetch the 12 SPDR sector ETFs (XLE XLB XLI XLY XLP XLV XLF XLK XLC XLU XLRE + SPY) daily **adjusted** close from a FREE source (**stooq** preferred, no key; `yfinance` fallback) to inception — decoupled from any production collector / paid API. Output = the ETF price dataset Component-2 consumes (`backtest/data/`).
- **Acceptance:** all 12 symbols fetched w/ adjusted close; per-symbol row counts + date spans verified (short-history XLRE/XLC flagged); a NO-NETWORK unit test on a canned fixture; venv-only (never system Python).

### B2 — C# signal-replay (Component 1) + the drift-check gate
`backtest/` tool reusing `HistoricalPatternEvaluationContext`, `RoslynExpressionCompiler`, `PatternConfigurationLoader` (all verified present). Walk monthly eval-dates (~1998→present), replay each pattern's `signalExpression` as-of, emit the signal-value dataset (CSV/Parquet) per `(pattern_id, signal_identity, eval_date, signal_value)`; null + `missing_data_reason` for as-of gaps (don't crash).
- **Acceptance:** signal-value dataset emitted for all `signalExpression` patterns (incl. currently-disabled where FRED series exist); **drift-check passes** — for a recent date where live cells exist, replayed `signal_value` == live cell ÷ weight ÷ known attenuators within float tolerance (the spec's non-negotiable gate; if it fails, the backtest is void); synthetic + drift-check unit tests pass.

### B3 — ETF relative-forward-returns dataset (Component 2)
Compute `rel_ret(s,t,h) = fwd_ret(ETF_s,t,h) − fwd_ret(SPY,t,h)` for `h ∈ {1,3,6}` months from the B1 adjusted closes.
- **Acceptance:** relative-returns table emitted aligned to B2 eval-dates; per-sector usable date ranges + observation counts recorded; short-history sectors (XLRE, XLC) flagged.

### B4 — Python analysis harness (Component 3)
Three modules in a `backtest/` Python venv (never system Python — [[feedback_always_venv]]): **Validation** (Spearman IC heatmaps per horizon, hit-rate, lead/lag peak-horizon), **Calibration** (per-(signal,sector) suggested weight on the 7-level grid + confidence flag + diff vs current authored weight; beta-sign must agree with IC-sign; in/out-of-sample split), **Regime/disagreement** (FRED-driven regime timeline vs NBER recessions + known drawdowns; short-window disagreement check, labeled small-sample).
- **Acceptance:** the three reports + figures + a calibrated-weights proposal JSON generated; **synthetic-data IC tests pass** (injected-correlation series → expected IC sign/magnitude; independent series → IC≈0); nothing auto-applied (research artifacts only); **B milestone NTFY** with the IC summary, the D1-weights validation verdict, and the backtest **evidence for decisions #4/#5** (`cu-au-ratio`, `baltic-freight-recession`, `cyclical-gdp-share-declining` — do they show historical power?).

---

## SUB-PROJECT C — Phase-4 presentation (depends on A3)
Goal: the sector×signal heatmap + magnitude-comparable disagreement view. Use the `grafana-dashboard` skill; deploy via `ansible-playbook playbooks/deploy.yml --tags dashboards` (grafana auto-reloads). **Gated on A3** (cells must be `signal_identity_id`-keyed).

### C1 — Sector×signal heatmap
New dashboard: rows = `signal_identity_id`, cols = `sector_code`, signed ±3 color from latest `matrix_cells`. Postgres datasource `bf4txnr3gnjswd`. (`atlas-matrix-primary` = sector×*phase* stays as-is.)
- **Acceptance:** heatmap renders real signed values from keyed `matrix_cells`; deployed via `--tags dashboards`; verified rendering (panel returns data, not "no data").

### C2 — Disagreement overlay + re-point + pipeline-dashboard repurpose
In-cell FRED-origin vs Sentinel-origin overlay on one ±3 axis for a selected `$signal_identity`; re-point `atlas-disagreement` per-cell panels from `pattern_id` → `signal_identity_id` grouping; repurpose `sentinel-matrix-pipeline.json` (its gRPC-bridge panels are dead post-A3) to `ObservationCellProjector` throughput + `macro_observations` write/skip.
- **Acceptance:** overlay shows FRED vs news on one ±3 scale for a both-observed signal and flags sign divergence; `sentinel-matrix-pipeline` shows projector metrics (no dead gRPC panels); deployed + verified; **C milestone NTFY**.

---

## OPEN-DECISION HANDLING
- **#2 macro-score role** (surface in A2). **Recommended default: derive-from-sector-scores** — the macro score = a weighted aggregate of the 11 sector scores. Rationale: the 8 categories that fed the old standalone macro score are deleted; deriving keeps one coherent source of truth and authors no new weights.
- **#3 regime granularity** (surface in A2). **Recommended default: per-sector 11 regimes as primary** (matches the sector×signal matrix; `SectorRegimeClassifier` already bands per-sector), **retain the existing global 6-state untouched** for continuity. (Effectively "both", no new classifier work beyond keeping both alive.)
- **#4 borderline CUTs** (`cu-au-ratio`, `baltic-freight-recession`) + **#5 `cyclical-gdp-share-declining`**: **defer to B4 evidence** — enable+realign if the backtest shows historical IC, drop if it doesn't. Surface with the B milestone.

## WHOLE-PROGRAM ACCEPTANCE (WS3 done)
1. `matrix_cells` keyed by `signal_identity_id`, multiple `source_collector`s per cell; legacy Path-2 deleted.
2. FRED-vs-news disagreement is magnitude-comparable in a single cell and visible on the sector×signal + disagreement dashboards.
3. Backtest harness produces IC validation + a calibrated-weights proposal + decisions-#4/#5 evidence (research artifacts, nothing auto-applied).
4. main is clean and current; no stray worktrees; STATE reflects verified truth.

## CRITICAL FILES (verified 2026-06-02)
- A: `ThresholdEngine/src/Services/CellProjector.cs` (reuse `CalculateFreshness`/`GetTemporalMultiplier`), `ThresholdEngine/src/Data/Entities/MatrixCellEntity.cs` (`SignalIdentityId`/`SourceCollector` exist, #585), `ThresholdEngine/src/Endpoints/MacroObservationEndpoints.cs` (read access), `SentinelCollector/src/Semantic/SourceTrustConfig.cs` (relocate→TE), `SignalMagnitudeCalculator.cs` + `Events/src/Events/Protos/matrix_updates.proto` + `ThresholdEngine/src/Workers/MatrixCellSentinelWorker.cs` (delete in A3), `MacroSubstrate` (macro_observations entity/contract).
- B: `ThresholdEngine/src/Entities/HistoricalPatternEvaluationContext.cs`, `ThresholdEngine/src/Compilation/RoslynExpressionCompiler.cs`, `ThresholdEngine/src/Configuration/PatternConfigurationLoader.cs`, `FinnhubCollector/src/Api/FinnhubApiClient.cs` (+ `Models/Candle.cs`), `docs/atlas-matrix-backtesting-spec.md`.
- C: `deployment/artifacts/monitoring/dashboards/{atlas-matrix-primary,atlas-disagreement,sentinel-matrix-pipeline}.json`; Postgres datasource `bf4txnr3gnjswd`.
