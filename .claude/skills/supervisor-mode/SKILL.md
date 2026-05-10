---
name: supervisor-mode
description: Supervisor mode workflow — orchestrate multi-agent execution against a structured plan with STATE.md tracking, race-aware dispatch, and trust-but-verify completion
argument-hint: optional plan path; supervisor reads STATE.md + CLAUDE.md by default
---

# SUPERVISOR_MODE [SKILL v2]

Operationalises CLAUDE.md `SUPERVISOR_MODE` into a concrete loop with race
guards, mid-stop recovery, and review-fix discipline learned from running
~100+ agent dispatches across multi-epic refactors.

## ETHOS
dispatch > direct_edit
verify > trust
commit_per_layer > commit_at_end
selective_pathspec > git_add_-A

## ROLE_BOUNDARY
TOUCHES: STATE.md | plan_files | CLAUDE.md | scripts/agent-prompts/
¬TOUCHES: code (impl|review|validation → subagent)
OWNS_REMOTE: push | PR_create | PR_update # never_subagent

## ENTRY
prereqs: structured_plan_input + CLAUDE.md + STATE.md (create_if_missing)
plan_input: any_form (mvp_plan.md | epic_doc | task_spec) ¬ rigid_format
WAKEUP_STEP_0: poll(atlas-claude-reply) BEFORE routine_work
IF ¬plan THEN brainstorming → writing-plans first

## STATE.md_SHAPE
ACTIVE: epic | phase | branch | story_in_flight | plan_path
STORY_QUEUE: [✓done | →active | ⧗blocked | ◯pending] # compact_status_legend
EXIT_STATUS: branch_state | test_counts | schema_changes (on epic_done)
DEPLOYMENT_TODOS: env_vars | post_deploy_smoke | infra_debt
OPEN_ASKS: user_decisions | ratifications | parallel_branch_strategy
CONTEXT: canonical_plans | recon_cache | open_questions

## TURN_LOOP
1. read(STATE.md, CLAUDE.md) # in_context_already_after_session_start
2. poll(atlas-claude-reply) IF wakeup | pre_status
3. select(next_dispatch) | deps_met ∧ disjoint_files ∧ priority_top
4. dispatch(subagent) | template + scope + budget_guard
5. on_completion → verify(git_log + git_status + sanity_read)
6. update(STATE.md) # mark_complete | flag_followups
7. milestone | blocker → ntfy_publish (atlas-claude-ask)
8. end_turn (background) | continue (foreground)

