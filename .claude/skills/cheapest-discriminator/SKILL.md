---
name: cheapest-discriminator
description: Use BEFORE spending anything to answer an open question - a GPU run, a deploy, a labelling round, a subagent dispatch, an hour of reading. Enumerate the candidate answers, name the cheapest observation that ELIMINATES one, spend that first, and run independent questions in parallel. Activate when choosing what to measure or dispatch next, when a comparison is "too noisy to decide", when two or more questions are open at once, when an expensive run is proposed, or when you catch yourself about to understand something fully before deciding anything. Not for irreversible acts - see WHEN_DEPTH_FIRST_WINS.
---

# CHEAPEST_DISCRIMINATOR [SKILL v1]

Exhaustive depth feels like rigor and is mostly latency. Measured 2026-09-05, one session, both ways.
THE WIN: an A/B declared "too noisy to decide" was not re-run. A ten-minute reread of prediction files
ALREADY ON DISK put the noise in the scorer's alignment key rather than the model -- numbers_f1 0.8947
(run-to-run range 0.0120) on a value-only key against the committed key's 0.3883 (range 0.0893), with a
~0.05 effect to detect. Zero GPU. THE LOSS, same session: two independent questions were sequenced, the
one that would have unblocked the comparison was written down and never run.
The lesson is in the win: the cheap observation did not ANSWER the question, it changed WHICH question
was blocking -- a payoff unreachable by descending the branch you happen to be standing on.

## ETHOS
trim branches, do not walk them             # eliminating a candidate is cheaper than confirming one
cheapest class first, always                # a free observation that decides beats an expensive one that decides
name the DECISION, then the work            # work that changes no act is not the next step, however interesting
independent -> parallel, no exceptions      # a queue of independent work costs the longer one and buys nothing
an uncontrolled cheap measure is not cheap  # it is a wrong answer at low cost

## THE_MOVE [before spending anything]
1. Write the DECISION in one line, and the acts it selects between.
2. Enumerate the candidate answers. All of them, briefly. This is the tree.
3. For each candidate, name the cheapest OBSERVATION that would ELIMINATE it.
   elimination, never confirmation # confirming the front-runner leaves every other branch standing
4. Sort those observations by COST CLASS; take the lowest non-empty class and run it NOW.
5. Ask what that observation CANNOT see, and what control proves it can see anything at all.

## COST_CLASSES [say the class out loud before spending; never skip a non-empty lower one]
FREE      artifacts already on disk | a committed scorecard or provenance sidecar | git log |
          a psql SELECT | reading a service card    # no new compute and nobody's consent
CHEAP     one local script over data you already have | a filtered devcontainer test |
          a Prometheus / Loki / Tempo query         # minutes, reversible, no scarce resource
EXPENSIVE GPU inference (~3.6s per article at concurrency 6; the five committed runs = ~33 min,
          3 at c6 and 2 at c1) | a scoped deploy (~4 min vLLM reload) |
          a paid frontier call ($3.40 / 96 requests, measured) |
          a labelling round | A HUMAN'S ATTENTION   # last is scarcest and least reversible
  19.64s in the scorecard is per-REQUEST latency, NOT per-article wall cost # budgeting from it, or
    re-running all five at c6, changes the ARM -- and the sidecar records no concurrency, so nobody
    can detect the substitution (docs/BACKLOG.md "BUDGET ~33 MINUTES, NOT ~11")

## INDEPENDENCE [one question settles it]
Does answering A change what you would ASK or DO in B? No -> independent -> dispatch BOTH now.
Both blocked on the same third question -> answer the third. That is a dependency, not a queue.
This repo removed the last excuse: N worktrees compile SIMULTANEOUSLY (CLAUDE.md OWNED) -- never sequence them.

## THE_TELL [depth wearing rigor's clothes]
The phrase is "let me understand this fully first". Not always wrong, but always untested.
TEST: name what you would DO under each candidate answer.
  different acts -> the understanding is load-bearing, get it
  same act under every answer -> it is completeness, park it; it changes nothing today
WORKED: settling the `source_entity` convention was written into CLAUDE.md as "the precondition for a
model swap". Measured once settled: worth at most +0.2048, and it did not reliably repay the
run-to-run swing -- the sign REVERSES BY ARM (pooled the range GROWS 15.6%, at c6 it GROWS 45.0%, at
c1 it SHRINKS 14.3%), and n=5 across two arms cannot say which way it cuts. The swing is what a
comparison must actually clear.
  # a correctly-identified precondition still did not move the blocked act. The question you BELIEVE
    gates the decision is a hypothesis, and the cheap measurement is what tests it

