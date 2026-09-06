---
name: re-derive
description: Terminate a disagreement by re-deriving the disputed quantity from its source instead of adjudicating the last opinion. Activate the moment a claim is disputed a SECOND time, when a refutation refutes a refutation, when a review round reopens a figure an earlier round already settled, when a dispute has become about WORDING, before relaying any refutation that licenses a DELETE, and whenever you are about to write "an earlier revision of this line read ...". Applies to measured figures, counts, citations, thresholds, partitions and derived RULES alike -- anything whose source can be read or run again.
---

# RE_DERIVE [SKILL v1]

A claim adjudicated a third time has a METHOD problem, not a content problem. Chained opinions converge on
noise: each round is genuinely responsive to the last, which is exactly why a loop feels like progress. A
refutation of a refutation is a third OPINION, not evidence. Only re-deriving the disputed quantity from its
source terminates the loop, and the loop costs most in the one direction review cannot audit.

## ETHOS
derive > adjudicate         # an argument answers the last speaker; a derivation answers the question
the SUBJECT is the dispute  # both sides can hold every number correct and still mean different quantities
unresolved-and-named > last-opinion-standing  # the last word reads as settled only because it is unanswered

## SEQUENCE [at the moment of doubt, in order]
1. count the passes over THIS proposition          # pass 3 may not be an opinion
2. restate the proposition and NAME ITS SUBJECT    # different subjects -> not a dispute at all
3. name the cheapest sufficient derivation and its cost -> run it
4. cannot run it -> record UNRESOLVED with what would settle it, and never send the opinion instead
5. licenses a DELETE -> step 3 is not optional, at any pass count

## PASS_COUNTER [the tripwire]
pass 1 the claim | pass 2 the first dispute | pass 3 adjudicating a dispute.
RULE: THE THIRD PASS OVER ONE PROPOSITION MAY NOT BE AN OPINION. Re-derive it, or record it unresolved.
count passes over the PROPOSITION -- not rounds, files, agents or PRs # three agents in one round is pass 2
WHY 2 AND NOT A ROUND NUMBER: below, every dispute that settled at pass 2 settled by going to the source and
  every one that reached pass 3 was still wrong on arrival # pass 3 has never been the pass that argued best
AND A DERIVATION ONLY ENDS THE LOOP IF IT IS PROPAGATED: the 3-pass loop below DID derive at pass 2, and ran
  a third anyway because the headline was left asserting what the body had already stopped saying # carry the
  derivation to every line that states the claim, or you have bought the third pass regardless
THE COUNT IS A SMOKE ALARM, NOT THE FIRE: the defect is adjudicating without deriving and it is already there
  at pass 2 # the counter exists because each round is responsive, courteous and wrong, so nobody notices
A DISPUTE THAT HAS TURNED TO WORDING IS THE SAME ALARM EARLIER # arguing about how a claim is PHRASED is how
  a pass 3 gets manufactured out of a pass 2 that should have gone to the data

## NAME_THE_QUANTITY [one sentence of work, before disputing anything]
Restate the proposition in your own words, then name its SUBJECT: what quantity, over what population, at
  what scale. Subjects differ -> you are not disputing, you are answering different questions -> say so, stop.
ARITHMETIC-FLAVOURED REFUTATIONS ARE THE DANGEROUS ONES # every number in one can be right while the quantity
  they concern is the wrong one -- the signature failure, not a rare one
a figure carrying no arm, scale or population is not yet a proposition # one 0.2048 was three numbers by arm
A DERIVED RULE HAS A SUBJECT TOO, the corpus it came from # a rule that keeps outvoting the case in front of
  you is the thing to re-derive, not the answers it is rejecting

## RE_DERIVE [what counts -- everyone believes they are already doing this]
Go to the thing itself: run the query, compute the partition, read the artifact, execute the code path.
TEST: the same answer comes out if every prior message in the thread is deleted, and you can hand over a
  command, a file plus anchor, or a computation another party re-runs unchanged # that IS the definition
NOT a derivation: re-reading the previous opinions harder, or arguing the wording better; a second reviewer,
  a bigger model, a tiebreak vote (three opinions is not a measurement); a number you REMEMBER being
  measured -- re-run it # every self-correction below started life as a remembered number
CHEAPEST SUFFICIENT, and state what it does not cover # an unaffordable rule is a rule that does not exist
SHIP THE RE-CHECK WITH THE ANSWER, the command beside the figure # a bare number is pass 1 of the next loop,
  restarting with the derivation already lost

## DELETION_ASYMMETRY [where the bar goes highest]
Acting on a false ordinary claim ADDS something wrong and the next reader sees it: the tree carries it.
Acting on a false REFUTATION deletes something right, and deletion is the one edit later review CANNOT audit
  -- reviewers read what IS in the tree, never what used to be # one such delete took a D-entry's INTENT
RULE: a refutation that licenses a delete gets a derivation at pass 2, never an argument.
  cannot derive -> KEEP the text, mark it disputed, name what settles it # a wrong sentence is recoverable

## CANNOT_RE_DERIVE [the record IS the deliverable]
Record UNRESOLVED with the proposition, its subject, both positions, and the artifact or measurement that
  would settle it -- specific enough that the next person closes it in one step.
