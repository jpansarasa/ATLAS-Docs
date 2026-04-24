# Sentinel v2 Redesign — STATE

> Supervisor's working memory. FIRST read + LAST written every session.
> Plan file: `/home/james/.claude/plans/we-need-to-fix-twinkly-dragon.md`
> Contract: §5.2 of the plan.

## Current phase
Phase 0 — DONE (merged to main; tag `redesign-phase0-complete`)
Phase 1 — starting (v2 architecture specification)
Last updated: 2026-04-24T17:00:00Z  <!-- refresh on every write -->

## VACATION AUTONOMOUS MODE (2026-04-24 → ~2026-05-01)
User in Spain (Europe/Madrid, CEST UTC+2). Authorized ops:
- ✓ `git push origin main` + tag push at phase boundaries (compile+tests green; guard enforces)
- ✓ Azure Foundry spend — burn freely (ledger tracks)
- ✓ DDL + ansible prod deploys — rollback image/snapshot must exist
- ✓ vLLM restart for LoRA training (Phase 3.4) — no pre-notify
- ✓ Self-schedule: 1200s active, 3600s waits, 4–6h routine progress ntfy
- Quiet hours: 22:00–05:00 UTC (Madrid 00:00–07:00 local); silent unless RED_FLAG
- Blocked / ambiguous → ntfy HIGH + wait + poll atlas-claude-reply each wake

HARD STOPS (NEVER auto-execute, always require user approval):
- ✗ Extraction__AutoApproveEnabled=true (Phase 4.5) — D9 rule, even if audit ≥49/50
- ✗ Retire v1 code paths (Phase 5.1)
- ✗ Promote new LoRA as production default
- ✗ observation_events.proto change
- ✗ ZFS rollback on live dataset (always clone+promote)
- ✗ Force push to main

User emergency overrides (ntfy atlas-claude-reply):
- `HALT` → stop dispatches, freeze STATE.md, wait
- `HALT AND ROLLBACK` → git reset to last `redesign-phaseN-complete` tag + restart services

## Last ntfy poll
topic=atlas-claude-reply last_seen_ts=0 (fresh MCP install; no messages yet)
via=mcp (sentinel-ntfy MCP registered in ~/.claude.json)

## Decisions locked (never-undo without supersedes entry)
- 2026-04-24: D1 LLM strategy = local_primary (Qwen2.5-32B-AWQ + sentinel-cove-v7 on vLLM) + cloud_oracle_only (Azure Foundry) — supersedes: none
- 2026-04-24: D2 Scope = full_redesign (extraction + resolution + gate + training) — supersedes: none
- 2026-04-24: D3 Training data = 6,000 examples across 7 buckets (§11.3) — supersedes: none
- 2026-04-24: D4 Spec update required in Phase 5.2 — supersedes: none
- 2026-04-24: D5 Timeline flexible; quality > speed — supersedes: none
- 2026-04-24: D6 Subagent-driven mode with external STATE.md — supersedes: none
- 2026-04-24: D7 Push-hook removed; discipline enforced by §2 + branch strategy — supersedes: none
- 2026-04-24: D8 ThresholdEngine SeriesCollectedEvent contract = UNCHANGED (non-negotiable) — supersedes: none
- 2026-04-24: D9 Kill switch stays ON through Phase 4.4; flips only at Phase 4.5 after audit ≥ 49/50 — supersedes: none
- 2026-04-24: D10 Cloud oracle routing: gold=Opus47, cross-check=Opus46, bulk=Sonnet46, smoke=Haiku45 — supersedes: none

## Open questions (blocking)
- (none)

## Last-verified infra state
As of 2026-04-24 (session start):
- db queue: Pending=27287 Rejected=56 Skipped=6 (Approved=0, quarantine kept)
- quarantined: 17066 rows total (OriginalSymbol populated on 11066 — audit trail intact)
- kill switch: ON (Extraction__AutoApproveEnabled=false in compose.yaml)
- unprocessed raw_content: 30437
- errored raw_content: 59
- ticker_in_quote rows: (not yet queried; track in Phase 2 telemetry)
- vllm-server health: up; serves sentinel-cove-v6.2 (v7 NOT loaded)
- sentinel-collector health: up
- timescaledb health: up
- AutoApproveEnabled: false
- last commit on main: 43b8758 feat(sentinel): scope_nextdata_reprocess utility
- main is 4 commits ahead of origin/main (NOT pushed yet): e4b0c32, 9c4c5e1, ce61558, 43b8758
- tag: redesign-phase0-complete at 43b8758 (local only)
- active feature branch: redesign/phase0-infra (merged; ff to 43b8758; can be deleted after next push)
- Phase 4 audit verdict (2026-04-24): RED — ~58% wrong Symbol on async Finnhub path (the defect driving this redesign)

