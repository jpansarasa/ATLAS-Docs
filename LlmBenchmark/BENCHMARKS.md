# LLM Extraction Benchmarks

Extraction quality for ATLAS Sentinel, measured on production's own CoD path.
Method and comparability rules: [`MEASUREMENT_SPACE.md`](MEASUREMENT_SPACE.md). Raw coordinates,
controls and sample-level detail: `docs/BACKLOG.md`. Swap criteria: `CLAUDE.md` §MODEL_ACCEPTANCE.

---

## Results

`numbers_f1`, 40 gold articles, production's CoD prompt path, vLLM with `response_format`
json_schema. Higher is better. Coordinate columns are part of the measurement — a score belongs to
a configuration, not to a model.

| Model | numbers_f1 | sd | n | vs production | Engine | Weight quant | Context |
|---|---:|---:|---:|---:|---|---|---:|
| **Gemma 4 31B** | **0.7570** | 0.0009 [*] | 3 | **+0.2417** | vLLM 0.28.0 | compressed-tensors INT4 g32 | 32,768 |
| **Qwen3.8-27B** | **0.7132** | 0.0109 | 3 | **+0.1979** | vLLM 0.19.0 | compressed-tensors INT4 g32 | 32,768 |
| **Gemma 3 27B** | **0.6177** | 0.0130 | 3 | **+0.1024** | vLLM 0.19.0 | compressed-tensors INT4 g128 | 32,768 |
| Qwen2.5-32B-AWQ | 0.5153 | 0.0106 | 5 | — *(production)* | vLLM 0.19.0 | AWQ 4-bit g128 | 32,768 |
| Mistral-Small 24B | 0.5105 | 0.0032 | 3 | −0.0048 | vLLM 0.19.0 | AWQ 4-bit g128 | 32,768 |
| Command-R 08-2024 | 0.3178 | 0.0098 | 3 | −0.1975 | vLLM 0.19.0 | AWQ 4-bit g128 | 32,768 |
| EXAONE 4.0 32B | 0.2218 | 0.0089 | 3 | −0.2935 | vLLM 0.19.0 | AWQ 4-bit g128 | 32,768 |
| GLM-4.7-Flash | *no score* | — | — | — | vLLM 0.19.0 | compressed-tensors INT4 g32 | 32,768 |

[*] One serve session's replicate sd, not a reproducibility figure — see the Gemma 4 bullet
under *Serving coordinates* below.

**Every "Weight quant" cell is read from the served checkpoint's own `config.json`, not from a
sidecar and never from a repo name** (rule 5). The scorecards resolve this axis for some arms and
record `weight_quantization: null, source: unresolved` for others — Gemma 4 among them, because its
checkpoint sat outside the cache the runner scans — so the column is a checkpoint read that a reader
can repeat: `quantization_config.weights.{num_bits,group_size}` plus `quant_method`. Three of the
eight rows were previously labelled from their repo name and are corrected here — Gemma 4 (named
`…qat-w4a16-ct`; the config contains neither string and the group is **32**, not the 128 "w4a16"
connotes), EXAONE and GLM-4.7-Flash (both labelled `Q4_K_M`, a llama.cpp k-quant, while both were
served to **vLLM** from HF checkpoints: EXAONE genuine AWQ 4-bit g128, GLM `compressed-tensors` INT4
g32 despite `AWQ-4bit` in its repo name). `Q4_K_M` is correct only in *Other measurement tracks*
below, which is the llama.cpp arm.

All three leaders beat production by more than the ~0.06 confound band, so each of those gains over
production is real. The order among the leaders is not: the top two are 0.0438 apart — inside that
same band — and were measured on different engines. **That order has since been re-taken
single-axis and it still does not separate** — see *One common coordinate* below, which removes the
engine confound and reaches the same verdict for a different reason.

**Mistral-Small is the control.** It lands within noise of production despite being re-measured on
the same engine and decoding path as every row but Gemma 4 — which is what rules out "these gains
are an artifact of the new harness" for the rows on 0.19.0. It does not cover the engine axis on
Gemma 4, whose +0.2417 stays a configuration delta with engine and model moving together.

