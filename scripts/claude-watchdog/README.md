# Claude watchdog

Background watchdog that detects long-running Claude Code sessions that
appear to be sitting idle, blocked on user input (permission prompt,
`AskUserQuestion`, or just the user-input cursor with nobody at the keyboard).

## What it does

`scan.py` — single source of truth for detection. Reads:
- `ps` output for live `claude` CLI processes and their elapsed time
- `~/.claude/projects/<slug>/*.jsonl` transcripts (newest per project only)

A session is flagged when **all** of the following hold:

| signal | default | env override |
| --- | --- | --- |
| session runtime (first event → now) | ≥ 30 min | `CLAUDE_WATCHDOG_MIN_RUNTIME_MIN` |
| time since the most recent JSONL event | ≥ 10 min | `CLAUDE_WATCHDOG_IDLE_MIN` |
| time since the most recent JSONL event | ≤ 180 min | `CLAUDE_WATCHDOG_MAX_IDLE_MIN` |
| a live `claude` PID has the transcript JSONL open via `/proc/<pid>/fd` | required | — |

The upper bound on idle time suppresses transcripts where the user moved
on to a new session within the same long-lived `claude` CLI process — the
old JSONL stops getting writes but that doesn't mean the session is
"waiting for input." Only fresh-but-stale sessions trip the alarm.

`notify.sh` — wraps `scan.py`. If `finding_count > 0`, publishes a single
summary message to NTFY topic `atlas-claude-ask`. Per-session cooldown
(default 60 min, env: `CLAUDE_WATCHDOG_COOLDOWN_MIN`) prevents re-alerting
on the same stuck session every 15 min until the user acts on it.

State + logs live under `~/.cache/claude-watchdog/`:
- `notify.log` — every run that takes action (alert sent, alert failed, cooldown hit)
- `last-alerted.json` — `{ "<session_id>": "<iso8601_utc>" }` for cooldown bookkeeping

## Schedule

Installed via the user crontab on mercury:

```
*/15 * * * * /home/james/ATLAS/scripts/claude-watchdog/notify.sh >/dev/null 2>&1
```

15-min cadence = Nyquist for the 30-min runtime gate; the longest a stuck
session can sit before being noticed is ~25 min.

## Disable

```bash
crontab -l | grep -v claude-watchdog | crontab -
```

To re-enable: `crontab -l | { cat; echo '*/15 * * * * /home/james/ATLAS/scripts/claude-watchdog/notify.sh >/dev/null 2>&1'; } | crontab -`

## Manual run

```bash
/home/james/ATLAS/scripts/claude-watchdog/scan.py            # JSON findings to stdout
/home/james/ATLAS/scripts/claude-watchdog/notify.sh          # scan + NTFY if findings
```

Force a test alert (ignores thresholds + cooldown):

```bash
CLAUDE_WATCHDOG_MIN_RUNTIME_MIN=1 \
CLAUDE_WATCHDOG_IDLE_MIN=1 \
CLAUDE_WATCHDOG_COOLDOWN_MIN=0 \
/home/james/ATLAS/scripts/claude-watchdog/notify.sh
```

## NTFY wiring

Topic: `atlas-claude-ask` on `https://ntfy.elasticdevelopment.com` with
basic auth `<user>:<pass>` (same channel the supervisor uses to ping the
user). Override with `NTFY_BASE_URL`, `NTFY_TOPIC`, `NTFY_AUTH` env vars
if needed.

### Auth setup

`notify.sh` reads credentials from `~/.config/claude-watchdog/auth`
(no newline, just `user:pass`). The real ATLAS NTFY credentials live in
ansible-vault — drop the same `atlas` user value into the file below and
chmod it to 600:

```bash
mkdir -p ~/.config/claude-watchdog
printf '<user>:<pass>' > ~/.config/claude-watchdog/auth
chmod 600 ~/.config/claude-watchdog/auth
```

If the file is missing and `NTFY_AUTH` isn't set in the environment, the
script logs and exits 0 — no unauthenticated requests are sent.
