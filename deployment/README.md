# ATLAS Deployment

Everything needed to deploy and run ATLAS infrastructure.

## Structure

```
deployment/
├── artifacts/                # WHAT to deploy
│   ├── compose.yaml.j2       # Main compose template (Jinja2)
│   ├── atlas.service         # Systemd unit for auto-start
│   ├── init-db.sh           # Database initialization
│   ├── monitoring/          # Prometheus, Grafana, Loki, Tempo configs
│   └── scripts/             # Helper scripts
└── ansible/                  # HOW to deploy
    ├── ansible.cfg          # Ansible configuration
    ├── inventory/           # Host inventory
    ├── group_vars/          # Variables and secrets
    ├── playbooks/           # Deployment playbooks
    ├── scripts/             # Validation scripts
    └── VAULT.md             # Secrets documentation
```

## Quick Start

```bash
cd ~/ATLAS/deployment/ansible
ansible-playbook playbooks/site.yml
```

This deploys everything:
- Creates `/opt/ai-inference/` directory structure
- Renders `compose.yaml` from Jinja2 template with secrets
- Builds all container images
- Enables `atlas.service` for auto-start
- Creates ZFS pre-deployment snapshot

## Services Overview

### Data Collectors
| Service | Ports | Description |
|---------|-------|-------------|
| fred-collector | 5001 (REST), 5002 (gRPC) | FRED economic data |
| alphavantage-collector | 5003 (HTTP), 5004 (gRPC) | Commodities (WTI, Copper) |
| nasdaq-collector | 5005 (HTTP), 5006 (gRPC) | LBMA gold prices |
| finnhub-collector | 5007 (HTTP), 5008 (gRPC) | Stocks, sentiment |

### Processing & Storage
| Service | Port | Description |
|---------|------|-------------|
| threshold-engine | 8080 | Pattern evaluation, regime detection |
| alert-service | 8081 | Notification dispatch (ntfy, email) |
| timescaledb | 5432 | PostgreSQL + time-series |

### AI Inference
| Service | Port | Description |
|---------|------|-------------|
| ollama-gpu | 11434 | RTX 5090 GPU inference |
| ollama-cpu | 11435 | CPU fallback |

### Observability
| Service | Port | Description |
|---------|------|-------------|
| prometheus | 9090 | Metrics storage |
| alertmanager | 9093 | Alert routing |
| grafana | 3000 | Dashboards |
| loki | 3101 | Log aggregation |
| tempo | 3200 | Distributed tracing |
| otel-collector | 4317 | OTLP receiver |

## Common Workflows

### Deploy after changes

```bash
# Edit compose template or configs
vim ~/ATLAS/deployment/artifacts/compose.yaml.j2

# Deploy
cd ~/ATLAS/deployment/ansible
ansible-playbook playbooks/site.yml

# Verify
sudo systemctl status atlas.service
cd /opt/ai-inference && sudo nerdctl compose ps
```

### Dry run (check mode)

```bash
ansible-playbook playbooks/site.yml --check --diff
```

### Service management

```bash
sudo systemctl status atlas.service
sudo systemctl restart atlas.service
sudo journalctl -u atlas.service -f
```

### View container logs

```bash
cd /opt/ai-inference
sudo nerdctl compose logs -f <service>
```

### Database access

```bash
sudo nerdctl exec -it timescaledb psql -U atlas -d atlas
```

## ZFS Snapshots

Pre-deployment snapshots are automatically created for safe rollback.

```bash
# Deploy with snapshot (default)
ansible-playbook playbooks/site.yml

# Deploy without snapshot
ansible-playbook playbooks/site.yml -e create_snapshot=false

# Rollback to specific snapshot
ansible-playbook playbooks/zfs-rollback.yml -e snapshot_tag=pre-deploy-20251130T120000

# Cleanup old snapshots (keep last 3)
ansible-playbook playbooks/zfs-cleanup.yml -e cleanup_mode=auto -e keep_snapshots=3
```

## Secrets Management

See [ansible/VAULT.md](ansible/VAULT.md) for encrypted credentials management.

```bash
# View secrets
ansible-vault view ansible/group_vars/vault.yml

# Edit secrets
ansible-vault edit ansible/group_vars/vault.yml
```

## Template Variables

`artifacts/compose.yaml.j2` uses Ansible variables from `ansible/group_vars/`:
- `atlas_db_user`, `atlas_db_name`, `atlas_db_password`
- `postgres_password`
- `fred_api_key`, `alphavantage_api_key`, `finnhub_api_key`

## Monitoring URLs

- **Grafana**: http://localhost:3000
- **Prometheus**: http://localhost:9090
- **Alertmanager**: http://localhost:9093

## Hardware

- **CPU**: AMD Threadripper 9960X
- **GPU**: RTX 5090 (32GB VRAM)
- **RAM**: 128GB
- **Storage**: NVMe 1.8TB (fast), SATA 5.2TB (bulk)
- **Runtime**: nerdctl + containerd
