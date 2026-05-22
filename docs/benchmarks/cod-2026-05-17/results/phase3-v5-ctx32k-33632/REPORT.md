# Phase 3 v5 ctx-32K single-cell validation — 33632 closes (mechanism = context cap)

**Date:** 2026-05-21
**Branch:** `experiment/phase3-v5-ctx32k-33632` off `main` (`2a4a1124`)
**Predecessor:** PR #397 (`phase3-v5-dedup-n9`, commit `438fb764`)
**Model:** `qwen3:30b-a3b-instruct-2507-q4_K_M` (4 B active / 30 B total MoE, q4_K_M GGUF)
**Backend:** `llama-server` (`http://localhost:11437`) + GBNF grammar-constrained decoding
**Schema:** v2.1 no-offset (`docs/grammars/cod-dsl-v2.1.gbnf` + `prompts/cod_dsl_v2_no_offset.txt` — v5: DECLARE-ONCE + KIND PRIORITY + PUNCTUATION-FREE)
**Runner:** `scripts/run_cod_dsl.py --schema v2_1 --backend llama-server --grammar docs/grammars/cod-dsl-v2.1.gbnf --variant v2_1_spec --article-id 33632 --n-predict 6144 --num-ctx 32768 --timeout 1500 --force --results-root docs/benchmarks/cod-2026-05-17/results/phase3-v5-ctx32k-33632 qwen3:30b-a3b-instruct-2507-q4_K_M`
**Scope:** single cell — 33632 only. n=1 by design.

---

## 1. Verdict (TL;DR)

**Mechanism CLOSED. 33632 was bound entirely by the 16K context cap.** Same v5 prompt, same grammar, same model, same article — re-running with `llama_server_ctx_size: 32768` (up from 16384) produces:

- `grammar_valid = True` (was `False` under 16K, parse error `Unexpected token $END` mid-block)
- `stop_type = eos` (was `limit` under 16K — model terminated naturally)
- `prompt_eval_count = 12549` (identical to PR #397 — same prompt → same tokens)
- `eval_count = 4490` (was capped at 3835 because 12549 + 3835 = 16384 exact)
- Total tokens consumed = **17 039** (well under 32 768; the cell needs only ~17K to close)
- `verifier_pass_rate = 0.65625` (defined now; was `None` because `gv=False`)
- 54 ENT / 18 EVT / 0 claims / 0 notes / **0 undeclared** / **0 duplicate declarations**

PR #397's diagnosis stands empirically: the v5 DECLARE-ONCE / KIND PRIORITY / PUNCTUATION-FREE interventions all landed at the prompt level on 33632 (no dup-decl, no undecl, no kind-latch); the cell could not emit `END` only because llama-server hit its `--ctx-size 16384` cap at exactly the 16384 boundary. Bumping the cap to 32K reveals that the v5 prompt CAN close 33632 — it just needs ~1K more tokens of headroom than v4's 16K budget allowed.

The 0.656 verifier (not 0.95+) is a separate observation about the v5 prompt's quality on this cell; it is NOT a halt-criterion update. This experiment's question was the binary "does the cell parse?" — answer: yes, when ctx is sufficient.

---

## 2. Delta vs PR #397 (`phase3-v5-dedup-n9`)

| 33632 metric            | PR #397 (ctx 16K)                                          | this (ctx 32K)                                  | delta                          |
|-------------------------|------------------------------------------------------------|-------------------------------------------------|--------------------------------|
| `llama_server_ctx_size` | 16384                                                      | 32768                                           | +16384                         |
| `grammar_valid`         | **False** (`Unexpected token $END`, ctx-cap mid-block)     | **True**                                        | **False → True**               |
| `stop_type`             | `limit`                                                    | `eos`                                           | cap-hit → natural-end          |
| `prompt_eval_count`     | 12549                                                      | 12549                                           | 0 (same prompt)                |
| `eval_count`            | 3835 (= 16384 − 12549, cap-exact)                          | 4490                                            | **+655** tokens                |
| total tokens            | 16384 (= cap)                                              | 17039                                           | +655 (still under 32K cap)     |
| `ent_count`             | 0 (parse abort)                                            | 54                                              | +54                            |
| `evt_count`             | 0 (parse abort)                                            | 18                                              | +18                            |
| `undeclared_ent_count`  | 0 (parse abort)                                            | 0                                               | 0                              |
| `verifier_pass_rate`    | None (gv=False → scoring abort)                            | **0.65625**                                     | defined                        |
| `verbatim_match_rate`   | None                                                       | 0.65625                                         | defined                        |
| `numeric_exact_match`   | None                                                       | 0.091 (1/11)                                    | defined                        |
| `wall_s`                | 649.0                                                      | 744.0                                           | +95 s (for +655 eval tokens)   |

**Token-arithmetic confirmation:** 12 549 + 3 835 = 16 384 exactly under 16K, and 12 549 + 4 490 = 17 039 under 32K. The cap was the binder, not the prompt mechanism. (See `ollama_meta.stop_type` flip from `limit` → `eos` for the directly observable signal.)

---

## 3. What this does and does not establish

**Establishes:**
1. PR #397's 33632 `gv=False` was caused by `llama_server_ctx_size=16384`, not by a residual DECLARE-ONCE / KIND-PRIORITY / PUNCTUATION-FREE prompt regression. The mechanism diagnosis in PR #397 §4.1 is empirically confirmed.
2. The v5 prompt's output budget for 33632 is ~17K tokens (12.5K prompt + 4.5K emission). 16K is just barely insufficient; 32K provides ample headroom (15.7K spare).
3. The v5 prompt continues to enforce all three v5 invariants on the longest-prompt cell in the n=9 corpus: 54 ENT with no duplicate IDs (DECLARE-ONCE), kind distribution dominated by `country / org` and not `concept` (KIND PRIORITY), and zero undeclared refs (PUNCTUATION-FREE).

**Does NOT establish:**
1. **Aggregate verifier improvement.** This is n=1 by design — a single-cell mechanism probe, not a re-run of the n=9 acceptance gate. PR #397's HALT verdict (verifier 0.782 mean vs 0.84 CONTINUE threshold) is unchanged by this result.
2. **A standing recommendation to deploy at 32K.** 16K is the v5 baseline for downstream experiments (Experiments B/C want 16K). The ctx-32K config is restored to 16K immediately after this probe (see §5).
3. **That 32K closes other cells.** Only 33632 is touched here. Cells 27702 (empty article) and 35352 (TSA passenger table) had their own non-ctx failure modes in PR #397 (hallucinated trigger, sampling variance respectively) that 32K would not affect.

---

## 4. Why 0.65625 verifier and not higher?

Free-form observation, not a halt criterion: the v5 prompt does close 33632 grammatically (gv=True, 0 undecl, 0 dup-decl) but the verifier_pass_rate is 0.656 — lower than the v4/v5 corpus average. Inspecting the output:

- `verbatim_match_rate = 0.65625` (= verifier_pass_rate by construction at this scale)
- `numeric_exact_match_rate = 0.091` (1 of 11 numerics exact; 0.182 semantic — model paraphrases percentages and currencies)
- `ticker_recall = None` (no tickers in this article)
- `org_recall = 0.0` (article cites Reuters + IEA + many country governments; model emits the countries but loses the bridging orgs in a few EVT subjects)
- 18 EVT with 18 missing required slots (~1 missing slot per EVT on average — model elides some optional-but-required-by-grammar fields)

This is a content-quality discussion, not a structural failure. The 16K ctx-cap masked these quality metrics in PR #397 (they were `None`); 32K surfaces them for the first time on this cell. They are well within the range of the rest of the n=9 corpus and do not change the PR #397 HALT verdict on v5.

---

## 5. Context restoration — back to 16K

The ctx-32K change is a single-cell probe, not a baseline shift. Per the experiment scope (downstream Experiments B and C want the v5 baseline at 16K), the ansible variable was restored immediately after this cell completed:

```yaml
# deployment/ansible/group_vars/all.yml
llama_server_ctx_size: 16384   # restored from 32768 after the 33632 probe
```

Restoration sequence:
1. `git checkout` ansible var back to 16384 (separate commit).
2. `ansible-playbook deployment/ansible/playbooks/deploy.yml --tags llama-server` (re-templates `/opt/ai-inference/compose.yaml`, restarts `llama-server` container).
3. `GET /v1/models` confirms `n_ctx = 16384`.
4. `GET /health` confirms `status: ok`.

(See the final commit in this branch for the restore + verification record.)

---

## 6. Files changed

- `deployment/ansible/group_vars/all.yml` — `llama_server_ctx_size: 16384 → 32768`, then `→ 16384` (two commits — bump + restore).
- `docs/benchmarks/cod-2026-05-17/results/phase3-v5-ctx32k-33632/qwen3_30b-a3b-instruct-2507-q4_K_M/v2_1_spec/33632.json` — single-cell result at ctx=32K.
- `docs/benchmarks/cod-2026-05-17/results/phase3-v5-ctx32k-33632/REPORT.md` — this file.

No code changes; no grammar changes; no prompt changes; no STATE.md touch; no plan touch.
