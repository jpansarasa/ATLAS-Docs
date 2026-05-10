# ThresholdEngine Configuration

Pattern definitions and service settings for the ThresholdEngine. The host service hot-reloads these files at runtime, so this directory is the operator-facing surface for tuning the engine without rebuilding the container.

## Overview

`ThresholdEngine/config/` is a data/configuration directory, not a code project. It holds the pattern catalog (organised by theme directory) and the `appsettings.json` for the service. The ThresholdEngine watches these files and reloads on change (hot-reload latency ~5s), so configuration drift is purely a deploy/file-sync question — not a code change.

This README documents the directory layout, the pattern JSON format, and how to deploy edits.

## Architecture

```
operator edit ─► config/patterns/*.json
                        │
                        ▼
        FileSystemWatcher (in ThresholdEngine host)
                        │
                        ▼
            PatternRegistry.Reload() ─► pattern evaluation
```

Pattern files live on a host-mounted volume (read-only inside the container). The `PatternConfig:Path` setting in `appsettings.json` points at the mount. Edits made on the host appear in the container immediately and are picked up by the watcher within ~5 seconds. No rebuild, no restart.

## Features

- **Hot reload** — file changes take effect within ~5s, no service restart.
- **Themed pattern catalog** — patterns live under theme subfolders (recession / liquidity / growth / valuation / nbfi / inflation) for human navigation; the directory name is descriptive, not part of the schema.
- **Embedded C# expressions** — patterns evaluate user-supplied C# (Roslyn-scripted) against an `EvaluationContext` exposing `ctx.GetLatest(seriesId)`, etc.
- **Sector projection** — patterns declare `sectorWeights` over the 11 ATLAS sectors; the engine multiplies the per-pattern signal by each sector weight to produce a per-cell projection.
- **Operator-friendly JSON** — schema is stable; unknown fields are rejected at load time so legacy keys cannot silently degrade.

## Configuration

| Key | Description | Default |
|---|---|---|
| `PatternConfig:Path` | Host-mount path the engine watches for pattern JSON | `/app/config/patterns` |
| `appsettings.json` | Standard ASP.NET Core configuration (logging, OpenTelemetry, ConnectionStrings) — see `ThresholdEngine/README.md` for the full env-var matrix | n/a |

Pattern JSON schema (per-file):

```json
{
  "patternId": "vix-deployment-l1",
  "name": "VIX Level 1 Deployment Trigger",
  "description": "VIX >22 indicates elevated volatility, context-dependent signal",
  "expression": "ctx.GetLatest(\"VIXCLS\") > 22m",
  "signalExpression": "ctx.GetLatest(\"VIXCLS\") > 30m ? 2m : 1m",
  "applicableRegimes": ["Crisis", "Recession", "LateCycle"],
  "requiredSeries": ["VIXCLS"],
  "sectorWeights": {
    "ENERGY": 0.0, "MATERIALS": 0.0, "INDUSTRIALS": 0.0,
    "CONS_DISC": 0.5, "CONS_STAPLES": 0.0, "HEALTHCARE": 0.0,
    "FINANCIALS": 1.0, "INFOTECH": 0.5, "COMM_SVC": 0.0,
    "UTILITIES": 0.0, "REAL_ESTATE": 0.0
  },
  "enabled": true,
  "metadata": {
    "deploymentLevel": "L1",
    "allocationAmount": "$25-100K"
  }
}
```

`sectorWeights` is required and must enumerate all 11 ATLAS sectors with explicit numeric weights; zeros are how you say "this pattern does not affect that sector".

## API Endpoints

This directory does not expose endpoints — it is a configuration mount consumed by the ThresholdEngine host. Pattern operations are exposed by the host service:

- `GET /api/patterns` — list loaded patterns (see [`ThresholdEngine/README.md`](../README.md))
- `POST /api/patterns/reload` — force a reload (normally not needed; the watcher handles it)
- `GET /api/patterns/{id}` — pattern detail and last-evaluation result

For the full surface, see the host project's API Endpoints section.

## Project Structure

```
ThresholdEngine/config/
├── patterns/                # Pattern catalog, organised by theme directory
│   ├── recession/           # Recession warning patterns (Sahm Rule, ISM contraction, ...)
│   ├── liquidity/           # VIX deployment, credit spreads, DXY positioning
│   ├── growth/              # GDP acceleration, ISM expansion, industrial production
│   ├── valuation/           # CAPE, Buffett Indicator, equity risk premium
│   ├── nbfi/                # Multi-indicator NBFI (Non-Bank FI) stress detection
│   └── inflation/           # CPI, PCE, breakeven inflation expectations
└── appsettings.json         # ASP.NET Core configuration for the host
```

## Development

### Editing patterns

1. Edit the appropriate JSON file under `patterns/{theme}/` on the host (the directory is bind-mounted into the container).
2. Save. The ThresholdEngine watcher picks up the change within ~5 seconds.
3. Verify via `GET /api/patterns/{id}` or the MCP tool `get_pattern` that the new definition is loaded and produces the expected signal.

### Authoring a new pattern

1. Pick the theme directory (or add a new one — just create a subfolder; the directory name is descriptive only).
2. Choose a `patternId` (kebab-case, unique).
3. Reference required series via `requiredSeries` so SecMaster can validate availability before evaluation.
4. Populate `sectorWeights` with all 11 ATLAS sectors (zeros for sectors the pattern does not affect).
5. Test the C# expression against `EvaluationContext` semantics — see `ThresholdEngine/src/Patterns/Expressions/` for the surface.

### Local validation

The host project ships pattern-load tests:

```bash
bash ThresholdEngine/.devcontainer/compile.sh
```

Failing JSON or unparseable expressions are caught at load time, not at first evaluation.

## Deployment

This directory is shipped to the host via the patterns-deploy ansible tag, separate from the container image:

```bash
ansible-playbook playbooks/deploy.yml --tags patterns
```

This task syncs the local files to `/opt/ai-inference/threshold-engine/config/patterns/` (the bind-mount source) and the container picks up the changes via the watcher. No container restart is required for pattern edits.

For service config (`appsettings.json`) changes, run the full service tag instead:

```bash
ansible-playbook playbooks/deploy.yml --tags threshold-engine
```

Per CLAUDE.md `DEPLOYMENT [HARD_STOP]`: never edit `/opt/ai-inference/compose.yaml` directly.

## Ports

This directory exposes no ports. The host service ports apply — see [`ThresholdEngine/README.md` Ports section](../README.md#ports).

## See Also

- [ThresholdEngine](../README.md) — host service, evaluation engine, REST API
- [ThresholdEngine MCP](../mcp/README.md) — MCP wrapper exposing pattern tools to AI assistants
- [SecMaster](../../SecMaster/README.md) — series resolution and metadata used by `requiredSeries` validation
- [FredCollector](../../FredCollector/README.md) — primary upstream for economic series referenced by patterns
