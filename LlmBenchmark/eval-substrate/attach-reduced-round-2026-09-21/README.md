# Attach reduced scoring round, 2026-09-21

The run behind the gate verdicts quoted in `LlmBenchmark/scripts/README.md` (A0 vs B-clean,
121 articles, 526 labelled owners of `attach-gold/attach_gold_g2_v1.json`). It is here
because a pass-or-stop decision was reported off these numbers while the only copy lived in
a session scratchpad: a remembered measurement, not a recorded one.

## WHAT IS HERE, AND WHAT IS DELIBERATELY NOT

Committed -- a GPU run cannot be reproduced from committed inputs, so the bytes ARE the
evidence:

| file | what it is |
|---|---|
| `attachclean-g4_vendor_r{1,2,3}-vllm028-20260921.predictions.jsonl` | the three B-clean replicates, `run_model.py --task pick` |
| `...predictions.jsonl.provenance.json` | each replicate's run provenance. **A REQUIRED SCORER INPUT** (`--adapter-meta`), not decoration: it carries `candidates.sha256`, and `eval_harness --task attach` REFUSES to score without it, because nothing else shows that `--candidates` is the list the model saw |
| `attachclean-g4_vendor_r{1,2,3}-vllm028-20260921.scorecard.json` | each replicate's scorecard |
| `attacha0-rows_sum-20260921.scorecard.json` | the A0 baseline arm's scorecard |
| `attachpaired-a0_vs_g4_vendor_r{1,2,3}-vllm028-20260921.json` | the paired-difference outputs, which are what gates 1 and 2 are stated on |
| `attach-g2-20260921.staleness.json` | the attestation that gated the run. Also **A REQUIRED SCORER INPUT** (`--staleness`): the accepted ids are re-checked before every scoring run |

NOT committed, because a committed input already determines them byte for byte. Both were
regenerated from the committed corpus, gold and frozen list during this round and came back
with the digest recorded below -- the substrate twice, the A0 arm three times:

| artifact | sha256 | regenerate with |
|---|---|---|
| `substrate_g2.json` | `a187850e339bd48d36be673e5ea784a2d608ee7e648d00ffbe54dd153a09ff6f` | `python3 LlmBenchmark/scripts/build_attach_substrate.py --corpus LlmBenchmark/attach-gold/attach_corpus_v1.json --gold LlmBenchmark/attach-gold/attach_gold_g2_v1.json --candidates LlmBenchmark/attach-gold/frozen/attach_candidates_g2_k20.json --out substrate_g2.json` |
| `a0_g2.jsonl` | `eb7d311d3f624d596656725c9670a481b01a299a772f0b97131bdefcef40f9f0` | `python3 LlmBenchmark/scripts/build_a0_predictions.py --gold LlmBenchmark/attach-gold/attach_gold_g2_v1.json --candidates LlmBenchmark/attach-gold/frozen/attach_candidates_g2_k20.json --out a0_g2.jsonl` |

The inputs those two digests are taken against, so a silent edit to either is visible:
gold `6914963a0b17bd211490a5e330d12d7ecf2257b38b4315858dec30cf563a6a48`, frozen candidates
`4ee067db697a78e4d8e7783aaa8eab66a403699ee15d1e8d1a69e1a0a9d2a2b0`.

Also not committed, and NOT an oversight: the round's scratchpad held `a0_anyid.jsonl` and
its scorecard, an A0 sensitivity arm that **no reduction this repo implements reproduces**.
It disagrees with today's `--reduction any_non_null` on 4 owners, and on every one of them
it holds the value that ordering `any_non_null`'s attachments by ASCENDING row count gives
-- the superseded leg that `build_a0_predictions.py`'s `reduction[any_non_null]` control now
kills. Committing it as `any_non_null` would record a measurement under the name of a rule
that did not produce it. The band figures below do not come from it.

## WHY A SUBDIRECTORY, AND NOT FLAT BESIDE THE cap8192 CARDS

Measured, not assumed. `check_staleness.py` assembles its corpus with a NON-RECURSIVE
`glob("*.json")` and skips only `.criteria.json` / `.provenance.json`, so flat placement
would have two effects, both bad and neither obvious:

