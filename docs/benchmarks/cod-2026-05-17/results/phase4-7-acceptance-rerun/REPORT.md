# Phase 4.7 acceptance run (n=50) — REPORT

_Generated: 2026-05-25T13:28:42Z_

**Parent plan:** `docs/plans/atlas-dsl-poc-phase4-gpu-semantic-verifier.md` §5 Phase 4.7 (acceptance run) + `docs/plans/atlas-dsl-poc-plan.md` §7.3 (gate criteria).

## 1. Run summary

- Sampled article ids: **50** (seed=4757 from `corpus.full72.jsonl`)
- Production-pipeline enriched docs: **47** (3 parse-failed at the v15 DSL grammar level)
- Foundry-labeling label sets: **47**
- Articles with at least one CLAIM: **24**
- Per-CLAIM verdicts compared: **137** (agree=27, disagree=110, foundry-missing=0)

## 2. Acceptance gate (§7.3, ≥85% agreement)

**Result:** **FAIL** — agreement = 0.1971 (19.71%); gate = ≥85%

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
| **full** | 1 | 5 | 0 |
| **partial** | 0 | 0 | 0 |
| **none** | 13 | 92 | 26 |

Agreement = sum of diagonal; disagreement = sum of off-diagonal.

## 5. Disagreement classes (frequency-sorted)

- **production=none / foundry=partial:** 92
- **production=none / foundry=full:** 13
- **production=full / foundry=partial:** 5

## 6. Parse failures

- Articles missing from production output: **3** (32859, 35352, 36050)
- Articles missing from foundry output: **3** (32859, 35352, 36050)
- CLAIMs production-verified but no Foundry label: **0**

## 7. Cost breakdown

- Production arm (vllm-server, `Qwen/Qwen2.5-32B-Instruct-AWQ`): **$0.0000** (production runtime = $0 per plan §6).
- Production-arm total wall-time across n=50: **18783 ms** (18.8 s for 137 CLAIM calls).
- Foundry (Claude 4.7) labeling arm: **$0.3222** (cap = $10.00; headroom = $9.6778).

## 8. Methodology notes

- The production pipeline runner (`run_phase4_7_production_pipeline.py`) mirrors the C# `ClaimVerifier.BuildPrompt` / `TryParseVerdict` byte-for-byte. Same prompt template, same verdict-parse rules.
- The Foundry runner (`run_phase4_7_foundry_labels.py`) uses the same prompt against Azure Foundry Claude 4.7 via the same Anthropic-compatible HTTP shape `run_phase1_4_foundry.py` uses.
- vllm-server serves the bare `Qwen/Qwen2.5-32B-Instruct-AWQ` (LoRA dropped 2026-05-17; --served-model-name alias dropped 2026-05-25). Both this Python harness and the C# `ClaimVerifier.ProductionModel` literal pin the same id; the acceptance measurement reflects what production does — same weights, same prompt, same parser.
- CLAIM block ids are synthesised `clm_NNNN` matching the C# orchestrator's prefix (`SemanticVerifierService.cs:42`), so the cross-arm join is by id.
- Entity resolution + sector tagging are NOT exercised in this acceptance run. The §7.3 gate is the claim verifier (≥85% agreement); the §7.3 fact-yield criterion is reported via a deterministic proxy (validated NUMs + non-empty ENTs) rather than the full SecMaster cascade, which would require standing up the SecMaster gRPC + OpenFIGI + Gemini stack out-of-process. See plan §5 Phase 4.7 — "Files (new — results only): REPORT.md, spot_check.jsonl, enriched_docs/*.json".

## 9. Artefact paths

