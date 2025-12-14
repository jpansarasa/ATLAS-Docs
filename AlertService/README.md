# AlertService

Centralized alert routing and notification dispatch service for ATLAS.

## Overview

AlertService receives alert webhooks from external monitoring (Alertmanager), queues them, and dispatches notifications to configured channels (Ntfy, Email) based on severity routing rules.

## Architecture

```mermaid
flowchart LR
    subgraph Inputs
        AM[Alertmanager]
        API[Webhook API]
    end

    subgraph Core [AlertService]
        Queue[In-Memory Queue]
        Dispatcher[Background Dispatcher]
        Router[Severity Router]
    end

    subgraph Channels
        Ntfy[Ntfy Channel]
        Email[Email Channel]
    end

    AM --> API
    API --> Queue
    Queue --> Dispatcher
    Dispatcher --> Router
    Router -- Critical/Warning --> Ntfy
    Router -- Critical --> Email
```

## Key Features

- **Unified Ingestion**: Accepts alerts in direct JSON format or Prometheus Alertmanager webhook format.
- **Async Processing**: Decouples ingestion from delivery using an in-memory background queue.
- **Severity Routing**: Configurable routing rules map alert severities (Critical, Warning, Info) to specific channels.
- **Multi-Channel**: Supports Ntfy (push notifications) and Email (SMTP) out of the box.
- **Observability**: Extensive metrics (queue depth, delivery latency) and tracing.

## Configuration

Configure via `appsettings.json` or Environment Variables (`__` separator).

### Channels

**Ntfy** (Push Notifications)
```json
"Channels": {
  "Ntfy": {
    "Enabled": true,
    "Endpoint": "https://ntfy.sh",
    "Topic": "atlas-alerts"
  }
}
```

**Email** (SMTP)
```json
"Channels": {
  "Email": {
    "Enabled": true,
    "SmtpHost": "smtp.example.com",
    "ToAddresses": ["oncall@example.com"]
  }
}
```

### Routing Rules

Define which severities go to which channels:

```json
"Routing": {
  "SeverityRoutes": {
    "critical": ["ntfy", "email"],
    "warning": ["ntfy"],
    "info": ["ntfy"]
  }
}
```

## Getting Started

**Note**: This service is designed to run as part of the larger ATLAS microservices architecture. It receives alerts from Prometheus Alertmanager for infrastructure monitoring.

### Development (Dev Containers)

The most robust way to develop is using the provided Dev Container, which includes the .NET SDK and tooling.

1. **Open in VS Code**: Open this folder and select "Reopen in Container".
2. **Configuration**: Update `src/appsettings.json` or set environment variables for your desired channels.
3. **Run Service**:
   ```bash
   cd src
   dotnet run
   ```

### Running the Full Stack

To run the entire ATLAS system (including AlertService):

```bash
cd ../deployment/ansible
ansible-playbook playbooks/deploy.yml
```

## API Endpoints

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/alerts` | POST | Ingests alerts from internal services or Alertmanager |
| `/health` | GET | Health check (returns 200 OK with status) |

### Alert Payloads

The `/alerts` endpoint accepts two formats:

**1. Direct Format** (Simple JSON)
Used for direct alert submission.

```json
{
  "source": "custom-source",
  "severity": "critical",
  "title": "Alert Title",
  "message": "Alert message describing the issue.",
  "metadata": { "key": "value" }
}
```

**2. Alertmanager Format** (Prometheus Standard)
Primary format used by Prometheus Alertmanager.

```json
{
  "alerts": [
    {
      "status": "firing",
      "labels": { "alertname": "HighCpu", "severity": "warning" },
      "annotations": { "description": "CPU > 90%" }
    }
  ]
}
```

## Project Structure

```
AlertService/
├── src/
│   ├── Channels/               # INotificationChannel implementations (Ntfy, Email)
│   ├── Endpoints/              # API route handlers
│   ├── Models/                 # Domain models (Alert, AlertRequest, Severity)
│   ├── Services/               # Queue, Dispatcher, Router
│   ├── Telemetry/              # OpenTelemetry metrics and tracing
│   ├── Program.cs              # Application entry point
│   ├── appsettings.json        # Configuration
│   └── Containerfile           # Production container
├── tests/                      # Unit tests
└── .devcontainer/
    └── devcontainer.json       # VS Code dev container config
```

## Deployment

AlertService runs on port 8080 internally (no host port exposed by default in compose.yaml).

Deploy via Ansible:
```bash
cd /home/james/ATLAS/deployment/ansible
ansible-playbook playbooks/deploy.yml --tags alert-service
```

Container configuration in `/opt/ai-inference/compose.yaml`:
- Image: `alert-service:latest`
- Internal Port: 8080
- Host Port: Not exposed (internal service only)
- Dependencies: otel-collector

## See Also

- [Prometheus Alertmanager](https://prometheus.io/docs/alerting/latest/alertmanager/) - Alert source
- [Observability](../docs/OBSERVABILITY.md) - Metrics documentation
