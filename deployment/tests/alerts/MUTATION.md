# Alert-rule mutation harness (MANUAL)

`run.sh` answers "do the alert rules still pass their tests". This answers the harder question:
**would the tests notice if the rule were wrong?** It changes the rule in one specific way and
re-runs the suite. A change the suite does not go RED on is a mutant that SURVIVED — a hole where
the rule could drift into that shape in production and nothing would say so.

The matrix is not wired into CI, on purpose. It is ~70 mutants x a container start each, which is
the wrong shape for a PR gate, and two of its four arms (EQUIVALENT, DECLINED) encode judgements a
gate cannot make. Run it when you change `sentinel.yml`'s
`SentinelCandidateSurfaceFilterCollapsed` rule or the fixtures that pin it, and diff the output
against a run from before your change.

`selftest.sh` IS in CI, and it drives this scorer through the CONTROL arms below. It is the answer
to "would this harness notice if the tests stopped testing" — it breaks the tooling one documented
way at a time and requires the guard for that break to fire with a message that names it. Two of
its cases are defects that shipped in the change which added this directory.

## Run it

```bash
export REPO=/path/to/worktree/deployment      # required; there is no default
cd deployment/tests/alerts

./selftest.sh                                  # do the guards still fire? (also runs in CI)
./matrix.sh                                    # the whole matrix, one verdict line per mutant
./run-mutant.sh drop-floor                     # one mutant
# is that fixture load-bearing? The name must match EXACTLY -- select.py refuses a near miss
# rather than silently selecting nothing.
./isolate.sh 'a bursty but healthy gate stays silent on the 6h window' win-1h-num win-1h-den
```

`matrix.sh` and `isolate.sh` both run the `none` control FIRST and **abort** if it does not come
back SURVIVED, printing nothing else. That is not defensive decoration: one schema-invalid key in
the test file used to make `none` — the mutant whose contract is that it survives — report KILLED,
after which the matrix narrated ~75 fabricated verdicts, gap-free, from a run that executed zero
assertions. A table whose control is broken has nothing to say.

`sudo nerdctl` is the default container runtime (mercury); override with `CONTAINER_RUN=docker`.
The prometheus image is pinned by digest in `run-mutant.sh` — the same pin `run.sh` carries, and it
must be bumped in the same change when the OTEL stack's prometheus is upgraded.

Logs land in `.mutation-logs/` as `<mutation>.<suite>.log`, the mirror in `.mutation-scratch/`; both
are gitignored, and the mirror is rebuilt from scratch on every single run. The suite is part of the
key because `isolate.sh` runs the same mutation names against three variants into one log directory,
and a mutation-only name left each variant overwriting the previous one's log.

## Exit codes

`run-mutant.sh` is the scorer. `matrix.sh` and `isolate.sh` render its codes; neither invents one.

| code | word | means |
|-----:|------|-------|
| 0 | `SURVIVED` | the suite stayed GREEN with the rule mutated. For a KILLABLE mutant this is the finding: add or sharpen a fixture. |
| 1 | `KILLED` | the suite went RED. The mutation is guarded. |
| 2 | `HARNESS_ERROR` | the run did not happen — bad `REPO`, a missing input, a failed copy, a `mkdir` that did not. **Never a verdict.** |
| 3 | `INERT_MUTATION` | the mutation changed not one byte. Scoring it would report a harness bug as a coverage hole. |
| 90 | `SYNTAX_ERROR` | the mutated rule does not parse. `promtool test rules` also exits 1 on this, which is indistinguishable from a fixture kill — so `check rules` is scored first and separately. |
| 91 | `RULE_SET_ERROR` | it parses, but it is not the rule set under test: the rule count moved, or the alert is gone by name. `promtool check rules` exits **0** on a zero-rule file (`SUCCESS: 0 rules found`), which is why the count is asserted rather than the rc. |
| 92 | `SIXTH_CLASS_DEFECT` | the suite loaded nothing or ran nothing. `promtool test rules` exits **0 / SUCCESS** when `rule_files` matches no path (stderr WARNING only) and when `tests:` is `[]`. |
| 94 | `SUITE_LOAD_ERROR` | the TEST FILE did not load, so no assertion ran. `promtool test rules` exits **1** for this and for a fixture kill alike — a key `UnmarshalStrict` rejects, an unparseable duration, a `values:` or `expr:` that does not parse. The rc cannot separate them, so promtool's output is required to show that assertions were actually evaluated before a non-zero rc is allowed to mean KILLED. |

