# ATLAS — DSL Extraction PoC Plan

**Status:** PoC, exploratory. Iterate freely; data drives the next step.
**Goal:** CPU extraction → GPU verification pipeline that emits validated facts to the signal × sector × industry × time matrix with sufficient fidelity to feed scoring. North-star metric: per-article numeric+entity recall on financial news, with provenance traceable to a source span.
**Non-goal:** premature production deployment. Production prompts at `/opt/ai-inference/prompts/` are not touched in this plan.

---

## 1. Context (read once, then skip)

Where we are after R1/R2/R3:

- **qwen3:30b-a3b** carries the load. Class A small models are noisy; Class B `gemma4:26b-a4b` is pathological; Class C never ran.
- **Numeric recall 19% (R1 prose) → 91.7% (R2 DSL spec)** on qwen3:30b. Entity recall flat-to-slightly-up (25% → 29-31%).
- **Worked-example arm regresses recall ~20pp** on Class B, repeatable across R2 + R3. Confirmed, not noise.
- **Verdict: TRENDING.** Spec arm clears prose by +4.4pp, under the +5pp CONFIRMED threshold; run-to-run drift larger than the gap.

What the literature says (see `docs/research/dsl-prior-art.md` for the full survey):

- **Schema-as-code IE** (GoLLIE, ICLR 2024; Code4Struct, ACL 2023) beats NL-instruction IE by 30-45pp F1 on zero-shot NER.
- **Numeric drift is mechanistically traceable** to FFN digit-frequency bias (Shao et al. 2025, "Benford's Curse"). Verbatim copy bypasses the bias; regenerated numbers drift toward learned priors. **Best mechanistic explanation for the 19% → 91.7% jump.**
- **Span-grounded generation** reduces hallucination at scale (GopherCite, DeepMind 2022; Attributed QA, Bohnet et al. 2022).
- **Structured/compressed CoT** beats NL CoT on numeric and symbolic tasks: PoT +12%, PAL +40% (GSM-hard), CoC +12% (BBH), CoS +60.8% (spatial), CoD/SoT 70-90% token reduction at equal accuracy.
- **Counter-evidence (Tam et al. 2024):** JSON-mode hurts *reasoning* but not extraction; the failure mode is schema-order ("answer" emitted before "reason"). Mechanism, not contradiction.
- **Faithful-CoT (Lyu et al. 2023)** is the architectural precedent for the CPU-extract → GPU-verify loop: make the chain executable, then the verifier doesn't evaluate prose, it runs the program.

The four bets this plan makes:

1. **Schema-as-code extraction** beats DSL-as-text extraction (Phase 3).
2. **Grammar-constrained decoding** eliminates parse failure as a metric (Phase 2).
3. **Explicit copy-mode slots with provenance pointers** preserve numerics by routing them through induction-head-style copying rather than FFN regeneration (Phase 3).
4. **Deterministic verification** on GPU resolves hallucinations the CPU model couldn't avoid (Phase 4).

---

## 2. Open questions to disambiguate

Before more prompt-tuning, disambiguate these — each is cheap and informative:

| # | Question | Cheapest test | Decision it unlocks |
|---|----------|---------------|---------------------|
| Q1 | Does CoD-style thinking add value beyond structure, or is structure the whole story? | Direct-extraction arm (no intermediate DSL drafting), n=10, qwen3:30b | Keep CoD vs go to direct-extraction-only |
| Q2 | Does schema field order (Tam et al.) explain the worked-example regression? | Two variants of v1 DSL: verbatim-first vs derived-first | Validates or invalidates mechanism (a)'s schema-design failure mode |
| Q3 | Does the in-house sentinel-cod-v6 LoRA reproduce R1's 78%/82% under DSL? | Run Class C (sentinel-cod-v6 + qwen2.5:32b-dense) | Whether to retrain LoRA on DSL-formatted data |
| Q4 | Is the qwen3:30b gain real or a 30B-specific artifact? | Add 1 frontier closed-weight model (Claude on Foundry, single arm) | External validity of the whole approach |
| Q5 | Does the structured-output gain survive grammar-constrained decoding? | Phase 2 (vLLM guided_decoding) | Whether the gain is from teaching grammar or from emitting structure |

---

## 3. Phase 0 — Instrumentation (DO FIRST, blocks everything else)

**Why first:** R2/R3 collapsed metrics; we can't actually tell what's improving. No more runs until this lands.

### 3.1 Restore per-category recall breakdown

R1 had: `ticker / company / sector / industry / numeric`. R2/R3 collapsed to `ent_rec / num_rec`. Restore the breakdown.

