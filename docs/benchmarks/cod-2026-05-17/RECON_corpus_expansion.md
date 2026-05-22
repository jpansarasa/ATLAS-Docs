# Recon — Corpus expansion for v8 baseline validation at scale

**Status:** recon only (no sweep). Surfaces options + costs to validate
that v8's aggregate verifier 0.907 / gv 9/9 holds beyond the Arm B n=9
set.

## 1. What "Arm B n=9" actually is

Defined operationally, not in a config. The sweep scripts (e.g.
`scripts/run_v10_sweep.sh`) hardcode the 9 IDs:

```
IDS=(27560 27702 29807 31149 31430 33632 34537 35352 36065)
```

These are the 10 rows of `corpus.jsonl` (the trimmed n=10 set from
2026-05-17 Round 1, stratified rss×5 / searxng×2 / fed-press×1 /
fed-speeches×1 / tsa×1) **minus 27772**, the 47K-char Jenoptik
earnings-call transcript that under llama-server constrained decoding
exhausts the 540 s `requests` timeout and produces an empty result
(documented in `scripts/build_phase2_sweep_report.py` Caveats §).

"Arm B" originates in Phase 2 §5.3 as the
**grammar-taught + GBNF-constrained + llama-server** arm (vs Arm A
free-decoder ollama, Arm C content-only). Phase 3 inherits the
nomenclature: every v5-v10 sweep is implicitly Arm B.

## 2. Available corpus surface area

| Source                                  | Rows                                |
| --------------------------------------- | ----------------------------------- |
| `corpus.jsonl` (active)                 | 10                                  |
| `corpus.full72.jsonl` (preserved)       | 72 (already ground-truth-labeled)   |
| `sentinel.raw_content` last 14 days     | 34 109                              |
| `sentinel.raw_content` last 30 days     | 56 068                              |

Of the production set, `rss` dominates (32 882 / 14d), which mirrors
the existing stratification weight (40 / 75 ≈ 53 %). Other strata
(`searxng-content`, `fed-press-*`, `fed-speeches-*`, `tsa-checkpoint`,
`challenger-rss`) are all present at comparable proportions to the
build_corpus.py weights.

**Key finding:** `corpus.full72.jsonl` already exists with **complete
ground-truth labels** (`ground-truth.jsonl` has 72 rows, mean 2161
input tok / 237 output tok per article). Scaling to 72 requires **zero
new GT generation cost** — just runner time.

## 3. Ground-truth pipeline + cost

`scripts/generate_ground_truth.py` is fully automated:

- reads `corpus.jsonl` → calls Claude Opus 4-7 via Azure Foundry with
  a 5-category extraction prompt (tickers / companies / sectors /
  industries / numerics_with_units), 60K-char input cap
- appends one JSONL row per article to `ground-truth.jsonl`
- resumable (skips IDs already written)

**Observed token usage** (n=72 historical):

- input mean 2161 / total 155 581 tok
- output mean 237 / total 17 091 tok

**Cost per article (Opus 4-7):**

| Channel                              | $ / article | $ / 100 articles |
| ------------------------------------ | ----------- | ---------------- |
| Anthropic direct ($15 in / $75 out)  | $0.050      | $5.03            |
| Azure Foundry (≈1/10× per memory)    | $0.005      | $0.50            |

Per the `feedback_azure_foundry_vs_anthropic_cost` memory ($9 vs $100
for same task), Foundry is the established channel for this kind of
GT generation.

## 4. Inference cost (sweep wall time)

llama-server is currently configured `--parallel 1 --ctx-size 32768`
(compose.yaml). PR #407 reverted from `--parallel 3` to solo mode
because the Sentinel 32K-context rule requires per-slot 32K, and
parallel-3 forced CPU offload that degraded throughput.

**v8 n=9 observed wall times** (`results/phase3-v8-layered-n9/NOTES.md`):

