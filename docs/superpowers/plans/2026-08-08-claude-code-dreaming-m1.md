# Dreaming M1 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build the nightly out-of-band memory consolidation loop: extract human turns from session transcripts, propose memory corrections with cited evidence, and apply approved changes behind guards.

**Architecture:** Four components joined by files on disk. A model-free Python extractor reduces 50 MB/day of transcripts to ~700 KB of human turns. A `claude -p` pass consolidates that against every memory surface into a numbered report. An interactive `/dream review` applies approvals behind an allowlist, snapshot, and content anchors. A systemd timer runs the first two at 03:00 and pushes to ntfy. The timer never writes to memory.

**Tech Stack:** Python 3.12 + pytest (extractor and apply path), bash (hooks, ntfy publish), systemd units, ansible, Claude Code skills.

**Spec:** `docs/superpowers/specs/2026-08-08-claude-code-dreaming-design.md`

## Global Constraints

- Write allowlist is exactly two paths: `~/.claude/projects/-home-james-ATLAS/memory/` and `/home/james/ATLAS/STATE.md`. Allowlist, never denylist.
- Report cap: **10 detailed items**; held-back findings counted and named in one line, never silently truncated.
- `PATTERN` findings require **at least 2 occurrences across at least 2 distinct session ids**.
- Parse-failure guard threshold: **1 percent** of lines.
- Tool-error snippets truncate at **200 characters**.
- Retention: `input/` 30 days, `snapshots/` 30 generations, `rejected.md` and `applied.md` permanent append-only.
- ntfy topic: `atlas-claude-ask`. Publish only when there are NEW proposals.
- Timer: 03:00 local. The timer runs extraction and consolidation only, never apply.
- `MEMORY.md` load ceiling: 200 lines or 25 KB, whichever first. Currently 91 lines / 17.4 KB.
- All agent-facing docs (skill files, hook headers) are **plain ASCII** -- no unicode operators. Per CLAUDE.md AGENT_DOC_STYLE.
- Every guard ships a test that constructs the violation and goes RED when the guard is deleted.
- Python code is never placed in `scripts/` (shell-only by convention).

## Deviation from the spec, recorded deliberately

The spec section 5.2 places the extractor at `scripts/dream-extract.py`. `scripts/` contains only shell (11 files, zero Python, no pytest config), so tests would have nowhere to live. This plan instead uses a self-contained `dream/` top-level directory, matching the established Python pattern in `backtest/`, `ntfy-mcp/`, and `gemini-resolver-mcp/`. Nothing else in the spec changes.

## File Structure

| File | Responsibility |
| --- | --- |
| `dream/dream_extract.py` | Transcript records in, filtered corpus out. Pure functions plus the D-1 plausibility guard. No model, no network, no memory writes. |
| `dream/dream_apply.py` | Approved findings in, guarded writes out. D-2 snapshot, D-3 allowlist, D-4 anchors, D-5 auto-apply scope, D-6 provenance, ledger. |
| `dream/paths.py` | Single source of truth for every path and threshold constant. Env-overridable so tests never touch real memory. |
| `dream/tests/test_extract.py` | Extractor unit tests: classify, strip, index. |
| `dream/tests/test_extract_guard.py` | D-1 guard tests. |
| `dream/tests/test_apply_safety.py` | D-2, D-3, D-4 guard tests. |
| `dream/tests/test_apply_semantics.py` | D-5, D-6, ledger, idempotence. |
| `dream/tests/fixtures/*.jsonl` | Hand-built transcript fixtures. |
| `dream/pytest.ini`, `dream/requirements.txt` | Test config and deps. |
| `.claude/skills/dream/SKILL.md` | Consolidation instructions and report schema. |
| `.claude/hooks/dream-pending-notice.sh` | SessionStart one-line notice when the report has unreviewed items. |
| `deployment/artifacts/atlas-dream.{service,timer}` | Schedule. |
| `deployment/artifacts/scripts/atlas-dream-run.sh` | Timer entrypoint: extract, consolidate, publish. |

Runtime state (never in the repo): `~/.claude/projects/-home-james-ATLAS/dream/{input,snapshots,digests}/`, `DREAM_REPORT.md`, `rejected.md`, `applied.md`.

---

# PR 1 -- Extractor and the D-1 guard

### Task 1: Project skeleton and path constants

**Files:**
- Create: `dream/pytest.ini`, `dream/requirements.txt`, `dream/paths.py`, `dream/tests/__init__.py`
- Test: `dream/tests/test_paths.py`

**Interfaces:**
- Consumes: nothing.
- Produces: `dream.paths` exporting `TRANSCRIPT_ROOT: Path`, `DREAM_ROOT: Path`, `MEMORY_ROOT: Path`, `STATE_MD: Path`, `ERROR_SNIPPET_CHARS: int = 200`, `PARSE_FAILURE_RATIO_MAX: float = 0.01`, `REPORT_ITEM_CAP: int = 10`, `INPUT_RETENTION_DAYS: int = 30`, `SNAPSHOT_GENERATIONS: int = 30`.

- [ ] **Step 1: Write the failing test**

```python
# dream/tests/test_paths.py
import importlib, os
from pathlib import Path


def test_roots_are_env_overridable(monkeypatch, tmp_path):
    monkeypatch.setenv("DREAM_TRANSCRIPT_ROOT", str(tmp_path / "tx"))
    monkeypatch.setenv("DREAM_ROOT", str(tmp_path / "dr"))
    monkeypatch.setenv("DREAM_MEMORY_ROOT", str(tmp_path / "mem"))
    monkeypatch.setenv("DREAM_STATE_MD", str(tmp_path / "STATE.md"))
    import dream.paths as p
    importlib.reload(p)
    assert p.TRANSCRIPT_ROOT == tmp_path / "tx"
    assert p.DREAM_ROOT == tmp_path / "dr"
    assert p.MEMORY_ROOT == tmp_path / "mem"
    assert p.STATE_MD == tmp_path / "STATE.md"


def test_constants_match_spec(monkeypatch, tmp_path):
    monkeypatch.setenv("DREAM_ROOT", str(tmp_path))
    import dream.paths as p
    importlib.reload(p)
    assert p.ERROR_SNIPPET_CHARS == 200
    assert p.PARSE_FAILURE_RATIO_MAX == 0.01
    assert p.REPORT_ITEM_CAP == 10
    assert p.INPUT_RETENTION_DAYS == 30
    assert p.SNAPSHOT_GENERATIONS == 30
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd /home/james/ATLAS/dream && python3 -m pytest tests/test_paths.py -v`
Expected: FAIL with `ModuleNotFoundError: No module named 'dream.paths'`

- [ ] **Step 3: Write the implementation**

```python
# dream/paths.py
"""Every path and threshold the dream tooling uses, in one place.

Env-overridable so tests never touch the real memory directory. A test that
can only run against live memory is a test nobody runs.
"""
import os
from pathlib import Path

_PROJECT = "-home-james-ATLAS"


def _env_path(var: str, default: str) -> Path:
    return Path(os.environ.get(var, default)).expanduser()


TRANSCRIPT_ROOT = _env_path("DREAM_TRANSCRIPT_ROOT", f"~/.claude/projects/{_PROJECT}")
DREAM_ROOT = _env_path("DREAM_ROOT", f"~/.claude/projects/{_PROJECT}/dream")
MEMORY_ROOT = _env_path("DREAM_MEMORY_ROOT", f"~/.claude/projects/{_PROJECT}/memory")
STATE_MD = _env_path("DREAM_STATE_MD", "/home/james/ATLAS/STATE.md")

INPUT_DIR = DREAM_ROOT / "input"
SNAPSHOT_DIR = DREAM_ROOT / "snapshots"
REPORT = DREAM_ROOT / "DREAM_REPORT.md"
REJECTED = DREAM_ROOT / "rejected.md"
APPLIED = DREAM_ROOT / "applied.md"
LOCKFILE = DREAM_ROOT / "review.lock"

ERROR_SNIPPET_CHARS = 200
PARSE_FAILURE_RATIO_MAX = 0.01
REPORT_ITEM_CAP = 10
INPUT_RETENTION_DAYS = 30
SNAPSHOT_GENERATIONS = 30
```

```ini
# dream/pytest.ini
[pytest]
# Tests import `dream.*` as a package; rootdir is the repo so the parent of
# dream/ lands on sys.path. No network and no live-memory access in any test:
# every path is redirected via the DREAM_* env vars in paths.py.
testpaths = tests
addopts = -q
```

```
# dream/requirements.txt
pytest>=8.0
```

Also create empty `dream/__init__.py` and `dream/tests/__init__.py`.

- [ ] **Step 4: Run test to verify it passes**

Run: `cd /home/james/ATLAS && python3 -m pytest dream/tests/test_paths.py -v`
Expected: 2 passed

- [ ] **Step 5: Commit**

