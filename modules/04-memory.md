# Module 4: Memory
> The most misunderstood module — it's both a tool the LLM calls and infrastructure the Agent Core reads automatically.

## What Is It

Memory gives your Agent a past. Without it, every conversation starts from zero. The user tells the Agent their name on Monday; on Tuesday the Agent asks again.

But Memory is not just "a database the Agent queries." That's the part everyone gets right. The part everyone gets wrong is the **dual role**. We verified this in all 4 projects — no exceptions:

**Role 1: Memory as a tool.** The LLM decides "I should save this" or "I need to recall X" and actively calls `memory_save` or `memory_search`. This is LLM-initiated. The Agent Core treats these like any other tool call — dispatch, execute, return result.

**Role 2: Memory as infrastructure.** Before each conversation turn, Agent Core builds the system prompt. Part of that prompt is injected memories — relevant context pulled from the Memory Store without the LLM asking. This is Core-initiated. The LLM never sees a `memory_search` call; it just finds relevant memories already sitting in its system prompt.

In the Hub-and-Spoke architecture, Memory is a **peer service** to Agent Core, sitting alongside Model, Tools, and Context Manager. But unlike Tools (which Core only calls when the LLM requests it) and Model (which Core calls every turn), Memory gets called from **both directions**: the LLM calls it through tool dispatch, and Core reads it during prompt construction.

The **Memory Store** hangs below Memory — vector databases, full-text search indexes, or plain files on disk. Memory owns the read/write interface; the store is just the backend.

## What Problem Does It Solve

**Cross-session continuity.** A coding Agent that forgets your project structure every restart is useless for multi-day tasks. hermes stores 8 different types of memory across pluggable backends. Claude Code persists memories to `~/.claude/memory/*.md` files that survive across sessions.

**Reducing repetition.** Users repeat themselves. "I prefer TypeScript." "Always use Tailwind." "My API key is in .env." Memory saves these once and auto-injects them, eliminating the need for users to restate preferences every conversation.

**Grounding the LLM.** Without memory, the LLM confabulates freely. With relevant memories injected into the system prompt, the model's responses are grounded in actual project context, user preferences, and prior decisions.

**Scaling context beyond the window.** A 200K-token context window is large but not infinite. A 6-month project accumulates millions of tokens of conversation history. Memory acts as a compressed index — storing the important bits and retrieving them on demand, rather than stuffing everything into the context.

**The dual-role gap.** If you build memory-as-tool-only, the LLM must remember to search memory every turn. It won't. LLMs are unreliable at self-prompting for information retrieval. If you build memory-as-infrastructure-only, the LLM can never explicitly save something for later. You need both roles, designed together from day one.

## How the 4 Projects Do It

### hermes-agent

**The strongest abstraction.** hermes defines a `MemoryProvider` ABC (Abstract Base Class) with 8 pluggable backends. The abstraction is clean enough that switching from SQLite to Redis requires changing one config line.

```python
# hermes: memory/provider.py (simplified)
class MemoryProvider(ABC):
    @abstractmethod
    async def store(self, key: str, content: str, metadata: dict) -> None: ...

    @abstractmethod
    async def search(self, query: str, limit: int = 5) -> list[MemoryItem]: ...

    @abstractmethod
    async def delete(self, key: str) -> bool: ...
```

**8 backends**: SQLite (default), Redis, PostgreSQL, Pinecone, Weaviate, Qdrant, ChromaDB, and plain file system. Each implements `MemoryProvider`. You pick one in the config.

**Dual role implementation**:
- *As tool*: `memory_save` and `memory_search` are registered tools that the LLM calls like any other tool. They delegate to the active MemoryProvider.
- *As infrastructure*: `prompt_builder` queries the MemoryProvider at the start of each turn, retrieves recent and relevant memories, and injects them into the system prompt under a `## Relevant Memories` section.

