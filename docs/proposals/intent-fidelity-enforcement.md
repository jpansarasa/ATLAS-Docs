# Intent-Fidelity Enforcement — Spec + Execution Plan

**Status:** APPROVED — this doc drives implementation PRs.
**Origin:** user ↔ supervisor design conversation; distills CLAUDE.md `INTENT_FIDELITY` + `GIGO` into enforcement mechanism.
**Curation:** per PHASE_TAGS this plan is retired from `main` at epic completion — and that retirement is itself governed by the machinery this doc specifies (plan-retirement-guard, §HOOKS): every load-bearing decision below must land as a D-entry before the doc is removed. This doc IS the design-decision record for the epic; losing a WHY from it is the exact failure the epic exists to fix.

---

## PROBLEM

Agent-implemented code compiles but violates design intent. The WHY lives in plan docs; plans are retired from `main` (PHASE_TAGS); nothing enforces intent downstream of the plan. Code inherits the WHAT and drifts into violating the design's ETHIC.

Worked incidents:
- **gemini-resolver drain (#823–825):** INTENT = "cheap lookups fan out in parallel; the frontier call is the RARE exception, earned only when all-cheap-failed on a genuinely hard entity, because THERE it pays dividends." The mechanism (call-on-miss) survived implementation; the precondition (rare ∧ hard ∧ dividend-paying) was lost → trash firehose to a paid frontier model, invisible until it hit a bill.
- **#818 FRED trash-routing:** news-NER junk routed into FRED search — same disease, gated at the destination instead of cleaned at the source.

Compounding: code is defensive **inbound** (validates what a fn RECEIVES) but sends junk **outbound** (GIGO is symmetric — a call is a boundary too), with **zero RED-on-violation test coverage of what calls SEND**. No test goes red when a guard is deleted.

## LEAK_POINTS [4 confirmed]

1. **Dispatch briefs compress out WHY** — supervisor briefs carry acceptance criteria and mechanism, not the design decision + justification; the implementing agent never sees the precondition.
2. **Review gates never check spec conformance** — existing review skills check style/observability/tests, not "does this diff honor the design decisions governing the touched code."
3. **Plan retirement orphans decisions** — PHASE_TAGS removes plan docs from `main`; the decisions inside them stop existing anywhere an agent reads.
4. **AGENT_README cards have no design-decision block** — cards carry invariants and gotchas but no numbered, guarded, tested decision record; there is nowhere durable to migrate a plan's WHY to.

---

## D_ENTRY_SPEC [§1 — the artifact]

Per-service numbered decisions (`D-1`, `D-2`, …) in a **DECISIONS block** in `<Service>/AGENT_README.md`.

**Line format:**
```
D-n <slug>: INTENT <why> / PRECOND <condition> / GUARD <class.method> @ file:line / TEST <TestClass.TestName>
```

**ATOMIC SET** (change-all-or-none — same discipline as the `:sig:` infix string contract):
1. D-entry in the card
2. `// INTENT(D-n):` comment at the guard site
3. guard code
4. guard test

**Supersession:** rewrite the entry **in the same PR as the code change**. No tombstones — `main` = current-state, git log = archive. Dispatch briefs must name **"supersedes D-n"** explicitly.

**CONFLICT RULE:** if a brief contradicts a D-entry without a named supersession → the agent **STOPs and reports**. Never route-around; never obey-stale. (The D-entry may be outdated, or the brief may be wrong — a human/supervisor decides; the implementing agent does not.)

**Scope discipline** (¬everything is a decision):
- ✓ exception paths (frontier last-resort, raw-DB write, privileged op)
- ✓ scarce-resource boundaries ($/GPU/quota)
- ✓ invariants with non-obvious preconditions
- ✗ ordinary mechanism — a service may declare `DECISIONS: none — no exception paths`
- \>~6 entries = smell (scope creep dilutes the signal); card stays ≤ ~1 page.

## HOOKS [§2 — mechanical delivery + gates]

All registered in `.claude/settings.json`; patterns follow existing hooks in `.claude/hooks/`.

**`design-intent-dispatch-guard.sh`** — PreToolUse, matcher `Agent`, **BLOCK**.
Implementation-shaped dispatch prompts (heuristic indicators: `commit` / `git add` / `compile.sh` / implement / fix / refactor) must contain a **"DESIGN INTENT" stanza**. `DESIGN INTENT: none — mechanical, no D-entries touched` passes. Content-agnostic — the hook checks stanza presence, not stanza correctness — so mid-epic pivots pass without hook edits.

**`service-decisions-context.sh`** — PreToolUse, matcher `Edit|Write`, **ADVISE** (testing-context.sh pattern).
Editing `<Service>/src/**` where the service card has a DECISIONS block → inject that block as `additionalContext`. Rationale: mechanical delivery of the decisions to the editing agent — no reliance on the agent having read the card.

**`plan-retirement-guard.sh`** — PreToolUse, matcher `Bash` on `git rm` of plan-doc paths (`docs/proposals/**`, `*PLAN*.md`, `*-design.md`), **ASK** (ansible-gate-guard pattern).
Injects a migration checklist — every load-bearing decision → D-entry landed? guard tests cited? — and requires confirmation. **Explicitly NOT a block:** a script cannot verify migration *semantics*; claiming it could would be a dishonest gate. The ASK puts the checklist in front of the agent at exactly the moment decisions are about to be orphaned (leak point 3).

**`testing-context.sh`** (existing, **enrich**) — add injected line:
```
outbound_boundary: test what you SEND — mock the external client, drive junk from upstream, assert never-invoked ∨ sanitized-payload; RED-on-guard-removal
```

## SKILLS [§3]

**supervisor-mode** (`SKILL.md` + `templates/story-implementation.md`):
- PROMPT_SHAPE gains a **mandatory impl-brief stanza**:
  - `decisions:` — the in-scope D-entries, **verbatim**
  - `supersedes:` — `D-n | none`
  - `guard_tests:` — a deliverable per new/changed guard
  - conflict rule as in D_ENTRY_SPEC (brief-contradicts-entry-without-supersession → STOP + report)
- PLAN_GROUNDING gains: extract plan-section design decisions into the brief **VERBATIM, not paraphrased**. Rationale: paraphrase is the compression step where WHY dies (leak point 1).

**architecture-cards:**
- `CARD_TEMPLATE.md` gains the DECISIONS block spec (D_ENTRY_SPEC format).
- `scripts/audit.sh` gains checks:
  - DECISIONS block present — **MEDIUM** if absent; explicit `DECISIONS: none` accepted
  - each entry has `GUARD @ file:line` + `TEST` citation — **HIGH** if missing
  - citations resolve: file exists + test name greps in the test project — **HIGH** if dead

**NEW project skill `.claude/skills/intent-review/SKILL.md`:**
Given a diff/PR → load the touched services' DECISIONS blocks → findings on:
1. guard touched without a named supersession + same-diff D-entry rewrite
2. new exception path (new external client call, budget bypass, privileged op) with **no** D-entry
3. guard tests missing or tautological — must assert **at the boundary through the real flow** (¬1=1 coverage)
4. `// INTENT(D-n):` comments not moved/updated with the code they annotate

Lives **project-side** because plugin review skills are cache-owned/uneditable. Invoked by the supervisor-mode review dispatch + the deploy skill. (Closes leak point 2.)

## CLAUDE_MD [§4]

- **Project `CLAUDE.md` — INTENT_FIDELITY** gains an ~8-line **MECHANICS** subsection: D-entry format pointer, the 4-artifact atomic set, supersession/halt rule, guard-test requirement.
- **Project `CLAUDE.md` — SERVICE_ARCHITECTURE** gains one line: cards carry DECISIONS; conflict-rule pointer.
- **User-level `~/.claude/CLAUDE.md` — TEST** gains ONE generic line (generic because user-level is cross-project):
  ```
  IF outbound_boundary_guard → test what_you_SEND (mock client, junk upstream, assert ¬invoked ∨ sanitized) # GIGO symmetric
  ```

## GUARD_TEST_CONTRACT [§5]

Construct the violation; assert refusal **AT the boundary, through the real flow**; mock ONLY the external client; test goes **RED if the guard is deleted**.

Examples:
- junk slugs pushed through extraction ingress → Gemini client mock records **zero** calls
- cheap-path success → frontier client **not invoked**
- past budget cap → **refusal**, not silent pass

Unit tests are the default; integration only where sanitization meets real serialization/protocol. **No coverage chasing** — a guard test exists because a D-entry exists, and for no other reason.

## BACKFILL [§6]

Three services — SentinelCollector, SecMaster, ThresholdEngine (the GIGO/Gemini lineage) — **one PR each**:
- Mine: retired plans (via phase tags), incidents #818 / #823–825, existing guards (`CandidateSurfaceFilter`, `IdentifierConfirmationService`, projector caps / |v|≤1.5)
- Produce: D-entries + `// INTENT(D-n):` comments
- Verify: cited guard tests exist and go RED on guard removal; write the missing ones.

Other 8 services: **going-forward accretion** — no backfill PR; D-entries accrue as their exception paths are next touched.

## ROLLOUT [§7]

**PR1 = mechanism** (all of HOOKS + SKILLS + CLAUDE_MD) + hook unit tests in the `.claude/hooks/test/` harness:
- impl dispatch, stanza present → **allow**
- impl dispatch, stanza absent → **block**
- service `src/**` edit, card has DECISIONS → **injection**
- `git rm` of plan doc → **ask**
- non-impl dispatch → **pass-through**

**PR2–4 = backfill**, one per service (BACKFILL order).

Then **live smoke:** one deliberate stanza-less impl dispatch → block observed; one service-file edit → injection observed.

Rationale for ordering: mechanism first so the backfill PRs themselves run under the new gates — the backfill is the mechanism's first real exercise.

## OUT_OF_SCOPE

**Approach C — lint machinery + max-blocking (REJECTED):** a linter cross-checking `INTENT(D-n)` IDs / `GUARD @ file:line` references against code, enforced as a hard block. Rejected because file:line references go stale during ordinary refactors → false-positive blocks exactly when churn is highest; and the ID cross-check yields ~90% of the value via the intent-review skill (a judgment gate) instead of a brittle mechanical one. audit.sh keeps the *resolvable-citation* checks as advisory findings (HIGH), not blocks.