## SUBAGENT_DISPATCH
PROMPT_SHAPE [≤400w]:
  scope: 1-3_deliverables_max
  commit_discipline: per_layer_immediately ¬ end_of_story
  add_pathspec: explicit `git add -- <paths>` ¬ -A | -u | .
  budget_guard: at ~70%_burn → commit_what_done + stash + report
  reporting_format: {commit_hashes, files, test_counts, deviations, blocked}
  hard_rules: ¬push ¬PR ¬touch(supervisor_owned)
  supervisor_owned: STATE.md | docs/plans/** | docs/llm/** | scripts/agent-prompts/**

CHOICE:
  general-purpose → bulk_impl
  Explore → read_only_recon
  pr-review-toolkit:* → review_passes
  code-simplifier → post-impl_polish
  Plan → architecture_decisions

CADENCE [parallelism rules]:
  same_branch + concurrent_agents → SEQUENCE # race risk
  disjoint_files + same_branch → parallel_OK
  different_branches | worktrees → fully_parallel
  ALWAYS_TELL_EACH: which_files_belong_to_other_in-flight_agents

## RACE_GUARDS [HARD_STOP]
GIT_ADD_RULE:
  ✓ git add -- <explicit_paths>
  ✗ git add -A | git add -u | git add .
  rationale: agent_A_stages + agent_B_commits_without_paths → wrong_message_sweeps_A's_files

SUPERVISOR_STAGING:
  IF agents_in_flight ∧ supervisor_must_commit:
    selective git add only supervisor_files
    verify staged_set_before_commit
    rationale: prevent_index_race

## MID_STOP_RECOVERY
DETECT: agent_msg_reads_mid_thought | tool_count_low | "now I'll..." | "Acknowledged. Now..."
PROCESS:
  1. git status --short → identify_WIP
  2. git log --oneline <last_known_commit>..HEAD → see_what_committed
  3. read(WIP_files | 1-2_key) → sanity_skim
  4. dispatch_fresh_continuation | cite_WIP_paths_explicitly
  5. ¬SendMessage (per CLAUDE.md FAILURE rule)
RATIONALE: agents_budget_~50-130_tool_uses | mid_stop_is_norm ¬ exception
PATTERN: prior_agent's_WIP_files → next_agent's_starting_state

## VERIFY_AFTER_AGENT [TRUST_BUT_VERIFY]
GATE: ¬mark_complete UNTIL:
  ✓ git log --oneline → expected_commits_present
  ✓ git status --short → working_tree_in_expected_state
  ✓ sanity_read(1-2_key_files) → match_agent_report
  ✗ accept_agent_summary_alone # agent_reports_intent ¬ outcome

## STORY_SPLIT_HEURISTIC
SPLIT_IF any:
  > 20_files_touched
  > 2_distinct_concerns (e.g. schema + reader_cutover)
  DB_schema + readers + writers (cross-cutting)
  cross_project_cascade (refactor_spans_>1_service)
  > 3_layers (entity + service + endpoint + tests + migration)
PATTERN [staged_commits]:
  additive_first → cutover → drop # build_green_per_phase
  layer_A → commit → verify → layer_B → commit → ...
  rationale: budget_caps_kill_unstaged_work | partial_progress_preserves

## REVIEW_FIX_LOOP
TRIGGER: PR_ready | major_milestone | pre_merge
PROCESS:
  1. dispatch(multi_agent_review) | parallel
     suite: pr-review-toolkit:review-pr | observability-review | readme-consistency | code-review
  2. aggregate_findings | severity_bucket {critical, important, suggestion}
  3. dispatch_fix_per_severity | commit-as-you-go | selective_pathspec
  4. push_only_after critical+important_addressed
  5. re-run_review → verify_no_regression + catch_new_issues
  6. iterate_until convergent
RATIONALE: 1st_pass_finds_most | 2nd_pass_catches_regressions_+_new_finds

## NTFY_CADENCE
PUBLISH (atlas-claude-ask):
  ✓ milestones (epic_done | PR_opened | review_complete)
  ✓ blocking_questions (user_decisions_required)
  ✓ async_asks (ratifications | direction_choices)
  ✗ per_story_completion # noisy → STATE.md_already_tracks
  ✗ internal_progress # supervisor_overhead

POLL (atlas-claude-reply):
  on: wakeup (WAKEUP_STEP_0)
  on: pre_status_report
  on: routine_checkpoint
  ¬PATTERN: continuous_background_polling # wasteful

## SUPERVISOR_GIT_OWNERSHIP
PUSH:
  ✓ supervisor (after CLAUDE.md GIT_PUSH gate satisfied)
  ✗ subagent (reject in every dispatch prompt)
PR_OPS:
  ✓ supervisor (gh pr create | gh pr update | gh pr review)
  ✗ subagent
RATIONALE: visible_to_others = needs_supervisor_judgment

## CONTEXT_HYGIENE
LONG_SUBAGENT_OUTPUT → /tmp/<task-id>/<file> ¬ supervisor_turn
AGENT_TRANSCRIPT_FILES: ¬read # overflow_risk
REUSE: scripts/agent-prompts/ templates ¬ rewrite_each_dispatch
STATE.md: read_first | write_last per_turn # supervisor_memory

## TEMPLATE_LIBRARY
LOCATION: scripts/agent-prompts/ # reusable, ≤400w each
SHAPES:
  story-implementation.md → bulk impl with TDD
  audit-recon.md → read-only blast-radius scan
  review-fix.md → consolidated severity-tiered fix
  destructive-migration.md → staged add-cutover-drop pattern
  test-gap-fill.md → coverage hardening
  readme-fix.md → consolidated README gap fix per readme-consistency report

## CLOUD_ORACLE_ROUTING
gold_label | architecture: opus-4-7
cross_check: opus-4-6
bulk_label | impl: sonnet-4-6
smoke | triage: haiku-4-5
cap: 500K_tpm_client_side

## ANTI [supervisor HARD_STOP]
✗ git_add_-A_with_concurrent_agents # race
✗ accept_agent_summary_without_verify # reality_drift
✗ SendMessage_to_failed_agent # CLAUDE.md FAILURE rule
✗ subagent_pushes_or_PR_creation # remote_is_supervisor_only
✗ per-story_NTFY # noisy
✗ unverified_rolling_commits # checkpoint_per_phase
✗ touch_code_directly # except STATE.md | plan | CLAUDE.md
✗ continuous_background_NTFY_polling # wasteful

## TRIGGER_PATTERNS [supervisor]
IF multi_agent_work THEN
  check(ROLE_BOUNDARY) →
  check(ENTRY) → STATE.md_init_if_missing →
  loop(TURN_LOOP) →
  apply(RACE_GUARDS, MID_STOP_RECOVERY, VERIFY_AFTER_AGENT) →
  apply(STORY_SPLIT_HEURISTIC for big_stories) →
  on_phase_complete → REVIEW_FIX_LOOP →
  on_milestone → NTFY_publish →
  on_done → COMPLETION_GATE

## COMPLETION_GATE [supervisor]
¬declare_done UNTIL ALL:
  1. all_critical_review_findings_addressed
  2. all_important_addressed | user_signoff_to_defer
  3. push_to_remote # after CLAUDE.md GIT_PUSH gate
  4. PR_review_passes_clean | re-run_review
  5. STATE.md_exit_status_recorded
  6. milestone_NTFY_published
