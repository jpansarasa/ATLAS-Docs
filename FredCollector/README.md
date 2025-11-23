# ATLAS FRED Data Collection System - Memory Bank

**CursorRIPER.sigma-lite Memory Bank for AI-Assisted Development**

This directory contains the complete specification for the ATLAS FRED Data Collection system using the CursorRIPER.sigma-lite pattern. These files enable Claude Code, Cursor, or other AI coding assistants to maintain full project context across development sessions.

**⚠️ IMPORTANT**: FredCollector is responsible ONLY for data collection from FRED API. Threshold evaluation and alerting are handled by separate microservices (ThresholdEngine and AlertService). See `../ARCHITECTURE.md` for the complete architectural decision record explaining this separation of concerns.

## 🏛️ Memory Bank Structure

This project uses **5 interconnected specification files** (σ₁ through σ₅):

### Core Files

| File | Symbol | Purpose | When to Read |
|------|--------|---------|--------------|
| [projectbrief.md](projectbrief.md) | σ₁ 📋 | Business context, requirements, success criteria | First, always |
| [systemPatterns.md](systemPatterns.md) | σ₂ 🏛️ | Architecture, components, technical design | After projectbrief |
| [techContext.md](techContext.md) | σ₃ 💻 | Technology stack, dependencies, deployment | With systemPatterns |
| [activeContext.md](activeContext.md) | σ₄ 🔮 | Current state, next actions, decisions | Every session start |
| [progress.md](progress.md) | σ₅ 📊 | Epic tracking, user stories, completion | After each component |

### Reading Order for AI Assistants

**First Session** (Complete context):

1. projectbrief.md → Understand the "why"
2. systemPatterns.md → Understand the "how"  
3. techContext.md → Understand the "with what"
4. activeContext.md → Understand "where we are"
5. progress.md → Understand "what's done"

**Subsequent Sessions** (Resume context):

1. activeContext.md → Pick up where you left off
2. progress.md → See what's been completed
3. Refer to other files as needed

## 🎯 Quick Start for AI Assistants

### If you're Claude Code or Cursor starting fresh

```bash
# 1. Read the memory bank (in order)
cat memory-bank/projectbrief.md
cat memory-bank/systemPatterns.md
cat memory-bank/techContext.md
cat memory-bank/activeContext.md
cat memory-bank/progress.md

# 2. Check current mode
# Mode is in activeContext.md header: Ω₃ Plan (currently planning)
# When implementation starts, update to: Ω₄ Execute

# 3. Begin with Sprint 0 tasks from progress.md
# First task: Create solution structure
dotnet new sln -n AtlasFredCollector
# ... follow US1.2 from progress.md
```

### If resuming after a break

```bash
# 1. Read active context only
cat memory-bank/activeContext.md

# 2. Check progress
cat memory-bank/progress.md

# 3. Continue from "Next Steps" section
```

## 📝 Workflow Modes

The project uses 5 explicit modes (Ω notation):

- **Ω₁ Research** (`/research` or `/r`) - Gather information, no code changes
- **Ω₂ Innovate** (`/innovate` or `/n`) - Brainstorm approaches
- **Ω₃ Plan** (`/plan` or `/p`) - Create specifications
- **Ω₄ Execute** (`/execute` or `/e`) - Implementation ← **CURRENT MODE**
- **Ω₅ Review** (`/review` or `/rev`) - Validation

**Current Mode**: Ω₅ Review (Production maintenance)

## 🔄 Update Protocol

After completing work, AI assistants should update these files:

### After Completing a User Story

1. **progress.md**: Mark story complete, update completion %
2. **activeContext.md**: Update "Recent Changes" with timestamp
3. **activeContext.md**: Update "Next Steps" section

### After Making an Architectural Decision

1. **activeContext.md**: Document decision in "Active Decisions"
2. **systemPatterns.md**: Update architecture if design changed

### After Encountering Issues

1. **activeContext.md**: Add to "Blockers & Risks" section
2. **progress.md**: Update risk log if significant

### Timestamp Format

Always use ISO 8601: `[2025-11-08 16:45:00]`

## 🎨 Symbolic Notation Guide

### Status Indicators

- 🟢 Complete
- 🟡 In Progress  
- 🔴 Not Started
- 🔵 Blocked
- 🟣 Deferred

### File Symbols

- σ₁ 📋 projectbrief.md
- σ₂ 🏛️ systemPatterns.md
- σ₃ 💻 techContext.md
- σ₄ 🔮 activeContext.md
- σ₅ 📊 progress.md

### Mode Symbols

- Ω₁-₅ (Omega) = Workflow modes
- Π₁-₄ (Pi) = Project phases
- Σ (Sigma) = Memory bank files

## 🏗️ Project Context

### What We're Building

ATLAS FRED Data Collection System - Automated economic indicator collection feeding a defensive wealth management framework for a $2M portfolio.

### Why It Matters

- Replace manual data collection (2 hours/week → 5 minutes/week)
- Enable automated VIX deployment system ($400K+ ready capital)
- Eliminate $500/month subscription costs
- Build government-independent data infrastructure

### Current State

- **Phase**: Π₄ Production (All epics complete, deployed)
- **Completion**: 100% (Epics E1-E12 complete)
- **Sprint**: Production maintenance
- **Last Updated**: 2025-11-23
- **Target MVP**: ✅ Complete

