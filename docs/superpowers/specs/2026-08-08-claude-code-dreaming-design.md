# Dreaming: out-of-band memory consolidation for Claude Code

Status: APPROVED, not yet implemented
Date: 2026-08-08
Scope: agent tooling (`.claude/skills/dream/`, `scripts/dream-extract.py`), not an ATLAS service
Supersedes: nothing. Related: `.claude/skills/compact-memories` (manual precursor), #922 (STATE.md untracked), #933 (LESSONS.md graduation rule)

---

## 1. Problem

Claude Code's auto memory is written **in band**: the agent curates memory during the same session in which it does the work. Three consequences, all three confirmed as live costs in this repo:

1. **Split focus.** Curation competes with the task for attention.
2. **Patterns obfuscated.** A session writing memory sees only itself. Nothing in the setup reads across 549 sessions, 16 worktrees, or N parallel subagents, so a lesson gets re-learned instead of recorded. Worked example: "a defect found in a branch is not a defect on main" was learned three separate times in one session (STATE.md, OPEN ITEMS).
3. **Staleness.** Entries that were true once now misdirect. Worked examples in STATE.md: a `#913/#914` mapping recorded SWAPPED, a retracted "27 of 39 containers ship no logs" claim, an autofix-chain note corrected as "the old note here was wrong", one `superseded, kept for context` block.

The fix is a pass that runs **out of band**, over a **cross-session corpus**, with **age and contradiction detection**. Nothing is dropped from that triple; all three failure modes were confirmed as real by the requester.

## 2. Non-goals

- Not a service. No container, no port, no compose entry, no OTEL pipeline.
- Not a replacement for CLAUDE.md. Policy stays hand-written.
- Not RAG. Rejected deliberately: an embedding index is opaque and cannot be hand-mutated. Every artifact this system produces stays human readable markdown or jsonl -- greppable, diffable, editable by hand. Retrieval may return later as a hybrid for "has this come up before", never as the substrate.
- Not multi-machine. Auto memory is machine-local by design; this runs on mercury only.

## 3. Provenance, and what is NOT claimed

The idea reached us via a YouTube video (`jI4ZVB_MPhU`) and a derivative blog post (`bloss0m.com/en/blog/82-karpathy-claude-code-dreaming/`). Both are content marketing for a paid community. What each contributes, and what is rejected:

KEPT: the diagnosis (the three failure modes above), and Karpathy's framing that a sleep-like distillation pass has no equivalent in an LLM that boots at zero tokens.

REJECTED, do not repeat these downstream:
- "Anthropic shipped a feature called dreaming." Searched the full Claude Code documentation index on 2026-08-08: the word appears on no page. No memory-consolidation, background-memory, or transcript-analysis page exists. Absence from the docs is not proof no enterprise feature exists, but neither source cites anything checkable.
- "6x task completion at Harvey and Rakuten." Unsourced in both. Discard entirely.
- "Enterprise-only, consumes API credits." Unverifiable, and it is the setup for selling the DIY version.
- The blog contains no prompt or skill text. There is nothing to copy; this design is our own.

rationale: recording this here so a future reader does not re-derive the credibility check, and does not cite the 6x number as if it were evidence.

## 4. Evidence

All measured on mercury, 2026-08-08.

Corpus:

| Measure | Value |
| --- | --- |
| Total transcripts | 549 files, 338 MB |
| Modified in last 24h | 70 files, 49.8 MB, 16,845 records |
| `assistant` records | 8,604 rec, 20.87 MB |
| `user` records | 5,218 rec, 23.51 MB |
| of which `tool_result` | 4,990 rec, 22.49 MB |
| **actual human turns** | **200 turns, 699.7 KB, mean 3,582 chars** |
| `attachment` | 1,272 rec, 3.78 MB |
| `queue-operation` | 446 rec, 1.13 MB |

