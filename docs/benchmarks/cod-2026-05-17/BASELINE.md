# CoD DSL Baseline — locked 2026-05-22

**Canonical prompt:** `prompts/cod_dsl_baseline.txt` (byte-identical content
to `prompts/cod_dsl_v8_layered.txt`, plus a header pointer comment).

**v8 layered** is the production CoD DSL baseline. Aggregate verifier
**0.907** / grammar-valid **9/9** on the Arm B n=9 corpus
(qwen3:30b-a3b-instruct-2507-q4_K_M, llama-server v2.1 GBNF, `--n-predict
8192`, `--num-ctx 32768`, `--timeout 1800`). Locked in after **v9** and
**v10** "careful trim" attempts both regressed on cell 33632
(DECLARE-ONCE mechanism collapse → 70+ concept-kind ENTs in v9, similar in
v10). The trim hypothesis was falsified twice; further reduction below the
v8 floor is not pursued.

Historical / reference prompts (kept for provenance, not for production):

| Prompt                                | Status                          |
| ------------------------------------- | ------------------------------- |
| `cod_dsl_v6_leaner.txt`               | reference — 0.895 aggregate     |
| `cod_dsl_v6_context_aware.txt`        | regression — steering displaced |
| `cod_dsl_v6_rag.txt`                  | regression — RAG few-shot       |
| `cod_dsl_v7_even_leaner.txt`          | regression — DECLARE-ONCE broke |
| `cod_dsl_v8_layered.txt`              | **canonical content (= baseline)** |
| `cod_dsl_v9_trimmed_layered.txt`      | regression — trim falsified     |
| `cod_dsl_v10_careful_trim.txt`        | regression — trim falsified ×2  |

The **LLM-strength layering hypothesis** (small CPU CoD for
pattern-recognition + classification; large GPU extraction for structured
emission — see `project_sentinel_llm_strength_layering` memory) is
validated by v8: adding article-type classification as a leading advisory
block (without relaxing any decree) recovered the v6+CA regressions on
33632/31149/35352 while preserving v6's mechanism guards.

Next step is corpus scale-out — see `RECON_corpus_expansion.md`.
