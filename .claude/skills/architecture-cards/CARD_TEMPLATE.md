# ARCHITECTURE — {ServiceName}   [read-first]

PURPOSE: {what it IS} ¬{what it IS NOT — the boundary people assume wrong}.

DATA MODEL + INVARIANTS:
  {Relations: `A 1—N B · 1—1 C ; →FK D`. Compact notation.}
  INV {name}: {rule}. {why it bites if violated}. ¬{the wrong assumption}.
  {⊥ independence facts: `X ⊥ Y` — two things that can coexist without each other.
   These are silent-bug breeding ground; name them explicitly.}
  TRUST: {real trust signal IS vs. what LOOKS like it but isn't}

PATHS ({name each distinct code path; do NOT conflate}):
  {name} [{transport} {port} · {consumer}→{handler}]
    transport-tag REQUIRED: HTTP :8080 | gRPC :5001 | gRPC-stream :5001 | MCP :{port}
    gRPC is ATLAS's PRIMARY internal transport — REST/openapi is NOT the whole API surface.
    does: {what it does}.
    does NOT: {the negative space — what agent will wrongly assume}.
    on-miss: {miss contract — NotFound? Warning? enqueue? self-seed?}.
  {repeat per path}

PROCESSING MODEL ("{governing maxim — one quoted phrase}"):
  {cascade as ordered chain with →. Name stages.}
  {terminal semantics: what "empty"/"miss" MEANS — "exhausted everything" ¬ "absent from local table"}

DISTINCTIONS:
  {A}(callerX,transportY) ≠ {B}(callerZ,transportW) — {because…different consumers/code/semantics}.
  {pairs people conflate; each line kills one class of wrong-guess}

CROSS-SERVICE: IN {who→this, sync|f-a-f}; OUT {this→who, sync|f-a-f}; FEEDS: {downstream artifact: matrix cell|alert|digest}.

GOTCHAS: ✗{anti-pattern agent will reach for} ✗{do-NOT-propose-X} ✗{symptom-fix that violates INV}

DECISIONS:
  D-n {slug}: INTENT {why} / PRECOND {condition} / GUARD {Class.Method} @ {file:line} / TEST {TestClass.TestName}
  {repeat per decision — or replace the whole block with the single line
   `DECISIONS: none — no exception paths` when the service genuinely has none}

SEE: README §Reference · {Code.cs:line(what to read)} · {openapi/proto for exhaustive catalog}
  gRPC contracts: SEE MUST reference the .proto file for any gRPC path (e.g. SEE: Events/src/Events/Protos/observation_events.proto).
  REST contracts: openapi.json or §API Endpoints. Both surfaces must be documented — REST reference ≠ complete API surface.

---
## TEMPLATE NOTATION

Symbols preferred over words:
  ¬   not / does not / never
  ⊥   independent of (two things that can exist without each other)
  →   implies / leads to / flows to
  ≠   distinct from / not the same as
  ×   multiplied / factor

Section priority (load-bearing first, long-tail to SEE):
  1. INV / ⊥ independence facts — the unenforced schema rules that bite
  2. PATHS ¬do + miss — the negative space and miss contract
  3. DISTINCTIONS — conflated pairs
  4. PROCESSING MODEL maxim + terminal semantics
  5. GOTCHAS — named do-not anti-patterns
  6. CROSS-SERVICE — wiring + feeds

Density gate: card ≤ ~1 page / ~55 non-blank lines.
  Exhaustive catalog → README.md §Reference (pointed to by SEE:).
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
explicitly. A brief that contradicts a D-entry without a named supersession → STOP and
report; never route-around, never obey-stale (the entry may be outdated OR the brief
wrong — a human/supervisor decides, not the implementing agent).

Scope discipline (¬everything is a decision):
  ✓ exception paths (frontier last-resort, raw-DB write, privileged op)
  ✓ scarce-resource boundaries ($/GPU/quota)
  ✓ invariants with non-obvious preconditions
  ✗ ordinary mechanism — a service may declare `DECISIONS: none — no exception paths`
  >~6 entries = smell (scope creep dilutes the signal); card stays ≤ ~1 page.

## AUDIT COMPLIANCE — literal labels REQUIRED [HARD_STOP]

`scripts/audit.sh` checks for literal text labels. A generated card MUST include these
exact strings or it will fail audit at generation time.

Required section headings (HIGH severity if absent):
  `DATA MODEL + INVARIANTS:` — the block heading (W3 also requires `INVARIANT ` (full word + space, e.g. `INVARIANT foo:`) or `⊥` inside it; the bare `INV ` abbreviation does NOT satisfy the grep)
  `CROSS-SERVICE:` — including the trailing colon

Required per PATH entry (MEDIUM W1/W2 if absent anywhere in PATHS):
  `does NOT:` — the negative-space anti-guess lever (W1 fires if this literal is absent)
  `on-miss:` — the miss contract (W2 fires if this literal is absent)

The abbrev-DSL forms `¬do:` / `miss:` / `INV:` MAY be used as aliases within a path but
do NOT satisfy the audit grep — the literals `does NOT` and `on-miss` MUST also appear.
Rationale: rollout-2 cards used only the symbols and required 16 post-hoc fixes because
`audit.sh` grep targets the literals, not the symbols.

DECISIONS block (advisory findings — NOT blocks; lint-blocking was rejected, spec §OUT_OF_SCOPE):
  `DECISIONS` literal absent anywhere in the card → MEDIUM W8 (`DECISIONS: none — no
  exception paths` satisfies it).
  Each `D-n` entry missing a `GUARD <class.method> @ file:line` or `TEST <TestClass.TestName>`
  citation → HIGH D_entry_no_citation. The TEST citation must be DOTTED
  (`TestClass.TestName`) — a bare `TEST word` does not count. A malformed GUARD
  (missing `:line`) is reported as MISSING, not as a distinct malformed signal.
  Dead citation — GUARD file missing under the service root, or the TEST method name not
  greppable in the service's test project (*.cs whose path RELATIVE to the service root
  contains *test*) → HIGH D_entry_dead_citation.
