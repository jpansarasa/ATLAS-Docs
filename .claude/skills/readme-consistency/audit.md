# Audit Signals

Reference for `scripts/audit.sh`. Each signal has a cheap implementation (no LLM)
and triggers a specific severity. The skill reads this file to know what to check.

## Severity legend

- **CRITICAL** — project dir has no README at all
- **HIGH** — README exists but missing one of the 10 required sections
- **MEDIUM** — staleness signal fires (see below)
- **LOW** — tone/depth deviation from gold example (LLM-only; skipped in cheap audit)

## Required 10 sections (HIGH if any missing)

`Overview`, `Architecture`, `Features`, `Configuration`, `API Endpoints`,
`Project Structure`, `Development`, `Deployment`, `Ports`, `See Also`.

For sub-components (`*/mcp/`, `*/config/`, `*/scripts/`), the lighter required
set is: `Overview`, `Usage`, `See Also`.

## MEDIUM-severity staleness signals

| ID | Signal | Cheap implementation |
|----|--------|----------------------|
| S1 | README older than newest src/ commit by >30d | `git log -1 --format=%ct -- README.md` vs. `git log -1 --format=%ct -- src/` |
| S2 | New entities not in Project Structure | count `src/Data/Entities/*.cs` files newer than README mtime; compare to README's section content |
| S3 | New endpoints not in API Endpoints | count `src/Endpoints/*Endpoints.cs` files newer than README mtime; verify each is referenced in README |
| S4 | New migrations not in Architecture | count `src/Data/Migrations/*.cs` files (excluding Designer.cs) newer than README mtime |
| S5 | New env vars in code not in Configuration | grep `IConfiguration["..."]` and `Environment.GetEnvironmentVariable\(` in `src/`; for each unique key, verify it's in the Configuration section of README |
| S6 | New port in compose.yaml.j2 not in Ports section | grep `5\d{3}:` mapping for the service vs. README's "Ports" |

Each signal that fires emits one finding. A single project can accumulate
multiple findings across signals.

## Output shape (per finding)

~~~json
{
  "path": "{relative-path}",
  "severity": "CRITICAL | HIGH | MEDIUM",
  "section": "{section-name or null}",
  "signal": "{S1..S6 | missing_readme | missing_section}",
  "message": "{human-readable detail}"
}
~~~

## Project enumeration rules

A path is a target if it has any of:
- `*.csproj` (any depth <= 2)
- `pyproject.toml`
- `package.json`

Targets are deduplicated to top-level project dirs. Sub-components (`*/mcp/`,
`*/config/`, `*/scripts/`) are listed separately if they have their own README
or are referenced as a sibling component.

Excluded paths:
- `/.git`, `/.pytest_cache`, `/node_modules`, `/bin`, `/obj`
- vendored code (anywhere matching `**/vendor/`)
- supervisor-owned dirs (`docs/superpowers/`)
