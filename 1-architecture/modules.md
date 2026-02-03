# OpenClaw Core Modules Analysis

> Phase 1 - Source Code Analysis: Core Module Deep Dive
> Last Updated: 2026-02-04

## Overview

OpenClaw is built around four core module clusters that handle communication, agent management, session state, and semantic memory. This document provides detailed analysis of each module's architecture and responsibilities.

---

## 1. Gateway Module (`src/gateway/`)

### Purpose
The Gateway is the central communication hub of OpenClaw. It provides a WebSocket-based server that manages connections between clients (CLI, mobile apps, external integrations) and the AI agent system.

### Key Components

#### 1.1 Gateway Client (`client.ts`)
**Class: `GatewayClient`**

Core WebSocket client for connecting to the Gateway server.

```typescript
type GatewayClientOptions = {
  url?: string;              // WebSocket URL (default: ws://127.0.0.1:18789)
  token?: string;            // Auth token
  password?: string;         // Auth password
  instanceId?: string;       // Client instance identifier
  clientName?: GatewayClientName;
  mode?: GatewayClientMode;  // 'backend' | 'probe' | etc.
  role?: string;             // 'operator' | etc.
  scopes?: string[];         // Permission scopes
  caps?: string[];           // Capabilities
  tlsFingerprint?: string;   // TLS certificate pinning
  onEvent?: (evt: EventFrame) => void;
  onHelloOk?: (hello: HelloOk) => void;
  // ... more callbacks
}
```

**Key Features:**
- **Automatic Reconnection**: Exponential backoff (1s → 30s max)
- **Device Authentication**: Public key-based device identity
- **Protocol Versioning**: Negotiates protocol version during handshake
- **Tick Watchdog**: Monitors connection health via periodic ticks
- **TLS Fingerprint Validation**: Certificate pinning for secure connections

**Connection Flow:**
1. WebSocket connection established
2. Server sends `connect.challenge` with nonce
3. Client signs payload with device private key
4. Client sends `connect` request with signed params
5. Server validates and responds with `HelloOk`
6. Tick watchdog starts monitoring

#### 1.2 Protocol (`protocol/`)
Defines the message framing protocol:

- **RequestFrame**: `{ type: "req", id, method, params }`
- **ResponseFrame**: `{ type: "res", id, ok, payload?, error? }`
- **EventFrame**: `{ event, payload, seq? }`

#### 1.3 Hooks System (`hooks.ts`)
External webhook integration for triggering agent actions.

**Endpoints:**
- `POST /hooks/wake` - Wake agent with text prompt
- `POST /hooks/agent` - Send message to agent session

**Security:**
- Token-based authentication (Bearer token or X-OpenClaw-Token header)
- Configurable max body size (default: 256KB)
- Request validation and sanitization

#### 1.4 Server Components (`server/`)

| File | Purpose |
|------|---------|
| `ws-connection.ts` | WebSocket connection management |
| `health-state.ts` | Server health monitoring |
| `hooks.ts` | Server-side hook handlers |
| `http-listen.ts` | HTTP server setup |
| `tls.ts` | TLS/SSL configuration |
| `plugins-http.ts` | HTTP plugin routing |

### Architecture Patterns

1. **Request/Response with Pending Map**: Tracks in-flight requests via UUID
2. **Event Streaming**: One-way server-to-client events with sequence numbers
3. **Gap Detection**: Detects missing event sequences for reliability

---

## 2. Agents Module (`src/agents/`)

### Purpose
Manages AI agent configuration, workspace isolation, tool execution, and skill integration.

### Key Components

#### 2.1 Agent Scope (`agent-scope.ts`)
Resolves agent-specific configurations from the global config.

**Key Functions:**
```typescript
// List all configured agent IDs
listAgentIds(cfg: OpenClawConfig): string[]

// Resolve the default agent
resolveDefaultAgentId(cfg: OpenClawConfig): string

// Resolve agent-specific configuration
resolveAgentConfig(cfg: OpenClawConfig, agentId: string): ResolvedAgentConfig

// Resolve agent workspace directory
resolveAgentWorkspaceDir(cfg: OpenClawConfig, agentId: string): string

// Resolve agent state directory
resolveAgentDir(cfg: OpenClawConfig, agentId: string): string
```

**Agent Configuration Structure:**
```typescript
type ResolvedAgentConfig = {
  name?: string;           // Display name
  workspace?: string;      // Working directory
  agentDir?: string;       // State directory
  model?: AgentModel;      // Primary + fallback models
  skills?: string[];       // Enabled skills filter
  memorySearch?: MemorySearchConfig;
  humanDelay?: HumanDelayConfig;
  heartbeat?: HeartbeatConfig;
  identity?: IdentityConfig;
  groupChat?: GroupChatConfig;
  subagents?: SubagentsConfig;
  sandbox?: SandboxConfig;
  tools?: ToolsConfig;
}
```

