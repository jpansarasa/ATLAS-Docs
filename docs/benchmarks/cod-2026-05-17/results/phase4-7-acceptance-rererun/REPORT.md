# Phase 4.7 acceptance run (n=50) — REPORT

_Generated: 2026-05-25T14:08:48Z_

**Parent plan:** `docs/plans/atlas-dsl-poc-phase4-gpu-semantic-verifier.md` §5 Phase 4.7 (acceptance run) + `docs/plans/atlas-dsl-poc-plan.md` §7.3 (gate criteria).

## 1. Run summary

- Sampled article ids: **50** (seed=4757 from `corpus.full72.jsonl`)
- Production-pipeline enriched docs: **47** (3 parse-failed at the v15 DSL grammar level)
- Foundry-labeling label sets: **47**
- Articles with at least one CLAIM: **24**
- Per-CLAIM verdicts compared: **137** (agree=14, disagree=123, foundry-missing=0)

## 2. Acceptance gate (§7.3, ≥85% agreement)

**Result:** **FAIL** — agreement = 0.1022 (10.22%); gate = ≥85%

Gate FAILED. Per plan §5 Phase 4.7, remediation = prompt iteration on the production claim-verifier (the `BuildPrompt` template). The top disagreement classes are listed in §5 below.

## 3. Per-article fact yield (validated NUMs + catalog-grounded ENTs)

- Mean: **17.57**
- p50: **11.0**
- p95: **54.0**
- min / max: **0 / 114**
- Articles measured: **47**

**Caveat.** Per `run_phase4_7_production_pipeline.compute_fact_yield`, this is a deterministic proxy: validated NUMs = `pass` verdicts from `verifier.verify_document`; grounded ENTs = ENTs with `verbatim_name` (catalog grounding via SecMaster gRPC is out of scope for the §7.3 gate per plan §5 Phase 4.7 — see runner docstring).

Histogram (fact-yield total → article count):

| fact_yield | articles |
|---:|---:|
| 0 | 6 |
| 2 | 3 |
| 3 | 1 |
| 4 | 1 |
| 5 | 2 |
| 6 | 5 |
| 8 | 2 |
| 11 | 4 |
| 12 | 2 |
| 13 | 2 |
| 15 | 2 |
| 17 | 1 |
| 18 | 1 |
| 20 | 2 |
| 23 | 2 |
| 24 | 2 |
| 28 | 1 |
| 30 | 1 |
| 35 | 1 |
| 42 | 1 |
| 46 | 1 |
| 47 | 1 |
| 57 | 1 |
| 65 | 1 |
| 114 | 1 |

Aggregated: validated NUMs total = **480**, grounded ENTs total = **346**.

## 4. Per-CLAIM confusion matrix (production rows × Foundry cols)

| production \ foundry | full | partial | none |
|---|---|---|---|
| **full** | 14 | 97 | 26 |
| **partial** | 0 | 0 | 0 |
| **none** | 0 | 0 | 0 |

Agreement = sum of diagonal; disagreement = sum of off-diagonal.

## 5. Disagreement classes (frequency-sorted)

- **production=full / foundry=partial:** 97
- **production=full / foundry=none:** 26

## 6. Parse failures

- Articles missing from production output: **3** (32859, 35352, 36050)
- Articles missing from foundry output: **3** (32859, 35352, 36050)
- CLAIMs production-verified but no Foundry label: **0**

## 7. Cost breakdown

- Production arm (vllm-server, `Qwen/Qwen2.5-32B-Instruct-AWQ`): **$0.0000** (production runtime = $0 per plan §6).
- Production-arm total wall-time across n=50: **0 ms** (0.0 s for 137 CLAIM calls).
- Foundry (Claude 4.7) labeling arm: **$0.3222** (cap = $10.00; headroom = $9.6778).

## 8. Methodology notes

