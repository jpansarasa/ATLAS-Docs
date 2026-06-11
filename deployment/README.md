# Deployment

Ansible-based deployment for ATLAS infrastructure and services on mercury.

## Overview

This directory holds the ansible playbooks, Jinja2 templates, systemd units, host-side scripts, and configuration that turn the ATLAS monorepo into a running stack under `/opt/ai-inference/`. All work targets `mercury` and runs locally — the inventory uses `ansible_connection: local` and the playbooks live alongside what they deploy (`become: true`, `become_user: root`, `become_ask_pass: false`).

## Architecture

```mermaid
flowchart TD
    subgraph Ansible[deployment/ansible]
        A[playbooks/deploy.yml]
        B[group_vars/all.yml + vault.yml]
        C[inventory/hosts.yml]
    end

    subgraph Templates[deployment/artifacts]
        T1[compose.yaml.j2]
        T2[compose.otel.yaml.j2]
        T3[*.service / *.timer]
        T4[scripts/*.sh]
        T5[monitoring/*]
    end

    subgraph Host[mercury filesystem]
        D1[/opt/ai-inference/compose.yaml]
        D2[/opt/otel/compose.otel.yaml]
        D3[/etc/systemd/system/atlas.service]
        D4[/etc/systemd/system/otel.service]
        D5[/etc/systemd/system/autofix-*.timer]
        D6[/etc/systemd/system/merged-pr-watcher.timer]
        D7[/etc/systemd/system/buildkit-prune.timer]
        D8[/etc/systemd/system/atlas-sentinel-quality-check.timer]
        D9[/etc/systemd/system/sandbox-manager.service]
        D10[/etc/systemd/system/container-targets.timer]
    end

    subgraph Stacks[Running stacks]
        S1[atlas.service → nerdctl compose up -d]
        S2[otel.service → nerdctl compose -f compose.otel.yaml up -d]
        S3[sandbox-manager → host systemd]
        S4[vllm-server → standalone nerdctl run]
    end

    A --> B
    A --> C
    A --> T1 & T2 & T3 & T4 & T5
    T1 --> D1
    T2 --> D2
    T3 --> D3 & D4 & D5 & D6 & D7 & D8 & D9 & D10
    D1 --> S1
    D2 --> S2
    D9 --> S3
    A -.->|nerdctl run -d --gpus all| S4
```

The OTEL stack (`compose.otel.yaml`) and the ATLAS application stack (`compose.yaml`) are two separate compose files under two separate systemd units (`otel.service`, `atlas.service`) so that observability can stay up across application redeploys. `atlas.service` `Requires=otel.service` so the OTEL bridge network exists before app containers start. vLLM is launched standalone (not in either compose file) and managed directly via `nerdctl run` — see the audit note in `ansible/TAG_GATING_AUDIT.md` for why.

## Features

- **Tag-based selective deployment** — rebuild a single service (or a focused subsystem like `autofix`, `quality-check`, `merged-pr-watcher`) without restarting the rest.
- **ZFS snapshots** — automatic pre-deployment snapshot on `nvme-fast/timeseries` and `nvme-fast/dashboard`; manual playbooks for rollback and retention.
- **Vault-encrypted secrets** — all credentials live in `ansible/group_vars/vault.yml`; vault password file is `~/.ansible_vault_pass` (set in `ansible.cfg`).
- **Smoke tests** — `smoke-test.yml` exercises container status, HTTP `/health` (internal services + MCP servers), TimescaleDB connectivity, GPU + vLLM, and recent Loki errors.
- **AutoFix + merged-PR + quality-check + buildkit-prune** — four independent systemd timer subsystems orchestrating alert-driven Claude Code runs, auto-deploy on PR merge, weekly Sentinel quality sampling, and BuildKit cache trimming.
- **sandbox-manager (host systemd)** — non-containerised because containerd-runtime-in-container fails snapshot-mount prep; runs as `atlas` user with a single-binary sudoers rule for `nerdctl`.

## Playbooks

