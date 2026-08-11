# ARCHITECTURE — gemini-resolver-mcp   [read-first]

PURPOSE: the LAST-RESORT paid confirmer — one company NAME -> one US-listed ticker:exchange via
Gemini grounded search. NOT a search service, NOT a catalog, NOT an entity resolver: it stores no
instruments and answers exactly one question, for callers that already exhausted the free ones.
A HOST systemd unit (`gemini-resolver-mcp.service`, :9300, WorkingDirectory=~/ATLAS/gemini-resolver-mcp),
NOT a compose service — ansible does not deploy the CODE, `nerdctl` cannot see it, and there is no
build step, so a `git pull` on the MAIN checkout reaches production on the unit's next restart.
Its ALERT RULES are a separate, ansible-managed artefact
(deployment/artifacts/monitoring/alerts/gemini-resolver.yml, shipped by
`--tags monitoring --skip-tags always`, which copies and HUPs Prometheus), so "not ansible-deployed"
is true of the process and FALSE of its monitoring: a code-only pull leaves every rule below
un-deployed. Despite the name it speaks HTTP, not MCP.

DATA MODEL + INVARIANTS:
  two SQLite files, separate on purpose: result cache (GEMINI_CACHE_DB) · call ledger
  (GEMINI_LEDGER_DB) -> the rolling window CallStats reconstructs at startup
  INV rolling-not-midnight: DAILY_CALL_CAP bounds a ROLLING 24h window, events aging out by
    timestamp. NOT a UTC-midnight reset — "today's usage" is not a thing here.
  INV cap-survives-restart: the window is rebuilt FROM the ledger, so a restart is not a cap
    reset. It WAS one before 2026-08-06 (3 of 12 restarts erased a saturated cap; rolling-24h
    peaked at 2,999 against 1,500). Deleting the ledger file IS a reset, and alerts.
  INV reservations-are-not-persisted: `_pending` is a promise to commit within ONE request; a
    persisted one would leak a permanently unusable slot. A hard kill between dispatch and commit
    loses up to 32 paid calls (the to_thread executor width), not one.
  INV fail-closed-direction: over-committing a cap slot is SAFE, releasing one that billed is the
    money leak. Every ambiguous outcome commits. Never trade this direction for elegance.
  ledger-availability ⊥ breaker-state ⊥ cache-health: all three break independently and NONE
    implies the others; each has its own alert and none is derivable from `resolving`.
  TRUST: `resolving` is the health signal, NOT `gemini_reachable` — models.list 200s straight
    through a credit outage in which every generateContent is refused.

PATHS:
  resolve [HTTP :9300 POST /resolve · SentinelCollector + SecMaster -> resolve()]
    does: cache -> company gate -> ledger fail-closed -> hourly backstop -> dispatch breaker ->
      daily-cap reserve -> ONE grounded call in a to_thread worker, shielded so a caller that
      walked away still gets its result committed and cached (SecMaster abandons at its 4s budget
      on ~1/3 of dispatches; without the shield those re-bill).
    does NOT: retry (retry_options=None -> stop_after_attempt(1)) · resolve anything but a company
      name · raise — fail-soft, every failure is a 200 with symbol=null.
    on-miss: 200 + symbol=null + cost_usd=0.0, INDISTINGUISHABLE at the caller from a dispatch
      REJECTION; SecMaster records both as GeminiCallOutcome.Success.
  health [HTTP :9300 GET /health] does: one live models.list probe + the stats fields.
  metrics [HTTP :9300 GET /metrics] does: the SAME stats helper, ZERO external I/O per scrape.
    does NOT: expose `reachable` — a slow probe would time out the scrape and blind the burn
      alerts on a Google problem.

