# CLAUDE.md [CODE.GEN.RULES v1.0]

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
OTEL → Loki(logs) + Prom(metrics) + Grafana(viz)
metric_purpose: {perf_tune, bottleneck_diagnosis}
config: lightweight(tunables+ext_connections) ¬ framework
when: sensible_value ¬ dogmatic

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

## ETHOS
pragmatic > dogmatic
working > perfect
maintain > clever

## PROJECT_CONVENTIONS [ATLAS]
compose_files: compose.yaml ¬ docker-compose.yml
dockerfile_files: Containerfile ¬ Dockerfile
development: devcontainer ¬ local_install
rationale: runtime_agnostic(nerdctl|docker|podman) + compose_v2_standard + clean_host
