# ROLLOUT — architecture-cards

The skill is additive. Landing the FIRST card for a service creates `AGENT_README.md`,
updates `README.md` with a 1-line pointer, and wires CLAUDE.md. Those structural edits
are the rollout, applied one service per PR. This note is the spec.

## FILE STRUCTURE [AGENT_README.md model]

1. The card lives in `<Service>/AGENT_README.md` — a standalone file an agent opens
   directly, separate from the human-facing `README.md`.
   - Rationale: embedding the card in `README.md` forces every agent read to pull the
     full multi-kilobyte README (card + Reference catalog), defeating the card's tightness.
     A separate `AGENT_README.md` lets an agent load only the dense mental model.
   - ✗ Embedding the card inside `README.md` as a `## ARCHITECTURE` block.
2. `README.md` keeps a 1-line pointer immediately after the H1/elevator line:
   ```
   > 🤖 **Agents:** read **[AGENT_README.md](AGENT_README.md)** first — the dense architecture card.
   ```
3. Density gate: `AGENT_README.md` <= ~1 page / ~55 non-blank lines. Anything exhaustive
   (endpoint tables, config matrices, project-structure trees) stays in `README.md §Reference`.

### Service file layout after rollout

```
<Service>/
├── AGENT_README.md   ← dense card (agent read-first)
└── README.md         ← human doc: H1 + pointer + Reference catalog
```

### README.md skeleton after rollout

```
# {ServiceName}
{one-line elevator}

> 🤖 **Agents:** read **[AGENT_README.md](AGENT_README.md)** first — the dense architecture card.

## Reference
### Overview / Architecture diagram / Features
### API Endpoints   (the catalog)
### Configuration   (the table)
### Project Structure / Development / Deployment / Ports / See Also
```

### AGENT_README.md skeleton

```
# {ServiceName} — architecture [agent read-first]

PURPOSE: …
DATA MODEL + INVARIANTS: …
PATHS (distinct code): …
RESOLUTION MODEL ("maxim"): …
DISTINCTIONS: …
CROSS-SERVICE: … ; FEEDS: …
GOTCHAS: ✗ … · ✗ …
SEE: README.md §Reference · {Code.cs}
```

## CLAUDE.md WIRING

Add a read-first pointer + a per-service HARD_STOP under `## SERVICE_ARCHITECTURE`, so
an agent is routed to the card BEFORE reasoning about a service. Proposed block:

```
## SERVICE_ARCHITECTURE [read-first] [HARD_STOP]
BEFORE reasoning about a service's architecture / API / data-model / resolution flow:
  READ <Service>/AGENT_README.md (the dense agent card) FIRST.
  rationale: the card front-loads negative-space (does-NOT / on-miss / invariants /
             DISTINCTIONS / GOTCHAS) an endpoint catalog can't convey.
✗ guess a service's shape from method names / the endpoint table # read the card
✗ "fix" a symptom by violating a card INVARIANT (e.g. bulk-preload, backfill-to-green)
cards:
  SecMaster:         SecMaster/AGENT_README.md
  ThresholdEngine:   ThresholdEngine/AGENT_README.md
  SentinelCollector + MacroSubstrate: SentinelCollector/AGENT_README.md
  FredCollector:     FredCollector/AGENT_README.md
  FinnhubCollector:  FinnhubCollector/AGENT_README.md
  # … one line per service as its card lands
```

## PRIORITIZED SERVICE LIST

Order by blast radius — services agents most often reason about wrongly / that gate
downstream artifacts go first.

| # | Service(s) | Why prioritized | Status |
|---|---|---|---|
| 0 | **SecMaster** | identity ⊥ collection + fuzzy/authoritative cascade; most-misread service; root of #619 | DONE |
| 1 | **ThresholdEngine** | central processing hub; evaluation cascade; pattern-vs-threshold conflation; calls SecMaster ResolveBatch | DONE |
| 2 | **SentinelCollector + MacroSubstrate** (news->matrix) | spans two dirs; classifier-gates-entry vs sector-is-a-dim is the recurring confusion (#615/#616); matrix FEED | DONE |
| 3 | **FredCollector** | upstream FRED catalog is a SEPARATE table reconciled INTO instruments (the TRUST signal); is_primary source-mapping | DONE |
| 4 | **FinnhubCollector** | live quote/news-sentiment vs stored series; confirm-source in SecMaster cascade | DONE |
| 5 | AlphaVantageCollector, NasdaqCollector, OfrCollector | remaining collectors; mostly ingest->persist (thin RESOLUTION MODEL, but card still names the boundary) | TODO |
| 6 | AlertService, CalendarService | edge consumers; smaller blast radius | TODO |

Each row = one PR: create `AGENT_README.md`, add pointer to `README.md`, add the
service's line to CLAUDE.md `## SERVICE_ARCHITECTURE`. Run the audit before/after per
the skill's Phase 1 / Phase 4 to confirm the gap closes.