- The production pipeline runner (`run_phase4_7_production_pipeline.py`) mirrors the C# `ClaimVerifier.BuildPrompt` / `TryParseVerdict` byte-for-byte. Same prompt template, same verdict-parse rules.
- The Foundry runner (`run_phase4_7_foundry_labels.py`) uses the same prompt against Azure Foundry Claude 4.7 via the same Anthropic-compatible HTTP shape `run_phase1_4_foundry.py` uses.
- vllm-server serves the bare `Qwen/Qwen2.5-32B-Instruct-AWQ` (LoRA dropped 2026-05-17; --served-model-name alias dropped 2026-05-25). Both this Python harness and the C# `ClaimVerifier.ProductionModel` literal pin the same id; the acceptance measurement reflects what production does — same weights, same prompt, same parser.
- CLAIM block ids are synthesised `clm_NNNN` matching the C# orchestrator's prefix (`SemanticVerifierService.cs:42`), so the cross-arm join is by id.
- Entity resolution + sector tagging are NOT exercised in this acceptance run. The §7.3 gate is the claim verifier (≥85% agreement); the §7.3 fact-yield criterion is reported via a deterministic proxy (validated NUMs + non-empty ENTs) rather than the full SecMaster cascade, which would require standing up the SecMaster gRPC + OpenFIGI + Gemini stack out-of-process. See plan §5 Phase 4.7 — "Files (new — results only): REPORT.md, spot_check.jsonl, enriched_docs/*.json".

## 9. Artefact paths

- Sample ids: `results/phase4-7-acceptance/sample_ids.txt`
- Production enriched docs: `results/phase4-7-acceptance-rererun/v2_pipeline/*.json` (47 files)
- Foundry labels: `results/phase4-7-acceptance/foundry_labels/*.json` (47 files)
- Per-CLAIM spot check (long form): `results/phase4-7-acceptance-rererun/spot_check.jsonl` (137 rows)
- This report: `results/phase4-7-acceptance-rererun/REPORT.md`

## 10. Three-data-point gate-progression comparison

