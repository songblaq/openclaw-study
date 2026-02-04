# Memory System - Summary

## Core Concept
Memory is **plain Markdown files** in workspace. Model only "remembers" what's written to disk.

## Memory Files

| File | Purpose | When to Load |
|------|---------|--------------|
| `memory/YYYY-MM-DD.md` | Daily log (append-only) | Today + yesterday at session start |
| `MEMORY.md` | Curated long-term memory | **Main session only** (never in groups) |

Location: `~/.openclaw/workspace/` (or configured workspace)

## When to Write
- Decisions, preferences, facts → `MEMORY.md`
- Day-to-day notes → `memory/YYYY-MM-DD.md`
- "Remember this" → Write it immediately (don't keep in RAM)

## Automatic Memory Flush

Before context compaction, OpenClaw runs a silent turn to remind the model to write memory.

```json5
{
  agents: {
    defaults: {
      compaction: {
        reserveTokensFloor: 20000,
        memoryFlush: {
          enabled: true,
          softThresholdTokens: 4000,
          systemPrompt: "Session nearing compaction. Store durable memories now.",
          prompt: "Write notes to memory/YYYY-MM-DD.md; reply NO_REPLY if nothing."
        }
      }
    }
  }
}
```

## Memory Search Tools

- `memory_search` - Semantic search over memory files
- `memory_get` - Read specific memory file by path

## Vector Search Config

```json5
{
  agents: {
    defaults: {
      memorySearch: {
        enabled: true,
        provider: "openai",  // openai | gemini | local
        model: "text-embedding-3-small",
        fallback: "openai"
      }
    }
  }
}
```

### Provider Auto-Selection
1. `local` if model path configured
2. `openai` if API key available
3. `gemini` if API key available
4. Disabled if none

## Hybrid Search (BM25 + Vector)

Combines semantic similarity + keyword relevance:

```json5
{
  agents: {
    defaults: {
      memorySearch: {
        query: {
          hybrid: {
            enabled: true,
            vectorWeight: 0.7,
            textWeight: 0.3
          }
        }
      }
    }
  }
}
```

## Extra Memory Paths

```json5
{
  agents: {
    defaults: {
      memorySearch: {
        extraPaths: ["../team-docs", "/srv/shared-notes/overview.md"]
      }
    }
  }
}
```

## Index Storage
- SQLite: `~/.openclaw/memory/<agentId>.sqlite`
- Uses sqlite-vec for fast vector search
- Auto-reindex on provider/model change

## QMD Backend (Experimental)

Local-first search with BM25 + vectors + reranking:

```json5
{
  memory: {
    backend: "qmd",
    qmd: {
      includeDefaultMemory: true,
      update: { interval: "5m" }
    }
  }
}
```

## Embedding Cache
Cache chunk embeddings to avoid re-embedding:

```json5
{
  agents: {
    defaults: {
      memorySearch: {
        cache: { enabled: true, maxEntries: 50000 }
      }
    }
  }
}
```

## Session Memory (Experimental)
Index session transcripts for search:

```json5
{
  agents: {
    defaults: {
      memorySearch: {
        experimental: { sessionMemory: true },
        sources: ["memory", "sessions"]
      }
    }
  }
}
```
