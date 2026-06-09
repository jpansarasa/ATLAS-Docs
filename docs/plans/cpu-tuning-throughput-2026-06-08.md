# CPU / Model Tuning — Sentinel CoD Extraction Throughput

Status: investigation plan (uncommitted). Author: agent. Date: 2026-06-08.
Goal: raise per-article CoD extraction **throughput** (articles/hr) on the CPU
`llama-server` running `qwen3-30b-a3b` (MoE, ~3B active params), WITHOUT
regressing extraction recall.

Hard constraints (MUST hold):
- model ≥30B params (`qwen3-30b-a3b-instruct-2507-q4_K_M`)
- 32K context per slot (SENTINEL rule; Phase-1 Exp-A proved <16K truncates)
- recall (extraction quality) must not regress — every config A/B gated by the
  truncation harness scorer + GPU judge.

KEY INSIGHT (drives all prioritization): per-article cost is
**DECODE-dominated**. Measured econ micro-benchmark (`/tmp/sentinel-remediation/
cod-econ-measure/`, 2026-06-07): decode = 67–100 % of compute; the 9.3K static
prompt prefix caches (warm prefill ≈ 0.1 s). So weight **decode-throughput**
levers highest; prefill/prompt levers are near-worthless here.

---

## 1. Current live config

### llama-server command (exact, from `/proc/1/cmdline` on container)
```
/app/llama-server
  --model /models/qwen3-30b-a3b-instruct-2507-q4_K_M.gguf
  --host 0.0.0.0 --port 8080
  --ctx-size 32768
  --threads 24
  --n-gpu-layers 0
  --parallel 1
  --flash-attn on
  --ubatch-size 2048
  --cache-type-k q8_0
```
- Image: `ghcr.io/ggml-org/llama.cpp:server`, build **b9544 (98d5e8ba8)**, GCC 14.2.
- cpuset (compose top-level): `0-23`. deploy limits: 24 CPU / 48 GiB.
- NOT set (defaults in effect): `--threads-batch` (= --threads = 24),
  `--batch-size` (2048), `--ubatch` set to 2048 (default would be 512),
  `--cache-type-v` (f16 — NOTE: only K is quantized, V stays f16),
  `--numa` (off), `--mlock` (off), `--no-mmap` (off → mmap ON),
  `--poll` (50), `--prio` (0 / normal), `--cpu-strict` (0), no speculative draft.
- Source of truth: `deployment/ansible/group_vars/all.yml` (`llama_server_*`
  vars) → `deployment/artifacts/compose.yaml.j2`. `--threads 24` and
  `--n-gpu-layers 0` are hard-coded in the template, not vars.

### Sampling params (live `/slots` + `LlamaServerClient.cs`)
- Client sends only: `temperature=0.0`, `n_predict=16384`, `grammar=<GBNF v2.3>`,
  `stop=[<|im_end|>,<|endoftext|>]`, `seed=null`. Everything else is
  llama-server **default**: `top_k=40, top_p=0.95, min_p=0.05, repeat_penalty=1.0`.
  At temp=0 the sampler chain is effectively greedy, so top_k/top_p are inert
  (no throughput lever there) — but the full sampler chain
  (`penalties→dry→top_n_sigma→top_k→typ_p→top_p→min_p→xtc→temperature`) still
  runs per token. Trimming the chain (`--samplers "temperature"` /
  `--top-k 1`) is a tiny micro-lever (see §5).

### Model / quant in use
- `qwen3-30b-a3b-instruct-2507-q4_K_M.gguf`, **18.56 GB** on disk, mounted RO
  from the ollama content-addressed blob store (sha256-78b329e7…). MoE, ~3B
  active params/token. Resident: VmRSS ≈ 42 GB (mmap pages, NOT swapped:
  VmSwap=0), container MEM 23.6 GiB / 48 GiB.

### Static prefix (the cached prompt prefix)
- Prompt file `cod_dsl_v8_baseline.txt`: 867 lines / 38,307 chars ≈ **9.3K
  tokens**. Confirmed live: `/slots` shows `n_prompt_tokens_cache=9295` on an
  in-flight call. Grammar `cod-dsl-v2.3.gbnf`: 9,277 bytes.

---

## 2. Hardware (TRX50 Threadripper — memory-bandwidth bound)

