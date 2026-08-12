# F4.6.4 A/B Audit Harness

On-demand A/B audit for the F4.6.4 entity-resolution prompt-grounding
feature. Runs the same 50-row stratified sample through two prompt
variants — control (`{resolved_entities}` = `(none)`) and treatment
(prepass-rendered Markdown table) — and grades the paired outputs against
the post-F4.6.3 failure-mode rubric. Output is a dated Markdown scorecard
that drives the rollout decision in plan §A/B test design + §Rollout / risk.

The harness itself is **invoke-on-demand** — no CI job runs it, and
nothing here writes to production tables. It calls the live extraction
stack (vLLM, SecMaster, spaCy NER sidecar, trafilatura) read-only. Its
offline unit tests do run in CI (`.github/workflows/python-tests.yml`);
the harness they cover does not.

Source-of-truth spec: the f4.6.4 OpenFIGI/NAICS/RAG phase-1 plan §A/B test
design (retired to git history — recovery via `docs/RELEASES.md`).

## Files

| File | Role |
|---|---|
| `compare_base_vs_resolved.py` | Per-row driver. Hits trafilatura → spaCy + SecMaster (treatment only) → vLLM twice per row. Emits `control.jsonl` + `treatment.jsonl` + `index.json`. |
| `ab_scorecard.py` | Loads the JSONL pair, runs the deterministic grading rubric, partitions by Subset A / B, evaluates acceptance gates, renders `f4.6.4-ab-YYYY-MM-DD.md` (+ optional JSON sidecar). |
| `test_ab_scorecard.py` | Unit tests over the grader, pairing logic, stub accounting, and the acceptance-gate decision matrix. Fully offline. |
| `test_compare_base_vs_resolved.py` | Unit tests over the driver's C#-parity surface (candidate minting, renderer cell shape). Fully offline — mocks every client. |
| `weekly_quality_check.sh` | systemd-timer wrapper: runs both scripts on a fresh sample, refuses an unscoreable run, publishes the weekly pulse and the regression alert. See below. |

## Prerequisites

A venv with `httpx` and `psycopg2` (or `psycopg2-binary`) is sufficient:

```bash
python3 -m venv .venv
. .venv/bin/activate
pip install httpx psycopg2-binary
```

Live services the harness expects on the host loopback (default ports):

| Service | Default URL | Purpose |
|---|---|---|
| TimescaleDB | `localhost:5432/atlas_data` | sample selection (`sentinel.raw_content`) |
| trafilatura sidecar | `http://localhost:3109` | HTML → text normalization |
| spaCy NER sidecar | `http://localhost:3110` | treatment NER pre-pass |
| SecMaster | `http://localhost:8083` | treatment `POST /api/resolve-entities` |
| vLLM | `http://localhost:8000` | qualitative extraction (`/v1/completions`) |

If any of these is on a non-default port, pass `--*-url` overrides.

The harness reads the production prompt template + schema **from this
repo**, so it always matches whatever's committed at HEAD. There is no
config drift between the harness and production: change the prompt,
rerun the harness, get a real A/B.

## Running the harness

```bash
# Plan §A/B test design — same 50-row stratified sample, seed=42.
python3 compare_base_vs_resolved.py \
  --limit 50 \
  --seed 42 \
  --output-dir /opt/ai-inference/training-data/f4.6.4-ab

# Score the run + emit the dated Markdown report.
python3 ab_scorecard.py \
  --input-dir /opt/ai-inference/training-data/f4.6.4-ab \
  --emit-json
```

The harness is resumable: if both JSONLs already contain a row id,
that id is skipped on subsequent invocations. Pass `--overwrite` to
truncate and rerun cleanly.

## Variants in detail

**Control.** `{resolved_entities}` is substituted with the literal
`(none)` sentinel — byte-equivalent to running pre-F4.6.4 production
with the flag off.

**Treatment.** Per row:

1. Trafilatura normalizes the raw content (same path Sentinel uses).
2. spaCy NER sidecar extracts ORG candidates (POST `/ner`, threshold 0.8).
   On sidecar failure the harness falls back to the ticker regex only —
   this matches the C# orchestrator's `FallbackRegex` outcome and the
   degraded behaviour is recorded on each row.