## Phase 0 artifact audit
| Artifact | Status |
|---|---|
| 0.1 CLAUDE.md SUPERVISOR_MODE section | DONE (line 395, 23 lines, ≤40 cap) |
| 0.2 STATE.md | DONE (this file) |
| 0.3 Pre-flight backups | DONE (2026-04-24T15:51:05Z) |
| 0.4 SentinelCollector/scripts/azure_oracle_client.py | DONE (428 lines; smoke $0.000222) |
| 0.5 SentinelCollector/tests/Golden/*.json (20 docs) | DONE + user-approved; slots 03, 08 re-picked per user direction |
| 0.6 sentinel-mcp (ntfy) | DONE (commit e4b0c32; MCP registered in ~/.claude.json) |
| 0.7 redesign/phase0-infra branch | DONE (created, 1 commit ahead of main) |
| agent-prompts/*.md templates | DONE (6 Phase 0 templates; more added per phase) |

## Subagent dispatch log (append-only; keep last 50 rows)
| ts UTC | phase | agent_type | model | task | outcome | artifact_path | commit |
|---|---|---|---|---|---|---|---|
| 2026-04-24T15:30:00Z | 0.2 | supervisor | n/a | write initial STATE.md | done | /home/james/ATLAS/SentinelCollector/STATE.md | (uncommitted) |
| 2026-04-24T15:50:00Z | 0.templates | general-purpose | Sonnet | create 6 Phase 0 prompt templates | done | SentinelCollector/scripts/agent-prompts/{update_claude_md,preflight_backups,azure_oracle_client,curate_golden_corpus,generic_review,generic_compile_test}.md | (uncommitted) |
| 2026-04-24T15:51:05Z | 0.3 | general-purpose | Sonnet | pre-flight backups | done | /opt/ai-inference/backups/sentinel-pre-v2-20260424T155105Z.dump | (uncommitted) |
| 2026-04-24T16:00:00Z | 0.1 | general-purpose | Sonnet | append SUPERVISOR_MODE to CLAUDE.md | done | /home/james/ATLAS/CLAUDE.md:395 (23 lines) | (uncommitted) |
| 2026-04-24T16:00:00Z | 0.4 | general-purpose | Sonnet | write azure_oracle_client.py + smoke Haiku 4.5 | done (3/3 lines; $0.000222) | SentinelCollector/scripts/azure_oracle_client.py (428 LoC) | (uncommitted) |
| 2026-04-24T16:09:00Z | 0.5 | general-purpose | Opus | golden regression corpus (20 slugs) | done (awaiting user spot-check); 18 exact-spec, 2 deviations (slots 03, 08 — see report) | SentinelCollector/tests/Golden/01..20-*.json | (uncommitted) |
| 2026-04-24T16:20:00Z | 0.5-fix | general-purpose | Opus | re-pick slots 03, 08 per user direction | done (03 → fed-press-monetary id=45 DPCREDIT/DFEDTARL/DFEDTARU; 08 → Challenger severance id=3516 pure skip) | SentinelCollector/tests/Golden/{03-fed-speech,08-historical-avg}.json | (uncommitted) |

## Next action queue (ordered)
1. Dispatch subagent: create Phase 0 agent-prompt templates (update_claude_md.md, preflight_backups.md, azure_oracle_client.md, curate_golden_corpus.md, generic_review.md, generic_compile_test.md) at SentinelCollector/scripts/agent-prompts/
2. Dispatch subagent (Sonnet) with update_claude_md.md: append SUPERVISOR_MODE section to CLAUDE.md
3. Dispatch subagent (Sonnet) with preflight_backups.md: ZFS snapshot + pg_dump + config snapshots + quarantine JSONL + restore verification
4. Dispatch subagent (Sonnet) with azure_oracle_client.md: write azure_oracle_client.py + smoke-test against Haiku 4.5
5. Dispatch subagent (Opus 4.7) with curate_golden_corpus.md: 20 golden docs + ntfy user 5 spot-check entries (USER CHECKPOINT)
6. Commit Phase 0 deliverables; await user approval to merge redesign/phase0-infra → main + tag redesign-phase0-complete

## Cloud oracle ledger
Total spend: $0.000222  (as of 2026-04-24T16:00:00Z)
Ledger path: /opt/ai-inference/training-data/azure-oracle-ledger.jsonl
By run:
- 2026-04-24T16:00Z: phase0-smoke claude-haiku-4-5 3 requests $0.000222 (smoke test)

## ntfy sent log (append-only; keep last 20)
| ts UTC | topic | title | msg_id |
|---|---|---|---|

## Backups registry
### 2026-04-24T15:51:05Z — pre-v2-redesign
| artifact | path | sha256 | note |
|---|---|---|---|
| zfs snapshot | `nvme-fast/timeseries@pre-v2-redesign-20260424T155105Z` | — | verified in `zfs list -t snapshot` |
| pg_dump | `/opt/ai-inference/backups/sentinel-pre-v2-20260424T155105Z.dump` | `73c7ccc1db7426a4fd78b9d9d104e11a9e50af5a8949c25373c713116edb009d` | 18.2 MB; **pg_restore WITHOUT `-n` flag** (dump is schema-scoped) |
| restore verify | — | — | 27371 rows MATCH live; throwaway DB dropped |
| compose.yaml | `/opt/ai-inference/backups/compose-pre-v2-20260424T155105Z.yaml` | — | 29K |
| appsettings.json | `/opt/ai-inference/backups/appsettings-pre-v2-20260424T155105Z.json` | — | 4.0K |
| prompts tgz | `/opt/ai-inference/backups/prompts-pre-v2-20260424T155105Z.tgz` | — | 8.3K |
| quarantine JSONL | `/opt/ai-inference/training-data/quarantine-snapshot-pre-v2-20260424T155105Z.jsonl` | — | 11066 lines (~6970 unique — export script fidelity issue; see §Phase-3-prereqs) |

## Known defects to track (not blocking Phase 0)
- **export_symbol_training_data.py fidelity**: embedded newlines in `text_quote` cause row-splitting (only ~6970 unique pairs from 11066 output lines). Fix before Phase 3 training-data use. Line ~36-40; use `COPY ... WITH (FORMAT csv)` or psycopg with row boundaries.
- **V1 densifier BLS-contamination injection (major finding 2026-04-24, slot 0.5 curation)**: the v1 densifier has systematically injected a boilerplate BLS payroll package ("256,000 nonfarm payroll / +46K healthcare / -21K retail / 4.1% UNRATE / 62.5% participation / 3.9% YoY / $35.69 hourly") across ~40%+ of post-processed rows, appearing verbatim in earnings calls, TSA data, Fed speeches, unrelated articles. Likely a large share of v1 misattributions trace here. Golden slots 02, 03, 11, 13, 16 explicitly test that v7 rejects this contamination. Track in Phase 1 prompt design + Phase 3 training negatives.
- **Async `ResolutionWorker` still description-only** (src/Workers/ResolutionWorker.cs:~196) — Phase 1/2 target.
- **Searxng prompts >16K tokens** → vLLM 400 (~55 errored) — Phase 2.5 target.
- **ChunkingThresholdTokens=24000 rarely fires** — Phase 2.5 lowers to 15000.

## Golden corpus resolution (user-approved 2026-04-24)
- Slots 01, 02, 04, 05, 06, 07, 09, 10, 11, 12, 13, 14, 15, 16, 17, 18, 19, 20: accepted as-is.
- Slot 03 re-picked → fed-press-monetary id=45 (Fed discount-rate minutes May-June 2025). Extracts DPCREDIT=5.25, DFEDTARL=5.00, DFEDTARU=5.25. PCE 3.5% projection rejected; 4.1% densifier contamination rejected (explicitly tagged in source).
- Slot 08 re-picked → Challenger severance benchmarking 2025 (id=3516). Pure skip (`expected_extractions: []`, `skip_reason=historical_average`). Every number is cohort-average/annual aggregate.

## Branch / tag plan
- redesign/phase0-infra (active) — Phase 0 deliverables land here
- merge → main + tag redesign-phase0-complete after user approval
- redesign/phase1-design (next) — v7 prompt + design doc + schema migration
- redesign/phase2-pipeline (next) — v2 code behind flag
- redesign/phase3-training (next) — LoRA v7
- redesign/phase4-cutover (next) — shadow + A/B + audit
- redesign/phase5-cleanup (next) — v1 retire + spec + continuous loop
