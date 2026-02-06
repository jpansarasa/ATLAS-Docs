# Deployment

Ansible-based deployment for ATLAS infrastructure and services on mercury.

## Overview

This directory contains everything needed to deploy ATLAS: Ansible playbooks that render Jinja2 templates with encrypted secrets, build container images via nerdctl, configure the observability stack, and manage systemd services. All services run on a single host (mercury) using `ansible_connection: local`.

## Architecture

```mermaid
flowchart TD
    subgraph Ansible
        A[playbooks/deploy.yml]
        B[group_vars/vault.yml]
        C[artifacts/compose.yaml.j2]
    end

    subgraph mercury
        D[/opt/ai-inference/compose.yaml]
        E[atlas.service systemd]
        F[nerdctl compose]
    end

    subgraph Services
        G[Collectors]
        H[ThresholdEngine]
        I[SecMaster]
        J[MCP Servers]
        K[Observability Stack]
    end

    A -->|template + secrets| D
    B --> A
    C --> A
    D --> E --> F
    F --> G & H & I & J & K
```

Ansible renders `compose.yaml` from Jinja2 template with vault secrets, builds container images from the monorepo, and manages the `atlas.service` systemd unit that runs `nerdctl compose up -d`.

## Features

- **Tag-based selective deployment**: Rebuild individual services without touching others
- **ZFS snapshots**: Automatic pre-deployment snapshots with rollback support
- **Vault-encrypted secrets**: API keys and credentials managed via Ansible Vault
- **Smoke tests**: Post-deployment health validation across all services
- **AutoFix**: Automated incident response via systemd timers and alert-driven scripts
- **OTEL stack**: Separate compose file for observability infrastructure (rarely restarted)

## Playbooks

| Playbook | Description |
|----------|-------------|
| `deploy.yml` | Main deployment: templates, builds, systemd, database setup |
| `site.yml` | Alias for `deploy.yml` |
| `smoke-test.yml` | Post-deployment health checks and log analysis |
| `zfs-snapshot.yml` | Create ZFS snapshots manually |
| `zfs-rollback.yml` | Rollback to a previous ZFS snapshot |
| `zfs-cleanup.yml` | Remove old snapshots (keep last N) |
| `test-templating.yml` | Validate Jinja2 template rendering |
| `test-vars.yml` | Verify variable resolution |

## Tags

| Tag | Scope |
|-----|-------|
| `fred-collector` | FredCollector |
| `threshold-engine` | ThresholdEngine |
| `alert-service` | AlertService + AutoFix components |
| `finnhub-collector` | FinnhubCollector |
| `finnhub-mcp` | FinnhubMcp |
| `ofr-collector` | OfrCollector |
| `ofr-mcp` | OfrMcp |
| `secmaster` | SecMaster + database extensions + embedding model |
| `secmaster-mcp` | SecMasterMcp |
| `alphavantage-collector` | AlphaVantageCollector |
| `nasdaq-collector` | NasdaqCollector |
| `calendar-service` | CalendarService |
| `fredcollector-mcp` | FredCollectorMcp |
| `thresholdengine-mcp` | ThresholdEngineMcp |
| `ollama-mcp` | OllamaMcp |
| `sentinel-collector` | SentinelCollector |
| `whisper-service` | WhisperService |
| `whisper-service-mcp` | WhisperServiceMcp |
| `finbert-sidecar` | FinBertSidecar |
| `build` | All service builds |
| `monitoring` | Monitoring configs, dashboards, alert rules |
| `dashboards` | Grafana dashboards and provisioning |
| `alerting` | Alert rules + Alertmanager config + Prometheus reload |
| `patterns` | ThresholdEngine pattern configs (hot reload) |
| `otel` | OTEL observability stack (collector, Loki, Tempo, Prometheus) |
| `snapshot` | ZFS snapshot tasks only |
| `autofix` | AutoFix watcher/runner systemd timers and scripts |
| `edge` | Cloudflare Workers deployment (requires explicit tag) |
| `llama-server` | llama.cpp server with GPU (requires `-e deploy_llama_server=true`) |

## Configuration

### Variables (`group_vars/all.yml`)

| Variable | Default | Description |
|----------|---------|-------------|
| `deployment_base` | `/opt/ai-inference` | Target deployment directory |
| `atlas_repo_path` | `/home/james/ATLAS` | Monorepo source path |
| `logs_path` | `{deployment_base}/logs` | Log storage |
| `timeseries_path` | `{deployment_base}/timeseries` | TimescaleDB data |
| `dashboard_path` | `{deployment_base}/dashboard` | Grafana data |
| `models_path` | `{deployment_base}/models` | Ollama models |
| `container_runtime` | `nerdctl` | Container runtime |