```bash
git add dream/__init__.py dream/paths.py dream/pytest.ini dream/requirements.txt dream/tests/__init__.py dream/tests/test_paths.py
git commit -m "feat(dream): path and threshold constants, env-overridable for tests"
```

---

### Task 2: Record classification

**Files:**
- Create: `dream/dream_extract.py`
- Test: `dream/tests/test_extract.py`

**Interfaces:**
- Consumes: `dream.paths.ERROR_SNIPPET_CHARS`.
- Produces: `classify(record: dict) -> str | None` returning `"human"`, `"assistant"`, `"tool_error"`, or `None` for dropped records.

- [ ] **Step 1: Write the failing test**

```python
# dream/tests/test_extract.py
from dream.dream_extract import classify


def _user(content):
    return {"type": "user", "message": {"content": content}}


def test_plain_user_text_is_human():
    assert classify(_user([{"type": "text", "text": "use the whisper mcp"}])) == "human"


def test_string_content_user_is_human():
    assert classify(_user("do the thing")) == "human"


def test_tool_result_is_dropped():
    rec = _user([{"type": "tool_result", "content": "x" * 5000}])
    assert classify(rec) is None


def test_failed_tool_result_is_tool_error():
    rec = _user([{"type": "tool_result", "is_error": True, "content": "boom"}])
    assert classify(rec) == "tool_error"


def test_meta_user_is_dropped():
    rec = _user([{"type": "text", "text": "hi"}])
    rec["isMeta"] = True
    assert classify(rec) is None


def test_assistant_with_prose_is_assistant():
    rec = {"type": "assistant", "message": {"content": [{"type": "text", "text": "I found it"}]}}
    assert classify(rec) == "assistant"


def test_assistant_with_only_tool_use_is_dropped():
    rec = {"type": "assistant", "message": {"content": [{"type": "tool_use", "name": "Bash", "input": {}}]}}
    assert classify(rec) is None


def test_structural_records_are_dropped():
    for t in ("attachment", "queue-operation", "mode", "permission-mode",
              "ai-title", "pr-link", "last-prompt", "file-history-snapshot",
              "file-history-delta", "system"):
        assert classify({"type": t}) is None
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd /home/james/ATLAS && python3 -m pytest dream/tests/test_extract.py -v`
Expected: FAIL with `ModuleNotFoundError: No module named 'dream.dream_extract'`

- [ ] **Step 3: Write the implementation**

```python
# dream/dream_extract.py
"""Reduce Claude Code session transcripts to the 0.4 percent that carries signal.

Measured 2026-08-08: 49.8 MB/day of transcripts contain 699.7 KB of actual
human turns. tool_result bodies alone are 22.49 MB. Feeding the raw corpus to a
model overshoots context by ~250x, so this filter is what makes the design
possible, not an optimization.

No model, no network, no writes outside DREAM_ROOT.
"""
from __future__ import annotations

from dream.paths import ERROR_SNIPPET_CHARS

# Records that carry no consolidation signal at all. Listed explicitly rather
# than inferred, so a new record type defaults to "unknown" and is visible in
# stats instead of being silently swallowed.
_STRUCTURAL = frozenset({
    "attachment", "queue-operation", "mode", "permission-mode", "ai-title",
    "pr-link", "last-prompt", "file-history-snapshot", "file-history-delta",
    "system",
})


def _parts(record: dict) -> list[dict]:
    content = record.get("message", {}).get("content")
    if isinstance(content, str):
        return [{"type": "text", "text": content}]
    if isinstance(content, list):
        return [p for p in content if isinstance(p, dict)]
    return []


def classify(record: dict) -> str | None:
    """Return the kind of signal this record carries, or None to drop it."""
    rtype = record.get("type")
    if rtype in _STRUCTURAL:
        return None

    if rtype == "user":
        # isMeta and compact summaries are harness-authored, not the user
        # speaking. Treating them as human turns teaches the pass "preferences"
        # invented by its own scaffolding.
        if record.get("isMeta") or record.get("isCompactSummary"):
            return None
        parts = _parts(record)
        results = [p for p in parts if p.get("type") == "tool_result"]
        if results:
            return "tool_error" if any(p.get("is_error") for p in results) else None
        return "human" if any(p.get("type") == "text" for p in parts) else None

    if rtype == "assistant":
        # Prose only. tool_use params are the agent talking to a tool, not
        # stating a conclusion that could later be falsified.
        return "assistant" if any(p.get("type") == "text" for p in _parts(record)) else None

    return None
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd /home/james/ATLAS && python3 -m pytest dream/tests/test_extract.py -v`
Expected: 8 passed

- [ ] **Step 5: Commit**

```bash
git add dream/dream_extract.py dream/tests/test_extract.py
git commit -m "feat(dream): classify transcript records by signal kind"
```

---

### Task 3: Noise stripping

**Files:**
- Modify: `dream/dream_extract.py`
- Test: `dream/tests/test_extract.py`

**Interfaces:**
- Consumes: nothing new.
- Produces: `strip_noise(text: str) -> str`.

- [ ] **Step 1: Write the failing test**

Append to `dream/tests/test_extract.py`:

```python
from dream.dream_extract import strip_noise


def test_system_reminder_block_removed():
    t = "keep this <system-reminder>injected harness context</system-reminder> and this"
    assert strip_noise(t) == "keep this and this"


def test_multiline_system_reminder_removed():
    t = "before\n<system-reminder>\nline one\nline two\n</system-reminder>\nafter"
    assert strip_noise(t) == "before\nafter"


def test_command_wrapper_normalized():
    t = "<command-message>supervisor-mode</command-message><command-name>/supervisor-mode</command-name>"
    assert strip_noise(t) == "[slash: /supervisor-mode]"


def test_command_wrapper_with_trailing_prose_kept():
    t = "<command-name>/dream</command-name>\nnow do the thing"
    assert strip_noise(t) == "[slash: /dream]\nnow do the thing"


def test_plain_text_untouched():
    assert strip_noise("nothing to strip here") == "nothing to strip here"
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd /home/james/ATLAS && python3 -m pytest dream/tests/test_extract.py -k strip -v`
Expected: FAIL with `ImportError: cannot import name 'strip_noise'`

- [ ] **Step 3: Write the implementation**

Add to `dream/dream_extract.py`:

```python
import re

# Harness-injected context wrapped in these tags is not the user's words.
# Without this the pass mines its own scaffolding for "preferences".
_SYSTEM_REMINDER = re.compile(r"<system-reminder>.*?</system-reminder>\s*", re.DOTALL)
_COMMAND_MESSAGE = re.compile(r"<command-message>.*?</command-message>\s*", re.DOTALL)
_COMMAND_NAME = re.compile(r"<command-name>(.*?)</command-name>\s*", re.DOTALL)


def strip_noise(text: str) -> str:
    """Remove harness scaffolding, normalize slash invocations to a marker.

    Invocations are kept (which skill ran is signal) but flattened so they do
    not read as prose the consolidator could mistake for a stated preference.
    """
    text = _SYSTEM_REMINDER.sub("", text)
    text = _COMMAND_MESSAGE.sub("", text)
    text = _COMMAND_NAME.sub(lambda m: f"[slash: {m.group(1).strip()}]\n", text)
    # Collapse the blank lines the substitutions leave behind.
    text = re.sub(r"\n{3,}", "\n\n", text)
    return text.strip()
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd /home/james/ATLAS && python3 -m pytest dream/tests/test_extract.py -v`
Expected: 13 passed

- [ ] **Step 5: Commit**

```bash
git add dream/dream_extract.py dream/tests/test_extract.py
git commit -m "feat(dream): strip harness scaffolding from kept text"
```

---

### Task 4: Session extraction and stats

**Files:**
- Modify: `dream/dream_extract.py`
- Create: `dream/tests/fixtures/normal_session.jsonl`
- Test: `dream/tests/test_extract.py`

**Interfaces:**
- Consumes: `classify`, `strip_noise`, `dream.paths.ERROR_SNIPPET_CHARS`.
- Produces: `SessionStats` dataclass with fields `session: str`, `total_lines: int`, `parse_failures: int`, `assistant_records: int`, `human_turns: int`; and `extract_session(path: Path) -> tuple[list[dict], SessionStats]`. Output records are dicts with keys `session`, `ts`, `turn`, `kind`, `text`, plus `tool` when `kind == "tool_error"`.

- [ ] **Step 1: Write the failing test**

Create `dream/tests/fixtures/normal_session.jsonl` -- one JSON object per line, no trailing newline issues:

```jsonl
{"type":"user","timestamp":"2026-08-07T14:00:00Z","message":{"content":[{"type":"text","text":"fix the collector"}]}}
{"type":"assistant","timestamp":"2026-08-07T14:00:05Z","message":{"content":[{"type":"text","text":"I will check the logs"}]}}
{"type":"assistant","timestamp":"2026-08-07T14:00:06Z","message":{"content":[{"type":"tool_use","name":"Bash","input":{"command":"ls"}}]}}
{"type":"user","timestamp":"2026-08-07T14:00:07Z","message":{"content":[{"type":"tool_result","content":"file listing here"}]}}
{"type":"user","timestamp":"2026-08-07T14:00:08Z","message":{"content":[{"type":"tool_result","is_error":true,"content":"ERRORTEXT"}]},"toolName":"Bash"}
{"type":"attachment","timestamp":"2026-08-07T14:00:09Z"}
```

