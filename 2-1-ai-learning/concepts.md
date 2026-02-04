# OpenClaw Core Concepts (AI Learning Reference)

*For AI agents running on OpenClaw - understanding the system you live in*

---

## 1. Architecture Overview

### Gateway-Centric Design
```
┌─────────────────────────────────────────────────────────────┐
│                        Gateway                               │
│  (Single long-lived process - the "brain" of OpenClaw)      │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐       │
│  │   Channels   │  │   Sessions   │  │    Cron      │       │
│  │  (WhatsApp,  │  │  (Context,   │  │  (Scheduler) │       │
│  │   Telegram,  │  │   Memory)    │  │              │       │
│  │   Discord)   │  │              │  │              │       │
│  └──────────────┘  └──────────────┘  └──────────────┘       │
│                                                              │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐       │
│  │    Nodes     │  │    Tools     │  │   Plugins    │       │
│  │  (iOS/Mac/   │  │  (exec,read, │  │  (channels,  │       │
│  │   Android)   │  │   browser)   │  │   memory)    │       │
│  └──────────────┘  └──────────────┘  └──────────────┘       │
│                                                              │
└─────────────────────────────────────────────────────────────┘
         ↑                    ↑                    ↑
    WebSocket            WebSocket            WebSocket
         │                    │                    │
    ┌────┴────┐          ┌────┴────┐          ┌────┴────┐
    │  Clients │          │  Nodes  │          │ WebChat │
    │(CLI,App) │          │(iOS/Mac)│          │         │
    └─────────┘          └─────────┘          └─────────┘
```

### Key Invariants
- **One Gateway per host** - controls all messaging surfaces
- **Gateway is source of truth** - all session state, token counts, config
- **WebSocket transport** - typed JSON protocol for all clients
- **Deterministic routing** - model never chooses where to send; routing is config-based

---

## 2. Sessions

### Session Keys
Sessions are buckets for conversation context. Key formats:

| Type | Format | Example |
|------|--------|---------|
| Main DM | `agent:<agentId>:<mainKey>` | `agent:main:main` |
| Group | `agent:<agentId>:<channel>:group:<id>` | `agent:main:discord:group:123` |
| Channel | `agent:<agentId>:<channel>:channel:<id>` | `agent:main:slack:channel:C123` |
| Thread | Base key + `:thread:<threadId>` | `...channel:123:thread:456` |
| Topic | Group key + `:topic:<topicId>` | `...group:123:topic:42` |
| Cron | `cron:<jobId>` | `cron:morning-brief` |
| Webhook | `hook:<uuid>` | `hook:abc123` |

### DM Scope Options
- `main` (default): All DMs collapse to one session (continuity)
- `per-peer`: Isolate by sender ID
- `per-channel-peer`: Isolate by channel + sender
- `per-account-channel-peer`: Isolate by account + channel + sender

### Session Lifecycle
1. **Creation**: On first message to a key
2. **Reuse**: Until reset policy triggers
3. **Reset**: Daily (4 AM local) or idle timeout, whichever first
4. **Manual reset**: `/new` or `/reset` commands

### Session Storage
```
~/.openclaw/agents/<agentId>/sessions/
├── sessions.json          # Key → session metadata map
├── <sessionId>.jsonl      # Transcript (append-only)
└── <sessionId>-topic-<threadId>.jsonl  # Topic transcripts
```

---

## 3. Memory System

### Memory Philosophy
> "The model only 'remembers' what gets written to disk."

Memory is **plain Markdown in the workspace**. Files are the source of truth.

### Memory Files
```
workspace/
├── MEMORY.md              # Curated long-term (main session only)
├── memory/
│   ├── 2026-02-04.md      # Daily logs (read today + yesterday)
│   └── cells/             # Atomic reusable knowledge
└── AGENTS.md              # Operating instructions
```

### Memory Tools
| Tool | Purpose |
|------|---------|
| `memory_search` | Semantic search over memory files |
| `memory_get` | Read specific file/lines |

### Vector Search
- Default: Enabled with auto-provider selection
- Backends: OpenAI embeddings, Gemini, local GGUF, or QMD
- Hybrid search: BM25 + vector for best of both
- Index: `~/.openclaw/memory/<agentId>.sqlite`

### Pre-Compaction Memory Flush
When session nears compaction threshold:
1. Silent agentic turn triggered
2. Model prompted to write durable notes
3. Model replies `NO_REPLY` (user sees nothing)
4. Then compaction proceeds

Config: `agents.defaults.compaction.memoryFlush`

---

## 4. Context Window Management

### Compaction
When context exceeds model window:
1. Older messages summarized
2. Summary stored in JSONL
3. Recent messages kept intact
4. Retry with compacted context

**Manual**: `/compact [instructions]`
**Auto**: Triggered when nearing limit

