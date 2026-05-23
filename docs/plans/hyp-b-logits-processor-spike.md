# Hypothesis B — vLLM Custom LogitsProcessor SPIKE (Architecture Verification)

**Date:** 2026-05-23
**Branch:** `experiment/hyp-b-logits-processor-spike`
**Verdict:** **NEED-CUSTOM-DEPLOY** — B is architecturally feasible; per-request callable LogitsProcessors are NOT supported via OpenAI `extra_body` (vLLM 0.19 startup-only registration). Composition with `structured_outputs.grammar` works in the correct order. Requires one-time `vllm-server` relaunch with `--logits-processors <FQCN>` flag.

Inputs: WebFetch `docs.vllm.ai/en/{stable,latest}/features/custom_logitsprocs.html` + direct source inspection of `vllm-server` container (vLLM **0.19.0** on prod).

## Open question from scoping doc (#419 §B)

> "Whether vLLM's `structured_outputs.grammar` and `logits_processors` compose. If they don't, B2 collapses to grammar OR stateful, not both."

Spike answers both this and the per-request feasibility question.

## WebFetch findings (cached 2026-05-23)

`docs.vllm.ai/en/latest/features/custom_logitsprocs.html`, verbatim:

> "Logits processors are loaded at initialization. **Critically, the set
> of loaded logits processors cannot be modified after the vLLM engine
> finishes loading, and new logits processors cannot be loaded on-demand
> for individual requests.**"

Three documented registration paths (all server-startup):
1. FQCN CLI — `vllm serve <model> --logits-processors module.path:Cls`
2. Python entry-points — `[project.entry-points."vllm.logits_processors"]`
3. Class object — `LLM(model=..., logits_processors=[Cls])` *(offline only)*

Per-request: only `vllm_xargs` → `SamplingParams.extra_args` — i.e. the
request **parameterises** an already-registered processor; it cannot
ship a new one.

`structured_outputs.html` does not document composition with custom procs.
Resolved by source inspection (below).

## Container introspection (vllm-server 0.19.0)

### `SamplingParams` fields

```python
# vllm.sampling_params.SamplingParams
logit_bias: dict[int, float] | None = None   # static per-token bias
extra_args: dict[str, Any]    | None = None  # → registered procs
# NO `logits_processors` field on SamplingParams.
```

### OpenAI ChatCompletion request schema

`vllm/entrypoints/openai/chat_completion/protocol.py:340`:

```python
vllm_xargs: dict[str, str|int|float|list] | None
```

`grep -rn '"logits_processors"' vllm/entrypoints/openai/` → **zero hits**.
The chat-completion request schema has NO `logits_processors` field. The
helper `get_logits_processors` exists in `engine/protocol.py:184` but has
no caller in the chat-completion path. So OpenAI `/v1/chat/completions`
on vLLM 0.19 **cannot** accept a per-request LogitsProcessor class via
`extra_body`.

### Composition order with `structured_outputs.grammar`

`vllm/v1/worker/gpu/model_runner.py`:

```python
# L840: grammar bitmask applied BEFORE sampler
self.structured_outputs_worker.apply_grammar_bitmask(logits, ...)
# L850: sampler runs custom logits_processors
sampler_output = self.sampler(logits, input_batch)
```

Inside `sampler()` → `apply_logits_processors` (`vllm/v1/sample/sampler.py:296`)
iterates `sampling_metadata.logitsprocs.non_argmax_invariant` and applies
each `processor.apply(logits)`.

**Deterministic order:**

```
raw → grammar bitmask (-inf on invalid) → custom procs → temp → top-p → sample
```

Custom processor inherits the grammar's format-validity floor. It cannot
contradict the grammar (already-masked tokens stay at -inf) but it can
further mask any subset of the grammar-allowed set — exactly what B needs.

## Architecture sketch

### DeclareOnceProc + KindCapProc (combined ~150 LOC)