**Memory types**: hermes distinguishes between factual memories ("the user's name is Eli"), procedural memories ("this project uses pnpm, not npm"), and episodic memories ("yesterday we debugged the auth flow and found a race condition"). The LLM's `memory_save` call includes a `type` field.

**Pros**: Best abstraction in all 4 projects. The MemoryProvider ABC is a textbook example of the Strategy pattern done right. Swapping backends is painless. The type system for memories adds useful structure.

**Cons**: 8 backends means 8 integration points to maintain. The SQLite backend is well-tested; some exotic backends have edge cases. Also, hermes doesn't do background memory tidying — memories accumulate and can become noisy over time.

### openclaw

**Dual registration, done right.** openclaw separates the Memory module into `memory-core` (the engine) and `memory-lancedb` (the default vector store). The key insight is **explicit dual registration**: memory registers itself both as a tool provider and as a prompt builder hook.

```typescript
// openclaw: memory-core (simplified)
export function onActivate(api: PluginAPI) {
  // Role 1: Register as tools
  api.registerTool({
    name: "memory_save",
    execute: async (params) => { /* save to LanceDB */ }
  });
  api.registerTool({
    name: "memory_search",
    execute: async (params) => { /* search LanceDB */ }
  });

  // Role 2: Register as prompt builder
  api.registerPromptBuilder({
    name: "memory-context",
    priority: 50,
    build: async (context) => {
      const relevant = await searchRelevant(context.lastMessage);
      return formatAsSystemPrompt(relevant);
    }
  });
}
```

**"Dreaming" — background memory tidying.** openclaw runs a background process that periodically reviews stored memories, merges duplicates, resolves contradictions, and re-ranks by relevance. This is inspired by how human memory consolidation works during sleep. The Agent doesn't need to be in a conversation for dreaming to happen.

**Memory Store**: LanceDB (embedded vector database). Chosen for local-first operation — no external server needed. Supports both vector similarity search and metadata filtering.

**Pros**: The dual registration pattern makes the dual role explicit in code, not implicit in convention. Dreaming solves the memory-rot problem that hermes doesn't address. LanceDB is a pragmatic choice — fast, embedded, zero ops.

**Cons**: Dreaming is computationally expensive (it calls the LLM to re-evaluate memories). If you're on a limited API budget, it can eat tokens. The memory-core/memory-lancedb split adds packaging complexity.

### NemoClaw

NemoClaw doesn't have its own memory system. It's a security sandbox runtime that wraps OpenClaw. Memory is not applicable here — but NemoClaw's credential sanitization is relevant: any memory that stores shell output or API responses should strip secrets before persisting. NemoClaw does this at the OS level with regex patterns for API keys, Bearer tokens, and passwords. Memory modules should do the same at the application level.

### Claude Code

**Two independent subsystems.** Claude Code's memory is split into two parts that barely talk to each other:

**Subsystem 1: SessionMemory.** Maintains the in-session conversation state. Messages, tool calls, tool results — all tracked in memory during the session. Persisted to JSONL files for history. This is short-term memory.

**Subsystem 2: extractMemories (background agent).** A separate, forked Agent process that runs in the background. It reads conversation transcripts and extracts salient facts into `~/.claude/memory/*.md` files. These files are loaded into the system prompt on next startup. The extraction happens asynchronously — it doesn't block the main conversation loop.

```
Main Agent Loop                    Background Extractor
     │                                     │
     │ ── conversation happens ──→         │
     │                                     │ (fork)
     │                                     │ reads transcript
     │                                     │ extracts facts
     │                                     │ writes to ~/.claude/memory/
     │                                     │
     │ ← next startup, Core loads ──────── │
     │    memory files into prompt          │
```

**The `/remember` command.** Users can explicitly tell Claude Code to remember something. This writes directly to the memory file without waiting for background extraction. It's the manual override for Role 1 (memory as tool).

**What Claude Code doesn't do**: It doesn't create structured skill files from memories. The extracted memories are flat facts ("this project uses pnpm") — not reusable methodologies. This is why Claude Code scores 3/10 on learning loop completeness vs hermes's 10/10.

