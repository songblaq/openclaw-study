# Cron vs Heartbeat - Summary

## Quick Decision

| Use Case | Recommended |
|----------|-------------|
| Check inbox every 30 min | Heartbeat |
| Daily report at 9am sharp | Cron (isolated) |
| Monitor calendar | Heartbeat |
| Weekly deep analysis | Cron (isolated) |
| Remind me in 20 minutes | Cron (`--at`) |
| Project health check | Heartbeat |

## Heartbeat: Periodic Awareness

Runs in **main session** at regular intervals (default: 30 min).

### When to Use
- Multiple periodic checks (batch inbox + calendar + weather)
- Need conversational context
- Timing can drift slightly
- Want to reduce API calls

### Advantages
- Batches multiple checks in one turn
- Context-aware decisions
- Smart suppression (`HEARTBEAT_OK`)
- Lower overhead

### Example: HEARTBEAT.md
```markdown
# Heartbeat checklist
- Check email for urgent messages
- Review calendar for events in next 2 hours
- If background task finished, summarize
```

### Config
```json5
{
  agents: {
    defaults: {
      heartbeat: {
        every: "30m",
        target: "last",
        activeHours: { start: "08:00", end: "22:00" }
      }
    }
  }
}
```

## Cron: Precise Scheduling

Runs at **exact times** in isolated or main sessions.

### When to Use
- Exact timing required
- Standalone tasks (no context needed)
- Different model/thinking settings
- One-shot reminders
- Tasks that would clutter main history

### Advantages
- Exact timing (cron expressions + timezone)
- Session isolation
- Model overrides
- Delivery control

### Example
```bash
openclaw cron add \
  --name "Morning briefing" \
  --cron "0 7 * * *" \
  --tz "America/New_York" \
  --session isolated \
  --message "Today's briefing" \
  --model opus \
  --deliver
```

## Decision Flowchart

```
Exact time needed?
  YES → Cron
  NO  ↓
Session isolation needed?
  YES → Cron (isolated)
  NO  ↓
Can batch with other checks?
  YES → Heartbeat
  NO  → Cron
```

## Combining Both (Recommended)

**Heartbeat** (every 30 min):
- Inbox scan
- Calendar check
- Pending tasks review

**Cron** (precise timing):
- `0 7 * * *` - Morning brief
- `0 9 * * 1` - Weekly review (Mondays)
- One-shot reminders

## Session Comparison

|  | Heartbeat | Cron (main) | Cron (isolated) |
|--|-----------|-------------|-----------------|
| Session | Main | Main (via event) | `cron:<jobId>` |
| History | Shared | Shared | Fresh each run |
| Context | Full | Full | None |
| Model | Main session | Main session | Can override |

## Cost Tips
- Keep HEARTBEAT.md small
- Batch similar checks into heartbeat
- Use cheaper model for routine cron tasks
- Use `target: "none"` for internal processing only
