# deployment/config

Cross-cutting deployment reference files. Source of truth for conventions that span multiple services and need a single authoritative document.

## Files

| File | Purpose |
|---|---|
| `ports.yml` | Reference document — every ATLAS service's port allocation in one place. Split into `external:` (host-mapped) and `internal_only:` (container-network only). Conventions: MCP servers 3100–3199, observability 3000–3099 + 9000–9999, collectors 5000–5099, infrastructure on standard ports. Not auto-imported by Ansible — playbooks + `compose.yaml` hard-code the same numbers; keep this file in sync when allocating a new port. |

## Conventions

- **`ports.yml` is the allocation registry.** When adding a new service or port, claim the slot here first, then mirror it into the playbook + compose.yaml. Audit drift by comparing this file against rendered services.
- **Don't edit `/opt/ai-inference/compose.yaml` directly.** Per CLAUDE.md `DEPLOYMENT [HARD_STOP]`, that file is ansible-managed and any direct edit will be wiped.
- **Internal-only is the default.** Most services should be `internal_only` and proxied through Grafana (for observability) or accessed via service-name DNS from sibling containers. Host-mapping a port is a deliberate choice and a new entry under `external:`.
- **No hardcoded IPs anywhere.** Per `feedback_no_hardcoded_ips` — container CNI IPs drift on restart; service-name DNS is stable.

## See Also

- [deployment/ansible](../ansible/) — playbooks that bring services up
- [deployment/artifacts/scripts](../artifacts/scripts/README.md) — host-side helpers
