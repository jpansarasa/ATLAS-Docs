# ATLAS Releases

Tagged phase-completion snapshots. Pull the tag to recover the full set of
plans / recon docs / iteration history that was in `docs/plans/` at that
moment:

    git fetch origin --tags
    git checkout <tag> -- docs/plans/

Or inspect a single file at a tag without checkout:

    git show <tag>:docs/plans/<filename>

## Tags

### `pre-docs-consolidation-2026-05-26`
Anchor tag preserving the full `docs/plans/` tree as it existed before
consolidation. Includes every iteration-history plan, recon doc, and spike
note that's now removed from main. Use this tag to recover any of the
deleted Phase 2 / Phase 3 / recon documents.

### `dsl-poc-phase4-done` (@ 93fd94d0)
**Phase 4 GPU semantic verifier — PASS** at recalibrated ≥80% Foundry
agreement gate. 80.64% via prompt iter-2-axis-C + IsSelfReferential
short-circuit on Qwen2.5-32B-Instruct-AWQ. Recalibration rationale in
`docs/plans/atlas-dsl-poc-phase4-gpu-semantic-verifier.md §11`.

## Phase summaries

### Phase 2 — llama.cpp llama-server CPU sibling (DONE 2026-05-21, PR #387 @ 25cb33fa)
llama-server on :11437 serving qwen3-30b-a3b via bind-mounted Ollama blob;
v1.1 GBNF wired through ollama's `format` parameter; runner `--backend`
flag for portability. Outcome: working CPU GBNF substrate, foundation for
Phase 3 schema iteration. Full plan at
`pre-docs-consolidation-2026-05-26:docs/plans/atlas-dsl-poc-phase2-llama-server.md`.

### Phase 3 — v2 schema + word-grounding arc (DONE 2026-05-24)
Multi-PR iteration: v2 baseline (#390) → v2.1 no-offset (#392, #393) →
ENT-ref cleanup (#394) → v6 leaner prompt (#400) → context-aware /
RAG / leaner-prompt sweep (#404–#412) → v2.2 token-grounding shelved
(#425) → v2.3 word-grounding (#426–#428) → v15 compound chunked
(#429, #431, #432). Outcome: word-grounding v2.3.1 + chunked compound
v15 are the leading production candidates; T=0 A/B verdict
DESIGN-SOUND. Deterministic Python verifier ships as
`docs/benchmarks/cod-2026-05-17/dsl/verifier_v2_3_1.py`. Iteration
history at `pre-docs-consolidation-2026-05-26:docs/plans/`
(`atlas-dsl-poc-phase3-v2-schema.md`, `token-grounding-recon.md`,
`three-hypotheses-scoping.md`, `hyp-b-logits-processor-spike.md`).

### Phase 4 — GPU semantic verifier (DONE 2026-05-26, see `dsl-poc-phase4-done` tag)
Tag covers the milestone above. Canonical record (kept on main):
`docs/plans/atlas-dsl-poc-phase4-gpu-semantic-verifier.md`. The 4.7s SFT
sub-arc (`atlas-dsl-poc-phase4-7-sft-cove-verifier.md`) is now subsumed
by §11 of the canonical record; the full sub-plan is preserved at
`pre-docs-consolidation-2026-05-26:docs/plans/atlas-dsl-poc-phase4-7-sft-cove-verifier.md`.

### Phase 5 — Matrix integration (IN FLIGHT)
See `docs/plans/atlas-dsl-poc-phase5-matrix-integration.md` for the
current plan.

## Going forward

Per `CLAUDE.md ## PHASE_TAGS`: tag at phase completion
(`<workstream>-phase<N>-done`), add an entry to this file, and delete
phase-specific iteration-history docs from `docs/plans/`. Reference
docs (architecture, specs, runbooks, schemas, grammars) stay on main.
