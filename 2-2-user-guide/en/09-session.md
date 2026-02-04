# Session Management - Summary

## Core Concept
- **One main session per agent** for direct chats
- Group/channel chats get isolated sessions

## Session Keys

| Source | Key Format |
|--------|------------|
| DM (default) | `agent:<agentId>:main` |
| DM (per-peer) | `agent:<agentId>:dm:<peerId>` |
| Group | `agent:<agentId>:<channel>:group:<id>` |
| Cron | `cron:<jobId>` |
| Webhook | `hook:<uuid>` |

## DM Scope Options

```json5
{
  session: {
    dmScope: "main"  // main | per-peer | per-channel-peer | per-account-channel-peer
  }
}
```

- `main`: All DMs share one session (continuity)
- `per-peer`: Isolate by sender
- `per-channel-peer`: Isolate by channel + sender
- `per-account-channel-peer`: Isolate by account + channel + sender

## Where State Lives
- Store: `~/.openclaw/agents/<agentId>/sessions/sessions.json`
- Transcripts: `~/.openclaw/agents/<agentId>/sessions/<SessionId>.jsonl`

## Session Lifecycle

### Reset Triggers
- Daily: 4:00 AM local (default)
- Idle: configurable `idleMinutes`
- Manual: `/new` or `/reset` command

### Config Example
```json5
{
  session: {
    reset: {
      mode: "daily",
      atHour: 4,
      idleMinutes: 120
    },
    resetByType: {
      dm: { mode: "idle", idleMinutes: 240 },
      group: { mode: "idle", idleMinutes: 120 }
    }
  }
}
```

## Send Policy
Block delivery for specific session types:

```json5
{
  session: {
    sendPolicy: {
      rules: [
        { action: "deny", match: { channel: "discord", chatType: "group" } },
        { action: "deny", match: { keyPrefix: "cron:" } }
      ],
      default: "allow"
    }
  }
}
```

Runtime override: `/send on|off|inherit`

## Identity Links
Map same person across channels:

```json5
{
  session: {
    identityLinks: {
      alice: ["telegram:123456789", "discord:987654321"]
    }
  }
}
```

## Chat Commands
- `/status` - Check agent status + context usage
- `/context list` - See what's in system prompt
- `/new` - Start fresh session
- `/new <model>` - New session with model override
- `/stop` - Abort current run
- `/compact` - Summarize older context

## Inspecting

```bash
# Show recent sessions
openclaw status

# List all sessions
openclaw sessions --json

# Filter active sessions
openclaw sessions --active 60
```

## Session Pruning
Old tool results are trimmed from in-memory context before LLM calls.
Does NOT rewrite JSONL history (transcript preserved).
