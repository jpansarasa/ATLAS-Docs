# ARCHITECTURE — {ServiceName}   [read-first]

PURPOSE: {what it IS} NOT {what it IS NOT — the boundary people assume wrong}.

DATA MODEL + INVARIANTS:
  {Relations: `A 1—N B · 1—1 C ; ->FK D`. Compact notation.}
  INV {name}: {rule}. {why it bites if violated}. NOT {the wrong assumption}.
  {⊥ independence facts: `X ⊥ Y` — two things that can coexist without each other.
   These are silent-bug breeding ground; name them explicitly.}
  TRUST: {real trust signal IS vs. what LOOKS like it but isn't}

PATHS ({name each distinct code path; do NOT conflate}):
  {name} [{transport} {port} · {consumer}->{handler}]
    transport-tag REQUIRED: HTTP :8080 | gRPC :5001 | gRPC-stream :5001 | MCP :{port}
    gRPC is ATLAS's PRIMARY internal transport — REST/openapi is NOT the whole API surface.
    does: {what it does}.
    does NOT: {the negative space — what agent will wrongly assume}.
    on-miss: {miss contract — NotFound? Warning? enqueue? self-seed?}.
  {repeat per path}

PROCESSING MODEL ("{governing maxim — one quoted phrase}"):
  {cascade as ordered chain with ->. Name stages.}
  {terminal semantics: what "empty"/"miss" MEANS — "exhausted everything" not "absent from local table"}

DISTINCTIONS:
  {A}(callerX,transportY) ≠ {B}(callerZ,transportW) — {because…different consumers/code/semantics}.
  {pairs people conflate; each line kills one class of wrong-guess}

CROSS-SERVICE: IN {who->this, sync|f-a-f}; OUT {this->who, sync|f-a-f}; FEEDS: {downstream artifact: matrix cell|alert|digest}.

GOTCHAS: ✗{anti-pattern agent will reach for} ✗{do-NOT-propose-X} ✗{symptom-fix that violates INV}

DECISIONS:
  D-n {slug}: INTENT {why} / PRECOND {condition} / GUARD {Class.Method} @ {file:line} / TEST {TestClass.TestName}
  {repeat per decision — or replace the whole block with a single escape line:
   `DECISIONS: none — no exception paths` when the service was audited and genuinely
   has none, or the not-yet-audited form — see §DECISIONS BLOCK "Two escape forms"}

SEE: README §Reference · {Code.cs:line(what to read)} · {openapi/proto for exhaustive catalog}
  gRPC contracts: SEE MUST reference the .proto file for any gRPC path (e.g. SEE: Events/src/Events/Protos/observation_events.proto).
  REST contracts: openapi.json or §API Endpoints. Both surfaces must be documented — REST reference != complete API surface.

---
## TEMPLATE NOTATION

Plain terse ASCII preferred over unicode operators (measured 2026-07-02: symbols cost
same-or-more tokens than the words AND break literal grep). `audit.sh` accepts the
ASCII forms as W3/W4 anchors, so no glyph is required; glyphs remain accepted for
existing cards — converge to ASCII on touch, never mass-rewrite:
  NOT / never   negation — write the word, not a symbol
  ->            implies / leads to / flows to
  INV {name}:   invariant / independence label — satisfies W3 (or `INVARIANT `; the ⊥ glyph still accepted)
  !=            distinct from / not the same as — satisfies W4 (the ≠ glyph still accepted)
  x             multiplied / factor

Section priority (load-bearing first, long-tail to SEE):
  1. INV / ⊥ independence facts — the unenforced schema rules that bite
  2. PATHS does-NOT + on-miss — the negative space and miss contract
  3. DISTINCTIONS — conflated pairs
  4. PROCESSING MODEL maxim + terminal semantics
  5. GOTCHAS — named do-not anti-patterns
  6. CROSS-SERVICE — wiring + feeds

Density gate: card <= ~1 page / ~55 non-blank lines.
  Exhaustive catalog -> README.md §Reference (pointed to by SEE:).
  Every claim must trace to code, NOT memory.

Omit a block only if the service genuinely has no instance of it; say so explicitly.

## DECISIONS BLOCK — per-service design-decision record [D_ENTRY_SPEC]

Numbered decisions (`D-1`, `D-2`, …). Line format (exact — audit.sh greps it):
  `D-n <slug>: INTENT <why> / PRECOND <condition> / GUARD <class.method> @ file:line / TEST <TestClass.TestName>`
  file:line is relative to the service root (the dir containing AGENT_README.md).

ATOMIC SET (change-all-or-none — same discipline as the `:sig:` infix string contract):
  1. D-entry in the card
  2. `// INTENT(D-n):` comment at the guard site
  3. guard code
  4. guard test

Supersession: rewrite the entry IN THE SAME PR as the code change. No tombstones —
main = current-state, git log = archive. Dispatch briefs must name **"supersedes D-n"**
explicitly.

WHERE A COMPANION EXISTS, SUPERSEDING IS A TWO-FILE EDIT and the second file is the one
that gets forgotten: rewriting the card entry while `DECISIONS.md §D-n` goes on asserting
the retired rule leaves the evidence contradicting the rule it is evidence FOR, and a
reader who follows the DETAIL pointer lands on the stale half. So the atomic set gains a
fifth member wherever the entry carries `/ DETAIL DECISIONS.md §D-n`: **the companion
section is rewritten or deleted in the SAME PR**, and a retired entry leaves BOTH files.
`scripts/verify-card-companion.py` gates the mechanical half (id, slug and GUARD citation
parity, CI-enforced via `scripts/tests/test_verify_card_companion.py`) and reports the
prose half as a non-gating advisory — it cannot check that two paragraphs AGREE, only that
the card names what the companion's rules name, so the reviewer still owns the reading. A brief that contradicts a D-entry — or a rule stated in a skill, a template or
CLAUDE.md — without a named supersession -> STOP and report, NAMING the rule and the
contradiction; never route-around, never obey-stale, and never silently obey a written rule
you believe is stale (the entry may be outdated OR the brief wrong — a human/supervisor
decides, not the implementing agent).

Scope discipline (not everything is a decision):
  ✓ exception paths (frontier last-resort, raw-DB write, privileged op)
  ✓ scarce-resource boundaries ($/GPU/quota)
  ✓ invariants with non-obvious preconditions
  ✗ ordinary mechanism — a service may declare `DECISIONS: none — no exception paths`
  >~6 entries = smell (scope creep dilutes the signal); card stays <= ~1 page.

Entry SIZE, which is the axis that actually broke [measured 2026-09-20]:
  A D-entry is ONE LINE: the rule, its precondition, its citations. The EVIDENCE behind a
  rule -- the measurement that established it, the alternatives rejected, the incident it
  came from -- does NOT belong on the card. Left inline it grows without limit and nothing
  notices, because it grows WITHIN a line: SentinelCollector reached 303,351 bytes across
  34 entries (one of them 32,788 bytes) on 138 non-blank lines, and the line-denominated
  W7 signal read the same as it would for a 20KB card. W9 (bytes) exists for this.
  Past a paragraph, the evidence moves to `<Service>/DECISIONS.md` under a `## D-n <slug>`
  heading and the card entry ends with `/ DETAIL DECISIONS.md §D-n`. The entry keeps the
  RULE and points at the detail; the detail is never deleted, and `§D-n` is gate-checked by
  `scripts/verify-pointers.py`, so the pointer cannot rot silently.
  D-n stays resolvable IN THE CARD either way -- `// INTENT(D-n):` comments, the
  intent-review skill and CLAUDE.md INTENT_FIDELITY all read the card's DECISIONS block, so
  an entry may shed its evidence but must never leave the card for the companion.

Two escape forms — the wording carries a CLAIM, pick the honest one:
  `DECISIONS: none — no exception paths`
    the AUDITED form: the service was reviewed and verifiably has no exception paths.
  `DECISIONS: none recorded yet — accrete on touch (not audited for exception paths; see CLAUDE.md INTENT_FIDELITY MECHANICS).`
    the NOT-YET-AUDITED form: no D-entry sweep has been done; record decisions as the
    service is touched. Never "upgrade" this line to the audited form without doing the sweep.

## AUDIT COMPLIANCE — literal labels REQUIRED [HARD_STOP]

`scripts/audit.sh` checks for literal text labels. A generated card MUST include these
exact strings or it will fail audit at generation time.

Required section headings (HIGH severity if absent):
  `DATA MODEL + INVARIANTS:` — the block heading (W3 also requires an invariant statement: `INVARIANT ` or `INV ` (word + space, e.g. `INV foo:`) or the `⊥` glyph; `INV:` with no space does NOT satisfy the grep)
  `CROSS-SERVICE:` — including the trailing colon

Required per PATH entry (MEDIUM W1/W2 if absent anywhere in PATHS):
  `does NOT:` — the negative-space anti-guess lever (W1 fires if this literal is absent)
  `on-miss:` — the miss contract (W2 fires if this literal is absent)

The abbrev-DSL forms `¬do:` / `miss:` MAY be used as aliases within a path but do NOT
satisfy the audit grep — the literals `does NOT` and `on-miss` MUST also appear.
Rationale: rollout-2 cards used only the symbols and required 16 post-hoc fixes because
`audit.sh` grep targets the literals, not the symbols.

DECISIONS block (advisory findings — NOT blocks; lint-blocking was rejected, spec §OUT_OF_SCOPE):
  `DECISIONS` literal absent anywhere in the card -> MEDIUM W8 (`DECISIONS: none — no
  exception paths` satisfies it).
  Each `D-n` entry missing a `GUARD <class.method> @ file:line` or `TEST <TestClass.TestName>`
  citation -> HIGH D_entry_no_citation. The TEST citation must be DOTTED
  (`TestClass.TestName`) — a bare `TEST word` does not count. A malformed GUARD
  (missing `:line`) is reported as MISSING, not as a distinct malformed signal.
  Dead citation — GUARD file missing under the service root, or the TEST method name not
  greppable in the service's test project (*.cs whose path RELATIVE to the service root
  contains *test*) -> HIGH D_entry_dead_citation.