Append to `dream/tests/test_extract.py`:

```python
from pathlib import Path
from dream.dream_extract import extract_session

FIXTURES = Path(__file__).parent / "fixtures"


def test_extract_session_keeps_only_signal():
    records, stats = extract_session(FIXTURES / "normal_session.jsonl")
    kinds = [r["kind"] for r in records]
    assert kinds == ["human", "assistant", "tool_error"]
    assert records[0]["text"] == "fix the collector"
    assert records[2]["tool"] == "Bash"


def test_extract_session_counts_stats():
    _, stats = extract_session(FIXTURES / "normal_session.jsonl")
    assert stats.human_turns == 1
    assert stats.assistant_records == 2
    assert stats.parse_failures == 0
    assert stats.total_lines == 6
    assert stats.session == "normal_session"


def test_tool_error_snippet_truncated_at_200(tmp_path):
    p = tmp_path / "long.jsonl"
    p.write_text(
        '{"type":"user","message":{"content":[{"type":"tool_result",'
        '"is_error":true,"content":"' + "z" * 5000 + '"}]}}\n'
    )
    records, _ = extract_session(p)
    assert len(records[0]["text"]) == 200


def test_unparseable_line_counted_not_fatal(tmp_path):
    p = tmp_path / "broken.jsonl"
    p.write_text(
        'NOT JSON AT ALL\n'
        '{"type":"user","message":{"content":[{"type":"text","text":"ok"}]}}\n'
    )
    records, stats = extract_session(p)
    assert stats.parse_failures == 1
    assert len(records) == 1
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd /home/james/ATLAS && python3 -m pytest dream/tests/test_extract.py -k session -v`
Expected: FAIL with `ImportError: cannot import name 'extract_session'`

- [ ] **Step 3: Write the implementation**

Add to `dream/dream_extract.py`:

```python
import json
from dataclasses import dataclass
from pathlib import Path


@dataclass
class SessionStats:
    session: str
    total_lines: int
    parse_failures: int
    assistant_records: int
    human_turns: int


def _text_of(record: dict) -> str:
    return "".join(
        p.get("text", "") for p in _parts(record) if p.get("type") == "text"
    )


def _error_of(record: dict) -> str:
    for p in _parts(record):
        if p.get("type") == "tool_result" and p.get("is_error"):
            body = p.get("content")
            if not isinstance(body, str):
                body = json.dumps(body)
            return body[:ERROR_SNIPPET_CHARS]
    return ""


def extract_session(path: Path) -> tuple[list[dict], SessionStats]:
    """Read one transcript, return kept records plus the stats D-1 checks."""
    session = path.stem
    out: list[dict] = []
    total = failures = assistants = humans = 0

    for turn, line in enumerate(path.read_text(errors="replace").splitlines()):
        if not line.strip():
            continue
        total += 1
        try:
            record = json.loads(line)
        except (json.JSONDecodeError, ValueError):
            failures += 1
            continue

        if record.get("type") == "assistant":
            assistants += 1

        kind = classify(record)
        if kind is None:
            continue

        if kind == "tool_error":
            text = _error_of(record)
        else:
            text = strip_noise(_text_of(record))
        if not text:
            continue

        if kind == "human":
            humans += 1

        entry = {
            "session": session,
            "ts": record.get("timestamp", ""),
            "turn": turn,
            "kind": kind,
            "text": text,
        }
        if kind == "tool_error":
            entry["tool"] = record.get("toolName", "unknown")
        out.append(entry)

    return out, SessionStats(session, total, failures, assistants, humans)
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd /home/james/ATLAS && python3 -m pytest dream/tests/test_extract.py -v`
Expected: 17 passed

- [ ] **Step 5: Commit**

```bash
git add dream/dream_extract.py dream/tests/fixtures/normal_session.jsonl dream/tests/test_extract.py
git commit -m "feat(dream): per-session extraction with stats"
```

---

### Task 5: The D-1 fail-loud guard

**Files:**
- Modify: `dream/dream_extract.py`
- Create: `dream/tests/fixtures/renamed_schema.jsonl`
- Test: `dream/tests/test_extract_guard.py`

**Interfaces:**
- Consumes: `SessionStats`, `dream.paths.PARSE_FAILURE_RATIO_MAX`.
- Produces: `class CorpusImplausible(Exception)`; `assert_corpus_plausible(stats: list[SessionStats], files_seen: int) -> None`.

- [ ] **Step 1: Write the failing test**

Create `dream/tests/fixtures/renamed_schema.jsonl` -- a transcript where the content field has been renamed, simulating upstream schema drift. Assistant records still parse, human turns yield nothing:

```jsonl
{"type":"user","timestamp":"2026-08-07T14:00:00Z","message":{"body":[{"type":"text","text":"fix the collector"}]}}
{"type":"assistant","timestamp":"2026-08-07T14:00:05Z","message":{"content":[{"type":"text","text":"working on it"}]}}
```

```python
# dream/tests/test_extract_guard.py
import pytest
from pathlib import Path
from dream.dream_extract import (
    CorpusImplausible, SessionStats, assert_corpus_plausible, extract_session,
)

FIXTURES = Path(__file__).parent / "fixtures"


def test_session_with_assistant_but_no_human_trips_guard():
    """A session cannot exist without a prompt. Zero human turns beside a live
    assistant record means the schema moved, not that the user was quiet."""
    stats = [SessionStats("s1", total_lines=2, parse_failures=0,
                          assistant_records=1, human_turns=0)]
    with pytest.raises(CorpusImplausible, match="human"):
        assert_corpus_plausible(stats, files_seen=1)


def test_renamed_field_fixture_trips_guard():
    _, stats = extract_session(FIXTURES / "renamed_schema.jsonl")
    with pytest.raises(CorpusImplausible):
        assert_corpus_plausible([stats], files_seen=1)


def test_quiet_day_does_not_trip_guard():
    """Volume-independent: one turn in one session is a quiet Sunday, not a bug."""
    stats = [SessionStats("s1", total_lines=4, parse_failures=0,
                          assistant_records=2, human_turns=1)]
    assert_corpus_plausible(stats, files_seen=1)


def test_session_with_no_assistant_records_is_exempt():
    """An abandoned session with no assistant reply is legitimate."""
    stats = [SessionStats("s1", total_lines=1, parse_failures=0,
                          assistant_records=0, human_turns=0)]
    assert_corpus_plausible(stats, files_seen=1)


def test_parse_failures_above_one_percent_trip_guard():
    stats = [SessionStats("s1", total_lines=1000, parse_failures=11,
                          assistant_records=5, human_turns=5)]
    with pytest.raises(CorpusImplausible, match="parse"):
        assert_corpus_plausible(stats, files_seen=1)


def test_parse_failures_at_one_percent_pass():
    stats = [SessionStats("s1", total_lines=1000, parse_failures=10,
                          assistant_records=5, human_turns=5)]
    assert_corpus_plausible(stats, files_seen=1)


def test_files_changed_but_no_sessions_trips_guard():
    with pytest.raises(CorpusImplausible, match="no sessions"):
        assert_corpus_plausible([], files_seen=7)


def test_no_files_and_no_sessions_is_fine():
    assert_corpus_plausible([], files_seen=0)
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd /home/james/ATLAS && python3 -m pytest dream/tests/test_extract_guard.py -v`
Expected: FAIL with `ImportError: cannot import name 'CorpusImplausible'`

- [ ] **Step 3: Write the implementation**

Add to `dream/dream_extract.py`:

```python
from dream.paths import PARSE_FAILURE_RATIO_MAX


class CorpusImplausible(Exception):
    """The extracted corpus cannot be trusted; refuse to produce a report."""


def assert_corpus_plausible(stats: list[SessionStats], files_seen: int) -> None:
    """INTENT(D-1): a dead extractor must never report health.

    The failure this prevents is silent: extraction returns nothing, the
    consolidation pass dutifully reports "memory looks current", and the
    operator believes it. That is a corpse detector -- it signals health after
    the thing it monitors is already dead.

    Deliberately NOT a volume threshold. A quiet Sunday is legitimately five
    turns, so any absolute floor is either noise or useless. The invariant is
    structural: a session cannot exist without a prompt.
    """
    if files_seen and not stats:
        raise CorpusImplausible(
            f"no sessions parsed from {files_seen} changed transcript files"
        )

    for s in stats:
        if s.assistant_records > 0 and s.human_turns == 0:
            raise CorpusImplausible(
                f"session {s.session}: {s.assistant_records} assistant records "
                f"but 0 human turns -- transcript schema has moved"
            )

    total = sum(s.total_lines for s in stats)
    failures = sum(s.parse_failures for s in stats)
    if total and failures / total > PARSE_FAILURE_RATIO_MAX:
        raise CorpusImplausible(
            f"parse failures {failures}/{total} exceed "
            f"{PARSE_FAILURE_RATIO_MAX:.0%} -- transcript format has changed"
        )
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd /home/james/ATLAS && python3 -m pytest dream/tests/test_extract_guard.py -v`
Expected: 8 passed

