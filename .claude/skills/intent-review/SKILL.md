---
name: intent-review
description: Review a diff/PR for design-intent conformance against the touched services' DECISIONS blocks (AGENT_README.md D-entries). Activate on every code review of an ATLAS service alongside review-pr/observability-review — checks what those skills do NOT: guard touched without a named supersession, new exception path without a D-entry, guard tests missing or tautological, INTENT(D-n) comments orphaned from the code they annotate. Lives project-side because plugin review skills are cache-owned/uneditable; invoked by the supervisor-mode review dispatch and the deploy skill.
argument-hint: PR number | branch | diff range (default: current branch vs main)
---

# INTENT_REVIEW [SKILL v1]

Code compiles but violates design intent: the mechanism survives implementation, the
precondition dies (gemini-resolver drain #823–825 — call-on-miss kept, rare AND hard AND
dividend-paying lost -> trash firehose to a paid frontier model). This skill closes leak
point 2 (review gates never check spec conformance): given a diff, load the touched
services' DECISIONS blocks and judge whether the diff honors the design decisions
governing the touched code.

## ETHOS
judgment gate, not mechanical lint           # Approach C (lint+block) REJECTED — file:line stales in ordinary refactors
findings, not blocks                         # advisory; REVIEW_FIX_LOOP aggregates + dispatches fixes
D-entries = the contract                     # card DECISIONS block is what the diff is judged against
atomic set change-all-or-none                # D-entry + INTENT comment + guard + guard-test move together
conflict -> STOP, never route-around, never obey-stale

## PHASES
1. SCOPE: resolve the diff (PR number -> `gh pr diff` | branch -> `git diff main...`) ->
   map touched paths -> services (`<Service>/src/**` -> Service).
2. LOAD: for each touched service, read `<Service>/AGENT_README.md` DECISIONS block.
   `DECISIONS: none` or no card -> only CHECK_2 applies to that service.
3. CHECK: run CHECK_1..CHECK_4 below against the diff + loaded D-entries.
4. REPORT: emit findings per REPORT_FORMAT; no findings -> explicit "intent-clean".

## CHECKS
CHECK_1 guard touched without supersession [critical]:
  diff modifies/deletes a GUARD-cited file:line region, code adjacent to an
  `// INTENT(D-n):` comment, or a cited guard test — WITHOUT a same-diff rewrite of the
  D-entry AND a named "supersedes D-n" in the driving brief/PR description.
  ALSO fires when the brief/PR intent contradicts a D-entry without naming supersession ->
  verdict is STOP-and-report; never route-around, never obey-stale (entry may be outdated
  OR brief wrong — human/supervisor decides).
CHECK_2 new exception path without D-entry [critical if scarce-resource $/GPU/quota; important otherwise]:
  diff introduces a new exception path — new external client call, paid-API/frontier-model
  call, budget/cap bypass, raw-DB write, privileged op — with NO new D-entry (+ atomic set)
  in the same diff. Exception paths exist for a SPECIFIC EARNED case; ungated = the
  gemini-resolver disease.
CHECK_3 guard test missing or tautological [important]:
  each new/changed guard in the diff has a test meeting GUARD_TEST_CONTRACT below.
  Missing -> important. Tautological (asserts setup, mocks the guard itself, 1=1 coverage,
  would stay GREEN if the guard were deleted) -> important — performative coverage is what
  drained the Gemini budget.
CHECK_4 intent comment orphaned [suggestion]:
  code annotated `// INTENT(D-n):` moved/refactored but the comment didn't move with it,
  references a D-n that no longer exists, or the D-entry's GUARD @ file:line now points at
  the old location. Stale intent > no intent — it actively misleads.

## GUARD_TEST_CONTRACT [canonical home — spec doc retires; CLAUDE.md points here]
Construct the violation; assert refusal AT the boundary, through the real flow; mock ONLY
the external client; test goes RED if the guard is deleted.
Examples:
  junk slugs pushed through extraction ingress -> Gemini client mock records ZERO calls
  cheap-path success -> frontier client NOT invoked
  past budget cap -> refusal, never silent-pass
Unit tests default; integration only where sanitization meets real serialization/protocol.
No coverage chasing — a guard test exists because a D-entry exists, and for no other reason.

## SEVERITY_MAP [REVIEW_FIX_LOOP-compatible]
critical:   CHECK_1 (incl. conflict-without-supersession -> STOP) | CHECK_2 at a $/GPU/quota boundary
important:  CHECK_2 (non-scarce exception path) | CHECK_3 (missing or tautological guard test)
suggestion: CHECK_4 (orphaned/stale INTENT comment) | scope-smell (>~6 D-entries, dilution)

## REPORT_FORMAT
=== intent-review: {diff_ref} ===
services: {list} (D-entries loaded: {N}; DECISIONS:none: {list})
CRITICAL ({N}):
  {service} D-{n} — {check} — {finding + file:line}
IMPORTANT ({N}):
  {service} — {check} — {finding + file:line}
SUGGESTION ({N}):
  {service} — {check} — {finding}
verdict: intent-clean | findings-emitted | STOP(conflict-without-supersession: D-{n})

## ANTI [HARD_STOP]
✗ block mechanically # findings feed REVIEW_FIX_LOOP judgment; lint+block rejected (spec §OUT_OF_SCOPE)
✗ route around conflict | obey stale entry # STOP + report; a human/supervisor arbitrates
✗ accept paraphrased D-entries # judge against the card's VERBATIM entries, not the brief's summary
✗ demand D-entries for ordinary mechanism # scope discipline; DECISIONS: none is legal
✗ green guard test after guard delete # the RED-on-removal property IS the test's reason to exist
✗ skip because card missing # no card -> CHECK_2 still applies; flag missing card as suggestion

## COMPLETION_GATE
never declare done UNTIL ALL:
  1. every touched service's DECISIONS block loaded (or its absence noted)
  2. all four CHECKs run against the full diff
  3. findings emitted with severities per SEVERITY_MAP (or explicit intent-clean)
  4. any CHECK_1 conflict surfaced as STOP verdict, never silently downgraded
