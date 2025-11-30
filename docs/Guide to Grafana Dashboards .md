# Definitive Guide to Grafana JSON Dashboard Provisioning for AI Coding Assistants

**The challenge is real**: LLMs consistently struggle with Grafana dashboard JSON because, as noted by practitioners, it's "poorly documented, without a schema file, with possible redundancies or deprecated features." This guide provides the precise rules, patterns, and anti-patterns needed to generate dashboards that render correctly.

## The core problems and their solutions

Four specific issues plague AI-generated Grafana dashboards: "No data" panels, duplicate values (10-20x in stat panels), broken visualizations, and heatmap/log panel failures. Each stems from predictable JSON configuration mistakes that can be eliminated with explicit rules.

**"No data" panels** occur when queries return results but panels can't interpret them. Root causes include rate intervals smaller than scrape intervals, missing aggregation in queries, wrong field mappings, and query/visualization type mismatches. **Duplicate values** in stat panels happen when `reduceOptions.values: true` is set (showing all rows), when multiple series return without aggregation, or when both `instant` and `range` queries are enabled. **Broken visualizations** result from missing `fieldConfig.defaults.custom` blocks, wrong unit specifications, and absent transformations. **Heatmap and log panel failures** stem from wrong data format configurations and missing panel-specific options.

---

## Dashboard JSON structure requirements

### Minimum viable dashboard

Every provisioned dashboard must include these fields:

```json
{
  "id": null,
  "uid": "unique-dashboard-uid",
  "title": "Dashboard Title",
  "schemaVersion": 41,
  "version": 0,
  "panels": [],
  "time": {"from": "now-6h", "to": "now"},
  "timezone": "browser",
  "editable": true,
  "refresh": ""
}
```

The `id` must be `null` for new dashboards—Grafana assigns it on save. The `uid` should be **8-40 lowercase alphanumeric characters with hyphens**, enabling stable URLs across environments. Setting `schemaVersion: 41` (current as of late 2025) ensures compatibility with modern panel types.

### Panel structure fundamentals

Every panel requires these fields without exception:

```json
{
  "id": 1,
  "type": "timeseries",
  "title": "Panel Title",
  "gridPos": {"x": 0, "y": 0, "w": 12, "h": 8},
  "datasource": {"type": "prometheus", "uid": "prometheus-uid"},
  "targets": [],
  "fieldConfig": {
    "defaults": {},
    "overrides": []
  },
  "options": {}
}
```

**Panel IDs must be unique integers** within a dashboard—duplicate IDs cause import failures. The **gridPos** uses a 24-column grid where `w` ranges 1-24, `h` is in grid units (each ~30 pixels, minimum 3), and `x`/`y` coordinates must not cause overlapping.

### Data source references

Always reference data sources by type and UID, never by name alone:

```json
"datasource": {
  "type": "prometheus",
  "uid": "actual-datasource-uid"
}
```

The `${DS_PROMETHEUS}` syntax from exported dashboards will fail in provisioning—replace all templated data source references with actual UIDs.

---

## Stat panel configuration to prevent duplicates

Stat panels are the most common source of duplicate value bugs. The **critical configuration** is `reduceOptions`:

```json
{
  "type": "stat",
  "options": {
    "reduceOptions": {
      "values": false,
      "calcs": ["lastNotNull"],
      "fields": ""
    },
    "colorMode": "value",
    "graphMode": "area",
    "justifyMode": "auto",
    "textMode": "auto",
    "orientation": "auto"
  },
  "fieldConfig": {
    "defaults": {
      "thresholds": {
        "mode": "absolute",
        "steps": [
          {"color": "green", "value": null},
          {"color": "red", "value": 80}
        ]
      },
      "unit": "short",
      "mappings": []
    },
    "overrides": []
  }
}
```

### Rules to prevent stat panel duplicates

| Rule | Why It Matters |
|------|----------------|
| Set `values: false` | `true` displays all rows from all series, causing apparent duplicates |
| Use `calcs: ["lastNotNull"]` | Reduces each series to single value; "last" may return null |
| Set `fields: ""` | Empty string selects all numeric fields; `"/.*/"`  can include non-numeric causing issues |
| Aggregate queries to single series | Multiple series = multiple values displayed |
| Use instant queries for current values | Range queries return multiple points per series |
| Never enable both `instant: true` AND `range: true` | Returns duplicate data points |

**Query-side fix for duplicates**: Always aggregate to a single series when displaying in stat panels:

