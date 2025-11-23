# ThresholdEngine Configuration

This directory contains pattern configurations and service settings for ThresholdEngine.

## Directory Structure

```
config/
├── patterns/              # Pattern configuration files
│   ├── recession/         # Recession warning patterns
│   ├── liquidity/         # VIX, credit spreads, liquidity patterns
│   ├── growth/            # Growth acceleration patterns
│   ├── valuation/         # Valuation opportunity patterns
│   └── nbfi/              # NBFI (Non-Bank Financial Institution) patterns
├── regimes.json          # Regime transition rules
└── appsettings.json      # Service configuration
```

## Pattern Configuration Format

Patterns are defined as JSON files with embedded C# expressions:

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

## Pattern Categories

- **Recession**: Defensive patterns (Sahm Rule, ISM contraction, etc.)
- **Liquidity**: VIX deployment, credit spreads, DXY positioning
- **Growth**: GDP acceleration, ISM expansion, industrial production
- **Valuation**: CAPE, Buffett Indicator, equity risk premium
- **NBFI**: Multi-indicator stress detection (Non-Bank Financial Institutions)

## Regime Transition Rules

The `regimes.json` file defines how macro scores map to regimes and transition rules.

## Hot Reload

Pattern configurations support hot reload - edit JSON files and changes take effect within 5 seconds without service restart.

