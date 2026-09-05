# LlmBenchmark/scripts

Python harness scripts for the Sentinel extraction-LoRA acceptance-criteria pipeline. Build the eval substrate, run a vLLM-served model against it, and score against the pinned acceptance metrics.

## Files

| Script | Purpose |
|---|---|
| `build_eval_substrate.py` | Deterministically builds the evaluation substrate (positives + negatives) for the Sentinel extraction acceptance-criteria harness. Writes the merged substrate plus a sidecar `criteria.json` documenting construction. Substrate itself is ~10 MB (10,185,438 bytes, measured 2026-09-04) and lives under `/opt/ai-inference/training-data/eval-substrates/` (not committed). |
| `run_model.py` | Drives a substrate through any **OpenAI-compatible** endpoint (vLLM, SGLang, llama.cpp `/v1`) and emits the predictions JSONL `eval_harness.py --predictions` consumes, plus a provenance sidecar recording the engine build. Stdlib only. |
| `eval_harness.py` | Scores predictions against an eval substrate, on **either of two tasks** (`--task`). `cove` (default): the 18 pinned metrics over the v6.2 substrate's own `{text_quote, value, period, certainty}` shape — a task production does not run. `cod`: production's CoD stage-1 shape `{article_type, entities, numbers, events, claims}`. Both scorers are pure functions (no vLLM / GPU / network — unit-testable); they score `--predictions`, or `--mock` gold-tautology predictions. It does **not** call a model — that is `run_model.py`'s job, and keeping them apart is what keeps the scorer offline-testable. |
| `test_run_model.py` | Unit tests over the runner: strict parsing (never salvage), the outbound payload, and engine identification. Fully offline — the HTTP boundary is stubbed. |
| `test_eval_harness.py` | Unit tests over both scorers, the scorecard builder and the control arms. Fully offline. One guard test per metric, each constructing the wrong-pairing case; the CoD alignment-key test asserts the OLD key scores 1.0 on it, so the trap it replaces is measured rather than asserted. |
| `check_staleness.py` | Grades the committed scorecards against what is running now: engine build, attribution, age. **One verdict is a pass** (`CURRENT`) and it requires a live comparison actually to have happened — with no reachable `--endpoint` every scorecard is `DRIFT_UNCHECKED` and the exit code is non-zero. Runs a known-bad control over its own classifier first and aborts (exit 3) if that control fails — the control uses its OWN fixed threshold, never `--max-age-days`. A file it cannot examine (unparseable, or parsing with no scorecard shape) is graded `UNEXAMINED` and COUNTED **whatever it is called**, because a file skipped in silence takes the denominator with it; the ONLY files it passes over are the named sidecars `*.criteria.json` and `*.provenance.json`. Not recursive, and that gap is open -- a card in a SUBDIRECTORY is never looked for (docs/BACKLOG.md MEASUREMENT DEBT). |
| `select_cod_gold_corpus.py` | Resolves the 40-article CoD gold corpus out of the substrate from a hand-written selection table, each row carrying why that article is in the set. Keyed on `(source_file, source_index)` and REFUSES a duplicate key -- the index alone is not unique, the substrate concatenates two builds that each number from zero. `--verify` re-resolves and compares content hashes against the committed corpus, so the set is fixed at selection. Stdlib only. |
| `build_cod_gold.py` | Builds CoD **stage-1** gold labels (`article_type, entities, numbers, events, claims`) for that corpus -- the shape production's `cod_json_v1.txt` emits, which the v6.2 substrate's `period`/`certainty` gold cannot score. Stages: `preflight` (identity + a structured-output enforcement probe on both labellers, and the expected bill printed before anything is bought), `primary`, `independent`, `adjudicate`, `assemble`. Hard-capped spend ledger, fail-closed, resumable per article. Reuses `run_model.build_payload` for request construction; needs the `anthropic` SDK, so unlike the rest of this directory it is not stdlib-only. |
| `verify_cod_gold.py` | Verifies a gold artifact with `jsonschema` against production's schema file -- deliberately NOT `build_cod_gold`'s own validator -- plus content pinning, origin parity and the recorded cap. Also checks what jsonschema CANNOT see. A required identity field present and BLANK is schema-valid and aligns with nothing, not even a copy of itself. And the ANCHOR check, which is the reason this file exists: `numbers.source_entity` must be a name copied exactly from that article's OWN `entities[]`, and on an `analyst_action` article it must not be the `analyst_firm` that PUBLISHED the figure. Article 429 shipped 23 gold-price forecasts anchored to the banks that published them and nothing in the build complained; a wrong anchor grounds a figure to the wrong instrument, which is the one defect class here a scorer cannot see downstream. `--selftest` runs ten controls -- eight known-bad mutations (bad enum, over-long string, an extra `period` key, a dropped origin entry, an unpinned hash, a blank `event.subject`, a `source_entity` off its own `entities[]`, an analyst firm anchored on an analyst action) each of which must be caught by name, plus two NEGATIVE controls that must stay silent: a blank `numbers.source_entity`, which production's prompt specifies for an ownerless figure, and that SAME firm anchor on a non-analyst article, where a firm may legitimately own its own number. Without the second, the anchor check would pass just as well written as a bare `ent_type == analyst_firm`, which would condemn every correct firm-owned figure. |
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

