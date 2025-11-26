# ThresholdEngine MCP - Progress Tracker

## Project Status: 📋 SPECIFICATION COMPLETE

**Created:** 2025-11-26  
**Last Updated:** 2025-11-26

---

## API Verification

| Endpoint | Exists | MCP Tool |
|----------|--------|----------|
| `GET /api/patterns` | ✅ | `threshold_list_patterns`, `threshold_categories` |
| `GET /api/patterns/{id}` | ✅ | `threshold_get_pattern` |
| `POST /api/patterns/evaluate` | ✅ | **`threshold_evaluate`** ⭐ |
| `POST /api/patterns/{id}/evaluate` | ✅ | `threshold_evaluate_pattern` |
| `POST /api/patterns/reload` | ✅ | `threshold_reload` |
| `PUT /api/patterns/{id}/toggle` | ✅ | (not exposed - admin) |
| `GET /health` | ✅ | `threshold_health` |
| `GET /swagger/v1/swagger.json` | ✅ (dev only) | `threshold_api_schema` |
| Regime history | ❓ Needs endpoint | `threshold_regime_history` |

**Note:** Need to enable swagger.json in production (see task 2025-11-26-enable-openapi-export.md)

---

## Tool Inventory (9 Tools)

### Evaluation Tools (2)
| Tool | Priority | API Status |
|------|----------|------------|
| **`threshold_evaluate`** | ⭐ Primary | ✅ Ready |
| `threshold_evaluate_pattern` | High | ✅ Ready |

### Pattern Discovery (3)
| Tool | Priority | API Status |
|------|----------|------------|
| `threshold_list_patterns` | Medium | ✅ Ready |
| `threshold_get_pattern` | Medium | ✅ Ready (needs enrichment) |
| `threshold_categories` | Medium | Aggregation (no new endpoint) |

### Regime & History (1)
| Tool | Priority | API Status |
|------|----------|------------|
| `threshold_regime_history` | Medium | ❓ Needs endpoint or audit log |

### Administrative (1)
| Tool | Priority | API Status |
|------|----------|------------|
| `threshold_reload` | Low | ✅ Ready |

### Discovery & Diagnostics (2)
| Tool | Priority | API Status |
|------|----------|------------|
| `threshold_health` | Low | ✅ Ready |
| `threshold_api_schema` | Low | ✅ Ready (needs prod enable) |

---

## Implementation Phases

### Phase 1: Core Framework
- [ ] Create solution structure
- [ ] Implement MCP stdio transport
- [ ] JSON-RPC message handling
- [ ] ThresholdEngine HTTP client
- [ ] Error handling

**Estimate:** 2-3 hours

### Phase 2: Primary Tool
- [ ] `threshold_evaluate` - wraps evaluate endpoint
- [ ] Response formatting (triggered patterns, category summaries)
- [ ] Signal/regime interpretation helpers

**Estimate:** 2-3 hours

### Phase 3: Pattern Tools
- [ ] `threshold_list_patterns`
- [ ] `threshold_get_pattern` (with threshold interpretation)
- [ ] `threshold_evaluate_pattern`
- [ ] `threshold_categories` (aggregation)

**Estimate:** 3-4 hours

### Phase 4: Supporting Tools
- [ ] `threshold_reload`
- [ ] `threshold_health`
- [ ] `threshold_api_schema`
- [ ] `threshold_regime_history` (if endpoint available)

**Estimate:** 2-3 hours

### Phase 5: Polish
- [ ] MCP Resources
- [ ] Claude Desktop testing
- [ ] Documentation
- [ ] Container deployment

**Estimate:** 1-2 hours

**Total Estimate:** 10-15 hours

---

## Dependencies

### API Changes Needed
1. **Enable swagger.json in production** - Task exists
2. **Add regime history endpoint** - For `threshold_regime_history` tool
   - Option A: New endpoint on ThresholdEngine
   - Option B: Query ConfigurationAuditLog for regime transitions
   - Option C: Compute from macro score history in FredCollector

---

## Key Insight

The `POST /api/patterns/evaluate` endpoint returns everything needed in one call:
- Current regime
- Macro score
- All 37 pattern evaluations with signals

This is the **primary tool** for conversations. One call gives Claude complete system awareness.

---

## Notes

### 2025-11-26 - Initial Specification
- Created README.md with 9 tools
- Created CLAUDE.md with coding guidelines
- All core ThresholdEngine API endpoints verified
- Added self-documentation tools (schema, health, categories)
- Added regime history tool (needs API endpoint)
- Primary tool `threshold_evaluate` gives complete state in one call
