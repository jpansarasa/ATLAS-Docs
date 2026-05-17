# Round 2 CoD prompts — v1 DSL target

Phase 0 of the Round 2 CoD benchmark. These prompts replace the prose-CoD
prompts from Round 1 and instead target the **Sentinel CoD DSL v1**
defined in `docs/plans/sentinel-edge-realization.md` Sec. 6.

Goal: measure whether per-model-class DSL prompts beat the universal
prose prompts from Round 1 on the same corpus, across:

- **DSL grammar validity** — does the output parse with the v1 parser?
- **Token preservation** — same metric as Round 1.
- **Schema completeness** — required slots populated on every block?
- **Downstream-extraction success** — does the DSL -> JSON parser
  produce clean rows, or does it fail / produce garbage?

## Files

```
cod_dsl_classA.txt           Class A template (small instruct, <=14B)
cod_dsl_classB.txt           Class B template (mid instruct, 20-30B)
cod_dsl_classC.txt           Class C template (large instruct, >=30B)
cod_dsl_worked_example.txt   Worked example (appendable A/B arm)
README.md                    This file
```

Every class template carries the full v1 grammar, the value-type
literals, and the eight locked design decisions — those are the
contract, not optional context.

What differs is the **scaffolding** around the contract:

| Class | Size band | Example blocks in base template | Format-guard tone |
|-------|-----------|--------------------------------|-------------------|
| A     | <=14B     | 3 mini-examples (ENT + EVT + CLAIM) | Strict, terse, "wrong syntax = rejected" |
| B     | 20-30B    | 1 minimal anchor example       | Balanced, declarative |
| C     | >=30B     | None (spec-only)                | Spec-driven, no hand-holding |

All three end with the same directive:

> Output ONLY the DSL document (header + blocks). No prose, no
> markdown, no explanations.

## Benchmark matrix

The Round 2 runner sweeps **model class x prompt x A/B example**:

| Axis            | Values |
|-----------------|--------|
| model_class     | A, B, C |
| prompt          | the class's own template |
| example_arm     | `base`  (template alone)  /  `with_example` (template + worked example appendix) |
| corpus          | shared Round 2 corpus (top-3 to top-5 prose-Round-1 models' source set) |

Per-class candidate models (initial set; runner reads from a
manifest, not from this README):

- **Class A**: `qwen2.5:7b`, `qwen3:8b`, `granite4-micro`, `phi4-mini`
- **Class B**: `mistral-small`, `gemma3`
- **Class C**: `qwen2.5:32b`, `sentinel-cove-v6.2-base`

Universal prompts were rejected up front: they bias one model class
over another, which confounds the prompt-vs-capacity question that
Round 2 is trying to answer.

## How the runner consumes these files

The runner is a thin wrapper. For each `(model, example_arm)` cell:

1. Look up the model's class -> pick the class template file.
2. If `example_arm == base`:
     prompt = file(class_template)
   else (`with_example`):
     prompt = file(class_template) + "\n\n" + file(cod_dsl_worked_example.txt)
3. Substitute the input placeholders in the prompt:
     `{{source_id}}`, `{{published_at}}`, `{{article_text}}`
4. Send the rendered prompt to the model with deterministic decoding
   (low temperature, capped max_tokens sized to the corpus's worst
   case + headroom).
5. Pipe the raw model output to the v1 DSL parser. Capture:
     - parse_ok (bool)
     - parse_error_class (if !ok)
     - block_count, ent_count, evt_count, claim_count, note_count
     - required_slot_completeness (per block)
6. Score against the corpus ground-truth (token preservation,
   schema completeness, downstream-extraction success).

The runner's manifest lives outside this directory (alongside the
benchmark code). These prompt files are **inputs only** — no logic,
no per-model conditional branches inside the templates.

## Placeholders

All three templates expect exactly these substitutions:

| Placeholder         | Meaning                                |
|---------------------|----------------------------------------|
| `{{source_id}}`     | URL or stable id of the source        |
| `{{published_at}}`  | ISO-8601 publish timestamp             |
| `{{article_text}}`  | the raw article body (no markup)       |

If you add a placeholder, update all three class templates AND this
README in the same change — drift across class templates is a
benchmark-validity bug, not a stylistic issue.

## Adding a new model class

When a new class is justified (e.g., a coding-tuned model in a band
that the existing classes do not represent well), add it by:

1. Copy the nearest existing class template to `cod_dsl_class<X>.txt`.
2. Adjust **only** the scaffolding — examples, format-guard tone,
   terseness — not the grammar, value-type literals, or the eight
   locked decisions. Those are the contract.
3. Add a row to the "Files" and matrix tables in this README.
4. Update the runner's manifest to map the new model(s) to the new
   class.
5. Run the full A/B example arm against at least one model in the
   new class before publishing scores for it.

## Non-goals for this phase

- No parser implementation lives here. The v1 parser is built in
  Phase 1 of the Round 2 plan.
- No production prompts at `/opt/ai-inference/prompts/` are touched
  by Phase 0; deployment happens after Round 2 picks a winning
  (class, prompt, example_arm) cell.
- No scoring code lives here. Scoring is the runner's job.
