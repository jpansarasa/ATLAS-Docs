# GPU-JSON-CoD role-flip — staged implementation + rollout plan (2026-06-09)

STATUS: PR-1 IMPLEMENTED (additive + inert). User-approved direction (role-flip CoD CPU→GPU, JSON-schema output).
OWNER-DECISION CHECKPOINT: the production cutover (PR-5) is the single user gate. Everything before it is additive/shadow and supervisor-drivable.

## TOPOLOGY — DISTRIBUTED role-flip (the load-bearing decision)
The flip is a **distribution of work across two inference tiers, not a consolidation onto one.**
- **GPU `vllm-server` (Qwen2.5-32B-Instruct-AWQ) → CoD.** The batched JSON-schema extraction (the throughput win) runs here.
- **CPU `ollama-cpu-gen` (qwen3-30b-a3b) → CoVe** (ClaimVerifier + sector tagging + the per-claim verification side of the pipeline). CoVe **MOVES off the GPU to the CPU** at cutover.
- RATIONALE — **don't pile everything on the GPU.** The single 32B-AWQ vLLM instance has a finite seq budget (`max_num_seqs=16`). CoD is a *batched* workload that wants as many of those seqs as it can get (that is where the ~35× comes from). If CoVe stays co-located it competes for the same 16 seqs and gets **starved behind the CoD batch** — the exact CPU-slot-contention failure the current architecture already fought (DigestNarrative / SectorTagger were pulled off the ambient client for this reason). Splitting CoVe onto the otherwise-freed CPU tier (the CPU `llama-server`/`ollama-cpu-gen` is *vacated* by moving CoD off it) keeps each tier doing the work it is sized for: GPU = high-throughput batched extraction, CPU = latency-tolerant per-claim verification.
- This supersedes any earlier "keep CoVe co-located on the GPU by default" framing in this doc — **the default is now CoVe-on-CPU**, and §7 records it as a settled direction rather than an open user call. CoVe-placement is its OWN later PR (see §6 PR-3b); **PR-1 ships ONLY the GPU CoD backend** and changes no CoVe wiring.

---

## 0. CONTEXT — what's validated and why this is a role-flip not a rewrite

Validated this investigation (`/tmp/sentinel-remediation/json-cod-quality/RESULTS.md`):
- Move CoD off the CPU `llama-server` (qwen3-30b-a3b, GBNF DSL) onto the **existing GPU `vllm-server`** (`Qwen/Qwen2.5-32B-Instruct-AWQ`) using **`response_format=json_schema`** (OpenAI-standard; NOT vLLM `guided_json`, NOT the GBNF DSL).
- The GBNF DSL's free-text copy-zone wildcards (`name`/`raw`/`trigger` spanning arbitrary article text) were the constrained-decode masking bottleneck. The bounded JSON schema (every string `maxLength`-capped + `additionalProperties:false`) keeps the decode mask small → throughput holds.
- Result: **~700 art/hr batched (~35× the ~20/hr CPU)** at **macro recall 0.777 on the clean path (n=23) ≈ the 0.75 qwen3-30b gold**, NUM-skip eliminated (679 NUM, 100% carry a value, 98% verbatim-in-article).
- Loop-guard (temp=0 degenerate-repetition to the token cap on a minority of articles): `repetition_penalty=1.1` + per-array `maxItems≈50–60` + `max_tokens≈4096`.

Why this is additive-friendly, NOT a rewrite of downstream:
- The current CPU path is `CpuDslExtractionService` (`Extraction:Backend=LlamaServerDsl`): `llama-server /completion (GBNF)` → `dsl-parser-mcp /parse` → `/verify` → `IDslToMergedExtractionAdapter.Adapt(...)` → `MergedExtractionResult`.
- **The adapter is the integration seam.** It already lowers a `DocumentAst` (ENT/NUM/EVT/CLAIM, `SentinelCollector/src/Models/Dsl/DslAst.cs`) + a `VerifierReport` onto the `MergedExtractionResult` that *every* downstream consumer (ExtractionProcessor, SemanticVerifierService/CoVe, projector, digest) already keys on.
- So if the new JSON path produces the **same `DocumentAst` shape**, the adapter and everything downstream are untouched. CoVe/ClaimVerifier + SectorTagger today call `vllm-server:8000` via dedicated named VllmClients in `DependencyInjection.cs` (`ClaimVerifier` L549, `SectorTaggerVllm` L570). The role-flip moves CoD ONTO that GPU instance; the distributed-topology end-state then moves CoVe OFF it onto the vacated CPU tier (a separate later PR — see §3 / §6 PR-3b) so the batched CoD does not starve the per-claim verifier. The seam invariance is what makes both moves additive: every consumer keys on `MergedExtractionResult`, not on which tier produced it.

