# Ansible deploy.yml — Tag Gating Audit

**Date:** 2026-05-14
**2026-06-11 update:** vllm-server has since moved INTO compose.yaml (GPU via
`deploy.resources.reservations.devices`, digest-pinned image). The tag-gating fix
below still stands — the deploy block (now `compose rm -sf` + `up -d` + health +
smoke) remains `[vllm-server]`-only. New consequence: full-stack restarts
(`systemctl restart atlas` = compose down/up) cycle vLLM only when `compose_file`
or one of the `*_build` registers changed AND the run is not scoped (`when: not
(scoped_restart | bool)`). The standalone-container legacy-cleanup task (one-time
cutover guard) also gates on `not (scoped_restart | bool)`, so a scoped run never
stops the serving container. In summary: a full-stack restart bounces vLLM only
when compose_file or a build changed; the systemd restart is itself gated by those
conditions, not by `[always]` unconditionally.
**Trigger:** vLLM outage caused by `ansible-playbook playbooks/deploy.yml --tags secmaster,sentinel-collector,spacy-ner`.
**Root cause:** vllm-server deploy block tagged `[always, vllm-server]`.

## Incident replay

`--tags secmaster,sentinel-collector,spacy-ner` cycled `vllm-server`. The
container's recreate hit an NVIDIA driver kernel/userspace mismatch
(kernel 580.142 / userspace 580.126.09) and vLLM was down for the rest of
the day.

In Ansible, the special `always` tag matches every tag-scoped run unless
the user passes `--skip-tags always`. The vllm-server block (originally
intended to be opt-in via `--tags vllm-server`) instead ran on every
selective deploy. Its first step is `nerdctl stop vllm-server`, followed
by `nerdctl rm` and a fresh `nerdctl run -d --gpus all ...`. That is the
exact cascade that surprised the operator.

## Findings

### Bug (fixed)

| Block / task | Pre-fix tags | Problem |
|--------------|-------------|---------|
| `Deploy vllm-server with GPU passthrough` (block, ~L836) | `[always, vllm-server]` | `always` makes the stop/remove/recreate run on every `--tags …` invocation. Caused the 2026-05-14 outage. |

### Fix

Removed `always`; left `vllm-server` only. Full no-tag deploys still
include the block (Ansible runs every task when no `--tags` filter is
present). Tag-scoped deploys now must pass `--tags vllm-server`
explicitly to touch vLLM. Inline comment added documenting the
incident so the next reader doesn't "helpfully" re-add `always`.

### Verified-correct tag declarations (no change needed)

- **Service identity / containerd setup** (L69–L116): `[always]` — required infra (uid/gid, sock perms). Idempotent file/group/user modules; do not restart vllm-server.
- **OTEL stack** (L147–L198): `[otel, monitoring]` consistently — not pulled into selective deploys.
*(This bullet was superseded 2026-06-11: vllm-server is now a compose service — see the update section. Full-stack restarts triggered by compose/template or image-rebuild changes cycle it; tag-scoped runs do not.)*
### Notes / non-issues

- The `Start or restart ATLAS infrastructure` task (L751) is `[always]` and
  restarts the whole compose stack when any build registered `.changed`.
  For a selective deploy like `--tags secmaster`, only `secmaster_build`
  fires; if it changed, the whole compose stack does restart. This is the
  documented (and accepted) blast radius for application services — those
  containers share secmaster/threshold-engine/etc. cross-deps and the
  unit is the supervision boundary. Out of scope for this audit; not the
  bug that bit us today.
- The `Setup atlas_secmaster database / pg_trgm` tasks (L897–L917) carry
  no tags, so they only run on no-filter deploys. That's fine — they
  query the DB and short-circuit if the database exists. Not a tag-gating
  bug, just lower priority than vllm.
- Handler-style `notify:` is not used anywhere in this playbook, so no
  handler-cascade audit is needed.

## Verification

`ansible-playbook deploy.yml --tags secmaster,sentinel-collector,spacy-ner --list-tasks`
**after** the fix: no `vllm-server` tasks appear.

`ansible-playbook deploy.yml --tags vllm-server --list-tasks` **after** the
fix: the six vllm-server tasks appear (plus `always` infrastructure setup,
which is by design).

`ansible-playbook deploy.yml --list-tasks` (no filter): vllm-server tasks
still appear — full deploys continue to (re)create vllm-server.

## Files touched

- `deployment/ansible/playbooks/deploy.yml` — one block's `tags:` changed
  from `[always, vllm-server]` to `[vllm-server]`, with an inline comment
  citing this audit.
