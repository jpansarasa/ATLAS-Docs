# The measurement space is a GRID, not a leaderboard

Status: design rule. Governs every comparison this harness produces.
Read with `CLAUDE.md` §MODEL_ACCEPTANCE, which sets the bar this document says how to MEET.

## THE CLAIM

"Which model is better" is not a question this harness can answer, because a model is not
what we measure. We measure a POINT in a multi-axis configuration space, and the score is a
property of the point, not of the model at its centre.

Every irreconcilable number in the 2026-09-04/06 epic was a different point read as a
different measurement of the same thing.

## THE AXES

| # | Axis | Values we have actually run | Recorded in the scorecard? |
|---|------|------------------------------|----------------------------|
| 1 | Engine + build + ITS DEPENDENCY STACK | vLLM 0.19.0, 0.28.0, llama.cpp; and the image's own `transformers` version | PARTLY -- engine build is fail-closed (`run_model.probe_engine`); the image's dependency versions are recorded NOWHERE |
| 2 | Model family + size | 9 families / 14 rows in BENCHMARKS.md; Qwen 2.5-32B, 3-30B, 3-32B, 3.8-27B, Gemma 3-27B, Gemma 4-31B, GLM-4.7-Flash, EXAONE 4.0-32B, Command-R-35B, Mistral-Small-24B, phi4-14B, deepseek-r1-32B, llama3.3-70B | YES, `model` + `model_revision` (HF cache ref) |
| 3 | Weight quantization | see VERIFIED QUANT COORDINATES below -- the repo names are NOT the schemes | NO -- was inferred from the repo NAME, which is a convention, not a field |
| 4 | KV cache dtype | fp8_e5m2, fp8_e4m3, unquantized | NO |
| 5 | Context length | 15360, 32768 | NO |
| 6 | Concurrency | `--max-num-seqs`, client parallelism | NO |
| 7 | Sampling | temp, seed, repetition_penalty, max_tokens | YES, `acceptance_evidence.request_sampling` |
| 8 | Prompt / schema / template bytes | cod_json_v1 pre- and post-#1017 | YES, sha256 in `request_bytes` |
| 9 | Task | substrate 16-block vs production CoD path | YES, `production_prompt_path` boolean |
| 10 | Eval population + alignment key | substrate v6.2, cod-gold 40, 3 recall populations | PARTLY -- `criteria_source` + `substrate_sidecar`, but the KEY CONVENTION is not a field |

Six recorded, four not. The four unrecorded ones are the four that move when we cross model
families -- which is the only comparison anyone actually wants.

## WHAT THIS EXPLAINS

