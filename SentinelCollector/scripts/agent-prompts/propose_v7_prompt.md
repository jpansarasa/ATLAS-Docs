# propose_v7_prompt.md — Phase 1.1 dispatch template

## Objective
Produce a production-ready v7 extract+resolve prompt for the Sentinel LLM path.
The prompt merges extraction + resolution into a single call: the LLM picks
`candidate_index` from a pre-retrieved SecMaster shortlist instead of emitting a
free-form Symbol string (root cause of the ~58% misattribution rate found in the
2026-04-24 Phase 4 audit).

## When to use this template
- Phase 1.1A: Opus 4.7 proposes (this is the fresh-start dispatch).
- Phase 1.1B: Opus 4.6 cross-reviews using a variant of this template
  (see `v7_prompt_cross_review.md`).
- Later iterations: re-run verbatim if golden-corpus eval regressions require
  a prompt rewrite. Tag with `run_label` like `v7-prompt-proposal-B`,
  `v7-prompt-proposal-C`, etc.

## Inputs (supervisor must assemble before dispatch)
- Plan sections §1 (quality bars), §6 (known defects), §11.1 (prompt skeleton),
  §11.4 (golden diversity matrix), §11.5 (source routing), §D1-D10.
- STATE.md notes on v1 defects (BLS densifier contamination; Dynatrace pattern;
  searxng oversize; async ResolutionWorker description-only lookup).
- Extraction JSON schema (14 fields listed in §1.1).
- Candidates block format: `[i] SYMBOL (type): Display Name`, up to 10 entries,
  produced upstream by `SecMaster.SearchInstrumentsAsync(query, limit=10)`.
- Source families routed to LLM path: `rss`, `challenger-rss`, `searxng-content`,
  `fed-rss` (`fed-minutes`, `fed-press-all`, `fed-speeches`). The deterministic
  parsers (`tsa-checkpoint`, `fed-sep`, `truflation-api`) do NOT hit this prompt.
- Skip list from §11.1 (non-US, demographic, historical-avg, methodology,
  redundant, BLS contamination, stand-alone shares).

## Constraints for the proposal
- Prompt must fit in ~2500 tokens (runs on every LLM call; token budget matters).
- 1-2 illustrative few-shots ONLY — pick the hardest positive (Dynatrace-pattern,
  explicit ticker) and hardest negative (BLS densifier contamination in an
  unrelated doc). No exhaustive enumeration.
- Use `{source}`, `{candidates_block}`, `{content}` placeholders.
- Must include three deliverables delimited by `<<<PROMPT_START>>>` /
  `<<<PROMPT_END>>>`, `<<<VARIANTS_START>>>` / `<<<VARIANTS_END>>>`,
  `<<<NOTES_START>>>` / `<<<NOTES_END>>>`:
    1. Production prompt template (verbatim, with placeholders).
    2. Source-specific variant hints for fed-speeches, rss/challenger, searxng.
    3. Design note (1-2 paragraphs): candidate_index-int vs symbol-string,
       strict null preference, open questions for cross-review.

## How to call (Azure Foundry, Opus 4.7)
```bash
source /home/james/.azure-foundry-keys  # AZURE_FOUNDRY_ENDPOINT, AZURE_FOUNDRY_API_KEY
/home/james/ATLAS/SentinelCollector/scripts/.venv/bin/python \
    /tmp/sentinel-remediation/propose_v7_dispatch.py
```
The dispatcher script embeds the full SYSTEM and USER prompts; edit that script
to iterate. Output lands at `/tmp/sentinel-remediation/v7_raw_response.txt` plus
`v7_meta.json` for usage stats. Budget guardrail: one dispatch costs ~$0.45 at
Opus 4.7 ($15/$75 per MTok).

## Outputs
1. `/opt/ai-inference/prompts/sentinel/extract_and_resolve_v7-proposal-<LABEL>.txt`
   — the prompt template.
2. `/opt/ai-inference/prompts/sentinel/extract_and_resolve_v7-proposal-<LABEL>-notes.md`
   — design notes, variants, usage stats, flagged uncertainties.
3. Ledger append to `/opt/ai-inference/training-data/azure-oracle-ledger.jsonl`
   (schema: `ts, run_label, model, input_tokens, output_tokens, cost_est, msg_id,
   duration_ms[, input_sha256]`).

## Acceptance
- Prompt template exists + parses (placeholders present, `<<<...>>>` stripped).
- Notes file exists with usage stats + design rationale + flagged uncertainties.
- Ledger appended with exactly one line for this run.
- Response contains all three delimited sections in order.
- No Anthropic-direct calls were used (Azure Foundry only, per user preference:
  10x cheaper than direct).

## Report shape (<=300 words)
- Proposal file path + word count.
- Notes file path + 3-5 bullet summary of key design choices.
- Template file path (this file).
- Azure cost (input/output tokens + $).
- Ledger line written: YES/NO.
- Uncertainties Opus flagged for cross-review.
