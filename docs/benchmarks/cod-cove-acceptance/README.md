# CoD → CoVe handoff acceptance harness

A differential test that validates the production GPU CoVe claim-verifier
against CoD outputs the deterministic benchmark verifier already confirmed are
verbatim-present in source.

## The invariant being tested

The CoD benchmark harness (`../cod-2026-05-17`) scores what the CPU extracts,
verbatim-verified against source. This harness takes those **validated** CoD
CLAIM outputs and feeds them to the GPU CoVe verifier. A *coherent*
verbatim-validated fact MUST verify as `consistent` (~100%). A miss on such a
fact is a **bug** (verifier or wiring), never legitimate model judgment, because
the fact is provably present in the same source the verifier reads.

Converse bound: facts CoD failed to extract are ~impossible for the verifier to
recover — so a healthy verifier here means any production yield gap is
attributable to **wiring**, not the verifier.

## Faithfulness to production

The verifier replica mirrors `SentinelCollector/src/Semantic/ClaimVerifier.cs`:

- **prompt** — `build_prompt()` is byte-identical to `ClaimVerifier.BuildPrompt`
  (Phase 4.7 iter-2 structured-finding shape).
- **short-circuit** — `is_self_referential()` mirrors `IsSelfReferential`
  (object == evidence, ordinal trim → `Full`).
- **verdict parse** — `try_parse_verdict()` mirrors `TryParseVerdict`: the 4-word
  grammar (`consistent`/`related`/`unrelated`/`insufficient_evidence`) collapses
  to the 3-valued `ClaimSupport` (`consistent`→Full, `related`→Partial,
  `unrelated`/`insufficient_evidence`→None), with the same word-boundary +
  preamble-tolerance contract.
- **model / request** — `Qwen/Qwen2.5-32B-Instruct-AWQ` via
  `POST /v1/completions`, `temperature=0.0`, `max_tokens=64`.

Same prompt + same model = faithful to verifier behavior. The C# wiring fix
changes only *what* reaches the verifier, not the verifier itself.

## "Validated" claim definition (the load-bearing gate)

A CoD CLAIM qualifies when ALL hold:

1. The result doc parses as v2/v2.1.
2. `dsl.verifier.verify_document` marks the CLAIM block `pass` (in-bounds
   evidence span when present + every subject/counterparty/speaker ref resolves).
3. The CLAIM carries `subject` + `predicate` + `object` slots.
4. The `object` text (quotes stripped) is a **verbatim substring** of the source
   article — "provably present in source".

`subject` is resolved to its verbatim entity name (what production passes).
`evidence_span` is widened from the verbatim object to the enclosing source
sentence(s) so the LLM verifier is actually exercised (object==evidence would
trip the self-referential short-circuit on every cell).

### Triple coherence

A verbatim object can still be an **incoherent** finding: the CPU extractor
sometimes slices a sentence-fragment into the `predicate` slot and attaches it to
a subject the source never pairs with that predicate (canonical case: the n=72
"layoffs roundup" article `29149`, one fabricated predicate fanned across 23
subjects). For such triples the verifier *correctly* answers `unrelated`/
`related` — that is **extraction quality, not a verifier miss**. The harness
splits the headline pass rate into **coherent** (verb-phrase predicate ≤4 words)
vs **degenerate** subsets and bases its verdict on the coherent subset.

## Run

```bash
# from this directory, with a venv carrying lark + requests
python cod_cove_harness.py                  # full run vs vllm-server
python cod_cove_harness.py --limit 40       # cap claims (pace shared vllm)
python cod_cove_harness.py --dry-run        # reconstruct claims, no LLM calls
python cod_cove_harness.py --endpoint http://localhost:8000
python test_cod_cove_harness.py             # offline self-tests (no network)
```

Artifacts (default `/tmp/sentinel-remediation/cod-cove-harness/`): `per_claim.jsonl`,
`misses.jsonl`, `summary.json`.

The harness reads `../cod-2026-05-17` for the DSL parser + deterministic verifier
+ corpora; create a venv there (or anywhere) with `lark` and `requests`.

## Expand it

Add a `(result-glob, corpus-jsonl)` tuple to `CORPORA` in `cod_cove_harness.py`.
The loader, verifier replica, and scorer are corpus-agnostic. New result dirs
only need v2/v2.1 DSL result JSONs in the `{article_id}.json` shape and a matching
corpus JSONL keyed by `id` with a `raw_text` field. Examples already wired:
`results/phase3-v8-baseline-n72-PhaseA/...` (n=72) and
`results/path-c-gpu-n9-arm-b/...` (Arm-B GPU). `corpus.expanded150.jsonl` + its
result dir can be added the same way.
