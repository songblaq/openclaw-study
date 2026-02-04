# OpenClaw Internal Mechanisms (AI Learning Reference)

*Deep dive into how OpenClaw works under the hood*

---

## 1. Agent Loop Lifecycle

The **agent loop** is the core execution path: intake → context assembly → model inference → tool execution → streaming replies → persistence.

### Entry Points
- **Gateway RPC**: `agent` and `agent.wait`
- **CLI**: `agent` command

### High-Level Flow
```
┌─────────────────────────────────────────────────────────────────┐
│                         Agent Loop                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1. agent RPC                                                    │
│     └─ validate params                                           │
│     └─ resolve session (sessionKey/sessionId)                    │
│     └─ persist session metadata                                  │
│     └─ return { runId, acceptedAt }                              │
│                                                                  │
│  2. agentCommand                                                 │
│     └─ resolve model + thinking/verbose defaults                 │
│     └─ load skills snapshot                                      │
│     └─ call runEmbeddedPiAgent (pi-agent-core runtime)           │
│     └─ emit lifecycle end/error if not emitted                   │
│                                                                  │
│  3. runEmbeddedPiAgent                                           │
│     └─ serialize via per-session + global queues                 │
│     └─ resolve model + auth profile                              │
│     └─ build pi session                                          │
│     └─ subscribe to pi events                                    │
│     └─ stream assistant/tool deltas                              │
│     └─ enforce timeout → abort if exceeded                       │
│     └─ return payloads + usage metadata                          │
│                                                                  │
│  4. subscribeEmbeddedPiSession                                   │
│     └─ tool events → stream: "tool"                              │
│     └─ assistant deltas → stream: "assistant"                    │
│     └─ lifecycle events → stream: "lifecycle"                    │
│                                                                  │
│  5. agent.wait                                                   │
│     └─ waitForAgentJob                                           │
│     └─ wait for lifecycle end/error for runId                    │
│     └─ return { status: ok|error|timeout, ... }                  │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Session Preparation
Before streaming begins:
1. **Workspace resolution** - created if missing; sandboxed runs redirect to sandbox root
2. **Skills loading** - loaded or reused from snapshot, injected into env + prompt
3. **Bootstrap files** - resolved and injected into system prompt
4. **Session write lock** - acquired; SessionManager opened and prepared

### Lifecycle Events
| Phase | Description |
|-------|-------------|
| `start` | Agent run beginning |
| `end` | Successful completion |
| `error` | Run failed with error |

### Timeouts
- **Agent runtime**: `agents.defaults.timeoutSeconds` (default 600s)
- **agent.wait**: default 30s (only affects wait, not agent execution)

---

## 2. Message Queue & Concurrency

OpenClaw serializes inbound runs to prevent collisions while allowing safe parallelism.

### Queue Architecture
```
Inbound Message
      │
      ▼
┌─────────────────┐
│   Lane Queue    │
│ (per session)   │─────┐
└─────────────────┘     │
                        ▼
              ┌─────────────────┐
              │  Global Lane    │
              │ (main/cron/     │
              │  subagent)      │
              └─────────────────┘
                        │
                        ▼
              ┌─────────────────┐
              │   Agent Run     │
              └─────────────────┘
