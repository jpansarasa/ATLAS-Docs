# Phase 3 v7 — INVALID-RUN

**Verdict**: INVALID-RUN (infra failure, not a v7 result).

**What happened**: llama-server container removed at 10:44:37 UTC by cap-expansion deploy
(PR #403, merge cecad20b). All 9 in-flight sweep connections dropped within an 820ms
window (10:44:37.890 → 10:44:38.710 UTC). Each cell got `ConnectionError` and the
harness wrote a failure stub (raw_output="", every metric null) instead of retrying.

**What's preserved**: v7 prompt (528 lines, -106 vs v6's 634 lines = -16.7%) committed
at 8b3a0479. The 9 failure-stub JSONs committed at 97bd4d8a are the harness record;
keep them, don't rewrite.

**What we don't know**: aggregate verifier, mechanism preservation (33632 dup-decl,
36065 kind-latch, 27702 punctuation-ident), prompt_eval delta. Cannot be checked
because raw_output is empty across all 9.

**v6 baseline correction**: v6 mean prompt_eval is 8080.7 tokens (recomputed from
canonical v6 results), NOT the ~6749 the original D brief cited. v7 retry must
compare against 8080.7.

**Retry plan**: after E and F complete (avoiding 3-concurrent-sweep server pressure),
re-run v7 on a fresh `experiment/phase3-v7-leaner-n9-retry` branch. v7 prompt unchanged.