All playbooks live in `ansible/playbooks/` and assume the working dir is `deployment/ansible/` so `inventory/hosts.yml` resolves via `ansible.cfg`.

| Playbook | Description |
|----------|-------------|
| `deploy.yml` | Main deployment: ZFS snapshot, atlas/containerd user+group, OTEL stack, compose template render, all service builds + image pulls, ThresholdEngine + SecMaster + Sentinel + CoD config sync, databases (`atlas_data`, `calendar_data`, `atlas_secmaster`), vLLM standalone container, llama-server + ollama-cpu-gen + ollama-cpu-embed, systemd units (atlas, autofix-runner, autofix-watcher, merged-pr-watcher, buildkit-prune, atlas-sentinel-quality-check, container-targets, sandbox-manager), and final container-status report. |
| `site.yml` | Thin wrapper: `import_playbook: deploy.yml`. |
| `smoke-test.yml` | Health validation. Sub-tags: `health`, `containers`, `internal`, `mcp`, `logs`, `loki`, `docker`, `gpu`, `database`. |
| `zfs-snapshot.yml` | Create a tagged ZFS snapshot manually (`-e snapshot_tag=NAME`). |
| `zfs-rollback.yml` | Stop atlas, rollback datasets to a snapshot, restart atlas. Interactive — prompts for `yes` confirmation. Required: `-e snapshot_tag=NAME`. |
| `zfs-cleanup.yml` | Remove snapshots. `cleanup_mode=manual` (default, requires `snapshot_tag`) or `cleanup_mode=auto -e keep_snapshots=N`. |
| `test-templating.yml` | Render `compose.yaml.j2` into `ansible/test-output/` and run `scripts/validate-rendered-template.sh` against it. Useful for verifying a template change without a real deploy. Connection: `localhost` / `local`. |
| `test-vars.yml` | Print `deployment_base`, `mcp_server_deploy_path`, `atlas_repo_path` — sanity check that group_vars resolve. |

## Tags

Every `--tags X` invocation in `deploy.yml` matches a tag declared on at least one task or block. `always` tasks run on every invocation regardless of `--tags`; `never` tasks (currently only the Cloudflare Worker block) require an explicit opt-in. The vLLM block carries only `vllm-server` (NOT `always`) because cycling it on incidental tag runs caused a driver-mismatch outage on 2026-05-14 — see `ansible/TAG_GATING_AUDIT.md`.

### Application service builds (each pairs with `build`)

| Tag | Scope |
|-----|-------|
| `fred-collector` | FredCollector image build |
| `threshold-engine` | ThresholdEngine image build + pattern-config sync (`patterns` sub-tag for hot reload only) |
| `alert-service` | AlertService image build; also gates the AutoFix + merged-PR-watcher subsystems |
| `finnhub-collector` | FinnhubCollector image build |
| `finnhub-mcp` | FinnhubMcp image build |
| `ofr-collector` | OfrCollector image build |
| `ofr-mcp` | OfrMcp image build |
| `alphavantage-collector` | AlphaVantageCollector image build |
| `nasdaq-collector` | NasdaqCollector image build |
| `calendar-service` | CalendarService image build |
| `secmaster` | SecMaster image build + `instruments` config sync + `atlas_secmaster` DB extensions (`pg_trgm`, `pgvector`) |
| `secmaster-mcp` | SecMasterMcp image build |
| `sentinel-collector` | SentinelCollector image build; pairs with `sentinel-prompts` (host-mount sync) and `cpu-cod-prompts` (CoD prompts sync) |
| `whisper-service` | WhisperService image build |
| `whisper-service-mcp` | WhisperServiceMcp image build |
| `finbert-sidecar` | FinBertSidecar image build |
| `fredcollector-mcp` | FredCollectorMcp image build |
| `thresholdengine-mcp` | ThresholdEngineMcp image build |
| `macro-substrate` | MacroSubstrate.Migrator one-shot image build |
| `spacy-ner` | spaCy NER sidecar image build (entity resolution pre-pass) |
| `dsl-parser-mcp` | DSL parser MCP sidecar image build (CPU parse + grounding verifier; `/parse` CPU DSL + `/parse_json` GPU JSON-CoD) |
| `sandbox-kernel` | Persistent IPython kernel image (per-agent/CoVe session sandboxes) |
| `sandbox-manager` | Host-systemd Python service that spawns sandbox-kernel containers; includes sudoers rule + nerdctl shim install |
| `reports-daily` / `reports-weekly` / `reports-monthly` | Reports.{Daily,Weekly,Monthly}Host image builds |
| `build` | Cross-cutting alias on every image-build task — rebuilds all services |

