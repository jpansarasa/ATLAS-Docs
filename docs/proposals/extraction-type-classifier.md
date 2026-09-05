# Extraction Type-Classifier — Feasibility Specification

**Status:** SPEC for a build/no-build decision. Nothing implemented; no production code written.
**Recommendation:** **DO-NOT-BUILD as specified**, and the case is stronger than the first draft of
this document made it. Every claim that broke under red-teaming broke *against* the classifier.
The routing prize the first draft found is 4x smaller than it claimed (0.75% of candidates, not
3.0% of mentions), and the thing it proposed to un-bin routes to a prompt table rather than to the
substrate. Meanwhile the real gap — **~60% of articles ground zero entities, 74% produce zero
macro signals, and a ranking-free cap discards 23% of candidates** — sits **upstream** of
everything this spec designs and cannot be reached from the seam it proposed. Detail in §0, §3,
§9, §10.
**Origin:** user brainstorm 2026-08-06 — "All text is a 'thing'. Maybe 'Animal, Vegetable,
Mineral?' in the context of the extraction helps routing."
**Snapshot:** baseline tag `pre-spec-20260806`; ZFS snapshots + image tags.
Restore limits: `/opt/ai-inference/backups/pre-spec-20260806/MANIFEST.md` (not re-derived here).
**Measured:** 2026-08-06T13:00Z–18:30Z on mercury, live prod containers, SELECT-only DB access,
zero live Gemini calls.
**Revision:** rev-2, 2026-08-06. Corrects rev-1 on the routing prize (§3), the denominator
(§POPULATIONS), the sample bias (§2), the cache figure (§5), the supersession target (§7), the
gate-skip counter (§a) and the SVV ordering (§10). The DO-NOT-BUILD verdict is unchanged.

---

## POPULATIONS (read this before any number below)

Six times in this workstream a number was quoted against a population that structurally could not
contain the counter-example. Rev-1 then made the mistake a seventh time, in a new way — see
BASIS below. Every figure in this document is tagged with one of these:

| tag | what it is | what it structurally CANNOT contain |
|---|---|---|
| **[INGRESS]** | 250 most-recent `text/html` `sentinel.raw_content` rows, trafilatura-normalized through the real `:3109` service, then the real `spacy-ner` candidate path (`:3110/ner`) re-run with the live `max_candidates=10`. 205 docs pass the >200-char floor. | Only ORG-labelled spans plus two regexes — 2,461 of 2,468 mentions are ORG, 7 are `TickerInQuote`, **0 cashtags in 205 articles**. Non-ORG-shaped macro concepts ("inflation", "jobless claims", "the yield curve") can **never** appear as candidates, which caps any measurable macro-routing prize far below what an ORG-only census can reveal. Also absent by construction: anything past ~10 distinct surfaces per doc; ALL-CAPS multi-word surfaces (`spacy-ner-service.py:116` drops them, so "BANK OF AMERICA" is structurally invisible); surfaces under 3 chars; anything past 32,000 chars; the regex fallback path (never exercised while the sidecar is up). One ~7-hour window, 4 sources, one day — no weekend/overnight/earnings-season mix. **[HARD_STOP] And it cannot be re-derived: this population is NOT reproducible by re-running the query that produced it.** The live trafilatura normalizer keeps a module-level duplicate-paragraph LRU in a long-lived process, so its output depends on what that process has already seen. The identical 250-path draw yielded **147 docs on one run and 189 on the next** — and the **205** this document is built on **exceeds both**, so every `[INGRESS]` figure in §0, §2 and §3 (including the **22.9%** cap loss driving SVV item 3) rests on a single draw that nobody, including the author, has reproduced. Treat each figure as one observation from a high draw, never as a rate; treat any re-run as a *different* population, not a check on this one. Mechanism and the mandatory freeze-to-disk procedure: §Appendix. |
| **[QUEUE]** | `entity_resolution_review_queue`, live accreting, never shrinks (every row `is_open`). | **Failures only.** A surface that resolved is absent by construction. Cannot bound any false-drop or real-issuer claim. Row counts are not reproducible citations — cite deltas. |
| **[PROM]** | Prometheus `increase(...)`, instant anchored to actual `date -u`. Window is `[24h]` unless a figure says otherwise; several say 7d, and the two are not interchangeable — 24h figures run ~1.4x the 7-day mean. | Counter resets (**check them per window, not once — see §0 and §a**; a `resets()[24h]` of 0 says nothing about `[7d]`); `increase()` extrapolates at boundaries (±1%). |
| **[SUBSTRATE]** | `atlas_data.macro_observations WHERE source_collector='sentinel'`, SELECT-only. | Only what was *written*. Cannot show what the classifier declined to emit. |
| **[PROBE]** | Direct calls to live `spacy-ner` / `llama-cpu-rag` with author-constructed inputs. | Small n, author-chosen. Indicative, not distributional. |
| **[AUTHOR]** | Hand labels. | Not adjudicated by anyone else. The taxonomy boundary cases (§4) are exactly where this is weakest. |

### BASIS — the denominator rev-1 got wrong

Rev-1 expressed every share as a fraction of **emitted spaCy ORG mentions** (2,461 over 205
articles = 12.45/article). That is not what crosses the wire. The sidecar dedups per document and
applies `max_candidates`; the prepass then collapses near-duplicates (D-7) and filters (D-1/D-5/
D-11/D-14). Live [PROM], 24h to 2026-08-06T18:07Z:

| stage | volume/day | per article |
|---|---:|---:|
| articles dispatched to the prepass | 1,330 | — |
| candidate surfaces **before** the ingress filter | 8,473 | **6.37** |
| rejected by `CandidateSurfaceFilter` | 2,254 | 1.69 |
| candidate surfaces **POSTed** to SecMaster resolve | 6,219 | 4.68 |

`secmaster_entity_resolution_candidates_per_request_sum` = 6,219.1 over
`..._count` = 1,274.2 requests. 6,219 + 2,254 = 8,473 pre-filter, and 8,473 / 1,330 = **6.37
candidates per article** — not 12.45 mentions.

