# ThresholdEngine Configuration

Pattern definitions, regime transition rules, and service settings for the ThresholdEngine. The host service hot-reloads these files at runtime, so this directory is the operator-facing surface for tuning the engine without rebuilding the container.

## Overview

`ThresholdEngine/config/` is a data/configuration directory, not a code project. It holds the pattern catalog (organised by category), the macro-regime transition rules, and the `appsettings.json` for the service. The ThresholdEngine watches these files and reloads on change (hot-reload latency ~5s), so configuration drift is purely a deploy/file-sync question — not a code change.

This README documents the directory layout, the pattern JSON format, the regime rules, and how to deploy edits.

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

Regime rules in `regimes.json` are loaded the same way and gate which patterns evaluate in which macro regime.

## Features

- **Hot reload** — file changes take effect within ~5s, no service restart.
- **Categorised pattern catalog** — recession / liquidity / growth / valuation / nbfi / inflation, each in its own subfolder.
- **Embedded C# expressions** — patterns evaluate user-supplied C# (Roslyn-scripted) against an `EvaluationContext` exposing `ctx.GetLatest(seriesId)`, etc.
- **Regime gating** — patterns declare `applicableRegimes`; the engine only evaluates them in matching macro regimes.
- **Operator-friendly JSON** — schema is stable; new patterns added by dropping a file in the right category subfolder.

## Configuration

| Key | Description | Default |
|---|---|---|
| `PatternConfig:Path` | Host-mount path the engine watches for pattern JSON | `/app/config/patterns` |
| `appsettings.json` | Standard ASP.NET Core configuration (logging, OpenTelemetry, ConnectionStrings) — see `ThresholdEngine/README.md` for the full env-var matrix | n/a |
| `regimes.json` | Macro-regime score thresholds and transition rules | shipped with image |

Pattern JSON schema (per-file):

```json
{
  "patternId": "vix-deployment-l1",
  "name": "VIX Level 1 Deployment Trigger",
  "description": "VIX >22 indicates elevated volatility, context-dependent signal",
  "category": "Liquidity",
  "expression": "ctx.GetLatest(\"VIXCLS\") > 22m",
  "signalExpression": "ctx.GetLatest(\"VIXCLS\") > 30m ? 2m : 1m",
  "applicableRegimes": ["Crisis", "Recession", "LateCycle"],
  "requiredSeries": ["VIXCLS"],
  "enabled": true,
  "metadata": {
    "deploymentLevel": "L1",
    "allocationAmount": "$25-100K"
  }
}
```

## API Endpoints

This directory does not expose endpoints — it is a configuration mount consumed by the ThresholdEngine host. Pattern operations are exposed by the host service:

- `GET /api/patterns` — list loaded patterns (see [`ThresholdEngine/README.md`](../README.md))
- `POST /api/patterns/reload` — force a reload (normally not needed; the watcher handles it)
- `GET /api/patterns/{id}` — pattern detail and last-evaluation result

For the full surface, see the host project's API Endpoints section.

## Project Structure

```
ThresholdEngine/config/
├── patterns/                # Pattern catalog, organised by category
│   ├── recession/           # Recession warning patterns (Sahm Rule, ISM contraction, ...)
│   ├── liquidity/           # VIX deployment, credit spreads, DXY positioning
│   ├── growth/              # GDP acceleration, ISM expansion, industrial production
│   ├── valuation/           # CAPE, Buffett Indicator, equity risk premium
│   ├── nbfi/                # Multi-indicator NBFI (Non-Bank FI) stress detection
│   └── inflation/           # CPI, PCE, breakeven inflation expectations
├── regimes.json             # Macro-regime transition rules
└── appsettings.json         # ASP.NET Core configuration for the host
```

## Development

### Editing patterns

1. Edit the appropriate JSON file under `patterns/{category}/` on the host (the directory is bind-mounted into the container).
2. Save. The ThresholdEngine watcher picks up the change within ~5 seconds.
3. Verify via `GET /api/patterns/{id}` or the MCP tool `get_pattern` that the new definition is loaded and produces the expected signal.

### Authoring a new pattern

1. Pick the category (or add a new one — just create a subfolder).
2. Choose a `patternId` (kebab-case, unique).
3. Reference required series via `requiredSeries` so SecMaster can validate availability before evaluation.
4. Test the C# expression against `EvaluationContext` semantics — see `ThresholdEngine/src/Patterns/Expressions/` for the surface.

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
