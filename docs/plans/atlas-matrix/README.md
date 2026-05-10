# ATLAS Matrix MVP — Plan Index

This directory hosts the per-epic execution plans for the matrix rework.
The **canonical product plan** is `/home/james/ATLAS/docs/atlas-matrix-mvp-plan.md`
(features, stories, acceptance criteria, decisions D1–D17).
The **handoff** is `/home/james/ATLAS/docs/atlas-matrix-handoff-v2.md`
(philosophy, framing, architectural contract).
The **supervisor hub** is `/home/james/ATLAS/STATE.md`.

The files here describe **how** each epic is sequenced, branched, and
dispatched — they link to the canonical plan rather than restate AC.

## Phase chart

| Phase | Epic(s) | Branch(es) | Depends on |
|---|---|---|---|
| 1 | 1 | `epic/1-secmaster-taxonomy` | — |
| 2 | 2, 3, 4 (Feature 4.6 only) | `epic/2-te-schema-rework`, `epic/3-macro-observations`, `epic/4-sentinel-rework` | Phase 1 merged |
| 3 | 4 (4.1–4.5), 5 | continuation of `epic/4-sentinel-rework`, `epic/5-matrix-storage` | Epic 3 merged + Epic 2 emitting cells |
| 4 | 6 | `epic/6-consumption-surfaces` | Phase 3 merged |

## Status legend (used in STATE.md and per-epic files)

- `◯` not started
- `→` in flight
- `⧗` blocked (note the blocker)
- `✓` done
- `✗` failed / abandoned (note why)

## Epic plan files

- [epic-1-secmaster-taxonomy.md](epic-1-secmaster-taxonomy.md)
- [epic-2-te-schema-rework.md](epic-2-te-schema-rework.md)
- [epic-3-macro-observations.md](epic-3-macro-observations.md)
- [epic-4-sentinel-rework.md](epic-4-sentinel-rework.md)
- [epic-5-matrix-storage.md](epic-5-matrix-storage.md)
- [epic-6-consumption-surfaces.md](epic-6-consumption-surfaces.md)

## Cross-cutting prerequisite

`docs/llm/ACCEPTANCE_CRITERIA.md` — D15 source-of-truth for the LoRA
extraction reliability bar. Must be ratified before Story 4.6.1.
