# LlmBenchmark/scripts

Python harness scripts for the Sentinel extraction-LoRA acceptance-criteria pipeline. Build the eval substrate, run a vLLM-served model against it, and score against the pinned acceptance metrics.

## Files

| Script | Purpose |
|---|---|
| `build_eval_substrate.py` | Deterministically builds the evaluation substrate (positives + negatives) for the Sentinel extraction acceptance-criteria harness. Writes the merged substrate plus a sidecar `criteria.json` documenting construction. Substrate itself is ~10 MB (10,185,438 bytes, measured 2026-09-04) and lives under `/opt/ai-inference/training-data/eval-substrates/` (not committed). |
| `run_model.py` | Drives a substrate through any **OpenAI-compatible** endpoint (vLLM, SGLang, llama.cpp `/v1`) and emits the predictions JSONL `eval_harness.py --predictions` consumes, plus a provenance sidecar recording the engine build. Stdlib only. |
| `eval_harness.py` | Computes the 18 pinned acceptance-criteria metrics against an eval substrate. `score_predictions(gold, pred)` is a pure function (no vLLM / GPU / network — unit-testable); it scores `--predictions` from `run_model.py`, or `--mock` gold-tautology predictions for harness verification. It does **not** call a model — that is `run_model.py`'s job, and keeping them apart is what keeps the scorer offline-testable. |
| `test_run_model.py` | Unit tests over the runner: strict parsing (never salvage), the outbound payload, and engine identification. Fully offline — the HTTP boundary is stubbed. |
| `test_eval_harness.py` | Unit tests over `score_predictions` and the substrate-construction logic. Fully offline. |
| `check_staleness.py` | Grades the committed scorecards against what is running now: engine build, attribution, age. **One verdict is a pass** (`CURRENT`) and it requires a live comparison actually to have happened — with no reachable `--endpoint` every scorecard is `DRIFT_UNCHECKED` and the exit code is non-zero. Runs a known-bad control over its own classifier first and aborts (exit 3) if that control fails — the control uses its OWN fixed threshold, never `--max-age-days`. A file it cannot examine (unparseable, or parsing with no scorecard shape) is graded `UNEXAMINED` and COUNTED **whatever it is called**, because a file skipped in silence takes the denominator with it; the ONLY files it passes over are the named sidecars `*.criteria.json` and `*.provenance.json`. Not recursive, and that gap is open -- a card in a SUBDIRECTORY is never looked for (docs/BACKLOG.md MEASUREMENT DEBT). |
| `test_check_staleness.py` | Unit tests over the staleness classifier, its self-check, and `main()`'s exit code. Fully offline. |

## Typical workflow

