---
name: readme-consistency
description: Audit README files across the monorepo for missing files, missing required sections, and content drift; optionally dispatch a fix subagent. Mirrors the review-fix-loop pattern from supervisor-mode.
argument-hint: optional flags — --non-interactive (auto-fix without confirm) | --audit-only (Phase 1 only) | --json (machine-parseable output)
---

# README_CONSISTENCY [SKILL v1]

Audit READMEs across the monorepo against a canonical 10-section template plus
staleness heuristics. Severity-bucket findings, dispatch a fix subagent under
user confirmation (interactive) or directly (non-interactive), then re-audit.

## ETHOS
audit_first ¬ assume_clean
template_canonical ¬ project_specific
fix_via_subagent ¬ direct_edit
re_audit ¬ trust_first_pass

## PHASES
1. ENUMERATE+AUDIT (read-only)
2. CONFIRM (interactive only; skipped under --non-interactive)
3. FIX (subagent dispatch)
4. RE_AUDIT + SUMMARY

## ENTRY
flags:
  --non-interactive | --auto    # skip confirm; for subagent dispatch
  --audit-only | --report-only  # Phase 1 only; emit gap report; exit
  --json                        # machine-parseable output (all modes)

mode_detection [HARD_STOP]:
  IF flag explicit → respect
  ELIF env READMECONSISTENCY_DEPTH >= 1 → force_non_interactive
  ELIF !isatty(stdin) → force_non_interactive
  ELSE interactive

recursion_guard:
  on_invoke: export READMECONSISTENCY_DEPTH=$((${READMECONSISTENCY_DEPTH:-0} + 1))
  IF depth > 2 → abort with error (prevents infinite recursion if subagent calls skill)

## PHASE_1 [audit, read-only]
1. cd into repo root (must contain CLAUDE.md or .git/)
2. enumerate target dirs:
   `bash .claude/skills/readme-consistency/scripts/enumerate-projects.sh`
3. for each target dir:
   `bash .claude/skills/readme-consistency/scripts/audit.sh <dir>`
   collect findings
4. emit aggregated gap report:
   IF --json → emit single JSON object aggregating all findings
   ELSE → emit human-readable text grouped by severity

## REPORT_FORMAT [text]
=== readme-consistency audit ===

CRITICAL ({N}):
  {path} — {message}

HIGH ({N}):
  {path} — {section} — {message}

MEDIUM ({N}):
  {path} — {signal} — {message}

Total: {N} findings.

## REPORT_FORMAT [json]
{
  "audited": [...],
  "summary": {"critical": N, "high": N, "medium": N, "low": N, "total": N}
}

## PHASE_2 [confirm, interactive only]
TRIGGER: mode == interactive ∧ findings > 0

PROCESS:
1. emit gap report (Phase 1 output)
2. ask user via AskUserQuestion:
   "Dispatch fix subagent for these findings?"
   options:
     - "Fix all (dispatch subagent)" → proceed to PHASE_3 with full findings list
     - "Select subset" → present multi-select picker over findings; proceed with selection
     - "Cancel" → exit cleanly (no Phase 3)
3. IF user_response == cancel → exit
4. IF user_response == select → re-prompt with multi-select per-finding
5. ELSE → proceed to PHASE_3 with full list

SKIP_CONDITIONS:
  - mode in {non_interactive, audit_only}
  - findings == 0

## PHASE_3 [fix dispatch]
TRIGGER: PHASE_2_approved ∨ mode == non_interactive

PROCESS:
1. construct subagent prompt:
   - scope: gap_report (full or selected subset)
   - canonical template: include literal contents of TEMPLATE.md inline
   - gold example: literal pointer "SecMaster/README.md (resolve relative to repo root)"
   - per-project commit discipline: "one commit per project README"
   - commit message format: "docs({project}): refresh README per readme-consistency audit"
   - HARD rules from CLAUDE.md SUPERVISOR_MODE: ¬push, ¬PR, selective `git add -- <paths>`, supervisor-owned files (STATE.md, docs/plans/**, scripts/agent-prompts/**) untouched
2. dispatch via Agent tool, subagent_type=general-purpose, run_in_background=false
   (foreground because we need the result for Phase 4; non-interactive callers can wrap in their own background dispatch)
3. on agent completion → record commit hashes for Phase 4 summary

PROMPT_TEMPLATE [pseudo]:
"You are fixing README gaps identified by the readme-consistency skill.

GAP REPORT:
{json or text}

CANONICAL 10-SECTION TEMPLATE:
{contents of .claude/skills/readme-consistency/TEMPLATE.md}

GOLD EXAMPLE for tone/depth:
{contents of SecMaster/README.md} or pointer

For each project in the gap report:
1. read existing README (if any)
2. apply minimum fix to address the listed findings (don't rewrite passing sections)
3. follow the 10-section template structure
4. commit with selective `git add -- <readme path>` and message
   `docs({project}): refresh README per readme-consistency audit`

DO NOT push. DO NOT open PR. Supervisor handles upstream.
DO NOT touch STATE.md, docs/plans/**, docs/llm/**, scripts/agent-prompts/**.

Report final commit hashes."

## PHASE_4 [re-audit]
TRIGGER: PHASE_3_completed

PROCESS:
1. re-run PHASE_1 (audit) with same scope
2. compare new findings vs. pre-fix:
   resolved = pre_findings - new_findings
   remaining = findings still present
   regressed = findings net-new (rare, but possible)
3. emit final summary:
   ```
   === readme-consistency final ===
   pre-fix findings: {N} ({severity breakdown})
   commits landed:    {list of hashes}
   resolved:          {N}
   remaining:         {N}
   regressed:         {N}
   exit_status:       {0 if clean ∨ 1 if remaining}
   ```
4. IF mode == non_interactive: exit with status code (0 = clean, 1 = remaining, 3 = phase_3 dispatch failed)
5. IF mode == interactive: print summary; if remaining > 0 ask user: "Dispatch follow-up?" (multi-select)

## ANTI [skill HARD_STOP]
✗ subagent_pushes # supervisor owns remote per CLAUDE.md
✗ subagent_opens_PR # same
✗ touch_supervisor_owned # STATE.md, docs/plans/**, docs/llm/**, scripts/agent-prompts/**
✗ skip_re_audit # Phase 4 is non-optional; closes the loop
✗ infinite_recursion # READMECONSISTENCY_DEPTH guard
✗ assume_isatty_under_subagent # always check explicit flag first

## COMPLETION_GATE
¬declare_done UNTIL ALL:
  1. PHASE_1 audit emitted gap report
  2. (interactive) user confirmed | (non-interactive) flag set
  3. PHASE_3 subagent reported completion with commit hashes
  4. PHASE_4 re-audit ran
  5. exit status reported (0|1|3 per mode)

## TRIGGER_PATTERNS [skill]
IF user_invokes(/readme-consistency [...flags]) THEN
  check(ENTRY mode_detection) →
  PHASE_1 audit →
  IF findings == 0 THEN exit_clean
  IF mode == audit_only THEN emit_report → exit
  PHASE_2 confirm IF interactive →
  PHASE_3 fix_dispatch →
  PHASE_4 re_audit →
  COMPLETION_GATE
