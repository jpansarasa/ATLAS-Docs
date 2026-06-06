# MacroSubstrate — architecture [agent read-first]

PURPOSE: shared EF write+read LIBRARY for `atlas_data.macro_observations` (canonical macro-obs store) + one-shot Migrator. ¬a running service: no listener, no ports, no Program.cs main, ¬owns-projector.

DATA MODEL + INVARIANTS (schema does NOT enforce):
  D4 value-XOR: exactly-one(ValueNumeric ⊻ ValueQualitative); Trust non-null IFF qualitative. EnsureValid() client-side + DB CHECK (`ck_macro_obs_value_xor`, `ck_macro_obs_trust_only_qualitative`).
  INV idem-heal: write = INSERT…ON CONFLICT(source_collector,source_id,observation_time) DO **UPDATE** (heal-on-rewrite, last-write-wins) ¬DO-NOTHING. Top-level README says "DO NOTHING" — STALE; code does guarded DO UPDATE.
  INV unchanged-skip: DO UPDATE guarded by IS-DISTINCT-FROM on value-bearing cols → byte-identical re-collect writes NOTHING; `WriteAsync→false` = unchanged-dup ¬error. id + idem-key cols never overwritten; ingestion_time refreshed on heal.
  INV order-obligation: heal has NO vintage/recency/trust gate → producers MUST deliver same idem-key in observation-order; out-of-order lets older value win. No arbitration column exists.
  INV soft-FK: SignalIdentityId (kebab text ≤64) + AtlasSectorCode → cross-DB `atlas_secmaster` → PG ¬enforces. owner-rec said UUID; actual col = text.
  INV SourceCollector = text ¬enum (new collector ¬needs migration).
  ⊥ versioned-read⊥SecMaster-conn: SecMaster conn OPTIONAL at registration; absent → write paths fine, as-of read paths throw `MappingVersionLookupUnavailableException`.
  ⊥ write⊥version-lookup: IMappingVersionLookup nullable-injected; only as-of READ consults it.

PATHS (distinct code — do not conflate):
  WriteAsync / WriteBatchAsync [in-proc DI · collectors fred/ofr/sentinel via IMacroObservationWriter]
    do: EnsureValid → parameterized upsert (heal-on-rewrite); batch = per-row single writes (one conflict ¬poisons siblings); return inserted/healed count.
    does NOT: ¬HTTP ¬gRPC ¬transport ¬batch-SaveChanges ¬row-version-churn-on-unchanged.
    on-miss: unchanged-dup → false (Debug log, no throw); D4 violation → InvalidOperationException pre-DB.
  QueryAsync / GetObservationsAtDateAsync [in-proc DI · read consumers — ThresholdEngine ObservationCellProjector, Reports.Substrate]
    do: bounded filter (signal/collector/sector/kind/time/minTrust/version|as-of); DefaultLimit=200, hard Take(Limit≤1000); ordering: Descending=false → observation_time ASC + id ASC (chronological default), Descending=true → observation_time DESC + id DESC (newest-first, e.g. WS3 ObservationCellProjector); AsNoTracking.
    does NOT: ¬compute-cells ¬project-matrix ¬own-the-projector (that = ThresholdEngine).
    on-miss: as-of pre-history (no rollup covers date) → EMPTY (contract, INFO ¬WARN); as-of ∧ ¬version-lookup → MappingVersionLookupUnavailableException; null-label rows EXCLUDED from as-of slice.
    guard: AsOfDate + MappingVersionLabel both set → ArgumentException (caller bug); filter.EnsureValid() enforces mutual exclusion pre-query.
  GetByIdempotencyKeyAsync [IMacroObservationRepository · point-lookup · tests + downstream confirmation]
    do: single-row fetch by (sourceCollector, sourceId, observationTime) — the natural idempotency key; returns MacroObservationEntity? (null = not yet written).
    does NOT: ¬paginated ¬filtered ¬versioned.
    use: test assertions that a specific write landed; downstream services confirming a particular obs is present.
  MigrateAsync [one-shot console `migrate-macro-substrate` · no listener]
    do: GetPending → MigrateAsync → exit 0; gates dependent collectors via `depends_on: service_completed_successfully`.
    does NOT: ¬long-running ¬serves-traffic. on-miss: missing conn-str | migration err → exit 1 (loud-fail, full exception).