Within one model family we already measure single-axis, and those numbers held up all epic:

  0.443 -> 0.494   axis 4 only (fp8_e5m2 -> unquantized KV), same model, same engine.  d=+0.051
  0.764 -> 0.7634  axis 1 only (vLLM 0.19.0 -> 0.28.0).                                d=-0.0006
  0.7634 -> 0.744  axis 3 only at fixed engine, but NOT the axis its labels claim -- see below. d=-0.019
  0.764 -> 0.694   axis 7 only (our sampling -> the model card's).                     d=-0.070
  0.7634 -> 0.7629 decode strategy only (+MTP speculative). F1 flat, wall clock -30%.

The 0.051 KV-dtype row is the ONLY reason we could bound the A/B confound at all. Single-axis
is what made it usable a week after it was taken.

Every cross-family number failed, and failed the same way:

  0.443 vs 0.764   axes 1, 2, 3, 4 moved together. Not attributable to the model.
  A 0.5153 +/- 0.0106  vs  B 0.7400 +/- 0.0072   (CoD task, production prompt path)
    d = +0.2247, and it is NOT quotable: model, weight-quant format, KV dtype and context
    length all moved between the arms.
  B 0.7400  vs  B' 0.6729 +/- 0.0100
    THE SAME CANDIDATE, measured twice, 0.067 apart -- larger than the weight-quant effect
    (0.019) and a hundred times the engine bump (0.0006). What differed between the two
    servings is recorded in NO artifact, which is exactly the defect this file exists for.

And axis 10 on its own produced three different "recall" numbers for one unchanged system:
0.467 (off-manifold probes), 0.775 (catalog names), 0.7916 (obs-weighted production strings).
A population is an axis. Changing it silently is the same error as changing the KV dtype
silently.

## VERIFIED QUANT COORDINATES [read from each checkpoint's own config.json, 2026-09-06]

Axis 3 was the row this file called a naming convention rather than a field. Reading the
three checkpoints we actually serve shows the names are not merely unreliable, they are WRONG:

  Qwen/Qwen2.5-32B-Instruct-AWQ        quant_method "awq", 4 bit, group_size 128,
    [INCUMBENT]                        version gemm, zero_point true.   <- the only true AWQ
  cyankiwi/Qwen3.8-27B-AWQ-INT4        quant_method "compressed-tensors", format
    [CANDIDATE]                        "pack-quantized", 4 bit int, group_size 32,
                                       asymmetric, mse observer.
                                       INT4 is true. AWQ IS FALSE -- it is not AWQ at all.
  unsloth/Qwen3.8-27B-NVFP4            quant_method "compressed-tensors", format
                                       "float-quantized", with DYNAMIC FP8 INPUT ACTIVATIONS
                                       on attention, linear_attn, lm_head and layers 56-63 MLP.
                                       A mixed scheme, and it quantizes ACTIVATIONS, which is
                                       a different thing from weight quantization.

TWO CONSEQUENCES, both of which change how existing numbers read.

1. THE ARM COMPARISON HAS A CONFOUND NOBODY NAMED. The incumbent is AWQ at group_size 128;
   the candidate is pack-quantized INT4 at group_size 32. That is a 4x difference in
   quantization GRANULARITY, and finer groups generally help quality -- in the candidate's
   favour. So part of any candidate-over-incumbent delta may be the quantization recipe
   rather than the model. This is not a reason to discount the direction (the measured fp8-KV
   price on this task is -0.0268 against a +0.1979 gap), but it is a named axis that is still
   moving, and it was invisible while both arms were called "AWQ".

2. ANYONE READING THE BASELINES TABLE IS MISLED IN THE FLATTERING DIRECTION. "Qwen2.5-32B-AWQ
   vs Qwen3.8-27B-AWQ-INT4" reads as two AWQ checkpoints, i.e. as an axis already controlled.
   It never was. A name is not a coordinate.

## A FAILURE ON ANOTHER AXIS IS NOT A MODEL SCORE

`BENCHMARKS.md` is the worked example, and it is worse than confounded -- three of its
fourteen rows are not extraction measurements at all, and they are the three that read as the
most decisive model verdicts:

  Command-R 35B    0.0%   its own finding says "OOM at 32K context (35B too large for KV
                          cache). At 8K both entries timeout." -> an AXIS 4/5 result written
                          into the model column. The model was never scored at a feasible point.
  llama3.3 70B     CRASH  "insufficient VRAM at q2_K" -> an AXIS 3 result. We chose the quant
                          that did not fit and recorded the model as crashing.
  Gemma 3 27B      0.0%   "too slow, times out on both" -> a wall-clock/serving outcome.
                          TIMEOUT is not a quality score; it is the absence of one.

A zero in a quality column is read forever as "this model is bad". What those three rows
actually say is "no feasible point was found under the constraints we happened to be running",
which is a statement about OUR configuration, not about the model. Gemma is the live case: it
appears twice, once as a total failure (Gemma 3, timeouts) and once as a credible 59.7% at
Q4_K_M on llama.cpp -- and that second row is the only Gemma number anyone would quote,
against Qwen rows served on a different engine at an unstated quantization.

Quantization already lives in the model NAME here, and inconsistently: `Gemma 4 31B (Q4_K_M)`,
`phi4:14b-q4_K_M` and `llama3.3:70b-instruct-q2_K` state it; `qwen2.5:32b-instruct`,
`mistral-small:24b` and `deepseek-r1:32b` do not. That is axis 3 surviving as a naming
convention across a whole leaderboard -- the defect the axis-3 row above names, at scale.