- [ ] **Step 5: Verify the guard is not decorative**

Comment out the `assistant_records > 0 and human_turns == 0` raise, re-run, confirm 2 tests go RED, then restore.

Run: `cd /home/james/ATLAS && python3 -m pytest dream/tests/test_extract_guard.py -v`
Expected while disabled: `test_session_with_assistant_but_no_human_trips_guard` and `test_renamed_field_fixture_trips_guard` both FAIL.

- [ ] **Step 6: Commit**

```bash
git add dream/dream_extract.py dream/tests/fixtures/renamed_schema.jsonl dream/tests/test_extract_guard.py
git commit -m "feat(dream): D-1 fail-loud corpus plausibility guard"
```

---

### Task 6: CLI entrypoint with guarded exit

**Files:**
- Modify: `dream/dream_extract.py`
- Test: `dream/tests/test_extract_guard.py`

**Interfaces:**
- Consumes: everything above, `dream.paths.INPUT_DIR`, `INPUT_RETENTION_DAYS`.
- Produces: `main(argv: list[str]) -> int` returning 0 on success and 2 on guard trip. Writes `INPUT_DIR/YYYY-MM-DD.jsonl` and prints a one-line JSON summary `{"sessions":N,"human_turns":N,"records":N,"path":"..."}` to stdout.

- [ ] **Step 1: Write the failing test**

Append to `dream/tests/test_extract_guard.py`:

```python
import importlib, json, os, shutil


def _isolated(monkeypatch, tmp_path, fixture_name):
    tx = tmp_path / "tx"
    tx.mkdir()
    shutil.copy(FIXTURES / fixture_name, tx / "sessionA.jsonl")
    monkeypatch.setenv("DREAM_TRANSCRIPT_ROOT", str(tx))
    monkeypatch.setenv("DREAM_ROOT", str(tmp_path / "dream"))
    import dream.paths, dream.dream_extract as dx
    importlib.reload(dream.paths)
    importlib.reload(dx)
    return dx, tmp_path / "dream"


def test_main_writes_input_and_exits_zero(monkeypatch, tmp_path):
    dx, dream_root = _isolated(monkeypatch, tmp_path, "normal_session.jsonl")
    assert dx.main(["--since-hours", "99999"]) == 0
    written = list((dream_root / "input").glob("*.jsonl"))
    assert len(written) == 1
    kinds = [json.loads(l)["kind"] for l in written[0].read_text().splitlines()]
    assert "human" in kinds


def test_main_writes_no_input_and_exits_two_on_guard_trip(monkeypatch, tmp_path):
    """The whole point of D-1: a tripped guard must leave NO artifact behind
    that a downstream pass could mistake for a clean corpus."""
    dx, dream_root = _isolated(monkeypatch, tmp_path, "renamed_schema.jsonl")
    assert dx.main(["--since-hours", "99999"]) == 2
    input_dir = dream_root / "input"
    written = list(input_dir.glob("*.jsonl")) if input_dir.exists() else []
    assert written == []
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd /home/james/ATLAS && python3 -m pytest dream/tests/test_extract_guard.py -k main -v`
Expected: FAIL with `AttributeError: module 'dream.dream_extract' has no attribute 'main'`

- [ ] **Step 3: Write the implementation**

Add to `dream/dream_extract.py`:

```python
import argparse
import sys
import time
from datetime import datetime, timezone

from dream.paths import INPUT_DIR, INPUT_RETENTION_DAYS, TRANSCRIPT_ROOT


def _prune_old_input() -> None:
    cutoff = time.time() - INPUT_RETENTION_DAYS * 86400
    if not INPUT_DIR.exists():
        return
    for old in INPUT_DIR.glob("*.jsonl"):
        if old.stat().st_mtime < cutoff:
            old.unlink()


def main(argv: list[str] | None = None) -> int:
    ap = argparse.ArgumentParser(description="Extract signal-bearing turns from transcripts.")
    ap.add_argument("--since-hours", type=float, default=24.0)
    args = ap.parse_args(argv)

    cutoff = time.time() - args.since_hours * 3600
    files = [p for p in TRANSCRIPT_ROOT.glob("*.jsonl") if p.stat().st_mtime >= cutoff]

    records: list[dict] = []
    stats: list[SessionStats] = []
    for path in sorted(files):
        recs, st = extract_session(path)
        records.extend(recs)
        stats.append(st)

    try:
        assert_corpus_plausible(stats, files_seen=len(files))
    except CorpusImplausible as exc:
        # No partial artifact. A report written from an untrusted corpus is
        # worse than no report, because it reads as "memory looks current".
        print(f"dream-extract: GUARD TRIPPED: {exc}", file=sys.stderr)
        return 2

    INPUT_DIR.mkdir(parents=True, exist_ok=True)
    day = datetime.now(timezone.utc).strftime("%Y-%m-%d")
    out = INPUT_DIR / f"{day}.jsonl"
    out.write_text("".join(json.dumps(r, ensure_ascii=False) + "\n" for r in records))
    _prune_old_input()

    print(json.dumps({
        "sessions": len(stats),
        "human_turns": sum(s.human_turns for s in stats),
        "records": len(records),
        "path": str(out),
    }))
    return 0


if __name__ == "__main__":
    raise SystemExit(main())
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd /home/james/ATLAS && python3 -m pytest dream/tests/ -v`
Expected: 27 passed

- [ ] **Step 5: Smoke against the real corpus (read-only)**

Run: `cd /home/james/ATLAS && DREAM_ROOT=/tmp/dream-smoke python3 -m dream.dream_extract --since-hours 24`
Expected: exit 0 and a JSON line whose `human_turns` is roughly 150-250 and `records` is well under 2000. If `human_turns` is 0, the guard should have tripped -- investigate rather than proceed.

- [ ] **Step 6: Commit and open PR 1**

```bash
git add dream/dream_extract.py dream/tests/test_extract_guard.py
git commit -m "feat(dream): extractor CLI, refuses to write output on guard trip"
```

Then push a branch and open PR 1 titled `feat(dream): transcript extractor and the D-1 fail-loud guard`.

---

# PR 2 -- The consolidation skill

### Task 7: Skill definition and report schema

**Files:**
- Create: `.claude/skills/dream/SKILL.md`
- Create: `dream/tests/test_report_schema.py`, `dream/report_schema.py`

**Interfaces:**
- Consumes: `dream.paths.REPORT_ITEM_CAP`.
- Produces: `parse_report(text: str) -> list[Finding]` where `Finding` is a dataclass with `number: int`, `ftype: str`, `target: str`, `severity: str`, `claim: str`, `evidence: list[str]`, `anchor: str`, `replacement: str`, `recheck: str`. Also `VALID_TYPES: frozenset[str]`.

- [ ] **Step 1: Write the failing test**

```python
# dream/tests/test_report_schema.py
import pytest
from dream.report_schema import VALID_TYPES, parse_report

SAMPLE = """# Dream Report 2026-08-08

## 1 - CORRECTION - STATE.md - high
Claim: autofix-watcher does not poll main's tip
Evidence: 89a0eb74 turn 42 2026-08-07T14:02Z > "the old note here was wrong"
Anchor: nothing polls main's tip
Replacement: nothing polls main's tip; it fires only on human merge
Recheck: none

## 2 - INDEX - memory/MEMORY.md - low
Claim: project_health_is_tempo_not_loki.md has no index line
Evidence: deterministic check: file present, absent from index
Anchor: - [UPS monitoring](reference_ups_monitoring.md)
Replacement: - [UPS monitoring](reference_ups_monitoring.md)\\n- [Health is Tempo](project_health_is_tempo_not_loki.md)
Recheck: file_exists:project_health_is_tempo_not_loki.md
"""


def test_parses_all_findings():
    findings = parse_report(SAMPLE)
    assert [f.number for f in findings] == [1, 2]
    assert findings[0].ftype == "CORRECTION"
    assert findings[0].target == "STATE.md"
    assert findings[0].severity == "high"


def test_evidence_is_captured():
    f = parse_report(SAMPLE)[0]
    assert len(f.evidence) == 1
    assert "89a0eb74" in f.evidence[0]


def test_anchor_and_replacement_captured():
    f = parse_report(SAMPLE)[0]
    assert f.anchor == "nothing polls main's tip"
    assert f.replacement.endswith("human merge")


def test_recheck_directive_captured():
    assert parse_report(SAMPLE)[1].recheck == "file_exists:project_health_is_tempo_not_loki.md"


def test_unknown_type_rejected():
    bad = SAMPLE.replace("CORRECTION", "INVENTED")
    with pytest.raises(ValueError, match="INVENTED"):
        parse_report(bad)


def test_finding_without_evidence_rejected():
    bad = "\n".join(l for l in SAMPLE.splitlines() if not l.startswith("Evidence:"))
    with pytest.raises(ValueError, match="evidence"):
        parse_report(bad)


def test_valid_types_match_spec():
    assert VALID_TYPES == frozenset({
        "CORRECTION", "STALE", "CONFLICT", "DUPLICATE",
        "NEW", "PATTERN", "INDEX", "GRADUATE",
    })
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd /home/james/ATLAS && python3 -m pytest dream/tests/test_report_schema.py -v`
Expected: FAIL with `ModuleNotFoundError: No module named 'dream.report_schema'`