| Run | PR | agree / total | agreement % | Δ vs prev |
|---|---|---:|---:|---:|
| Original | #449 | 48 / 137 | **35.04%** | — |
| Post-bug-fix (PR #450 prompt/parser) | #451 | 27 / 137 | **19.71%** | −15.33 pp |
| Post-similarity-rework (PR #452) | this | 14 / 137 | **10.22%** | −9.49 pp |

Agreement got worse twice in a row. Neither the prompt/parser fix nor the
structured-similarity rework moved the gate in the right direction. The
explanation is **a third systematic issue specific to the acceptance
harness**, not to the production C# code path.

## 11. Third systematic issue — harness evidence_span convention

### 11.1 What the run did

PR #452 added a **self-referential short-circuit** in
`ClaimVerifier.VerifyAsync` (`ClaimVerifier.cs:224`): when
`objectText.Trim() == evidenceSpan.Trim()` (the "I believe X / evidence:
X" tautology), the verifier skips the LLM call and grounds
`ClaimSupport.Full` directly with the `[claim-verifier] self-referential`
rationale marker. In production C# the trigger is rare — `evidenceSpan`
is `extraction.TextQuote` (the verbatim article quote the chunked
extractor found) while `objectText` is `extraction.Description + " " +
extraction.Value`, so byte equality is the exceptional case.

In **this harness**, however, `extract_claim_inputs` derives
`evidence_span` from `object_text` directly
(`run_phase4_7_production_pipeline.py:341` — "v15 `object` typically
carries a verbatim article quote; fall back to subject+object
concatenation"). When `object_text` is non-empty, `evidence_span =
object_text` byte-for-byte. **All 137 / 137 CLAIM blocks (100%) trip
the short-circuit** and ground `Full`. Total vllm round-trip wall-time
across the run: **0 ms**. No LLM verdict was ever solicited.

Confusion matrix collapses to a single non-zero row:

| production \ foundry | full | partial | none |
|---|---|---|---|
| **full** | 14 | 97 | 26 |
| **partial** | 0 | 0 | 0 |
| **none** | 0 | 0 | 0 |

Per-CLAIM agreement is therefore exactly the Foundry-`full` rate over
the n=137 pool: **14 / 137 = 10.22%**.

### 11.2 Why this is the harness, not the C# code

The C# unit tests for PR #452
(`SentinelCollector/tests/SentinelCollector.UnitTests/Semantic/ClaimVerifierTests.cs`)
cover the `IsSelfReferential` short-circuit explicitly: the test
expectations require non-tautological `evidenceSpan` + `objectText` to
take the LLM path. The production `SemanticVerifierService.TryVerifyClaimAsync`
(`SemanticVerifierService.cs:319-325`) supplies them from two distinct
extraction fields, so byte equality is rare.

The harness's `extract_claim_inputs` is a Phase 4.7-acceptance-specific
shim documented in PR #451's REPORT.md §10.5 as a known mismatch with
production: "If production C# Phase 5 passes a richer span (e.g. the
surrounding NUM/EVT context), the rerun's evidence_span shape would not
match production. Worth cross-checking what
`SemanticVerifierService.TryVerifyClaimAsync` actually feeds
`IClaimVerifier.VerifyAsync` for `evidenceSpan`." PR #452 turned that
known mismatch into a hard short-circuit; the harness now exercises a
code path that is rare in production.

### 11.3 Sample of self-referential pairs (from spot_check.jsonl)

The pattern is uniform across the n=50 sample. For article 27772 (the
Jenoptik AG earnings article, 37 CLAIMs):

| claim_block_id | object_text | evidence_span |
|---|---|---|
| clm_0000 | `EPS` | `EPS` |
| clm_0001 | `revenue` | `revenue` |
| clm_0002 | `stock` | `stock` |
| clm_0003 | `margin` | `margin` |

These are CLAIM `object:` slots emitted as bare nouns by the v15 DSL —
exactly what the harness then pins as both `object_text` and
`evidence_span`. The short-circuit (correctly, per its contract) treats
these as trivially-Full.

### 11.4 Hard rule — STOP and report

Per the supervisor brief for this PR:

> If the rerun reveals a third systematic issue (not bug-floor, not
> wrong-question), STOP and report — don't iterate on prompt or parser
> without supervisor direction.

The 35.04% baseline (#449) was a parser-artefact false floor (bug-floor).
The 19.71% rerun (#451) surfaced the real wrong-question disagreement
between Qwen-strict and Foundry-charitable. This 10.22% rerun (#452 +
harness) surfaces a third issue: **the harness's `evidence_span`
convention is incompatible with PR #452's self-referential short-circuit
in production-faithful semantics**. The Python harness now exercises a
code path that fires on <1% of production calls but on 100% of harness
calls.

No iteration applied. Three candidate next steps for supervisor
consideration only (none touched in this PR):

- **Plumb a real `evidenceSpan` into the harness.** The v15 CLAIM block
  itself doesn't carry an evidence span (verified by inspecting
  `Document.claims[i].slots` for n=50 — only `subject` / `predicate` /
  `object` / `polarity`). Production extracts `TextQuote` from a
  separate v2 extraction pipeline that the harness does not run.
  Options: (a) Re-run the chunked extractor against the n=50 sample to
  materialise v2 `TextQuote` values; (b) Use the raw article text as a
  whole-document `evidence_span` (would defeat the spirit of the
  short-circuit but would put real evidence in front of the model).
- **Skip the short-circuit in harness mode.** Add a CLI flag (e.g.
  `--no-self-referential-shortcut`) so the acceptance harness exercises
  the LLM path even when input shape would normally trip the circuit.
  Would re-establish parity with PR #451's measurement at the cost of
  diverging from C# behaviour on the bypass step.
- **Acknowledge the harness/production semantic gap explicitly.** The
  acceptance gate as currently designed cannot exercise PR #452's
  prompt rework end-to-end because the harness inputs always trip the
  short-circuit. The §7.3 gate may need to be re-cut against a
  different sample or a different upstream extraction.

Production C# behaviour is unaffected — PR #452's short-circuit fires
correctly on byte-equality, the production `evidenceSpan` shape rarely
hits that case, and the structured-similarity prompt change cannot be
measured against the n=50 acceptance pool until the harness's
`evidence_span` derivation is brought in line with production
semantics.

### 11.5 Foundry cost

Foundry labels were **not re-run** for this rerun — the existing
PR #449 labels (`results/phase4-7-acceptance/foundry_labels/*.json`)
are deterministic with temperature=0 against fixed prompts and were
reused unchanged. Foundry cost for this rerun: **$0.0000**.
