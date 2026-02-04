# Discord Integration - Summary

## Quick Setup
1. Create Discord bot in Developer Portal
2. Enable **Message Content Intent** + **Server Members Intent**
3. Set token: `DISCORD_BOT_TOKEN=...` or config
4. Invite bot to server with message permissions
5. Start gateway

## Minimal Config

```json5
{
  channels: {
    discord: {
      enabled: true,
      token: "YOUR_BOT_TOKEN"
    }
  }
}
```

## Session Keys
- DMs: `agent:<agentId>:main` (shared main session)
- Guilds: `agent:<agentId>:discord:channel:<channelId>`

## DM Access Control

```json5
{
  channels: {
    discord: {
      dm: {
        enabled: true,
        policy: "pairing",  // pairing | allowlist | open | disabled
        allowFrom: ["123456789012345678", "username"]
      }
    }
  }
}
```

## Guild Configuration

```json5
{
  channels: {
    discord: {
      guilds: {
        "GUILD_ID": {
          requireMention: true,
          users: ["USER_ID"],
          channels: {
            help: { 
              allow: true, 
              requireMention: true,
              skills: ["docs"],
              systemPrompt: "Short answers only."
            }
          }
        }
      }
    }
  }
}
```

## Mention Gating
- `requireMention: true` (default): bot only replies when @mentioned
- Use `mentionPatterns` for text-based triggers

## Tool Actions
| Action | Default | Description |
|--------|---------|-------------|
| reactions | enabled | React + list reactions |
| messages | enabled | Read/send/edit/delete |
| threads | enabled | Create/list/reply |
| pins | enabled | Pin/unpin/list |
| moderation | disabled | Timeout/kick/ban |

## Reply Threading
- Off by default
- Enable: `channels.discord.replyToMode: "first"` or `"all"`
- Use `[[reply_to_current]]` or `[[reply_to:<id>]]` tags

## Limits
- Text chunk: 2000 chars (configurable)
- Max lines per message: 17
- Media: 8 MB default

## Troubleshooting
- "Used disallowed intents": Enable Message Content Intent
- No replies: Check bot permissions + allowlists
- Run `openclaw doctor` and `openclaw channels status --probe`
