# MacroSubstrate.Client

Public contract + DTO for the `macro_observations` write path. Lets collectors depend on the writer interface without pulling EF Core.

## Overview

`MacroSubstrate.Client` defines two types: `IMacroObservationWriter` (the writer contract) and `MacroObservation` (the canonical DTO with D4 invariants validated client-side). Collectors take `IMacroObservationWriter` via DI; the EF-backed implementation lives in the sibling `MacroSubstrate` package, and collectors that prefer a raw-INSERT path can implement the interface themselves and skip EF entirely.

Pairs with `ATLAS.Events.Client` for the `AtlasSectorCode` enum carried on each observation.

## Contract

### `IMacroObservationWriter`

```csharp
Task<bool> WriteAsync(MacroObservation observation, CancellationToken ct = default);
Task<int>  WriteBatchAsync(IEnumerable<MacroObservation> observations, CancellationToken ct = default);
```

- `WriteAsync` returns `true` if inserted, `false` if the idempotency key collided (duplicate skipped — never an error).
- `WriteBatchAsync` returns the count actually inserted. Implementations should batch in chunks appropriate to their transport (EF default: 500).
- Idempotency key: `(source_collector, source_id, observation_time)`. Duplicates are a successful no-op.

### `MacroObservation` (D4 invariants)

Validated by `EnsureValid()` and enforced by DB `CHECK` constraints:

- Exactly one of `ValueNumeric` / `ValueQualitative` is set.
- `Trust` is non-null **iff** the row is qualitative (range `[0.0, 1.0]` enforced at the DB).
- `ObservationTime` must be UTC — local-kind values loud-fail at write time.

`SignalIdentityId` is a soft FK to `atlas_secmaster.signal_identities` — cross-DB, so PostgreSQL can't enforce it. Format: kebab-case text (max 64), matching SecMaster's actual column type.

## Exceptions

| Type | Raised when |
|---|---|
| `MappingVersionLookupUnavailableException` | A read path needs versioned mapping resolution but `IMappingVersionLookup` was not registered (typically: SecMaster connection string absent at startup). |

## Project Structure

```
MacroSubstrate.Client/
  IMacroObservationWriter.cs                       — writer contract
  MacroObservation.cs                              — DTO + EnsureValid()
  MappingVersionLookupUnavailableException.cs      — read-path failure type
```

## Development

Library only; built and tested as part of the consumer projects. No standalone devcontainer.

```bash
dotnet build MacroSubstrate/src/MacroSubstrate.Client/MacroSubstrate.Client.csproj
```

## Deployment

No deployment artifact. Referenced as a project reference from collectors and from the EF-backed `MacroSubstrate` package.

## See Also

- [MacroSubstrate](../MacroSubstrate/README.md) — EF-backed implementation
- [MacroSubstrate.Migrator](../MacroSubstrate.Migrator/README.md) — one-shot migration runner
- [Events](../../../Events/README.md) — `AtlasSectorCode` lives here
