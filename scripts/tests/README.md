# scripts/tests

Guard tests for the tools in `scripts/`. Run them by hand; nothing here is wired to a timer or a hook.

| File | Covers | Run |
|---|---|---|
| `new-epic-selftest.sh` | `../new-epic.sh` | `bash scripts/tests/new-epic-selftest.sh` |
| `test_verify_citations.py` | `../verify-citations.py` | needs **pytest**, which is not installed on this host — run it in a venv. `unittest discover` does NOT work on it: the file imports pytest and uses `tmp_path`, fixtures and `parametrize`, so discovery fails at import. |

## What a case here has to do

These are **known-bad controls**, not unit tests. Each one breaks the tool under test in one
documented way and requires the guard for that break to fire **with a message that names it** — an
exit code alone cannot distinguish "the guard fired" from "something else broke on the way to it",
which is the conflation the guards themselves exist to prevent.

Three rules the `new-epic` suite learned the hard way and that any new case should follow:

- **Assert the mutation landed.** A `sed` whose anchor stops matching produces a byte-identical copy,
  and the case then passes while testing the unmutated tool. `cmp -s` before scoring the verdict.
- **Test the success path, not only refusal.** An earlier revision of the `new-epic` suite tested
  refusal thoroughly and checked only the first line of what a successful run wrote. A strip that
  silently dropped 21 of 51 body lines — the WRITE_GATE roster, STANDING DIRECTIVES, ACTIVE EPIC
  and the ACCEPTANCE heading — passed 14/14.
- **Vary the axis the code reads.** Feeding a regex its own literal confirms the regex, not the
  shape. `## KNOWN DEFECTS` passing while `**Known Defects**` slipped through was found exactly that
  way.

Fixtures live under `mktemp` and `run_script_in` refuses an empty or repo-resident fixture path — an
unchecked `mktemp` returning empty makes `cd "" && bash …` run against the real checkout.
