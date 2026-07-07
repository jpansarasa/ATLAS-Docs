# AlertService — architecture [agent read-first]

PURPOSE: Alertmanager-webhook sink -> severity-route -> fan-out(ntfy/email/autofix). No DB, no gRPC, no MCP, not a metric-source, not the alert-rule-owner (rules live in Prometheus/Alertmanager). HTTP-only, internal :8080, no host port.

DATA MODEL + INVARIANTS (schema/config does NOT enforce):
  OPERATIONAL: UP by design (re-enabled 2026-06-10 #656; verified end-to-end Alertmanager->alert-service->ntfy `atlas-alert`). restart=`unless-stopped`. Webhook target + autofix-queue consumer wired.
  INV queue⊥backpressure: `Channel.CreateUnbounded` -> 503/QueueFull fires ONLY on writer cancel (shutdown), NEVER capacity. depth-gauge != admission gate.
  INV route-table-divergence: appsettings.json (SHIPPED) = critical->[ntfy,email,autofix], error->[ntfy,autofix], warning->[ntfy], info->[ntfy]. error = notify+autofix but NO email/page (that is critical's job — Option A, D-1). C# `RoutingOptions` class default = critical->[ntfy,email], warning->[ntfy], info->[ntfy] (NO autofix, NO `error` key). class default applies ONLY if `Routing` section absent entirely. never read class to know prod routing.
  INV dedup⊥direct: dedup keyed `fingerprint:(status ?? "firing")`, 30-min window, ONLY when Fingerprint non-empty. null Status treated as "firing" in the key. Alertmanager always sets Fingerprint -> deduped; direct-format alerts usually lack it -> NEVER deduped (every POST fires).
  INV dedup-state⊥durable: in-mem `ConcurrentDictionary` -> restart wipes window -> re-fires.
  INV autofix-severity-gate: `AutoFixChannelOptions.EnabledSeverities` (configurable list, default ["critical","error"]) is a HARD gate checked BEFORE rate-limit. severity not in list -> channel returns false, alert dropped silently (no metric bump for severity miss). warning is NOT in the list -> a lone warning never autofixes; only a flood escalated to `error` (D-1) passes.
  INV autofix-opt-out: `alert.Metadata["autofix"] == "disabled"` (case-insensitive) -> channel skips unconditionally, regardless of severity or rate-limit.
  INV autofix-limit⊥per-alert: rate-limit (RateLimitMinutes / MaxSessionsPerDay) is `static` PROCESS-WIDE, shared across ALL alerts+channels, reset on restart + UTC-day rollover. one noisy alert starves autofix for all.
  INV ntfy-priority: NtfyChannel maps severity -> ntfy Priority header: critical->"urgent", error->"high", warning->"high", else->"default". set on every outbound POST; no appsettings override.
  INV severity-fallback: unknown severity -> `info` route (no drop). severity lowercased before lookup.
  INV email-double-gate: enabled = Enabled AND ToAddresses non-empty. ToAddresses=[] => silently disabled even if Enabled=true.

PATHS (one entry path only):
  POST /alerts [HTTP :8080 · Alertmanager + direct callers]
    do: parse dual-format -> 1 webhook->N Alert -> enqueue per-alert -> 202{queued:N}. background dispatcher dedupes -> routes by severity -> fans to enabled channels in PARALLEL (Task.WhenAll).
    does NOT: synchronous-deliver (enqueue then 202 BEFORE send); retry-on-channel-fail; persist; auth-inbound.
    on-miss: 0 valid alerts (no alerts[] AND no title) -> 400 invalid-alert. writer-cancel -> 503 queue-full. channel SendAsync false/throws -> logged + metric, alert DROPPED (no requeue).
  GET /health [HTTP :8080] -> 200 {status:healthy}. liveness only (no queue-depth, no channel-reachability).

PROCESSING MODEL:
  ingest(sync, per-request) -> unbounded Channel -> single NotificationDispatcher BackgroundService (SingleReader) -> DequeueAll loop -> dedup -> severity-route -> channel.SendAsync fan-out. Channel.IsEnabled gated at fan-out (disabled channel = silent skip).
  dual-format parse: alerts[] present -> Alertmanager (Title=labels.alertname, Severity=labels.severity dflt warning, Message=annotations.description??summary, Source="alertmanager", Metadata=labels). else title present -> direct (Severity dflt info, Source dflt unknown).

DISTINCTIONS:
  autofix-channel(writes {sessionId}.json to host-mounted /opt/ai-inference/autofix-queue for OUT-OF-PROCESS host runner) ≠ ntfy/email(in-process HTTP/SMTP send). autofix success = file written, NOT fix applied.
  SendAsync->false (graceful, channel-decided) ≠ throw (exception). both = alerts.failed metric + DROP; neither requeues.
  appsettings route-table ≠ C# class default (autofix presence diverges).
  202 Accepted (queued) ≠ delivered. delivery is async, fire-and-forget, unobserved by caller.

CROSS-SERVICE: Alertmanager -> POST /alerts (webhook inbound). OUT: ntfy HTTP, SMTP email, autofix-queue file-write (host-mounted). No DB, no gRPC, no MCP, no outbound-registration. FEEDS: host autofix runner (out-of-process).

GOTCHAS:
  ✗ treat-503-as-backpressure (unbounded -> only shutdown)
  ✗ read-RoutingOptions-class-for-prod-routing (appsettings overrides)
  ✗ expect-dedup-on-direct-alerts (no fingerprint -> none)
  ✗ assume-autofix-limit-per-alert (static, process-wide, restart-reset)
  ✗ assume-autofix-fires-for-info-or-warning (EnabledSeverities gate blocks both — default list is [critical,error]; a lone warning never autofixes, only a flood escalated to error does — D-1)
  ✗ forget-metadata-autofix-disabled (per-alert opt-out checked before rate-limit and severity gate)
  ✗ assume-ntfy-sends-without-priority (Priority header always set: critical->urgent, error->high, warning->high, else default)
  ✗ assume-null-Status-dedup-key-is-empty (null Status -> "firing" in dedup key, not empty-string)
  ✗ assume-202-means-sent ✗ expect-channel-retry ✗ Email-Enabled-without-ToAddresses
  ✗ look-for-gRPC/MCP/DB/inbound-auth (none exist)

DECISIONS:
  D-1 warning-flood-escalation: INTENT individual warnings are dashboard-only and never auto-fixed (#757); but a SUSTAINED per-service warning FLOOD drowns real signal (2026-06-24 CatalogInstrumentMissingSector ~109k/day = ~90% of all platform warnings, unnoticed for weeks) so a flood MUST escalate to autofix. Option A (user-chosen): escalate the flood to `error` severity = autofix + ntfy-notify, NOT `critical` (so no page/email). / PRECOND sustained per-service Warning-log rate above threshold (>500/15m) held for 30m -> ServiceLogWarningsElevated emits `severity: error`; a lone or transient warning does NOT qualify. / GUARD three-part chain, change-all-or-none: (1) ServiceLogWarningsElevated Loki rule [deployment/artifacts/monitoring/loki-rules/fake/atlas-logs.yaml] emits error ONLY on sustained flood; (2) alertmanager `error` route -> alert-service, then appsettings error route = [ntfy,autofix] (no email); (3) AutoFixChannel.IsAutoFixEnabled EnabledSeverities gate stays [critical,error] @ AutoFixChannel.cs — admits `error`, rejects `warning`. / TEST AutoFixChannelTests.SendAsync_ErrorSeverity_DefaultGate_QueuesFile (error passes gate) + SendAsync_WarningSeverity_DefaultGate_DoesNotQueue (warning rejected). / SUPERSEDES #757's absolute "a warning never reaches autofix" — narrowed: an INDIVIDUAL warning still never autofixes (unchanged); only a flood escalated to error does.

SEE: README.md (config tables, metrics, payload formats, compose volumes/healthcheck) · NotificationDispatcher.cs:62-83(dedup+null-status-fallback) · NotificationDispatcher.cs:88-110(route) · AlertEndpoints.cs:78-118(dual-format parse) · AutoFixChannel.cs:118-136(IsAutoFixEnabled: opt-out + severity gate) · AutoFixChannel.cs:9-23(EnabledSeverities default + D-1 comment) · AutoFixChannel.cs:25-31,139-174(static rate-limit) · NtfyChannel.cs:91-105(MapSeverityToPriority/Tags) · AlertQueue.cs:19(unbounded) · appsettings.json:45-52 vs RoutingOptions(NotificationDispatcher.cs:11-19)(route divergence)
