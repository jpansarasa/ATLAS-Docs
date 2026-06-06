# WeakCardFixture — architecture [agent read-first]

PURPOSE: serves widgets.

DATA MODEL + INVARIANTS: Widget has many SourceMappings.

PATHS: ResolveBatch returns a SourceMapping for a symbol.

RESOLUTION MODEL: looks up locally then externally.

DISTINCTIONS: widgets and gadgets are different.

CROSS-SERVICE: talks to collectors and the processor.

GOTCHAS: be careful with caching.

SEE: README.md §Reference.
