# Reference: sizing a brief, splitting a story, restarting a stalled agent

Read this when picking a template, when a brief is running long, when a story looks too big for one
dispatch, or when an agent stopped mid-thought. SKILL.md DISPATCH carries the mandatory stanzas —
those are NOT here, because they must be present when the dispatch is written.

## Templates — pick one, never hand-roll
`templates/` holds one per dispatch class. Name the tool AND show the shape.
  story-implementation  — an epic story; canonical Git-ops + Design-intent stanzas live here
  implementation-fix    — a MEASURED defect: FRESH branch or FIX round
  recon-measurement     — read-only, one question
  spec-plan-authoring   — a doc that goes through a PR
  docs-accuracy         — facts in docs/cards/memories
  deploy                — already-merged work to prod
  claim-verification    — is one agent report true
Review dispatches have no template — the shape is SKILL.md MERGE_GATE.

## SIZE [the fenced block is what gets pasted, not the file]
<=700w, and justify anything past ~550.
Impl briefs add the mandatory Design-intent + Git-ops stanzas on top (~600w realized).
Measure a template's fenced block:  awk '/^```$/{f=!f;next} f' <template> | wc -w
# the budget is the rule; a snapshot of what the templates measured on some past day is not, and
# copying one here only creates a number to keep in sync

## STORY_SPLIT_HEURISTIC
split if any:
  >20 files touched
  >2 distinct concerns (e.g. schema + readers + writers)
  cross-cutting (DB + readers + writers in one story)
  cross-project cascade
  >3 layers (entity + service + endpoint + tests + migration)
pattern [staged]: additive first -> cutover -> drop
  layer A -> commit -> verify -> layer B -> commit -> ...
  rationale: budget caps kill unstaged work; partial progress preserves

## MID_STOP_RECOVERY [agents budget 50-130 tool uses]
detect: "Acknowledged. Now…" | mid-thought | tool count low
recover:
  1. git status --short -> identify WIP
  2. git log --oneline <last>..HEAD -> see committed
  3. read WIP files (<=1-2 key, <=30 lines)
  4. dispatch a FRESH continuation citing the WIP paths explicitly
  5. never SendMessage(failed agent) — SKILL.md CONFIG FAILURE