```promql
# WRONG - Returns multiple series (one per pod/container combination)
container_memory_working_set_bytes{pod="my-pod"}

# CORRECT - Aggregated to single value
sum(container_memory_working_set_bytes{pod="my-pod", container!=""})
```

---

## Time series panel configuration

Time series panels require the `fieldConfig.defaults.custom` block—omitting it causes broken or default rendering:

```json
{
  "type": "timeseries",
  "fieldConfig": {
    "defaults": {
      "custom": {
        "drawStyle": "line",
        "lineInterpolation": "linear",
        "lineWidth": 1,
        "fillOpacity": 10,
        "gradientMode": "none",
        "spanNulls": false,
        "showPoints": "auto",
        "pointSize": 5,
        "stacking": {"mode": "none", "group": "A"},
        "axisPlacement": "auto",
        "axisLabel": "",
        "scaleDistribution": {"type": "linear"},
        "hideFrom": {"legend": false, "tooltip": false, "viz": false},
        "thresholdsStyle": {"mode": "off"}
      },
      "color": {"mode": "palette-classic"},
      "unit": "short",
      "thresholds": {
        "mode": "absolute",
        "steps": [{"color": "green", "value": null}]
      }
    },
    "overrides": []
  },
  "options": {
    "legend": {"displayMode": "list", "placement": "bottom", "showLegend": true},
    "tooltip": {"mode": "single", "sort": "none"}
  }
}
```

### Field overrides structure

Use overrides for per-series customization:

```json
"overrides": [
  {
    "matcher": {"id": "byName", "options": "cpu_usage"},
    "properties": [
      {"id": "color", "value": {"fixedColor": "red", "mode": "fixed"}},
      {"id": "custom.lineWidth", "value": 2},
      {"id": "custom.axisPlacement", "value": "right"},
      {"id": "unit", "value": "percent"}
    ]
  }
]
```

Matcher types: `byName` (exact field name), `byRegexp` (pattern), `byType` (number/string/time), `byFrameRefID` (query A/B/C).

---

## Table panel configuration to avoid rudimentary appearance

Tables look "rudimentary" when missing transformations and cell styling:

```json
{
  "type": "table",
  "fieldConfig": {
    "defaults": {
      "custom": {
        "align": "auto",
        "cellOptions": {"type": "auto"},
        "inspect": false,
        "filterable": true
      }
    },
    "overrides": [
      {
        "matcher": {"id": "byName", "options": "status"},
        "properties": [
          {"id": "custom.cellOptions", "value": {"type": "color-background"}},
          {"id": "custom.width", "value": 100}
        ]
      }
    ]
  },
  "options": {
    "showHeader": true,
    "cellHeight": "sm",
    "footer": {"show": false, "reducer": ["sum"], "fields": ""}
  }
}
```

### Required transformations by data type

| Data Source | Transformation Needed |
|-------------|----------------------|
| Multiple time series | `seriesToRows` or `merge` |
| Multiple queries | `merge` |
| Time series for sparklines | `timeSeriesTable` |
| Wide format needing column control | `organize` |

```json
"transformations": [
  {"id": "merge", "options": {}},
  {
    "id": "organize",
    "options": {
      "excludeByName": {"__name__": true},
      "renameByName": {"value": "Current Value"}
    }
  }
]
```

---

## Heatmap panel configuration

Heatmaps fail silently without correct data format configuration:

```json
{
  "type": "heatmap",
  "options": {
    "calculate": true,
    "calculation": {
      "xBuckets": {"mode": "size", "value": "1h"},
      "yBuckets": {"mode": "count", "value": "10"}
    },
    "color": {
      "mode": "scheme",
      "scheme": "Oranges",
      "steps": 64
    },
    "cellGap": 1,
    "filterValues": {"le": 1e-9},
    "legend": {"show": true},
    "tooltip": {"show": true, "yHistogram": false},
    "yAxis": {"axisPlacement": "left", "reverse": false}
  }
}
```

**For Prometheus histograms** (pre-bucketed data), set `calculate: false` and ensure legend format is `{{le}}`. Set `yBucketBound: "upper"` because Prometheus uses upper bounds.

---

## Log panel configuration for Loki

```json
{
  "type": "logs",
  "datasource": {"type": "loki", "uid": "loki-uid"},
  "targets": [{
    "expr": "{job=\"mysql\"} |= \"error\" | json",
    "refId": "A"
  }],
  "options": {
    "showTime": true,
    "showLabels": false,
    "showCommonLabels": false,
    "wrapLogMessage": true,
    "prettifyLogMessage": true,
    "enableLogDetails": true,
    "dedupStrategy": "none",
    "sortOrder": "Descending"
  }
}
```