### Secrets (`group_vars/vault.yml`)

Encrypted with Ansible Vault. Key secrets include `atlas_db_password`, `fred_api_key`, `finnhub_api_key`, `alphavantage_api_key`, `nasdaq_api_key`, `smtp_*`, and `ntfy_*` credentials.

```bash
ansible-vault view ansible/group_vars/vault.yml
ansible-vault edit ansible/group_vars/vault.yml
```

### Port Conventions

| Range | Purpose |
|-------|---------|
| 3000-3099 | Observability UIs (Grafana: 3000) |
| 3100-3199 | MCP servers (SSE access for Claude Code) |
| 5000-5099 | Internal collector gRPC |
| 8080 | Default internal HTTP (all .NET services) |
| 9000-9999 | Prometheus, exporters, Alertmanager |
| 11434-11435 | Ollama GPU/CPU |

## Project Structure

```
deployment/
├── ansible/
│   ├── ansible.cfg           # Connection, vault password file
│   ├── group_vars/
│   │   ├── all.yml           # Paths, ports, non-sensitive config
│   │   └── vault.yml         # Encrypted credentials (Ansible Vault)
│   ├── inventory/hosts.yml   # mercury (local connection)
│   ├── playbooks/            # deploy, smoke-test, zfs-* playbooks
│   └── scripts/              # Template validation, ZFS tuning
├── artifacts/
│   ├── compose.yaml.j2       # Main compose template (Jinja2)
│   ├── compose.otel.yaml.j2  # OTEL stack compose template
│   ├── atlas.service          # systemd unit (nerdctl compose)
│   ├── otel.service           # systemd unit (OTEL stack)
│   ├── autofix-*.service      # AutoFix systemd units + timers
│   ├── init-db.sh             # Database initialization
│   ├── monitoring/            # Prometheus, Loki, Tempo, Grafana configs
│   │   ├── alerts/            # Prometheus alert rules per service
│   │   ├── dashboards/        # Grafana JSON dashboards
│   │   └── provisioning/      # Grafana datasource/dashboard setup
│   └── scripts/               # autofix.sh, container-targets generator
└── config/
    └── ports.yml              # Port allocation reference
```

## Development

### Quick Start

```bash
ansible-playbook playbooks/deploy.yml                              # full deployment
ansible-playbook playbooks/deploy.yml --check --diff               # dry run
ansible-playbook playbooks/deploy.yml --tags fred-collector        # single service
ansible-playbook playbooks/deploy.yml --tags ofr-collector,ofr-mcp # multiple services
ansible-playbook playbooks/deploy.yml --tags dashboards            # dashboards only
ansible-playbook playbooks/deploy.yml --tags patterns              # hot-reload patterns
ansible-playbook playbooks/deploy.yml -e create_snapshot=false     # skip ZFS snapshot
```

### Smoke Tests

```bash
ansible-playbook playbooks/smoke-test.yml                     # Full test
ansible-playbook playbooks/smoke-test.yml --tags health       # Health checks only
ansible-playbook playbooks/smoke-test.yml -e check_logs=false # Skip log analysis
```

Checks container status, `/health` endpoints for all services and MCP servers, database connectivity, and recent Loki error logs.

### Service Management

```bash
sudo systemctl status atlas.service                    # systemd status
sudo nerdctl compose ps                                # container status (at /opt/ai-inference)
sudo nerdctl compose logs -f fred-collector             # follow service logs
sudo nerdctl exec timescaledb psql -U ai_inference -d atlas_data  # database shell
```

### ZFS Snapshots

Automatic pre-deployment snapshots are created on `nvme-fast/timeseries` and `nvme-fast/dashboard`.

```bash
ansible-playbook playbooks/zfs-snapshot.yml -e snapshot_tag=before-migration  # manual
ansible-playbook playbooks/zfs-rollback.yml -e snapshot_tag=pre-deploy-20251216T120000
ansible-playbook playbooks/zfs-cleanup.yml -e cleanup_mode=auto -e keep_snapshots=3
```

## Monitoring URLs

| Service | URL |
|---------|-----|
| Grafana | http://mercury:3000 |
| Prometheus | http://mercury:9090 |
| Alertmanager | http://mercury:9093 |

## Hard Stop

Never edit `/opt/ai-inference/compose.yaml` directly. It is Ansible-managed and will be overwritten on next deployment. Always edit `artifacts/compose.yaml.j2` and redeploy.

## See Also

- [VAULT.md](ansible/VAULT.md) - Ansible Vault usage and secret management
- [config/ports.yml](config/ports.yml) - Complete port allocation reference
- [docs/ARCHITECTURE.md](../docs/ARCHITECTURE.md) - System design
