# CLAUDE.md [CODE.GEN.RULES v1.0]

## ETHOS
pragmatic > dogmatic
working > perfect
maintain > clever

## PRIORITY_0 [always_enforce]
COMPLETE_IMPL: accept_criteria → full ∧ ¬TODO ∧ ¬placeholder
  rationale: TODO = placeholder_debt(never_done | should_do_now)
MIN_MAINTAIN: ∀line justify(existence) | fewest_lines : maintain_easy
  rationale: every_line = error_opportunity
LANG_NATIVE: Rust≠Python≠C#≠TS≠PS | ¬port_pattern
  principle: native_style ¬ pattern_force

## PROBLEM_SOLVING [meta_process] [PRIORITY_0]
NEVER: repeat(failed_attempt) # insanity_definition
PROCESS:
  1. understand(problem) → root_cause
  2. hypothesis(fix) → rationale
  3. attempt(fix)
  4. IF fail → analyze(why) ¬ revert_thrash
  5. track(attempted) → ¬repeat
DEBUG_METHOD:
  observe → hypothesize → test → learn → iterate
  ¬pattern: try → fail → try_same → fail → revert → try_original
IF stuck THEN reassess(understanding) ¬ retry(same_approach)

## TASK_MANAGEMENT [complex_work]
IF complex THEN create(STATE.md) # compact_encoding
FILE: STATE.md
STRUCTURE:
  GOAL → {accept_criteria, constraints}
  ARCH → {decisions, rationale, implications}
  ATTEMPTED → {approach, result, fail_reason} # ¬repeat
  STATUS → {✓done, →active, ⧗blocked, ◯next}
  CONTEXT → {files, deps, external_systems}
UPDATE_TRIGGER: task_start | major_decision | attempt_fails | step_completes | stuck
COMPLEX_INDICATORS: multi_file | debugging_non_trivial | refactor_significant | new_system | conversation_depth > 10
PROACTIVE: create + update_without_asking

## BOUNDARY_HANDLING [fn_scope]
ENTRY → validate(params) : feedback(clear)
  example: validate(index ∈ [0,len)) else "Index 5 exceeds bounds [0,3)"
INTERNAL_CALL → trust(validated) : propagate(exception)
  rationale: assume_good_handling
EXTERNAL_CALL → handle{log(WARN)+continue} | fail{capture(context)+raise}
  context_min = {index, state, params}
  rationale: external=uncertainty_boundary

## LOG_RULES [production_default=WARN]
quality: enable_diagnosis ¬ filter_noise
format: structured(Loki_parse=true)
template_good: "Null at items[{idx}] user={uid} state={s}"
template_bad: "Error occurred"
chain: log_good + err_good → test_less
purpose: runtime_debug ¬ noise
LEVELS:
  LogInformation: routine_ops | expected_retries | client_disconnect
    rationale: filtered_in_prod, visible_in_dev
  LogWarning: unexpected_but_recoverable | degraded_state
  LogError: failures | exceptions | requires_attention

## DOC [selective]
CLI → help(required)
fn → params(brief) # autocomplete_aid
inline → minimal # rot_risk
design → .md ¬ code
principle: self_doc + strategic_comment

## TEST [selective]
IF complex ∧ fail_prone → unit
IF idempotent ∧ valuable → integration
¬IF {simple | log_suffices | setup_complex}
coverage=100 → smell
rationale: test_right_things ¬ test_everything

## OBS_STACK [where_sensible ¬ everywhere]
OTEL → Loki(logs) + Prom(metrics) + Tempo(traces) + Grafana(viz)
metric_purpose: {perf_tune, bottleneck_diagnosis}
config: lightweight(tunables+ext_connections) ¬ framework
when: sensible_value ¬ dogmatic
DEBUGGING [service_issues]:
  1. Grafana FIRST → Tempo(error traces) + Loki(error logs)
  2. ¬nerdctl_logs # container_stdout = last_resort
  rationale: exceptions_have_trace_context + searchable + dashboarded
  if_no_data_in_grafana → check otel-collector + check service restart timing
  stack: Serilog.Sinks.OpenTelemetry → otel-collector → Loki
METRICS:
  location: uncertainty_boundaries ¬ internal_layers
    ✓ service_edge: HTTP/gRPC endpoints (latency, errors, throughput)
    ✓ external_api: upstream calls (FRED, SecMaster) - rate limits, degradation
    ✓ database: significant operations (bulk inserts, complex queries)
    ✓ cross_process: gRPC client calls - downstream health
    ✓ long_running_ops: backfill, batch processing - duration, counts
    ✗ internal_method: use traces instead
    ✗ double_count: repository + service for same DB call
  test: actionable ∧ variable ∧ observed
    actionable: degradation → clear investigation path
    variable: actually changes over time (not always ~same)
    observed: dashboarded or alerted (unobserved = waste + complexity)
  tags: bounded_cardinality_only
    ✓ method_name | status_code | service_name | series_id (<100)
    ✗ event_id | user_id | request_id | correlation_id (unbounded)
  streaming: per_event_at_boundary for long_running_visibility
