# Ansible deploy.yml — Tag Gating Audit

**Date:** 2026-05-14
**2026-06-11 update:** vllm-server has since moved INTO compose.yaml (GPU via
`deploy.resources.reservations.devices`, digest-pinned image). The tag-gating fix
below still stands — the deploy block (now `compose rm -sf` + `up -d` + health +
smoke) remains `[vllm-server]`-only. New consequence: full-stack restarts
(`systemctl restart atlas` = compose down/up) cycle vLLM, ~3.5-4 min of GPU model
reload.

**2026-08-06 correction — the restart is UNCONDITIONAL, not change-gated.** The
paragraph above previously said the systemd restart fires "only when
`compose_file` or one of the `*_build` registers changed", and that it is gated
by those conditions "not by `[always]` unconditionally". That reading is wrong,
and it is the reading that talks an operator out of CLAUDE.md's DEPLOYMENT
warning. The register it depends on can never be unchanged:

- `deploy.yml:442` **deletes** `compose.yaml` — `state: absent`, `tags: [always]`,
  no `when:`.
- `deploy.yml:448` re-templates it and `register: compose_file`. Because the file
  was just deleted, the template always creates it, so `compose_file.changed` is
  **always true**.
- `deploy.yml:1278` — the systemd `state:` expression ORs ~23 terms, and its
  *first* term is `(compose_file | default({})).changed`. It therefore always
  resolves to `restarted`, never `started`.

So every non-scoped invocation is a full-stack restart regardless of what (if
anything) was rebuilt. `when: not (scoped_restart | bool)` on that task remains
the one real gate, alongside `--skip-tags always`. Verified via
`ansible-playbook --list-tasks` (ansible-core 2.16.3), which resolves tag
selection without executing: `--tags patterns` selects "Start or restart ATLAS
infrastructure"; `--tags patterns --skip-tags always` selects only the three
pattern tasks.

The standalone-container legacy-cleanup task (one-time cutover guard) does gate
on `not (scoped_restart | bool)`, so a scoped run never stops the serving
container. That part stands.
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
*(This bullet was superseded 2026-06-11: vllm-server is now a compose service — see the update section. Any full-stack restart cycles it, and per the 2026-08-06 correction every non-scoped run is one; only `-e scoped_restart=true` or `--skip-tags always` avoids it.)*
### Notes / non-issues

- The `Start or restart ATLAS infrastructure` task (now L1275) is `[always]`
  and restarts the whole compose stack on **every** non-scoped invocation —
  see the 2026-08-06 correction at the top. This bullet previously read "when
  any build registered `.changed` … if it changed, the whole compose stack
  does restart", which wrongly implies a selective deploy that rebuilt nothing
  leaves the stack alone. It does not: `compose_file` is deleted and
  re-templated under `tags: [always]`, so the restart condition is always
  satisfied. The blast radius itself is still the documented and accepted one
  for application services — those containers share
  secmaster/threshold-engine/etc. cross-deps and the unit is the supervision
  boundary — it is simply unconditional rather than change-gated. Out of scope
  for this audit; not the bug that bit us in 2026-05.
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
