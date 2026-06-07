# CoD Truncation-Value Experiment — Design Spec

- **Date:** 2026-06-07
- **Status:** Approved design, pre-implementation
- **Area:** Sentinel extraction throughput
- **Origin:** Brainstorm — "does reading the full article add significant value over headline + lead?"

## Problem & motivation

Sentinel CoD extraction (`qwen3-30b-a3b` on the CPU `llama-server`) is throughput-bound at **~20 articles/hr**. The dominant cost is **prefill**: ~133s to ingest a ~10K-token article before the first output token. CPU tuning is exhausted — measured, `--parallel 1` + flash-attn + ubatch + KV-q8 fixed correctness/latency (restored full 32K ctx) but left *aggregate* throughput roughly flat; the 30B-on-CPU ceiling is ~20/hr regardless of knobs.

Prefill cost scales with **input size**, so the highest-leverage untested lever is the **amount of article text per extraction**. The question:

> Does reading the full article body add *material* value to CoD extraction over a truncated read (headline + lead paragraphs, possibly + tail)?

If not, truncating long articles collapses the prefill and multiplies throughput — **without** GPU offload, a smaller model (the SENTINEL ≥30B quality rule holds), or reducing the article *count* (full coverage preserved). This is an empirical question → measure it, don't assume.

## Hypotheses

- **H1 — wire/RSS news:** full body adds ~nothing; inverted-pyramid journalism front-loads who/what/when, so head-only truncation recovers ~all observations.
- **H2 — analytical / FOMC / long-form:** the material conclusion lives in the *closing* paragraphs (hypothesis-and-conclusion structure), so **head + tail** truncation recovers the value where head-only would miss it.
- **H3 — length:** all prefill savings are on *long* articles; short articles are already cheap and out of scope. Length both stratifies the analysis and gates the eventual policy.

## Experimental design

### Strata (by article shape, not just source)
1. **Wire/RSS news** — `rss`, `searxng-content` (templated, inverted-pyramid).
2. **Analytical / long-form** — incl. FOMC/Fed statements, analyst deep-dives.
3. **Primary docs** — only if a distinct shape emerges in the sample.

~50–100 **long** articles per stratum (short articles excluded — not the target). "Long" defined by full-body token count, threshold drawn from the prefill-cost distribution (≈ > 3–4K tokens, to be set from data).

### Truncation variants (shape × depth)
- **Head-only** (news): `{ headline, +1 para, +2, +3, … , full }`.
- **Head + tail** (analytical): `{ head-N + tail-M }` across a small depth grid.
- **Paragraph-based** splitting (maps to inverted-pyramid structure); fall back to sentence/token boundaries where stored text lacks paragraph breaks.
- **Control:** model, DSL prompt, and decoding params **identical** across all variants — *only the article-text slice changes*, so any output difference is attributable to input size alone.

### Reference / baseline
- Each sampled article's **stored production full-text `extracted_observations`** is the gold reference — free (already computed) and proven reliable in production.
- **Fresh-control subset** (~10–15 articles): re-run full-text extraction fresh to confirm stored baselines are reproducible under the current model/prompt (guards against baseline drift).

## Metrics

### A — Observation recall (the spine)
Per article × truncation variant: **recall = |truncated ∩ baseline| / |baseline|** over the DSL observations (ENT/EVT/CLAIM). Averaged per stratum × shape × depth → the **recall-vs-truncation curve**; the *knee* (where added text stops adding recall) is the answer.
- **Matching:** an **LLM-judge aligns** the two observation sets semantically (pairs equivalent observations despite phrasing/DSL differences), rather than brittle exact-field string matching.
- **Precision check:** does thinner context introduce *hallucinated* or *mis-attributed* observations (present in truncated, absent/contradicted in baseline)? Reported alongside recall — truncation must not trade recall for fabrication.

### C — Miss characterization (the judge)
For observations the baseline has but a truncated variant misses, an **LLM-judge classifies each miss as material** (signal/digest-relevant — a number, an entity action, a forward-looking statement) **vs boilerplate** (disclaimers, background, repetition). Recall loss of *only boilerplate* ⇒ safe to truncate; *any material loss* ⇒ not.

## Mechanics

Pure **offline batch** — no production disruption:
1. Sample long articles per stratum from stored `sentinel.raw_content` (+ their stored full-text `extracted_observations` as baseline).
2. Paragraph-split each; generate the truncation variants (head-only + head+tail grids).
3. Run CoD extraction on each truncated variant — cheap (short prompts; this is the hypothesis). Execute on the existing `llama-server` during a low-ingestion window, or a side instance, to avoid contending with the production backlog drain.
4. Score recall (A) + precision + run the miss-judge (C).
5. Emit per-stratum recall curves + miss breakdowns + a recommended truncation policy.

## Decision rule & rollout

- **Per stratum**, select the cheapest `(shape, depth)` where **recall ≥ bar AND the judge finds zero *material* misses**.
- **Conservatism:** strict for **analytical** (high recall bar + zero material misses — a dropped FOMC fact is expensive); looser for **wire news**. Pick the data knee, then apply a safety margin.
- **Rollout:** implement the per-stratum truncation in the live extraction path, applied to **long** articles only, *before* the CoD LLM call. Short articles unchanged. Keys off measurable features (source + length + structure) and errs conservative on classification.
- **Validation:** post-rollout, confirm prefill (`sentinel_extraction_stage_duration_seconds{stage=cod_llm}` / llama-server `pp`) drops and aggregate throughput rises on the #638 dashboard; sample-audit live truncated extractions for material parity.

## Non-goals / out of scope

- Changing the model, GPU offload, or the ≥30B rule.
- Reducing article *count* — this trims per-article text only; full coverage preserved.
- The `NewsSignalClassifier` path (separate, GPU/vLLM) — unless the experiment shows it is also input-bound (follow-up, not this spec).

## Risks & mitigations

- **Baseline trust** — stored full-text extractions assumed correct/reproducible → mitigated by the fresh-control subset.
- **Matching validity** — the LLM-judge alignment must be validated on a hand-checked subset before the recall numbers are trusted.
- **Stratum leakage** — mis-classifying an analytical doc as wire-news could truncate away material → the policy keys off measurable features and errs conservative; the analytical bar is zero-material-miss.

## Success criteria

A defended, per-stratum truncation policy with measured recall/precision curves and a material-miss audit, ready to roll into the extraction path — answering "what does the body add" with data, and (if H1/H2 hold) delivering a multiple-× prefill reduction on long articles, hence a throughput multiple, with no loss of material extraction quality.
