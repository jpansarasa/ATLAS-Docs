# Reference: the wake listener — arming it, and why each part is load-bearing

Read this at session start when arming the listener, or when the loop stopped waking and you need
to know whether the Monitor died. SKILL.md CONFIG WAKE_LISTENER carries the one-line requirement.

Event-driven, not cron-poll — idle ticks are context rot plus a per-tick full cache miss.

ARM at session start (persistent Monitor, supervisor session ONLY — subagents NEVER Monitor
[[feedback_agent_long_wait_pattern]]):

    Monitor(command: while true; do curl -sN -K ~/.config/ntfy/claude-reply.curlrc
              'https://ntfy.elasticdevelopment.com/atlas-claude-reply/json'
              | jq --unbuffered -c 'select(.event=="message")';
              echo "$(date -u +%FT%TZ) stream closed, reconnecting" >&2; sleep 5; done,
            description: "atlas-claude-reply (user -> supervisor)", persistent: true)

jq filter is LOAD-BEARING: the stream emits open/keepalive events roughly every 45s — unfiltered
  they re-create the tick rot 20x over.
reconnect loop is LOAD-BEARING: the proxy cuts held streams, and the cut arrives as a CLEAN close
  right after an event — indistinguishable from normal completion, so nothing errors; without the
  loop every drop costs a wake turn. Reconnect logs to stderr (the output file), not to events.

ON EVENT -> ntfy_poll_new via MCP -> TURN_LOOP.
  MCP is the ack cursor and the source of truth; the monitor is a wake SIGNAL only. The poll also
  covers any messages that arrived inside a reconnect gap.
ON MONITOR-EXIT notification (only the loop itself dying, now that drops reconnect) -> re-arm,
  then poll_new.

RETIRED: the 15-min wakeup cron. A fixed-interval wake fires whether or not anything happened, so
  an idle night is ~25 identical tick pairs = transcript rot + stale-prompt drift + ~290k uncached
  tokens per tick.