Human turns are **0.4 percent** of the daily corpus. A naive "read the last 24h of transcripts" -- which is what both sources describe -- overshoots context by roughly 250x. Deterministic extraction is not an optimization, it is what makes the design possible. This is GIGO applied literally: clean at the source, do not hand 50 MB to a model and hope.

Memory surface:

| Measure | Value |
| --- | --- |
| `MEMORY.md` index | 91 lines of 200, **17.4 KB of the 25 KB load ceiling** |
| Topic files | 92 files, 424 KB |
| Files carrying `modified:` frontmatter | 3 of 92 |
| Dangling index entries | 0 |
| Files present but absent from the index | 1 (`project_health_is_tempo_not_loki.md`) |

Two findings that shape the design:

- **Bytes bind before lines.** The index is at 70 percent of its byte budget with 92 topic files and growing. Content past 25 KB is silently dropped at load. Consolidation must therefore be net-neutral-or-negative on index size: it proposes retirements, not only additions.
- **`modified:` is not a usable staleness signal.** Only 3 of 92 files carry it, and Claude Code never adds frontmatter to a file that lacks it, so most never will. Staleness comes from filesystem mtime plus contradiction detection.

The unindexed file is a live, present-tense instance of exactly the defect class this system targets: it exists, it is invisible at session start, and deterministic checking found it in 30 seconds.

## 5. Architecture

Four components, each independently testable, connected by files on disk.

```
~/.claude/projects/-home-james-ATLAS/*.jsonl          50 MB/day, 549 sessions
   |
   |  [1] dream-extract        deterministic, no model, no network
   |      keep    human turns, assistant prose, failed-tool index
   |      drop    tool_result bodies, attachments, queue-ops, mode records
   |      strip   system-reminder blocks, command wrappers, interrupt noise
   v
dream/input/YYYY-MM-DD.jsonl                          ~700 KB/day (0.4%)
   |
   |  [2] dream-consolidate    one `claude -p` pass, Opus
   |      reads   today's input, all memory surfaces, rejected.md
   |      writes  nothing but the report
   v
dream/DREAM_REPORT.md                                 numbered proposals + evidence
   |
   |  [3] /dream review        interactive only, never from the timer
   |      approve -> memory/*.md, STATE.md   (snapshot first)
   |      reject  -> rejected.md, never re-proposed
   v
memory/ and STATE.md                                  CLAUDE.md, LESSONS.md: proposals only
```

`[4] dream.timer` -- systemd, 03:00 local, ansible-managed. Runs 1 then 2. Never runs 3.

### 5.1 Why the boundaries fall there

- **Extraction is deterministic and model-free** so it is testable against fixtures with exact assertions, and so transcript schema drift surfaces as a test failure rather than a quiet quality drop. The `.jsonl` shape is undocumented internal structure and will change.
- **Consolidation never writes.** Analysis and mutation are separate processes with separate failure modes. A bad analysis yields a bad report you reject; it cannot corrupt memory.
- **Application is deterministic given approvals.** Once items are approved, applying them requires no judgment -- so the risky step, writing an unbacked STATE.md, has no model in the loop.

### 5.2 Layout

| Artifact | Location | rationale |
| --- | --- | --- |
| Skill, extractor | `.claude/skills/dream/`, `scripts/dream-extract.py` | git-tracked, ansible-deployable, reviewable as a PR |
| input, report, rejected, applied, snapshots, digests | `~/.claude/projects/-home-james-ATLAS/dream/` | machine-local derived state, mirrors where auto memory already lives, never pollutes the repo |

## 6. Component 1: `dream-extract`

Pure function: transcript paths in, filtered corpus out. No model, no network, no writes to memory.

### 6.1 Keep

| Kept | rationale |
| --- | --- |
| Human turns (`type=user`, not `tool_result`, not `isMeta`) | corrections and preferences, the primary signal |
| Assistant prose (text blocks only, `tool_use` params dropped) | needed for "claim later falsified", the retraction pattern behind three of STATE.md's corrections |
| Failed-tool index `{tool, error[:200], session, turn}` | bodies dropped, errors kept; catches the same command failing across three agents |
| Interrupt markers plus the tool call being attempted | an interrupt is the user saying "stop, wrong thing" with the wrong thing recorded beside it; highest-density correction signal in the corpus |

