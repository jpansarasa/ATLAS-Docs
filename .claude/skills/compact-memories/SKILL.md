---
name: compact-memories
description: Sweep memory descriptions + MEMORY.md hooks to the density bar, verify index integrity, and flag retirement/merge candidates (report-only) — lead with trigger, cut filler, leave tight ones alone. Invoke when memory bloat is suspected or after bulk memory additions.
---

# compact-memories

Dispatches a sweep agent (sonnet/haiku) over `~/.claude/projects/-home-james-ATLAS/memory/` to tighten the two scan surfaces — `description:` frontmatter and MEMORY.md hook lines — to the density bar encoded in `feedback_agent_doc_format`, verify the index, and REPORT retirement candidates.

## ETHOS
snapshot first # memory dir is NOT in git — without the snapshot one bad sed is unrecoverable
verify by diff, never trust agent self-report
compress lines AND flag retirement, never auto-delete # entry COUNT drives bloat more than line length; deletion = human judgment

## THE BAR
description + hook = **keyword-dense scan surface**: lead with the relevance-trigger, cut filler, preserve WHEN-relevant signal. Body = readable prose (don't compress). The test: does the first 10 words tell a future session whether to recall this?

## WHAT TO SWEEP

**description:** field (one line, YAML frontmatter):
- Lead with the distinctive concept / trigger (not "This memory…", "Remember that…", "We learned…", "A memory about…", "In order to…")
- Cut preamble that restates the file name or the fact that it's a memory
- PRESERVE: specific values (model names, thresholds, tool names, service names), ¬/✗ markers, metric numbers — these ARE the trigger
- Leave descriptions <=~140 chars that are already tight (don't churn for marginal gain)

**MEMORY.md hook** (`- [Title](file.md) — <hook>`):
- Hook text after ` — ` must be <=~120 chars; cut prose that duplicates what's already in the title
- Same filler rules apply

## RETIREMENT / MERGE [REPORT-ONLY — never delete]
Flag with a one-line reason each:
- completed one-time authorizations / project states (the event has passed)
- superseded: fact now encoded in CLAUDE.md, a skill, or an AGENT_README D-entry — cite WHERE
- date-bound facts whose date is >~60 days old and whose status plausibly changed — cite the date
- near-duplicate pairs -> propose the single merged description

## INDEX INTEGRITY [mechanical]
- MEMORY.md line whose `(file.md)` target does not exist -> report (stale delete)
- memory file absent from MEMORY.md -> report (index gap)

## NEVER TOUCH
- `name:` / `metadata:` / `[[links]]` / body prose
- YAML structure / quote style
- Already-tight entries (if it passes the bar, leave it)
- No deletions of any kind — retirement is a report, not an action

## DISPATCH BRIEF (use sonnet or haiku)

```
DESIGN INTENT: none — mechanical docs sweep, no D-entries touched.

Sweep ~/.claude/projects/-home-james-ATLAS/memory/ — tighten description: frontmatter + MEMORY.md hooks to the density bar; verify index; report retirement candidates.

STEP 0 (before any edit): cp -r the memory dir to the session scratchpad as memory-backup-<epoch>; report the path. Abort if the copy fails.

BAR: lead with relevance-trigger, cut filler ("This memory", "Remember that", "We learned", "A memory about", "In order to"), preserve specific values + ¬/metric markers. Body = untouched.

RULES:
- description: -> one line, YAML-valid (preserve quote style), <=~140 chars target; skip if already tight
- MEMORY.md hook (after " — ") -> <=~120 chars; skip if already tight
- NEVER edit: name / metadata / [[links]] / body / anything outside description: line + MEMORY.md hook line. NEVER delete a file or an index line.

REPORT-ONLY SCANS (no edits):
- retirement candidates: completed one-time states | superseded by CLAUDE.md/skill/AGENT_README (cite where) | date-bound >~60d (cite date) | near-duplicates (propose merged description) — one-line reason each
- index integrity: MEMORY.md targets that don't exist; files missing from MEMORY.md
- bodies >~300 words, for manual review

VERIFY AFTER EDIT (all must pass; on failure restore from snapshot and report):
1. diff -r <snapshot> <live>: every changed line is a description: line or a MEMORY.md hook line — nothing else differs
2. every .md file still has all three frontmatter keys (name:, description:, metadata:)
3. every MEMORY.md (file.md) target exists

OUTPUT: snapshot path · N tightened / M left-as-is · 6–8 before->after examples · verify results (1–3) · retirement report · index-integrity report · bloated-body list
```

## COMPLETION_GATE
never declare done UNTIL ALL:
1. snapshot path reported (exists in scratchpad)
2. diff-vs-snapshot shows ONLY description:/hook lines changed
3. frontmatter-keys check passes on every file
4. index integrity reported (dangling + orphans)
5. counts + examples + retirement report emitted (retirement = report, user decides)