- p50 321 s, mean 349 s, p95 798 s, max 798 s, total 52 min

Wall-time at solo-mode `--parallel 1`:

| n    | Serial wall (mean 349 s) | Notes                             |
| ---- | ------------------------ | --------------------------------- |
| 30   | ~2.9 h                   | quick scale check                 |
| 100  | ~9.7 h                   | overnight, statistical validation |
| 500  | ~48 h                    | multi-day, strong validation      |

**Parallel options** (would require compose edit + RAM headroom check):

- `--parallel 2 --ctx-size 32768`: ~2× throughput, 64 GiB KV-cache
  ballpark — likely RAM-feasible but unproven on mercury
- `--parallel 3 --ctx-size 49152`: tried in PR #403, reverted in #407
  for CPU-offload regression

For n≥100, recommend keeping solo mode and just letting it run
overnight; the deterministic, sequential profile matches existing
sweep scripts with zero compose churn.

## 5. The 27772 wall-clock outlier

47K-char article (8.3× corpus p95) repeatedly exhausts the
constrained-decoder budget. Three options for a scaled corpus:

1. **Apply a 30K-char input cap** to the corpus filter. Drops outliers
   uniformly; matches the existing GT cap (60K → effective 15K input
   tok which is well under 32K ctx).
2. **Keep outliers but bump `--n-predict` to 16 384 + `--timeout 3600`**
   for the long tail. Burns wall time on edge cases.
3. **Mark outliers in corpus and report them separately** (n=N_total
   with k excluded for length). Honest, but complicates aggregate
   scoring.

Recommendation: option 1 (drop outliers, document the cap) — matches
how 27772 is *de facto* already excluded.

## 6. Recommended path

**Phase A — free scale-out (n=72, zero new GT, ~7 h sweep):**

- Use `corpus.full72.jsonl` + existing `ground-truth.jsonl` as-is
- Wire up a sweep script equivalent to `run_v10_sweep.sh` but with the
  full 72-row corpus, prompt = `cod_dsl_baseline.txt`
- Apply 27772-style length filter (e.g. skip raw_text > 30 000 chars,
  log skip list)
- Expected wall: 72 × 349 s ≈ 7 h, single overnight run
- Cost: $0 (GT already exists, inference is local)
- **Validates whether v8's 0.907 / 9/9 holds at 8× the cell count
  without any cash outlay**

**Phase B — fresh production sample (if Phase A passes):**

- n=100, pulled from `sentinel.raw_content` last 14 days, same
  stratification as `build_corpus.py`
- GT generation: ~$0.50 on Foundry (or $5 on Anthropic direct)
- Sweep wall: ~10 h overnight
- **Validates that v8 generalizes to articles outside the original
  May-17 fortnight**

**Phase C — defer (only if Phase A or B reveals regression):**

- n=500 fresh production sample
- GT: ~$2.50 Foundry / $25 Anthropic
- Sweep wall: ~2 days
- Only worth running if a Phase B regression needs statistical
  confirmation; otherwise diminishing returns vs Phase B.

## 7. Open questions for the user

1. **Length cap policy** — confirm option 1 (drop > 30 000 chars) is
   acceptable for scaled corpora, or prefer option 2/3.
2. **Phase B trigger** — auto-proceed to Phase B if Phase A passes
   (aggregate ≥ 0.85, gv ≥ 90 %), or pause for review.
3. **GT channel** — Foundry default ($0.50 for n=100) unless the user
   prefers a different judge model.

## 8. Concrete next actions (not in this PR)

- Implement length-filtered sweep wrapper (new script under
  `scripts/run_baseline_sweep.sh`) parameterized by N + corpus path.
- Build a per-cell delta report comparing n=72 v8 against the existing
  n=9 v8 numbers (verifies no per-cell regression on the original 9).
- If Phase B is approved, run `build_corpus.py` against the last 14
  days, call `generate_ground_truth.py`, then sweep.
