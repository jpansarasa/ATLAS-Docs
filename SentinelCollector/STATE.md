# Sentinel v2 Redesign — STATE

> Supervisor's working memory. FIRST read + LAST written every session.
> Plan file: `/home/james/.claude/plans/we-need-to-fix-twinkly-dragon.md`
> Contract: §5.2 of the plan.

## Current phase
Phase 0 — DONE (tag `redesign-phase0-complete` pushed)
Phase 1 — DONE (PR #196; tag `redesign-phase1-complete` pushed at 38a2af9)
Phase 2 — DONE (PR #197; tag `redesign-phase2-complete` pushed at be79ab2; deployed + verified on mercury with flags OFF)
Phase 3 — DONE (take2b LoRA r=8 alpha=16 with q/k/v/o + gate/up/down MLP layers; epoch-3 chosen; eval FAILED Symbol target but PASSED null-precision; ~half of misses traced to Phase 2.1 candidate-generator quality not LoRA)
Phase 4.1 — LIVE since 2026-04-30 (shadow mode for fed-rss; v7-e3 writes to extracted_observations_shadow, v6.2 keeps prod). 72h soak passed.
Phase 4.2 — DONE 2026-05-02 (PR #198 merged at e38f0c5; tag `redesign-phase4-rag-fix` pushed; RagSynthesis hypothesis materialization + 4 follow-up review fixes deployed; 539/539 tests green; 0 new errors in Loki post-deploy)
Phase 4.3 — IN PROGRESS as of 2026-05-03 — V2 precision fix portfolio. Plan approved: 5 PRs across 5 levers (server RAG hardening, client guards, minScore raise, Gemini fallback flip, catalog enrichment + FRED dedupe). Plan file: `/home/james/.claude/plans/let-s-plan-this-in-mighty-boot.md`. Target: ≥95% precision on V2 shadow audit (recall may dip below V1's 58%). PR-1 OPEN as #200 awaiting user review/merge.
Last updated: 2026-05-03T10:42:00Z

## Live alert + open decision (2026-05-03)
**Alert** SentinelLowResolutionRate (P4 warning) fired at 21:27 local. Alert rule: `sum(rate(sentinel_secmaster_resolution_total{status="resolved"}[5m])) / sum(rate(sentinel_secmaster_resolution_total[5m])) < 0.5 for 15m` (`deployment/artifacts/monitoring/alerts/sentinel.yml:80`).

**Real numbers** (last 30m as of update):
- V1 prod (sentinel-collector → extracted_observations): **96/164 = 58% resolved** — healthy
- V2 shadow (extracted_observations_shadow): **50/144 = 35% resolved** — below alert threshold
- Pre-fix V2 was 0%. Phase 4.2 fix is doing its job; precision is the new bottleneck.

**Precision problem** Spot-check of V2 hybrid_subject "successful" matches showed ~50%+ are bound to the WRONG instrument:
- ✓ Equinix→EQIX, Nike→NKE, IMAX→IMAX, Savara→SVRA, Digital Realty→DLR, Scorpio Tankers→STNG
- ✗ Klarna→MRNA (Moderna), Micron→MCHP (Microchip), Wingstop→WKSP (real: WING), ClearSign Combustion→CLSK (CleanSpark), Phreesia→FFZY (real: PHR), EPA→AEP (American Electric Power), supramax index→SPX (S&P 500), Forgent Power→FTNT (Fortinet), Klarna→MRNA

**Root cause hypothesis** SecMaster's vector search + RAG hypothesis matches on embedding similarity, not entity identity. Two contributing factors:
1. FRED catalog backfill polluted symbol space — many tickers got hijacked as Economic Indicators with wrong names (e.g. ALHC→"Alignment Healthcare revenue", MU→"Aroundtown share buyback", CBOE→"California consumer confidence index volatility").
2. minScore threshold (0.6 in `ResolveLocalFromQuoteAsync`) is too generous; nearest-neighbor often shares embedding space without sharing identity.

**Triage options presented to user (none chosen yet)**
1. Roll back PR #198 → restores 0% V2 resolved, no false positives. Counterproductive.
2. Raise SecMaster `minScore` 0.6 → 0.85+ (recall ↓ for precision ↑)
3. Name-overlap guard in `DeterministicResolver`: only accept materialized lookup if `lookup.Name` shares a token with `subject_entity`
4. **(my recommendation)** Asset-class hint: if subject ends in Inc/Corp/Ltd, reject candidates with `asset_class IN ('Indicator','Economic','Economic Indicator')`. Directly attacks the FRED-overlap problem (Micron→MCHP, supramax→SPX, EPA→AEP all fit).
5. Acknowledge alert + plan precision PR; V2 is shadow-only, no immediate prod harm.

**Containment** V2 still shadow-only (`UseV2Pipeline=false`, only fed-rss source opted in). False positives stay in `extracted_observations_shadow`, do NOT reach event bus / threshold engine / alert path. Production extractions go through V1 (sentinel-cove-v6.2), unaffected.

**Open question** User to decide: option 2/3/4/5 or new direction. Any code change needs new branch (current chore/state-phase4-rag-fix is docs-only; redesign/phase3-training merged + deleted).

## Controlled experiments 2026-05-02 (settle "is the LoRA broken or the pipeline broken?")

User pushed back: "we changed too many variables, change one thing at a time." Set up A/B/C:

- **Exp A** (LoRA-only A/B, no resolver, no schema): v6.2 emits subject_entity 0%, v7-e3 emits 100%, both ~100% JSON-valid. v7-e3 also faster (14s p50 vs 21s) and more selective (172 vs 319 extractions on 50 docs). LoRA training succeeded.
- **Exp B** (Pipeline only, perfect Opus 4.7 inputs, my-script `q=subj|quote` shape): 56/59 = 95% resolved.
- **Exp C** (LoRA + pipeline, my-script shape): 25/26 = 96%. Matched B → LoRA emits canonical names well enough.
- **Production-shape diagnostic** (`q=subj` + `context=quote`, what `SecMasterClient.ResolveLocalFromQuoteAsync` actually sends): triggers RagSynthesis path. Returns `hypothesis="MCD"` (correct) but `instrumentId=null` and `symbol=null`. DeterministicResolver checks `instrumentId != null` and falls through to NoResolution. **This is why production eval showed 0% — the RAG hypothesis is being thrown away.**

### Real bug — FIXED in PR #198 (merged 2026-05-02 at e38f0c5)
SecMaster's `RagSynthesis` returns the right ticker as `hypothesis` but doesn't materialize an `instrumentId`.

Chosen fix: option 1 (SentinelCollector side). `DeterministicResolver.TryHybridResolveAsync` now calls `GetInstrumentBySymbolAsync(hypothesis)` after `Trim().ToUpperInvariant()` normalization. New trace tags surface the path: `hybrid.stage`, `hybrid.hypothesis`, `hybrid.hypothesis_resolved`, `hybrid_via_hypothesis`. LogInformation on both materialization-success AND hypothesis-present-but-unresolved (so a future SecMaster catalog drift can't silently recreate the original 0%-resolution mystery).

Note 3 (by-symbol 404) was a false alarm — production code uses path-style `/api/instruments/by-symbol/{symbol}` which works; only my experiment script used the broken query form.

Open follow-up (lower priority): SecMaster could materialize the instrumentId on its side in the RagSynthesis branch (option 2). Would centralize the fix and benefit any other client. Not urgent now that the SentinelCollector side handles it.

### Outputs
- /opt/ai-inference/training-data/exp-a-lora-ab.{jsonl,report.md}
- /opt/ai-inference/training-data/exp-b-pipeline-ab-v2.{jsonl,report.md}
- /opt/ai-inference/training-data/exp-c-end-to-end.{jsonl,report.md}
- Scripts: SentinelCollector/scripts/experiment_{a,b,c}_*.py

## Phase 3 summary (cumulative spend $447 Azure + ~70h GPU)
- Take1 (q/k/v/o, r=16): epoch-3 Symbol 25.7% / null-prec 59% — HARD FAIL
- Take2b (q/k/v/o + gate/up/down MLP, r=8 alpha=16): epoch-3 Symbol 34.5% / null-prec **100%** / JSON-valid 99.8% — Symbol still under 90% target but null-precision passes
- Sample analysis: ~half the Symbol "misses" are not actual model failures — subject_entity wording differences matched too strictly; over-extraction relative to gold; Phase 2.1 candidate generator returning wrong top-10 (e.g. CMC vs CMCO)

## Phase 4.1 shadow mode setup (live 2026-04-30)
- New `Extraction__V2Model=sentinel-cove-v7-e3` env var added (lets v2 use a different LoRA than v1)
- VllmClient + sibling clients (Ollama, CpuOllama, LlamaServer) accept `modelOverride` param
- MergedExtractionService passes `_options.V2Model`
- compose.yaml.j2 set: ShadowMode=true, V2EnabledSources__0=fed-rss, V2Model=sentinel-cove-v7-e3, UseV2Pipeline=false (v1 still owns prod), Model=sentinel-cove-v6.2 (unchanged)
- AutoApproveEnabled stays false (D9 hard stop)
- Image rollback tag: sentinel-collector:pre-shadow

## Follow-up ideas surfaced 2026-04-30 (added to plan, not built yet)
1. **RAG-based CandidateGenerator** (~1-2 days): replace SecMaster.SearchInstrumentsAsync bag-of-words text search with a vector embedding RAG over instrument name+aliases+sector. Would address ~half the Symbol misses (CMC vs CMCO type cases) directly without retraining LoRA. Build AFTER Phase 4.1 shadow numbers tell us how big the candidate-generator-quality gap is.
2. **Gemini MCP fallback resolver** (~2 days): when local SecMaster lookup is low-confidence, ask Gemini (Google Finance grounded) to resolve subject -> ticker. Rule 2.5 in DeterministicResolver between hybrid_resolve and NoResolution. Per-call cost ~$0.001, monthly ~$8-15. Build AFTER Phase 4.1 shadow tells us how often we land in NoResolution.
3. **Gemma 4 31B Dense pivot** (~3-4 days, fallback only): if all Qwen LoRA iterations fail. Native structured JSON output. New chat template + quant pipeline integration. Defer until shadow + RAG + Gemini all exhausted.

Order: shadow -> data -> decide between (1), (2), (3) based on what shadow shows.

## Phase 1 summary (Azure spend ~$5.30)
- v7 prompt: 100% JSON-valid, 100% Symbol-correct (resolvable), 100% null-precision on golden corpus
- Architecture doc: `docs/sentinel-v2-architecture.md` (3077w) + Opus 4.6 cross-review
- Schema migration `AddV2ObservationFields` committed (unapplied); 6 nullable cols + 2 CONCURRENT indexes + shadow table
- v7 prompt: `/opt/ai-inference/prompts/sentinel/extract_and_resolve_v7.txt` + variants for fed-speeches/searxng/rss

## VACATION AUTONOMOUS MODE — ENDED 2026-05-02
User returned home; direct chat resumed. Vacation authorizations no longer active by default. Ntfy comms remain wired (mandatory STEP 0 poll on session start, persistent Monitor on `atlas-claude-reply`).

Historical block (kept for reference, not active):
Authorized ops during 2026-04-24 → 2026-05-02 window:
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
topic=atlas-claude-reply last_seen_ts=1777730320 (2026-05-03 ~02:18 UTC) — clear
via=mcp (sentinel-ntfy MCP registered in ~/.claude.json)
persistent Monitor `bybnaj6r1` re-armed 2026-05-03 — fires per user message on atlas-claude-reply

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
As of 2026-05-03T02:00:00Z:
- last commit on main: e38f0c5 fix(sentinel): materialize SecMaster RagSynthesis hypothesis into instrument_id (#198) — PR #199 (chore/state-phase4-rag-fix STATE.md update) merged 2026-05-03T09:26:50Z
- tags: redesign-phase0-complete, redesign-phase1-complete, redesign-phase2-complete, redesign-phase4-rag-fix (all on main)
- active feature branch: redesign/phase4-precision-secmaster (PR-1 of 5 in progress — Lever A + Lever C-server)
- vllm-server: UP, serving sentinel-cove-v6.2 + sentinel-cove-v7-e1/e2/e3 LoRAs (v7-e3 active in V2 shadow)
- sentinel-collector: UP since 2026-05-02T22:59:57Z (fix-applied build)
- secmaster + secmaster-mcp: UP (10.4.1.200:8080 internal-only; no host port)
- gemini-resolver-mcp: UP on :9300 (paid tier confirmed; ~$0.00006/call)
- timescaledb: UP
- AutoApproveEnabled: false (D9 hard stop, unchanged)
- UseV2Pipeline: false (V2 stays shadow; production extractions go through V1)
- V2EnabledSources: [fed-rss] only
- Phase 4 audit verdict (2026-04-24): RED — ~58% wrong Symbol on async Finnhub path. NOT yet re-audited post Phase 4.2; precision regression revealed 2026-05-03 (see "Live alert + open decision").

## Open PRs
- #200 Sentinel V2 precision — PR-1: SecMaster RAG hardening + minScore raise (branch redesign/phase4-precision-secmaster) — awaiting user review

## Phase 4.3 PR portfolio (5 PRs, plan approved 2026-05-03)

| PR | Branch | Service | Levers | Status |
|---|---|---|---|---|
| PR-1 | redesign/phase4-precision-secmaster | SecMaster | A (server RAG hardening) + C-server (minScore 0.5→0.75 + endpoint passthrough + RagStrictMode kill switch) | OPEN as #200 — 179/179 tests, 0 errors/warnings; awaiting review |
| PR-2 | redesign/phase4-precision-collector | SentinelCollector | B (client guards) + C-client + D (Gemini flip) | PENDING PR-1 merge + deploy |
| PR-3 | redesign/phase4-embedding-template | SecMaster | E foundation (prose template + schema migration) | PENDING |
| PR-4 | redesign/phase4-catalog-enrichment | SecMaster | E body (Finnhub enrichment worker) | PENDING |
| PR-5 | redesign/phase4-fred-dedupe | SecMaster + SentinelCollector | E completion (FRED-pollution remediation) | PENDING |

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
| 2026-05-03T10:30:00Z | 4.3.PR-1.impl | general-purpose | Sonnet | Implement Lever A + Lever C-server (RAG hardening, NameTokenizer, NO_MATCH, asset-class filter, minScore 0.5→0.75, RagStrictMode kill switch) | done — 179/179 tests, 0 errors, 0 warnings; agent flagged 2 follow-ups for PR-2 | SecMaster/src/Services/{NameTokenizer.cs,RagService.cs,HybridResolutionService.cs,LocalResolutionResult.cs,ILocalResolver.cs}; SecMaster/tests/Services/* | de4182c |
| 2026-05-03T10:36:00Z | 4.3.PR-1.review | code-reviewer | Sonnet | Validate PR-1 implementation against plan + CLAUDE.md | done — FIX-REQUIRED: appsettings DefaultMinScore not raised; ILocalResolver default still 0.5f; LogDebug→LogInformation hot-path regression; 4th NIT noted | report inlined to supervisor | (review-only) |
| 2026-05-03T10:40:00Z | 4.3.PR-1.fix | supervisor | Opus | Apply 4 validator-surfaced fixes directly (small + targeted) | done — appsettings.json, ILocalResolver.cs, RagService.cs, compose.yaml.j2 | 260830c | 260830c |
| 2026-05-03T10:42:00Z | 4.3.PR-1.push | supervisor | Opus | Push branch + open PR #200 against main | done — PR #200 OPEN | https://github.com/jpansarasa/ATLAS/pull/200 | 260830c |

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

## Branch / tag plan (current)
DONE:
- redesign/phase0-infra → main + tag redesign-phase0-complete
- redesign/phase1-design → main (PR #196) + tag redesign-phase1-complete @ 38a2af9
- redesign/phase2-pipeline → main (PR #197) + tag redesign-phase2-complete @ be79ab2
- redesign/phase3-training → main (PR #198) + tag redesign-phase4-rag-fix @ e38f0c5 (covers Phase 3 LoRA + Phase 4.1 shadow + Phase 4.2 RagSynthesis fix; branch deleted)

OPEN:
- chore/state-phase4-rag-fix (PR #199) — STATE.md update only

NEXT (when precision decision lands):
- redesign/phase4-precision — implement chosen option (asset-class filter / minScore raise / name-overlap guard / etc)
- redesign/phase4-cutover — flip UseV2Pipeline=true after audit ≥ 49/50 + precision green
- redesign/phase5-cleanup — v1 retire + spec + continuous loop
