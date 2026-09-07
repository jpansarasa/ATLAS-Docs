# The measurement space is a grid, not a leaderboard

**Status:** design rule. Governs every comparison this harness produces.
Results live in [`BENCHMARKS.md`](BENCHMARKS.md). Raw coordinates and controls live in
`docs/BACKLOG.md`. The swap bar is `CLAUDE.md` §MODEL_ACCEPTANCE.

---

## The rule

A score is a property of a **configuration**, not of the model at its centre. "Which model is
better" is not a question this harness answers directly: every number it produces is a point in an
eleven-axis space, and moving any axis moves the number.

A figure without its coordinate does not identify a run.

---

## The axes

| # | Axis | Recorded in the scorecard? |
|---|---|---|
| 1 | Engine, build, **and the image's dependency stack** | Partly — build is fail-closed (`probe_engine`); `transformers` / `torch` versions are **not** recorded |
| 2 | Model family + size | Yes — `model` + `model_revision` |
| 3 | Weight quantization | Yes — from the checkpoint's own `config.json`, never the repo name |
| 4 | KV cache dtype | Yes — declared; falsy → `not_recorded` |
| 5 | Context length | Yes — probed from `/v1/models`; null on llama.cpp |
| 6 | Concurrency (`--max-num-seqs` + client) | Yes — both halves checked independently |
| 7 | Sampling | Yes — `request_sampling` |
| 8 | Prompt / schema / template bytes | Yes — sha256 in `request_bytes` |
| 9 | Task | Yes — `production_prompt_path` |
| 10 | Eval population + alignment key | Partly — sources recorded, the **key convention** is not a field |
| 11 | Chat template / system prompt | **No** — the sha256 is recorded, but a *derived* template is a different string with the same provenance |

Axes 1 and 11 are the two still stamped by hand, and each has cost a wrong conclusion.

**Axis 1 is not a version number.** vLLM 0.19.0 both can and cannot load Gemma 4, depending on
whether the image ships `transformers` >= 5.16.1 — and `probe_engine` stamps `0.19.0` for both. A
model eliminated by a dependency is indistinguishable from one that failed on merit.

---

## Admissibility

A comparison between two scorecards is admissible only if **one** of:

- their coordinates differ in **exactly one axis**, and the delta is attributed to that axis; or
- it declares itself a **configuration comparison**, names every axis that moved, and attributes the
  delta to **nothing**.

Forbidden:

- quoting a delta whose arms differ on an **unrecorded** axis — unfalsifiable, not merely imprecise,
  because no later reader can discover what moved
- carrying a bare score between documents

---

## Two questions, two designs

**Q1 — attribution.** *Does this model extract better?* Both arms at a **common feasible point**:
the intersection of the two models' feasible configs, which is worse than either model's own best.
Every axis but 2 is pinned. Answers a causal question; does not tell you what to deploy.

**Q2 — deployment.** *Which configuration should we run?* Each candidate at its own best point. The
answer is a **configuration**, never a model, and the delta cannot be attributed to any single axis.

Both are legitimate. Quoting a Q2 delta as a model result is the error this file exists to prevent.
The results in `BENCHMARKS.md` are Q2 by construction, because that is the question a swap asks.

---

## Cross-family comparison is bounded at ~0.06

The chat template **cannot be held fixed across model families** — each model requires its own. So
axis 11 moves with axis 2 by construction in every cross-family comparison. That is a property of
the question, not an oversight better discipline removes.

Axis 11 is worth **~0.05-0.06 F1**, measured: an injected "helpful assistant" system block. That is
larger than KV dtype, engine step and concurrency combined.

**So a cross-family gap below ~0.06 is not a model result.** It may still be a valid configuration
claim. Above the band, the model claim survives.

Templates differ substantively, not cosmetically: Gemma 4's pre-fills a thought channel and closes
it — thinking suppression baked in — while a bare ChatML turn on a thinking-capable model does not.
Those are different experiments wearing the same label.