So "Qwen family dominates -- 4 of top 5" is the inference this document forbids. It may well be
true. This table cannot show it: those four Qwen rows sit on two different backends at
unstated quants, the Gemma row is Q4_K_M on llama.cpp, and most of the losers are Ollama.

THE ONE CORRECTLY DESIGNED COMPARISON IN THAT FILE is its "Backend Comparison (Same Model)"
section, which holds the model fixed and varies the engine: qwen2.5:32b at 64.6% on llama.cpp
vs 56.7% on Ollama. Single-axis, and it found 7.9pp -- four times the weight-quant effect and
larger than the whole ~0.05 effect our current swap is trying to detect. (Caveat that keeps it
honest: those rows came from the retired client that dropped the seed, so the magnitude is not
reproducible. The lesson is that the axis is real and big, not that 7.9pp is the number.)
The engine is not a detail to be held constant by luck.

## MEASURED: THE RULE PAID OUT ON ITS FIRST TEST [2026-09-06]

`BENCHMARKS.md` records Gemma 3 27B as `0.0% | TIMEOUT | ERROR | FAIL`, with the finding
"too slow, times out on both test cases". Re-qualified at a feasible point on production's
own CoD path:

  Gemma 3 27B  numbers_f1 0.6220 (n=2, sd 0.0151, range 0.6113-0.6327)
  incumbent    numbers_f1 0.5153 (n=5)                                 d = +0.107

It BEATS the model currently in production, and it is not slow: 40 gold articles in 103.7s
and 103.1s wall at concurrency 6, 0 call errors, 0 schema_invalid, 0 truncated, every
finish_reason "stop". Coordinate: vLLM 0.19.0, RedHatAI/gemma-3-27b-it-quantized.w4a16,
compressed-tensors pack-quantized, fp8_e4m3 KV, max_model_len 32768, max-num-seqs 16,
util 0.95, KV pool 47,296 tokens, TRITON_ATTN -- identical to the candidate's serving point
in every axis but the model and its quant format.

The original zero was an ENGINE and DECODING artifact: llama.cpp with grammar-free
generation and salvage-parsing, against vLLM with `response_format` json_schema here. A
whole model family was written off on a coordinate we no longer run, in a decoding mode this
project bans.

TWO FURTHER ELIMINATIONS WERE RE-EXAMINED AND ONE STANDS, AS A COORDINATE FINDING:
  llama3.3-70B    NO feasible point at any vLLM-servable quant. 4-bit weights ALONE are
                  37.0-40.8 GiB on a 31.8 GiB card, before one KV byte. fp8 KV is irrelevant:
                  the binding constraint is WEIGHTS. Sub-4-bit exists only as exl2/exl3/AQLM,
                  which vLLM does not serve. That is a statement about our hardware, not the
                  model, and it is the correct SHAPE for a negative result.
  Command-R 35B   the v01 build has no GQA -- 64 kv_heads x 40 layers = 640 KiB/token, 5x the
                  incumbent -- and its max_position_embeddings is 8192, so it was NEVER a 32K
                  model. The old "OOM at 32K" was asking for something the checkpoint could
                  not do. The family's CURRENT build (32B, WITH GQA, 80 KiB/token) is what
                  should have been tested.

## A NEW AXIS, FOUND BY TRIPPING OVER IT

Gemma 4 31B failed to boot on the pinned vLLM image: it ships `transformers` 4.57.6, which
cannot parse `model_type: "gemma4"`. vLLM 0.19.0 itself DOES register
`Gemma4ForConditionalGeneration` -- so "the engine supports this model" was true and the
engine still could not load it. Fixed with a derived image bumping ONLY transformers to
5.16.1, torch and vLLM held.

