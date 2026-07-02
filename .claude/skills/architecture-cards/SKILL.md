---
name: architecture-cards
description: Maintain per-service "architecture cards" — a dense, READ-FIRST mental model that leads each service README (PURPOSE, DATA MODEL + INVARIANTS, PATHS with does/does-NOT/on-miss, RESOLUTION MODEL, DISTINCTIONS, CROSS-SERVICE, GOTCHAS). Activate when creating or editing a service, onboarding to a service's design, BEFORE reasoning about a service's architecture / API / data-model / resolution flow, or when auditing docs for drift. Audit a service for a complete card; if missing/incomplete, generate or update it from the service's code+docs, place it read-first, and wire it into CLAUDE.md. Mirrors the audit→dispatch-fix loop of readme-consistency.
argument-hint: optional — SERVICE (audit one service) | --audit-only (Phase 1 only) | --non-interactive (auto-fix without confirm) | --json (machine-parseable output)
---

# ARCHITECTURE_CARDS [SKILL v1]

A per-service **architecture card** is a dense, READ-FIRST block at the top of the
service README. It is the mental model an agent loads BEFORE touching the service —
the negative space (`does NOT`, `on-miss`, `⊥` independence, conflated `DISTINCTIONS`,
`GOTCHAS`) that a catalog of endpoints can never convey. It exists to stop the
recurring failure mode: an agent guesses a service's shape from method names,
conflates two distinct paths, or "fixes" a symptom by violating an invariant.

This skill audits each service for a complete card, and — when one is missing or
incomplete — generates/updates it from the service's code+docs, places it read-first,
and wires a pointer into CLAUDE.md. It mirrors the audit→confirm→dispatch-fix→re-audit
loop of the sibling `readme-consistency` skill.

## ETHOS
audit_first ¬ assume_present
card_models_negative_space ¬ catalog_of_endpoints   # does-NOT > does
read_first_in_README ¬ separate_file_agent_wont_open
dense ≤ ~1_page ¬ exhaustive (README §Reference owns the catalog)
fix_via_subagent ¬ direct_edit
re_audit ¬ trust_first_pass

## WHY_A_CARD [the load-bearing claim]
A README endpoint table answers "what can I call". A card answers "what will bite me":
  - two paths share a verb but are different code/consumers (resolve-entities vs ResolveBatch)
  - an invariant the schema does NOT enforce (identity ⊥ collection)
  - the on-miss contract (NotFound vs Warning-carrying-identity)
  - the governing maxim of a cascade ("fuzzy proposes, authoritative confirms")
  - the anti-pattern an agent will reach for ("bulk-preload to fix a miss")
These are the #1 anti-guess levers. The card front-loads them so the model never has
to reconstruct them from method signatures.

## PHASES
1. ENUMERATE+AUDIT (read-only)
2. CONFIRM (interactive only; skipped under --non-interactive)
3. FIX (subagent dispatch — generate/update card from code+docs)
4. RE_AUDIT + SUMMARY

## ENTRY
args:
  SERVICE                       # positional; audit just this service dir (else all)
flags:
  --non-interactive | --auto    # skip confirm; for subagent dispatch
  --audit-only | --report-only  # Phase 1 only; emit gap report; exit
  --json                        # machine-parseable output (all modes)

mode_detection [HARD_STOP]:
  IF flag explicit → respect
  ELIF env ARCHITECTURECARDS_DEPTH >= 1 → force_non_interactive
  ELIF !isatty(stdin) → force_non_interactive
  ELSE interactive

recursion_guard:
  on_invoke: export ARCHITECTURECARDS_DEPTH=$((${ARCHITECTURECARDS_DEPTH:-0} + 1))
  IF depth > 2 → abort with error (prevents infinite recursion if subagent calls skill)

## PHASE_1 [audit, read-only]
1. cd into repo root (must contain CLAUDE.md or .git/)
2. enumerate target services:
   `bash .claude/skills/architecture-cards/scripts/enumerate-services.sh`
   (reads CLAUDE.md `## SERVICES` so the audit set tracks the canonical list, not a glob)
3. for each service (or the single positional SERVICE):
   `bash .claude/skills/architecture-cards/scripts/audit.sh <dir>`
   collect findings
4. emit aggregated gap report:
   IF --json → emit single JSON object aggregating all findings
   ELSE → emit human-readable text grouped by severity

## REPORT_FORMAT [text]
=== architecture-cards audit ===

CRITICAL ({N}):
  {service} — {message}            # no card at all

