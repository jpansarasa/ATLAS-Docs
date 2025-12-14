# Pattern Weighting & Temporal Metadata

**Epic**: ThresholdEngine Signal Quality Enhancement  
**Created**: 2025-12-14  
**Status**: ◯ Specification Ready  
**Owner**: Product  
**Implementer**: Engineering  

## GOAL

Enhance ThresholdEngine pattern evaluation from naive equal-weighting to sophisticated signal quality scoring incorporating:
- Individual pattern reliability weights (0.0-1.0)
- Temporal classification (Leading, Coincident, Lagging)
- Signal freshness decay based on data staleness
- Regime-aware temporal weighting adjustments

### Accept Criteria
- [ ] All 58 patterns have weight, temporal metadata defined
- [ ] Pattern schema extended and validated (JSON Schema)
- [ ] Database schema supports new fields (EF migration)
- [ ] Evaluation engine applies weighted scoring
- [ ] Signal freshness decay implemented with exponential curve
- [ ] Temporal multipliers applied during regime transitions
- [ ] MCP endpoints expose new metadata
- [ ] Unit tests validate weighted vs naive scoring differences
- [ ] Grafana dashboards visualize pattern contributions
- [ ] Documentation updated (ARCHITECTURE.md, ThresholdEngine/README.md)
- [ ] Zero breaking changes to existing pattern evaluation API
- [ ] Hot reload continues to function with new fields

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

### Target State (Weighted + Temporal)

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

**Benefit**: High-quality signals (Sahm, Yield Curve) contribute proportionally more than low-quality signals.

### Design Decisions

| Decision | Rationale | Implication |
|----------|-----------|-------------|
| **Exponential decay** for freshness | Models information half-life naturally | Requires `signalDecayDays` per pattern |
| **Temporal multiplier** during transitions | Boost leading indicators when regime changing | Requires macro score trend tracking |
| **Category normalization** by sum of weights | Prevents category dilution with many patterns | Category scores remain comparable |
| **Default weight = 1.0** | Backward compatible, explicit opt-in | Existing patterns work unchanged |
| **Metadata in JSON** | Hot reload without recompile | Pattern config files grow ~40% |

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
      "description": "Signal timing relative to economic events"
    },
    "leadTimeMonths": {
      "type": "integer",
      "minimum": -12,
      "maximum": 24,
      "default": 0,
      "description": "Months ahead (positive) or behind (negative) of event"
    },
    "signalDecayDays": {
      "type": "integer",
      "minimum": 1,
      "maximum": 365,
      "default": 30,
      "description": "Days until signal loses 63% strength (1/e). Fast=7, Moderate=30, Slow=180"
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
    public int SignalDecayDays { get; set; } = 30;
    public decimal Confidence { get; set; } = 0.75m;
}

public enum TemporalType
{
    Leading,
    Coincident,
    Lagging
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

