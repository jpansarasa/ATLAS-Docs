# F4.6.4 A/B Audit Harness

On-demand A/B audit for the F4.6.4 entity-resolution prompt-grounding
feature. Runs the same 50-row stratified sample through two prompt
variants — control (`{resolved_entities}` = `(none)`) and treatment
(prepass-rendered Markdown table) — and grades the paired outputs against
the post-F4.6.3 failure-mode rubric. Output is a dated Markdown scorecard
that drives the rollout decision in plan §A/B test design + §Rollout / risk.

This harness is **invoke-on-demand**. Nothing here runs in CI. Nothing
here writes to production tables. It calls the live extraction stack
(vLLM, SecMaster, spaCy NER sidecar, trafilatura) read-only.

Source-of-truth spec:
[`docs/plans/atlas-matrix/f4.6.4-openfigi-naics-rag-phase1.md §A/B test design`](../../docs/plans/atlas-matrix/f4.6.4-openfigi-naics-rag-phase1.md).

## Files

| File | Role |
|---|---|
| `compare_base_vs_resolved.py` | Per-row driver. Hits trafilatura → spaCy + SecMaster (treatment only) → vLLM twice per row. Emits `control.jsonl` + `treatment.jsonl` + `index.json`. |
| `ab_scorecard.py` | Loads the JSONL pair, runs the deterministic grading rubric, partitions by Subset A / B, evaluates acceptance gates, renders `f4.6.4-ab-YYYY-MM-DD.md` (+ optional JSON sidecar). |
| `test_ab_scorecard.py` | Unit tests over the grader, pairing logic, and the acceptance-gate decision matrix. Fully offline. |

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
cd scripts/f4.6.4-ab
python3 -m unittest test_ab_scorecard.py
```

The grader, the pair/partition logic, and the acceptance gate boundaries
are all deterministic and exercise inline fixtures — no external services
required.

## Weekly automated quality check

Outside the F4.6.4 A/B itself, the same harness now powers a continuous
quality-monitoring loop on the qualitative extraction surface
(`sector_affinity`, `regime_hint`, `sentiment`). Wired up after a manual
audit on 2026-05-14 found 16-18% MAJOR rate on v6.2 base post-revert
versus the historical 4% target — there was no automated tripwire.

| Aspect | Value |
|---|---|
| Wrapper | `scripts/f4.6.4-ab/weekly_quality_check.sh` |
| systemd service | `atlas-sentinel-quality-check.service` |
| systemd timer | `atlas-sentinel-quality-check.timer` (`OnCalendar=Mon *-*-* 09:23:00`) |
| Run dir | `/opt/ai-inference/training-data/sentinel-quality-check-YYYYMMDD-HHMM/` |
| Log file | `/opt/ai-inference/logs/sentinel-quality-check/weekly-YYYY-MM-DD.log` |
| Regression threshold | overall MAJOR (treatment) > 25% → alert |
| NTFY topic | `atlas-claude-ask` on `https://ntfy.elasticdevelopment.com` |
| Seed | ISO-week (`date -u +%G%V`) — same value all week, fresh weekly |
| Model pinned | `sentinel-cove-v6.2` (current production) |

The wrapper always publishes a low-priority weekly summary so operators
have a constant pulse on quality (not just regression-only). On regression
it adds a default-priority `WARN` alert with the acceptance-verdict block
inlined.

Deploy via ansible:

```bash
ansible-playbook deployment/ansible/playbooks/deploy.yml --tags quality-check
```

Manual fire (smoke / fault-finding):

```bash
sudo systemctl start atlas-sentinel-quality-check.service
journalctl -u atlas-sentinel-quality-check.service --since '15 min ago'
```

Threshold rationale: post-revert v6.2 baseline runs land at 16-18% MAJOR;
25% leaves headroom for n=50 sampling variance while still flagging real
regression. Tighten the threshold once the LoRA work brings the baseline
down toward the 4% target.

## Deferred / out of scope

- **Statistical significance.** With n=50 the harness reports descriptive
  rates only. The plan's acceptance gates are calibrated for that sample
  size; if you bump `--limit` you may want to add a CI/p-value sidecar.
- **VCR fixtures.** Plan §Test plan explicitly defers VCR cassettes for
  OpenFIGI; this harness inherits that decision and calls live services.
