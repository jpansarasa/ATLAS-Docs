# Template: deploy to production

Shipping already-reviewed, already-merged work to production — one service or a batch, plus the
verify that proves it.
mercury IS this host — never ssh, never ansible against a remote. Pairs with the `deploy` skill
(the COMPLETION_GATE checklist); this carries the moves and the traps.

```
DEPLOY — {what}, merged to main as `{sha}`. User has authorised this deploy{, including <named
side effect>}.

DESIGN INTENT: none — mechanical deployment of already-reviewed, already-merged work.
supersedes: none
guard_tests: none

No interactive commands (`-it`, `--ask-vault-pass`, y/n prompts); sudo is passwordless.
If anything fails, STOP and report — do not improvise, do not roll back, do not restart the stack.

TRAJECTORY
1. `git -C /home/james/ATLAS checkout main && git -C /home/james/ATLAS pull --ff-only`; confirm
   HEAD is `{sha}`. `-C` on BOTH — it binds one command, so the second would run in your cwd.
2. Rollback floor: `ansible-playbook playbooks/zfs-snapshot.yml` from `deployment/ansible/`
   (`inventory/hosts.yml` is the ansible.cfg default). ZFS is the disaster floor, not a
   per-service undo — the dataset holds both databases, so a rollback discards unrelated writes.
3. CLASSIFY THE UNIT FIRST: container (image build + compose) or HOST systemd unit?
   `gemini-resolver-mcp` is a host unit with a venv and no image, so an image build for it is
   wasted work. Verify against `/opt/ai-inference/compose.yaml` and `systemctl list-units`.
4. Container path: `{Service}/.devcontainer/build.sh --no-cache`. Confirm the IMAGE changed via
   `nerdctl inspect --type image {image}` `.Created` — not `CreatedSince` (lies about reused
   layers), and never a bare `nerdctl inspect {name}`, which returns the IMAGE when a container
   shares the name. Record the container's `.Id` (`--type container {svc}`) for step 5.
5. Scoped deploy, from `deployment/ansible/`:
   `ansible-playbook playbooks/deploy.yml --tags {tag} --skip-tags build -e
   "scoped_restart=true scoped_services={svc}"`. The `-e` is load-bearing: every restart path in
   the play is `tags: [always]`, so tags cannot steer between them — only the `when:` can. `{svc}`
   is the compose service name, not always the ansible tag.
   - bare `--tags {tag}` -> `systemctl restart atlas` = compose down/up of EVERY service incl a
     ~3.5-4 min vLLM GPU reload, and it resurrects a deliberately-stopped alert-service.
   - `--skip-tags always` -> skips the compose re-render, BOTH scoped paths and the freshness
     gate; the play exits 0 having changed nothing. Correct for `--tags dashboards` only.
   THEN CONFIRM THE CONTAINER WAS REPLACED, not just the image: `.Id` differs from step 4's,
   `.Created` post-dates the deploy, `.State.Status` is `running`. A fresh image behind an
   unchanged container is the silent failure — step 7 then smokes the OLD code.
6. Prometheus: `alerts/` is a DIRECTORY mount, so HUP suffices; `prometheus.yml` is a SINGLE-FILE
   mount that pins the old inode after an ansible copy and needs a container restart.
   Syntax-check what landed (`sudo nerdctl exec prometheus promtool check rules
   /etc/prometheus/alerts/{file}.yml`), then confirm the rules are in the LOADED ruleset via the
   Prometheus API — on disk is not loaded.
7. Smoke: `ansible-playbook playbooks/smoke-test.yml`, plus each service's `/health` and its MCP
   `health` tool. `nerdctl` 1.7.7 discards depends_on and healthchecks, so services race
   TimescaleDB — check the boot banner, not just "container Up".
8. Loki errors in the 10 min after EACH restart, anchored to actual `date -u`; ground truth is
   `severity_text` (query form in recon-measurement.md).
9. PROVE THE FIX IN PRODUCTION, bounded: exercise the real work path. A 200 from a health
   endpoint is not evidence the fix works.

FAILURE MODES SEEN HERE -> THE CHECK
- Stale image deployed silently. Check: `--no-cache`, the image `.Created`, AND the container
  `.Id` — image freshness is not container freshness.
- A deploy that deploys nothing and reports success. Check: step 5's `.Id` comparison.
  `--skip-tags always` produces exactly this — green play, untouched containers, passing smokes.
- Cascade: a restart takes the whole stack down. Check: the scoped path snapshots peer container
  IDs before/after and fails on a cascade; confirm it ran.
- All-zero readings right after a restart are a NULL sample, not health. Check: wait for real
  traffic, then measure, and say which it is.
- 202 from alert-service does not mean delivered. Check: confirm at the ntfy topic.

COMPLETION GATE — report NUMBERS: unit status and MainPID before/after each restart; every health
probe; new alert rules present in Prometheus and their state; before/after value of the metric the
fix targets; Loki errors per 10-min post-restart window; explicit confirmation that no unrelated
container restarted. Long output -> /tmp/sentinel-remediation/deploy-{N}/. Report (<=20 lines).
```

## Notes for the supervisor

- Name the authorised side effects explicitly. A deploy that resets a counter or zeroes a ledger
  by design looks identical to a regression from the agent's side; unnamed, it comes back as a
  false alarm, and the real one gets the same treatment next time.
- Say which alert must NOT fire, and why it cannot legitimately fire. "A brand-new series cannot
  reset, so a reset alert here would be a real finding" turns a deploy into a test with a
  predicted outcome, instead of a checklist.
