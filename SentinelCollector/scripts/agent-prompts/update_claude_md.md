# update_claude_md — Phase 0.1

# Objective
Append exactly one `SUPERVISOR_MODE` section to `/home/james/ATLAS/CLAUDE.md`. No other edits. Match existing token-dense `KEY: value # comment` style (see `## PRIORITY_0`, `## DEPLOYMENT`).

# Inputs
- Target file: `/home/james/ATLAS/CLAUDE.md`
- Plan file (reference only): `/home/james/.claude/plans/we-need-to-fix-twinkly-dragon.md`
- Azure keys path: `/home/james/.azure-foundry-keys`
- Ntfy creds: user `atlas`, pass `atlasP@ssw0rd`, server `https://ntfy.elasticdevelopment.com`

# Commands
1. Read `/home/james/ATLAS/CLAUDE.md` once to confirm current tail. Do NOT modify any existing line.
2. Append the section below verbatim (adjust only if an existing `## SUPERVISOR_MODE` already exists — in that case, abort and report).

```
## SUPERVISOR_MODE [sentinel_v2_redesign]
PLAN: /home/james/.claude/plans/we-need-to-fix-twinkly-dragon.md # strategy, phases, contracts
STATE: /home/james/ATLAS/SentinelCollector/STATE.md # supervisor memory, read_first | write_last
SUPERVISOR_TOUCHES_CODE: never # except STATE.md, plan_file, CLAUDE.md
ALL_IMPL|REVIEW|VALIDATION: subagent # dispatch ¬ direct_edit
TEMPLATES: /home/james/ATLAS/SentinelCollector/scripts/agent-prompts/ # reusable, ≤400w each

CLOUD_ORACLE: Azure_Foundry # /home/james/.azure-foundry-keys
  gold_label|architecture: claude-opus-4-7
  cross_check: claude-opus-4-6
  bulk_label|impl: claude-sonnet-4-6
  smoke|triage: claude-haiku-4-5
  cap: 500K_tpm_client_side | ledger=/opt/ai-inference/training-data/azure-oracle-ledger.jsonl

NTFY:
  server: https://ntfy.elasticdevelopment.com | auth: atlas:atlasP@ssw0rd
  publish(user←supervisor): atlas-claude-ask
  poll(user→supervisor): atlas-claude-reply
  mcp: sentinel-ntfy # registered in ~/.claude.json; tools: ntfy_publish|poll_new|poll_since|ack

WAKEUP_STEP_0: poll(atlas-claude-reply) BEFORE routine_work # learned 2026-04-24
FAILURE: bad_subagent_result → fix_prompt + dispatch_fresh ¬ SendMessage
CONTEXT: long_output → /tmp/sentinel-remediation/<file> ¬ supervisor_turn
```

3. Save. Run `wc -l /home/james/ATLAS/CLAUDE.md` before and after; diff should be exactly the appended lines.

# Acceptance
- Section length ≤40 lines.
- No existing rule modified (`git diff /home/james/ATLAS/CLAUDE.md` shows only additions after the last existing line).
- Format matches neighboring sections (ALL_CAPS keys, `#` trailing comments, no prose paragraphs).
- Final newline preserved.

# Report shape
- `before_lines`: int
- `after_lines`: int
- `lines_added`: int (must equal appended section length)
- `git_diff_summary`: one-line (e.g. "+40 -0, append-only at EOF")
- `abort_reason`: null | string (e.g. "SUPERVISOR_MODE already present at line N")
