# Pattern Weighting & Temporal Metadata

**Epic**: ThresholdEngine Signal Quality Enhancement  
**Created**: 2025-12-14  
**Status**: ◯ Specification Ready  
**Owner**: Product  
**Implementer**: Engineering  
**Version**: 1.2  

## GOAL

Enhance ThresholdEngine pattern evaluation from naive equal-weighting to sophisticated signal quality scoring incorporating:
- Individual pattern reliability weights (0.0-1.0)
- Temporal classification (Leading, Coincident, Lagging)
- Publication frequency awareness (decay only overdue data)
- Signal freshness decay based on data staleness
- Regime-aware temporal weighting adjustments

### Accept Criteria
- [ ] All 58 patterns have weight, temporal, publication frequency metadata defined
- [ ] Publication frequency validation framework implemented and all patterns pass
- [ ] Domain expert review and sign-off on all publication frequencies
- [ ] Pattern schema extended and validated (JSON Schema) - 6 new fields
- [ ] Database schema supports new fields (EF migration with data updates)
- [ ] Evaluation engine applies weighted scoring
- [ ] Signal freshness decay implemented with publication-frequency-aware exponential curve
- [ ] Multi-series pattern handling implemented (use slowest publication frequency)
- [ ] Health checks for missing/severely stale data with alerting
- [ ] Temporal multipliers applied during regime transitions
- [ ] MCP endpoints expose new metadata
- [ ] Unit tests validate weighted vs naive scoring differences
- [ ] Unit tests validate publication frequency awareness (monthly data at 15 days = 100% fresh)
- [ ] Grafana dashboards visualize pattern contributions and data staleness
- [ ] Documentation updated (ARCHITECTURE.md, ThresholdEngine/README.md)
- [ ] Zero breaking changes to existing pattern evaluation API
- [ ] Hot reload continues to function with new fields
- [ ] Rollback procedure documented and tested

### Constraints
- Backward compatible: patterns without weights default to 1.0
- Configuration over code: all weights tunable via JSON
- No performance regression in evaluation loop
- Preserve existing pattern evaluation contract

---

## ARCHITECTURE

### Current State (Naive Equal-Weighting)

```csharp
// MacroScoreCalculator.cs - Current Implementation
var categoryScore = patterns
    .Where(p => p.enabled)
    .Average(p => EvaluatePattern(p).signal);

var macroScore = categories
    .Sum(c => c.categoryScore * c.categoryWeight);
```

**Problem**: Sahm Rule (100% historical accuracy) contributes equally to Consumer Sentiment (noisy, lagging).

### Target State (Weighted + Temporal + Freshness)

```csharp
// MacroScoreCalculator.cs - Target Implementation
var categoryScore = patterns
    .Where(p => p.enabled && IsApplicableToRegime(p))
    .Sum(p => GetWeightedSignal(p)) 
    / patterns.Sum(p => p.weight);

decimal GetWeightedSignal(Pattern p, decimal rawSignal) {
    var freshness = CalculateFreshnessFactor(p);
    var temporalMultiplier = GetTemporalMultiplier(p);
    return rawSignal * p.weight * freshness * temporalMultiplier;
}
```

**Benefit**: High-quality signals (Sahm, Yield Curve) contribute proportionally more than low-quality signals. Current data maintains full strength; only overdue data decays.

### Design Decisions

| Decision | Rationale | Implication |
|----------|-----------|-------------|
| **Publication frequency awareness** | ISM 15 days old is current, not stale | Requires `publicationFrequencyDays` per pattern |
| **Multi-series: use slowest frequency** | Pattern only "current" when all data is current | Composite patterns remain conservative |
| **Exponential decay after overdue** | Models information half-life naturally | Requires `signalDecayDays` per pattern |
| **Health checks for stale data** | Alert on data feed failures | Requires monitoring infrastructure |
| **Temporal multiplier** during transitions | Boost leading indicators when regime changing | Requires macro score trend tracking |
| **Category normalization** by sum of weights | Prevents category dilution with many patterns | Category scores remain comparable |
| **Default weight = 1.0** | Backward compatible, explicit opt-in | Existing patterns work unchanged |
| **Metadata in JSON** | Hot reload without recompile | Pattern config files grow ~50% |

---

## SCHEMA_CHANGES

### 1. Pattern Configuration Schema

**File**: `ThresholdEngine/config/pattern-schema.json`

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "required": ["patternId", "name", "category", "expression"],
  "properties": {
    "patternId": {"type": "string"},
    "name": {"type": "string"},
    "category": {"enum": ["Recession", "Liquidity", "Growth", "NBFI", "Valuation", "Inflation", "Commodity", "Currency"]},
    "expression": {"type": "string"},
    "signalExpression": {"type": "string"},
    
    // NEW FIELDS
    "weight": {
      "type": "number",
      "minimum": 0.0,
      "maximum": 1.0,
      "default": 1.0,
      "description": "Pattern reliability weight. 0.9+ = proven, 0.6-0.8 = solid, 0.4-0.5 = supplementary"
    },
    "temporalType": {
      "enum": ["Leading", "Coincident", "Lagging"],
      "default": "Coincident",
      "description": "Signal timing relative to economic events (NOT publication timing)"
    },
    "leadTimeMonths": {
      "type": "integer",
      "minimum": -12,
      "maximum": 24,
      "default": 0,
      "description": "Months ahead (positive) or behind (negative) of event"
    },
    "publicationFrequencyDays": {
      "type": "integer",
      "minimum": 0,
      "maximum": 365,
      "default": 30,
      "description": "Expected days between publications. 0=real-time, 1=daily, 7=weekly, 30=monthly, 90=quarterly. For multi-series patterns, use slowest (most restrictive) frequency."
    },
    "signalDecayDays": {
      "type": "integer",
      "minimum": 1,
      "maximum": 365,
      "default": 30,
      "description": "Days until signal loses 63% strength AFTER becoming overdue (1/e). Fast=7, Moderate=30, Slow=180. Generally >= publicationFrequencyDays except for persistent signals."
    },
    "confidence": {
      "type": "number",
      "minimum": 0.0,
      "maximum": 1.0,
      "default": 0.75,
      "description": "Historical accuracy / reliability. Informational only, not used in calculation"
    },
    
    // EXISTING FIELDS (unchanged)
    "applicableRegimes": {"type": "array"},
    "requiredSeries": {"type": "array"},
    "maxLookbackDays": {"type": "integer"},
    "enabled": {"type": "boolean"},
    "metadata": {"type": "object"}
  }
}
```

### 2. Database Schema (EF Core)

**Entity**: `PatternConfiguration` (ThresholdEngine.Infrastructure/Data/Entities/)

```csharp
public class PatternConfiguration
{
    // Existing fields...
    public string PatternId { get; set; }
    public string Name { get; set; }
    public PatternCategory Category { get; set; }
    // ...

