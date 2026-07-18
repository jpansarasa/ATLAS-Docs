# SecMaster — architecture [agent read-first]

PURPOSE: identity+classification(NAICS/ATLAS-sector)+source-routing. Not price/obs (atlas_data=cross-DB read-only).

DATA MODEL + INVARIANTS:
  Instrument 1-N SourceMapping · 1-N Alias · 1-1 Embedding(bge-m3,unique(InstrumentId,Model)); ->FK AtlasSector(11)
  SignalIdentity = macro handle (FOMC/CPI), not an Instrument; sector=null always.
  AtlasSectorCode/RollupVersionId = plain cols, not DB-FK (real FK->atlas_sectors on naics_sector_rollup).
  INV identity⊥coll: 0 SourceMappings->ResolveAsync=SourceResolution{Warning} not NotFound. "CatalogInstrumentMissingSector" fires whenever AtlasSectorCode=null.
  INV sector-gate⊥asset-class: AtlasSectorCode req only Equity; Indicator/Economic/Rate/Commodity/Currency/Crypto=null-sector valid. Never gate non-Equity (~87% catalog dropped).
  INV asset-class-canonical: asset_class stored as ONE canonical spelling, normalized at the persist boundary (D-5); consumers compare by exact value. In-app Equity sector gate (InstrumentMeetsSectorGate) is already OrdinalIgnoreCase — the case divergence bit value-exact consumers (ops gap query, SearchAsync filter, AV-promote), not that gate.
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
  Gen resilience: breaker=process-SINGLETON(5 consecutive SERVER faults [5xx/conn]->open 60s; SERVER-fault-ONLY — our budget/caller cancels [TaskCanceledException] are NOT counted, they are bounded by the RAG budget CTS not the breaker, else healthy-but-slow flips to a 60s hard outage that flaps+floods) + in-flight gate(4 = runner --parallel slots; cancelled-while-queued never sent) + n_predict cap(64); NO retry on generation — runner computes abandoned non-streaming generations to completion, so re-sends/unbounded fan-in = self-sustaining saturation.
  RAG latency budget (measured 2026-06-11 on llama-cpu-rag, embed island pegged): p50 4.8s solo / 8.7s @c2 / 11.6s @c4 (max 12.5s), prefill ~150 tok/s (~3.3s of solo wall = CPU floor for dense 7b), decode ~24 tok/s (19 @c4) -> ctx cap 600 est-tokens (per-candidate split, snippets clipped, not candidates dropped) + one-sentence answer (21-30 tok) + 50s budget (~4x the c4 worst-burst above; headroom for the 7b CPU runner under SUSTAINED 4-slot queuing — the prior 25s ~2x-bench was too tight once production queue-wait stacks above the bench); queued-budget guard: if <RagMinGenerationBudgetSeconds(default 18s; invariant min-budget<timeout) remain after vector search, abstain(rag_degraded{reason=insufficient_budget}), never send the doomed generation. Budget>solo safe: gate caps fan-in, breaker bounds persistent failure. Throughput ~21 gen/min at gate 4 covers historical 17/min storms.

GOTCHAS:
  ✗ bulk-preload ✗ backfill-rows-to-green ✗ NotFound="not-in-table" ✗ gate-non-Equity-sector
  ✗ Embed-1-1 ✗ trust-entity-default-model ✗ AtlasSectorCode/RollupVersionId-as-FK
  ✗ ALIAS_MATCH-emitted(dead-wire) ✗ Economic-from-untrusted-collector
  ✗ freq/lag/prefer-as-sort ✗ LookupSource-live-prod ✗ ContextFactor=0-as-lower
  ✗ send-non-company-to-Gemini (D-1: PAID last-resort; ungated=$100-drain 2026-06-30) [[INTENT_FIDELITY]] [[GIGO]]
  ✗ compare asset_class case-sensitively in a NEW consumer w/o canonicalizing input (D-5: canonical at write, but harden — see SearchAsync/AV-promote) ✗ add a class to AssetClassNormalizer registry to MERGE two classes (case-only; merging=schema decision)
  ✗ Polly-breaker-in-per-request-policy-selector # fresh state each request = never opens (7b retry-storm bug)