    // Update existing patterns from metadata (if present)
    // Or leave at defaults and update JSON configs
}
```

---

## IMPLEMENTATION_PLAN

### Phase 1: Schema & Data (Week 1)

**Tasks**:
1. Update `pattern-schema.json` with new fields
2. Create EF migration `Add_PatternWeightingFields`
3. Update all 58 pattern JSON files with initial weights/metadata
4. Run migration against dev database
5. Verify hot reload still functions

**Files Changed**:
- `ThresholdEngine/config/pattern-schema.json`
- `ThresholdEngine/src/Data/Migrations/YYYYMMDDHHMMSS_Add_PatternWeightingFields.cs`
- All 58 pattern JSON files in `ThresholdEngine/config/patterns/**/*.json`
- `ThresholdEngine/src/Entities/PatternConfiguration.cs`
- `ThresholdEngine/src/Enums/TemporalType.cs` (new)

**Tests**:
- JSON schema validation passes for all patterns
- Migration applies cleanly (up/down)
- Hot reload detects weight changes
- Pattern loader deserializes new fields

**Acceptance**: 
- ✓ All patterns have `weight`, `temporalType`, `signalDecayDays` defined
- ✓ `dotnet ef database update` succeeds
- ✓ No schema validation errors in logs

---

### Phase 2: Evaluation Engine (Week 2)

**Tasks**:
1. Implement `CalculateFreshnessFactor(pattern)` with exponential decay
2. Implement `GetTemporalMultiplier(pattern, isTransitioning)`
3. Refactor `MacroScoreCalculator.CalculateCategoryScore()` to use weights
4. Update `PatternEvaluationContext` to expose `MacroScoreTrend`
5. Add telemetry for weighted contributions (OpenTelemetry metrics)

**Files Changed**:
- `ThresholdEngine/src/Services/MacroScoreCalculator.cs`
- `ThresholdEngine/src/Entities/PatternEvaluationContext.cs`
- `ThresholdEngine/src/Services/PatternEvaluationService.cs`
- `ThresholdEngine/src/Telemetry/Metrics.cs`

**Algorithm**: Freshness Decay

```csharp
decimal CalculateFreshnessFactor(PatternConfiguration pattern)
{
    var requiredSeries = pattern.RequiredSeries.FirstOrDefault();
    if (string.IsNullOrEmpty(requiredSeries))
        return 1.0m;

    var lastUpdate = _observationCache.GetLatestDate(requiredSeries);
    if (!lastUpdate.HasValue)
        return 0.0m;  // No data = no signal

    var daysSinceUpdate = (DateTime.UtcNow - lastUpdate.Value).TotalDays;
    
    // Exponential decay: e^(-days / decayDays)
    // At decayDays: factor = 1/e ≈ 0.37
    // At 2*decayDays: factor ≈ 0.14
    var decayFactor = Math.Exp(-daysSinceUpdate / pattern.SignalDecayDays);
    
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

**Tests**:
- Unit test: `WeightedScoring_PrioritizesHighQualitySignals()`
- Unit test: `FreshnessDecay_ReducesStaleSignals()`
- Unit test: `TemporalMultiplier_BoostsLeadingDuringTransitions()`
- Integration test: Compare weighted vs naive macro scores
- Verify Sahm Rule (0.95 weight) > Consumer Sentiment (0.50 weight)

**Acceptance**:
- ✓ Weighted category scores sum correctly
- ✓ Freshness decay applied to observations >7 days old
- ✓ Temporal multipliers active when `MacroScoreTrend > 0.5`
- ✓ All existing tests still pass (backward compatibility)

---

### Phase 3: API & Observability (Week 3)

**Tasks**:
1. Extend MCP `get_pattern` endpoint to return new fields
2. Add `GET /api/patterns/contributions` endpoint showing weighted breakdown
3. Create Grafana dashboard: "Pattern Signal Contributions"
4. Update Prometheus metrics to track weighted vs raw signals
5. Add OpenTelemetry spans for freshness/temporal calculations

**Files Changed**:
- `ThresholdEngineMcp/src/Tools/GetPatternTool.cs`
- `ThresholdEngine/src/Endpoints/PatternEndpoints.cs` (new endpoint)
- `ThresholdEngine/grafana-dashboards/pattern-contributions.json` (new)
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
          "rawSignal": -2.0,
          "freshnessFactor": 0.98,
          "temporalMultiplier": 1.0,
          "weightedSignal": -1.86
        },
        {
          "patternId": "freight-recession",
          "weight": 0.55,
          "rawSignal": -2.0,
          "freshnessFactor": 0.61,
          "temporalMultiplier": 0.7,
          "weightedSignal": -0.47
        }
      ]
    }
  ]
}
```

**Grafana Dashboard**: Pattern Contributions

- Stat panels: Macro Score, Category Scores (weighted)
- Table: Pattern contributions sorted by weighted signal magnitude
- Timeseries: Top 10 pattern signals over time
- Heatmap: Pattern freshness factors (red=stale, green=fresh)

**Tests**:
- API test: Contributions endpoint returns correct weighted breakdown
- MCP test: `get_pattern` includes new metadata fields
- Manual: Verify dashboard loads without errors

**Acceptance**:
- ✓ MCP returns `weight`, `temporalType`, etc.
- ✓ Contributions endpoint shows weighted vs raw signals
- ✓ Grafana dashboard visualizes top contributors
- ✓ Metrics exported to Prometheus

---

### Phase 4: Documentation & Calibration (Week 4)

**Tasks**:
1. Update `ARCHITECTURE.md` with weighted scoring explanation
2. Update `ThresholdEngine/README.md` with new pattern fields
3. Create `docs/PATTERN_WEIGHTING_GUIDE.md` for weight assignment
4. Document decay curves and temporal multipliers
5. Backtest weighted vs naive scoring on historical data (if available)
6. Adjust weights based on backtest results

**Files Changed**:
- `docs/ARCHITECTURE.md`
- `ThresholdEngine/README.md`
- `docs/PATTERN_WEIGHTING_GUIDE.md` (new)
- `ThresholdEngine/config/patterns/**/README.md` (update tables with weights)

**Deliverables**:
- Weight assignment guide (criteria for 0.9+, 0.75-0.89, 0.60-0.74, etc.)
- Temporal classification guide (how to determine Leading vs Lagging)
- Decay curve visualization (Excel/Python chart)
- Backtest report comparing regime transitions

**Acceptance**:
- ✓ All documentation updated
- ✓ Engineers understand how to assign weights to new patterns
- ✓ Weight assignments justified with historical accuracy data (where available)

---

## PATTERN_WEIGHTS [initial_proposal]

### Recession (30% category weight)

| Pattern ID | Weight | Temporal | Decay | Rationale |
|------------|--------|----------|-------|-----------|
| sahm-rule | 0.95 | Coincident | 90 | 100% historical accuracy since 1970 |
| yield-curve-inversion | 0.90 | Leading | 180 | Reliable 12-18mo lead indicator |
| ism-contraction | 0.85 | Coincident | 30 | Timely monthly business survey |
| initial-claims-spike | 0.80 | Coincident | 30 | Weekly labor market data |
| industrial-production-contraction | 0.75 | Coincident | 45 | Real output measure |
| jobs-contraction | 0.75 | Coincident | 30 | Monthly payrolls data |
| continuing-claims-warning | 0.60 | Lagging | 60 | Confirms labor weakness |
| freight-recession | 0.55 | Lagging | 180 | Persistent signal, slow to turn |
| consumer-confidence-collapse | 0.50 | Lagging | 60 | Sentiment-based, noisy |

### Liquidity (20% category weight)

| Pattern ID | Weight | Temporal | Decay | Rationale |
|------------|--------|----------|-------|-----------|
| credit-spread-widening | 0.90 | Leading | 30 | Market stress predictor 3-6mo lead |
| vix-deployment-l2 | 0.85 | Coincident | 7 | Extreme volatility, context-dependent |
| vix-deployment-l1 | 0.75 | Coincident | 7 | Elevated volatility, context-dependent |
| fed-liquidity-contraction | 0.70 | Leading | 120 | QT transmission lag 12-18mo |
| dxy-risk-off | 0.65 | Coincident | 14 | Dollar strength flight-to-safety |

### NBFI (10% category weight)

| Pattern ID | Weight | Temporal | Decay | Rationale |
|------------|--------|----------|-------|-----------|
| nbfi-escalation | 0.90 | Coincident | 30 | Multi-indicator composite |
| chicago-conditions-index | 0.85 | Coincident | 30 | Broad financial conditions |
| standing-repo-stress | 0.85 | Coincident | 14 | Fed emergency facility usage |
| stlouis-stress-index | 0.80 | Coincident | 30 | Regional financial stress |
| nbfi-hy-spreads | 0.75 | Leading | 30 | Credit market stress signal |
| reverse-repo-liquidity | 0.70 | Coincident | 30 | Fed RRP drain/injection |

### Growth (20% category weight)

| Pattern ID | Weight | Temporal | Decay | Rationale |
|------------|--------|----------|-------|-----------|
| gdp-acceleration | 0.85 | Coincident | 90 | Official output measure |
| industrial-production-expansion | 0.80 | Coincident | 45 | Monthly real output |
| housing-starts | 0.75 | Leading | 60 | 6-9mo lead on construction |
| retail-sales-surge | 0.70 | Coincident | 30 | Consumer spending proxy |

### Inflation (10% category weight)

| Pattern ID | Weight | Temporal | Decay | Rationale |
|------------|--------|----------|-------|-----------|
| pce-acceleration | 0.90 | Lagging | 90 | Fed's preferred measure |
| core-cpi-sticky | 0.85 | Lagging | 90 | Persistent inflation signal |
| cpi-acceleration | 0.80 | Lagging | 90 | Headline inflation |
| pce-above-target | 0.80 | Lagging | 90 | Fed target deviation |
| breakeven-5y5y-forward | 0.75 | Leading | 60 | Market inflation expectations |
| breakeven-5y-elevated | 0.70 | Leading | 60 | TIPS-derived expectations |
| breakeven-10y-elevated | 0.70 | Leading | 60 | Long-term expectations |

### Valuation (0% category weight - informational)

| Pattern ID | Weight | Temporal | Decay | Rationale |
|------------|--------|----------|-------|-----------|
| buffett-indicator | 0.60 | Lagging | 180 | Market cap proxy, slow-moving |
| equal-weight-indicator | 0.55 | Lagging | 90 | Less MAG7 concentration |

### Currency (10% category weight)

| Pattern ID | Weight | Temporal | Decay | Rationale |
|------------|--------|----------|-------|-----------|
| dxy-risk-off | 0.75 | Coincident | 14 | Dollar strength risk sentiment |

### Commodity (0% category weight - informational)

| Pattern ID | Weight | Temporal | Decay | Rationale |
|------------|--------|----------|-------|-----------|
| copper-gold-ratio | 0.70 | Leading | 90 | Industrial demand 6-12mo lead |

---

## TESTING_STRATEGY

### Unit Tests (ThresholdEngine.UnitTests)

```csharp
[Fact]
public void WeightedScoring_HighQualitySignalsDominante()
{
    // Given: Sahm Rule (weight 0.95) fires -2
    //        Consumer Sentiment (weight 0.50) fires -2
    var patterns = new[] {
        new Pattern { Id = "sahm-rule", Weight = 0.95m, Signal = -2m },
        new Pattern { Id = "consumer-collapse", Weight = 0.50m, Signal = -2m }
    };

    // When: Calculate weighted category score
    var score = calculator.CalculateCategoryScore(patterns);

    // Then: Score closer to -2 than naive average would give
    // Weighted: (-2*0.95 + -2*0.50) / (0.95+0.50) = -2.9/1.45 = -2.0
    // Naive: (-2 + -2) / 2 = -2.0
    // Not a good example - let me fix:
    
    patterns[1].Signal = -1m;  // Consumer sentiment less severe
    // Weighted: (-2*0.95 + -1*0.50) / 1.45 = -2.4/1.45 = -1.66
    // Naive: (-2 + -1) / 2 = -1.5
    score.Should().BeApproximately(-1.66m, 0.01m);
}

