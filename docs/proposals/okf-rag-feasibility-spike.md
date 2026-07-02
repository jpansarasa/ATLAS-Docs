# OKF / LLM-Wiki RAG — Local-Curation Feasibility Spike

**Status:** APPROVED (direction) — spec awaiting user review before plan.
**Origin:** user brainstorm 2026-07-02. Investigate whether the LLM-wiki / Open
Knowledge Format pattern is worth adopting for ATLAS RAG, given local-only LLMs.
**Snapshot:** baseline tag `pre-okf-rag-investigation` @ `c558bb52` — revert here if the
approach does not beat the current path. Spike lives on a branch; main untouched until proven.
**Sources:** video `P_E29-87THI` ("Google's OKF: Why a Folder Beats the Vector Database",
transcribed via WhisperService) + Karpathy LLM-wiki gist + Google Cloud OKF v0.1 blog.

---

## THE PATTERN

Classic RAG re-derives knowledge every query: chunk docs -> embed -> vector-search ->
hand the model fresh disconnected snippets. It never remembers; every query starts from zero.

Karpathy's LLM-wiki flips it: the LLM curates a compounding folder of cross-linked markdown
ONCE — one concept per file, YAML frontmatter, an index file, a log file, links form a graph.
Read the finished synthesis instead of recomputing it. Google's OKF v0.1 formalizes this as a
portable spec: a bundle is a folder, every file is one concept, one hard rule (each file states
its `type`), readers forgive everything else.

The video is skeptical and names three catches — all load-bearing for us:
1. A field is not a process — OKF has a timestamp slot but no mechanism to stay current; a
   shared folder rots in a month.
2. LLMs are bad at clean markdown at scale — mangled headers, links to files that were never
   created. Google "fixed" it by ordering readers to forgive the mess.
3. Standardizes the container, not the meaning — the required `type` is free-form ("BigQuery
   Table" vs "table" vs "relational asset" are all valid, all different languages).

## WHY ATLAS IS WELL-POSITIONED

We already run a disciplined LLM-wiki and it refutes catches 1 and 3:
- AGENT_README cards = one concept per file. CLAUDE.md cards-index = the `index.md`.
- `[[wikilinks]]` = the cross-link graph. CARD_TEMPLATE = the schema.
- audit.sh = the lint that catches invented links / missing sections (catch 2, made mechanical).
- hooks + supervisor loop = the maintenance PROCESS Google dropped (catch 1).
- D-entry format (INTENT/PRECOND/GUARD/TEST) = typed meaning, not a free-form label (catch 3).

So if any local-LLM curation is going to hold up in production, our constraints give it the best
shot — we already solved the parts that make these bundles rot.

## THE QUESTION (the crux)

Can our LOCAL models curate an OKF folder FAITHFULLY? Ingest a source, synthesize it into
existing cross-linked pages, keep links valid, not invent files, emit parseable frontmatter, and
detect a contradiction. Every public demo dodges this by using a frontier model. Our curator is
qwen2.5:7b (llama-cpu-rag, CPU) or qwen2.5:32B-AWQ (vLLM, GPU). Catch 2 predicts the 7b fails
clean-markdown-at-scale; the 32b is the real question.

Retrieval is NOT the crux — bge-m3 + pgvector already does fuzzy surface-to-entity matching well.
The wiki pattern's superpower is compounding SYNTHESIS, a different problem than resolution.

## NON-GOALS

- No production wiring, no MCP tool, no pgvector replacement, no SecMaster change.
- No commitment to a winning application surface (entity resolution vs macro synthesis vs
  per-entity news accumulation). The spike answers feasibility; surface selection is a later call.
- No new schema beyond reusing CARD_TEMPLATE conventions + minimal frontmatter (`type` required).

## EXPERIMENT DESIGN (offline, branch-only)

Domain slice (DEFAULT, reskins cheaply — see OPEN SUB-CHOICES): a tiny OKF bundle for 2-3 ATLAS
sectors built from real inputs — a few FRED series descriptions + 5-10 real `raw_content` news
articles + excerpts of docs/atlas-sectorweights-methodology.md. Small enough to hand-verify, real
enough to force genuine synthesis.

Three operations (per Karpathy):
- INGEST (the hard one): existing page + new source -> model rewrites the page.
- QUERY: a question answerable only by cross-referencing 2+ pages -> model reads index, picks the
  file(s), answers from the bundle (not parametric memory).
- LINT: run audit-style checks (dead links, missing `type`, orphans) on the model's output.

Two model arms: qwen2.5:7b (llama-cpu-rag) vs qwen2.5:32B-AWQ (vLLM). Answers not just WHETHER but
WHICH local model. Operational constraint: the 32b is shared with production extraction — run spike
batches at low priority / when extraction is idle; do not starve prod.

Schema + lint reuse our own conventions (CARD_TEMPLATE frontmatter + an audit-style check script),
not an invented format.

## MEASUREMENT (business-outcome, RED-on-failure)

1. Structural validity — % generated files with parseable YAML frontmatter + required `type` +
   ZERO links to nonexistent files. Bar: >=95%. Below this, the model cannot be trusted to
   self-maintain (catch 2).
2. Synthesis fidelity — across a page's ingest cycle over N seeded facts: prior-fact recall >=0.9
   (does not drop what it knew) AND new-fact integration (yes) AND planted-contradiction flagged
   (yes). Judged by the 32b or an Azure-Foundry oracle on a fixed rubric (generate-dense /
   validate-cheap, the loop we already use).
3. Curation cost — median tokens + wall-clock per ingest per arm. No hard bar, but flag if
   per-ingest wall-clock is large (e.g. > ~30s on this slice): "pay once up front" economics
   depend on curation being amortizable, and the 7b runs ~24 tok/s so a full-page rewrite may be
   minutes.

## DELIVERABLE + GO/NO-GO (tri-state, honest)

A short findings doc: structural numbers + synthesis score per arm, cost per ingest, and one of
three verdicts:
- GO-local — a local model curates faithfully -> a full local wiki subsystem is viable; pick a
  surface and design it.
- HYBRID — local can CONSUME a bundle but not MAINTAIN it -> oracle curates offline/batch (Azure
  Foundry), local reads. Still a win: cheap reads, occasional expensive writes.
- NO-GO — neither local model curates well enough and oracle-curation is not worth the cost ->
  revert to `pre-okf-rag-investigation`, stay on bge-m3 + pgvector. A clean negative is a valid,
  cheap result.

## SNAPSHOT / REVERT

- Baseline tag `pre-okf-rag-investigation` @ `c558bb52` (pushed) = the revert point.
- All spike code + the throwaway bundle live on a feature branch; nothing merges to main unless the
  verdict is GO-local or HYBRID and a follow-up design is approved.
- The spike itself is non-destructive (scratch bundle, offline inference), so the tag primarily
  protects the downstream production phase — taken now because it is free and correct.

## OPEN SUB-CHOICES (for user review)

- Domain slice: DEFAULT = sector-knowledge synthesis (highest value if it works, exercises real
  synthesis). Reskins to (a) SecMaster entity-resolution instrument bundle, or (b) per-entity news
  context accumulation — same harness, different inputs. Change at review if preferred.
- Judge model: 32b (free, local, shared with prod) vs Azure-Foundry oracle (paid, stronger,
  independent of the arm under test). DEFAULT = oracle for the synthesis-fidelity score to avoid a
  model grading its own family; 32b acceptable for a first cheap pass.