```bash
# 1. (Re)build the substrate from upstream training-data inputs.
python LlmBenchmark/scripts/build_eval_substrate.py

# 2. Run a candidate model against any OpenAI-compatible engine.
#    --endpoint works for vLLM (:8000), SGLang, and llama.cpp's own /v1 (:8080).
python3 LlmBenchmark/scripts/run_model.py \
    --substrate /opt/ai-inference/training-data/eval-substrates/<dated>.json \
    --endpoint http://localhost:8000 \
    --model Qwen/Qwen2.5-32B-Instruct-AWQ \
    --out /tmp/preds.jsonl

# 2b. Same runner, HOSTED endpoint -- how a candidate is scored without a GPU reload.
#     --api-key-file takes a PATH, never the token: a token on a command line is in the
#     shell history, in ps output and in every transcript. It is never written to provenance.
#     A router is UNIDENTIFIABLE by construction (it does not expose the provider's engine
#     build), so --allow-unidentified-engine is required and honest there -- and the
#     `:provider` suffix is then the only attribution the scorecard can carry, which is why
#     the runner REFUSES a router model without one. Strict-schema enforcement is a
#     per-provider property: unpinned, the router picks, and it need not pick twice alike.
python3 LlmBenchmark/scripts/run_model.py \
    --substrate /opt/ai-inference/training-data/eval-substrates/<dated>.json \
    --endpoint https://router.huggingface.co \
    --api-key-file ~/.hf-inference \
    --model Qwen/Qwen3.8-27B:deepinfra \
    --allow-unidentified-engine \
    --prompt-file SentinelCollector/src/cod-prompts/cod_json_v1.txt \
    --schema-file SentinelCollector/src/cod-prompts/cod_json_schema_v1.json \
    --out /tmp/preds.jsonl

# 3. Grade it. --adapter-meta carries the engine build into the scorecard.
python3 LlmBenchmark/scripts/eval_harness.py \
    --substrate /opt/ai-inference/training-data/eval-substrates/<dated>.json \
    --predictions /tmp/preds.jsonl \
    --adapter-meta /tmp/preds.jsonl.provenance.json \
    --model-label "qwen2.5-32b-awq @ vllm-0.19.0" \
    --out /tmp/scorecard.json

# 4. Are the numbers we already have still current?
#    --endpoint is what makes CURRENT reachable: without a live engine to compare the
#    recorded build against, nothing has been checked, so nothing passes. Run it without
#    one and every scorecard verdicts DRIFT_UNCHECKED and the exit code is 1 -- that is
#    the tool working, not a corpus full of stale numbers.
#
#    THIS INVOCATION CANNOT EXIT 0 ON THE COMMITTED CORPUS, and that is not a bug to chase:
#    baseline-20260510-mock.scorecard.json is a gold-tautology harness check, so it verdicts
#    MOCK -> stale, permanently. Read the per-row table, not the exit code, when running it by
#    hand; the exit code is for a corpus of real measurements.
python3 LlmBenchmark/scripts/check_staleness.py \
    --scorecards LlmBenchmark/eval-substrate \
    --endpoint http://localhost:8000 --endpoint http://localhost:8080

# 5. Unit-test the grader and the runner (all offline, no pytest needed).
python3 LlmBenchmark/scripts/test_eval_harness.py
python3 LlmBenchmark/scripts/test_run_model.py
python3 LlmBenchmark/scripts/test_check_staleness.py
```

## Why the runner records the engine build

An inference-engine benchmark decays. Measured 2026-09-03: this box RUNS llama.cpp build
10603 (image built 2026-08-24) while the newest llama.cpp figure in `BENCHMARKS.md` is
2026-04-03, and `BENCHMARKS.md`'s "Backend Comparison" section compares llama.cpp against
**Ollama**, retired 2026-06-11. Nothing in a result file recorded an engine build, so none
of that was visible from the results themselves.

`run_model.py` therefore **refuses to emit predictions it cannot attribute** — if neither
`/version` (vLLM) nor `/props` (llama.cpp) identifies the engine, it exits 2 rather than
producing an unattributable number. `--allow-unidentified-engine` overrides it and stamps
`engine_identified: false` so the gap appears in the scorecard instead of being absent from
it. `check_staleness.py` then compares a scorecard's recorded build against the live one --
and when it *cannot* (no endpoint given, the endpoint did not answer, or the scorecard names
an engine but not a build) it says so with a stale verdict and a non-zero exit. Unknown is
not current: the first version of that file reported 7/8 `CURRENT` and exit 0 on an
invocation with no `--endpoint` at all, having compared nothing.

A **multi-provider router stays unidentified on purpose**, and the override is not a
loophole there. Measured 2026-09-05, `router.huggingface.co` answers 404 to both identity
probes and 200 to `/v1/models` with 138 entries, each carrying a `providers[]` array. The
engine that runs the weights is the *provider's*, whose build the router does not expose —
so the run is unattributable in exactly the sense the gate exists for, and the `:provider`
pin is what supplies the attribution instead (recorded as `provenance.provider`). That is
why the runner refuses a router model with no pin rather than warning about it: the same
`/v1/models` response that makes a bare id *look* valid is the one proving the router will
choose for you.

## What the run cost

`usage` is captured per record and summed into `provenance.usage` — token counts plus
`estimated_cost` where the provider reports one (deepinfra does). Cost is `null`, never
`0.0`, when nothing reported a price: a provider that does not price and a run that was
free are different facts, and `calls > 0 AND cost == $0` reads as free until the bill
arrives. `cost_reported_by` says how many records actually carried a price, and
`cost_per_record` divides by those rather than by the record count.

## See Also

- [LlmBenchmark](../) — parent project (C# benchmark runner + `BENCHMARKS.md`)
- [SentinelCollector](../../SentinelCollector/README.md) — owner of the extraction LoRA being evaluated
- [SentinelCollector/scripts](../../SentinelCollector/scripts/README.md) — training-data generation + QLoRA fine-tuning