    // NEW FIELDS
    public decimal Weight { get; set; } = 1.0m;
    public TemporalType TemporalType { get; set; } = TemporalType.Coincident;
    public int LeadTimeMonths { get; set; } = 0;
    public int PublicationFrequencyDays { get; set; } = 30;
    public int SignalDecayDays { get; set; } = 30;
    public decimal Confidence { get; set; } = 0.75m;
}

public enum TemporalType
{
    Leading,      // Signal predicts events 3+ months ahead
    Coincident,   // Signal reflects current state
    Lagging       // Signal confirms past events
}
```

### 3. Publication Frequency Reference Table

**File**: `ThresholdEngine/config/series-publication-frequencies.json` (NEW)

```json
{
  "description": "Reference table for validation: series_id -> publication_frequency_days",
  "frequencies": {
    "VIXCLS": 0,
    "DEXUSEU": 0,
    "BAMLH0A0HYM2": 0,
    "DGS10": 1,
    "DGS2": 1,
    "RRPONTSYD": 1,
    "WRESBAL": 7,
    "ICSA": 7,
    "CCSA": 7,
    "NAPM": 30,
    "UNRATE": 30,
    "CPIAUCSL": 30,
    "PCEPI": 30,
    "INDPRO": 30,
    "GDP": 90
  }
}
```

**Migration**: `Add_PatternWeightingFields`

```csharp
protected override void Up(MigrationBuilder migrationBuilder)
{
    migrationBuilder.AddColumn<decimal>(
        name: "weight",
        table: "pattern_configurations",
        type: "decimal(3,2)",
        nullable: false,
        defaultValue: 1.0m);

    migrationBuilder.AddColumn<string>(
        name: "temporal_type",
        table: "pattern_configurations",
        type: "varchar(20)",
        nullable: false,
        defaultValue: "Coincident");

    migrationBuilder.AddColumn<int>(
        name: "lead_time_months",
        table: "pattern_configurations",
        nullable: false,
        defaultValue: 0);

    migrationBuilder.AddColumn<int>(
        name: "publication_frequency_days",
        table: "pattern_configurations",
        nullable: false,
        defaultValue: 30);

    migrationBuilder.AddColumn<int>(
        name: "signal_decay_days",
        table: "pattern_configurations",
        nullable: false,
        defaultValue: 30);

    migrationBuilder.AddColumn<decimal>(
        name: "confidence",
        table: "pattern_configurations",
        type: "decimal(3,2)",
        nullable: false,
        defaultValue: 0.75m);

    // Update existing patterns with correct publication frequencies
    // Real-time patterns (0 days)
    migrationBuilder.Sql(@"
        UPDATE pattern_configurations 
        SET publication_frequency_days = 0, signal_decay_days = 7
        WHERE pattern_id IN ('vix-deployment-l1', 'vix-deployment-l2', 'credit-spread-widening', 'dxy-risk-off', 'nbfi-hy-spreads');
    ");
    
    // Daily patterns (1 day)
    migrationBuilder.Sql(@"
        UPDATE pattern_configurations 
        SET publication_frequency_days = 1, signal_decay_days = 14
        WHERE pattern_id IN ('standing-repo-stress', 'reverse-repo-liquidity', 'yield-curve-inversion');
    ");
    
    // Weekly patterns (7 days)
    migrationBuilder.Sql(@"
        UPDATE pattern_configurations 
        SET publication_frequency_days = 7, signal_decay_days = 30
        WHERE pattern_id IN ('initial-claims-spike', 'continuing-claims-warning', 'fed-liquidity-contraction');
    ");
    
    // Quarterly patterns (90 days)
    migrationBuilder.Sql(@"
        UPDATE pattern_configurations 
        SET publication_frequency_days = 90, signal_decay_days = 90
        WHERE pattern_id IN ('gdp-acceleration', 'buffett-indicator');
    ");
    
    // Yield curve gets special treatment: daily data, very persistent signal
    migrationBuilder.Sql(@"
        UPDATE pattern_configurations 
        SET publication_frequency_days = 1, signal_decay_days = 180
        WHERE pattern_id = 'yield-curve-inversion';
    ");
}

protected override void Down(MigrationBuilder migrationBuilder)
{
    migrationBuilder.DropColumn(name: "weight", table: "pattern_configurations");
    migrationBuilder.DropColumn(name: "temporal_type", table: "pattern_configurations");
    migrationBuilder.DropColumn(name: "lead_time_months", table: "pattern_configurations");
    migrationBuilder.DropColumn(name: "publication_frequency_days", table: "pattern_configurations");
    migrationBuilder.DropColumn(name: "signal_decay_days", table: "pattern_configurations");
    migrationBuilder.DropColumn(name: "confidence", table: "pattern_configurations");
}
```

---

## IMPLEMENTATION_PLAN

### Phase 1: Schema & Data (Week 1)

**Tasks**:
1. **Create publication frequency reference table**
   - Research publication schedule for each FRED/Finnhub/OFR series
   - Document in `series-publication-frequencies.json`
   - Get domain expert review and sign-off

2. **Update pattern schema**
   - Add 6 new fields to `pattern-schema.json`
   - Update validation rules

3. **Update all 58 pattern JSON files** (detailed process):
   - For each pattern:
     a. Identify all `requiredSeries` (may be 1 or multiple)
     b. Look up publication frequency for each series in reference table
     c. Set `publicationFrequencyDays = MAX(series frequencies)` (most restrictive)
        - Example: NBFI Escalation uses HY spreads (0), Standing Repo (1), NFCI (30) → use 30
     d. Set `signalDecayDays` based on signal persistence (see weight assignment guide)
     e. Validate: `signalDecayDays >= publicationFrequencyDays` OR document exception
        - Exception example: Yield curve has pubFreq=1, decay=180 (very persistent)
     f. Set `weight` based on 4 criteria (accuracy, timeliness, clarity, quality)
     g. Set `temporalType` based on when signal predicts (Leading/Coincident/Lagging)
     h. Set `confidence` based on historical track record

4. **Create validation tests**
   - Automated test: Compare pattern `publicationFrequencyDays` against reference table
   - Automated test: Verify multi-series patterns use MAX(frequencies)
   - Automated test: Verify `signalDecayDays >= publicationFrequencyDays` (with documented exceptions)
   - Manual checklist: Domain expert reviews all 58 configurations

5. **Create EF migration**
   - Add columns with defaults
   - Include SQL updates for existing patterns
   - Test on dev database (fresh + existing data)

6. **Verify hot reload**
   - Change weight in JSON file
   - Verify pattern reloads within 5 seconds
   - Verify new fields are picked up

**Files Changed**:
- `ThresholdEngine/config/series-publication-frequencies.json` (NEW)
- `ThresholdEngine/config/pattern-schema.json`
- `ThresholdEngine/src/Data/Migrations/YYYYMMDDHHMMSS_Add_PatternWeightingFields.cs`
- All 58 pattern JSON files in `ThresholdEngine/config/patterns/**/*.json`
- `ThresholdEngine/src/Entities/PatternConfiguration.cs`
- `ThresholdEngine/src/Enums/TemporalType.cs` (NEW)
- `ThresholdEngine.Tests/Validation/PublicationFrequencyValidationTests.cs` (NEW)

**Tests**:
- JSON schema validation passes for all patterns
- Migration applies cleanly (up/down)
- Publication frequency validation test passes
- Hot reload detects weight changes
- Pattern loader deserializes new fields including `publicationFrequencyDays`

**Acceptance**: 
- ✓ Series publication frequency reference table created and reviewed
- ✓ All patterns have `weight`, `temporalType`, `publicationFrequencyDays`, `signalDecayDays` defined
- ✓ Automated validation passes: publication frequencies match reference or documented exception
- ✓ Multi-series patterns use MAX(series frequencies)
- ✓ Domain expert sign-off on all publication frequencies
- ✓ `dotnet ef database update` succeeds
- ✓ No schema validation errors in logs

---

### Phase 2: Evaluation Engine (Week 2)

**Tasks**:
1. Implement `CalculateFreshnessFactor(pattern)` with publication-frequency-aware exponential decay
2. Add multi-series handling logic
3. Add health checks for missing/stale data
4. Implement `GetTemporalMultiplier(pattern, isTransitioning)`
5. Refactor `MacroScoreCalculator.CalculateCategoryScore()` to use weights
6. Update `PatternEvaluationContext` to expose `MacroScoreTrend`
7. Add telemetry for weighted contributions (OpenTelemetry metrics)

**Files Changed**:
- `ThresholdEngine/src/Services/MacroScoreCalculator.cs`
- `ThresholdEngine/src/Entities/PatternEvaluationContext.cs`
- `ThresholdEngine/src/Services/PatternEvaluationService.cs`
- `ThresholdEngine/src/Telemetry/Metrics.cs`
- `ThresholdEngine/src/Health/PatternDataHealthCheck.cs` (NEW)

**Algorithm**: Publication-Frequency-Aware Freshness Decay (Multi-Series)

```csharp
decimal CalculateFreshnessFactor(PatternConfiguration pattern)
{
    if (pattern.RequiredSeries == null || !pattern.RequiredSeries.Any())
        return 1.0m;

    // For multi-series patterns: all series must be current for pattern to be current
    // Use the most restrictive (oldest) observation date
    DateTime? oldestUpdate = null;
    
    foreach (var seriesId in pattern.RequiredSeries)
    {
        var lastUpdate = _observationCache.GetLatestDate(seriesId);
        
        if (!lastUpdate.HasValue)
        {
            // Missing data for any series = no signal for entire pattern
            _logger.LogWarning(
                "Pattern {PatternId} missing data for series {SeriesId}", 
                pattern.PatternId, seriesId);
            _metrics.RecordPatternMissingData(pattern.PatternId, seriesId);
            return 0.0m;
        }
        
        if (oldestUpdate == null || lastUpdate.Value < oldestUpdate.Value)
        {
            oldestUpdate = lastUpdate.Value;
        }
    }

    if (!oldestUpdate.HasValue)
        return 0.0m;

    var daysSincePublication = (DateTime.UtcNow - oldestUpdate.Value).TotalDays;
    
    // Calculate days OVERDUE (negative means still current)
    var daysOverdue = Math.Max(0, daysSincePublication - pattern.PublicationFrequencyDays);
    
    // Health check: Alert on severely stale data
    if (daysOverdue > pattern.PublicationFrequencyDays * 3)
    {
        _logger.LogError(
            "Pattern {PatternId} severely overdue: {DaysOverdue} days (pubFreq={PubFreq})", 
            pattern.PatternId, daysOverdue, pattern.PublicationFrequencyDays);
        _metrics.RecordSeverelyOverduePattern(pattern.PatternId, daysOverdue);
    }
    
    if (daysOverdue == 0)
        return 1.0m;  // Still current within publication cycle
    
    // Only apply decay to overdue data
    var decayFactor = Math.Exp(-daysOverdue / pattern.SignalDecayDays);
    
    return (decimal)decayFactor;
}
```

**Algorithm**: Temporal Multiplier

```csharp
decimal GetTemporalMultiplier(PatternConfiguration pattern, bool isRegimeTransitioning)
{
    if (!isRegimeTransitioning)
        return 1.0m;

    return pattern.TemporalType switch
    {
        TemporalType.Leading => 1.3m,      // Boost leading indicators during transitions
        TemporalType.Coincident => 1.0m,   // Neutral
        TemporalType.Lagging => 0.7m,      // Downweight lagging during transitions
        _ => 1.0m
    };
}
```

**Health Check**:

```csharp
// ThresholdEngine/src/Health/PatternDataHealthCheck.cs
public class PatternDataHealthCheck : IHealthCheck
{
    public async Task<HealthCheckResult> CheckHealthAsync(HealthCheckContext context, CancellationToken ct)
    {
        var patterns = await _patternRepo.GetAllEnabledAsync(ct);
        var issues = new List<string>();
        
        foreach (var pattern in patterns)
        {
            var freshness = _calculator.CalculateFreshnessFactor(pattern);
            
            if (freshness == 0.0m)
                issues.Add($"{pattern.PatternId}: No data");
            else if (freshness < 0.1m)
                issues.Add($"{pattern.PatternId}: Severely stale ({freshness:P0})");
        }
        
        if (issues.Any())
        {
            return HealthCheckResult.Degraded(
                $"{issues.Count} patterns with data issues", 
                data: new Dictionary<string, object> { ["issues"] = issues });
        }
        
        return HealthCheckResult.Healthy($"All {patterns.Count} patterns have current data");
    }
}
```

**Tests**:
- Unit test: `FreshnessDecay_CurrentMonthlyData_NoDecay()` - ISM 15 days old = 100% fresh
- Unit test: `FreshnessDecay_OverdueMonthlyData_Decays()` - ISM 50 days old (20 days overdue) = 51% fresh
- Unit test: `FreshnessDecay_RealTimeData_DecaysImmediately()` - VIX 1 day old = 86% fresh
- Unit test: `FreshnessDecay_MultiSeries_UsesOldest()` - Pattern with 2 series, one current, one stale = stale
- Unit test: `FreshnessDecay_MissingSeries_ReturnsZero()` - Pattern missing data for any series = 0
- Unit test: `WeightedScoring_PrioritizesHighQualitySignals()`
- Unit test: `TemporalMultiplier_BoostsLeadingDuringTransitions()`
- Integration test: Compare weighted vs naive macro scores
- Integration test: Health check detects severely stale patterns

**Acceptance**:
- ✓ Weighted category scores sum correctly
- ✓ Freshness decay applied only to overdue observations
- ✓ Current monthly data (15 days old, 30-day cycle) maintains 100% strength
- ✓ Multi-series patterns use oldest observation date
- ✓ Missing data for any series returns 0 freshness
- ✓ Health check alerts on stale data (>3× publication frequency)
- ✓ Temporal multipliers active when `MacroScoreTrend > 0.5`
- ✓ All existing tests still pass (backward compatibility)

---

### Phase 3: API & Observability (Week 3)

**Tasks**:
1. Extend MCP `get_pattern` endpoint to return new fields including `publicationFrequencyDays`
2. Add `GET /api/patterns/contributions` endpoint showing weighted breakdown
3. Add `GET /api/patterns/health` endpoint showing data freshness
4. Create Grafana dashboard: "Pattern Signal Contributions"
5. Create Grafana dashboard: "Pattern Data Health"
6. Update Prometheus metrics to track weighted vs raw signals
7. Add OpenTelemetry spans for freshness/temporal calculations

**Files Changed**:
- `ThresholdEngineMcp/src/Tools/GetPatternTool.cs`
- `ThresholdEngine/src/Endpoints/PatternEndpoints.cs` (new endpoints)
- `ThresholdEngine/grafana-dashboards/pattern-contributions.json` (NEW)
- `ThresholdEngine/grafana-dashboards/pattern-data-health.json` (NEW)
- `ThresholdEngine/src/Telemetry/Metrics.cs`

**New Endpoint**: `GET /api/patterns/contributions`

```json
{
  "macroScore": -4.4,
  "macroScoreTrend": -0.2,
  "isTransitioning": false,
  "categories": [
    {
      "category": "Recession",
      "categoryWeight": 0.30,
      "rawScore": -2.8,
      "weightedScore": -3.1,
      "patterns": [
        {
          "patternId": "sahm-rule",
          "weight": 0.95,
          "publicationFrequencyDays": 30,
          "requiredSeries": ["UNRATE"],
          "daysSincePublication": 15,
          "daysOverdue": 0,
          "rawSignal": -2.0,
          "freshnessFactor": 1.0,
          "temporalMultiplier": 1.0,
          "weightedSignal": -1.90
        },
        {
          "patternId": "freight-recession",
          "weight": 0.55,
          "publicationFrequencyDays": 30,
          "requiredSeries": ["RAILFRTCARLOADSD11"],
          "daysSincePublication": 50,
          "daysOverdue": 20,
          "rawSignal": -2.0,
          "freshnessFactor": 0.51,
          "temporalMultiplier": 0.7,
          "weightedSignal": -0.39
        }
      ]
    }
  ]
}
```

**New Endpoint**: `GET /api/patterns/health`

```json
{
  "timestamp": "2025-12-14T10:30:00Z",
  "totalPatterns": 58,
  "enabledPatterns": 54,
  "healthStatus": "Degraded",
  "issues": [
    {
      "patternId": "freight-recession",
      "severity": "Warning",
      "message": "20 days overdue (pubFreq=30, decay=180)",
      "freshness": 0.51,
      "lastUpdate": "2025-11-24T00:00:00Z"
    },
    {
      "patternId": "custom-indicator-xyz",
      "severity": "Error",
      "message": "No data available",
      "freshness": 0.0,
      "lastUpdate": null
    }
  ],
  "summary": {
    "current": 45,
    "slightlyOverdue": 7,
    "severelyOverdue": 2,
    "noData": 0
  }
}
```

**Grafana Dashboard 1**: Pattern Signal Contributions

- Stat panels: Macro Score, Category Scores (weighted)
- Table: Pattern contributions sorted by weighted signal magnitude
  - Columns: Pattern, Category, Weight, Pub Freq, Days Since, Days Overdue, Raw Signal, Freshness, Temporal, Weighted Signal
- Timeseries: Top 10 pattern signals over time
- Heatmap: Pattern freshness factors (red=stale, green=fresh)

**Grafana Dashboard 2**: Pattern Data Health

- Gauge: Overall health score (% patterns with fresh data)
- Table: Patterns with issues (sorted by severity)
- Histogram: Days overdue distribution
- Timeseries: Data staleness over time (count of patterns >7, >30, >90 days overdue)
- Alert: Trigger when >10% patterns are severely overdue

**Tests**:
- API test: Contributions endpoint returns correct weighted breakdown
- API test: Health endpoint identifies stale patterns
- MCP test: `get_pattern` includes new metadata fields
- Manual: Verify dashboards load without errors
- Manual: Verify health alerts fire correctly

**Acceptance**:
- ✓ MCP returns `weight`, `temporalType`, `publicationFrequencyDays`, etc.
- ✓ Contributions endpoint shows weighted vs raw signals with freshness details
- ✓ Health endpoint identifies data issues
- ✓ Grafana dashboards visualize top contributors and data health
- ✓ Metrics exported to Prometheus
- ✓ Alerts configured for stale data

---

### Phase 4: Documentation & Calibration (Week 4)

**Tasks**:
1. Update `ARCHITECTURE.md` with weighted scoring explanation
2. Update `ThresholdEngine/README.md` with new pattern fields
3. Create rollback procedure documentation
4. Create configuration validation guide
5. Backtest weighted vs naive scoring on historical data (if available)
6. Dry-run evaluation with new configs on recent historical data
7. Adjust weights based on backtest results

**Files Changed**:
- `docs/ARCHITECTURE.md`
- `ThresholdEngine/README.md`
- `ThresholdEngine/docs/ROLLBACK_PROCEDURE.md` (NEW)
- `ThresholdEngine/docs/CONFIG_VALIDATION.md` (NEW)
- `ThresholdEngine/config/patterns/**/README.md` (update tables with weights)

**Rollback Procedure**: `ThresholdEngine/docs/ROLLBACK_PROCEDURE.md`

```markdown
# Pattern Configuration Rollback Procedure

## Detection
Monitor for these signals indicating bad configuration:
- Macro score delta from baseline >2.0 within 24 hours
- >20% of patterns showing 0% freshness
- Health check "Unhealthy" status
- User reports of unexpected regime changes

## Immediate Rollback

### Option 1: Revert Configuration Files (Recommended)
1. Stop ThresholdEngine service
2. Git revert pattern JSON files to previous commit
3. Restart ThresholdEngine service (hot reload picks up old configs)
4. Monitor macro score returns to baseline

### Option 2: Database Update (If Config Files Lost)
```sql
-- Revert all patterns to defaults
UPDATE pattern_configurations 
SET weight = 1.0,
    publication_frequency_days = 30,
    signal_decay_days = 30,
    temporal_type = 'Coincident',
    lead_time_months = 0,
    confidence = 0.75;
```

## Validation After Rollback
- Check health endpoint: All patterns should show current data
- Compare macro score to baseline: Should be within ±0.5
- Review evaluation logs: No errors or warnings
- Monitor for 24 hours before re-attempting deployment

## Root Cause Analysis
- Review configuration changes that triggered rollback
- Check publication frequency validation tests
- Verify data feeds are operational
- Document findings and adjust deployment procedure
```

**Configuration Validation Guide**: `ThresholdEngine/docs/CONFIG_VALIDATION.md`

```markdown
# Pattern Configuration Validation Guide

## Pre-Deployment Validation

### 1. Automated Tests
Run all validation tests:
```bash
dotnet test ThresholdEngine.Tests --filter Category=Validation
```

Expected: 100% pass rate

### 2. Publication Frequency Validation
```bash
dotnet test --filter PublicationFrequencyValidation
```

Validates:
- All patterns have publication frequencies defined
- Multi-series patterns use MAX(series frequencies)
- Frequencies match reference table or have documented exceptions

### 3. Dry-Run Evaluation
```bash
# Run evaluation with new configs on last 90 days of data
dotnet run --project ThresholdEngine.Tools -- dry-run --days 90 --config-path /path/to/new/configs
```

Review output:
- Macro score timeline (should show smooth trends, no sudden jumps)
- Pattern contribution breakdown
- Data freshness distribution

### 4. Manual Review Checklist
- [ ] All 58 patterns reviewed by domain expert
- [ ] No pattern has weight >0.95 without historical justification
- [ ] Publication frequencies match known data release schedules
- [ ] Decay periods >= publication frequencies (with documented exceptions)
- [ ] Multi-series patterns documented which series determines frequency

## Post-Deployment Validation

### Immediate (0-2 hours)
- Monitor macro score: Should be within ±1.0 of baseline
- Check health endpoint: All patterns showing fresh data
- Review logs: No errors or warnings

### Short-term (2-48 hours)
- Compare weighted vs naive scores: Document delta
- Monitor regime transition triggers: Should align with market conditions
- Review pattern contribution changes: High-quality patterns should dominate

### Long-term (1-4 weeks)
- Backtest regime detection accuracy
- Validate VIX deployment triggers align with market volatility
- Gather feedback from users on macro score reliability
```

**Deliverables**:
- Weight assignment guide (criteria for 0.9+, 0.75-0.89, 0.60-0.74, etc.) ✓ Complete
- Temporal classification guide (how to determine Leading vs Lagging) ✓ Complete
- Publication frequency guide ✓ Complete
- Rollback procedure documented ✓ Complete
- Configuration validation guide ✓ Complete
- Decay curve visualization (Excel/Python chart)
- Backtest report comparing regime transitions
- Dry-run evaluation report

**Acceptance**:
- ✓ All documentation updated
- ✓ Rollback procedure tested on staging
- ✓ Dry-run evaluation shows reasonable results
- ✓ Engineers understand how to assign weights to new patterns
- ✓ Weight assignments justified with historical accuracy data (where available)

---

## PATTERN_WEIGHTS [initial_proposal]

### Recession (30% category weight)

| Pattern ID | Weight | Temporal | Pub Freq | Decay | Series Count | Rationale |
|------------|--------|----------|----------|-------|--------------|-----------|
| sahm-rule | 0.95 | Coincident | 30 | 90 | 1 (UNRATE) | 100% historical accuracy since 1970 |
| yield-curve-inversion | 0.90 | Leading | 1 | 180 | 2 (DGS10, DGS2) | Reliable 12-18mo lead, signal persists |
| ism-contraction | 0.85 | Coincident | 30 | 30 | 1 (NAPM) | Timely monthly business survey |
| initial-claims-spike | 0.80 | Coincident | 7 | 30 | 1 (ICSA) | Weekly labor market data |
| industrial-production-contraction | 0.75 | Coincident | 30 | 45 | 1 (INDPRO) | Real output measure |
| jobs-contraction | 0.75 | Coincident | 30 | 30 | 1 (PAYEMS) | Monthly payrolls data |
| continuing-claims-warning | 0.60 | Lagging | 7 | 60 | 1 (CCSA) | Confirms labor weakness |
| freight-recession | 0.55 | Lagging | 30 | 180 | 1+ | Persistent signal, slow to turn |
| consumer-confidence-collapse | 0.50 | Lagging | 30 | 60 | 1 (UMCSENT) | Sentiment-based, noisy |

### Liquidity (20% category weight)

| Pattern ID | Weight | Temporal | Pub Freq | Decay | Series Count | Rationale |
|------------|--------|----------|----------|-------|--------------|-----------|
| credit-spread-widening | 0.90 | Leading | 0 | 30 | 1 (BAMLH0A0HYM2) | Market stress predictor 3-6mo lead |
| vix-deployment-l2 | 0.85 | Coincident | 0 | 7 | 1 (VIXCLS) | Extreme volatility, context-dependent |
| vix-deployment-l1 | 0.75 | Coincident | 0 | 7 | 1 (VIXCLS) | Elevated volatility, context-dependent |
| fed-liquidity-contraction | 0.70 | Leading | 7 | 120 | 1 (WALCL) | QT transmission lag 12-18mo |
| dxy-risk-off | 0.65 | Coincident | 0 | 14 | 1 (DEXUSEU) | Dollar strength flight-to-safety |

### NBFI (10% category weight)

| Pattern ID | Weight | Temporal | Pub Freq | Decay | Series Count | Rationale |
|------------|--------|----------|----------|-------|--------------|-----------|
| nbfi-escalation | 0.90 | Coincident | 30 | 30 | 3+ (HY, Repo, NFCI) | Multi-indicator composite, use slowest pubFreq |
| chicago-conditions-index | 0.85 | Coincident | 30 | 30 | 1 (NFCI) | Broad financial conditions |
| standing-repo-stress | 0.85 | Coincident | 1 | 14 | 1 (RRPONTSYD) | Fed emergency facility usage |
| stlouis-stress-index | 0.80 | Coincident | 30 | 30 | 1 (STLFSI) | Regional financial stress |
| nbfi-hy-spreads | 0.75 | Leading | 0 | 30 | 1 (BAMLH0A0HYM2) | Credit market stress signal |
| reverse-repo-liquidity | 0.70 | Coincident | 1 | 30 | 1 (RRPONTSYD) | Fed RRP drain/injection |

### Growth (20% category weight)

| Pattern ID | Weight | Temporal | Pub Freq | Decay | Series Count | Rationale |
|------------|--------|----------|----------|-------|--------------|-----------|
| gdp-acceleration | 0.85 | Coincident | 90 | 90 | 1 (GDP) | Official output measure |
| industrial-production-expansion | 0.80 | Coincident | 30 | 45 | 1 (INDPRO) | Monthly real output |
| housing-starts | 0.75 | Leading | 30 | 60 | 1 (HOUST) | 6-9mo lead on construction |
| retail-sales-surge | 0.70 | Coincident | 30 | 30 | 1 (RSXFS) | Consumer spending proxy |

---

## TESTING_STRATEGY

### Unit Tests (ThresholdEngine.UnitTests)

```csharp
[Fact]
public void FreshnessDecay_CurrentMonthlyData_NoDecay()
{
    // Given: ISM published 15 days ago (monthly cycle = 30 days)
    var pattern = new Pattern { 
        PublicationFrequencyDays = 30,
        SignalDecayDays = 30 
    };
    var observationDate = DateTime.UtcNow.AddDays(-15);
    
    // When: Calculate freshness
    var freshness = calculator.CalculateFreshnessFactor(pattern, observationDate);
    
    // Then: Full strength, not overdue
    freshness.Should().Be(1.0m);
}

[Fact]
public void FreshnessDecay_MultiSeries_UsesOldestObservation()
{
    // Given: Pattern with 2 series, one current, one overdue
    var pattern = new Pattern {
        RequiredSeries = new[] { "SERIES_A", "SERIES_B" },
        PublicationFrequencyDays = 30,
        SignalDecayDays = 30
    };
    
    // Series A: 15 days old (current)
    _cache.Setup(c => c.GetLatestDate("SERIES_A"))
        .Returns(DateTime.UtcNow.AddDays(-15));
    
    // Series B: 50 days old (20 days overdue)
    _cache.Setup(c => c.GetLatestDate("SERIES_B"))
        .Returns(DateTime.UtcNow.AddDays(-50));
    
    // When: Calculate freshness
    var freshness = calculator.CalculateFreshnessFactor(pattern);
    
    // Then: Uses oldest (50 days) = 20 days overdue = e^(-20/30) ≈ 0.513
    freshness.Should().BeApproximately(0.513m, 0.01m);
}

[Fact]
public void FreshnessDecay_MissingSeriesData_ReturnsZero()
{
    // Given: Pattern with 2 series, one has no data
    var pattern = new Pattern {
        RequiredSeries = new[] { "SERIES_A", "SERIES_B" },
        PublicationFrequencyDays = 30,
        SignalDecayDays = 30
    };
    
    _cache.Setup(c => c.GetLatestDate("SERIES_A"))
        .Returns(DateTime.UtcNow.AddDays(-15));
    
    _cache.Setup(c => c.GetLatestDate("SERIES_B"))
        .Returns((DateTime?)null);  // Missing data
    
    // When: Calculate freshness
    var freshness = calculator.CalculateFreshnessFactor(pattern);
    
    // Then: No data for any series = zero freshness
    freshness.Should().Be(0.0m);
}

[Fact]
public void FreshnessDecay_SeverelyOverdue_LogsError()
{
    // Given: Pattern 100 days overdue (pubFreq=30, so >3x threshold)
    var pattern = new Pattern {
        PatternId = "test-pattern",
        PublicationFrequencyDays = 30,
        SignalDecayDays = 30
    };
    var observationDate = DateTime.UtcNow.AddDays(-130);
    
    // When: Calculate freshness
    var freshness = calculator.CalculateFreshnessFactor(pattern, observationDate);
    
    // Then: Error logged and metric recorded
    _logger.Verify(l => l.LogError(
        It.IsAny<string>(), 
        "test-pattern", 
        100, 
        30), 
        Times.Once);
    
    _metrics.Verify(m => m.RecordSeverelyOverduePattern("test-pattern", 100), Times.Once);
}
```

### Validation Tests (ThresholdEngine.Tests/Validation)

```csharp
[Fact]
public void PublicationFrequency_AllPatterns_MatchReference()
{
    // Given: Load all pattern configs and reference table
    var patterns = LoadAllPatternConfigs();
    var reference = LoadPublicationFrequencyReference();
    
    var errors = new List<string>();
    
    foreach (var pattern in patterns)
    {
        // For each required series, verify publication frequency
        var expectedFreqs = pattern.RequiredSeries
            .Select(s => reference.GetFrequency(s))
            .ToList();
        
        var expectedMax = expectedFreqs.Max();
        
        if (pattern.PublicationFrequencyDays != expectedMax)
        {
            errors.Add($"{pattern.PatternId}: Expected {expectedMax} (max of {string.Join(",", expectedFreqs)}), got {pattern.PublicationFrequencyDays}");
        }
    }
    
    // Then: No errors
    errors.Should().BeEmpty();
}

[Fact]
public void DecayPeriod_AllPatterns_ValidRelativeToPublicationFrequency()
{
    var patterns = LoadAllPatternConfigs();
    var documented exceptions = LoadDocumentedExceptions(); // e.g., yield curve
    
    var errors = new List<string>();
    
    foreach (var pattern in patterns)
    {
        if (documentedExceptions.Contains(pattern.PatternId))
            continue;
        
        if (pattern.SignalDecayDays < pattern.PublicationFrequencyDays)
        {
            errors.Add($"{pattern.PatternId}: Decay ({pattern.SignalDecayDays}) < PubFreq ({pattern.PublicationFrequencyDays}) without documented exception");
        }
    }
    
    errors.Should().BeEmpty();
}
```

---

## METRICS & OBSERVABILITY

### New Prometheus Metrics

```csharp
// ThresholdEngine/src/Telemetry/Metrics.cs

public static readonly Histogram<double> PatternWeightedSignal = Meter.CreateHistogram<double>(
    "threshold_engine.pattern.weighted_signal",
    unit: "score",
    description: "Weighted pattern signal including freshness and temporal adjustments");

public static readonly Histogram<double> PatternFreshnessFactor = Meter.CreateHistogram<double>(
    "threshold_engine.pattern.freshness_factor",
    unit: "ratio",
    description: "Signal freshness decay factor (0.0-1.0)");

public static readonly Histogram<double> PatternDaysOverdue = Meter.CreateHistogram<double>(
    "threshold_engine.pattern.days_overdue",
    unit: "days",
    description: "Days beyond publication frequency (0 = current, >0 = stale)");

public static readonly Counter<long> PatternMissingData = Meter.CreateCounter<long>(
    "threshold_engine.pattern.missing_data_total",
    description: "Count of patterns with missing data for required series");

public static readonly Counter<long> PatternSeverelyOverdue = Meter.CreateCounter<long>(
    "threshold_engine.pattern.severely_overdue_total",
    description: "Count of patterns >3x publication frequency overdue");

public static readonly Counter<long> TemporalMultiplierApplications = Meter.CreateCounter<long>(
    "threshold_engine.temporal_multiplier.applications_total",
    description: "Count of temporal multiplier applications by type");

// Tagged by: pattern_id, category, temporal_type, series_id
```

---

## RISKS & MITIGATIONS

| Risk | Impact | Probability | Mitigation |
|------|--------|-------------|------------|
| **Incorrect publication frequencies** | High | Medium | Reference table validation, domain expert review, automated tests |
| **Incorrect weights** skew macro score | High | Medium | Conservative initial weights (0.60-0.80), backtest, dry-run evaluation |
| **Multi-series logic error** | High | Low | Extensive unit tests, document which series determines frequency |
| **Data feed failure undetected** | High | Medium | Health checks, alerts on severely overdue patterns, monitoring dashboard |
| **Performance regression** in eval loop | Medium | Low | Benchmark before/after, cache freshness calculations |
| **Hot reload breaks** with new fields | Medium | Low | Extensive testing, rollback plan |
| **Database migration fails** on production | High | Low | Test on staging first, manual rollback script ready |
| **Temporal logic bugs** during transitions | Medium | Medium | Extensive unit tests, regime transition test cases |
| **Bad config deployed** | High | Low | Dry-run validation, rollback procedure, monitoring alerts |

---

## ROLLOUT_PLAN

### Dev Environment
1. Apply migration, update patterns with publication frequencies, deploy
2. Run all validation tests (automated + manual checklist)
3. Run dry-run evaluation on last 90 days of data
4. Verify Grafana dashboards
5. Validate monthly indicators maintain 100% freshness mid-cycle
6. Test rollback procedure
7. Iterate on weights based on observations

### Staging
1. Deploy via Ansible with `--tags threshold-engine`
2. Monitor for 48 hours
3. Compare weighted vs naive scores (document delta)
4. Verify publication frequency logic (check ISM at day 15 = 100% fresh)
5. Test health check alerts (manually delay data feed)
6. Adjust weights if needed
7. Practice rollback procedure

### Production
1. **Pre-deployment validation**:
   - All tests pass (automated + manual)
   - Dry-run evaluation shows reasonable results
   - Domain expert sign-off on all configurations
   - Rollback procedure documented and tested

2. **Deployment** (low-traffic window):
   - Deploy via Ansible
   - Monitor macro score (should be within ±1.0 of baseline)
   - Check health endpoint (all patterns fresh)
   - Review evaluation logs (no errors)

3. **Post-deployment monitoring** (0-2 hours):
   - Macro score stability (alert if delta >2.0)
   - Pattern contribution breakdown (verify high-quality signals dominate)
   - Data freshness distribution (no unexpected staleness)

4. **Short-term monitoring** (2-48 hours):
   - Weighted vs naive score comparison
   - Regime transition triggers align with market conditions
   - VIX deployment logic functions correctly

5. **Long-term monitoring** (1-4 weeks):
   - Keep naive scoring in metrics for comparison
   - Monitor days-overdue metrics for data pipeline issues
   - Gather user feedback on macro score reliability
   - Consider weight adjustments based on results

6. **Rollback triggers**:
   - Macro score delta from baseline >2.0 within 24 hours
   - >20% of patterns showing 0% freshness
   - Health check "Unhealthy" status
   - User reports of unexpected regime changes

---

## ACCEPTANCE_CHECKLIST

**Schema & Data**:
- [ ] Series publication frequency reference table created and reviewed
- [ ] Pattern schema extended (6 new fields including `publicationFrequencyDays`)
- [ ] EF migration created with SQL updates for existing patterns
- [ ] All 58 patterns updated with weights/temporal/publication metadata
- [ ] Multi-series patterns use MAX(series frequencies) with documentation
- [ ] Automated validation passes: publication frequencies match reference
- [ ] Domain expert sign-off on all publication frequencies
- [ ] JSON schema validation passes
- [ ] Hot reload tested with weight changes

**Evaluation Engine**:
- [ ] Weighted scoring implemented
- [ ] Publication-frequency-aware freshness decay implemented
- [ ] Multi-series pattern handling (use oldest observation)
- [ ] Health checks for missing/stale data implemented
- [ ] Current monthly data (15 days, 30-day cycle) maintains 100% strength
- [ ] Overdue data decays exponentially
- [ ] Temporal multipliers applied during transitions
- [ ] Backward compatibility maintained (default weight=1.0)
- [ ] No performance regression (<100ms eval time)

**API & Observability**:
- [ ] MCP endpoints return new fields including `publicationFrequencyDays`
- [ ] Contributions API endpoint functional with days-overdue info
- [ ] Health API endpoint identifies stale patterns
- [ ] Grafana "Pattern Contributions" dashboard created and tested
- [ ] Grafana "Pattern Data Health" dashboard created and tested
- [ ] Prometheus metrics exported
- [ ] Health check alerts configured
- [ ] OpenTelemetry spans added

**Testing**:
- [ ] Unit tests: weighted scoring, publication-aware freshness, temporal
- [ ] Unit tests: multi-series handling, missing data handling
- [ ] Validation tests: publication frequency correctness
- [ ] Integration tests: macro score calculation
- [ ] API tests: contributions and health endpoints
- [ ] Manual testing: dashboard, hot reload, rollback procedure

**Documentation**:
- [ ] ARCHITECTURE.md updated
- [ ] ThresholdEngine/README.md updated
- [ ] Pattern weighting guide updated ✓
- [ ] Signal decay reference updated ✓
- [ ] Rollback procedure documented ✓
- [ ] Configuration validation guide created ✓
- [ ] Pattern category READMEs updated with weights

**Deployment**:
- [ ] Dry-run evaluation on historical data successful
- [ ] Ansible playbook tested
- [ ] Rollback procedure tested on staging
- [ ] Staging deployment successful (48-hour monitoring)
- [ ] Production deployment successful
- [ ] Post-deployment validation complete

---

## REFERENCES

### ATLAS Documentation
- [ARCHITECTURE.md](../docs/ARCHITECTURE.md) - System design principles
- [ThresholdEngine/README.md](../ThresholdEngine/README.md) - Service documentation
- [CLAUDE.md](../CLAUDE.md) - Implementation standards

### Task Documentation
- [Pattern Weight Assignment Guide](./pattern-weight-assignment-guide.md) - Weight criteria
- [Signal Decay Reference](./pattern-signal-decay-reference.md) - Mathematical formulas

### Related Patterns
- Configuration-driven evaluation (existing)
- Hot reload (existing)
- Regime-aware signals (VIX context-dependent, existing)

### External References
- Exponential decay: https://en.wikipedia.org/wiki/Exponential_decay
- Weighted averages: https://en.wikipedia.org/wiki/Weighted_arithmetic_mean
- Leading/Lagging indicators: https://www.investopedia.com/terms/l/leadingindicator.asp

---

## NOTES

**Temporal Type vs Publication Frequency** (Critical Distinction):
- **Temporal Type**: When the signal predicts relative to economic events
  - Leading: Signal precedes events by 3+ months (e.g., yield curve inversion predicts recession 12-18mo ahead)
  - Coincident: Signal reflects current state (e.g., ISM measures current business conditions)
  - Lagging: Signal confirms past events (e.g., CPI confirms past inflation)
- **Publication Frequency**: How often the data source publishes updates
  - Real-time (0): VIX, credit spreads, DXY (continuous market data)
  - Daily (1): Treasury yields, repo rates
  - Weekly (7): Jobless claims
  - Monthly (30): ISM, CPI, payrolls
  - Quarterly (90): GDP

**Examples**:
- **Credit Spreads**: Leading (predicts stress 3-6mo ahead) + Real-time (pubFreq=0)
- **Yield Curve**: Leading (predicts recession 12-18mo ahead) + Daily (pubFreq=1) + Very persistent (decay=180)
- **CPI**: Lagging (confirms past inflation) + Monthly (pubFreq=30)
- **Sahm Rule**: Coincident (reflects current unemployment) + Monthly (pubFreq=30)

**Why publication frequency awareness?**
- ISM published 15 days ago is the **current** data (next release in 15 days)
- Without this, we'd penalize patterns for not updating faster than they can
- Real-time data (VIX) decays immediately; monthly data stays fresh 30 days

**Why exponential decay after overdue?**
- Models information half-life naturally (radioactive decay analogy)
- Simple single-parameter control (`signalDecayDays`)
- At `signalDecayDays` overdue: signal strength = 37% (1/e)
- At 2×`signalDecayDays` overdue: signal strength = 14% (1/e²)

**Why multi-series use oldest observation?**
- Pattern only "current" when **all** required data is current
- Example: NBFI Escalation requires HY spreads (real-time), Standing Repo (daily), NFCI (weekly)
  - If NFCI is 10 days old → pattern is 10 days old
  - Even though HY spreads and Repo are current
- Conservative approach: avoids false positives from incomplete data

**Why normalize by sum of weights?**
- Category with 10 patterns (avg weight 0.7) shouldn't dominate category with 3 patterns (avg weight 0.9)
- Prevents "pattern proliferation" from inflating category scores
- Maintains category comparability across different pattern counts

**Why boost leading indicators during transitions?**
- Regime changes are the highest-risk events
- Leading indicators provide early warning
- Once confirmed (coincident signals), reduce leading boost
- Lagging indicators downweighted to avoid false confirmations

**Default values rationale**:
- `weight=1.0`: Backward compatible, neutral starting point
- `temporalType=Coincident`: Most patterns are real-time snapshots
- `publicationFrequencyDays=30`: Monthly data most common
- `signalDecayDays=30`: Monthly data typical, moderate decay
- `confidence=0.75`: Informational only, conservative assumption

**Two-parameter freshness model**:
- `publicationFrequencyDays`: Objective fact about data availability
- `signalDecayDays`: Subjective judgment about signal persistence
- Example: Yield curve has `pubFreq=1` (daily) but `decay=180` (very persistent signal)
- Example: ISM has `pubFreq=30` (monthly) and `decay=30` (monthly relevance)
