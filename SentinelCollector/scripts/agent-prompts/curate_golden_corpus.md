# curate_golden_corpus — Phase 0.5

# Objective
Curate 20 ground-truth documents from `sentinel.raw_content` covering every diversity slot below. Emit one JSON file per doc at `/home/james/ATLAS/SentinelCollector/tests/Golden/<slug>.json`. After writing, request a human spot-check on 5 random entries via ntfy and wait for reply.

# Inputs
- DB: `sudo nerdctl exec timescaledb psql -U ai_inference -d atlas_data`
- Output dir: `/home/james/ATLAS/SentinelCollector/tests/Golden/` (create if missing)
- Ntfy MCP: `sentinel-ntfy` (`ntfy_publish`, `ntfy_poll_since`). Fallback: curl (`atlas:atlasP@ssw0rd`, `https://ntfy.elasticdevelopment.com`).
- Plan ref: `/home/james/.claude/plans/we-need-to-fix-twinkly-dragon.md` §11.4

# Diversity matrix — exactly one doc per slot (slug = kebab-case)
1. dynatrace-pattern — quote has NYSE:DT, description ambiguous "price target"
2. tsa-checkpoint — tabular page
3. fed-speech — long-form FOMC/speech
4. searxng-oversized — search-result page, oversized prompt
5. challenger-press — headline number + narrative
6. multi-ticker-article — decide subject
7. foreign-release — e.g. India PMI → emit `[]`
8. historical-average — must skip
9. conditional-claim — "if tariffs…"
10. analyst-upgrade — multiple targets
11. earnings-call-snippet
12. index-movement — SPY/DJIA
13. sector-aggregate — healthcare employment etc.
14. methodology-discussion — must skip
15. etf-vs-underlying
16. nextjs-ssr-article
17. macro-indicator — UNRATE/CPI
18. revised-estimate — with supersedes
19. commodity-price
20. crypto-reference

# Commands
1. Per slot, query candidates: `SELECT id, source, title, LEFT(content,200) FROM sentinel.raw_content WHERE <slot-filter> ORDER BY fetched_at DESC LIMIT 20;` — pick one.
2. Write `<slug>.json`:
```json
{
  "slug": "dynatrace-pattern",
  "raw_content_id": "<uuid>",
  "source": "<source>",
  "text_snippet": "<first 800 chars>",
  "expected_extractions": [
    {"subject_entity":"Dynatrace","correct_symbol_or_null":"DT",
     "certainty":"expected","skip_reason_if_any":null}
  ],
  "notes": "<why_chosen>"
}
```
For skip slots (7, 8, 14): `expected_extractions: []` with top-level `skip_reason_if_any`.

3. After all 20 files written, pick 5 random slugs. Body: one bullet per pick: `• <slug> [id]: subject=<s>, symbol=<sym_or_null>, skip=<reason_or_none>`.
4. Publish to `atlas-claude-ask` (title: "Sentinel Phase 0.5 golden spot-check"). Record `publish_ts`.
5. Poll `atlas-claude-reply` (since=publish_ts) until reply; ack via MCP. Timeout 6h; if none, report `human_reply:null` and exit cleanly.

# Acceptance
- 20 files exist, filenames = slugs (case-sensitive, kebab-case).
- Valid JSON; required keys: `slug`, `raw_content_id`, `source`, `text_snippet` (≤800 chars), `expected_extractions`.
- All `raw_content_id` distinct.
- Ntfy publish succeeded; reply polled or timeout recorded.

# Report shape
- `files`: list of 20 absolute paths
- `summary_table`: 20 rows `{slug, raw_content_id, expected_symbol|null, expect_skip:bool}`
- `spot_check_published`: `{ts_iso, slugs_chosen:[5], message_id}`
- `human_reply`: string | null
- `errors`: list
