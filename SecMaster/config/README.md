# SecMaster/config

Instrument configuration files consumed by SecMaster at startup. Mounted into the container at `/app/config` (overridable via `InstrumentConfiguration:ConfigDirectory`).

## Layout

```
config/
  instruments/
    <symbol>.json           — one file per instrument
```

Each `.json` file declares a single instrument that should exist in the SecMaster catalog after startup seeding, including its metadata + source mapping intent. Anything not in this directory still flows in via collector gRPC registration; this directory is the **operator-curated** set that we want present regardless of whether any collector has registered them yet.

## File shape

```json
{
  "symbol": "ADP_EMPLOYMENT",
  "name": "ADP National Employment Report",
  "description": "Monthly estimate of private nonfarm payroll employment derived from ADP payroll processing data ...",
  "assetClass": "Economic",
  "instrumentType": "Indicator",
  "frequency": "Monthly",
  "country": "US",
  "isActive": true,
  "metadata": {
    "source": "ADP Research Institute",
    "series_type": "Alternative",
    "temporal_type": "Coincident",
    "lead_time_months": 0,
    "related_official": "PAYEMS"
  }
}
```

| Field | Notes |
|---|---|
| `symbol` | Canonical SecMaster symbol — typically uppercase, underscore-separated. Used as the natural key during startup seeding. |
| `name` | Human-readable name shown in admin UIs and RAG context. |
| `description` | Long-form description — feeds the semantic-search embedding (BGE-M3, 1024-dim). Substantive description text materially improves retrieval quality. |
| `assetClass` / `instrumentType` / `frequency` | Categorical metadata mirroring the EF entity. |
| `country` | ISO 3166-1 alpha-2. |
| `isActive` | If `false`, the instrument is loaded but excluded from active resolution. |
| `metadata` | Free-form key/value bag for collector-specific hints (`series_type`, `temporal_type`, `lead_time_months`, `related_official`, etc.). Surfaced through the API as-is. |

## When to add / remove a file

- **Add** when introducing an instrument we want present in the catalog independently of any collector having registered it. Common for alternative-data series that flow through Sentinel rather than a structured collector.
- **Remove** when the instrument is fully owned by a collector and no longer needs operator-curated metadata. Once collectors register the symbol, the registration path becomes the source of truth.

## Anti-pattern reminder

Per CLAUDE.md `DATABASE [ef_core]` — these files are **inputs to app-level seeding on startup**, not raw SQL applied at deploy. Edits land via container restart, not psql.

## See Also

- [SecMaster](../README.md) — service README (config keys + seeding entry points)
- [SecMaster/src/Endpoints/InstrumentEndpoints.cs](../src/Endpoints/InstrumentEndpoints.cs) — API surface for the resulting instruments