- [ ] **Step 3: Write the implementation**

```python
# dream/report_schema.py
"""Parse DREAM_REPORT.md into findings the apply path can act on.

The report is markdown a human reads and a parser consumes. Parsing is strict:
a finding without evidence is rejected at parse time rather than surfaced for
approval, because "no quote, no proposal" is only enforceable if the code
enforces it.
"""
from __future__ import annotations

import re
from dataclasses import dataclass, field

VALID_TYPES = frozenset({
    "CORRECTION", "STALE", "CONFLICT", "DUPLICATE",
    "NEW", "PATTERN", "INDEX", "GRADUATE",
})

_HEADER = re.compile(r"^##\s+(\d+)\s+-\s+(\w+)\s+-\s+(\S+)\s+-\s+(\w+)\s*$")
_FIELD = re.compile(r"^(Claim|Evidence|Anchor|Replacement|Recheck):\s*(.*)$")


@dataclass
class Finding:
    number: int
    ftype: str
    target: str
    severity: str
    claim: str = ""
    evidence: list[str] = field(default_factory=list)
    anchor: str = ""
    replacement: str = ""
    recheck: str = "none"


def parse_report(text: str) -> list[Finding]:
    findings: list[Finding] = []
    current: Finding | None = None

    for line in text.splitlines():
        header = _HEADER.match(line)
        if header:
            num, ftype, target, sev = header.groups()
            if ftype not in VALID_TYPES:
                raise ValueError(f"finding {num}: unknown type {ftype}")
            current = Finding(int(num), ftype, target, sev)
            findings.append(current)
            continue
        if current is None:
            continue
        m = _FIELD.match(line)
        if not m:
            continue
        key, value = m.groups()
        if key == "Evidence":
            current.evidence.append(value)
        elif key == "Claim":
            current.claim = value
        elif key == "Anchor":
            current.anchor = value
        elif key == "Replacement":
            current.replacement = value.replace("\\n", "\n")
        elif key == "Recheck":
            current.recheck = value

    for f in findings:
        if not f.evidence:
            raise ValueError(f"finding {f.number}: no evidence cited -- refusing")
    return findings
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd /home/james/ATLAS && python3 -m pytest dream/tests/test_report_schema.py -v`
Expected: 7 passed

- [ ] **Step 5: Write the skill**

Create `.claude/skills/dream/SKILL.md` with frontmatter `name: dream` and a description naming both modes. Body must state, in plain ASCII:

- corpus is `DREAM_ROOT/input/<today>.jsonl`, never the raw transcripts
- read every memory surface plus `rejected.md` before proposing
- the eight finding types and the per-type evidence requirement (CORRECTION cites both sides; PATTERN needs 2+ occurrences across 2+ distinct session ids; STALE carries a `Recheck:` directive rather than a quote)
- the four admission gates: durable, non-derivable, actionable, not already covered
- cap at 10 detailed items; count and name held-back findings in one line
- emit exactly the `## N - TYPE - target - severity` block format above
- never write anything except `DREAM_REPORT.md`

- [ ] **Step 6: Commit and open PR 2**

```bash
git add dream/report_schema.py dream/tests/test_report_schema.py .claude/skills/dream/SKILL.md
git commit -m "feat(dream): consolidation skill and strict report schema"
```

---

# PR 3 -- The guarded apply path

### Task 8: D-3 write-path allowlist

**Files:**
- Create: `dream/dream_apply.py`
- Test: `dream/tests/test_apply_safety.py`

**Interfaces:**
- Consumes: `dream.paths.MEMORY_ROOT`, `STATE_MD`.
- Produces: `class WriteRefused(Exception)`; `assert_allowed_target(path: Path) -> Path` returning the resolved path or raising.

- [ ] **Step 1: Write the failing test**

```python
# dream/tests/test_apply_safety.py
import importlib, pytest
from pathlib import Path


@pytest.fixture
def apply_mod(monkeypatch, tmp_path):
    mem = tmp_path / "memory"; mem.mkdir()
    state = tmp_path / "STATE.md"; state.write_text("# state\nnothing polls main's tip\n")
    monkeypatch.setenv("DREAM_MEMORY_ROOT", str(mem))
    monkeypatch.setenv("DREAM_STATE_MD", str(state))
    monkeypatch.setenv("DREAM_ROOT", str(tmp_path / "dream"))
    import dream.paths, dream.dream_apply as da
    importlib.reload(dream.paths); importlib.reload(da)
    return da, mem, state


def test_memory_file_allowed(apply_mod):
    da, mem, _ = apply_mod
    (mem / "a.md").write_text("x")
    assert da.assert_allowed_target(mem / "a.md") == (mem / "a.md").resolve()


def test_state_md_allowed(apply_mod):
    da, _, state = apply_mod
    assert da.assert_allowed_target(state) == state.resolve()


def test_claude_md_refused(apply_mod):
    """The guard that stops a confidently-wrong proposal reaching policy."""
    da, _, _ = apply_mod
    with pytest.raises(da.WriteRefused, match="not an allowed target"):
        da.assert_allowed_target(Path("/home/james/ATLAS/CLAUDE.md"))


def test_traversal_out_of_memory_refused(apply_mod):
    da, mem, _ = apply_mod
    with pytest.raises(da.WriteRefused):
        da.assert_allowed_target(mem / ".." / ".." / "etc" / "passwd")


def test_symlink_escaping_memory_refused(apply_mod, tmp_path):
    """resolve() before the containment check, or a symlink is a free bypass."""
    da, mem, _ = apply_mod
    outside = tmp_path / "outside.md"; outside.write_text("x")
    link = mem / "sneaky.md"; link.symlink_to(outside)
    with pytest.raises(da.WriteRefused):
        da.assert_allowed_target(link)
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd /home/james/ATLAS && python3 -m pytest dream/tests/test_apply_safety.py -v`
Expected: FAIL with `ModuleNotFoundError: No module named 'dream.dream_apply'`

- [ ] **Step 3: Write the implementation**

```python
# dream/dream_apply.py
"""Apply approved dream findings behind guards. Deterministic given approvals.

No model runs here. Once a human has approved item N, applying it involves no
judgment -- which is why the riskiest write in the system (STATE.md, which has
had no git backup since #922) has no model in the loop.
"""
from __future__ import annotations

from pathlib import Path

from dream.paths import MEMORY_ROOT, STATE_MD


class WriteRefused(Exception):
    """A write was refused at the boundary. Never downgrade this to a warning."""


def assert_allowed_target(path: Path) -> Path:
    """INTENT(D-3): enumerate what is PERMITTED, never what is forbidden.

    PRECOND: target resolves inside MEMORY_ROOT, or is exactly STATE_MD.

    #935 spent three rounds proving the opposite approach fails: each round
    claimed writes were closed, each was falsified by probing a shape the
    denylist had not enumerated. Two allowed paths is a surface small enough to
    verify exhaustively.

    resolve() runs BEFORE the containment check so a symlink planted inside
    MEMORY_ROOT cannot point anywhere it likes.
    """
    resolved = Path(path).resolve()
    if resolved == STATE_MD.resolve():
        return resolved
    memory_root = MEMORY_ROOT.resolve()
    if resolved.is_relative_to(memory_root) and resolved != memory_root:
        return resolved
    raise WriteRefused(
        f"{resolved} is not an allowed target; allowed are {memory_root}/** and {STATE_MD}"
    )
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd /home/james/ATLAS && python3 -m pytest dream/tests/test_apply_safety.py -v`
Expected: 5 passed

- [ ] **Step 5: Commit**

```bash
git add dream/dream_apply.py dream/tests/test_apply_safety.py
git commit -m "feat(dream): D-3 write-path allowlist, refuses everything else"
```

---

### Task 9: D-2 snapshot before write

**Files:**
- Modify: `dream/dream_apply.py`
- Test: `dream/tests/test_apply_safety.py`

**Interfaces:**
- Consumes: `dream.paths.SNAPSHOT_DIR`, `SNAPSHOT_GENERATIONS`, `MEMORY_ROOT`, `STATE_MD`.
- Produces: `snapshot_or_refuse(stamp: str) -> Path` returning the created snapshot directory, raising `WriteRefused` if it cannot be verified.

