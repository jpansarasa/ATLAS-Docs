---
name: grafana-dashboards
description: Authoring, editing, debugging, or reviewing Grafana dashboards and queries. Activate when the user creates, edits, or troubleshoots a dashboard, panel, or visualization; mentions Grafana, Loki, Prometheus, Tempo, or TimescaleDB query patterns; writes or reviews PromQL, LogQL, TraceQL, or Grafana-flavored SQL; works with dashboard .json files or files under a dashboards/ directory; sees "no data" or rendering issues in panels; asks about datasource provisioning, variables, thresholds, units, or panel types. Activate even if the user doesn't say "Grafana" explicitly — phrases like "the panel isn't showing data", "fix this query", "build a chart for X metric", or sharing a dashboard JSON all count.
---

# GRAFANA_DASHBOARDS [SKILL v1] [json_provisioning]

ETHOS: query_correctness > visual_polish | scrape-aware_intervals > hardcoded | bounded_cardinality > convenient_tags

COMMON_FAILURES: "No data" | duplicate_values | broken_viz | heatmap_fail
  root_cause: config_mistakes → predictable → eliminable

## DASHBOARD_STRUCTURE
id: null # grafana_assigns_on_save
uid: [a-z0-9-]{8,40} # stable_urls_across_envs
schemaVersion: 41 # current_2025
version: 0 # new_dashboards
required_fields: {time, timezone, panels, editable, refresh}

## PANEL_STRUCTURE
id: unique_int # duplicate → import_fail
gridPos: {x:[0-23], y:int, w:[1-24], h:[3+]} # 24col_grid, ~30px/unit
datasource: {type, uid} # ¬string_name, ¬${DS_NAME}
required: {targets, fieldConfig.defaults, options}

## STAT_PANEL [prevent_duplicates]
reduceOptions.values: false # true → shows_all_rows → duplicates
reduceOptions.calcs: ["lastNotNull"] # "last" → may_return_null
reduceOptions.fields: "" # empty → all_numeric, ¬"/.*/"
query: sum(metric{...}) # aggregate → single_series
instant: true ∧ range: false # ¬both_true → duplicate_data
rationale: multiple_series | range_query | values:true → duplicates

## TIMESERIES_PANEL
fieldConfig.defaults.custom: required # omit → broken_render
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
transformations_required:
  multiple_queries → [{id: "merge"}]
  time_series_data → [{id: "seriesToRows"}]
  sparklines → [{id: "timeSeriesTable"}]
  column_control → [{id: "organize", options: {excludeByName, renameByName}}]
fieldConfig.defaults.custom: {align, cellOptions: {type}, filterable}
overrides: matcher{byName|byRegexp|byType} → properties{cellOptions, width}

## HEATMAP_PANEL
calculate: true # grafana_buckets_data
calculation: {xBuckets: {mode, value}, yBuckets: {mode, value}}
prometheus_histogram: calculate: false ∧ legendFormat: "{{le}}" ∧ yBucketBound: "upper"

## LOG_PANEL [loki]
query_type: log_query # ¬metric_query
options: {showTime, wrapLogMessage, prettifyLogMessage, enableLogDetails, dedupStrategy, sortOrder}
✗ rate() | count_over_time() # these → timeseries_panel

## PROMQL [prometheus_queries]
rate_interval: $__rate_interval # ¬[1m] ¬[5m] # auto_adjusts_to_scrape
  rationale: hardcoded < scrape_interval → "No data"
container_filter: container!="POD", container!="" # always
aggregation: sum by (label1, label2) # control_cardinality
memory: container_memory_working_set_bytes # ¬usage_bytes (includes_cache)
legendFormat: "{{pod}} - {{container}}" # explicit ¬ auto_generated
  rationale: post_aggregation {{__name__}} → empty

## LOGQL [loki_queries]
log_panel: {job="app"} |= "error" | json # no_aggregation
timeseries_panel: sum by (level) (count_over_time({job="app"} | json [5m]))
unwrap: | unwrap field | __error__="" # always_filter_parse_errors

## TRACEQL [tempo_queries]
queryType: "traceql"
operators: span.attr | resource.attr | status | span:duration
example: {resource.service.name="frontend" && status=error}

## POSTGRES [timescaledb]
time_column: required # named "time", datetime | unix_epoch
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

## PANEL_TYPE ↔ QUERY_TYPE [match_required]
log_panel ↔ log_query (LogQL without aggregation)
timeseries_panel ↔ metric_query (PromQL, or LogQL with rate/count_over_time)
stat_panel ↔ single_value (instant query or aggregated range)
table_panel ↔ tabular_result (instant or LogQL with parsing)
✗ mismatch → empty_panel | render_error

## ANTI [HARD_STOP @end_for_recency]
✗ ${DS_NAME} # templated_datasource → provisioning_fail
✗ rate(metric[1m]) # hardcoded_interval → "No data"
✗ omit fieldConfig.defaults # → broken_render
✗ reduceOptions.values: true # → duplicates
✗ stat_panel + multiple_series + no_aggregation # → duplicates
✗ instant: true ∧ range: true # → duplicate_data
✗ log_query → timeseries_panel # → no_results
✗ metric_query → log_panel # → no_results
✗ editing dashboard JSON in Grafana UI when deployed via IaC # → drift_with_next_deploy

## DEBUG_FLOW [no_data | wrong_data]
1. confirm panel_type ↔ query_type match (see table above)
2. check datasource binding (variable resolved? uid present?)
3. run query in Explore view with same time range → reproduces?
4. check $__rate_interval vs scrape_interval
5. check label cardinality (sum by clause matches existing labels?)
6. check fieldConfig.defaults + unit + min/max thresholds
7. timezone | time_column format if TimescaleDB
