# DecisionsNoneBareFixture — architecture [agent read-first]

PURPOSE: source of truth for gadget identity + routing. NOT a price store.

DATA MODEL + INVARIANTS:
  Gadget 1—N SourceMapping ; →FK Sector(11)
  INVARIANT identity ⊥ collection: a Gadget may be identified yet have 0 SourceMappings.

PATHS (distinct code — do not conflate):
  resolve-gadgets [HTTP · news→GadgetService] does: surface→canonical gadget. does NOT: route to a collector. on-miss: fail-through → review-queue.

RESOLUTION MODEL ("fuzzy proposes, authoritative confirms"):
  local→external PROPOSES → CONFIRM → persist. Empty = "exhausted every lookup".

DISTINCTIONS:
  resolve-gadgets(news,HTTP) ≠ ResolveBatch(series,gRPC) — different consumers/code.

CROSS-SERVICE: collectors→register(f-a-f); FEEDS: the matrix sector dim.

GOTCHAS: ✗ bulk-preload to fix a miss.

DECISIONS:none

SEE: README.md §Reference.
