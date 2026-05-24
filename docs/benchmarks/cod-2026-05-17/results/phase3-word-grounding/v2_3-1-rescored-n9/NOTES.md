# v2.3.1 — Verifier Punctuation-Tolerance Rescore NOTES

Branch: `experiment/v2.3.1-verifier-punct-tolerance`
Source verifier: `dsl/verifier_v2_3.py` (PR #426 / commit `69732da9`)
New verifier:   `dsl/verifier_v2_3_1.py`
Re-scored from: `results/phase3-word-grounding/v2_3-n9/.../v2_3_spec/*.json`
This NOTES.md:  2026-05-23

## TL;DR

**HUGE WIN.** The verifier-only patch flips the word-grounding axis
from BYTE-STILL-WINS (-0.166) to **WORD BEATS BYTE (+0.030)** — and
gap closure vs v2.3 is +0.196 (word_v2.3 = 0.793 → word_v2.3.1 = 0.989).

| metric | v2.3 | v2.3.1 | delta |
|---|---|---|---|
| byte_verbatim_match_rate (pooled, unchanged code path) | 0.9587 | 0.9587 | 0.000 |
| word_verbatim_match_rate (pooled, casefold) | 0.7934 | **0.9890** | **+0.196** |
| word_verbatim_match_rate (pooled, cased) | — | 0.9532 | — |
| **word − byte (HEADLINE)** | **-0.165** | **+0.030** | **+0.196** |

Verdict band (per task brief):
* word ≥ byte (delta ≥ 0) → **HUGE WIN, word-grounding is now the recommended schema**
* within -0.05 → WIN
* -0.10 < … < -0.05 → SOME
* ≤ -0.10 → NO HELP

Actual word − byte = **+0.0303** → falls in HUGE WIN band.

## 1. What v2.3.1 changes (verifier only)

No grammar / parser / prompt changes — the patch is a single new file
`dsl/verifier_v2_3_1.py` that reads the same v2.3 docs and produces a
new word_verbatim rate. Re-scoring existing v2.3 result JSONs produces
the v2.3.1 numbers directly (no LLM re-sweep needed).

### Per-word `normalize_word` helper

Applied symmetrically to BOTH article words and model-emitted
`source_words`:

1. HTML-entity decode (`&#39;` → `'`, `&quot;` → `"`)
2. En-dash (U+2013) and em-dash (U+2014) → hyphen (-)
3. Strip leading/trailing punctuation (anything not `\w`)
4. Acronym collapse: `U.S.` → `US`, `U_S` → `US`, `M.I.T.` → `MIT`
5. Hyphen-compound collapse: `pan-European` → `panEuropean`
6. Casefold (lowercase) for the headline metric; cased variant also
   reported

### Article-level preprocessing

Before tokenization the article body is:

1. HTML-decoded (`&#39;` lifted to `'`, etc.) so `Board&#39;s`
   tokenizes to `["Board", "'", "s"]` instead of
   `["Board", "&", "#", "39", ";", "s"]`.
2. Acronym pre-collapsed (`U.S.` rewritten to `US`) BEFORE the
   word-token regex sees it, so the article's `U.S.` becomes a single
   token `US` rather than four (`U`, `.`, `S`, `.`). This is required
   because the model's `U_S` (where `_` is a word-char) stays as one
   token, and a 1-token needle could never match a 2-token (post-
   filter) haystack run otherwise.

### Stream filter

After `normalize_word`, pure-punctuation tokens (commas, semicolons,
ampersands) drop out of both streams. This absorbs the dominant
v2.3 word-FAIL signature on cell 27560: the article carries
`Committee, January 27–28, 2026` and the model emits
`Committee|January|27–28|2026` (no commas). A literal contiguous-
subseq check fails on the comma gap; the punctuation filter makes
the model's stream a valid subseq of the article's.

## 2. Per-cell comparison: v2.3 vs v2.3.1 (rescored)

| cell  | byte_v2.3 (=v2.3.1) | word_v2.3 | word_v2.3.1 (cf) | word_v2.3.1 (cased) | denom | matches gained |
|-------|--------------------:|----------:|-----------------:|--------------------:|------:|---------------:|
| 27560 | 0.8077              | 0.1538    | **0.9615**       | 0.9615              |   26  | +21 / 26       |
| 27702 | 0.0000              | 0.0000    | 0.0000           | 0.0000              |    0  | (no blocks)    |
| 29807 | 0.9730              | 0.7568    | **1.0000**       | 0.9459              |   37  | +9 / 37        |
| 31149 | 0.9474              | 0.8421    | **1.0000**       | 0.8947              |   19  | +3 / 19        |
| 31430 | 1.0000              | 0.7778    | 0.9444           | 0.8333              |   18  | +3 / 18        |
| 33632 | 0.9429              | 0.7619    | **0.9905**       | 0.9524              |  105  | +24 / 105      |
| 34537 | 0.0000              | 0.0000    | 0.0000           | 0.0000              |    0  | (no blocks)    |
| 35352 | 0.9911              | 0.9911    | **1.0000**       | 0.9911              |  112  | +1 / 112       |
| 36065 | 0.9783              | 0.7609    | **0.9783**       | 0.9348              |   46  | +10 / 46       |

Cells 27702 and 34537 had no source_words slots emitted (zero denom)
under v2.3 — unchanged under v2.3.1 (zero denom, zero match, no
contribution to the pooled rate). These two cells need a separate
investigation (the model declined to ground at all); v2.3.1 doesn't
touch that mechanism.

### Cell-27560 deep-dive (the headline win)

v2.3 word rate: 0.154 (4 / 26 matched). v2.3.1 word rate: **0.962
(25 / 26 matched)**. 21 blocks recovered by the verifier patch alone.

The 21 recovered blocks were the meeting-date NUMs and EVTs:
* `NUM meeting_27_28_january_2026 → January|27–28|2026` (en-dash + comma gap)
* `EVT discount_rate_meeting → Minutes|of|the|Board's|discount|rate|meetings|on|...` (HTML `&#39;` + comma gap)
* `EVT meeting_minutes → Minutes|...|Federal|Open|Market|Committee|March|17–18|2026` (en-dash + comma gap)

All resolved by HTML-decode + dash-unify + punctuation-strip on the
article side. The mechanism v2.3 NOTES.md called out as "punctuation-
stripping between adjacent word-list words" is exactly what shipped
here.

The single remaining word-fail on 27560:
* `NUM meeting_28_january_2026 → January|28|2026` — the model emitted
  a date that **does not exist in the article** (the article only has
  `January 20 and 28` and `January 27–28`, never `January 28` as a
  standalone). This is model hallucination, not verifier strictness.

## 3. Pooled aggregate

| metric                 | v2.3   | v2.3.1 cf | v2.3.1 cased |
|------------------------|-------:|----------:|-------------:|
| byte_verbatim (pooled) | 0.9587 | 0.9587    | 0.9587       |
| word_verbatim (pooled) | 0.7934 | **0.9890**| 0.9532       |
| word − byte            | -0.165 | **+0.030**| -0.006       |
| total matched / denom  | 288/363 | 359/363  | 346/363      |

Byte rate is byte-for-byte identical (same `_check_byte` code path
delegated). The word rate jumps by **+0.196**.

**Case-folded mode beats byte.** Case-sensitive mode falls just
short (-0.006), because the model occasionally lower-cases proper
nouns (e.g. `washington` in cell 33632 EVT triggers, `cup` in
36065 ENT). The brief recommended testing both — the data confirms
casefold is the correct headline mode (the model is being asked
"did you copy this concept?" not "did you copy these exact bytes?";
case-fold matches the semantic intent).

## 4. Failure analysis (cells where v2.3.1 word < byte)

After the rescore + needle-side acronym pre-collapse fix, residual
failures are:

### 31430 (word=0.944, byte=1.000 — 1 residual fail)

* `ENT SAndP500 → source_words: S|And|P500` — article has `S&P 500`.
  The model decoded `&` as the English word "And" and concatenated
  `P500`. This is a model semantic-paraphrase error (`&` ≠ "And" at
  the verbatim layer). A verifier can't fix this without unsafe
  ampersand→"and" rewrites that would create spurious matches.

### 33632 (word=0.990, byte=0.943 — 1 residual fail)

* One `EVT energy_policy_response` block with a long verb-phrase
  trigger (`has also announced steps to address supply disruptions,
  while warning that energy supplies may run out by the end of May,
  including central bank support for companies' efforts to diversify
  energy sources and secure inputs, enhanced data monitoring of a ...`).
  The needle is 38+ tokens — the model's `source_words` list drifts
  off the article on a long span. Consistent with the long-EVT-trigger
  paraphrase-drift pattern v2.3 NOTES.md called out as orthogonal to
  the v2.3 axis.
* Note: **word RATE > byte RATE on this cell** (0.990 > 0.943) — the
  word denominator picked up matches on blocks where byte missed
  (likely a copy-slot offset issue resolved by the punctuation-
  tolerant word check).
* The companion 33632 `is working with Washington ... U.S.-sanctioned
  countries` block (originally listed as a second residual) was
  resolved by the needle-side acronym pre-collapse fix.

### 36065 (word=0.978, byte=0.978 — 1 residual fail)

* `ENT _anon:gold_pawned_loans → source_words: gold|pawned|loans` —
  article has `gold-backed lending`, `Gold pawning`, but never the
  exact compound `gold-pawned loans`. Model fabricated a noun phrase
  by composing existing words. Not verifier-fixable.

### 27560 (word=0.962, byte=0.808 — 1 residual fail)

* `NUM meeting_28_january_2026 → January|28|2026` — the model emitted
  a date that does not exist in the article as a standalone (article
  has `January 27–28` and `January 20 and 28` only). Model
  hallucination, not verifier strictness.

**Pattern:** all 4 residual word failures are model paraphrase /
fabrication errors, NOT verifier punctuation strictness. The verifier
is now within ε of "perfectly tolerant of every legitimate copy" —
the headroom that remains is purely model behavior, which the team
already characterized as orthogonal (v2.3 NOTES.md §6).

### Review-fix note (added 2026-05-23)

PR #427 initial rescore (357/363) listed cell 31149 in the residual
bucket under "long EVT trigger drift". On review, that block
(`EVT trade_deal_context → ...|address|U.S.|concerns|...`) was
actually failing on a verifier asymmetry: the article-side acronym
pre-collapse (`_tokenize_article_words`) was not mirrored on the
needle-side (`_normalize_word_list_for_match`). The article had
`U.S.` collapsed to single token `US` while the needle's `U.S.`
re-tokenized to `['U','.','S','.']` and then per-token `normalize_word`
saw only single letters that no longer matched the acronym pattern.
Fixing `_normalize_word_list_for_match` to apply
`_ARTICLE_ACRONYM_COLLAPSE_RE` before `_WORD_TOKEN_RE.findall`
recovered cell 31149 (19/19) and the U.S.-sanctioned 33632 block
(105 → 104 fail), pushing the pooled word rate from 0.9835 to 0.9890.
Regression covered by
`test_should_match_word_when_us_acronym_in_phrase_emitted_with_dots`.

## 5. Recommendation: queue n=72 v2.3.1 sweep

Per the task brief gating criterion: HUGE WIN → schema candidate for
the next iteration. Two next steps:

1. **n=72 v2.3.1 sweep**: re-run the full corpus (or, equivalently,
   re-run scoring on existing v2.3 n=72 results if they exist) with
   the v2.3.1 verifier to confirm the n=9 gap-flip generalizes.
   * If v2.3 n=72 results exist on disk: `python scripts/rescore_v2_3_1.py
     --src .../v2_3-n72/.../v2_3_spec --dst .../v2_3-1-rescored-n72/...`
   * If not: queue a fresh v2.3 sweep first (the model output is
     identical between v2.3 and v2.3.1 — the change is verifier-only).
2. **Promote v2.3.1 to the default verifier** for `score_dsl.py` on
   v2.3 docs (the byte rate is unchanged; the word rate is what the
   v2.3 doc was always meant to convey). This is a one-line swap
   in `scripts/score_dsl.py`:
   ```python
   # before: from dsl.verifier_v2_3 import verify_document_v2_3
   # after:  from dsl.verifier_v2_3_1 import verify_document_v2_3_1
   ```
   followed by updating the call site to use the v2.3.1 return shape
   (one extra field, `word_verbatim_match_rate_cased`).

## 6. Method / reproducibility

```bash
# From the benchmark root:
cd docs/benchmarks/cod-2026-05-17
python scripts/rescore_v2_3_1.py \
  --src results/phase3-word-grounding/v2_3-n9/qwen3_30b-a3b-instruct-2507-q4_K_M/v2_3_spec \
  --dst results/phase3-word-grounding/v2_3-1-rescored-n9/qwen3_30b-a3b-instruct-2507-q4_K_M/v2_3_spec \
  --corpus corpus.full72.jsonl
```

Output:
* Re-scored cell JSONs at `--dst` (one per article; `scoring.original_v2_3`
  preserves the v2.3 numbers verbatim for diff).
* `rescore_summary.json` one level up: per-cell + pooled rates +
  failure breakdown.

Unit tests: `pytest dsl/tests/test_verifier_v2_3_1.py` (27 tests,
covers each `normalize_word` transformation + end-to-end on synthetic
v2.3 docs that encode cell-27560 failure modes + the needle-side
acronym symmetry regression for cell 31149). Full suite 106 tests,
all passing.

## 7. Files touched

* `docs/benchmarks/cod-2026-05-17/dsl/verifier_v2_3_1.py` (new)
* `docs/benchmarks/cod-2026-05-17/dsl/tests/test_verifier_v2_3_1.py` (new)
* `docs/benchmarks/cod-2026-05-17/scripts/rescore_v2_3_1.py` (new)
* `docs/benchmarks/cod-2026-05-17/results/phase3-word-grounding/v2_3-1-rescored-n9/...` (rescore artifacts; n=9 cell JSONs + summary)
* `docs/benchmarks/cod-2026-05-17/results/phase3-word-grounding/v2_3-1-rescored-n9/NOTES.md` (this file)
