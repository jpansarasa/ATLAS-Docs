# Phase 3 v6+RAG retry-v2 — NOTES

**Sweep**: leave-one-out k=2 RAG few-shot, base prompt = `cod_dsl_v6_rag.txt`
**Cells complete**: 9/9  |  **Penalized mean** (gv-fail=0): 21.5%  |  **Scored-only mean**: 96.7%  |  **gv-fails**: 7/9  |  **Δ penalized vs v6-leaner (0.895)**: -0.680
**Verdict**: REGRESSION — penalized_mean=0.215 < 0.87

## Server config (solo mode)
- `--parallel 1`
- `--ctx-size 32768`
- url: http://127.0.0.1:11437
- model: qwen3:30b-a3b-instruct-2507-q4_K_M

## Per-cell results

| id | grammar | verifier | num_exact | org_rec | undecl | prompt_eval | eval | wall | stop |
|----|---------|----------|-----------|---------|--------|-------------|------|------|------|
| 27560 | 100.0% | 100.0% | n/a | n/a | 0 | 10921 | 8192 | 832.8s | limit |
| 27702 | 0.0% | n/a | n/a | n/a | 0 | n/a | n/a | 900.1s | n/a |
| 29807 | 100.0% | 93.3% | 11.1% | 80.0% | 1 | 19410 | 1084 | 491.7s | eos |
| 31149 | 0.0% | n/a | n/a | n/a | 0 | 20619 | 1431 | 657.5s | eos |
| 31430 | 0.0% | n/a | n/a | n/a | 0 | n/a | n/a | 900.1s | n/a |
| 33632 | 0.0% | n/a | n/a | n/a | 0 | 14091 | 8192 | 748.7s | limit |
| 34537 | 0.0% | n/a | n/a | n/a | 0 | n/a | n/a | 900.1s | n/a |
| 35352 | 0.0% | n/a | n/a | n/a | 0 | n/a | n/a | 900.1s | n/a |
| 36065 | 0.0% | n/a | n/a | n/a | 0 | n/a | n/a | 900.1s | n/a |

## Retrieval audit (k=2, leave-one-out)

| id | neighbors (cosine) | self_leak | prompt_chars | few_shot_chars |
|----|--------------------|-----------|--------------|----------------|
| 27560 | 34537(0.401), 31149(0.252) | no | 40854 | 7116 |
| 27702 | 29807(0.306), 31149(0.220) | no | 43598 | 8488 |
| 29807 | 31149(0.491), 33632(0.432) | no | 68926 | 21152 |
| 31149 | 33632(0.511), 29807(0.491) | no | 71360 | 22369 |
| 31430 | 31149(0.465), 36065(0.329) | no | 53406 | 13392 |
| 33632 | 31149(0.511), 29807(0.432) | no | 43598 | 8488 |
| 34537 | 27560(0.401), 36065(0.320) | no | 54392 | 13885 |
| 35352 | 31149(0.184), 27560(0.151) | no | 42150 | 7764 |
| 36065 | 33632(0.344), 31430(0.329) | no | 70280 | 21829 |

## Mechanism preservation (vs PR #400 v6-leaner baseline)

| id | mechanism | v6-leaner verifier | v6+RAG verifier |
|----|-----------|--------------------|-----------------|
| 33632 | dup-decl guard | 92.2% | n/a |
| 36065 | kind-latch | 89.2% | n/a |
| 27702 | punctuation-ident | 100.0% | n/a |

## Side-by-side vs v6-leaner (PR #400)

| id | v6-leaner | v6+RAG | Δ |
|----|-----------|--------|---|
| 27560 | 66.7% | 100.0% | +33.3% |
| 27702 | 100.0% | n/a | n/a |
| 29807 | 81.2% | 93.3% | +12.1% |
| 31149 | 86.7% | n/a | n/a |
| 31430 | 90.5% | n/a | n/a |
| 33632 | 92.2% | n/a | n/a |
| 34537 | 100.0% | n/a | n/a |
| 35352 | 99.1% | n/a | n/a |
| 36065 | 89.2% | n/a | n/a |

## Verdict

**REGRESSION — penalized_mean=0.215 < 0.87**

- ≥ 0.91 mean ⇒ PROGRESS (RAG few-shot earns its keep)
- 0.87–0.91 ⇒ BOUNDED (mechanisms preserved, no improvement)
- < 0.87 ⇒ REGRESSION (RAG actively hurts; revert)