**Critical distinction**: Log panels use log queries (returning strings), time series panels use metric queries with `rate()`, `count_over_time()`. Using a metric query in a logs panel or vice versa produces no results.

---

## Prometheus query patterns for containerd metrics

### Preventing "No data" with correct rate intervals

**Always use `$__rate_interval`** instead of hardcoded intervals:

```promql
# WRONG - May return "No data" when zoomed in
rate(container_cpu_usage_seconds_total[1m])

# CORRECT - Automatically adjusts to scrape interval
rate(container_cpu_usage_seconds_total[$__rate_interval])
```

The `$__rate_interval` is calculated as `max($__interval + scrape_interval, 4 * scrape_interval)`, guaranteeing enough data points exist.

### Containerd metrics templates

**CPU usage (percentage)**:
```promql
sum by (pod, container) (
  rate(container_cpu_usage_seconds_total{
    container!="POD",
    container!=""
  }[$__rate_interval])
) * 100
```

**Memory working set** (preferred over `usage_bytes` which includes cache):
```promql
sum by (pod, container) (
  container_memory_working_set_bytes{container!="POD", container!=""}
)
```

**Network receive rate**:
```promql
sum by (pod) (
  rate(container_network_receive_bytes_total[$__rate_interval])
)
```

### Legend formatting to prevent display duplicates

```promql
# Explicit legend prevents confusion from auto-generated labels
legendFormat: "{{pod}} - {{container}}"
```

After aggregation functions like `sum`, `{{__name__}}` returns empty—use explicit labels or static text.

### Query anti-patterns

| Anti-Pattern | Problem | Solution |
|--------------|---------|----------|
| `rate(metric[1m])` with 60s scrape | No data on zoom | Use `$__rate_interval` |
| No `by` in aggregation | All labels retained, duplicates | Add `by (label1, label2)` |
| Missing `container!=""` filter | Includes pause containers | Always filter empty container |
| `container_memory_usage_bytes` | Includes kernel cache | Use `working_set_bytes` |

---

## Loki LogQL query patterns

### Log queries vs metric queries

**For log panels** (displaying raw logs):
```logql
{container="query-frontend"} |= "error" | json
```

**For time series panels** (metrics from logs):
```logql
sum by (level) (count_over_time({job="app"} | json [5m]))
```

### Unwrap for numeric field aggregation

```logql
avg_over_time(
  {container="nginx"} 
  | json 
  | unwrap response_latency_seconds 
  | __error__=""
  [1m]
) by (cluster)
```

**Always include `| __error__=""`** after unwrap to filter parsing errors.

---

## Tempo TraceQL query patterns

```json
{
  "datasource": {"type": "tempo", "uid": "tempo-uid"},
  "targets": [{
    "query": "{resource.service.name=\"frontend\" && status=error}",
    "queryType": "traceql",
    "limit": 20,
    "refId": "A"
  }],
  "type": "traces"
}
```

Key TraceQL operators: `span.` (span attributes), `resource.` (resource attributes), `status` (error/ok/unset), `span:duration` (duration comparison).

---

## TimescaleDB/PostgreSQL query patterns

### Mandatory requirements for time series format

```sql
SELECT
  time_bucket('5 minutes', time) AS time,
  avg(value) AS value
FROM metrics
WHERE $__timeFilter(time)
GROUP BY 1
ORDER BY 1
```

**Three rules that prevent rendering failures**:
1. Must include column named `time` returning datetime or Unix epoch
2. Must be sorted by `time` ascending
3. Must set query format to "time_series"

### Grafana macros

| Macro | Usage |
|-------|-------|
| `$__timeFilter(column)` | `WHERE $__timeFilter(created_at)` |
| `$__timeFrom()` / `$__timeTo()` | Range boundaries |
| `$__timeGroup(column, interval)` | Time bucketing |

---

## Unit specifications

Common mistake: using wrong unit strings. Valid units include:

| Category | Units |
|----------|-------|
| Time | `s`, `ms`, `µs`, `ns`, `m`, `h`, `d` |
| Data | `decbytes`, `bytes`, `bits`, `kbytes`, `mbytes` |
| Data rate | `Bps`, `bps`, `KBps`, `MBps` |
| Percentage | `percent` (0-100), `percentunit` (0-1) |
| Throughput | `reqps`, `ops`, `rps`, `iops` |

---

