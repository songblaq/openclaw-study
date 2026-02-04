# OpenClaw Configuration Reference

Complete reference for OpenClaw configuration options. For AI agents to quickly look up settings and their purposes.

---

## Table of Contents

1. [Configuration Basics](#configuration-basics)
2. [Environment Variables](#environment-variables)
3. [Gateway Configuration](#gateway-configuration)
4. [Agent Configuration](#agent-configuration)
5. [Channel Configuration](#channel-configuration)
6. [Tool Configuration](#tool-configuration)
7. [Model Configuration](#model-configuration)
8. [Session Configuration](#session-configuration)
9. [Message Configuration](#message-configuration)
10. [Skills & Plugins](#skills--plugins)
11. [Automation (Cron, Hooks)](#automation-cron-hooks)
12. [Quick Reference Tables](#quick-reference-tables)

---

## Configuration Basics

### File Location
- Primary: `~/.openclaw/openclaw.json`
- Format: JSON5 (comments + trailing commas allowed)
- Override: `OPENCLAW_CONFIG_PATH` env var

### Config Includes
Split configs using `$include`:
```json5
{
  agents: { $include: "./agents.json5" },
  broadcast: { $include: ["./client1.json5", "./client2.json5"] }
}
```

### Env Var Substitution
Reference env vars in config strings:
```json5
{
  gateway: { auth: { token: "${OPENCLAW_GATEWAY_TOKEN}" } }
}
```

### Validation
- Strict schema validation on startup
- Unknown keys cause startup failure
- Run `openclaw doctor` to diagnose issues

---

## Environment Variables

### Precedence (highest → lowest)
1. Process environment
2. `.env` in current working directory
3. Global `.env` at `~/.openclaw/.env`
4. Config `env` block
5. Shell env import (`env.shellEnv.enabled`)

### Key Environment Variables

| Variable | Purpose |
|----------|---------|
| `OPENCLAW_CONFIG_PATH` | Override config file location |
| `OPENCLAW_STATE_DIR` | State directory (default: `~/.openclaw`) |
| `OPENCLAW_GATEWAY_PORT` | Gateway port override |
| `OPENCLAW_GATEWAY_TOKEN` | Gateway auth token |
| `OPENCLAW_GATEWAY_PASSWORD` | Gateway password |
| `OPENCLAW_AGENT_DIR` | Agent directory override |
| `OPENCLAW_LOAD_SHELL_ENV` | Enable shell env import |

### Provider API Keys

| Variable | Provider |
|----------|----------|
| `ANTHROPIC_API_KEY` | Anthropic |
| `OPENAI_API_KEY` | OpenAI |
| `OPENROUTER_API_KEY` | OpenRouter |
| `GOOGLE_API_KEY` / `GEMINI_API_KEY` | Google |
| `GROQ_API_KEY` | Groq |
| `ZAI_API_KEY` | Z.AI |
| `MINIMAX_API_KEY` | MiniMax |
| `BRAVE_API_KEY` | Brave Search |
| `ELEVENLABS_API_KEY` | ElevenLabs TTS |
| `FIRECRAWL_API_KEY` | Firecrawl |

### Channel Tokens

| Variable | Channel |
|----------|---------|
| `TELEGRAM_BOT_TOKEN` | Telegram |
| `DISCORD_BOT_TOKEN` | Discord |
| `SLACK_BOT_TOKEN` | Slack |
| `SLACK_APP_TOKEN` | Slack Socket Mode |

---

## Gateway Configuration

### `gateway`

```json5
{
  gateway: {
    mode: "local",           // "local" | "remote"
    port: 18789,             // WS + HTTP port
    bind: "loopback",        // "loopback" | "lan" | "tailnet"
    
    auth: {
      mode: "token",         // "token" | "password"
      token: "your-token",
      password: "your-password",
      allowTailscale: true
    },
    
    controlUi: {
      enabled: true,
      basePath: "/openclaw"
    },
    
    tailscale: {
      mode: "off",           // "off" | "serve" | "funnel"
      resetOnExit: true
    },
    
    reload: {
      mode: "hybrid",        // "hybrid" | "hot" | "restart" | "off"
      debounceMs: 300
    },
    
    remote: {
      url: "ws://gateway.tailnet:18789",
      transport: "ssh",      // "ssh" | "direct"
      token: "token"
    }
  }
}
```

### Port Mapping
| Service | Default Port |
|---------|-------------|
| Gateway (WS + HTTP) | 18789 |
| Bridge (legacy) | 18790 |
| Control UI | 18791 |
| Browser CDP | 18792 |
| Canvas Host | 18793 |

---

## Agent Configuration

### `agents.defaults`

```json5
{
  agents: {
    defaults: {
      workspace: "~/.openclaw/workspace",
      repoRoot: "~/Projects/myrepo",
      skipBootstrap: false,
      bootstrapMaxChars: 20000,
      userTimezone: "America/Chicago",
      timeFormat: "auto",          // "auto" | "12" | "24"
      
      // Model settings
      model: {
        primary: "anthropic/claude-opus-4-5",
        fallbacks: ["openrouter/deepseek/deepseek-r1:free"]
      },
      imageModel: {
        primary: "openrouter/qwen/qwen-2.5-vl-72b-instruct:free"
      },
      
      // Runtime defaults
      thinkingDefault: "low",      // "off" | "low" | "high"
      verboseDefault: "off",
      elevatedDefault: "on",
      timeoutSeconds: 600,
      mediaMaxMb: 5,
      maxConcurrent: 3,
      contextTokens: 200000,
      
      // Model catalog with aliases
      models: {
        "anthropic/claude-opus-4-5": { alias: "opus" },
        "anthropic/claude-sonnet-4-5": { alias: "sonnet" },
        "openai/gpt-5.2": { alias: "gpt", params: { maxTokens: 8192 } }
      },
      
      // Heartbeat
      heartbeat: {
        every: "30m",
        target: "last",
        model: "openai/gpt-5.2-mini",
        prompt: "Read HEARTBEAT.md...",
        includeReasoning: false
      },
      
      // Sub-agents
      subagents: {
        model: "minimax/MiniMax-M2.1",
        maxConcurrent: 1,
        archiveAfterMinutes: 60
      },
      
      // Exec defaults
      exec: {
        backgroundMs: 10000,
        timeoutSec: 1800,
        cleanupMs: 1800000,
        notifyOnExit: true
      },
      
      // Streaming
      blockStreamingDefault: "off",
      blockStreamingBreak: "text_end",
      blockStreamingChunk: { minChars: 800, maxChars: 1200 },
      humanDelay: { mode: "off" },   // "off" | "natural" | "custom"
      
      // Typing indicators
      typingMode: "instant",         // "never" | "instant" | "thinking" | "message"
      typingIntervalSeconds: 6
    }
  }
}
```

### `agents.list[]` (Multi-Agent)

```json5
{
  agents: {
    list: [
      {
        id: "main",
        default: true,
        name: "Main Agent",
        workspace: "~/.openclaw/workspace",
        agentDir: "~/.openclaw/agents/main/agent",
        
        model: "anthropic/claude-opus-4-5",  // or { primary, fallbacks }
        
        identity: {
          name: "Samantha",
          theme: "helpful sloth",
          emoji: "🦥",
          avatar: "avatars/samantha.png"
        },
        
        groupChat: {
          mentionPatterns: ["@openclaw", "reisponde"]
        },
        
        sandbox: {
          mode: "off",
          scope: "agent",
          workspaceAccess: "rw"
        },
        
        tools: {
          profile: "coding",
          allow: ["read", "write", "exec"],
          deny: ["browser"]
        },
        
        subagents: {
          allowAgents: ["*"]        // or specific agent ids
        }
      }
    ]
  }
}
```

### `bindings[]` (Agent Routing)

```json5
{
  bindings: [
    { 
      agentId: "work", 
      match: { 
        channel: "whatsapp", 
        accountId: "biz",
        peer: { kind: "dm", id: "+15551234567" },
        guildId: "123456789012345678"
      } 
    }
  ]
}
```

### Context Pruning

```json5
{
  agents: {
    defaults: {
      contextPruning: {
        mode: "adaptive",            // "off" | "adaptive" | "aggressive"
        keepLastAssistants: 3,
        softTrimRatio: 0.3,
        hardClearRatio: 0.5,
        minPrunableToolChars: 50000,
        softTrim: { maxChars: 4000, headChars: 1500, tailChars: 1500 },
        hardClear: { enabled: true, placeholder: "[Old tool result content cleared]" },
        tools: { deny: ["browser", "canvas"] }
      }
    }
  }
}
```

### Compaction

```json5
{
  agents: {
    defaults: {
      compaction: {
        mode: "default",             // "default" | "safeguard"
        reserveTokensFloor: 20000,
        memoryFlush: {
          enabled: true,
          softThresholdTokens: 4000,
          systemPrompt: "Session nearing compaction...",
          prompt: "Write lasting notes to memory/YYYY-MM-DD.md..."
        }
      }
    }
  }
}
```

### Sandbox Configuration

```json5
{
  agents: {
    defaults: {
      sandbox: {
        mode: "non-main",            // "off" | "non-main" | "all"
        scope: "agent",              // "session" | "agent" | "shared"
        workspaceAccess: "none",     // "none" | "ro" | "rw"
        workspaceRoot: "~/.openclaw/sandboxes",
        
        docker: {
          image: "openclaw-sandbox:bookworm-slim",
          containerPrefix: "openclaw-sbx-",
          workdir: "/workspace",
          readOnlyRoot: true,
          tmpfs: ["/tmp", "/var/tmp", "/run"],
          network: "none",           // "none" | "bridge"
          user: "1000:1000",
          capDrop: ["ALL"],
          env: { LANG: "C.UTF-8" },
          setupCommand: "apt-get update && apt-get install -y git",
          pidsLimit: 256,
          memory: "1g",
          cpus: 1
        },
        
        browser: {
          enabled: false,
          image: "openclaw-sandbox-browser:bookworm-slim",
          cdpPort: 9222,
          headless: false,
          enableNoVnc: true,
          autoStart: true
        },
        
        prune: {
          idleHours: 24,
          maxAgeDays: 7
        }
      }
    }
  }
}
```

---

## Channel Configuration

### Common Channel Options

| Option | Description |
|--------|-------------|
| `enabled` | Enable/disable channel |
| `dmPolicy` | `"pairing"` \| `"allowlist"` \| `"open"` \| `"disabled"` |
| `allowFrom` | DM allowlist (E.164 for WhatsApp) |
| `groupPolicy` | `"open"` \| `"disabled"` \| `"allowlist"` |
| `groupAllowFrom` | Group sender allowlist |
| `historyLimit` | Messages to include as context |
| `mediaMaxMb` | Max inbound media size |
| `textChunkLimit` | Outbound text chunk size |
| `chunkMode` | `"length"` \| `"newline"` |
| `replyToMode` | `"off"` \| `"first"` \| `"all"` |

### `channels.whatsapp`

```json5
{
  channels: {
    whatsapp: {
      dmPolicy: "pairing",
      allowFrom: ["+15555550123"],
      sendReadReceipts: true,
      textChunkLimit: 4000,
      
      groups: {
        "*": { requireMention: true },
        "120363403215116621@g.us": {
          allowFrom: ["@admin"],
          systemPrompt: "Keep answers brief."
        }
      },
      
      // Multi-account
      accounts: {
        default: {},
        biz: {}
      }
    }
  }
}
```

### `channels.telegram`

```json5
{
  channels: {
    telegram: {
      botToken: "123456:ABC...",
      dmPolicy: "pairing",
      allowFrom: ["tg:123456789"],
      historyLimit: 50,
      replyToMode: "first",
      linkPreview: true,
      streamMode: "partial",         // "off" | "partial" | "block"
      
      groups: {
        "*": { requireMention: true },
        "-1001234567890": {
          topics: {
            "99": { requireMention: false }
          }
        }
      },
      
      customCommands: [
        { command: "backup", description: "Git backup" }
      ],
      
      actions: { reactions: true, sendMessage: true },
      reactionNotifications: "own"   // "off" | "own" | "all"
    }
  }
}
```

### `channels.discord`

```json5
{
  channels: {
    discord: {
      token: "your-bot-token",
      mediaMaxMb: 8,
      allowBots: false,
      replyToMode: "off",
      historyLimit: 20,
      textChunkLimit: 2000,
      maxLinesPerMessage: 17,
      
      dm: {
        enabled: true,
        policy: "pairing",
        allowFrom: ["1234567890"],
        groupEnabled: false
      },
      
      guilds: {
        "123456789012345678": {
          slug: "my-server",
          requireMention: false,
          reactionNotifications: "own",
          users: ["987654321098765432"],
          channels: {
            general: { allow: true },
            help: { allow: true, requireMention: true }
          }
        }
      },
      
      actions: {
        reactions: true,
        stickers: true,
        polls: true,
        messages: true,
        threads: true,
        pins: true
      }
    }
  }
}
```

### `channels.slack`

```json5
{
  channels: {
    slack: {
      botToken: "xoxb-...",
      appToken: "xapp-...",
      historyLimit: 50,
      allowBots: false,
      reactionNotifications: "own",
      replyToMode: "off",
      textChunkLimit: 4000,
      
      dm: {
        enabled: true,
        policy: "pairing",
        allowFrom: ["U123"],
        groupEnabled: false
      },
      
      channels: {
        "#general": { allow: true, requireMention: true }
      },
      
      thread: {
        historyScope: "thread",      // "thread" | "channel"
        inheritParent: false
      },
      
      slashCommand: {
        enabled: true,
        name: "openclaw",
        ephemeral: true
      }
    }
  }
}
```

### `channels.signal`

```json5
{
  channels: {
    signal: {
      dmPolicy: "pairing",
      allowFrom: ["+15555550123"],
      historyLimit: 50,
      reactionNotifications: "own",
      reactionAllowlist: ["+15551234567"]
    }
  }
}
```

### `channels.imessage`

```json5
{
  channels: {
    imessage: {
      enabled: true,
      cliPath: "imsg",
      dbPath: "~/Library/Messages/chat.db",
      dmPolicy: "pairing",
      allowFrom: ["+15555550123", "chat_id:123"],
      historyLimit: 50,
      includeAttachments: false,
      mediaMaxMb: 16,
      service: "auto",
      region: "US"
    }
  }
}
```

### `channels.googlechat`

```json5
{
  channels: {
    googlechat: {
      enabled: true,
      serviceAccountFile: "/path/to/service-account.json",
      audienceType: "app-url",
      audience: "https://gateway.example.com/googlechat",
      webhookPath: "/googlechat",
      
      dm: {
        enabled: true,
        policy: "pairing"
      },
      
      groupPolicy: "allowlist",
      groups: {
        "spaces/AAAA": { allow: true, requireMention: true }
      }
    }
  }
}
```

### `channels.mattermost`

```json5
{
  channels: {
    mattermost: {
      enabled: true,
      botToken: "mm-token",
      baseUrl: "https://chat.example.com",
      dmPolicy: "pairing",
      chatmode: "oncall",            // "oncall" | "onmessage" | "onchar"
      oncharPrefixes: [">", "!"]
    }
  }
}
```

---

## Tool Configuration

### `tools`

```json5
{
  tools: {
    // Base tool profile
    profile: "coding",               // "minimal" | "coding" | "messaging" | "full"
    
    // Global allow/deny
    allow: ["read", "write", "exec"],
    deny: ["browser", "canvas"],
    
    // Per-provider restrictions
    byProvider: {
      "google-antigravity": { profile: "minimal" },
      "openai/gpt-5.2": { allow: ["group:fs", "sessions_list"] }
    },
    
    // Sandbox tool policy
    sandbox: {
      tools: {
        allow: ["exec", "process", "read", "write", "edit"],
        deny: ["browser", "canvas", "nodes", "cron"]
      }
    },
    
    // Elevated exec
    elevated: {
      enabled: true,
      allowFrom: {
        whatsapp: ["+15555550123"],
        discord: ["steipete", "1234567890123"]
      }
    },
    
    // Agent-to-agent messaging
    agentToAgent: {
      enabled: false,
      allow: ["home", "work"]
    },
    
    // Exec settings
    exec: {
      backgroundMs: 10000,
      timeoutSec: 1800,
      cleanupMs: 1800000,
      notifyOnExit: true,
      applyPatch: { enabled: false }
    },
    
    // Web tools
    web: {
      search: {
        enabled: true,
        apiKey: "BRAVE_API_KEY",
        maxResults: 5,
        timeoutSeconds: 30,
        cacheTtlMinutes: 15
      },
      fetch: {
        enabled: true,
        maxChars: 50000,
        timeoutSeconds: 30,
        readability: true,
        firecrawl: {
          enabled: true,
          apiKey: "FIRECRAWL_API_KEY"
        }
      }
    },
    
    // Media understanding
    media: {
      concurrency: 2,
      image: {
        enabled: true,
        maxBytes: 10485760,
        timeoutSeconds: 60,
        models: [{ provider: "openai", model: "gpt-4o-mini" }]
      },
      audio: {
        enabled: true,
        maxBytes: 20971520,
        models: [{ provider: "openai", model: "gpt-4o-mini-transcribe" }]
      },
      video: {
        enabled: true,
        maxBytes: 52428800,
        models: [{ provider: "google", model: "gemini-3-flash-preview" }]
      }
    }
  }
}
```

### Tool Groups

| Group | Tools |
|-------|-------|
| `group:runtime` | `exec`, `bash`, `process` |
| `group:fs` | `read`, `write`, `edit`, `apply_patch` |
| `group:sessions` | `sessions_list`, `sessions_history`, `sessions_send`, `sessions_spawn`, `session_status` |
| `group:memory` | `memory_search`, `memory_get` |
| `group:web` | `web_search`, `web_fetch` |
| `group:ui` | `browser`, `canvas` |
| `group:automation` | `cron`, `gateway` |
| `group:messaging` | `message` |
| `group:nodes` | `nodes` |
| `group:openclaw` | All built-in tools |

### Tool Profiles

| Profile | Description |
|---------|-------------|
| `minimal` | `session_status` only |
| `coding` | `group:fs`, `group:runtime`, `group:sessions`, `group:memory`, `image` |
| `messaging` | `group:messaging`, session tools |
| `full` | No restriction |

---

## Model Configuration

### `models.providers`

```json5
{
  models: {
    mode: "merge",                   // "merge" | "replace"
    providers: {
      "custom-proxy": {
        baseUrl: "http://localhost:4000/v1",
        apiKey: "LITELLM_KEY",
        api: "openai-completions",   // or "openai-responses", "anthropic-messages", "google-generative-ai"
        models: [
          {
            id: "llama-3.1-8b",
            name: "Llama 3.1 8B",
            reasoning: false,
            input: ["text"],
            cost: { input: 0, output: 0 },
            contextWindow: 128000,
            maxTokens: 32000
          }
        ]
      }
    }
  }
}
```

### Built-in Providers

| Provider | Env Variable | Notes |
|----------|--------------|-------|
| `anthropic` | `ANTHROPIC_API_KEY` | Claude models |
| `openai` | `OPENAI_API_KEY` | GPT models |
| `openrouter` | `OPENROUTER_API_KEY` | Multi-provider gateway |
| `google` / `gemini` | `GOOGLE_API_KEY` | Gemini models |
| `groq` | `GROQ_API_KEY` | Fast inference |
| `zai` | `ZAI_API_KEY` | Z.AI GLM models |
| `minimax` | `MINIMAX_API_KEY` | MiniMax M2.1 |
| `opencode` | `OPENCODE_API_KEY` | OpenCode Zen |
| `kimi-coding` | `KIMI_API_KEY` | Kimi Coding |

### Model Aliases (Built-in)

| Alias | Model |
|-------|-------|
| `opus` | `anthropic/claude-opus-4-5` |
| `sonnet` | `anthropic/claude-sonnet-4-5` |
| `gpt` | `openai/gpt-5.2` |
| `gpt-mini` | `openai/gpt-5-mini` |
| `gemini` | `google/gemini-3-pro-preview` |
| `gemini-flash` | `google/gemini-3-flash-preview` |

---

## Session Configuration

### `session`

```json5
{
  session: {
    scope: "per-sender",
    dmScope: "main",                 // "main" | "per-peer" | "per-channel-peer" | "per-account-channel-peer"
    mainKey: "main",
    
    identityLinks: {
      alice: ["telegram:123456789", "discord:987654321012345678"]
    },
    
    reset: {
      mode: "daily",                 // "daily" | "idle"
      atHour: 4,
      idleMinutes: 60
    },
    
    resetByType: {
      thread: { mode: "daily", atHour: 4 },
      dm: { mode: "idle", idleMinutes: 240 },
      group: { mode: "idle", idleMinutes: 120 }
    },
    
    resetTriggers: ["/new", "/reset"],
    heartbeatIdleMinutes: 30,
    
    store: "~/.openclaw/agents/{agentId}/sessions/sessions.json",
    
    agentToAgent: {
      maxPingPongTurns: 5
    },
    
    sendPolicy: {
      default: "allow",
      rules: [
        { action: "deny", match: { channel: "discord", chatType: "group" } }
      ]
    }
  }
}
```

---

## Message Configuration

### `messages`

```json5
{
  messages: {
    responsePrefix: "🦞",            // or "auto", or template like "[{model}]"
    ackReaction: "👀",
    ackReactionScope: "group-mentions",  // "group-mentions" | "group-all" | "direct" | "all"
    removeAckAfterReply: false,
    
    groupChat: {
      historyLimit: 50
    },
    
    // Inbound debouncing
    inbound: {
      debounceMs: 2000,
      byChannel: {
        whatsapp: 5000,
        slack: 1500
      }
    },
    
    // Queue behavior
    queue: {
      mode: "collect",               // "steer" | "followup" | "collect" | "interrupt"
      debounceMs: 1000,
      cap: 20,
      drop: "summarize"              // "old" | "new" | "summarize"
    },
    
    // TTS
    tts: {
      auto: "always",                // "off" | "always" | "inbound" | "tagged"
      mode: "final",                 // "final" | "all"
      provider: "elevenlabs",
      maxTextLength: 4000,
      elevenlabs: {
        voiceId: "voice_id",
        modelId: "eleven_multilingual_v2"
      }
    }
  }
}
```

### Response Prefix Templates

| Variable | Description |
|----------|-------------|
| `{model}` | Short model name |
| `{modelFull}` | Full model identifier |
| `{provider}` | Provider name |
| `{thinkingLevel}` | Current thinking level |
| `{identity.name}` | Agent identity name |

---

## Skills & Plugins

### `skills`

```json5
{
  skills: {
    allowBundled: ["gemini", "peekaboo"],
    
    load: {
      extraDirs: ["~/Projects/my-skills"]
    },
    
    install: {
      preferBrew: true,
      nodeManager: "npm"
    },
    
    entries: {
      "nano-banana-pro": {
        apiKey: "GEMINI_KEY",
        env: { GEMINI_API_KEY: "GEMINI_KEY" }
      },
      sag: { enabled: false }
    }
  }
}
```

### `plugins`

```json5
{
  plugins: {
    enabled: true,
    allow: ["voice-call"],
    deny: ["deprecated-plugin"],
    
    load: {
      paths: ["~/Projects/my-plugin"]
    },
    
    entries: {
      "voice-call": {
        enabled: true,
        config: { provider: "twilio" }
      }
    }
  }
}
```

---

## Automation (Cron, Hooks)

### `cron`

```json5
{
  cron: {
    enabled: true,
    maxConcurrentRuns: 2
  }
}
```

### Cron Job Schema (via tool)

```json5
{
  name: "Morning check",
  schedule: { 
    kind: "cron",                    // "at" | "every" | "cron"
    expr: "0 9 * * *",
    tz: "America/New_York"
  },
  payload: {
    kind: "systemEvent",             // or "agentTurn"
    text: "Good morning! Check tasks."
  },
  sessionTarget: "main",             // "main" | "isolated"
  enabled: true
}
```

### `hooks`

```json5
{
  hooks: {
    enabled: true,
    token: "shared-secret",
    path: "/hooks",
    maxBodyBytes: 262144,
    presets: ["gmail"],
    
    mappings: [
      {
        match: { path: "gmail" },
        action: "agent",
        wakeMode: "now",
        name: "Gmail",
        sessionKey: "hook:gmail:{{messages[0].id}}",
        messageTemplate: "From: {{messages[0].from}}...",
        deliver: true,
        channel: "last"
      }
    ],
    
    gmail: {
      account: "user@gmail.com",
      topic: "projects/xxx/topics/gmail-watch",
      includeBody: true,
      model: "openai/gpt-5.2-mini"
    }
  }
}
```

---

## Quick Reference Tables

### DM Policy Options

| Value | Behavior |
|-------|----------|
| `pairing` | Unknown senders get pairing code (default) |
| `allowlist` | Only allow senders in `allowFrom` |
| `open` | Allow all (requires `allowFrom: ["*"]`) |
| `disabled` | Ignore all DMs |

### Group Policy Options

| Value | Behavior |
|-------|----------|
| `open` | Allow all groups (mention-gating still applies) |
| `allowlist` | Only configured groups (default) |
| `disabled` | Block all group messages |

### Thinking Levels

| Level | Description |
|-------|-------------|
| `off` | No extended thinking |
| `low` | Minimal thinking |
| `high` | Full reasoning |

### Sandbox Modes

| Mode | Behavior |
|------|----------|
| `off` | No sandboxing |
| `non-main` | Sandbox non-main sessions only |
| `all` | Sandbox all sessions |

### Sandbox Scopes

| Scope | Behavior |
|-------|----------|
| `session` | Container per session |
| `agent` | Container per agent (default) |
| `shared` | Shared container across agents |

### Workspace Access

| Mode | Behavior |
|------|----------|
| `none` | Isolated sandbox workspace (default) |
| `ro` | Agent workspace mounted read-only |
| `rw` | Agent workspace mounted read-write |

---

## File Locations Summary

| Path | Purpose |
|------|---------|
| `~/.openclaw/openclaw.json` | Main config |
| `~/.openclaw/.env` | Global environment vars |
| `~/.openclaw/workspace/` | Default agent workspace |
| `~/.openclaw/agents/<id>/` | Per-agent directories |
| `~/.openclaw/credentials/` | Channel credentials |
| `~/.openclaw/sessions/` | Session transcripts |
| `~/.openclaw/skills/` | Installed skills |
| `~/.openclaw/extensions/` | Installed plugins |
| `~/.openclaw/sandboxes/` | Sandbox workspaces |

---

*Last updated: 2026-02-04*
*Based on OpenClaw documentation*
