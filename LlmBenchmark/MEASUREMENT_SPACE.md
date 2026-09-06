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
| 1 | Engine + build | vLLM 0.19.0, vLLM 0.28.0, llama.cpp | YES, fail-closed (`run_model.probe_engine`) |
| 2 | Model family + size | 9 families / 14 rows in BENCHMARKS.md; Qwen 2.5-32B, 3-30B, 3-32B, 3.8-27B, Gemma 3-27B, Gemma 4-31B, GLM-4.7-Flash, EXAONE 4.0-32B, Command-R-35B, Mistral-Small-24B, phi4-14B, deepseek-r1-32B, llama3.3-70B | YES, `model` + `model_revision` (HF cache ref) |
| 3 | Weight quantization | AWQ, AWQ-INT4, NVFP4, (GGUF untried) | NO -- inferred from the repo NAME, which is a convention, not a field |
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
  0.7634 -> 0.744  axis 3 only (AWQ-INT4 -> NVFP4) at fixed engine.                    d=-0.019
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
✗ never let the repo name stand in for the weight-quant field
✗ never read a within-family single-axis delta as a cross-family result
✗ never quote a delta from arms "not served alike" -- fix the serving, do not caveat the number
✗ never treat the eval population or the alignment key as a constant; they are axis 10
✗ never write an infeasibility into a quality column -- OOM, TIMEOUT and CRASH are coordinate
  findings ("no feasible point under X"), and recording them as 0.0% defames the model for good
✗ never conclude a FAMILY is weak from rows that were never served at a feasible point; the
  candidate set is 9 families, and most of them we eliminated on OUR configuration, not on theirs