### Serving coordinates required to reproduce these numbers

Not preferences — without them the score above is not what you get.

- **Qwen3.8-27B requires thinking disabled.** Its vendor chat template enables reasoning by default;
  at production's 4,096-token completion budget that leaves the JSON unclosed on **110 of 120
  documents** and the run is unscoreable. The 0.7132 is the thinking-off arm.
- **Gemma 4 31B needs vLLM 0.28.0, and no *stock-derived* 0.19.0 image serves it** — re-pinned
  wheels, nothing else; patching vLLM source was not tried. Measured 2026-09-07; the arm above
  stands on stock `vllm/vllm-openai:v0.28.0` (`transformers` 5.15.1). 0.19.0 is not short a
  dependency, it is short a **capability**, so Gemma 4 stays the one arm off production's engine and
  the +0.2417 remains a configuration delta with engine and model moving together. Detail in
  *An engine axis that a version bump cannot cross* below.
  **Its 0.0009 sd is not a reproducibility figure.** The same arm at the same coordinate was re-taken
  in a second serve session and scored 0.7346; pooled over both, n=6, it is **0.7458 sd 0.0130**. The
  0.7570 above is one session's mean and is left as measured — see *One common coordinate* below.
- **EXAONE 4.0 fails to terminate** inside the 4,096-token budget on 8–9 of 40 articles. Those
  score zero and are included in its 0.2218.

---

## One common coordinate, one engine [2026-09-07]

Every row in the table above is either cross-engine or inside the confound band, so no single-axis
model comparison existed. This is that comparison: three models, **one coordinate**, on the latest
vLLM release. It does **not** replace the rows above — for the two Qwen rows it is a different
serving point and the two sets are not interchangeable. **Gemma 4 is the exception**: its pinned row
is at *this* coordinate, so those two numbers are two samples of one point, and what they do is
below.

| Model | numbers_f1 | replicate sd | n | article-level 95% CI | precision | recall |
|---|---:|---:|---:|---|---:|---:|
| **Gemma 4 31B** | **0.7346** | 0.0065 | 3 | [0.6611, 0.7955] | 0.7292 | 0.7400 |
| **Qwen3.8-27B** (thinking-off) | **0.6481** | 0.0012 | 3 | [0.5477, 0.7326] | 0.6062 | 0.6963 |
| **Qwen2.5-32B-AWQ** (production) | **0.5131** | 0.0231 | 3 | [0.4180, 0.5982] | 0.5537 | 0.4781 |
| Qwen2.5-32B-AWQ *at its own vendor template* | 0.4605 | 0.0206 | 3 | [0.3664, 0.5394] | 0.4920 | 0.4331 |

**The shared coordinate, and each half of it says where it comes from** — the sidecars carry some of
this and not the rest, and one sentence claiming all of it was how an unrecorded axis stayed
unrecorded once already.

*Read back from all twelve provenance sidecars, byte-identical across them* (`engine` `vllm`
`0.28.0` on a fail-closed probe · `--kv-cache-dtype fp8_e4m3` · `--max-model-len 32768`, **probed**
from `/v1/models` · `--max-num-seqs 6` · client `--concurrency 6` · temperature 0, seed 42,
`repetition_penalty` 1.1, `max_tokens` 4096, every other sampling knob `null` = not sent ·
`--task cod --endpoint-mode completions` · `cod_json_v1.txt` sha `0dd66dde…` +
`cod_json_schema_v1.json` sha `1b971904…` · substrate sha `008c338d…` · `limited: false` ·
`production_prompt_path: true`). *Read from the run's own coordinate and boot records* —
`coord.*.txt` and `boot.*.log`, untracked under `/tmp/sentinel-remediation/common-coordinate-latest/`
— `transformers` 5.15.1 / `torch` 2.13.0+cu130 (axis 1: `MEASUREMENT_SPACE.md` records these as
**not** in the sidecar), `--gpu-memory-utilization 0.90`, `--generation-config vllm`, and everything
in *Serving notes* below. *Verifiable in this repo, not in a sidecar*: the gold sha `bc9c5b4c…` is
`sha256sum LlmBenchmark/cod-gold/cod_stage1_gold_v1.json` — no scorecard field carries it.

