# Reference: the wake listener — arming it, and why each part is load-bearing

Read this at session start when arming the listener, or when the loop stopped waking and you need
to know whether the Monitor died. SKILL.md CONFIG WAKE_LISTENER carries the one-line requirement.

Event-driven, not cron-poll — idle ticks are context rot plus a per-tick full cache miss.

ARM at session start (supervisor session ONLY -- subagents NEVER Monitor
[[feedback_agent_long_wait_pattern]]). The Monitor tool CAPS every watch at 30 minutes and has no
persistent flag, so the listener must END ITSELF one minute inside the cap:

    Monitor(command: echo "LISTENER atlas-claude-reply armed $(date -u +%FT%TZ)" >&2;
              timeout 1740 bash -c 'while true; do curl -sN -K ~/.config/ntfy/claude-reply.curlrc
                https://ntfy.elasticdevelopment.com/atlas-claude-reply/json
                | jq --unbuffered -c "select(.event==\"message\")";
                echo "$(date -u +%FT%TZ) stream closed, reconnecting" >&2; sleep 5; done';
              echo "LISTENER atlas-claude-reply self-exit rc=$? $(date -u +%FT%TZ)" >&2; exit 0,
            description: "atlas-claude-reply (user -> supervisor)", timeout_ms: 1800000)

SELF-EXIT is LOAD-BEARING: a watch KILLED at the cap leaves NO completion record -- its notice has no
  <status>, <tool-use-id> or <output-file>, and its output file is empty. The next session's orphan
  scan then cannot tell "expired on schedule" from "abandoned mid-run" and lists EVERY expired
  listener as unfinished # measured 2026-09-21: eleven listed, ten had expired normally. A self-exit
  records <status>completed</status> and the harness appends "[exited with code 0]" to the file.
  # controls 2026-09-21: a self-exiting monitor, then this exact shape at an 8s deadline -- both
  # recorded completed; a live curl inside timeout dies with its process group (rc=124, then exit 0)
the LISTENER line is LOAD-BEARING: stderr, so it is NOT an event, and it names the task inside its own
  output file. The one listener live at a real process exit still shows as an orphan -- classify it:
    for id in <listed ids>; do head -1 <tasks-dir>/$id.output; done
  "LISTENER atlas-claude-reply armed" = the expected residual. Anything else = investigate it.

jq filter is LOAD-BEARING: the stream emits open/keepalive events roughly every 45s — unfiltered
  they re-create the tick rot 20x over.
reconnect loop is LOAD-BEARING: the proxy cuts held streams, and the cut arrives as a CLEAN close
  right after an event — indistinguishable from normal completion, so nothing errors; without the
  loop every drop costs a wake turn. Reconnect logs to stderr (the output file), not to events.

ON EVENT -> ntfy_poll_new via MCP -> TURN_LOOP.
  MCP is the ack cursor and the source of truth; the monitor is a wake SIGNAL only. The poll also
  covers any messages that arrived inside a reconnect gap.
ON MONITOR-EXIT notification -> re-arm, then poll_new. A self-exit every 29 minutes is the DESIGNED
  steady state, never a fault # budget ~2 re-arms per hour; a session of N hours pays ~2N.

RETIRED: the 15-min wakeup cron. A fixed-interval wake fires whether or not anything happened, so
  an idle night is ~25 identical tick pairs = transcript rot + stale-prompt drift + ~290k uncached
  tokens per tick.
