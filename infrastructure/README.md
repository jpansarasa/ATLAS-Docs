# ATLAS Infrastructure

Infrastructure-as-code definitions for the ATLAS platform. This directory contains the **WHAT** (service definitions, configs), while `../ansible/` contains the **HOW** (deployment automation).

## Directory Contents

```
infrastructure/
├── compose.yaml.j2       # Main compose template (Jinja2) - source of truth
├── atlas.service         # Systemd unit for auto-start
├── init-db.sh           # Database initialization script
├── monitoring/
│   ├── prometheus.yml    # Prometheus scrape config
│   ├── alertmanager.yml  # Alert routing rules
│   ├── alerts/          # Prometheus alert rules
│   ├── dashboards/      # Grafana dashboard JSON
│   ├── grafana.ini      # Grafana configuration
│   ├── loki-config.yaml # Log aggregation config
│   ├── tempo.yaml       # Distributed tracing config
│   └── otel-collector-config.yaml
└── scripts/
    └── generate-container-targets.sh
```

## Deployment

```bash
cd ~/ATLAS/ansible
ansible-playbook playbooks/site.yml
```

This renders `compose.yaml.j2` → `/opt/ai-inference/compose.yaml` with secrets from vault.

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

### MCP Servers (Claude Desktop)
| Service | Port | Description |
|---------|------|-------------|
| ollama-mcp | 3100 | Ollama inference |
| fredcollector-mcp | 3103 | FRED data access |
| thresholdengine-mcp | 3104 | Pattern evaluation |

### Observability
| Service | Port | Description |
|---------|------|-------------|
| prometheus | 9090 | Metrics storage |
| alertmanager | 9093 | Alert routing |
| grafana | 3000 | Dashboards |
| loki | 3101 | Log aggregation |
| tempo | 3200 | Distributed tracing |
| otel-collector | 4317 | OTLP receiver |

## Common Operations

### Service Status
```bash
sudo systemctl status atlas.service
cd /opt/ai-inference && sudo nerdctl compose ps
```

### View Logs
```bash
sudo nerdctl compose logs -f <service>
sudo journalctl -u atlas.service -f
```

### Database Access
```bash
sudo nerdctl exec -it timescaledb psql -U atlas -d atlas
```

### Pull Ollama Model
```bash
sudo nerdctl exec -it ollama-gpu ollama pull llama3.2:3b
```

## Monitoring URLs

- **Grafana**: http://localhost:3000
- **Prometheus**: http://localhost:9090
- **Alertmanager**: http://localhost:9093

## Template Variables

`compose.yaml.j2` uses Ansible variables from `ansible/group_vars/`:
- `atlas_db_user`, `atlas_db_name`, `atlas_db_password`
- `postgres_password`
- `fred_api_key`, `alphavantage_api_key`, `finnhub_api_key`

See [ansible/VAULT.md](../ansible/VAULT.md) for secrets management.

## Hardware

- **CPU**: AMD Threadripper 9960X
- **GPU**: RTX 5090 (32GB VRAM)
- **RAM**: 128GB
- **Storage**: NVMe 1.8TB (fast), SATA 5.2TB (bulk)
- **Runtime**: nerdctl + containerd

## See Also

- [Ansible](../ansible/README.md) - Deployment automation
- [STATE.md](../STATE.md) - Current system status