3. Ticker regex pulls `(EXCH:TICKER)` and `$TICKER` cashtags. Bare
   uppercase body tokens are **not** lifted by the Python harness because
   that dictionary lookup against the `instruments` table lives in the
   C# `CompanyNameCandidateExtractor`; the audit accepts the narrower
   Python-side recall as a documented degraded mode.
4. Union (deduped on `(surface, kind)`) capped at 10 — matches
   `ExtractionOptions.EntityResolutionMaxCandidatesPerArticle`.
5. POSTs `{candidates, articleContext, minConfidence=0.8}` to
   SecMaster's `/api/resolve-entities`.
6. Top-5 by descending confidence is rendered into the Markdown table
   that `SentinelCollector.Services.ResolvedEntitiesRenderer.Render`
   produces in production. (Cell shape is byte-equivalent.)

The renderer is mirrored from the C# implementation; whenever the C#
renderer changes you must update `render_resolved_entities` in
`compare_base_vs_resolved.py` to match. The treatment prompt cell shape
is the load-bearing detail; if it drifts the audit no longer represents
production.

## Stratification

50-row default mirrors the plan's audit population. Per-source fractions
are tuned for **both subsets to populate**:

| Source | Fraction | Why |
|---|---|---|
| `rss` | 0.50 | news with company mentions; populates Subset A |
| `searxng-content` + `searxng` | 0.30 | mixed entity coverage |
| `fed-*`, `challenger-rss`, `tsa-checkpoint` | 0.18 | entity-light macro; populates Subset B |
| `validation-content` | 0.02 | bench/regression seed |

If your run produces Subset A < 30 or Subset B < 10, the population is too
narrow for the comparison to mean anything — bump `--limit` until both
subsets exceed those thresholds.

A 50-row request does not currently deliver 50 rows. `sample_stratified`
drops a stratum silently when its source has no eligible rows, and
`searxng` (target 5) and `validation-content` (target 1) have both dried
up, so ten of the fifteen runs on record delivered 44. Six of those 44 are
title-gated stubs the harness deliberately does not send to the model.
Read any per-run denominator off the scorecard, not off `--limit`.

## Acceptance gates

`ab_scorecard.py` evaluates the five gates from plan §A/B test design and
maps the result to one of three decisions:

| Decision | Gates required |
|---|---|
| `SHIP_FULL` | Subset A MAJOR (treatment) ≤ 1% (stretch); Subset B no regress (+1pp); overall ≤ 4%; |Δ confidence| ≤ 0.05 |
| `SHIP_50` | Subset A MAJOR (treatment) ≤ control − 3pp (minimum); Subset B no regress; overall ≤ 4%; |Δ confidence| ≤ 0.05 |
| `HOLD` | Any other outcome — flag stays off; root-cause per `PROBLEM_SOLVING` |

The decision matrix is unit-tested with boundary cases in
`test_ab_scorecard.py::AcceptanceTests`.

## Testing the harness itself

```bash
cd scripts/sentinel-quality-check
python3 -m unittest test_ab_scorecard.py
```

The grader, the pair/partition logic, the stub accounting, and the
acceptance gate boundaries are all deterministic and exercise inline
fixtures — no external services required. `.github/workflows/python-tests.yml`
runs these (plus `test_compare_base_vs_resolved.py`) under pytest on every
push and PR that touches this directory.

## Weekly automated quality check

Outside the F4.6.4 A/B itself, the same harness now powers a continuous
quality-monitoring loop on the qualitative extraction surface
(`sector_affinity`, `regime_hint`, `sentiment`). Wired up after a manual
audit on 2026-05-14 found 16-18% MAJOR rate on v6.2 base post-revert
versus the historical 4% target — there was no automated tripwire.

