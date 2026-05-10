# Epic 4 — SentinelCollector Rework

## Goal

Reframe Sentinel as disconnect detector and unofficial-channel signal
aggregator. Three observation types extracted (numeric instrument-tagged,
numeric macro-series-tagged, qualitative macro-tagged). Sentinel exits
math entirely. CoVe applies to qualitative (D11). LLM extraction
reliability brought to a measurable acceptance bar (D13, D15, D16).

## Branch

`epic/4-sentinel-rework` off `main`. Feature 4.6 starts in Phase 2 (in
parallel with Epic 2 and Epic 3); Features 4.1–4.5 start in Phase 3 once
Epic 3's substrate is live.

## Phase + dependencies

- Phase: **2** (Feature 4.6 only) and **3** (Features 4.1–4.5)
- Blocked by:
  - Feature 4.6: D15 ratification (`docs/llm/ACCEPTANCE_CRITERIA.md`) +
    Epic 1 signal-identity catalog.
  - Features 4.1–4.5: Epic 3 substrate live + Epic 2 emitting cells +
    Epic 5 Feature 5.5 emitting threshold-cross events.
- Blocks: Epic 6's qualitative-aware panels.

## Canonical AC

`/home/james/ATLAS/docs/atlas-matrix-mvp-plan.md` Epic 4 (lines 417–531).

## Features (with execution notes)

### 4.1 — Sentinel exits math
- **4.1.1** Math-path audit and removal. Recon already identified
  the lone path: `SentinelCollector/src/Services/CorroborationScanner.cs`
  (`MultiSignalGate.ComputeScore`). Story removes it; any consumer
  consults TE instead. Post-audit Sentinel persists structured
  observations + trust attribute only.

### 4.2 — Numeric instrument-tagged extraction (verify existing)
- **4.2.1** Verify and document. Existing extraction path produces
  this type today (`ExtractedObservation` schema). Sector tag
  attachment routes through Epic 1 SecMaster lookup. Gaps with the
  contract filed as defects.

### 4.3 — Numeric macro-series-tagged extraction
- **4.3.1** Routing to `macro_observations`. When extraction matches
  a known macro series, the observation writes into Epic 3's substrate
  (instead of `extracted_observations`). Signal identity from SecMaster
  (Feature 1.7) tags it.

### 4.4 — Qualitative macro-tagged extraction (NEW)
- **4.4.1** Qualitative extraction prompt and schema. Separate prompt
  + structured-output schema from numeric. Outputs: sentiment polarity,
  sector affinity, lead-lag estimate, regime hint. Validated against
  schema; malformed → reject + diagnostic log. LoRA adapter applied
  (informed by Feature 4.6 outputs).
- **4.4.2** Qualitative observation persistence. Writes into
  `macro_observations`. Trust attribute from extraction job, clamped
  to defined range. Sector tag attached when output specifies one of
  the 11 sectors; absent for sector-agnostic outputs.
- **4.4.3** CoVe applied to qualitative (D11). Existing
  `ChainOfVerification` + `ChainOfVerificationToolAugmented` reused.
  Verification step documented; failures reduce trust attribute rather
  than reject. CoVe overhead and latency effect measured.

### 4.5 — Validation trigger subscription (D12)
- **4.5.1** Event subscription and narrative search. Recon found
  `ValidationEventConsumerWorker` already subscribing to
  `ThresholdCrossedEvent` over gRPC streaming. Story extends it: on
  event (sector score crossing real-valued threshold), query
  SearXNG / RSS / Cloudflare for narratives on the named sector and
  time window. Resulting observations write through Feature 4.4 path
  with trigger source logged in provenance.

### 4.6 — LLM extraction reliability track (Phase 2 — D13, D15, D16)
- **4.6.1** Import inherited acceptance criteria. Read
  `docs/llm/ACCEPTANCE_CRITERIA.md` (PS2 ratification). Pin thresholds
  in this epic; commit a copy of the values into the LoRA training
  artifact metadata.
- **4.6.2** Evaluation harness against labelled dataset. Substrate
  decision required first (see pre-kickoff: v6.2-cove with 70,895
  records vs. v4-diverse vs. subset). Harness runs
  qualitative-extraction prompt + current LoRA against the dataset.
  Output: scorecard with per-output-type metrics + delta from
  acceptance criteria. Reproducible; baseline scorecard captured
  before iteration begins.
- **4.6.3** Iteration cycles. Each: hypothesis → modification (LoRA
  adjustment, prompt change, data augmentation, CoVe tuning) →
  re-evaluate → document delta. 4-week timebox + $500 Azure cap (D16).
  Spend tracked at weekly checkpoints. All artefacts (configs,
  prompts, training runs, scorecards) committed.
- **4.6.4** Success / failure decision. At budget exhaustion, scorecard
  vs. acceptance criteria. Success → LoRA + prompt deployed → Feature
  4.4 enables full-trust qualitative writes. Failure → architect call
  on extend / ship constrained / defer (high trust-gate threshold).

## Files to touch

- `SentinelCollector/src/Services/CorroborationScanner.cs` **(remove
  ComputeScore math)**
- `SentinelCollector/src/Services/ExtractionService.cs` (qualitative
  prompt routing)
- `SentinelCollector/src/Extraction/ChainOfVerification.cs`,
  `ChainOfVerificationToolAugmented.cs` (extend for qualitative
  schema; do not rewrite)
- `SentinelCollector/src/Extraction/IExtractionPromptProvider.cs` (new
  qualitative prompt)
- New qualitative output schema entity + repository.
- `SentinelCollector/src/Workers/ValidationEventConsumerWorker.cs`
  (extend with narrative-search dispatch)
- `SentinelCollector/src/Data/Repositories/ObservationRepository.cs`
  (route to `macro_observations` for macro types)
- `SentinelCollector/scripts/train_qlora_unsloth.py` (Feature 4.6.3
  iteration toolkit; existing)
- `LlmBenchmark/` (eval harness extension for symbol-match,
  null-precision, qualitative-output metrics)
- New: `LlmBenchmark/QualitativeAccuracyTests.cs` paralleling
  `ExtractionAccuracyTests.cs` thresholds.

## Subagent dispatch notes

- Feature 4.6 is its own ralph-loop-shaped track. Use
  `superpowers:executing-plans` for the iteration cycles; per-cycle
  scorecard committed.
- Feature 4.1 (math removal): one `general-purpose` agent. Small,
  mechanical.
- Feature 4.4 (new qualitative path): one `general-purpose` agent
  with TDD discipline (`superpowers:test-driven-development`); CoVe
  is reused, not rewritten.
- Feature 4.5 (extend ValidationEventConsumerWorker): one
  `general-purpose` agent.
- PR review: `pr-review-toolkit:review-pr` per story PR. The
  qualitative prompt + schema PR additionally gets
  `pr-review-toolkit:type-design-analyzer` for the new schema.

## Verification (epic-exit AC)

- `CorroborationScanner.ComputeScore` deleted; no math path remains in
  Sentinel (compile-driven check + grep).
- Qualitative observation lands in `macro_observations` with trust
  attribute + sector tag (when applicable). CoVe trace visible in
  OTEL.
- TE emits a sector-threshold-cross event; Sentinel narrative search
  fires and writes a follow-up observation with trigger provenance.
- Feature 4.6 scorecard run vs. `ACCEPTANCE_CRITERIA.md` shows
  pass-or-architect-call; record committed.
- Build + deploy clean; Loki silent post-deploy in 10-min window.