---

## Axes 2-5 are coupled by the GPU

You cannot hold KV dtype and context fixed while swapping a 32B model for a 27B one at the same
VRAM budget; the feasible set moves with the model. The confound is the **default outcome** of
running each model at its own best configuration, not carelessness.

---

## The feasible set has a floor, and it is quality

A configuration that loads and emits tokens is not automatically a feasible point.

- **q2 and sub-3-bit are banned.** *"They produce tokens, not answers."*
- Never squeeze a model into memory by lowering precision. **"No feasible point on this hardware"
  is a legitimate, final answer** and a better one than a bad score.
- The remedy for a model that does not fit is to report that it does not fit. `llama3.3-70B` earned
  a `CRASH` row because someone crammed it in at q2_K.

**An infeasibility is not a score.** OOM, TIMEOUT and CRASH state something about our configuration.
Recording one as `0.0%` defames the model permanently: six of nine families eliminated that way were
later re-qualified, and not one elimination had been a measurement of the model.

---

## Precision is model-dependent

Same publisher, engine, image and coordinate; only the model changed:

| | Q6_K vs Q4_K_M |
|---|---|
| Gemma 3 | **−0.0278**, 12.2× se, ranges disjoint |
| Qwen3.8-27B | **+0.0117**, 1.2× se, ranges overlapping |

The sign reverses and the separation vanishes. **A precision result does not transfer between
models**; it must be re-taken per model. What generalises is the method, not the answer.

---

## A name is not a coordinate

Read from each checkpoint's own `config.json`:

| Repo name | Actual scheme |
|---|---|
| `Qwen/Qwen2.5-32B-Instruct-AWQ` | `awq`, 4-bit, group 128 — **the only true AWQ** |
| `cyankiwi/Qwen3.8-27B-AWQ-INT4` | compressed-tensors `pack-quantized`, 4-bit int, group 32 — **not AWQ** |
| `unsloth/Qwen3.8-27B-NVFP4` | compressed-tensors `float-quantized`, dynamic **FP8 input activations** |

Two of three are named for a quantization they do not use; the third quantizes *activations*, which
is a different axis from weight quantization. Group 128 against group 32 is a 4x granularity
difference that stayed invisible while both arms were called "AWQ".

---

## The engine axis is open

This project has run three engines — llama.cpp, then Ollama, now vLLM — and re-measured none of the
switches. The engine in production is the residue of the last decision, not a constraint on the
next. `run_model.py` drives any OpenAI-compatible engine by design.

Unscored and legitimate to evaluate: SGLang, TensorRT-LLM, ExLlamaV2/V3, MLC, ktransformers.

**The screen, and it is cheap:** an engine must accept `seed` **and** perform real constrained
decoding — a grammar that *masks sampling*, not one that only drafts speculatively. Without both, no
admissible scorecard can exist on it.

**The VRAM ceiling is an assumption too.** Every feasibility argument above presumes a GPU-resident
engine. This box is 31.8 GiB VRAM **and** 125 GB RAM, 48 threads, 312 GB free disk — a ~125B MoE at
int4 is ~62 GB and fits in RAM. The region above 32B has never been sampled. There the question is
latency, and latency is measurable.

---

## Graduation

This stops being a document the moment a checker refuses a two-scorecard comparison whose
coordinates differ on more than one axis. Until then it is enforced by whoever last read it.

---

## Hard stops

✗ never compare across model families without pinning axes 3-6 at a common point
✗ never let a repo name stand in for the weight-quant field
✗ never treat activation quantization as the same axis as weight quantization
✗ never carry a precision result across models
✗ never read a within-family single-axis delta as a cross-family result
✗ never quote a delta from arms not served alike — fix the serving, do not caveat the number
✗ never treat the eval population or the alignment key as a constant
✗ never write an infeasibility into a quality column
✗ never conclude a family is weak from rows never served at a feasible point
✗ never read a matching engine version as a matching engine
✗ never call a point feasible because it loads
