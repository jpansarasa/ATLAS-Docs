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