* the paired outputs and the attestation are `*.json` that are not scorecards -- dropped in
  flat they grade `UNEXAMINED` and are COUNTED, taking the corpus from 27 files to 29;
* the scorecards would not be graded either way. An attach scorecard carries `metrics` but
  no `summary` block, which is what that tool reads, so **all four grade `UNEXAMINED`**
  (measured: `0/2 current` on a two-card staging directory). Flat placement buys this run
  nothing and costs the shared corpus its denominator.

A subdirectory is invisible to that glob, which is the right answer for files the tool
cannot grade. That `--task attach` scorecards are ungradeable at all is filed as measurement
debt in `docs/BACKLOG.md`.

## RE-DERIVING

Every figure quoted in `LlmBenchmark/scripts/README.md` comes back from these files plus the
committed gold and frozen list. Re-derived end to end 2026-09-21; the scorecards this
produces equal the committed ones on EVERY metric value.

```bash
G=LlmBenchmark/attach-gold/attach_gold_g2_v1.json
F=LlmBenchmark/attach-gold/frozen/attach_candidates_g2_k20.json
D=LlmBenchmark/eval-substrate/attach-reduced-round-2026-09-21
# regenerate the two artifacts above first, then, per replicate n:
B=$D/attachclean-g4_vendor_r${n}-vllm028-20260921
python3 LlmBenchmark/scripts/eval_harness.py --task attach --substrate substrate_g2.json \
    --attach-gold $G --candidates $F --staleness $D/attach-g2-20260921.staleness.json \
    --adapter-meta $B.predictions.jsonl.provenance.json \
    --predictions $B.predictions.jsonl --out card_r$n.json
python3 LlmBenchmark/scripts/paired_attach_diff.py --attach-gold $G --candidates $F \
    --a a0_g2.jsonl --b $B.predictions.jsonl --out paired_r$n.json \
    --a-alt baseline_first=a0_baseline_first.jsonl \
    --a-alt any_non_null=a0_any_non_null.jsonl \
    --a-alt tie_to_abstention=a0_tie_to_abstention.jsonl \
    --a-alt tie_to_largest_uuid=a0_tie_to_largest_uuid.jsonl
```

| figure | where it is quoted | re-derived value |
|---|---|---|
| b `wrong_attachment_rate` | gate 1, first term | 0.1578 / 0.1559 / 0.1559 for r1/r2/r3 -> **0.1559-0.1578** |
| b `correct_attachment_recall` | gate 2 | **0.8101** on all three replicates |
| paired recall CI lower bound, over 5 reductions x 3 replicates | gate 2 | **+0.4170 to +0.4465** |
| A0 `wrong_attachment_rate` band | gate 1, "the widest the band reaches" | **0.1331-0.1730** |

The replicate range is NOT the band: a single `--a` gives +0.4367 on all three replicates,
and the quoted +0.4170 to +0.4465 spans the five BASELINE REDUCTIONS. Reading one for the
other reports a replicate spread as a sensitivity band.

## REDACTION

Five path fields held an absolute scratchpad path when these files were written and were
rewritten to the artifact name at the point of writing: `.substrate` (provenance sidecars),
`.staleness.path` and `.adapter_metadata.substrate` (scorecards), `.a` and `.b` (paired
outputs). THREE of the five sit beside the sha256 that carries the identity; `.a` and `.b`
do NOT, and saying they did was wrong. Each paired output holds exactly one digest-valued
string, `.bootstrap.replicates_sha256`, which is an in-memory bootstrap draw and attests
neither arm -- so those 6 sites are outside what `stage_and_verify.py --verify` can read.
Nothing checkable was lost, because both identities are recorded elsewhere: `a0_g2.jsonl`'s
in the not-committed table above, and the B-clean predictions' in
`LlmBenchmark/scripts/README.md`. Neither is recorded BESIDE the field, which is the
convention the verifier reads, and closing that is a writer change on the next round
(docs/BACKLOG.md). Nothing else was altered -- predictions, metrics and every digest are
byte for byte as produced. No credential or hostname was present;
`endpoint: http://localhost:8000` is kept deliberately, as the run's own engine coordinate.

