# Audit Signals

Reference for `scripts/audit.sh`. Each signal has a cheap implementation (grep/awk, no
LLM) and triggers a specific severity. The skill reads this file to know what to check.
The cheap audit verifies a card EXISTS, is READ-FIRST, and has the required blocks; the
deeper "is this block actually negative-space, not a restated catalog?" judgement is an
LLM-only check (LOW, skipped in cheap audit — the subagent enforces it in Phase 3).

## Severity legend

- **CRITICAL** — service has no `AGENT_README.md` at all
- **HIGH** — card exists but missing a required block
- **MEDIUM** — weak card: a required block is present but empty/degenerate — the
  catalog-without-model signals (no `does NOT:`, no `on-miss:`, no `DISTINCTIONS`, etc.)
- **LOW** — block is present but reads as a restated endpoint catalog rather than
  negative-space (LLM-only; skipped in cheap audit)

## Required blocks (HIGH if any missing)

The card lives in `<Service>/AGENT_README.md` — a standalone file an agent opens first,
separate from the human-facing README.md. Required blocks (matched as ALL-CAPS labels
anywhere in the file):

`PURPOSE`, `DATA MODEL + INVARIANTS`, `PATHS`, `RESOLUTION MODEL` (or `PROCESSING`),
`DISTINCTIONS`, `CROSS-SERVICE`, `GOTCHAS`, `SEE`.

A pure collector with no resolution cascade may legitimately have a thin
`PROCESSING/RESOLUTION MODEL` — but the label must still be present (stating "n/a —
straight ingest->persist, no cascade") so the absence is intentional, not forgotten.

## Agent-read-first rule

The card lives in `AGENT_README.md` at the service root — separate from `README.md`.
This ensures agents open a tight card file directly rather than pulling the full
multi-kilobyte README. `README.md` carries a 1-line pointer immediately after its H1:
`> 🤖 **Agents:** read **[AGENT_README.md](AGENT_README.md)** first`.

## MEDIUM-severity weakness signals (catalog-without-model)

| ID | Signal | Cheap implementation |
|----|--------|----------------------|
| W1 | No `does NOT:` anywhere in PATHS | grep card body for `does NOT` (case-insensitive). Absent -> fire. The negative space is the #1 anti-guess lever. |
| W2 | No `on-miss:` anywhere in PATHS | grep card for `on-miss`. Absent -> the miss contract is undocumented. |
| W3 | No invariant / independence fact | grep card for `INVARIANT` or `⊥`. Absent -> schema-unenforced rules undocumented. |
| W4 | DISTINCTIONS block has no `≠` pair | DISTINCTIONS present but no `≠` line -> degenerate; it documents no conflation. |
| W5 | CROSS-SERVICE has no FEEDS / direction | grep card for `FEEDS` or `→`. Absent -> downstream artifact / call direction missing. |
| W6 | Stub card | `AGENT_README.md` has < 12 non-blank lines -> card is a stub. |
| W7 | Card oversized | card body > ~55 non-blank lines -> density lost; the catalog belongs in `README.md §Reference`. |

Each signal that fires emits one finding. A single service can accumulate multiple
findings across signals.

## Output shape (per finding)

~~~json
{
  "path": "{service-dir}",
  "severity": "CRITICAL | HIGH | MEDIUM",
  "block": "{block-name or null}",
  "signal": "{W1..W7 | missing_card | missing_block | not_read_first}",
  "message": "{human-readable detail}"
}
~~~

## Service enumeration rules

The audit set is the canonical `## SERVICES` list in CLAUDE.md (collectors / processing /
alerting / calendar / metadata / substrate), resolved to top-level service dirs, NOT a
filesystem glob — so a new service shows as a CRITICAL gap the moment it's added to
CLAUDE.md, even before its AGENT_README.md exists. Sentinel's news->matrix pipeline spans
two dirs (`SentinelCollector` for news/NER, `MacroSubstrate` for the matrix); both are
in scope.

Excluded: `mcp/`, `config/`, `scripts/` sub-components (they have no architecture of
their own — they front the parent service, whose card covers them); `Events/`,
`deployment/`, `docs/` (shared, not services).

## Catalog-without-model — the failure this skill targets

A service with only a big `## API Endpoints` + `## Configuration` README and no
`AGENT_README.md` is the exact artifact this skill flags. The catalog tells an agent
what it CAN call; it never tells the agent what will bite (two same-verb paths, an
unenforced invariant, the on-miss contract, the cascade maxim, the anti-pattern). The
`AGENT_README.md` model ensures the card is a tight, separately-openable file that an
agent loads at low cost before touching any service.