### Critical files (read before touching)
| Concern | Path |
|---|---|
| Backend enum + all extraction tunables | `SentinelCollector/src/Configuration/ExtractionOptions.cs` (`InferenceBackend`, L6-41) |
| CPU DSL service (the template to mirror) | `SentinelCollector/src/Services/CpuDslExtractionService.cs` |
| The integration seam | `SentinelCollector/src/Extraction/IDslToMergedExtractionAdapter.cs` |
| Sidecar parser/verifier endpoints | `SentinelCollector/dsl-parser-mcp/app.py` (`/parse`, `/verify`) |
| AST contract both paths must produce | `SentinelCollector/src/Models/Dsl/DslAst.cs` (`DocumentAst`) |
| vLLM JSON `response_format` client pattern | `SentinelCollector/src/Services/VllmClient.cs` (`GenerateStructuredAsync`, L132) |
| Backend → client + service DI | `SentinelCollector/src/DependencyInjection.cs` (transport L288-337; service bind L770-781) |
| CoD config (parser endpoint, prompt/grammar files) | `SentinelCollector/src/Configuration/CpuCodOptions.cs` |
| Worker batch/parallel degree | `SentinelCollector/src/Workers/ExtractionProcessor.cs` (`MaxDegreeOfParallelism`, L139-145) |
| Compose env (sentinel-collector + vllm-server) | `deployment/artifacts/compose.yaml.j2` (L843-894) |
| vLLM launch + ZFS snapshot + group_vars | `deployment/ansible/playbooks/deploy.yml`; `deployment/ansible/group_vars/all.yml` (`vllm_*`, `sentinel_max_concurrent_extractions`) |
| Validated schema/prompt/scorer (REUSE) | `/tmp/sentinel-remediation/json-cod-quality/{json_schema.py,cod_json_prompt.txt,run_json_cod.py,analyze_scores.py}` |

---

## 1. LAYER 1 — CoD output DSL→JSON: the GPU-vLLM JSON-CoD extraction backend (ADDITIVE)

Goal: a new `IMergedExtractionService` that runs JSON-CoD on `vllm-server`, selectable by a NEW backend value, **alongside** the existing `LlamaServerDsl` path. No behavior change until a config flip.

### 1.1 Productionize schema + prompt (port from `/tmp` artifacts → repo, host-mounted)
- New host-mounted prompt: `/prompts/cod/cod_json_v1.txt` (port `cod_json_prompt.txt` verbatim; keep `{{article_text}}`/`{{source_id}}` placeholders — `CpuDslExtractionService.BuildPrompt` already uses these).
- New schema asset `cod_json_schema_v1.json` shipped to `/prompts/cod/` (port `json_schema.py:COD_JSON_SCHEMA`). The schema, not GBNF, is the mask-cheap constraint:
  - every string field `maxLength`-capped (40–120) + `additionalProperties:false` (preserves the speed win — no free-text wildcard),
  - `numbers[]` items REQUIRE `{source_text, value, unit, context}` (the explicit `value` field is the NUM-skip fix),
  - **add `maxItems` per array (loop-guard, §4)** — `entities/numbers/events/claims` ~50–60.
- Add `CpuCod__JsonSchemaFile=cod_json_schema_v1.json` and `CpuCod__JsonPromptFile=cod_json_v1.txt` to `CpuCodOptions` (additive fields; existing GBNF fields untouched).