[Fact]
public void FreshnessDecay_ReducesStaleSignals()
{
    // Given: Pattern with 30-day decay, observation 60 days old
    var pattern = new Pattern { SignalDecayDays = 30 };
    var observationDate = DateTime.UtcNow.AddDays(-60);

    // When: Calculate freshness
    var freshness = calculator.CalculateFreshnessFactor(pattern, observationDate);

    // Then: e^(-60/30) = e^-2 ≈ 0.135
    freshness.Should().BeApproximately(0.135m, 0.01m);
}

[Fact]
public void TemporalMultiplier_BoostsLeadingDuringTransitions()
{
    // Given: Leading indicator during regime transition
    var pattern = new Pattern { TemporalType = TemporalType.Leading };
    var isTransitioning = true;

    // When: Get temporal multiplier
    var multiplier = calculator.GetTemporalMultiplier(pattern, isTransitioning);

    // Then: Boosted to 1.3x
    multiplier.Should().Be(1.3m);
}

[Fact]
public void TemporalMultiplier_NoBoostWhenStable()
{
    // Given: Leading indicator in stable regime
    var pattern = new Pattern { TemporalType = TemporalType.Leading };
    var isTransitioning = false;

    // When: Get temporal multiplier
    var multiplier = calculator.GetTemporalMultiplier(pattern, isTransitioning);

    // Then: No boost (1.0x)
    multiplier.Should().Be(1.0m);
}
```

### Integration Tests

```csharp
[Fact]
public async Task WeightedMacroScore_DiffersFromNaive()
{
    // Given: Historical observations for all patterns
    await SeedHistoricalData();

    // When: Calculate macro score with weights
    var weightedScore = await calculator.CalculateMacroScoreAsync();
    
    // And: Calculate with all weights = 1.0
    var naiveScore = await calculator.CalculateMacroScoreAsync(useWeights: false);

    // Then: Scores differ (high-quality signals shift result)
    weightedScore.Should().NotBe(naiveScore);
    
    // And: Difference documented for regression testing
    _logger.LogInformation("Weighted: {Weighted}, Naive: {Naive}, Delta: {Delta}",
        weightedScore, naiveScore, weightedScore - naiveScore);
}
```

### Acceptance Tests

- [ ] Pattern JSON files validate against updated schema
- [ ] Database migration applies cleanly (fresh DB + existing DB)
- [ ] Hot reload picks up weight changes within 5 seconds
- [ ] Macro score calculation completes in <100ms
- [ ] Contributions API endpoint returns in <200ms
- [ ] Grafana dashboard loads without errors
- [ ] All 58 patterns have weights between 0.40-0.95
- [ ] No pattern has `signalDecayDays` < 7 or > 365
- [ ] Temporal types distributed: ~20% Leading, ~60% Coincident, ~20% Lagging

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

public static readonly Counter<long> TemporalMultiplierApplications = Meter.CreateCounter<long>(
    "threshold_engine.temporal_multiplier.applications_total",
    description: "Count of temporal multiplier applications by type");

// Tagged by: pattern_id, category, temporal_type
```

