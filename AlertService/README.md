# AlertService

Receives alert webhooks, queues them, and dispatches notifications to configured channels based on severity routing rules.

## Architecture

```
Webhook → Queue → Dispatcher → [Ntfy, Email] Channels
```

1. **Webhook endpoint** accepts alerts (direct or Alertmanager format)
2. **In-memory queue** buffers alerts for processing
3. **Background dispatcher** routes alerts to channels by severity
4. **Notification channels** send alerts to external services

Full OpenTelemetry instrumentation for traces, metrics, and logs.

## Endpoints

### POST /alerts

Accepts alerts in two formats:

**Direct format:**
```json
{
  "source": "my-service",
  "severity": "critical",
  "title": "Service down",
  "message": "Service X is unresponsive",
  "metadata": {
    "host": "prod-server-01"
  }
}
```

**Alertmanager webhook format:**
```json
{
  "alerts": [
    {
      "status": "firing",
      "labels": {
        "alertname": "HighMemoryUsage",
        "severity": "warning",
        "instance": "server-02"
      },
      "annotations": {
        "summary": "Memory usage above 90%",
        "description": "Memory usage has exceeded threshold"
      },
      "startsAt": "2025-11-23T10:00:00Z"
    }
  ]
}
```

Alertmanager alerts are automatically transformed:
- `labels.alertname` → Title
- `labels.severity` → Severity
- `annotations.description` or `annotations.summary` → Message
- All labels → Metadata

**Response:** `202 Accepted` with `{"queued": N}`

### GET /health

Health check endpoint. Returns `200 OK` with `{"status": "healthy"}`.

## Configuration

All configuration via `appsettings.json` or environment variables (format: `Section__Key`).

### Notification Channels

#### Ntfy (Push Notifications)

```json
{
  "Channels": {
    "Ntfy": {
      "Enabled": true,
      "Endpoint": "https://ntfy.sh",
      "Topic": "atlas-alerts",
      "Username": "",
      "Password": ""
    }
  }
}
```

Maps severity to ntfy priority and tags:
- `critical` → `urgent` priority, 🚨 tags
- `warning` → `high` priority, ⚠️ tags
- `info` → `default` priority, ℹ️ tags

#### Email (SMTP)

```json
{
  "Channels": {
    "Email": {
      "Enabled": false,
      "SmtpHost": "smtp.example.com",
      "SmtpPort": 587,
      "UseSsl": true,
      "Username": "alerts@example.com",
      "Password": "your-password",
      "FromAddress": "alerts@atlas.local",
      "FromName": "ATLAS Alerts",
      "ToAddresses": ["oncall@example.com"]
    }
  }
}
```

### Routing

Routes alerts to channels by severity:

```json
{
  "Routing": {
    "SeverityRoutes": {
      "critical": ["ntfy", "email"],
      "warning": ["ntfy"],
      "info": ["ntfy"]
    }
  }
}
```

Unknown severities fall back to `info` routing.

### OpenTelemetry

```json
{
  "OpenTelemetry": {
    "OtlpEndpoint": "http://otel-collector:4317",
    "ServiceName": "alert-service",
    "ServiceVersion": "1.0.0"
  }
}
```

Exports traces and metrics to OTLP endpoint. Logs go to console and OTLP.

## Development

Uses devcontainer for consistent development environment.

**Start development:**
```bash
# Open in devcontainer (VSCode will prompt)
# Or manually:
cd .devcontainer
docker compose up -d
```

**Run service:**
```bash
cd src/AlertService
dotnet run
```

Service listens on http://localhost:8080

**Test webhook:**
```bash
curl -X POST http://localhost:8080/alerts \
  -H "Content-Type: application/json" \
  -d '{
    "source": "test",
    "severity": "warning",
    "title": "Test alert",
    "message": "This is a test"
  }'
```

## Deployment

**Build container:**
```bash
cd src/AlertService
docker build -f Containerfile -t alert-service:latest .
```

**Run container:**
```bash
docker run -p 8080:8080 \
  -e Channels__Ntfy__Topic=my-alerts \
  -e Channels__Email__Enabled=true \
  -e Channels__Email__ToAddresses__0=oncall@example.com \
  alert-service:latest
```

## Observability

### Metrics (Prometheus)

- `alert_service_alerts_received_total{source, severity}` - Alerts received
- `alert_service_alerts_queued_total{severity}` - Alerts queued
- `alert_service_alerts_dispatched_total{severity, channel_count}` - Alerts dispatched
- `alert_service_alerts_sent_total{channel, severity}` - Successful sends
- `alert_service_alerts_failed_total{channel, severity, reason}` - Failed sends
- `alert_service_channel_send_duration_ms{channel}` - Channel send duration histogram
- `alert_service_processing_duration_ms{severity}` - Alert processing duration
- `alert_service_queue_depth` - Current queue size

### Traces (Tempo/Jaeger)

Traces for:
- `AlertEndpoints.HandleAlert` - Webhook request handling
- `NotificationDispatcher.DispatchAlert` - Routing and dispatch
- `NotificationDispatcher.SendToChannel` - Individual channel sends
- `NtfyChannel.Send` / `EmailChannel.Send` - Channel implementations

### Logs (Loki)

Structured logs with OpenTelemetry trace correlation:
- Alert received, queued, sent
- Channel failures with context
- Dispatcher startup with enabled channels

Logs include `SpanId` and `TraceId` for correlation with distributed traces. Serilog's native span support automatically enriches log entries with trace context from OpenTelemetry Activity.

Log level defaults to `Warning` for production.

## Integration Examples

### Prometheus Alertmanager

Add to Alertmanager `alertmanager.yml`:

```yaml
receivers:
  - name: 'atlas'
    webhook_configs:
      - url: 'http://alert-service:8080/alerts'
        send_resolved: false

route:
  receiver: 'atlas'
  group_wait: 10s
  group_interval: 10s
  repeat_interval: 1h
```

### Direct Webhook

From any service:

```python
import requests

requests.post('http://alert-service:8080/alerts', json={
    'source': 'my-app',
    'severity': 'critical',
    'title': 'Database connection failed',
    'message': 'Unable to connect to primary database',
    'metadata': {'db': 'postgres-prod', 'retry_count': 3}
})
```

## Channel Implementation

To add a new channel, implement `INotificationChannel`:

```csharp
public interface INotificationChannel
{
    string Name { get; }
    bool IsEnabled { get; }
    Task<bool> SendAsync(Alert alert, CancellationToken cancellationToken = default);
}
```

Register in `Program.cs`:
```csharp
builder.Services.AddSingleton<INotificationChannel, MyChannel>();
```

Add to routing configuration to use.

