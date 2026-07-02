# SecMaster — architecture [agent read-first]

PURPOSE: identity+classification(NAICS/ATLAS-sector)+source-routing. Not price/obs (atlas_data=cross-DB read-only).

DATA MODEL + INVARIANTS:
  Instrument 1-N SourceMapping · 1-N Alias · 1-1 Embedding(bge-m3,unique(InstrumentId,Model)); ->FK AtlasSector(11)
  SignalIdentity = macro handle (FOMC/CPI), not an Instrument; sector=null always.
  AtlasSectorCode/RollupVersionId = plain cols, not DB-FK (real FK->atlas_sectors on naics_sector_rollup).
  INV identity⊥coll: 0 SourceMappings->ResolveAsync=SourceResolution{Warning} not NotFound. "CatalogInstrumentMissingSector" fires whenever AtlasSectorCode=null.
  INV sector-gate⊥asset-class: AtlasSectorCode req only Equity; Indicator/Economic/Rate/Commodity/Currency/Crypto=null-sector valid. Never gate non-Equity (~87% catalog dropped).
  INV lazy-load: self-seeds from SEEN; never bulk-preload.
  INV is_primary: FredCollector is_primary=SOLE trust signal, not asset_class str. empty FRED=ANOMALOUS.
  INV Embed 1-N: unique(InstrumentId,Model), not 1-1. entity-default="nomic-embed-text" DRIFTS; runtime=bge-m3. never infer active model from entity-default.

PATHS (distinct code — do not conflate):
  resolve-entities [HTTP /api/resolve-entities · Sentinel->EntityResolutionService]
    does: NER surface+articleCtx->instr+NAICS+sector; dedup(surface,kind); isolated DbCtx+timeout; self-seeds.
    does NOT: serve collectors; serve ThresholdEngine; gate matrix entry.
    on-miss: EnqueueForReview(idempotent open(surface,kind); gated ReviewQueueEnabled)+spanErr; candidate=null; batch survives.
  ResolveBatch [gRPC :5001 · ThresholdEngine->ResolutionService]
    does: symbol->best active SourceMapping. RANK: freq/maxLagDays(filter) -> PreferCollector SHORT-CIRCUITS sort -> [S]OrderByDesc(is_primary)->ThenBy(priority asc)->ThenBy(lag asc).
    does NOT: NER; discovery; OpenFIGI; self-seed.
    on-miss(no instr)=NotFound; on-miss(instr,0src)=SourceResolution+Warning(identity+sector+naics).
  register [gRPC :5001 · collectors; fire-and-forget]:
    GUARD: Economic/EconomicIndicator only TrustedMacroCollectors; non-macro len<2->reject.
    proto ALIAS_MATCH=3 exists; C# never emits -> dead-on-wire.
    metadata["alias"]->series_id alias ensure (post-success, best-effort): Symbol gets UPPERCASED + resolution is case-SENSITIVE, so collectors alias the verbatim consumer id (mixed-case OFR mnemonic, FH/{sym} stream id); conflict (alias resolves elsewhere) -> skip+WARN, never steal; failure never fails the registration (outcome appended to Message).

RESOLUTION MODEL ("fuzzy proposes, authoritative confirms"):
  local(.95/.85/.75)->[ticker]secmaster(.95)->edgar(.90)->OpenFIGI[.85-.90]->signal-alias(.90)->discovery-PROPOSES->CONFIRM(OpenFIGI->Finnhub->Gemini)->persist+embed|review-queue.
  UNCONFIRMED->null. NotFound="tried everything", not "not in table".
  GEMINI leg = PAID grounded-search LAST RESORT — company-name->ticker:exchange ONLY, gated BEFORE the call (D-1); skips counted secmaster_entity_resolution_gemini_gate_skipped{reason}. ✗send-every-fallthrough: ungated it drained a $100 prepay in 6d, hid 12d behind a green /health (2026-06-30). [[INTENT_FIDELITY]] [[GIGO]]
  ContextFactor: articleCtx non-empty AND canonical does not overlap(suffix-norm)->score=0->conf=0->ALWAYS<MinConf 0.8->DROPPED null, not down-ranked.