**Pros**: Background extraction is non-blocking. The user never waits for memory operations. The forked agent approach keeps memory logic completely isolated from the main loop — a crash in extraction doesn't affect the conversation.

**Cons**: Two independent subsystems means two codebases to maintain and two sets of bugs. The extraction quality depends on the background agent's prompt — if it misses something, the user has no way to know until next session. No vector search; memories are loaded in full, relying on the LLM to find relevance.

## Comparison Table

| Dimension | hermes | openclaw | NemoClaw | Claude Code |
|-----------|--------|----------|----------|-------------|
| **Architecture** | MemoryProvider ABC | memory-core + memory-lancedb plugin | N/A | SessionMemory + extractMemories |
| **Backend count** | 8 pluggable | 1 (LanceDB) | N/A | File system (MD files) |
| **Dual role** | Implicit (tools + prompt_builder both use Provider) | Explicit dual registration (tool + promptBuilder) | N/A | Two independent subsystems |
| **As tool** | `memory_save`, `memory_search` | `memory_save`, `memory_search` | N/A | `/remember` command + extractMemories |
| **As infrastructure** | prompt_builder queries Provider each turn | registerPromptBuilder hook, priority 50 | N/A | Load `~/.claude/memory/*.md` at startup |
| **Vector search** | Yes (Pinecone, Weaviate, Qdrant, ChromaDB) | Yes (LanceDB) | N/A | No (full file load) |
| **Background tidying** | No | Yes ("dreaming" consolidation) | N/A | Yes (background fork extraction) |
| **Memory types** | Factual, procedural, episodic | Untyped (metadata tags) | N/A | Flat facts only |
| **Credential sanitization** | No | No | OS-level regex | No |
| **Abstraction quality** | Best (Strategy pattern ABC) | Good (plugin boundary) | N/A | Weak (two disconnected systems) |

## Best Practices

**If you only do one thing**: Design for dual role from day one. Don't build memory-as-tool-only and plan to bolt on infrastructure injection later. The two roles share the same store and the same data model. Splitting them after the fact means either duplicating data or building brittle bridges.

**Six rules from the 4 projects**:

1. **One interface, two callers.** Your MemoryProvider (or equivalent) should expose `save()`, `search()`, and `delete()`. Agent Core calls `search()` during prompt construction. The tool dispatcher calls all three on LLM request. Same interface, two entry points.

2. **Inject sparingly.** Don't dump all memories into the system prompt. hermes retrieves the top 5 by relevance. openclaw's prompt builder has a priority system. If you inject 50 memories, you're wasting 2,000+ tokens and the LLM will ignore most of them anyway. Rank by recency and relevance, cap at 5-10.

3. **Sanitize before storing.** Shell output contains API keys. Conversation transcripts contain passwords users accidentally typed. Claude Code doesn't sanitize. hermes doesn't sanitize. NemoClaw sanitizes at the OS level but not the memory level. Learn from what they all missed: run a credential-stripping pass before `save()`.

4. **Background processing is worth the complexity.** openclaw's dreaming and Claude Code's background extraction both solve the same problem: memory operations shouldn't block the user. If you can fork a background process or run a periodic job, do it. The implementation cost is modest; the UX improvement is substantial.

5. **Type your memories.** hermes distinguishes factual, procedural, and episodic. This matters for retrieval: when the LLM asks "how did we solve this before?" it needs episodic memories. When it needs "what framework does this project use?" it needs factual. Untyped memory stores return a mix and waste context tokens on irrelevant results.

6. **Plan for memory rot.** Memories go stale. The user switches from npm to pnpm, but the old "uses npm" memory is still in the store. openclaw's dreaming process handles this by periodically re-evaluating memories. If you don't build dreaming, at least add a `last_accessed` timestamp and a `confidence` score that decays over time.

## Step-by-Step: How to Build It

### Step 1: Define the MemoryProvider interface

