# STATE.md [ThresholdEngine]

## GOAL
Pattern weighting & temporal metadata for sophisticated signal quality scoring.

### Accept Criteria
- ✓ All 57 patterns have weight, temporal, publication frequency metadata
- ✓ Pattern schema extended (6 new fields)
- ✓ Evaluation engine applies weighted scoring
- ✓ Signal freshness decay with publication-frequency awareness
- ✓ Temporal multipliers during regime transitions
- ✓ Health checks for missing/stale data
- ✓ MCP endpoints expose new metadata
- ✓ API endpoints for contributions and health
- ✓ Grafana dashboards for observability
- ✓ Documentation updated
- ◯ Unit tests for freshness decay formulas (optional)
- ◯ Rollback procedure documentation (optional)

### Constraints
- Backward compatible (default weight=1.0)
- Configuration over code (all weights in JSON)
- No performance regression
- Preserve existing API contract

---

## ARCH

### Design Decisions
| Decision | Rationale | Implication |
|----------|-----------|-------------|
| Publication frequency awareness | ISM 15 days old is current, not stale | Requires `publicationFrequencyDays` per pattern |
| Multi-series: use oldest date | Pattern only current when all data current | Conservative approach |
| Exponential decay after overdue | Models information half-life naturally | Requires `signalDecayDays` per pattern |
| Temporal multiplier during transitions | Boost leading indicators when regime changing | ±30% adjustment |
| Category normalization by sum of weights | Prevents category dilution | Category scores remain comparable |

### Weighted Scoring Formula
```
weightedSignal = signal × weight × freshnessFactor × temporalMultiplier × confidence
categoryScore = Σ(weightedSignals) / Σ(weights)
macroScore = Σ(categoryScore × categoryWeight) / Σ(categoryWeights)
```

### Freshness Decay
```
freshness = daysSince <= pubFreq ? 1.0 : exp(-(daysSince - pubFreq) / decayDays)
floor = 0.10 (never completely ignore)
```

### Temporal Multipliers
- Leading: +30% during transitions
- Coincident: neutral
- Lagging: -30% during transitions

---

## STATUS

### Phase 1: Schema & Data ✓
- ✓ `TemporalType.cs` enum created
- ✓ `PatternConfiguration.cs` updated (6 fields)
- ✓ `pattern-schema.json` validation
- ✓ `series-publication-frequencies.json` reference
- ✓ All 57 pattern JSON files updated

### Phase 2: Evaluation Engine ✓
- ✓ `PatternEvaluationResult.cs` carry-through fields
- ✓ `PatternEvaluationService.cs` populate new fields
- ✓ `MacroScoreCalculator.cs` weighted scoring
- ✓ Freshness decay implementation
- ✓ Temporal multipliers implementation
- ✓ `PatternDataHealthCheck.cs` for stale data
- ✓ Telemetry metrics in `ThresholdEngineMeter.cs`

### Phase 3: API & Observability ✓
- ✓ `GET /api/patterns/contributions` endpoint
- ✓ `GET /api/patterns/health` endpoint
- ✓ MCP client models updated
- ✓ MCP tools expose new fields
- ✓ Grafana `pattern-contributions.json`
- ✓ Grafana `pattern-data-health.json`
- ✓ Health check registered in Program.cs

### Phase 4: Documentation ✓
- ✓ `README.md` updated with weighted scoring section
- ◯ `ROLLBACK_PROCEDURE.md` (optional)
- ◯ `CONFIG_VALIDATION.md` (optional)

---

## CONTEXT

### Files Changed
```
ThresholdEngine/
├── src/
│   ├── Enums/TemporalType.cs                    [NEW]
│   ├── Entities/PatternConfiguration.cs         [MODIFIED]
│   ├── Entities/PatternEvaluationResult.cs      [MODIFIED]
│   ├── Services/PatternEvaluationService.cs     [MODIFIED]
│   ├── Services/MacroScoreCalculator.cs         [MODIFIED]
│   ├── Endpoints/PatternEndpoints.cs            [MODIFIED]
│   ├── HealthChecks/PatternDataHealthCheck.cs   [NEW]
│   ├── Telemetry/ThresholdEngineMeter.cs        [MODIFIED]
│   └── Program.cs                               [MODIFIED]
├── config/
│   ├── pattern-schema.json                      [MODIFIED]
│   ├── series-publication-frequencies.json      [NEW]
│   └── patterns/**/*.json                       [ALL 57 MODIFIED]
├── grafana-dashboards/
│   ├── pattern-contributions.json               [NEW]
│   └── pattern-data-health.json                 [NEW]
└── README.md                                    [MODIFIED]
```

### Dependencies
- ThresholdEngineMcp (client models updated)

### External Systems
- Prometheus (new metrics)
- Grafana (new dashboards)

---

## METRICS [new]

| Metric | Type | Description |
|--------|------|-------------|
| `thresholdengine.pattern.weighted_signal` | Histogram | Weighted signal values |
| `thresholdengine.pattern.freshness_factor` | Histogram | Freshness decay factors |
| `thresholdengine.pattern.days_overdue` | Histogram | Days past publication frequency |
| `thresholdengine.pattern.missing_data_total` | Counter | Missing data events |
| `thresholdengine.pattern.severely_overdue_total` | Counter | Severely overdue alerts |
| `thresholdengine.temporal_multiplier.applications_total` | Counter | Temporal adjustments applied |

---

**UPDATED**: 2025-12-15 | **STATUS**: ✓ complete