HIGH ({N}):
  {service} — {block} — {message}  # card present but missing a required block, or not read-first

MEDIUM ({N}):
  {service} — {signal} — {message} # weak card: catalog-without-model, missing does-NOT / on-miss

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
   "Dispatch fix subagent to generate/update these architecture cards?"
   options:
     - "Fix all (dispatch subagent)" → PHASE_3 with full findings list
     - "Select subset" → multi-select picker over findings; proceed with selection
     - "Cancel" → exit cleanly (no Phase 3)
3. IF user_response == cancel → exit
4. IF user_response == select → re-prompt with multi-select per-finding
5. ELSE → proceed to PHASE_3 with full list

SKIP_CONDITIONS:
  - mode in {non_interactive, audit_only}
  - findings == 0

## PHASE_3 [fix dispatch — generate/update the card]
TRIGGER: PHASE_2_approved ∨ mode == non_interactive

## GENERATE_VALIDATE_ITERATE [card quality loop — run inside Phase 3]
PROVEN LOOP (replaces single-pass generation):
  1. GENERATE: large model (Opus) generates card draft from code+docs per CARD_TEMPLATE.md.
     Every block traces to a file:line — no memory-derived claims.
  2. VALIDATE: adversarial model (Sonnet/Haiku) validates EACH claim independently:
     a. reconstruct service behavior from card ONLY (simulate fresh-agent reading)
     b. score load-bearing fidelity: for each INV/PATH/DISTINCTION/GOTCHAS item — does the
        code support the claim? cite file:line or flag as "unsupported"
     c. identify: omitted does-NOT / missed miss-contract / wrong terminal semantics / rambling
        prose that should be one terse line (plain words + "->"; NOT unicode operators —
        measured 2026-07-02: ¬/∧/⊥ cost same-or-more tokens than "not"/"and" and break literal grep)
     d. emit: {supported: N, unsupported: N, gaps: [...], suggestions: [...]}
  3. FIX: generator incorporates validator findings.
  4. ITERATE: repeat VALIDATE → FIX until:
     a. unsupported_claims == 0
     b. all load-bearing anti-patterns covered in GOTCHAS
     c. card ≤ ~1 page (density gate)
  rationale: single-pass generation drifts from code; adversarial validate catches
    unsupported claims, missing negative-space, and rambling prose that should be terse lines.
  model_economy: validator can be a smaller/cheaper model — it reads the card + one code
    file per claim, not the whole codebase. Generator does the expensive full-read pass.

