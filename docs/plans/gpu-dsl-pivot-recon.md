# GPU DSL-Trained Extraction Track — Pivot Scoping

**Created:** 2026-05-23
**Status:** RECON — doc-only, production untouched.
**Trigger:** PR #416 (v11 anti-concept-flood) regressed to 0.824 vs v8 0.907.
  Small-model prompt-engineering track is **CLOSED** per the
  `sentinel-llm-strength-layering` memory contingent.
**Architectural rule (recall):** CoD (small CPU MoE) does pattern-recognition
  + verbatim copy + classification + RAG. GPU (large model) does structured
  DSL emission with grammar in weights or in decoder. CoD and GPU never swap
  responsibilities. [project_atlas_inference_topology, project_sentinel_llm_strength_layering]

---

## 1. GPU infrastructure inventory

### Current vLLM serving

- **Container** `vllm-server` (`docker.io/vllm/vllm-openai:latest`, vLLM 0.19.0 V1).
- **Served model** `Qwen/Qwen2.5-32B-Instruct-AWQ` (`awq_marlin`). Alias
  `sentinel-cove-v6.2` is served-model-name only — base model. LoRA flags
  **removed 2026-05-17** (`deploy.yml:789-792`); 2026-05-12 audit showed base
  4% MAJOR error vs LoRA-v7-e3 20% MAJOR.
- **Runtime args:** `--max-model-len 32768 --max-num-seqs 16
  --gpu-memory-utilization 0.92 --kv-cache-dtype fp8_e5m2
  --enable-auto-tool-choice --tool-call-parser hermes`. No `--enable-lora`.

### VRAM / hardware

- RTX 5090: 32607 MiB total, **30944 used, 1166 free** (steady-state, 0% util).
  92% mem budget already saturated.
- LoRA hot-swap NOT available without redeploy. Re-enabling requires
  `--enable-lora --max-loras N --lora-modules ...` (~2–6 GiB extra VRAM per
  adapter at rank 16–32, tightens the 0.92 mem budget).

### Grammar-constrained decoding — LIVE TEST RESULT

Smoke-tested 2026-05-23 against the running vllm-server:
- `POST /v1/completions` with `structured_outputs: {grammar: 'root ::= ("YES"|"NO")'}`
  → returned `"YES"` (constrained). **WORKS.**
- Same with `extra_body.guided_grammar` → unconstrained "Is the following…".
  Use `structured_outputs`, not `extra_body`.

