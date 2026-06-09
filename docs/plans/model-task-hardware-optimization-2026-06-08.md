# Model × Task × Hardware Optimization — Sentinel Extraction Throughput

Status: investigation plan (uncommitted). Date: 2026-06-08. Author: agent.
Pairs with `docs/plans/cpu-tuning-throughput-2026-06-08.md` (single-model CPU
tuning; this plan is the orthogonal **model-and-device-assignment** axis).

GOAL: find the optimal assignment of LLM models to the two Sentinel extraction
tasks across the two compute devices, maximizing **articles/hr** at
**recall ≥ the current CoD champion's floor**, with **both devices in use**.

The two tasks (very different shapes):
- **CoD** — DSL extraction. Decode-heavy (~855 completion tok/art), THE
  bottleneck (~2.2 art/hr, 120–375 s/art p50). One call per article.
- **CoVe** — claim verification. Very light: ~64 output tokens, only fires on
  articles carrying ≥1 CLAIM (~24/47 articles in the n=72 sample → ~half),
  ~0.5 % GPU compute duty. One short call per CLAIM.

The two devices:
- **CPU** `llama-server` (b9544) on a Threadripper 9960X (24c, AVX-512,
  4-of-8 RAM channels) — runs the CoD champion today.
- **GPU** vLLM on an RTX 5090 (32 GB). **Compute** ~0.5 % busy, but **VRAM is
  NOT free**: the resident Qwen2.5-32B-AWQ verifier holds **29.8 / 32 GB**
  (`--gpu-memory-utilization 0.92`, kv fp8). Any GPU CoD model must either
  share VRAM with the verifier (≈2 GB headroom — impossible for a 32B) OR the
  verifier must shrink/move so a CoD model can co-reside.

> CONSTRAINT THAT REFRAMES EVERYTHING: "GPU 99.5 % idle" = **compute** idle, not
> **VRAM** idle. A 32 GB card running a 30 GB resident verifier has ~2 GB free —
> it cannot also host a 32B/30B CoD model at AWQ INT4 (~18–20 GB) + 32K KV
> without evicting or shrinking the verifier. The device-assignment problem is a
> **VRAM-allocation** problem, not a compute-scheduling one.

---

## 1. Prior CoD benchmarks — what was tested, scores, top performers

Two metric eras. Don't conflate them.

### Era 1 — "preservation rate" (chunked-summary CoD, pre-DSL), `cod-2026-05-17/REPORT.md`, n=10
Token-preservation vs Opus-4-7 ground truth. Single-pass for ≥24B, chunked for ≤14B.

| Model | Size | Tickers | Companies | Sectors | Numerics | p50 lat |
|---|---|---|---|---|---|---|
| **qwen3:30b-a3b-instruct-2507-q4_K_M** | 30B-MoE | **25 %** | **40 %** | 100 % | **19 %** | **49 s** |
| sentinel-cod-v6 (LoRA, n=2) | 7B-LoRA | 0 % | 78 % | 100 % | 82 % | 353 s |
| granite4.1:8b | 8B | 0 % | 38 % | 100 % | 18 % | 433 s |
| qwen3:8b | 8B | 0 % | 24 % | 100 % | 15 % | 1373 s |
| gemma4:26b-a4b-it | 26B-MoE | 0 % | 0 % | — | 0 % | 380 s |
| phi4-mini:3.8b | 3.8B | 0 % | 33 % | 100 % | 11 % | 487 s |
| qwen2.5:7b-instruct (then-prod) | 7B | 0 % | 24 % | 0 % | 13 % | 343 s |