### Sub-tags (config sync / hot reload, no rebuild)

| Tag | Scope |
|-----|-------|
| `patterns` | ThresholdEngine pattern YAML sync + HTTP POST `/api/patterns/reload` (hot reload) |
| `instruments` | SecMaster instrument config sync (paired with `secmaster`) |
| `sentinel-prompts` | Sync `SentinelCollector/src/prompts/` → `/opt/ai-inference/prompts/sentinel/` (overwrites host edits) |
| `cpu-cod-prompts` | Sync `SentinelCollector/src/cod-prompts/` → `/opt/ai-inference/prompts/cod/` (overwrites host edits) |

### Observability + monitoring

| Tag | Scope |
|-----|-------|
| `otel` (alias: `monitoring`) | Render `compose.otel.yaml`, deploy `otel.service`, create `ai-inference` shared network, start/restart OTEL stack |
| `dashboards` | Grafana dashboards + provisioning directory sync (Grafana auto-reloads) |
| `alerting` (alias: `monitoring`) | Prometheus alert rules + Alertmanager config sync + Prometheus `kill -HUP 1` reload |

### Runtime / model backends

| Tag | Scope |
|-----|-------|
| `vllm-server` | Stop+remove+recreate the standalone `vllm-server` container with GPU passthrough. Intentionally NOT `always` (see TAG_GATING_AUDIT.md). |
| `llama-server` (alias: `dsl-poc`) | Pull `ghcr.io/ggml-org/llama.cpp:server`, `compose up -d llama-server`, wait on `/health`, fetch `/props` for model-identity check |
| `ollama` / `ollama-cpu-gen` | Pull pinned `ollama/ollama` image, recreate `ollama-cpu-gen` container, verify `ollama --version` |
| `ollama-cpu-embed` (alias: `models`) | `ollama pull bge-m3` against the embed runner |

### Maintenance / supervisor / one-off

| Tag | Scope |
|-----|-------|
| `autofix` (alias: `alert-service`) | Deploy `autofix.sh`, `autofix-runner.sh`, `autofix-watcher.sh`; install/enable `autofix-runner.{service,timer}` + `autofix-watcher.{service,timer}`; assert `/etc/autofix/claude.env` exists with the long-lived Claude OAuth token |
| `merged-pr-watcher` (alias: `alert-service`) | Deploy `merged-pr-watcher.sh` + units. Timer is intentionally NOT enabled by the playbook — operator enables manually after first dry-run review. |
| `quality-check` (alias: `maintenance`) | Deploy `atlas-sentinel-quality-check.{service,timer}` (weekly Sentinel sampling Monday 09:23) |
| `buildkit-prune` (alias: `maintenance`) | Deploy `buildkit-prune.{service,timer}` for periodic BuildKit cache trim |
| `snapshot` | ZFS pre-deploy snapshot block only (skipped via `-e create_snapshot=false`) |
| `atlas-systemd` / `orphan-cleanup` | Disable + remove the pre-2026-04-17 orphan `ai-inference.service`; drop the legacy `financial_news` bootstrap DB |
| `edge` / `sentinel-edge` | Cloudflare Worker deploy via the sentinel-edge devcontainer + `wrangler` inside it. Tagged `never` — only runs when explicitly requested. |

## Configuration

### Variables (`ansible/group_vars/all.yml`)

