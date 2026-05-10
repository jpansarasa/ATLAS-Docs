# {ProjectName}

## Overview

{1-2 paragraph elevator description: what does this service do, what's it for, what's the bigger system it fits into?}

## Architecture

{ASCII or text-described component breakdown. Show the key boundaries: ingress (API/queue), processing, persistence, egress. Reference key dependencies (other services, databases, queues). One short paragraph + a diagram or component list is plenty.}

## Features

- {Feature 1 — what it does for the user/system}
- {Feature 2}
- {Feature 3}

## Configuration

| Key | Description | Default |
|---|---|---|
| `appsettings:Section:Key` | {what it controls} | {default value} |
| `Environment Variable` | {what it controls} | {default value} |

{Connection strings, ports, integration endpoints, feature flags. List every env var or config key that affects runtime behavior.}

## API Endpoints

### REST API (port {N})

| Method | Path | Description |
|---|---|---|
| `GET` | `/api/...` | {what it does} |
| `POST` | `/api/...` | {what it does} |

### gRPC Services (port {N})

- `{ServiceName}.{MethodName}` — {what it does, request/response shape}

### Admin API

(only include if present)

### Health Endpoints

- `GET /health` — liveness
- `GET /ready` — readiness

## Project Structure

```
{ProjectName}/
  src/
    Data/
      Entities/    — {N entities: list them or describe what they represent}
      Migrations/  — EF migrations
    Services/      — {core business logic}
    Endpoints/     — {REST surface}
    Workers/       — {background processing}
  tests/
    {ProjectName}.UnitTests/
    {ProjectName}.IntegrationTests/
  config/          — {what's here}
  data/            — {seed data files}
```

## Development

### Prerequisites

- .NET 10 SDK (or via devcontainer — preferred per project conventions)
- Docker / nerdctl + Compose v2
- {Project-specific: e.g., access to TimescaleDB, FRED API key, etc.}

### Getting Started

```bash
# Build via devcontainer (preferred)
bash {ProjectName}/.devcontainer/compile.sh

# Run tests
bash {ProjectName}/.devcontainer/compile.sh   # tests run by default
```

### Build Container

```bash
bash {ProjectName}/.devcontainer/build.sh
```

## Deployment

- **Image:** `{project-name}:latest`
- **Ansible tag:** `--tags {project-name}`
- **Deploy command:**

```bash
ansible-playbook playbooks/deploy.yml --tags {project-name}
```

Per CLAUDE.md `DEPLOYMENT [HARD_STOP]`: never edit `/opt/ai-inference/compose.yaml` directly.

## Ports

| Service | Host | Container |
|---|---|---|
| HTTP | {50xx} | 8080 |
| gRPC | (internal only) | 5001 |

## See Also

- {Sibling service link 1}
- {Sibling service link 2}