- [ ] **Step 1: Write the failing test**

Append to `dream/tests/test_apply_safety.py`:

```python
def test_snapshot_captures_state_and_memory(apply_mod):
    da, mem, state = apply_mod
    (mem / "a.md").write_text("memory content")
    snap = da.snapshot_or_refuse("20260808T030000Z")
    assert (snap / "STATE.md").read_text() == state.read_text()
    assert (snap / "memory.tar").exists()


def test_snapshot_refuses_when_state_unreadable(apply_mod, monkeypatch):
    """STATE.md has no git backup. If the only backup cannot be taken, the
    write must not happen."""
    da, _, state = apply_mod
    state.unlink()
    with pytest.raises(da.WriteRefused, match="snapshot"):
        da.snapshot_or_refuse("20260808T030000Z")


def test_snapshot_prunes_to_generation_cap(apply_mod):
    da, mem, _ = apply_mod
    from dream.paths import SNAPSHOT_GENERATIONS
    for i in range(SNAPSHOT_GENERATIONS + 5):
        da.snapshot_or_refuse(f"2026080{i//10}T0{i%10}0000Z")
    from dream.paths import SNAPSHOT_DIR
    assert len(list(SNAPSHOT_DIR.iterdir())) == SNAPSHOT_GENERATIONS
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd /home/james/ATLAS && python3 -m pytest dream/tests/test_apply_safety.py -k snapshot -v`
Expected: FAIL with `AttributeError: module 'dream.dream_apply' has no attribute 'snapshot_or_refuse'`

- [ ] **Step 3: Write the implementation**

Add to `dream/dream_apply.py`:

```python
import shutil
import tarfile

from dream.paths import SNAPSHOT_DIR, SNAPSHOT_GENERATIONS


def snapshot_or_refuse(stamp: str) -> Path:
    """INTENT(D-2): STATE.md has had no git backup since #922, so this snapshot
    is its only recovery path.

    PRECOND: a verified snapshot exists before the first write of any run.

    Refuses rather than proceeding unbacked. A write we cannot undo is worse
    than a change we did not make.
    """
    dest = SNAPSHOT_DIR / stamp
    try:
        dest.mkdir(parents=True, exist_ok=True)
        shutil.copy2(STATE_MD, dest / "STATE.md")
        with tarfile.open(dest / "memory.tar", "w") as tar:
            tar.add(MEMORY_ROOT, arcname="memory")
    except OSError as exc:
        raise WriteRefused(f"snapshot failed, refusing to write: {exc}") from exc

    if not (dest / "STATE.md").exists() or not (dest / "memory.tar").exists():
        raise WriteRefused("snapshot incomplete, refusing to write")

    generations = sorted(SNAPSHOT_DIR.iterdir())
    for old in generations[:-SNAPSHOT_GENERATIONS]:
        shutil.rmtree(old, ignore_errors=True)
    return dest
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd /home/james/ATLAS && python3 -m pytest dream/tests/test_apply_safety.py -v`
Expected: 8 passed

- [ ] **Step 5: Commit**

```bash
git add dream/dream_apply.py dream/tests/test_apply_safety.py
git commit -m "feat(dream): D-2 snapshot before write, refuse if unbacked"
```

---

### Task 10: D-4 content-anchored edits

**Files:**
- Modify: `dream/dream_apply.py`
- Test: `dream/tests/test_apply_safety.py`

**Interfaces:**
- Consumes: `assert_allowed_target`, `report_schema.Finding`.
- Produces: `class AnchorMissing(Exception)`; `apply_finding(finding: Finding, target: Path) -> None` which raises `AnchorMissing` and leaves the file byte-identical when the anchor is absent or ambiguous.

- [ ] **Step 1: Write the failing test**

Append to `dream/tests/test_apply_safety.py`:

```python
from dream.report_schema import Finding


def _finding(anchor, replacement, target="STATE.md"):
    f = Finding(1, "CORRECTION", target, "high")
    f.anchor, f.replacement = anchor, replacement
    f.evidence = ["s1 turn 1"]
    return f


def test_anchor_present_applies(apply_mod):
    da, _, state = apply_mod
    da.apply_finding(_finding("nothing polls main's tip", "it fires on human merge"), state)
    assert "it fires on human merge" in state.read_text()


def test_anchor_absent_refuses_and_leaves_file_identical(apply_mod):
    """The report is written at 03:00 and reviewed hours later. If a morning
    session rewrote the section, the edit must refuse, never fuzzy-match."""
    da, _, state = apply_mod
    before = state.read_bytes()
    with pytest.raises(da.AnchorMissing):
        da.apply_finding(_finding("text that is not there", "whatever"), state)
    assert state.read_bytes() == before


def test_ambiguous_anchor_refuses(apply_mod):
    """Two matches means we cannot know which one the proposal meant."""
    da, _, state = apply_mod
    state.write_text("dup line\ndup line\n")
    before = state.read_bytes()
    with pytest.raises(da.AnchorMissing, match="2 times"):
        da.apply_finding(_finding("dup line", "fixed"), state)
    assert state.read_bytes() == before


def test_applying_twice_is_a_noop_second_time(apply_mod):
    da, _, state = apply_mod
    f = _finding("nothing polls main's tip", "it fires on human merge")
    da.apply_finding(f, state)
    with pytest.raises(da.AnchorMissing):
        da.apply_finding(f, state)
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd /home/james/ATLAS && python3 -m pytest dream/tests/test_apply_safety.py -k anchor -v`
Expected: FAIL with `AttributeError: module 'dream.dream_apply' has no attribute 'apply_finding'`

- [ ] **Step 3: Write the implementation**

Add to `dream/dream_apply.py`:

```python
from dream.report_schema import Finding


class AnchorMissing(Exception):
    """The text a finding meant to change is no longer present verbatim."""


def apply_finding(finding: Finding, target: Path) -> None:
    """INTENT(D-4): the file moves between report time and review time, so
    position is not identity -- only content is.

    PRECOND: anchor text present exactly once, else refuse and re-queue.

    Never fuzzy-match and never fall back to nearest position. Same discipline
    as the tree-hash push marker: key on content, never on position.
    """
    resolved = assert_allowed_target(target)
    body = resolved.read_text()
    occurrences = body.count(finding.anchor)

    if occurrences == 0:
        raise AnchorMissing(
            f"finding {finding.number}: anchor absent from {resolved}; re-queued"
        )
    if occurrences > 1:
        raise AnchorMissing(
            f"finding {finding.number}: anchor appears {occurrences} times in "
            f"{resolved}; ambiguous, re-queued"
        )

    resolved.write_text(body.replace(finding.anchor, finding.replacement, 1))
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd /home/james/ATLAS && python3 -m pytest dream/tests/test_apply_safety.py -v`
Expected: 12 passed

- [ ] **Step 5: Commit**

```bash
git add dream/dream_apply.py dream/tests/test_apply_safety.py
git commit -m "feat(dream): D-4 content-anchored edits, refuse on drift or ambiguity"
```

---

### Task 11: D-5 auto-apply scope, D-6 provenance, and the ledger

**Files:**
- Modify: `dream/dream_apply.py`
- Test: `dream/tests/test_apply_semantics.py`

**Interfaces:**
- Consumes: `apply_finding`, `dream.paths.APPLIED`, `REJECTED`.
- Produces: `auto_apply_eligible(finding: Finding) -> bool`; `stamp_provenance(text: str, finding: Finding, run_id: str) -> str`; `record_applied(finding: Finding, run_id: str) -> None`; `record_rejected(finding: Finding, reason: str) -> None`; `fingerprint(finding: Finding) -> str`.

- [ ] **Step 1: Write the failing test**

