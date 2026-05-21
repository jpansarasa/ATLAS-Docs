# DSL PoC — Phase 3 — v2 schema (copy-mode-first + source_span pointers)

**Status:** scoping. No code, no deploy. Single source of truth for Phase 3
work until split into stories and merged.
**Parent plan:** `docs/plans/atlas-dsl-poc-plan.md` §6.
**Predecessor:** Phase 2 (`atlas-dsl-poc-phase2-llama-server.md`, PR #387 +
sweep results + v1.2 terminator in PR #389).
**Topology rule (immovable, PR #386):** CPU = extraction (this plan still on
`ollama-cpu-gen` + `llama-server`). GPU = summarization + verification
(Phase 4, only the deterministic verifier skeleton lands here as a Phase 3
story — the LLM-based semantic tier stays Phase 4).
**Decision context (Phase 2 outcome, plan §11.1 / §11.1.1):** v1.2's `END`
terminator validated the truncation-vs-loop mechanism on n=3 prior-fail
articles, but the underlying §5.3 acceptance gate (`grammar_valid = 100%`
+ recall) was not cleared. Phase 3 attacks the schema itself: every block
emits its verbatim copy slot(s) first, every copy slot carries a
`source_span: [start, end]` integer pair pointing into the input article,
and the deterministic verifier validates `output_value == input[start:end]`
byte-for-byte. This converts the entity- and number-hallucination class
into a hard constraint instead of a soft metric.

---

## Goal

Define v2 of the CoD DSL — a copy-zone-first schema with first-class
provenance pointers (`source_span: [start, end]` integer pairs into the
input article) — and stand up enough plumbing (grammar, prompt, runner
schema flag, parser, deterministic verifier skeleton, scoring extension)
to score qwen3:30b on the v2 spec arm against the same n=10 corpus used in
Phase 1.x / Phase 2. v2 lives **alongside** v1.2: both grammars, both
prompts, both parser entry points, both result trees. The runner gains a
`--schema {v1|v2}` flag; v1 is the default for backward compat with every
prior result. The deterministic verifier (plan §7.1, normally Phase 4)
lands here as a skeleton because v2's whole reason to exist is to make
that verifier a one-line string-equality check.

## Non-goal

This plan does **not** retire v1.2 (the v1.2 grammar, prompts, runner
path, and result trees stay intact and addressable until Phase 4 closes
out and v2 demonstrably ships into the matrix pipeline). It does not
touch the GPU semantic verifier (Phase 4 §7.2 — LLM-based claim
verification, sector tagging, entity resolution stay out of scope). It
does not change the inference topology (extraction stays on CPU; the
deterministic verifier is pure Python with no LLM call so it can run
co-located with the runner). It does not modify production prompts at
`/opt/ai-inference/prompts/`. It does not extend the v1.2 GBNF / Lark
grammar — v2 is a parallel grammar file, parallel prompt file, parallel
parser entry. It does not run a full multi-class / multi-model sweep —
Phase 3 acceptance is single-arm (qwen3:30b, v2 spec) on the n=10 corpus,
plus the 3 prior-fail articles (27560, 31149, 31430) as smoke.

---

## Recon (load-bearing facts)

Captured here so each story can reference rather than re-derive.

| Fact | Value |
|---|---|
| v1.2 GBNF | `docs/grammars/cod-dsl-v1.gbnf` — header + 1-128 blocks + `END\n` terminator; SHAPE only, slot VALUES free-form. Repetition bounds (`{0,255}` on value, `{0,128}` on block-list, `{0,64}` on idents) hand-tuned for llama.cpp's `MAX_REPETITION_THRESHOLD=2000` + rule-expansion combinatorial cap (see header comment above `value` rule). v2 inherits ALL of these caps unchanged. |
| v1 Lark grammar | `docs/benchmarks/cod-2026-05-17/dsl/grammar.lark` — block-and-slot shape; `SLOT_LINE` captures the full `  - name: value\n` line as a single terminal; `end_marker` optional for backward compat. |
| Parser | `docs/benchmarks/cod-2026-05-17/dsl/parser.py` — two-pass (Lark tree → Document; resolve ENT refs). Lenient front-end (`_normalize_for_lenient_parse`) handles 4+ shape repairs (multi-word ENT names, multi-word slot names, bare CLAIM, nested list items, header repair, trailing-narrative pruning, in-block blank-line collapse). v2 parser MUST keep the same lenient front-end behavior — too much hard-won corner-case handling to relitigate. |
| AST types | `docs/benchmarks/cod-2026-05-17/dsl/types.py` — `Slot`, `ENT`, `EVT`, `CLAIM`, `NOTE`, `Document`. `SourceSpan = Union[str, list[str]]` today (v1 carries the verbatim quote-text as the source_span value, NOT an offset pair). v2 must extend this to also carry `tuple[int, int]` while preserving the existing `str | list[str]` shape for v1 documents. |
| Class B prompt | `docs/benchmarks/cod-2026-05-17/prompts/cod_dsl_classB.txt` — grammar-taught (full GRAMMAR section + value-type literals + 8 locked design decisions + minimal anchor example). Ends with the `END`-line-is-REQUIRED directive added for v1.2. v2 prompt is class-agnostic (single template — Phase 3 runs qwen3:30b only). |
| Prompt assembly | `docs/benchmarks/cod-2026-05-17/scripts/prompt_assembly.py` — `assemble_prompt(model, variant, ...)`; `VALID_VARIANTS` is a flat tuple. v2 adds variants `v2_spec` (and reserves `v2_spec+example` though no example template lands in Phase 3). |
| Runner | `docs/benchmarks/cod-2026-05-17/scripts/run_cod_dsl.py` — `run_cell(...)` with `backend`, `grammar_path`, `n_predict` kwargs. v2 adds a `--schema {v1|v2}` flag that (a) selects the prompt variant (`v2_spec` instead of `spec`), (b) selects the GBNF file (`docs/grammars/cod-dsl-v2.gbnf` instead of v1), (c) routes results to `results/phase3-v2/<model_safe>/<variant>/<id>.json` (separate tree — no cross-contamination with Phase 2 / Round 2/3). |
| Scorer | `docs/benchmarks/cod-2026-05-17/scripts/score_dsl.py` — Phase 0 instrumentation in place (`ticker_recall`, `org_recall`, `numeric_exact_match_rate`, `numeric_semantic_match_rate`, `unit_preservation_rate`, `span_copy_fraction`). v2 adds `verbatim_match_rate` (fraction of copy slots whose value equals `input[source_span]`) and `span_validity_rate` (fraction of copy slots whose `source_span` is a well-formed `[int, int]` within `[0, len(input))`). |
| Aggregator | `docs/benchmarks/cod-2026-05-17/scripts/aggregate_report.py` — `_iter_results` globs `results/<round>/*/*/*.json` (any tree under `results/`). New v2 metric fields surface automatically once the scorer emits them; aggregator gets a one-line extension for the v2-only metric columns. |
| n=10 corpus | `docs/benchmarks/cod-2026-05-17/corpus.jsonl` (full72 also available). Articles 27560, 31149, 31430 are the v1.2 prior-fail set used as Phase 3 smoke. |
| Ground truth | `docs/benchmarks/cod-2026-05-17/ground-truth.jsonl` — same shape as Round 1/2/3; v2 scoring reuses it as-is for recall, and adds verifier-side metrics that are scored against the input article text (no ground-truth dependency). |
| Plan §7.1 contract | "Every copy slot's value MUST be a verbatim substring of `input_text[source_span[0]:source_span[1]]`. If it isn't, the block is rejected by the verifier." Phase 3 implements this exactly. |

### What stays from v1.2 unchanged

- The header (`DSL: v2` is the only line change — version tag bumps).
- The `END\n` terminator (PR #389 mechanism). v2 keeps it; the prompt
  keeps its "emit `END` and stop" directive. Without this the constrained
  decoder has no stop criterion and we re-introduce the truncation
  pathology.
- All llama.cpp repetition bounds: `{0,128}` block-list cap, `{0,255}`
  value char cap, `{0,64}` ident-tail caps, `{0,127}` qual-val cap. These
  were calibrated against the parser's `MAX_REPETITION_THRESHOLD=2000` +
  combinatorial rule-expansion cap (see `cod-dsl-v1.gbnf` header). v2 has
  more rules (copy zone + derived zone per block), so the combinatorial
  budget is tighter, not looser — DO NOT raise any bound without
  re-probing llama.cpp.
- The lenient Lark front-end. Every preprocessor pass
  (`_repair_header`, `_repair_nested_list_items`, `_prune_trailing_narrative`,
  `_collapse_blank_lines_in_blocks`, `_normalize_for_lenient_parse`) carries
  forward unchanged for v2. Model-output shape pathologies are
  schema-version-orthogonal.
- The `source_span` slot NAME. v1 carried it as a verbatim string; v2
  carries it as a `[start, end]` integer pair. Same slot name keeps the
  AST extraction path (`_extract_source_span`) in one place; the type of
  the parsed value is what differs.

### What changes for v2

- Every block has an explicit COPY ZONE (first) and DERIVED ZONE (after).
  In the GBNF this surfaces as a fixed slot-order — `slot` is no longer
  a free-repeat list; each block has named copy-slot rules and named
  derived-slot rules emitted in spec'd order.
- `NUM` becomes a first-class block kind (plan §6.1 principle 4). v1
  buried numerics inside EVT / CLAIM slot values; v2 promotes them so
  they can carry their own `source_span` and be verifier-checkable
  independently.
- Every copy slot carries a `source_span: [start, end]` integer pair as
  the immediately-following slot. The grammar makes this REQUIRED.
- The `subject` slot (which holds ENT refs) is unchanged shape — refs
  are NOT copy slots, they're symbolic resolution targets.

---

## v2 schema design

### Block-level skeleton (Lark)

```lark
// docs/benchmarks/cod-2026-05-17/dsl/grammar_v2.lark
//
// v2 differs from v1 by: (a) version tag is v2, (b) NUM block kind
// added, (c) copy-slot + source_span slots are grammar-required and
// emitted in a fixed order per block.
//
// EVERYTHING else (header shape, SLOT_LINE terminal regex, qualifier
// shape, %ignore comments) mirrors v1.

start: header _NL? (block _NL?)* end_marker?

end_marker: "END" _NL

header: "DSL:" version_tag _NL \
        "SOURCE:" header_value _NL \
        "TIMESTAMP:" header_value _NL

version_tag: /v2/      // v2 ONLY for this grammar; v1 docs use grammar.lark

block: ent_block | num_block | evt_block | claim_block | note_block

// ---------- ENT (copy zone: name + source_span; derived zone: type, ticker) ----------

ent_block: "ENT" ent_id "/" ent_type qualifiers? _NL ent_copy_slots ent_derived_slots?

ent_copy_slots: name_slot source_span_slot
ent_derived_slots: (ticker_slot ticker_span_slot?)? other_slot*

name_slot:        SLOT_LINE_NAME            // "  - name: \"...\"\n"
source_span_slot: SLOT_LINE_SOURCE_SPAN     // "  - source_span: [<int>, <int>]\n"
ticker_slot:      SLOT_LINE_TICKER          // "  - ticker: \"...\"\n"
ticker_span_slot: SLOT_LINE_TICKER_SPAN     // "  - ticker_source_span: [<int>, <int>]\n"
other_slot:       SLOT_LINE                 // any other "  - key: value\n"

// ---------- NUM (copy zone: raw + source_span [+ context_span]; derived zone: value, unit, magnitude) ----------

num_block: "NUM" num_id _NL num_copy_slots num_derived_slots

num_copy_slots:    raw_slot source_span_slot context_span_slot?
num_derived_slots: value_slot unit_slot magnitude_slot? other_slot*

raw_slot:           SLOT_LINE_RAW            // "  - raw: \"...\"\n"
context_span_slot:  SLOT_LINE_CONTEXT_SPAN   // "  - context_span: [<int>, <int>]\n"
value_slot:         SLOT_LINE_VALUE          // "  - value: <float>\n"
unit_slot:          SLOT_LINE_UNIT           // "  - unit: <enum>\n"
magnitude_slot:     SLOT_LINE_MAGNITUDE      // "  - magnitude: <enum>\n"

// ---------- EVT (copy zone: trigger + trigger_span; derived zone: subject, time, refs) ----------

evt_block: "EVT" event_kind qualifiers? _NL evt_copy_slots evt_derived_slots

evt_copy_slots:    trigger_slot trigger_span_slot
evt_derived_slots: subject_slot time_slot? other_slot*

trigger_slot:      SLOT_LINE_TRIGGER        // "  - trigger: \"...\"\n"
trigger_span_slot: SLOT_LINE_TRIGGER_SPAN   // "  - trigger_span: [<int>, <int>]\n"
subject_slot:      SLOT_LINE_SUBJECT        // existing v1 ENT-ref shape
time_slot:         SLOT_LINE_TIME

// ---------- CLAIM (copy zone: evidence_span (no value, span only); derived: subj/pred/obj/polarity) ----------
//
// NOTE: CLAIM is the one block where the copy zone is span-ONLY — the
// "verbatim" is implicitly input[evidence_span[0]:evidence_span[1]]; the
// slot doesn't repeat it.  This trades schema symmetry for output
// brevity; the verifier still has everything it needs.

claim_block: "CLAIM" claim_kind qualifiers? _NL claim_copy_slots claim_derived_slots

claim_copy_slots:    evidence_span_slot
claim_derived_slots: subject_slot predicate_slot? object_slot? polarity_slot? other_slot*

evidence_span_slot:  SLOT_LINE_EVIDENCE_SPAN  // "  - evidence_span: [<int>, <int>]\n"
predicate_slot:      SLOT_LINE_PREDICATE
object_slot:         SLOT_LINE_OBJECT
polarity_slot:       SLOT_LINE_POLARITY

// ---------- NOTE (escape hatch; copy zone optional) ----------

note_block: "NOTE" note_label? _NL (source_span_slot)? other_slot*

// Terminals: every SLOT_LINE_* variant is a refinement of the v1
// SLOT_LINE regex anchored on the slot name. Authoring approach: keep
// SLOT_LINE as the catch-all but constrain individual slot-name terminals
// for the typed slots so the LALR table picks the right production.
// (If the LALR conflicts are intractable we fall back to v1's single
// SLOT_LINE terminal + Python-side ordering validation; see Open Q4.)
```

### Block-level skeleton (GBNF)

```gbnf
# docs/grammars/cod-dsl-v2.gbnf
#
# Mirrors v2 Lark. SHAPE only — value content (numbers, enums, quoted
# strings) free-form; the Python verifier enforces typing post-hoc.
#
# Repetition bounds inherited unchanged from v1.2 (see cod-dsl-v1.gbnf
# header notes — DO NOT raise without re-probing llama.cpp's
# MAX_REPETITION_THRESHOLD + rule-expansion cap).

root ::= header block-list end-block

header ::= "DSL: v2\n" "SOURCE: " line-rest "\n" "TIMESTAMP: " line-rest "\n"

block-list ::= ("\n"* block){0,128} "\n"*

end-block ::= "END\n" "\n"*

block ::= ent-block | num-block | evt-block | claim-block | note-block

# ENT: header + name copy slot + source_span + optional derived slots.
ent-block ::= "ENT " ent-id " / " ent-type qualifiers? "\n"
              ent-name-slot
              source-span-slot
              ent-derived-slot*

ent-name-slot     ::= "  - name: \"" string-content "\"\n"
source-span-slot  ::= "  - source_span: [" int ", " int "]\n"

ent-derived-slot ::= ticker-slot
                   | ticker-span-slot
                   | generic-slot

ticker-slot      ::= "  - ticker: \"" string-content "\"\n"
ticker-span-slot ::= "  - ticker_source_span: [" int ", " int "]\n"

# NUM: header + raw copy slot + source_span + optional context_span + derived.
num-block ::= "NUM " ident "\n"
              num-raw-slot
              source-span-slot
              context-span-slot?
              num-derived-slot*

num-raw-slot      ::= "  - raw: \"" string-content "\"\n"
context-span-slot ::= "  - context_span: [" int ", " int "]\n"

num-derived-slot ::= num-value-slot | num-unit-slot | num-mag-slot | generic-slot
num-value-slot   ::= "  - value: " value "\n"
num-unit-slot    ::= "  - unit: " value "\n"
num-mag-slot     ::= "  - magnitude: " value "\n"

# EVT: header + trigger copy slot + trigger_span + derived.
evt-block ::= "EVT " event-kind qualifiers? "\n"
              evt-trigger-slot
              evt-trigger-span-slot
              evt-derived-slot+

evt-trigger-slot      ::= "  - trigger: \"" string-content "\"\n"
evt-trigger-span-slot ::= "  - trigger_span: [" int ", " int "]\n"
evt-derived-slot      ::= generic-slot   # subject/time/refs go here

# CLAIM: header + evidence_span (span-only copy zone) + derived.
claim-block ::= "CLAIM " claim-kind qualifiers? "\n"
                evidence-span-slot
                claim-derived-slot+

evidence-span-slot ::= "  - evidence_span: [" int ", " int "]\n"
claim-derived-slot ::= generic-slot

# NOTE: optional source_span + free slots.
note-block ::= "NOTE" note-label-opt "\n" source-span-slot? generic-slot*

# Generic slot — falls back to v1.2's slot shape for everything not
# slot-named in the typed productions above (subject, time, predicate, ...).
generic-slot ::= "  - " slot-name ": " value "\n"

# Integer literal for span pointers (digits, bounded — see header notes
# on bound calibration).
int ::= digit digit{0,9}        # up to 10 digits = 10^10 chars headroom
digit ::= [0-9]

# All other terminals (ent-id, ent-type, slot-name, value, line-rest,
# string-content, qualifiers, etc.) inherited verbatim from v1.2.
# Bounds (block-list {0,128}, value {0,255}, ident-tail {0,63}, qual-val
# {0,127}) carry forward unchanged.
```

### The hard verifier contract (plan §7.1)

For every block with a copy slot, the verifier validates:

```python
# For ENT.name:
assert ent.name == input_text[ent.source_span[0]:ent.source_span[1]]
# For NUM.raw:
assert num.raw == input_text[num.source_span[0]:num.source_span[1]]
# For EVT.trigger:
assert evt.trigger == input_text[evt.trigger_span[0]:evt.trigger_span[1]]
# For CLAIM.evidence_span (span-only — no copy text to compare):
assert 0 <= claim.evidence_span[0] < claim.evidence_span[1] <= len(input_text)
# For NUM.context_span (optional):
assert input_text[num.source_span[0]:num.source_span[1]] \
       in input_text[num.context_span[0]:num.context_span[1]]
```

Any failure → block rejected, logged, NOT counted toward recall. The
acceptance gate (below) is for blocks that PASS verification.

---

## Story breakdown

Seven stories, ordered. Each names files, acceptance, explicit `git add`
paths. Stories 1-3 are pure scaffolding (no behavior change to v1 path);
4-7 add v2 behavior and metrics.

### Story 1 — v2 grammar files (GBNF + Lark)

**Files (new):**
- `docs/grammars/cod-dsl-v2.gbnf` (new — GBNF skeleton above, fully
  authored with bound notes + design comments mirroring v1.2 style).
- `docs/benchmarks/cod-2026-05-17/dsl/grammar_v2.lark` (new — Lark
  skeleton above; lives next to v1's `grammar.lark`).

**Acceptance:**
- `docs/grammars/cod-dsl-v2.gbnf` posts to `llama-server` via the existing
  canary probe (`_probe_grammar_pipeline_healthy`) without
  `MAX_REPETITION_THRESHOLD` or rule-expansion errors. (One-shot probe;
  no LLM call needed.)
- `Lark(grammar_v2.lark.read_text(), start="start", parser="lalr")`
  constructs without conflicts. If LALR conflicts surface (likely on the
  typed-slot terminals — see Open Q4), fall back to v1's single
  `SLOT_LINE` terminal + Python-side ordering validation; document the
  fallback in the grammar header comment.
- v1.2 grammar files unchanged.

**Git add:**
- `docs/grammars/cod-dsl-v2.gbnf`
- `docs/benchmarks/cod-2026-05-17/dsl/grammar_v2.lark`

### Story 2 — v2 prompt template

**Files (new):**
- `docs/benchmarks/cod-2026-05-17/prompts/cod_dsl_v2.txt` (new — class-
  agnostic; full v2 GRAMMAR section + COPY ZONE / DERIVED ZONE
  explanation + worked example with explicit `[start, end]` source_span
  pairs + the v1.2 `END`-line directive).

The worked example MUST show:
- One ENT block with `name`, `source_span: [start, end]`, derived fields.
- One NUM block with `raw`, `source_span`, `value`, `unit`.
- One EVT block with `trigger`, `trigger_span`, `subject` (ENT ref).
- One CLAIM block with `evidence_span` (no copy text), derived fields.
- The source sentence labelled with character offsets so the model sees
  what the span integers correspond to.

**Acceptance:**
- `prompt_assembly.py` can render the v2 template (Story 3 wires the
  variant; Story 2 just authors the file).
- Template's required-placeholder set matches v1 (`source_id`,
  `published_at`, `article_text`).

**Git add:**
- `docs/benchmarks/cod-2026-05-17/prompts/cod_dsl_v2.txt`

### Story 3 — runner `--schema {v1|v2}` flag + prompt-variant wiring

**Files (modified):**
- `docs/benchmarks/cod-2026-05-17/scripts/prompt_assembly.py` — add
  `v2_spec` to `VALID_VARIANTS`; route to `cod_dsl_v2.txt` template.
- `docs/benchmarks/cod-2026-05-17/scripts/run_cod_dsl.py` — add
  `--schema {v1|v2}` flag (default `v1`); when `v2`, select grammar
  `docs/grammars/cod-dsl-v2.gbnf`, prompt variant `v2_spec`, and route
  results to `results/phase3-v2/<model_safe>/<variant>/<id>.json`.

**Acceptance:**
- `run_cod_dsl.py qwen3:30b-... --schema v1 --variant spec --article-id <id>`
  produces a result file under `results/dsl-roundN/...` IDENTICAL to a
  current run (additive change only — defaults preserve all historical
  behavior).
- `run_cod_dsl.py qwen3:30b-... --schema v2 --variant v2_spec --article-id <id>`
  posts the v2 prompt + v2 GBNF and writes to `results/phase3-v2/...`.
- Default-without-`--schema` continues to write to `results/dsl-roundN/`.
- New unit test asserts the result-path selection + variant-to-grammar
  mapping (no live LLM call required).

**Git add:**
- `docs/benchmarks/cod-2026-05-17/scripts/prompt_assembly.py`
- `docs/benchmarks/cod-2026-05-17/scripts/run_cod_dsl.py`
- `docs/benchmarks/cod-2026-05-17/scripts/tests/test_run_cod_dsl_schema_v2.py` (new)

### Story 4 — AST extension: source_span carries `tuple[int, int]`

**Files (modified):**
- `docs/benchmarks/cod-2026-05-17/dsl/types.py` — widen `SourceSpan` to
  `Union[str, list[str], tuple[int, int]]`. Add a `is_span_pointer`
  helper so verifier / scorer can branch.
- `docs/benchmarks/cod-2026-05-17/dsl/parser.py` — new parser entry
  `parse_text_v2(text: str)` that loads `grammar_v2.lark`, walks the v2
  tree (typed slots), and emits the same `Document` AST. The lenient
  front-end (`_normalize_for_lenient_parse`) is reused unchanged.
  `_extract_source_span` is extended to recognize the `[int, int]` shape
  and emit a 2-tuple; the v1 `str | list[str]` path is untouched.

The v2 entry MUST NOT regress v1: `parse_text(...)` still loads
`grammar.lark` and emits v1-shape spans.

**Acceptance:**
- Existing parser tests (`dsl/tests/`) all pass unchanged.
- New tests for `parse_text_v2`:
  - Round-trips an authored v2 fixture (header + 1 ENT + 1 NUM + 1 EVT +
    1 CLAIM + END).
  - Emits `source_span` as `(int, int)` on every copy-bearing block.
  - Rejects v2 input that omits a required copy slot.
  - Rejects v1 input (wrong version tag).

**Git add:**
- `docs/benchmarks/cod-2026-05-17/dsl/types.py`
- `docs/benchmarks/cod-2026-05-17/dsl/parser.py`
- `docs/benchmarks/cod-2026-05-17/dsl/tests/test_parser_v2.py` (new)
- `docs/benchmarks/cod-2026-05-17/dsl/tests/fixtures/v2_minimal.dsl` (new)

### Story 5 — deterministic verifier skeleton (plan §7.1)

**Files (new):**
- `docs/benchmarks/cod-2026-05-17/dsl/verifier.py` (new) —
  `verify_document(doc: Document, input_text: str) -> VerificationReport`.
  Pure Python, no LLM call.

`VerificationReport` per block emits:
- `block_kind`, `block_id` (ENT local_id / NUM id / EVT event_kind / CLAIM
  claim_kind)
- `verdict ∈ {pass, soft_fail, hard_fail}`
- `failures`: list of `{slot, expected, got, reason}`

Hard-fail criteria (plan §7.1):
- copy-slot value ≠ `input_text[source_span[0]:source_span[1]]`
- `source_span` not a well-formed `[int, int]` with
  `0 <= start < end <= len(input_text)`
- ENT/NUM ref in EVT/CLAIM doesn't resolve to a declared block

Soft-fail criteria (logged, not blocking):
- NUM `value_normalized` doesn't reconcile with `raw` + `unit` +
  `magnitude` (e.g., `raw="$5.3 billion"`, `unit="USD"`, `magnitude="BILLIONS"`,
  derived `value` ≠ 5_300_000_000). Plan §7.1 line 3. Phase 3 lands
  the check; the reconciliation rules themselves are a Phase 4
  enrichment.

Out of scope (Phase 4): semantic / LLM verifier (claim verification,
sector tagging, entity catalog lookup). Plan §7.2.

**Acceptance:**
- Unit tests cover: (a) all-pass document, (b) one hard-fail per failure
  mode, (c) NUM soft-fail, (d) UTF-8 input with non-ASCII chars (the
  source_span semantics — byte vs char offset — are pinned by these
  tests; see Open Q1).
- Verifier runs in <100ms on a typical n=10 corpus document (no LLM).

**Git add:**
- `docs/benchmarks/cod-2026-05-17/dsl/verifier.py`
- `docs/benchmarks/cod-2026-05-17/dsl/tests/test_verifier.py`
- `docs/benchmarks/cod-2026-05-17/dsl/tests/fixtures/v2_*.dsl` (extra
  fixtures for fail-mode coverage)

### Story 6 — scoring extension: verbatim_match_rate + span_validity_rate

**Files (modified):**
- `docs/benchmarks/cod-2026-05-17/scripts/score_dsl.py` — extend
  `score(...)` to also emit, when `article_text` is supplied AND the
  document parses as v2:
  - `verbatim_match_rate`: fraction of copy slots (ENT.name, NUM.raw,
    EVT.trigger) where `value == input[source_span]` byte-for-byte.
  - `span_validity_rate`: fraction of copy slots whose source_span is a
    well-formed `[int, int]` within input bounds.
  - `verifier_pass_rate`: fraction of blocks for which the verifier
    returns `pass`.
  All three default to `None` when the document is v1 (preserves Phase 0+
  result-schema additivity).
- `docs/benchmarks/cod-2026-05-17/scripts/aggregate_report.py` — surface
  the three new metrics in a v2-only column block in the report;
  v1 results render `n/a`.

**Acceptance:**
- Scorer unit tests: v2 fixture scores expected values; v1 fixture emits
  `None` for the three new metrics (no regression on Phase 0 schema).
- Aggregator runs on the existing `results/dsl-round3` tree and produces
  an identical-modulo-new-columns REPORT.md (no v2 result files yet, so
  the v2 columns are all `n/a` in the smoke run).

**Git add:**
- `docs/benchmarks/cod-2026-05-17/scripts/score_dsl.py`
- `docs/benchmarks/cod-2026-05-17/scripts/aggregate_report.py`
- `docs/benchmarks/cod-2026-05-17/scripts/tests/test_score_dsl_v2.py`

### Story 7 — smoke run on 3 prior-fail articles + n=10 acceptance

**Files (new — results only):**
- `docs/benchmarks/cod-2026-05-17/results/phase3-v2/qwen3_30b.../v2_spec/27560.json`
- `docs/benchmarks/cod-2026-05-17/results/phase3-v2/qwen3_30b.../v2_spec/31149.json`
- `docs/benchmarks/cod-2026-05-17/results/phase3-v2/qwen3_30b.../v2_spec/31430.json`
- ... + the other 7 articles from the n=10 corpus.
- `docs/benchmarks/cod-2026-05-17/results/phase3-v2/REPORT.md` (generated).

**Acceptance (Phase 3 gate; plan §6.4 rephrased):**

On `qwen3:30b-a3b-instruct-2507-q4_K_M`, v2 spec arm, n=10 corpus:
1. `numeric_exact_match_rate ≥ 0.85` (mean across the n=10).
2. `ticker_recall ≥ 0.40` (mean).
3. `org_recall ≥ 0.50` (mean).
4. `verifier_pass_rate ≥ 0.95` per article (deterministic verifier
   accepts ≥95% of blocks the model emits — the verifier validates 100%
   of copy slots WITHOUT an LLM call).
5. `verbatim_match_rate` reported alongside as the underlying mechanism
   evidence (no threshold; the gate is verifier_pass_rate, which is a
   superset check).
6. The 3 prior-fail articles produce parseable v2 output (grammar_valid
   = true on all 3).

If gate 4 fails it points at a prompt-engineering problem (model
emits copy slots whose values don't match the spans it cites). If gates
1-3 fail it points at a recall problem (model is hallucinating less but
also extracting less). If gate 6 fails the schema itself or the
v1.2-style truncation pathology is back.

**Git add (sequential commits after gate passes):**
- All result JSONs under `results/phase3-v2/qwen3_30b.../v2_spec/`.
- `results/phase3-v2/REPORT.md`.

---

## Acceptance criteria for Phase 3 (consolidated)

On qwen3:30b v2 spec arm, n=10:
- `numeric_exact_match_rate ≥ 0.85`
- `ticker_recall ≥ 0.40`
- `org_recall ≥ 0.50`
- `verifier_pass_rate ≥ 0.95` per article (deterministic verifier
  validates 100% of copy slots, no LLM call)
- 3 prior-fail articles (27560, 31149, 31430) all yield grammar-valid
  parses on v2.

If gates met → Phase 3 closes; Phase 4 (semantic verifier) is unblocked.
If gates miss → diagnose via verifier_pass_rate vs verbatim_match_rate
vs numeric/recall gaps; do not push v2 forward into Phase 4 until
copy-mode is mechanistically demonstrated.

---

## Open questions (surface to supervisor; do not decide)

1. **Byte offsets vs character offsets.** The plan §6.2 example writes
   `source_span: [start, end]` with no offset semantics. Python string
   slicing is character-based (Unicode codepoints). LLM token offsets
   would be model-specific and useless across decoders. UTF-8 byte
   offsets are what most NLP tooling assumes. Recommended: **character
   (codepoint) offsets** to keep `input_text[start:end]` the natural
   one-line check, but pin the choice in the prompt + verifier in one
   place so it's audit-able. The fact that prior CoD-DSL outputs are
   ASCII-dominant (financial news) means this rarely matters in
   practice — but the n=10 corpus contains at least one article with `€`
   and `£` symbols, so a wrong choice will silently mis-validate.

2. **0-indexed vs 1-indexed.** Python convention is 0-indexed,
   half-open `[start, end)`. Plan §6.2 writes `source_span: [start, end]`
   without specifying. Recommended: **0-indexed, half-open** —
   matches Python slicing directly, no off-by-one in the verifier. Pin
   in the prompt's worked example.

3. **How aggressively to deprecate v1.2?** Plan says "v2 alongside v1.2";
   this scoping plan honors that. But Phase 4 (GPU verifier) is built
   for v2's copy-slot contract. Do we keep v1.2 addressable forever
   ("the historical baseline tree"), or do we set a sunset (e.g., delete
   the v1 grammar 30 days after v2 ships into the matrix integration in
   Phase 5)? Recommend: keep v1 indefinitely as a reference; mark v1
   `deprecated, for reference only` in the grammar header once v2 ships.

4. **LALR conflicts on typed slot terminals.** v2's Lark grammar tries
   to constrain individual slot-name terminals (`SLOT_LINE_NAME`,
   `SLOT_LINE_RAW`, etc.) so the parser table picks the right typed
   production. LALR is unforgiving of overlapping terminals. If the
   conflicts are intractable, the fallback is v1's single `SLOT_LINE`
   terminal + Python-side ordering validation in `parse_text_v2`. This
   trades grammar-level enforcement for parser-level enforcement — same
   end-to-end strictness, but the constrained-decoder won't enforce slot
   order (only the post-hoc parser will). Decision needed if Story 1
   can't make the typed-terminal approach work.

5. **NUM block — when is "numeric" a block vs a slot value?** Plan §6.2
   makes NUM first-class. But many numerics in financial news are
   ratios, dates, periods (`Q3 2026`, `5yr + 2x2yr_options`) that don't
   fit `raw / value / unit / magnitude` cleanly. Recommended: NUM is for
   numerics the verifier can mechanically check (money, percent, count,
   bps); other numeric-shaped tokens stay as EVT/CLAIM slot values
   (verifier doesn't try to score them). Pin in the prompt.

6. **CLAIM evidence_span — span-only or also `evidence_text` copy slot?**
   v2 sketch above makes CLAIM the one block where the copy zone is
   span-only (no verbatim text repeated). This saves tokens but makes
   the verifier check weaker — the verifier can only validate that the
   span is in-bounds, not that the model is actually referring to that
   span. Alternative: require both `evidence_text` (verbatim) and
   `evidence_span` (pointer), like the other blocks. Trade-off: stricter
   verification vs more output tokens. Recommend evidence_text required
   if Story 7's verifier_pass_rate gate misses on CLAIMs specifically.

---

## Rollback path

v2 is **additive** — every story leaves the v1 path intact:

- Story 1 adds new grammar files; v1 grammar files unchanged.
- Story 2 adds a new prompt; v1 prompts unchanged.
- Story 3 adds a flag (default `v1`); existing invocations behave
  identically.
- Story 4 widens a Union; existing parser entry unchanged.
- Story 5 adds a new module (`verifier.py`); no existing module changes.
- Story 6 adds new metric fields (default `None` for v1 docs); existing
  fields unchanged.
- Story 7 writes to a new results tree (`results/phase3-v2/`); existing
  result trees untouched.

To back out:

1. Stop invoking `--schema v2` (the runner default keeps v1 behavior;
   no operator action needed beyond reverting CI / cron jobs).
2. Delete `docs/benchmarks/cod-2026-05-17/results/phase3-v2/` (no other
   tree references it).
3. Optionally `git revert` the merge commits for stories 1-6 (story 7 is
   results-only — no code to revert).
4. v1.2 path continues to function as it did before Phase 3.

There is no shared-state coupling between v1 and v2 at any layer
(grammar, prompt, runner, parser, scorer, results tree). The rollback
cost is one-PR-revert per story or zero ("just stop using the flag").

---

## Phase ordering rationale

```
Phase 2 (v1.2 terminator, sweep complete — gv% gate not cleared)
  │
  ▼
Phase 3 (THIS) — v2 schema + deterministic verifier
  ├─ Story 1: v2 grammar files (GBNF + Lark)
  ├─ Story 2: v2 prompt
  ├─ Story 3: runner --schema v2 flag
  ├─ Story 4: AST extension (source_span as [int, int])
  ├─ Story 5: deterministic verifier skeleton (plan §7.1)
  ├─ Story 6: scoring extension (verbatim_match_rate, span_validity_rate)
  └─ Story 7: smoke + n=10 acceptance run
  │
  ▼
Phase 4 (GPU semantic verifier — plan §7.2; unblocked iff Phase 3 gates met)
```

Phase 3 is ~1 week per the parent plan §11. Each story is a single PR
with explicit `git add` paths; stories 1-3 are pure scaffolding and can
land in parallel, stories 4-6 are sequential (4 unblocks 5 unblocks 6),
story 7 is sequential after 1-6.
