---
name: grafana-dashboards
description: Authoring, editing, debugging, or reviewing Grafana dashboards and queries. Activate when the user creates, edits, or troubleshoots a dashboard, panel, or visualization; mentions Grafana, Loki, Prometheus, Tempo, or TimescaleDB query patterns; writes or reviews PromQL, LogQL, TraceQL, or Grafana-flavored SQL; works with dashboard .json files or files under a dashboards/ directory; sees "no data" or rendering issues in panels; asks about datasource provisioning, variables, thresholds, units, or panel types. Activate even if the user doesn't say "Grafana" explicitly — phrases like "the panel isn't showing data", "fix this query", "build a chart for X metric", or sharing a dashboard JSON all count.
---

# GRAFANA_DASHBOARDS [SKILL v1] [json_provisioning]

ETHOS: query correctness > visual polish | scrape-aware intervals > hardcoded | bounded cardinality > convenient tags

COMMON_FAILURES: "No data" | duplicate values | broken viz | heatmap fail
  root cause: config mistakes -> predictable -> eliminable

## DASHBOARD_STRUCTURE
id: null # grafana assigns on save
uid: [a-z0-9-]{8,40} # stable URLs across envs
schemaVersion: 41 # current 2025
version: 0 # new dashboards
required_fields: {time, timezone, panels, editable, refresh}

## PANEL_STRUCTURE
id: unique int # duplicate -> import fail
gridPos: {x:[0-23], y:int, w:[1-24], h:[3+]} # 24-col grid, ~30px/unit
datasource: {type, uid} # not a string name, not ${DS_NAME}
required: {targets, fieldConfig.defaults, options}

## STAT_PANEL [prevent_duplicates]
reduceOptions.values: false # true -> shows all rows -> duplicates
reduceOptions.calcs: ["lastNotNull"] # "last" -> may return null
reduceOptions.fields: "" # empty -> all numeric, not "/.*/"
query: sum(metric{...}) # aggregate -> single series
instant: true AND range: false # never both true -> duplicate data
rationale: multiple series | range query | values:true -> duplicates

## TIMESERIES_PANEL
fieldConfig.defaults.custom: required # omit -> broken render
  drawStyle: line | bars | points
  lineInterpolation: linear | smooth | stepBefore | stepAfter
  lineWidth: [1-10], fillOpacity: [0-100], gradientMode: none | opacity | hue
  spanNulls: false | true | threshold_ms
  stacking: {mode: none | normal | percent, group: "A"}
  axisPlacement: auto | left | right | hidden
  thresholdsStyle: {mode: off | line | area | line+area}
color: {mode: palette-classic | fixed | thresholds}
options: {legend: {displayMode, placement, showLegend}, tooltip: {mode, sort}}

## TABLE_PANEL
transformations required:
  multiple queries -> [{id: "merge"}]
  time series data -> [{id: "seriesToRows"}]
  sparklines -> [{id: "timeSeriesTable"}]
  column control -> [{id: "organize", options: {excludeByName, renameByName}}]
fieldConfig.defaults.custom: {align, cellOptions: {type}, filterable}
overrides: matcher{byName|byRegexp|byType} -> properties{cellOptions, width}

## HEATMAP_PANEL
calculate: true # grafana buckets data
calculation: {xBuckets: {mode, value}, yBuckets: {mode, value}}
prometheus histogram: calculate: false AND legendFormat: "{{le}}" AND yBucketBound: "upper"

## LOG_PANEL [loki]
query type: log query # not metric query
options: {showTime, wrapLogMessage, prettifyLogMessage, enableLogDetails, dedupStrategy, sortOrder}
✗ rate() | count_over_time() # these -> timeseries panel

## PROMQL [prometheus_queries]
rate interval: $__rate_interval # not [1m], not [5m] # auto-adjusts to scrape
  rationale: hardcoded < scrape interval -> "No data"
container filter: container!="POD", container!="" # always
aggregation: sum by (label1, label2) # control cardinality
memory: container_memory_working_set_bytes # not usage_bytes (includes cache)
legendFormat: "{{pod}} - {{container}}" # explicit, not auto-generated
  rationale: post-aggregation {{__name__}} -> empty

## LOGQL [loki_queries]
log panel: {job="app"} |= "error" | json # no aggregation
timeseries panel: sum by (level) (count_over_time({job="app"} | json [5m]))
unwrap: | unwrap field | __error__="" # always filter parse errors

## TRACEQL [tempo_queries]
queryType: "traceql"
operators: span.attr | resource.attr | status | span:duration
example: {resource.service.name="frontend" && status=error}

## POSTGRES [timescaledb]
time column: required # named "time", datetime | unix epoch
order: ORDER BY time ASC # mandatory
filter: WHERE $__timeFilter(column)
format: "time_series"
bucket: time_bucket('5 minutes', time) AS time

## UNITS [field.unit]
time: s | ms | µs | ns | m | h | d
data: decbytes | bytes | bits | kbytes | mbytes
rate: Bps | bps | KBps | MBps
percent: percent(0-100) | percentunit(0-1)
throughput: reqps | ops | rps | iops

## PANEL_TYPE <-> QUERY_TYPE [match_required]
log panel <-> log query (LogQL without aggregation)
timeseries panel <-> metric query (PromQL, or LogQL with rate/count_over_time)
stat panel <-> single value (instant query or aggregated range)
table panel <-> tabular result (instant or LogQL with parsing)
✗ mismatch -> empty panel | render error

## ANTI [HARD_STOP @end_for_recency]
✗ ${DS_NAME} # templated datasource -> provisioning fail
✗ rate(metric[1m]) # hardcoded interval -> "No data"
✗ omit fieldConfig.defaults # -> broken render
✗ reduceOptions.values: true # -> duplicates
✗ stat panel + multiple series + no aggregation # -> duplicates
✗ instant: true AND range: true # -> duplicate data
✗ log query -> timeseries panel # -> no results
✗ metric query -> log panel # -> no results
✗ editing dashboard JSON in Grafana UI when deployed via IaC # -> drift with next deploy

## DEBUG_FLOW [no_data | wrong_data]
1. confirm panel type <-> query type match (see table above)
2. check datasource binding (variable resolved? uid present?)
3. run query in Explore view with same time range -> reproduces?
4. check $__rate_interval vs scrape interval
5. check label cardinality (sum by clause matches existing labels?)
6. check fieldConfig.defaults + unit + min/max thresholds
7. timezone | time column format if TimescaleDB
