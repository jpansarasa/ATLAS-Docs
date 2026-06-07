---
name: compact-memories
description: Sweep memory descriptions + MEMORY.md hooks to the density bar — lead with trigger, cut filler, leave tight ones alone. Invoke when memory bloat is suspected or after bulk memory additions.
---

# compact-memories

Dispatches a sweep agent (sonnet/haiku) over `~/.claude/projects/-home-james-ATLAS/memory/` to tighten the two scan surfaces — `description:` frontmatter and MEMORY.md hook lines — to the density bar encoded in `feedback_agent_doc_format`.

## THE BAR
description + hook = **keyword-dense scan surface**: lead with the relevance-trigger, cut filler, preserve WHEN-relevant signal. Body = readable prose (don't compress). The test: does the first 10 words tell a future session whether to recall this?

## WHAT TO SWEEP

**description:** field (one line, YAML frontmatter):
- Lead with the distinctive concept / trigger (not "This memory…", "Remember that…", "We learned…", "A memory about…", "In order to…")
- Cut preamble that restates the file name or the fact that it's a memory
- PRESERVE: specific values (model names, thresholds, tool names, service names), ¬/✗ markers, metric numbers — these ARE the trigger
- Leave descriptions ≤~140 chars that are already tight (don't churn for marginal gain)

**MEMORY.md hook** (`- [Title](file.md) — <hook>`):
- Hook text after ` — ` must be ≤~120 chars; cut prose that duplicates what's already in the title
- Same filler rules apply

## NEVER TOUCH
- `name:` / `metadata:` / `[[links]]` / body prose
- YAML structure / quote style
- Already-tight entries (if it passes the bar, leave it)

## DISPATCH BRIEF (use sonnet or haiku)

```
Sweep ~/.claude/projects/-home-james-ATLAS/memory/ — tighten description: frontmatter + MEMORY.md hooks to the density bar.

BAR: lead with relevance-trigger, cut filler ("This memory", "Remember that", "We learned", "A memory about", "In order to"), preserve specific values + ¬/metric markers. Body = untouched.

RULES:
- description: → one line, YAML-valid (preserve quote style), ≤~200 chars target; skip if already tight
- MEMORY.md hook (after " — ") → ≤~120 chars; skip if already tight
- NEVER edit: name / metadata / [[links]] / body / anything outside description: line + MEMORY.md hook line

OUTPUT FORMAT (before editing):
1. For each file you'd change: show BEFORE → AFTER description (one line each)
2. MEMORY.md: show changed hook lines (BEFORE → AFTER)
3. Count: N tightened / M left-as-is
4. YAML-validity confirmation (grep -c '^description:' count matches edited files)
5. REPORT-ONLY (¬edit): any BODY paragraphs that look bloated (>~300 words), for manual review

Then apply the edits.
```

## OUTPUT CONTRACT
- Count tightened vs left-as-is
- 6–8 before→after examples in final report
- YAML-valid confirmation
- Report-only list of bloated bodies (don't edit them)