**Key finding:** vLLM 0.19 + AWQ + `structured_outputs.grammar` is functional
on the live deployment. Unblocks **Path C** with no training, runner change only.
(`project_turboquant_blocked` is about TurboQuant's V1 monkey-patch; unrelated.)

### Existing LoRA training infrastructure

- `/opt/ai-inference/qlora-venv/` — unsloth 2026.4.8, peft 0.18.1, trl 0.24.0,
  transformers 4.57.6. Wired for QLoRA on RTX 5090.
- `SentinelCollector/scripts/train_qlora_unsloth.py` — proven script (produced
  sentinel-cove-v6.1/v6.2, sentinel-extraction-v2/v3/v5, v7-take2). Pins
  acceptance criteria from `SentinelCollector/training/acceptance_criteria.json`.
- Past adapters: rank 16, alpha 16, dropout 0.05–0.1, base
  `unsloth/Qwen2.5-32B-Instruct-bnb-4bit`.

---

## 2. DSL training-data inventory

### Available pairs (article → structured emission)

| Source | Count | Quality | Pair shape |
|---|---|---|---|
| `docs/benchmarks/cod-2026-05-17/corpus.full72.jsonl` | 72 articles | input only | article text only |
| `docs/benchmarks/cod-2026-05-17/ground-truth.jsonl` | 72 rows | **entity lists only** (tickers, companies, sectors, industries, numerics) — **NOT gold DSL** | Opus-4-7 generated; flat list, no DSL structure |
| PR #414 v8 n=72 results | 72 emissions | **66/72 gv-valid**, mean vpr=0.854 (scored-only); 42 high (vpr≥0.9), 12 mid, 12 low, 5 gv-fail, 1 error | qwen3:30b-a3b-instruct-2507 v8 prompt emissions (full DSL v2.1) |
| PR #410 v8 n=9 results | 9 emissions | 9/9 gv-valid, mean vpr=0.907 | same model, curated Arm B set |
| PR #400 v6 n=9 results | 9 emissions | mixed quality | same model |
| `/opt/ai-inference/training-data/sentinel-v6.2-cove.json` | 537 instruction/input/output rows | CoVe extraction training — **NOT DSL** | legacy JSON schema, instrument grounding focus |
| `/opt/ai-inference/training-data/sentinel-v6.1-cove.json` | 14,775 lines (≈?) | same | legacy |
| `/opt/ai-inference/training-data/sentinel-v4-diverse.json` | 409,714 lines (39MB) | same | legacy |
| `/opt/ai-inference/training-data/v7-train-20260425T095810Z.prompt-response.jsonl` | 4,950 rows | v7 master-prompt instruction/input/output/_meta | JSON extraction, NOT DSL |

### Gold-DSL availability

- **Human-verified gold DSL: 0 rows.** Nobody has manually labeled article →
  DSL v2.1 emission pairs.
- **LLM-generated DSL (highest-quality, semi-validated): 51 rows** = 42 high-vpr
  + 9 vpr≥0.9 from n=9 sweeps that didn't overlap n=72. Realistic post-dedup pool
  is the **42 high-vpr v8 emissions** that the deterministic verifier (`docs/benchmarks/cod-2026-05-17/dsl/verifier.py`) green-lit at ≥0.9.
- **LLM-generated DSL (full pool, with quality filter): 54** (high + mid, vpr≥0.7).
- **Ground-truth entity lists: 72 rows** — usable for *evaluation*
  (the verifier already uses them), NOT for SFT training input.

### Generation cost for n≈1000

- v8 baseline ran 72 cells in 5h 5min wall = ~254s mean, 116s median wall on
  qwen3:30b-a3b llama-server `--parallel 1`.
- Scaling: 1000 cells × 254s = ~70h wall on the current solo backend. With
  `--parallel 3` (PR #403) ≈23h. With vLLM (AWQ 32B + paged attention +
  continuous batching) → estimate **3–6h** at `--max-num-seqs 16`.
- Cost of human verification: even spot-check at 10% = 100 articles × 5min
  manual labour = ~8h focused work. Full human curation (1000 × 15min) = 250h.
- Cost of LLM auto-label via Claude Opus 4-7 (Azure Foundry — see
  `feedback_azure_foundry_vs_anthropic_cost` memory): n=1000 articles × ~1.5K
  input + 1K output tokens ≈ $3–8 total at Foundry pricing.

---

## 3. Training-path options

### Path A — Local QLoRA fine-tune on existing infra

- **Approach:** Reuse `train_qlora_unsloth.py` against
  `unsloth/Qwen2.5-32B-Instruct-bnb-4bit` (same base as v6.1/v7), with
  `{instruction, input: article, output: DSL}` pairs. Rank 16, alpha 16, ~3
  epochs. Re-enable `--enable-lora` on vllm-server.
- **Infra:** Mercury RTX 5090 qlora-venv (v6.1/v7 are proof-of-existence).
- **Data req:** Bootstrap 51 high-quality v8 emissions + auto-generate 500–1000
  via Opus/Foundry, filter by verifier ≥0.9 + entity-recall ≥0.8.
- **Cost:** ~$5–10 Foundry + ~6–12h training wall. Inference cost zero.
- **Timeline:** ~3 days (1d prep, 1d train, 1d eval).
- **Risk:** HIGH. The 2026-05-12 audit explicitly showed LoRA over-extracted
  (base 4% vs LoRA-v7-e3 20% MAJOR). That LoRA was trained on un-audited Opus
  labels. Path A repeats the workflow with verifier pre-filter only; if Opus
  systematic bias is the root cause, the verifier won't catch it.

### Path B — Cloud LoRA fine-tune (cleaner data, larger sweep)

- **Approach:** Same as Path A but on cloud GPU (A100 80GB ≈ $1.50/h, H100
  ≈ $2–4/h). Larger batch + longer seq. Generate 2K–5K pairs, rank 32, 5 epochs.
- **Data req:** 2K–5K pairs with explicit anti-bias prompts + per-source
  stratification + cross-validation hold-outs.
- **Cost:** $50–150 GPU + $10–30 data-gen + ~12h setup.
- **Timeline:** ~1 week (most overhead is data-quality engineering).
- **Risk:** Medium. Larger data + bias guards address Path A's failure mode,
  but same fundamental question: is auto-labeled LLM output trustworthy enough
  to train on? Honest answer: we don't know because we never built the
  bias-detection harness.

### Path C — Grammar-constrained decoding on base AWQ (NO fine-tune) ★

- **Approach:** Compile `docs/grammars/cod-dsl-v2.1.gbnf` and pass it via
  `structured_outputs.grammar` on every vLLM completion. v8 prompt as system;
  the grammar enforces structure at the decoder layer.
- **Infra:** Zero new. §1 smoke test confirms vLLM 0.19 + AWQ honors it.
- **Data req:** Zero training data.
- **Cost:** Engineering only — wire `structured_outputs` into the runner +
  benchmark vs v8 on n=72.
- **Timeline:** ~1d to wire + benchmark; ~2d if grammar tightening needed.
- **Risk:** LOW. Grammar guarantees gv-valid 100% by construction. The
  concept-flood failure that killed v8 at scale is decoder-layer enforceable
  (a `kind=concept` cap of 5 becomes syntactic, not behavioural). 32B base has
  stronger instruction-following than 30B-MoE 4B-active.

### Path D — Validator-driven retry loop

- **Approach:** CPU CoD emits draft DSL → `dsl/verifier.py` scores → if
  vpr<threshold or gv-fail, prepend verifier's complaint and re-emit. ≤3 retries.
- **Cost:** Engineering only ~2d. Timeline ~3d. No training, no GPU.
- **Risk:** The v8→v9→v10→v11 sweep already did this in batch form. v11
  NOTES shows the failure mode: **mechanism guards displace, they don't
  eliminate** (concept cap displaced to event-flood; cell 33632 dropped from
  vpr 0.969 to 0.000). Retry-with-complaint requires the small model can act
  on the complaint — the layering memory says it can't. Likely converges to
  v8 ceiling at 3× latency.

### Topology check

All four paths respect `project_atlas_inference_topology`: CoD stays on
CPU-MoE, GPU handles structured emission (or in Path D, GPU is unused).
**Path C is the cleanest topology fit** — GPU does grammar-constrained
emission consuming CPU-CoD output.

---

## 4. Two-stage pipeline sketch (Path C target)

Per the layering rule:

- **CPU CoD** (qwen3:30b-a3b-instruct on ollama-cpu-gen)
  → article-type classification + verbatim entity/numeric/quote copy +
  RAG few-shot retrieval (k=2 leave-one-out over labeled corpus) →
  emits annotated source bundle.
- **GPU emission** (vllm-server, Qwen2.5-32B-Instruct-AWQ)
  → `structured_outputs: {grammar: cod-dsl-v2.1.gbnf}` + minimal system
  prompt (article-type→kind-priority steering; behaviour-reasoning offloaded
  to grammar) → emits DSL v2.1, gv-valid by construction.
- **Deterministic verifier** (`dsl/verifier.py`)
  → entity recall vs `ground-truth.jsonl`, numeric exact+semantic match,
  span backfill via `input_text.find()`, vpr aggregate. gv-valid is now
  100% by grammar.

### Interface (CoD → GPU)

CoD passes a compact annotated bundle (NOT free prose) to GPU:

```json
{
  "article_text": "...verbatim original...",
  "article_type": "earnings_release | fed_speech | analyst_note | macro_print | other",
  "entities": {
    "tickers": ["..."],
    "companies": ["..."],
    "numerics_with_units": ["..."],
    "key_quotes": ["...verbatim...", "..."]
  },
  "rag_neighbours": [
    {"source_id": "...", "dsl": "...full DSL v2.1..."},
    {"source_id": "...", "dsl": "..."}
  ]
}
```

The GPU prompt becomes ~50 lines of role + grammar pointer + neighbour
injection — not the 846-line v8 prompt. The grammar replaces 80% of the prompt's
mechanism guards.

---

## 5. Recommended sequencing

### Rank-ordered options

1. **Path C (grammar-constrained decoding on base AWQ)** — START HERE.
   - Lowest risk, fastest signal, zero training. The live smoke test in §1
     confirms the mechanism works. Wire `structured_outputs` into a new sweep
     variant on the n=72 corpus and compare aggregate vpr vs v8 0.7826.
   - If Path C clears the 0.85 penalized threshold, **we're done** for the
     extraction stage and the GPU pivot doesn't need training at all.
   - If Path C lands 0.78–0.85, we have a much-stronger Path A starting point
     (grammar-anchored draft → tightened by a small LoRA on top of base AWQ).
   - This also exercises the two-stage pipeline shape (§4) without committing
     to fine-tune topology choices.

2. **Path D (validator-retry)** — RUN IN PARALLEL with Path C as a budget-cap
   experiment. Cheap to build, gives a comparison point. If Path D ties or
   beats Path C, that's interesting; if it converges to v8 0.78 (likely), it
   confirms the layering memory's premise that we've hit the small-model
   ceiling and the GPU pivot is mandatory.

3. **Path A (local QLoRA)** — DEFER pending Path C result. Only justified if
   Path C lands below 0.80 AND we can demonstrate bias-detection on the
   auto-labeled corpus that wasn't present in the 2026-05-12 LoRA failure.
   Bias-detection is a meaningful chunk of prep work — building it before we
   need it is gold-plating.

4. **Path B (cloud LoRA)** — DEFER until Path A clears local-scale validation.
   Cloud is justified by data-volume scale (2K–5K pairs) which we don't have
   yet and can't legitimately auto-generate without bias controls.

### Recommended first move

Open a new experiment branch (`experiment/gpu-path-c-grammar-decode-n72`) that:
1. Adds a `--structured-grammar <path>` flag to `scripts/run_cod_dsl.py`.
2. Translates the live `dsl/grammar-v2_1.lark` to GBNF for vLLM
   (or reuses `docs/grammars/cod-dsl-v2.1.gbnf` if it's already current).
3. Runs the n=72 corpus through Qwen2.5-32B-AWQ on vLLM with grammar constraint
   + the v8 prompt (or a trimmed variant since grammar replaces mechanism guards).
4. Compares against the PR #414 v8 n=72 baseline (0.7826 penalized / 0.8538 scored).

Wall clock: ~6h sweep + 1d engineering = **3 day spike**.

---

## Open questions for user

1. **Path C vs Path A bet:** Is the user willing to commit to a grammar-first
   path (Path C) before spending training compute? The historical pattern has
   been "iterate the prompt → run sweep → regress" 11 times. Path C is the
   first attempt that moves the structural constraint OUT of the prompt and
   INTO the decoder. Recon recommends Path C-first; user-approval required to
   close the prompt-iteration door.

2. **What counts as "gold DSL"?** We have 0 human-verified DSL emissions and
   ~50 LLM-emissions the verifier scored ≥0.9. If we go Path A/B, do we
   accept verifier-passing emissions as gold? If so we need a bias-detection
   harness; if not we need a human-labeling budget (~250h for n=1000).

3. **Cloud GPU budget:** Path B assumes the user is willing to spend
   $50–150 on rented A100/H100 time. Confirm? (Mercury can probably do Path
   A on its own given v6.1/v7 trained successfully here — Path B is only
   needed if data-volume scaling becomes the bottleneck.)

4. **Re-enabling `--enable-lora` on vllm-server:** Even Path C doesn't need
   this. But Path A/B require an ansible change to add `--enable-lora
   --max-loras 2 --lora-modules sentinel-dsl-v1=/opt/ai-inference/models/...`
   to `deploy.yml`. This costs ~2–6 GiB VRAM (already at 92% util). Confirm
   that pushing past 0.92 is acceptable, or that we drop `--max-num-seqs`
   from 16 to 12 to make room.

---

## Appendix: live data points used

- vLLM probe: model=`sentinel-cove-v6.2` aliased to `Qwen/Qwen2.5-32B-Instruct-AWQ`
- vLLM args: see §1 (no `--enable-lora`). VRAM: 30944/32607 MiB.
- Grammar smoke: `structured_outputs.grammar` → "YES" (constrained).
- v8 n=72: 42 high-vpr, 12 mid, 12 low, 5 gv-fail, 1 err — see
  `docs/benchmarks/cod-2026-05-17/results/phase3-v8-baseline-n72-PhaseA/NOTES.md`.
- v11 regression: 0.824 vs v8 0.907 (n=9) —
  `docs/benchmarks/cod-2026-05-17/results/phase3-v11-anti-concept-flood-n9/NOTES.md`.
- Training infra: `SentinelCollector/scripts/train_qlora_unsloth.py` +
  `/opt/ai-inference/qlora-venv/`. LoRA-removal rationale:
  `deployment/ansible/playbooks/deploy.yml:789-792`.