### Grafana Dashboard Panels

**Panel 1**: Pattern Signal Contributions (Table)
- Columns: Pattern, Category, Weight, Raw Signal, Freshness, Temporal, Weighted Signal
- Sort: By absolute weighted signal descending
- Purpose: Identify dominant signals

**Panel 2**: Weighted vs Naive Macro Score (Timeseries)
- Lines: Weighted Score, Naive Score
- Purpose: Visualize impact of weighting system

**Panel 3**: Signal Freshness Heatmap
- X: Time, Y: Patterns, Color: Freshness factor (0-1)
- Purpose: Identify stale data sources

**Panel 4**: Category Contribution Breakdown (Pie/Bar)
- Segments: Category weighted contributions to macro score
- Purpose: See which categories drive current regime

---

## RISKS & MITIGATIONS

| Risk | Impact | Probability | Mitigation |
|------|--------|-------------|------------|
| **Incorrect weights** skew macro score | High | Medium | Conservative initial weights (0.60-0.80), backtest before deployment |
| **Performance regression** in eval loop | Medium | Low | Benchmark before/after, cache freshness calculations |
| **Hot reload breaks** with new fields | Medium | Low | Extensive testing, rollback plan |
| **Database migration fails** on production | High | Low | Test on staging first, manual rollback script ready |
| **Temporal logic bugs** during transitions | Medium | Medium | Extensive unit tests, regime transition test cases |