→ **qwen3:30b-a3b is the only model that recovers tickers (25 %) AND is ~7×
faster** (single-pass, no chunking). It won Round 1 decisively and became the
CoD base. (sentinel-cod-v6's 78 %/82 % is n=2 and a legacy adapter — not weighted.)

### Era 2 — DSL CoD with verifier score (the canonical metric now)
- **`BASELINE.md` (locked 2026-05-22):** `cod_dsl_v8_layered` on
  qwen3:30b-a3b → **aggregate verifier 0.907, grammar-valid 9/9** (n=9 Arm-B).
  v6_leaner = 0.895. v9/v10 trims both **regressed** (DECLARE-ONCE collapse) —
  the v8 prompt floor is locked, no further trim pursued.
- **`dsl-round2/3` (n=10×7 models, per-category recall):** qwen3:30b-a3b leads
  class B on num_sem (~89–91 %), org (~31–36 %), grammar-valid (90–100 %).
  The ≤8B class-A models are grammar-fragile (gv 20–90 %, wild variance) and
  never recover tickers reliably. gemma4:26b-a4b collapses on the DSL grammar
  (gv 0–30 %). **No open model dislodged qwen3:30b-a3b for CoD.**
- **`cod-truncation-2026-06-07` (current harness, this plan's quality metric):**
  qwen3:30b-a3b at **full context** → **recall mean 0.610 / median 0.750**,
  precision ~0.20 (n=29). This `depth=full` cell is the **CoD-quality floor any
  candidate must beat** — measured by `scorer.py`'s GPU-judge recall.

### CoVe verifier models — `phase4-7-model-bench/REPORT.md` (n=137 CLAIMs)
| Verifier | Params | Quant | Agreement vs Foundry-Claude | Δ vs Qwen | wall |
|---|---|---|---|---|---|
| **Qwen2.5-32B-Instruct-AWQ** (prod) | 32B | AWQ INT4 | **71.53 %** | baseline | 97 s |
| Granite-4.1-30B-AWQ | 30B | AWQ INT4 | 56.20 % | **−15.3 pp** | 20 s |
| Mistral-Small-3.2-24B-AWQ | 24B | AWQ INT4 | 18.98 %* | −52.6 pp | 6 s |

*Mistral-3.2 emits empty body on `/v1/completions` (chat-only SFT) — a
prompt-path artifact, not pure quality. **Both ~30B swaps regressed**; the
~14 pp Qwen-vs-Claude gap is **structural to the prompt/CLAIM shape**, not the
model — so CoVe gains come from prompt/verdict-vocab, not a different ~30B open
model. **Qwen2.5-32B-AWQ is the incumbent CoVe champion.**

**TOP PERFORMERS carried forward:** CoD = `qwen3-30b-a3b-instruct-2507-q4_K_M`
(CPU). CoVe = `Qwen2.5-32B-Instruct-AWQ` (GPU). Both are the things to beat.

---

## 2. Candidate shortlist

Selection rule: peers/successors of the two champions that are (a) ≥14–32B-class,
(b) strong at structured extraction (CoD) and/or judgment (CoVe), (c) quant-
feasible on the target device, (d) open-licensed, (e) already on disk OR a cheap
pull. MoE prioritized for CPU (low active params); AWQ/GPTQ for the 32 GB GPU.

| # | Model | Total / active | On disk? | Quant | Device fit | License | Role candidacy |
|---|---|---|---|---|---|---|---|
| C0 | **qwen3-30b-a3b-instruct-2507** | 30B / 3B MoE | ✅ ollama+GGUF | q4_K_M (18 GB) | CPU champ; GPU-AWQ if verifier moves | Apache-2.0 | **CoD champion (control)** |
| C1 | **GLM-4.7-Flash** | 30B / 3B MoE | ✅ `/models/GLM-4.7-Flash-Q4_K_M.gguf` (18.3 GB) | q4_K_M | CPU (direct a3b peer) | MIT | **CoD** — top new CPU candidate |
| C2 | **granite-4.x (MoE, JSON-native)** | ~30B-A3B class | granite4.1:8b ✅; 30B-MoE = pull | q4 GGUF / AWQ | CPU or GPU | Apache-2.0 | CoD (JSON-native) + CoVe (lost CoVe by 15 pp @30B-dense, MoE untested) |
| C3 | **EXAONE-4.0-32B** | 32B dense | ✅ `/llama-server/EXAONE-4.0-32B-Q4_K_M.gguf` (19.3 GB) | q4_K_M | CPU slow (dense 32B); GPU-AWQ | exaone (research) | CoD/CoVe — dense, license-gated, lower priority |
| C4 | **gemma-4-31b-it** (a4b MoE) | 31B / ~4B MoE | ✅ `/llama-server/gemma-4-31b-it` (18.3 GB) | q4_K_M | CPU | Gemma | CoD — but gemma4:26b-a4b **collapsed on DSL grammar** (gv 0–30 %) in R2/R3; low prior |
| C5 | **qwen2.5-32B-Instruct** | 32B dense | ✅ ollama + GGUF | q4 / AWQ | GPU-AWQ (CoVe incumbent) / CPU slow | Apache-2.0 | **CoVe champion (control)** |
| C6 | **qwen3-32B (dense)** | 32B dense | pull | AWQ/GPTQ (~18 GB) | GPU-AWQ | Apache-2.0 | CoVe — newer dense Qwen, may close the 14 pp gap |
| C7 | **qwen3-next:80b** | 80B / ~3B MoE | ✅ ollama (50 GB) | q4 | CPU (fits 48 GiB? 50 GB → tight/over) | Apache-2.0 | CoD stretch — higher quality, only ~3B active; **memory-budget risk** |
| (D) | Qwen3-1.7B-Q4_K_M (draft) | 1.7B | ✅ `/drafts/` | q4 | CPU draft for C0 | Apache-2.0 | **already in the in-flight speculative bench** |

Notes:
- **C1 GLM-4.7-Flash is the headline new CoD candidate**: identical
  architecture class to the champion (30B-A3B MoE, ~3B active → same ~8–10 tok/s
  decode envelope on CPU), MIT-licensed, **already on disk** → zero-download
  A/B. If it matches/beats qwen3-30b-a3b recall it's a drop-in; if it's faster
  at equal recall it raises throughput directly.
- **C2 Granite-4 MoE**: IBM's JSON-native line; the 8B is grammar-fragile in
  R2/R3 and the 30B-dense lost CoVe by 15 pp — but a **30B-A3B MoE granite** is
  untested for CoD and is the natural "JSON-native MoE on CPU" probe.
- **GPU CoD is bounded by VRAM, not appetite**: any 30–32B AWQ CoD model needs
  ~18–20 GB + 32K KV ≈ 24–28 GB — it **cannot co-reside** with the 29.8 GB
  verifier. GPU-CoD is only viable if CoVe moves to a **smaller** verifier
  (freeing VRAM) or to the CPU. This is the central trade in §5.
- Dropped: command-r:35b, llama3.3:70b (sub-10 TPS / OOM in R1 cull),
  mistral-small (CoVe 19 %/`/completions` artifact), sentinel-* legacy adapters.

---

## 3. CoVe quality metric

CoVe quality is **not** the truncation recall metric — it's verifier-agreement.
Two existing, production-faithful harnesses already define it; reuse them:

**Primary — agreement vs the gold verifier (`phase4-7-model-bench` method).**
- Fixed claim set: the **n=137 CLAIMs** (47 articles, seed=4757) with **frozen
  Foundry Claude-4.7 labels** already on disk (`foundry_labels/*.json`).
- Score = **% agreement with the Foundry-Claude labels** on the 3-valued
  collapse {full, partial, none}, using the **byte-identical production prompt +
  parser** (`ClaimVerifier.BuildPrompt`/`TryParseVerdict`, iter-5).
- DECISION RULE (from that report): a candidate "matters" only if |Δ| ≥ 5.0 pp
  vs the Qwen2.5-32B baseline (71.53 %). Below that = noise, keep the incumbent.
- This is the **right CoVe metric** because the gold is a frontier model and the
  prompt path is production-exact; it directly measures verdict quality.

**Secondary / regression guard — `cod-cove-acceptance/cod_cove_harness.py`.**
- Feeds **verbatim-validated** CoD CLAIMs (provably present in source) to the
  candidate verifier. A coherent verbatim-true fact MUST verify `consistent`
  (~100 %); a miss = **bug**, not judgment. Gives a clean floor: any CoVe
  candidate must keep ~100 % on the coherent subset, or it's broken/mis-wired.

> CoVe scoring = **agreement vs frozen Foundry labels on the n=137 set
> (primary, pp gate = ±5)** + **coherent-verbatim pass-rate ~100 % (regression
> guard)**. CoD quality = **`scorer.py` GPU-judge recall on `depth=full`**,
> PASS = recall ≥ 0.61 (champion floor, within judge noise).

---

## 4. Eval matrix {model} × {task} × {feasible device}

metrics per cell: **Q** = quality (CoD: scorer recall @full vs 0.61 floor; CoVe:
agreement-pp vs 71.53 floor) · **S** = decode tok/s · **A** = projected art/hr.
Legend: ✅feasible · ⚠conditional (VRAM/grammar/license) · ✗infeasible ·
**[IF]** = covered by in-flight work (don't duplicate) · **★** = recommended.

### CoD task (decode-heavy — the bottleneck)
| Model | CPU (llama-server) | GPU (vLLM, needs VRAM) |
|---|---|---|
| C0 qwen3-30b-a3b (champ) | ✅ control: Q=0.61, S~8–10, A~2.2 **[IF: A/B gold + speculative]** | ⚠ AWQ ~24–28 GB w/KV → **only if verifier moves** **[IF: A/B GPU arm]** |
| C1 GLM-4.7-Flash ★ | ✅ **★ run first** — peer MoE, on disk, gv+recall vs champ | ⚠ same VRAM bind as C0 |
| C2 granite-4-MoE | ⚠ pull; JSON-native; grammar-risk (8B was fragile) | ⚠ VRAM bind |
| C3 EXAONE-4.0-32B | ⚠ dense 32B on CPU → slow (~½ champ tok/s) | ⚠ VRAM bind + license |
| C4 gemma-4-31b-a4b | ⚠ DSL-grammar collapse risk (R2/R3) | ⚠ VRAM bind |
| C7 qwen3-next:80b | ⚠ 50 GB vs 48 GiB budget → **memory-risk**, ~3B active | ✗ 50 GB ≫ 32 GB |
| (D) +Qwen3-1.7B draft | ✅ speculative on C0/C1 **[IF: speculative bench running]** | — |

### CoVe task (light — ~0.5 % duty, runs on whatever has spare cycles)
| Model | GPU (vLLM) | CPU (llama-server) |
|---|---|---|
| C5 Qwen2.5-32B-AWQ (champ) | ✅ control Q=71.53 pp, resident now | ✗ dense 32B CPU = too slow to share w/ CoD |
| C6 qwen3-32B-AWQ ★ | ✅ **★ probe** — may close 14 pp gap; same VRAM class | ✗ |
| C2 granite-4-MoE-AWQ | ✅ but −15 pp @30B-dense prior; MoE retest only if VRAM matters | ⚠ MoE could co-run on CPU if it frees the GPU |
| smaller verifier (14B AWQ) | ✅ **VRAM-freeing probe** — Qwen2.5-14B judged "lightweight viable" (web); frees ~12 GB for GPU-CoD | ✗ |

**Coverage of in-flight work:** the **CoD A/B (Qwen2.5-32B-AWQ-GPU vs
qwen3-30b-a3b-gold)** covers the C0-GPU and the gold-recall control cells. The
**CPU speculative bench** (`llama-bench-spec`, Qwen3-1.7B draft, n-max 8) covers
the (D) row on C0. **Do not re-run those.** This plan adds: C1 GLM-4.7-Flash on
CPU (CoD), C6 qwen3-32B on GPU (CoVe), and the smaller-verifier VRAM-freeing
probe — the cells the in-flight work does **not** touch.

---

## 5. Prioritized experiment sequence (max info / effort)

Prod-pause authorized. The two in-flight runs already occupy the "champion CoD
A/B" and "speculative" lanes; this sequence fills the **model-substitution** and
**device-reallocation** gaps without colliding with them.

### The central decision the matrix forces
Throughput is gated by **CoD decode**, and the GPU's spare resource is
**compute, gated behind VRAM the verifier owns**. So the highest-leverage
question is binary:

> **Can we put a CoD model on the GPU?** Only if CoVe is shrunk/moved to free
> VRAM. CoVe is tiny (~0.5 % duty) and tolerant of a smaller verifier *iff*
> agreement holds. **So the unlock is: shrink the verifier → free ~12 GB → host
> a second-pool CoD model on GPU → GPU CoD decode is 5–8× the CPU.** That single
> reallocation can multiply throughput far more than any CPU knob.

### Sequence

**Exp 1 — CoD model swap on CPU (C1 GLM-4.7-Flash), zero-cost, FIRST.**
- Why first: on disk, MIT, exact peer of the champion, no GPU contention, runs
  on a disjoint CCD set while prod drains (same A/B method as the cpu-tuning plan
  §6). Pure win-or-neutral: if GLM matches recall at higher decode tok/s →
  drop-in throughput gain; if lower recall → discard, cost ≈ 1 hr.
- Method: ephemeral `llama-bench` on CCDs 2–3, same v8 prompt + v2.3 GBNF +
  temp0 + n_predict, fixed 12-article econ set → decode tok/s; then full
  truncation `depth=full` set → `scorer.py` recall. PASS = recall ≥ 0.61.

**Exp 2 — VRAM-freeing verifier probe (the throughput unlock), HIGHEST CEILING.**
- Score a **smaller AWQ verifier** (Qwen2.5-14B-AWQ ~9–10 GB, and C6 qwen3-32B
  for the quality-up direction) on the **n=137 frozen-Foundry CoVe set** +
  coherent-verbatim guard. PASS = agreement within −5 pp of 71.53 AND ~100 % on
  coherent-verbatim.
- If a 14B holds agreement: the verifier drops from 29.8 GB → ~12 GB, **freeing
  ~17 GB** — enough to **co-host a 30B-A3B AWQ CoD model + 32K KV on the GPU**.
  This is the gate to Exp 3. Run read-only against a *second* ephemeral vLLM on
  the freed budget; never evict the prod verifier until proven.

**Exp 3 — GPU CoD second pool (conditional on Exp 2 PASS), THE THROUGHPUT WIN.**
- Stand up qwen3-30b-a3b-AWQ (or GLM-4.7-Flash-AWQ) on the GPU alongside the
  shrunk verifier. Route articles: GPU pool takes the bulk; CPU pool keeps
  draining. GPU decode 5–8× CPU → **projected CoD throughput multiplies**.
- Quality gate: `scorer.py` recall @full on the GPU model ≥ 0.61 (the in-flight
  C0-GPU A/B already measures qwen3-30b-a3b-AWQ recall — **reuse its result**,
  don't re-run). Both devices now in use: GPU = primary CoD + CoVe, CPU = CoD
  overflow / long-article 32K pool.

**Exp 4 — synthesis & assignment lock.**
- Combine: best CoD model×device (from Exp 1 + in-flight A/B + speculative) and
  best CoVe verifier (Exp 2). Project pipeline art/hr = (GPU CoD art/hr +
  CPU CoD art/hr), with CoVe absorbed in GPU spare duty.
- Decision table (fill from results):

| Assignment | CoD device/model | CoVe device/model | recall | proj art/hr | both-in-use |
|---|---|---|---|---|---|
| **Baseline (today)** | CPU qwen3-30b-a3b | GPU Qwen2.5-32B-AWQ | 0.61 | ~2.2 | ✓ |
| A: +CPU speculative | CPU a3b + 1.7B draft | GPU Qwen2.5-32B | 0.61 | ~3–6 (1.5–3×) | ✓ |
| B: CPU model swap | CPU GLM-4.7-Flash | GPU Qwen2.5-32B | ? | ? | ✓ |
| **C: GPU CoD pool ★** | **GPU a3b-AWQ + CPU overflow** | **GPU 14B-AWQ (shrunk)** | ? | **≫ (5–8× on GPU pool)** | ✓ |

> Expected optimum = **C** (GPU does the heavy CoD decode at 5–8× CPU; a shrunk
> verifier frees the VRAM; CPU keeps a 32K long-article + overflow pool → both
> devices saturated). C is gated entirely on **Exp 2** (can a smaller verifier
> hold agreement?). If Exp 2 fails, fall back to **A/B** (CPU speculative +
> best CPU model), GPU stays CoVe-only — still both-in-use, smaller win.

### Safety / A-B hygiene (read-only on prod)
- Prod llama-server keeps draining; all benches run on **disjoint CCDs** (CPU)
  or a **second ephemeral vLLM on freed VRAM** (GPU) — never evict the prod
  verifier until a candidate passes the pp gate. Telemetry/config = SELECT-only.
- Every output-changing cell gated by `scorer.py` (CoD) or the n=137 agreement
  harness (CoVe). Speculative = assert byte-identical (zero recall risk).

---

## 6. Open items
- Confirm a 30B-A3B **granite-4 MoE** AWQ exists on HF before committing C2 (the
  8B is fragile; MoE is the interesting probe).
- qwen3-next:80b (50 GB) vs the 48 GiB container budget — measure resident set
  before trusting it as a CPU CoD option (likely over-budget).
- Exp 3 routing lives in `CpuDslExtractionService` / a token-length+device
  classifier — same wiring as the cpu-tuning plan's §5-H two-pool design; build
  once, share.
