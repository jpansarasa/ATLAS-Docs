# backtest/ — offline matrix-backtesting research area

**This is offline research tooling, not a deployed service.** Nothing here runs
in `compose.yaml`, on the gRPC mesh, or under ansible. It runs on demand from a
developer machine to produce datasets the matrix-backtesting harness analyzes
(spec retired to git history: `git show 5cf265f7:docs/atlas-matrix-backtesting-spec.md`).

The pieces, in build order (per that spec):

- **B1** `fetch_etf_history.py` — 12 SPDR sector ETFs + SPY, adjusted daily close (Component 2's data dependency).
- **B2** `SignalReplay/` (C#) — replays production `signalExpression`s over history → `data/signal_values.csv` (Component 1).
- **B3** `relative_returns.py` — `rel_ret(s,t,h) = fwd_ret(ETF_s) − fwd_ret(SPY)` (Component 2 output).
- **B4** `analysis/` + `analyze_matrix.py` — the Python analysis harness (Component 3): IC validation, weight calibration, regime/disagreement.

## Source: Yahoo Finance (free, keyless) — why not stooq

The spec named stooq as the preferred free source with yfinance as the
acceptable fallback. As of this work, **stooq's keyless CSV download is gone**:
`https://stooq.com/q/d/l/?s=<sym>.us&i=d` now returns a "get your apikey"
captcha page instead of CSV, so it is no longer no-cost. We therefore use the
**Yahoo Finance v8 chart API** (`query1.finance.yahoo.com/v8/finance/chart`),
which is keyless, free, returns full history to inception, and ships a
Yahoo-computed split+dividend-adjusted close.

We hit the v8 JSON endpoint **directly** (via `curl_cffi` browser impersonation)
rather than through the `yfinance` package: yfinance 1.4.1's session wrapper
drops the TLS CA-bundle override this host needs and silently returns empty
frames. Going direct is also more transparent (we carry raw OHLCV alongside the
adjusted close) and not coupled to yfinance's churning internals.

The `adj_close` column is Yahoo's pre-computed `adjclose` series taken
**verbatim**: the split + dividend adjustment is performed upstream by Yahoo,
not by this script (no dividend backout, no split math here). The output carries
the adjusted series and raw OHLCV only — **no dividend/split event columns are
emitted** (see `OUTPUT_COLUMNS` in `fetch_etf_history.py`).

## Setup (venv only — never system Python)

```bash
python3 -m venv backtest/.venv
backtest/.venv/bin/pip install -r backtest/requirements.txt          # B1/B3 fetch + returns
backtest/.venv/bin/pip install -r backtest/requirements-analysis.txt # + B4 analysis (scipy, matplotlib)
```

## Fetch the dataset

```bash
backtest/.venv/bin/python backtest/fetch_etf_history.py        # writes CSV + Parquet
backtest/.venv/bin/python backtest/fetch_etf_history.py --format parquet
```

Output (`backtest/data/`, git-ignored — re-fetchable, not source):
`etf_daily_adjusted.{parquet,csv}`, one row per `(symbol, date)`:

| column | meaning |
|---|---|
| `date` | UTC trading date (ISO-8601) |
| `symbol` | ETF ticker (XLE…XLRE, SPY) |
| `sector` | ATLAS sector code; SPY → `MARKET` |
| `open` `high` `low` `close` | raw OHLC |
| `adj_close` | Yahoo's split + dividend adjusted close, **verbatim** — Component 2's return input |
| `volume` | daily volume |

`SPY` carries `sector = MARKET`; the 11 sector ETFs map 1:1 to the ATLAS sector
codes per the GICS SPDR correspondence (see `SECTOR_ETFS` in
`fetch_etf_history.py`). Short-history sectors are expected: **XLRE → 2015-10,
XLC → 2018-06**; the rest trade from ~1998–99.

## B4 — Python analysis harness (Component 3)

`analysis/` holds three modules over the three inputs (B2 `signal_values.csv`,
B3 `relative_returns.{parquet,csv}`, the pattern `sectorWeights` JSONs):

- **`validation`** — per `(signal, sector, horizon)` Spearman rank-IC of
  `cell = signal × weight` vs forward `rel_ret`, sign-agreement hit-rate, and the
  empirical lead/lag peak-horizon; emits one IC heatmap per horizon.
- **`calibration`** — per `(pattern, sector)` a *suggested* weight on the
  methodology 7-level grid, gated by beta-sign↔IC-sign agreement + significance +
  an in/out-of-sample split. **Proposal only — never edits a pattern file.**
- **`regime`** — a Python port of `SectorRegimeClassifier`'s contiguous-band
  taxonomy; FRED-driven regime timeline vs NBER recessions + known drawdowns
  (event study), plus the small-sample FRED-vs-news disagreement check.

Run (writes reports + figures + the proposal JSON to `data/analysis/`, git-ignored):

```bash
backtest/.venv/bin/python backtest/analyze_matrix.py
```

Outputs: `validation_report.md`, `calibration_report.md`, `regime_report.md`,
`decisions_verdict.md` (the D1-weights + decisions #4/#5 verdict), the IC heatmaps
+ regime chart under `figures/`, and `calibrated_weights_proposal.json` (drop-in
per-pattern `sectorWeights` blocks). **Nothing is auto-applied.**

> **Data-window caveat.** The harness runs on whatever as-of FRED history the live
> DB carries. ATLAS began collecting in late 2025, so the FRED `AsOf` vintages span
> only ~7 months — IC has minimal statistical power and the calibration proposes 0
> everywhere (overfitting discipline holding). The spec's named ALFRED
> point-in-time-vintage hardening step is what unlocks a deep-history backtest; the
> harness structure does not change. Every report stamps this caveat.

## Tests (no network)

```bash
cd backtest && .venv/bin/python -m pytest
```

The B1 tests parse a canned Yahoo v8 fixture (`tests/fixtures/`); the B3 tests check
the relative-return math; the B4 tests (`tests/test_analysis_synthetic.py`) feed the
analysis modules synthetic series with a *known* injected correlation and assert the
computed IC sign/magnitude, hit-rate, calibrated-weight sign, grid-snap, and regime
boundaries come out as constructed. **No test hits the network or the DB.** The live
fetch + signal-replay are run once, by hand, to produce the datasets.
