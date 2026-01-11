# Loki Log Analysis - 7 Day Review (2026-01-04 to 2026-01-11)

## GOAL
Analyze and remediate warnings/errors from production logs.

## ISSUES FOUND

### P1 - CRITICAL (blocking functionality)

#### 1. sentinel-collector: ThresholdEngine gRPC unavailable ⚠️ ROOT CAUSE FOUND
- **Symptom**: "ThresholdEngine unavailable, will retry in 60 seconds" logged every 60s continuously
- **Frequency**: ~1440/day (every minute)
- **Source**: `ValidationEventConsumerWorker.cs`
- **Root cause**: **threshold-engine container is in a zombie state**
  - Container shows as "running" in `nerdctl compose ps`
  - Process PID 1 exists (`dotnet ThresholdEngine.dll`)
  - Port 5001 returns "Connection refused"
  - Port 8080 times out
  - Last log entry was Jan 8 23:21:31 - **3 days ago**
  - Container is unresponsive/hung
- **Impact**: Sentinel cannot receive threshold crossed events for validation query generation
- **Fix**: Restart threshold-engine container: `sudo nerdctl compose -f /opt/ai-inference/compose.yaml restart threshold-engine`

#### 2. sentinel-collector: EdgeSync ack returning InternalServerError
- **Symptom**: "Edge ack POST /sync/ack failed... with status InternalServerError"
- **Frequency**: ~100+/day
- **Source**: `EdgeSyncWorker.cs` → `EdgeSyncClient.cs:69`
- **Root cause**: Cloudflare Workers edge service at `sentinel-edge.james-pansarasa.workers.dev` returning HTTP 500
- **Impact**: Processed content cannot be acknowledged, causing reprocessing
- **Investigation needed**: Check Cloudflare Workers logs, verify D1 database connectivity

---

### P2 - MODERATE (degraded behavior)

#### 3. fred-collector: TEDRATE series returning no observations
- **Symptom**: "No observations returned for series TEDRATE"
- **Frequency**: Daily (once per collection cycle)
- **Source**: `DataCollectionService.cs`
- **Root cause**: FRED API returning empty data for this series
- **Impact**: Missing TED Spread rate data
- **Investigation needed**: Check FRED API for TEDRATE status, may be discontinued

#### 4. fred-collector: FRED API BadGateway retries
- **Symptom**: "FRED API request failed (attempt 1/3). Retrying... Error: BadGateway"
- **Frequency**: Sporadic (~1-2/day)
- **Source**: `FredApiClient.cs`
- **Root cause**: Upstream FRED API instability
- **Impact**: Temporary delays, recovers with retry
- **Status**: Acceptable - retry logic working as designed

---

### P3 - LOW (cosmetic / informational)

#### 5. Multiple services: Startup messages logged at Warning level
- **Symptom**: Informational startup banners logged as Warning
- **Examples**: "FRED Collector", "Environment: Production", "Database migrations applied"
- **Root cause**: Incorrect log level in startup code
- **Impact**: Log noise, pollutes warning filters
- **Fix**: Change startup logging from `LogWarning` to `LogInformation`

#### 6. fred-collector: DataProtection container warnings
- **Symptoms**:
  - "Storing keys in a directory... may not be persisted outside container"
  - "No XML encryptor configured. Key may be persisted in unencrypted form"
- **Root cause**: ASP.NET DataProtection using default ephemeral storage
- **Impact**: Low - no features rely on DataProtection persistence
- **Fix**: Either suppress warnings or configure persistent key storage

#### 7. SecMaster: HTTP/2 not enabled warning
- **Symptom**: "HTTP/2 is not enabled for [::]:8080... TLS is not enabled"
- **Root cause**: Internal services using HTTP without TLS
- **Impact**: None - internal traffic uses HTTP/1.1 fine
- **Status**: Informational only, can suppress

#### 8. threshold-engine: OfrCollector startup retry
- **Symptom**: "OfrCollector not ready, retry 1/5 in 2s"
- **Frequency**: Once per service restart
- **Root cause**: OfrCollector slower to start than threshold-engine
- **Impact**: None - recovers on retry
- **Status**: Normal startup behavior

---

## REMEDIATION PLAN

### Immediate (P1)

| Task | Action | Files |
|------|--------|-------|
| Fix ThresholdEngine gRPC connectivity | Add port 5001 exposure in compose for threshold-engine, or fix network connectivity | `/opt/ai-inference/compose.yaml` |
| Debug EdgeSync 500 errors | Check Cloudflare Workers dashboard, review D1 database state | External - Cloudflare dashboard |

### Short-term (P2)

| Task | Action | Files |
|------|--------|-------|
| Investigate TEDRATE series | Check FRED API status for TEDRATE, possibly discontinue or replace | `FredCollector/src/Configuration/` |

### Low priority (P3)

| Task | Action | Files |
|------|--------|-------|
| Fix startup log levels | Change startup banners from LogWarning to LogInformation | Multiple `Program.cs` files |
| Suppress DataProtection warnings | Add DataProtection config or suppress log category | `FredCollector/src/Program.cs` |

---

## STATUS
- ✓ Log analysis complete
- ◯ Investigate ThresholdEngine gRPC connectivity
- ◯ Debug EdgeSync 500 errors
- ◯ Check TEDRATE series status
- ◯ Fix startup log levels