TRACING:
  catch_blocks: activity?.SetStatus(ActivityStatusCode.Error, ex.Message)
    rationale: exceptions_must_appear_in_traces
  spans: service_operations ¬ internal_methods
ALERTING [metrics_without_alerts = dashboard_archaeology]:
  P1_PAGE: service_down | data_loss | security
  P2_URGENT: error_spike | latency_degraded | init_fail
  P3_NOTIFY: warning_threshold | capacity_approaching
  golden_signals: latency(p95) + traffic(rate) + errors(%) + saturation
  health: liveness(restart) + readiness(routing) + startup(warmup)

## IDIOM_MAP [lang_specific]
Rust: Result<T,E> | pattern_match | ownership
Python: except | [x for x] | with | : Type
C#: except | async | LINQ(judicious)
TypeScript: types(¬over_eng) | async
PowerShell: Verb-Noun | pipeline | help

## ANTI [hard_stop]
✗ base_Entity{id,name} # base_class_syndrome
✗ TODO # placeholder_debt
✗ pattern_port # C→functional, Python→Java
✗ log_spam # INFO_prod
✗ coverage_chase # 100%_target
✗ abstract_premature # interface_soup
✗ revert_to_known_broken # problem_solving
✗ repeat_failed_attempt # insanity
✗ fix_without_understanding # thrashing

## TRIGGER_PATTERNS [code_gen]
IF generate_code THEN
  check(PRIORITY_0) →
  check(PROBLEM_SOLVING[if_debugging]) →
  check(TASK_MANAGEMENT[if_complex]) →
  check(BOUNDARY_HANDLING[context]) →
  check(IDIOM_MAP[lang]) →
  check(ANTI) →
  apply(LOG_RULES, DOC[type], TEST[if_needed], OBS_STACK[if_sensible])

## VERIFY [before_commit]
IF code_change THEN verify(compiles) BEFORE commit
METHODS:
  dotnet: {Project}/.devcontainer/compile.sh [--no-test]
  container: {Project}/.devcontainer/build.sh [--no-cache]
  fallback: ASK_USER("Can't verify - use devcontainer or skip?")
¬PATTERN: "straightforward" → commit_anyway # this_is_how_bugs_ship

## GRAFANA_DASHBOARD [json_provisioning]
COMMON_FAILURES: "No data" | duplicate_values | broken_viz | heatmap_fail
  root_cause: config_mistakes → predictable → eliminable
DASHBOARD_STRUCTURE:
  id: null # grafana_assigns_on_save
  uid: [a-z0-9-]{8,40} # stable_urls_across_envs
  schemaVersion: 41 # current_2025
  version: 0 # new_dashboards
  required_fields: {time, timezone, panels, editable, refresh}
PANEL_STRUCTURE:
  id: unique_int # duplicate → import_fail
  gridPos: {x:[0-23], y:int, w:[1-24], h:[3+]} # 24col_grid, ~30px/unit
  datasource: {type, uid} # ¬string_name, ¬${DS_NAME}
  required: {targets, fieldConfig.defaults, options}
STAT_PANEL [prevent_duplicates]:
  reduceOptions.values: false # true → shows_all_rows → duplicates
  reduceOptions.calcs: ["lastNotNull"] # "last" → may_return_null
  reduceOptions.fields: "" # empty → all_numeric, ¬"/.*/"
  query: sum(metric{...}) # aggregate → single_series
  instant: true ∧ range: false # ¬both_true → duplicate_data
  rationale: multiple_series | range_query | values:true → duplicates
TIMESERIES_PANEL:
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
TABLE_PANEL:
  transformations_required:
    multiple_queries → [{id: "merge"}]
    time_series_data → [{id: "seriesToRows"}]
    sparklines → [{id: "timeSeriesTable"}]
    column_control → [{id: "organize", options: {excludeByName, renameByName}}]
  fieldConfig.defaults.custom: {align, cellOptions: {type}, filterable}
  overrides: matcher{byName|byRegexp|byType} → properties{cellOptions, width}
HEATMAP_PANEL:
  calculate: true # grafana_buckets_data
  calculation: {xBuckets: {mode, value}, yBuckets: {mode, value}}
  prometheus_histogram: calculate: false ∧ legendFormat: "{{le}}" ∧ yBucketBound: "upper"
LOG_PANEL [loki]:
  query_type: log_query # ¬metric_query
  options: {showTime, wrapLogMessage, prettifyLogMessage, enableLogDetails, dedupStrategy, sortOrder}
  ✗ rate() | count_over_time() # these → timeseries_panel
PROMQL [prometheus_queries]:
  rate_interval: $__rate_interval # ¬[1m] ¬[5m] # auto_adjusts_to_scrape
    rationale: hardcoded < scrape_interval → "No data"
  container_filter: container!="POD", container!="" # always
  aggregation: sum by (label1, label2) # control_cardinality
  memory: container_memory_working_set_bytes # ¬usage_bytes (includes_cache)
  legendFormat: "{{pod}} - {{container}}" # explicit ¬ auto_generated
    rationale: post_aggregation {{__name__}} → empty
