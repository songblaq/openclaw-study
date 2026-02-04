# Setup Guide - Summary

## Core Principle
Keep customization **outside** the repo:
- Config: `~/.openclaw/openclaw.json`
- Workspace: `~/.openclaw/workspace`

## Bootstrap

```bash
openclaw setup
```

## Two Workflows

### 1. Stable Workflow (macOS App)
1. Install + launch OpenClaw.app
2. Complete onboarding permissions
3. Gateway runs in Local mode (app manages it)
4. Link channels: `openclaw channels login`
5. Verify: `openclaw health`

### 2. Bleeding Edge (Terminal Gateway)
For TypeScript development with hot reload:

```bash
pnpm install
pnpm gateway:watch
```

Point macOS app to Local mode → attaches to running gateway.

## Credential Storage Map

| Item | Location |
|------|----------|
| WhatsApp | `~/.openclaw/credentials/whatsapp/<accountId>/creds.json` |
| Telegram bot token | config/env or `channels.telegram.tokenFile` |
| Discord bot token | config/env |
| Slack tokens | config/env |
| Pairing allowlists | `~/.openclaw/credentials/<channel>-allowFrom.json` |
| Model auth profiles | `~/.openclaw/agents/<agentId>/agent/auth-profiles.json` |
| Legacy OAuth | `~/.openclaw/credentials/oauth.json` |

## Linux Note
Linux uses systemd **user** service. Enable lingering to prevent gateway stop on logout:

```bash
sudo loginctl enable-linger $USER
```

## Updating
- Keep `~/.openclaw/` as "your stuff"
- Update source: `git pull` + `pnpm install`
- Personal prompts/config stay outside the repo