### Session Pruning
- Trims old **tool results** (not messages)
- In-memory only (doesn't modify JSONL)
- Happens before each LLM call

### Context Commands
- `/status` - Context usage, compaction count
- `/context list` - What's in system prompt
- `/context detail` - Full context breakdown

---

## 5. Channel Routing

### Routing Priority
1. Exact peer match (`bindings.peer`)
2. Guild/Team match (Discord/Slack)
3. Account match
4. Channel match
5. Default agent

### Multi-Agent
- Multiple agents can have separate workspaces
- Bindings route inbound to specific agents
- Broadcast groups run multiple agents for same peer

### Reply Routing
- Replies go **back to origin channel**
- Model doesn't choose destination
- Cross-channel messaging uses `sessions_send` or `message` tool

---

## 6. Automation

### Cron Jobs
Persistent scheduler inside Gateway. Two execution styles:

**Main Session** (`sessionTarget: "main"`):
- Uses `payload.kind: "systemEvent"`
- Injects into heartbeat prompt
- Shares main session context

**Isolated Session** (`sessionTarget: "isolated"`):
- Uses `payload.kind: "agentTurn"`
- Fresh session per run (`cron:<jobId>`)
- No context carry-over
- Can deliver to channels

### Schedule Types
| Kind | Use | Example |
|------|-----|---------|
| `at` | One-shot | `{ "kind": "at", "atMs": 1738262400000 }` |
| `every` | Interval | `{ "kind": "every", "everyMs": 3600000 }` |
| `cron` | Cron expr | `{ "kind": "cron", "expr": "0 7 * * *" }` |

### Heartbeat
- Periodic poll of main session
- Default prompt checks `HEARTBEAT.md`
- Good for: Multiple checks, needs main context
- Alternative to: Multiple cron jobs

### Cron vs Heartbeat Decision
| Use Cron When | Use Heartbeat When |
|---------------|-------------------|
| Exact timing needed | Timing can drift |
| Isolated context | Needs conversation context |
| Different model/thinking | Same model as main |
| One-shot reminders | Batch periodic checks |
| Channel delivery | Internal processing |

---

## 7. Tools & Skills

### Core Tools (Always Available)
- `read`, `write`, `edit` - File operations
- `exec`, `process` - Shell commands
- `browser` - Web automation
- `web_search`, `web_fetch` - Web access
- `memory_search`, `memory_get` - Memory access
- `cron` - Scheduling
- `message` - Channel messaging
- `sessions_*` - Session management
- `nodes` - Device control

### Skills
Loaded from (workspace wins on conflict):
1. Bundled (shipped with OpenClaw)
2. Managed: `~/.openclaw/skills`
3. Workspace: `<workspace>/skills`

Skills provide specialized capabilities via `SKILL.md` files.

### Tool Policy
- `TOOLS.md` = User guidance for how to use tools
- Does NOT control which tools exist
- Actual availability controlled by config/policy

---

## 8. Agent Workspace

### Bootstrap Files (Injected on Session Start)
| File | Purpose | Security |
|------|---------|----------|
| `AGENTS.md` | Operating instructions | Always loaded |
| `SOUL.md` | Persona, tone, boundaries | Always loaded |
| `USER.md` | User profile | Always loaded |
| `TOOLS.md` | Tool usage notes | Always loaded |
| `IDENTITY.md` | Agent name/vibe | Always loaded |
| `MEMORY.md` | Long-term memory | **Main session only** |
| `BOOTSTRAP.md` | First-run ritual | Deleted after use |
| `HEARTBEAT.md` | Heartbeat checklist | Read on heartbeat |

### Workspace Layout
```
workspace/
├── AGENTS.md
├── SOUL.md
├── USER.md
├── TOOLS.md
├── IDENTITY.md
├── MEMORY.md
├── HEARTBEAT.md
├── memory/
│   ├── YYYY-MM-DD.md
│   └── cells/
├── tasks/
│   ├── active/
│   └── archive/
├── skills/
└── projects/
```

---

## 9. Message Queue & Streaming

### Queue Modes
| Mode | Behavior |
|------|----------|
| `steer` | Inject into current run after tool calls |
| `followup` | Hold until turn ends, then new turn |
| `collect` | Batch until turn ends |

### Block Streaming
- Off by default
- Sends completed blocks as they finish
- Controlled by `blockStreamingDefault`, `blockStreamingBreak`
- Non-Telegram channels need explicit enable

### Silent Replies
- `NO_REPLY` - Say nothing
- `HEARTBEAT_OK` - Heartbeat ack, no action needed

---

## 10. Security Boundaries

### Trust Model
- Gateway is the trust boundary
- All WS clients need auth token
- Nodes require pairing approval
- Local connections can auto-approve

### File Access
- Workspace is the allowed root
- Memory tools limited to `MEMORY.md` + `memory/`
- Tool policy controls exec permissions

### Send Policy
Block delivery for specific session types:
```json5
{
  session: {
    sendPolicy: {
      rules: [
        { action: "deny", match: { channel: "discord", chatType: "group" } }
      ],
      default: "allow"
    }
  }
}
```

---

## Quick Reference

### Essential Commands (In-Chat)
| Command | Action |
|---------|--------|
| `/status` | Session status, context usage |
| `/new` | Fresh session |
| `/reset` | Alias for /new |
| `/compact` | Manual compaction |
| `/stop` | Abort current run |
| `/context list` | Show context sources |

### Response Patterns
| Situation | Response |
|-----------|----------|
| Nothing to say | `NO_REPLY` |
| Heartbeat, no action | `HEARTBEAT_OK` |
| Reply to specific msg | `[[reply_to:<id>]]` |
| Reply to current | `[[reply_to_current]]` |

### State Locations
| What | Where |
|------|-------|
| Config | `~/.openclaw/openclaw.json` |
| Sessions | `~/.openclaw/agents/<id>/sessions/` |
| Memory index | `~/.openclaw/memory/<id>.sqlite` |
| Cron jobs | `~/.openclaw/cron/jobs.json` |
| Logs | `~/.openclaw/logs/` |

---

*Last updated: 2026-02-04*