| Aspect | Value |
|---|---|
| Wrapper | `scripts/sentinel-quality-check/weekly_quality_check.sh` |
| systemd service | `atlas-sentinel-quality-check.service` |
| systemd timer | `atlas-sentinel-quality-check.timer` (`OnCalendar=Mon *-*-* 09:23:00`) |
| Run dir | `/opt/ai-inference/training-data/sentinel-quality-check-YYYYMMDD-HHMM/` |
| Log file | `/opt/ai-inference/logs/sentinel-quality-check/weekly-YYYY-MM-DD.log` |
| Regression threshold | MAJOR (treatment) over **graded** rows — usable and non-stub — > 25% → alert. `of usable` and `of n` are published beside it, not gated on |
| Unscoreable guards | > 40% BROKEN (worse arm) **or** < 20 graded (non-stub usable) rows → exit 1 + high-priority alert, no weekly pulse |
| NTFY topic | `atlas-claude-ask` on `https://ntfy.elasticdevelopment.com` |
| Seed | ISO-week (`date -u +%G%V`) — same value all week, fresh weekly. Does **not** make the draw reproducible: the sampler shuffles a table that changes between runs, and four runs sharing seed=202620 overlapped by 0-1 rows of 50 |
| Model pinned | `Qwen/Qwen2.5-32B-Instruct-AWQ` — the only model vLLM serves. The `sentinel-cove-v6.2` alias was dropped 2026-05-25; leaving it pinned here is what produced eleven consecutive dead runs |

The wrapper always publishes a low-priority weekly summary so operators
have a constant pulse on quality (not just regression-only). On regression
it adds a default-priority `WARN` alert with the acceptance-verdict block
inlined.

Deploy via ansible, from `deployment/ansible/` (its `ansible.cfg` supplies
the inventory):

```bash
ansible-playbook playbooks/deploy.yml --tags quality-check --skip-tags always
```

`--skip-tags always` is not optional. `quality-check` names no compose
service — the four tasks it carries are a log directory plus the systemd
unit and timer — so the scoped-restart form does not apply, and the bare
`--tags quality-check` this file used to document selects 41 tasks, not 4:
everything tagged `always`, including `Remove existing compose.yaml to
force regeneration` and `Start or restart ATLAS infrastructure`. That is a
full-stack restart with a ~4 min vLLM GPU reload to install a timer.
Measured with `--list-tasks` (ansible-core 2.16.3), which parses tag
selection without executing anything: 41 tasks bare, 4 with the skip.

Manual fire (smoke / fault-finding):

```bash
sudo systemctl start atlas-sentinel-quality-check.service
journalctl -u atlas-sentinel-quality-check.service --since '15 min ago'
```

Threshold rationale: post-revert v6.2 baseline runs land at 16-18% MAJOR;
25% leaves headroom for sampling variance at the ~44 rows a full draw
actually delivers (not the 50 requested — two strata are dry, see
Stratification) while still flagging real regression. Tighten the
threshold once the LoRA work brings the baseline down toward the 4%
target.

Two guards sit in front of that threshold, because a MAJOR rate computed
over nothing is not a pass:

| Guard | Default | Refuses when |
|---|---|---|
| `SENTINEL_QC_MAX_BROKEN_PCT` | 40 | the worse arm graded more than 40% of paired rows BROKEN — the proportional collapse that produced eleven green dead runs |
| `SENTINEL_QC_MIN_SCORED_ROWS` | 20 | fewer than 20 rows were graded against model output — usable rows that are **not** title-gated stubs |

The stub exclusion is the load-bearing half of the second one. A stub is
graded CORRECT without any model output being read and can never be
BROKEN, so stubs inflate both the usable count and the BROKEN percentage's
complement: 15 BROKEN plus 25 stubs at n=40 reports 37.5% BROKEN, 25
usable rows and 0.0% MAJOR, and neither guard sees a problem unless the
floor counts graded rows. The same exclusion is why the threshold above
divides by graded rows: a stub in that denominator deflates the rate
exactly as a BROKEN row does. Either refusal exits 1 (systemd marks the
unit failed) and publishes a high-priority ntfy instead of the weekly pulse.
`ab_scorecard.py` emits `usable`, `usable_nonstub` and `stubs` side by
side so the divergence is visible in the scorecard as well.

## Deferred / out of scope

- **Statistical significance.** With n=50 the harness reports descriptive
  rates only. The plan's acceptance gates are calibrated for that sample
  size; if you bump `--limit` you may want to add a CI/p-value sidecar.
- **VCR fixtures.** Plan §Test plan explicitly defers VCR cassettes for
  OpenFIGI; this harness inherits that decision and calls live services.