**What moves, in full.** Model, revision and chat template move by design. Four more move as
engine-forced consequences of the model, exactly as axis 11 does: compute dtype (`torch.float16` for
the incumbent, `torch.bfloat16` for both challengers), the resolved weight-quantization method
(`auto_awq` for the incumbent, `compressed-tensors` for both challengers), `max_num_batched_tokens`
(2,048 for the Qwens; the engine raises Gemma 4 to 2,496 for its prefix-LM/video path) and the
attention backend (`FLASHINFER` on both Qwens; Gemma 4's heterogeneous head dims force
`TRITON_ATTN`). None is settable at a common value while the model moves, which is the point — but
"only model, revision and chat template move" was too few, and the four are read from the boot logs,
not from a sidecar.

**REPLICATE sd IS NOT THE UNCERTAINTY, and reading it as one is how this file's earlier gaps got
overstated.** Three reruns over the *same* 40 articles measure decoding jitter; between-article
variability is the dominant term and is invisible to them. Each arm's sd is 3.9–77x smaller than the
half-width of its article-level CI. The CIs above and the deltas below come from a **paired
article-level cluster bootstrap**, 2,000 draws, resampling the 40 articles and scoring every arm on
the same draw with the harness's own scorer.

**AND THE SAME ARM SCORED 0.7570 ABOVE — SAME COORDINATE, NOT A DIFFERENT SERVING POINT.** The
disclaimer opening this section holds for the Qwen rows (0.19.0, seqs 16 / util 0.95) and is **false
for Gemma 4**, whose pinned row was taken at exactly this section's point. Two samples of one
coordinate, 0.7570 (sd 0.0009) against 0.7346 (sd 0.0065): −0.0224, or 3.5x the replicate sd.

**No axis differs — that was checked, not assumed.** Both runs' `coord.*.txt` agree on engine and
image, `transformers` 5.15.1 / `torch` 2.13.0+cu130, model and revision `52f3f65b…` out of the same
checkpoint directory, all five serve flags, chat-template sha `7a8a39d2…`, prompt/schema/substrate
shas, sampling, and client concurrency 6. The engine-config line vLLM logs at boot is **identical
between the two boots** — `torch.bfloat16`, `compressed-tensors`, `TRITON_ATTN`,
`MarlinLinearKernel`, `max_num_batched_tokens` 2,496, chunked prefill and prefix caching on, the same
cudagraph sizes, the same forced `--disable_chunked_mm_input` — down to **GPU KV cache size 68,892
tokens** and **29,018 / 3,070 MiB** VRAM in both. `eval_harness.py` is sha `831c4be1…` in both, the
gold is `bc9c5b4c…` in both, and `usage.prompt_tokens` is **86,633** in both, so the requests were
byte-identical. The gap is in generation, and the coordinate record does not explain it because there
is nothing in it left to explain it with.

**The sd is what is wrong, and the mechanism is the serve session.** Each triple ran back-to-back
against **one** vLLM boot, sharing that boot's prefix cache and one batch-composition history, so a
replicate sd is a **within-session** statistic. The cache was warm on two of the three boots —
51.1–67.4% hit rate on the Gemma 4 boot and 55.0–70.8% on the Qwen2.5 boot that served both its
triples — and **cold on the Qwen3.8 boot, 0.0% on all 49 of its engine metric lines**, which leaves
batch composition as that arm's whole share of the mechanism. Measured on the predictions: within a
session the replicates return identical text on 9–13 of 40 records (pinned) and 13–18 of 40 (here);
**across** the two sessions the same pairs agree on only **4–11 of 40**. The pinned arm's 0.0009 is
not evidence of determinism — its outputs were the *least* stable of the six runs, and three
differing output sets happened to score within 0.0018 of each other.