DECISIONS:
  D-1 gemini-last-resort-earned: INTENT frontier Gemini = PAID RARE exception, EARNED only when all cheap sources (OpenFIGI->Finnhub) failed on a plausibly-hard company name — THERE it pays dividends; ungated call-on-miss = #823 drain ($100/6d) + silent wrong-ticker corruption / PRECOND kind==CompanyName AND plausible-name (no money, no abbrev(EPS,IPO...), no markup, no code-slug, no boilerplate); defense-in-depth: gemini-resolver (host systemd :9300, outside repo) re-gates server-side (_company_gate) + fail-closed daily CALL cap / GUARD IdentifierConfirmationService.ShouldResolveViaGemini @ src/Services/IdentifierConfirmationService.cs:229 / TEST IdentifierConfirmationServiceTests.junk_surface_never_reaches_the_paid_gemini_api
  D-2 fuzzy-proposes-authoritative-confirms: INTENT discovery top-match = PROPOSAL only; resolves+self-seeds ONLY when an authoritative source grounds it in a real FIGI/ticker; unconfirmed->null, never floored-score (old flat 0.50 = 51%-exhausted bug; wrong-neighbour WULF->USAF would corrupt catalog+matrix) / PRECOND confirmation.Confirmed before resolve or persist / GUARD EntityResolutionService.TryResolveViaUpstreamDiscoveryAsync @ src/Services/EntityResolutionService.cs:877 / TEST EntityResolutionServiceTests.should_keep_candidate_null_when_confirmation_returns_unconfirmed_anti_hallucination — RED-on-delete held JOINTLY w/ exhaustive-switch backstop (:889 default-throw->catch :922-930 degrades null): single-line guard delete keeps boundary outcome BY DESIGN (layered defense); test REDs on the entry's real failure mode (floored-score/self-seeded resolution), not guard-line presence
  D-3 notfound-means-exhausted: INTENT NotFound = "tried everything", not "not-in-table" — the contract that lets callers stop at <=1 SecMaster + 1 external call; a true miss = SIGNAL -> review queue (idempotent surface+kind), never silent-null / PRECOND every cascade leg missed incl confirm-cascade+Gemini / GUARD EntityResolutionService.EnqueueForReviewAsync @ src/Services/EntityResolutionService.cs:659 / TEST EntityResolutionServiceTests.should_enqueue_to_review_queue_when_full_fail_through_exhausted_incl_gemini
  D-4 register-ingress-trust-gate: INTENT catalog integrity at register ingress — LLM extraction can never legitimately publish macro data (LoRA-era: 365 hallucinated Economic rows Dec25–May26); reject at the SOURCE, never backfill-clean later / PRECOND Economic|EconomicIndicator => collector in TrustedMacroCollectors AND non-macro symbol len>=2 / GUARD RegistrationService.EvaluateGuard @ src/Services/RegistrationService.cs:149 / TEST RegistrationServiceGuardTests.should_reject_economic_indicator_from_untrusted_collector
  D-5 asset-class-canonical-at-write: INTENT one canonical spelling per asset_class so value-exact consumers never split on case — a lowercase `equity` is invisible to the ops "Equity missing sector" gap query (asset_class='Equity'), dropped by the catalog SearchAsync filter, mis-typed by AV-promote; case-variants (equity/Equity etf/ETF commodity/Commodity) born upstream at SentinelCollector SecMasterClient .ToLowerInvariant + Gemini lowercase DTO. Clean at the persist choke (covers register/HTTP-admin/config/self-seed/FRED-reconciliation + future callers), migration NormalizeAssetClassCase canonicalizes existing rows; CASE-ONLY, never merges classes (long-tail fred_series/fx/treasury/Interest Rates/Monetary/Credit/Economic Indicator = OPEN taxonomy Q). Also REJECTS blank/whitespace/null LOUDLY (GIGO): NOT NULL admits "" and EvaluateGuard has no emptiness check, so a blank class would persist a row invisible to every asset_class='X' consumer — refuse at the source, never store it / PRECOND every InstrumentEntity persist (Added|Modified) supplies a NON-BLANK class; blank/whitespace/null throws InvalidOperationException{symbol,state} / GUARD AssetClassNormalizer.ApplyTo @ src/Data/AssetClassNormalizer.cs:92 (wired SecMasterDbContext.SaveChanges[Async]) / TEST AssetClassCanonicalizationTests.should_treat_a_lowercase_equity_identically_to_a_canonical_one_in_the_equity_sector_gap_query + should_reject_a_blank_or_null_asset_class_at_the_persist_boundary

SEE: README.md §Reference (API endpoints, config tables) · Events/src/Events/Protos/secmaster.proto (SecMasterRegistry+SecMasterResolver gRPC contracts) · EntityResolutionService.cs:1180(ContextFactor) · ResolutionService.cs(ranking) · RegistrationService.cs:149-165(EvaluateGuard)