```
recall metrics (per article, per model, per variant):
  - ticker_recall        # e.g., $AAPL, NVDA
  - org_recall           # company / organization names
  - person_recall        # CEOs, officials, analysts cited
  - location_recall      # geography
  - instrument_recall    # securities, indices, currencies
  - sector_recall        # GICS / NAICS sector tags
  - industry_recall      # GICS / NAICS industry tags
  - numeric_recall       # any number with a unit
    └─ numeric_exact_match_rate   # string-identical to source
    └─ numeric_semantic_match_rate  # "5 billion" ~ "5_000_000_000"
    └─ unit_preservation_rate     # currency, percentage, basis points, etc.
```

Acceptance: `aggregate_report.py` emits all of these. Existing R2/R3 results re-scored under new metrics so we have apples-to-apples vs R1.

### 3.2 Add span-copy fraction

For each output token, compute whether it appears as a substring of the input. Aggregate fraction per output.

```python
def span_copy_fraction(output_text: str, input_text: str, n: int = 4) -> float:
    """Fraction of output n-grams (n=4 chars) that appear verbatim in input."""
    # n=4 chosen to capture meaningful chunks while staying robust to tokenization
    output_ngrams = [output_text[i:i+n] for i in range(len(output_text)-n+1)]
    return sum(1 for ng in output_ngrams if ng in input_text) / max(len(output_ngrams), 1)
```

Hypothesis: span-copy fraction correlates with numeric_exact_match_rate and (weakly) with org_recall. If it does, that's direct mechanistic evidence for the copy-mode-vs-paraphrase-mode story (mechanism b).

Acceptance: metric emitted per (model, variant, article); correlation analysis in the aggregate report.

### 3.3 Latency-per-recall-point

```
cost_per_recall_point = total_wall_seconds / (mean_target_recall * 100)
```

Where `mean_target_recall` is the weighted mean of numeric + ticker + org recalls (the slots that feed sector scoring). This is the actual cost metric for production decisions; raw latency is misleading because higher recall justifies higher cost.

### 3.4 Standardize result schema

One result file per `(model, variant, article)`, all metrics in a fixed schema, JSON. So we never have to backfill metrics across runs again.

```json
{
  "model": "qwen3:30b-a3b-instruct-2507-q4_K_M",
  "model_class": "B",
  "variant": "spec",
  "schema_order": "verbatim_first",
  "decoding": "free",
  "article_id": 27772,
  "wall_seconds": 1095.3,
  "grammar_valid": true,
  "parse_error": null,
  "block_counts": {"ent": 12, "evt": 8, "claim": 3, "note": 1, "num": 47},
  "span_copy_fraction": 0.73,
  "recall": {
    "ticker": 0.50, "org": 0.42, "person": 0.0, "location": 0.0,
    "instrument": 0.50, "sector": 1.0, "industry": 0.33,
    "numeric": 0.91, "numeric_exact": 0.78, "numeric_semantic": 0.91,
    "unit_preservation": 0.85
  },
  "schema_completeness": 1.0,
  "undeclared_refs": 0,
  "missing_required_slots": 0
}
```

---

## 4. Phase 1 — Disambiguating experiments

All run with Phase 0 instrumentation in place. n=10 articles unless stated. All on existing R2/R3 corpus for direct comparability.

### 4.1 Direct-extraction arm (Q1)

New prompt: extract directly into v1 DSL with no preamble, no thinking, no draft. Just emit the document.

Compare three arms on qwen3:30b:
- `cod_dsl_spec` (current best — DSL CoD)
- `cod_dsl_direct` (NEW — no intermediate thinking)
- `prose_cod` (R1 prose baseline)

Decision:
- If `direct ≥ spec` → CoD adds no value beyond structure; retire CoD across the board.
- If `spec >> direct` → CoD-style intermediate state matters; keep it.
- If `prose_cod > both` → unexpected; rethink. (Low probability based on R1 vs R2/R3 numeric jump.)

### 4.2 Schema-order experiment (Q2)

Two variants of the v1 DSL schema:
- `variant_v1a`: verbatim slots first, derived slots last (current)
- `variant_v1b`: derived slots first, verbatim slots last (Tam et al. failure mode)

Run on qwen3:30b only, n=10, both spec and spec+example arms.

If `v1b` significantly worse than `v1a` → confirms the Tam et al. mechanism; lock in verbatim-first as a design rule for all future schemas.
If no difference → mechanism is elsewhere; document negative result.

### 4.3 Close the Class C hole (Q3)

Run the remaining Class C model R2/R3 skipped:
- `qwen2.5:32b-instruct-q4_K_M` (dense reference, BENCHMARKS.md leader)

