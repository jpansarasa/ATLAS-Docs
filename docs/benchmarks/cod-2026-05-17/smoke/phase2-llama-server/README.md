# Phase 2 llama-server smoke

Captured on first cold deploy of the `llama-server` service
(2026-05-21) per `docs/plans/atlas-dsl-poc-phase2-llama-server.md`
stories 1–4. Endpoint: `http://localhost:11437` (host) →
`http://llama-server:8080` (container).

## Files

| File | Purpose | Result |
|---|---|---|
| `01-health.txt` | `GET /health` liveness | 200 `{"status":"ok"}` |
| `02-v1-models.json` | `GET /v1/models` model identity | qwen3-30b-a3b-instruct-2507-q4_K_M.gguf, n_params 30.5B, size 18.55 GB, n_ctx 16384 |
| `03-ext-ok-request.json` / `03-ext-ok-response.json` | Minimal GBNF (`root ::= "EXT_OK"`) | `content="EXT_OK"`, 5 tokens, ~29 tok/s |
| `04-v1grammar-request.json` / `04-v1grammar-response.json` | v1 GBNF (`docs/grammars/cod-dsl-v1.gbnf`) load test | Server accepted the grammar (HTTP 200) but output was NOT constrained to the `DSL: v… SOURCE: …` header — see "v1-grammar finding" below |

## Defaults chosen (open questions in the scoped plan)

- `--ctx-size 16384` — matches `run_cod_dsl.py --num-ctx` default; like-for-like with the Phase 2 ollama-cpu-gen baseline arm.
- `--parallel 1` — mirrors `OLLAMA_NUM_PARALLEL=1`; Phase 2 sweep is sequential.
- `--n-gpu-layers 0` — explicit CPU-only (CPU = extraction per the topology rule).
- Image tag — upstream `ghcr.io/ggml-org/llama.cpp:server` (mutable). Build info from `/props`: `b9246-871b0b70f`. SHA pinning deferred until Phase 2 results land.
- systemd — deferred; `atlas.service`'s `nerdctl compose up` already manages this service via the compose file.

## v1-grammar finding (Story 6 — RESOLVED 2026-05-21)

The minimal `root ::= "EXT_OK"` grammar enforces correctly. The v1 GBNF
at `docs/grammars/cod-dsl-v1.gbnf` was originally **accepted** by
llama-server (no HTTP 400, no parse-error in the response body) but the
model's output did not start with the required `DSL: v` literal — see
`04-v1grammar-response.json`'s `content` field.

A control experiment (POST with `grammar: "this is garbage"`)
confirmed that `llama-server`'s `/completion` silently ignores invalid
grammars and falls through to unconstrained sampling. The diagnostic
commit `5df053c8` pinpointed the cause as v1.0's high-bound repetition
counts (esp. `value-char{0,2047}`, line 129) tripping llama.cpp's
GBNF-parser sanity threshold (`MAX_REPETITION_THRESHOLD = 2000`, plus
a combinatorial rule-expansion cap that's empirically tighter still).

**Fix:** v1.1 bounds pass — `value-char{0,255}`, `block-list{0,128}`,
`line-rest{0,255}`, `string-content{0,255}`, `qual-val-char{0,127}`
unchanged. See the commit `fix(dsl-poc/phase2): v1 GBNF bounds — pass
llama.cpp parser` and the in-grammar header comment above the `value`
rule for the full empirical story.

**Validation evidence:** see `06-grammar-validation-success.md` plus
the `.json` request/response envelope and `.log.txt` raw container
logs from the call. HTTP 200, no parse error in container logs,
output begins with `DSL: v1\n` — all three Story 6 acceptance
criteria met.

## Sibling-container side effect (deploy artifact, not server artifact)

`ansible-playbook ... --tags llama-server` re-rendered
`/opt/ai-inference/compose.yaml` (the template task is tagged
`always`) and the downstream `[always]`-tagged
"Start or restart ATLAS infrastructure" task cycled the systemd
`atlas.service`. Net effect: most compose-managed containers
restarted in addition to `llama-server` coming up new. All siblings
recovered cleanly within ~10 s and were back `Up` before the smoke
tests ran. `vllm-server` (managed outside this compose file) was
not touched. Documented for visibility — the deploy block itself
only acts on `llama-server`; the side effect lives in the `always`
tasks earlier in the playbook.
