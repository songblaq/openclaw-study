# OpenClaw Design Patterns & Flow

> Phase 1 - Source Code Analysis: Entry Points, Flows, and Design Patterns
> Last Updated: 2026-02-04

## Entry Points & Execution Flow

### 1. CLI Entry Flow

```
openclaw.mjs
    │
    ├─ Enable Node.js compile cache
    │
    └─► dist/entry.js (compiled from entry.ts)
           │
           ├─ Set process title ("openclaw")
           ├─ Install warning filters
           ├─ Normalize environment
           ├─ Handle --no-color flag
           │
           ├─ ensureExperimentalWarningSuppressed()
           │     └─ Respawn with NODE_OPTIONS if needed
           │
           ├─ parseCliProfileArgs()
           │     └─ Extract --profile flag, apply env overrides
           │
           └─► cli/run-main.ts :: runCli()
                  │
                  ├─ loadDotEnv(), normalizeEnv()
                  ├─ ensureOpenClawCliOnPath()
                  ├─ assertSupportedRuntime()
                  │
                  ├─ tryRouteCli()  ← Fast path for special commands
                  │
                  ├─ enableConsoleCapture()
                  │
                  ├─► buildProgram() (Commander.js)
                  │      ├─ configureProgramHelp()
                  │      ├─ registerPreActionHooks()
                  │      └─ registerProgramCommands()
                  │
                  ├─ registerSubCliByName()  ← Lazy load subcommands
                  │
                  ├─ registerPluginCliCommands()
                  │
                  └─► program.parseAsync(argv)
```

### 2. Gateway Server Flow

```
openclaw gateway [--port N]
    │
    └─► commands/gateway/gateway.ts
           │
           ├─ startGatewaySidecars()
           │     ├─ startBrowserControlServerIfEnabled()
           │     ├─ startGmailWatcher()
           │     ├─ loadInternalHooks()
           │     ├─ startPluginServices()
           │     └─ scheduleRestartSentinelWake()
           │
           ├─► Gateway WebSocket Server (ws-connection.ts)
           │     ├─ HTTP listener (TLS optional)
           │     ├─ WebSocket upgrade handler
           │     └─ Connection authentication
           │
           ├─► Channel Plugins Start
           │     └─ Each channel (discord, telegram, etc.) connects
           │
           └─► Boot Check (boot.ts)
                 └─ Read BOOT.md, run one-time agent command
```

### 3. Message Routing Flow

```
Incoming Message (WhatsApp/Telegram/Discord/...)
    │
    ├─ Channel Plugin receives message
    │
    ├─► resolveAgentRoute() (routing/resolve-route.ts)
    │     │
    │     ├─ Extract: channel, accountId, peer (kind, id), guildId/teamId
    │     │
    │     ├─ listBindings(cfg) → Filter by channel & accountId
    │     │
    │     ├─ Match Priority:
    │     │   1. binding.peer       (exact peer match)
    │     │   2. binding.peer.parent (thread parent inheritance)
    │     │   3. binding.guild      (Discord guild)
    │     │   4. binding.team       (Slack team)
    │     │   5. binding.account    (specific account)
    │     │   6. binding.channel    (any account on channel)
    │     │   7. default            (resolveDefaultAgentId)
    │     │
    │     └─► Return: { agentId, sessionKey, mainSessionKey, matchedBy }
    │
    ├─► buildAgentSessionKey()
    │     Format: agent:<agentId>:<channel>:<peerKind>:<peerId>
    │
    └─► Agent handles message with resolved session
```

---

## Core Design Patterns

### 1. Request/Response Protocol

The Gateway uses a framed WebSocket protocol with three message types:

```typescript
// Request (client → server)
type RequestFrame = {
  type: "req";
  id: string;        // UUID for correlation
  method: string;    // RPC method name
  params?: unknown;  // Method parameters
};

// Response (server → client)
type ResponseFrame = {
  type: "res";
  id: string;        // Matches request id
  ok: boolean;       // Success flag
  payload?: unknown; // Result on success
  error?: string;    // Error message on failure
};

// Event (server → client, one-way)
type EventFrame = {
  event: string;     // Event type
  payload?: unknown; // Event data
  seq?: number;      // Sequence number for gap detection
};
```

