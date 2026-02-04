# Cron Jobs - Summary

## What Is It?
Gateway's built-in scheduler for timed tasks.
- Runs inside the Gateway process
- Persists jobs in `~/.openclaw/cron/jobs.json`
- Survives restarts

## Two Execution Styles

### 1. Main Session (`sessionTarget: "main"`)
- Enqueues system event
- Runs on next heartbeat with full context
- Payload: `kind: "systemEvent"`

### 2. Isolated Session (`sessionTarget: "isolated"`)
- Fresh session `cron:<jobId>`
- No prior conversation context
- Posts summary to main session
- Payload: `kind: "agentTurn"`

## Schedule Types

| Type | CLI Flag | Description |
|------|----------|-------------|
| `at` | `--at` | One-shot timestamp |
| `every` | `--every` | Fixed interval |
| `cron` | `--cron` | 5-field cron expression |

## Quick Start

### One-shot Reminder
```bash
openclaw cron add \
  --name "Reminder" \
  --at "2026-02-01T16:00:00Z" \
  --session main \
  --system-event "Reminder: check the docs" \
  --wake now \
  --delete-after-run
```

### Recurring Job with Delivery
```bash
openclaw cron add \
  --name "Morning brief" \
  --cron "0 7 * * *" \
  --tz "America/Los_Angeles" \
  --session isolated \
  --message "Summarize overnight updates." \
  --deliver \
  --channel slack \
  --to "channel:C1234567890"
```

### With Model Override
```bash
openclaw cron add \
  --name "Deep analysis" \
  --cron "0 6 * * 1" \
  --session isolated \
  --message "Weekly analysis" \
  --model "opus" \
  --thinking high \
  --deliver
```

## CLI Commands

```bash
# List jobs
openclaw cron list

# Run immediately
openclaw cron run <jobId> --force

# View run history
openclaw cron runs --id <jobId>

# Edit job
openclaw cron edit <jobId> --message "New prompt"

# Delete job
openclaw cron remove <jobId>
```

## Configuration

```json5
{
  cron: {
    enabled: true,
    store: "~/.openclaw/cron/jobs.json",
    maxConcurrentRuns: 1
  }
}
```

Disable cron: `cron.enabled: false` or `OPENCLAW_SKIP_CRON=1`

## JSON Schema (Tool Calls)

### One-shot (main)
```json
{
  "name": "Reminder",
  "schedule": { "kind": "at", "atMs": 1738262400000 },
  "sessionTarget": "main",
  "wakeMode": "now",
  "payload": { "kind": "systemEvent", "text": "Reminder text" },
  "deleteAfterRun": true
}
```

### Recurring (isolated)
```json
{
  "name": "Morning brief",
  "schedule": { "kind": "cron", "expr": "0 7 * * *", "tz": "America/Los_Angeles" },
  "sessionTarget": "isolated",
  "payload": {
    "kind": "agentTurn",
    "message": "Summarize updates.",
    "deliver": true,
    "channel": "slack",
    "to": "channel:C123"
  }
}
```

## Delivery Targets
- WhatsApp: `+15551234567`
- Discord: `channel:<id>` or `user:<id>`
- Slack: `channel:<id>` or `user:<id>`
- Telegram: `-1001234567890` or `-1001234567890:topic:123`