| Variable | Default | Description |
|----------|---------|-------------|
| `atlas_repo_path` | `/home/james/ATLAS` | Monorepo source root (referenced by every `nerdctl build` chdir + template/copy `src:`) |
| `deployment_base` | `/opt/ai-inference` | Target deployment root on mercury |
| `logs_path` | `{{ deployment_base }}/logs` | Log + state volumes for grafana/loki/tempo/prometheus/alertmanager/autofix |
| `models_path` | `{{ deployment_base }}/models` | Ollama + GGUF blob storage |
| `timeseries_path` | `{{ deployment_base }}/timeseries` | TimescaleDB data |
| `dashboard_path` | `{{ deployment_base }}/dashboard` | Grafana persistent data |
| `mcp_server_deploy_path` | `{{ deployment_base }}/mcp-server` | MCP server deploy root (declared; currently consumed only by `test-vars.yml`) |
| `compose_deploy_path` | `{{ deployment_base }}/compose` | Compose deploy root (declared; no current consumer — kept for forward use) |
| `container_runtime` | `nerdctl` | Container runtime (kept as a var so podman/docker swap is a one-line change) |
| `atlas_db_user` | env `ATLAS_DB_USER` or `atlas_user` | TimescaleDB application user |
| `atlas_db_name` | env `ATLAS_DB_NAME` or `atlas_data` | Primary application database |
| `ollama_cpu_gen_url` | `http://ollama-cpu-gen:11434` | CPU generation runner (qwen2.5:7b, sentinel-cod-v6) |
| `ollama_cpu_embed_url` | `http://ollama-cpu-embed:11434` | CPU embedding runner (bge-m3) — split out 2026-05 to remove intra-container contention |
| `llama_server_url` | `http://llama-server:8080` | llama.cpp GBNF-constrained extraction backend (DSL PoC Phase 2) |
| `llama_server_ctx_size` | `32768` | llama-server total context (divided across `--parallel` slots) |
| `llama_server_parallel` | `1` | llama-server concurrent slots; see all.yml header for ctx-vs-parallel math |
| `vllm_base_model` | `Qwen/Qwen2.5-32B-Instruct-AWQ` | HF model id served by vllm-server (LoRA removed 2026-05-17; served-model-name alias dropped 2026-05-25) |
| `vllm_max_model_len` | `32768` | vLLM context window |
| `vllm_max_num_seqs` | `16` | Cap concurrent sequences — default 256 OOMs on RTX 5090 with 32B-AWQ + 32K ctx |
| `vllm_gpu_memory_utilization` | `0.92` | vLLM GPU memory fraction |
| `vllm_lora_*` | (forensic) | Retained as historical record; not consumed since LoRA removal |
| `sentinel_max_concurrent_extractions` | `4` | SentinelCollector → vLLM concurrency cap (~18K KV-cache tokens per slot at 72K budget) |
| `sentinel_digest_public_base_url` | `http://mercury.elasticdevelopment.com:5091` | Daily-digest ntfy click-through (FQDN — bare `mercury` only resolves on-LAN) |
| `sentinel_edge_endpoint` / `sentinel_edge_api_key` | from vault | Cloudflare Worker edge endpoint + auth |
| `ports_external.*` | see below | Host-mapped ports — referenced by compose template + smoke tests |
| `ports_mcp.*` | see below | MCP server host ports (Claude Code SSE access) |
| `ports_internal.*` | see below | Container-internal ports (documented for reference) |

### Secrets (`ansible/group_vars/vault.yml`)

Encrypted with Ansible Vault. The vault password is auto-loaded from `~/.ansible_vault_pass` (configured in `ansible.cfg`). Inventoried groups:

- **Database** — `postgres_password`, `atlas_db_password`
- **Collector API keys** — `fred_api_key`, `alphavantage_api_key`, `finnhub_api_key`, `nasdaq_api_key`, `openfigi_api_key`
- **Internal API auth** — `api_key`, `api_key_enabled`
- **Notifications** — `ntfy_endpoint`, `ntfy_topic`, `ntfy_username`, `ntfy_password`, `email_enabled`, `email_smtp_host`, `email_smtp_port`, `email_username`, `email_password`, `email_from`, `email_to`
- **Grafana** — `vault_grafana_admin_password`, `vault_grafana_google_client_id`, `vault_grafana_google_client_secret`
- **Sentinel** — `vault_sentinel_edge_endpoint`, `vault_sentinel_edge_api_key`
- **Cloudflare** — `vault_cloudflare_api_token` (used only by the `edge`/`sentinel-edge` block)

```bash
ansible-vault view ansible/group_vars/vault.yml
ansible-vault edit ansible/group_vars/vault.yml
```

See `ansible/VAULT.md` for the full secret-management workflow, including how to add a new secret and rotate keys.

### Ports

Source of truth is `ansible/group_vars/all.yml` (`ports_external`, `ports_mcp`, `ports_internal`). `config/ports.yml` is the cross-cutting allocation registry — keep both in sync when claiming a new port.

| Range / port | Purpose |
|--------------|---------|
| 3000 | Grafana UI |
| 3102–3108 | MCP servers (markitdown 3102, fred 3103, threshold 3104, finnhub 3105, ofr 3106, secmaster 3107, whisper 3108) |
| 5001 | gRPC events between collectors and ThresholdEngine (internal only) |
| 5091 | SentinelCollector review UI (host-mapped for browser access) |
| 5432 | TimescaleDB (host-mapped for dev/psql) |
| 8000 | vllm-server OpenAI-compatible API |
| 8080 | Default internal HTTP for every .NET service (and the llama-server container port) |
| 8090 | WhisperService external API |
| 3109 / 3110 | trafilatura / spacy-ner sidecars (host-mapped for cross-container reach) |
| 9090 / 9093 / 9100 / 9835 | Prometheus / Alertmanager / node-exporter / gpu-exporter (internal) |
| 11435 | ollama-cpu-gen (generation runner) |
| 11436 | ollama-cpu-embed (embedding runner — internal-style, but host-mapped) |
| 11437 | llama-server (CPU GBNF extraction) |
| 4317 / 4318 / 8888 / 8889 | OTEL collector gRPC / HTTP / metrics / prom exporter (internal) |
| 3100 / 3200 | Loki / Tempo (internal — Grafana proxies queries) |

## Project Structure