#### 2.2 Bash Tools (`bash-tools/`)
Shell command execution with safety controls.

| File | Purpose |
|------|---------|
| `bash-tools.exec.ts` | Command execution logic |
| `bash-tools.process.ts` | Process management (list, kill, send-keys) |
| `bash-process-registry.ts` | Track running processes |
| `bash-tools.shared.ts` | Shared utilities |

**Execution Features:**
- PTY support for interactive commands
- Background process continuation
- Approval workflow integration
- Timeout management

#### 2.3 Auth Profiles (`auth-profiles/`)
Manages API key rotation and failover.

**Key Concepts:**
- **Profile Order Resolution**: Prioritizes keys by last success/failure
- **Cooldown Management**: Backoff on rate limits
- **Chutes**: Specialized routing for different model providers

#### 2.4 Skills System (`skills/`)
Modular capability extensions for agents.

**Skill Loading:**
1. Skills discovered from `~/.openclaw/skills/` or configured paths
2. Each skill has a `SKILL.md` defining its purpose
3. Skills can expose tools, context, or both
4. Agent config can filter which skills are active

#### 2.5 Sandbox (`sandbox/`)
Isolated execution environments for untrusted code.

#### 2.6 Tools (`tools/`)
Built-in tool implementations exposed to the AI model.

---

## 3. Sessions Module (`src/sessions/`)

### Purpose
Manages conversation state, message policies, and transcript persistence.

### Key Components

#### 3.1 Send Policy (`send-policy.ts`)
Controls whether outbound messages are allowed.

```typescript
type SessionSendPolicyDecision = "allow" | "deny";

function resolveSendPolicy(params: {
  cfg: OpenClawConfig;
  entry?: SessionEntry;
  sessionKey?: string;
  channel?: string;
  chatType?: SessionChatType;
}): SessionSendPolicyDecision
```

**Policy Resolution:**
1. Check session-level override
2. Evaluate rules in order (first match wins)
3. Apply default policy

**Rule Matching:**
- `match.channel`: Filter by channel (discord, telegram, etc.)
- `match.chatType`: Filter by chat type (group, channel, dm)
- `match.keyPrefix`: Filter by session key prefix

#### 3.2 Session Key Utils (`session-key-utils.ts`)
Parses and manipulates session identifiers.

**Session Key Format:**
```
agent:<agentId>:<channel>:<type>:<identifier>

Examples:
- agent:main:discord:channel:123456789
- agent:main:telegram:group:987654321
```

#### 3.3 Transcript Events (`transcript-events.ts`)
Event system for transcript file updates.

```typescript
function onSessionTranscriptUpdate(
  callback: (update: { sessionFile: string }) => void
): () => void  // Returns unsubscribe function
```

#### 3.4 Model & Level Overrides
- `model-overrides.ts`: Per-session model selection
- `level-overrides.ts`: Thinking level configuration

---

## 4. Memory Module (`src/memory/`)

### Purpose
Provides semantic search over workspace files and session transcripts using embedding vectors.

### Key Components

#### 4.1 Memory Index Manager (`manager.ts`)
**Class: `MemoryIndexManager`**

The central orchestrator for memory indexing and search.

```typescript
class MemoryIndexManager implements MemorySearchManager {
  // Factory method with caching
  static async get(params: {
    cfg: OpenClawConfig;
    agentId: string;
  }): Promise<MemoryIndexManager | null>

  // Search interface
  async search(query: string, opts?: {
    maxResults?: number;
    minScore?: number;
    sessionKey?: string;
  }): Promise<MemorySearchResult[]>

  // Sync index with filesystem
  async sync(params?: {
    reason?: string;
    force?: boolean;
    progress?: (update: MemorySyncProgressUpdate) => void;
  }): Promise<void>

  // Status for debugging
  status(): MemoryProviderStatus
}
```

**Memory Sources:**
- `memory`: Files in `workspace/memory/` and `MEMORY.md`
- `sessions`: Session transcript files (`.jsonl`)

#### 4.2 Embedding Providers

| Provider | File | Description |
|----------|------|-------------|
| OpenAI | `embeddings-openai.ts` | text-embedding-3-small/large |
| Gemini | `embeddings-gemini.ts` | Google's embedding API |
| Local | via Ollama/LlamaCpp | Self-hosted models |

**Provider Selection:**
1. Try configured provider (openai/gemini/local)
2. On failure, fallback to secondary provider
3. Track failures for future prioritization

#### 4.3 Hybrid Search (`hybrid.ts`)
Combines vector similarity with keyword search.

**Algorithm:**
1. Run BM25 keyword search (FTS5)
2. Run vector similarity search (sqlite-vec)
3. Merge results with configurable weights
4. Re-rank and deduplicate

```typescript
const merged = mergeHybridResults({
  vector: vectorResults,
  keyword: keywordResults,
  vectorWeight: 0.7,  // Default
  textWeight: 0.3,    // Default
});
```

