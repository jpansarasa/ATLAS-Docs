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

**75 long articles per stratum** (+ a 15-article fresh-control subset); short articles excluded — not the target. **"Long" default ≥ 4,000 body tokens** (prefill > ~60s — where the savings concentrate); the supervisor may refine the threshold from the observed token distribution.

### Truncation variants (shape × depth)
- **Head-only** (news): `{ headline-only, +1 para, +2, +3, full }`.
- **Head + tail** (analytical): `{ head-2+tail-1, head-2+tail-2, head-3+tail-2, full }`.
- Concrete starting grids; the supervisor may add a level if a curve's knee is unresolved.
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

- **Per stratum**, select the cheapest `(shape, depth)` where **recall clears the bar AND the judge finds zero *material* misses**. Default bars: **wire news ≥ 0.95 recall; analytical ≥ 0.98 recall** (both with zero material misses). Tune within this conservatism from the data.
- **Conservatism:** strict for **analytical** (high recall bar + zero material misses — a dropped FOMC fact is expensive); looser for **wire news**. Pick the data knee, then apply a safety margin.
- **Rollout:** implement the per-stratum truncation in the live extraction path, applied to **long** articles only, *before* the CoD LLM call. Short articles unchanged. Keys off measurable features (source + length + structure) and errs conservative on classification.
- **Validation:** post-rollout, confirm prefill (`sentinel_extraction_stage_duration_seconds{stage=cod_llm}` / llama-server `pp`) drops and aggregate throughput rises on the #638 dashboard; sample-audit live truncated extractions for material parity.

## Execution plan (phased — for autonomous supervisor)

Run heads-down through the gates below; each phase has an explicit done-condition the supervisor self-verifies before advancing. Track progress in STATE.md.

**Phase 0 — Harness + matching validation [GATE].** Build the offline harness (sample → paragraph-split → truncate → CoD-extract each variant → score). Validate the LLM-judge observation-matching on a hand-labeled subset (~25 baseline↔truncated observation pairs): the judge's pairing must agree with the hand labels ≥ ~90%.
- *Done:* harness runs end-to-end on ~5 articles; matching validated ≥ 90%.
- *Escalate (NTFY) if:* matching can't clear ~90% — the recall metric would be untrustworthy and the approach needs a rethink.

**Phase 1 — Pilot one stratum [GATE].** Run the full sweep on the strongest-hypothesis stratum (wire/RSS news) only; sanity-check the recall-curve shape + the harness outputs.
- *Done:* a coherent recall-vs-depth curve for wire news + a miss breakdown.
- *Escalate if:* the curve is incoherent/anomalous in a way that isn't a self-resolvable harness bug.

**Phase 2 — Full sweep.** Run all strata × shapes × depths. Iterate freely on harness/prompt issues — autonomous, no gate.
- *Done:* recall + precision + miss-classification data for every (stratum, shape, depth) cell.

**Phase 3 — Analysis + policy proposal [GATE].** Produce per-stratum recall curves, the material-miss audit, and a concrete recommended per-stratum truncation policy (shape, depth, length-trigger) with the projected prefill/throughput gain.
- *Done:* a written results doc + the proposed policy.
- *Escalate (NTFY) with the policy proposal* — changing what the extractor reads is a product decision and a gate before touching production. Also escalate if **no safe truncation exists for a stratum** (hypotheses failed — needs direction).

**Phase 4 — Rollout [USER-GATED].** Only after the user approves the policy: implement truncation in the live extraction path (long articles, pre-CoD) behind the per-stratum policy; deploy; validate the prefill drop + throughput rise on the #638 dashboard; sample-audit live parity. Production-behavior change → user-approved cutover.

## Autonomy boundaries — decide vs escalate

**Supervisor decides autonomously** (no NTFY): sample sizes + the long-token threshold (from the observed distribution), the truncation depth grids, harness/prompt implementation + iteration, the matching/judge prompts, the analysis, and recall-bar tuning *within* the stated conservatism (strict analytical, looser news).

**Escalate via NTFY (`atlas-claude-ask`, single-line) only on:**
1. **Phase-0 matching-validation failure** — the metric itself isn't trustworthy.
2. **Hypotheses fail** — a stratum shows no truncation that preserves material recall (premise wrong → needs direction).
3. **Phase-3 policy proposal** — present the per-stratum policy + projected gain for approval before any production change.
4. **Phase-4 rollout** — the production cutover is user-gated.

Otherwise: run heads-down, track in STATE.md, end each turn with `▶ continuing autonomously`. The user should not need to be at the keyboard for Phases 0–2.

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
