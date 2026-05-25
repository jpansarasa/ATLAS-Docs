# DSL PoC — Phase 4.7s — SFT fine-tuning the GPU CoVe verifier

**Status:** scoping. No code, no deploy. Single source of truth for the
Phase 4.7 SFT follow-up until split into stories and merged.
**User approval:** 2026-05-25, after Phase 4.7 iter-5 plateau at **71.53%
agreement** and the 3-way model ablation (PR #458, `94330a42`) confirmed
the 14pp gap is structural to the ~30B AWQ open-model class on the
iter-5 prompt + DSL CLAIM-shape (Granite-30B -15.33pp, Mistral-24B
-52.55pp vs Qwen baseline).
**Parent plan:** `docs/plans/atlas-dsl-poc-phase4-gpu-semantic-verifier.md`
(Phase 4.7 acceptance §5; gates §11).
**Sibling memories:** `[[project_lora_v6_iteration]]` (prior infra),
`[[project_sentinel_llm_strength_layering]]` (the argument for SFT-on-GPU),
`[[atlas-inference-topology]]` (CPU=extraction, GPU=verification),
`[[project_vllm_sentinel]]` (AWQ base + LoRA adapter serving).

---

## 1. Executive summary

Prior LoRA failed because **v6 combined the CoD (extraction) and CoVE
(verification) tasks in one adapter** — the model was trained to answer,
not to be correct (audit `compose.yaml.j2:808-816`: F4.6.3 sector-rotation
5x inflation, fabricated `horizon_days`, MAJOR error 4% base vs 20%
v7-e3). With Phase 3 v15 + word-grounding shipping the CoD task on the
CPU side and the deterministic verifier (`verifier_v2_3_1.py`) handling
the lexical/structural checks, the GPU CoVe verifier (currently bare
`Qwen2.5-32B-Instruct-AWQ` per `compose.yaml.j2:822`) has **one clean
training target**: `(evidence_span, structured-finding-slots) →
verdict ∈ {full, partial, none}` (verdict vocabulary aligned with
`ClaimVerifier.cs` `BuildPrompt` at L173-197 + `ClaimVerification.cs`
`SupportTag`).

**Goal.** Close the 14pp Foundry-agreement gap on the Phase 4.7 acceptance
gate (≥85%) OR document the structural floor on the held-out set with
sufficient signal to retire per-claim Foundry agreement as the sole
gate.

**Training data mix.** ~4000 examples, mostly synthesized at $0 from the
CPU's verified DSL extractions (`verifier_v2_3_1.py`-validated): ~25%
strong-positive (`consistent`), ~37% strong-negative (`unrelated` random
pairings + hard near-miss slot-perturbations), with ~37% (~1500)
ambiguous middle-ground labeled by Foundry Claude 4.7 at **~$3.50**.
This preserves Qwen's conservative-correctness while teaching the
precise positive/negative boundary — instead of mirroring Foundry's
`partial`-skewed distribution.

**Duration estimate.** ~3 weeks: 1 week data scaffolding + cloud
expansion, 1 week training + diagnostic iteration, 1 week serving +
production cutover + observability. 8 phased PRs (4.7s.1 → 4.7s.8),
each ≤500 LOC.

---

## 2. Architectural argument — why SFT is right NOW vs why it was wrong before

**Why v6 LoRA failed (re-frame).** Per `[[project_lora_v6_iteration]]`:
v6 trained 32B QLoRA across `q/k/v/o + MLP` (7 modules, 134M trainable
params) on 342 Opus-labeled examples × 3 epochs. Result: F1 +1.2pp,
precision **-8.2pp**, Value Exact **-13.4pp**. The audit memo at
`compose.yaml.j2:808-821` is blunter — LoRA was producing wrong-context
values and inflating signal categories (5x sector_rotation, fabricated
`horizon_days`, MAJOR error 4% base vs 20% v7-e3).

The single-task lens explains why: v6 simultaneously tried to (a)
span-identify the right substring, (b) synthesize structured slots,
AND (c) be faithful to source. Loss signal couldn't disentangle
"structurally valid" from "actually correct" — gradients flowed to the
former (easier to satisfy).

**What's different now.** Per
`[[project_sentinel_llm_strength_layering]]` (activation lock-in
2026-05-21) + `[[atlas-inference-topology]]`:

| Layer | Model | Task | State |
|---|---|---|---|
| CoD (CPU) | qwen3:30b-a3b on `llama-server`:11437 | DSL emission (v15 word + chunked) | **DONE** — T=0 pooled 0.8261 (STATE.md L37) |
| Deterministic verifier | Python | lexical/structural pass/soft/hard | **DONE** — `verifier_v2_3_1.py` |
| GPU CoVe verifier | `Qwen2.5-32B-AWQ` on `vllm-server`:8000 | classify `(evidence_span, slots) → verdict` | **base only** — iter-5 plateau 71.53% |

CoVe verifier input is now **pre-extracted slots + evidence_span**
(`run_phase4_7_production_pipeline.py:389-462` + `ClaimVerifier.cs:173-197`)
— never the full article (per `[[feedback-no-full-article-to-gpu-verifier]]`).
The target is **classification over a tiny verdict vocabulary**, not
structured synthesis. Loss signal can converge. The layering rule
"don't train a 7-30B to do what 32B does natively" inverts here — we
train the 32B GPU model on its own task.

---

## 3. Training data design

The CPU layer produces DSL extractions validated by `verifier_v2_3_1.py` (per the T=0 A/B verdict, pooled word-grounding 0.8261 on n=72). Each validated extraction is a known-correct mapping from {subject, predicate, object_text, evidence_span} to a `consistent` verdict. We leverage this to generate strong-signal training cases at $0 — and use Foundry only for the genuinely ambiguous middle-ground.

### Buckets

| Bucket | Source | Volume | Cost | What it teaches |
|---|---|---:|---:|---|
| `consistent` (positive) | Verified CPU extraction + its actual source span | ~1000 | $0 | Confident match recognition |
| `unrelated` strong (negative) | Verified CPU extraction + RANDOM other-article span | ~1000 | $0 | Topic-mismatch rejection |
| `unrelated`/`insufficient` hard (negative) | Verified CPU extraction + slot-perturbed (value flipped, subject swapped, predicate inverted) + same article evidence | ~500 | $0 | Near-miss rejection — catches BAD extractions, not just unrelated ones |
| `partial`/`related` (ambiguous) | Foundry Claude 4.7 labeled — real corpus ambiguity | ~1500 | ~$3.50 | Calibration on genuinely borderline cases |
| **Total** | — | **~4000** | **~$3.50** | — |

### Distribution rationale

Result: ~25% confident-positive / ~37% confident-negative (~25% strong + ~12% hard near-miss) / ~37% partial. The 37% strong-`none` representation directly trains for decisive rejection — and the hard near-miss subset teaches the LoRA to say `none` even when subject/topic MATCH but the finding doesn't actually hold.

This is FUNDAMENTALLY DIFFERENT from training the LoRA to mirror Foundry's distribution (which would skew toward `partial` since Foundry is charitable). The redesigned mix preserves Qwen's conservative-correctness from the G1 finding while teaching the precise positive/negative boundary the production CoVE needs.

### Synthetic-generation specifics

1. **`consistent` generation**: take 1000 validated CLAIMs from the v15 sweep results (`docs/benchmarks/cod-2026-05-17/results/v15-n72-T0/qwen3_30b-a3b-instruct-2507-q4_K_M/v2_3_spec/`). For each, the `subject`/`predicate`/`object_text` slots are the structured finding; the evidence_span is the article text span where the model copied the object_text from (recoverable via byte-range or word-grounding `source_words` slot).

2. **`unrelated` strong generation**: take 1000 validated CLAIMs from set A; pair each with an evidence_span from a randomly-chosen UNRELATED article (different SOURCE_id, different TIMESTAMP day) from set B. Sanity check: no accidental subject overlap (drop if subject string appears in the unrelated span).

3. **`unrelated`/`insufficient` hard generation**: take 500 validated CLAIMs. Perturb ONE of the structured slots:
   - Value perturbation: NUM-block value sign flip (+5% → −5%, $100M → $1B)
   - Subject perturbation: subject swapped with another ENT-id from the same article
   - Predicate inversion: predicate replaced with a contradictory verb ("increased" → "decreased", "exceeded" → "missed")
   - Same article evidence_span carried over verbatim

   Label: `unrelated` (perturbation broke the claim) or `insufficient_evidence` (perturbation made the structured slot unparseable). Judgment per-perturbation-type, documented in the generator.

4. **`partial` Foundry generation**: 1500 ambiguous CLAIMs labeled by Foundry Claude 4.7. SAMPLE these from EITHER (a) corpus CLAIMs that the CPU extraction generated with low `verifier_v2_3_1` word-grounding confidence (i.e. the model expressed uncertainty), OR (b) corpus CLAIMs where the structured slot data isn't a verbatim source span (i.e. some natural-language paraphrasing happened). These are the cases where ambiguity is real and Foundry's judgment adds signal beyond what deterministic rules can provide.

### Train/val/test split

- 70% train (~2800)
- 15% val (~600) — for in-training eval + early stopping
- 15% held-out test (~600) — final eval; NOT used in training, NOT contaminated by the iter-1..10 prompt-tuning n=50 sample

Held-out test sample comes from a FRESH n=100 article pull from the full72 corpus that has NEVER been used in Phase 4.7 iter-N runs. Multiple labels per held-out article (avg ~6 CLAIMs each → ~600 test examples).

---

## 4. Training infrastructure

### 4.1 LoRA-on-AWQ base (per `[[project_vllm_sentinel]]`)

- **Base:** `Qwen/Qwen2.5-32B-Instruct-AWQ` (production;
  `compose.yaml.j2:822`).
- **Target modules:** `q_proj, k_proj, v_proj, o_proj` only (attention,
  no MLP). Per `[[project_lora_v6_iteration]]` root cause #2, v6
  over-targeted.
- **Rank:** `r=8, alpha=16` (vs v6's r=16, alpha=32 at
  `train_qlora_unsloth.py:258-259`). Verdict classification needs less
  capacity than v6's structured emission. **§9 Q3.**
- **Epochs:** 1 (v6 overfit at 3). **LR:** 2e-5. **Dropout:** 0.1.
- **Loss masking:** train ONLY on the assistant verdict token (HF
  `DataCollatorForCompletionOnlyLM`). Adapter doesn't regenerate the
  prompt template.

### 4.2 Hardware + training-window posture

- **GPU:** RTX 5090, 32GB VRAM. Unsloth fits 32B QLoRA at 26.9GB
  (per `[[project_vllm_sentinel]]`) — no headroom for vllm-server in
  parallel.
- **Production pause:** training requires unloading vllm-server.
  Estimated wall-time/run: 2-4h at 1 epoch / ~1.5-3k examples.
- **Checkpoint cadence:** save every 50 steps; val-set eval at every save.
- **Cloud alternative:** Azure ML A100-80G spot ≈ $3-5/run, eliminates
  the pause. **§9 Q4.**

### 4.3 Reuse the existing training harness

Extend `SentinelCollector/scripts/train_qlora_unsloth.py`:

- New `--example-format claim-verdict` mode that uses
  `ClaimVerifier.BuildPrompt` template verbatim (replaces v6's
  `format_example` at L241-249).
- New `--loss-mask-prompt` flag (default ON for verdict mode).
- `--target-modules attention-only` shortcut at L289 (existing path).
- `load_acceptance_criteria` fail-fast at L270 extended to validate
  Foundry-label provenance (run_label + ledger cross-check).

---

## 5. Eval framework

### 5.1 Three independent signals (NOT just Foundry agreement)

Per the iter-5 G1 finding (production over-predicts `partial`, Foundry
under-predicts it), Foundry-agreement-as-sole-gate is flawed. The SFT
plan requires **three independent signals**, ALL must clear:

**Signal 1 — Held-out Foundry agreement.** Fresh n=100 + the 15%
Foundry-only test slice. Target: **≥85%** (parent §11.2). Held-out so
SFT can't be optimized against it.

**Signal 2 — Human spot-check agreement.** Same n=100, human verdict
on a **50-CLAIM subset** = ground truth. Target: **≥85%** (parent
§7.3). Foundry is *an* oracle — not ground truth. If Signal 2 ≥
Signal 1 + 5pp, SFT may be truer than Foundry; re-open the gate.

**Signal 3 — Downstream signal quality (monotonicity over time).**
Shadow-run the SFT verifier for **7 days post-deploy** against the live
SentinelCollector. Test: do canonical cases (FOMC + explicit rate
guidance) consistently produce the right verdict at T=0 across re-runs
+ across time? Surfaced via the v2 Pipeline Health dashboard (PR #435).

### 5.2 Failure-mode taxonomy

| Signal 1 | Signal 2 | Interpretation |
|---|---|---|
| Miss | Hit | SFT learned truth; Foundry is the wrong oracle. Document, proceed. |
| Hit | Miss | SFT overfit to Foundry bias. Reduce rank, increase dropout, retry. |
| Miss | Miss | Verdict vocabulary is wrong. 79-disagreement class in iter-5 suggests `partial` is over-broad — consider 5-way (`full / mostly / partial / weak / none`). Escalate as new RFC. |

### 5.3 Generic-capability eval (regression guard)

Tiny **MT-Bench subset** (10 questions) pre/post SFT. Target: ≤5%
overall-score regression. If regression: reduce rank or restrict
modules further.

---

## 6. Phased PR sequence (8 PRs, all ≤500 LOC, independently mergeable)

Per `[[feedback-no-placeholder-prs]]`: each PR carries substantive
implementation; no scaffolding-only PRs. Per `[[feedback-drop-feature-flags]]`:
no default-OFF gates.

### Phase 4.7s.1 — Training data scaffold

**Scope.** `SentinelCollector/scripts/build_cove_sft_dataset.py`:

1. Read Foundry labels from `phase4-7-acceptance/foundry_labels/` +
   scan `phase4-7-iter-*/foundry_labels/` for additions.
2. Read v2_pipeline output for matching `claim_inputs`.
3. Cross-validate via `run_phase4_7_production_pipeline.build_prompt`
   (matches `ClaimVerifier.cs:173` verbatim).
4. Emit §3.3 JSONL split into train/val/test.
5. Auto-generate synthetic-full + synthetic-none from
   `phase3-word-grounding/v15-1-rescored-n72/`.
6. Identify fresh held-out n=100 candidate IDs (NOT in n=50, NOT in
   full72) → `held_out_n100_candidate_ids.txt`.

**Files (new):**
- `SentinelCollector/scripts/build_cove_sft_dataset.py` + tests
- `docs/benchmarks/cod-2026-05-17/results/phase4-7-sft/dataset/v1/`

**Acceptance.** Unit tests: Foundry-label → JSONL round-trip;
synthetic-full deterministic; synthetic-none excludes self-referential
pairs (`ClaimVerifier.cs:163-182`); class-balanced within ±5pp of
source-pool distribution; no train ↔ val ↔ test article-id leakage.

### Phase 4.7s.2 — LoRA training harness extension

**Scope.** Extend `train_qlora_unsloth.py` per §4.3
(claim-verdict mode, loss masking, attention-only shortcut). New
`train_cove_verifier.py` wrapper pins §4.1 hyperparams + writes
per-step eval report. Eval loop runs val-set verdict prediction at
every save; logs to TensorBoard + `eval_step_NNN.json`. No actual
training here — that's 4.7s.4.

**Files (new/modified):**
- `train_qlora_unsloth.py` (extend)
- `train_cove_verifier.py`, `eval_cove_verifier.py` (new) + tests

**Acceptance.** Unit tests: prompt template byte-matches
`ClaimVerifier.cs:173-197`; loss mask covers only assistant tokens;
eval loop reports per-class precision + recall. Live training gated
behind `COVE_SFT_LIVE_TRAIN=1` env.

### Phase 4.7s.3 — Cloud labeling expansion

**Scope.** `run_cove_sft_foundry_expansion.py`:

1. Read held-out n=100 candidate IDs from 4.7s.1.
2. Re-run v15 production pipeline against uncovered full72 articles
   (22) + the new n=100.
3. Submit new CLAIMs to Foundry via `run_phase4_7_foundry_labels.py`
   pattern (cost-capped, ledger-tracked). Hard cap **$10**, per §9 Q1.
4. Write labels into `results/phase4-7-sft/foundry_labels_expansion/`.
5. Re-run `build_cove_sft_dataset.py` to regenerate train/val/test.

**Files (new):** `run_cove_sft_foundry_expansion.py`,
`results/phase4-7-sft/foundry_labels_expansion/`, `REPORT-expansion.md`.

**Acceptance.** Ledger entries match run output; spend ≤ cap;
n_examples ≥ 1,000 OR explicit STATE update if corpus density doesn't
permit.

### Phase 4.7s.4 — First LoRA run + diagnostic report

**Scope.** Wall-time the harness from 4.7s.2 against the dataset from
4.7s.1+4.7s.3. Train one adapter (`sentinel-cove-sft-v1`). Save to
`/opt/ai-inference/models/sentinel-cove-sft-v1/`. Generate diagnostic
REPORT.md covering: training loss curves, val-set agreement at each
checkpoint, eval on the held-out test set (Signal 1 §5.1), MT-Bench
regression check (§5.3).

**Files (new — results only):**
- `docs/benchmarks/cod-2026-05-17/results/phase4-7-sft/training-v1/REPORT.md`
- `docs/benchmarks/cod-2026-05-17/results/phase4-7-sft/training-v1/loss_curves.png`
- `docs/benchmarks/cod-2026-05-17/results/phase4-7-sft/training-v1/checkpoint_eval.jsonl`
- `docs/benchmarks/cod-2026-05-17/results/phase4-7-sft/training-v1/mtbench_regression.md`

**Acceptance.** Loss curves monotone-down with no NaN; val-set
agreement strictly > iter-5 baseline (71.53%) at the best checkpoint;
test-set agreement reported (gate check, not pass/fail here — gate
decision in 4.7s.6). MT-Bench regression ≤5pp.

### Phase 4.7s.5 — LoRA adapter integration with vllm-server

**Scope.** Per `[[project_vllm_sentinel]]`: vLLM args become
`--enable-lora --lora-modules sentinel-cove-sft=/lora --max-lora-rank 8
--generation-config vllm`. Update
`deployment/ansible/playbooks/deploy.yml` + `compose.yaml.j2:811-820`
to re-enable `--enable-lora` (currently disabled per 2026-05-17
LoRA-removal). The per-request `model` field selects base vs adapter at
inference time:

- **CoVe verifier path** (`ClaimVerifier.cs:200`): new
  `ClaimVerifier__VllmModel=sentinel-cove-sft` option (default = base
  HF id; opt-in to adapter).
- **Extraction path** (`Extraction__V2Model`): stays on base —
  extraction does NOT use the SFT adapter (CPU layer per
  `[[atlas-inference-topology]]`, plus the v6 wrong-task history at
  `compose.yaml.j2:808-820`).

Smoke n=100 via the existing production-pipeline runner with
`--vllm-model sentinel-cove-sft`.

**Files (new/modified):**
- `deployment/ansible/playbooks/deploy.yml`, `compose.yaml.j2` (LoRA
  mount + env knob)
- `SentinelCollector/src/Configuration/ClaimVerifierOptions.cs`
- `SentinelCollector/src/Semantic/ClaimVerifier.cs` (read `VllmModel`,
  default to existing `ProductionModel` literal)
- `SentinelCollector/test/Semantic/ClaimVerifierTests.cs`

**Acceptance.** vllm-server serves both base + SFT adapter;
`ClaimVerifier` test with `VllmModel=sentinel-cove-sft` verifies
adapter request routing; compile.sh passes; smoke n=100 emits with
non-zero adapter hits in vllm-server logs.

### Phase 4.7s.6 — Production cutover (no flag, ZFS-snapshot guarded)

**Scope.** Direct cutover. Pre-deploy: ZFS snapshot of `/var/lib/atlas`.
Deploy: `ansible-playbook --tags vllm-server,sentinel-collector`.
Post-deploy: 30-min smoke via
`atlas_semantic_verifier_claim_pass_rate`; if >10pp drop vs baseline,
rollback.

**Gate (user-ratified before merge):**

- Signal 1 (held-out Foundry agreement) ≥85% — REQUIRED.
- Signal 2 (human spot-check, 50 CLAIM) ≥85% — REQUIRED.
- MT-Bench regression ≤5pp — REQUIRED.
- Any miss: recycle to 4.7s.4 (hyperparam) or §5.2 (failure-mode).

**Files (modified):** `STATE.md` (cutover record).

**Acceptance.** ZFS snapshot exists; deploy succeeds; 30-min smoke
green; STATE.md update merged.

### Phase 4.7s.7 — 4-week production observability

**Scope.** Extend `v2-pipeline-health.json` (PR #435, `b4481493`) with
a `Pre-SFT vs Post-SFT comparison` row:

- `atlas_semantic_verifier_claim_pass_rate` over 28d with cutover
  marker.
- Per-verdict-class rate (`sum by (verdict)(rate(...))`) — should show
  iter-5 over-`partial` skew correcting.
- `atlas_semantic_verifier_call_latency_ms` p50/p95 — per
  `[[project_vllm_sentinel]]` LoRA-on-AWQ measured 62 vs 76 tok/s
  (~18% slowdown — acceptable).
- Malformed-response rate (should stay 0; iter-5 baseline 0/137).

New P2 alert: `SemanticVerifierPassRateRegression` — fires if 1h-window
pass-rate drops >10pp vs 7-day-baseline post-cutover.

**Files (modified):** `v2-pipeline-health.json`, `sentinel.rules.yml`.

**Acceptance.** Panels render; `promtool check rules` clean; 4-week
REPORT.md committed documenting Signal 3 status.

### Phase 4.7s.8 — Rollback rehearsal + cleanup

**Scope.** Per `[[feedback-completion-gate]]`:

1. `git revert` 4.7s.5+4.7s.6.
2. `ansible-playbook playbooks/deploy.yml --tags vllm-server,sentinel-collector`.
3. Verify `ClaimVerifier__VllmModel` falls back to base id; vllm-server
   stops serving the adapter.
4. Pre/post-rollback verdict distribution should match pre-SFT iter-5
   baseline.
5. Validate ZFS snapshot is restorable.

Cleanup: archive `phase4-7-sft/dataset/v1/` to
`/opt/ai-inference/training-data/archive/`; prune intermediate
checkpoints; keep v1 adapter + dataset for re-training.

**Acceptance.** Rollback rehearsed in 15-min window; verdict
distribution matches pre-SFT baseline within ±2pp.

---

## 7. Rollback strategy

Per `[[feedback-drop-feature-flags]]`: no default-OFF flag.

- **4.7s.1-4.7s.4, 4.7s.7-4.7s.8:** additive scripts/results/docs.
  `git revert` no-op for runtime.
- **4.7s.5:** `git revert` ansible/compose + redeploy. Restores
  base-model serving in <15 min (vllm cold-start is the only cost).
- **4.7s.6 (cutover):** `git revert` + ansible redeploy + ZFS snapshot
  recovery if state corruption detected. ZFS snapshot is the catch-all.
  `ClaimVerifier__VllmModel` defaults to base when unset, so partial
  rollback degrades gracefully.

**Hard constraint.** Single-component rollback only. If SFT regresses
one downstream consumer but not another, fix-forward via a follow-on
PR — do not half-roll-back.

---

## 8. Observability deliverables

Detailed in §6 Phase 4.7s.7. Pairs with existing v2 Pipeline Health
dashboard (`v2-pipeline-health.json`, PR #435 + #448 — already shipping
`atlas_semantic_verifier_claim_pass_rate`, the SectorTagger coverage
gauge, and the entity-resolver grounding rate per Phase 4.6).

New panels appended in a `SFT verifier — pre vs post` row:

1. `atlas_semantic_verifier_claim_pass_rate` over 28d with cutover marker.
2. Per-verdict-class rate (`sum by (verdict)(rate(...))`).
3. Per-CLAIM latency p50/p95 vs base-model baseline.
4. Malformed-response rate (should stay at 0).

New P2 alert: `SemanticVerifierPassRateRegression` (§6 Phase 4.7s.7).

---

## 9. Open questions (user decisions needed)

1. **Cloud labeling budget — ANSWERED (2026-05-25).** Redesigned data
   strategy (§3) uses Foundry only for the ~1500 ambiguous/`partial`
   middle-ground at ~$3.50 actual — well under the $10 cap, user-approved.
   Strong-positive + strong-negative (~2500 examples) are synthesized at
   $0 from CPU-verifier-validated DSL extractions. Per
   `[[feedback-azure-foundry-vs-anthropic-cost]]`.

2. **Foundry-agreement gate — retire or keep?** Iter-5 71.53% +
   PR #458 structural-floor finding argues for retiring it as the
   *sole* gate and elevating Signal 2 + Signal 3. §5.1 requires ALL
   THREE signals; weighting when Signal 1 < 85% but Signal 2 ≥ 85% is
   a user call.

3. **LoRA hyperparams — v6 config or new?** §4.1: attention-only,
   r=8/α=16, 1 epoch, lr=2e-5, dropout=0.1. v6 ran r=16/α=32 at
   `train_qlora_unsloth.py:258-259`. Classification needs less
   capacity. **Recommend §4.1 conservative**; veto only if user wants
   different starting point.

4. **Training window — local or cloud?** Local pauses production
   extraction 2-4h/run. Cloud A100-80G spot ≈ $3-5/run, no pause.
   **Recommend local for first run** (verifier paused for benchmark
   anyway); switch to cloud if iteration count >2-3 retrains.

---

## 10. Risks + mitigations

**Risk 1 — Overfit to Foundry bias.** Iter-5's 79-disagreement bucket
(`production=none / foundry=partial`) suggests Foundry over-prefers
`partial`. SFT inherits that bias.
*Mitigation:* Signal 2 (§5.1) is held independent of Foundry; §5.2
re-opens the gate if Signal 2 > Signal 1 + 5pp.

**Risk 2 — LoRA degrades general capability.** Per
`[[project_lora_v6_iteration]]` root cause #2, MLP-targeting changes
reasoning. §4.1 restricts to attention only.
*Mitigation:* MT-Bench-subset regression check (§5.3) gates cutover
(4.7s.6 REQUIRED ≤5pp).

**Risk 3 — AWQ + LoRA serving regression.** Per
`[[project_vllm_sentinel]]`, ~18% slowdown (62 vs 76 tok/s).
Per-CLAIM iter-5 wall = 0.71s; +18% = +0.13s/CLAIM, inside any
plausible SLA.
*Mitigation:* `call_latency_ms` panel in 4.7s.7 flags >18% as
integration bug.

**Risk 4 — Held-out n=100 small for statistical confidence.**
*Mitigation:* the cutover gate requires ALL THREE signals; the 4-week
observability window catches anything n=100 missed.

---

## 11. Success criteria (concrete, measurable)

1. **Held-out Foundry agreement ≥85%** (n=100 fresh + 15% test slice).
2. **Human spot-check agreement ≥85%** (50-CLAIM subset of held-out).
3. **Downstream signal quality maintained or improved** vs iter-5 over
   7-day post-cutover window (v2 Pipeline Health panels).
4. **MT-Bench subset regression ≤5pp** pre vs post SFT.
5. **ZFS-snapshot rollback in <15 min** (rehearsed in 4.7s.8).
6. **Cloud labeling spend ~$3.50** (Foundry-only for the ~1500
   ambiguous middle-ground per §3; strong-positive + strong-negative
   buckets are synthesized at $0 from CPU-verifier-validated DSL
   extractions).

All six clear → Phase 4.7s closes; SFT adapter = production CoVe
verifier; the SFT pipeline becomes the retraining surface. Any miss →
§5.2 failure-mode triage + escalate.

---

## 12. Phase ordering rationale

```
Phase 4.7 acceptance iter-5 (PR #456) — agreement 71.53% PLATEAU
  │
  ▼ (user approval 2026-05-25)
Phase 4.7s (THIS) — SFT fine-tuning of the GPU CoVe verifier
  ├─ 4.7s.1: training data scaffold        ┐
  ├─ 4.7s.2: LoRA training harness         ├─ can land in parallel
  ├─ 4.7s.3: cloud labeling expansion      ┘
  ├─ 4.7s.4: first LoRA run + diagnostic
  ├─ 4.7s.5: adapter integration with vllm-server
  ├─ 4.7s.6: production cutover (gated; no flag)
  ├─ 4.7s.7: 4-week production observability
  └─ 4.7s.8: rollback rehearsal + cleanup
  │
  ▼ (success criteria §11 all-clear)
Phase 5 (matrix integration — parent plan §8) — re-unblocked at higher signal quality
```

4.7s.1, 4.7s.2, 4.7s.3 are independent (different files/scripts) and
can land in parallel. 4.7s.4 sequential after 4.7s.1+4.7s.3 (needs
data). 4.7s.5 sequential after 4.7s.4 (needs adapter). 4.7s.6
sequential after 4.7s.5 (deploys it). 4.7s.7 sequential after 4.7s.6
(monitors it). 4.7s.8 sequential after 4.7s.7 (closes the loop).

---

## 13. References

- Parent: `docs/plans/atlas-dsl-poc-phase4-gpu-semantic-verifier.md` §5
  (Phase 4.7), §11 (gates).
- iter-5 baseline:
  `docs/benchmarks/cod-2026-05-17/results/phase4-7-iter-5-axis-c-rule-first/REPORT.md`.
- 3-way ablation (PR #458):
  `docs/benchmarks/cod-2026-05-17/results/phase4-7-model-bench/REPORT.md`.
- Production ClaimVerifier:
  `SentinelCollector/src/Semantic/ClaimVerifier.cs` (`BuildPrompt`
  L173-197, `VerifyAsync` L200, `IsSelfReferential` L225-238) +
  `ClaimVerification.cs` (`SupportTag`).
- v15 runner (shared prompt/parser):
  `docs/benchmarks/cod-2026-05-17/scripts/run_phase4_7_production_pipeline.py`
  (`build_prompt` L120, `extract_claim_inputs` L389-462).
- Foundry runner + ledger:
  `docs/benchmarks/cod-2026-05-17/scripts/run_phase4_7_foundry_labels.py`,
  `/opt/ai-inference/training-data/azure-oracle-ledger.jsonl`.
- LoRA harness: `SentinelCollector/scripts/train_qlora_unsloth.py`
  (`train` L252).
- vllm-server compose block + LoRA-removal audit:
  `deployment/artifacts/compose.yaml.j2` L808-822, L887-889.
- v2 Pipeline Health (extension target):
  `deployment/artifacts/monitoring/dashboards/v2-pipeline-health.json`.
- Memories: `[[project_lora_v6_iteration]]`,
  `[[project_sentinel_llm_strength_layering]]`,
  `[[atlas-inference-topology]]`, `[[project_vllm_sentinel]]`,
  `[[feedback-azure-foundry-vs-anthropic-cost]]`,
  `[[feedback-drop-feature-flags]]`, `[[feedback-no-placeholder-prs]]`,
  `[[feedback-completion-gate]]`,
  `[[feedback-no-full-article-to-gpu-verifier]]`.
