# ntfy MCP Server

MCP server exposing ntfy publish/poll as structured tools with `last_seen_ts` persistence per topic. Used by the Sentinel v2 supervisor to communicate with the user without curl boilerplate.

> Originally named `sentinel-mcp` / `sentinel_ntfy`. Renamed to `ntfy-mcp` / `ntfy` because the bridge is not Sentinel-specific. The MCP server name registered in `~/.claude.json` remains `sentinel-ntfy` for backward compatibility — update it together with the `command` path when convenient.

## Tools

| Tool | Params | Returns | State mutation |
|------|--------|---------|---------------|
| `ntfy_publish` | `topic`, `title`, `body`, `priority?`, `tags?` | `{id, time, topic}` | none |
| `ntfy_poll_new` | `topic`, `max_messages?=50` | `{messages[], last_seen_ts}` | advances `last_seen_ts` |
| `ntfy_poll_since` | `topic`, `since`, `max_messages?=50` | `{messages[], last_seen_ts}` | none (diagnostic) |
| `ntfy_ack` | `topic`, `msg_id?` | `{acked_ts}` | advances `last_seen_ts` |

`since` for `ntfy_poll_since` accepts ntfy duration strings (`6h`, `1d`) or a unix timestamp as a string.

## Configuration

Auth and endpoint via env vars — never hardcoded:

| Var | Default | Required |
|-----|---------|----------|
| `NTFY_ENDPOINT` | `https://ntfy.elasticdevelopment.com` | no |
| `NTFY_USER` | `atlas` | no |
| `NTFY_PASSWORD` | — | **yes** |
| `NTFY_STATE_FILE` | `/opt/ai-inference/sentinel-mcp-state.json` | no |

> The default state file path keeps the legacy `sentinel-mcp-state.json` filename to preserve existing `last_seen_ts` state across the rename.

## State file

Shape at `/opt/ai-inference/sentinel-mcp-state.json`:

```json
{
  "topics": {
    "atlas-claude-reply": { "last_seen_ts": 1777060000 },
    "atlas-claude-ask":   { "last_seen_ts": 0 }
  },
  "schema_version": 1
}
```

Loaded on startup; saved atomically (write-to-tmp then rename) on every state-mutating call. Missing file is created with defaults.

## Install

```bash
cd /home/james/ATLAS/ntfy-mcp
python3 -m venv .venv
source .venv/bin/activate
pip install -e .
```

## Claude Code registration

Add to `~/.claude.json` under `mcpServers` (read-modify-write — do NOT overwrite the whole file):

```json
"sentinel-ntfy": {
  "type": "stdio",
  "command": "/home/james/ATLAS/ntfy-mcp/.venv/bin/python",
  "args": ["-m", "ntfy"],
  "env": {
    "NTFY_ENDPOINT": "https://ntfy.elasticdevelopment.com",
    "NTFY_USER": "atlas",
    "NTFY_PASSWORD": "<password>",
    "NTFY_STATE_FILE": "/opt/ai-inference/sentinel-mcp-state.json"
  }
}
```

> The server-name key `sentinel-ntfy` is the legacy MCP server name. You may rename it to `ntfy` at the same time you update the `command` path — just keep both consistent with this repo's pyproject `[project.scripts]` entry.

Restart Claude Code after editing `~/.claude.json` to activate.

## Smoke test (client only)

```bash
source .venv/bin/activate
python3 -c "
import asyncio, os
from ntfy.client import NtfyClient
async def main():
    c = NtfyClient('https://ntfy.elasticdevelopment.com', 'atlas', os.environ['NTFY_PASSWORD'])
    r = await c.publish('atlas-claude-ask', title='MCP smoke test', body='ntfy-mcp built and registered')
    print('published id=', r['id'])
    msgs = await c.poll_since('atlas-claude-ask', since='5m')
    print(f'saw {len(msgs)} recent messages')
asyncio.run(main())
"
```

## Run tests

```bash
source .venv/bin/activate
NTFY_PASSWORD=<password> pytest tests/ -v
# Skip network calls:
SENTINEL_MCP_SKIP_NETWORK=1 pytest tests/ -v
```