### MVP Scope (Must Have)

- 7 recession indicators automated (Initial Claims, ISM, Consumer Sentiment, SOFR, 10Y, Unemployment, Fed Funds)
- Threshold alerting (email + ntfy.sh notifications)
- 24-month historical backfill
- REST API for querying data
- Container deployment (Linux containers via containerd/nerdctl)

## 🔧 Technical Stack Summary

**Language**: C# 13 / .NET 9
**Database**: TimescaleDB (PostgreSQL + time-series)
**Key Libraries**: EF Core, Polly, Quartz.NET, Serilog, MailKit
**Deployment**: Linux containers via containerd/nerdctl on Threadripper (24c/48t, 128GB RAM)
**External API**: FRED API (120 requests/minute limit)

## 📡 Series Management API

FredCollector exposes admin endpoints for dynamic series management without code changes:

**Endpoints (port 5001, requires X-API-Key header):**
- `POST /api/admin/series` - Add new FRED series (auto-fetches metadata from FRED API)
- `PUT /api/admin/series/{seriesId}/toggle` - Enable/disable series collection
- `DELETE /api/admin/series/{seriesId}` - Remove a series
- `GET /api/admin/series` - List all configured series

## 🔍 Series Discovery API

Search and discover FRED economic data series with filtering and sorting:

**Endpoint:** `GET /api/series/search` (port 5001, requires X-API-Key header)

**Query Parameters:**
| Parameter | Type | Description |
|-----------|------|-------------|
| `query` | string | Search text (required, 1-200 chars) |
| `limit` | int | Max results (optional, 1-100, default: 20) |
| `frequency` | string | Filter: daily, weekly, monthly, quarterly, annual |
| `minPopularity` | int | Minimum popularity score (0-100) |
| `activeOnly` | bool | Only series with data in last 90 days |
| `orderBy` | enum | Sort: Popularity, Title, LastUpdated, Relevance |

**Example:**
```bash
curl -H "X-API-Key: $API_KEY" \
  "http://localhost:5001/api/series/search?query=unemployment&limit=5&orderBy=Popularity"
```

**Response:**
```json
{
  "results": [
    {
      "seriesId": "UNRATE",
      "title": "Unemployment Rate",
      "frequency": "Monthly",
      "popularity": 95,
      "isAlreadyCollected": true,
      "dataAge": "5 days",
      "isDiscontinued": false
    }
  ],
  "totalCount": 150,
  "diagnostics": { "durationMs": 234, "fromCache": false }
}
```

**Features:**
- Searches FRED's 800,000+ economic series
- `isAlreadyCollected` flag shows if series is configured in ATLAS
- `dataAge` human-readable time since last observation
- Full OpenTelemetry tracing for debugging

**Add Series Request:**
```json
{
  "seriesId": "UNRATE",
  "category": "Employment",
  "alertThreshold": 5.0,
  "thresholdDirection": "Above",
  "backfill": true
}
```

**Features:**
- Auto-fetches title, description, frequency from FRED API
- Auto-generates cron schedule based on data frequency
- Optional backfill of 24 months historical data (default: enabled)
- Validates series exists before adding

## 📚 Key References

### FRED API Documentation

- Base URL: <https://api.stlouisfed.org/fred/>
- Docs: <https://fred.stlouisfed.org/docs/api/fred/>
- Rate Limit: 120 requests/minute

### Architecture Patterns

- Repository pattern for data access
- Circuit breaker + retry (Polly)
- Token bucket rate limiting
- Event-driven alerting (System.Threading.Channels)

### ATLAS Framework Context

The target system feeds into a sophisticated macro-economic framework tracking:

- 7 confirmed recession indicators
- VIX deployment system (>22, >30 thresholds)
- Shadow banking stress monitoring
- Macro score calculation (-30 to +30 scale)

Full ATLAS framework documentation: `../sigma.md`

## 🚀 Getting Started (Human Developers)

If you're a human developer (not an AI assistant):

1. **Read the memory bank files in order** to understand the complete context
2. **Follow Sprint 0 tasks** from progress.md to setup your environment
3. **Use the mode commands** to signal your current activity:
   - Start coding: Document mode change to Ω₄ Execute in activeContext.md
   - Need to research something: Switch to Ω₁ Research
   - Reviewing code: Switch to Ω₅ Review
4. **Update memory bank files** as you complete work (see Update Protocol above)

## 📞 Support

**Project Owner**: James (Portfolio Manager, Head Architect)  
**Created**: 2025-11-08  
**Pattern**: CursorRIPER.sigma-lite v1.0

## 🔐 Important Notes

- **API keys**: Never commit to source control, use user secrets or Windows Credential Manager
- **Memory bank location**: Keep in `/docs/memory-bank/` within solution root
- **Update discipline**: AI assistants MUST update files after completing work
- **Context persistence**: These files enable seamless handoff between sessions

---

**Production Status**: ✅ Complete
**Blocking Issues**: None
**Current Mode**: Ω₅ Review (Production maintenance)
**Epics Complete**: E1-E12 (100%)
**Tests Passing**: 378/378

For AI assistants: Start with `cat activeContext.md` to see current status and next steps.