### 6.2 Drop and strip

Dropped whole: `tool_result` bodies (22.49 MB), `attachment` (3.78 MB), `queue-operation`, `mode`, `permission-mode`, `ai-title`, `pr-link`, `last-prompt`, `file-history-snapshot`, `file-history-delta`.

Stripped from kept text:
- `<system-reminder>` blocks -- injected harness context, not the user's words. Without this the pass learns "preferences" from its own scaffolding.
- `<command-message>` / `<command-name>` wrappers -- normalized to `[slash: /name]` so invocation stays visible without reading as prose.

### 6.3 Output

`dream/input/YYYY-MM-DD.jsonl`, one record per kept turn:

```json
{"session":"89a0eb74","ts":"2026-08-07T14:02:11Z","turn":42,"role":"user","kind":"human","text":"..."}
```

### 6.4 Fail-loud guard (D-1)

Not a volume threshold. A quiet Sunday is legitimately five turns, so any magic number is either noise or useless. The guard is structural and volume-independent:

> **Every session containing at least one assistant record must yield at least one human turn.** A session cannot exist without a prompt.

Plus: parse failures above 1 percent of lines; zero sessions found when files were modified in the window.

Any trip: **exit non-zero, ntfy at high priority, write no report.**

rationale: the failure this prevents is silent. Extraction returns nothing, the pass reports "memory looks current", and you believe it. That is a corpse detector -- it reports health after the thing it monitors is already dead.

## 7. Component 2: `dream-consolidate`

One `claude -p` pass. Reads today's input, every memory surface (memory/, STATE.md, both CLAUDE.md, LESSONS.md, skills), and `rejected.md`. Writes only `DREAM_REPORT.md`.

### 7.1 Finding taxonomy

| Group | Type | Action | May write |
| --- | --- | --- | --- |
| Against memory | `CORRECTION` | entry contradicted by later evidence, rewrite | memory, STATE.md |
| | `STALE` | referent gone or superseded, nothing contradicts it, delete | memory, STATE.md |
| | `CONFLICT` | two entries disagree, arbitrate | proposal only |
| | `DUPLICATE` | same fact in two places, merge | memory |
| Into memory | `NEW` | durable fact established, recorded nowhere | memory |
| | `PATTERN` | recurred at least twice across at least two distinct sessions, uncaptured | memory |
| About memory | `INDEX` | unindexed file, dangling link, index over byte budget | memory |
| | `GRADUATE` | lesson now enforced by a hook, test, or CLAUDE.md, retire it | proposal only |

`PATTERN` is the class that justifies the architecture; nothing else in the setup can produce it. `GRADUATE` automates #933's rule that a lesson's goal is to stop being a lesson.

### 7.2 Evidence standard

**No quote, no proposal.** Every finding carries verbatim text plus a locator (session id, turn index, ISO 8601 timestamp). If the pass cannot quote it, it inferred it -- and inference is what put the wrong entries in memory to begin with.

Per-class requirements, and the differences are the point:

- `CORRECTION` cites **both sides**: the memory line contradicted, and the transcript text contradicting it. A one-sided correction is an opinion.
- `PATTERN` cites **at least two occurrences from at least two distinct session ids.** One session repeating itself is not a cross-session pattern; without this discriminator the class degenerates into "things said twice".
- `STALE` is evidenced by a **deterministic check**, not a quote: file absent, symbol absent from tree, PR closed.

**Staleness checks re-run at apply time and are never trusted from the report.** A file missing at 03:00 may exist at 09:00. The report records the check; approval re-executes it and refuses if the result flipped.

rationale: without re-execution the system acquires the exact defect it exists to remove -- a confidently wrong claim, aged six hours.

### 7.3 Admission gates

