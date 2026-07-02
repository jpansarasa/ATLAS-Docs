# AsciiAnchorsFixture — architecture [agent read-first]

PURPOSE: source of truth for gizmo identity + routing. NOT a price store.

DATA MODEL + INVARIANTS:
  Gizmo 1—N SourceMapping ; →FK Sector(11)
  INV identity independent of collection: a Gizmo may be identified yet have 0 SourceMappings.

PATHS (distinct code — do not conflate):
  resolve-gizmos [HTTP · news→GizmoService] does: surface→canonical gizmo. does NOT: route to a collector. on-miss: fail-through → review-queue.

RESOLUTION MODEL ("fuzzy proposes, authoritative confirms"):
  local→external PROPOSES → CONFIRM → persist. Empty = "exhausted every lookup".

DISTINCTIONS:
  resolve-gizmos(news,HTTP) != ResolveBatch(series,gRPC) — different consumers/code.

CROSS-SERVICE: collectors→register(f-a-f); FEEDS: the matrix sector dim.

GOTCHAS: never bulk-preload to fix a miss.

DECISIONS: none — no exception paths

SEE: README.md §Reference.