---

## ROLLOUT_PLAN

### Dev Environment
1. Apply migration, update patterns, deploy
2. Run integration tests
3. Verify Grafana dashboard
4. Iterate on weights based on observations

### Staging
1. Deploy via Ansible with `--tags threshold-engine`
2. Monitor for 48 hours
3. Compare weighted vs naive scores
4. Adjust weights if needed

### Production
1. Deploy during low-traffic window
2. Monitor macro score stability
3. Alert if macro score delta from baseline > 2.0
4. Keep naive scoring in metrics for 30 days for comparison

---

## ACCEPTANCE_CHECKLIST

**Schema & Data**:
- [ ] Pattern schema extended (5 new fields)
- [ ] EF migration created and tested
- [ ] All 58 patterns updated with weights/metadata
- [ ] JSON schema validation passes
- [ ] Hot reload tested with weight changes

**Evaluation Engine**:
- [ ] Weighted scoring implemented
- [ ] Freshness decay calculation correct (exponential)
- [ ] Temporal multipliers applied during transitions
- [ ] Backward compatibility maintained (default weight=1.0)
- [ ] No performance regression (<100ms eval time)

**API & Observability**:
- [ ] MCP endpoints return new fields
- [ ] Contributions API endpoint functional
- [ ] Grafana dashboard created and tested
- [ ] Prometheus metrics exported
- [ ] OpenTelemetry spans added

**Testing**:
- [ ] Unit tests: weighted scoring, freshness, temporal
- [ ] Integration tests: macro score calculation
- [ ] API tests: contributions endpoint
- [ ] Manual testing: dashboard, hot reload

**Documentation**:
- [ ] ARCHITECTURE.md updated
- [ ] ThresholdEngine/README.md updated
- [ ] Pattern weighting guide created
- [ ] Pattern category READMEs updated with weights

**Deployment**:
- [ ] Ansible playbook tested
- [ ] Rollback procedure documented
- [ ] Staging deployment successful
- [ ] Production deployment successful

---

## REFERENCES

### ATLAS Documentation
- [ARCHITECTURE.md](../docs/ARCHITECTURE.md) - System design principles
- [ThresholdEngine/README.md](../ThresholdEngine/README.md) - Service documentation
- [CLAUDE.md](../CLAUDE.md) - Implementation standards

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

**Why exponential decay?**
- Models information half-life naturally (radioactive decay analogy)
- Simple single-parameter control (`signalDecayDays`)
- At `signalDecayDays`: signal strength = 37% (1/e)
- At 2×`signalDecayDays`: signal strength = 14% (1/e²)

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
- `signalDecayDays=30`: Monthly data typical, moderate decay
- `confidence=0.75`: Informational only, conservative assumption