PROCESS:
1. construct subagent prompt:
   - scope: gap_report (full or selected subset)
   - card template: include literal contents of CARD_TEMPLATE.md inline
   - worked exemplar: include literal contents of EXEMPLAR_SECMASTER.md inline
     (the SecMaster tight card is the gold standard for density + negative-space)
   - source of truth: generator MUST derive each block from the service's CODE
     (Services/*.cs, Endpoints/*.cs, Data/Entities/*.cs, *.proto, openapi) and existing
     README, NOT from memory. Negative-space (`¬do`, `miss`, `DISTINCTIONS`)
     comes from reading the resolution/cascade code paths.
   - quality loop: run GENERATE→VALIDATE→ITERATE per above before committing
   - placement: create `<Service>/AGENT_README.md` (the tight card); add a 1-line pointer
     to `README.md` immediately after the H1. Demote any large catalog tables to
     `README.md §Reference`. The card's `SEE:` line points there.
   - per-service commit discipline: "one commit per service (AGENT_README.md + README.md)"
   - commit message format: "docs({service}): add/refresh AGENT_README.md architecture card"
   - HARD rules: ¬push, ¬PR, selective `git add -- <paths>`,
     supervisor-owned files (STATE.md, .claude/skills/supervisor-mode/**) untouched
2. dispatch via Agent tool, subagent_type=general-purpose, run_in_background=false
   (foreground because we need the result for Phase 4)
3. on agent completion → record commit hashes for Phase 4 summary

PROMPT_TEMPLATE [pseudo]:
"You are generating/refreshing per-service ARCHITECTURE cards flagged by the
architecture-cards skill. A card is a dense, standalone mental model in AGENT_README.md —
the negative space (¬do / miss / invariants / DISTINCTIONS / GOTCHAS) that an endpoint catalog can't convey.

QUALITY LOOP (REQUIRED — do not skip):
  GENERATE (Opus): read code → draft card per CARD_TEMPLATE.md (every claim traces to file:line).
  VALIDATE (Sonnet/Haiku): reconstruct service behavior from card ONLY →
    score each claim vs code (cite file:line or flag unsupported) →
    emit {supported: N, unsupported: N, gaps: [...], suggestions: [...]}.
  FIX: incorporate validator findings. ITERATE until unsupported_claims==0 AND card ≤ ~1 page.

GAP REPORT:
{json or text}

CARD TEMPLATE:
{contents of .claude/skills/architecture-cards/CARD_TEMPLATE.md}

WORKED EXEMPLAR (gold standard for density + negative-space):
{contents of .claude/skills/architecture-cards/EXEMPLAR_SECMASTER.md}

For each service in the gap report:
1. READ the service's code (Services/*.cs, Endpoints/*.cs, Data/Entities/*.cs, *.proto)
   and existing README. Derive every block FROM THE CODE — especially the negative space:
   the `¬do` per path, the `miss` contract, the cascade maxim, the conflated
   DISTINCTIONS, the GOTCHAS. Do NOT invent these from memory.
2. Run GENERATE→VALIDATE→ITERATE loop until quality gate clears.
3. Write the card to `<Service>/AGENT_README.md`. Keep it ≤ ~55 non-blank lines.
4. Add a 1-line pointer to `README.md` immediately after the H1/elevator:
   `> 🤖 **Agents:** read **[AGENT_README.md](AGENT_README.md)** first — the dense architecture card.`
   Demote any large catalog tables to `README.md §Reference`. Do NOT delete the catalog.
5. Commit with selective `git add -- <service>/AGENT_README.md <service>/README.md` and
   message `docs({service}): add/refresh AGENT_README.md architecture card`.

DO NOT push. DO NOT open PR. Supervisor handles upstream.
DO NOT touch STATE.md or .claude/skills/supervisor-mode/**.

Report final commit hashes + validator fidelity scores."

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
   === architecture-cards final ===
   pre-fix findings: {N} ({severity breakdown})
   commits landed:    {list of hashes}
   resolved:          {N}
   remaining:         {N}
   regressed:         {N}
   exit_status:       {0 if clean ∨ 1 if remaining}
   ```
4. IF mode == non_interactive: exit with status code (0 = clean, 1 = remaining, 3 = phase_3 dispatch failed)
5. IF mode == interactive: print summary; if remaining > 0 ask user: "Dispatch follow-up?" (multi-select)

## ROLLOUT [placement + wiring — see ROLLOUT.md]
This skill is additive. The first time a card lands for a service it changes the README
SHAPE (read-first card → demoted Reference catalog) and wires CLAUDE.md. Those structural
edits are governed by ROLLOUT.md (placement rule, CLAUDE.md SERVICES wiring template,
prioritized service order). Apply them in the rollout phase, one service at a time, each
behind its own PR. SecMaster ships as the worked exemplar (EXEMPLAR_SECMASTER.md).

## ANTI [skill HARD_STOP]
✗ separate_card_file # card must LEAD the README; an agent opens the README, not card.md
✗ catalog_masquerading_as_card # endpoint table ≠ card; the card is the negative space
✗ omit_does_NOT # the `does NOT:` line per path is the #1 anti-guess lever — never skip
✗ omit_on_miss # the miss contract (NotFound vs Warning) is load-bearing
✗ card_over_1_page # density is the point; the catalog lives in README §Reference
✗ subagent_pushes # supervisor owns remote per CLAUDE.md
✗ subagent_opens_PR # same
✗ touch_supervisor_owned # STATE.md, .claude/skills/supervisor-mode/**
✗ skip_re_audit # Phase 4 is non-optional; closes the loop
✗ infinite_recursion # ARCHITECTURECARDS_DEPTH guard
✗ derive_card_from_memory # every block traces to code/docs, not the model's prior

## COMPLETION_GATE
¬declare_done UNTIL ALL:
  1. PHASE_1 audit emitted gap report
  2. (interactive) user confirmed | (non-interactive) flag set
  3. PHASE_3 subagent reported completion with commit hashes
  4. PHASE_4 re-audit ran
  5. exit status reported (0|1|3 per mode)

## TRIGGER_PATTERNS [skill]
IF (about_to_reason_about_a_service ∨ creating/editing_a_service ∨ user_invokes(/architecture-cards [...])) THEN
  check(ENTRY mode_detection) →
  PHASE_1 audit →
  IF findings == 0 THEN exit_clean (card is current; READ it before reasoning)
  IF mode == audit_only THEN emit_report → exit
  PHASE_2 confirm IF interactive →
  PHASE_3 fix_dispatch →
  PHASE_4 re_audit →
  COMPLETION_GATE