### 1.2 New service `GpuJsonExtractionService : IMergedExtractionService`
Mirror `CpuDslExtractionService`'s 4-stage structure and telemetry, but:
- **Stage 1 (LLM):** call `vllm-server` via `VllmClient.GenerateStructuredAsync` (already does `response_format=json_schema`, L156-164) with the bounded schema + loop-guard params (§4). Load prompt/schema host-mounted with the same mtime cache.
- **Stage 1.5 (parse):** POST the JSON to the NEW sidecar endpoint `dsl-parser-mcp /parse_json` (Layer 2) → the SAME `DocumentAst`.
- **Stage 2 (verify):** UNCHANGED — `dsl-parser-mcp /verify` over the `DocumentAst` (byte+word grounding is format-agnostic; it keys on `source_span`/`raw`/`verbatim_name`, not on DSL syntax).
- **Stage 3 (adapt):** UNCHANGED — `IDslToMergedExtractionAdapter.Adapt(...)`.
- Reuse `SentinelMeter.CpuExtraction*` counters with a `path=gpu_json` tag (or add `GpuJsonExtraction*` mirrors) so dashboards split the two paths during shadow.

### 1.3 New backend value + DI binding
- Add `InferenceBackend.VllmJson` to `ExtractionOptions.cs`.
- `DependencyInjection.cs`:
  - transport block (L299-337): `VllmJson` registers the `VllmClient` HttpClient against `Extraction:VllmEndpoint` (same as `VllmServer`).
  - service bind (L770-781): `VllmJson` → `services.AddScoped<IMergedExtractionService, GpuJsonExtractionService>()`. All other backends unchanged.
