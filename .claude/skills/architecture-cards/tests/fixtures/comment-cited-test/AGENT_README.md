# CommentCitedTestFixture — architecture [agent read-first]

PURPOSE: source of truth for span identity + routing. NOT a price store.

DATA MODEL + INVARIANTS:
  Span 1—N SourceMapping ; →FK Sector(11)
  INVARIANT identity ⊥ collection: a Span may be identified yet have 0 SourceMappings.

PATHS (distinct code — do not conflate):
  resolve-spans [HTTP · news→SpanService] does: surface→canonical span. does NOT: route to a collector. on-miss: fail-through → review-queue.

RESOLUTION MODEL ("fuzzy proposes, authoritative confirms"):
  local→external PROPOSES → CONFIRM → persist. Empty = "exhausted every lookup".

DISTINCTIONS:
  resolve-spans(news,HTTP) ≠ ResolveBatch(series,gRPC) — different consumers/code.

CROSS-SERVICE: collectors→register(f-a-f); FEEDS: the matrix sector dim.

GOTCHAS: ✗ bulk-preload to fix a miss.

DECISIONS:
  D-1 span-guard: INTENT refuse junk spans at the boundary / PRECOND span fails sanity gate / GUARD SpanGuard.Apply @ src/SpanGuard.cs:5 / TEST SpanGuardTests.Refuses_junk_span_at_boundary

SEE: README.md §Reference.
