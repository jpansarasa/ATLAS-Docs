# Template: recon / measurement

The largest class: standalone investigation, plus the read-only evidence-gathering that feeds a
PR review. Also where agents hand-roll most — a differential harness, a bespoke census, direct
sidecar probes — usually when the answer was already reachable from existing tooling.

Narrower relative: `claim-verification.md`, for checking one report's claims. Use that when the
question is "is this report true", this when the question is "what is actually happening".

```
RECON — read-only. Answer one question with evidence: **{the question, stated as a question}**

Make NO code, config, DB or container changes. SELECT-only. No commits, no branches, no push, no
restarts, no live paid-API calls, no GPU benchmark runs without saying so first (vLLM is production
extraction). No interactive commands; sudo is passwordless.

TRAJECTORY
1. ANCHOR: run `date -u +%FT%TZ` and put the value at the top of the write-up. Every window is
   relative to that ACTUAL instant. A guessed hour shifts a [24h] range and undercounts ~2x.
2. POPULATIONS TABLE, before any number: one row per source — {tag, what it is, what it
   structurally CANNOT contain}. A sample that excludes its own counter-examples reads as
   confirmation, so this is the error that survives review. Known traps:
   the entity_resolution_review_queue holds only FAILURES; the resolver journal is failure-only;
   a census of what REACHED a destination cannot show what was dropped before it.
3. CHECK IT ALREADY EXISTS FIRST. Before writing a harness, look: `.claude/skills/*/scripts/`,
   `scripts/sentinel-quality-check/`, `SentinelCollector/scripts/`, the service MCP tools. The
   capability is usually present and unused.
4. Read the relevant `{Service}/AGENT_README.md` DECISIONS block before reasoning about the
   service. The card front-loads does-NOT / on-miss / invariants an endpoint list cannot.
5. Gather per source, tagging every figure with its population tag: Loki/Prometheus via the
   grafana MCP (`query_loki_logs`, `query_prometheus`); DB via
   `sudo nerdctl exec timescaledb psql -U ai_inference -d atlas_data`, SELECT only; code cited at
   file:line under a NAMED sha, saying whether that sha is what production actually runs.
6. BUILD THE FUNNEL with numbers, stage by stage, and name the biggest single drop.
7. Try ONCE to falsify each of your own "X cannot happen" / "this is the only path" claims. Those
   are the ones that have been wrong.
8. Full write-up to `/tmp/sentinel-remediation/{slug}/`. Return ONLY a <=25 line summary.

FAILURE MODES SEEN HERE -> THE CHECK
- `{severity_text=~"Error|Fatal"}` returns EMPTY and reads as "zero errors". `severity_text` is
  OTLP structured metadata here, NOT an indexed stream label, so it cannot appear in the stream
  selector. Correct form: `{service_name=~".+"} | severity_text=~"Error|Fatal"`.
  RE-VERIFY: `list_loki_label_names` returned exactly `["service_name"]` on 2026-08-06. Run it;
  if that set has grown, the label may now be indexed and the selector form may work.
- An Error-only view misses the incident. Sustained degradation logs at Warning while Error stays
  near zero, so the loudest problem on the stack can be invisible to an Error filter — a 100:1
  Warning-to-Error ratio is unremarkable for a degraded service. Check both levels before calling
  it calm.
- An empty instant query on a freshly restarted cumulative counter is a NULL sample, not a gap.
  Check: range query, or compare against a working service, before calling it broken.
- Some services log compact text (`[ERR]`), not JSON. Check: `severity_text` via Loki as ground
  truth, or grep both patterns.
- Measuring quality only on the rows that survived excludes the population the question is about.
  Check: step 2 forces you to write this down.

Report (<=25 lines): {the answer, the funnel with numbers, the deciding evidence, what you could
NOT establish and why}. "The problem is elsewhere" is a legitimate and often correct answer.
```

## Notes for the supervisor

- Give it ONE question, in bold. A brief with several questions returns an inventory, because
  nothing in it forces a conclusion; a single question has to be answered or declared unanswerable.
- Every mechanism, threshold and count in the brief is a hypothesis. Say so; agents correcting the
  brief on evidence is the expected outcome.
- Do not read the write-up into supervisor context — that is what the <=25 line summary is for.