A candidate must be all four. These restate existing memory policy rather than inventing new rules:

- **durable** -- true beyond this task, not "#935 is blocked"
- **non-derivable** -- not obtainable from the repo, git log, or CLAUDE.md
- **actionable** -- changes what a future agent does
- **not already covered** by an existing entry

### 7.4 Volume

Cap the report at **10 detailed items**, ranked by severity. Held-back findings are **counted and named in one line**, never silently truncated.

A cold pass over 549 historical sessions could surface hundreds, so backfill is an explicit, chunked, one-time mode -- not the nightly path.

### 7.5 Report shape

```markdown
## 3 - CORRECTION - STATE.md - high
Claim      autofix-watcher does not poll main's tip; it fires only on human merge
Memory     > "nothing polls main's tip"                STATE.md, DEPLOY/RUNTIME
Evidence   > "the old note here was wrong"             89a0eb74, turn 42, 2026-08-07T14:02Z
Edit       <unified diff, content-anchored>
Recheck    none (textual)
```

## 8. Component 3: `/dream review` -- the write path

Interactive only. Never invoked by the timer.

### 8.1 Snapshot first (D-2)

Before any write in a run: `STATE.md` verbatim and the memory directory as a tarball, to `dream/snapshots/<iso8601>/`. Once per apply session, not per item. Retain 30 generations (424 KB each, about 12 MB total).

rationale: STATE.md has been gitignored since #922 and has **no recovery path whatsoever**. This snapshot is currently its only backup, and closes that standing risk independent of whether any proposal is ever useful.

### 8.2 Path allowlist, enforced in code (D-3)

Writes are refused for any target outside exactly two roots:

- `~/.claude/projects/-home-james-ATLAS/memory/`
- `/home/james/ATLAS/STATE.md`

An **allowlist of two permitted paths**, not a denylist of forbidden ones.

rationale: directly downstream of #935. Three rounds there each claimed writes were closed, and each was falsified by probing outside the test corpus, because enumerating what is forbidden leaves every unenumerated shape permitted. Two allowed paths is a surface small enough to verify exhaustively.

### 8.3 Anchored edits (D-4)

Proposals carry **content anchors, not line numbers**. An edit applies only if its anchor text is present verbatim. If the anchor is gone, the edit **refuses and re-queues** for the next pass. Never fuzzy-match, never fall back to nearest position. Refusals are reported, never swallowed.

rationale: the report is generated overnight and reviewed hours later, by which time a morning session may have rewritten the section. Same discipline as the tree-hash push marker: key on content, never on position. A silently skipped edit is indistinguishable from an applied one, which is how you come to trust a memory that was never written.

Application is per item, not all-or-nothing. One refusal does not abort the batch.

### 8.4 Auto-apply, narrowly (D-5)

Exactly one class auto-applies: `INDEX` findings that are purely mechanical -- adding an index line for an existing file, dropping a link to a deleted one. These have a single deterministic correct answer and are self-verifying, since the check that found the defect confirms the repair.

**Auto-apply runs inside `/dream review`, not from the timer.** "Auto" means it skips the approve prompt, never that it happens unattended. The timer-never-writes invariant of section 5 holds without exception: the 03:00 run produces a report in which eligible `INDEX` items are marked pending-auto, and they are applied when you next open review.

Hard constraint: **auto-applied items still appear in the report, marked as already applied.**

rationale: auto-apply that is invisible is how memory drifts. Auto-apply that is disclosed is work you did not have to click through. Disabling it is one config line; everything then falls back to the gate.

### 8.5 Ledger and provenance (D-6)

Every applied change appends to `dream/applied.md`: timestamp, finding type, target, diff, evidence locator. Snapshot plus ledger gives a manual undo path requiring no tooling.

Every memory entry the pass authors carries inline provenance -- run id and evidence locator.

rationale: without provenance a wrong dream-authored memory is indistinguishable from a hand-written fact; the next pass reads it as ground truth and compounds it. With it, any entry traces back to the quote that justified it, and a `CORRECTION` against a dream-authored entry is visibly the system catching its own error.

