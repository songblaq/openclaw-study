# Gateway Configuration - Summary

## Config File
`~/.openclaw/openclaw.json` (JSON5 format - comments + trailing commas allowed)

## Validation
- Strict schema validation on startup
- Unknown keys → Gateway refuses to start
- Run `openclaw doctor` to diagnose
- Run `openclaw doctor --fix` to apply migrations

## Minimal Config

```json5
{
  agents: { defaults: { workspace: "~/.openclaw/workspace" } },
  channels: { whatsapp: { allowFrom: ["+15555550123"] } },
}
```

## Key Configuration Sections

### Environment Variables
```json5
{
  env: {
    OPENROUTER_API_KEY: "sk-or-...",
    vars: { GROQ_API_KEY: "gsk-..." }
  }
}
```

Substitution in config: `${VAR_NAME}` syntax.

### Logging
```json5
{
  logging: {
    level: "info",
    file: "/tmp/openclaw/openclaw.log",
    consoleLevel: "info",
    redactSensitive: "tools"
  }
}
```

### Agent Identity
```json5
{
  agents: {
    list: [{
      id: "main",
      identity: {
        name: "Samantha",
        theme: "helpful sloth",
        emoji: "🦥",
        avatar: "avatars/samantha.png"
      }
    }]
  }
}
```

### Multi-Agent Routing
```json5
{
  agents: {
    list: [
      { id: "home", default: true, workspace: "~/.openclaw/workspace-home" },
      { id: "work", workspace: "~/.openclaw/workspace-work" }
    ]
  },
  bindings: [
    { agentId: "home", match: { channel: "whatsapp", accountId: "personal" } },
    { agentId: "work", match: { channel: "whatsapp", accountId: "biz" } }
  ]
}
```

### Message Queue
```json5
{
  messages: {
    queue: {
      mode: "collect",  // steer | followup | collect | interrupt
      debounceMs: 1000,
      cap: 20
    }
  }
}
```

### Response Prefix
```json5
{
  messages: {
    responsePrefix: "[{model} | think:{thinkingLevel}]",
    ackReaction: "👀",
    ackReactionScope: "group-mentions"
  }
}
```

## DM Policies

| Policy | Behavior |
|--------|----------|
| `pairing` | Unknown senders get pairing code (default) |
| `allowlist` | Only allow senders in `allowFrom` |
| `open` | Allow all (requires `allowFrom: ["*"]`) |
| `disabled` | Ignore all |

## Group Policies
- `open`: groups bypass allowlists
- `disabled`: block all group messages
- `allowlist`: only allow configured groups (default)

## Config RPC

```bash
# Get current config
openclaw gateway call config.get --params '{}'

# Patch config (merge update)
openclaw gateway call config.patch --params '{
  "raw": "{ channels: { telegram: { groups: { \"*\": { requireMention: false } } } } }",
  "baseHash": "<hash>"
}'
```

## Config Includes
Split config into multiple files:

```json5
{
  gateway: { port: 18789 },
  agents: { $include: "./agents.json5" },
  broadcast: { $include: ["./client1.json5", "./client2.json5"] }
}
```