DISTINCTIONS:
  resolve-entities(HTTP,EntityResolutionSvc,news) ≠ ResolveBatch(gRPC,ResolutionSvc,series) — different consumers/code.
  symbol-forward(ResolveSymbol) ≠ LookupSource(collector-id-reverse; DORMANT — integration-tests only).
  MATRIX gated by NewsSignalClassifier(signal-dim); resolve-entities=sector-dim only, does NOT gate entry.
  FILTER(freq/lag/preferCollector) ≠ SORT(is_primary->priority->lag). PreferCollector=sort-bypass not filter.
  ContextFactor=0->DROPPED ≠ "scored lower"; identity/classification indep of collection/source-mapping.

CROSS-SERVICE: collectors->register(f-a-f); ThresholdEngine->ResolveBatch(sync); Sentinel->resolve-entities(sync).
  OUT: collectors REST, OpenFIGI, Gemini, llama-cpu-embed(embeddings), llama-cpu-rag(RAG generation). FEEDS: ThresholdEngine matrix via sector grounding.
  BOTH runners = llama.cpp llama-server since 2026-06-11 — NO ollama container remains (ollama exodus; topology = vLLM GPU + llama.cpp CPU). Generation (llama-cpu-rag, qwen2.5:7b q4_K_M): /completion (raw prompt, no chat template; truncation=stop_type:limit or legacy stopped_limit — `stop:true` even at cap, NOT a signal). Embeddings (llama-cpu-embed, bge-m3): OpenAI-style /v1/embeddings {input}->data[0].embedding; SAME GGUF blob as the retired ollama runner, vectors interchangeable with existing pgvector rows (cosine >=0.999999 on prod texts) -> NO re-embed; model field=informational(one model/process). Llm__* names + ILlmClient = engine-agnostic config seam, not engine-pinned. Gen-swap rationale (rag-cpu-scaleout bench): ollama 0.24/0.30 bundled CPU runner decoded same GGUF at 0.2-1.3 tok/s vs llama.cpp ~24 tok/s (~30x ENGINE gap; HT/contention theory REFUTED, second-order +/-30%).
  Gen resilience: breaker=process-SINGLETON(5 consecutive fails incl client-aborts->open 60s) + in-flight gate(4 = runner --parallel slots; cancelled-while-queued never sent) + n_predict cap(64); NO retry on generation — runner computes abandoned non-streaming generations to completion, so re-sends/unbounded fan-in = self-sustaining saturation.
  RAG latency budget (measured 2026-06-11 on llama-cpu-rag, embed island pegged): p50 4.8s solo / 8.7s @c2 / 11.6s @c4 (max 12.5s), prefill ~150 tok/s (~3.3s of solo wall = CPU floor for dense 7b), decode ~24 tok/s (19 @c4) -> ctx cap 600 est-tokens (per-candidate split, snippets clipped, not candidates dropped) + one-sentence answer (21-30 tok) + 25s budget (~2x worst burst; old 60s was sized for retired ollama runner); queued-budget guard: if <RagMinGenerationBudgetSeconds(default 12s ~ c4 p50) remain after vector search, abstain(rag_degraded{reason=insufficient_budget}), never send the doomed generation. Budget>solo safe: gate caps fan-in, breaker bounds persistent failure. Throughput ~21 gen/min at gate 4 covers historical 17/min storms.