**Corroborated on the incumbent, which has its own cross-session pair at this coordinate.** The
earlier sweep's 0.28.0 control — same model and revision, seqs 6 / util 0.90, `fp8_e4m3`, 32,768,
the same vendor template `d95247e8…`, KV 57,040 tokens, `FLASHINFER` — scored **0.4545** (sd 0.0162);
the `q25_vendor` arm here scores **0.4605** (sd 0.0206). **+0.0060 across sessions, 0.29 sd.** The
one declared difference between them is inert: the control passed `--quantization awq_marlin` and the
engine resolved it to `quantization=auto_awq` regardless, the same value the unflagged arm here
resolved to. So the between-session term is present for both models. It merely hides inside the
incumbent's larger within-session sd and cannot hide inside Gemma 4's 0.0009.

**So quote the arm at this coordinate as n=6: mean 0.7458, sd 0.0130, range 0.7280–0.7577.** Neither
0.7570 nor 0.7346 is wrong and neither is the reproducible figure. Nothing above turns on it —
0.0224 is a third of the half-width of this arm's article-level CI and an order below its +0.2214
over production. **A replicate sd prices decoding jitter inside one serve session. It is not a
reproducibility figure, and no number of reruns inside that one session makes it one.**

| Comparison | Δ | 95% CI | Verdict |
|---|---:|---|---|
| Gemma 4 − production | **+0.2214** | [+0.1192, +0.3288] | **established** |
| Qwen3.8 − production | **+0.1351** | [+0.0616, +0.2183] | **established** |
| **Gemma 4 − Qwen3.8** | +0.0862 | **[−0.0034, +0.1797]** | **crosses zero — NOT established** |
| production ChatML − Qwen2.5 vendor template | +0.0518 | [−0.0014, +0.1146] | crosses zero — not established |

**Both challengers beat production single-axis; the order between them is still not a result.** The
Gemma 4 − Qwen3.8 gap is now *outside* the ~0.06 band and 13x the replicate sd, and it still fails
once the 40 articles are resampled. Closing it needs **more articles, not more reruns**.

**The confound band, re-derived not imported.** The fourth row is the same model at the same
coordinate under its own vendor template, whose only material difference from production's bare
ChatML is an injected "You are Qwen … helpful assistant" system block — `MEASUREMENT_SPACE.md`'s
own axis-11 intervention. Measured here at **+0.0518 [−0.0014, +0.1146]**: the point estimate
corroborates the ~0.05–0.06 magnitude, the interval does not establish it at stage 1. Direction is
worth recording — production's bare ChatML **beat** the vendor template.

**Abstention is not what separates these models at stage 1.** All-arrays-empty is **0/120 in every
arm**; `numbers[]`-empty is 21/120 for production against 24/120 for both challengers — a 2.5-point
spread running the *opposite* way to the ~51%-vs-9% measured on stage-2 positives, which is a
**different task, gold and sweep** (the stage-2 CoVe run, not these twelve) and is quoted here only
for its direction. `numbers_f1` is a
micro-averaged harmonic mean and *would* reward abstention, so precision and recall are tabled above:
production is low on **both**, and Gemma 4 leads on **both** simultaneously. Every arm is a true n=3
(3 distinct outcomes of 3 runs; runs agree on only 10–20 of 40 records, because temperature 0 + seed
42 does **not** make a concurrent 40-document run reproducible — continuous batching varies batch
composition).

**Serving notes.** Qwen3.8 needs its **vendor template with thinking explicitly OFF** —
`…assistant\n<think>\n\n</think>\n\n`. Deriving a template by rendering the tokenizer's own
`apply_chat_template` **without** `enable_thinking=False` silently yields the thinking-ON template,
because Qwen3 accepts the kwarg and defaults it True rather than raising. Gemma 4's derived template
reproduces the pinned sha `7a8a39d2…` of its 0.7570 row exactly.

