# Extending Sentinel LoRA training data with negative examples

**Status:** Proposal for your review
**Author:** Late-night design session, 2026-04-07
**Target:** sentinel-cove-v6.3 (or next training iteration)

## TL;DR

- Current LoRA training is positive-only. The model learns "document → extract something" but never sees explicit "document → `[]`" signal from fine-tuning.
- The skip behavior we saw in production today (foreign content, political news correctly returning empty arrays) is leaking through from base Qwen2.5's instruction following, **not** from the LoRA. That's fragile under continued training — as you iterate v6.3, v6.4 with more positive examples, the statistical pressure will pull the model toward over-extraction.
- **Recommendation: add a ~20% chunk of hard negatives to the next training run.** Easy negatives (random non-economic text) teach nothing. The useful negatives are documents that *look like they should yield data* but shouldn't, for specific reasons drawn from the skip rules in `initial_extraction.txt`.
- You already have 22,312 production items that returned `[]` sitting in `sentinel.raw_content`. That's the source pool — don't synthesize new ones.
- Before committing to this, measure v6.2's current false-positive rate (hallucinated observations) on a sample of the 9,610 historical observations. If it's already low, this is insurance against drift. If it's high, it's urgent.

## Current state of the art

| Metric | Value | Source |
|---|---|---|
| Total raw_content items | 23,659 | `sentinel.raw_content` |
| Items that yielded observations | 1,347 (5.7%) | `DISTINCT raw_content_id` in `extracted_observations` |
| Items that returned `[]` | 22,312 (94.3%) | processed but no observations |
| Total observations persisted | 9,610 | `extracted_observations` |
| Errored items (historical, pre-vLLM) | ~1,100 | `processing_error IS NOT NULL` |

Production already tells you the distribution: **~94% of real-world RSS/SearXNG input yields nothing extractable**. Your training distribution should at least acknowledge this exists, even if you don't match the 94% ratio (see "Ratio" below).

## The problem with positive-only training

Your current training data (`generate_training_from_production.py`, `generate_diverse_training.py`, `generate_production_training.py`) constructs examples of the form:

```json
{
  "instruction": "Extract economic data points...",
  "input": "<document containing real economic data>",
  "output": "[<one or more valid extractions>]"
}
```

The LoRA learns `P(output | document)` where `output` is always a non-empty array. There is **no training signal** for:

- "Document mentions numbers but they're foreign (India, EU, China) → return `[]`"
- "Document mentions numbers but they're historical averages → return `[]`"
- "Document is political/editorial with no numeric data → return `[]`"
- "Document is a methodology note → return `[]`"

Today in production, v6.2 correctly filtered Trump-Iran politics, India services PMI, BOJ regional statements. That is a genuinely good result — but it's load-bearing on Qwen2.5-32B-Instruct's base instruction-following. The LoRA's attention-projection deltas (rank 8, q/k/v/o_proj) don't explicitly encode the skip rules. Every additional training epoch on positive-only data is a small nudge toward "always produce output." That nudge accumulates.

## The case for adding negatives

Three concrete benefits:

1. **Explicit skip-rule encoding.** The model learns which structural features trigger `[]` rather than pattern-matching on phrases. A document about "India's services PMI rising to 58.4" will structurally resemble a US ISM release — the difference (country) is a single token. Negative examples teach the model that token matters for the decision, not just the overall document shape.