### CPU topology (`lscpu`)
- **AMD Ryzen Threadripper 9960X**, 24 physical cores / 48 threads (SMT2),
  single socket, base/boost up to 5.49 GHz.
- ISA: AVX-512 full suite incl. `avx512_bf16`, `avx512_vnni`, `avx512vbmi2`,
  `vaes` — llama.cpp will use AVX-512 + bf16 kernels.
- Caches: L1d 1.1 MiB (24×48 KiB), L2 24 MiB (24×1 MiB), **L3 128 MiB across 4
  instances** → **4 CCDs of 6 cores each**.

### CCD / L3 map (load-bearing for affinity)
`/sys/.../cache/index3/shared_cpu_list`:
| CCD | physical cores | logical CPUs (incl. SMT siblings) |
|-----|----------------|-----------------------------------|
| 0   | 0–5            | 0-5, 24-29 |
| 1   | 6–11           | 6-11, 30-35 |
| 2   | 12–17          | 12-17, 36-41 |
| 3   | 18–23          | 18-23, 42-47 |

→ cpuset `0-23` = **exactly the 24 physical cores, one logical CPU per core,
spread across all 4 CCDs**. SMT siblings (24-47) are NOT in the set, so
`--threads 24` already pins 1 thread per physical core across all CCDs — a sane
default. ollama-cpu-gen (24-39) and ollama-cpu-embed (40-47) sit on the SMT
siblings of CCDs 0–3, so when they run they contend for the *same physical
cores'* execution units even though the cpusets look disjoint by logical-CPU
number. **This is a real, previously-undocumented contention source** (logical-
disjoint ≠ physical-disjoint).

### NUMA (`numactl --hardware`)
- **Single NUMA node** (node0 = all 48 CPUs, 128 GB). `numa_balancing=0`.
- Implication: classic dual-socket NUMA placement is N/A. BUT the 4 CCDs each
  have their own memory controllers / Infinity-Fabric path; this is "NUMA-per-CCD"
  behavior the OS doesn't expose as nodes. `--numa numactl` / NPS settings give
  little here (one node); the lever that matters is **CCD-local thread packing**
  (fewer CCDs, less cross-CCD IF traffic) — see §5.

### RAM (`dmidecode -t memory`)
- **4× 32 GB Kingston DDR5-4800 = 128 GB, 4 channels populated** (CHANNEL
  A/C/E/G only; B/D/F/H empty), 2 rank each, ECC. TRX50 is an **8-channel**
  platform → the box runs at **HALF its memory-bandwidth potential**. Confirmed
  via `dmidecode` (4 modules) + `MemTotal=131230604 kB`. Populating the 4 empty
  channels (4 more DDR5 DIMMs) ≈ doubles DRAM bandwidth — THE single biggest
  hardware throughput factor for a DRAM-bound MoE decode (but see latency caveat
  below).
  - DDR5-4800 per channel ≈ 38.4 GB/s. Current **4-ch ≈ 154 GB/s**; full
    **8-ch ≈ 307 GB/s** (achievable by filling B/D/F/H — a hardware upgrade).
  - At ~8 tok/s decode reading ~3B active params × ~0.5 B/param (q4) ≈ 1.5 GB/tok,
    8 tok/s ⇒ ~12 GB/s effective — far below even 4-ch peak, so we are **latency-
    /overhead-bound at the active-expert gather, not raw-bandwidth-saturated**.
    This reframes the problem: MoE per-token cost is dominated by scattered
    expert-weight gathers + attention over 9–10K-token KV, not by streaming a
    dense weight matrix. Confirms speculative decoding (§5) as the top lever.

---

## 3. Already-tested (and how rigorous)

### Commit `3a802551` (2026-06-07) — current config origin
Changed (all attributed to "benchmark a99d6898"):
- `--parallel 2 → 1` (claim: parallel=2 was **2.8× slower per request**;
  parallel=1 also restores full 32K/slot vs 16K)
- `--flash-attn off → on`
- `--ubatch-size 512 → 2048`
- `--cache-type-k f16 → q8_0`
- Target stated: ~13–15 tg t/s, ~100+ pp t/s.

