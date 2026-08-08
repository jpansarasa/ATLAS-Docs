# Template: Tier-1 claim verification

Fast adversarial check of an agent's REPORT before the supervisor acts on it or relays it.
Not a review of the work — a verification of the claims. Cheap, narrow, background, ~2 min.

Dispatch this whenever a substantive agent report arrives. Skip it for mechanical work
(compile re-runs, pushes, marker refreshes, worktree ops) — nothing there to be adversarial about.

Why it exists: agent reports state inferences in the same voice as measurements, and the
supervisor is the only checkpoint between a subagent and the user. Unaided it is a poor one —
the refutation otherwise arrives from a later deep review, after the claim was relayed as fact.
See SKILL.md §TIER1_CLAIM_CHECK.

```
Verify the CLAIMS in the report below. Do NOT redo the work, do NOT review the design,
do NOT read the whole diff. You are checking whether what it SAYS is true.

DESIGN INTENT: none — mechanical claim verification. Change no files.
supersedes: none
guard_tests: none

READ-ONLY. Change nothing. No pushes, no deploys, no restarts. DB SELECT-only.
No live paid-API calls. If a test suite is involved, run it only with the project's
network-disabling flag.

For each factual claim in the report:
1. Does the number REPRODUCE? Re-run the query or count. Report the value you got, not
   whether it "matched" — a number I can compare beats a verdict I have to trust.
2. Does the file:line CITATION exist and say what is claimed? Line numbers drift with any
   edit above them, and nothing turns red when they do — a stale citation still parses.
3. Is anything asserted WITHOUT evidence — a mechanism named but never probed, a cause
   inferred from correlation, a "verified" that means "read the code"?
4. POPULATION: for every measurement, what can that sample structurally NOT contain?
   Known traps here: the entity_resolution_review_queue holds only FAILURES; SecMaster
   logs only failure paths; the resolver journal is failure-only; a census of surfaces
   that REACHED a destination cannot show what was dropped before it. Check it every
   time — a sample that excludes its own counter-examples reads as confirmation.
5. Any claim of the form "X cannot happen" or "this is the only path" — try once to
   falsify it. Those are the ones that have been wrong.
6. SHAPES the bad claims keep arriving in — check the report against each: a saturating
   counter read as a hard cap; a difference quoted without its run-to-run spread; a
   component described from its NAME instead of its wiring; an untagged metric read as
   zero volume; a cleanup that would delete the real rows it counted.

Report (<=12 lines): a table of {claim, verdict CONFIRMED/REFUTED/UNVERIFIABLE, the value
you measured}. Lead with anything REFUTED. If everything checks out, say so plainly in one
line — a clean result is a useful result and padding it wastes the round.

State explicitly which claims you could NOT check and why.

--- REPORT UNDER VERIFICATION ---
{paste the agent's report verbatim}
```

## Notes for the supervisor

- Dispatch it in the same turn the report arrives; do not relay first and verify after.
  Relaying first is the exact failure this exists to stop.
- Feed it the report VERBATIM. Summarising re-introduces the supervisor's own errors,
  which is one of the things being checked.
- A REFUTED claim does not necessarily invalidate the work — usually the mechanism was
  right and a figure was wrong. Fix the record, keep the fix.
- Do not chain this into a full review. If it surfaces something structural, that is a
  signal to dispatch a real adversarial review, not to widen this one.