Both arms (spec, spec+example), n=10. Same corpus.

(sentinel-cod-v6 is excluded: not a Class C candidate, treated as a cautionary
tale — it overfit prose CoD output-plausibility and degraded under R1.)

Decision:
- qwen2.5:32b beats qwen3:30b-MoE → reconsider MoE-over-dense default.
- qwen2.5:32b matches qwen3:30b → MoE-active-3B is sufficient at this scale.
- qwen2.5:32b dramatically worse → unexpected; investigate before Phase 2.

### 4.4 Frontier closed-weight reference (Q4)

One arm, one model, n=10. Claude 4.7 via Azure Foundry (already integrated for ground-truth scoring per R1). DSL spec arm only.

This is the **external-validity check**. If Claude crushes qwen3:30b by 30+pp on numeric recall, the gain is real and worth chasing. If Claude is only marginally better, qwen3:30b is at the capability frontier for this task and we should focus on Phase 2-4 rather than chasing a bigger model.

Budget cap: $5 in Foundry credits. Stop if exceeded.

---

## 5. Phase 2 — Grammar-constrained decoding

**Premise:** the v1 grammar is a context-free grammar; we should not be teaching it to the model with examples and hoping. We should make it impossible for the decoder to emit invalid syntax.

### 5.1 Convert v1 DSL grammar to GBNF

vLLM supports `guided_decoding` with regex, json_schema, and grammar (Lark or GBNF). Pick GBNF for compatibility.

```
# docs/grammars/cod-dsl-v1.gbnf
root        ::= header block*
header      ::= "SOURCE:" source_id "\n" "DATE:" iso_date "\n"
block       ::= ent_block | num_block | evt_block | claim_block | note_block
ent_block   ::= "ENT" ws ident "\n" ent_slot+
ent_slot    ::= "  " ent_key ": " value "\n"
ent_key     ::= "name" | "ticker" | "type" | "source_span" | ...
# ...etc
```

### 5.2 Wire vLLM guided_decoding

```python
# scripts/run_dsl_benchmark.py (sketch)
from vllm import LLM, SamplingParams
from vllm.sampling_params import GuidedDecodingParams

guided = GuidedDecodingParams(grammar=open("docs/grammars/cod-dsl-v1.gbnf").read())
params = SamplingParams(
    temperature=0.0,
    max_tokens=8192,
    guided_decoding=guided,
)
```

### 5.3 Acceptance

- `grammar_valid = true` on 100% of outputs (definitionally, but verify the parser agrees).
- `numeric_recall` on qwen3:30b spec arm not worse than current 91.7%.
- Latency overhead < 30% vs free decoding.

### 5.4 Decision unlocked

With gv% pinned at 100%, two arms become testable:
- `grammar_taught_prompt + free_decoder` (current best)
- `content_only_prompt + grammar_constrained_decoder` (new — prompt no longer wastes tokens teaching the grammar)

If the content-only prompt + constrained decoder matches the grammar-taught prompt + free decoder, we save substantial input tokens per request and the prompt becomes maintainable. This is the highest-leverage simplification on the table.

---

## 6. Phase 3 — Schema redesign around copy mode

**Premise:** the current v1 DSL treats all slots symmetrically. The literature says verbatim copies and derived values behave fundamentally differently in the decoder. Make that explicit in the schema.

### 6.1 v2 DSL design principles

1. **Copy slots come first.** Every block has a "copy zone" (verbatim spans from input) before any "derived zone" (normalized values, type tags, classifications).
2. **Every extracted value has a provenance pointer.** `source_span: [char_start, char_end]` referencing the input string. Required, not optional.
3. **Derived values come after the verbatim they derive from.** `value: "$5.3 billion"` then `value_normalized: 5_300_000_000` then `value_unit: "USD"`.
4. **Numerics are first-class.** Separate `NUM` block type rather than burying numbers inside `EVT` or `CLAIM`.
5. **No paraphrasing in copy slots.** Grammar-enforced where possible (e.g., `name` must appear as substring of input — checked by verifier).

### 6.2 v2 schema sketch

