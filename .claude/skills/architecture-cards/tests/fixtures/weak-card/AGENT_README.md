# WeakCardFixture — architecture [agent read-first]

PURPOSE: serves widgets.

DATA MODEL + INVARIANTS: Widget has many SourceMappings.

PATHS: ResolveBatch returns a SourceMapping for a symbol.

RESOLUTION MODEL: looks up locally then externally.

DISTINCTIONS: widgets and gadgets are different.

CROSS-SERVICE: talks to collectors and the processor.

GOTCHAS: be careful with caching.

D-1 no-guard: INTENT gate the expensive call / PRECOND cache-miss / TEST CacheGuardTests.Should_refuse_expensive_call_on_cache_hit
D-2 no-test: INTENT gate the expensive call / PRECOND cache-miss / GUARD CacheGuard.Apply @ src/Services/CacheGuard.cs:6
D-3 dead-guard-live-test: INTENT gate the expensive call / PRECOND cache-miss / GUARD Gone.Method @ src/Removed.cs:10 / TEST CacheGuardTests.Should_refuse_expensive_call_on_cache_hit
D-4 live-guard-dead-test: INTENT gate the expensive call / PRECOND cache-miss / GUARD CacheGuard.Apply @ src/Services/CacheGuard.cs:6 / TEST WeakTests.Only_named_in_source_file
D-5 malformed-guard: INTENT gate the expensive call / PRECOND cache-miss / GUARD CacheGuard.Apply @ src/Services/CacheGuard.cs / TEST CacheGuardTests.Should_refuse_expensive_call_on_cache_hit
D-6 latest-trap: INTENT prefer the LATEST snapshot on read / PRECOND cache-miss / GUARD CacheGuard.Apply @ src/Services/CacheGuard.cs:6

SEE: README.md §Reference.
