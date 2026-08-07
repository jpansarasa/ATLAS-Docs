---
name: deploy
description: ATLAS deployment workflow - smoke tests, PR, health verification via Loki, prompts mount check, VRAM report, image rollback
---

# Deploy Workflow

Applies `VERIFY_TEST.COMPLETION_GATE` and `PROBLEM_SOLVING.EVIDENCE_GATE` from `~/.claude/CLAUDE.md` (neither is in the project CLAUDE.md).

1. Run smoke tests BEFORE declaring done — report pass/fail counts
2. Open PR with test results in description
3. Invoke the intent-review skill on the PR diff (design-intent conformance vs the
   touched services' DECISIONS blocks) — address critical/important findings BEFORE merge
4. After merge, verify deployment health via Loki (errors in last 10 min)
5. Check for volume-mount overrides on /prompts paths
6. Report VRAM/memory status post-deploy
7. List configs/gates touched (no silent workarounds around ansible gates)

## POST_DEPLOY_SMOKE [the named gate for steps 1 and 4]

`ansible-playbook playbooks/smoke-test.yml` — run from `deployment/ansible/`. Read-only
health validation, NOT a deployment: container status, internal service `/api/health`
endpoints via the grafana container, MCP `/health`, Loki error query, GPU + vllm-server.
Sub-tags: `health`, `containers`, `internal`, `mcp`, `logs`, `loki`, `docker`, `gpu`, `database`.

Run it AFTER the deploy completes, before declaring done. Step 4's Loki check narrows the
same window; this playbook is the broad gate, the Loki query is the targeted follow-up.

## ROLLBACK [snapshot before, restore on smoke failure]

Ported from `autofix-watcher.sh` (the auto-deploy timer, disabled 2026-08-07 — the
mechanism was sound, the trigger was not). A human deploy now owns both halves.

Snapshot BEFORE `build.sh`, not before the ansible run. `build.sh` is what overwrites
`:latest` (`sudo nerdctl build -t {svc}:latest .`); the scoped deploy form carries
`--skip-tags build` and builds nothing, so by the time ansible starts `:latest` ALREADY
points at the new image — snapshotting there tags the BROKEN binary as the rollback target.
The rebuild leaves the old image untagged and otherwise unrecoverable.

The watcher could snapshot immediately before its ansible run because it invoked a BARE
`--tags`, which selects the build task (`deploy.yml` build tasks carry `tags: [{svc}, build]`).
That form is forbidden here, so the snapshot had to move earlier.

Per service, in this order:

```
sudo nerdctl image inspect "${svc}:latest" >/dev/null 2>&1 \
  && sudo nerdctl tag "${svc}:latest" "${svc}:autofix-prev"   # 1. snapshot the OLD image
{Project}/.devcontainer/build.sh                               # 2. :latest -> the NEW image
ansible-playbook playbooks/deploy.yml --tags "${svc}" --skip-tags build \
  -e "scoped_restart=true scoped_services=${svc}"              # 3. recreate, no build
ansible-playbook playbooks/smoke-test.yml                      # 4. the gate
```

Tag name inherited from the watcher, so the string stays greppable to its origin.

Only snapshot for a SCOPED deploy naming specific services. A broad all-services deploy is
too large to auto-roll-back — if it fails smoke, escalate to the operator instead.

IF post-deploy smoke FAILS, restore the binaries — do NOT leave a bad deploy live:

```
sudo nerdctl tag "${svc}:autofix-prev" "${svc}:latest"          # per service
sudo nerdctl compose -f /opt/ai-inference/compose.yaml up -d --force-recreate <svcs>
ansible-playbook playbooks/smoke-test.yml                        # re-run to confirm
```

Then report: rolled back + post-rollback smoke result, or "rollback INCOMPLETE — prod may
be in a bad state, manual action NOW" if the restore or the re-run failed.

The git merge stays on main — roll back the live binary ONLY. A human fixes forward or
reverts the commit; silently reverting main hides that the merge happened.

## ANTI [HARD_STOP]

✗ run `build.sh` before taking the image snapshot # build.sh overwrites :latest; snapshotting after it tags the NEW image as the rollback target, so the restore reinstates the broken binary
✗ snapshot "before the ansible run" # the scoped form skips build — :latest is already new by then; that timing was the watcher's, and the watcher ran a bare --tags
✗ bare `--tags {anything}` # unconditional full-stack restart incl a ~4min vLLM reload, see CLAUDE.md DEPLOYMENT
✗ revert main to undo a bad deploy # roll back the image, let a human decide the git action
✗ re-enable `autofix-watcher.timer` to get auto-deploy back # ansible enforces disabled; the trigger is the failure mode, not the mechanism