THE ENGINE VERSION IS NOT THE ENGINE. `probe_engine` records `0.19.0` for both the image that
loads Gemma 4 and the one that cannot, so two runs that differ in whether a model can exist
at all stamp the identical engine coordinate. A model eliminated on this axis would look
exactly like a model that failed on its merits -- which is how Gemma got eliminated the first
time.

## THE COUPLING THAT MAKES THIS HARD

Axes 2-5 are coupled BY THE GPU. You cannot hold KV dtype and context fixed while swapping a
32B model for a 27B one at the same VRAM budget; the feasible set moves with the model. So
the confound is not carelessness, it is the default outcome of running each model at its own
best configuration.

That means there are TWO legitimate questions, and they have different designs:

  Q1 ATTRIBUTION -- "does this model extract better?"
     Serve both arms at a COMMON FEASIBLE POINT: the intersection of both models' feasible
     configs, which will be worse than either model's own best. Every axis but 2 is pinned.
     Answers a causal question. Does NOT tell you what to deploy.

  Q2 DEPLOYMENT -- "which configuration should we run?"
     Each candidate at its own best point. Answers the real operational question.
     The answer is a CONFIGURATION, never a model, and the delta CANNOT be attributed to any
     single axis. Quoting a Q2 delta as a model result is the error this file names.