### Rigor assessment — **WEAK / not reproducible**
- "benchmark a99d6898" is **NOT a git commit** (no object, not in `docs/`).
  It is an ad-hoc run ID with **no committed results doc, no script, no raw
  numbers**. The only durable artifact is the *econ micro-benchmark* (§4), which
  measured the decode/prefill split but did NOT sweep parallel/ubatch/flash/KV.
- So the 4 changes in `3a802551` are **plausible but unverified against the
  current corpus**; the "throughput ~flat regardless of knobs" intuition is
  folklore, not a logged sweep. The stated target (13–15 tg t/s) is **NOT met in
  prod** — live decode is ~8–10 tg t/s (§4). Treat all of `3a802551` as
  re-testable, including re-validating parallel=1 > parallel=2 (it is likely
  right for *latency*, but for *backlog throughput* a 2-slot 16K config trades
  per-request latency for concurrency — only re-bench settles it, AND 16K/slot
  violates the SENTINEL 32K rule so it's out unless the rule is revisited).

### Other relevant commits
- `#638` (049b1787): per-stage OTEL timing — the telemetry this plan re-baselines
  from. Metrics: `sentinel_extraction_stage_duration_seconds{stage}`,
  `sentinel_llm_completion_tokens{stage}`, `sentinel_llm_prompt_tokens{stage}`,
  `sentinel_article_total_duration_seconds`.
- `#639` / e4257764: `MaxConcurrentExtractions 4→1` to match `--parallel 1`
  (eliminated slot-queue failures). Confirmed live: `slot_wait` p50 ≈ floor.
- No prior thread-count / affinity / NUMA / speculative experiments exist in git
  (searched). **This entire lever space is untested.**

---

## 4. Re-baseline (now = 1 live slot) — from PROD telemetry

Measured 2026-06-08 ~11:15Z, Prometheus `bf2ya9fqus268c`, 6 h rate windows
(read-only; no contending extraction launched):

| metric | value | source |
|--------|-------|--------|
| cod_llm stage p50 | **120–375 s/article** | `histogram_quantile(0.5, …stage_duration…{cod_llm})` |
| resolve / persist / slot_wait p50 | ~2.5 s (histogram floor) | same — **non-cod stages are negligible** |
| avg completion tokens / article | **855 tok** | `…completion_tokens_sum/count{cod_llm}` |
| avg prompt tokens / article | **10,116 tok** (≈9.3K static + ~0.8K article) | `…prompt_tokens_sum/count{cod_llm}` |
| end-to-end article tok/s | **2.64 tok/s** | completion_sum / stage_duration_sum |
| **articles/hr** | **~2.2/hr** (6 h avg); ranges 1.1–9.2/hr by article-length mix | `rate(article_total_count)*3600` |

Pure-decode rate (from econ micro-bench `/completion timings`, isolates
`predicted_ms`): **~8–10 tok/s** decode; **~35–50 tok/s** prefill. The gap
between 2.64 (end-to-end) and ~8 (pure decode) is the prefill of the ~0.8K
non-cached article tokens + serde, amortized over only 855 completion tokens.

Backlog math at baseline: at ~2.2 art/hr a 1–2K-article backlog drains in days.
The throughput target is to **multiply art/hr**, which (since decode dominates)
≈ multiply decode tok/s.

> Re-baseline conclusion: the lever that matters is raw **decode tok/s**.
> Everything that doesn't move decode tok/s (prefill knobs, KV-K type already
> done, slot count) is noise here.

---

## 5. FULL untested lever space

Weighted for DECODE impact. "Decode impact" = expected effect on tok/s during
generation (the 67–100 % cost). Risk/quality noted per lever.

### A. Speculative decoding — **HIGHEST-LEVERAGE decode win (untested)**
- `-md <draft.gguf> --spec-draft-n-max N --spec-draft-p-min P` (+ optional
  `--cpu-mask-draft` to dedicate a CCD to the draft). Build b9544 supports it.
- Why it fits: decode is the cost; greedy temp=0 + GBNF-constrained output is
  **highly predictable** (DSL has rigid structure: `ENT/NUM/EVT/CLAIM` blocks,
  fixed delimiters) → high draft acceptance. A small same-vocab draft
  (qwen3-0.6B or 1.7B, **must share the qwen3 tokenizer/vocab**) verified in
  batch by the 30B target turns serial 8 tok/s into 2–4× via accepted runs.
