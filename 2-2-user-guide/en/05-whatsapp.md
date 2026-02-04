# WhatsApp Integration - Summary

## Quick Setup
1. Use a **separate phone number** (recommended)
2. Configure in `~/.openclaw/openclaw.json`
3. Run `openclaw channels login` → scan QR
4. Start gateway

## Minimal Config

```json5
{
  channels: {
    whatsapp: {
      dmPolicy: "allowlist",
      allowFrom: ["+15551234567"]
    }
  }
}
```

## Phone Number Options

### Dedicated Number (Recommended)
- Spare/old Android phone + eSIM
- Can use WhatsApp Business on same device

### Personal Number (Fallback)
- Enable self-chat mode
- Message yourself for testing

```json5
{
  channels: {
    whatsapp: {
      selfChatMode: true,
      dmPolicy: "allowlist",
      allowFrom: ["+15551234567"]
    }
  }
}
```

## Login/Logout

```bash
# Login (scan QR)
openclaw channels login
openclaw channels login --account <id>  # multi-account

# Logout
openclaw channels logout
```

Credentials: `~/.openclaw/credentials/whatsapp/<accountId>/creds.json`

## DM Policies
| Policy | Behavior |
|--------|----------|
| `pairing` (default) | Unknown senders get code, must approve |
| `allowlist` | Only allow `allowFrom` list |
| `open` | Allow all (requires `["*"]`) |
| `disabled` | Ignore all DMs |

## Group Configuration

```json5
{
  channels: {
    whatsapp: {
      groupPolicy: "allowlist",
      groups: {
        "*": { requireMention: true }
      }
    }
  },
  agents: {
    list: [{
      id: "main",
      groupChat: { mentionPatterns: ["@openclaw", "reisponde"] }
    }]
  }
}
```

## Multi-Account

```json5
{
  channels: {
    whatsapp: {
      accounts: {
        default: {},
        personal: {},
        biz: {}
      }
    }
  }
}
```

## Auto-Acknowledge Reactions

```json5
{
  channels: {
    whatsapp: {
      ackReaction: {
        emoji: "👀",
        direct: true,
        group: "mentions"  // always | mentions | never
      }
    }
  }
}
```

## Limits
- Text chunk: 4000 chars
- Inbound media: 50 MB
- Outbound media: 5 MB (auto-optimized)

## Troubleshooting

**Not linked**
```bash
openclaw channels login  # scan QR again
```

**Linked but disconnected**
```bash
openclaw doctor  # or restart gateway
```

**Bun runtime issues**
- WhatsApp (Baileys) unreliable on Bun
- Use **Node** for gateway