### 8.6 Concurrency

The pass applies only in an interactive session, so it cannot collide with itself. A lockfile in `dream/` prevents two `/dream review` sessions overlapping. Against parallel worktrees, the anchor check is the real protection: a subagent that rewrote the section invalidates the anchor and the edit refuses rather than clobbering.

## 9. Component 4: schedule and notification

systemd timer at 03:00 local, ansible-managed, running components 1 and 2.

Cloud routines are structurally unavailable: auto memory is machine-local, "not shared across machines or cloud environments", so a cloud routine cannot read mercury's transcripts or write its memory directory.

### 9.1 ntfy contract

Publish to `atlas-claude-ask`, **only when there are new proposals** -- not the standing backlog. Re-pinging eight unreviewed items nightly is how a channel gets muted.

```
ATLAS Dream - 6 new proposals
STALE: STATE.md says autofix-watcher runs ansible - retracted 2026-08-07
3 stale, 2 new facts, 1 index repair  |  corpus: 187 turns / 12 sessions
Review: /dream review
```

Verdict first, highest-severity item second (mobile-first format).

The `corpus:` line is load-bearing. It makes a healthy zero distinguishable from a broken zero: "0 proposals from 187 turns" is a quiet day, "0 proposals from 0 turns" is a dead extractor reporting success. Without it the two render identically.

Reuse the existing #881 systemd `OnFailure` ntfy path rather than minting a second credential path.

Extraction failure is a **separate, higher-priority message**, and no report is written.

### 9.2 SessionStart notice

When `DREAM_REPORT.md` holds unreviewed items, SessionStart emits one line: count and highest-severity item.

rationale: a pending report nobody opens is worthless. ntfy is the push; this is the pull-side reminder at the moment you are actually able to act.

## 10. Milestones

### M1 -- the working loop

| PR | Contents | Model-free verification |
| --- | --- | --- |
| 1 | `dream-extract.py`, fixtures, fail-loud guard, unit tests | yes, pure function with exact assertions |
| 2 | `/dream` skill: consolidate, report format | partial, report schema is assertable |
| 3 | `/dream review`: allowlist, anchors, snapshot, ledger | yes, deterministic given approvals |
| 4 | systemd timer, ansible, ntfy, SessionStart notice | integration |

PRs 1 and 3 carry the guards and are fully testable with no model in the loop. Deliberate: the parts that can silently corrupt something are the parts a test can pin exactly.

### M2 -- the digest layer

Built **only after M1 has run about two weeks and produced findings worth having.**

Per-day digests persisted (corrections issued, preferences repeated, facts established, claims later falsified), consolidation reading weeks of digests rather than one day of turns, plus a chunked one-time backfill over the 549 historical sessions.

rationale: the digest schema gets designed against findings M1 actually produced, not findings guessed at in advance. That is the entire reason for sequencing it second.

Model split follows the project's economy rule: extraction uses no model, consolidation is judgment-heavy and uses Opus, M2's per-day digest map stage uses Sonnet or Haiku.

## 11. Tests

Beyond extractor unit tests (fixture in, exact records out, covering each keep/drop/strip rule), four guard tests, each written to go **RED when its guard is deleted**:

| Test | Constructs | Asserts |
| --- | --- | --- |
| Schema drift | fixture with a renamed field | non-zero exit AND no report written |
| Allowlist | proposal targeting `CLAUDE.md` | refused at the boundary through the real apply flow |
| Anchor drift | report generated, target mutated, then apply | refuses and re-queues, target byte-identical |
| Rejection durability | reject an item, re-run on same corpus; then add a new citation | not re-proposed; then proposed again |

Plus idempotence (applying the same report twice is a no-op) and a snapshot test (snapshot exists and matches pre-state).

**Mutation pass required:** delete each guard line in turn, confirm the suite goes red. #935 shipped a battery in which deleting `is_gate_path "/$t"` left the suite fully green. There are only a handful of guards here, so this is cheap.

