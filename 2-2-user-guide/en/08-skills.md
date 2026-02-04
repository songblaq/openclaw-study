# Skills System - Summary

## What Are Skills?
Skill folders teach the agent how to use tools. Each skill has `SKILL.md` with YAML frontmatter + instructions.

## Locations (Precedence)
1. `<workspace>/skills` (highest)
2. `~/.openclaw/skills` (managed/local)
3. Bundled skills (lowest)

Same-name skills: workspace wins.

## Skill Format

```markdown
---
name: nano-banana-pro
description: Generate images via Gemini 3 Pro
metadata: { "openclaw": { "requires": { "bins": ["uv"], "env": ["GEMINI_API_KEY"] }, "primaryEnv": "GEMINI_API_KEY" } }
---

# Instructions here...
```

## Gating (Load-time Filters)

Skills are filtered based on:
- `requires.bins` - binaries on PATH
- `requires.env` - environment variables
- `requires.config` - config paths that must be truthy
- `os` - platform restriction (`darwin`, `linux`, `win32`)

## Config Overrides

```json5
{
  skills: {
    entries: {
      "nano-banana-pro": {
        enabled: true,
        apiKey: "GEMINI_KEY_HERE",
        env: { GEMINI_API_KEY: "KEY" }
      },
      peekaboo: { enabled: true },
      sag: { enabled: false }
    }
  }
}
```

## ClawHub (Skills Registry)

Browse: https://clawhub.com

```bash
# Install skill
clawhub install <skill-slug>

# Update all
clawhub update --all

# Sync (scan + publish)
clawhub sync --all
```

## Multi-Agent Skills
- **Per-agent**: `<workspace>/skills` (agent-specific)
- **Shared**: `~/.openclaw/skills` (all agents)
- Extra folders: `skills.load.extraDirs` in config

## Token Impact
When skills are loaded, they're injected into system prompt:
- Base overhead: ~195 characters
- Per skill: ~97 chars + name + description + location

## Security Notes
- Treat third-party skills as **untrusted code**
- Read skills before enabling
- Prefer sandboxed runs for risky tools
- `env` and `apiKey` inject into host process (not sandbox)

## Skills Watcher
Auto-refresh on `SKILL.md` changes:

```json5
{
  skills: {
    load: {
      watch: true,
      watchDebounceMs: 250
    }
  }
}
```

## Session Snapshot
Skills are snapshotted when session starts. Changes apply on next new session (or hot reload via watcher).
