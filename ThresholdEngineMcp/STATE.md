# STATE.md [ThresholdEngineMcp]

## GOAL
MCP server for Claude Code integration with ThresholdEngine pattern evaluation.

### Current Status
✓ Core MCP tools implemented
✓ Pattern weighting fields exposed (2025-12-15)

---

## ARCH

### Tools Available
| Tool | Description | Updates |
|------|-------------|---------|
| `evaluate` | Evaluate ALL enabled patterns | +weight, temporalType, daysSinceLatestData |
| `list_patterns` | List pattern configurations | +weight, temporalType, publicationFrequencyDays |
| `get_pattern` | Get specific pattern details | +all 6 weighting fields |
| `evaluate_pattern` | Evaluate single pattern | +weight, temporalType, freshness fields |
| `categories` | List categories with counts | unchanged |
| `health` | Service health status | unchanged |
| `api_schema` | OpenAPI specification | unchanged |
| `reload` | Hot-reload patterns | unchanged |

---

## STATUS

### Pattern Weighting Integration ✓ [2025-12-15]
- ✓ `ClientModels.cs` Pattern record updated (6 new fields)
- ✓ `ClientModels.cs` PatternEvaluation record updated (5 new fields)
- ✓ `ThresholdEngineTools.cs` responses include weighting metadata

---

## CONTEXT

### Files Changed
```
ThresholdEngineMcp/
└── src/
    ├── Client/Models/ClientModels.cs     [MODIFIED]
    └── Tools/ThresholdEngineTools.cs     [MODIFIED]
```

### Dependencies
- ThresholdEngine API (upstream)

### New Response Fields

**Pattern record**:
```json
{
  "weight": 0.75,
  "temporalType": "Coincident",
  "leadTimeMonths": 0,
  "publicationFrequencyDays": 30,
  "signalDecayDays": 30,
  "confidence": 0.75
}
```

**PatternEvaluation record**:
```json
{
  "weight": 0.75,
  "temporalType": "Coincident",
  "publicationFrequencyDays": 30,
  "signalDecayDays": 30,
  "daysSinceLatestData": 15
}
```

---

**UPDATED**: 2025-12-15 | **STATUS**: ✓ complete
