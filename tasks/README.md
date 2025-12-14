# ATLAS Tasks & Specifications

**Purpose**: Implementation specifications for ATLAS enhancements  
**Audience**: Product Owners, Engineering Teams  
**Updated**: 2025-12-14  

---

## Overview

This directory contains detailed implementation specifications for ATLAS features and enhancements. Each specification includes:

- **Accept Criteria**: Clear definition of done
- **Architecture**: Design decisions and rationale
- **Implementation Plan**: Phased approach with file changes
- **Testing Strategy**: Unit, integration, and acceptance tests
- **Rollout Plan**: Deployment stages and monitoring

---

## Active Specifications

### Pattern Weighting & Temporal Metadata

**Status**: ◯ Specification Ready  
**Epic**: ThresholdEngine Signal Quality Enhancement  
**Documents**:
1. [Implementation Spec](./pattern-weighting-temporal-metadata.md) - Complete technical specification
2. [Weight Assignment Guide](./pattern-weight-assignment-guide.md) - Criteria for setting pattern weights
3. [Signal Decay Reference](./pattern-signal-decay-reference.md) - Mathematical formulas and examples

**Summary**:

Enhance ThresholdEngine from naive equal-weighting to sophisticated signal quality scoring with:
- Individual pattern reliability weights (0.0-1.0)
- Temporal classification (Leading, Coincident, Lagging)
- Signal freshness decay based on data staleness
- Regime-aware temporal weighting adjustments

**Key Deliverables**:
- 58 patterns updated with weight metadata
- Pattern schema extended (5 new fields)
- Database migration for new fields
- Weighted evaluation engine
- Exponential freshness decay
- Temporal multipliers during regime transitions
- Grafana dashboard showing pattern contributions
- Zero breaking changes to existing API

**Phases**:
1. **Week 1**: Schema & Data - Update JSON patterns, create EF migration
2. **Week 2**: Evaluation Engine - Implement weighted scoring algorithms
3. **Week 3**: API & Observability - Endpoints, metrics, dashboards
4. **Week 4**: Documentation & Calibration - Guides, backtesting, tuning

**Impact**:

| Improvement | Before (Naive) | After (Weighted) |
|-------------|----------------|------------------|
| Sahm Rule contribution | Equal to Consumer Sentiment | 1.9× more influential (0.95 vs 0.50 weight) |
| Stale data handling | No adjustment | Exponential decay (37% at decay period) |
| Regime transitions | No adjustment | Leading indicators boosted 30% |
| Signal quality | Implicit | Explicit (documented, tunable) |

---

## Specification Format

All specifications follow the structure defined in [CLAUDE.md](../CLAUDE.md):

```markdown
# Feature Name

**Epic**: Epic Name
**Created**: Date
**Status**: ◯ Ready | → Active | ✓ Done | ⧗ Blocked
**Owner**: Product | Engineering

## GOAL
- Accept Criteria (checkboxes)
- Constraints

## ARCHITECTURE
- Current State
- Target State
- Design Decisions (table)

## IMPLEMENTATION_PLAN
- Phase 1: Description
  - Tasks
  - Files Changed
  - Tests
  - Acceptance

## SCHEMA_CHANGES
- JSON schemas
- Database migrations
- Entity updates

## TESTING_STRATEGY
- Unit tests
- Integration tests
- Acceptance tests

## METRICS & OBSERVABILITY
- Prometheus metrics
- Grafana dashboards
- OpenTelemetry spans

## RISKS & MITIGATIONS
- Risk assessment table

## ROLLOUT_PLAN
- Dev → Staging → Production
- Monitoring strategy

## ACCEPTANCE_CHECKLIST
- Final verification checkboxes

## REFERENCES
- Related docs
- External references
```

---

## How to Use

### For Product Owners

1. **Create Specification**: Use template above, focus on GOAL and ACCEPTANCE_CHECKLIST
2. **Define Architecture**: Work with engineers on design decisions
3. **Review Implementation Plan**: Ensure phasing makes sense
4. **Approve Acceptance Criteria**: Clear definition of done

### For Engineers

1. **Read Specification**: Understand goal, architecture, constraints
2. **Follow Implementation Plan**: Work through phases sequentially
3. **Check Acceptance Criteria**: Validate each deliverable
4. **Update Status**: Mark tasks complete in STATUS section

### For Reviewers

1. **Verify Completeness**: All sections filled, no TODOs
2. **Check Acceptance Criteria**: Measurable, testable, achievable
3. **Validate Architecture**: Aligns with [ARCHITECTURE.md](../docs/ARCHITECTURE.md)
4. **Review Risks**: Mitigations are concrete

---

## Specification Status

| Status | Symbol | Meaning |
|--------|--------|---------|
| Specification Ready | ◯ | Complete spec, ready for engineering |
| Active Development | → | Implementation in progress |
| Blocked | ⧗ | Waiting on dependency or decision |
| Completed | ✓ | Deployed to production |
| Deferred | △ | Postponed, may revisit |
| Abandoned | ✗ | Not pursuing |

---

## ATLAS Design Principles

All specifications must respect ATLAS core principles:

### Single Responsibility
- Collectors: Data ingestion only
- ThresholdEngine: Pattern evaluation only
- AlertService: Notification delivery only
- MCP Servers: API translation only

### Event-Driven Communication
- gRPC streaming between services
- Shared event contracts
- Pub/sub pattern where appropriate

### Configuration Over Code
- Patterns, thresholds, weights in JSON
- Hot reload without restart
- Admin APIs for runtime changes

### Observability First
- OpenTelemetry instrumentation
- Structured logging (Loki)
- Metrics (Prometheus)
- Tracing (Tempo)
- Dashboards (Grafana)

### Backward Compatibility
- No breaking API changes
- Default values for new fields
- Migration path for existing data

---

## Related Documentation

### ATLAS Core Docs
- [ARCHITECTURE.md](../docs/ARCHITECTURE.md) - System design
- [EXECUTIVE-SUMMARY.md](../docs/EXECUTIVE-SUMMARY.md) - Platform overview
- [GRPC-ARCHITECTURE.md](../docs/GRPC-ARCHITECTURE.md) - Event streaming
- [OBSERVABILITY.md](../docs/OBSERVABILITY.md) - Monitoring stack

### Implementation Standards
- [CLAUDE.md](../CLAUDE.md) - Code generation rules
- [STATE.md Format](../CLAUDE.md#TASK_MANAGEMENT) - Task tracking

### Service Documentation
- [ThresholdEngine/README.md](../ThresholdEngine/README.md)
- [FredCollector/README.md](../FredCollector/README.md)
- [SecMaster/README.md](../SecMaster/README.md)

---

## Contact

**Product Owner**: James Layton  
**Engineering Lead**: TBD  
**Repository**: [ATLAS Monorepo](https://github.com/user/atlas)  

---

## Notes

- Specifications are living documents - update as implementation reveals issues
- Keep ACCEPTANCE_CHECKLIST current with actual deliverables
- Document design decisions and rationale (helps future maintainers)
- Reference existing patterns and conventions where possible
- When in doubt, favor clarity over brevity

**Remember**: These specs are for *engineers to implement*, not for us to code ourselves. Focus on clear requirements, acceptance criteria, and design rationale.
