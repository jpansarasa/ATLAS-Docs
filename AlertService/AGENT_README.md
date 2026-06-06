# AlertService — architecture [agent read-first]

PURPOSE: Alertmanager-webhook sink → severity-route → fan-out(ntfy/email/autofix). ¬DB ¬gRPC ¬MCP ¬metric-source ¬alert-rule-owner(rules live in Prometheus/Alertmanager). HTTP-only, internal :8080, no host port.

DATA MODEL + INVARIANTS (schema/config does NOT enforce):
  OPERATIONAL ⊥ deploy: service kept DOWN (PoC window — output not useful). restart=`unless-stopped` → ANY non-scoped ansible run RESURRECTS it. ✗scope: `-e scoped_restart=true scoped_services=...` to avoid. Webhook target + autofix-queue consumer stay wired while down.
  INV queue⊥backpressure: `Channel.CreateUnbounded` → 503/QueueFull fires ONLY on writer cancel (shutdown), NEVER capacity. depth-gauge ≠ admission gate.
  INV route-table-divergence: appsettings.json (SHIPPED) = critical→[ntfy,email,autofix], warning→[ntfy,autofix], info→[ntfy]. C# `RoutingOptions` class default = critical→[ntfy,email], warning→[ntfy], info→[ntfy] (NO autofix). class default applies ONLY if `Routing` section absent entirely. ¬read class to know prod routing.
  INV dedup⊥direct: dedup keyed `fingerprint:(status ?? "firing")`, 30-min window, ONLY when Fingerprint non-empty. null Status treated as "firing" in the key. Alertmanager always sets Fingerprint → deduped; direct-format alerts usually lack it → NEVER deduped (every POST fires).
  INV dedup-state⊥durable: in-mem `ConcurrentDictionary` → restart wipes window → re-fires.
  INV autofix-severity-gate: `AutoFixChannelOptions.EnabledSeverities` (configurable list, default ["critical","warning"]) is a HARD gate checked BEFORE rate-limit. severity not in list → channel returns false, alert dropped silently (no metric bump for severity miss).
  INV autofix-opt-out: `alert.Metadata["autofix"] == "disabled"` (case-insensitive) → channel skips unconditionally, regardless of severity or rate-limit.
  INV autofix-limit⊥per-alert: rate-limit (RateLimitMinutes / MaxSessionsPerDay) is `static` PROCESS-WIDE, shared across ALL alerts+channels, reset on restart + UTC-day rollover. one noisy alert starves autofix for all.
  INV ntfy-priority: NtfyChannel maps severity → ntfy Priority header: critical→"urgent", warning→"high", else→"default". set on every outbound POST; no appsettings override.
  INV severity-fallback: unknown severity → `info` route (¬drop). severity lowercased before lookup.
  INV email-double-gate: enabled = Enabled ∧ ToAddresses non-empty. ToAddresses=[] ⇒ silently disabled even if Enabled=true.

PATHS (one entry path only):
  POST /alerts [HTTP :8080 · Alertmanager + direct callers]
    do: parse dual-format → 1 webhook→N Alert → enqueue per-alert → 202{queued:N}. background dispatcher dedupes → routes by severity → fans to enabled channels in PARALLEL (Task.WhenAll).
    does NOT: ¬synchronous-deliver (enqueue then 202 BEFORE send) ¬retry-on-channel-fail ¬persist ¬auth-inbound.
    on-miss: 0 valid alerts (no alerts[] ∧ no title) → 400 invalid-alert. writer-cancel → 503 queue-full. channel SendAsync false/throws → logged + metric, alert DROPPED (no requeue).
  GET /health [HTTP :8080] → 200 {status:healthy}. liveness only (¬queue-depth ¬channel-reachability).

PROCESSING MODEL:
  ingest(sync, per-request) → unbounded Channel → single NotificationDispatcher BackgroundService (SingleReader) → DequeueAll loop → dedup → severity-route → channel.SendAsync fan-out. Channel.IsEnabled gated at fan-out (disabled channel = silent skip).
  dual-format parse: alerts[] present → Alertmanager (Title=labels.alertname, Severity=labels.severity dflt warning, Message=annotations.description??summary, Source="alertmanager", Metadata=labels). else title present → direct (Severity dflt info, Source dflt unknown).

DISTINCTIONS:
  autofix-channel(writes {sessionId}.json to host-mounted /opt/ai-inference/autofix-queue for OUT-OF-PROCESS host runner) ≠ ntfy/email(in-process HTTP/SMTP send). autofix success = file written, NOT fix applied.
  SendAsync→false (graceful, channel-decided) ≠ throw (exception). both = alerts.failed metric + DROP; neither requeues.
  appsettings route-table ≠ C# class default (autofix presence diverges).
  202 Accepted (queued) ≠ delivered. delivery is async, fire-and-forget, unobserved by caller.

CROSS-SERVICE: Alertmanager → POST /alerts (webhook inbound). OUT: ntfy HTTP, SMTP email, autofix-queue file-write (host-mounted). ¬DB ¬gRPC ¬MCP ¬outbound-registration. FEEDS: host autofix runner (out-of-process).

GOTCHAS:
  ✗ non-scoped-ansible-deploy (resurrects a service the user keeps DOWN)
  ✗ treat-503-as-backpressure (unbounded → only shutdown)
  ✗ read-RoutingOptions-class-for-prod-routing (appsettings overrides)
  ✗ expect-dedup-on-direct-alerts (no fingerprint → none)
  ✗ assume-autofix-limit-per-alert (static, process-wide, restart-reset)
  ✗ assume-autofix-fires-for-info (EnabledSeverities gate blocks it — default list is [critical,warning])
  ✗ forget-metadata-autofix-disabled (per-alert opt-out checked before rate-limit and severity gate)
  ✗ assume-ntfy-sends-without-priority (Priority header always set: critical→urgent, warning→high, else default)
  ✗ assume-null-Status-dedup-key-is-empty (null Status → "firing" in dedup key, ¬empty-string)
  ✗ assume-202-means-sent ✗ expect-channel-retry ✗ Email-Enabled-without-ToAddresses
  ✗ look-for-gRPC/MCP/DB/inbound-auth (none exist)

SEE: README.md (config tables, metrics, payload formats, compose volumes/healthcheck) · NotificationDispatcher.cs:62-83(dedup+null-status-fallback) · NotificationDispatcher.cs:88-110(route) · AlertEndpoints.cs:78-118(dual-format parse) · AutoFixChannel.cs:111-130(IsAutoFixEnabled: opt-out + severity gate) · AutoFixChannel.cs:9-16(EnabledSeverities default) · AutoFixChannel.cs:21-24,132-171(static rate-limit) · NtfyChannel.cs:90-95(MapSeverityToPriority) · AlertQueue.cs:19(unbounded) · appsettings.json:44-50 vs RoutingOptions(NotificationDispatcher.cs:11-19)(route divergence)
