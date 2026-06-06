# WeakCardFixture

A read-first card with all block labels present, but no negative-space: no does-NOT,
no on-miss, no invariant. Catalog-without-model dressed as a card.

## ARCHITECTURE — WeakCardFixture   [read-first]
PURPOSE: serves widgets.
DATA MODEL + INVARIANTS: Widget has many SourceMappings.
PATHS: ResolveBatch returns a SourceMapping for a symbol.
RESOLUTION MODEL: looks up locally then externally.
DISTINCTIONS: widgets and gadgets are different.
CROSS-SERVICE: talks to collectors and the processor.
GOTCHAS: be careful with caching.
SEE: README §Reference.

## Reference
### API Endpoints
| Method | Path | Description |
|---|---|---|
| `GET` | `/api/widgets` | list |
| `POST` | `/api/resolve` | resolve |
### Configuration
| Key | Description | Default |
|---|---|---|
| `Db:Conn` | connection | none |
| `Api:Key` | api key | none |
