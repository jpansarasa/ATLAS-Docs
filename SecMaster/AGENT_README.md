# SecMaster — architecture [agent read-first]

PURPOSE: identity+classification(NAICS/ATLAS-sector)+source-routing. ¬price/obs(atlas_data=cross-DB read-only).

DATA MODEL + INVARIANTS:
  Instrument 1—N SourceMapping · 1—N Alias · 1—1 Embedding(bge-m3,unique(InstrumentId,Model)); →FK AtlasSector(11)
  SignalIdentity = macro handle (FOMC/CPI) ¬Instrument; sector=null always.
  AtlasSectorCode/RollupVersionId = plain cols ¬DB-FK (real FK→atlas_sectors on naics_sector_rollup).
  INV identity⊥coll: 0 SourceMappings→ResolveAsync=SourceResolution{Warning}¬NotFound. "CatalogInstrumentMissingSector" fires whenever AtlasSectorCode=null.
  INV sector-gate⊥asset-class: AtlasSectorCode req only Equity; Indicator/Economic/Rate/Commodity/Currency/Crypto=null-sector valid. ¬gate non-Equity (~87% catalog dropped).
  INV lazy-load: self-seeds from SEEN; ¬bulk-preload.
  INV is_primary: FredCollector is_primary=SOLE trust signal ¬asset_class str. empty FRED=ANOMALOUS.
  INV Embed 1-N: unique(InstrumentId,Model)¬1-1. entity-default="nomic-embed-text" DRIFTS; runtime=bge-m3. ¬infer active model from entity-default.

PATHS (distinct code — do not conflate):
  resolve-entities [HTTP /api/resolve-entities · Sentinel→EntityResolutionService]
    does: NER surface+articleCtx→instr+NAICS+sector; dedup(surface,kind); isolated DbCtx+timeout; self-seeds.
    does NOT: ¬collector ¬ThresholdEngine ¬gates-matrix-entry.
    on-miss: EnqueueForReview(idempotent open(surface,kind); gated ReviewQueueEnabled)+spanErr; candidate=null; batch survives.
  ResolveBatch [gRPC :5001 · ThresholdEngine→ResolutionService]
    does: symbol→best active SourceMapping. RANK: freq/maxLagDays(filter) → PreferCollector SHORT-CIRCUITS sort → [S]OrderByDesc(is_primary)→ThenBy(priority↑)→ThenBy(lag↑).
    does NOT: ¬NER ¬discovery ¬OpenFIGI ¬self-seed.
    on-miss(¬instr)=NotFound; on-miss(instr,0src)=SourceResolution+Warning(identity+sector+naics).
  register [gRPC :5001 · collectors; fire-and-forget]:
    GUARD: Economic/EconomicIndicator only TrustedMacroCollectors; non-macro len<2→reject.
    proto ALIAS_MATCH=3 exists; C# ¬emits → dead-on-wire.

RESOLUTION MODEL ("fuzzy proposes, authoritative confirms"):
  local(.95/.85/.75)→[ticker]secmaster(.95)→edgar(.90)→OpenFIGI[.85-.90]→signal-alias(.90)→discovery-PROPOSES→CONFIRM(OpenFIGI→Finnhub→Gemini)→persist+embed|review-queue.
  UNCONFIRMED→null. NotFound="tried everything" ¬"not in table".
  ContextFactor: articleCtx non-empty∧canonical¬overlaps(suffix-norm)→score=0→conf=0→ALWAYS<MinConf 0.8→DROPPED null¬down-ranked.

DISTINCTIONS:
  resolve-entities(HTTP,EntityResolutionSvc,news) ≠ ResolveBatch(gRPC,ResolutionSvc,series) — different consumers/code.
  symbol-forward(ResolveSymbol) ≠ LookupSource(collector-id-reverse; DORMANT — integration-tests only).
  MATRIX gated by NewsSignalClassifier(signal-dim); resolve-entities=sector-dim only ¬gates-entry.
  FILTER(freq/lag/preferCollector) ≠ SORT(is_primary→priority→lag). PreferCollector=sort-bypass ¬filter.
  ContextFactor=0→DROPPED ≠ "scored lower"; identity/classification ⊥ collection/source-mapping.

CROSS-SERVICE: collectors→register(f-a-f); ThresholdEngine→ResolveBatch(sync); Sentinel→resolve-entities(sync).
  OUT: collectors REST, OpenFIGI, Gemini, Ollama(embed/RAG). FEEDS: ThresholdEngine matrix via sector grounding.
  Ollama-gen resilience: breaker=process-SINGLETON(5 consecutive fails incl client-aborts→open 60s) + in-flight gate(2; cancelled-while-queued never sent) + num_predict cap(256); NO retry on generation — runner computes abandoned generations to completion, so re-sends/unbounded fan-in = self-sustaining saturation.

GOTCHAS:
  ✗ bulk-preload ✗ backfill-rows-to-green ✗ NotFound="not-in-table" ✗ gate-non-Equity-sector
  ✗ Embed-1-1 ✗ trust-entity-default-model ✗ AtlasSectorCode/RollupVersionId-as-FK
  ✗ ALIAS_MATCH-emitted(dead-wire) ✗ Economic-from-untrusted-collector
  ✗ freq/lag/prefer-as-sort ✗ LookupSource-live-prod ✗ ContextFactor=0-as-lower
  ✗ Polly-breaker-in-per-request-policy-selector # fresh state each request = never opens (7b retry-storm bug)

SEE: README.md §Reference (API endpoints, config tables) · Events/src/Events/Protos/secmaster.proto (SecMasterRegistry+SecMasterResolver gRPC contracts) · EntityResolutionService.cs:1104-1115(ContextFactor) · ResolutionService.cs(ranking) · RegistrationService.cs:135-151(EvaluateGuard)
