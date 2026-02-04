# OpenClaw Troubleshooting Patterns

Quick reference for diagnosing and resolving common OpenClaw issues.

---

## Table of Contents

1. [Diagnostic Commands](#diagnostic-commands)
2. [Startup Issues](#startup-issues)
3. [Authentication Problems](#authentication-problems)
4. [Channel Issues](#channel-issues)
5. [Agent & Session Issues](#agent--session-issues)
6. [Tool Issues](#tool-issues)
7. [Performance Issues](#performance-issues)
8. [Platform-Specific Issues](#platform-specific-issues)
9. [Common Error Messages](#common-error-messages)
10. [Recovery Procedures](#recovery-procedures)

---

## Diagnostic Commands

### Quick Triage (in order)

| Command | Purpose | When to Use |
|---------|---------|-------------|
| `openclaw status` | Local summary | First check |
| `openclaw status --all` | Full diagnosis (safe to share) | Debug reports |
| `openclaw status --deep` | Gateway health + provider probes | Config looks OK but not working |
| `openclaw gateway status` | Service state, PID, last error | Service "loaded" but not running |
| `openclaw gateway probe` | Gateway discovery + reachability | Wrong gateway target |
| `openclaw channels status --probe` | Channel status from gateway | Gateway up but channels misbehave |
| `openclaw logs --follow` | Live logs | Need actual failure reason |
| `openclaw doctor` | Config validation + fixes | Startup failures |

### Log Locations

| Log Type | Location |
|----------|----------|
| File logs (structured) | `/tmp/openclaw/openclaw-YYYY-MM-DD.log` |
| macOS service | `~/.openclaw/logs/gateway.log` |
| Linux service | `journalctl --user -u openclaw-gateway.service` |
| Session files | `~/.openclaw/agents/<agentId>/sessions/` |
| Credentials | `~/.openclaw/credentials/` |

### Enable Debug Logging

```json5
{
  logging: {
    level: "debug",           // or "trace" for max verbosity
    consoleLevel: "debug",
    consoleStyle: "pretty"
  }
}
```

Or run with verbose flag:
```bash
openclaw gateway --verbose
openclaw channels login --verbose
```

---

## Startup Issues

### Gateway Won't Start — Configuration Invalid

**Symptom:** Gateway refuses to start with config validation errors.

**Cause:** Unknown keys, malformed values, or invalid types in config.

**Fix:**
```bash
openclaw doctor           # See all issues
openclaw doctor --fix     # Apply migrations/repairs
```

### "Gateway start blocked: set gateway.mode=local"

**Symptom:** Gateway refuses to start.

**Cause:** `gateway.mode` is unset or not `local`.

**Fix:**
```bash
openclaw configure                          # Run wizard
# or
openclaw config set gateway.mode local      # Direct set
```

### Service Installed but Nothing Running

**Symptom:** Service appears "loaded" but no process running.

**Diagnosis:**
```bash
openclaw gateway status   # Check runtime state
openclaw doctor           # Check config
openclaw logs --follow    # See failure reason
```

### Port Already in Use (18789)

**Symptom:** `EADDRINUSE` error.

**Diagnosis:**
```bash
openclaw gateway status   # Shows listeners
# or
lsof -nP -iTCP:18789 -sTCP:LISTEN
```

**Fix:** Stop existing process or change port:
```bash
openclaw gateway stop
# or change port in config
```

### Service Running but Port Not Listening

**Symptom:** Service reports running but nothing on port.

**Causes:**
- `gateway.mode` not `local`
- Non-loopback bind without auth configured
- Tailscale bind but no tailnet interface

**Fix for auth:**
```json5
{
  gateway: {
    bind: "lan",
    auth: {
      mode: "token",
      token: "your-token"
    }
  }
}
```

---

## Authentication Problems

### No API Key Found for Provider "anthropic"

**Symptom:** Model requests fail with missing auth.

**Cause:** Agent's auth store is empty (auth is per-agent).

**Fix Options:**
```bash
# Re-run onboarding
openclaw onboard

# Or paste setup-token
openclaw models auth setup-token --provider anthropic

# Or copy auth from another agent
cp ~/.openclaw/agents/main/agent/auth-profiles.json \
   ~/.openclaw/agents/<new-agent>/agent/
```

**Verify:**
```bash
openclaw models status
```

### OAuth Token Refresh Failed (Anthropic Claude Subscription)

**Symptom:** Stored OAuth token expired and refresh failed.

**Fix (recommended):** Switch to setup-token:
```bash
openclaw models auth setup-token --provider anthropic
openclaw models status
```

### Control UI Fails on HTTP ("device identity required")

**Symptom:** Dashboard fails over plain HTTP.

**Cause:** Browser in non-secure context blocks WebCrypto.

**Fix Options:**
1. Use HTTPS via Tailscale Serve
2. Open locally: `http://127.0.0.1:18789/`
3. Enable insecure auth:
   ```json5
   { gateway: { controlUi: { allowInsecureAuth: true } } }
   ```

### All Models Failed

**Checklist:**
1. Credentials present? (`openclaw models status`)
2. Model routing correct? (`agents.defaults.model.primary`)
3. Check logs for exact provider error
4. Use `/model status` in chat

---

## Channel Issues

### WhatsApp Disconnected

**Diagnosis:**
```bash
openclaw status --deep
openclaw logs --limit 200 | grep "connection\|disconnect\|logout"
```

**Fix:** Usually auto-reconnects. If not:
```bash
# Restart gateway
openclaw gateway restart

# If logged out
openclaw channels logout
openclaw channels login --verbose   # Re-scan QR
```

### Messages Not Triggering

**Check 1: Sender allowlisted?**
```bash
openclaw status   # Look for AllowFrom
```

**Check 2: Mention required in groups?**
- Check `agents.list[].groupChat.mentionPatterns`
- Check `channels.<provider>.groups`

**Check 3: Logs**
```bash
openclaw logs --follow | grep "blocked\|skip\|unauthorized"
```

### Pairing Code Not Arriving

**Check:**
```bash
openclaw pairing list <channel>   # Max 3 pending per channel
openclaw logs --follow | grep "pairing request"
```

### Image + Mention Not Working

**Known Issue:** WhatsApp sometimes doesn't include mention metadata with image-only messages.

**Workaround:** Add text with the mention:
- ❌ `@openclaw` + image
- ✅ `@openclaw check this` + image

### Discord Not Replying in Server

**Checklist:**
1. `channels.discord.groupPolicy: "open"` OR guild allowlist configured
2. Use numeric channel IDs in config
3. `requireMention` goes under `guilds`, not top-level
4. Bot has Message Content Intent enabled
5. Run `openclaw channels status --probe`

### Telegram Block Streaming Not Splitting

**Checklist:**
1. `agents.defaults.blockStreamingDefault` is `"on"`
2. `channels.telegram.blockStreaming` not `false`
3. `channels.telegram.streamMode: "off"` (if you want multi-message)
4. Chunk/coalesce settings not too high

---

## Agent & Session Issues

### Session Not Resuming

**Check 1: Session file exists?**
```bash
ls -la ~/.openclaw/agents/<agentId>/sessions/
```

**Check 2: Reset window too short?**
```json5
{
  session: {
    reset: {
      mode: "daily",
      atHour: 4,
      idleMinutes: 10080  // 7 days
    }
  }
}
```

**Check 3:** Did someone send `/new` or reset trigger?

### Agent Was Aborted

**Causes:**
- User sent `stop`, `abort`, `esc`, `wait`, or `exit`
- Timeout exceeded
- Process crashed

**Fix:** Just send another message. Session continues.

### Agent Timing Out

**Default:** 30 minutes.

**Fix for long tasks:**
```json5
{
  reply: { timeoutSeconds: 3600 }  // 1 hour
}
```

Or use `process` tool to background long commands.

### Main Chat Running in Sandbox Workspace

**Symptom:** `pwd` shows `~/.openclaw/sandboxes/...`

**Cause:** `sandbox.mode: "non-main"` keys off `session.mainKey`. Groups/channels use their own keys.

**Fix:**
- Set `agents.list[].sandbox.mode: "off"` for host workspace
- Or set `workspaceAccess: "rw"` for host workspace access in sandbox

### "Agent failed before reply: Unknown model"

**Cause:** Model name no longer supported (older/insecure models rejected).

**Fix:**
```bash
openclaw models list   # See available models
openclaw models scan   # Scan for supported models
```

---

## Tool Issues

### Skill Missing API Key in Sandbox

**Symptom:** Skill works on host but fails in sandbox.

**Cause:** Sandboxed exec doesn't inherit host `process.env`.

**Fix:**
```json5
{
  agents: {
    defaults: {
      sandbox: {
        docker: {
          env: { MY_API_KEY: "..." }
        }
      }
    }
  }
}
```

Then recreate sandbox:
```bash
openclaw sandbox recreate --agent <id>
```

### Media Send Failing

**Check 1:** File path valid?
```bash
ls -la /path/to/file
```

**Check 2:** File too large?
- Images: max 6MB
- Audio/Video: max 16MB
- Documents: max 100MB

**Check 3:** Logs
```bash
grep "media\|fetch\|download" /tmp/openclaw/openclaw-*.log | tail -20
```

### Cloud Code Assist API Error: Invalid Tool Schema

**Cause:** Tool schema compatibility issue with strict endpoint.

**Fix:**
1. Update OpenClaw to latest
2. Avoid unsupported JSON Schema keywords:
   - `anyOf/oneOf/allOf`
   - `patternProperties`
   - `additionalProperties`
   - `minLength/maxLength`
   - `format`

---

## Performance Issues

### High Memory Usage

**Cause:** Conversation history in memory.

**Fix:**
```json5
{
  session: {
    historyLimit: 100   // Max messages to keep
  }
}
```

Or restart periodically.

### Service Environment (PATH Issues)

**Symptom:** Commands not found in service.

**Cause:** Service runs with minimal PATH, excludes version managers.

**Paths included:**
- macOS: `/opt/homebrew/bin`, `/usr/local/bin`, `/usr/bin`, `/bin`
- Linux: `/usr/local/bin`, `/usr/bin`, `/bin`

**Fix:** Set env in `~/.openclaw/.env` or use `tools.exec.pathPrepend`.

---

## Platform-Specific Issues

### macOS: App Crashes on Permission Prompts

**Fix 1:** Reset TCC cache:
```bash
tccutil reset All bot.molt.mac.debug
```

**Fix 2:** Change bundle ID in build script and rebuild.

### macOS: Gateway Stuck on "Starting..."

**Check port:**
```bash
lsof -nP -iTCP:18789 -sTCP:LISTEN
```

**Stop supervisor:**
```bash
openclaw gateway stop
```

### Linux: Browser Not Starting

**Symptom:** `"Failed to start Chrome CDP on port 18800"`

**Cause:** Snap-packaged Chromium on Ubuntu.

**Fix:** Install Google Chrome:
```bash
wget https://dl.google.com/linux/direct/google-chrome-stable_current_amd64.deb
sudo dpkg -i google-chrome-stable_current_amd64.deb
```

Then configure:
```json5
{ browser: { executablePath: "/usr/bin/google-chrome-stable" } }
```

---

## Common Error Messages

| Error | Cause | Quick Fix |
|-------|-------|-----------|
| `No API key found for provider` | Missing auth for provider | `openclaw models auth setup-token --provider <name>` |
| `Gateway start blocked: set gateway.mode=local` | Mode not configured | `openclaw config set gateway.mode local` |
| `Address already in use` | Port conflict | `openclaw gateway stop` or change port |
| `Agent was aborted` | User interrupt or timeout | Send another message |
| `Unknown model` | Deprecated/unsupported model | Update to supported model |
| `Configuration invalid` | Bad config keys/values | `openclaw doctor --fix` |
| `device identity required` | HTTP without secure context | Use HTTPS or localhost |
| `OAuth token refresh failed` | Expired OAuth | Use setup-token instead |

---

## Recovery Procedures

### Soft Reset

```bash
openclaw gateway restart
```

### Medium Reset

```bash
openclaw gateway stop
openclaw doctor --fix
openclaw gateway start
```

### Hard Reset (Keeps Sessions)

```bash
openclaw gateway stop
openclaw gateway uninstall
openclaw doctor --fix
openclaw gateway install
openclaw gateway start
```

### Nuclear Reset (Loses Everything)

⚠️ **Warning:** Loses all sessions, requires re-pairing channels.

```bash
openclaw gateway stop
openclaw gateway uninstall
trash ~/.openclaw
openclaw onboard             # Fresh setup
openclaw channels login      # Re-pair WhatsApp
openclaw gateway start
```

---

## Quick Diagnostic Checklist

When something breaks:

1. [ ] `openclaw status` — Quick overview
2. [ ] `openclaw gateway status` — Service running?
3. [ ] `openclaw doctor` — Config valid?
4. [ ] `openclaw logs --follow` — What's the actual error?
5. [ ] `openclaw models status` — Auth working?
6. [ ] `openclaw channels status --probe` — Channels connected?

If still stuck:
1. Search GitHub issues
2. Check Discord community
3. Share `openclaw status --all` (redacted)

---

*Last updated: 2026-02-04*
