# ATLAS Ansible Automation

Ansible is the source of truth for ATLAS infrastructure deployments.

This directory contains the **HOW** (deployment automation), while `../infrastructure/` contains the **WHAT** (service definitions, configs).

## Quick Start

```bash
cd ~/ATLAS/ansible
ansible-playbook playbooks/site.yml
```

This deploys everything:
- Creates `/opt/ai-inference/` directory structure
- Deploys `compose.yaml` from Jinja2 template
- Builds all containers
- Enables `atlas.service` for auto-start
- Creates ZFS pre-deployment snapshot

## Structure

```
ansible/
├── ansible.cfg           # Ansible configuration
├── inventory/hosts.yml   # Local connection
├── group_vars/
│   ├── all.yml          # Common variables
│   └── vault.yml        # Encrypted secrets (see VAULT.md)
├── playbooks/
│   ├── site.yml         # Deploy everything
│   ├── infrastructure.yml
│   └── zfs-*.yml        # ZFS snapshot management
└── scripts/             # Helper scripts
```

## Common Workflows

### Deploy after changes

```bash
# Edit compose template or configs
vim ~/ATLAS/infrastructure/compose.yaml.j2

# Deploy
ansible-playbook playbooks/site.yml

# Verify
sudo systemctl status atlas.service
cd /opt/ai-inference && sudo nerdctl compose ps
```

### Dry run (check mode)

```bash
ansible-playbook playbooks/site.yml --check --diff
```

### Rebuild specific service

```bash
# Rebuild and restart single container
cd /opt/ai-inference
sudo nerdctl compose build fred-collector
sudo nerdctl compose up -d fred-collector
```

## ZFS Snapshots

Pre-deployment snapshots are automatically created for safe rollback.

**Datasets**: `nvme-fast/timeseries`, `nvme-fast/dashboard`
**Format**: `pre-deploy-YYYYMMDDTHHMMSS`

### Commands

```bash
# Deploy with snapshot (default)
ansible-playbook playbooks/site.yml

# Deploy without snapshot
ansible-playbook playbooks/site.yml -e create_snapshot=false

# Rollback to specific snapshot
ansible-playbook playbooks/zfs-rollback.yml -e snapshot_tag=pre-deploy-20251130T120000

# Cleanup old snapshots (keep last 3)
ansible-playbook playbooks/zfs-cleanup.yml -e cleanup_mode=auto -e keep_snapshots=3

# List snapshots
zfs list -t snapshot | grep pre-deploy
```

### Auto-snapshot retention

System uses `zfs-auto-snapshot`:
- Frequent: 8 (2 hours)
- Hourly: 24 (1 day)
- Daily: 7 (1 week)
- Weekly: 4 (1 month)
- Monthly: 6 (6 months)

## Secrets Management

See [VAULT.md](VAULT.md) for encrypted credentials management.

```bash
# View secrets
ansible-vault view group_vars/vault.yml

# Edit secrets
ansible-vault edit group_vars/vault.yml
```

## Philosophy

- **Infrastructure-as-code**: All deployments automated and version-controlled
- **Idempotent**: Running playbooks multiple times produces the same result
- **Single source of truth**: Edit in `~/ATLAS/`, deploy with ansible, never edit `/opt/ai-inference/` directly

## Runtime

- **Container runtime**: nerdctl + containerd
- **Deployment base**: `/opt/ai-inference`
- **Service management**: systemd (`atlas.service`)