```
deployment/
├── README.md                       # this file
├── ansible/
│   ├── ansible.cfg                 # inventory path, vault password file, become defaults
│   ├── VAULT.md                    # vault workflow + secret rotation
│   ├── TAG_GATING_AUDIT.md         # 2026-05-14 vllm-outage post-mortem + tag-gating verification
│   ├── group_vars/
│   │   ├── all.yml                 # paths, ports, model ids, runtime tunables
│   │   └── vault.yml               # encrypted credentials (Ansible Vault)
│   ├── inventory/
│   │   └── hosts.yml               # single host `mercury` (ansible_connection: local)
│   ├── playbooks/                  # deploy, smoke-test, zfs-*, test-* (see Playbooks table)
│   └── scripts/
│       ├── README.md
│       ├── validate-rendered-template.sh  # invoked by test-templating.yml
│       └── zfs-tune-snapshots.sh          # one-shot ZFS auto-snapshot retention policy
├── artifacts/
│   ├── compose.yaml.j2              # main ATLAS compose template (~31 services)
│   ├── compose.otel.yaml.j2         # OTEL stack compose template (prometheus, grafana, loki, tempo, otel-collector, exporters)
│   ├── atlas.service                # systemd unit: nerdctl compose up -d
│   ├── otel.service                 # systemd unit: nerdctl compose -f compose.otel.yaml up -d
│   ├── sandbox-manager.service      # host systemd unit (FastAPI service running as `atlas`)
│   ├── autofix-runner.service       # AutoFix alert processor (oneshot, james user)
│   ├── autofix-runner.timer
│   ├── autofix-watcher.service      # AutoFix PR-merge watcher (oneshot, james user)
│   ├── autofix-watcher.timer
│   ├── merged-pr-watcher.service    # Generalised auto-deploy on any merged `auto-deploy`-labelled PR
│   ├── merged-pr-watcher.timer      # NOT enabled by playbook — operator enables post-review
│   ├── buildkit-prune.service       # Periodic BuildKit cache trim
│   ├── buildkit-prune.timer
│   ├── atlas-sentinel-quality-check.service  # Weekly Sentinel quality sampling
│   ├── atlas-sentinel-quality-check.timer
│   ├── init-db.sh                   # docker-entrypoint-initdb.d bootstrap (first init only)
│   ├── monitoring/
│   │   ├── alerts/                  # Prometheus alert rules per service (.yml)
│   │   ├── dashboards/              # Grafana JSON dashboards
│   │   ├── provisioning/            # Grafana datasources + dashboards provisioning
│   │   ├── loki-rules/              # Loki recording/alerting rules
│   │   ├── scripts/                 # Monitoring-side helpers
│   │   ├── gpu-exporter/            # Custom NVIDIA GPU exporter (built by compose.otel.yaml.j2)
│   │   ├── ups-exporter/            # APC UPS exporter (built by compose.otel.yaml.j2)
│   │   ├── prometheus.yml / loki-config.yaml / tempo.yaml / alertmanager.yml / otel-collector-config.yaml / grafana.ini / robots.txt
│   ├── sandbox-kernel/              # IPython kernel image build context (`sandbox-kernel:latest`)
│   ├── sandbox-manager/             # FastAPI host service (manager_server.py, requirements.txt)
│   ├── spacy-ner/                   # spaCy NER sidecar build context
│   ├── trafilatura/                 # trafilatura sidecar build context
│   └── scripts/                     # host-side scripts (see artifacts/scripts/README.md)
└── config/
    ├── README.md
    └── ports.yml                     # cross-cutting port allocation registry
```

## Development

### Quick Start

All commands assume `cd deployment/ansible/` so the `ansible.cfg` inventory + vault paths resolve.

```bash
ansible-playbook playbooks/deploy.yml                              # full deployment (excludes `never`-tagged edge block)
ansible-playbook playbooks/deploy.yml --check --diff               # dry run
ansible-playbook playbooks/deploy.yml --tags fred-collector        # single service rebuild
ansible-playbook playbooks/deploy.yml --tags ofr-collector,ofr-mcp # multiple services
ansible-playbook playbooks/deploy.yml --tags dashboards            # dashboards only (Grafana auto-reloads)
ansible-playbook playbooks/deploy.yml --tags patterns              # ThresholdEngine pattern hot-reload (no rebuild)
ansible-playbook playbooks/deploy.yml --tags sentinel-prompts      # sync Sentinel prompts to host mount
ansible-playbook playbooks/deploy.yml --tags vllm-server           # cycle vllm-server (recreate the standalone container)
ansible-playbook playbooks/deploy.yml --tags edge,sentinel-edge    # Cloudflare Worker deploy (opt-in)
ansible-playbook playbooks/deploy.yml -e create_snapshot=false     # skip ZFS pre-deploy snapshot
```

### Smoke Tests

```bash
ansible-playbook playbooks/smoke-test.yml                      # full: containers + internal HTTP + MCP + DB + GPU/vLLM + Loki errors
ansible-playbook playbooks/smoke-test.yml --tags health        # health checks only (skip log analysis)
ansible-playbook playbooks/smoke-test.yml -e check_logs=false  # skip Loki/container log scan
ansible-playbook playbooks/smoke-test.yml --tags gpu           # GPU + vllm-server health only
```

The playbook checks: `nerdctl compose ps` for unhealthy containers, `/api/health` or `/health` (per-service mapping in the playbook vars), all MCP `/health` endpoints, `nvidia-smi` + `vllm-server` `/health`, TimescaleDB `SELECT 1`, and the last `log_lookback_minutes` (default 5) of Loki errors via the grafana container. Falls back to `nerdctl logs` if Loki is unavailable.

### Template change verification