**Weight quantization, read from each checkpoint's own `config.json`** — a checkpoint read, not a
sidecar read: the runner resolves this axis for the two Qwen arms and records `weight_quantization:
null, source: unresolved` for Gemma 4, whose checkpoint sat outside the cache it scans. Production is
the only true AWQ (`quant_method awq`, 4-bit, **group 128**). **Both challengers are
`compressed-tensors` `pack-quantized` INT4 at group 32**, differing only in symmetry — Gemma 4
symmetric with no zero-point, Qwen3.8 asymmetric with an int8 one. So axis 3 does **not** separate
the two leaders on bits or granularity; it separates the incumbent from both, and an earlier
"cannot be equalised" overstated it. Gemma 4's activations are unquantized (`input_activations:
null`) and its `kv_cache_scheme` is null.

The engine reported `quantization=auto_awq` for the incumbent where production pins `awq_marlin`,
then logged `Using MarlinLinearKernel for AutoAWQMarlinLinearMethod`. **On 0.28.0 the distinction is
a spelling**: the earlier control passed `--quantization awq_marlin` explicitly and its boot resolved
to `quantization=auto_awq` too. Whether 0.19.0 does the same is untested. The engine also forced
`--disable_chunked_mm_input` for Gemma 4. KV capacity is **57,040 (production) / 161,412 (Qwen3.8) / 68,892 (Gemma 4)** tokens at an
identical setting, and the **weights are not what drive it**: Qwen3.8's are *larger* than
production's (19.57 vs 18.00 GiB of safetensors) and it still gets 2.8x the cache. Per-token KV
footprint is an architecture property, not a weight-size one. All of this paragraph is read from the
boot logs and the checkpoints; none of it is in a scorecard.

**0.28.0 is the latest release and it did not fault.** From the GitHub and Docker Hub responses the
run captured (`gh-latest.json`, `dh-latest.json`, `dh-v0280.json`, kept with the boot logs, not in a
scorecard): `releases/latest` → `v0.28.0` (`prerelease: false`); Docker Hub's `latest` carries the
identical digest; `v0.28.1`, `v0.29.0` and `v0.29.0rc4` are 404 as images — the 0.29 RCs are git tags
only. Three model boots (85 / 135 / 165 s, from the boot logs) served **480** requests at concurrency
6 — twelve runs of 40, the incumbent's two template arms out of one boot — with **0 call errors, 0
`schema_invalid`, 0 truncations, `json_valid` 1.0**, and those five *are* sidecar fields.
`fp8_e4m3` on every arm per `CLAUDE.md` §VLLM_UPGRADE, now exercised on three models rather than one.

**Not established by this run:** the order between the two leaders; whether any of it holds at
production's own serving point (seqs 16 / util 0.92, untested for the challengers); whether
`awq_marlin` and `auto_awq` differ on **0.19.0**, production's engine — on 0.28.0 they resolve to the
same config and the same kernel, so only the older engine is open; anything about releases between
0.19.0 and 0.28.0; and **how wide the
between-session spread really is** — two sessions per model is enough to show the term exists and
not enough to size it. And the
production arm's 0.5131 here is **not** comparable to the 0.5153 in the table above — that row is
0.19.0 at seqs 16 / util 0.95, and the two being ≈0.51 is a coincidence a reader must not act on.

---

## Not viable on this hardware

Recorded as coordinate findings, not scores — the constraint is ours, not the model's.

| Model | Binding constraint |
|---|---|
| llama3.3-70B | 4-bit weights alone are 37–41 GiB on a 31.8 GiB card. Sub-4-bit is banned. No feasible point. |
| Command-R 35B **v01** | No GQA (640 KiB/token) and `max_position_embeddings` 8192 — never a 32K model. Superseded by the 08-2024 build above. |
| GLM-4.7-Flash | Non-terminating unconstrained, empty arrays constrained. A decoding/template interaction. |

---

## An engine axis that a version bump cannot cross

**Question:** does a *stock-derived* vLLM 0.19.0 — stock 0.19.0 with its wheels re-pinned, nothing
else — serve Gemma 4 31B, so the +0.2417 could be re-taken on production's own engine?

**Answer, measured 2026-09-07: no, and no re-pin can change it.** Recorded as a coordinate finding,
never as a score. The arm produced no `numbers_f1` because it never reached a first token.

The derived image the earlier note called untested (`vllm-requal:tx5161` = stock 0.19.0 +
`transformers` 5.16.1) *was* built and *was* pointed at Gemma 4. It failed, and re-running it at the
exact coordinate of the 0.7570 row — `--max-model-len 32768 --max-num-seqs 6
--gpu-memory-utilization 0.90 --kv-cache-dtype fp8_e4m3 --generation-config vllm`, same substrate
sha `008c338d`, same gold sha `bc9c5b4c`, same 69-byte chat template `7a8a39d2` — reproduces the
failure.

`tx5161` is one committed layer over the digest-pinned `vllm_image` in
`deployment/ansible/group_vars/all.yml` — `pip install --no-deps -U transformers==5.16.1 tokenizers
huggingface_hub safetensors`, torch left at 2.10.0+cu129. Rebuild from that line; the image is local
only and one prune from gone.

**The mechanism is not the one the note assumed.** `model_type: gemma4` parses fine, and 0.19.0
registers `Gemma4ForConditionalGeneration` and ships `gemma4.py`. Serving dies earlier, in
`ModelConfig.__post_init__`:

```
# vllm/transformers_utils/model_arch_config_convertor.py, get_head_size(), inside the image
head_dim = getattr(self.hf_text_config, "head_dim", 0)
-> transformers.integrations.heterogeneity ...
   AmbiguousGlobalPerLayerAttributeError: 'head_dim' is a per-layer attribute