- Draft candidates: **none on disk** — `qwen3:8b` (5.2 GB) is too big to be a
  good draft (its own decode ~ target's). NEED to pull/convert
  **Qwen3-0.6B / Qwen3-1.7B GGUF** (same Qwen3 vocab → compatible). One-time
  ~0.5–1.5 GB download; fits the 48 GiB budget alongside the 30B.
- Decode impact: **HIGH (est. 1.5–3× on structured output)**. Risk: MED
  (CPU draft steals cores from target — mitigate with `--cpu-mask-draft` pinning
  draft to one CCD, target to the other 3; or run draft on the SMT siblings).
  Quality impact: **NONE** — speculative decode is exact (verifier guarantees
  identical tokens to non-speculative greedy). This is the rare gain with **zero
  recall risk** → still gate once to prove output identity, then trust.

### B. Thread count / `--threads-batch` / SMT / affinity
- Levers: `-t` (try 16/20/24/32/48), `-tb` separate batch threads,
  `--cpu-strict 1`, `--cpu-mask`/`--cpu-range`, `--prio 2/3`, `--poll 0/100`.
- MoE decode is memory-latency-bound: more threads past the bandwidth knee can
  HURT (cross-CCD coherence). cpuset 0-23 already = 1 thread/physical core/all
  CCDs (good). Worth testing: (a) **fewer CCDs** (e.g. cpuset `0-11` = 2 CCDs,
  `-t 12`) to cut cross-CCD IF traffic — may raise tok/s on a latency-bound
  workload; (b) **SMT on** (`-t 48`, cpuset `0-47`) — usually hurts compute-bound
  but can help latency-bound by hiding memory stalls; (c) `--prio 2` +
  `--poll 100` to cut scheduler/wake latency; (d) `--cpu-strict 1` to stop the
  OS migrating threads across CCDs mid-decode.
- Decode impact: **MED** (±10–30 %, direction non-obvious → must measure).
  Risk: LOW (runtime flags, instant revert). Quality: NONE.
- NOTE the §2 finding: ollama-cpu-gen/embed sit on SMT siblings of CCDs 0-3 →
  when they run they steal physical-core throughput from llama-server. Test
  llama-server throughput **with the ollama gen/embed containers paused** to
  measure the contention tax (it may be large during concurrent embedding).

### C. NUMA
- `--numa numactl` / `--numa distribute` / `--numa isolate`. Single NUMA node →
  classic NUMA placement is **near-useless** here (no remote node). The only
  real "NUMA" axis is CCD-local packing (covered in B). Decode impact: **LOW**.
  Risk LOW. Quality NONE. (Document as tested-and-flat, don't over-invest.)

### D. KV-cache type — partially done, one lever left
- Already: `--cache-type-k q8_0`. **V is still f16** (`--cache-type-v` unset).
  Setting `-ctv q8_0` halves V-cache bandwidth/footprint → at 10K-token KV +
  flash-attn this can lift decode tok/s a few %. Going `q4` on either K or V is a
  **recall risk** (precision of attention over long context) → gate hard.
- Decode impact: **LOW–MED** (V→q8_0: small but free). Risk LOW (q8) / MED (q4).
  Quality: LOW (q8) — still gate once.

### E. batch / ubatch
- `--ubatch 2048` already set (helps prefill, not decode). For a single
  sequence, ubatch barely touches decode (1 token/step). Decode impact: **~0**.
  Low priority. (Larger ubatch can marginally help the one-shot prefill of the
  ~0.8K non-cached article tokens, but prefill is already cheap.)

### F. `--mlock` / `--no-mmap`
- Model is mmap'd (no-mmap unset), VmSwap=0 so **not currently swapping** →
  `--mlock` mainly insures against future swap-out of model pages under memory
  pressure (system swap is 2.4 GiB used by *other* procs). `--no-mmap` forces a
  full RAM copy (slower load, marginally faster steady-state page access). On a
  latency-bound decode, mlock'ing the active-expert pages could shave tail
  latency. Decode impact: **LOW**. Risk LOW. Quality NONE. Cheap insurance —
  add `--mlock` early.

### G. Model-quant alternatives (within quality limits)
- Current q4_K_M. Options that could raise decode tok/s by shrinking
  bytes-read/token: **q4_0 / q4_K_S / IQ4_XS** (smaller, faster, slightly lower
  quality) vs **q5_K_M / q6_K** (slower, higher quality — wrong direction).
  IQ4_XS often matches q4_K_M quality at lower size → possible free decode win.
  Decode impact: **LOW–MED**. Risk MED (re-quant + must re-prove recall).
  Quality: MED → **gate hard**. Lower priority than A/B (more effort: requantize).
- A bf16 native build benefit: the model is q4 but activations run in the
  AVX-512 bf16 path already; no lever there.

### H. Dynamic / smaller ctx + more `--parallel` for the SHORT-article majority
- Corpus skews short: wire/RSS median ~547 tok, p90 ~1148 (config.py). The 32K
  rule is to protect the rare long article. A **two-pool design**: keep the
  32K/parallel=1 server for long articles; add a SECOND llama-server instance
  with a SMALLER ctx (e.g. 8K) and `--parallel 3–4` for the short majority,
  routed by article token count. This multiplies throughput on the bulk of
  traffic WITHOUT violating 32K for long ones (they go to the 32K pool).
  - Decode impact: **HIGH for the short-article majority** (3–4 concurrent
    short decodes), but: (i) two servers contend for the same 24 cores unless
    CCD-split (12+12), halving each pool's threads; (ii) added routing
    complexity in `CpuDslExtractionService` / a token-length classifier;
    (iii) MoE-on-CPU parallel slots were measured 2.8× slower/req — but that was
    at 32K/16K split; at 8K/parallel=4 the picture differs and is **untested**.
  - Risk: MED-HIGH (new wiring, slot contention). Quality: NONE if long articles
    still get 32K. **Highest-ceiling but highest-effort** — sequence it after the
    cheap single-server levers (A/B/D/F) prove their gains.

### I. Sampler-chain trim (micro)
- Client could send `top_k=1` / `--samplers temperature` so the per-token
  sampler chain (9 stages) collapses to greedy argmax. At 855 tok/article the
  saving is microseconds/token — **negligible** vs decode. Decode impact: ~0.
  List for completeness; do last or skip.

### J. `--reasoning off` / `--reasoning-budget 0`
- qwen3-30b-a3b-**instruct** is non-thinking, but the chat template has think
  scaffolding. Confirm no `<think>` tokens are being generated (they'd inflate
  completion_tokens). `--reasoning off` / budget 0 guarantees it. Quick check
  against current 855-tok avg output. Decode impact: LOW (only if think tokens
  are present). Risk LOW. Quality NONE.

---

## 6. Prioritized benchmark plan

Ranked by gain × (low-risk) × (low-effort). Metric for every run:
**(1) pure decode tok/s** (`predicted_n / predicted_ms` from `/completion`
timings) and **(2) articles/hr** (extrapolated = 3600 / mean per-article wall on
the fixed article set). Quality gate on every config that can affect output.

### A/B method (don't disrupt the draining prod backlog)
The prod llama-server is busy draining the backlog. To benchmark a config
WITHOUT stealing its cores or corrupting its cache:
1. Spin a **second, ephemeral llama-server** container `llama-bench`, **pinned to
   a disjoint CCD set** so it does not steal prod's cores. Prod currently uses
   cpuset `0-23` (all 4 CCDs) — so for a clean A/B, **temporarily** repin prod
   to CCDs 0-1 (`0-11`) and give the bench CCDs 2-3 (`12-23`), OR run the bench
   during a low-backlog window. Cleaner: pause `ollama-cpu-gen`+`ollama-cpu-embed`
   (they steal SMT-sibling cycles, §2/§B) and split the 24 cores 12/12 between
   prod and bench for the duration. Mount the SAME RO GGUF blob.
2. Drive it with the **econ micro-bench harness** (`econ_measure.py`, already
   prod-fidelity: same prompt, GBNF, temp=0, n_predict, stop) on a FIXED set of
   ~12 articles spanning the length strata (reuse `econ_articles.jsonl`). Each
   config → `calls.jsonl` → `econ_analyze.py` → decode tok/s + wall.
3. Never point the bench at prod's port; never share the slot.

### Quality guardrail (recall must not regress)
For any config that changes OUTPUT (G quant, D KV-q4, H ctx-routing) OR to prove
A speculative is token-identical:
- Run the **truncation harness scorer** (`SentinelCollector/tools/
  truncation-experiment/scorer.py`) — restore it from
  `.claude/worktrees/agent-a7871f8ad3d1fed9c/.../scorer.py` (the file is only in
  worktrees; copy back to the tool dir). It scores recall/precision/hallucination
  via the **GPU vLLM judge** (Qwen2.5-32B-AWQ), which does NOT contend with the
  CoD CPU server.
- Method: run the SAME fixed article set through (a) the current prod config
  ("baseline" extractions) and (b) the candidate config; feed both to
  `scorer.py` with the baseline as the gold set. **PASS = candidate recall ≥
  baseline recall** (within judge noise; reuse the matching-validation GATE from
  Phase 0b — judge↔hand 100 %). For speculative (A): assert **byte-identical
  output** to greedy → no judge needed, but run scorer once as belt-and-braces.

### Test SEQUENCE
| # | Lever | Config to try | Expected decode gain | Quality gate |
|---|-------|---------------|----------------------|--------------|
| 0 | **Re-baseline** | current prod config on the fixed 12-article set | — (establish numbers) | record baseline extractions for scorer gold |
| 1 | **F + J (free insurance)** | add `--mlock`; confirm `--reasoning off` (no think tokens) | LOW (tail/insurance) | none (no output change) — spot-check |
| 2 | **B (threads/affinity sweep)** | `-t {12@CCD0-1, 24@all, 48@SMT}`, `--cpu-strict 1`, `--prio 2`, `--poll 100`; also measure with ollama gen/embed PAUSED | MED ±10–30 % | none (greedy, no output change) |
| 3 | **D (KV V-quant)** | add `-ctv q8_0` (keep K q8_0) | LOW–MED (few %) | scorer A/B (q8 low-risk) |
| 4 | **A (speculative) — TOP LEVER** | pull Qwen3-0.6B + 1.7B GGUF; `-md`, sweep `--spec-draft-n-max {4,8,16}`, `--spec-draft-p-min {0,0.5}`, `--cpu-mask-draft`=1 CCD | **HIGH 1.5–3×** | assert byte-identical output (+ 1 scorer run) |
| 5 | **G (quant)** *(only if 1–4 fall short)* | requantize/obtain IQ4_XS or q4_0 GGUF | LOW–MED | scorer A/B — hard gate |
| 6 | **H (short-pool routing)** *(highest ceiling, highest effort)* | 2nd llama-server 8K/parallel=4 for <4K-tok articles, 32K pool for long; CCD-split 12/12 | **HIGH on short majority** | scorer on both pools; long articles must keep 32K |

### Recommended FIRST benchmark
**Step 0 + Step 4 (speculative decoding) is the headline experiment**, because:
decode is 67–100 % of cost, the output is GBNF-rigid (high draft acceptance),
the gain is multiplicative (1.5–3×), and it has **zero recall risk** (exact
decode). Do Step 1 (free `--mlock`/reasoning-off) and Step 2 (thread/affinity,
no-output-change, instant revert) first only because they are 30-minute,
risk-free measurements that also set a clean baseline for the speculative run.
Concretely: re-baseline (Step 0) → `--mlock` + `--reasoning off` (Step 1) →
pull Qwen3-1.7B GGUF and run the speculative sweep (Step 4) pinned to a disjoint
CCD set so prod keeps draining.

---

## 7. Open items / confirm during baseline
- **RAM channels: CONFIRMED 4-of-8 populated (128 GB, channels A/C/E/G).**
  Filling B/D/F/H doubles bandwidth to ~307 GB/s. BUT §2 math shows we're
  latency-bound (≈12 GB/s effective at 8 tok/s), not bandwidth-saturated, so the
  DRAM upgrade helps LESS than speculative decoding — defer until software levers
  (A/B/D) are exhausted; it's a hardware-purchase decision, not a tuning knob.
- **Physical-core contention**: quantify the throughput tax of ollama-cpu-gen/
  embed running on CCD-0..3 SMT siblings (pause them, re-measure).
- **Restore `scorer.py`** from the worktree into the tool dir before any
  quality-gated run.
- Re-validate (don't assume) the `3a802551` choices on the current corpus —
  especially whether parallel=1 is still optimal for *throughput* (it is for
  latency); 16K/slot is out under the 32K rule regardless.