```typescript
// contracts/memory.ts
interface MemoryItem {
  id: string;
  content: string;
  type: "factual" | "procedural" | "episodic";
  createdAt: Date;
  lastAccessed: Date;
  accessCount: number;
  metadata: Record<string, unknown>;
}

interface MemoryProvider {
  save(content: string, type: MemoryItem["type"], metadata?: Record<string, unknown>): Promise<string>;  // returns id
  search(query: string, options?: { limit?: number; type?: MemoryItem["type"] }): Promise<MemoryItem[]>;
  delete(id: string): Promise<boolean>;
  getRecent(limit: number): Promise<MemoryItem[]>;
}
```

**Decision point**: Vector search from day one, or start with full-text search? If your Agent runs locally and you expect fewer than 10,000 memories, SQLite FTS5 is more than enough. Add vector search when keyword matching starts returning irrelevant results. hermes supports both; Claude Code gets away with neither (just loading full files).

### Step 2: Build the simplest backend first

Start with SQLite. It's embedded, zero-config, battle-tested.

```typescript
// memory/sqlite-provider.ts
class SQLiteMemoryProvider implements MemoryProvider {
  private db: Database;

  constructor(dbPath: string) {
    this.db = new Database(dbPath);
    this.db.exec(`
      CREATE TABLE IF NOT EXISTS memories (
        id TEXT PRIMARY KEY,
        content TEXT NOT NULL,
        type TEXT NOT NULL,
        created_at TEXT NOT NULL,
        last_accessed TEXT NOT NULL,
        access_count INTEGER DEFAULT 0,
        metadata TEXT DEFAULT '{}'
      );
      CREATE VIRTUAL TABLE IF NOT EXISTS memories_fts
        USING fts5(content, content=memories, content_rowid=rowid);
    `);
  }

  async search(query: string, options?: SearchOptions): Promise<MemoryItem[]> {
    const limit = options?.limit ?? 5;
    // FTS5 search with relevance ranking
    const rows = this.db.prepare(`
      SELECT m.*, rank
      FROM memories_fts f
      JOIN memories m ON f.rowid = m.rowid
      WHERE memories_fts MATCH ?
      ORDER BY rank
      LIMIT ?
    `).all(query, limit);

    // Update access timestamps
    for (const row of rows) {
      this.db.prepare(`
        UPDATE memories SET last_accessed = ?, access_count = access_count + 1 WHERE id = ?
      `).run(new Date().toISOString(), row.id);
    }

    return rows.map(toMemoryItem);
  }

  // save() and delete() implementations...
}
```

### Step 3: Register memory as a tool (Role 1)

```typescript
// memory/tools.ts
import { register } from "../tools/registry";
import { getProvider } from "./index";

register({
  name: "memory_save",
  description: "Save information to long-term memory for future reference",
  parameters: {
    type: "object",
    properties: {
      content: { type: "string", description: "What to remember" },
      type: { type: "string", enum: ["factual", "procedural", "episodic"] },
    },
    required: ["content", "type"]
  },
  isConcurrencySafe: true,
  execute: async (params) => {
    const { content, type } = params as { content: string; type: MemoryItem["type"] };
    const sanitized = stripCredentials(content);  // ← never skip this
    const id = await getProvider().save(sanitized, type);
    return { success: true, output: { id, message: "Saved to memory" } };
  }
});

register({
  name: "memory_search",
  description: "Search long-term memory for relevant information",
  parameters: {
    type: "object",
    properties: {
      query: { type: "string", description: "What to search for" },
      limit: { type: "number", description: "Max results", default: 5 },
    },
    required: ["query"]
  },
  isConcurrencySafe: true,
  execute: async (params) => {
    const { query, limit } = params as { query: string; limit?: number };
    const results = await getProvider().search(query, { limit });
    return { success: true, output: results };
  }
});
```

### Step 4: Wire memory as infrastructure (Role 2)

This is where most builders stop — and where the dual role becomes real.

