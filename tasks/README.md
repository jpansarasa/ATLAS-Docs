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
**Version**: 1.2 (Implementation Spec), 1.1 (Guides)  
**Documents**:
1. [Implementation Spec](./pattern-weighting-temporal-metadata.md) - Complete technical specification (45 pages)
2. [Weight Assignment Guide](./pattern-weight-assignment-guide.md) - Criteria for setting pattern weights
3. [Signal Decay Reference](./pattern-signal-decay-reference.md) - Mathematical formulas and examples

**Summary**:

Enhance ThresholdEngine from naive equal-weighting to sophisticated signal quality scoring with:
- Individual pattern reliability weights (0.0-1.0)
- Temporal classification (Leading, Coincident, Lagging)
- **Publication frequency awareness** (decay only overdue data)
- Signal freshness decay based on data staleness
- Regime-aware temporal weighting adjustments

**Key Innovation - Publication Frequency Awareness**:

Traditional approach would penalize monthly indicators for being "15 days old". Our approach recognizes that ISM Manufacturing published 15 days ago is **current** (next release in 15 days) and maintains 100% signal strength. Only data that is **overdue** (beyond its normal publication cycle) undergoes exponential decay.

**Key Deliverables**:
- 58 patterns updated with weight/temporal/publication frequency metadata
- Pattern schema extended (**6 new fields**: weight, temporalType, leadTimeMonths, **publicationFrequencyDays**, signalDecayDays, confidence)
- Publication frequency reference table with validation framework
- Database migration for new fields (with SQL updates for existing patterns)
- Weighted evaluation engine with multi-series handling
- Exponential freshness decay (publication-frequency-aware)
- Health checks for missing/stale data with alerting
- Temporal multipliers during regime transitions
- Grafana dashboards: "Pattern Contributions" and "Pattern Data Health"
- MCP endpoints exposing contribution breakdowns
- Rollback procedure documentation
- Configuration validation guide
- Zero breaking changes to existing API

**Phases**:
1. **Week 1**: Schema & Data
   - Create publication frequency reference table
   - Update JSON patterns with all 6 fields
   - Create EF migration with SQL data updates
   - Automated validation tests
   - Domain expert review and sign-off

2. **Week 2**: Evaluation Engine
   - Implement publication-frequency-aware decay
   - Multi-series pattern handling (use oldest observation)
   - Health checks for missing/stale data
   - Weighted scoring algorithms
   - Temporal multipliers

3. **Week 3**: API & Observability
   - Contributions API endpoint
   - Health API endpoint
   - Grafana dashboards (contributions + health)
   - Prometheus metrics
   - Alerts for data staleness

4. **Week 4**: Documentation & Calibration
   - Update ARCHITECTURE.md
   - Rollback procedure
   - Configuration validation guide
   - Dry-run evaluation on historical data
   - Weight tuning based on backtest

**Impact**:

| Improvement | Before (Naive) | After (Weighted + Fresh) |
|-------------|----------------|--------------------------|
| Sahm Rule contribution | Equal to Consumer Sentiment | 1.9× more influential (0.95 vs 0.50 weight) |
| Monthly data (15 days old) | 60% strength (incorrect penalty) | **100% strength** (still current) |
| Overdue data handling | No adjustment | Exponential decay (37% at decay period) |
| Regime transitions | No adjustment | Leading indicators boosted 30% |
| Signal quality | Implicit | Explicit (documented, tunable, validated) |
| Data feed failures | Silent degradation | Health alerts, monitoring dashboard |
| Multi-series patterns | Undefined | Conservative (use oldest observation) |

**Critical Refinements Made**:
- **v1.0**: Initial specification (5 fields, naive decay)
- **v1.1**: Added publication frequency awareness (6 fields, decay only overdue data)
- **v1.2**: Added validation framework, health checks, rollback procedures, multi-series handling

**Example - ISM Manufacturing**:
```
Published: December 1
Current Date: December 15
Days Since Publication: 15
Publication Frequency: 30 days (monthly)
Days Overdue: 0 (still current, next release in 15 days)
Freshness: 100% ✅
```

**Example - VIX (Real-Time)**:
```
Last Update: 1 day ago
Publication Frequency: 0 days (real-time)
Days Overdue: 1
Freshness: 86% (e^(-1/7))
```

**Expected Macro Score Impact**:
- Current: -4.4 (Late Cycle, naive equal-weighting)
- Weighted: -5.2 to -5.5 (high-quality current signals dominate)
- Improvement: More accurate regime detection, fewer false signals

---

## Specification Format

All specifications follow the structure defined in [CLAUDE.md](../CLAUDE.md):

```markdown
# Feature Name

**Epic**: Epic Name
**Created**: Date
**Status**: ◯ Ready | → Active | ✓ Done | ⧗ Blocked
**Owner**: Product | Engineering
**Version**: X.Y

## GOAL
- Accept Criteria (checkboxes)
- Constraints

## ARCHITECTURE
- Current State
- Target State
- Design Decisions (table)

## SCHEMA_CHANGES
- JSON schemas
- Database migrations
- Entity updates
- Reference tables (if applicable)

## IMPLEMENTATION_PLAN
- Phase 1: Description
  - Tasks (detailed, actionable)
  - Files Changed
  - Tests
  - Acceptance (specific, measurable)

## TESTING_STRATEGY
- Unit tests (with code examples)
- Integration tests
- Validation tests
- Acceptance tests

## METRICS & OBSERVABILITY
- Prometheus metrics
- Grafana dashboards
- OpenTelemetry spans
- Health checks

## RISKS & MITIGATIONS
- Risk assessment table

## ROLLOUT_PLAN
- Dev → Staging → Production
- Monitoring strategy
- Rollback procedure

## ACCEPTANCE_CHECKLIST
- Final verification checkboxes

## REFERENCES
- Related docs
- External references

## NOTES
- Design rationale
- Critical distinctions
- Edge cases
```