93 is missing on purpose: `runsel.py` owns it for the seventh class, and reusing it here would put
two different findings on one number.

Codes 2, 3, 90, 91, 92 and 94 all exist for one reason: each of those states used to be reported as
KILLED or as a pass. Every defect this harness has had failed toward a **reassuring** answer.

## The arms in `matrix.sh`

- **KILLABLE** — a fixture could kill it and should. `SURVIVED` here is a gap.
- **SIDE-SCOPED** — the `-num` / `-den` variants. The unsuffixed swaps hit both operands and cannot
  answer "which side is unguarded"; the arm carries no expectation, the measurement is the answer.
- **EQUIVALENT** — changes the rule's text without changing what it DECIDES. No fixture can kill it,
  and `KILLED` here means the equivalence claim is wrong.
- **DECLINED** — genuinely killable, measured, and judged not worth a guard. The reason lives in
  `sentinel_test.yml` next to the fixtures. `KILLED` here means something now guards it: update the
  note.

## Why isolation never uses `--run`

`promtool test rules --run '<selector>'` exits **0 / SUCCESS** having run zero groups when the
selector matches no group name — silently, with output byte-identical to a clean full run. A case
renamed by a comment reword therefore turns "this fixture alone kills the mutant" into a run of
nothing in which every mutant SURVIVES, and the proof reads perfect.

`isolate.sh` physically subsets the suite with `select.py` (which exits non-zero on a name that
matches nothing) and pins the resulting case count via `EXPECT_CASES`, which `preflight.py` asserts
on the mirror. If you must use `--run` for something, `runsel.py <test-file> <expected-count>
<selector>...` asserts the selection selected what you intended — non-zero is necessary but not
sufficient, because a selector meant to isolate one fixture that quietly matches four is the same
class of lie.

## Files

| file | role |
|------|------|
| `run-mutant.sh` | scores ONE mutant; owns every exit code above |
| `mutate.py` | applies one named mutation, scoped to the alert's own block |
| `preflight.py` | classes 6 and 9 on the mirror: the suite really loads the mutated bytes and really runs cases |
| `matrix.sh` | the full catalogue, four arms; aborts on a broken control |
| `isolate.sh` | `only` / `minus` for one fixture; runs a `none` control per variant |
| `select.py` | emits a suite carrying only, or all but, the named groups |
| `runsel.py` | asserts what a `--run` selector would actually select |
| `selftest.sh` | breaks each of the above on purpose and requires its guard to fire (runs in CI) |

The checkers live beside these. `run.sh` runs two of them — `check-matchers.py` (class 8,
`alertname` and `alertstate` literals that can never select) and `check-assertion-counts.py`
(class 9, the suite's shape against `expected-counts.yml`). It does **not** run
`check-suite-ratchet.py` (class 9's own fail-open: the manifest that can be regenerated to match a
drained suite): that one scores a CHANGE and needs the manifest as of the merge base, which `run.sh`
has no access to, so it is its own step in `.github/workflows/alert-rules.yml`. A green `run.sh` is
not a statement about whether the suite got smaller.

`mutate.py`'s catalogue is specific to `SentinelCandidateSurfaceFilterCollapsed`: it names that
rule's metrics, window, threshold and floor. Another rule needs another catalogue — a mutation that
does not apply exits 3 rather than pretending to have run.