**Implementation Pattern:**
```typescript
// Pending request tracking
const pending = new Map<string, { resolve, reject, timer }>();

// Send request with timeout
async function request(method, params, timeoutMs = 30000) {
  const id = randomUUID();
  const promise = new Promise((resolve, reject) => {
    const timer = setTimeout(() => reject(new Error("timeout")), timeoutMs);
    pending.set(id, { resolve, reject, timer });
  });
  ws.send(JSON.stringify({ type: "req", id, method, params }));
  return promise;
}

// Handle response
function handleResponse(frame: ResponseFrame) {
  const entry = pending.get(frame.id);
  if (entry) {
    clearTimeout(entry.timer);
    pending.delete(frame.id);
    frame.ok ? entry.resolve(frame.payload) : entry.reject(frame.error);
  }
}
```

### 2. Plugin Architecture

OpenClaw uses a plugin system for channel integrations and extensions:

```
extensions/
├── discord/           # Discord channel plugin
├── telegram/          # Telegram channel plugin
├── whatsapp/          # WhatsApp channel plugin
├── slack/             # Slack channel plugin
├── signal/            # Signal channel plugin
└── ...
```

**Plugin Interface:**
```typescript
interface ChannelPlugin {
  id: string;                        // Unique identifier
  name: string;                      // Display name
  
  // Lifecycle
  start(ctx: PluginContext): Promise<void>;
  stop(): Promise<void>;
  
  // Messaging
  sendMessage(params: SendParams): Promise<void>;
  
  // Optional capabilities
  canEdit?: boolean;
  canDelete?: boolean;
  canReact?: boolean;
  
  // CLI commands (optional)
  registerCommands?(program: Command): void;
}
```

**Plugin Loading:**
```typescript
// plugins/loader.ts
export function loadOpenClawPlugins(cfg: OpenClawConfig) {
  const plugins = [];
  
  // Built-in plugins (always loaded)
  plugins.push(discordPlugin);
  plugins.push(telegramPlugin);
  
  // Extension plugins (from extensions/ directory)
  for (const ext of discoverExtensions(cfg.extensionsDir)) {
    plugins.push(loadExtension(ext));
  }
  
  return { plugins, registry: buildRegistry(plugins) };
}
```

### 3. Session Key Management

Session keys uniquely identify conversation contexts:

```
Format: agent:<agentId>:<channel>:<peerKind>:<peerId>

Examples:
- agent:main:discord:dm:123456789
- agent:main:telegram:group:987654321
- agent:main:whatsapp:dm:15551234567
- agent:blaq:discord:channel:444333222

DM Scopes (configurable):
- main:           All DMs share one session
- per-peer:       Each peer gets own session
- per-channel-peer: Per channel + peer
- per-account-channel-peer: Full isolation
```

**Identity Links:**
```yaml
# Collapse multiple accounts to one identity
session:
  identityLinks:
    "user-main":
      - "telegram:dm:111222333"
      - "discord:dm:444555666"
      - "whatsapp:dm:15551234567"
```

### 4. Binding-based Routing

Agent routing is configured declaratively:

```yaml
bindings:
  # Route specific Discord channel to agent "blaq"
  - agentId: blaq
    match:
      channel: discord
      accountId: default
      peer:
        kind: channel
        id: "1467633382056788151"
  
  # Route entire Discord guild to agent "helper"
  - agentId: helper
    match:
      channel: discord
      guildId: "1465345516094099571"
  
  # Fallback: all Telegram to "main" agent
  - agentId: main
    match:
      channel: telegram
      accountId: "*"
```

### 5. Provider Abstraction (Embeddings)

Embedding providers are abstracted behind a common interface:

```typescript
interface EmbeddingProvider {
  name: string;
  model: string;
  dimensions: number;
  
  embed(texts: string[]): Promise<number[][]>;
  embedSingle(text: string): Promise<number[]>;
}

// Factory pattern for provider selection
function createEmbeddingProvider(cfg: MemorySearchConfig): EmbeddingProvider {
  switch (cfg.provider) {
    case "openai":
      return new OpenAIEmbeddingProvider(cfg);
    case "gemini":
      return new GeminiEmbeddingProvider(cfg);
    case "local":
      return new LocalEmbeddingProvider(cfg);
    default:
      return autoDetectProvider(cfg);
  }
}
```

**Graceful Degradation:**
```typescript
async function embedWithFallback(texts: string[]): Promise<number[][]> {
  try {
    return await primaryProvider.embed(texts);
  } catch (err) {
    log.warn(`Primary provider failed: ${err.message}`);
    if (fallbackProvider) {
      log.info("Falling back to secondary provider");
      return await fallbackProvider.embed(texts);
    }
    throw err;
  }
}
```

### 6. Event-Driven Updates

File and session changes trigger sync via events:

```typescript
// Transcript update events
const listeners = new Set<(update: { sessionFile: string }) => void>();

export function onSessionTranscriptUpdate(
  callback: (update: { sessionFile: string }) => void
): () => void {
  listeners.add(callback);
  return () => listeners.delete(callback);
}

export function emitSessionTranscriptUpdate(sessionFile: string) {
  for (const listener of listeners) {
    listener({ sessionFile });
  }
}

// Usage: Memory index watches for updates
const unsubscribe = onSessionTranscriptUpdate(({ sessionFile }) => {
  this.dirty = true;
  this.scheduleSync();
});
```

### 7. Lazy Command Registration

Commands are loaded on-demand to speed up CLI startup:

```typescript
// program/register.subclis.ts
export async function registerSubCliByName(program: Command, name: string) {
  const loaders: Record<string, () => Promise<void>> = {
    gateway: () => import("./commands/gateway").then(m => m.register(program)),
    agent: () => import("./commands/agent").then(m => m.register(program)),
    config: () => import("./commands/config").then(m => m.register(program)),
    // ...
  };
  
  const loader = loaders[name];
  if (loader) {
    await loader();
  }
}

// Only load the command being invoked
const primary = getPrimaryCommand(argv);  // e.g., "gateway"
if (primary) {
  await registerSubCliByName(program, primary);
}
```

### 8. Singleton with Cache Key

Resource managers use cache keys for instance reuse:

```typescript
const INDEX_CACHE = new Map<string, MemoryIndexManager>();

class MemoryIndexManager {
  static async get(params: {
    cfg: OpenClawConfig;
    agentId: string;
  }): Promise<MemoryIndexManager | null> {
    const key = buildCacheKey(params);
    
    const existing = INDEX_CACHE.get(key);
    if (existing) {
      return existing;
    }
    
    const instance = await MemoryIndexManager.create(params);
    INDEX_CACHE.set(key, instance);
    return instance;
  }
}

function buildCacheKey(params: { cfg: OpenClawConfig; agentId: string }): string {
  const { agentId } = params;
  const workspaceDir = resolveAgentWorkspaceDir(params.cfg, agentId);
  const settings = JSON.stringify(params.cfg.agents?.defaults?.memorySearch ?? {});
  return `${agentId}:${workspaceDir}:${settings}`;
}
```

### 9. Atomic Database Swap

Safe reindexing via temp database:

```typescript
async function rebuildIndex() {
  const tempDbPath = `${dbPath}.tmp`;
  
  // Build new index in temp location
  const tempDb = await createDatabase(tempDbPath);
  await populateIndex(tempDb, files);
  await tempDb.close();
  
  // Atomic swap
  await fs.rename(tempDbPath, dbPath);
  
  // Reopen with new data
  this.db = await openDatabase(dbPath);
}
```

### 10. Tick Watchdog

Connection health monitoring via periodic pings:

```typescript
class GatewayClient {
  private tickInterval?: NodeJS.Timeout;
  private lastTickTime = Date.now();
  
  private startTickWatchdog() {
    this.tickInterval = setInterval(() => {
      const now = Date.now();
      const elapsed = now - this.lastTickTime;
      
      if (elapsed > TICK_TIMEOUT_MS) {
        this.handleConnectionLost("tick timeout");
        return;
      }
      
      this.sendTick();
    }, TICK_INTERVAL_MS);
  }
  
  private handleTick() {
    this.lastTickTime = Date.now();
  }
}
```

