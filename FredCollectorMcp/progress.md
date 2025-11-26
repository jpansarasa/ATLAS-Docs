# FredCollector MCP - Progress Tracker

## Project Status: 📋 SPECIFICATION COMPLETE

**Created:** 2025-11-26  
**Last Updated:** 2025-11-26

---

## API Verification

| Endpoint | Exists | MCP Tool |
|----------|--------|----------|
| `GET /api/series` | ✅ | `fred_list_series`, `fred_categories` |
| `GET /api/series/{id}` | ❓ Needs addition | `fred_get_series` |
| `GET /api/series/{id}/latest` | ✅ | `fred_get_latest` |
| `GET /api/series/{id}/observations` | ✅ | `fred_get_observations` |
| `GET /api/series/search` | ✅ | `fred_search` |
| `GET /api/alerts/recent` | ✅ | `fred_recent_alerts` |
| `GET /api/macro-score` | ✅ | `fred_macro_score` |
| `GET /health` | ✅ | `fred_health` |
| `GET /swagger/v1/swagger.json` | ✅ (dev only) | `fred_api_schema` |

**Note:** Need to enable swagger.json in production (see task 2025-11-26-enable-openapi-export.md)

---

## Tool Inventory (10 Tools)

### Data Tools (7)
| Tool | Priority | API Status |
|------|----------|------------|
| `fred_list_series` | High | ✅ Ready |
| `fred_get_series` | Medium | ❓ Needs endpoint |
| `fred_get_latest` | High | ✅ Ready |
| `fred_get_observations` | High | ✅ Ready |
| `fred_search` | Medium | ✅ Ready |
| `fred_recent_alerts` | Medium | ✅ Ready |
| `fred_macro_score` | Medium | ✅ Ready |

### Discovery & Diagnostics (3)
| Tool | Priority | API Status |
|------|----------|------------|
| `fred_categories` | Medium | Aggregation (no new endpoint) |
| `fred_health` | Low | ✅ Ready |
| `fred_api_schema` | Low | ✅ Ready (needs prod enable) |

---

## Implementation Phases

### Phase 1: Core Framework
- [ ] Create solution structure
- [ ] Implement MCP stdio transport
- [ ] JSON-RPC message handling
- [ ] FredCollector HTTP client
- [ ] Error handling

**Estimate:** 2-3 hours

### Phase 2: Data Tools
- [ ] `fred_list_series`
- [ ] `fred_get_latest`
- [ ] `fred_get_observations`
- [ ] `fred_search`
- [ ] `fred_recent_alerts`
- [ ] `fred_macro_score`

**Estimate:** 3-4 hours

### Phase 3: Discovery Tools
- [ ] `fred_categories` (aggregation)
- [ ] `fred_get_series` (needs API endpoint)
- [ ] `fred_health`
- [ ] `fred_api_schema`

**Estimate:** 2-3 hours

### Phase 4: Polish
- [ ] MCP Resources
- [ ] Claude Desktop testing
- [ ] Documentation
- [ ] Container deployment

**Estimate:** 1-2 hours

**Total Estimate:** 8-12 hours

---

## Dependencies

### API Changes Needed
1. **Enable swagger.json in production** - Task exists
2. **Add GET /api/series/{id}** - Returns single series with full metadata

---

## Notes

### 2025-11-26 - Initial Specification
- Created README.md with 10 tools
- Created CLAUDE.md with coding guidelines
- Most FredCollector API endpoints verified
- Added self-documentation tools (schema, health, categories)
