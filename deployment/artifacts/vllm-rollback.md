# vllm-server compose migration — cutover + rollback

2026-06-11: vllm-server moved from a standalone `nerdctl run` container into
`compose.yaml` (service `vllm-server`). This file records the exact pre-migration
launch command (rollback path) and the cutover procedure.

## Cutover (supervisor-scheduled — pauses GPU inference)

```bash
cd /home/james/ATLAS/deployment/ansible
ansible-playbook playbooks/deploy.yml --tags vllm-server
```

Ordering inside that run (all automatic):
1. `compose.yaml` re-rendered with the new `vllm-server` service (`[always]`).
2. Digest-pinned image ref ensured in the local store (`[always]` pre-pull task —
   nerdctl only registers `name@digest` when pulled BY digest; near-instant, blobs
   already local). The compose up never depends on Docker Hub mid-cutover.
3. Legacy standalone container detected by missing `com.docker.compose.project`
   label and stopped+removed (`[always]`, gated `not scoped_restart`) — GPU
   inference DOWN from here.
4. `compose_file.changed` ⇒ `systemctl restart atlas` (full compose down/up, gated
   `when: not (scoped_restart | bool)`) brings up vllm-server as a compose service.
   NOTE: the whole main stack cycles (~1 min). Scoped runs bypass this task entirely.
5. The `[vllm-server]` block skips its recreate when: (a) legacy cleanup registered
   changed (step 4 already started it fresh), (b) systemd restart registered changed
   (compose down/up cycled it), or (c) any scoped_restart=true run — the recreate is excluded from all scoped runs; the scoped-restart task owns rm/up when vllm-server is explicitly named.
   Then waits on `/health` (60×10 s) + runs a 1-token `/v1/completions` smoke.

Cutover must be a plain `--tags vllm-server` run, NOT `scoped_restart=true`:
the scoped path skips both the legacy cleanup and the full-stack restart, so a
pre-cutover scoped run simply leaves the legacy container serving (harmless,
cutover deferred to the next plain run).

Expected GPU inference downtime: ~3.5-4 min (weights ~110 s + engine init/CUDA
graphs ~60 s + container churn). Consumers that degrade during the window:
sentinel-collector extraction/classification, reports-* narrative.

## Rollback (re-create the standalone container)

**SINCE 2026-09-07 THIS BLOCK ROLLS BACK TWO THINGS AT ONCE, and that is the point.**
It is the pre-compose-migration launch, which is also the pre-Gemma-4 one: Qwen2.5-32B-AWQ
on the 0.19-era image at 16 seqs / 0.92 util / `fp8_e5m2`. Running it does NOT merely undo
the container topology — it puts the INCUMBENT model back, at numbers_f1 0.5131 against
Gemma 4's 0.7346 (SentinelCollector/AGENT_README.md D-29). Every flag below belongs to that
coordinate and none may be mixed with the current one: `--quantization awq_marlin` belongs to
the Qwen AWQ checkpoint, whose `config.json` declares `awq` — on the compressed-tensors Gemma
checkpoint it forces a method the checkpoint does not declare, and what 0.28.0 does with that
mismatch is UNESTABLISHED (a 0.28.0 control on the Qwen checkpoint had the flag REWRITTEN to
`auto_awq` rather than refused, so "it would fail the boot" is not a claim any artifact supports;
correct 2026-09-07). `fp8_e5m2` is correct on 0.19 and faults under concurrent decode on 0.28.0.
Roll back the WHOLE block or none of it.

**SINCE 2026-09-07 THE COMPOSE PATH BOOTS WITH `HF_HUB_OFFLINE=1`** (D-29: it reproduces the
acceptance boots, which all ran offline). A model rollback through compose therefore REQUIRES the
target's weights already in `/opt/ai-inference/models/huggingface-cache` — it will not fetch. The
incumbent is staged: 18G at `refs/main` = `5c7cb76a268fc6cfbb9c4777eb24ba6e27f9ee6c`, the revision
the acceptance run served, so this rollback resolves from cache. Re-check before relying on it:
`cat /opt/ai-inference/models/huggingface-cache/hub/models--Qwen--Qwen2.5-32B-Instruct-AWQ/refs/main`.
The manual `nerdctl run` below sets no such variable and is the escape hatch if that check fails.