LOGQL [loki_queries]:
  log_panel: {job="app"} |= "error" | json # no_aggregation
  timeseries_panel: sum by (level) (count_over_time({job="app"} | json [5m]))
  unwrap: | unwrap field | __error__="" # always_filter_parse_errors
TRACEQL [tempo_queries]:
  queryType: "traceql"
  operators: span.attr | resource.attr | status | span:duration
  example: {resource.service.name="frontend" && status=error}
POSTGRES [timescaledb]:
  time_column: required # named "time", datetime | unix_epoch
  order: ORDER BY time ASC # mandatory
  filter: WHERE $__timeFilter(column)
  format: "time_series"
  bucket: time_bucket('5 minutes', time) AS time
UNITS:
  time: s | ms | µs | ns | m | h | d
  data: decbytes | bytes | bits | kbytes | mbytes
  rate: Bps | bps | KBps | MBps
  percent: percent(0-100) | percentunit(0-1)
  throughput: reqps | ops | rps | iops
ANTI [grafana_hard_stop]:
  ✗ ${DS_NAME} # templated_datasource → provisioning_fail
  ✗ rate(metric[1m]) # hardcoded_interval → "No data"
  ✗ omit fieldConfig.defaults # → broken_render
  ✗ reduceOptions.values: true # → duplicates
  ✗ stat_panel + multiple_series + no_aggregation # → duplicates
  ✗ instant: true ∧ range: true # → duplicate_data
  ✗ log_query → timeseries_panel # → no_results
  ✗ metric_query → log_panel # → no_results

## PROJECT_CONVENTIONS [ATLAS]
compose_files: compose.yaml ¬ docker-compose.yml
dockerfile_files: Containerfile ¬ Dockerfile
development: devcontainer ¬ local_install
rationale: runtime_agnostic(nerdctl|docker|podman) + compose_v2_standard + clean_host

## DEPLOYMENT [ATLAS] [HARD_STOP]
✗ NEVER edit /opt/ai-inference/compose.yaml directly
✓ ALWAYS use ansible for deployments
  playbook: ansible-playbook playbooks/deploy.yml --tags {service}
  inventory: deployment/inventory/hosts
rationale: compose.yaml = ansible-managed ∧ direct_edit = config_drift

## CONTAINER_BUILD [ATLAS]
IMAGE: {service-name}:latest # fred-collector ✓ fredcollector ✗
  verify: /opt/ai-inference/compose.yaml
BUILD: {Project}/.devcontainer/build.sh [--no-cache]
  from: /home/james/ATLAS # monorepo context required
DEPLOY: ansible --tags {service} # ¬nerdctl_manual

## DATABASE [ef_core]
SCHEMA: EF migrations only # ¬raw_sql_scripts
SEED_DATA: EF HasData() | app-level seeding on startup
DEPLOYMENT: app handles own migrations/seeding
  rationale: single_source_of_truth + version_controlled + testable
DEBUG_ONLY: raw psql queries for inspection
  ✓ SELECT to verify state
  ✗ INSERT/UPDATE/DELETE to fix state # fix_in_code_instead
ANTI:
  ✗ raw SQL scripts during deployment
  ✗ bypassing EF to seed/migrate
  ✗ manual database fixes # fix_root_cause_in_app

## BUILD [devcontainer]
compile: {Project}/.devcontainer/compile.sh [--no-test]
image: {Project}/.devcontainer/build.sh [--no-cache]
deploy: ansible-playbook playbooks/deploy.yml --tags {service}
filter_test: nerdctl compose exec -T {svc}-dev dotnet test --filter 'Name~{Test}'

## SERVICES [monorepo]
collectors: FredCollector, AlphaVantageCollector, NasdaqCollector, FinnhubCollector, OfrCollector
processing: ThresholdEngine
alerting: AlertService
calendar: CalendarService
metadata: SecMaster
mcp: FredCollectorMcp, ThresholdEngineMcp, FinnhubMcp, OfrCollectorMcp, SecMasterMcp
shared: Events/, deployment/, docs/

## DATA_FLOW
Collectors →gRPC:5001→ ThresholdEngine →metrics→ Prometheus → Alertmanager → AlertService → ntfy|email
Collectors →gRPC:8080→ SecMaster (registration, fire-and-forget)
ThresholdEngine →gRPC:8080→ SecMaster (resolution, context-based routing)
gRPC: internal_only (container-to-container)
HTTP: 8080 internal, 50xx host

## STACK
.NET9/C#13 | TimescaleDB | nerdctl/containerd | OTEL→Loki/Prom/Tempo→Grafana | Serilog

## EXECUTION_CONTEXT [HARD_STOP]
YOU_ARE_ON: mercury # this machine, the production server
✗ ssh mercury # you are already here
✗ ansible mercury # you are already here
✓ sudo nerdctl ... # direct command
✓ sudo systemctl ... # direct command
✓ ansible-playbook -i inventory/hosts.yml ... # deploys TO mercury, runs FROM mercury
rationale: ssh/ansible to localhost = unnecessary_hop + templating_issues + confusion
REMEMBER: run commands directly with sudo, not via ssh or ansible targeting mercury