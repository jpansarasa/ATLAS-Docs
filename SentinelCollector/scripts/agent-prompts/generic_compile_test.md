# generic_compile_test — reusable post-impl verify

# Objective
Run the full SentinelCollector devcontainer compile+test, parse its output, and report pass/fail counts plus the first failing test (if any). Fail-closed: any parse ambiguity is a FAIL.

# Inputs
- Script: `/home/james/ATLAS/SentinelCollector/.devcontainer/compile.sh` (MUST be run WITHOUT `--no-test`).
- Repo root: `/home/james/ATLAS` (cwd before running).
- Output log: `/tmp/sentinel-remediation/compile-$(date -u +%Y%m%dT%H%M%SZ).log` (create `/tmp/sentinel-remediation/` if missing).
- Optional `timeout_sec`: default 1800 (30 min) — dotnet restore + test run can be slow on cold cache.

# Commands
1. `mkdir -p /tmp/sentinel-remediation`
2. `ISO=$(date -u +%Y%m%dT%H%M%SZ); LOG=/tmp/sentinel-remediation/compile-${ISO}.log`
3. Run:
   `cd /home/james/ATLAS && timeout ${timeout_sec:-1800} SentinelCollector/.devcontainer/compile.sh 2>&1 | tee "$LOG"; EXIT=${PIPESTATUS[0]}`
4. Parse the log (do NOT rerun). Patterns:
   - **errors**: last line matching `^\s*(\d+)\s+Error\(s\)` (dotnet build summary). If the line says `0 Error(s)`, `errors=0`.
   - **warnings**: `^\s*(\d+)\s+Warning\(s\)` in the same summary block.
   - **tests passed**: sum of `Passed!\s+-\s+Failed:\s+\d+,\s+Passed:\s+(\d+)` across all `dotnet test` invocations in the log (one per test project).
   - **tests failed**: sum of `Failed:\s+(\d+)` across the same matches.
   - **tests skipped**: sum of `Skipped:\s+(\d+)`.
   - **first failing test**: first line matching `^\s*Failed\s+(\S+)` or `X\s+(\S+)\s+\[.*FAIL` (xUnit formats). Capture full test fully-qualified name.
   - **build failure** (pre-test): `Build FAILED.` preceding the test section — treat as compile fail, tests not meaningful.
5. If the exit code is nonzero but errors=0 and tests_failed=0, the script crashed — report `crash=true` with last 40 lines of log.

# Acceptance
- Log file exists, non-zero size.
- Parsed counts are integers (no nulls). If a pattern didn't match at all, report 0 and add a `parse_warnings` entry.
- If `errors>0` OR `tests_failed>0` OR `crash=true` → verdict FAIL, else PASS.

# Report shape
- `verdict`: `"PASS"` | `"FAIL"`
- `log_path`: absolute path
- `exit_code`: int
- `errors`: int
- `warnings`: int
- `tests_passed`: int
- `tests_failed`: int
- `tests_skipped`: int
- `first_failing_test`: string | null (FQ test name)
- `crash`: bool
- `parse_warnings`: list of strings (patterns that failed to match)
- `log_tail`: last 40 lines (only if `verdict == "FAIL"`)