```

### Queue Modes
| Mode | Behavior |
|------|----------|
| `steer` | Inject into current run after next tool boundary |
| `followup` | Enqueue for next turn after current run ends |
| `collect` | Coalesce all queued messages into single followup (default) |
| `steer-backlog` | Steer now AND preserve for followup |
| `interrupt` | Abort active run, run newest message |

### Concurrency Configuration
```json5
{
  agents: {
    defaults: {
      maxConcurrent: 4,  // Global lane cap
    }
  },
  messages: {
    queue: {
      mode: "collect",
      debounceMs: 1000,   // Wait for quiet before followup
      cap: 20,            // Max queued per session
      drop: "summarize",  // Overflow: old|new|summarize
    }
  }
}
```

### Lane Types
- **Session lane** (`session:<key>`): One active run per session
- **Main lane**: Inbound + heartbeats (cap: 4 default)
- **Subagent lane**: Background runs (cap: 8 default)
- **Cron lane**: Scheduled jobs (parallel with main)

---

## 3. Session Management

### Session Storage
```
~/.openclaw/agents/<agentId>/sessions/
├── sessions.json              # Key → metadata map
├── <sessionId>.jsonl          # Transcript (append-only)
└── <sessionId>-topic-<id>.jsonl  # Topic transcripts
```

### JSONL Transcript Format
Each line is a JSON event:
```jsonl
{"type":"user","content":"Hello","timestamp":"..."}
{"type":"assistant","content":"Hi!","timestamp":"..."}
{"type":"tool","name":"read","result":"...","timestamp":"..."}
{"type":"compaction","summary":"...","timestamp":"..."}
```

### Session Lifecycle
1. **Creation**: On first message to a key
2. **Reuse**: Until reset policy triggers
3. **Reset**: Daily at 4 AM local OR idle timeout (whichever first)
4. **Manual reset**: `/new` or `/reset` commands

### Session Key Resolution
```
agent:<agentId>:<mainKey>
agent:<agentId>:<channel>:group:<id>
agent:<agentId>:<channel>:channel:<id>
... + :thread:<threadId>
... + :topic:<topicId>
```

### DM Scope Options
| Scope | Behavior |
|-------|----------|
| `main` | All DMs → one session (default) |
| `per-peer` | Isolate by sender ID |
| `per-channel-peer` | Isolate by channel + sender |
| `per-account-channel-peer` | Isolate by account + channel + sender |

---

## 4. Compaction Pipeline

When context exceeds model window, compaction summarizes older history.

### Compaction Flow
```
Session nearing limit
        │
        ▼
┌───────────────────────┐
│ Memory Flush (silent) │ ← Optional: write durable notes
└───────────────────────┘
        │
        ▼
┌───────────────────────┐
│  Identify cutoff      │ ← Keep recent, summarize old
└───────────────────────┘
        │
        ▼
┌───────────────────────┐
│   LLM summarization   │ ← Generate compact summary
└───────────────────────┘
        │
        ▼
┌───────────────────────┐
│  Persist to JSONL     │ ← Summary stored permanently
└───────────────────────┘
        │
        ▼
┌───────────────────────┐
│    Retry request      │ ← With compacted context
└───────────────────────┘
```

### Compaction vs Pruning
| Compaction | Pruning |
|------------|---------|
| Summarizes messages | Trims tool results |
| **Persists** to JSONL | **In-memory** only |
| Full conversation | Only tool outputs |
| Triggered by context limit | Triggered by cache TTL |

### Memory Flush
Before compaction, a **silent agentic turn** can run:
1. Model prompted to write durable notes
2. Model replies `NO_REPLY` (user sees nothing)
3. Then compaction proceeds

Config: `agents.defaults.compaction.memoryFlush`

---

## 5. Streaming Architecture

### Two Streaming Layers
1. **Block streaming (channels)**: Emit completed blocks as assistant writes
2. **Draft streaming (Telegram only)**: Update draft bubble with partial text

### Block Streaming Flow
```
Model output
  └─ text_delta events
       │
       ▼
  ┌──────────────┐
  │   Chunker    │ ← minChars/maxChars/breakPreference
  └──────────────┘
       │
       ▼
  ┌──────────────┐
  │  Coalescer   │ ← Optional: merge before send
  └──────────────┘
       │
       ▼
  ┌──────────────┐
  │ Channel Send │ ← Block replies
  └──────────────┘