```

Gemma 4 31B is genuinely heterogeneous — `head_dim` 256 with 16 KV heads on sliding-attention
layers, `global_head_dim` 512 with 4 on the 10 full-attention layers of 60. `Gemma4TextConfig`
inherits `HeterogeneousConfigMixin` and declares both fields per-layer, so **every** `transformers`
that knows `gemma4` refuses the flat read, and the one that permits it cannot parse the model:

| `transformers` | knows `gemma4`? | permits 0.19.0's flat `head_dim` read? |
|---|---|---|
| 4.57.6 (stock 0.19.0) | no | yes — no heterogeneity module at all |
| 5.15.1 (stock 0.28.0) | yes | **no** — raises |
| 5.16.1 (the derived image) | yes | **no** — raises |

Only the 5.16.1 row is a boot attempt — the one above. The other two were read out of the
`transformers` packages themselves: 4.57.6 ships no heterogeneity module, 5.15.1 carries the same
`Gemma4TextConfig`. Neither was put in front of Gemma 4 on 0.19.0.

That pincer is why no wheel re-pin exists. Of the two escapes one was tried and failed, the other
was ruled out unrun: `--hf-overrides '{"allow_global_per_layer_attribute_access": true}'` does not
reach the config before `ModelConfig` reads it, and setting the flag on the checkpoint would hand a
homogeneous engine one head size for a model with two — a point that loads without being feasible,
which is why it was not attempted.

0.28.0 clears it by *capability*, not by dependency: a per-layer `model_arch_config[layer_idx]`, a
`transformers_utils/configs/gemma4.py` that materialises the per-layer overrides, and
`get_num_kv_heads(arch_config=…)` sized one layer at a time. 0.19.0 represents head size as a single
scalar (`max(head_dim, global_head_dim)`) and has no per-layer path to fix. Closing this would mean
patching vLLM source, which is no longer a stock-derived image and no longer this question.

**What this does not establish.** It says nothing about Gemma 4's quality — this section moves
nothing about the 0.28.0 score, which a later re-run at the same coordinate restates as n=6
0.7458 sd 0.0130 (*One common coordinate*); 0.7570 stands unchanged
on 0.28.0. It does not price the engine step *for Gemma 4*; the incumbent controls put that step at
+0.0005 (0.4540 → 0.4545, well inside noise) on Qwen2.5-AWQ **at Gemma 4's serving point** — seqs 6
/ util 0.90, on the control arm's own chat template, not the seqs 16 / util 0.95 point the 0.5153
row above was taken at, which is why neither level is that row's — and whether an engine result
transfers between models is untested: the non-transfer rule below was measured on **precision**, not
on engines. And it is a statement about **0.19.0**,
not about every engine between it and 0.28.0 — the intermediate releases were not tested.

---

## Other measurement tracks

**These do not convert to the table above.** Different task, metric or engine.

| Track | What it measures | Qwen3.8 | Gemma 3 | Incumbent |
|---|---|---:|---:|---:|
| Substrate `aggregate_f1` | 16 instruction blocks, not production's task | 0.764 | — | 0.443 |
| GGUF ladder, llama.cpp | chat mode, 8,192/slot, Q4_K_M | 0.6745 | 0.6263 | — |
| GGUF ladder, llama.cpp | Q6_K | 0.6862 | 0.5985 | — |
| GGUF ladder, llama.cpp | Q8_0 | 0.6554 | — | — |

Two findings from the ladder that bear on configuration choices:

- **Precision is model-dependent.** Q6_K beats Q4_K_M on Qwen3.8 by +0.0117 (overlapping ranges — no
  measurable difference) but *loses* on Gemma 3 by −0.0278 (12.2× se, disjoint). A precision result
  does not transfer between models.
- **llama.cpp is a measurement path, not a deployment one.** 2.54× slower than vLLM on the same 40
  articles at the same concurrency. It is used here only because vLLM has no usable rung above
  4-bit at 27B.

---

## Comparability rules, short form

Full statement in [`MEASUREMENT_SPACE.md`](MEASUREMENT_SPACE.md).

1. A score belongs to a **configuration**, not a model. Quote the coordinate with the number.
2. A comparison is valid only if the arms differ in **one axis**, or if it declares itself a
   configuration comparison and attributes the delta to nothing.
3. A cross-family gap **below ~0.06** is not a model result — the chat template alone is worth that,
   and it cannot be held fixed across families.
4. **OOM, TIMEOUT and CRASH are not scores.** They are statements about our configuration.
5. Never infer a quantization scheme from a repo name. Audited 2026-09-07 against every `config.json`
   behind the results table: **three of the eight** are named for a scheme the config does not state
   — `gemma-4-31B-it-qat-w4a16-ct` (no "qat", no "w4a16" anywhere in it; group **32**),
   `Qwen3.8-27B-AWQ-INT4` and `GLM-4.7-Flash-AWQ-4bit` (both `compressed-tensors`, not `awq`). The
   group size is the half a name omits and the half that matters: 32 against 128 is a 4x granularity
   difference, and it stayed invisible while both arms were called "AWQ".

---

---

## Running the benchmarks

**Python track** (`scripts/`) — production's CoD path, any OpenAI-compatible engine, records the
full coordinate in every scorecard and refuses to emit a result it cannot attribute. This produces
the numbers in the tables above.

```bash
python3 LlmBenchmark/scripts/run_model.py --task cod --endpoint-mode completions \
    --prompt-file cod_json_v1.txt --schema-file cod_json_schema_v1.json --chat-template ...
