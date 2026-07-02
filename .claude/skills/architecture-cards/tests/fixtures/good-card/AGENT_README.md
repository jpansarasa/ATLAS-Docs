# GoodCardFixture — architecture [agent read-first]

PURPOSE: source of truth for widget identity + routing. NOT a price store.

DATA MODEL + INVARIANTS:
  Widget 1—N SourceMapping · 1—1 Embedding ; →FK Sector(11)
  INVARIANT identity ⊥ collection: a Widget may be identified yet have 0 SourceMappings; resolution surfaces identity (Warning, not NotFound).
  TRUST: an is_primary SourceMapping is the trust signal — NOT the asset_class string.

PATHS (distinct code — do not conflate):
  resolve-entities [HTTP · news/NER → EntityResolutionService] does: surface→canonical widget. does NOT: route to a collector. on-miss: fail-through → review-queue.
  ResolveBatch [gRPC :5001 · processor → ResolutionService] does: symbol→best SourceMapping. does NOT: NER/discovery. on-miss(no widget)=NotFound; on-miss(0 sources)=Warning carrying identity.

RESOLUTION MODEL ("fuzzy proposes, authoritative confirms"):
  local(exact→fuzzy→vector)→external PROPOSES → CONFIRM → persist OR review-queue. Empty = "exhausted every lookup", NOT "absent from table".

DISTINCTIONS:
  resolve-entities(news,HTTP) ≠ ResolveBatch(series,gRPC) — different consumers/code.

CROSS-SERVICE: collectors→register(f-a-f); processor→ResolveBatch(sync); FEEDS: the matrix sector dim.

GOTCHAS: ✗ bulk-preload to fix a miss · ✗ NotFound≠"not in table".

DECISIONS:
  D-1 frontier-last-resort: INTENT cheap lookups fan out; frontier call is the RARE earned exception / PRECOND all-cheap-failed ∧ genuinely-hard entity / GUARD WidgetGuard.Apply @ src/Services/WidgetGuard.cs:6 / TEST WidgetGuardTests.Should_not_invoke_frontier_when_cheap_path_succeeds

SEE: README.md §Reference · EntityResolutionService.cs · ResolutionService.cs.