2. **Drift resistance.** As you iterate v6.3, v6.4, v6.5 with more positive examples (you're building a dataset that grows over time), each training run statistically pulls toward "produce content." Negative examples counterbalance that pressure. Without them, you'll eventually notice a false-positive bump — probably around v6.5 or v6.6 — and it'll be harder to diagnose at that point.

3. **Training distribution ≈ inference distribution.** Your production workload is 94% empty-array inputs. Training on 100% positive inputs is a covariate shift. You can fix this without matching the 94% ratio (which would waste capacity) but moving from 0% to ~20% closes most of the gap.

## The case against (or caveats)

To be fair, here are the real reasons to be cautious:

1. **Base model already has the behavior.** Qwen2.5-32B-Instruct follows the "return `[]` if nothing applies" instruction pretty well even untrained for it. Your LoRA doesn't need to *replace* that behavior, only *protect* it from drift. This is why ~20%, not 50%, is the right dose.

2. **Bad negatives are worse than no negatives.** If you throw random Wikipedia articles at the model with `[]` as the label, it learns "short, generic text → skip" which is a useless rule. Worse, it can bias the model away from real extractions that happen to have unusual phrasing. Only *hard* negatives teach anything useful.

3. **Class imbalance bias.** Too many negatives → the model becomes a `[]` machine. I've seen this happen at 40%+ ratios. Stay conservative.

4. **Curation cost.** Good negatives require human review. You can't just `SELECT * FROM raw_content WHERE processed_at IS NOT NULL AND NOT EXISTS (observations)` and call it training data — some of those items returned `[]` because the old model *failed to extract valid data*, not because skipping was correct. Adding those as negatives would poison the training set by teaching the model to skip content it should extract from.

## The taxonomy of useful negatives

Drawn from `/opt/ai-inference/prompts/sentinel/initial_extraction.txt`:

| Skip category | What the document looks like | Why the LoRA needs to see it |
|---|---|---|
| **Foreign data** | "India's services PMI rose to 58.4 in March, the RBI reported" | Structurally identical to US ISM releases. Only country tokens differ. Without explicit negatives, the LoRA might extract this and the skip rule lives only in the prompt (which is compressible by the model's attention). |
| **Historical averages** | "Payrolls have averaged 173K over the last three months" | Contains a number, a time period, even a unit. Every structural cue says "extract me." The only thing stopping extraction is semantic: "this is a rolling average, not a point-in-time release." |
| **Redundant mentions** | Article mentions "4.1% unemployment" in three different paragraphs | LoRA should extract once with one text_quote. Training on the single-paragraph version doesn't teach deduplication across a full article. |
| **Demographic breakdowns** | "Unemployment among women aged 25-34 rose to 4.8%" | Real US data, real numbers, wrong granularity. Without a negative example, the LoRA will happily extract this. |
| **Methodology notes** | "The BLS uses establishment survey weighting at 60/40" | Sounds statistical, has a number. Should skip. |
| **Prior period context (unless revision)** | "Last quarter's 3.2% growth gave way to this quarter's preliminary estimate" | The "3.2%" is context, not the target. The target is what comes after "this quarter's". |
| **Editorial without numbers** | "Fed officials grew more hawkish at the meeting, signaling concern about inflation" | No numbers at all. Should skip. LoRA might try to extract sentiment as a data point. |
| **Political framing with economic topic** | "Trump pushed back against Fed rate decisions at yesterday's rally" | Economic topic, political content, no releases. Should skip. |

Note that some of these (historical averages, demographic breakdowns, prior period context) are **actually present in documents that also contain valid extractions**. So the negative example format isn't always "whole document → `[]`". Sometimes it's "whole document → extract X and Y but *not* Z" which is more like a discrimination example than a pure negative. I'll address both below.

## Recommended approach

### Phase 1: Measure v6.2's baseline (before any training changes)

Before spending effort on v6.3, quantify the current model's failure modes. Two metrics matter:

**Metric A: False positive rate (how often does v6.2 hallucinate?)**

Sample 100 random observations from `extracted_observations` and human-review them:

```sql
SELECT eo.id, eo.description, eo.value, eo.unit, eo.text_quote,
       eo.period, rc.source_url, rc.source
FROM sentinel.extracted_observations eo
JOIN sentinel.raw_content rc ON rc.id = eo.raw_content_id
ORDER BY random()
LIMIT 100;
```

For each one, answer: *is this a legitimate extraction?* Categories:
- ✅ Correct
- ❌ Hallucinated value (number doesn't appear in source)
- ❌ Wrong entity (extracted foreign data)
- ❌ Wrong period (historical average, prior period context)
- ❌ Redundant (same data point extracted multiple times)
- ❌ Should have been skipped per rules (demographics, methodology)

If ✅ is >95%, v6.2 is healthy and negatives are insurance. If ✅ is <85%, this is urgent and should block other training iterations.

**Metric B: False negative rate (how often does v6.2 miss real data?)**

Harder to measure without ground truth, but you can approximate. Sample 50 items from `raw_content` where `processed_at IS NOT NULL` AND `NOT EXISTS (observations)` AND the source URL contains a known-data-heavy domain (`bls.gov`, `federalreserve.gov`, `adp.com`, `census.gov`). For each, read the document and answer: *did this actually contain extractable data?*

If the false negative rate is high, you have a different problem — the model is over-filtering. Adding negatives would make that worse.

### Phase 2: Source candidate negatives from production

Use the existing `raw_content` table as the source pool. **Do not synthesize negatives.** The production data has exactly the distribution your model sees in production, which is what you want.

Query candidates:

```sql
-- Candidates: items that returned [] in production, grouped by likely skip reason
-- based on source URL heuristics. This is a starting point for human review,
-- not a final training set.

SELECT rc.id,
       rc.source,
       LEFT(rc.source_url, 100) AS url,
       LEFT(rc.context_summary, 300) AS summary,
       CASE
           WHEN rc.source_url ~* '(india|china|japan|europe|ecb|boj|pboc|rbi|uk\.gov|ons\.gov)'
               THEN 'foreign_data'
           WHEN rc.source_url ~* '(opinion|editorial|commentary|politics|trump|biden|harris)'
               THEN 'political'
           WHEN rc.source_url ~* '(methodology|technical-notes|faq|about)'
               THEN 'methodology'
           ELSE 'unclassified'
       END AS suspected_category
FROM sentinel.raw_content rc
WHERE rc.processed_at IS NOT NULL
  AND rc.processing_error IS NULL
  AND NOT EXISTS (
      SELECT 1 FROM sentinel.extracted_observations eo
      WHERE eo.raw_content_id = rc.id
  )
  AND rc.context_summary IS NOT NULL  -- has content to work with
ORDER BY random()
LIMIT 500;
```

The heuristic classifier is rough — its purpose is to bucket candidates so you don't spend all review time on one category. Target distribution for the final negative set:

| Category | Target count | Why |
|---|---|---|
| Foreign economic data | 80 | Most structurally similar to positives, highest risk of false extraction |
| Historical averages / prior period | 40 | Subtle temporal discrimination |
| Demographic breakdowns | 30 | Real US data, wrong granularity |
| Methodology / technical notes | 20 | Easy negatives but cheap to include |
| Political framing / editorial | 20 | Low risk but represents real production noise |
| Redundant mentions (whole article) | 10 | Tests deduplication |
| **Total** | **200** | ~20% of a 1000-example v6.3 training set |

The number 200 assumes your next training run is roughly 1000 examples total. Scale proportionally.

### Phase 3: Human review and labeling

For each candidate, decide:

1. **Keep as pure negative?** The entire document should yield `[]`. Example: an article about India's PMI release.
2. **Keep as partial-extraction discriminator?** Document contains *some* valid extractions and *some* things that should be skipped. Example: a BLS release that includes both the headline payrolls number (extract) and historical comparison averages (skip). These are the most valuable examples because they teach fine-grained discrimination.
3. **Reject as "model was right to skip but not for interesting reasons"?** Example: a 2-sentence RSS headline that genuinely contains no data. These add no signal and just bias the model toward `[]`.
4. **Reject as "model was wrong to skip"?** Example: a BLS release that the model failed to extract from — this is a *false negative*, and including it as a negative example would actively damage the training set. Flag these for investigation as positive training candidates instead.

I'd estimate reviewing 500 candidates to get 200 good ones, at roughly 30 seconds per review = ~4 hours of work. It's the expensive part of this proposal, and it's the part that can't be automated.

### Phase 4: Training data format

Two cases, matching the discrimination above:

**Case 1: Pure negative (whole document → `[]`)**

```json
{
  "instruction": "Extract economic data points from the following document as a JSON array. Skip non-US data, demographics, methodology, historical averages, and editorial content.",
  "input": "<full document text>",
  "output": "[]",
  "metadata": {
    "skip_reason": "foreign_data",
    "source_id": 26198,
    "human_reviewed": true
  }
}
```

**Case 2: Discrimination example (document with partial extractable content)**

```json
{
  "instruction": "Extract economic data points from the following document as a JSON array...",
  "input": "<full document text containing both extractable and skippable content>",
  "output": "[{valid extraction 1}, {valid extraction 2}]",
  "metadata": {
    "discriminator": true,
    "skipped_items": ["historical_3mo_average_173k", "breakdown_by_age_group"],
    "source_id": 12345,
    "human_reviewed": true
  }
}
```

The `metadata` field is not passed to the model during training — it's for your own auditing of the training set. Keep it separate from `input`/`output`.

### Phase 5: Training run

Mechanical changes to `train_qlora_unsloth.py`:

- No hyperparameter changes needed for the negative addition itself
- Consider a *slight* LR reduction (10-20%) if you see loss oscillation — negatives have different loss dynamics than positives because the target is much shorter (`[]` vs a full array)
- **Do shuffle** positives and negatives together. Do not train positives first then negatives. The model should see them interleaved.
- Keep the positive:negative ratio constant within batches if possible (i.e., if batch size is 10 and ratio is 80/20, each batch should have ~8 positives and ~2 negatives). This stabilizes training.

### Phase 6: Evaluation

Before shipping v6.3, run against a held-out eval set with known labels. The LlmBenchmark project has scaffolding for this. The metrics that matter:

- **Extraction F1 on positive examples** (should stay the same as v6.2 or improve)
- **False positive rate on held-out negatives** (should drop significantly vs v6.2)
- **False negative rate on positive examples** (should NOT increase — if it does, you have too many negatives)

If false negative rate increases, reduce the negative ratio from 20% to 15% and retrain.

## Open questions for you

1. **Do you want to measure v6.2 first (Phase 1) or go straight to curating?** If you trust v6.2's production behavior, Phase 1 is skippable, but you lose the before/after comparison.

2. **Is the 200-example negative set sized right?** I assumed ~1000-example training runs based on the existing script names. If your actual training runs are much larger (say 5000+ examples), scale the negatives to ~15-20% of total. If they're smaller, 200 might be too many.

3. **How much review time are you willing to spend?** 4 hours of careful review gets you a high-quality negative set. 1 hour gets you a rushed set that's probably net-negative. If you can't commit to the full review, it might be better to skip this entirely and rely on base model behavior.

4. **Should discrimination examples (Case 2) count toward the positive or negative budget?** I'd argue neither — they're a third category, and you want both pure negatives *and* discrimination examples in roughly equal numbers (say, 100 pure negatives + 100 discrimination examples = 200 total). Your call.

5. **Consider adding this metric to the benchmark:** `empty_array_agreement_rate` — on the held-out eval set, what fraction of documents where the ground truth is `[]` does the model also output `[]`? This is the direct measure of the behavior you're trying to protect.

## Scripts ready to run in the morning

I've left a sampling script at `/opt/ai-inference/scripts/sample_negative_candidates.py` that runs the candidate query above and dumps the results to a CSV for human review. Details below.

---

## Overnight system state (baseline for tomorrow's review)

**Time snapshot:** 2026-04-07 ~02:48 UTC

- vLLM backend active, `MaxConcurrentExtractions=4`
- 174 items processed since deploy, 0 new errors
- vLLM showing 3-4 concurrent requests steady-state
- Prefix cache hit rate dropped from 92% (synthetic tests with identical prompts) to 26% (production with varied content) — expected
- Generation throughput ~91-106 tok/s per request
- llama-server stopped; ollama-cpu idle (CoD no longer routed through it)

**Commands to run first thing in the morning:**

```bash
# Did anything break overnight?
sudo nerdctl ps --filter name=vllm-server --filter name=sentinel-collector --format '{{.Names}} {{.Status}}'

# What's been processed since you went to bed?
sudo nerdctl exec timescaledb psql -U ai_inference -d atlas_data -c "
SELECT
    COUNT(*) FILTER (WHERE processed_at > '2026-04-07 02:48:00') AS processed_overnight,
    COUNT(*) FILTER (WHERE processing_error IS NOT NULL
                     AND collected_at > '2026-04-07 02:48:00') AS new_errors,
    COUNT(DISTINCT eo.raw_content_id) FILTER (WHERE eo.extracted_at > '2026-04-07 02:48:00') AS items_with_observations
FROM sentinel.raw_content rc
LEFT JOIN sentinel.extracted_observations eo ON eo.raw_content_id = rc.id;"

# Any error spikes?
sudo nerdctl exec timescaledb psql -U ai_inference -d atlas_data -c "
SELECT processing_error, COUNT(*)
FROM sentinel.raw_content
WHERE collected_at > '2026-04-07 02:48:00' AND processing_error IS NOT NULL
GROUP BY processing_error;"

# vLLM still healthy?
curl -s http://localhost:8000/health && echo ' OK'
sudo nerdctl logs vllm-server --tail 5 2>&1 | grep -E 'Running|error|ERROR'

# Sample the negative candidate pool (for Phase 2 above)
sudo nerdctl exec timescaledb psql -U ai_inference -d atlas_data -f /dev/stdin < /opt/ai-inference/scripts/sample_negative_candidates.sql
```

If any of those surface surprises, investigate before touching anything. The system is stable as of the snapshot time — any deviation is news.