python3 LlmBenchmark/scripts/eval_harness.py --task cod --cod-gold
```

**C# track** — the quick-benchmark screen, drives SentinelCollector's own clients.

```bash
cd LlmBenchmark && ./run-benchmarks.sh                       # llama.cpp arm, 2 entries
./run-benchmarks.sh --filter "Category=LlmBenchmark"          # full run

# vLLM arm — the variable must reach the container, so NOT run-benchmarks.sh
cd ../SentinelCollector/.devcontainer && sudo nerdctl compose up -d
sudo nerdctl compose exec -T -e BENCHMARK_BACKEND=VllmServer \
    -e VLLM_ENDPOINT=http://vllm-server:8000 \
    -e BENCHMARK_MODEL=Qwen/Qwen2.5-32B-Instruct-AWQ \
    sentinel-collector-dev \
    dotnet test /workspace/LlmBenchmark/LlmBenchmark.csproj --filter "Category=QuickBenchmark"
```

`run-benchmarks.sh` takes `--filter` and nothing else. All three `run-*.sh` wrappers are the
llama.cpp arm only and refuse a `BENCHMARK_BACKEND=VllmServer` invocation rather than silently
running llama.cpp under a vLLM banner.

## Hardware

RTX 5090, 32 GB VRAM (31.8 GiB usable) · Threadripper 9960X, 24 cores / 48 threads · 125 GB RAM ·
Context 32K. All benchmark models run fully on GPU. Production CoD/RAG generation also uses CPU
llama.cpp (`llama-cpu-rag`); this file tracks the GPU-served extraction track.