```python
# dream/tests/test_apply_semantics.py
import importlib, pytest
from dream.report_schema import Finding


@pytest.fixture
def apply_mod(monkeypatch, tmp_path):
    mem = tmp_path / "memory"; mem.mkdir()
    (tmp_path / "STATE.md").write_text("# state\n")
    monkeypatch.setenv("DREAM_MEMORY_ROOT", str(mem))
    monkeypatch.setenv("DREAM_STATE_MD", str(tmp_path / "STATE.md"))
    monkeypatch.setenv("DREAM_ROOT", str(tmp_path / "dream"))
    import dream.paths, dream.dream_apply as da
    importlib.reload(dream.paths); importlib.reload(da)
    (tmp_path / "dream").mkdir(parents=True, exist_ok=True)
    return da, mem


def _f(ftype, recheck="none", number=1):
    f = Finding(number, ftype, "memory/MEMORY.md", "low")
    f.evidence = ["deterministic check"]
    f.recheck = recheck
    return f


def test_only_index_findings_auto_apply(apply_mod):
    da, _ = apply_mod
    assert da.auto_apply_eligible(_f("INDEX")) is True
    for t in ("CORRECTION", "STALE", "NEW", "PATTERN", "DUPLICATE", "CONFLICT", "GRADUATE"):
        assert da.auto_apply_eligible(_f(t)) is False, t


def test_index_finding_with_recheck_is_not_auto(apply_mod):
    """A recheck directive means the claim is time-sensitive, so it needs the
    gate even though its type is INDEX."""
    da, _ = apply_mod
    assert da.auto_apply_eligible(_f("INDEX", recheck="file_exists:x.md")) is False


def test_provenance_stamp_carries_run_and_evidence(apply_mod):
    da, _ = apply_mod
    out = da.stamp_provenance("- [Thing](thing.md) - hook", _f("NEW"), "20260808T030000Z")
    assert "20260808T030000Z" in out
    assert "deterministic check" in out
    assert out.startswith("- [Thing](thing.md) - hook")


def test_applied_ledger_appends(apply_mod):
    da, _ = apply_mod
    from dream.paths import APPLIED
    da.record_applied(_f("INDEX", number=1), "run1")
    da.record_applied(_f("NEW", number=2), "run1")
    assert APPLIED.read_text().count("run1") == 2


def test_rejected_fingerprint_is_stable_across_rewording(apply_mod):
    da, _ = apply_mod
    a = _f("CORRECTION"); a.claim = "the watcher does not poll main"
    b = _f("CORRECTION"); b.claim = "The watcher never polls main's tip!"
    assert da.fingerprint(a) == da.fingerprint(b)


def test_rejected_fingerprint_differs_across_type(apply_mod):
    da, _ = apply_mod
    a = _f("CORRECTION"); a.claim = "same words"
    b = _f("STALE"); b.claim = "same words"
    assert da.fingerprint(a) != da.fingerprint(b)
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd /home/james/ATLAS && python3 -m pytest dream/tests/test_apply_semantics.py -v`
Expected: FAIL with `AttributeError: module 'dream.dream_apply' has no attribute 'auto_apply_eligible'`

- [ ] **Step 3: Write the implementation**

Add to `dream/dream_apply.py`:

```python
import hashlib
import re as _re
from datetime import datetime, timezone

from dream.paths import APPLIED, REJECTED


def auto_apply_eligible(finding: Finding) -> bool:
    """INTENT(D-5): mechanical index repair is not judgment, but invisible
    mutation is how memory drifts.

    PRECOND: type is INDEX and the claim carries no time-sensitive recheck.

    Runs inside /dream review, never from the timer. "Auto" means it skips the
    approve prompt, never that it happens unattended -- the timer-never-writes
    invariant holds without exception. Auto-applied items still appear in the
    report marked as applied.
    """
    return finding.ftype == "INDEX" and finding.recheck.strip().lower() == "none"


def stamp_provenance(text: str, finding: Finding, run_id: str) -> str:
    """INTENT(D-6): a wrong dream-authored memory must be traceable to its
    evidence, not indistinguishable from a hand-written fact.

    Without this, the next pass reads its own error as ground truth and
    compounds it.
    """
    cite = finding.evidence[0] if finding.evidence else "no-evidence"
    return f"{text} <!-- dream:{run_id} ev:{cite} -->"


def fingerprint(finding: Finding) -> str:
    """Identity of a CLAIM, not of its wording.

    Fingerprinting exact text lets a rejected proposal return tomorrow with a
    reworded claim. Normalizing to type + target + content words makes a
    rejection stick.
    """
    words = sorted(set(_re.findall(r"[a-z]{4,}", finding.claim.lower())))
    key = f"{finding.ftype}|{finding.target}|{' '.join(words)}"
    return hashlib.sha1(key.encode()).hexdigest()[:16]


def _now() -> str:
    return datetime.now(timezone.utc).strftime("%Y-%m-%dT%H:%M:%S.0000000Z")


def record_applied(finding: Finding, run_id: str) -> None:
    APPLIED.parent.mkdir(parents=True, exist_ok=True)
    with APPLIED.open("a") as fh:
        fh.write(
            f"{_now()} run={run_id} {finding.ftype} {finding.target} "
            f"fp={fingerprint(finding)} claim={finding.claim}\n"
        )


def record_rejected(finding: Finding, reason: str) -> None:
    REJECTED.parent.mkdir(parents=True, exist_ok=True)
    with REJECTED.open("a") as fh:
        fh.write(
            f"{_now()} {finding.ftype} {finding.target} fp={fingerprint(finding)} "
            f"reason={reason} claim={finding.claim}\n"
        )
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd /home/james/ATLAS && python3 -m pytest dream/tests/ -v`
Expected: 40 passed

- [ ] **Step 5: Run the mutation pass on every guard**

For each of `assert_allowed_target`'s raise, `snapshot_or_refuse`'s raise, `apply_finding`'s two raises, and `auto_apply_eligible`'s type check: comment it out, run the full suite, confirm at least one test goes RED, restore.

Run: `cd /home/james/ATLAS && python3 -m pytest dream/tests/ -q`
Expected: every disabled guard produces at least one failure. Any guard whose removal leaves the suite green has no real test -- write one before proceeding. #935 shipped a battery where deleting a guard line left everything green.

- [ ] **Step 6: Add the review entrypoint to the skill**

Extend `.claude/skills/dream/SKILL.md` with the `review` mode: acquire `LOCKFILE`, parse the report, snapshot once, auto-apply eligible INDEX findings and mark them applied in the report, prompt for each remaining finding, re-run any `Recheck:` directive before applying and refuse if the result flipped, record applied and rejected, and report anchor refusals explicitly rather than swallowing them.

- [ ] **Step 7: Commit and open PR 3**

```bash
git add dream/dream_apply.py dream/tests/test_apply_semantics.py .claude/skills/dream/SKILL.md
git commit -m "feat(dream): D-5 auto-apply scope, D-6 provenance, ledger and rejection memory"
```

---

# PR 4 -- Schedule, notification, and the pending notice

### Task 12: Timer entrypoint script and ntfy publish

**Files:**
- Create: `deployment/artifacts/scripts/atlas-dream-run.sh`
- Create: `deployment/artifacts/atlas-dream.service`, `deployment/artifacts/atlas-dream.timer`

**Interfaces:**
- Consumes: `python3 -m dream.dream_extract` exit codes (0 ok, 2 guard trip) and its stdout JSON summary.
- Produces: a systemd unit `atlas-dream.service` and timer `atlas-dream.timer`.

- [ ] **Step 1: Write the entrypoint**

```bash
#!/usr/bin/env bash
# Nightly dream run: extract, consolidate, notify.
#
# Runs extraction and consolidation ONLY. It never applies anything -- the
# timer-never-writes invariant is what keeps an unattended model out of
# STATE.md, which has had no git backup since #922.
#
# EXIT
#   0  ran clean (with or without proposals)
#   2  extraction guard tripped; NO report written, high-priority ntfy sent
set -uo pipefail

REPO=/home/james/ATLAS
DREAM_ROOT="${DREAM_ROOT:-$HOME/.claude/projects/-home-james-ATLAS/dream}"
REPORT="${DREAM_ROOT}/DREAM_REPORT.md"
ENV_FILE="${ATLAS_NOTIFY_ENV:-/opt/ai-inference/ntfy-notify.env}"
TOPIC=atlas-claude-ask

# shellcheck source=/dev/null
[[ -r "${ENV_FILE}" ]] && . "${ENV_FILE}"

publish() {  # $1 title, $2 body, $3 priority
  curl -sS --max-time 20 --retry 2 \
    -u "${NTFY_USER}:${NTFY_PASSWORD}" \
    -H "Title: $1" -H "Priority: $3" \
    -d "$2" "${NTFY_ENDPOINT}/${TOPIC}" >/dev/null
}

cd "${REPO}" || exit 1

if ! SUMMARY="$(python3 -m dream.dream_extract --since-hours 24 2>&1)"; then
  # A dead extractor must never look like a quiet night. This is the one case
  # that pages rather than informs.
  publish "ATLAS Dream FAILED" "extraction guard tripped, no report written
${SUMMARY}" high
  exit 2
fi

TURNS=$(printf '%s' "${SUMMARY}" | python3 -c 'import json,sys; print(json.load(sys.stdin)["human_turns"])')
SESSIONS=$(printf '%s' "${SUMMARY}" | python3 -c 'import json,sys; print(json.load(sys.stdin)["sessions"])')

BEFORE_FP=""
[[ -f "${REPORT}" ]] && BEFORE_FP="$(sha1sum "${REPORT}" | cut -d' ' -f1)"

claude --print "/dream consolidate" >/dev/null 2>&1

AFTER_FP=""
[[ -f "${REPORT}" ]] && AFTER_FP="$(sha1sum "${REPORT}" | cut -d' ' -f1)"

# Publish only on NEW proposals. Re-pinging a standing backlog nightly is how a
# channel gets muted, and a muted channel is the same as no channel.
if [[ -n "${AFTER_FP}" && "${AFTER_FP}" != "${BEFORE_FP}" ]]; then
  COUNT=$(grep -c '^## [0-9]' "${REPORT}" 2>/dev/null || echo 0)
  TOP=$(grep -m1 '^## [0-9]' "${REPORT}" 2>/dev/null || echo "see report")
  # The corpus line is load-bearing: it is what separates "0 proposals from 187
  # turns" (a quiet day) from "0 proposals from 0 turns" (a dead extractor).
  publish "ATLAS Dream - ${COUNT} new proposals" "${TOP}
corpus: ${TURNS} turns / ${SESSIONS} sessions
Review: /dream review" default
fi
exit 0
```