## Scoring production's CoD stage 1 (`--task cod`)

The default `cove` task grades the substrate's own numeric shape. **Production does not emit
that shape.** CoD stage 1 emits one object with four list-valued keys, defined by
`SentinelCollector/src/cod-prompts/cod_json_schema_v1.json`, and `period` / `certainty` /
`text_quote` are not in it — they arrive from later pipeline stages. `--task cod` scores
that object on its own terms, against
`LlmBenchmark/eval-substrate/cod-stage1.criteria.json` (**provisional, not
ratified**; the ratified CoVe file is deliberately untouched because eight committed
scorecards cite it as their provenance).

```bash
# With gold. Gold rows are {source_file, source_index, gold: {...CoD object...}},
# JSONL or a JSON list; only records carrying BOTH gold and a prediction are scored, and
# `coverage` in the scorecard says how many that was.
python3 LlmBenchmark/scripts/eval_harness.py --task cod \
    --substrate /opt/ai-inference/training-data/eval-substrates/<dated>.json \
    --cod-gold /path/to/cod-gold.jsonl \
    --predictions /tmp/preds.jsonl \
    --out /tmp/cod-scorecard.json

# WITHOUT gold. Still worth running: the self-consistency metrics need none, and
# every gold-dependent metric reports null with a reason rather than 0.0.
python3 LlmBenchmark/scripts/eval_harness.py --task cod \
    --substrate <substrate> --predictions /tmp/preds.jsonl --out /tmp/card.json
```

### The alignment key, and why it is not `source_text`

The CoVe scorer aligns a prediction to gold on `text_quote` — a verbatim sentence, long
enough to identify *which* fact is meant. CoD's nearest field, `source_text`, is a 1–3 token
numeric literal: two unrelated figures both written `"$15"` overlap perfectly, and every
per-field accuracy downstream inherits that wrong pairing. `value` fails the same way (both
normalize to `15`).

A CoD number's identity is instead **what it measures and who owns it** — `context` and
`source_entity`, the two fields the prompt itself defines that way, both `required` by the
schema so keying on them penalizes a model that omits them. A pair is a candidate only if it
clears *both* floors independently (0.5 each); the mean only ranks among candidates. `value`
and `unit` are deliberately **out** of the key — anything in the key reads 1.0 by
construction, and value accuracy is the number a model swap turns on. Value equality is used
only to break ties between candidates that already agree on identity (a guidance range).

### The two control arms

- **`--mock` is a CEILING that cannot fail.** It feeds gold back as the prediction, so a
  perfect column says the plumbing ran and nothing else. The scorecard says so in
  `mode_note` and records `controls.mock_ceiling.verdict = NOT_A_CONTROL`.
- **`controls.shuffled_gold` is the arm that can fail**, and it runs on *every* invocation
  of either task. Each prediction is scored against gold from a different article (rotation
  by `n//2`, never by 1 — adjacent substrate records are near-duplicates). Its expected
  floor is near zero and is known **without trusting the gold or the model**, so a
  respectable number there means the scorer is broken by construction. `FLOOR_BREACHED` is
  the verdict; a corpus of near-duplicate articles reads the same way, which is why the note
  names both causes.

Measured 2026-09-05, Qwen2.5-32B-AWQ @ vLLM 0.19.0 on production's CoD request shape, all
597 substrate articles: unshuffled `numbers_f1` **0.9993**, shuffled `numbers_f1` **0.0106**
(`FLOOR_OK`). The mock arm's own scorecard passed 22 of 24 measurable metrics — and both
failures were `json_valid` and `source_entity_referential_integrity`, the metrics that need
no gold and so cannot be flattered by feeding it back.

**The tautology ceiling is not 1.0, and that is a finding rather than a rounding error.**
`events_f1` ceils at 0.9405 and `claims_f1` at 0.9838. The schema's `required` is satisfied
by `""`, so the model emits events with a blank `subject` and claims with a blank `object`;
an item whose identity fields are blank cannot align with *anything*, including a perfect
copy of itself, and scores as a false positive **and** a false negative. That penalty is
invisible and reads as a missed fact, so it is published as
`diagnostics.identityless_predicted_items` — measured `{events: 150/2520, claims: 38/2351,
numbers: 5/7045, entities: 0/5527}`, which accounts for each ceiling exactly.

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
