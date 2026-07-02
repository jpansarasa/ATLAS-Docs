# EXEMPLAR — SecMaster architecture card (tight abbrev-dsl)

This is the gold-standard worked instance. Notice how value concentrates in
`does NOT` / `on-miss` / `⊥` independence / conflated `DISTINCTIONS` / `ANTI` —
none of which an endpoint catalog conveys. When generating a new card, match
this density and negative-space depth, not a thinner endpoint summary.

The block below is the card exactly as it leads `SecMaster/README.md`.

---

## ARCHITECTURE — SecMaster   [read-first]

PURPOSE: identity+classification(NAICS/ATLAS-sector)+source-routing. NOT price/obs(atlas_data=cross-DB read-only).

DATA MODEL + INVARIANTS:
  Instrument 1—N SourceMapping · 1—N Alias · 1—1 Embedding(bge-m3,unique(InstrumentId,Model)); ->FK AtlasSector(11)
  SignalIdentity = macro handle (FOMC/CPI) NOT Instrument; sector=null always.
  AtlasSectorCode/RollupVersionId = plain cols NOT DB-FK (real FK->atlas_sectors on naics_sector_rollup).
  INV identity⊥coll: 0 SourceMappings->ResolveAsync=SourceResolution{Warning} NOT NotFound. "CatalogInstrumentMissingSector" fires whenever AtlasSectorCode=null.
  INV sector-gate⊥asset-class: AtlasSectorCode req only Equity; Indicator/Economic/Rate/Commodity/Currency/Crypto=null-sector valid. Never gate non-Equity (~87% catalog dropped).
  INV lazy-load: self-seeds from SEEN; never bulk-preload.
  INV is_primary: FredCollector is_primary=SOLE trust signal NOT asset_class str. empty FRED=ANOMALOUS.
  INV Embed 1-N: unique(InstrumentId,Model) NOT 1-1. entity-default="nomic-embed-text" DRIFTS; runtime=bge-m3. Never infer active model from entity-default.

PATHS (distinct code — do not conflate):
  resolve-entities [HTTP /api/resolve-entities · Sentinel->EntityResolutionService]
    does: NER surface+articleCtx->instr+NAICS+sector; dedup(surface,kind); isolated DbCtx+timeout; self-seeds.
    does NOT: no collector, no ThresholdEngine, does not gate matrix entry.
    on-miss: EnqueueForReview(idempotent open(surface,kind); gated ReviewQueueEnabled)+spanErr; candidate=null; batch survives.
  ResolveBatch [gRPC :5001 · ThresholdEngine->ResolutionService]
    does: symbol->best active SourceMapping. RANK: freq/maxLagDays(filter) -> PreferCollector SHORT-CIRCUITS sort -> [S]OrderByDesc(is_primary)->ThenBy(priority↑)->ThenBy(lag↑).
    does NOT: no NER, no discovery, no OpenFIGI, no self-seed.
    on-miss(no instr)=NotFound; on-miss(instr,0src)=SourceResolution+Warning(identity+sector+naics).
  register [gRPC · collectors; fire-and-forget]:
    GUARD: Economic/EconomicIndicator only TrustedMacroCollectors; non-macro len<2->reject.
    proto ALIAS_MATCH=3 exists; C# never emits -> dead-on-wire.

PROCESSING MODEL ("fuzzy proposes, authoritative confirms"):
  local(.95/.85/.75)->[ticker]secmaster(.95)->edgar(.90)->OpenFIGI[.85-.90]->signal-alias(.90)->discovery-PROPOSES->CONFIRM(OpenFIGI->Finnhub->Gemini)->persist+embed|review-queue.
  UNCONFIRMED->null. NotFound="tried everything" NOT "not in table".
  ContextFactor: articleCtx non-empty AND canonical not-overlapping(suffix-norm)->score=0->conf=0->ALWAYS<MinConf 0.8->DROPPED null NOT down-ranked.

DISTINCTIONS:
  resolve-entities(HTTP,EntityResolutionSvc,news) ≠ ResolveBatch(gRPC,ResolutionSvc,series) — different consumers/code.
  symbol-forward(ResolveSymbol) ≠ LookupSource(collector-id-reverse; DORMANT — integration-tests only).
  MATRIX gated by NewsSignalClassifier(signal-dim); resolve-entities=sector-dim only, does not gate entry.
  FILTER(freq/lag/preferCollector) ≠ SORT(is_primary->priority->lag). PreferCollector=sort-bypass NOT filter.
  ContextFactor=0->DROPPED ≠ "scored lower"; identity/classification ⊥ collection/source-mapping.

CROSS-SERVICE: collectors->register(f-a-f); ThresholdEngine->ResolveBatch(sync); Sentinel->resolve-entities(sync).
  OUT: collectors REST, OpenFIGI, Gemini, Ollama(embed/RAG). FEEDS: ThresholdEngine matrix via sector grounding.

GOTCHAS: ✗bulk-preload ✗backfill-rows-to-green ✗NotFound="not-in-table" ✗gate-non-Equity-sector
  ✗Embed-1-1 ✗trust-entity-default-model ✗AtlasSectorCode/RollupVersionId-as-FK
  ✗ALIAS_MATCH-emitted(dead-wire) ✗Economic-from-untrusted-collector
  ✗freq/lag/prefer-as-sort ✗LookupSource-live-prod ✗ContextFactor=0-as-lower

SEE: README §Reference (API endpoints, config tables) · secmaster.proto · EntityResolutionService.cs:1104-1115(ContextFactor) · ResolutionService.cs(ranking) · RegistrationService.cs:135-151(EvaluateGuard)

---

## Why this card is the gold standard (annotation — not part of the card)

- **PURPOSE carries a NOT.** "NOT price/obs" kills the single most common wrong assumption up front.
- **INV identity⊥coll named and consequenced.** 0 SourceMappings = Warning, not NotFound. That was a real root cause (#619).
- **Two same-shaped verbs pulled apart.** resolve-entities (HTTP, news, NER, self-seeds) vs ResolveBatch (gRPC, series, no discovery) are different code and different consumers. DISTINCTIONS names the trap.
- **The cascade has a maxim.** "fuzzy proposes, authoritative confirms" compresses the whole resolution model into one durable phrase; "NotFound='tried everything' NOT 'not in table'" pre-empts the recurring misread.
- **GOTCHAS are imperative do-NOTs.** Each ✗ is an anti-pattern an agent has actually reached for.
- **Terse notation.** NOT/-> plus the audit-greppable ⊥/≠ compress without losing load-bearing precision.
