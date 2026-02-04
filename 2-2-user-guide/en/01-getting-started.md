# Getting Started - Summary

## Goal
Zero → First working chat as quickly as possible.

## Fastest Path
1. **Quick test**: `openclaw dashboard` → chat in browser at `http://127.0.0.1:18789/`
2. **Full setup**: `openclaw onboard --install-daemon` (wizard handles everything)

## Prerequisites
- Node.js ≥ 22
- Brave Search API key (recommended): `openclaw configure --section web`
- Windows: Use **WSL2** (native Windows is untested)

## Installation

```bash
# macOS/Linux
curl -fsSL https://openclaw.ai/install.sh | bash

# Windows PowerShell
iwr -useb https://openclaw.ai/install.ps1 | iex

# Alternative (npm)
npm install -g openclaw@latest
```

## Onboarding Wizard

```bash
openclaw onboard --install-daemon
```

Configures:
- Local vs Remote gateway
- Auth (OAuth or API keys)
- Channels (WhatsApp, Telegram, Discord, etc.)
- Background service (launchd/systemd)
- Gateway token

## Start Gateway

```bash
# Check status
openclaw gateway status

# Manual run (foreground)
openclaw gateway --port 18789 --verbose
```

⚠️ **Bun warning**: WhatsApp and Telegram have issues on Bun. Use **Node**.

## Quick Verify

```bash
openclaw status
openclaw health
openclaw security audit --deep
```

## Connect Chat Surface

### WhatsApp
```bash
openclaw channels login
# Scan QR via WhatsApp → Settings → Linked Devices
```

### Telegram / Discord
- Wizard can write tokens
- Or manual config (see channel docs)

## DM Safety (Pairing)
Default: unknown DMs get a pairing code (not processed until approved).

```bash
openclaw pairing list whatsapp
openclaw pairing approve whatsapp <code>
```

## Test Send

```bash
openclaw message send --target +15555550123 --message "Hello from OpenClaw"
```

## Key Paths
- Config: `~/.openclaw/openclaw.json`
- Workspace: `~/.openclaw/workspace`
- Credentials: `~/.openclaw/credentials/`
- Sessions: `~/.openclaw/agents/<agentId>/sessions/`
- Logs: `/tmp/openclaw/`
