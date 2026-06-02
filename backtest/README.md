# backtest/ — offline matrix-backtesting research area

**This is offline research tooling, not a deployed service.** Nothing here runs
in `compose.yaml`, on the gRPC mesh, or under ansible. It runs on demand from a
developer machine to produce datasets the matrix-backtesting harness analyzes
(`docs/atlas-matrix-backtesting-spec.md`).

This first slice (WS3 B1, re-sourced) is **Component 2's data dependency**: the
12 SPDR sector ETFs + SPY, split/dividend-adjusted daily close, to inception.
Downstream components (C# signal-replay, relative-returns, Python analysis) land
separately.

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
backtest/.venv/bin/pip install -r backtest/requirements.txt
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

## Tests (no network)

```bash
cd backtest && .venv/bin/python -m pytest
```

The unit tests parse a canned Yahoo v8 fixture (`tests/fixtures/`) and assert the
adjusted-close column is present, dates are ascending, and a 2:1 split is
reflected in the adjusted series. **No test hits the network.** The live fetch
is run once, by hand, to produce the dataset.