- [ ] **Step 2: Write the units**

```ini
# deployment/artifacts/atlas-dream.service
[Unit]
Description=ATLAS Dream - nightly out-of-band memory consolidation
After=network-online.target
Wants=network-online.target
# Reuses the #881 notifier rather than minting a second alert path.
OnFailure=atlas-unit-failure-notify@%n.service

[Service]
Type=oneshot
ExecStart=/opt/ai-inference/scripts/atlas-dream-run.sh
User=james
Group=james
WorkingDirectory=/home/james/ATLAS

Environment=HOME=/home/james
Environment=PATH=/usr/local/bin:/usr/bin:/bin:/home/james/.local/bin
EnvironmentFile=/etc/autofix/claude.env

StandardOutput=journal
StandardError=journal
SyslogIdentifier=atlas-dream

NoNewPrivileges=true
ProtectSystem=strict
ProtectHome=read-only
# Deliberately NOT /home/james/ATLAS: the nightly run must never write to the
# repo. It needs the transcripts and its own state directory, nothing else.
ReadWritePaths=/home/james/.claude /tmp
PrivateTmp=false

MemoryMax=2G
CPUQuota=200%
```

```ini
# deployment/artifacts/atlas-dream.timer
[Unit]
Description=ATLAS Dream Timer - 03:00 nightly

[Timer]
OnCalendar=*-*-* 03:00:00
AccuracySec=1m
Persistent=true

[Install]
WantedBy=timers.target
```

- [ ] **Step 3: Verify the units parse before deploying**

Run: `systemd-analyze verify deployment/artifacts/atlas-dream.service deployment/artifacts/atlas-dream.timer`
Expected: no output (clean). Resolve any warning before continuing.

- [ ] **Step 4: Dry-run the entrypoint without publishing**

Run: `cd /home/james/ATLAS && DREAM_ROOT=/tmp/dream-smoke ATLAS_NOTIFY_ENV=/dev/null bash deployment/artifacts/scripts/atlas-dream-run.sh; echo "exit=$?"`
Expected: `exit=0`. `NTFY_*` are unset so `publish` fails harmlessly under `set -uo pipefail` without `-e`; confirm the extraction step still wrote `/tmp/dream-smoke/input/`.

- [ ] **Step 5: Commit**

```bash
git add deployment/artifacts/scripts/atlas-dream-run.sh deployment/artifacts/atlas-dream.service deployment/artifacts/atlas-dream.timer
git commit -m "feat(dream): nightly timer, entrypoint, and ntfy contract"
```

---

### Task 13: Ansible install and the SessionStart pending notice

**Files:**
- Modify: `deployment/ansible/playbooks/deploy.yml` (new tasks, tag `dream`)
- Create: `.claude/hooks/dream-pending-notice.sh`
- Modify: `.claude/settings.json` (add `SessionStart` hook)

**Interfaces:**
- Consumes: the units and script from Task 12.
- Produces: ansible tag `dream`; a SessionStart hook emitting `hookSpecificOutput.additionalContext`.

- [ ] **Step 1: Write the hook**

```bash
#!/usr/bin/env bash
# SessionStart: one line when DREAM_REPORT.md holds unreviewed findings.
#
# NON-BLOCKING -- always exits 0. ntfy is the push at 03:00; this is the
# pull-side reminder at the moment the operator can actually act on it. A
# pending report nobody opens is worth nothing.
set -uo pipefail

REPORT="${DREAM_ROOT:-$HOME/.claude/projects/-home-james-ATLAS/dream}/DREAM_REPORT.md"
[[ -r "${REPORT}" ]] || exit 0

count="$(grep -c '^## [0-9]' "${REPORT}" 2>/dev/null || echo 0)"
[[ "${count}" -gt 0 ]] || exit 0

top="$(grep -m1 '^## [0-9]' "${REPORT}" 2>/dev/null | sed 's/^## //')"
printf '{"hookSpecificOutput":{"hookEventName":"SessionStart","additionalContext":"DREAM: %s unreviewed finding(s). Top: %s. Run /dream review."}}\n' \
  "${count}" "${top//\"/}"
exit 0
```

- [ ] **Step 2: Test the hook both ways**

Run: `DREAM_ROOT=/tmp/dream-empty bash .claude/hooks/dream-pending-notice.sh; echo "exit=$?"`
Expected: no output, `exit=0`.

Run: `mkdir -p /tmp/dream-x && printf '## 1 - CORRECTION - STATE.md - high\n' > /tmp/dream-x/DREAM_REPORT.md && DREAM_ROOT=/tmp/dream-x bash .claude/hooks/dream-pending-notice.sh`
Expected: a single JSON line containing `DREAM: 1 unreviewed finding(s)`.

A hook that emits on the empty case is noise; one that stays silent on the populated case is useless. Both directions must be checked.

- [ ] **Step 3: Wire the hook into settings.json**

Add a `SessionStart` key alongside the existing `PreToolUse` and `PostToolUse` arrays:

```json
"SessionStart": [
  {
    "hooks": [
      {
        "type": "command",
        "command": "$CLAUDE_PROJECT_DIR/.claude/hooks/dream-pending-notice.sh"
      }
    ]
  }
]
```

- [ ] **Step 4: Add the ansible tasks**

Insert after the autofix-runner timer block (around `deploy.yml:2167`), tagged `[dream]` so it is never swept into a full-stack restart:

```yaml
    - name: Deploy dream run script
      copy:
        src: "{{ atlas_repo_path }}/deployment/artifacts/scripts/atlas-dream-run.sh"
        dest: "{{ deployment_base }}/scripts/atlas-dream-run.sh"
        owner: james
        group: james
        mode: '0755'
      tags: [dream]

    - name: Deploy atlas-dream systemd service
      copy:
        src: "{{ atlas_repo_path }}/deployment/artifacts/atlas-dream.service"
        dest: /etc/systemd/system/atlas-dream.service
        owner: root
        group: root
        mode: '0644'
      tags: [dream]

    - name: Deploy atlas-dream systemd timer
      copy:
        src: "{{ atlas_repo_path }}/deployment/artifacts/atlas-dream.timer"
        dest: /etc/systemd/system/atlas-dream.timer
        owner: root
        group: root
        mode: '0644'
      tags: [dream]

    - name: Enable and start atlas-dream timer
      systemd:
        name: atlas-dream.timer
        enabled: true
        daemon_reload: true
        state: started
      tags: [dream]
```

- [ ] **Step 5: Verify tag selection before running anything**

Run: `cd deployment/ansible && ansible-playbook playbooks/deploy.yml --tags dream --skip-tags always --list-tasks`
Expected: exactly the four tasks above. If any other task appears, the tag is leaking -- fix before executing. A bare `--tags dream` without `--skip-tags always` triggers a full-stack restart including a ~4 minute vLLM reload; never run that form.

- [ ] **Step 6: Deploy and verify**

Run: `cd deployment/ansible && ansible-playbook playbooks/deploy.yml --tags dream --skip-tags always`
Then: `systemctl status atlas-dream.timer && systemctl list-timers atlas-dream.timer`
Expected: timer active, next elapse tomorrow at 03:00.

- [ ] **Step 7: Force one real run and confirm the whole chain**

Run: `sudo systemctl start atlas-dream.service && journalctl -u atlas-dream -n 40 --no-pager`
Expected: extraction summary in the journal, exit 0, and either a report with findings plus one ntfy message, or no report and no message. Confirm on your phone that the ntfy arrived if findings were produced.

- [ ] **Step 8: Commit and open PR 4**

```bash
git add .claude/hooks/dream-pending-notice.sh .claude/settings.json deployment/ansible/playbooks/deploy.yml
git commit -m "feat(dream): ansible install, timer enablement, SessionStart pending notice"
```

---

## After M1 lands

Run for two weeks, then check the spec's section 12 criteria before considering M2:

- accept rate at or above 50 percent
- **at least one `PATTERN` finding accepted** -- if none, M2 must not be built
- `MEMORY.md` flat or below 17.4 KB
- review under 5 minutes a day
- zero writes outside the two allowed paths, audited from `applied.md`

Kill criteria are in spec section 13. The strongest one: if any dream-authored memory is found to have misdirected real work, stop and audit provenance rather than tuning and continuing.