PROCESSING MODEL: none. Stateless pass-through repo + design-time migrator. Retry envelopes only: runtime DbContext EnableRetryOnFailure(5x/10s); Migrator (10x/15s, cold-start).

DISTINCTIONS:
  MacroSubstrate(stores raw macro_observations) ≠ ObservationCellProjector(reads them, writes matrix_cells) — projector is a ThresholdEngine BackgroundService, NOT here. MacroSubstrate has NO worker.
  this-card ≠ SentinelCollector news→matrix card: Sentinel WRITES obs here (source_collector=sentinel, ExtractionJobId set); the news→matrix PROJECTION is ThresholdEngine's `ObservationCellProjector` reading `IMacroObservationRepository`.
  heal-on-rewrite (DO UPDATE, last-write-wins) ≠ DO NOTHING (frozen first-write). Code = DO UPDATE.
  MinTrust floor narrows ONLY qualitative; numeric rows (Trust=null) ALWAYS pass irrespective of floor.
  GetObservationsAtDateAsync(version-equality slice) ≠ GetByIdRangeAsync(surrogate-id cursor, no version filter) ≠ GetByIdempotencyKeyAsync(single-row point-lookup by natural key — no version, no pagination).
  QueryAsync(AsOfDate + MappingVersionLabel simultaneously) → ArgumentException (mutual-exclusion guard in filter.EnsureValid()); only one versioning axis may be specified.
  AtlasData conn (write target) ≠ SecMaster conn (version-label read only).

CROSS-SERVICE: FredCollector/OfrCollector/SentinelCollector → WriteAsync(in-proc IMacroObservationWriter). ThresholdEngine ObservationCellProjector → QueryAsync(in-proc IMacroObservationRepository). SecMaster conn (OPTIONAL, read-only) → version-label lookup for as-of reads. FEEDS: macro_observations → ThresholdEngine matrix projection.

GOTCHAS:
  ✗ "MacroSubstrate owns the projector / runs news→matrix" — ThresholdEngine does.
  ✗ trust top-level README "ON CONFLICT DO NOTHING" — actual = guarded DO UPDATE.
  ✗ WriteAsync=false treated as error — = unchanged-dup no-op.
  ✗ out-of-order same-idem-key writes — older value wins (no vintage gate).
  ✗ expect as-of read to error on pre-history — returns EMPTY by contract.
  ✗ versioned read without SecMaster conn — throws MappingVersionLookupUnavailableException.
  ✗ set both AsOfDate and MappingVersionLabel on QueryAsync filter — ArgumentException from filter.EnsureValid(); pick one versioning axis.
  ✗ skip OTEL registration — spans + `macro_substrate.mapping_version_lookup_failures_total` silently dropped.
  ✗ treat SignalIdentityId/AtlasSectorCode as PG-enforced FK — cross-DB soft FK.
  ✗ audit unlabeled rows by scanning manually — call `GetObservationCoverageByVersionAsync(ct)` on IMacroObservationRepository; returns `IReadOnlyList<MappingVersionCoverage>` (record: `string? VersionLabel`, `long RowCount`) ordered by descending count; null-label bucket = rows that missed version-stamping and are excluded from every as-of slice.

SEE: README.md §Schema/Config/Contract (cols, CHECK, conn keys) · src/MacroSubstrate.Client/MacroObservation.cs (D4 EnsureValid) · src/MacroSubstrate/Data/Repositories/MacroObservationRepository.cs:132-212 (heal-on-rewrite upsert) · src/MacroSubstrate/Versioning/NpgsqlMappingVersionLookup.cs (as-of label) · src/MacroSubstrate/DependencyInjection.cs (AddMacroSubstrate; optional SecMaster) · ThresholdEngine/src/Workers/ObservationCellProjector.cs:208 (PROJECTOR OWNER — reads IMacroObservationRepository).