```

### Chunking Algorithm
- **Low bound**: Don't emit until buffer >= `minChars`
- **High bound**: Split before `maxChars`
- **Break preference**: `paragraph` → `newline` → `sentence` → `whitespace` → hard
- **Code fences**: Never split inside; close + reopen if forced

### Streaming Configuration
```json5
{
  agents: {
    defaults: {
      blockStreamingDefault: "off",  // "on" | "off"
      blockStreamingBreak: "text_end",  // "text_end" | "message_end"
      blockStreamingChunk: {
        minChars: 200,
        maxChars: 800,
        breakPreference: "paragraph"
      },
      blockStreamingCoalesce: {
        minChars: 500,
        maxChars: 2000,
        idleMs: 500
      },
      humanDelay: "off"  // "off" | "natural" | { minMs, maxMs }
    }
  }
}
```

### Event Streams
| Stream | Content |
|--------|---------|
| `lifecycle` | phase: start/end/error |
| `assistant` | text deltas from model |
| `tool` | tool start/update/end events |

---

## 6. Hooks System

Event-driven automation for commands and lifecycle events.

### Hook Discovery (Precedence Order)
1. **Workspace**: `<workspace>/hooks/`
2. **Managed**: `~/.openclaw/hooks/`
3. **Bundled**: `<openclaw>/dist/hooks/bundled/`

### Hook Structure
```
my-hook/
├── HOOK.md          # Metadata + documentation
└── handler.ts       # Handler implementation
```

### Event Types
| Category | Events |
|----------|--------|
| Command | `command:new`, `command:reset`, `command:stop` |
| Agent | `agent:bootstrap` |
| Gateway | `gateway:startup` |
| Plugin | `tool_result_persist` (synchronous) |

### Handler Interface
```typescript
const handler: HookHandler = async (event) => {
  // event.type: 'command' | 'session' | 'agent' | 'gateway'
  // event.action: string (e.g., 'new', 'reset')
  // event.sessionKey: string
  // event.timestamp: Date
  // event.messages: string[] (push to send to user)
  // event.context: { sessionEntry, workspaceDir, cfg, ... }
};
```

### Bundled Hooks
| Hook | Event | Purpose |
|------|-------|---------|
| `session-memory` | `command:new` | Save context to memory on reset |
| `command-logger` | `command` | Audit log to commands.log |
| `boot-md` | `gateway:startup` | Run BOOT.md at startup |
| `soul-evil` | `agent:bootstrap` | Swap SOUL.md during purge window |

---

## 7. Plugin Architecture

Plugins extend OpenClaw with tools, channels, commands, and services.

### Plugin Discovery (Precedence Order)
1. **Config paths**: `plugins.load.paths`
2. **Workspace extensions**: `<workspace>/.openclaw/extensions/`
3. **Global extensions**: `~/.openclaw/extensions/`
4. **Bundled**: `<openclaw>/extensions/` (disabled by default)

### Plugin Structure
```
my-plugin/
├── openclaw.plugin.json   # Manifest
├── index.ts               # Entry point
└── (optional) skills/     # Bundled skills
```

### Plugin API
```typescript
export default function (api) {
  // Register tools
  api.registerTool({ name: "my_tool", ... });
  
  // Register channels
  api.registerChannel({ plugin: myChannelPlugin });
  
  // Register RPC methods
  api.registerGatewayMethod("myplugin.status", handler);
  
  // Register CLI commands
  api.registerCli(({ program }) => { ... });
  
  // Register auto-reply commands
  api.registerCommand({ name: "mystatus", handler });
  
  // Register services
  api.registerService({ id: "my-service", start, stop });
  
  // Register provider auth flows
  api.registerProvider({ id: "acme", auth: [...] });
}
```

### Plugin Hooks
| Hook | When |
|------|------|
| `before_agent_start` | Before run starts |
| `agent_end` | After completion |
| `before_compaction` / `after_compaction` | Compaction cycle |
| `before_tool_call` / `after_tool_call` | Tool execution |
| `tool_result_persist` | Before persisting tool results |
| `message_received` / `message_sending` / `message_sent` | Message lifecycle |
| `session_start` / `session_end` | Session lifecycle |
| `gateway_start` / `gateway_stop` | Gateway lifecycle |

### Plugin Slots
Exclusive categories (only one active):
```json5
{
  plugins: {
    slots: {
      memory: "memory-core"  // or "memory-lancedb" or "none"
    }
  }
}
```

---

## 8. Context Pruning

Trims old tool results to reduce context bloat (in-memory, not persisted).

### Pruning Flow
```
Before LLM call
      │
      ▼
┌─────────────────────┐
│ Check cache TTL     │ ← Only run if last call > TTL
└─────────────────────┘
      │
      ▼
┌─────────────────────┐
│ Identify candidates │ ← Tool results only
└─────────────────────┘
      │
      ▼
