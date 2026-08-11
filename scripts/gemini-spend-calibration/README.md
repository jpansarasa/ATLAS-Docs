# gemini-spend-calibration

Offline harness for calibrating the surface gate in front of the paid Gemini resolver. Two scripts,
stdlib only, no venv and no install step:

| File | Purpose |
|---|---|
| `capture-window.py` | Freezes a window of what actually reached the paid boundary, as JSONL. |
| `probe-replay.py` | Probes SearXNG once per distinct surface, caches every raw response, and scores the issuer-probe signals into a CSV. Sets **no thresholds** — it scores and reports. |
| `mutation-check.py` | Deletes each guard in `probe-replay.py` in turn and requires the suite to notice. This is what the coverage claim is worth. |
| `test_probe_replay.py` | Unit tests for the scorer. |
| `testdata/synthetic-capture.jsonl` | 17 capture records, 11 distinct surfaces — the reference input. |
| `testdata/reference-cache/` | The **frozen, committed** SearXNG responses for those 11 surfaces, drawn 2026-08-11T13:19:24Z–13:20:19Z. `MANIFEST.sha256` covers them. |

## Running it

`pytest` is **not installed on this host** and there is no venv to create — `unittest discover` is
the runner. `pytest` here fails with `No module named pytest`, which is an environment answer, not
a test result.

```bash
# the suite (29 tests)
python3 -m unittest discover -s scripts/gemini-spend-calibration

# what the suite is worth: 12 mutants, 11 killed, 1 declared equivalent
python3 scripts/gemini-spend-calibration/mutation-check.py

# re-CHECK the frozen reference draw -- no network
cd scripts/gemini-spend-calibration
python3 probe-replay.py --input testdata/synthetic-capture.jsonl \
    --cache-dir testdata/reference-cache --offline --reference-check \
    --patterns proposed --out /tmp/scored.csv

# the cache is intact -- INDEPENDENTLY, without trusting the harness's own code
cd testdata/reference-cache && sha256sum -c MANIFEST.sha256
```

A cache dir carrying `MANIFEST.sha256` is **frozen**, and the harness treats it that way on every
run: it verifies the envelopes before replaying them (exit 5 on any mismatch — edited, missing, or
added after the freeze — and no CSV is written), and it refuses to probe live into such a dir
without `--refresh`. That closes the cheap accident of pointing `--cache-dir` at
`testdata/reference-cache` and forgetting `--offline`, which draws live and writes new envelopes
into the committed draw. `--refresh` still works and still leaves the manifest stale on purpose, so
the next replay says loudly that the population changed.

## Why the cache is committed

The 11-surface reference table is the only labelled data that exists, and it is what the eventual
threshold gets sanity-checked against. A cache frozen to disk but not to git can only be
re-**drawn**, never re-**checked** — and re-drawing is the operation that changes the population,
because SearXNG's engine mix varies minute to minute (`Ford Motor` has been observed at both 9 and
11 listed hits for that reason alone). With the responses committed, `--offline` reproduces the
table byte for byte; without them, a differing re-draw and a real regression look identical.

A live re-draw **will** differ from the committed table. That is not a regression, and the frozen
numbers must never be edited to match one. `--refresh` draws a new population; `--offline` is the
only mode that re-checks this one.

Full rationale — query template, engine attrition, the per-result vs per-surface suppression limit,
the index/fund-identifier false positives — is in `probe-replay.py`'s module docstring. Read it
before changing a pattern: every score is a statement about one query string against one engine
mix, so an edit there voids any threshold derived from these numbers.