- Sample ids: `results/phase4-7-acceptance/sample_ids.txt`
- Production enriched docs: `results/phase4-7-acceptance-rerun/v2_pipeline/*.json` (47 files)
- Foundry labels: `results/phase4-7-acceptance/foundry_labels/*.json` (47 files; reused unchanged from PR #449 — Foundry labels are deterministic with temperature=0 against fixed prompts)
- Per-CLAIM spot check (long form): `results/phase4-7-acceptance-rerun/spot_check.jsonl` (137 rows)
- This report: `results/phase4-7-acceptance-rerun/REPORT.md`

## 10. Comparison vs PR #449 baseline (the 35.04% gate-fail run)

PR #449 baseline (pre-ClaimVerifier-fix): **48 agree / 89 disagree = 35.04%**.
PR #451 rerun (post-PR-#450 ClaimVerifier fix): **27 agree / 110 disagree = 19.71%**.

**Delta: -15.33 percentage points.** Agreement got WORSE, not better.

### 10.1 What changed in the production arm

PR #450 landed three production fixes that this rerun exercises:

1. **vllm-server `--served-model-name sentinel-cove-v6.2` alias dropped** —
   served model is now the canonical `Qwen/Qwen2.5-32B-Instruct-AWQ`.
2. **`ClaimVerifier.BuildPrompt` appends terse-verdict instruction** —
   `Respond with exactly one word: yes, no, or partial. No preamble, no explanation.`
3. **`ClaimVerifier.TryParseVerdict` walks forward over word boundaries** —
   accepts the verdict word even when preamble precedes it.

The Python harness in this PR mirrors all three changes byte-for-byte
(`run_phase4_7_production_pipeline.py`):
- `build_prompt` appends the same terse-verdict suffix.
- `try_parse_verdict` walks forward over word boundaries with the same
  contract as `MatchesVerdictWordAt`.
- Default confidence rebased to match C# `DefaultVerdictConfidence`
  (Full=0.7, None=0.7, Partial=0.5 — was 0.85/0.85/0.6 pre-fix).

Verification: 137 / 137 CLAIM calls now successfully parse (vs 48 / 137
in PR #449). The parser/prompt fixes work. The malformed-response failure
mode is gone.

### 10.2 Per-CLAIM transition matrix (baseline production → rerun production)

| baseline | rerun | count |
|---|---|---:|
| none | none | 89 |
| partial | none | 24 |
| full | none | 18 |
| full | full | 6 |

All 24 prior `partial` verdicts collapsed to `none`. 18 of 24 prior
`full` verdicts also collapsed to `none`. Production is now `none` on
131 / 137 claims (95.6%).

### 10.3 Of the 89 prior disagreements, how many now agree?

**4.** (The other 85 still disagree, just in a different cell.)

### 10.4 New top disagreement classes (post-fix)

- **production=none / foundry=partial : 92** (was 61 — up 31)
- **production=none / foundry=full    : 13** (was  6 — up  7)
- **production=full / foundry=partial :  5** (was 16 — down 11)

### 10.5 What this means (interpretation — NOT a remediation proposal)

The 35.04% baseline was an accident of the broken parser: ~89 of 137
CLAIM verdicts were `[claim-verifier] malformed-response` → fail-soft to
`none`, and 22 of those 89 happened to match Foundry's `none` for
unrelated reasons. That false-floor inflated the apparent agreement.

With the parser fixed, the model now actually emits a verdict word.
Qwen2.5-32B-Instruct-AWQ's actual behavior on the v15
extraction shape is: **emit `no` 115 / 137 = 84% of the time**. Foundry
Claude 4.7 emits `partial` 99 / 137 = 72% of the time. The two arms now
disagree on a real semantic question, not a parsing artifact.

Reading the spot check (`spot_check.jsonl`), Foundry's `partial`
rationales repeatedly note the same thing: the `evidence_span` passed to
the verifier is the v15 CLAIM `object:` slot value, which for many
articles is a single noun like `EPS`, `revenue`, `stock`, `margin` —
genuinely too thin to fully support the surrounding claim text. Foundry
treats thin-but-related evidence as `partial`; Qwen treats it as `no`.

**Per the hard rule on this run ("do NOT iterate on the prompt or parser
without supervisor direction"), this report stops here.** Possible
remediation directions for supervisor consideration only:

- The v15 emitter encodes evidence in `object:` slots. The
  `extract_claim_inputs` shim in `run_phase4_7_production_pipeline.py`
  passes `object_text` as `evidence_span`. If production C# Phase 5
  passes a richer span (e.g. the surrounding NUM/EVT context), the
  rerun's evidence_span shape would not match production. Worth
  cross-checking what `SemanticVerifierService.TryVerifyClaimAsync`
  actually feeds `IClaimVerifier.VerifyAsync` for `evidenceSpan`.
- The terse prompt may be over-coercing. The C# tests assert
  `"normally yes, but ..." → Full` as INTENTIONAL — but the parallel
  intent "evidence is thin → partial" is not encoded; the model is
  defaulting to `no` instead.
- Foundry's `partial` baseline may itself be too generous given the
  task framing.

None of those are touched in this PR.