To roll back only the engine while keeping Gemma 4, do not use this block: revert
`vllm_image` in `deployment/ansible/group_vars/all.yml` and redeploy — one variable, and
note that Gemma 4 is NOT servable on 0.19.0 at all, so that path lands nowhere.

**THE MODEL ROLLBACK IS NOT ENGINE-ONLY EITHER, in exactly the way the forward swap is not.**
Putting Qwen back on the engine — by this block or by reverting `vllm_base_model` — leaves
`sentinel-collector:latest` still carrying Gemma 4's chat template and stop tokens, because
`Extraction__ChatTemplate` / `Extraction__StopTokens` are baked in from `src/appsettings.json`
with no bind mount and no compose env override. A restarted collector would then send a Gemma
template to a Qwen engine: still valid JSON, still green dashboards, an arm nobody scored. The
Reports model id is a C# literal and 400s to a heuristic fallback the same way. So a model
rollback in either direction means rebuilding `sentinel-collector` and the three `reports-*`
images and RECREATING those containers — the rebuild moves the template, the recreate moves
the templated `Extraction__Model` env. A scoped `vllm-server` restart does neither.

If the compose service misbehaves, revert this commit + redeploy, or manually:

```bash
cd /opt/ai-inference
sudo nerdctl compose rm -sf vllm-server
sudo nerdctl run -d --gpus all \
  --name vllm-server \
  --network ai-inference \
  -v /opt/ai-inference/models/huggingface-cache:/root/.cache/huggingface \
  -e HF_HOME=/root/.cache/huggingface \
  -p 8000:8000 \
  --restart unless-stopped \
  vllm/vllm-openai@sha256:d9a5c1c1614c959fde8d2a4d68449db184572528a6055afdd0caf1e66fb51504 \
  --model Qwen/Qwen2.5-32B-Instruct-AWQ \
  --quantization awq_marlin \
  --max-model-len 32768 \
  --max-num-seqs 16 \
  --gpu-memory-utilization 0.92 \
  --kv-cache-dtype fp8_e5m2 \
  --generation-config vllm \
  --enable-auto-tool-choice \
  --tool-call-parser hermes
```

**`--kv-cache-dtype fp8_e5m2` above is bound to the digest pinned in this block** (the
2026-04 / vLLM 0.19-era image), and is correct only there. If you ever re-target this
rollback at a newer vLLM, the flag must become `fp8_e4m3`: measured 2026-09-04, e5m2 on
0.28.0 serves one request and then faults under concurrent decode (CUDA illegal memory
access) and stays 503. A sequential smoke test cannot see it — the failure needs
concurrency >= 2. e4m3 is the one-flag fix and its quality cost is measured null.
See CLAUDE.md `VLLM_UPGRADE`.

(Identical to the captured pre-migration config — `sudo nerdctl inspect vllm-server`
2026-06-11 — except the image is digest-pinned here; the original used the floating
`:latest` tag, which resolved to this manifest digest `sha256:d9a5c1c1614c…`.
The actual image ID — the config blob — is `sha256:22bea33788…`; nerdctl's
IMAGE ID column misleadingly shows the truncated digest.)

The digest-pinned `nerdctl run` above needs the `name@digest` ref in the local
store (pulled by digest — primed on the host 2026-06-11 and re-ensured by the
deploy pre-pull task) or Docker Hub reachable. If the digest ref is somehow
unavailable, fall back to `vllm/vllm-openai:latest` — but first verify it still
resolves to image ID `22bea33788…` (`nerdctl image inspect --format '{{.ID}}'`),
since :latest floats.

After a manual rollback, `compose.yaml` still contains the `vllm-server` service:
the next full-stack restart would collide on the container name. Treat manual
rollback as a stopgap; revert the migration commit and redeploy promptly.