```bash
ansible-playbook playbooks/test-templating.yml
# Renders compose.yaml.j2 to deployment/ansible/test-output/compose-rendered.yaml
# and runs ansible/scripts/validate-rendered-template.sh against it.
```

### Service Management

```bash
sudo systemctl status atlas.service                                       # ATLAS app stack
sudo systemctl status otel.service                                        # OTEL stack
sudo systemctl status sandbox-manager.service                             # host-side FastAPI
sudo systemctl list-timers 'autofix-*' merged-pr-watcher.timer \
    atlas-sentinel-quality-check.timer buildkit-prune.timer               # all maintenance timers
cd /opt/ai-inference && sudo nerdctl compose ps                           # container roster (WorkingDirectory of atlas.service)
sudo nerdctl compose logs -f fred-collector
sudo nerdctl exec timescaledb psql -U ai_inference -d atlas_data          # database shell (container user is ai_inference)
```

### ZFS Snapshots

Pre-deploy snapshots are taken automatically on `nvme-fast/timeseries` and `nvme-fast/dashboard`. Snapshot tag format: `pre-deploy-YYYYMMDDTHHMMSS` (basic ISO 8601). The most recent tag is written to `/opt/ai-inference/last-snapshot.txt`.

```bash
ansible-playbook playbooks/zfs-snapshot.yml -e snapshot_tag=before-migration
ansible-playbook playbooks/zfs-rollback.yml -e snapshot_tag=pre-deploy-20260527T120000
ansible-playbook playbooks/zfs-cleanup.yml -e cleanup_mode=auto -e keep_snapshots=3
ansible-playbook playbooks/zfs-cleanup.yml -e snapshot_tag=pre-deploy-20260527T120000  # manual single-tag cleanup
```

## Monitoring URLs

| Service | URL |
|---------|-----|
| Grafana | http://mercury:3000 |
| SentinelCollector review UI | http://mercury:5091 (also `http://mercury.elasticdevelopment.com:5091` for off-LAN clients) |
| WhisperService API | http://mercury:8090 |
| vLLM OpenAI API | http://mercury:8000 |

Prometheus, Alertmanager, Loki, Tempo, and the OTEL collector are intentionally NOT host-mapped — Grafana proxies queries to them across the `ai-inference` container network. The exporters (node, gpu, ups) are scraped by Prometheus over the same network.

## Hard Stops

- **Never edit `/opt/ai-inference/compose.yaml` directly.** It is rendered from `artifacts/compose.yaml.j2`; the next `deploy.yml` run will overwrite any direct edits. The playbook actively deletes the file before re-rendering (the "Remove existing compose.yaml to force regeneration" step).
- **Never edit `/opt/otel/compose.otel.yaml` directly.** Same pattern — rendered from `artifacts/compose.otel.yaml.j2`.
- **Never edit `/etc/systemd/system/atlas.service` (or the other `*.service` / `*.timer` units copied by `deploy.yml`) directly.** They are copies of the files in `deployment/artifacts/`.
- **Never run `ssh mercury` or `ansible mercury` from this repo.** The host you're typing on IS mercury — the inventory uses `ansible_connection: local` (see CLAUDE.md `EXECUTION_CONTEXT`).

## See Also

- [ansible/VAULT.md](ansible/VAULT.md) — Ansible Vault usage and secret management
- [ansible/TAG_GATING_AUDIT.md](ansible/TAG_GATING_AUDIT.md) — 2026-05-14 vLLM outage post-mortem + tag-gating verification
- [ansible/scripts/README.md](ansible/scripts/README.md) — playbook-invoked helpers
- [artifacts/scripts/README.md](artifacts/scripts/README.md) — host-side scripts (AutoFix orchestration, container targets, seeding)
- [config/README.md](config/README.md) — `config/ports.yml` allocation registry
- [docs/ARCHITECTURE.md](../docs/ARCHITECTURE.md) — system design
- [docs/OBSERVABILITY.md](../docs/OBSERVABILITY.md) — metrics, tracing, logging conventions