never ship the last opinion standing as a conclusion # it reads as settled only because it is unanswered
never leave the disputed value circulating unlabelled # it becomes the next loop's pass 1, laundered

## ADJUDICATION_IS_SOMETIMES_RIGHT [not a blanket ban]
A genuine judgement call has NO source to return to. Tell one by asking what you would RUN:
  nothing to run, and both parties keep their positions on identical data -> judge it
  you can name the command and are declining the cost -> that is COST, not judgement; price it out loud
  ground truth exists but is inconvenient (a rebuild, a rerun, a human decision) -> escalate, never adjudicate
It still terminates by a DECISION WITH A NAMED OWNER, not by a third opinion # a convention, a threshold and
  a taxonomy are decisions, and who decided is the part that stops the reopening

## FAILURE_DETECTOR [how a reader knows this skill is not firing]
PRIMARY, observable in git because the record is the claim's own file history: passes over one proposition,
  and WHAT THE SETTLING PASS DID. `git log -p` the line -- 3+ revisions of one proposition, or a settling
  revision carrying no command and no computation, means this did not fire -- but a LOW count proves NOTHING
  until you have read the squash caveat below. Self-correction is greppable:
  `grep -ci 'an earlier revision' docs/BACKLOG.md` prints 8 today, 6 of them inside ONE entry -- and that is
  a FLOOR, because the phrase wrapped across a line break does not match.
THE PASS COUNT IS BLIND ON A SQUASH-MERGING REPO -- OURS -- AND BLIND TO BOTH LOOPS BELOW, which are intra-PR:
  every pass inside one PR collapses into a single commit on main, so
  `git log -L '/0.2048 IS AN UPPER BOUND/,+1:docs/BACKLOG.md'` returns 2 revisions for the THREE-pass loop
  below. Count intra-PR passes from the PR's OWN commits (`gh pr view <N> --json commits`), never main's log.
  THE OTHER TWO CLAUSES SURVIVE SQUASHING: a settling revision carrying no command still reads as one, and the
  `an earlier revision` grep works entirely, because the prose records its own supersession in the tree.
  PERISHABLE: the branch is deleted on merge and the default refspec does not fetch `refs/pull/*`, so ALL SIX
  of #1017's commits -- the four carrying this evidence among them -- are ALREADY unreachable objects,
  surviving only on the machine that made them. Re-derive from a merged PR early or not at all.
SECONDARY: REVERSALS. A -> not-A -> A over one proposition means a true thing was deleted and restored; count
  them, then note that the ones nobody restored are unmeasurable, which is why they are the real cost.
REJECTED as detectors: elapsed time and round count (a slow correct derivation scores worse than a fast wrong
  loop); agent count (three agents in one pass is not three passes); confident tone (every loop here had it).

## EVIDENCE [what this cost -- the incidents are not the content]
#1016, one bound, three passes on its WORDING: "upper bound BY CONSTRUCTION" -> "measured, not by
  construction" -> headline still contradicting the body. The SECOND pass derived it -- maximum-cardinality
  matching on the same graph, 1,465 pairs against greedy's 1,451, available at pass 1 -- and a third ran
  anyway because that derivation was never PROPAGATED: the headline went on asserting what the body had
  already stopped saying. Pass 3 computed nothing; it ended the loop by reading the entry against itself.
  `docs/BACKLOG.md`, "IS A MACRO SERIES AN OWNER?"
#1017 COUNTER-EXAMPLE, the shape to copy: a first-contact review refuted `496 of 518 conform` by re-deriving
  the partition (490 + 6 + 22 = 518). One pass, settled -- not more careful, it went to the source.
THIS SKILL'S OWN REVIEW, and it happened to a TOOL, not a person: a dispatched `review-pr` agent confirmed the
  `496 of 518` text really is in the backlog and reported it as "the #1016 entry" -- verifying the artifact
  without verifying its source, reproducing the exact citation error it was checking for.
#1017: a `source_entity` precedence rule generalised from a bake-off whose corpus was 4-of-5 macro discarded
  the cross-check's CORRECT owner repeatedly before the RULE was questioned instead of the answers; its own
  comment records the count. `LlmBenchmark/scripts/build_cod_gold.py`, SOURCE_ENTITY PRECEDENCE.
2026-08-13: three agents, three rounds, one figure -- right from the start. Relayed as CRITICAL it deleted a
  true sentence from four files and took two more rounds to undo; every number in the refutation was correct
  and only the SUBJECT differed (a cost BAND tested against a claim about one LEG of that cost).

## ANTI [HARD_STOP @end for recency]
never open a third pass on one proposition with an opinion
never dispute a claim before restating its proposition and naming its subject
never count a better argument, a second reviewer or a bigger model as a derivation
never act on a refutation that licenses a DELETE without a derivation
never ship the last opinion standing as a conclusion -- record it unresolved and name what would settle it
never carry a measured number forward without the command that reproduces it
never adjudicate a question whose ground truth you are declining to fetch -- price it or escalate it
never re-derive a rule's answers when the RULE is what has been outvoting them