```python
# atlas/sentinel/dsl_logits.py  (mounted into vllm-server)
from vllm.v1.sample.logits_processor import AdapterLogitsProcessor
import torch, re

class DslStatefulProc(AdapterLogitsProcessor):
    """Per-request stateful: DECLARE-ONCE + per-kind caps."""

    @classmethod
    def validate_params(cls, params):
        x = params.extra_args or {}
        if "kind_caps" in x and not isinstance(x["kind_caps"], dict):
            raise ValueError("kind_caps must be dict[str,int]")

    def is_argmax_invariant(self): return False

    def new_req_logits_processor(self, params):
        x = params.extra_args or {}
        if not (x.get("declare_once") or x.get("kind_caps")):
            return None  # opt-out
        return _Impl(self._tokenizer, x)

class _Impl:
    _ANON = re.compile(rb'_anon:([A-Za-z0-9_]+)')
    _KIND = re.compile(rb'\bENT\s+(\w+)\b|\bNUM\b|\bEVT\b')
    def __init__(self, tok, cfg):
        self.tok, self.cfg = tok, cfg
        self.declared: set[bytes] = set()
        self.kind_counts: dict[str,int] = {}
    def __call__(self, output_token_ids, logits):
        text = self.tok.decode(output_token_ids[-64:]).encode()
        for m in self._ANON.finditer(text): self.declared.add(m.group(1))
        for m in self._KIND.finditer(text):
            k = (m.group(1) or m.group(0)).decode().lower()
            self.kind_counts[k] = self.kind_counts.get(k,0) + 1
        # Mask: tokens completing a duplicate _anon:<id>; tokens opening
        # a kind that has hit its cap. Both implemented via tokenizer trie
        # precomputed once at __init__ (O(|vocab|)).
        for tid in self._mask_ids(): logits[tid] = float('-inf')
        return logits
```

### Tokenizer-boundary mitigation

`_anon:<id>` and `ENT concept` are multi-token in Qwen BPE. Two-step:
1. **Decode-then-detect** last 64 output tokens (~50us/call); update sets.
2. **Span-completion mask** via precomputed `(prefix → next_token_id)`
   trie built from tokenizer vocab at `__init__`. O(|vocab|) one-time.

### Deploy path (B-impl, not this PR)

1. Mount `atlas/sentinel/dsl_logits.py` into vllm-server container.
2. Relaunch with `--logits-processors atlas.sentinel.dsl_logits:DslStatefulProc --logits-processor-pattern '^atlas\.sentinel\.'`.
3. Per-request:
   ```python
   extra_body = {
     "vllm_xargs": {"declare_once": True, "kind_caps": {"concept":5,"org":10}},
     "structured_outputs": {"grammar": GBNF_V2_1},  # composes
   }
   ```

## Decision matrix

| Question | Answer | Evidence |
|---|---|---|
| Per-request LogitsProcessor CLASS via OpenAI `extra_body`? | **NO** | source: no field in request schema; doc: "cannot be modified after engine finishes loading" |
| Per-request CONFIG of pre-registered processor via `vllm_xargs`? | **YES** | `ChatCompletionRequest.vllm_xargs` → `extra_args` → `validate_params` per request |
| Compose with `structured_outputs.grammar`? | **YES** (favourable order) | grammar bitmask **before** sampler runs custom procs |
| Server restart required? | **YES — one-time** | startup-only registration |
| Throughput hit? | Deferred to B-impl smoke | requires sidecar dev server |

## POC status

**Skipped per spike brief** ("If POC fails or per-request LogitsProcessor
not supported → SKIP step 2"). Per-request CLASS injection is not
supported, so the POC cannot run against the production vllm-server :8000
without relaunching it with our FQCN + pattern flags — a production
config change the spike brief explicitly forbids.

B-impl bootstrap will need a `vllm-server-dev` sidecar on :8001 with the
test FQCN, OR an offline `LLM(...)` script (incompatible with current
sidecar topology — vllm-server holds the GPU).

## Revised cost vs scoping doc (#419 §B)

| Phase | Scoping doc | This spike |
|---|---|---|
| 1w spike | "verify composes with grammar" | DONE. Per-request CLASS = NO; composition = YES. |
| 1w impl + unit-test | ~150 LOC processor + trie + replay | unchanged |
| 0.5w runner integration | unchanged | unchanged |
| 0.5w sweep | unchanged | unchanged |
| **+ NEW: 0.5w deploy plumbing** | — | dev sidecar + mount + ansible role + canary |

**Revised total: 2.5–3.5 weeks** (vs scoping doc 2–3w). Deploy plumbing is
the only material increase; spike-falsification risk eliminated.

## Reframing of risk

Scoping doc flagged B as HIGH risk. This spike narrows the risk to the
**deploy/ops axis only** — the capability is confirmed, composition is
correct, and the architecture is sound. The HIGH rating is justified
only by the production launch-config dependency, not by API uncertainty.

## STATE.md one-liner

Hyp-B spike: per-request LogitsProcessor CLASSES via OpenAI `extra_body` are NOT supported (vLLM 0.19 startup-only registration); composition with `structured_outputs.grammar` confirmed favourable (grammar bitmask BEFORE custom procs); verdict NEED-CUSTOM-DEPLOY (relaunch `vllm-server` with `--logits-processors <FQCN>`); B-impl cost revised to 2.5–3.5wk including deploy plumbing.
