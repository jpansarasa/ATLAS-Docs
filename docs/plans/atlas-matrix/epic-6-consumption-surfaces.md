# Epic 6 — Three Consumption Surfaces

## Goal

All three surfaces (MCP, dashboard, reports) consume `macro_observations`
and matrix data per handoff §4. None is downstream of another.
Disagreement is preserved across all three.

## Branch

`epic/6-consumption-surfaces` off `main` (forked when Phase 3 merges).

## Phase + dependencies

- Phase: **4**
- Blocked by: Epic 5 (matrix data + per-sector regimes + derived view)
  + Epic 4 (qualitative observations with trust + threshold-cross
  triggers wired) + Epic 3 (substrate is queryable).

## Canonical AC

`/home/james/ATLAS/docs/atlas-matrix-mvp-plan.md` Epic 6 (lines 626–724).

## Features (with execution notes)

### 6.1 — MCP query surface
- **6.1.1** `macro_observations` query tool. Query by signal, sector,
  time range, observation type. Below-threshold qualitative observations
  returned with trust attribute visible (D3: gate is TE-side only).
  Versioned-mapping query path honored.
- **6.1.2** Matrix query tools. Cell time series, sector vector, signal
  vector, cross-correlation (single-pair + all-pairs scan).
- **6.1.3** Regime query tools. Per-sector regime time series (D9).
  Sector × phase reference matrix query (Story 5.5.3).

### 6.2 — Dashboard
- **6.2.1** Sector × phase matrix panel (primary view per D8).
  Renders Story 5.5.3 derived view as two-axis matrix. Rows: 11
  sectors. Columns: regime phases per D9/D17. Cell vectors are
  real-valued — visualization choice (sparkline, mini-histogram,
  signed-magnitude bars) is developer call. **Must not flatten to a
  single aggregate.** Drilldown reveals contributing observations.
  Reference image (`1778343398312_image.png`) is layout inspiration
  only.
- **6.2.2** Sector score and regime trajectory panels. Per-sector:
  score time series with regime overlay. Aggregate: 11-sector regime
  panel showing current state and recent transitions.
- **6.2.3** Direct-substrate panel. At least one panel reads
  `macro_observations` directly (not via TE cells) per §4.
  Below-threshold observations visible with trust indicator.
- **6.2.4** Disagreement panel. Signal-vs-signal disagreement and
  official-vs-unofficial divergence. **Load-bearing matrix value, not
  side-panel** (handoff §1 corollary).

### 6.3 — Reports (D10 — common content structure)
Daily, weekly, monthly cadences share the structure; window varies.
- **6.3.1** Common report template. Sections: news summary (LLM-
  rendered, grounded in observations, reads `macro_observations`
  directly per §4), word cloud, sector radar (11 sectors), macro signal
  radar. Output formats configurable.
- **6.3.2** Daily report. Window: previous trading day.
- **6.3.3** Weekly report. Window: previous week. Mondays.
- **6.3.4** Monthly report. Window: previous calendar month.
  Surfaces all-pairs lead-lag scan (Story 5.3.2) where relevant.

## Files to touch

- `SecMaster/mcp/Tools/SecMasterTools.cs` (new methods) OR a new
  `AtlasMatrix/mcp/` project hosting matrix-specific tools. Decision
  left to the agent based on tool boundary cleanliness.
- New project or extension: `Reports/` (LLM-rendered narratives over
  `macro_observations`).
- `deployment/grafana/dashboards/atlas-matrix-primary.json` **(new)**
- `deployment/grafana/dashboards/atlas-matrix-trajectory.json` **(new)**
- `deployment/grafana/dashboards/atlas-disagreement.json` **(new)**

## Subagent dispatch notes

- Story 6.1.x: one `general-purpose` agent for MCP tool authoring;
  reuse SecMaster MCP tooling patterns.
- Story 6.2.x: one `general-purpose` agent + Grafana dashboard
  patterns from CLAUDE.md `GRAFANA_DASHBOARD` rules. Common panel
  failures (No data, duplicate values, broken viz) are documented
  there — agent must read CLAUDE.md before producing JSON.
- Story 6.3.x: one `general-purpose` agent. Reports are a new
  service or scheduled job — implementation choice deferred to
  agent's design step.
- PR review: `pr-review-toolkit:review-pr` per story PR.

## Verification (epic-exit AC)

- All MCP tools queryable end-to-end; below-threshold qualitative
  observations visible with trust attribute.
- Sector × phase matrix panel renders for all 11 sectors with
  multi-signal cell visualization preserved (no consensus flattening).
- Drilldown from a cell reveals contributing observations including
  qualitative ones.
- Daily / weekly / monthly reports generate end-to-end. News summary
  cites observation references; word cloud + radars render.
- Disagreement panel surfaces a real divergence (synthetic or live).
- Build + deploy clean; Loki silent post-deploy.