## WHEN_DEPTH_FIRST_WINS [this is a default, not a prohibition]
Exhaust the tree when being WRONG is expensive to undo:
  irreversible or destructive   # a merge, a prod deploy, a migration, a raw-DB write, `git clean -x`
  fails silently toward success # a guard or gate; ambiguity must resolve to DENY -> `guard-change`
  one-way consent               # a human's attention, a paid budget, a claim published to the user
Exhaust it ALSO when the cheap instrument is unproven.
  # same session: the rescorer's own shuffled-gold control judged the MEAN across runs and averaged a
    real breach (0.4570, 4.6x its bar) down to a passing 0.0457 -- green, exit 0, and blind
  -> the FIRST cost of a cheap discriminator is its known-bad control. Pay that, then trust the number.

## FAILURE_DETECTOR [a rule that cannot detect its own violation is the defect class this skill is about]
D1 ANSWER_LATENCY [primary -- chosen because BOTH endpoints are durable in git]
  posed:    `git log -S'<phrase distinctive to the entry>' --date=iso --reverse -- docs/BACKLOG.md | head -1`
  answered: the commit carrying the deciding number
  latency far exceeding the COST CLASS of the deciding observation means depth-first happened.
  --date=iso, NEVER --date=short # the whole CoD thread above lands on one calendar day; day
    granularity reports zero latency for exactly the sessions this detector exists to catch
  choose the phrase, then CHECK the oldest hit is the same question # a pickaxe matches a TOKEN:
    `-S'source_entity'` on that file returns 2026-08-15, an unrelated D-1 coverage entry
    -> NARROW the phrase until the oldest hit IS the question, and re-run. Note the direction: an
       unrelated older hit INFLATES the latency, so this one manufactures a violation rather than
       hiding one -- the opposite of `--date=short`, and both are wrong
  WHY THIS ONE: the session transcript rotates and STATE.md is untracked, so anything measured only
    in-session is unfalsifiable a week later -- the defect that left a rescorer in /tmp until one
    tmpwatch made a whole backlog entry uncheckable
D2 SERIAL_DISPATCH [in-flight, self-catch, free]
  at each dispatch, count OPEN independent questions against dispatches IN FLIGHT (STATE.md names them).
  open > 1 AND in flight == 1 -> you are serializing. State the dependency, or dispatch the other now.
  # honest limit: observable only while the session lives. It catches you; it cannot audit you
D3 EXPENSIVE_BEFORE_FREE [per-question audit]
  an EXPENSIVE artifact dated before the first free analysis THAT COULD HAVE RUN FIRST -- one whose
  inputs all predate the spending. `ls -l` the run outputs, `git log` the script that consumes them.
  an analysis that CONSUMES the expensive output could not have gone first, and finding one after it
    is NOT a violation # measure-then-analyse is the legitimate order; a detector that convicts it
    teaches the false lesson "do not measure before analysing"
  # the rescore that reframed the five GPU runs is exactly that case -- it reproduces all five
    published `numbers_f1` values, so it had nothing to read until they existed. What the timing
    DOES convict: run 1's output existed ~2.5 min in, and ~30 further GPU-minutes were spent before
    any prediction file was read (docs/BACKLOG.md "BUDGET ~33 MINUTES, NOT ~11")
NOT USED: "count of sequential dispatches with no dependency" as an AUDIT -- dispatch records live only
  in the rotating transcript, so it is D2's in-flight form or it is nothing.

## ALREADY_COVERED [go there, not restated here]
running N agents at once      -> `superpowers:dispatching-parallel-agents`, supervisor-mode `references/parallel-dispatch.md`
a tool reporting its dullness -> CLAUDE.md TOOL_UPKEEP; controls with teeth -> `guard-change` items 2 and 7
what a sample CANNOT contain  -> memory `feedback_evidence_population_bias`
where a finding is written    -> CLAUDE.md WHERE_WORK_LANDS

## ANTI [HARD_STOP @end for recency]
✗ spend an EXPENSIVE class while a FREE one bearing on the same question is unexploited
✗ sequence two questions that do not gate each other        # the queue IS the whole cost
✗ measure when every answer leads to the same next act      # interesting is not decision-relevant
✗ trust a cheap number from an instrument with no known-bad control  # cheap and wrong is not cheap
✗ write the discriminator down and not run it               # the named-and-unrun option is the loss above
✗ confirm the front-runner                                  # elimination trims the tree, confirmation walks it
✗ breadth-first an irreversible act                         # exhaust it; wrong there is not undoable
✗ report an answer-latency from --date=short                # a single-day session reports zero