Confirmed independently on the [INGRESS] slice, through the live sidecar: 205 articles yielded
**2,461 ORG mentions** (rev-1's figure reproduces exactly) but only **1,334 candidates** after
per-document dedup and the cap. 2,461 / 1,334 = **rev-1's denominator was inflated 1.85x**, and
the measured 6.51 candidates/article agrees with the 6.37 derived from [PROM] above.

*Rev-1 is also internally inconsistent here:* 2,461 mentions / 205 articles = **12.04**, not the
12.45 it quotes. 12.45 implies ~198 articles, as does its own 1,264 / 6.37 = 198.4. Two different
article counts are in use in one document.

Every percentage below states its basis explicitly as one of:

- **% of candidates (pre-filter)** — denominator 8,473/day, 6.37/article. Use this for anything
  about what the ingress filter sees.
- **% of candidates (POSTed)** — denominator 6,219/day, 4.68/article. Use this for anything about
  what SecMaster resolves or what the paid resolver could see.
- **% of mentions** — denominator 12.04/article (rev-1 says 12.45; see above). Use this **only**
  for statements about spaCy's own output, i.e. the label table in §a.
- **% of the head-150 sample** — the basis of every hand-labelled composition figure in §2.
  It is neither of the above and must never be described as "% of ingress volume".

Where I could not measure something, it says so. Nothing here is rounded up to confidence.

---

## §0 THE GAP THAT REPRIORITISES EVERYTHING — ~60% GROUND NOTHING, 74% SIGNAL NOTHING

Rev-1 buried this as an afterthought in §a ("Added: a genuine finding the brief did not
anticipate"). It is the largest measured fact in the document and it belongs first, because it
determines whether anything downstream is worth building.

**[PROM]** `sentinel_entity_resolution_prepass_dispatched_total`:

| window | resolved | empty | **empty share** | resets in window |
|---|---:|---:|---:|---:|
| 24h to 2026-08-06T18:07Z | 758.1 | 572.1 | **43.0%** | 0 |
| **7 days** | 2,650 | 4,036 | **60.3%** | **8** per series |

*Scoping correction: an earlier draft cited a single `resets()[24h]` = 0 above both rows, which
licenses only the 24h one. Re-checked at the same 18:07Z anchor, `resets([7d])` is **8** on both
`resolved` and `empty` (1 on `fallback_regex`) — the 7-day window is not reset-free and never was.
**The 60.3% figure is unaffected**: it comes from `increase()`, which is reset-correct by
construction. What was wrong was the justification, not the number. Cited as a worked instance of
this document's own "range-query with `resets()` before quoting anything" rule — the rule was stated
and then applied to the wrong window in the same section.*

**Quote the 7-day figure, not the 24-hour one.** Per-day empty share over the week: Jul 31 55.7%,
Aug 1 63.2%, Aug 2 **75.4%**, Aug 3 69.5%, Aug 4 68.7%, Aug 5 60.4%, Aug 6 43.1%. **The 43% that
this section was originally written around is the *best* day of the week, not the norm.** The
honest headline is **~60% of articles resolve zero entities, ranging 43–75% daily** — which is a
stronger finding than the one it replaces.

Volume is bursty too: 7-day mean **956 dispatches/day** against 1,325 in the last 24h. Article
count cross-checks independently — `sentinel_spacy_ner_requests_total{outcome="ok"}` = 1,324.2 in
the same 24h.

**Not an outage.** The series is continuous with no cliff, `fallback_regex` runs at ~1/day, and
`timeout`/`unavailable` do not appear at all. This is the steady state, and the steady state is
bad.

**[PROM]** `sentinel_news_signal_classifier_request_total`, **24h only** — I did not range this
one over 7 days, and given how far the prepass 24h figure sat from its weekly mean, **treat 74.3%
as a single-day reading, not a weekly rate**:

| outcome | /day | share |
|---|---:|---:|
| `success` | 342.1 | 25.7% |
| **`empty`** | **987.2** | **74.3%** |
| `timeout` | 0.0 | 0% |

**74.3% of articles produced ZERO macro signals on the measured day.** `outcome=empty` is a
*correct no-op* by design (`sentinel.yml:320` says so explicitly, and most news genuinely carries
no macro signal) — but it is also bounded by the fact that there are only 85 things an article is
*allowed* to map to (§3), and 81 of those 85 have already fired in the last 30 days.

Two further losses at the same seam:

- **The `max_candidates` cap discards 23% of candidates, and it discards the *cleaner* ones.**
  **[INGRESS]** 205 articles through the live sidecar (`:3110/ner`), live cap confirmed = **10**
  (`SpacyNer__MaxCandidates` is absent from `/opt/ai-inference/compose.yaml`, the running container
  env and `appsettings.json`, so the C# default applies):

  > 2,468 mentions → **1,730 after per-document dedup** (−29.9%) → **1,334 after the cap** (−22.9%)
  > = 396 slots and **341 distinct surfaces** dropped. The cap binds on **70 of 205 documents
  > (34.1%)**.

  **The sidecar applies no ranking at all.** `extract_candidates`
  (`spacy-ner-service.py:78-121`) dedups on `(surface.lower(), kind)` and truncates in *probe
  order* (TickerInQuote → cashtag → spaCy ORG), then **raw document order**. Consequences:
  - **Positional, not frequency-based.** Dropped surfaces have median first-mention index **20**
    vs **4** for survivors. A heavily-repeated entity introduced late is discarded wholesale —
    `EUR` (7 mentions, first at index 111), `Fed` (6 mentions, index 73), `TXN` (5, index 12).
  - **The cap sheds cleaner material than it keeps.** Cap-dropped candidates are **90.2%
    Keep-class** under `CandidateSurfaceFilter` versus **83.7%** among survivors, because junk
    (bylines, datelines, page furniture) clusters *early* in a document and therefore occupies the
    retained slots. **The cap is mildly counter-productive, not merely lossy.**

  Whatever the cap drops is invisible to every stage downstream, including everything this spec
  proposes. **Raising this cap, or giving the sidecar any ranking at all, is a config/one-function
  change and is a larger lever than any component in this document.**
- **The ORG-only filter is a recall problem, not only a precision one.** **[PROBE]** `Fortinet`,
  `Brent`, `CPI` and `unemployment rate` produce **no entity at all** from
  `spacy-ner-service.py:109-119` — not a wrong label, no span. They never become candidates.

**Why this reprioritises the rest of the document.** Every design in §5 sits *after* the sidecar.
No classifier placed downstream can recover an entity the sidecar never emitted, or an article the
classifier already scored empty. A component that improves routing for the ~40% that ground
something cannot touch the ~60% that ground nothing — and the ~60% is the larger number.

**And it interacts with §a in a way rev-1 missed.** §a's structural argument (the spaCy label is
uninformative) is correct **for the current, narrow emission**. It is conditional on the ORG-only
filter. The moment emission is widened — which is what fixing this gap means — the surviving
population stops being ORG-by-construction and the label stops being conditionally constant.
Rev-1 deleted the label-plumbing component and simultaneously identified the recall gap without
noticing that **widening emission is precisely the population where plumbing the label starts to
matter.** The two findings are coupled. If the recall gap is ever addressed by widening the
sidecar's label filter, §a must be re-derived, not cited.

---

## §a IS spaCy's TYPE SIGNAL ALREADY EMITTED AND DISCARDED?

**Short answer: the signal is discarded, and recovering it buys ~nothing *at today's emission
width*. Investigate-first was still right — it killed the largest proposed component before it was
designed. But the conclusion is conditional, not absolute (see §0).**

### The mechanism

The brief's stated mechanism — "the sidecar maps EVERY ORG span to `CompanyName`" — is true but
understates it. `deployment/artifacts/spacy-ner/spacy-ner-service.py:109-119`:

```python
for ent in doc.ents:
    if ent.label_ != "ORG":
        continue                      # <- every non-ORG span dies here
    ...
    if add(surface, "CompanyName"):   # <- kind hardcoded, never read from the model
```

It is not that ORG/PERSON/GPE/PRODUCT are collapsed into one field. **Non-ORG spans are never
emitted at all.** spaCy computes the full label set — the `ner` pipe runs over the whole document
— and the sidecar drops 74.7% of what it produced one line before it would have crossed the wire.

**[INGRESS] — basis: % of MENTIONS** (this is a statement about spaCy's own output, so mentions is
the correct denominator here). Of 10,052 entity mentions in 205 real articles:

| label | mentions | share | fate |
|---|---:|---:|---|
| **ORG** | 2,542 | **25.3%** | emitted as `CompanyName` |
| DATE | 1,693 | 16.8% | discarded |
| PERSON | 1,539 | 15.3% | discarded |
| GPE | 1,038 | 10.3% | discarded |
| CARDINAL | 974 | 9.7% | discarded |
| PERCENT | 798 | 7.9% | discarded |
| MONEY | 473 | 4.7% | discarded |
| NORP / LOC / ORDINAL / WORK_OF_ART / TIME / PRODUCT / EVENT / FAC / QUANTITY / LAW / LANGUAGE | 995 | 9.9% | discarded |

An independent re-draw — 217 articles, spaCy run **inside the live `spacy-ner` container** with
the service's exact model load — gives 9,431 occurrences with 2,464 ORG = **26.1%**, consistent
with the 25.3% above. (The two draws differ in article count for the trafilatura reason in the
Appendix, not because either is wrong.) **The label distribution is the one figure in rev-1 that
reproduced without correction.**

The wire contract has no room for the distinction either: `EntityCandidateKind` is a 3-member enum
mirrored on both sides (`SentinelCollector/src/Extraction/EntityCandidate.cs:10-20`,
`SecMaster/src/Models/EntityResolutionModels.cs:15-25`), and only `CompanyName` is ever produced
by the NER path.

### Why recovering it buys ~nothing today

The junk that reaches the resolver is, **on today's emission width**, the subset spaCy already
labelled ORG. Reading the label back tells you "ORG" — which is the filter condition. The label is
*conditionally constant on the surviving population*, so its information content there is close to
zero.

This is the identical defect the brief correctly identified in D-1's `kind==CompanyName`
precondition, one layer upstream. It is the same bug twice, not two bugs.

**The absolute phrasing in rev-1 was too strong, and the correction is now measured.** The sidecar
emits a surface if **any** occurrence is ORG, so a surface can be ORG in one sentence and
PERSON/GPE elsewhere. Those "minority-ORG" surfaces have a non-constant label, and a "drop
minority-ORG surfaces" rule would be *actionable* on them.

**[INGRESS]** 217 articles, spaCy run **inside the live `spacy-ner` container** with the service's
exact model load and mention filter (`spacy-ner-service.py:105-119`): 9,431 entity occurrences,
2,464 ORG mentions, **1,676 candidate rows pre-cap** over **1,234 distinct surfaces**.

| population | pure-ORG | mixed |
|---|---:|---:|
| candidate rows pre-cap (n=1,676) | **96.00%** | 4.00% |
| distinct surfaces (n=1,234) | 95.46% | 4.54% |

Non-ORG labels inside the mixed rows (n=121 occ): PERSON 82, GPE 24, NORP 5, PRODUCT 4, other 6.
**Minority-ORG** (ORG < 50% of that surface's occurrences in that document) is **17 / 1,676 =
1.01%** of candidate rows — 0.89% of distinct surfaces, 3.52% occurrence-weighted.

**And the rule would be unsafe.** Judging all 17 against the live catalog and `edgar_filers`:
**6 of 17 are REAL** — `Datadog` twice (DDOG, on its earnings-crash day, the literal subject of
both stories), `Tesla` (TSLA), `Tata Sons`, `Hang Seng`, `OpenAI`. Precision is **59–65%**, not the
~50% the red-team review estimated, but that correction does not help: a coin-flip-plus rule whose
errors are *silent drops of the article's actual subject* is exactly what D-11/D-14's PRECOND
forbids. The `<=50%` variant is strictly worse — the 32 tie rows are dominated by real issuers
(Nvidia, Micron, Tesla, Sandisk, Nexstar, Commvault, Liberty Broadband).

So the honest claim is **"approximately zero, not exactly zero, and only at today's emission
width"** — not "definitionally unable". The counter-population is real, it is ~1%, and acting on
it is net-negative.

*Population limit, and it matters:* by construction of the ORG-only filter the minimum ORG share
is 1/N, so a 0%-ORG case can never appear — the measured purity is **upward-biased by design**.
Regex-probe candidates (`Ticker`/`TickerInQuote`) carry no spaCy label at all and can never be
"mixed", so a label rule is inapplicable to them. And a surface appearing 30 times as plain tokens
and once as ORG scores 1/1 = pure.

Measured three ways:

1. **[PROBE]** spaCy's own label on discriminating pairs, in realistic sentences:
   `CFO`→ORG · `IBM`→ORG · `EPS`→ORG · `PMI`→ORG · `Tech Stock`→ORG · `Reuters`→ORG · `LLC`→ORG ·
   `the Federal Reserve`→ORG · `the Korea Exchange`→ORG. Bare `Trump`→**ORG**.
   `Henrique Morello - Morgan Stanley` (the analyst-attribution fusion of D-14)→**ORG**.
   And real issuers are *missed entirely*: `Fortinet`→no entity, `Brent`→no entity,
   `CPI`→no entity, `unemployment rate`→no entity (§0).

2. **[QUEUE]** Re-parsing each queued surface inside its own stored `context_excerpt`, two
   independently drawn samples:

   | | ORG (incl. partial) | non-ORG type labels | not an entity |
   |---|---:|---:|---:|
   | top-400 by occurrence (91,240 occ) | **68.3%** | 4.0% | 27.9% |
   | random 600 distinct (1,824 occ) | **70.6%** | 3.2% | 26.5% |

   A non-ORG label would have caught **3–4%** of the junk. The ~27% "not an entity" is largely
   surfaces that never came through spaCy at all — the V2-direct `SubjectEntity` leg (D-6) is LLM
   output and bypasses the sidecar entirely — which a spaCy-label fix cannot touch by construction.
   *Population limit:* [QUEUE] holds failures only, so it cannot bound the false-drop side.

3. **The structural argument, which remains the strongest of the three — with its scope stated.**
   PERSON spans are *already* dropped correctly (15.3% of mentions). So the persons that reach the
   paid resolver are precisely the ones spaCy **mislabelled as ORG** (`Trump`, `Warsh`,
   `Henrique Morello - …`). For those, a correct label would not have helped, because the label
   *is* the wrong one. That argument is airtight for single-label surfaces. It does **not** cover
   the mixed-label counter-population, which is why that is measured separately rather than
   asserted away.

### The kind field is inert — but rev-1's counter reading was wrong

**[PROM]** `secmaster_entity_resolution_gemini_gate_skipped_total`. Rev-1 read the raw counter and
reported "**eighteen lifetime skips**" (`not_company_kind` 17, `code_slug` 1), calling it "the
cleanest possible confirmation". **That reading is false.** The counter had been reset by a
service restart shortly before the reading; an instantaneous counter value is not a lifetime
total, and rev-1's own POPULATIONS table warns about exactly this ("counter resets") without
applying it. See §8 for the corrected rate and the reason set over a full 7-day range.

**The substantive point survives and is unchanged:** the gate is inert. The precondition can only
fire for `Ticker`/`TickerInQuote`, and the NER path hardcodes `CompanyName`, so it skips a
fraction of a percent of dispatches. `gemini_resolver_gated_24h` = **0** is confirmed exactly.
The diagnosis was right; the arithmetic behind it was not, and the arithmetic is what a reader
would have checked.

### What this does to the scope

- **Deleted:** the "plumb spaCy's label through" component, and the `EntityCandidateKind` widening
  it implied. That was the cheapest proposed component and the one with the best precedent. It
  does not survive measurement **at today's emission width** — and that qualifier is load-bearing
  (§0).
- **Survives:** the case for *semantic* classification (§2), because it never depended on the
  spaCy label.
- **Reprioritised:** the recall gap, promoted to §0.

---

## §2 WHAT CAN AN LLM DO THAT GATES PROVABLY CANNOT?

### The composition — and the sample bias rev-1 did not state

Rev-1 hand-labelled the **top-150 surfaces by occurrence** and generalised to the population. That
sample is not representative, and the bias has a direction. Objective shape features only,
head-150 vs tail [INGRESS]:

| feature | \[mention basis\] head / tail | \[candidate basis\] head / tail (n=150 / 861) |
|---|---:|---:|
| deflation (distinct / occurrences) | **0.132 / 0.867** | 0.317 / **1.000** |
| ALL-CAPS acronym-shaped (≤5 chars) | 32.7% / 15.6% → **2.1x over** | 24.0% / 16.7% → **1.44x over** |
| carries a corporate suffix (Inc/Corp/Ltd/…) | 4.7% / 12.8% → **2.7x under** | 8.0% / 13.6% → **1.7x under** |
| ≥4 whitespace tokens | 7.3% / 22.0% → **3.0x under** | 13.3% / 22.6% → **1.7x under** |

**Direction of the bias: the head over-represents short ALL-CAPS acronym-shaped surfaces —
agency/index/ticker-like — and under-represents corporate-suffix and long multi-token names,
precisely the class most likely to resolve to a real equity.** So the composition table below
**overstates the non-issuer classes and understates issuers**, the direction that flatters the
classifier's case.

Two things to note. First, moving to the correct (candidate) basis roughly **halves** the
distortion, because per-document dedup already removes most of the repetition that inflated the
head — so the bias is real but not catastrophic. Second, and less comfortably: the top-150 covers
**35.5% of candidates but only 14.8% of distinct surfaces**, and **tail deflation is exactly
1.000 — every one of the 861 unexamined surfaces occurs precisely once.** Generalising hand-labels
from 150 to the population is extrapolating from a differently-shaped 15% minority onto a tail
that is entirely singletons.

**[INGRESS]/[AUTHOR] — basis: % of mentions WITHIN the head-150 sample.** This is the honest
statement of what rev-1's roll-up measures. Rev-1 labelled it "% of ingress volume", which reads
as a share of the whole population; it is not. It is a share of a 150-surface head that covers
45.6% of mentions and, per the table above, is not representative of the other 54.4%.

| roll-up | share of the head-150 | what it is |
|---|---:|---|
| should reach the equity cascade | ~27% | issuer + instrument |
| genuine economic object, wrong destination | ~26% | institution + economic-series + commodity + currency |
| not an economic object at all | ~47% | fragment + media-outlet + concept + metric-jargon + person + role-title + place |

**These cannot be projected onto the candidate population** without correcting for the bias above,
and the bias runs against issuers — so the true issuer share of candidates is higher than 27% and
the true garbage share is lower than 47%. The decomposition — *misrouted* versus *garbage* — is
the part that matters and it survives; the exact split does not. Re-drawing this over a random
sample of distinct surfaces is item 3 of §10.

### The classes only semantics can separate — and why 9.8% is a FLOOR, not an estimate

Shape rules provably cannot separate `CFO` from `IBM`, or `Fortinet` from `the Korea Exchange`
(both ORG, both plausible 1-2 token Title-Case). D-14 already litigated and *rejected* a shape
discriminator for the attribution class at **5% precision**. So the semantics-only claim is sound
in principle. The question is how much volume it is worth.

Rev-1 answered: `person` 4.4% + `acronym-ambiguous` 4.0% + `role-title` 1.4% ≈ **9.8%** — and
called it "% of ingress volume". It is **9.8% of the head-150 sample**, on the mentions basis, and
therefore inherits the bias above. **It is also a floor by construction, and rev-1 presented it as
an estimate.** It counts only the classes with *no*
enumeration, which implicitly treats the classes that *are* governed by enumerations —
`institution` (D-1) + `media-outlet` (D-11) + `metric-jargon` (D-14) + `concept`, together
**41.9%** of the head-150 labels — as **fully covered**. Curated exact-match lists are precisely
the mechanism that fails on the member it has not enumerated yet. The residual an enumeration
misses is exactly the semantic classifier's remaining value, and it is not counted in 9.8%.

The honest statement is: **≥9.8%, with an unmeasured addition from enumeration misses inside the
41.9%.** Rev-1 said "plus an unmeasured share" in a trailing clause and then used 9.8% as the
number in the break-even table (§8) as though it were the total. That is the error.

Note the ceiling this implies in the other direction: D-14's census put bare acronyms at **17.5%**
of the *paid-resolver* stream [QUEUE]; the same class is ~4% of the head-150 [INGRESS]. Both can
be true — the resolver stream is the residue after everything easy resolved, so it concentrates
hard classes. **The larger number is the wrong one to size an ingress-placed component with.**

### Can the CPU model actually do it? — measured, not asserted

**[PROBE]** 40 real ingress surfaces, each with its sentence, against live `llama-cpu-rag`
(qwen2.5-7b-instruct-q4_K_M) using llama.cpp `json_schema` — the decode-time grammar mechanism
`SecMaster/src/Services/LlmClient.cs:217` already uses in production:

- **Accuracy 34/40 = 85%** exact match on an 11-way taxonomy.
- **Zero unparseable outputs.** Constrained decoding held on every call.
- **Latency p50 0.4s, p95 4.0s, max 4.5s** — an order of magnitude cheaper than the RAG path's
  4.8s p50, because the prompt is ~350 tokens and the completion is ~10.

The six errors split by *direction*:

| surface | expected | got | direction |
|---|---|---|---|
| `LLC` | fragment | **abstain** | safe — abstain is the designed escape |
| `CFO` | fragment | person | safe — non-issuer, still routed away |
| `Research Division` | fragment | concept | safe — non-issuer |
| `Warsh` | person | institution | safe — non-issuer |
| `Tech Stock` | concept | **issuer** | **unsafe — routes junk INTO the paid cascade** |
| `Anthrop` (truncation) | fragment | **issuer** | **unsafe — same** |

Four of six degrade to "not an issuer", which is the correct outcome for the money. Two route junk
into the equity cascade — i.e. **degrade to today's behaviour**, not worse than it.

`Tech Stock` is a nice illustration — D-14's curated `MarketJargon` rule catches it
deterministically, at 100% precision, for zero marginal cost, and it is mutation-tested. But it is
**n=1**, and rev-1 promoted it to "the finding that matters", which it is not. See §9 claim 4 for
where that argument actually comes from.

---

## §3 WHAT DOES IT ROUTE RATHER THAN REJECT? — REV-1 WAS WRONG HERE

Rev-1 framed this as the prize and claimed the investigation "found something better than the
proposed build": *the receiver already exists, is already wired, and the ingress filter is binning
its inputs before they can reach it.* **That claim is wrong on both halves.** The receiver that
matters is not the one rev-1 found, and the ingress filter is not what stands between news and the
substrate. Correcting it removes the spec's headline recommendation entirely (§10).

### What un-binning "Fed" actually buys: one sectorless row in a prompt table

The stoplist and the Step-5 receiver are both real. `CandidateSurfaceFilter.cs:137-178`
`InstitutionStoplist` contains, verbatim: `"European Central Bank"`, `"ECB"`, `"Federal Reserve"`,
`"Fed"`, `"Bank of England"`, `"BoE"`, `"Bank of Japan"`, `"BoJ"`, `"People's Bank of China"`,
`"PBoC"`, `"Treasury"`, `"Congress"`, `"Senate"`, `"OPEC"` (plus IMF/BIS/WTO/OECD, the US federal
agency set, and the trailing-"Fed" regional rule). Rejected at
`EntityResolutionPrepass.cs:404-416` with `reason=institution`, never POSTed. And cascade Step 5
(`SecMaster/src/Services/EntityResolutionService.cs:630-638`) is deliberately kind-agnostic and
would resolve them.

**But trace what a Step-5 resolution actually carries.**
`EntityResolutionService.cs:773-783` — the `SignalIdentityDirect` return:

```csharp
return new ResolvedEntity(
    Surface: candidate.Surface,
    CanonicalName: identity.Label,
    Ticker: null,          Figi: null,
    NaicsCode: null,       AtlasSectorCode: null,
    MappingVersionLabel: null, ...);
```

The comment at `:630-638` states the intent outright: *"Sector is intentionally null — these
signals do not roll up to an ATLAS equity sector."* Now enumerate every consumer of the prepass
bag. There are exactly three:

| consumer | what it does with a null-sector entity |
|---|---|
| `DeterministicResolver.LiftSector` (`DeterministicResolver.cs:255`) | `.Where(e => !string.IsNullOrWhiteSpace(e.AtlasSectorCode) && …)` — **excluded** |
| `MacroObservationRouter.DeriveArticleSector` (`MacroObservationRouter.cs:246-249`) | `if (string.IsNullOrWhiteSpace(code)) continue;` — **excluded** |
| `ResolvedEntitiesRenderer.Render` (`ResolvedEntitiesRenderer.cs:43`) | renders one markdown row with empty Ticker / AtlasSector / NAICS — **a prompt table row** |

**Un-binning "Fed" buys a sectorless prompt row. Never an observation, never an instrument, never
a matrix cell.** Rev-1's "the prize already exists and we bin it" inverted the causality: the bin
is not what stops it reaching the substrate; it would not reach the substrate if the bin were
removed.

*One honest caveat, unmeasured:* the prompt row is fed to the extraction LLM, whose output
descriptions feed the substrate paths below. So there is an indirect, second-order channel by
which a prompt row could nudge a description. It is speculative, it is not measured here, and it
is not a basis for shipping anything.

### The prize is already being collected — through a door the filter does not touch

**[SUBSTRATE]** 7 days to 2026-08-06, `macro_observations WHERE source_collector='sentinel'`:

- **3,879 rows**, spanning **76 distinct signal identities** (of 85 in the catalog; 81 of 85 have
  fired in the last 30 days — the catalog is effectively saturated).
- Top signals: `oil-price` 452, `inflation-expectations` 335, `nasdaq-100` 178, `dxy-dollar-index`
  163, `ust-10y-yield` 160, `cpi-headline-yoy` 159, `sp500-fwd-pe` 136, `usd-jpy` 129,
  `fed-funds-rate` 116.
- Lifetime: `cpi-headline-yoy` **1,990**, `fed-funds-rate` **1,270**.

**Every one of those 3,879 rows was written while `Fed` was being binned at the ingress.** So the
macro entities rev-1 said we are "throwing away" are in fact the platform's highest-volume
substrate signals, arriving continuously.

**The mechanism, precisely — and this corrects the red-team review as well as rev-1.** All 3,879
rows carry the `{rawContentId}:sig:{signalIdentityId}` key built by
`MacroObservationRouter.BuildNewsSignalSourceId` (`:279-280`), which is reached only from
`TryPlanNewsSignalWrite`. Its `signal.SignalIdentityId` comes from **`NewsSignalClassifier`** — a
GPU vLLM call that reads the **whole article** and emits an id constrained by a JSON-schema `enum`
**built per-call from the loaded catalog's ids** (`NewsSignalClassifierSchema.cs:33-53`,
`NewsSignalClassifier.cs:68,200`). It never sees an NER candidate surface. `CandidateSurfaceFilter`
is not on this path at any point.

The other substrate path — `TryPlanMacroWrite`, which *does* use
`_catalog.TryResolve(observation.Description)` (`MacroObservationRouter.cs:91`) — is near-dead:
**81 rows lifetime, 0 in the last 7 days.** So it is *not* correct to say the substrate is fed by
an alias lookup on the LLM's description. It is fed by a schema-constrained classifier over the
whole article, and the alias lookup feeds a path that has written nothing this week. This
distinction changes the SVV (§10) — it is the reason alias expansion is **not** the top item.

### Size of the routing prize, on the candidate basis

Rev-1 reported **3.0% of ingress volume** (7 distinct / 73 mentions) as the binned routing prize.
Measured on the candidate basis over the same 205 articles, matching against all 358
ids+aliases with the real `MacroDescriptionNormalizer` form:

| set | distinct | candidates | % of 1,334 |
|---|---:|---:|---:|
| has a live receiver at all | 5 | 13 | **0.97%** |
| **binned by the InstitutionStoplist** | **4** | **10** | **0.75%** |
| reaches SecMaster today | 1 (`Brent`→`oil-price`) | 3 | 0.22% |

**The binned routing prize is 0.75% of candidates, not 3.0%** — and on the mention basis the same
set is 2.63% (9 distinct / 65 mentions), so rev-1's 3.0% was very nearly an artifact of the
denominator alone.

| surface | receiver | docs containing | docs surviving cap | candidates |
|---|---|---:|---:|---:|
| `Fed` | `fed-funds-rate` | 9 | 7 | 7 |
| `Brent` | `oil-price` | 6 | 3 | 3 (**Keep**) |
| `Federal Reserve` | `fed-funds-rate` | 1 | 1 | 1 |
| `BOJ` / `BoJ` | `boj-policy-rate` | 1 each | 1 each | 1 each |

**Why the mention basis inflates it so badly:** `Fed` alone accounts for **50 of the 65
receiver-matching mentions, across just 9 documents**. Per-document dedup collapses 50 → 9 before
the cap is even consulted. Counting mentions counts the same nine articles fifty times.

*(Correction to the red-team review, which claimed `Fed` appears in 17 docs and is emitted from 5
because the cap drops it in 12: the mechanism is real but ~6x smaller — **9 docs contain it, 7
survive, the cap drops 2**.)*

**Two receiver losses neither rev-1 nor the review found, both larger than the stoplist:**

- **The cap alone hides 4 receivers entirely.** Pre-cap the slice has 9 distinct receiver-matching
  surfaces / 22 candidates; post-cap, 5 / 13. `BoE`, `DXY`, `FOMC` and `NFP` never reach the wire
  at all. **44% of the distinct routing prize is lost to the cap, not to the stoplist** — which is
  §0's problem, not §3's, and is one more reason the SVV leads with the cap.
- **A `"the "`-prefix asymmetry between two components that must agree.**
  `CandidateSurfaceFilter.IsInstitution` strips a leading `"the "` before matching its stoplist;
  `MacroSignalIdentityCatalog` does **not** strip it before the alias lookup. So
  `"The Bank of Japan"` is correctly identified as an institution but then misses the receiver it
  is an alias of. That is a genuine one-line defect, independent of everything else in this spec.

Either way the conclusion does not turn on the exact figure: the prize routes to a prompt row
(above), so its size is close to irrelevant.

### Un-binning is not merely low-value — it is actively risky

Rev-1 never checked the cascade **order**. Step 1 (`HybridResolutionService` local exact/fuzzy/
vector) runs at `EntityResolutionService.cs:456`, long before Step 5 at `:630`. The cascade
**returns at the first hit** — it does not rank across steps. And these surfaces collide with real
catalog instruments [SecMaster catalog, SELECT-only, 2026-08-06]:

| surface | Step-1 collision | why `IsHighConfidenceMatch` fires |
|---|---|---|
| `BoE` | **`BOE` = BLACKROCK ENH GLBL DVD TR** | **exact symbol match** — the strongest tier |
| `Fed` | `FED.CO` = FAST EJENDOM DANMARK | symbol prefix, len ≥ 2 |
| `ECB` | `ECB.WA` = ECB SA | symbol prefix, len ≥ 2 |
| `BoJ` | `1758.HK` = Bojun Education Co Ltd | name prefix, len ≥ 3 |
| `Federal Reserve` | `M13010US33460M156NNBR` = Federal Reserve Bank Discount Rates for Minneapolis, MN | name prefix |

*(Correction to the red-team review: `ECB.WA` carries `discovery_source = FinnhubCollector`. It is
a genuine vendor row, not a SentinelCollector self-seed.)*

The **only** thing preventing each of these from resolving to a wrong instrument is
`HybridResolutionService.NameAppearsInContext` (`:564`) — a plain case-insensitive substring test
of the instrument's full name against the article text. Rev-1 never named this guard. Two
properties make relying on it uncomfortable:

- It **passes by default when no context is supplied** (`if (string.IsNullOrWhiteSpace(context))
  return true;`). Any caller that omits `ArticleContext` gets the wrong-ticker resolution.
- It protects by *coincidence of naming*, not by design for signal identities. It happens that
  "Fast Ejendom Danmark" does not appear in an article about the Fed. Nothing enforces that.

And when Step 1 *is* correctly dodged and Step 5 fires, the result scores
`SignalIdentityDirect = 0.90` (`:1152`) against local-fuzzy's `0.85` (`:1143`). The prepass then
does `OrderByDescending(e => e.Confidence).Take(EntityResolutionMaxResolvedEntities /* = 5 */)`
(`EntityResolutionPrepass.cs:219-226`). **So an un-binned central bank outranks a real
fuzzy-matched issuer and displaces it from the 5-row prompt table** — to contribute a row with no
ticker and no sector. The displaced issuer was the row that could have lifted a sector.

*(A note on the score comparison: 0.90 > 0.85 does **not** mean Step 5 beats Step 1 in the
cascade — the cascade short-circuits, so Step 1 wins by running first regardless of score. The
0.90-vs-0.85 ranking bites at the prepass `Take(5)`, which is a different place. Both matter, for
different reasons.)*

### Routing targets, honestly assessed

| type | receiver that exists | reachable from ingress today? |
|---|---|---|
| **macro signals** (rates, CPI, oil, FX, indices) | **85 `signal_identities`, via `NewsSignalClassifier` over the whole article** | **YES — already, at ~3,900 rows/7d. Nothing at the entity ingress is involved.** |
| **economic-series** via cascade Step 5 | 33 `macro` signal identities | Resolves, but to a **sectorless prompt row** (above). Not to the substrate |
| economic-series → FRED search | `CatalogService.cs:147` | **NO — deliberately off.** `EntityResolutionService.cs:836` passes `allowEconomicDiscovery: false` ("a news surface is never a FRED economic series"). This is #818's fix; re-opening it per-type would need a named supersession |
| **equity / issuer** | the full cascade | **YES** — today's default sink for everything, and the one path that produces a sector |
| **commodity** | 4 `commodity` signal identities + 23 `Commodity` instruments | `oil-price` is the **highest-volume** substrate signal (452/7d) — via the classifier, not the ingress. No name→AlphaVantage-series path from the ingress exists |
| **currency** | 4 `fx` signal identities | `usd-jpy` 129/7d, `dxy-dollar-index` 163/7d — again via the classifier |
| **institution — central bank** | 5, as policy-rate aliases | Binned at ingress — and un-binning routes to a prompt row, not a receiver |
| **institution — exchange** | **NONE.** `"Korea Exchange"` → no match, verified live | **NO RECEIVER EXISTS** |
| **institution — regulator** | **NONE.** SEC/FDIC/CFTC/FDA/CDC are stoplist entries only | **NO RECEIVER EXISTS** |
| person / place / concept / fragment | none, by design | reject is correct for these |

**The binding constraint is receiver coverage, not classification** — and the receiver that
matters is the **signal-identity catalog read by the GPU classifier**, not the entity cascade.
Building a classifier that confidently identifies `the Korea Exchange` as an institution changes
nothing: there is nowhere to send it and no defined semantics for what a macro platform would *do*
with an exchange mention. Classification without a receiver is a more expensive way to reject.

Corroborating damage, worth recording: the catalog contains junk written by the resolution
pipeline itself — `DHOILNYH="President Donald Trump"`, `GASREGW="Donald Trump"`, `DX="Japan"`,
`KC="Colombia"`, `PALL="Iran"`, and (found during this revision) **`NVS`** — Novartis — carrying
the name **"Seeking Alpha"**, and `KTEC` carrying **"the Hang Seng Tech"**. Person, place and
boilerplate surfaces landed on real instruments through the equity/discovery cascade. That is the
wrong-instrument corruption the `NameAppearsInContext` discussion above is about, already realised
in stored data.

**A methodological consequence for anyone re-measuring this:** a "catalog hit ⇒ real issuer" test
is itself corrupted by these rows. `Seeking Alpha` will "hit the catalog" and it is disclosure
boilerplate. Ground issuer judgements in `edgar_filers` plus the article context, not in
`instruments.name` alone.

---

## §4 THE PROPOSED TAXONOMY — ARGUED

Proposed: issuer / instrument / economic-series / commodity / institution / person / place /
concept / fragment. Measured against it (§2) it held up as a *description*. As a *routing key* it
is partly wrong, and I would change three things.

**Keep the spirit.** Animal-Vegetable-Mineral is right that the cut must be coarse, cheap and
mutually exclusive. The 11-way version scored 85% on a 7B; a finer taxonomy would not.

**Add `currency`.** Measured ~3% of candidates [INGRESS] — comparable to `commodity` and
`economic-series`, the two the brief named explicitly, and there are 4 `fx` signal identities.
Omitting it would send a well-served type to the wrong place.

**Split `concept`, or accept that it is a bin label.** `concept` + `metric-jargon` +
`media-outlet` absorb ~20% of the head-150 labels across three genuinely different things
(`Reuters` is a real issuer under another hat — NYT/News Corp/Fox all trade; `RSI` is a TA
indicator; `NFL` is neither). Collapsing them is fine **only** because all three route to reject.
The moment any of them gets a receiver, the class has to split. Say so now.

**`instrument` vs `issuer` is the weakest boundary and it is load-bearing.** `Nasdaq` is an
exchange (institution), an index (instrument), *and* NDAQ (issuer) — and `nasdaq-100` is
separately a live signal identity at 178 rows/7d. `Paramount` is an issuer and a studio. My own
labels are arguable here, and a 7B's will be too. Since both route to the same cascade today, the
distinction costs nothing now — but it is not the clean cut the taxonomy implies.

**`fragment` earns its place** — the largest single non-economic class, and the one where the
model agreed with the deterministic rules most often. Which is also why it does not need the model.

---

## §5 PLACEMENT, MODEL, OUTPUT, CACHE

Specified as designed, so the decision is made against a real design rather than a sketch.

**Placement: after `CandidateSurfaceFilter`, before the SecMaster POST**, inside
`EntityResolutionPrepass` (the D-1/D-5 seam). Rationale: #907's deterministic rules cost nothing
and are mutation-tested; the classifier must not get the chance to overrule them. Placing it after
also cuts call volume by the 2,254/day the filter already removes [PROM].

It **cannot** be placed to fix §0. An entity the sidecar never emitted (`Fortinet`, `CPI`), one
dropped by `max_candidates`, or an article the news-signal classifier already scored `empty`, is
not visible at this seam or at any seam downstream of it. **This is the single most important
constraint on the whole design.**

**Model: `llama-cpu-rag` (qwen2.5-7b-instruct-q4_K_M), never GPU vLLM.** Confirmed config:
`--parallel 4 --ctx-size 8192 --threads 16 --n-gpu-layers 0` → **2,048 tokens per slot**, not
8,192. Budget prompts against 2,048. GPU is production extraction — and note it is *already* doing
the news-signal classification that actually feeds the substrate (§3); a second classifier
competing for it costs the one that works.

**Contention is a real constraint, but not the blocker I expected.** All 4 CPU slots are already
claimed by SecMaster's `LlmGenerationGate` (`MaxConcurrentGenerations = 4`, explicitly sized to
`--parallel 4`). At the measured 0.4s p50 [PROBE] and the call volumes in §8, the classifier's
duty cycle is ~1–3% of one slot. It still needs **its own gate budget** rather than sharing
SecMaster's, or it will push `secmaster_rag_degraded_total{reason="timeout"}` and trip
`SecMasterRagBudgetTimeoutsSustained` (`deployment/artifacts/monitoring/alerts/secmaster.yml:38`).

**A new client is required.** `grep` for `llama-cpu-rag|11438` returns **zero hits in
SentinelCollector**. Its existing `LlamaServerClient` points at `llama-server` — the `--parallel 1`
DSL rollback runner — and DI comments warn repeatedly against occupying that slot. Copy the
established pattern: dedicated named `HttpClient` + `AddSharedCircuitBreaker`, as
`DependencyInjection.cs:606-641` already does for the sector-tagger.

**Output: llama.cpp `json_schema`**, an `enum` over the taxonomy plus an explicit `"abstain"`,
`temperature 0.0`, `n_predict 24`. Never free-text parsed — the mechanism is proven at
`LlmClient.cs:217` and held on 40/40 calls [PROBE]. `abstain` must route to *today's* behaviour
(the equity cascade), never to reject, so an uncertain model can never silently drop a real issuer
— the D-11/D-14 precondition.

**Cache: the brief's assumption does not survive, but rev-1's counter-evidence was also wrong.**

The brief specified "a persistent cache keyed on the surface so steady-state call rate approaches
zero". Rev-1 refuted it with two measurements, one of which does not hold:

- **[INGRESS] — holds.** Rarefaction over 205 articles: the distinct-surface set does **not**
  saturate. The new-surface rate is still 40–47% in the last decile; over the final quintile the
  steady-state miss rate is ~47%. News entity surfaces are a heavy long tail.
- **[PROM] — rev-1's 14.3% is real but is the wrong window, and it was used to argue the wrong
  thing.** Rev-1 quoted the live Gemini resolver's SQLite cache at 14.3–15.3% and treated it as a
  pessimistic floor on what any cache can achieve. Three different figures are all true, over
  different windows:

  | window | hit rate | source |
  |---|---:|---|
  | lifetime, 96 days | **14.39%** | the cache's own counters: 9,096 hits / (9,096 + 54,134 misses) |
  | entries written in the last 24h | 29.80% | cohort of 1,932 entries |
  | ~20h, pre-#911 | **33.55%** | `gemini_resolver_cache_hit_rate_24h` @ 2026-08-06T16:05Z |
  | now, 108-minute-old ledger | 22.26% | same gauge, post-deploy cold start |

  14.39% reproduces from the cache DB to two decimals — *it is not a ledger artifact*, and the
  red-team review's diagnosis of the mechanism was wrong even though its conclusion was right. It
  is a **96-day average diluted by 25,037 TTL-dead entries still resident (46% of the store — the
  cache never prunes)** and by the #772 poisoned-cache era. **Use ~33%.** Rev-1's "caches do not
  help" argument does not survive; rev-1's 15.3% upper bound could not be reproduced at all.

  The key is genuinely **compound** — `sha256("subject:{len(residue)}:{residue}|{subject_lower}")`
  where `residue` = description tokens minus subject/quantity/scale/legal-form tokens
  (`cache.py:152-212`) — so one surface can occupy several keys. **But that is a deliberate,
  priced tradeoff documented in the resolver's own D-1** ("an extra paid call is the cheaper error
  than a wrong instrument served free for a month"), not accidental fragmentation. The +22.6%
  figure is the code's own measurement over a 30-day journal (5,453 vs 4,447 keys); it is cited
  here as documented, **not re-measured** — the value payload stores no subject, so distinct
  surfaces vs distinct keys is not independently reproducible from the DB.

Net effect on the design: a persistent cache **does** help materially more than rev-1 allowed, but
the [INGRESS] rarefaction — the directly relevant measurement, on the right population — still
rules out a hit rate near 100%. **A realistic steady-state is 55–75%**, and §8's cost range is
unchanged. There is no existing persistent surface→classification store to reuse
(`openfigi_lookup_cache` is keyed on uppercased ticker; SecMaster's `EmbeddingCache` is in-process
and dies on restart), so this is a new table plus an EF migration either way.

---

## §6 FAILURE BEHAVIOUR: FAIL-OPEN, AND THE ALERT THAT MAKES IT SAFE

**Fail-open** (user's decision): classifier unreachable/timeout/circuit-open → the candidate
proceeds to the SecMaster cascade exactly as today. A classifier outage must never stop news
extraction.

**The mandatory consequence.** An unmonitored fail-open silently reverts to the junk flood the
moment the classifier dies — precisely the 4-day silent outage that started this work
(`SecMaster/AGENT_README.md:61`: "a call that never lands leaves the resolver ledger idle, not
alarming"). So fail-open is only acceptable with:

1. **Metric.** `sentinel_entity_type_classified_total{type, outcome}` where `outcome ∈
   {classified, abstain, unavailable, timeout, circuit_open}`. Bounded cardinality: 11 × 5.
2. **A wired alert** on the *fail-open path being load-bearing*, not on the corpse:
   `SentinelTypeClassifierFailingOpen` — `unavailable+timeout+circuit_open` share > 0.3 over
   `[30m]` with a `>= 5` dispatch floor, `severity=warning`. Template in-repo:
   `NewsSignalClassifierFailingHard` (`deployment/artifacts/monitoring/alerts/sentinel.yml:305`).
3. **An alert test.** `deployment/tests/alerts/sentinel_test.yml`, picked up automatically by the
   `./*_test.yml` glob in `deployment/tests/alerts/run.sh` (confirmed present, executable,
   promtool-in-container, not currently CI-wired). House style: `promql_expr_test` against the
   `ALERTS` series, never `alert_rule_test` + `exp_alerts`.

**Precedent that this is not paranoia — and the finding worth acting on independently of this
spec:** `sentinel_candidate_surface_filtered_total` (`SentinelMeter.cs:1009-1010`, tags
`reason`+`mode`) is **metered but entirely unwatched**. Re-verified 2026-08-06: `grep -rn
"candidate_surface_filtered" deployment/` returns **zero files**, across 13 alert files, 22
dashboards, the provisioning/alerting tree **and** the live `/opt/ai-inference/monitoring/` mount
— no alert rule, no dashboard panel anywhere. #907's filters, doing 2,254 rejections/day (825/day
on a 7-day mean), could stop firing and nothing would page. That gap is cheaper to close than this
whole spec, and §10 specifies the predicate — including why the obvious predicate is wrong. The
`mode` tag reads **100% `enforce`**, zero `shadow`, so the filter is live everywhere.

---

## §7 INTERACTION WITH WHAT ALREADY EXISTS

- **#907 / D-14 structural filters — KEEP, unchanged, and run FIRST.** Deterministic,
  mutation-tested, proven in prod (2,254 rejections/day [PROM]). The classifier is a second stage
  behind them or it is a regression. Nothing in this spec supersedes D-14.
- **The `institution` class belongs to D-1, not D-14 — rev-1 named the wrong entry.** D-1
  (`SentinelCollector`, `ingress-junk-filter`, #824) PRECOND enumerates the reject set as
  "markup-chars | dotted-id | slug+digit | money/number | metric-abbrev | country | **institution**
  | crypto". D-14's PRECOND is "shape-based, NEVER name-based" and explicitly lists bare acronyms,
  market indices and person names as out of scope — `institution` is not its class. Rev-1's §10.1
  proposed "an explicit supersedes D-14 (institution class only)". **That is an INTENT_FIDELITY
  defect inside the spec's own residual**: it would have superseded an entry that does not carry
  the precondition being changed, leaving D-1's PRECOND standing and contradicted. Corrected here;
  §10 drops the item entirely, so no supersession is required at all.
- **D-1 (`SecMaster`, gemini-last-resort-earned) — NOT superseded.** Its `kind==CompanyName`
  precondition is inert (§a) but the rest of the gate is live. If a classifier were ever built,
  that PRECOND should be *extended* (named supersession required) to read a classified type rather
  than `kind`. Note the two D-1s are different entries on different service cards; cite the card.
- **D-11 / D-14 preconditions constrain the design.** "A false-positive silently DROPS A REAL
  RESOLUTION, worse than junk the $0 cap refuses." This is why `abstain` routes to the cascade and
  why the classifier may never be the sole reason to reject. **This precondition alone is
  sufficient to reject a reject-on-classifier-verdict design** — see §9.
- **`NewsSignalClassifier` is the component this spec is actually adjacent to** (§3), not the
  entity cascade. It already does schema-constrained classification over the whole article against
  a curated catalog. Anything proposing "classify then route" on this platform should start by
  reading it, because it is that pattern, in production, working.
- **The SecMaster cascade Step 5** needs no change — it is already kind-agnostic and already
  correct. It is simply not a substrate path.
- **`QueryAnalyzer` + `SmartRouter`** (`SecMaster/src/Services/SmartRouter.cs:81-89`) is a
  classify-then-route dispatcher that already exists — `Economic→FRED, Equity→Finnhub, Rate→OFR,
  Commodity|Currency→AlphaVantage`. It is wired only to `GET /api/collectors/search`, which
  SentinelCollector never calls. If a router is ever built, **extend this rather than write a
  second one** — but note `QueryAnalyzer` is keyword scoring, and it is what mis-infers "Economic"
  from "federal reserve", the exact bug `allowEconomicDiscovery: false` exists to suppress.

---

## §8 COST / BENEFIT, WITH REAL NUMBERS

**Volume [PROM], 24h to 2026-08-06T18:07Z:** 1,330 articles/day; **8,473 candidates/day
pre-filter** (6.37/article, corroborated by the [INGRESS] slice's 6.51); **6,219 candidates/day
POSTed** to SecMaster resolve (4.68/article). Spec places the classifier after the filter →
**6,219/day**.

*One unreconciled discrepancy, stated rather than smoothed over:* the Sentinel-side counter
`sentinel_secmaster_resolution_total` reads **8,085/24h** against SecMaster's receiving-side
`secmaster_entity_resolution_candidates_per_request_sum` = 6,219. The two count different things
(the Sentinel-side series includes resolution calls beyond the prepass candidate POST, e.g. the
D-6 V2-direct leg). Rev-1's 6,170/day is the right *kind* of number and reproduces on the
receiving side; the 8,085 is not a correction to it. **Do not average them.** Anyone sizing this
component should re-derive which counter answers their question.

Note also that all per-day figures here are from a 24h window that runs **~1.4x the 7-day mean**
(956 dispatches/day over 7 days, §0). Sizing on the 24h numbers is conservative for cost and
optimistic for benefit.

**Model call volume after caching:** at 55–75% hit rate (§5), **1,550–2,800 calls/day**
(1.1–1.9/min).

**Latency added to the ingress path:** p50 0.4s, p95 4.0s per *uncached* surface [PROBE].
Per article (4.68 candidates post-filter, ~25–45% uncached): **~0.6–0.8s p50 added per article**,
assuming per-candidate calls are issued concurrently within the existing gate. The prepass is
already an async fail-soft step off the critical path, so this is throughput cost, not user
latency.

**CPU cost:** ~1,550–2,800 calls/day × ~0.4s ≈ **10–19 minutes of one slot per day**, ~1–3% duty
cycle. Genuinely cheap — *not* the constraint.

**Engineering cost:** a new `llama-cpu-rag` HTTP client + circuit breaker + its own gate; a cache
table + EF migration; the classifier service + prompt; metric + alert + alert test; a D-entry with
a guard test; the prompt-tuning loop implied by 85% accuracy. Comparable in scope to #907.

**What we lose on misclassification [PROBE]:** 15% error rate. 4-in-6 observed errors degrade to
"not an issuer" (correct for the money, harmless if `abstain`-routed). 2-in-6 route junk into the
cascade — **today's behaviour**. The genuine new risk is a real issuer classified `person`/
`concept` and dropped; unmeasured here, and the reason `abstain` must never reject. At 85%
accuracy on a boundary as blurry as issuer/instrument/institution (§4), a "reject on classifier
verdict alone" design is not defensible.

### The paid-resolver pressure condition

The second break-even condition is that the paid resolver must be the binding constraint. [PROM]
`secmaster_gemini_resolver_calls_total{outcome="cap_exhausted"}` = **917.1/24h** — the value rev-1
quoted is exact, though it cited a metric name (`gemini_resolver_cap_exhausted*`) that does not
exist. The 1,500/day cap is still being hit, so this condition **is** currently met.
`live_calls_1h` could not be reproduced at rev-1's 130: the gauge moved 0 → 149 → 85 inside one
hour. It is a rolling burst gauge and **no single reading of it is a citable figure.**

**Correction to rev-1 on `gemini_resolver_live_calls_24h` — the diagnosis is backwards.** Rev-1
called the gauge "misleadingly named" and told future readers to prefer other series. The opposite
is now true. **PR #911** (commit `91101ad9`, merged 2026-08-06T16:15Z) replaced an in-memory deque
— which zeroed on every restart and structurally could not exceed process age, and *that* was the
misleading era — with a rolling 24h window reconstructed from the **durable on-disk ledger**
(`server.py:744-765`). The name is now accurate by construction. Rev-1's reading of 168 was taken
**75 minutes after that deploy created a fresh ledger**; the gauge history shows it pegged at 1500
until 16:20Z, then 3 at 16:30Z, then climbing 68 → 121 → 149 → 168 → 213. The correct note for a
future reader is **"the gauge was warming up"**, not "the metric is misnamed".

**Correction to rev-1 on the gate-skip counter.** Rev-1's "**eighteen lifetime skips**" is not a
lifetime figure — the counter was reset by the 2026-08-05T17:05Z restart, so 18 was ~25 hours of
accumulation read as all-time. Reset-corrected over 7 days (`increase()` across **8 resets**):
**111 skips = 15.9/day**, across **four** reasons, not two — `not_company_kind` 76, `boilerplate`
28, `code_slug` 7, `markup_artifact` 0. Against 20,939 dispatches that passed the gate, skips are
**0.53%** of gate evaluations (rev-1's implied denominators, and the red-team review's 0.24%,
could not be reproduced — the exact share depends on a denominator neither stated).

**The substantive point holds decisively and is what the section should have rested on:** the gate
refuses about one surface in 190, because the precondition can only fire for
`Ticker`/`TickerInQuote` and the NER path hardcodes `CompanyName`.

### Break-even

The classifier must beat **the cheapest alternative that captures the same value**, not the status
quo.

| | measured value | cost |
|---|---|---|
| **§10 deterministic** | closes the unwatched-filter gap; measures the two real gaps (§0) so the next decision is grounded | ~1 day |
| **This spec** | ≥9.8% **of the head-150 sample** (a floor on a biased sample, §2 — not a candidate-population figure) in classes no shape rule reaches, of which the value is *rejection*, which #907 already does more precisely where it applies. **Zero** additional substrate rows, because the substrate path does not run through this seam (§3) | ~1 PR of #907 scale + a permanent 7B dependency in the ingress path |

**Break-even: the classifier pays for itself when the set of candidate surfaces with a live
receiver *that reaches the substrate* grows past roughly 10–15%.** Today that set is
**approximately zero at this seam** — every substrate row comes from the GPU news-signal
classifier over the whole article, which this component does not touch. Rev-1 put us "a factor of
3–5 below break-even" using a 3.4% figure that measured the wrong thing. The correct statement is
that **the ratio is not defined at this seam yet**, because the seam has no substrate receiver at
all. It acquires one only if §10.1 succeeds.

---

## §9 REASONS NOT TO BUILD THIS

1. **The routing prize routes to a prompt table, not to the substrate.** Step 5 returns
   `Ticker`/`AtlasSectorCode`/`NaicsCode` = null; the only prepass consumer that does not filter
   null-sector entities is the prompt renderer (§3). The 3,879 substrate rows/7d arrive through a
   GPU classifier the ingress filter never touches. **This is the decisive reason and it did not
   exist in rev-1 — rev-1 argued the opposite.**
2. **The real gap is upstream and unreachable from here.** ~60% of articles ground zero entities
   (43–75% daily); 74% produce zero macro signals; the `max_candidates` cap discards 23% of
   candidates in raw document order, sheds cleaner material than it keeps, and hides 4 of the 9
   receiver-matching surfaces outright (§0, §3). A component placed after the sidecar cannot
   address any of it.
3. **The binding constraint is receiver coverage, not classification.** Exchanges and regulators
   have no receiver and no defined semantics on a macro platform. Classifying them is a more
   expensive reject.
4. **85% accuracy is not enough to reject on, and this follows from a standing precondition, not
   from a measurement.** D-11/D-14 PRECOND: a false-positive *silently drops a real resolution*,
   which is worse than junk the $0-capped resolver refuses. A 15%-error classifier can therefore
   only ever be advisory — it annotates junk rather than removing it. **This argument is
   independent of any experiment** and is the load-bearing one.
5. **On one measurable head-to-head the 7B lost to the deterministic rule** (`Tech
   Stock`→issuer). *Demoted from rev-1, which called this "the finding that matters".* It is
   **n=1**, hand-picked, and it cannot carry that weight. It is corroboration for claim 4, not a
   finding in its own right.
6. **The free win isn't there** (§a). The spaCy label is conditionally constant on the population
   that reaches the resolver — though only *at today's emission width*, and widening emission
   (claim 2's fix) is exactly what would change that. Coupled, not independent.
7. **It adds a permanent inference dependency to the news ingress path** — a new client, circuit
   breaker, gate budget, cache table, alert — to a pipeline whose failure modes are already hard
   to reason about (D-1 through D-14 on one card).
8. **The cache assumption in the brief is wrong** (§5), so the steady-state call rate is
   1,550–2,800/day, not ~0. Cheap, but permanently non-zero.
9. **#907 has not been given time to be measured.** It shipped recently and deliberately left bare
   acronyms alone. The next cheapest junk reduction is extending its curated sets — same
   mechanism, already tested, already understood.

**The strongest argument FOR building it**, stated fairly: a large share of candidates is a
genuine economic object being sent to the wrong resolver, and almost none of it has anywhere to
go *from this seam*. If receiver coverage is expanded — which is a *data* project
(signal-identity rows, labels and descriptions), not a code one — then a classifier becomes the
thing that makes that investment pay. **Build the receivers first; the classifier is the second
half of that project, not the first.** That ordering is unchanged from rev-1 and is the one thing
rev-1 got right about §10.

---

## §10 SMALLEST VIABLE VERSION

Rev-1's SVV led with "un-bin five central banks". **That item is dropped, not narrowed** — §3
shows it routes to nothing that reaches the substrate, it is worth a handful of candidates per
slice, it displaces real issuers at 0.90 inside a 5-slot cap, it depends on an unnamed
`NameAppearsInContext` guard against live wrong-ticker collisions, and it named the wrong D-entry
to supersede. Replaced by a measurement. In priority order:

### 1. Widen the signal-identity catalog — the only lever on the pipe that actually writes

`NewsSignalClassifier` picks from a JSON-schema `enum` built per call from the loaded catalog
(`NewsSignalClassifierSchema.cs:33-53`) and is grounded by a prompt table rendering
`id — Label: Description` (`NewsSignalClassifier.cs:284-302`). **85 identities; 81 have fired in
30 days.** The enum is saturated: the classifier is using essentially everything it is allowed to
use, and returning `empty` on 74.3% of articles.

- **Primary: add signal-identity ROWS**, and improve the `label` / `description` text of existing
  ones. Those three fields are literally what the classifier sees. Data change:
  `SecMaster/data/signal-identities-v1.csv`.
- **Note carefully — aliases are NOT rendered into the classifier prompt** and are not in its
  enum. Expanding `aliases_json` therefore does **not** widen the live substrate pipe. It helps
  only (a) `MacroSignalIdentityCatalog.TryResolve`, which feeds `TryPlanMacroWrite` — **81 rows
  lifetime, 0 in the last 7 days** — and (b) cascade Step 5, which returns a sectorless prompt
  row. Alias expansion is worth doing for tidiness and for the numeric path, but **it is not the
  high-leverage item**, and the red-team review that promoted it to #1 rested on a
  misattribution of which method writes the `:sig:` rows (§3).
- *Measure before and after:* `sentinel_news_signal_classifier_request_total{outcome}` — the
  `empty` share is the acceptance criterion. If added rows do not move it, the 74.3% is genuine
  no-signal news and the catalog was never the constraint.

### 2. Alert on `sentinel_candidate_surface_filtered_total` — as a RATIO, never a raw rate

The metric for the mechanism doing all the real junk removal, and nothing watches it (§6,
re-verified: zero hits across `deployment/`). Rev-1 specified the item but not the predicate, and
the obvious predicate is a can't-fail alert: a bare `rate(...) == 0` fires on every quiet ingress
period and says nothing about whether *filtering* broke. The failure worth paging on is
**filtering collapses while ingress continues**, so the predicate must be a ratio against dispatch
volume, mirroring `NewsSignalClassifierFailingHard` (`sentinel.yml:305-313`) — same shape,
same `clamp_min` denominator, same volume floor:

```yaml
- alert: SentinelCandidateSurfaceFilterCollapsed
  expr: |
    (
      sum(increase(sentinel_candidate_surface_filtered_total[6h]))
      / clamp_min(
          sum(increase(sentinel_candidate_surface_filtered_total[6h]))
          + sum(increase(secmaster_entity_resolution_candidates_per_request_sum[6h])), 1e-9)
    ) < 0.04
    and
    sum(increase(secmaster_entity_resolution_candidates_per_request_sum[6h])) >= 100
  for: 1h
  labels: { severity: warning, team: extraction, service: sentinel }
```

**The window and the threshold are both measured, not guessed — and the obvious choices are
wrong.** This counter is extremely bursty: 2,254/day over the last 24h but **825/day over 7 days**
(the 24h figure is 2.7x the weekly mean). So the ratio is **26.6% on 24h but 14.0% on 7 days**, and
a threshold picked from the 24h number alone would false-fire constantly. Measured over 6 days at
30-minute resolution:

| statistic | 6h-window ratio |
|---|---:|
| 7-day mean | 14.0% |
| 2nd percentile | 7.02% |
| **observed minimum** | **6.47%** |
| observed minimum 6h candidate volume | 141 candidates |

A `[1h]` window touches 0.0 during quiet periods, so **the window must be ≥6h**. `< 0.04` sits
~40% below the observed 6-day floor and ~70% below the weekly mean — a real collapse, not
variance. The `>= 100` volume floor sits under the observed 141-candidate minimum, so it suppresses
only a genuine ingress stall (a different alert's job) and never a normal quiet period.

Ship with a `promql_expr_test` in `deployment/tests/alerts/sentinel_test.yml` asserting **both**
directions: it fires when the ratio collapses with volume present, and it does **not** fire when
volume itself goes to zero. A test for only the first direction would pass on a can't-fail alert.

*Incidental defect found while specifying this:* the counter's own XML doc
(`SentinelMeter.cs:1002-1006`) lists a **stale** reason set — it omits the four largest live
reasons (`market_jargon`, `truncated_span`, `multiline_fragment`, `bare_corporate_suffix`) and
lists a `garbled_fragment` that never fires. Fix it in the same PR.

### 3. Measure the two gaps in §0 before designing anything else

Replaces rev-1's dropped item 1. Cheap, and it is the measurement every subsequent decision needs:

- **`max_candidates` cap loss is now measured at 22.9%** — 396 slots and **341 distinct surfaces**
  per 205-article slice, binding on 34.1% of documents, and shedding *cleaner* material than it
  keeps (§0). It also hides **4 of the 9 receiver-matching surfaces** outright (§3). What remains
  is to label a sample of the 341 for real issuers, then raise the cap — or, better, give
  `extract_candidates` any ranking at all, since it currently truncates in raw document order.
  **The cheapest single action in this document.** It is listed here rather than at #1 only
  because it needs that one sample first; if the sample comes back rich in real issuers, do it
  immediately and ahead of item 1.
- **Characterise the ~60% `empty` prepass outcome.** Is it articles with no companies (correct), or
  articles whose companies the ORG-only filter missed (`Fortinet`, §0)? Sample and label. The
  answer determines whether the sidecar's label filter should widen — which, per §0, is also the
  condition under which §a's deleted component becomes worth revisiting.
- **Re-run the [INGRESS] measurement on the candidate basis** after any of the above, and
  re-derive §2's composition without the head-150 bias (draw a random sample of distinct surfaces,
  not the head).

**Then decide.** If widening the catalog (item 1) moves the classifier's `empty` share, that is
the receiver expansion §9 says must come first, and this spec should be revisited with a real
break-even ratio rather than an undefined one.

---

## §11 ROLLBACK

Baseline tag `pre-spec-20260806`, with ZFS snapshots and image tags.
`/opt/ai-inference/backups/pre-spec-20260806/MANIFEST.md` records what a restore does **not**
cover — read it before relying on the snapshot; it is not re-derived here.

Nothing in §10 is destructive. Item 1 is a CSV data change behind the existing catalog-refresh
path (and is reversible by reverting the CSV — note the catalog refreshes on a cadence, so the
revert is not instantaneous). Item 2 is a new alert file plus its promtool test. Item 3 is
measurement only. Items 1–2 revert by `git revert` + `ansible-playbook playbooks/deploy.yml
--tags sentinel` (and `--tags alerts` for item 2 — note `project_prometheus_hup_stale_inode`:
single-file bind mounts pin the old inode, so restart the container, do not HUP).

---

## APPENDIX — REPRODUCING THE MEASUREMENTS

All scripts ran read-only against live prod. The method is recorded so a red-team can re-derive
rather than trust.

- **[PROM]** Prometheus has no published host port:
  `sudo nerdctl exec prometheus wget -qO- 'http://localhost:9090/api/v1/query?query=…&time=<epoch>'`
  with `<epoch>` from actual `date -u +%s`. **Always range-query a counter over 7d before quoting
  it** — rev-1's two arithmetic errors (§a, §8) were both instantaneous reads of counters that had
  been reset. Key series:
  `sentinel_entity_resolution_prepass_dispatched_total{outcome}`,
  `sentinel_news_signal_classifier_request_total{outcome}`,
  `sentinel_candidate_surface_filtered_total{reason,mode}`,
  `secmaster_entity_resolution_candidates_per_request_{sum,count}`.
- **[SUBSTRATE]** `sudo nerdctl exec timescaledb psql -U ai_inference -d atlas_data -c "SELECT …
  FROM macro_observations WHERE source_collector='sentinel' …"`. The `:sig:` infix in `source_id`
  distinguishes the news-signal path from the numeric (`{id}:{hash}`) and qualitative
  (`{id}:qual:{hash}`) paths — that string is the D-`:sig:` contract, do not reformat it.
- **[INGRESS]** `SELECT raw_file_path FROM sentinel.raw_content WHERE content_type='text/html'
  ORDER BY collected_at DESC LIMIT 250` → `POST :3109/extract-file {"path": …}` → the live
  `spacy-ner` endpoint `POST :3110/ner` (no host port; resolve the CNI IP at runtime, never
  hardcode it — see `feedback_no_hardcoded_ips`), body truncated to 32,000 chars exactly as
  `SpacyNerClient.ExtractAsync` frames it, so per-document dedup and the cap are applied by the
  service rather than re-implemented. **Report candidates, not mentions** — see §POPULATIONS BASIS.
  Pre-dedup mention counts are not observable over the wire; obtain them from an **ephemeral
  `spacy-ner:latest` replica** (`--rm --network none`, read-only mount) and validate it
  byte-identical to the live service on the same documents before trusting it. Never probe the
  production sidecar for this.

  **Irreproducibility — mechanism and procedure.** The verdict is stated where it is load-bearing,
  in the `[INGRESS]` row of §POPULATIONS; this is the detail behind it. The live trafilatura
  normalizer is nondeterministic: `trafilatura.extract(deduplicate=True)` keeps a **module-level
  duplicate-paragraph LRU in a long-lived process**, so its output depends on what that process has
  already seen. The identical 250-path draw yielded **147 docs on one run and 189 on the next**, and
  the **205** behind this document's figures exceeds both; on a 40-path probe, three consecutive
  passes gave 24 / 15 / 15 non-empty with 22 of 40 paths changing output. **Freeze the extracted text
  to disk and compute every figure from the frozen copy**, and treat any re-run as a *different*
  population. This invalidates rev-1's "the measurement scripts are reproducible from §Appendix"
  promise, and it is the most likely reason two honest measurements of "the same" slice will disagree.
- **[QUEUE]** two draws from `entity_resolution_review_queue WHERE is_open AND kind='CompanyName'
  AND length(context_excerpt)>80`. Each surface re-parsed inside its own `context_excerpt`.
  Failures only — cannot bound a false-drop claim.
- **[PROBE]** `POST llama-cpu-rag:8080/completion` with `json_schema`, `temperature 0.0`,
  `n_predict 24`.
- Catalog/collision checks by direct SELECT against `atlas_secmaster.instruments`,
  `edgar_filers` and `signal_identities` (`aliases_json` is a jsonb column on `signal_identities`,
  not a separate table).

**Known weaknesses of this evidence, stated so a red-team does not have to find them:**

- The [INGRESS] population is a single ~7-hour slice over 4 sources on one day. It cannot
  represent earnings-season, macro-print days, weekends or the overnight session — and per the
  trafilatura note above, it is not even reproducible against itself.
- **[PROM] 24h figures run ~1.4x the 7-day mean.** Every "/day" number in this document taken from
  a `[24h]` window is from a busy day. Where the 7-day figure differs materially (the empty share,
  §0; the filter rate, §10.2) both are given. Where only a 24h figure is given, treat it as an
  upper bound on volume.
- **Counter resets are pervasive** — Sentinel and SecMaster counters reset at 2026-08-05T17:05Z,
  the gate-skip series shows 8 resets in 7 days, and #911 cold-started every `gemini_resolver_*`
  gauge on 2026-08-06T16:25Z. Rev-1's two arithmetic errors both came from reading a raw counter
  as a lifetime total. **Range-query with `resets()` before quoting anything.**
- **The §2 composition derives from the head-150 by occurrence, which is measurably unrepresentative**
  (§2). Its roll-ups over-state non-issuer classes. A random draw over distinct surfaces is the
  correction and is item 3 of §10.
- The type labels are one author's and the issuer/instrument/institution boundary is genuinely
  ambiguous (§4).
- The [PROBE] n=40 is too small for a confidence interval on 85%, and the `Tech Stock` head-to-head
  is n=1 (§9 claim 5).
- The cache rarefaction is cold-start over one slice; the corrected ~33% figure is on a
  differently-selected (adversarial) population behind a deliberately compound key. Neither is a
  clean estimate of what a surface-keyed *classification* cache would achieve. The +22.6%
  fragmentation figure is **cited from the resolver's own D-1 docstring, not re-measured** — the
  cache value payload stores no subject, so distinct-surfaces-vs-distinct-keys is not derivable
  from the DB.
- **`sentinel_secmaster_resolution_total` (8,085/24h) and
  `secmaster_entity_resolution_candidates_per_request_sum` (6,219/24h) disagree** and were not
  reconciled (§8). They count different things; neither is "the" candidate volume without saying
  which side of the wire you mean.
- **9.8% (§2) is a floor by construction**, not an estimate — it assumes the curated enumerations
  covering 41.9% of the head-150 labels are complete.