```typescript
// core/prompt-builder.ts  (inside Agent Core)
async function buildSystemPrompt(lastUserMessage: string): Promise<string> {
  const parts: string[] = [];

  // 1. Identity and instructions
  parts.push(loadIdentityPrompt());

  // 2. Available tools
  parts.push(formatToolDescriptions(getAllTools()));

  // 3. ← MEMORY INJECTION (Role 2)
  const memories = await memoryProvider.search(lastUserMessage, { limit: 5 });
  if (memories.length > 0) {
    parts.push("## Relevant Context from Memory");
    for (const mem of memories) {
      parts.push(`- [${mem.type}] ${mem.content}`);
    }
  }

  // 4. Loaded skills (from Skill Store)
  parts.push(loadRelevantSkills());

  return parts.join("\n\n");
}
```

**Decision point**: Search by last user message, or by conversation summary? Searching by the latest message works for direct questions ("what framework does this project use?"). For complex multi-turn conversations, you may want to search by a summary of the last 3-5 turns. Start simple — search by last message. Add summary-based search if retrieval quality drops.

### Step 5: Add credential sanitization

```typescript
// memory/sanitize.ts
const PATTERNS = [
  /(?:api[_-]?key|token|secret|password|bearer)\s*[:=]\s*["']?[\w\-\.]{20,}/gi,
  /(?:sk|pk|ak|rk)[-_][a-zA-Z0-9]{20,}/g,    // common API key prefixes
  /Bearer\s+[a-zA-Z0-9\-._~+\/]+=*/g,          // Bearer tokens
  /-----BEGIN (?:RSA |EC )?PRIVATE KEY-----/g,   // PEM keys
];

function stripCredentials(content: string): string {
  let sanitized = content;
  for (const pattern of PATTERNS) {
    sanitized = sanitized.replace(pattern, "[REDACTED]");
  }
  return sanitized;
}
```

Run this on every `save()` call. No exceptions. This is what NemoClaw does at the OS level — you should do it at the application level too.

### Step 6: Add background extraction (optional but recommended)

Instead of relying solely on the LLM calling `memory_save`, periodically extract memories from conversation transcripts.

```typescript
// memory/extractor.ts
async function extractMemories(transcript: Message[]): Promise<void> {
  // Run in background — don't block the main loop
  const extraction = await callLLM({
    system: "Extract important facts, preferences, and decisions from this conversation. Return as JSON array.",
    messages: [{ role: "user", content: formatTranscript(transcript) }],
    model: "fast-cheap-model"  // don't use the expensive model for extraction
  });

  const items = JSON.parse(extraction);
  for (const item of items) {
    const existing = await provider.search(item.content, { limit: 1 });
    if (existing.length === 0 || existing[0].content !== item.content) {
      await provider.save(stripCredentials(item.content), item.type);
    }
  }
}
```

**Decision point**: Run extraction after every conversation, or on a schedule? Claude Code does it after every session. openclaw's dreaming runs periodically. For most Agents, after-session extraction is simpler and sufficient. Schedule-based extraction makes sense when you have many short conversations per day.

## Common Pitfalls

**1. Building tool-only memory.** You register `memory_save` and `memory_search` as tools, pat yourself on the back, and ship. Then you realize the LLM only calls `memory_search` about 30% of the time it should. Why? Because the model doesn't know what it doesn't know. If the user mentioned their preference 3 sessions ago, the LLM has no signal that a memory exists — it won't think to search. Infrastructure injection (Role 2) solves this by proactively surfacing relevant memories. Build both roles from day one.

**2. Injecting too much.** You search memory and inject the top 20 results into the system prompt. That's 1,500 tokens of context the LLM has to wade through every single turn. Most of it is irrelevant. hermes caps at 5. Start there. If the LLM consistently misses relevant context, bump to 10. Never go above 15 — at that point you're hurting more than helping.

