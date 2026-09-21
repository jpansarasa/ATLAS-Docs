# LlmBenchmark/scripts

Python harness scripts for the Sentinel extraction-LoRA acceptance-criteria pipeline. Build the eval substrate, run a vLLM-served model against it, and score against the pinned acceptance metrics.

## Files

| Script | Purpose |
|---|---|
| `build_eval_substrate.py` | Deterministically builds the evaluation substrate (positives + negatives) for the Sentinel extraction acceptance-criteria harness. Writes the merged substrate plus a sidecar `criteria.json` documenting construction. Substrate itself is ~10 MB (10,185,438 bytes, measured 2026-09-04) and lives under `/opt/ai-inference/training-data/eval-substrates/` (not committed). |
| `run_model.py` | Drives a substrate through any **OpenAI-compatible** endpoint (vLLM, SGLang, llama.cpp `/v1`) and emits the predictions JSONL `eval_harness.py --predictions` consumes, plus a provenance sidecar recording the engine build, the **sha256 of the prompt and schema files whose bytes actually reached a request**, and of the chat template string it applied (a path is not a prompt — see below). Takes the same **`--task`** as the scorer and must agree with it: `cove` (default) keeps the extraction array under `predicted_extractions`, `cod` keeps production's stage-1 object under `prediction`. Stdlib only. |
| `eval_harness.py` | Scores predictions against an eval substrate, on **either of two extraction tasks** (`--task`), plus `--task attach` (which catalog instrument each gold owner landed on -- see **Scoring attachment** below). `cove` (default): the 18 pinned metrics over the v6.2 substrate's own `{text_quote, value, period, certainty}` shape — a task production does not run. `cod`: production's CoD stage-1 shape `{article_type, entities, numbers, events, claims}`. Both scorers are pure functions (no vLLM / GPU / network — unit-testable); they score `--predictions`, or `--mock` gold-tautology predictions. It does **not** call a model — that is `run_model.py`'s job, and keeping them apart is what keeps the scorer offline-testable. |
| `attach_candidates.py` | The FROZEN candidate file shared by `run_model.py --task pick` and `eval_harness.py --task attach`: its loader/validator (owners `E1..E60`, each owner's own list `C1..Cm` with m <= k <= 20, uuid ids, the four coordinate axes required, `catalog_snapshot` an ISO-8601 UTC timestamp, **no owner marked `degraded`** unless the caller passes `allow_degraded`), the `{{owners}}`/`{{candidates}}` block rendering, and `pick_schema()`, which `LlmBenchmark/attach-pick/pick_schema.json` must equal. Stdlib only. See **Scoring attachment** below. |
| `build_attach_substrate.py` | Converts `attach-gold/attach_corpus_v1.json` (an OBJECT keyed by `record_id`) into the LIST-shaped substrate `run_model.py` iterates. The conversion is a change of IDENTITY, not a reshape: every consumer joins on `(source_file, source_index)` and `record_id` reaches none of them, so the gold row -- the only artifact carrying both -- is what pairs them. Refuses (exit 2, named) a corpus entry whose `content` no longer hashes to its own recorded `content_sha256` (the article was edited after the corpus was built, so the gold's labels describe the OLD text), two gold rows claiming one join key, a gold article the frozen candidate file does not cover (`run_model --task pick` would refuse the run), a frozen article the gold does not label (silent everywhere else), and four input-shape problems. Writes only after re-reading the file through `eval_harness._load_substrate` -- the consumer's own loader, never a second copy -- and re-joining it against the gold and the frozen list. **Twelve known-bad controls run before every conversion** (`--selftest`, not optional), driving this same CLI in a subprocess and asserting OUTPUT and EXIT CODE: a CLEAN fixture that must exit 0 with the expected two records -- their join keys, their text AND their `source`, which `record_source_id` turns into the prompt's SOURCE_ID line on all 121 committed rows, so a control that set it and never read it back asserted a proper subset of what the model is sent (the NULL control, without which a red cannot be told from a runner that failed to start, LESSONS L20) -- and one fixture per refusal above, each breaking exactly ONE axis. The two key-direction fixtures are SEPARATE, and each breaks ONE direction, because one fixture breaking both is killed by whichever guard runs first, leaving the other deletable with every control green -- measured on the first version of this file and again on 2026-09-21. **11 of its 16 `raise`s go red when deleted; the 5 in `verify_round_trip` do not** -- they re-read the WRITTEN file, so no INPUT reaches them and they are a writer-bug detector, filed unpinned with that measurement in docs/BACKLOG.md. |
| `build_a0_predictions.py` | Converts the G2 gold's per-owner `baseline` (production's own stored resolution of that owner's rows) into the id-shaped predictions JSONL that scores as **arm A0**. THE REDUCTION IS THE MEASUREMENT, and A0 IS THE BAR gates 1-3 are stated against, so the reduction moves the bar: the alternatives are named in `REDUCTIONS` and built by this same code path (`--reduction`), and `paired_attach_diff.py --a-alt` reports the resulting band. The default `rows_sum` sums `rows` per distinct instrument_id (null, production's abstention, is one of the keys), takes the largest, and breaks a tie toward an ATTACHED id then toward the smallest uuid; it attaches **142 of the committed G2's 526 owners**. Against it, re-derived by running this file's own `reduce_baseline` over that gold: `baseline_first` (whose key includes the METHOD the scorer never sees) **disagrees on 6**, every one at a top-row-count tie, and attaches 138; `any_non_null` (which reads an owner production left unattached 7-to-1 as a clean attachment) **disagrees on 17** and attaches 159; `tie_to_abstention` **disagrees on 10**, the tie set exactly, and attaches 132; `tie_to_largest_uuid` **disagrees on 3** and attaches 142. **Ties are 10 of the 526 and are NOT all abstention-versus-attachment**: 8 put an abstention level with an attachment and 2 tie two ATTACHMENTS, which with the one three-way tie makes 3 decided by the arbitrary smallest-uuid leg. An owner with NO `baseline` is REFUSED by name, never emitted as null: an unknown is not an abstention. Output is `{"attachments": [{owner, instrument_id \| null}]}`, which `attach_schema_valid` returns None for -- A0 is id-shaped, was never a model's structured output, and is NOT MEASURABLE by pass gate 5 by construction -- and the script proves that by RUNNING `resolve_attach_owners` over the written file rather than by asserting it. **Eighteen known-bad controls run before every conversion**: CLEAN (the NULL control), one malformed gold per input-reachable refusal, and the `reduction` fixture driven once per named rule, whose five answer vectors must match AND be pairwise distinct -- a fixture that stops separating two rules names them rather than passing. Its fourth owner is there for a SORT LEG rather than a verdict: flipping `any_non_null`'s row-count ordering left the other three owners' vectors intact while moving 5 of the committed G2's 526 owners and the band's widest wrong-attachment rate from 0.1730 to 0.1749. **10 of its 14 `raise`s go red when deleted**; the 3 in `verify_scorer_reads_it` are a writer-bug detector no input reaches (docs/BACKLOG.md). |
| `paired_attach_diff.py` | The **paired** article-cluster bootstrap of the difference between two attach arms, which `eval_harness.py` does not compute ("paired A0 comparisons are not computed here") and which pass gates 1 and 2 of the candidates-in-extraction-prompt plan are stated on. One draw serves BOTH arms, so the article-composition noise they share cancels; drawing each arm its own sample is the other candidate rule, and it answers a non-zero interval for an arm compared against ITSELF, which is the fixture the `identity` control uses to tell the two apart -- **but only on an owner set whose ARTICLES DIFFER**: on byte-identical cluster rows both rules sum the same numbers and the control reports OK under the very mutant it exists to catch, which it did until 2026-09-21. An empty denominator is **NOT MEASURABLE**, carrying the same `not_measurable_reason` shape `score_attach` writes. `--a-alt NAME=PATH` (repeatable) adds ALTERNATIVE baseline arms -- the other `build_a0_predictions.py --reduction` rules -- and the output file and stdout then carry the resulting BAND beside the point estimates; without one, `a_reduction_sensitivity.measured` is `false` with the reason, so a scorecard that never varied the bar cannot be read as though it had. The metric rule is not restated: numerator and denominator per owner come from `eval_harness._attach_indicators`. **Seven controls run on every invocation** -- `identity`, `population` (arm a must MEASURE all twelve metrics, since a metric with an empty denominator was not tested by `identity` at all), `known_sign` (an oracle arm against an all-none arm built from the same owners: recall differs by exactly +1.0, wrong-attachment by exactly 0.0), and, under `--selftest` only, four that drive this CLI end to end in a subprocess asserting its exit code and stdout: `cli_not_measurable`, `cli_sensitivity_band`, `cli_empty_arm` (an arm keyed on a join key the gold does not carry is REFUSED at rc 2; without that refusal the tool REPORTS a comparison against an arm that scored nothing) and `cluster_alignment`. That last one is the only control that can see **which article list the two arms are keyed on**, because it is the only fixture whose arms cover DIFFERENT articles in both directions (arm a 1-3, arm b 2-4): on any fixture where both arms cover the same articles the union-keyed rule and a per-arm-keyed one produce the same output byte for byte. It is also the only NON-degenerate interval any control asserts, so it is the only one that moves when the percentile cut or the resample count moves -- `identity`'s [0,0] and `known_sign`'s [1,1] survive both changes (measured 2026-09-21). |
| `test_run_model.py` | Unit tests over the runner: strict parsing (never salvage), the outbound payload, engine identification, and that a CoD response is **kept** under the key the scorer reads — the contract test imports `eval_harness._cod_object` and asserts the join rather than restating the key. Fully offline — the HTTP boundary is stubbed. |
| `test_eval_harness.py` | Unit tests over both scorers, the scorecard builder and the control arms. Fully offline. One guard test per metric, each constructing the wrong-pairing case; the CoD alignment-key test asserts the OLD key scores 1.0 on it, so the trap it replaces is measured rather than asserted. |
| `check_staleness.py` | Grades the committed scorecards against what is running now: engine build, attribution, age. **One verdict is a pass** (`CURRENT`) and it requires a live comparison actually to have happened — with no reachable `--endpoint` every scorecard is `DRIFT_UNCHECKED` and the exit code is non-zero. Runs a known-bad control over its own classifier first and aborts (exit 3) if that control fails — the control uses its OWN fixed threshold, never `--max-age-days`. A file it cannot examine (unparseable, or parsing with no scorecard shape) is graded `UNEXAMINED` and COUNTED **whatever it is called**, because a file skipped in silence takes the denominator with it; the ONLY files it passes over are the named sidecars `*.criteria.json` and `*.provenance.json`. Not recursive, and that gap is open -- a card in a SUBDIRECTORY is never looked for (docs/BACKLOG.md MEASUREMENT DEBT). |
| `select_cod_gold_corpus.py` | Resolves the 40-article CoD gold corpus out of the substrate from a hand-written selection table, each row carrying why that article is in the set. Keyed on `(source_file, source_index)` and REFUSES a duplicate key -- the index alone is not unique, the substrate concatenates two builds that each number from zero. `--verify` re-resolves and compares content hashes against the committed corpus, so the set is fixed at selection. Stdlib only. |
| `build_cod_gold.py` | Builds CoD **stage-1** gold labels (`article_type, entities, numbers, events, claims`) for that corpus -- the shape production's `cod_json_v1.txt` emits, which the v6.2 substrate's `period`/`certainty` gold cannot score. Stages: `preflight` (identity + a structured-output enforcement probe on both labellers, and the expected bill printed before anything is bought), `primary`, `independent`, `adjudicate`, `assemble`. Hard-capped spend ledger, fail-closed, resumable per article. Reuses `run_model.build_payload` for request construction; needs the `anthropic` SDK, so unlike the rest of this directory it is not stdlib-only. THE LABELLING INSTRUCTION IS PINNED TO PRODUCTION'S PROMPT: three sentences carry the `source_entity` convention (the SERIES owns a macro print, the country does not, `""` is for no ONE named owner) and every stage REFUSES before a paid call if the adjudication instruction, the text `alignability()` would write into the artifact's `blank_by_design.why`, or the prompt stops stating one -- the first build's crosscheck BLANKED two correct rows citing a clause that has since been deleted, and a rebuild on a stale instruction would revert the gold. IT GATES THE PRODUCERS, NOT THE COMMITTED ARTIFACT: the shipped `cod_stage1_gold_v1.json` is never opened, and its `why` states none of the three verbatim while this gate reads green (docs/BACKLOG.md MEASUREMENT DEBT). Opening it here would deadlock the tool -- after a rule change the committed artifact necessarily still states the old rule, so the gate would refuse the very `--stage assemble` that rewrites it. `--selftest` runs eleven controls -- nine mutations cut from production's OWN bullet, one known-good base and the live check -- and aborts at exit 3 ahead of all of them on any of three conditions the controls cannot report on themselves: production's `source_entity` bullet can no longer be LOCATED in the prompt at all (`- source_entity:` .. `Emit each distinct numeric value ONCE`), which would otherwise hand every check an empty rule that every site contains and therefore agrees with; the prompt no longer states a pinned clause, which turns every mutation into one cascade reciting the same absence (measured: 0 of 11 pass, loudly -- the abort buys one message and a distinct rc, not the difference between pass and fail); or the guard's own shape no longer matches `EXPECTED_CLAUSE_NAMES` / `EXPECTED_SITE_NAMES`, literals that derive from neither structure under test, because emptying `SOURCE_ENTITY_CLAUSES` emptied the mutation set with it and printed `2/2 controls behaved as required` at rc 0 with the gate fully disabled. A run that ends in a control count other than eleven fails on its count for the same reason -- the code compares `!=`, so a set that grew unnoticed is as red as one that shrank. Needs no corpus and buys nothing, and it no longer waits for someone to remember: CI runs this `--selftest` and `verify_cod_gold.py`'s and `rescore_alignment_keys.py`'s on every push or PR touching these scripts, the CoD prompts or the gold (`.github/workflows/python-tests.yml`). WHAT IT STILL DOES NOT PIN -- clause TEXTS, site PROVENANCE, and the `main()` wiring itself -- is measured in docs/BACKLOG.md MEASUREMENT DEBT. |
| `verify_cod_gold.py` | Verifies a gold artifact with `jsonschema` against production's schema file -- deliberately NOT `build_cod_gold`'s own validator -- plus content pinning, origin parity and the recorded cap. Also checks what jsonschema CANNOT see. A required identity field present and BLANK is schema-valid and aligns with nothing, not even a copy of itself. And the ANCHOR check, which is the reason this file exists: `numbers.source_entity` must be a name copied exactly from that article's OWN `entities[]`, and on an `analyst_action` article it must not be the `analyst_firm` that PUBLISHED the figure. Article 429 shipped 23 gold-price forecasts anchored to the banks that published them and nothing in the build complained; a wrong anchor grounds a figure to the wrong instrument, which is the one defect class here a scorer cannot see downstream. `--selftest` runs ten controls -- eight known-bad mutations (bad enum, over-long string, an extra `period` key, a dropped origin entry, an unpinned hash, a blank `event.subject`, a `source_entity` off its own `entities[]`, an analyst firm anchored on an analyst action) each of which must be caught by name, plus two NEGATIVE controls that must stay silent: a blank `numbers.source_entity`, which production's prompt specifies when no ONE named entity owns the number -- three cases, and a macro print is NOT one of them, the series owns that, and that SAME firm anchor on a non-analyst article, where a firm may legitimately own its own number. Without the second, the anchor check would pass just as well written as a bare `ent_type == analyst_firm`, which would condemn every correct firm-owned figure. |
| `build_cove_gold.py` | Builds CoVe **stage-2** gold (`text_quote, value, period, certainty, source_entity, ...`) over the SAME 40 articles, at production's stage-2 decoding — **16,384 completion tokens and no `repetition_penalty`**, which is `ChainOfVerification` -> `GenerateStructuredAsync` with neither override, and NOT stage 1's 4,096 + 1.1. It gives `--task cove` a gold that is not the v6.2 substrate's own unaudited `output` arrays. Production's prompt reaches the labeller through the per-record `instruction` and not `--prompt-file`: `render_prompt` substitutes `{{article_text}}` and nothing else, so `{{source}}` and `{{content_type}}` are substituted here and `{{content}}` is rewritten to the placeholder — the document then lands INSIDE the template where `GetInitialExtractionPrompt` puts it. The request schema is parsed out of `ExtractionSchema.cs`; there is no JSON copy of it in the repo and committing one would be a second definition that drifts. Stages: `preflight` (identity, an enum+pattern enforcement probe AND a probe of production's own ARRAY-rooted schema, plus the bill printed before anything is bought), `primary`, `crosscheck`, `assemble`. Hard-capped fail-closed ledger, resumable per article. THE CROSS-CHECK ARM WAS CHOSEN BY A SCREEN ON THE REAL TASK and four candidates failed it (each recorded in `REJECTED_CANDIDATES` with its measurement) — a toy sentence passed two models that returned `[]` on a real article. Cross-family by construction, and deliberately neither the incumbent's family (Qwen) nor the swap candidate's (Gemma). Needs `jsonschema`; no `anthropic` SDK. `--selftest` runs twelve controls over the two pure functions the paid stages rest on — five known-bad admissions (a foreign quote, a blank quote, a null `value`, a string `value`, a figure the article never states), three NEGATIVE controls that must be ADMITTED (a real item, a re-wrapped quote, and the prompt's own mandated `500 MW -> 0.5 GW` conversion, whose value is not a literal and which a careless digit gate would reject), the scoreability classifier tested at inputs DERIVED from its own rule rather than at restated literals, and the collision counter. It aborts at exit 3 ahead of all of them if `ExtractionSchema.cs` no longer parses to production's CoVe shape or `initial_extraction.txt` no longer carries `{{content}}` / `{{source}}` / `{{content_type}}` — either would make every control vacuous rather than red. Buys nothing, needs no corpus and no credential. |
| `verify_cove_gold.py` | Verifies the CoVe gold: `jsonschema` against production's schema **re-derived from `ExtractionSchema.cs`**, content pinning, origin parity, the recorded cap, and the check that matters most — FRESHNESS, because the schema is a DERIVED artifact and one that outlives its source describes a shape production no longer emits, with both files still parsing. Also the two things jsonschema cannot see: a `text_quote` present and blank (schema-valid, and the scorer aligns on that field ALONE so it matches nothing), and a quote that is not a substring of its own article — the only invention test available without a paid adjudicator. `--selftest` runs fifteen controls: twelve known-bad mutations each caught by name, and three NEGATIVE controls that must stay silent — a null `source_entity` (nullable in production's schema and null on most macro figures), a quote re-spaced but real (the substring test normalises whitespace on purpose), and an article with no figures at all (the corpus holds ten by design, and a checker that condemned them would condemn every negative in the set). |
| `build_attach_gold.py` | Builds the **attachment gold** (candidates-in-extraction-prompt plan, story 1) into `LlmBenchmark/attach-gold/`: which catalog instrument owns each owner an extraction named, as `{accept: [instrument ids], verdict: INSTRUMENT \| NONE_IN_CATALOG \| NO_SINGLE_OWNER, catalog_at, rationale}` in the row shape `eval_harness.py --task attach` reads. **G1** `attach_gold_g1_v1.json` = the 165 (article, `source_entity`) owners of the CoD gold, keyed by its own `source_file`/`source_index`, all labelled. **G2** `attach_gold_g2_v1.json` = 121 production articles from the 30 days to 2026-09-17T11:00Z, keyed `("sentinel.raw_content", raw_content.id)`, stratified in a fixed draw order (D-33's 197 GeminiFallback ids, fed, yield/index/commodity/FX/country owner surfaces, each live method, rss-mirror, searxng-content), with each stratum's population recorded for re-weighting: 526 owners, all labelled, plus 179 empty-owner rows under `empty_owner_rows` (outside `owners`: production makes no pick for an ungrounded owner, D-15) and production's stored attachment per owner as `baseline`. `unresolved_owners` is empty in this build, and `assemble` exits 1 unless it is: the scorer reads only `owners`, so an owner left there is not skipped. It is ABSENT from gold, and an arm that attaches its surface is booked a NOT_IN_GOLD invented-owner attachment, counted wrong (PR #1064's review probe: an arm giving the then-unresolved PPI owner PPIFIS scored wrong_attachment_rate 1/522). Article text is in `attach_corpus_v1.json`; the ids each owner's labellers were shown are in `attach_pools_v1.json` under its `unit_id`, so a gold that lacks a row can be told from a wrong pick. Two labeller families on the HF router at deepinfra (DeepSeek-V4-Pro; Qwen3.8 at `reasoning_effort` low, measured to leave six answers unchanged at under half the tokens) see the article, each owner's quotes WITH the values the extraction took from them, and a SQL-built pool (exact symbol/alias, owner-token ILIKE, trigram top 50, bge-m3 top 20 by exact scan); a disagreement goes to a blind third call. **A NOT_IN_POOL answer never becomes NONE_IN_CATALOG on the labellers' word** (`decide`): the terms they sought are searched in the WHOLE catalog, pool included, and a STRONG match (exact symbol/alias, trigram >= 0.6, cosine >= 0.8) sends the owner to a round-3 adjudication shown the matched rows first; round 3 can reject a trigram or vector match, but an EXACT match it rejects leaves the owner unresolved until a hand override settles it; an override may say NONE_IN_CATALOG against an exact match only by naming that symbol in `rejects_exact` (10 such collisions in this build, e.g. MICH for a New York Fed survey figure, CT from "Hartford, CT"). The first build searched outside the pool only and promoted every surviving NOT_IN_POOL, so sterling became NONE with DEXUSUK found exactly and "$36 trillion national debt" with GFDEBTN in the pool; `labeller-runs/notinpool.json` is that round-1 check, `matchcheck.json` the whole-catalog one. "Returned a row" is not a match: the vector leg always returns five, and it matched 307 of 307 answers. Before writing, the check runs a live probe through its own code path and refuses (naming the failure) unless DEXUSUK, sought by symbol and placed in the pool, comes back in-pool and strong, and unless "USD/JPY exchange rate" exact-matches no USD row. Each was proven by breaking it in a scratch copy: pool rows dropped, "/" split restored. Thirty-two owner labels were re-judged by hand against the article and SELECTs and live in `attach-gold/overrides_v1.json`, one rationale each (20 from PR #1064's rounds, 12 more in the follow-up: two sector ETFs refused under rule 5, six accept sets cut to the home listing under rule 3, two FRED twins added, one owner moved off a mis-copied quote onto its own commodity, and the one label the catalog drift changed); `assemble` refuses an entry whose ids do not resolve to its symbols. **Controls, in `attach_controls_v1.json` and every gold header:** the recon's 50 hand labels (identity 48/50 final; 43 and 44 per arm, capped at 44 by the pool); the same 50 with the hand row REMOVED (0 forced neighbours final, 8 from DeepSeek alone; a pick inside the unit's own full-pool accept set counts as a twin, and each record's `removed_row_control` carries the three answers so the 4 twin exemptions can be audited); the whole-catalog check's recall of the removed row (41/44, in-sample: the floors were chosen on it); no NONE_IN_CATALOG owner against an exact match in the assembled gold (0); the whole-catalog check still returns the in-pool row for three fixture owners, sterling (DEXUSUK), the national debt (GFDEBTN) and Jumbo SA (BELA.AT), which is the control a check that stops returning pool rows fails while every other control stays green (shown by a scratch run); no unresolved owner (0); inter-labeller identity agreement 0.75 (G1) / 0.74 (G2), verdict kappa 0.75 / 0.67. The bill is printed before any purchase (`bill`, `adjudicate --round 3 --bill-only`); the ZZZ probe and a 66-control `--selftest` (run in CI as its own step) (including `decide` refusing NONE against an exact match, the pool-leg fixture control and "USD/JPY" never becoming a USD symbol term, each mutation-checked) run before every paid batch; a hard cap reserves each call's worst case on one ledger (`labeller-runs/ledger.json`, $13.55 over 1,412 calls for this build). `staleness` reads the committed G1 and G2 files, SELECTs every accepted id of `owners` and `empty_owner_rows` from `atlas_secmaster.instruments`, and exits 1 listing each unit whose accepted id is missing, inactive or retired; `--attestation <file>` writes the record `eval_harness.py --task attach --staleness` REFUSES to score without, so it is no longer a step someone remembers (`assemble` also records the same check in its header). It exits 1 on a re-queue AND on a gold whose check examined no labelled owner or no accepted id -- "0 stale of 0 examined" is what a gold read through the wrong key also reports, so it is not a pass at either end. It does NOT detect a row added since `catalog_at` that would now own a NONE owner; the pool `surface`, `symbols_searched`, the pool ids and each labeller's `sought` terms are recorded for that check, and `catalog_drift_v1.json` is that check run over a window, on the same three legs and floors as the whole-catalog check. It found exactly one such row on 2026-09-20: DEXUSNZ, created 2026-09-18, which the "kiwi" owner's labellers had sought by symbol and the 2026-09-17 check had not found. Every owner carried the `catalog_at` of the check its round was shown (the pool at 11:12Z, the round-2 check at 12:13Z, the whole-catalog check at 12:56Z, 12:57Z or 13:29Z for an override, all 2026-09-17) until the follow-up re-stamped all 870 to one value, 2026-09-20T12:02:05Z, against the measured gap in `attach-gold/catalog_drift_v1.json` (see **When the 24 h window has already passed** below), and `--task attach` refuses a frozen candidate list whose `catalog_snapshot` is more than 24 h away from any of them. OWNER-LEVEL LIMITS a scorer inherits: an owner whose rows the extraction mis-grouped gets one label for all of them; at most 3 quotes per owner are shown; a quantity that is not a financial measure (a duration) draws NO_SINGLE_OWNER. |
| `test_check_staleness.py` | Unit tests over the staleness classifier, its self-check, and `main()`'s exit code. Fully offline. |
| `rescore_alignment_keys.py` | Re-scores CoD `numbers[]` predictions under ALTERNATIVE alignment keys, which `eval_harness.py` cannot do -- `_number_key` is hard-coded and has no flag. Answers one question only: is a published `numbers_f1` measuring the MODEL or the KEY? It imports the harness's own `_cod_align`, `_token_f1`, `_source_entity_affinity` and both floors rather than re-deriving them, so a change to the scorer moves these rows too. **No row it prints is acceptance evidence** and each says why in its own note: keys holding `value` read 1.0 by construction, and `macro_owner_waived` waives on the GOLD's `ent_type`, which presumes the very question it sizes. Two controls run every invocation: `shuffled_gold` (every prediction scored against a different article's gold; near-zero or `FLOOR_BREACHED`, judged PER RUN and naming the offending one -- a mean across runs diluted a 0.4570 breach to 0.0457 and printed `FLOOR_OK`) and, with `--scorecard`, `reproduces_published_numbers_f1` (the committed-key row must equal what the harness published for the same run, to 1e-9). `--selftest` mutation-verifies both on a synthetic corpus it builds -- fourteen controls, each complaint required BY NAME, three of them NEGATIVE controls that must stay quiet and three at n>1 runs, because a defect in how a tool AGGREGATES across units is invisible to a mutation exercised on one unit. FOUR pin the figures the tool itself publishes: the verdict must sweep all seven keys (a fixture that breaches on the value keys ALONE, committed at 0.0000), the bar must stay the harness's (a floor at 1.11x it, where every other fixture sits at 3.3x), one run must print `range n/a` and never `0.0000`, and a 3x3 witness where the waiver genuinely COSTS greedy two true positives holds the monotonicity counter and the max-cardinality headroom off zero. It needs no predictions file, which is what makes this tool checkable from the repo alone. Backs the alternative-key table in docs/BACKLOG.md MEASUREMENT DEBT, "The CoD gold cannot yet back a model swap" -- whose PREDICTIONS are not in the repo, the same disclosure the shuffled-gold figures below carry. |

## Typical workflow

```bash
# 1. (Re)build the substrate from upstream training-data inputs.
python LlmBenchmark/scripts/build_eval_substrate.py

# 2. Run a candidate model against any OpenAI-compatible engine.
#    --endpoint works for vLLM (:8000), SGLang, and llama.cpp's own /v1 (:8080).
python3 LlmBenchmark/scripts/run_model.py \
    --substrate /opt/ai-inference/training-data/eval-substrates/<dated>.json \
    --endpoint http://localhost:8000 \
    --model google/gemma-4-31B-it-qat-w4a16-ct \
    --out /tmp/preds.jsonl

# 2b. Same runner, HOSTED endpoint -- how a candidate is scored without a GPU reload.
#     --api-key-file takes a PATH, never the token: a token on a command line is in the
#     shell history, in ps output and in every transcript. It is never written to provenance.
#     A router is UNIDENTIFIABLE by construction (it does not expose the provider's engine
#     build), so --allow-unidentified-engine is required and honest there -- and the
#     `:provider` suffix is then the only attribution the scorecard can carry, which is why
#     the runner REFUSES a router model without one. Strict-schema enforcement is a
#     per-provider property: unpinned, the router picks, and it need not pick twice alike.
#     --task cod is REQUIRED to keep a CoD response: production's prompt returns one
#     OBJECT, the default `cove` reader expects an ARRAY, and without the flag the runner
#     writes `predicted_extractions: []` for every record and still exits 0.
#
#     THIS ROUTE CANNOT PRODUCE MODEL_ACCEPTANCE EVIDENCE, and the reason is not the
#     unidentified engine. Measured 2026-09-05: router.huggingface.co answers 404 to
#     /v1/completions, so `--endpoint-mode completions` -- production's own wire shape,
#     client-side template and all -- is unavailable there and the run falls back to the
#     chat endpoint, which is a DIFFERENT code path from the one production runs. The
#     scorecard says so on its own: `acceptance_evidence.production_prompt_path: false`.
#     Use this route to compare candidates without a GPU reload; take acceptance evidence
#     on a local engine that serves /v1/completions.
#     Also measured: concurrency 12 drew 504s from the router on 11 of 12 calls. Keep
#     --concurrency low here; the default 8 is tuned for a local engine, not a gateway.
python3 LlmBenchmark/scripts/run_model.py \
    --substrate /opt/ai-inference/training-data/eval-substrates/<dated>.json \
    --endpoint https://router.huggingface.co \
    --api-key-file ~/.hf-inference \
    --model Qwen/Qwen3.8-27B:deepinfra \
    --allow-unidentified-engine \
    --task cod \
    --prompt-file SentinelCollector/src/cod-prompts/cod_json_v1.txt \
    --schema-file SentinelCollector/src/cod-prompts/cod_json_schema_v1.json \
    --max-tokens 8192 \
    --out /tmp/preds.jsonl

# 3. Grade it. --adapter-meta carries the engine build into the scorecard.
python3 LlmBenchmark/scripts/eval_harness.py \
    --substrate /opt/ai-inference/training-data/eval-substrates/<dated>.json \
    --predictions /tmp/preds.jsonl \
    --adapter-meta /tmp/preds.jsonl.provenance.json \
    --model-label "qwen2.5-32b-awq @ vllm-0.19.0" \
    --out /tmp/scorecard.json

# 4. Are the numbers we already have still current?
#    --endpoint is what makes CURRENT reachable: without a live engine to compare the
#    recorded build against, nothing has been checked, so nothing passes. Run it without
#    one and every scorecard verdicts DRIFT_UNCHECKED and the exit code is 1 -- that is
#    the tool working, not a corpus full of stale numbers.
#
#    THIS INVOCATION CANNOT EXIT 0 ON THE COMMITTED CORPUS, and that is not a bug to chase:
#    baseline-20260510-mock.scorecard.json is a gold-tautology harness check, so it verdicts
#    MOCK -> stale, permanently. Read the per-row table, not the exit code, when running it by
#    hand; the exit code is for a corpus of real measurements.
python3 LlmBenchmark/scripts/check_staleness.py \
    --scorecards LlmBenchmark/eval-substrate \
    --endpoint http://localhost:8000 --endpoint http://localhost:8080

# 5. Unit-test the grader and the runner (all offline, no pytest needed).
python3 LlmBenchmark/scripts/test_eval_harness.py
python3 LlmBenchmark/scripts/test_run_model.py
python3 LlmBenchmark/scripts/test_check_staleness.py
```

## Scoring production's CoD stage 1 (`--task cod`)

The default `cove` task grades the substrate's own numeric shape. **Production does not emit
that shape.** CoD stage 1 emits one object with four list-valued keys, defined by
`SentinelCollector/src/cod-prompts/cod_json_schema_v1.json`, and `period` / `certainty` /
`text_quote` are not in it — they arrive from later pipeline stages. `--task cod` scores
that object on its own terms, against
`LlmBenchmark/eval-substrate/cod-stage1.criteria.json` (**provisional, not
ratified**; the ratified CoVe file is deliberately untouched because eight committed
scorecards cite it as their provenance).

**`--task cod` is a flag on BOTH halves, and they do not check each other.** The runner
decides which shape it KEEPS and under which key (`prediction` for CoD,
`predicted_extractions` for CoVe); the scorer decides which key it READS. A mismatch is
not an error on either side — the scorer simply finds nothing under the key it reads and
reports a model that extracted nothing. `provenance.task` records the runner's half, so
`--adapter-meta` is what lets a reader catch the disagreement afterwards.

```bash
# 1. PRODUCE CoD predictions. Without --task cod this writes `predicted_extractions: []`
#    for every record and exits 0 -- the request is right, the response is discarded.
#    --task cod REFUSES to run without --prompt-file and --schema-file (or
#    --no-structured-output): it changes only how the RESPONSE is parsed, so left alone
#    the request still carries the substrate's CoVe instruction and the array-shaped
#    default schema, and the model would comply and score zero.
#    EVERY SCORED SAMPLING AXIS IS SPELLED OUT rather than inherited, which is why three flags
#    below carry a value the parser would have supplied anyway. A DEFAULT IS WHAT LET THREE
#    AXES DRIFT AT ONCE: --concurrency defaults to 8 against a scored 6; --max-tokens defaulted
#    to 4096 while this block asked for 8192; --repetition-penalty defaults to None, meaning
#    the knob is NOT SENT, while every scorecard records 1.1. Two of the three were still
#    wrong after the round that fixed the first, because a default is invisible in the command
#    a reader sees, moves with the parser, and never sits beside the number it must match.
#    THE VALUES BELONG TO THE SCORECARDS: max_tokens 8192 (D-30, 2026-09-13; the twelve
#    acceptance arms recorded 4096), repetition_penalty 1.1, temperature 0.0, seed 42,
#    concurrency 6 -- what the cap8192-* arms and, at 4096, all twelve arms on
#    measure/common-coordinate-latest recorded under
#    acceptance_evidence.request_sampling.recorded, and what D-29's PRECOND carries. Read them
#    there, not here. At any other value the run is not wrong, it is UNCOMPARABLE to the row it
#    is checked against -- a re-score under MODEL_ACCEPTANCE, not a tune.
#    4096 WAS MEASURED SUFFICIENT FOR THE GOLD, not assumed: 480 CoD responses across those twelve
#    arms all finished `stop`, 0 truncated, and no completion in six later arms exceeded 3,888
#    tokens. IT WAS STILL TOO LOW FOR PRODUCTION (D-30): Gemma 4 truncated 2.3-3.2% of weekday
#    articles at 4096, which the gold cannot show, so the budget moved to 8192 on production
#    evidence and the re-score is a same-session control. A cut-off response does fail
#    json.loads and land in schema_invalid, which reads as bad JSON discipline rather than as a
#    budget finding -- so `truncated` in the run summary counts it separately, and a non-zero
#    value there means the GOLD has started to reach the cap (which re-scores).
#    THE UNSENT KNOBS STAY UNSPELLED, because "not sent" has no flag: top_p, top_k, min_p and
#    presence_penalty are null in every scorecard, and naming one would ADD an axis.
#    For the WIDE run that exists to break a sick engine rather than to score a healthy one,
#    see 1b below.
python3 LlmBenchmark/scripts/run_model.py --task cod \
    --substrate /opt/ai-inference/training-data/eval-substrates/<dated>.json \
    --endpoint http://localhost:8000 --model google/gemma-4-31B-it-qat-w4a16-ct \
    --endpoint-mode completions \
    --prompt-file SentinelCollector/src/cod-prompts/cod_json_v1.txt \
    --schema-file SentinelCollector/src/cod-prompts/cod_json_schema_v1.json \
    --chat-template $'<bos><|turn>user\n{0}<turn|>\n<|turn>model\n<|channel>thought\n<channel|>' \
    --concurrency 6 --temperature 0.0 --seed 42 --repetition-penalty 1.1 \
    --max-tokens 8192 --out /tmp/preds.jsonl

# 1b. THE FAULT PROBE -- A DIFFERENT RUN, AND ITS OUTPUT IS NOT A SCORE. One command cannot
#     do both jobs. The fp8_e5m2 fault class needs concurrent decode (>= 2) to appear at all:
#     the engine starts fine, serves ONE request, then faults and stays 503 -- and the deploy
#     gate is a /health wait plus one SEQUENTIAL 1-token completion, so it cannot see it and
#     reports success over a dead engine. Running WIDE is what surfaces that before production
#     serves a request.
#     IT DIVERGES FROM THE SCORED COORDINATE ON THREE AXES, NOT ONE, and naming only the width
#     would repeat the silent-inheritance defect one level up: width 8 where 6 was scored,
#     max_tokens 16384 where 8192 is (it sat at 8192 against a scored 4096 until D-30 moved the
#     coordinate onto it, so the probe moved too), and no repetition_penalty where the scored
#     runs sent 1.1.
#     Wide and long is what keeps decode concurrent for longer, which is what surfaces the
#     fault. So NEVER compare its numbers to a BENCHMARKS.md row or to the acceptance run
#     above; the output is named fault-probe-* for that reason.
#     What you read off it is whether the engine SURVIVED: call errors, 503s, truncation, and
#     whether it finished at all.
python3 LlmBenchmark/scripts/run_model.py --task cod \
    --substrate /opt/ai-inference/training-data/eval-substrates/<dated>.json \
    --endpoint http://localhost:8000 --model google/gemma-4-31B-it-qat-w4a16-ct \
    --endpoint-mode completions \
    --prompt-file SentinelCollector/src/cod-prompts/cod_json_v1.txt \
    --schema-file SentinelCollector/src/cod-prompts/cod_json_schema_v1.json \
    --chat-template $'<bos><|turn>user\n{0}<turn|>\n<|turn>model\n<|channel>thought\n<channel|>' \
    --concurrency 8 \
    --max-tokens 16384 --out /tmp/fault-probe-preds.jsonl

# 2a. SCORE, with gold. Gold rows are {source_file, source_index, gold: {...CoD object...}},
#     as JSONL, a JSON list, or a build_cod_gold.py artifact carrying them under `articles`
#     — which is what the committed gold file below is, and it is read directly. Only
#     records carrying BOTH gold and a prediction are scored, and `coverage` in the
#     scorecard says how many that was.
python3 LlmBenchmark/scripts/eval_harness.py --task cod \
    --substrate /opt/ai-inference/training-data/eval-substrates/<dated>.json \
    --cod-gold LlmBenchmark/cod-gold/cod_stage1_gold_v1.json \
    --predictions /tmp/preds.jsonl \
    --adapter-meta /tmp/preds.jsonl.provenance.json \
    --out /tmp/cod-scorecard.json

# 2b. SCORE, WITHOUT gold. Still worth running: the self-consistency metrics need none, and
#     every gold-dependent metric reports null with a reason rather than 0.0.
python3 LlmBenchmark/scripts/eval_harness.py --task cod \
    --substrate <substrate> --predictions /tmp/preds.jsonl \
    --adapter-meta /tmp/preds.jsonl.provenance.json --out /tmp/card.json
```

A `--limit`ed run writes `<out>.substrate.json` holding exactly those N records — score
against **that** file, not the full substrate, or the join drops every record the run
never made.

### The alignment key, and why it is not `source_text`

The CoVe scorer aligns a prediction to gold on `text_quote` — a verbatim sentence, long
enough to identify *which* fact is meant. CoD's nearest field, `source_text`, is a 1–3 token
numeric literal: two unrelated figures both written `"$15"` overlap perfectly, and every
per-field accuracy downstream inherits that wrong pairing. `value` fails the same way (both
normalize to `15`).

A CoD number's identity is instead **what it measures and who owns it** — `context` and
`source_entity`, the two fields the prompt itself defines that way, both `required` by the
schema so keying on them penalizes a model that omits them. A pair is a candidate only if it
clears *both* floors independently (0.5 each); the mean only ranks among candidates. `value`
and `unit` are deliberately **out** of the key — anything in the key reads 1.0 by
construction, and value accuracy is the number a model swap turns on. Value equality is used
only to break ties between candidates that already agree on identity (a guidance range).

### The two control arms

- **`--mock` is a CEILING that cannot fail.** It feeds gold back as the prediction, so a
  perfect column says the plumbing ran and nothing else. The scorecard says so in
  `mode_note` and records `controls.mock_ceiling.verdict = NOT_A_CONTROL`.
- **`controls.shuffled_gold` is the arm that can fail**, and it runs on *every* invocation
  of either task. Each prediction is scored against gold from a different article (rotation
  by `n//2`, never by 1 — adjacent substrate records are near-duplicates). Its expected
  floor is near zero and is known **without trusting the gold or the model**, so a
  respectable number there means the scorer is broken by construction. `FLOOR_BREACHED` is
  the verdict; a corpus of near-duplicate articles reads the same way, which is why the note
  names both causes.

Measured 2026-09-05, Qwen2.5-32B-AWQ @ vLLM 0.19.0 on production's CoD request shape, all
597 substrate articles: unshuffled `numbers_f1` **0.9993**, shuffled `numbers_f1` **0.0106**
(`FLOOR_OK`). *Provenance of that corpus, stated because it is a caveat on the numbers and
not on the scorer:* it predates `run_model.py --task cod`, so the predictions file it was
taken over came from a one-off generator that is **not** in this repo. The scorer is
unaffected — those are its numbers over a corpus it was handed — but re-deriving them from
the repo alone means re-running step 1 above, which is a full 597-article GPU run.
The mock arm's own scorecard passed 22 of 24 measurable metrics — and both
failures were `json_valid` and `source_entity_referential_integrity`, the metrics that need
no gold and so cannot be flattered by feeding it back.

**The tautology ceiling is not 1.0, and that is a finding rather than a rounding error.**
`events_f1` ceils at 0.9405 and `claims_f1` at 0.9838. The schema's `required` is satisfied
by `""`, so the model emits events with a blank `subject` and claims with a blank `object`;
an item whose identity fields are blank cannot align with *anything*, including a perfect
copy of itself, and scores as a false positive **and** a false negative. That penalty is
invisible and reads as a missed fact, so it is published as
`diagnostics.identityless_predicted_items` — measured `{events: 150/2520, claims: 38/2351,
numbers: 5/7045, entities: 0/5527}`, which accounts for each ceiling exactly.

## Scoring attachment (`--task pick` -> `--task attach`)

Story 2 of the candidates-in-extraction-prompt plan: the harness for scoring which catalog row a
number's OWNER is attached to. The gold is built separately; everything here is offline.

**Candidates are an input, never a live lookup.** `run_model.py --candidates-file` reads a frozen
file and substitutes `{{owners}}` and `{{candidates}}` per record. Provenance records the file's
sha256 (null when no request carried it), `generator`, `generator_version`, `k` and
`catalog_snapshot` under `candidates`. The runner refuses (exit 2, before any inference) a file
missing an axis, a `catalog_snapshot` that is not an ISO-8601 UTC timestamp, a substrate record with
no frozen entry, a prompt placeholder with no file, and a file no placeholder would carry. An
article with no owners gets no call (production makes none) and a `no_call` row, whose
`schema_valid` is **null**, not true: nothing was sent and nothing was parsed.

**Labels are per-owner namespaces.** Each owner's own list is `C1..Cm`, `m <= k <= 20`, and `C2`
means THAT owner's second row. One deduped block per article would need an enum of owners x k: the
committed CoD gold's largest article has 21 named owners, and 21 x 20 = 420 overran the 200-label
enum this harness first shipped. `C1..C20` is the enum size plan M3 compiled and accept-tested under
xgrammar 0.2.3; 1,200 labels (60 owners x 20) was never measured. It also makes another owner's row
unspellable. The cost: a row listed under two owners is printed twice.

```json
{"generator": "...", "generator_version": "...", "k": 20, "catalog_snapshot": "2026-09-17T10:24:55Z",
 "articles": [{"source_file": "...", "source_index": 0,
   "owners": [{"label": "E1", "owner": "Danieli", "quote": "...",
     "candidates": [{"label": "C1", "id": "<instrument uuid>", "symbol": "DAN.MI",
                     "name": "Danieli & C Officine Meccaniche SpA", "asset_class": "Equity",
                     "exchange": "IM", "country": "IT"}]}]}]}
```

`--task pick` keeps `{"picks": [{"owner": "E1", "pick": "C7" | "none"}]}` under `prediction`, with
the committed clean-arm prompt and the FIXED schema (`E1..E60`, `C1..C20` + `none`, one schema for
every article):

**The substrate is built, not hand-written, and arm A0 comes out of the gold.** The attach corpus is an
object keyed by `record_id`; the runner needs a list keyed by the scorer's `(source_file, source_index)`.

```bash
python3 LlmBenchmark/scripts/build_attach_substrate.py \
    --corpus LlmBenchmark/attach-gold/attach_corpus_v1.json \
    --gold <gold.json> --candidates <frozen.json> --out <substrate.json>

# Arm A0: production's stored attachments, scored with no request sent. Its rows are
# id-shaped, so the scorer reports `schema_invalid` as NOT MEASURABLE and pass gate 5 is
# exempt for it BY CONSTRUCTION -- `picks` is false, so the rows_judged-0 refusal does not apply.
python3 LlmBenchmark/scripts/build_a0_predictions.py \
    --gold <gold.json> --candidates <frozen.json> --out <a0.jsonl>

# ... and the same gold under each ALTERNATIVE reduction, which is what the sensitivity
# band below is measured over. A0 is the BAR gates 1-3 are stated against, so the rule that
# reduces `baseline` to one verdict moves the bar.
for r in baseline_first any_non_null tie_to_abstention tie_to_largest_uuid; do
    python3 LlmBenchmark/scripts/build_a0_predictions.py --reduction "$r" \
        --gold <gold.json> --candidates <frozen.json> --out "<a0_$r.jsonl>"
done
```

```bash
python3 LlmBenchmark/scripts/run_model.py --task pick --endpoint-mode completions \
    --prompt-file LlmBenchmark/attach-pick/pick_prompt_clean.txt \
    --schema-file LlmBenchmark/attach-pick/pick_schema.json \
    --candidates-file <frozen.json> --chat-template <production template> \
    --substrate <same> --endpoint http://localhost:8000 --model <served> --out /tmp/pick.jsonl

# The accepted ids are re-checked against the live catalog BEFORE every scoring run. The scorer
# refuses to run without the attestation, so this is a step the run cannot skip:
python3 LlmBenchmark/scripts/build_attach_gold.py staleness \
    --gold <gold.json> --attestation /tmp/staleness.json   # exit 1 = re-label, do not score

python3 LlmBenchmark/scripts/eval_harness.py --task attach --substrate <same> \
    --attach-gold <gold.json> --candidates <frozen.json> --predictions /tmp/pick.jsonl \
    --staleness /tmp/staleness.json \
    --adapter-meta /tmp/pick.jsonl.provenance.json --out /tmp/attach-scorecard.json
    # G1 (the 40-article CoD gold): add --cod-gold LlmBenchmark/cod-gold/cod_stage1_gold_v1.json
    #   --extraction-predictions <the CoD run whose owners were frozen>
```

**Gates 1 and 2 are stated on the PAIRED interval, which the scorer above does not compute.** Two
per-arm CIs cannot answer "the paired 95% CI of the difference excludes 0": overlapping intervals do not
mean the difference straddles zero, and non-overlapping ones are not the test the gate names.

```bash
python3 LlmBenchmark/scripts/paired_attach_diff.py --attach-gold <gold.json> \
    --candidates <frozen.json> --a <a0.jsonl> --b <arm.jsonl> --out <paired.json> \
    --a-alt baseline_first=<a0_baseline_first.jsonl> \
    --a-alt any_non_null=<a0_any_non_null.jsonl> \
    --a-alt tie_to_abstention=<a0_tie_to_abstention.jsonl> \
    --a-alt tie_to_largest_uuid=<a0_tie_to_largest_uuid.jsonl>
```

**The baseline arm is the bar, so its own construction is a coordinate of the result.** Each
`--a-alt` is re-compared against the same fixed arm b, and the output file and stdout carry the
resulting BAND -- per metric, the min and max of A0's own rate, of the difference, and of the CI
bounds -- beside the point estimates. Drop the `--a-alt`s and the record still reports
`a_reduction_sensitivity: {"measured": false, "reason": ...}`, so a scorecard that never varied the
bar cannot be read as though it had. A band entry is `null` when ANY arm could not measure that
metric: a band over a set containing an unmeasured member is not a band.

Measured 2026-09-21, arm A0 under all five reductions against the three B-clean replicates
committed at `LlmBenchmark/eval-substrate/attach-reduced-round-2026-09-21/` (unweighted G2
rates -- neither this tool nor `eval_harness.py` applies the plan's "re-weighted to the 7d
method mix"): **no gate VERDICT changes anywhere in the band.**
EVERY FIGURE BELOW IS RE-DERIVABLE FROM COMMITTED ARTIFACTS, and four of them were re-derived
end to end on 2026-09-21, reproducing the committed scorecards on every metric value. The
exact commands, the digests, and which two artifacts are regenerated rather than committed
(the substrate and the A0 arm, both byte-reproducible from the committed corpus and gold) are
in that directory's `README.md` -- not restated here. The predictions the b-side figures come
from: r1 `d886871bdd59cc47...`, r2 `e9d0523b70a49962...`, r3 `005e321840c9cf20...`. Gate 1
fails on both terms for every arm (b's wrong-attachment rate is 0.1559-0.1578 against a ceiling of
0.0665-0.0865, and every paired CI of the difference straddles 0); gate 2 passes for every arm
(recall 0.8101 against a floor of 0.2316-0.2569, CI lower bound +0.4170 to +0.4465 against
-0.06); gate 3 fails for every arm, and its first term -- none-owner attach rate <= 10% against a
measured 0.2353-0.2388 -- does not reference A0 at all, so no reduction can move it. Flipping gate
1 would need A0's wrong-attachment rate above 0.3118; the widest the band reaches is 0.1730.


**`schema_invalid` is per row, and it can be NOT MEASURABLE.** The plan's pass gate 5 reads "every arm:
0 schema_invalid, 0 truncated, 0 structured_output_fallback", so the scorecard has to be able to count
it. `diagnostics.schema` does: `rows_schema_valid`, `rows_schema_invalid`, `owners_on_schema_invalid_rows`,
and `schema_invalid`, which is **null with a `not_measurable_reason`** when no scored row had a schema to
conform to rather than 0. **Three populations are not measurable, not two**: arm A0's id-shaped rows
(production's stored attachments were never a model's structured output), a failed call, and a `no_call`
row — which carried `schema_valid: true` until 2026-09-21, so an arm of nothing but `no_call` rows
reported a *measured* `schema_invalid: 0` and passed gate 5 having parsed nothing. `rows_no_call` is
counted apart from `rows_without_schema` because they prescribe different actions, and a **pick arm with
`rows_judged: 0` is now refused (exit 2)** rather than scored — A0 is not, since `picks` is false there.
The runner's own per-row `schema_valid` flag wins where it exists and is a BOOL (`null` is the runner's
"no verdict"); a row without one is judged here by `attach_candidates.is_pick_shape_valid`, the same
fallback `--task cod` uses — but never for a `no_call` row, whose placeholder `{"picks": []}` that
fallback calls valid.
A row the schema rejected refuses EVERY owner of its article (`refusals.schema_invalid`) rather than
having picks read out of whatever survived the parse. The third gate term has no measurement because it
has no mechanism: `run_model.py` never retries a request unstructured, so `structured_output_fallback` is
0 by construction whenever `acceptance_evidence.request_sampling.structured_output` is true — and the arm
is a grammar-free one, not a fallback, when it is false.

**Gold** rows are `{source_file, source_index, owners: [{owner, accept: [instrument uuids], verdict:
INSTRUMENT | NONE_IN_CATALOG | NO_SINGLE_OWNER, catalog_at, rationale}]}` (JSONL, a list, or under
`articles`). Refused with exit 2 and the problem named: a missing label key, an unknown verdict,
INSTRUMENT with an empty `accept`, a NONE verdict with a non-empty one, a non-uuid id, a `catalog_at`
without a timezone, a blank rationale, a duplicated article or owner.

**What the scorer refuses (exit 2, named).** A run with no `--staleness` attestation, or one that is not
an `attach_gold.staleness` record, does not carry an entry whose sha256 is the gold being scored (a path
match proves nothing about a file edited since), is over 24 h old or dated in the future, re-queued any
label, examined nothing at all, or examined FEWER owners or distinct accepted ids than the scorer just
parsed out of the same bytes — `measured` is binary, and the wrong-key hazard it exists for yields a
partial count as readily as a zero one. Only `<` denies: the builder counts `empty_owner_rows` and the
scorer does not, so G2 legitimately attests 705 owners against the 526 it scores. Pick predictions
without `--candidates`, or without
`--adapter-meta` recording `candidates.sha256`, or with a `--candidates` file whose sha differs from
the recorded one. A frozen file carrying a DEGRADED owner: the freezer marks one when
`/api/semantic/candidates` answered with a tier missing (a zero-magnitude query embedding drops the
vector leg), so rows that belong in the top `k` are absent and the file still validates — every
candidate-recall figure computed from it understates the catalog. `--allow-degraded-candidates` scores
it anyway and writes every such owner into the scorecard's `candidates.degraded_owners`; `run_model.py`
has no such flag and refuses outright, because nothing can stamp "truncated list" onto a model's
answers after the fact and re-freezing costs minutes. `freeze_candidates.py` poisons one owner of its
own output on every run and fails (exit 2, before writing) unless that refusal fires by name. A gold `catalog_at` more than 24 h from the file's `catalog_snapshot`: the plan's
staleness rule re-checks accepted ids for RETIREMENT before scoring, but nothing catches a row ADDED
between labelling and freezing, and the window bounds that (measured 2026-09-17: 83.7 instruments
created per day over 14 d, peak 121, on 27,391 active). The human output prints whether the file was
verified against the run and the catalog drift; the scorecard carries both, with the gold's
`catalog_at` range.

**When the 24 h window has already passed, re-DATE the gold rather than re-buying it.**
`build_attach_gold.py drift --window-since <iso>` measures the gap and writes
`attach-gold/catalog_drift_v1.json`: which rows the catalog GAINED inside the window could own a labelled
owner, on the same three legs and floors as the whole-catalog check (exact symbol/alias, trigram >= 0.6,
bge-m3 cosine >= 0.8), plus the accepted-id and retirement halves. `assemble --restamp-catalog-at <iso>
--restamp-evidence <file>` then rewrites every label's `catalog_at`.

**The evidence is checked, not believed.** The gate RE-RUNS `drift_scan` -- the same function `drift`
wrote the file with -- over the build's own owners, and `restamp_refusals` refuses unless: the evidence's
`measured_at` IS the new stamp and is no older than `RESTAMP_MAX_EVIDENCE_AGE`; its `window_since` opens
before the oldest stamp in the build; its `scanned_owners` is EXACTLY this build's owner-key set (so a
truncated, falsified or borrowed scan is named, and `owners_scanned` must match that list's length); the
re-run names no owner its `owners_in_doubt` omits; its `additions_scanned` equals the re-run's own live
count; its `floors` equal this file's; the live `stale_ids` is empty; and every owner it puts in doubt
carries an override stamped at the new value -- a doubted owner is re-judged, never re-stamped. The gold
header records the evidence's sha256 and the re-run's own result side by side under `restamp`. Proven
2026-09-20 by tampering with a copy of the evidence: dropping one doubted owner, emptying the list, and
falsifying `owners_scanned` each exit 1 naming the fault. It does NOT touch the harness's 24 h tolerance,
and it deliberately covers nothing created after `measured_at`: that residue is exactly what the window
bounds, so the freeze follows the re-stamp inside it. What the scan cannot see is counted in its own
output (`vector_leg_fell_back_to_the_unit_key`, `owners_with_no_term_at_all`, `vector_leg_blind_to`) and
carried in docs/BACKLOG.md.

**Alignment.** One-to-one. Without G1 inputs, exact after normalising case and whitespace: "Apple"
never matches "Apple Hospitality REIT". For G1, through the CoD number alignment (`_number_key`), so
an extraction that wrote "Apple" can own gold's "Apple Inc."; when an extraction MERGED two gold
owners under one surface, the one with fewer aligned numbers is an extraction miss, never a second
recipient of the same pick. A predicted owner matched twice raises. Each gold owner resolves to one of
`correct`, `wrong` (judged against ITS OWN accept set), `abstained`, `refused` (a label outside THAT
owner's list, an owner skipped or picked twice), `unavailable` (the call failed) or `extraction_miss`
(no predicted owner reached the pick). An unmatched blank gold owner is `abstained`: no pick is made
for `""` by contract. A predicted owner no gold owner matched that was ATTACHED is an invented owner
(`NOT_IN_GOLD`), counted as a wrong attachment. Predictions may instead carry
`{"attachments": [{owner, instrument_id | null}]}` (a baseline such as production's stored rows).

**Metrics** (value, numerator/denominator, article-cluster percentile bootstrap 95% CI): wrong-
attachment rate over gold owners plus invented attachments (an attachment on a NONE owner is wrong),
invented-owner attachment rate on the same base, correct-attachment recall, precision (over all
attachments), and per gold owner: attach rate, abstain rate, abstain accuracy on NONE owners, NONE
owner attach rate, extraction-miss rate, candidate recall of the frozen lists, pick accuracy given
the right row is listed, forced-neighbour rate given it is not. `numbers_f1` (G1) is a diagnostic
point value. The bootstrap record carries the seed (an int; None is refused), resamples, articles,
draws and the sha256 of every replicate value, so two runs can be shown to have drawn alike. Token,
latency and fallback counts are run properties in the provenance; paired A0 comparisons are not
computed here.

**Controls, run every invocation, built from the gold owners alone** so every arm exercises them
identically: `oracle` (last accepted id of each set, NONE owners abstain: recall 1, precision 1,
wrong 0), `all_none` (recall 0, wrong 0), `shuffled_picks` (each owner given an accepted id of
another article, skipping its own set: the wrong count must EQUAL the attachments made, every verdict
class exercised) and `accept_swap` (each owner given the accepted id of ANOTHER owner of the SAME
article: the wrong count must rise from the oracle's by exactly the swaps made, at least one
INSTRUMENT owner swapped). Any verdict other than `OK` writes the scorecard with `valid: false` and
exits 3.

**What the controls do not see.** They judge and aggregate; they do not align or resolve labels. An
alignment defect that hands one predicted owner to two gold owners raises (exit 1). Any other defect
there, or in a metric's population (a miss counted as an abstention, a refusal as an abstention, a
denominator that drops misses or invented owners), leaves every control green and exits 0: unit
tests pin those, and at real size (a 467-owner G2-shaped and a 172-owner G1-shaped fixture) only an
independent re-derivation of the counts shows them.

## Why the runner records the engine build

An inference-engine benchmark decays. Measured 2026-09-03: this box RUNS llama.cpp build
10603 (image built 2026-08-24) while the newest llama.cpp figure in `BENCHMARKS.md` is
2026-04-03, and `BENCHMARKS.md`'s "Backend Comparison" section compares llama.cpp against
**Ollama**, retired 2026-06-11. Nothing in a result file recorded an engine build, so none
of that was visible from the results themselves.

`run_model.py` therefore **refuses to emit predictions it cannot attribute** — if neither
`/version` (vLLM) nor `/props` (llama.cpp) identifies the engine, it exits 2 rather than
producing an unattributable number. `--allow-unidentified-engine` overrides it and stamps
`engine_identified: false` so the gap appears in the scorecard instead of being absent from
it. `check_staleness.py` then compares a scorecard's recorded build against the live one --
and when it *cannot* (no endpoint given, the endpoint did not answer, or the scorecard names
an engine but not a build) it says so with a stale verdict and a non-zero exit. Unknown is
not current: the first version of that file reported 7/8 `CURRENT` and exit 0 on an
invocation with no `--endpoint` at all, having compared nothing.

A **multi-provider router stays unidentified on purpose**, and the override is not a
loophole there. Measured 2026-09-05, `router.huggingface.co` answers 404 to both identity
probes and 200 to `/v1/models` with 138 entries, each carrying a `providers[]` array. The
engine that runs the weights is the *provider's*, whose build the router does not expose —
so the run is unattributable in exactly the sense the gate exists for, and the `:provider`
pin is what supplies the attribution instead (recorded as `provenance.provider`). That is
why the runner refuses a router model with no pin rather than warning about it: the same
`/v1/models` response that makes a bare id *look* valid is the one proving the router will
choose for you.

## Why the artifact records the prompt's digest, not only its path

Same decay, one level down. `provenance.request_shape.prompt_file` is the path **as typed on
the command line**, and until now nothing opened it — so a finished run's prompt bytes were
unrecoverable. Measured 2026-09-06: of 25 scorecards on this host carrying a `request_shape`
at all, **22 share one byte-identical value** and every one stamps
`acceptance_evidence.production_prompt_path: true` — 8 of them scored the prompt as it stood
before #1017 rewrote that same path, 14 after, and the artifact does not change across the
boundary. (Those files live in `/tmp`, not the repo; every scorecard *committed* here
predates `request_shape` entirely.) The prompts are distinguishable in reality —
`SentinelCollector/src/cod-prompts/cod_json_v1.txt` against the host mount
`/opt/ai-inference/prompts/cod/cod_json_v1.txt`, which lag each other between deploys — and
were identical in the artifact. No wording of a caveat could recover that; three attempts
tried.

The runner now records `prompt_file_sha256`, `schema_file_sha256` and `chat_template_sha256`
**alongside** the paths — a path plus a hash is strictly more than a path — and the scorer
surfaces them in `acceptance_evidence.request_bytes`. A digest is written only for a file the
run actually **read**: `--no-structured-output` names a `--schema-file` and sends none of it,
so that run records the path and a `null` digest rather than attesting to bytes nothing sent.

A digest is written **only for bytes that reached a request**, and null otherwise — `--limit 0`,
`--no-structured-output`, a schema file whose JSON parses falsy (which `build_payload` silently
replaces with the built-in), a chat template outside completions mode. A digest reads like
proof, so one attesting to a file the run merely *named* would be worse than the bare path it
improves on. The scorer's `unrecorded` therefore keys off the **absence of the field**, not the
falsiness of its value: a run predating these digests carries no `*_sha256` key at all, while a
run that has them writes the key always and a null there is the runner *saying* the element was
not sent.

**Nothing here adjudicates, deliberately.** The scorer cannot see the mount production serves
from, so a hardcoded "production" digest compiled into it would be a guarantee it has no way
to keep — the same class of false claim being fixed. A human compares the recorded digest
against the mount, and now has something to compare. `production_prompt_path` is unchanged in
meaning and still narrow: it says the request had production's **shape** — completions mode,
a prompt file and schema file *named*, a template that really wraps — and never says **which**
prompt. A scorecard predating these fields says so in `request_bytes.unrecorded`, and one that
recorded no request shape at all says *that*, rather than borrowing the wording for a run that
sent nothing. An absent digest is not a matching one.

**The note is chosen per element, over three elements that need not agree.** One caption picked
for the whole block states a fact about all three from evidence about one, and every mixed record
is then over- or under-stated. So `request_bytes.note` is cut on what each NAMED element answers,
and the answer is three-valued, not two: its digest is **recorded**, or it is **null with its key
present** (the runner saying this element reached no request), or **its key is absent entirely**
(an artifact predating the digests, named in `unrecorded`). Six states follow:

| state | when |
|---|---|
| no shape at all | the run recorded no `request_shape` |
| shape naming nothing | the instruction came from the substrate |
| **recorded** | nothing named is missing a digest key, and at least one is identified — a named element sitting at null is *not* a gap and does not leave this state |
| **partial** *(new)* | at least one identified **and** at least one named element whose digest key is absent |
| **unverifiable** | nothing identified, and at least one named element's digest key is absent |
| **not sent** *(new)* | every named element carries its key and every one is null — nothing reached a request |

The two new states are the ones the old block mis-captioned. A `--limit 0` run that named
production's prompt was told it "named no prompt file … the instruction came from the substrate";
it is now *not sent*, which is a finding rather than a null. And a mixed record was forced into a
caption true of only one half; it is now *partial*. Separately, `recorded` was **re-worded** rather
than added: it used to send a run that digested only its chat template to check "the prompt and
schema files that served it" — files it never named — and now names no element at all. The wording
also stops treating all three elements as paths: `--chat-template` takes the **template itself**
and the artifact holds it verbatim, so a missing digest there costs the attestation that it
reached a request, never the content.

### And the same blindness on the sampling side

No conjunct of `production_prompt_path` reads a decoding knob either, so the shape can be
production's while the decoding is not. Measured 2026-09-06: benchmark runs sent
`repetition_penalty: null` and `max_tokens: 8192` where the service sent a loop guard
(`CpuCodOptions.JsonRepetitionPenalty` = 1.1) and half that budget -- 4096 then; 8192 since D-30, so
the budget half of the divergence has closed while the penalty half stands -- and every scorecard
stamped `production_prompt_path: true`. A run that decoded differently from production is
not a run on production's path, whatever the shape says — that is the finding, and it stands
on its own.

**What this section deliberately does not claim.** It is tempting to pin the article-35
entity runaway on the missing loop guard. The evidence does not carry it: *both* arms of the
two-arm prompt comparison ran with `repetition_penalty` unset, so a constant cannot explain a
difference that appeared in one arm only, and production's own rationale describes the trap as
one object *repeated*, where article 35 produced *distinct invented* names. `docs/BACKLOG.md`
holds that question open — whether the corrected prompt's `macro_indicator` instruction caused
the runaway or merely uncovered it — and one prompt edit plus one re-run settles it. Recording
sampling earns its place because the gap is real, not because it has been shown to be that
gap's cause.

The runner already recorded `provenance.sampling`; nothing surfaced it next to the pass. The
scorer now carries it as `acceptance_evidence.request_sampling`, under the same rule: it
**reports**, it does not compare. **Not sent is not unrecorded** — the runner omits an unset
knob rather than defaulting one nobody chose, so `repetition_penalty: null` inside a recorded
block means *this run ran without the loop guard*, which is a fact. A knob the artifact never
carried is named in `unrecorded` instead, per knob and not per block: `seed` was added to the
runner at #1009, so every scorecard written before it carries nine keys and no `seed` — judged
on the container alone those read as fully recorded, with the determinism knob a model
comparison turns on simply absent.

**The note now follows that per-knob data**; it used to be chosen for the block as a whole. So one
missing knob reached for the wording "this run wrote no sampling block" — false of a run that wrote
nine of ten. Re-scoring the eight scorecards committed here: **seven were given that sentence**,
which is every one of them produced by a real run. The eighth, the only one the wording fitted, is
the harness's own gold-tautology mock, which called no model. A partially recorded block now says
so and names the knobs it cannot speak for.

## What the run cost

`usage` is captured per record and summed into `provenance.usage` — token counts plus
`estimated_cost` where the provider reports one (deepinfra does). Cost is `null`, never
`0.0`, when nothing reported a price: a provider that does not price and a run that was
free are different facts, and `calls > 0 AND cost == $0` reads as free until the bill
arrives. `cost_reported_by` says how many records actually carried a price, and
`cost_per_record` divides by those rather than by the record count.

## See Also

- [LlmBenchmark](../) — parent project (C# benchmark runner + `BENCHMARKS.md`)
- [SentinelCollector](../../SentinelCollector/README.md) — owner of the extraction LoRA being evaluated
- [SentinelCollector/scripts](../../SentinelCollector/scripts/README.md) — training-data generation + QLoRA fine-tuning
