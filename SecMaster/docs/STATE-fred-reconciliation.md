# STATE — FRED catalog reconciliation (feat/secmaster/fred-catalog-reconciliation)

## GOAL
Durable self-healing reconciliation: keep SecMaster `instruments` in sync with the FredCollector
catalog. Fix PR #330's collateral mis-quarantine of 15 legit FRED macro indicators + register the
never-registered `Y033RC1Q027SBEA`.

accept_criteria:
- Hosted service runs on STARTUP + guarded admin endpoint to re-run.
- Source of truth = FredCollector `/api/series` (via `FredCollectorClient.GetActiveSeriesAsync`).
- REACTIVATE only quarantined instruments carrying an ACTIVE FredCollector `is_primary` mapping.
- REGISTER missing series via trusted `IRegistrationService.RegisterAsync` (Collector=FredCollector).
- NEVER touch SentinelCollector-only / hallucinated rows (trust boundary — do not undermine #330).
- Idempotent; INFO-log per action; metric counters; config kill-switch.

## VERIFIED FACTS (prod inspection, read-only)
- 365 total Quarantined-LoRA-Era rows. EXACTLY 15 carry an active FredCollector is_primary mapping:
  CCSA DGS10 GDP GDPC1 HOUST ICSA IPMAN MSPUS PCE PCEPI PI RSAFS RSXFS T5YIE TCU.
- 14 of those 15 also carry a stray active SentinelCollector non-primary mapping (PCEPI has none) → deactivate those.
- Y033RC1Q027SBEA: 0 instruments in SecMaster → REGISTER.
- All 16 are IsActive=true in FredCollector catalog (76 series total).
- 0 quarantined rows have a non-primary/inactive FredCollector mapping → filter has no edge cases.
- FredCollector registers with Collector="FredCollector", AssetClass="Economic", SourceId=SeriesId.

## FILTER (trust boundary)
REACTIVATE iff: instrument.is_active=false AND active FredCollector is_primary source mapping exists.
(asset_class value 'Quarantined-LoRA-Era' is NOT part of the gate — the FredCollector is_primary
mapping is the sole trust signal, more robust than a string match.)

## ARCH
- New `FredCatalogReconciliationOptions` (kill-switch RunOnStartup).
- New `IFredCatalogReconciliationService` + impl (the filter + reactivate/register logic).
- New repo method: GetBySymbolIncludingInactiveAsync (GetBySymbolAsync filters active → can't see quarantined).
- New `FredCatalogReconciliationHostedService` (startup, mirrors SignalIdentityImporterHostedService).
- Guarded admin endpoint in AdminEndpoints: POST /api/admin/catalog/reconcile-fred (re-run).
- Metric: secmaster.fred_reconciliation.{reconciled,reactivated,registered}.

## STATUS
- ✓done: implementation (no schema change — pure data reconciliation via existing entities)
- ✓done: unit filter tests (10) + integration tests (2) — all pass
- ✓done: full build clean (0 err / 0 warn); UnitTests 972 pass, IntegrationTests 133 pass
- ◯next: PR (no merge / no deploy)

## FIRST PROD RUN (verified against live prod DB + FredCollector catalog)
- REACTIVATE (15): CCSA DGS10 GDP GDPC1 HOUST ICSA IPMAN MSPUS PCE PCEPI PI RSAFS RSXFS T5YIE TCU
- REGISTER (1): Y033RC1Q027SBEA
- UNTOUCHED: 350 SentinelCollector-only quarantined rows (trust boundary)
- NO-OP: 60 already-active FRED series