---

## How to Use

### For Product Owners

1. **Create Specification**: Use template above, focus on GOAL and ACCEPTANCE_CHECKLIST
2. **Define Architecture**: Work with engineers on design decisions
3. **Document Critical Distinctions**: Explain non-obvious concepts (e.g., temporal type vs publication frequency)
4. **Review Implementation Plan**: Ensure phasing makes sense, tasks are actionable
5. **Approve Acceptance Criteria**: Clear, measurable, testable definition of done

### For Engineers

1. **Read Specification**: Understand goal, architecture, constraints
2. **Check Prerequisites**: Reference tables, data migrations, validation frameworks
3. **Follow Implementation Plan**: Work through phases sequentially
4. **Write Tests First**: Unit tests define behavior before implementation
5. **Check Acceptance Criteria**: Validate each deliverable
6. **Update Status**: Mark tasks complete in STATUS section

### For Reviewers

1. **Verify Completeness**: All sections filled, no TODOs, no ambiguities
2. **Check Acceptance Criteria**: Measurable, testable, achievable
3. **Validate Architecture**: Aligns with [ARCHITECTURE.md](../docs/ARCHITECTURE.md)
4. **Review Edge Cases**: Multi-series patterns, missing data, stale data handled?
5. **Check Validation**: Automated tests prevent incorrect configurations?
6. **Verify Rollback**: Can deployment be safely reverted?
7. **Review Risks**: Mitigations are concrete, not hand-wavy

---

## Specification Quality Checklist

Before marking specification as "Ready":

**Completeness**:
- [ ] All sections present and filled
- [ ] No TODOs or placeholders
- [ ] Examples provided for complex concepts
- [ ] Edge cases documented

**Clarity**:
- [ ] No ambiguous language ("should", "might", "probably")
- [ ] Technical terms defined or linked
- [ ] Diagrams/tables for complex relationships
- [ ] Code examples for algorithms

**Actionability**:
- [ ] Tasks are specific and measurable
- [ ] File paths and function names provided
- [ ] Acceptance criteria are testable
- [ ] Migration scripts included (if needed)

**Safety**:
- [ ] Rollback procedure documented
- [ ] Validation framework prevents bad configs
- [ ] Health checks detect failures
- [ ] Monitoring alerts configured

**Validation**:
- [ ] Reference tables for data validation
- [ ] Automated tests for correctness
- [ ] Manual review checklist for domain experts
- [ ] Dry-run capability for pre-deployment testing

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
- **Validation frameworks** to prevent bad configs

### Observability First
- OpenTelemetry instrumentation
- Structured logging (Loki)
- Metrics (Prometheus)
- Tracing (Tempo)
- Dashboards (Grafana)
- **Health checks** for data quality

### Backward Compatibility
- No breaking API changes
- Default values for new fields
- Migration path for existing data
- **Rollback procedures** for safe deployment

### Data Quality & Validation
- Reference tables for validation
- Automated tests against known values
- Domain expert review for critical configs
- Health monitoring for data staleness

---

## Common Pitfalls to Avoid

### Vague Acceptance Criteria
❌ **Bad**: "All patterns have publication frequencies set correctly"  
✅ **Good**: "All patterns have publication frequencies validated against reference table, domain expert sign-off complete, automated validation test passes"

### Missing Edge Cases
❌ **Bad**: Algorithm only handles single-series patterns  
✅ **Good**: Algorithm handles single-series, multi-series (use oldest), missing data (return 0), severely stale data (alert)

### No Validation Framework
❌ **Bad**: Assume engineers will set publication frequencies correctly  
✅ **Good**: Reference table + automated validation + manual review checklist

### Hand-Wavy Rollback
❌ **Bad**: "Revert if issues arise"  
✅ **Good**: Documented triggers (macro score delta >2.0), rollback script, monitoring alerts, tested on staging

### Ambiguous Terminology
❌ **Bad**: Using "decay period" to mean both publication frequency and signal decay  
✅ **Good**: Clear distinction: `publicationFrequencyDays` (when data publishes) vs `signalDecayDays` (how long signal matters after overdue)

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

## Notes

- Specifications are living documents - update as implementation reveals issues
- Keep ACCEPTANCE_CHECKLIST current with actual deliverables
- Document design decisions and rationale (helps future maintainers)
- Reference existing patterns and conventions where possible
- When in doubt, favor clarity over brevity
- **Critical distinctions** deserve dedicated sections (e.g., temporal type vs publication frequency)
- **Validation frameworks** prevent configuration errors at deployment time
- **Rollback procedures** are not optional - they're part of the specification

**Remember**: These specs are for *engineers to implement*, not for us to code ourselves. Focus on:
1. Clear requirements and acceptance criteria
2. Design rationale and critical distinctions
3. Validation frameworks and safety mechanisms
4. Actionable tasks with specific deliverables
5. Edge case handling and error scenarios

**Quality Standard**: A specification is ready when an engineer who has never seen the codebase can implement the feature with confidence, knowing:
- Exactly what to build
- How to validate correctness
- What edge cases to handle
- How to rollback if needed
- When they're done