---

## Configuration Patterns

### Hierarchical Config Resolution

```yaml
agents:
  defaults:           # Applied to all agents
    workspace: ~/.openclaw/workspace
    model:
      primary: anthropic/claude-sonnet-4-20250514
    memorySearch:
      enabled: true
  
  list:
    - id: main        # Inherits from defaults
      name: Blaq
    
    - id: helper      # Overrides defaults
      model:
        primary: openai/gpt-4o
```

**Resolution Logic:**
```typescript
function resolveAgentConfig(cfg: OpenClawConfig, agentId: string): ResolvedAgentConfig {
  const defaults = cfg.agents?.defaults ?? {};
  const agent = cfg.agents?.list?.find(a => a.id === agentId) ?? {};
  
  return {
    ...defaults,
    ...agent,
    model: {
      ...defaults.model,
      ...agent.model,
    },
    memorySearch: {
      ...defaults.memorySearch,
      ...agent.memorySearch,
    },
  };
}
```

### Environment Variable Overrides

```typescript
// infra/env.ts
export function normalizeEnv() {
  // CLI profile can override env vars
  if (process.env.OPENCLAW_PROFILE) {
    applyProfileEnv(process.env.OPENCLAW_PROFILE);
  }
  
  // Normalize boolean-ish values
  for (const key of Object.keys(process.env)) {
    if (key.startsWith("OPENCLAW_")) {
      process.env[key] = normalizeBoolEnv(process.env[key]);
    }
  }
}

export function isTruthyEnvValue(value: string | undefined): boolean {
  return ["1", "true", "yes", "on"].includes((value ?? "").toLowerCase());
}
```

---

## Error Handling Patterns

### Graceful Shutdown

```typescript
process.on("SIGINT", async () => {
  log.info("Received SIGINT, shutting down...");
  
  // Stop accepting new connections
  server.close();
  
  // Drain existing connections
  await Promise.all(connections.map(c => c.close()));
  
  // Stop plugins
  await Promise.all(plugins.map(p => p.stop()));
  
  // Flush logs
  await logger.flush();
  
  process.exit(0);
});
```

### Retry with Exponential Backoff

```typescript
async function retryWithBackoff<T>(
  fn: () => Promise<T>,
  options: { maxRetries?: number; baseDelayMs?: number } = {}
): Promise<T> {
  const { maxRetries = 5, baseDelayMs = 1000 } = options;
  
  for (let attempt = 0; attempt < maxRetries; attempt++) {
    try {
      return await fn();
    } catch (err) {
      if (attempt === maxRetries - 1) throw err;
      
      const delay = baseDelayMs * Math.pow(2, attempt);
      const jitter = Math.random() * delay * 0.1;
      await sleep(delay + jitter);
    }
  }
  
  throw new Error("unreachable");
}
```

---

## Summary

| Pattern | Purpose | Location |
|---------|---------|----------|
| Request/Response Protocol | Structured RPC over WebSocket | `gateway/protocol/` |
| Plugin Architecture | Modular channel integrations | `plugins/`, `extensions/` |
| Session Key Management | Conversation context isolation | `routing/session-key.ts` |
| Binding-based Routing | Declarative agent assignment | `routing/resolve-route.ts` |
| Provider Abstraction | Swappable embedding backends | `memory/embeddings-*.ts` |
| Event-Driven Updates | Reactive sync triggers | `sessions/transcript-events.ts` |
| Lazy Command Registration | Fast CLI startup | `cli/program/register.subclis.ts` |
| Singleton with Cache Key | Resource reuse | `memory/manager.ts` |
| Atomic Database Swap | Safe reindexing | `memory/sqlite.ts` |
| Tick Watchdog | Connection health monitoring | `gateway/client.ts` |

---

*Generated by Blaq for OpenClaw analysis project*
*Phase 1 Complete: Architecture analysis done*
