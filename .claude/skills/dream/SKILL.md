---
name: dream
description: Out-of-band memory consolidation for Claude Code, two modes. Mode consolidate (nightly, non-interactive) reads today's extracted transcript corpus plus every memory surface and proposes evidence-gated corrections into DREAM_REPORT.md, writing nothing else. Mode review (interactive only) applies approved proposals behind an allowlist, a snapshot, and content-anchored edits, auto-applying only mechanical index repairs and recording every applied and rejected finding. Invoke consolidate from the timer; invoke review by hand.
---

# dream

Nightly memory consolidation, two modes. Mode consolidate is nightly,
non-interactive, and read-only except for `DREAM_REPORT.md`. Mode review is
the only mode ever allowed to write to `memory/` or `STATE.md`, and it only
runs when a human invoked it by hand. If you are running consolidate, do not
perform any review-mode action, and do not write to memory/ or STATE.md
under any circumstance.

## Mode: consolidate

### Corpus

Read exactly one file: `DREAM_ROOT/input/<today>.jsonl` (today = the run
date, `YYYY-MM-DD`). This is the deterministically extracted corpus --
human turns, assistant prose, failed-tool index, interrupt markers -- already
reduced from the raw session transcripts.

NEVER read the raw transcripts directly
(`~/.claude/projects/-home-james-ATLAS/*.jsonl` or any path under
`DREAM_TRANSCRIPT_ROOT`). They are roughly 250x larger than the extracted
corpus and will not fit in context. If the extracted corpus is missing or
empty, that is a finding for the operator, not a reason to fall back to the
raw transcripts.

### Read before proposing anything

Before writing a single finding, read all of:

- every file under `MEMORY_ROOT` (the topic files) plus `MEMORY.md`, the index
- `STATE.md`
- both CLAUDE.md files: the user-level `~/.claude/CLAUDE.md` and the
  project-level `CLAUDE.md` at the repo root
- every `LESSONS.md` under `.claude/skills/`
- `DREAM_ROOT/rejected.md`
- `DREAM_ROOT/applied.md`

`rejected.md` is not optional context -- it is the record of what a human
already declined. A claim that was rejected must not be re-proposed unless
the evidence set backing it has changed (new quote, new session, new
locator). Proposing the same rejected claim on the same evidence is noise
that trains the operator to stop reading the report.

`applied.md` is the same discipline for the other outcome: a claim already
WRITTEN must not be re-proposed. The correction it asked for is in the file
now, so re-proposing it is proposing a change to text that already says the
thing. The apply path refuses those (D-7), so a report full of them is a
report of items that cannot be applied -- and the operator spends the review
discovering that one refusal at a time.

The two ledgers key on DIFFERENT identities, deliberately, and the difference
matters when you decide whether something is a re-proposal. `rejected.md`
keys on the CLAIM (`fingerprint`), because a decline is a durable judgment
that must survive rewording. `applied.md` keys on the EDIT (`apply_identity`
= type + target + anchor + replacement), because an application is a one-time
act on a file state: when the world moves again, the same claim becomes
legitimately true again and earns a new, different edit. So a recurring
finding -- STATE.md naming a stale head, most of all -- is NOT a re-proposal
just because its wording matches an applied entry. Judge it on whether the
text it would write is already there. They are also not enforced alike: the
applied side is checked in code at the write boundary, while nothing reads
`rejected.md` programmatically, so that side is a duty rather than a guard.

### Finding taxonomy

Eight finding types, each `ftype` in `dream.report_schema.VALID_TYPES`. The
general evidence standard is "no quote, no proposal": every finding carries
at least one verbatim quote plus a locator (session id, turn index, ISO 8601
timestamp) UNLESS its type's rule below says otherwise. A finding the parser
cannot support with evidence is rejected at parse time, not surfaced for
approval -- do not attempt to write a finding with no `Evidence:` line for
any type, `STALE` and `INDEX` included; for those two, the `Evidence:` line
names the deterministic check performed, not a quote.