```
ENT <ident>
  # COPY ZONE
  name: "<verbatim from input>"
  source_span: [start, end]
  # DERIVED ZONE
  type: ORG | PERSON | INSTRUMENT | LOCATION | OTHER
  ticker: "<verbatim or null>"
  ticker_source_span: [start, end] | null

NUM <ident>
  # COPY ZONE
  raw: "<verbatim numeric expression from input>"
  source_span: [start, end]
  context_span: [start, end]   # surrounding sentence for downstream
  # DERIVED ZONE
  value: <float>
  unit: "USD" | "EUR" | "percent" | "bps" | "count" | ...
  magnitude: ONES | THOUSANDS | MILLIONS | BILLIONS | ...

EVT <ident>
  # COPY ZONE
  trigger: "<verbatim trigger phrase from input>"
  trigger_span: [start, end]
  # DERIVED ZONE
  type: EARNINGS | GUIDANCE | M_AND_A | REGULATORY | MACRO | ...
  participants: [<ent_ref>, ...]
  numerics: [<num_ref>, ...]
  time: <iso_date or "T+0">

CLAIM <ident>
  # COPY ZONE
  evidence_span: [start, end]   # the sentence(s) supporting this claim
  # DERIVED ZONE
  subject: <ent_ref>
  predicate: REPORTED | GUIDED | ANNOUNCED | EXCEEDED | MISSED | ...
  object: <ent_ref | num_ref>
  polarity: + | - | 0
```

### 6.3 The hard rule that makes verification trivial

Every value in a copy slot **must** be a verbatim substring of `input_text[source_span[0]:source_span[1]]`. If it isn't, the block is rejected by the verifier. This single constraint:

- Eliminates Benford's-curse-style digit drift (the digit either appears in the source span or it doesn't).
- Eliminates entity paraphrasing.
- Lets the GPU verifier check a string equality, no LLM call needed.
- Provides natural per-fact provenance for downstream sector scoring.

### 6.4 Acceptance

- `qwen3:30b` on v2 spec arm: `numeric_exact_match_rate ≥ 0.85`, `ticker_recall ≥ 0.40`, `org_recall ≥ 0.50`.
- Verifier (Phase 4) can validate 100% of copy slots without an LLM call.

---

## 7. Phase 4 — GPU verifier (the 5090 side)

**Premise:** CPU does pattern-matching extraction. GPU does judgment. Don't mix them.

### 7.1 Deterministic tier (no LLM, runs first)

Inputs: v2 DSL document + original article text.
For each block:
- Check every copy slot's value matches `input[source_span]` byte-for-byte.
- Check every `<ent_ref>` and `<num_ref>` resolves to a declared block.
- Check `value_normalized` matches `raw` + `unit` + `magnitude` (e.g., "$5.3 billion" + USD + BILLIONS → 5_300_000_000).

Reject any block that fails. Emit `pass / soft_fail / hard_fail` per block.

This catches the hallucination class without an LLM call. Cheap, fast, deterministic.

### 7.2 Semantic tier (LLM, runs on hard cases only)

Only invoked for blocks that passed deterministic but where downstream needs a judgment:

**Entity resolution:** ENT block name → instrument catalog entry → NAICS sector + industry.
- Currently a fuzzy-match against the catalog; upgrade to embedding-similarity with qwen2.5:32b as the embedder if the existing logic underperforms.
- Output: `(sector_code, industry_code, confidence)`.

**Claim verification:** for each CLAIM, prompt the GPU model:
- "Does the span `<evidence_span>` support the claim `<subject> <predicate> <object>`? Answer yes/no/partial."
- Use Claude 4.7 on Foundry or qwen2.5:32b locally; A/B benchmark both.
- Output: `support ∈ {full, partial, none}`, `confidence ∈ [0,1]`.

**Sector tagging:** for each validated EVT, derive sector/industry from participants and event type.
- Single-shot prompt with NAICS taxonomy in context.
- Output: ranked list of sector codes with confidences.

### 7.3 Acceptance

- 100% of v2 copy slots pass deterministic verification or are flagged.
- Claim verifier agrees with human spot-check on ≥85% of n=50 sample.
- End-to-end: per-article fact yield (validated NUMs + ENTs that resolve to catalog) is measured and tracked.

---

## 8. Phase 5 — Integration with ATLAS matrix

Once Phases 1-4 land, wire the pipeline:

```
SentinelCollector
  └─ raw article + source_id + published_at
       │
       ▼
CPU extraction (qwen3:30b on ollama-cpu-gen, v2 DSL, grammar-constrained)
  └─ DSL document
       │
       ▼
Deterministic verifier (Python, no LLM)
  └─ validated blocks + rejected blocks (logged for analysis)
       │
       ▼
Semantic verifier (5090 GPU, qwen2.5:32b or Foundry Claude)
  └─ blocks with sector/industry tags + claim confidences
       │
       ▼
ThresholdEngine
  └─ per-cell signal updates: (sector × industry × time) ← signal_magnitude
       │
       ▼
ATLAS matrix
```

### 8.1 What feeds sector/industry scoring

For each article:
- Validated NUM blocks where context names a sector or industry → magnitude contribution
- Validated EVT blocks with non-zero polarity → direction
- Claim confidence < threshold → downweight or discard