#### 4.4 Storage (`sqlite.ts`, `sqlite-vec.ts`)
SQLite-based storage with vector extension.

**Tables:**
- `files`: File metadata (path, hash, mtime, source)
- `chunks`: Text chunks with line numbers
- `chunks_vec`: Vector embeddings (virtual table)
- `chunks_fts`: Full-text search index
- `embedding_cache`: Cached embeddings by content hash
- `meta`: Index metadata (model, provider, dimensions)

#### 4.5 Batch Processing
For efficient initial indexing:

- `batch-openai.ts`: OpenAI Batch API integration
- `batch-gemini.ts`: Gemini batch embedding

**Batch Features:**
- Concurrent batch submission
- Progress tracking
- Retry with exponential backoff
- Failure limit before fallback

#### 4.6 Sync Strategy

**Incremental Sync:**
1. Compare file hashes against stored values
2. Re-chunk and re-embed only changed files
3. Clean up stale entries

**Full Reindex Triggers:**
- Model change
- Provider change
- Chunk size change
- Forced sync

**Session Delta Tracking:**
- Monitor transcript file growth
- Trigger sync on byte/message thresholds
- Debounce rapid updates

---

## Module Interactions

```
┌─────────────────────────────────────────────────────────────────┐
│                         Gateway                                  │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────────────┐ │
│  │ Protocol │  │ WS Server│  │  Hooks   │  │  HTTP Plugins    │ │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────────┬─────────┘ │
└───────┼─────────────┼─────────────┼──────────────────┼──────────┘
        │             │             │                  │
        ▼             ▼             ▼                  ▼
┌─────────────────────────────────────────────────────────────────┐
│                         Agents                                   │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────────────┐ │
│  │  Scope   │  │ Bash Exec│  │  Skills  │  │  Auth Profiles   │ │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────────┬─────────┘ │
└───────┼─────────────┼─────────────┼──────────────────┼──────────┘
        │             │             │                  │
        ▼             ▼             ▼                  ▼
┌─────────────────────────────────────────────────────────────────┐
│                        Sessions                                  │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────────────┐ │
│  │Send Policy│ │ Key Utils│  │Transcripts│ │  Model Override  │ │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────────┬─────────┘ │
└───────┼─────────────┼─────────────┼──────────────────┼──────────┘
        │             │             │                  │
        ▼             ▼             ▼                  ▼
┌─────────────────────────────────────────────────────────────────┐
│                         Memory                                   │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────────────┐ │
│  │  Manager │  │Embeddings│  │Hybrid Srch│ │  SQLite + Vec    │ │
│  └──────────┘  └──────────┘  └──────────┘  └──────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
```

---

## Key Design Patterns

### 1. Singleton with Cache Key
Memory manager uses a cache key combining agentId + workspace + settings hash:
```typescript
const key = `${agentId}:${workspaceDir}:${JSON.stringify(settings)}`;
const existing = INDEX_CACHE.get(key);
if (existing) return existing;
```

### 2. Graceful Degradation
When primary embedding provider fails:
1. Log failure reason
2. Switch to fallback provider
3. Trigger full reindex with new provider
4. Continue operating

### 3. Event-Driven Sync
File changes → Watcher event → Debounce → Sync trigger:
```typescript
this.watcher.on("change", () => {
  this.dirty = true;
  this.scheduleWatchSync();  // Debounced
});
```

### 4. Atomic Index Updates
Safe reindex via temp database swap:
1. Create temp database
2. Build new index
3. Swap files atomically
4. Delete old database

---

## Configuration Reference

### Agent Configuration (`openclaw.yaml`)
```yaml
agents:
  defaults:
    workspace: ~/.openclaw/workspace
  list:
    - id: main
      name: Blaq
      workspace: ~/.openclaw/workspace
      model:
        primary: anthropic/claude-sonnet-4-20250514
        fallbacks:
          - openai/gpt-4o
      skills:
        - browser
        - memory
      memorySearch:
        enabled: true
        provider: openai
        sources:
          - memory
          - sessions
```

### Memory Search Configuration
```yaml
memorySearch:
  enabled: true
  provider: openai  # openai | gemini | local | auto
  fallback: gemini  # Fallback provider
  model: text-embedding-3-small
  sources:
    - memory
    - sessions
  chunking:
    tokens: 500
    overlap: 50
  query:
    maxResults: 10
    minScore: 0.5
    hybrid:
      enabled: true
      vectorWeight: 0.7
      textWeight: 0.3
  sync:
    watch: true
    watchDebounceMs: 2000
    intervalMinutes: 30
    onSearch: true
    onSessionStart: true
```

---

## Next Steps

- [ ] Document channel plugins architecture
- [ ] Analyze routing and message flow
- [ ] Document config schema completely
- [ ] Trace a complete request lifecycle