- Validator: `GpuJsonExtractionService` requires a `VllmClient` (cast-guard mirroring `CpuDslExtractionService`'s `LlamaServerClient` guard, L106-111) so a misconfig fails loud at first call, not silently.

DELIVERABLE PR-1 = new service + schema/prompt assets + backend enum + DI, all behind a backend value nothing sets yet. Unit test: JSON→`DocumentAst` mapping fidelity + cast-guard. `compile.sh` (with tests) green.

---

## 2. LAYER 2 — Parser: `dsl-parser-mcp /parse_json` → the SAME `DocumentAst`

Goal: deterministically lift the validated JSON document into the existing `DocumentAst` so `/verify` + the adapter + downstream are unchanged.

- New FastAPI endpoint in `SentinelCollector/dsl-parser-mcp/app.py`: `POST /parse_json` taking `{json_text, input_text}` → `ParseResponse{document_ast, parse_errors}` (same wire shape as `/parse`, so the .NET `DslParserClient` reuses its `ParseResponseWire`/`DocumentAst` deserialization).
- Mapping (JSON keys → `DocumentAst`, per `DslAst.cs`):
  - `entities[] {name, ent_type, ticker?}` → `DslEnt{local_id, ent_type, verbatim_name=name}` (synthesize stable `local_id` from name/ticker for EVT/CLAIM `*_refs`).
  - `numbers[] {source_text, value, unit, context}` → `DslNum{num_id, raw=source_text, slots=[value, unit, context]}`.
  - `events[] {event_kind, trigger, subject, object?}` → `DslEvt`.
  - `claims[] {claim_kind, subject, object, predicate?, polarity?}` → `DslClaim{claim_kind, subject, claim_text=object, slots=[predicate], ...}`.
- **`source_span` is the verifier's anchor.** JSON-CoD emits verbatim copy slots (`source_text`/`name`/`trigger`) but no `[start,end)` offset. The parser computes the span by locating each verbatim slot in `input_text` (first/best occurrence) and emitting the `[start,end)` tuple `/verify` expects (`app.py:_span`, `verifier_v2_3_1.is_span_pointer`). Where a slot isn't found verbatim, leave `source_span=None` → verifier grounds Support=None (the existing safe-degrade; §8.2 forbids a cell with no source span). This keeps the byte/word grounding gate intact for the JSON path.
- Keep `/parse` (GBNF) untouched — both endpoints coexist during shadow.
- Tests: `tests/test_api.py` extension — JSON→AST round-trip + span-location on the validated `json-cod-final-dedup.jsonl` fixtures; assert `/verify` over the produced AST yields a sane byte/word rate vs the known-good gold.

DELIVERABLE PR-2 = `/parse_json` + span-locator + tests. Sidecar image rebuild via `--tags dsl-parser-mcp`.

---

## 3. LAYER 3 — Backend wiring (compose / group_vars)

Goal: route CoD to GPU vLLM; the distributed end-state moves CoVe to the CPU tier (PR-3b, separate). **No live flip in this PR** — wire the additive config so shadow (Layer 4) can run.

- `compose.yaml.j2` sentinel-collector env (additive, alongside existing `Extraction__Backend=LlamaServerDsl`):
  - `CpuCod__JsonSchemaFile`, `CpuCod__JsonPromptFile` (new prompt/schema assets).
  - The loop-guard knobs (§4) as env so they're tunable without rebuild.
- **VRAM note (32B-AWQ instance):** `group_vars/all.yml` already runs `Qwen/Qwen2.5-32B-Instruct-AWQ`, `max_model_len=32768`, `gpu_memory_utilization=0.92`, `max_num_seqs=16` on the RTX 5090. CoD now adds load to this instance. The 32B-AWQ weights + KV-cache for 16 seqs at 32K already fit (it's the production serving config); JSON-CoD requests are ≤~4K output (loop-guard) so they're *lighter* per-request than CoVe's full-context calls. No model/VRAM change for the CoD add. The CoVe→CPU move (below) then *removes* the per-claim verification load from this instance, giving the batched CoD the seq budget unshared. The deploy VRAM report (deploy skill) confirms headroom under batched CoD load before cutover.
- **The throughput win requires batching** (this is the load-bearing wiring detail): the ~700 art/hr number is *batched* on `max_num_seqs=16`. Today `sentinel_max_concurrent_extractions=1` and `ExtractionProcessor.MaxDegreeOfParallelism=MaxConcurrentExtractions` (L139-145) — that `=1` was forced by the CPU `llama-server --parallel 1` slot (the `:sig:` card + `ExtractionOptions.MaxConcurrentExtractions` doc both cite it). On the GPU path there is no single-slot constraint, so the flip MUST also raise `sentinel_max_concurrent_extractions` (e.g. 8–12, ≤ `vllm_max_num_seqs`) — otherwise CoD runs serially and the 35× win never materializes. Gate this raise to the `VllmJson` backend (don't raise it while `LlamaServerDsl` is live — it would re-create the CPU-slot contention the value was set to avoid).
- **CoVe placement (DISTRIBUTED decision, see §7):** the end-state moves CoVe (ClaimVerifier + SectorTagger) OFF the GPU vLLM onto the CPU tier vacated by CoD. Rationale: a batched CoD workload on `max_num_seqs=16` will starve per-claim CoVe calls if they share the instance — the same single-slot-contention class the DigestNarrative/SectorTagger dedicated-client workaround already documents. Moving CoVe to the CPU `ollama-cpu-gen` (which CoD is leaving) keeps GPU=batched-extraction, CPU=latency-tolerant-verification. This is a SEPARATE later PR (PR-3b) — it re-points the `ClaimVerifier` / `SectorTaggerVllm` / `DigestNarrative` named clients (and the per-call timeout budget, which is already CPU-aware via `SemanticPipelineTimeoutSeconds`) at the CPU endpoint. **Not in PR-1.** PR-1 leaves all CoVe wiring exactly as-is.

DELIVERABLE PR-3 = additive compose/group_vars for the CoD path (no `Backend` flip). Templating-only; renders clean.
DELIVERABLE PR-3b = CoVe→CPU client re-point (ClaimVerifier + SectorTagger + DigestNarrative named clients → CPU endpoint; per-call timeout already CPU-aware). Gated to land before cutover so the GPU seq budget is unshared when CoD goes live.

---

## 4. LAYER 4 — Loop-guard wired into the GPU request

The validated fix, wired into `GpuJsonExtractionService`'s vLLM call (and exposed as tunable config, defaults from RESULTS.md):
- `repetition_penalty=1.1` — `VllmClient.GenerateStructuredAsync` request body needs a new optional field (`VllmCompletionRequest` already carries `seed`/`stop`; add `repetition_penalty`). Forwarded only when set.
- `max_tokens≈4096` — per-call override (NOT the global `MaxCompletionTokens=16384`); a JSON-CoD doc that hasn't closed by 4K is in a loop, so cap + salvage.
- `maxItems≈50–60` per array — in the SCHEMA (Layer 1), so the constrained decoder itself can't emit an unbounded array. This is the structural guard; `repetition_penalty` is the sampling guard; together they closed the residual 5/30 loopers.
- Partial-JSON salvage: on a `max_tokens` truncation, attempt to parse the longest valid prefix (mirror `merge_and_fill.py`) → recover pre-truncation obs rather than dropping the article. Emit a `path=gpu_json,outcome=truncated_salvaged` counter so the rate is observable/alertable.

DELIVERABLE: folded into PR-1 (the service) + PR-1's schema. Called out separately because it's the one thing that must be present *before* shadow or the minority-loop articles regress.

---

## 5. LAYER 5 — STAGED cutover

### 5.1 Shadow / canary
- Mechanism: reuse the existing shadow muscle. Run `GpuJsonExtractionService` over a copy of the live queue writing to a shadow sink (`sentinel.extracted_observations_shadow` via `IShadowObservationWriter`, already DI-registered) while `LlamaServerDsl` stays the production writer. (If per-service shadow dispatch is cleaner than the `ShadowMode`/`V2EnabledSources` plumbing, a thin shadow-runner that calls the new service on the same RawContent ids and writes shadow rows is acceptable — decide at PR-4 time; do NOT entangle with the V1/V2 `ShadowMode` flag which gates a different axis.)
- Compare GPU-JSON-CoD shadow vs `LlamaServerDsl` prod on the same articles: obs count, NUM coverage, verifier byte/word rate, recall proxy, per-article wall time + tok/s, loop/truncation rate.

### 5.2 Re-validate at scale (the quality gate)
- Run the validated scorer (`SentinelCollector/tools/truncation-experiment/scorer.py` + GPU judge) over **n≥72** shadow articles against the qwen3-30b gold baselines (`docs/benchmarks/cod-truncation-2026-06-07/p1-sample.jsonl`).
- PASS CRITERIA (all): macro recall **≥0.75**; NUM coverage ~100% with verbatim-present ≥0.95; loop/truncation rate below the shadow-observed ceiling; throughput ≥ several× CPU on the batched path. n≥72 and the 0.75 floor are hard gates.

### 5.3 Cutover (THE user checkpoint)
- One config switch: `Extraction__Backend=LlamaServerDsl` → `VllmJson` in `compose.yaml.j2`, paired with the `sentinel_max_concurrent_extractions` raise (§3). Deploy `--tags sentinel-collector` (+ render).
- This is the single point requiring user sign-off (per the approved direction). Everything before it is reversible-by-default (additive, shadow-only).
- Post-cutover smoke (deploy skill): VRAM report, Loki errors last 10 min, confirm matrix cells + ntfy digest still populate (the `cod_cutover_zeroed_digest` failure class — digest and matrix are SEPARATE consumers; verify BOTH), confirm `path=gpu_json,outcome=success` climbing and the CPU `llama-server` (old CoD host) going idle. With PR-3b in place, CoVe load should be visible on `ollama-cpu-gen` (the CPU tier CoD vacated), NOT on the GPU vLLM — verify the GPU seqs are CoD's.

### 5.4 Rollback (one revert away at every stage)
- Config: flip `Extraction__Backend` back to `LlamaServerDsl` + restore `sentinel_max_concurrent_extractions=1`; redeploy `--tags sentinel-collector`. The CPU path is left fully wired through cutover (don't delete it until §6 PR-6), so rollback is a one-line env revert.
- Code: `git revert` the cutover PR.
- Data/infra: ZFS snapshot before cutover (deploy.yml auto-snapshots `nvme-fast/timeseries`; `zfs-snapshot.yml -e snapshot_tag=pre-gpu-json-cod` for an explicit tag) → `zfs rollback` if shadow/prod rows need unwinding.

---

## 6. PR DECOMPOSITION (additive → cutover → drop)

| PR | Scope | Risk | Reversible |
|---|---|---|---|
| **PR-1** ✅ DONE | `GpuJsonExtractionService` + schema/prompt assets + `InferenceBackend.VllmJson` + DI bind + loop-guard (§4) + `IDslParserClient.ParseJsonAsync` (`/parse_json`) client method + unit tests | low — dead code until a backend nothing sets | n/a (inert) |
| **PR-2** | `dsl-parser-mcp /parse_json` + verbatim span-locator + sidecar tests; image rebuild | low — new endpoint, `/parse` untouched | n/a (additive) |
| **PR-3** | Additive compose/group_vars (new CoD config + loop-guard env); NO `Backend` flip | low | n/a (additive) |
| **PR-3b** | **CoVe → CPU** — re-point `ClaimVerifier` / `SectorTaggerVllm` / `DigestNarrative` named clients (and reaffirm the CPU-aware per-call timeout) at the CPU inference endpoint so the batched CoD does not starve per-claim verification on the GPU's 16 seqs | med — moves a live workload across tiers | config/client revert |
| **PR-4** | Shadow runner + dashboards/metrics split (`path=gpu_json`) + scale-validation harness (n≥72 scorer) | med — runs the new path live in shadow | shadow-only; stop runner |
| **PR-5** | **CUTOVER** — `Backend=VllmJson` + `sentinel_max_concurrent_extractions` raise. **USER GATE.** | high | config revert (§5.4) + ZFS |
| **PR-6** | After soak: drop `LlamaServerDsl` backend, `CpuDslExtractionService`, GBNF assets, `/parse` (GBNF), `llama-server` from compose/deploy. (`ollama-cpu-gen` is RETAINED — it now hosts CoVe per the distributed topology.) | low (post-stable) | git revert |

Sequencing: PR-1 ⇒ PR-2 (PR-1's service calls PR-2's endpoint, but compiles against the contract independently — can land in either order, PR-4 needs both). PR-3 after PR-1/PR-2. **PR-3b (CoVe→CPU) before PR-5** so the GPU seq budget is unshared at cutover. PR-4 after PR-3. PR-5 is the gate. PR-6 only after a clean soak.

---

## 7. OPEN DECISIONS / RISKS

1. **CoVe placement — SETTLED (distributed).** CoVe (ClaimVerifier + SectorTagger + DigestNarrative) MOVES off the GPU vLLM onto the CPU tier vacated by CoD, so the batched CoD does not starve per-claim verification on the shared 16 seqs. This is the user-approved "don't pile everything on the GPU" direction; it is implemented as PR-3b (before cutover), not deferred to a shadow-contention trigger. No open user call — recorded here only as the topology rationale. (If shadow later shows the CPU tier can't keep up with CoVe demand, the fallback is to split CoVe across both tiers, but that is a tuning escalation, not the default.)
2. **`maxItems` clip on ultra-dense articles.** The 50–60 cap is both the loop-guard AND a hard truncation on a genuinely 70+-fact article (rare). Accepts a small recall hit on the densest tail to kill the loop. RECOMMEND accept (over-extraction is already the system's bias; the tail is rare). **User call:** accept the clip, or add a "dense-article" re-run path above the cap?
3. **`max_num_seqs` headroom.** With CoVe moved to CPU (decision #1), the GPU's 16 seqs are CoD's to use; raising `sentinel_max_concurrent_extractions` toward 16 maximizes throughput against an unshared budget. RECOMMEND start at 8, tune up in shadow against the VRAM/latency report. (Tuning detail, not a blocker — flag only if shadow shows the 700 art/hr needs the full 16.)
4. **Shadow plumbing choice** (existing `ShadowMode` axis vs a thin per-service shadow-runner) — PR-4 implementation detail; supervisor-decidable, no user call needed.

---

## 8. NON-GOALS (explicitly out of scope)
- No change to the verifier (`verifier_v2_3_1`), the adapter, the projector, the digest, or the `:sig:` matrix-feed contract — the whole point is downstream-invariant.
- No LoRA (removed 2026-05-17; base `Qwen2.5-32B-Instruct-AWQ` only).
- No new feature flags beyond the backend value + the shadow toggle (per the no-feature-flags rule; the backend switch IS the toggle and PR-6 removes the old arm).