### 8.2 What does NOT go in the matrix

- Unvalidated claims
- Numerics that failed the deterministic copy-slot check
- Entities that didn't resolve to the catalog

This is the **safety property**: the matrix never receives a value that wasn't anchored to a source span. Provenance is end-to-end.

### 8.3 Acceptance

- End-to-end demo on n=10 articles from R2/R3 corpus.
- Per-article wall time within production budget (TBD — see open decisions below).
- Per-cell signal updates auditable back to source articles.

---

## 9. Out of scope

- ✗ Touching production prompts at `/opt/ai-inference/prompts/`
- ✗ LoRA retraining on DSL data (Phase 1.3 decides whether this is needed; the retraining itself is a follow-on project)
- ✗ ATLAS matrix UI/visualization changes
- ✗ Changes to SentinelCollector (input side fixed for this PoC)
- ✗ ThresholdEngine reweighting (consume signals as-emitted; tuning is downstream)
- ✗ Production deployment to Mercury (PoC runs on existing benchmark infra)
- ✗ Adding new model classes (Class C is the last addition; A/B/C is the matrix)

---

## 10. Open decisions requiring human input

Claude Code should pause and surface these rather than deciding alone:

1. **Production latency budget.** What's the per-article wall-time ceiling for the production pipeline? This determines whether qwen3:30b's ~50s p50 is acceptable or whether we need a smaller class A model with the v2 schema doing the heavy lifting.
2. **Retire prose CoD?** After Phase 1.1 (direct-extraction arm), the data may say prose CoD is obsolete. Decision is human's — there may be legacy reasons to keep a prose path.
3. **LoRA retraining priority + objective.** Any future LoRA needs a training
   objective that rewards source-anchored extraction, not output plausibility —
   sentinel-cod-v6 demonstrates the failure mode of optimizing for the latter
   (overfit the prose CoD shape, lost recall on the underlying facts). If we
   revisit LoRA, the loss function must include verbatim-copy + source-span
   fidelity terms. Decision: pursue this before or after Phase 4?
4. **Foundry vs local Claude for semantic verification.** Cost vs latency vs Anthropic API dependency. Need an architectural call.
5. **NAICS vs GICS for the matrix taxonomy.** ATLAS docs reference both. Lock one before Phase 4.

---

## 11. Phase ordering rationale (TL;DR)

```
Phase 0 (instrumentation) ────► blocks 1-5; do first
   │
   ▼
Phase 1 (disambiguating experiments) ────► answers Q1-Q5; ~1 week
   │
   ▼
Phase 2 (grammar-constrained decoding) ────► removes gv% as a variable; ~3 days
   │
   ▼
Phase 3 (v2 schema with copy mode) ────► where the mechanism payoff lives; ~1 week
   │
   ▼
Phase 4 (GPU verifier) ────► what makes the output trustworthy; ~1 week
   │
   ▼
Phase 5 (matrix integration) ────► demonstrates end-to-end; ~3 days
```

Total ~4 weeks of focused effort. Each phase has explicit acceptance criteria; gates are data-driven not calendar-driven. If a phase blows up the assumptions of the next phase, re-plan rather than push through.

---

## 12. References (full survey at `docs/research/dsl-prior-art.md`)

- **GoLLIE** — Sainz et al., ICLR 2024 — schema-as-Python-class IE
- **Code4Struct / CodeIE** — ACL 2023 — code-pretrained LMs as IE engines
- **Outlines** — Willard & Louf 2023 — FSM-based grammar-constrained decoding
- **Faithful CoT** — Lyu et al., IJCNLP 2023 — extract-then-execute architecture (precedent for CPU→GPU loop)
- **Program-of-Thoughts** — Chen et al., TMLR 2023 — disentangle computation from reasoning
- **GopherCite** — Menick et al., DeepMind 2022 — quote-grounded generation
- **FActScore** — Min et al., EMNLP 2023 — atomic-fact decomposition for faithfulness
- **Benford's Curse** — Shao et al. 2025 — FFN neuron-level explanation for numeric hallucination
- **Lost in the Middle** — Liu et al., TACL 2024 — position bias in long contexts
- **In-Context Learning and Induction Heads** — Olsson et al., Anthropic 2022 — circuit basis for copy mode
- **Let Me Speak Freely?** — Tam et al., EMNLP Industry 2024 — schema-order failure mode (counter-evidence to be addressed in design)
- **Chain-of-Draft** — Xu et al. 2025 — token-efficient compressed CoT
- **Sketch-of-Thought** — Aytes et al., EMNLP 2025 — adaptive compressed-CoT routing
