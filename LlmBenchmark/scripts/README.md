# LlmBenchmark/scripts

Python harness scripts for the Sentinel extraction-LoRA acceptance-criteria pipeline. Build the eval substrate, run a vLLM-served model against it, and score against the pinned acceptance metrics.

## Files

| Script | Purpose |
|---|---|
| `build_eval_substrate.py` | Deterministically builds the evaluation substrate (positives + negatives) for the Sentinel extraction acceptance-criteria harness. Writes the merged substrate plus a sidecar `criteria.json` documenting construction. Substrate itself is ~10 MB (10,185,438 bytes, measured 2026-09-04) and lives under `/opt/ai-inference/training-data/eval-substrates/` (not committed). |
| `run_model.py` | Drives a substrate through any **OpenAI-compatible** endpoint (vLLM, SGLang, llama.cpp `/v1`) and emits the predictions JSONL `eval_harness.py --predictions` consumes, plus a provenance sidecar recording the engine build, the **sha256 of the prompt and schema files whose bytes actually reached a request**, and of the chat template string it applied (a path is not a prompt — see below). Takes the same **`--task`** as the scorer and must agree with it: `cove` (default) keeps the extraction array under `predicted_extractions`, `cod` keeps production's stage-1 object under `prediction`. Stdlib only. |
| `eval_harness.py` | Scores predictions against an eval substrate, on **either of two tasks** (`--task`). `cove` (default): the 18 pinned metrics over the v6.2 substrate's own `{text_quote, value, period, certainty}` shape — a task production does not run. `cod`: production's CoD stage-1 shape `{article_type, entities, numbers, events, claims}`. Both scorers are pure functions (no vLLM / GPU / network — unit-testable); they score `--predictions`, or `--mock` gold-tautology predictions. It does **not** call a model — that is `run_model.py`'s job, and keeping them apart is what keeps the scorer offline-testable. |
| `test_run_model.py` | Unit tests over the runner: strict parsing (never salvage), the outbound payload, engine identification, and that a CoD response is **kept** under the key the scorer reads — the contract test imports `eval_harness._cod_object` and asserts the join rather than restating the key. Fully offline — the HTTP boundary is stubbed. |
| `test_eval_harness.py` | Unit tests over both scorers, the scorecard builder and the control arms. Fully offline. One guard test per metric, each constructing the wrong-pairing case; the CoD alignment-key test asserts the OLD key scores 1.0 on it, so the trap it replaces is measured rather than asserted. |
| `check_staleness.py` | Grades the committed scorecards against what is running now: engine build, attribution, age. **One verdict is a pass** (`CURRENT`) and it requires a live comparison actually to have happened — with no reachable `--endpoint` every scorecard is `DRIFT_UNCHECKED` and the exit code is non-zero. Runs a known-bad control over its own classifier first and aborts (exit 3) if that control fails — the control uses its OWN fixed threshold, never `--max-age-days`. A file it cannot examine (unparseable, or parsing with no scorecard shape) is graded `UNEXAMINED` and COUNTED **whatever it is called**, because a file skipped in silence takes the denominator with it; the ONLY files it passes over are the named sidecars `*.criteria.json` and `*.provenance.json`. Not recursive, and that gap is open -- a card in a SUBDIRECTORY is never looked for (docs/BACKLOG.md MEASUREMENT DEBT). |
| `select_cod_gold_corpus.py` | Resolves the 40-article CoD gold corpus out of the substrate from a hand-written selection table, each row carrying why that article is in the set. Keyed on `(source_file, source_index)` and REFUSES a duplicate key -- the index alone is not unique, the substrate concatenates two builds that each number from zero. `--verify` re-resolves and compares content hashes against the committed corpus, so the set is fixed at selection. Stdlib only. |
| `build_cod_gold.py` | Builds CoD **stage-1** gold labels (`article_type, entities, numbers, events, claims`) for that corpus -- the shape production's `cod_json_v1.txt` emits, which the v6.2 substrate's `period`/`certainty` gold cannot score. Stages: `preflight` (identity + a structured-output enforcement probe on both labellers, and the expected bill printed before anything is bought), `primary`, `independent`, `adjudicate`, `assemble`. Hard-capped spend ledger, fail-closed, resumable per article. Reuses `run_model.build_payload` for request construction; needs the `anthropic` SDK, so unlike the rest of this directory it is not stdlib-only. THE LABELLING INSTRUCTION IS PINNED TO PRODUCTION'S PROMPT: three sentences carry the `source_entity` convention (the SERIES owns a macro print, the country does not, `""` is for no ONE named owner) and every stage REFUSES before a paid call if the adjudication instruction, the text `alignability()` would write into the artifact's `blank_by_design.why`, or the prompt stops stating one -- the first build's crosscheck BLANKED two correct rows citing a clause that has since been deleted, and a rebuild on a stale instruction would revert the gold. IT GATES THE PRODUCERS, NOT THE COMMITTED ARTIFACT: the shipped `cod_stage1_gold_v1.json` is never opened, and its `why` states none of the three verbatim while this gate reads green (docs/BACKLOG.md MEASUREMENT DEBT). Opening it here would deadlock the tool -- after a rule change the committed artifact necessarily still states the old rule, so the gate would refuse the very `--stage assemble` that rewrites it. `--selftest` runs eleven controls -- nine mutations cut from production's OWN bullet, one known-good base and the live check -- and aborts at exit 3 ahead of all of them on any of three conditions the controls cannot report on themselves: production's `source_entity` bullet can no longer be LOCATED in the prompt at all (`- source_entity:` .. `Emit each distinct numeric value ONCE`), which would otherwise hand every check an empty rule that every site contains and therefore agrees with; the prompt no longer states a pinned clause, which turns every mutation into one cascade reciting the same absence (measured: 0 of 11 pass, loudly -- the abort buys one message and a distinct rc, not the difference between pass and fail); or the guard's own shape no longer matches `EXPECTED_CLAUSE_NAMES` / `EXPECTED_SITE_NAMES`, literals that derive from neither structure under test, because emptying `SOURCE_ENTITY_CLAUSES` emptied the mutation set with it and printed `2/2 controls behaved as required` at rc 0 with the gate fully disabled. A run that ends in a control count other than eleven fails on its count for the same reason -- the code compares `!=`, so a set that grew unnoticed is as red as one that shrank. Needs no corpus and buys nothing, and it no longer waits for someone to remember: CI runs this `--selftest` and `verify_cod_gold.py`'s and `rescore_alignment_keys.py`'s on every push or PR touching these scripts, the CoD prompts or the gold (`.github/workflows/python-tests.yml`). WHAT IT STILL DOES NOT PIN -- clause TEXTS, site PROVENANCE, and the `main()` wiring itself -- is measured in docs/BACKLOG.md MEASUREMENT DEBT. |
| `verify_cod_gold.py` | Verifies a gold artifact with `jsonschema` against production's schema file -- deliberately NOT `build_cod_gold`'s own validator -- plus content pinning, origin parity and the recorded cap. Also checks what jsonschema CANNOT see. A required identity field present and BLANK is schema-valid and aligns with nothing, not even a copy of itself. And the ANCHOR check, which is the reason this file exists: `numbers.source_entity` must be a name copied exactly from that article's OWN `entities[]`, and on an `analyst_action` article it must not be the `analyst_firm` that PUBLISHED the figure. Article 429 shipped 23 gold-price forecasts anchored to the banks that published them and nothing in the build complained; a wrong anchor grounds a figure to the wrong instrument, which is the one defect class here a scorer cannot see downstream. `--selftest` runs ten controls -- eight known-bad mutations (bad enum, over-long string, an extra `period` key, a dropped origin entry, an unpinned hash, a blank `event.subject`, a `source_entity` off its own `entities[]`, an analyst firm anchored on an analyst action) each of which must be caught by name, plus two NEGATIVE controls that must stay silent: a blank `numbers.source_entity`, which production's prompt specifies when no ONE named entity owns the number -- three cases, and a macro print is NOT one of them, the series owns that, and that SAME firm anchor on a non-analyst article, where a firm may legitimately own its own number. Without the second, the anchor check would pass just as well written as a bare `ent_type == analyst_firm`, which would condemn every correct firm-owned figure. |
| `build_cove_gold.py` | Builds CoVe **stage-2** gold (`text_quote, value, period, certainty, source_entity, ...`) over the SAME 40 articles, at production's stage-2 decoding — **16,384 completion tokens and no `repetition_penalty`**, which is `ChainOfVerification` -> `GenerateStructuredAsync` with neither override, and NOT stage 1's 4,096 + 1.1. It gives `--task cove` a gold that is not the v6.2 substrate's own unaudited `output` arrays. Production's prompt reaches the labeller through the per-record `instruction` and not `--prompt-file`: `render_prompt` substitutes `{{article_text}}` and nothing else, so `{{source}}` and `{{content_type}}` are substituted here and `{{content}}` is rewritten to the placeholder — the document then lands INSIDE the template where `GetInitialExtractionPrompt` puts it. The request schema is parsed out of `ExtractionSchema.cs`; there is no JSON copy of it in the repo and committing one would be a second definition that drifts. Stages: `preflight` (identity, an enum+pattern enforcement probe AND a probe of production's own ARRAY-rooted schema, plus the bill printed before anything is bought), `primary`, `crosscheck`, `assemble`. Hard-capped fail-closed ledger, resumable per article. THE CROSS-CHECK ARM WAS CHOSEN BY A SCREEN ON THE REAL TASK and four candidates failed it (each recorded in `REJECTED_CANDIDATES` with its measurement) — a toy sentence passed two models that returned `[]` on a real article. Cross-family by construction, and deliberately neither the incumbent's family (Qwen) nor the swap candidate's (Gemma). Needs `jsonschema`; no `anthropic` SDK. `--selftest` runs twelve controls over the two pure functions the paid stages rest on — five known-bad admissions (a foreign quote, a blank quote, a null `value`, a string `value`, a figure the article never states), three NEGATIVE controls that must be ADMITTED (a real item, a re-wrapped quote, and the prompt's own mandated `500 MW -> 0.5 GW` conversion, whose value is not a literal and which a careless digit gate would reject), the scoreability classifier tested at inputs DERIVED from its own rule rather than at restated literals, and the collision counter. It aborts at exit 3 ahead of all of them if `ExtractionSchema.cs` no longer parses to production's CoVe shape or `initial_extraction.txt` no longer carries `{{content}}` / `{{source}}` / `{{content_type}}` — either would make every control vacuous rather than red. Buys nothing, needs no corpus and no credential. |
| `verify_cove_gold.py` | Verifies the CoVe gold: `jsonschema` against production's schema **re-derived from `ExtractionSchema.cs`**, content pinning, origin parity, the recorded cap, and the check that matters most — FRESHNESS, because the schema is a DERIVED artifact and one that outlives its source describes a shape production no longer emits, with both files still parsing. Also the two things jsonschema cannot see: a `text_quote` present and blank (schema-valid, and the scorer aligns on that field ALONE so it matches nothing), and a quote that is not a substring of its own article — the only invention test available without a paid adjudicator. `--selftest` runs fifteen controls: twelve known-bad mutations each caught by name, and three NEGATIVE controls that must stay silent — a null `source_entity` (nullable in production's schema and null on most macro figures), a quote re-spaced but real (the substring test normalises whitespace on purpose), and an article with no figures at all (the corpus holds ten by design, and a checker that condemned them would condemn every negative in the set). |
| `test_check_staleness.py` | Unit tests over the staleness classifier, its self-check, and `main()`'s exit code. Fully offline. |
| `rescore_alignment_keys.py` | Re-scores CoD `numbers[]` predictions under ALTERNATIVE alignment keys, which `eval_harness.py` cannot do -- `_number_key` is hard-coded and has no flag. Answers one question only: is a published `numbers_f1` measuring the MODEL or the KEY? It imports the harness's own `_cod_align`, `_token_f1`, `_source_entity_affinity` and both floors rather than re-deriving them, so a change to the scorer moves these rows too. **No row it prints is acceptance evidence** and each says why in its own note: keys holding `value` read 1.0 by construction, and `macro_owner_waived` waives on the GOLD's `ent_type`, which presumes the very question it sizes. Two controls run every invocation: `shuffled_gold` (every prediction scored against a different article's gold; near-zero or `FLOOR_BREACHED`, judged PER RUN and naming the offending one -- a mean across runs diluted a 0.4570 breach to 0.0457 and printed `FLOOR_OK`) and, with `--scorecard`, `reproduces_published_numbers_f1` (the committed-key row must equal what the harness published for the same run, to 1e-9). `--selftest` mutation-verifies both on a synthetic corpus it builds -- fourteen controls, each complaint required BY NAME, three of them NEGATIVE controls that must stay quiet and three at n>1 runs, because a defect in how a tool AGGREGATES across units is invisible to a mutation exercised on one unit. FOUR pin the figures the tool itself publishes: the verdict must sweep all seven keys (a fixture that breaches on the value keys ALONE, committed at 0.0000), the bar must stay the harness's (a floor at 1.11x it, where every other fixture sits at 3.3x), one run must print `range n/a` and never `0.0000`, and a 3x3 witness where the waiver genuinely COSTS greedy two true positives holds the monotonicity counter and the max-cardinality headroom off zero. It needs no predictions file, which is what makes this tool checkable from the repo alone. Backs the alternative-key table in docs/BACKLOG.md MEASUREMENT DEBT, "The CoD gold cannot yet back a model swap" -- whose PREDICTIONS are not in the repo, the same disclosure the shuffled-gold figures below carry. |

## Typical workflow

```bash
# 1. (Re)build the substrate from upstream training-data inputs.
python LlmBenchmark/scripts/build_eval_substrate.py

# 2. Run a candidate model against any OpenAI-compatible engine.
#    --endpoint works for vLLM (:8000), SGLang, and llama.cpp's own /v1 (:8080).
python3 LlmBenchmark/scripts/run_model.py \
    --substrate /opt/ai-inference/training-data/eval-substrates/<dated>.json \
    --endpoint http://localhost:8000 \
    --model google/gemma-4-31B-it-qat-w4a16-ct \
    --out /tmp/preds.jsonl

# 2b. Same runner, HOSTED endpoint -- how a candidate is scored without a GPU reload.
#     --api-key-file takes a PATH, never the token: a token on a command line is in the
#     shell history, in ps output and in every transcript. It is never written to provenance.
#     A router is UNIDENTIFIABLE by construction (it does not expose the provider's engine
#     build), so --allow-unidentified-engine is required and honest there -- and the
#     `:provider` suffix is then the only attribution the scorecard can carry, which is why
#     the runner REFUSES a router model without one. Strict-schema enforcement is a
#     per-provider property: unpinned, the router picks, and it need not pick twice alike.
#     --task cod is REQUIRED to keep a CoD response: production's prompt returns one
#     OBJECT, the default `cove` reader expects an ARRAY, and without the flag the runner
#     writes `predicted_extractions: []` for every record and still exits 0.
#
#     THIS ROUTE CANNOT PRODUCE MODEL_ACCEPTANCE EVIDENCE, and the reason is not the
#     unidentified engine. Measured 2026-09-05: router.huggingface.co answers 404 to
#     /v1/completions, so `--endpoint-mode completions` -- production's own wire shape,
#     client-side template and all -- is unavailable there and the run falls back to the
#     chat endpoint, which is a DIFFERENT code path from the one production runs. The
#     scorecard says so on its own: `acceptance_evidence.production_prompt_path: false`.
#     Use this route to compare candidates without a GPU reload; take acceptance evidence
#     on a local engine that serves /v1/completions.
#     Also measured: concurrency 12 drew 504s from the router on 11 of 12 calls. Keep
#     --concurrency low here; the default 8 is tuned for a local engine, not a gateway.
python3 LlmBenchmark/scripts/run_model.py \
    --substrate /opt/ai-inference/training-data/eval-substrates/<dated>.json \
    --endpoint https://router.huggingface.co \
    --api-key-file ~/.hf-inference \
    --model Qwen/Qwen3.8-27B:deepinfra \
    --allow-unidentified-engine \
    --task cod \
    --prompt-file SentinelCollector/src/cod-prompts/cod_json_v1.txt \
    --schema-file SentinelCollector/src/cod-prompts/cod_json_schema_v1.json \
    --max-tokens 8192 \
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

**`--task cod` is a flag on BOTH halves, and they do not check each other.** The runner
decides which shape it KEEPS and under which key (`prediction` for CoD,
`predicted_extractions` for CoVe); the scorer decides which key it READS. A mismatch is
not an error on either side — the scorer simply finds nothing under the key it reads and
reports a model that extracted nothing. `provenance.task` records the runner's half, so
`--adapter-meta` is what lets a reader catch the disagreement afterwards.

```bash
# 1. PRODUCE CoD predictions. Without --task cod this writes `predicted_extractions: []`
#    for every record and exits 0 -- the request is right, the response is discarded.
#    --task cod REFUSES to run without --prompt-file and --schema-file (or
#    --no-structured-output): it changes only how the RESPONSE is parsed, so left alone
#    the request still carries the substrate's CoVe instruction and the array-shaped
#    default schema, and the model would comply and score zero.
#    EVERY SCORED SAMPLING AXIS IS SPELLED OUT rather than inherited, which is why four flags
#    below carry a value the parser would have supplied anyway. A DEFAULT IS WHAT LET THREE
#    AXES DRIFT AT ONCE: --concurrency defaults to 8 against a scored 6; --max-tokens defaults
#    to 4096 while this block asked for 8192; --repetition-penalty defaults to None, meaning
#    the knob is NOT SENT, while every scorecard records 1.1. Two of the three were still
#    wrong after the round that fixed the first, because a default is invisible in the command
#    a reader sees, moves with the parser, and never sits beside the number it must match.
#    THE VALUES BELONG TO THE SCORECARDS: max_tokens 4096, repetition_penalty 1.1,
#    temperature 0.0, seed 42, concurrency 6 -- what all twelve arms on
#    measure/common-coordinate-latest recorded under
#    acceptance_evidence.request_sampling.recorded, and what D-29's PRECOND carries. Read them
#    there, not here. At any other value the run is not wrong, it is UNCOMPARABLE to the row it
#    is checked against -- a re-score under MODEL_ACCEPTANCE, not a tune.
#    4096 WAS MEASURED SUFFICIENT, not assumed: 480 CoD responses across those twelve arms all
#    finished `stop`, 0 truncated. An earlier note here called 8192 "a floor" on the theory
#    that a CoD object does not fit in 4096; the scored runs refute it. A cut-off response does
#    fail json.loads and land in schema_invalid, which reads as bad JSON discipline rather than
#    as a budget finding -- so check `truncated` in the run summary, which counts it separately
#    for exactly that reason, and raise the budget only if it is non-zero (which re-scores).
#    THE UNSENT KNOBS STAY UNSPELLED, because "not sent" has no flag: top_p, top_k, min_p and
#    presence_penalty are null in every scorecard, and naming one would ADD an axis.
#    For the WIDE run that exists to break a sick engine rather than to score a healthy one,
#    see 1b below.
python3 LlmBenchmark/scripts/run_model.py --task cod \
    --substrate /opt/ai-inference/training-data/eval-substrates/<dated>.json \
    --endpoint http://localhost:8000 --model google/gemma-4-31B-it-qat-w4a16-ct \
    --endpoint-mode completions \
    --prompt-file SentinelCollector/src/cod-prompts/cod_json_v1.txt \
    --schema-file SentinelCollector/src/cod-prompts/cod_json_schema_v1.json \
    --chat-template $'<bos><|turn>user\n{0}<turn|>\n<|turn>model\n<|channel>thought\n<channel|>' \
    --concurrency 6 --temperature 0.0 --seed 42 --repetition-penalty 1.1 \
    --max-tokens 4096 --out /tmp/preds.jsonl

# 1b. THE FAULT PROBE -- A DIFFERENT RUN, AND ITS OUTPUT IS NOT A SCORE. One command cannot
#     do both jobs. The fp8_e5m2 fault class needs concurrent decode (>= 2) to appear at all:
#     the engine starts fine, serves ONE request, then faults and stays 503 -- and the deploy
#     gate is a /health wait plus one SEQUENTIAL 1-token completion, so it cannot see it and
#     reports success over a dead engine. Running WIDE is what surfaces that before production
#     serves a request.
#     IT DIVERGES FROM THE SCORED COORDINATE ON THREE AXES, NOT ONE, and naming only the width
#     would repeat the silent-inheritance defect one level up: width 8 where 6 was scored,
#     max_tokens 8192 where 4096 was, and no repetition_penalty where the scored runs sent 1.1.
#     Wide and long is what keeps decode concurrent for longer, which is what surfaces the
#     fault. So NEVER compare its numbers to a BENCHMARKS.md row or to the acceptance run
#     above; the output is named fault-probe-* for that reason.
#     What you read off it is whether the engine SURVIVED: call errors, 503s, truncation, and
#     whether it finished at all.
python3 LlmBenchmark/scripts/run_model.py --task cod \
    --substrate /opt/ai-inference/training-data/eval-substrates/<dated>.json \
    --endpoint http://localhost:8000 --model google/gemma-4-31B-it-qat-w4a16-ct \
    --endpoint-mode completions \
    --prompt-file SentinelCollector/src/cod-prompts/cod_json_v1.txt \
    --schema-file SentinelCollector/src/cod-prompts/cod_json_schema_v1.json \
    --chat-template $'<bos><|turn>user\n{0}<turn|>\n<|turn>model\n<|channel>thought\n<channel|>' \
    --concurrency 8 \
    --max-tokens 8192 --out /tmp/fault-probe-preds.jsonl

# 2a. SCORE, with gold. Gold rows are {source_file, source_index, gold: {...CoD object...}},
#     as JSONL, a JSON list, or a build_cod_gold.py artifact carrying them under `articles`
#     — which is what the committed gold file below is, and it is read directly. Only
#     records carrying BOTH gold and a prediction are scored, and `coverage` in the
#     scorecard says how many that was.
python3 LlmBenchmark/scripts/eval_harness.py --task cod \
    --substrate /opt/ai-inference/training-data/eval-substrates/<dated>.json \
    --cod-gold LlmBenchmark/cod-gold/cod_stage1_gold_v1.json \
    --predictions /tmp/preds.jsonl \
    --adapter-meta /tmp/preds.jsonl.provenance.json \
    --out /tmp/cod-scorecard.json

# 2b. SCORE, WITHOUT gold. Still worth running: the self-consistency metrics need none, and
#     every gold-dependent metric reports null with a reason rather than 0.0.
python3 LlmBenchmark/scripts/eval_harness.py --task cod \
    --substrate <substrate> --predictions /tmp/preds.jsonl \
    --adapter-meta /tmp/preds.jsonl.provenance.json --out /tmp/card.json
```

A `--limit`ed run writes `<out>.substrate.json` holding exactly those N records — score
against **that** file, not the full substrate, or the join drops every record the run
never made.

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
(`FLOOR_OK`). *Provenance of that corpus, stated because it is a caveat on the numbers and
not on the scorer:* it predates `run_model.py --task cod`, so the predictions file it was
taken over came from a one-off generator that is **not** in this repo. The scorer is
unaffected — those are its numbers over a corpus it was handed — but re-deriving them from
the repo alone means re-running step 1 above, which is a full 597-article GPU run.
The mock arm's own scorecard passed 22 of 24 measurable metrics — and both
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

## Why the artifact records the prompt's digest, not only its path

Same decay, one level down. `provenance.request_shape.prompt_file` is the path **as typed on
the command line**, and until now nothing opened it — so a finished run's prompt bytes were
unrecoverable. Measured 2026-09-06: of 25 scorecards on this host carrying a `request_shape`
at all, **22 share one byte-identical value** and every one stamps
`acceptance_evidence.production_prompt_path: true` — 8 of them scored the prompt as it stood
before #1017 rewrote that same path, 14 after, and the artifact does not change across the
boundary. (Those files live in `/tmp`, not the repo; every scorecard *committed* here
predates `request_shape` entirely.) The prompts are distinguishable in reality —
`SentinelCollector/src/cod-prompts/cod_json_v1.txt` against the host mount
`/opt/ai-inference/prompts/cod/cod_json_v1.txt`, which lag each other between deploys — and
were identical in the artifact. No wording of a caveat could recover that; three attempts
tried.

The runner now records `prompt_file_sha256`, `schema_file_sha256` and `chat_template_sha256`
**alongside** the paths — a path plus a hash is strictly more than a path — and the scorer
surfaces them in `acceptance_evidence.request_bytes`. A digest is written only for a file the
run actually **read**: `--no-structured-output` names a `--schema-file` and sends none of it,
so that run records the path and a `null` digest rather than attesting to bytes nothing sent.

A digest is written **only for bytes that reached a request**, and null otherwise — `--limit 0`,
`--no-structured-output`, a schema file whose JSON parses falsy (which `build_payload` silently
replaces with the built-in), a chat template outside completions mode. A digest reads like
proof, so one attesting to a file the run merely *named* would be worse than the bare path it
improves on. The scorer's `unrecorded` therefore keys off the **absence of the field**, not the
falsiness of its value: a run predating these digests carries no `*_sha256` key at all, while a
run that has them writes the key always and a null there is the runner *saying* the element was
not sent.

**Nothing here adjudicates, deliberately.** The scorer cannot see the mount production serves
from, so a hardcoded "production" digest compiled into it would be a guarantee it has no way
to keep — the same class of false claim being fixed. A human compares the recorded digest
against the mount, and now has something to compare. `production_prompt_path` is unchanged in
meaning and still narrow: it says the request had production's **shape** — completions mode,
a prompt file and schema file *named*, a template that really wraps — and never says **which**
prompt. A scorecard predating these fields says so in `request_bytes.unrecorded`, and one that
recorded no request shape at all says *that*, rather than borrowing the wording for a run that
sent nothing. An absent digest is not a matching one.

**The note is chosen per element, over three elements that need not agree.** One caption picked
for the whole block states a fact about all three from evidence about one, and every mixed record
is then over- or under-stated. So `request_bytes.note` is cut on what each NAMED element answers,
and the answer is three-valued, not two: its digest is **recorded**, or it is **null with its key
present** (the runner saying this element reached no request), or **its key is absent entirely**
(an artifact predating the digests, named in `unrecorded`). Six states follow:

| state | when |
|---|---|
| no shape at all | the run recorded no `request_shape` |
| shape naming nothing | the instruction came from the substrate |
| **recorded** | nothing named is missing a digest key, and at least one is identified — a named element sitting at null is *not* a gap and does not leave this state |
| **partial** *(new)* | at least one identified **and** at least one named element whose digest key is absent |
| **unverifiable** | nothing identified, and at least one named element's digest key is absent |
| **not sent** *(new)* | every named element carries its key and every one is null — nothing reached a request |

The two new states are the ones the old block mis-captioned. A `--limit 0` run that named
production's prompt was told it "named no prompt file … the instruction came from the substrate";
it is now *not sent*, which is a finding rather than a null. And a mixed record was forced into a
caption true of only one half; it is now *partial*. Separately, `recorded` was **re-worded** rather
than added: it used to send a run that digested only its chat template to check "the prompt and
schema files that served it" — files it never named — and now names no element at all. The wording
also stops treating all three elements as paths: `--chat-template` takes the **template itself**
and the artifact holds it verbatim, so a missing digest there costs the attestation that it
reached a request, never the content.

### And the same blindness on the sampling side

No conjunct of `production_prompt_path` reads a decoding knob either, so the shape can be
production's while the decoding is not. Measured 2026-09-06: benchmark runs sent
`repetition_penalty: null` and `max_tokens: 8192` where the service sends a loop guard
(`CpuCodOptions.JsonRepetitionPenalty` = 1.1) and half that budget, and every scorecard
stamped `production_prompt_path: true`. A run that decoded differently from production is
not a run on production's path, whatever the shape says — that is the finding, and it stands
on its own.

**What this section deliberately does not claim.** It is tempting to pin the article-35
entity runaway on the missing loop guard. The evidence does not carry it: *both* arms of the
two-arm prompt comparison ran with `repetition_penalty` unset, so a constant cannot explain a
difference that appeared in one arm only, and production's own rationale describes the trap as
one object *repeated*, where article 35 produced *distinct invented* names. `docs/BACKLOG.md`
holds that question open — whether the corrected prompt's `macro_indicator` instruction caused
the runaway or merely uncovered it — and one prompt edit plus one re-run settles it. Recording
sampling earns its place because the gap is real, not because it has been shown to be that
gap's cause.

The runner already recorded `provenance.sampling`; nothing surfaced it next to the pass. The
scorer now carries it as `acceptance_evidence.request_sampling`, under the same rule: it
**reports**, it does not compare. **Not sent is not unrecorded** — the runner omits an unset
knob rather than defaulting one nobody chose, so `repetition_penalty: null` inside a recorded
block means *this run ran without the loop guard*, which is a fact. A knob the artifact never
carried is named in `unrecorded` instead, per knob and not per block: `seed` was added to the
runner at #1009, so every scorecard written before it carries nine keys and no `seed` — judged
on the container alone those read as fully recorded, with the determinism knob a model
comparison turns on simply absent.

**The note now follows that per-knob data**; it used to be chosen for the block as a whole. So one
missing knob reached for the wording "this run wrote no sampling block" — false of a run that wrote
nine of ten. Re-scoring the eight scorecards committed here: **seven were given that sentence**,
which is every one of them produced by a real run. The eighth, the only one the wording fitted, is
the harness's own gold-tautology mock, which called no model. A partially recorded block now says
so and names the knobs it cannot speak for.

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
