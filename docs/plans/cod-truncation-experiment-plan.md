# CoD Truncation Experiment — Supervisor Execution Plan

> **Companion to the design spec** `docs/superpowers/specs/2026-06-07-cod-truncation-value-experiment-design.md` (the *why/what* — experiment science + decision rule).
> This plan is *how the supervisor executes it*: the pipeline, dispatch decomposition, artifacts, and per-turn grounding. **Re-read the current-phase section + the spec each turn; track running state in STATE.md.**

## Pipeline (walk backward from the current target)

```
raw_content sample (per stratum, long ≥4K tok)
 └─[P0 harness]→ truncated variants + per-variant CoD extractions + scores (JSONL)
   └─[P2 sweep]→ full (stratum × shape × depth) score matrix
     └─[P3 analyze]→ recall curves + material-miss audit + proposed per-stratum policy
       └─[P4 rollout]→ live pre-CoD truncation (long articles) + dashboard validation
```
Baseline for every score = the article's **stored production full-text `extracted_observations`** (+ a 15-article fresh-control re-run subset).

## Fidelity invariant [HARD]
The harness MUST drive the **real CoD extraction path** (same `llama-server` model, same DSL prompt, same parse) on truncated input — NOT a reimplementation. Only the article-text slice varies. A reimplemented extractor would silently invalidate the whole comparison. [[feedback_benchmark_not_production]]

## Artifacts & locations
- **Harness:** `SentinelCollector/tools/truncation-experiment/` (scripts + a short README) — or a `scripts/` location if cleaner; the harness-build agent decides and records it here.
- **Data + scores:** `docs/benchmarks/cod-truncation-2026-06-07/` — `sample.jsonl`, `variants.jsonl`, `extractions.jsonl`, `scores.jsonl`.
- **Results + policy:** `docs/benchmarks/cod-truncation-2026-06-07/results.md`.

## Phases → dispatch decomposition

**P0 — Harness + matching validation [GATE]**
- Dispatch: (a) *harness-build* agent — per-stratum sampler (long ≥4K tok), paragraph-splitter, variant-generator (head-only + head+tail grids), CoD-runner via the REAL path, scorer skeleton; (b) *matching-validation* agent — hand-label ~25 baseline↔truncated observation pairs, run the LLM-judge matcher, measure agreement vs labels.
- **Done:** harness runs ~5 articles end-to-end; judge-matching ≥ ~90% vs hand labels.
- **NTFY** if matching can't clear ~90% (recall metric untrustworthy → rethink).

**P1 — Pilot, wire/RSS news only [GATE]**
- Dispatch: *pilot-sweep* agent — head-only depth grid on the news stratum (75 articles) → recall curve + miss breakdown.
- **Done:** coherent recall-vs-depth curve for wire news.
- **NTFY** if anomalous beyond a self-fixable harness bug.

**P2 — Full sweep**
- Dispatch: per-stratum sweep agents — news (head-only); analytical (head-only + head+tail); primary-docs (if a distinct stratum emerges). Outputs are disjoint JSONL per stratum, so agents can run in parallel on the *data* side; but the CoD runs share the single `llama-server` — sequence/throttle them (run during low-ingestion windows or accept slower) to avoid starving the production backlog drain.
- **Done:** full score matrix for every (stratum, shape, depth) cell. Autonomous — no gate.

**P3 — Analysis + policy proposal [GATE]**
- Dispatch: *analysis* agent — recall/precision curves per stratum × shape, the material-miss audit (judge), and a concrete proposed policy (per stratum: shape, depth, length-trigger) + projected prefill/throughput gain → `results.md`.
- **Done:** `results.md` + the proposed policy.
- **NTFY the policy proposal** — product decision + gate before any production change. Also NTFY if **no safe truncation exists for a stratum** (hypotheses failed → direction needed).

**P4 — Rollout [USER-GATED]**
- Only after user approves the policy: dispatch implementation (truncation in the pre-CoD path, behind the per-stratum policy, long articles only) → review-fix-loop → merge → deploy (scoped) → validate prefill drop + aggregate-throughput rise on the #638 dashboard → sample-audit live parity. Production-behavior change → user-approved cutover.

## STATE.md tracking (experiment block — keep lean)
Phase status (P0..P4 ✓done/→active/⧗blocked), the long-token threshold actually used, the matching-validation result, per-stratum curve summaries as they land, the proposed policy, the rollout status. Current truth + pointers only — not a per-turn diary.

## Escalation (NTFY gates) — else heads-down
Per the spec's autonomy boundaries, NTFY `atlas-claude-ask` (single-line) ONLY on: (1) P0 matching-validation fail, (2) hypotheses fail for a stratum, (3) the P3 policy proposal, (4) the P4 cutover. Otherwise run autonomously, dispatch all impl/test/build to subagents (worktree-isolate parallel *code* agents), end turns with `▶ continuing autonomously`. The user is not at the keyboard for P0–P2.
