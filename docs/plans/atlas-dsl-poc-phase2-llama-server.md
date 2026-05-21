# DSL PoC — Phase 2 — `llama-server` sibling deployment (option B2)

**Status:** scoping. No code, no deploy. Single source of truth for Phase 2
infrastructure work until split into branches and merged.
**Parent plan:** `docs/plans/atlas-dsl-poc-plan.md` §5.
**Topology rule (immovable, PR #386):** CPU = extraction (this plan).
GPU = summarization + verification (Phase 4, out of scope here).
**Decision context (STATE.md ACTIVE):** ollama 0.24 silently drops raw GBNF
via `format` — empirically verified. JSON-schema works but loses GBNF
expressiveness. User picked option **B2**: deploy llama.cpp's `llama-server`
as a new sibling service on the same CPU host as `ollama-cpu-gen`, keeping
`ollama-cpu-gen` untouched for everything else (sentinel-cod-v6, qwen2.5:7b,
gemma4, etc.).

---

## Goal

Stand up a llama.cpp `llama-server` container alongside `ollama-cpu-gen`
that serves `qwen3:30b-a3b-instruct-2507-q4_K_M` with native GBNF
grammar-constrained decoding, reusing the **existing** ~18 GB GGUF blob
already on disk under ollama's content-addressed store. The Phase 2
benchmark runner (`docs/benchmarks/cod-2026-05-17/scripts/`) gains a third
backend target and emits results into the same per-`(model, variant,
article)` schema so `aggregate_report.py` scores Phase 2 cells without
modification. Plan §5.3 acceptance gates apply: `grammar_valid = true` on
100% of outputs, numeric_recall not worse than the current 91.7%, latency
overhead < 30% vs free decoding on `ollama-cpu-gen`.

## Non-goal

This plan does **not** retire `ollama-cpu-gen`, change ollama's role for any
other model (sentinel-cod-v6, qwen2.5:7b, embeddings), or move extraction
off CPU. It does not touch the GPU path (`vllm-server`,
sentinel-cove-v6.2), Phase 3 (v2 schema), Phase 4 (GPU verifier), or
production prompts at `/opt/ai-inference/prompts/`. It does not build a
custom llama.cpp image — upstream `ghcr.io/ggml-org/llama.cpp:server` is
the default. It does not introduce a new model file: the ollama-managed
GGUF blob is bind-mounted read-only into the new container; no 18 GB
re-download.

---

## Recon (load-bearing facts)

Captured here so each story can reference rather than re-derive.

| Fact | Value |
|---|---|
| Compose template | `deployment/artifacts/compose.yaml.j2` (Ansible `template:` → `/opt/ai-inference/compose.yaml`; deploy.yml task at line 209) |
| Group vars / ports | `deployment/ansible/group_vars/all.yml` (`ports_external`, `ports_internal`) |
| `ollama-cpu-gen` image | `ollama/ollama:0.24.0`, host port `11435` → container `11434`, 28 CPUs / 80 GB RAM cap, volume `/opt/ai-inference/models:/root/.ollama` |
| Model manifest path | `/opt/ai-inference/models/models/manifests/registry.ollama.ai/library/qwen3/30b-a3b-instruct-2507-q4_K_M` |
| Model blob path | `/opt/ai-inference/models/models/blobs/sha256-78b329e716e7e9775973d392cd132b1f1ff1c8287a992887caeb6fd6c56ba9cc` (18,556,685,856 bytes; `file` reports `data`, magic bytes confirm `GGUF`) |
| v1 GBNF | `docs/grammars/cod-dsl-v1.gbnf` (163 lines, ready) |
| Existing runner | `docs/benchmarks/cod-2026-05-17/scripts/run_cod_dsl.py` (uses `OLLAMA_URL`, default `http://localhost:11435`, posts to `/api/generate`) |
| Host listening ports | 22, 53, 139, 445, 1338, 3000, 3102, 3109, 3551, 5432, 8000, 8090, 9144, 9200, 9300, 11435, 11436. **Port 8080 free on host**; container-side `8080` is already in use by every internal HTTP service but those are not host-mapped. |
| Tagging convention | Per-service tag, e.g. `tags: [ollama, ollama-cpu-gen]`. Compose template re-rendered on every run (`tags: [always]`), so adding a service entry to the template is automatic on any `deploy.yml` invocation. |
| Container runtime | `nerdctl` via containerd; `nerdctl compose up -d <svc>` is the idempotent upgrade pattern (deploy.yml lines 975-1001 for the ollama-cpu-gen analogue). |

**Image choice.** Default to `ghcr.io/ggml-org/llama.cpp:server` (upstream
CPU image; serves on `:8080` by default). Reasons: maintained by the
project, no monorepo build context, already in use elsewhere on host
(`/opt/ai-inference/llama.cpp/` is a checked-out source tree —
informational; we don't need to build from it).

**Model mount strategy.** Bind-mount the ollama blobs directory read-only
and point `llama-server` at the blob path directly. Ollama's storage is
sha256-content-addressed and the blob *is* a GGUF (verified magic bytes
above). One bind + one `--model` flag, no copy, no re-download. The
manifest path is informational only — `llama-server` doesn't read ollama
manifests; we resolve the digest once during recon (already done above)
and hard-code the path in the compose entry. The hard-coded digest is the
**accepted coupling**: if the qwen3 model is ever re-pulled with a new
digest, this compose entry will silently load the stale blob. Documented
in story 3's acceptance.

**Host port choice.** `11437` for the host-side mapping. Adjacent to
`ollama-cpu-gen` (11435) and `ollama-cpu-embed` (11436), already free,
keeps the "CPU LLM runners cluster in 1143x" convention legible. Container
listens on its default `8080`; we map `11437:8080`. Add
`ports_external.llama_server: 11437` to `group_vars/all.yml`.

**Resource budget.** `ollama-cpu-gen` caps at 28 CPUs / 80 GB; the host
has more headroom but both runners are intended to be active during
Phase 2 A/B (free-decoder on ollama vs grammar-constrained on llama-server
against the same article corpus). Cap `llama-server` at 16 CPUs / 40 GB
for the q4_K_M 30B model — enough for the model itself (~18 GB on disk,
~22 GB resident with KV cache at 16 K context, two concurrent slots).
Both runners can be active without saturating the box; if they do
compete, sequence the A/B run rather than parallelizing it.

---

## Story breakdown

One story per layer. Each names files touched, acceptance criteria, and
explicit `git add` paths. Stories are independent commits within a single
PR unless flagged otherwise.

### Story 1 — Reserve port + service URL var

**Files**
- `deployment/ansible/group_vars/all.yml`

**Change**
- Under `ports_external`, add `llama_server: 11437`.
- Add a top-level `llama_server_url: http://llama-server:8080` mirroring
  the `ollama_cpu_gen_url` convention. (Used by Story 5's runner env.)

**Acceptance**
- `ansible-playbook --syntax-check playbooks/deploy.yml` passes.
- `ports_external.llama_server` resolvable in the template (Story 2 uses it).

**git add**
- `git add -- deployment/ansible/group_vars/all.yml`

---

### Story 2 — Compose service entry

**Files**
- `deployment/artifacts/compose.yaml.j2`

**Change**
Add a new service block, sibling to `ollama-cpu-gen`. Skeleton (final
content authored against the template's Jinja conventions, not committed
in this scoping doc):

```yaml
  llama-server:
    image: ghcr.io/ggml-org/llama.cpp:server
    container_name: llama-server
    command:
      - "--model"
      - "/models/qwen3-30b-a3b-instruct-2507-q4_K_M.gguf"
      - "--host"
      - "0.0.0.0"
      - "--port"
      - "8080"
      - "--ctx-size"
      - "16384"
      - "--threads"
      - "16"
      - "--parallel"
      - "1"
    ports:
      - "{{ ports_external.llama_server }}:8080"
    volumes:
      # Read-only mount of the ollama-managed GGUF blob. Digest is
      # pinned to the current qwen3:30b-a3b-instruct-2507-q4_K_M
      # manifest (resolved 2026-05-21). If the model is re-pulled and
      # the digest changes, this mount goes stale silently — Story 4
      # smoke-test must catch a model-identity mismatch.
      - /opt/ai-inference/models/models/blobs/sha256-78b329e716e7e9775973d392cd132b1f1ff1c8287a992887caeb6fd6c56ba9cc:/models/qwen3-30b-a3b-instruct-2507-q4_K_M.gguf:ro
    restart: unless-stopped
    deploy:
      resources:
        limits:
          memory: 40G
          cpus: '16'
```

**Acceptance**
- `nerdctl compose -f /opt/ai-inference/compose.yaml config` (post-render)
  exits 0 and shows the new service.
- `ansible-playbook playbooks/deploy.yml --tags llama-server --check` is
  a no-op rerun after first apply.

**git add**
- `git add -- deployment/artifacts/compose.yaml.j2`

**Note** — the template is re-rendered on every `deploy.yml` run (tag
`always`, deploy.yml line 217). No extra Ansible task is needed to *land*
the service entry. Story 3 adds an explicit upgrade-and-verify block so
that `--tags llama-server` is a meaningful selective deploy.

---

### Story 3 — Ansible upgrade-and-verify block

**Files**
- `deployment/ansible/playbooks/deploy.yml`

**Change**
Mirror the existing `ollama-cpu-gen` upgrade block (deploy.yml lines
975-1001). Add a new block, tagged `[llama-server]`:

1. `nerdctl pull ghcr.io/ggml-org/llama.cpp:server`
2. `nerdctl compose up -d llama-server` (chdir `/opt/ai-inference`)
3. Wait/retry on `curl -fsS http://localhost:11437/health` until 200,
   max 12 retries × 10 s (model load on cold start is ~30-90 s on CPU
   for an 18 GB GGUF).
4. Debug-print the `/props` endpoint's `default_generation_settings` so
   the deploy log records which model the server actually loaded —
   the smoke check against the manifest digest lives in Story 4.

**Acceptance**
- `ansible-playbook playbooks/deploy.yml --tags llama-server` brings the
  container up from cold in under 3 minutes wall.
- Idempotent rerun is a no-op (no "changed" tasks on second run).

**git add**
- `git add -- deployment/ansible/playbooks/deploy.yml`

---

### Story 4 — Smoke test

**Files**
- `deployment/ansible/playbooks/smoke-test.yml`

**Change**
Add to `smoke-test.yml` (tag `[health, llama-server]`):

1. **Liveness:** `GET http://localhost:11437/health` returns 200 with
   `{"status":"ok"}` (llama.cpp server idiom).
2. **Model identity:** `GET http://localhost:11437/props` returns a
   JSON body whose `default_generation_settings.model` path contains the
   pinned digest substring (`sha256-78b329e7...` or the basename
   `qwen3-30b-a3b-instruct-2507-q4_K_M.gguf`). This is the guard against
   the "ollama re-pulled, digest drifted, stale blob loaded" failure mode
   flagged in Story 2.
3. **GBNF smoke:** `POST http://localhost:11437/completion` with prompt
   `"Output a single letter A or B: "`, grammar `root ::= "A" | "B"`,
   `n_predict: 1`. Assert response body's `content` is exactly `"A"` or
   `"B"` (proves the grammar pipeline is wired end-to-end, independent of
   model quality).

**Acceptance**
- All three checks pass on `ansible-playbook playbooks/smoke-test.yml
  --tags llama-server`.
- Smoke runs unattended in `deploy.yml`'s normal flow (no extra invocation
  needed by an operator).

**git add**
- `git add -- deployment/ansible/playbooks/smoke-test.yml`

---

### Story 5 — Benchmark runner backend wiring

**Files**
- `docs/benchmarks/cod-2026-05-17/scripts/run_cod_dsl.py`

**Change**
Generalize the runner from "ollama-only" to "ollama or llama-server",
selected by a `--backend {ollama,llama-server}` flag. Default stays
`ollama` so all existing scripts (`run_phase1_*.sh`, `run_round*.sh`)
remain byte-identical in behavior.

- New env: `LLAMA_SERVER_URL` (default `http://localhost:11437`).
- New helper `_llama_server_completion(...)` parallel to `_ollama_generate`,
  posting to `/completion` with `grammar: <gbnf_text>` when a grammar
  file is supplied via `--grammar docs/grammars/cod-dsl-v1.gbnf`.
- Result schema unchanged. Fields that don't exist on the llama-server
  response (`eval_count`, `prompt_eval_count`, `total_duration`) get
  mapped from llama-server's equivalents (`tokens_evaluated`,
  `tokens_predicted`, `timings.predicted_ms`) so
  `aggregate_report.py` keeps working without modification. The result
  record gains one new optional field `backend: "ollama" | "llama-server"`
  for traceability.

**Acceptance**
- `python run_cod_dsl.py --backend ollama --model qwen3:30b-a3b-instruct-2507-q4_K_M --variant spec --article-id <X>`
  produces a result file byte-identical to a current run (modulo
  timestamp + the new `backend` field).
- `python run_cod_dsl.py --backend llama-server --grammar docs/grammars/cod-dsl-v1.gbnf --variant spec --article-id <X>`
  produces a result file with `grammar_valid: true` on a sample of n=3
  articles. (Phase 2 acceptance §5.3 will sweep n=10+.)
- `pytest docs/benchmarks/cod-2026-05-17/scripts/tests/` passes (existing
  tests must not regress; add one test exercising the llama-server
  payload-shape mapping with a `responses`-mocked transport).

**git add**
- `git add -- docs/benchmarks/cod-2026-05-17/scripts/run_cod_dsl.py`
- `git add -- docs/benchmarks/cod-2026-05-17/scripts/tests/` (only new
  or modified files in this directory)

---

### Story 6 — v1 GBNF compatibility validation against llama.cpp

**Files**
- `docs/grammars/cod-dsl-v1.gbnf` (only if changes needed)
- `docs/grammars/cod-dsl-v1.notes.md` (new — only if material findings)

**Change**
The v1 GBNF was authored against the GBNF spec but never executed against
llama.cpp's parser. This story is a validation pass, not a rewrite:

1. Run `/opt/ai-inference/llama.cpp/tools/gbnf-validator` (or the
   equivalent host-side binary; the source tree at
   `/opt/ai-inference/llama.cpp/` already contains it) against the v1
   GBNF on a sample of known-good outputs from the Phase 1 corpus.
2. If the validator rejects, capture the exact error and make the
   smallest local change that unblocks it. Document any change in a
   commit message + a one-paragraph note at the top of the GBNF file
   (rationale, not mechanics).
3. If no changes are needed, no commit on this story — close it as
   "validated, no change required."

**Acceptance**
- Validator accepts the grammar.
- A known-good Phase 1 output (one of the `gv%=100%` results) parses
  cleanly under llama-server's grammar enforcement when fed back as a
  forced-completion seed.

**git add** (only if changes required)
- `git add -- docs/grammars/cod-dsl-v1.gbnf`
- `git add -- docs/grammars/cod-dsl-v1.notes.md`

---

## Rollback path

Phase 2 results land in a few days. If the data says grammar-constrained
decoding doesn't beat the current free-decoder baseline, llama-server is
removed cleanly:

1. **Drop the service.** Delete the `llama-server:` block from
   `deployment/artifacts/compose.yaml.j2` and the upgrade-and-verify
   block from `deploy.yml`. `ansible-playbook playbooks/deploy.yml`
   re-renders the compose file without the service; `nerdctl compose up
   -d --remove-orphans` (already part of deploy.yml's normal flow if
   present, else added as a one-liner) tears down the now-orphaned
   container.
2. **Reclaim port.** Remove `llama_server: 11437` and `llama_server_url`
   from `group_vars/all.yml`.
3. **Drop smoke test.** Remove the `[llama-server]` block from
   `smoke-test.yml`.
4. **Image cleanup.** `sudo nerdctl rmi ghcr.io/ggml-org/llama.cpp:server`
   on mercury (manual, one-shot).
5. **Runner stays.** `run_cod_dsl.py`'s `--backend llama-server` flag is
   harmless if the server isn't running (caller error if invoked).
   Leave it in place for any future revisit.
6. **Model blob stays.** The mounted GGUF is ollama's blob; ollama still
   owns it. No file deletion needed.

ZFS snapshot from the pre-deploy task (deploy.yml line 23) provides the
nuclear-option rollback if the cleanup itself goes wrong.

---

## Open questions (for supervisor decision)

Do not decide these in-line in implementation PRs. Surface them.

1. **Context size.** `ollama-cpu-gen` runs the same model at 16 K (the
   benchmark runner default). Phase 2's grammar-constrained arm doesn't
   need more, but the v2 schema (Phase 3) introduces source spans that
   may inflate input. Lock at 16 K for Phase 2 and revisit before
   Phase 3, or stretch to 32 K now for headroom?
2. **Parallel slots.** `--parallel 1` matches `ollama-cpu-gen`'s
   `NUM_PARALLEL=1` posture (and the SENTINEL rule in CLAUDE.md). The
   Phase 2 benchmark sweep is sequential, so this is fine. If a later
   phase wants concurrent extraction over a large corpus, slot count is
   the tunable; document the floor here as 1 and revisit then.
3. **A/B contention.** Both runners active simultaneously on 28 + 16
   pinned CPUs leaves headroom on a typical 64-thread box but no hard
   guarantee against scheduler contention degrading p50 latency. Default
   posture: sequence the A/B (free-decoder ollama, then
   grammar-constrained llama-server) rather than parallelize. Confirm
   before Story 5's first multi-cell sweep.
4. **Blob-digest pinning.** Story 4's smoke test catches a digest
   mismatch but doesn't prevent it. Should the deploy.yml upgrade block
   also re-resolve the digest from the ollama manifest and rewrite the
   compose mount path? Trades a moving-piece-in-Ansible for a
   silent-staleness risk. Conservative default for this PoC: hard-coded
   digest + smoke-test guard. Revisit if ollama model upgrades become
   routine.
5. **systemd integration.** `ollama-cpu-gen` is reached by the existing
   `atlas.service` unit's `nerdctl compose up` flow. Confirm
   `llama-server` is started by the same path (it should be — it's
   another service in the same compose file) before declaring Story 3
   acceptance.

---

## STATE.md one-liner

`Phase 2 scoping landed: docs/plans/atlas-dsl-poc-phase2-llama-server.md on feat/dsl-poc/phase2-scoping (6 stories: ports, compose entry, ansible upgrade block, smoke test, runner backend wiring, GBNF validation; rollback + open questions enumerated).`