## API provisioning via POST /api/dashboards/db

```json
{
  "dashboard": {
    "id": null,
    "uid": "my-dashboard",
    "title": "My Dashboard",
    "schemaVersion": 41,
    "panels": [...]
  },
  "folderUid": "folder-uid",
  "message": "Initial creation",
  "overwrite": false
}
```

Set `overwrite: true` for updates. Response includes `uid`, `url`, `version` for confirmation.

---

## Complete CLAUDE.md rules section

```markdown
## Grafana Dashboard JSON Generation Rules

### Dashboard Structure
- Set `id: null` for new dashboards
- Generate stable `uid`: 8-40 chars, lowercase alphanumeric + hyphens
- Set `schemaVersion: 41` (current version)
- Set `version: 0` for new dashboards
- Always include `time`, `timezone`, `panels` fields

### Panel Configuration
- Assign unique integer `id` to each panel (auto-increment from 1)
- `gridPos.w`: 1-24 (24-column grid)
- `gridPos.h`: minimum 3, in grid units
- Reference datasources by `{type, uid}` object, never string name
- Always include `fieldConfig.defaults` with thresholds

### Stat Panels (Preventing Duplicates)
- Set `options.reduceOptions.values: false`
- Set `options.reduceOptions.calcs: ["lastNotNull"]`
- Aggregate queries to single series: `sum(metric{...})`
- Use instant queries for current values
- Never enable both `instant: true` AND `range: true`

### Time Series Panels
- Always include `fieldConfig.defaults.custom` block
- Specify `drawStyle`, `lineWidth`, `fillOpacity`
- Include `options.legend` and `options.tooltip`

### Table Panels
- Add `transformations: [{id: "merge"}]` for multiple queries
- Add `transformations: [{id: "seriesToRows"}]` for time series data
- Configure `custom.cellOptions` for styling

### Prometheus Queries
- Always use `$__rate_interval` in rate() and irate()
- Always include `container!="POD", container!=""` filters
- Use `sum by (labels)` to aggregate and control cardinality
- Use `container_memory_working_set_bytes` not `usage_bytes`
- Set custom `legendFormat`: "{{pod}} - {{container}}"

### Loki Queries
- Log panels: Use log queries without aggregation functions
- Time series panels: Wrap with rate(), count_over_time()
- Include `| __error__=""` after unwrap expressions

### PostgreSQL/TimescaleDB
- Column named `time` is mandatory for time series
- Must include `ORDER BY time`
- Use `$__timeFilter(column)` in WHERE clause
- Set format to "time_series"

### Anti-Patterns to Avoid
- Never use `${DS_NAME}` templated datasource references
- Never hardcode rate intervals like `[1m]` or `[5m]`
- Never omit `fieldConfig.defaults` structure
- Never set `reduceOptions.values: true` in stat panels
- Never forget aggregation when displaying single values
```

---

## Authoritative reference links

| Resource | URL |
|----------|-----|
| Dashboard JSON Model | https://grafana.com/docs/grafana/latest/dashboards/build-dashboards/view-dashboard-json-model/ |
| Dashboard API | https://grafana.com/docs/grafana/latest/developer-resources/api-reference/http-api/dashboard/ |
| Provisioning Guide | https://grafana.com/docs/grafana/latest/administration/provisioning/ |
| Prometheus Data Source | https://grafana.com/docs/grafana/latest/datasources/prometheus/ |
| Loki LogQL | https://grafana.com/docs/loki/latest/query/ |
| Tempo TraceQL | https://grafana.com/docs/tempo/latest/traceql/ |
| PostgreSQL Data Source | https://grafana.com/docs/grafana/latest/datasources/postgres/ |
| Dashboard Linter | https://github.com/grafana/dashboard-linter |
| Grafonnet | https://github.com/grafana/grafonnet |
| MCP Grafana Server | https://github.com/grafana/mcp-grafana |

## Conclusion

The path to reliable AI-generated Grafana dashboards lies in explicit configuration rules rather than relying on LLM pattern matching. The key insight from Grafana Labs' own engineering team: LLMs "return helpful but verbose responses" that often miss the precise JSON structure requirements. By encoding the rules in this guide—particularly the stat panel `reduceOptions` configuration, mandatory `$__rate_interval` usage, and transformation requirements for tables—an AI coding assistant can generate dashboards that render correctly on first deployment. The dashboard-linter tool provides excellent validation, and the MCP Grafana server patterns demonstrate how to manage context efficiently when working with large dashboard JSONs.