We have been running Q2 and quoting it as Q1 for the whole epic. `CLAUDE.md`
§MODEL_ACCEPTANCE is written model-shaped ("a candidate ships only with a SCORECARD ... that
BEATS the incumbent's") for a decision that is configuration-shaped.

## THE ENGINE AXIS IS OPEN, AND THE VRAM CEILING IS AN ASSUMPTION

This file has twice enumerated axis 1 as though it were a short list, and twice been wrong in the
same direction. First "vLLM 0.19.0, 0.28.0, llama.cpp". Then, when the precision question came up,
"vLLM's rungs are 4-bit and 8-bit; Q6 is a llama.cpp path" -- framing the axis as a BINARY between
the engine we run and the one we used to run.

THE PROJECT'S OWN HISTORY SAYS OTHERWISE: llama.cpp, then Ollama, now vLLM. Three engines, each
adopted for a reason that was good at the time, and NONE of the switches re-measured afterward. The
engine in production is the residue of the last decision, not a constraint on the next one.

AND THE HARNESS WAS BUILT FOR THIS. `LlmBenchmark/scripts/README.md` describes `run_model.py` as
driving "any OpenAI-compatible engine (vLLM, SGLang, llama.cpp /v1)". It is a general instrument
that has been used as a single-engine one. Candidates nobody here has scored: SGLang, TensorRT-LLM,
ExLlamaV2/V3, MLC, ktransformers, colibri.

  ✗ never treat the ENGINE IN PRODUCTION as the boundary of the engine axis -- it is one sample
  ✗ never answer "can we do X" with "our engine cannot" until the axis has been enumerated

**THE CEILING MOVES TOO.** Every feasibility argument in this file so far -- Command-R's OOM,
llama3.3-70B having no point, "no rung above 4-bit fits at 27B" -- silently assumes the model lives
in 31.8 GiB of VRAM. That is true of GPU-resident engines and of nothing else. CPU-primary MoE
runtimes (colibri's stated premise: "Run frontier MoE models -- 744B to 2.8T parameters -- on
consumer and heterogeneous hardware") put the budget at this box's 125 GB of RAM and 312 GB of free
disk instead, with only the active experts hot per token.

MEASURED CAPACITY, so the next feasibility claim is arithmetic and not a guess:
  RAM 125 GB total (~40 GB free while the GPU sweep runs) | 48 threads, Threadripper 9960X,
  1 NUMA node | 312 GB free on /home | VRAM 31.8 GiB
  -> a ~125B MoE at int4 is ~62 GB and FITS IN RAM; a 284B is ~142 GB and does not, without
     disk streaming; a 744B at ~372 GB is disk-resident or nothing.
So the model-size axis has a whole region above 32B that we have never sampled, and rejected
without ever pricing. Latency is the real question there, not capacity, and latency is measurable.

## THE FEASIBLE SET HAS A FLOOR, AND IT IS QUALITY, NOT VRAM

A configuration that loads and emits tokens is NOT automatically a feasible point. The search
for "what fits" degenerates the moment fitting is the only test -- which is exactly how
llama3.3-70B earned its `CRASH` row: someone crammed it in at q2_K. The remedy for a model
that does not fit is to REPORT that it does not fit, never to lower the precision until it
does.

  ✗ q2 and sub-3-bit are BANNED outright. User, 2026-09-06, verbatim:
    "skip the q2 quants. they produce tokens, not answers"
  ✗ never squeeze a model into memory by dropping below the floor -- "no feasible point on
    this hardware" is a legitimate, final answer and a better one than a bad score

AND THE FLOOR MAY BE HIGHER THAN WHERE WE HAVE BEEN STANDING THE WHOLE TIME. User, same day:
"even Q4 might not make sense. Q6 might be the floor." That is not a preference, it is an
unmeasured claim about an axis, and it lands on every number this project owns:

  incumbent  Qwen2.5-32B-Instruct-AWQ        4 bit
  candidate  Qwen3.8-27B "AWQ-INT4"          4 bit
  Gemma 3    gemma-3-27b-it-quantized.w4a16  4 bit

EVERY FIGURE WE HAVE EVER PRODUCED SITS AT ONE POINT ON THE PRECISION AXIS, and 4-bit was never
chosen against an alternative -- it is what fits, adopted as though it were neutral.

**MEASURED 2026-09-06, AND THE ANSWER IS NO.** Q6_K is a REGRESSION against Q4_K_M on this
workload. Same model, publisher and release (unsloth/gemma-3-27b-it-GGUF), same
`general.quantization_version`, same engine and image (llama.cpp server-cuda b10820), same context,
KV, constrained decoding, prompt, schema, sampling and gold. The ONLY difference is
`general.file_type` 15 vs 18, read from the GGUF headers rather than the filenames:

  numbers_f1   Q4_K_M 0.6263 (sd 0.0039)  Q6_K 0.5985 (sd 0.0003)   d = -0.0278, 12.2x se
               Q4 [0.6275, 0.6295, 0.6219]   Q6 [0.5988, 0.5985, 0.5983]  -- RANGES DISJOINT
  recall -0.0257 (14.1x se) | precision -0.0297 | entities_f1 -0.0019 (FLAT, 0.5x se)
  latency Q4 38.22 s/doc, Q6 43.58 s/doc -- 14% SLOWER for 5.6 GB more VRAM

So 4-bit is VINDICATED for this workload, by measurement rather than assumption. The number axis
moved against precision; the entity axis did not move at all. Limits, stated because they bound the
claim: one model, one task, one publisher, and k-quants are not strictly ordered by bpw (Q4_K_M and
Q6_K use different per-tensor type mixes), so this compares two real artifacts and not an abstract
precision dial.

AND THE ENGINE IT TOOK TO ASK THE QUESTION ANSWERS A SECOND ONE. vLLM's 8-bit rung at 27B is not
merely tight, it is operationally unusable: 27.26 GiB of weights leave a 4,176-token KV pool, below
this eval's own 7,617-token worst case, and throughput collapses from 419.6 to 92.8 tok/s with only
2 of 6 requests resident. That is why the ladder had to move engines. And llama.cpp CUDA is slower
on the same 40 articles at the same concurrency 6, on both measures, which agree:
wall clock 260.9 s vs 102.7 s = **2.54x**; per-request 38.22 s vs 14.28 s = 2.68x. Both findings
point the same way: stay on vLLM, stay at 4-bit.

  A RATE IS NOT A LATENCY, and an earlier revision of this line said 15x by dividing llama.cpp's
  PER-REQUEST latency (38.22 s) by vLLM's THROUGHPUT figure (2.57 s/doc = 102.7 s wall / 40 docs
  at concurrency 6). The two quantities differ by the concurrency factor, so the ratio inflated
  ~6x. Either measure is fine; mixing them is not. Re-derive across an agent boundary.

THE LADDER IS ENGINE-COUPLED, which makes this axis 1 x axis 3 and not axis 3 alone. vLLM
serves 4-bit, 8-bit (w8a16 / w8a8) and bf16; a true 6-bit is essentially a GGUF rung (Q6_K),
which is a llama.cpp path. So "Q6 as the floor" may not be REACHABLE on vLLM at all -- and if
it is not, then testing the floor either drops to a smaller model or brings llama.cpp back
into contention, on the same engine axis that BENCHMARKS.md already prices at 7.9pp. Do not
invent a rung that does not exist; report the ladder that does.

## THE ADMISSIBILITY RULE

A comparison between two scorecards is admissible only if ONE of:

  ✓ their coordinates differ in EXACTLY ONE axis, and the delta is attributed to that axis; or
  ✓ it declares itself a Q2 configuration comparison, names every axis that moved, and
    attributes the delta to NOTHING.

  ✗ never quote a delta whose arms differ on an unrecorded axis -- it is unfalsifiable, not
    merely imprecise: no later reader can even discover what moved
  ✗ never carry a bare score across documents. A number that does not name its coordinate
    does not identify a run. `CLAUDE.md` already learned this for the engine axis alone,
    after a bare 0.763 turned out to be the BLOCKED 0.28.0 build; the fix was applied to the
    LABEL and never to the experimental DESIGN.

## THE GAP TO CLOSE

Four fields. Until they exist, a scorecard cannot locate itself in the grid, and the rule
above cannot be checked by anything but memory.

| Axis | How to obtain it | Fail mode if guessed |
|------|------------------|----------------------|
| 3 weight quant | `quantization_config.quant_method` from the model's own `config.json` in the HF cache -- NOT the repo name | vLLM's `--quantization` can select a method the name does not state |
| 4 KV cache dtype | server flag; VERIFY whether vLLM exposes it on any endpoint, else declare it at run time | the axis with the largest measured single-axis effect (0.051) is the one we record least |
| 5 context length | VERIFY `max_model_len` in vLLM's `/v1/models` entry; else declare | trades against 4 and 6 for the same VRAM, so it is never independently chosen |
| 6 concurrency | client-side and known to the runner: `--max-num-seqs` + request parallelism | already named in STATE.md as what makes an arm comparison unfalsifiable |

Two are probe-able and two must be declared. Declared fields are the weaker kind, so they
fail closed the way axis 1 does: a run that cannot state its coordinate does not get to
produce a comparable scorecard.

GRADUATION: this rule stops being a document the moment a checker refuses a two-scorecard
comparison whose coordinates differ on more than one axis. Until then it is an opinion held
by whoever last read it, which is the failure mode that produced the 0.067 spread above.

## HARD STOPS

✗ never compare scorecards across model families without pinning axes 3-6 at a common point
✗ never let the repo name stand in for the weight-quant field -- MEASURED: two of the three
  checkpoints we serve are named for a quantization they do not use
✗ never treat activation quantization as the same axis as weight quantization
✗ never call a point feasible because it LOADS -- q2 and sub-3-bit are banned outright; 4-bit
  is now MEASURED rather than assumed, and Q6_K lost to it by 0.0278 at 12.2x se
✗ never read a within-family single-axis delta as a cross-family result
✗ never quote a delta from arms "not served alike" -- fix the serving, do not caveat the number
✗ never treat the eval population or the alignment key as a constant; they are axis 10
✗ never write an infeasibility into a quality column -- OOM, TIMEOUT and CRASH are coordinate
  findings ("no feasible point under X"), and recording them as 0.0% defames the model for good
✗ never conclude a FAMILY is weak from rows that were never served at a feasible point; the
  candidate set is 9 families, and most of them we eliminated on OUR configuration, not on theirs
✗ never read a matching engine VERSION as a matching engine -- the image's dependency stack
  (transformers, torch) decides which models can load at all, and it is recorded nowhere