**3. No deduplication.** The user says "I prefer TypeScript" in session 1, 4, 7, and 12. You now have 4 copies of the same fact. When you inject memories, you waste 3x the tokens. openclaw's dreaming process merges duplicates. At minimum, check for near-duplicates before saving — a simple string similarity threshold (Jaccard > 0.8) catches the obvious cases.

**4. Stale memories contradicting reality.** "Project uses npm" was saved 3 months ago. The team switched to pnpm last month. The LLM now gets conflicting signals. This is memory rot, and none of the 4 projects solve it perfectly. hermes doesn't address it. openclaw's dreaming helps but isn't guaranteed to catch everything. Your best defense: add a `confidence` field that decays over time, and sort results by `confidence * relevance` rather than relevance alone.

**5. Treating Memory Store choice as a day-one decision.** You spend a week evaluating Pinecone vs Weaviate vs ChromaDB before writing a single line of Agent code. Stop. hermes's MemoryProvider ABC proves you can swap backends later. Start with SQLite FTS5. It handles up to ~50,000 memories with sub-millisecond search. Add vector search when (if) you outgrow it.

**6. Forgetting to sanitize.** The user accidentally pastes an API key into chat. The Agent saves it to memory. Next session, it gets injected into the system prompt and potentially leaked through tool outputs. Neither hermes nor Claude Code sanitize memory content. NemoClaw sanitizes shell output but not memory. Be better than all of them — sanitize on write.

**7. Two disconnected systems.** Claude Code's SessionMemory and extractMemories barely communicate. The session system doesn't know what the extractor saved. The extractor doesn't know what the session system considers important. This works because Claude Code's memory needs are simple (flat facts). For a more sophisticated Agent, you want a single MemoryProvider serving both roles — which is exactly what hermes and openclaw do.

## Cross-Module Contracts

### What Memory expects from other modules

| Source Module | What Memory Needs | Contract |
|--------------|------------------|----------|
| **Agent Core** | The current user message (for relevance search during prompt build) | `string` — the latest user message or conversation summary |
| **Agent Core** | Tool dispatch for memory_save/memory_search calls | Standard tool dispatch (same as any other tool) |
| **Tools** | Registration mechanism for memory tools | `register(tool: Tool)` — memory registers itself through the tool registry |
| **Context Manager** | Awareness that injected memories consume token budget | Memory injection happens before Context Manager's budget calculation |

### What Memory provides to other modules

| Target Module | What Memory Provides | Contract |
|--------------|---------------------|----------|
| **Agent Core** | Formatted memory injection for system prompt | `search(query, options) → MemoryItem[]` — Core calls this during prompt build |
| **Agent Core** | Tool execution results for memory_save/memory_search | Standard `ToolResult` format |
| **Tools** | Nothing directly — Memory and Tools are peer services | (no direct dependency, but memory tools are registered in the tool registry) |
| **Context Manager** | Token count of injected memories (for budget tracking) | Implicit — Core includes memory content in the prompt, Context Manager measures the total |

### The dual-role wiring diagram

```
                          ┌──────────────────┐
                          │   Agent Core     │
                          │  (prompt builder) │
                          └───────┬──────────┘
                                  │
                    ┌─────────────┼─────────────┐
                    │             │              │
              Role 2 (read)      │        Role 1 (tool calls)
              "inject relevant   │        "LLM called memory_save"
               memories into     │        "LLM called memory_search"
               system prompt"    │
                    │             │              │
                    ▼             │              ▼
              ┌──────────────────┴────────────────┐
              │         MemoryProvider             │
              │    save() / search() / delete()    │
              └──────────────────┬─────────────────┘
                                 │
                                 ▼
                          ┌────────────┐
                          │ Memory     │
                          │ Store      │
                          │ (SQLite /  │
                          │  Vector DB │
                          │  / Files)  │
                          └────────────┘
```

Both roles hit the same MemoryProvider. Both roles read/write the same Memory Store. This is the key architectural insight. Two callers, one interface, one store.