GOTCHAS:
  ✗ bulk-preload ✗ backfill-rows-to-green ✗ NotFound="not-in-table" ✗ gate-non-Equity-sector
  ✗ Embed-1-1 ✗ trust-entity-default-model ✗ AtlasSectorCode/RollupVersionId-as-FK
  ✗ ALIAS_MATCH-emitted(dead-wire) ✗ Economic-from-untrusted-collector
  ✗ freq/lag/prefer-as-sort ✗ LookupSource-live-prod ✗ ContextFactor=0-as-lower
  ✗ send-non-company-to-Gemini (D-1: PAID last-resort; ungated=$100-drain 2026-06-30) [[INTENT_FIDELITY]] [[GIGO]]
  ✗ Polly-breaker-in-per-request-policy-selector # fresh state each request = never opens (7b retry-storm bug)

DECISIONS:
  D-1 gemini-last-resort-earned: INTENT frontier Gemini = PAID RARE exception, EARNED only when all cheap sources (OpenFIGI->Finnhub) failed on a plausibly-hard company name — THERE it pays dividends; ungated call-on-miss = #823 drain ($100/6d) + silent wrong-ticker corruption / PRECOND kind==CompanyName AND plausible-name (no money, no abbrev(EPS,IPO...), no markup, no code-slug, no boilerplate); defense-in-depth: gemini-resolver (host systemd :9300, outside repo) re-gates server-side (_company_gate) + fail-closed daily CALL cap / GUARD IdentifierConfirmationService.ShouldResolveViaGemini @ src/Services/IdentifierConfirmationService.cs:229 / TEST IdentifierConfirmationServiceTests.junk_surface_never_reaches_the_paid_gemini_api
  D-2 fuzzy-proposes-authoritative-confirms: INTENT discovery top-match = PROPOSAL only; resolves+self-seeds ONLY when an authoritative source grounds it in a real FIGI/ticker; unconfirmed->null, never floored-score (old flat 0.50 = 51%-exhausted bug; wrong-neighbour WULF->USAF would corrupt catalog+matrix) / PRECOND confirmation.Confirmed before resolve or persist / GUARD EntityResolutionService.TryResolveViaUpstreamDiscoveryAsync @ src/Services/EntityResolutionService.cs:877 / TEST EntityResolutionServiceTests.should_keep_candidate_null_when_confirmation_returns_unconfirmed_anti_hallucination — RED-on-delete held JOINTLY w/ exhaustive-switch backstop (:889 default-throw->catch :922-930 degrades null): single-line guard delete keeps boundary outcome BY DESIGN (layered defense); test REDs on the entry's real failure mode (floored-score/self-seeded resolution), not guard-line presence
  D-3 notfound-means-exhausted: INTENT NotFound = "tried everything", not "not-in-table" — the contract that lets callers stop at <=1 SecMaster + 1 external call; a true miss = SIGNAL -> review queue (idempotent surface+kind), never silent-null / PRECOND every cascade leg missed incl confirm-cascade+Gemini / GUARD EntityResolutionService.EnqueueForReviewAsync @ src/Services/EntityResolutionService.cs:659 / TEST EntityResolutionServiceTests.should_enqueue_to_review_queue_when_full_fail_through_exhausted_incl_gemini
  D-4 register-ingress-trust-gate: INTENT catalog integrity at register ingress — LLM extraction can never legitimately publish macro data (LoRA-era: 365 hallucinated Economic rows Dec25–May26); reject at the SOURCE, never backfill-clean later / PRECOND Economic|EconomicIndicator => collector in TrustedMacroCollectors AND non-macro symbol len>=2 / GUARD RegistrationService.EvaluateGuard @ src/Services/RegistrationService.cs:149 / TEST RegistrationServiceGuardTests.should_reject_economic_indicator_from_untrusted_collector

SEE: README.md §Reference (API endpoints, config tables) · Events/src/Events/Protos/secmaster.proto (SecMasterRegistry+SecMasterResolver gRPC contracts) · EntityResolutionService.cs:1180(ContextFactor) · ResolutionService.cs(ranking) · RegistrationService.cs:149-165(EvaluateGuard)