`stage_and_verify.py` beside these files is the transform, committed with its controls so
the redaction can be reproduced rather than only confirmed absent. It refuses an absolute
path its map does not cover, and it COPIES a file it has no mapping to apply instead of
re-serializing it -- a rule it gained the hard way. The first staging re-dumped every json
with sorted keys, including the attestation, which had nothing to redact: 381 bytes became
382 and the `staleness.sha256` all four scorecards record stopped resolving. That file is
back to its as-produced bytes and `control_noop_bytes` now fails by name if it happens again.

There is no `--stage`: the pre-staging originals are not committed and the scratch directory
is gone, so `stage()` is the transform OF RECORD and its controls are the only thing that
runs it. The CLI is these two commands and nothing else.

```bash
D=LlmBenchmark/eval-substrate/attach-reduced-round-2026-09-21
python3 $D/stage_and_verify.py --selftest                       # the controls
python3 $D/stage_and_verify.py --verify $D --repo-root LlmBenchmark
```

BOTH ARE IN CI, in the `Attachment round tooling selftests` step of
`.github/workflows/python-tests.yml`, which the `LlmBenchmark/eval-substrate/**` path
trigger fires on, and `scripts/tests/test_attach_round_selftest_wiring.py` pins that
`--selftest` actually reaches its control set. Until 2026-09-21 neither command ran unless
a person remembered -- and the defect they exist to catch had already happened once, on
this directory. Proven capable of red on the COMMITTED artifacts, not only on its fixtures:
appending one byte to `attach-g2-20260921.staleness.json` takes the step to rc 2 with a
MISMATCH at each of the four scorecards recording that digest; reverted and re-run green.

`--verify` resolves a digest against the artifact its sibling field names: **35 of the 77
digest-valued strings** in these documents, over 5 distinct (name, digest) pairs, 0
unresolvable and 0 mismatching. `--repo-root LlmBenchmark` is required rather than the repo
root, because the harness records gold and candidate paths relative to `LlmBenchmark/`;
without it 9 report NOT RESOLVABLE, which is the tool declining to call an unchecked digest
checked.

TWO RULES, and the split is deliberate. `{path, sha256}` is this round's own attestation
contract and REFUSES: a pair whose file is absent is reported and the run exits 2. Every
other spelling is read opportunistically and never refuses -- `<stem>_sha256` looks for
`<stem>` then `<stem>_path`, a bare `sha256` for `path` then `file`, and a site whose sibling
names nothing, or names something absent, is COUNTED UNREAD rather than failed. That is what
lets `substrate_sha256` stay a counted absence: `substrate_g2.json` is deliberately not
committed. An escape -- `..` or a symlink out of the roots the invocation named -- refuses
under both rules, so a verdict's scope equals the scope its arguments declared. It does NOT
mean the verdict is about artifacts this round produced, and it cannot: under the CI
invocation the second root is all of `LlmBenchmark/` (325 tracked files against this
directory's 16), and 31 of the 35 read sites legitimately name the gold (`2f68dd3b`), the
frozen candidates (`c64d5593`) and the pick prompt and schema -- none produced by this round.

The tool prints what it could NOT identify, so the blindness is in its own output:

```
coverage: 35 of 77 digest-valued string(s) in these documents were IDENTIFIED; 42 carry no
sibling this tool can resolve (chat_template_sha256 x9, generator_version x10,
prompt_file_sha256 x3, replicates_sha256 x7, schema_file_sha256 x3, substrate_sha256 x10)
```

Those 42 are not reader defects: two spellings digest a template STRING and an in-memory
bootstrap draw, six sites carry no sibling at all, ten name the uncommitted substrate, and
ten are `generator_version`, whose key is not `_sha256`-suffixed. The per-spelling census and
what would close each is in `docs/BACKLOG.md`.

Neither the coverage nor the verdict is a guarantee about the round. Both commands are
advisory checks a human reads: on this plan branch protection cannot block a merge on a red
one.
