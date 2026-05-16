# deployment/ansible/scripts

Shell helpers invoked from Ansible playbooks. These are NOT meant for direct day-to-day use — they exist because the playbook that needs them has constraints (validation, host-side setup) that are easier to express in a tiny script than in a long `ansible.builtin.shell` task.

## Files

| Script | Invoked from | Purpose |
|---|---|---|
| `validate-rendered-template.sh` | `test-templating.yml` | Validates a rendered `compose.yaml` for syntactic correctness + minimum required keys before the deploy step swaps it in. Usage: `validate-rendered-template.sh <path-to-rendered-yaml>`. Exits non-zero on validation failure so the playbook halts. |
| `zfs-tune-snapshots.sh` | manual (one-shot host setup) | Tunes ZFS auto-snapshot retention on ATLAS datasets to a 49-snapshot policy (frequent/hourly/daily/weekly/monthly). Read the header for the exact policy before running. Idempotent — safe to re-run. |

## When to run these directly

- **`validate-rendered-template.sh`** — `bash deployment/ansible/scripts/validate-rendered-template.sh /tmp/compose.yaml` if you're testing a template change manually. Otherwise let the playbook drive it.
- **`zfs-tune-snapshots.sh`** — only when retention policy itself needs updating (rare). The playbooks do not invoke this on every deploy.

## See Also

- [deployment/ansible](../) — playbooks + inventory
- [deployment/artifacts/scripts](../../artifacts/scripts/README.md) — host-side helpers consumed by systemd timers + AutoFix
