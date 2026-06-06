# NotReadFirstFixture

A complete card, but it is preceded by an `## Overview` H2 — so it is not read-first.

## Overview
Does widget things. This section appears before the card, violating read-first.

## ARCHITECTURE — NotReadFirstFixture   [read-first]
PURPOSE: routing of widgets. NOT a price store.
DATA MODEL + INVARIANTS:
  Widget 1—N SourceMapping ; →FK Sector(11)
  INVARIANT identity ⊥ collection: a Widget may exist with 0 SourceMappings.
PATHS (distinct code):
  ResolveBatch [gRPC :5001 · processor] does: symbol→SourceMapping. does NOT: discovery. on-miss: NotFound.
RESOLUTION MODEL ("fuzzy proposes, authoritative confirms"): local→external. Empty = exhausted.
DISTINCTIONS:
  identity ≠ collection — one can exist without the other.
CROSS-SERVICE: processor→ResolveBatch(sync); FEEDS: the matrix.
GOTCHAS: ✗ bulk-preload.
SEE: README §Reference.
