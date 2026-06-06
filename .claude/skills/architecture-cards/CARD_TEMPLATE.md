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
    does: {what it does}. ¬do: {the negative space — what agent will wrongly assume}.
    miss: {miss contract — NotFound? Warning? enqueue? self-seed?}.
  {repeat per path}

PROCESSING MODEL ("{governing maxim — one quoted phrase}"):
  {cascade as ordered chain with →. Name stages.}
  {terminal semantics: what "empty"/"miss" MEANS — "exhausted everything" ¬ "absent from local table"}

DISTINCTIONS:
  {A}(callerX,transportY) ≠ {B}(callerZ,transportW) — {because…different consumers/code/semantics}.
  {pairs people conflate; each line kills one class of wrong-guess}

CROSS-SERVICE: IN {who→this, sync|f-a-f}; OUT {this→who, sync|f-a-f}; FEEDS: {downstream artifact: matrix cell|alert|digest}.

GOTCHAS: ✗{anti-pattern agent will reach for} ✗{do-NOT-propose-X} ✗{symptom-fix that violates INV}

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