PROCESSING MODEL ("a call that never billed must not spend the budget; a call that never answered
must not claim the account is alive"):
  every dispatch ends in one of THREE, never two: REJECTED (401/403/429 on the SDK's numeric
  .code) releases its slot and extends the streak · ACCEPTED (a body arrived, parsed or not)
  commits and clears the streak · NEUTRAL (504/503/timeout/transport, and a raise out of the
  thread pool) leaves streak AND open-state untouched.
  the CAP side of NEUTRAL splits, and the split IS fail-closed-direction rather than an
  inconsistency: a DISPATCHED non-answer COMMITS (it reached Google and may have billed),
  while a raise out of the to_thread pool RELEASES — that call never left the process, so
  nothing could bill. Do not "simplify" NEUTRAL to a single cap verdict.
  terminal semantics: "no resolution" means the model declined to ground it — the ORDINARY
  outcome for a genuine last resort, never a defect signal.

DISTINCTIONS:
  dispatch_rejected != api_accepted — the first decides the CAP SLOT, the second the BREAKER.
    Only the pair is complete; collapsing them let one 504 close an open breaker.
  429-from-us != 429-from-Google — ledger-unavailable, hourly cap, daily cap and breaker all
    answer 429 WITHOUT calling Gemini (SecMaster buckets them cap_exhausted). A Google 429 never
    reaches the caller as a 429 at all: it is a 200 with symbol=null.
  `resolving` != `up` != `gemini_reachable` — process serves HTTP / Google answers models.list /
    calls actually resolve. Three different outages.
  breaker OPEN != breaker refusing-everything — the half-open window is still `is_open` and
    admits exactly one probe per cooling period.
  cache hit != live call — served BEFORE gate, ledger and breaker; no quota, no ledger, and
    invisible to every burn and liveness signal.

CROSS-SERVICE: IN SentinelCollector (HTTP sync) · SecMaster IdentifierConfirmationService (HTTP
sync, DOMINANT by volume). OUT generativelanguage.googleapis.com (paid).
FEEDS: SecMaster catalog self-seed -> instrument identity -> the macro matrix; a wrong ticker here
is written, embedded and reinforced downstream.

GOTCHAS:
  ✗ edit ~/ATLAS/gemini-resolver-mcp while the unit is live # no build step; work in a worktree
  ✗ read a quiet burn alert as "not spending" # a rejected call RELEASES its slot, so
    live_calls_1h/24h FALL as a credit outage worsens — quieter, not louder
  ✗ treat symbol=null as a failure # a last resort legitimately fails to ground most subjects
  ✗ restart to clear a fail-closed ledger state # a restart IS a cap reset, the exact thing the
    ledger prevents, and a transient lock re-arms itself in ~30s
  ✗ propose a rejection RATIO for the breaker # CONSECUTIVE is deliberate: a topped-up account
    resumes at full rate at once instead of waiting out a window
  ✗ assume a journal 429 is rate limiting # every rejection body measured over 30 days reads
    "Your prepayment credits are depleted"
  ✗ alert on a rolling gauge to catch a lost ledger # the window decays to zero legitimately;
    only the cumulative counter can go backwards

DECISIONS: [each INTENT is WHY + the non-obvious precondition. The measured forensics — incident
counts, catalog measurements, reproductions — live AT the cited GUARD site and are deliberately
not duplicated here; follow the citation.]
  D-1 fail-closed-durable-cap: INTENT the resolver-side leg of SecMaster D-1 (gemini-last-resort-earned): a daily CALL cap bounding the paid frontier boundary, DURABLE because a guard holding only inside one process lifetime is not fail-closed (every restart handed the full quota back), and REFUSING while the ledger is unaccountable because UNKNOWN 24h spend is not ZERO spend at a service that sits AT its cap routinely. Deliberately the OPPOSITE of the instrument-validator cascade's fail-OPEN — neither default may be propagated to the other by analogy. The result cache keeps serving throughout (no quota, no ledger). / PRECOND every LIVE dispatch requires ensure_ledger_available() true AND a reservation inside DAILY_CALL_CAP; cache hits and gate refusals bypass both because they cost nothing / GUARD CallStats.try_reserve_live @ gemini_resolver/server.py:720 + CallStats.ensure_ledger_available @ gemini_resolver/server.py:636 (the fail-closed refusal is wired at server.py:968 — the `if not await stats.ensure_ledger_available():` itself, NOT the INTENT(D-1) comment 28 lines above it that reads like the construct) / TEST test_cap_persistence.test_daily_cap_still_refuses_after_a_restart (+ test_an_unreadable_ledger_refuses_live_calls_but_still_serves_the_cache, test_re_arming_after_a_transient_STARTUP_fault_does_not_hand_back_the_cap)
  D-2 accepted-not-merely-unrejected: INTENT the breaker's streak is a claim about the ACCOUNT, so only a call the API ANSWERED may reset it or close an open breaker. Reading "not rejected" as "success" made every 504/503/timeout declare recovery, and the 504 is the MODAL failure in the 30-day journal (293 against 229 rejections), so that reading was wrong on the commonest case. It is also why GeminiResolverNotResolving could never fire: the same defect capped the resolving gauge's dwell at 0 to ~330s, restarting its `for: 10m` every cooling period. The probe slot carries the same intent — claimed BEFORE the cap reservation so an open breaker cannot burn a slot, therefore every path that takes it must give it back, and a probe spent on a real HTTPS call must RE-ARM the cooling clock while one that never dispatched must NOT. Giving it back is OWNED as well as mandatory: a request admitted through a CLOSED breaker holds no claim, yet its outcome can land while a LATER request's probe is in flight, so an outcome recorder that cleared _probing handed a live probe's slot to the next arrival — measured, an unresolved probe alone correctly gave try_admit (False, False), and after a straggler's rejection or neutral outcome the next period gave (True, True), at a 300s cooling as readily as at 0.1s. / PRECOND record_success ONLY from ResolverResult.api_accepted; every exception path routes to record_neutral_failure, which leaves _consecutive and _opened_at untouched; _probing is cleared ONLY by release_probe and ONLY by the request holding the claim — on the reservation-refused branch, and in a finally INSIDE the shield so every dispatch outcome including a raise reaches it while a cancelled handler cannot release one still billing / GUARD the code that actually REFUSES is DispatchBreaker.try_admit @ gemini_resolver/server.py:272 (wired at server.py:1007); the state it reads is written by DispatchBreaker.record_neutral_failure @ gemini_resolver/server.py:346 (wired at server.py:1065 and :1093) + DispatchBreaker.release_probe @ gemini_resolver/server.py:365 (wired at server.py:1034 and :1115) + DispatchBreaker.record_success @ gemini_resolver/server.py:294 (wired at server.py:1091) / TEST test_cap_accounting.test_a_gateway_timeout_does_not_reset_the_rejection_streak (+ test_a_gateway_timeout_probe_leaves_the_breaker_open_and_re_arms_its_cooling, test_a_probe_refused_by_the_daily_cap_does_not_wedge_the_breaker, test_a_probe_that_raises_out_of_the_thread_pool_does_not_wedge_the_breaker; the ownership pair is test_a_stragglers_dispatch_outcome_cannot_release_a_live_probes_claim — no non-owner may release — and test_every_probe_outcome_gives_the_claim_back, its parametrized converse over all five owner paths, without which ownership-checking would only trade a stolen claim for a wedged breaker)
  D-3 rejected-call-releases-its-slot: INTENT a call the API refused BEFORE billing cost nothing, so its reservation returns to the cap instead of being spent; committing it kept resolution dead for up to 24h AFTER credits came back. Gated on the SDK's numeric .code and NEVER on `rationale` prose, which is free text and partly model-authored — coupling cap accounting to prose makes a model's wording change a billing change. The allow-list is exactly {401, 403, 429} and NOT "any 4xx": releasing a slot for a call that DID bill is the under-count direction #911 closed, so a status whose billing outcome is uncertain must keep committing. This release is what makes D-2's breaker mandatory rather than optional — it removed the accidental throttle a depleted account used to get from its own failures filling the cap. / PRECOND DISPATCH_REJECTED_CODES membership on an isinstance-narrowed APIError; a non-APIError carrying .code is a transport failure and still commits / GUARD gemini_client._is_dispatch_rejection @ gemini_resolver/gemini_client.py:35 (wired at server.py:1067) / TEST test_cap_accounting.test_credit_depletion_429_releases_its_cap_slot (+ test_an_auth_rejection_releases_its_cap_slot for 401/403; test_dispatched_failures_still_consume_a_cap_slot is the fails-when-broken half, and its 400/404 cases are what pin the allow-list's NARROWNESS — widening _is_dispatch_rejection to `400 <= code < 500` REDs exactly those two and nothing else)
  D-4 wrong-ticker-worse-than-the-bill: INTENT the result-cache key must never merge two DISTINCT instruments — SecMaster D-1's asymmetry applied to the cache. A wrong ticker served at cached:true / cost_usd 0 is UNBILLED, so no burn alert can see it, for the full 30-day TTL, and SecMaster self-seeds it into the catalog where EmbeddingService reinforces it through vector similarity. A repeated paid call costs $0.0004; a silent wrong instrument corrupts the matrix permanently. So normalization drops a digit ONLY where it is a QUANTITY (currency glyph, percent, scale word) and KEEPS a leading sign, which IS the instrument. Both rules are catalog-measured at the guard site: over-stripping collapses whole tenor families onto one key, and splitting on every non-alphanumeric turns a leveraged INVERSE into the long — a direction inversion whose output looks healthy and points exactly the wrong way. / PRECOND every cache read and write goes through make_cache_key; a normalization change is MEASURED against the catalog, never reasoned about / GUARD cache.make_cache_key @ gemini_resolver/cache.py:152 / TEST test_billing_waste.test_tenor_families_are_not_served_each_others_answers (+ test_leveraged_inverse_is_not_served_the_leveraged_longs_ticker, test_same_subject_different_instrument_is_not_served_the_wrong_answer; test_quantity_drift_is_not_a_disambiguator is the fails-when-broken half — UNDER-normalizing is what fails it, while OVER-normalizing passes it and instead REDs TWO of the three above, the tenor-families and leveraged-inverse fixtures. Both directions re-measured 2026-08-11; the earlier "over-normalizing passes the first three and fails that one" was inverted.)

SEE: README.md (config table, endpoint catalog, runbook) · SecMaster/AGENT_README.md D-1 (the
caller-side gate this service re-gates behind) ·
deployment/artifacts/monitoring/alerts/gemini-resolver.yml (every alert + the recorded OPEN GAPs)
· gemini-resolver-mcp.service (the unit; env IS the config surface)