| Type | Meaning | May write | Evidence rule |
| --- | --- | --- | --- |
| CORRECTION | a memory entry is contradicted by later evidence | memory, STATE.md | cite BOTH sides: `Anchor:` holds the verbatim memory line being contradicted, `Evidence:` holds the verbatim transcript quote (with locator) that contradicts it. A one-sided correction -- a quote with no memory line named, or a memory line named with no contradicting quote -- is an opinion, not a correction. Reject it yourself before it reaches the report. If the claim is about MUTABLE WORLD STATE rather than about code or history, it MUST also carry a `text_absent:` recheck naming the contradiction that would falsify it -- see `Recheck:` below. |
| STALE | the referent is gone or superseded and nothing contradicts it | memory, STATE.md | `Evidence:` names the deterministic check (file absent, symbol absent from tree, PR closed); `Recheck:` carries the machine-runnable directive the apply path re-executes before writing, because the report is generated at 03:00 and reviewed hours later -- the check can flip in between. Never a quote here. |
| CONFLICT | two entries disagree with each other | proposal only | `Evidence:` quotes or locates both disagreeing entries; a human arbitrates, so do not propose a `Replacement:`. |
| DUPLICATE | the same fact lives in two places | memory | `Evidence:` locates both copies; propose the merge as `Anchor:`/`Replacement:` against the copy to retire. |
| NEW | a durable fact was established and is recorded nowhere | memory | `Evidence:` is the verbatim transcript quote establishing the fact, with locator. |
| PATTERN | the same lesson recurred and is uncaptured | memory | at least 2 occurrences from at least 2 DISTINCT session ids -- one session repeating itself inside its own turns is not a cross-session pattern. Carry one `Evidence:` line per occurrence (2+ lines), each with its own session id and locator, so the distinct-session requirement is checkable by inspection. |
| INDEX | unindexed file, dangling index link, or index over its byte budget | memory | `Evidence:` names the deterministic check (e.g. "file present, absent from index"); `Recheck:` MUST carry a real directive whenever the finding depends on a file's existence -- `file_exists:<path>` for an unindexed file, `file_absent:<path>` for a dangling link to retire. `none` only when the finding depends on nothing that can change (e.g. a pure byte-budget claim). INDEX auto-applies on TYPE, so emitting `none` buys nothing and only discards the check that stops a repair writing a dangling link. |
| GRADUATE | a lesson is now enforced by a hook, a test, or CLAUDE.md, so the memory entry has done its job | proposal only | `Evidence:` locates the hook/test/CLAUDE.md line that now enforces it; do not propose a `Replacement:`, a human confirms retirement. |

### Admission gates

A candidate finding must pass all four before it is written to the report.
These restate existing memory policy; they do not add new rules:

- **durable** -- true beyond this task or this session, not a status that
  expires ("#935 is blocked" is not durable; "#935 was blocked by X, fixed
  in commit Y" is)
- **non-derivable** -- not something a future agent can get by reading the
  repo, `git log`, or CLAUDE.md; if it is already derivable, it does not
  belong in memory at all
- **actionable** -- changes what a future agent actually does; a fact that
  changes no decision is trivia, not a finding
- **not already covered** -- no existing memory entry already states this,
  even loosely; if one does, the real finding is probably CORRECTION,
  DUPLICATE, or GRADUATE against that entry, not NEW

A candidate failing any gate is dropped silently -- gate failures are not
held-back findings and are not counted in the held-back line below.

### Volume cap

Emit at most **10 detailed items**, ranked by severity (high before medium
before low; within a severity, most-actionable first). Number them
sequentially starting at 1 in that ranked order.

Every finding that passed the admission gates but did not make the cap MUST
be counted and named in one line at the end of the report, after the last
finding block, in exactly this shape:

```
5 more findings held back: STALE(memory/project_x.md), NEW(STATE.md), ...
```

That shape is a count, then the literal text `more finding held back:` (use
`finding`, singular, only when the count is exactly 1; `findings` otherwise),
then a comma-separated list of `TYPE(target)` pairs for what was held back.
`dream.report_schema.parse_report` recognizes a line matching this shape as
report-level content -- it never raises on it and never attaches it to the
finding above it as a field -- but ONLY when the shape matches exactly. Free
text in its place (a different phrase, no leading count) is not recognized
and is treated as an unrecognized line inside the preceding finding's block,
which raises. When there is nothing held back, omit this line entirely.

Never truncate silently. A silently dropped finding is indistinguishable
from one that was never found, and that gap is exactly the failure mode
this system exists to close.

### Report format

Emit each finding as a block. The header line is parsed by
`dream.report_schema.parse_report` with an exact pattern -- match it
literally, including the spaces around each `-`:

```
## N - TYPE - target - severity
```

- `N` -- the sequential number from Volume cap above, an integer
- `TYPE` -- one of the eight types above, exactly as spelled (all caps)
- `target` -- a single token with no spaces: `STATE.md`, or a path under
  `memory/` such as `memory/MEMORY.md`
- `severity` -- `high`, `medium`, or `low`

Follow the header with field lines, one per line, in any order, each parsed
by key:

- `Claim:` -- one line, plain-language statement of the finding
- `Evidence:` -- one or more lines (repeat the key for each occurrence);
  required on every finding, no exceptions, see the taxonomy table for what
  each type's evidence line must contain
- `Anchor:` -- the verbatim text currently in the target file that the edit
  matches against; the apply path (mode review) refuses the edit if this
  text is not present verbatim at apply time, so quote it exactly, including
  punctuation
- `Replacement:` -- the literal text to substitute for `Anchor:`. To propose
  a multi-line replacement, write a literal two-character `\n` where a
  newline belongs; the parser converts every `\n` in this field into a real
  newline. This is how a single-line report format carries a multi-line
  edit.
- `Recheck:` -- one of four directives, or `none`. A directive asserts the
  condition that must HOLD for the write to be valid, and `apply_finding`
  re-executes it at write time, so a condition that changed since 03:00
  refuses rather than writing.
  - `file_exists:<path>` -- the referent must still be there (INDEX repairs)
  - `file_absent:<path>` -- it must still be gone (STALE retirements)
  - `text_absent:<needle>` -- the target must NOT contain this text
  - `text_present:<needle>` -- the target must STILL contain this text

  The `file_*` operands are paths resolved against the repo root. The `text_*`
  operands are NOT paths: they are literal needles matched against the
  finding's own target, which is why they can guard a `memory/` file at all
  (MEMORY_ROOT is outside the repo root, so no `file_*` operand can name one).
  Matching is exact and case-sensitive; the needle may contain colons, since
  only the first one separates verb from operand.

  Anything else -- an unknown verb, a missing operand -- is treated as
  UNSATISFIED and refuses the write, as is a `file_*` operand that escapes the
  repo root or a `text_*` target that cannot be read: "I cannot evaluate this
  precondition" must never read as "this precondition holds".

  **Which findings need one.** STALE and INDEX must carry a `file_*` directive
  as their taxonomy row requires. Beyond that, the test is what the claim is
  ABOUT, not what type it is:
  - a claim about MUTABLE WORLD STATE -- a service's status, credits, a PR's
    state, anything you would phrase as "currently" -- MUST carry a directive,
    because the six hours between 03:00 and review are enough for a human to
    act and for you to be wrong. Write `text_absent:` naming the contradiction
    that would falsify you, in the wording a correction would most likely use.
  - a claim about CODE OR HISTORY -- a line number, a shipped rule, what
    happened in a PR -- may use `none`. Those do not move overnight.
  Measured 2026-08-11: of ten applied findings, the one that had become false
  by review time was the only one making a live-state claim; the other nine
  re-verified clean, exact line citations included. A CORRECTION is the type
  most likely to be making such a claim, and carried `none` by construction
  until this directive existed.

A finding with no `Evidence:` line, or a `TYPE` outside the eight listed
above, is not a report the parser will accept -- it raises rather than
passing the finding through for human approval. Do not generate either
shape.

### Write boundary

This mode writes **nothing except `DREAM_REPORT.md`**, and nothing else in
`DREAM_ROOT`, and nothing under `MEMORY_ROOT`, and never `STATE.md` or any
CLAUDE.md. If a proposal looks worth writing directly, that impulse is the
thing to resist: the review mode exists precisely so a write has a human
approval, a snapshot, and a content anchor between the model and memory.

## Mode: review

Interactive only. Invoked by hand, never from the timer -- the timer never
writes, without exception, and review mode is the one place in this system
where a write is allowed to happen at all. Every function this mode calls
lives in `dream.dream_apply`; do not reimplement any of its logic inline.

### Setup

1. Acquire `dream.paths.LOCKFILE` before touching anything: create it with
   an atomic, exclusive operation (e.g. `open(LOCKFILE, "x")`, which fails if
   the file already exists) and write the run id into it. If acquisition
   fails, another review session is in progress -- stop and report that,
   do not wait or retry. Release the lock (delete the file) when review
   ends, including on error; a stale lock left behind blocks every future
   review session, so the release must run from a `finally`, not just the
   success path.
2. Generate one run id for the whole session: an ISO 8601 **basic** UTC
   timestamp matching `^\d{8}T\d{6}Z$` exactly, e.g. `20260808T030000Z`.
   The extended form (`2026-08-08T03:00:00Z`) is refused:
   `snapshot_or_refuse` validates the shape, because pruning sorts
   generations by name and `-` sorts before `0`, so one extended stamp
   makes pruning delete the newest snapshots and keep the oldest. Use this
   same run id as the snapshot stamp and pass it to every
   `stamp_provenance` call this session makes, so every write from one
   session traces back to one snapshot. You do NOT pass it to
   `record_applied`: `snapshot_or_refuse` retains the stamp and
   `apply_finding` writes the ledger entry itself (D-7), because a ledger
   the guard depends on cannot be written by a step a model can forget.
3. Build the review queue with `dream.pending.pending_findings(text)` on the
   text of `dream.paths.REPORT` -- not with `parse_report`, whose result is
   every finding the report carries, applied or not. A missing report means
   nothing to review: say so and stop, before reading it. So does an empty
   pending list, whether the report is empty or entirely applied already.
   The pending list is exactly what the SessionStart notice counts, and it
   is the list BOTH passes below walk. Walking the raw parse instead
   re-offers applied findings one at a time for the apply path to refuse
   individually, which is the queue this step exists to avoid -- and a
   PARTIALLY applied report, where that queue is interleaved with real work
   rather than the whole of it, is the common case, not an edge one. Do NOT
   reach past it to `already_applied` alone -- that is only the ledger half;
   the content half (`applied_on_content`) is what covers a lost ledger and
   a pre-D-7 entry (though not a SHORTENING edit, whose replacement is a
   substring of its own anchor: content cannot prove one landed, so those
   are settled by the ledger alone), and re-deriving either from prose here
   is the same second implementation D-7 exists to forbid. `pending_findings` parses the
   report itself, and raises the same `ValueError` on an unparseable one.
4. Take exactly one snapshot for the whole session with
   `dream.dream_apply.snapshot_or_refuse(run_id)`, before the first write.
   If it raises `WriteRefused`, stop -- do not apply anything unbacked.
   This step is not advisory: `apply_finding` refuses every write until
   this call has succeeded, so skipping it fails the session closed rather
   than writing without a recovery point.

### Targets

Pass `finding.target` to `apply_finding` **verbatim**, exactly as the report
carries it (`STATE.md`, or `memory/<path>`). Do not resolve it, do not join
it to a directory, do not turn it into an absolute path.
`dream.dream_apply.resolve_target` maps the token onto the real file and
`assert_allowed_target` gates the result. Resolving it yourself is what put
the process CWD in the write path, and the timer runs with CWD set to the
repo -- every `memory/...` write refused as a result.

### Auto-apply pass

For each finding in the pending list from step 3 -- never the raw
`parse_report` result -- check `dream.dream_apply.auto_apply_eligible(finding)`.
When true (INDEX type -- the type alone, regardless of what `Recheck:`
carries), apply it without prompting:

- do NOT re-run the directive yourself and do NOT skip the finding because
  it carries one. `apply_finding` evaluates `Recheck:` itself and raises
  `WriteRefused` if the condition has flipped, so an INDEX finding whose
  file was deleted since 03:00 refuses rather than writing a dangling link.
  Enforcement lives in code precisely so this paragraph cannot be the thing
  standing between a stale claim and the operator's memory.

  An earlier revision gated auto-apply on `recheck == none`. That was wrong
  in two ways. It made the carve-out unreachable for every INDEX finding
  whose claim depends on a file existing -- the common case, and the one the
  taxonomy above requires a directive for. (A pure byte-budget claim carries
  `none` legitimately and did still qualify, so the carve-out was not
  literally dead.) And, true of all of them, it rewarded emitting
  `Recheck: none` to obtain auto-apply -- trading the safety check for the
  convenience. Never reintroduce that condition.
- call `dream.dream_apply.apply_finding(finding, target)`. It writes the
  applied-ledger entry itself, so do NOT call `record_applied` -- a second
  call just duplicates the line. Then edit the finding's block in
  `DREAM_REPORT.md` to mark it applied by appending `[applied <run_id>]` to
  the header line. An auto-applied item must still be visible in the report
  -- an invisible mutation is exactly the failure mode this system exists to
  prevent, even when the mutation itself was low-risk. The mark is for the
  human reading the report and nothing keys off it: a finding is retired by
  the ledger (`already_applied`) or by the target's own content
  (`applied_on_content`), and neither of those reads the mark -- nor does
  the SessionStart notice, which is those two and nothing else. So
  forgetting the mark cannot resurrect a finding, and writing it cannot make
  the report unparseable (the header pattern accepts the trailing marker --
  it did not until 2026-08-09, when one marked header bricked the whole
  report).
- on `AnchorMissing` or `WriteRefused`, do not silently drop the finding:
  report the refusal explicitly (see Anchor refusals below) and leave it
  for the human queue below, do not re-attempt it as if it were still
  pending approval.

### Human approval pass

For every finding in that same pending list NOT auto-applied -- again never
the raw parse -- prompt the operator with the finding's
claim, evidence, and (for CONFLICT/GRADUATE, which never carry a
`Replacement:`) the fact that this one is proposal-only and cannot be
auto-written even on approval -- surface that distinction before asking,
do not let the operator approve a write that cannot happen.

On approval:

1. Do NOT evaluate `finding.recheck` yourself. `apply_finding` re-runs the
   directive at write time and raises `WriteRefused` naming the directive
   and what it observed, so an approved claim that stopped being true in the
   hours since 03:00 refuses instead of writing. Re-implementing the check
   here would put a second, drifting copy of the rule in prose beside the
   one in code. Report the refusal by name (finding number, directive, what
   the exception said), and do not silently re-queue it as still-pending --
   the operator approved a claim that is no longer true, so surface that
   plainly rather than either applying it anyway or dropping it without
   comment.
2. Call `dream.dream_apply.apply_finding(finding, target)`. For findings
   that author new memory content (`NEW`, `PATTERN`, or a `DUPLICATE`
   merge), stamp the replacement text with `stamp_provenance` before it
   reaches `apply_finding`, so the citation lands in the file, not just in
   the ledger.
3. On success, mark the finding applied in `DREAM_REPORT.md` the same way
   the auto-apply pass does. Do NOT call `record_applied` -- `apply_finding`
   has already written the ledger. The mark is what makes the report
   readable as a record of what was done; it is NOT what produced the false
   "ten unreviewed" count on 2026-08-09 -- that was the header-counting
   grep, now replaced. Marking all ten would have printed the same ten.
4. On `AnchorMissing` or `WriteRefused`, do not treat this as a quiet skip:
   report it explicitly to the operator (finding number, exception message)
   so a drifted anchor is visible and re-queueable on the next report,
   rather than disappearing with no trace. TWO `WriteRefused` messages need
   naming for what they are -- done, not drift -- and both key on the EDIT,
   never on the claim (D-7: identity is type + target + anchor +
   replacement, so the same claim recurring with different replacement text
   is a DIFFERENT edit and applies normally):
   - `already applied by run <id>` means this exact edit is recorded in
     `applied.md`.
   - `the replacement text is already present in <target>` means the target
     itself already holds this edit's result, whether or not the ledger
     records it. This is the half that survives a lost ledger, a pre-D-7
     entry, and a crash between the write and the record -- so there is
     usually no ledger line behind it at all, and looking for one to
     reconcile against will find nothing.
   Both are the guard working -- re-applying an append-shaped finding
   duplicates its content silently, because such a replacement embeds its
   own anchor and so never consumes it. Report either one, never work
   around it, and never edit the ledger to get past it without the operator
   deciding that explicitly.

On rejection, call `record_rejected(finding, reason)` with the operator's
stated reason. This is what lets consolidate mode's `rejected.md` read
recognize the same claim tomorrow even if it comes back reworded --
`fingerprint` keys on normalized claim content, not exact text, precisely
so a rejection survives a rewording.

### Anchor refusals

Never catch `AnchorMissing` or `WriteRefused` and continue without a
visible report line for that finding. A refusal that reaches the operator
as "5 of 7 findings applied" with no account of the other 2 is
indistinguishable from a bug that silently dropped them -- always name
which findings were refused and why, in the same summary that reports what
was applied.

### End of session

Report a summary: which findings were auto-applied, which were
human-approved and applied, which were rejected (with reason), and which
were refused (anchor drift or a write refusal) and therefore remain
pending for the next report. Write this summary from inside the same
`finally` that releases `LOCKFILE` (Setup step 1), so the two happen
together on every exit path, ordinary or not -- if the summary cannot be
produced, that failure must not suppress the lock release, or a stale lock
blocks every future session over a problem review mode already survived.