## 12. Success criteria, measured at two weeks

- Accept rate **at or above 50 percent**. Below that you are reviewing noise.
- **At least one `PATTERN` finding accepted.** If two weeks of real work yields none, the cross-session premise is wrong for this setup and **M2 must not be built**.
- `MEMORY.md` byte count **flat or falling** from 17.4 KB. A pass that grows the index toward the 25 KB cliff worsens the problem it exists to solve.
- Review under **5 minutes a day**.
- **Zero writes outside the two allowed paths**, audited from the ledger.

## 13. Kill criteria

A memory system you do not trust is worse than none, because it launders bad facts through apparent process. Stop if any of:

- accept rate below 25 percent over ten nights -- the pass generates plausible noise; disable or narrow classes
- the extraction guard trips repeatedly with no schema change -- the invariant is wrong
- review skipped five days running -- it is not earning its slot
- **any dream-authored memory is found to have misdirected real work** -- the precise harm this exists to prevent. Stop and audit provenance; do not tune and continue.

Graceful by construction: every artifact is markdown, snapshots, and an append-only ledger. Killing it means disabling one timer. Nothing to migrate, nothing to unwind.

## 14. Retention and assumptions

- `dream/input/*.jsonl`: 30 days
- `dream/snapshots/`: 30 generations
- `dream/digests/`: permanent (M2)
- `rejected.md`, `applied.md`: permanent, append-only
- Quiet nights are silent -- no proposals means no push
- `/dream review` is interactive only and never runs from the timer

## DECISIONS

D-1 fail-loud-extraction: INTENT a dead extractor must never report health / PRECOND any session with an assistant record yields at least one human turn, parse failures under 1 percent, sessions found when files changed / GUARD `dream_extract.assert_corpus_plausible` @ scripts/dream-extract.py / TEST `TestExtractGuards.test_renamed_field_exits_nonzero_and_writes_no_report`

D-2 snapshot-before-write: INTENT STATE.md has no git backup since #922, so the apply path is its only recovery mechanism / PRECOND a snapshot exists and is verified before the first write of any run / GUARD `dream_apply.snapshot_or_refuse` / TEST `TestApplySafety.test_write_refused_when_snapshot_fails`

D-3 write-path-allowlist: INTENT enumerate what is permitted, never what is forbidden, because #935 proved denylists leak at every unenumerated shape / PRECOND target resolves inside memory/ or is exactly STATE.md / GUARD `dream_apply.assert_allowed_target` / TEST `TestApplySafety.test_claude_md_target_refused_through_real_flow`

D-4 content-anchored-edits: INTENT the file moves between report time and review time, so position is not identity / PRECOND anchor text present verbatim, else refuse and re-queue / GUARD `dream_apply.match_anchor_exact` / TEST `TestApplySafety.test_mutated_target_refuses_and_leaves_file_identical`

D-5 auto-apply-scope: INTENT mechanical index repair is not judgment, but invisible mutation is how memory drifts / PRECOND finding type is INDEX, repair is self-verifying, and the item still appears in the report marked applied / GUARD `dream_apply.auto_apply_eligible` / TEST `TestAutoApply.test_non_index_finding_never_auto_applies`

D-6 provenance-inline: INTENT a wrong dream-authored memory must be traceable to its evidence, not indistinguishable from a hand-written fact / PRECOND every authored entry carries run id and evidence locator / GUARD `dream_apply.stamp_provenance` / TEST `TestProvenance.test_authored_entry_carries_evidence_locator`

---

## HARD STOPS

- never write outside the two allowlisted paths
- never apply an edit whose anchor is absent
- never write a report when the extraction guard trips
- never trust a `STALE` check from report time; re-run it at apply time
- never silently truncate findings; count and name what was held back
- never re-propose a rejected claim unless the evidence set differs
- never build M2 before M1 has yielded an accepted `PATTERN` finding
