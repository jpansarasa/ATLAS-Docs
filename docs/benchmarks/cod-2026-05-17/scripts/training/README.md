# Phase 4.7s.2 — LoRA training harness

Infrastructure for SFT-fine-tuning the GPU CoVe verifier with a LoRA
adapter on top of `Qwen/Qwen2.5-32B-Instruct-AWQ`. Built per Phase
4.7s.2 of `docs/plans/atlas-dsl-poc-phase4-7-sft-cove-verifier.md`.

**This PR delivers the harness — no live training runs here.** Phase
4.7s.4 wall-times the first run; this scaffold gives that PR a
config-validated, prompt-locked, eval-instrumented launch surface.

## Files

| file | purpose |
|---|---|
| `train_lora.py` | training entrypoint (PEFT + transformers + trl); `--dry-run` validates config |
| `lora_config.yaml` | hyperparameters (attention-only LoRA per v6 lesson) |
| `prompt_template.py` | byte-locked port of `ClaimVerifier.BuildPrompt` |
| `eval_lora.py` | metrics + REPORT.md generation from a LoRA checkpoint |
| `mt_bench_subset_regression.py` | generic-capability regression gate (§5.3) |
| `tests/test_train_lora_harness.py` | substantive coverage of all the above |

## Install (training-only deps)

The bench venv `.venv-bench` ships with `pyyaml` and `pytest` only;
training-specific deps must be installed there explicitly when Phase
4.7s.4 lands. **Do not install into the production sentinel-collector
container** — these libraries are CPU+GPU-heavy and have no runtime
role in production.

```bash
# from the repo root
source .venv-bench/bin/activate
pip install \
    'torch>=2.4' \
    'transformers>=4.45' \
    'peft>=0.13' \
    'trl>=0.11' \
    'datasets>=3.0' \
    'accelerate>=0.34' \
    'bitsandbytes>=0.44' \
    'tensorboard>=2.18'
```

The config validation, prompt-template tests, eval metric code, and
MT-Bench regression integration all work **without** those deps —
heavy imports are lazy and gated behind clear `ImportError` pointers
to this README.

## Quick start

```bash
# 1. Validate config + data without launching training (4.7s.2 gate):
.venv-bench/bin/python docs/benchmarks/cod-2026-05-17/scripts/training/train_lora.py \
    --dry-run

# 2. Run unit tests:
.venv-bench/bin/python -m pytest \
    docs/benchmarks/cod-2026-05-17/scripts/training/tests/test_train_lora_harness.py -v

# 3. (Phase 4.7s.4) launch real training:
.venv-bench/bin/python docs/benchmarks/cod-2026-05-17/scripts/training/train_lora.py
```

## Architectural constraints (do not relax without an RFC)

1. **Attention-only LoRA.** Target modules must be a subset of
   `{q_proj, k_proj, v_proj, o_proj}`. v6 LoRA included MLP targets
   and destroyed general capability per
   `[[project_lora_v6_iteration]]`. The config validator rejects MLP
   names at load time.

2. **Production-locked base.** `model_id` defaults to
   `Qwen/Qwen2.5-32B-Instruct-AWQ` — the same string
   `ClaimVerifier.ProductionModel` literal hosts. A different base
   would break adapter swap-compatibility with vllm-server.

3. **Pre-anchored spans as input.** Training rows must carry
   `subject / predicate / object_text / evidence_span` — never the
   full article. The prompt is built from those four slots per the
   2026-05-25 BuildPrompt rework.

4. **Four-verdict completion vocabulary.** Labels are exactly
   `consistent / related / unrelated / insufficient_evidence`. The
   runtime production parser collapses to the three-valued
   `ClaimSupport` enum; the adapter is trained on the full four-way
   distinction.

5. **Production extraction must pause during training windows.** The
   5090 hosts both training and the vllm-server; concurrent execution
   OOMs. Operator-controlled — there is no automatic stop in this
   harness.

## How the prompt stays in sync

`prompt_template.build_prompt` is a verbatim port of
`SentinelCollector/src/Semantic/ClaimVerifier.cs::BuildPrompt`. The
test `test_prompt_byte_matches_csharp_buildprompt` parses the C# source
in CI and asserts byte-equality with the Python emission. Any drift in
either file fails the test immediately — no manual diff required.