┌─────────────────────┐
│ Skip protected      │ ← Last N assistants, images
└─────────────────────┘
      │
      ▼
┌─────────────────────┐
│ Soft trim oversized │ ← Keep head+tail, truncate middle
└─────────────────────┘
      │
      ▼
┌─────────────────────┐
│ Hard clear old      │ ← Replace with placeholder
└─────────────────────┘
```

### Pruning Modes
| Mode | Behavior |
|------|----------|
| `off` | Disabled (default) |
| `cache-ttl` | Run when last Anthropic call > TTL |

### Pruning Levels
| Level | Action | Applies When |
|-------|--------|--------------|
| Soft trim | Keep head+tail, truncate middle | Result > softTrim.maxChars |
| Hard clear | Replace with placeholder | Context ratio > hardClearRatio |

### Configuration
```json5
{
  agent: {
    contextPruning: {
      mode: "cache-ttl",
      ttl: "5m",
      keepLastAssistants: 3,
      softTrimRatio: 0.3,
      hardClearRatio: 0.5,
      minPrunableToolChars: 50000,
      softTrim: {
        maxChars: 4000,
        headChars: 1500,
        tailChars: 1500
      },
      hardClear: {
        enabled: true,
        placeholder: "[Old tool result content cleared]"
      },
      tools: {
        allow: ["exec", "read"],
        deny: ["*image*"]
      }
    }
  }
}
```

---

## 9. Reply Shaping & Suppression

### Payload Assembly
Final reply consists of:
1. Assistant text (+ optional reasoning)
2. Inline tool summaries (when verbose + allowed)
3. Error text (when model errors)

### Suppression Rules
- `NO_REPLY` → Silent token, filtered from output
- `HEARTBEAT_OK` → Heartbeat ack, discarded
- Messaging tool duplicates → Removed from final payload
- Empty payload + tool error → Fallback error reply

### Reply Tags
```
[[reply_to_current]]     → Reply to triggering message
[[reply_to:<id>]]        → Reply to specific message ID
```

---

## 10. Gateway Startup Sequence

### Boot Flow
```
openclaw gateway
      │
      ▼
┌─────────────────────┐
│ Load configuration  │
└─────────────────────┘
      │
      ▼
┌─────────────────────┐
│ Start sidecars      │ ← Background services
└─────────────────────┘
      │
      ▼
┌─────────────────────┐
│ Load plugins        │ ← Discovery + registration
└─────────────────────┘
      │
      ▼
┌─────────────────────┐
│ Load hooks          │ ← Discovery + registration
└─────────────────────┘
      │
      ▼
┌─────────────────────┐
│ Start channels      │ ← WhatsApp, Telegram, etc.
└─────────────────────┘
      │
      ▼
┌─────────────────────┐
│ Start WebSocket     │ ← Listen on port
└─────────────────────┘
      │
      ▼
┌─────────────────────┐
│ Fire gateway:startup│ ← Hook event
└─────────────────────┘
      │
      ▼
┌─────────────────────┐
│ Start cron scheduler│
└─────────────────────┘
```

### Signal Handling
- `SIGUSR1` → In-process restart (when authorized)
- `SIGINT` / `SIGTERM` → Graceful shutdown

---

## Quick Reference: Internal Flows

### Inbound Message Flow
```
Channel receives message
  → Queue by session key
  → Queue to global lane
  → Run agent loop
  → Stream response
  → Send to channel
```

### Tool Execution Flow
```
Model requests tool call
  → Emit tool:start
  → Execute tool
  → Sanitize result (size, images)
  → Emit tool:end
  → Persist to JSONL
  → Continue inference
```

### Memory Lookup Flow
```
memory_search called
  → Embed query (configured provider)
  → Vector search (SQLite+Vec)
  → BM25 search (hybrid)
  → Merge results
  → Return snippets with path+lines
```

### Compaction Trigger Flow
```
Context nearing limit
  → Optional memory flush
  → Identify compaction point
  → Summarize via LLM
  → Persist summary
  → Rebuild messages array
  → Retry original request
```

---

*Last updated: 2026-02-04*
