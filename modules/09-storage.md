# Module 9: Storage

> Persistence is not a horizontal layer. It's vertical — each service owns its own data.

---

## What Is It

Storage is **not** a standalone module in the same sense as Agent Core or Tools. There is no `StorageService` class. There is no `DataLayer` abstraction. There is no central database that all modules talk to.

Instead, each service in the Hub-and-Spoke architecture has its own storage backend — or none at all:

```
Agent Core ───→ Session Store  (conversation history)
             ───→ Config Store   (agent configuration)
Memory ────────→ Memory Store   (long-term memory, vector DB / FTS)
Tools ─────────→ Skill Store    (skill files, MD/YAML)
Model ─────────→ nothing        (stateless, calls external APIs)
Context Mgr ───→ nothing        (operates on in-memory messages array)
```

Physically, these might share a single SQLite file. hermes does exactly this — one `.db` file with tables for sessions, memories, skills. But logically, every piece of data has exactly one owner. Session history belongs to Agent Core. Long-term memories belong to Memory. Skill files belong to Tools.

**Why this matters:** Drawing persistence as an independent horizontal layer — a `DataLayer` that sits below everything and provides generic CRUD — is architecturally wrong. We made this mistake in the initial architecture model and it took four rounds of correction to fix. The problem is that a generic `DataLayer` implies shared ownership, shared schema, and shared access patterns. In practice, Agent Core reads/writes conversations in append-only streams, Memory does vector similarity search, and Tools reads flat files from disk. These have nothing in common. Forcing them through a unified abstraction creates unnecessary coupling and zero reuse.

---

## What Problem Does It Solve

Each store solves a specific persistence problem for its owning service:

**Session Store — "What happened in this conversation?"**
Conversation history must survive process restarts. If the Agent crashes mid-task, the user expects to resume where they left off. Session Store persists the messages array — user messages, assistant messages, tool calls, tool results — keyed by session ID.

**Config Store — "How is this Agent configured?"**
Agent identity, model selection, tool permissions, memory settings, custom instructions. These change rarely but must be readable by multiple services. Config Store is read-heavy, write-rare, shared-read.

**Memory Store — "What does this Agent remember long-term?"**
Extracted facts, user preferences, project context, learned patterns. Memory Store supports semantic search (vector similarity or full-text search) because the retrieval question is "what's relevant to this query?" not "give me record #47."

**Skill Store — "What has this Agent learned to do?"**
Skill files — Markdown or YAML documents describing procedures the Agent has created from experience. Tools manages them (CRUD). Agent Core consumes them (loads into system prompt). Skill Store is the simplest backend: just files on disk.

---

## How the 4 Projects Do It

### hermes-agent: One SQLite, Four Logical Stores

hermes is the clearest example of the vertical storage model.

**Physical layer:** A single SQLite file at `~/.hermes/hermes.db`. One database, multiple tables, each owned by a different service.

**Session Store:** The `conversations` table stores serialized message arrays. Keyed by session ID and timestamp. Agent Core reads and writes this directly. Append-only during a session; full history available for resumption.

**Config Store:** YAML files at `~/.hermes/config/`. Agent configuration, model settings, tool preferences. Loaded at startup, hot-reloadable.

**Memory Store:** Eight pluggable backends behind a `MemoryProvider` ABC. SQLite is the default, but Redis, filesystem, and vector DB backends are available. Memory service owns the schema and access patterns. Other services go through Memory's API, never touching the tables directly.

The 8 backends: `sqlite`, `redis`, `filesystem`, `json`, `yaml`, `vector` (ChromaDB), `postgres`, `custom`. In practice, most deployments use `sqlite` — the others exist because hermes's `MemoryProvider` abstraction makes adding new backends trivial.

**Skill Store:** Flat files at `~/.hermes/skills/*.md`. YAML frontmatter for metadata (name, trigger conditions, version). Markdown body for the skill content. The `skill_manage` tool handles CRUD. `prompt_builder` scans the directory each turn and injects relevant skills.

**Key insight:** Despite sharing one SQLite file, each service's data is cleanly separated. Memory doesn't read the conversations table. Agent Core doesn't write to the memory tables. The single file is a deployment convenience, not an architectural coupling.

### openclaw: Separation by Package

openclaw's storage is distributed across its architecture, with cleaner physical boundaries than hermes.

**Session Store:** Managed by `pi-agent-core` (the embedded agent runtime). Conversations are persisted per-lane (openclaw's concurrency unit). Each lane gets its own conversation history, isolated from other lanes.

**Config Store:** Configuration lives in the control plane. Agents, channels, tool permissions — all managed through the WebSocket-based management interface. Config changes propagate to running agents via config drift detection (the system notices when config has changed and reloads without restart).

**Memory Store:** Dual-registration pattern. Memories are registered both as searchable data (for retrieval) and as tool metadata (so the LLM knows what's available). openclaw also has a "dreaming" system — background processing that consolidates and reorganizes memories when the Agent is idle.

**Skill Store:** Workspace-level skills. Manual management — no auto-creation like hermes. Skills are loaded per-workspace, so different deployments can have different skill sets.

**Architecture difference:** openclaw's storage is more distributed than hermes's. No single SQLite file. Each subsystem manages its own persistence independently. This makes the system harder to inspect (you can't just open one database to see everything) but easier to scale (each store can be deployed independently).

### Claude Code: Files + SQLite Hybrid

Claude Code uses the simplest storage model of the three Agents, matching its single-user local deployment model.

**Session Store:** JSONL files for conversation history. Each session is a file. Simple to inspect (just `cat` the file), simple to back up (just `cp`). SQLite is used for indexing and fast lookups, but the source of truth is the JSONL.

**Config Store:** JSON files in the project directory and user home directory. `.claude/settings.json` for project config. `~/.claude/` for global config. No database, no control plane — just files.

**Memory Store:** Two subsystems. `SessionMemory` tracks in-session state (messages, tool calls). Long-term memory goes to Markdown files at `~/.claude/memory/*.md`. Memory extraction runs automatically — after significant interactions, the system extracts key facts and appends them to the memory files. No vector search, no FTS — just loads the full file into context. This works because Claude Code targets single-user scenarios where the memory file stays under a few thousand tokens.

**Skill Store:** `~/.claude/skills/` directory. Semi-automated — the `/remember` command creates memory entries, and users can manually create skill files. Less sophisticated than hermes's auto-creation but functional for the use case.

### NemoClaw

NemoClaw is not an Agent — it's a security sandbox runtime. It has no conversation history, no memory, no skills. Its storage is limited to sandbox configuration (which goes through Infrastructure, not the Agent storage model).

One relevant detail: NemoClaw's Landlock filesystem policies define what the Agent inside the sandbox can persist and where. This means NemoClaw indirectly controls where Session Store, Memory Store, and Skill Store can write. Config is always mounted read-only.

---

## Comparison Table

| Aspect | hermes | openclaw | Claude Code | NemoClaw |
|--------|--------|----------|-------------|----------|
| **Session Store** | SQLite `conversations` table | pi-agent-core per-lane | JSONL files + SQLite index | N/A |
| **Config Store** | YAML files | Control plane (WebSocket) | JSON files (project + global) | Read-only mount (Landlock) |
| **Memory Store** | SQLite default, 8 pluggable backends | Dual-registration + dreaming | Markdown files, loaded in full | N/A |
| **Skill Store** | `~/.hermes/skills/*.md` (auto-created) | Workspace skills (manual) | `~/.claude/skills/` (semi-auto) | N/A |
| **Physical layout** | Single SQLite file | Distributed per-subsystem | Files + SQLite hybrid | N/A |
| **Schema ownership** | Each service owns its tables | Each package owns its data | Each subsystem owns its files | N/A |
| **Vector search** | Optional (ChromaDB backend) | Optional (dual-registration) | None | N/A |
| **Full-text search** | SQLite FTS5 | Built-in | None (loads full file) | N/A |
| **Hot reload** | Config yes, schema no | Config drift detection | Config yes | N/A |

---

## Best Practices

**Use SQLite for everything in v1.** One file. Zero infrastructure. Sub-millisecond reads. Handles tens of thousands of records without breaking a sweat. hermes proves this works for production Agent workloads. Don't evaluate Postgres, Redis, Pinecone, or ChromaDB on day one. You are not Google. You do not have Google's scaling problems.

**Each service owns its own tables or schemas.** Agent Core reads and writes the `sessions` table. Memory reads and writes the `memories` table. Neither touches the other's data. If Memory needs conversation history, it asks Agent Core through a function call, not by reading the sessions table directly. This is the vertical storage principle.

**Do not create a generic DataLayer abstraction.** The temptation is strong: "all four stores do CRUD, so let's make a `DataLayer<T>` with `get()`, `put()`, `list()`, `delete()`." The problem is that each store has fundamentally different access patterns. Session Store is append-only streams. Memory Store does similarity search. Config Store is read-heavy with rare writes. Skill Store is flat files. A generic CRUD abstraction either becomes so abstract it's useless or so concrete it only fits one use case.

**Config is read-shared, write-rare.** Multiple services need to read config (Agent Core reads identity, Model reads provider settings, Tools reads permissions). But only one thing writes config — the user or admin. Make config immutable during a session. Load at startup. If you need hot-reload, use file watchers or config drift detection (openclaw's approach), not a mutable shared object.

**Separate the conversation log from the session state.** The raw message array (what was said) is not the same as the session state (current step, pending tool calls, accumulated context). hermes mixes these in one table and it creates complexity. Better: messages in an append-only log, session state in a separate record that tracks current position, compression status, and metadata.

**Start with full-text search. Add vector search later.** If your Agent runs locally and you expect fewer than 50,000 memories, SQLite FTS5 handles it with sub-millisecond performance. Add vector search when keyword matching starts returning irrelevant results — which, for most Agent use cases, is much later than you think.

**Keep skill files as plain Markdown.** hermes uses `.md` files with YAML frontmatter. This is the right call. Skills are human-readable, human-editable, and version-controllable. Don't put skills in a database. Don't serialize them to binary. The entire value of the learning loop (Module 4 + emergent behavior) depends on skills being inspectable and modifiable.

---

## Step-by-Step: How to Build It

### Phase 1: Session Store

Start here. Without session persistence, your Agent forgets everything on restart.

```
// Schema — one table
CREATE TABLE sessions (
  id          TEXT PRIMARY KEY,
  created_at  TEXT DEFAULT (datetime('now')),
  updated_at  TEXT DEFAULT (datetime('now')),
  metadata    TEXT  -- JSON: model, user, task summary
);

CREATE TABLE messages (
  id          INTEGER PRIMARY KEY AUTOINCREMENT,
  session_id  TEXT REFERENCES sessions(id),
  role        TEXT NOT NULL,  -- 'user', 'assistant', 'tool', 'system'
  content     TEXT NOT NULL,
  tool_call   TEXT,           -- JSON, null for non-tool messages
  created_at  TEXT DEFAULT (datetime('now'))
);

CREATE INDEX idx_messages_session ON messages(session_id, id);
```

Agent Core interface:
```
sessionStore.createSession() → sessionId
sessionStore.appendMessage(sessionId, message) → void
sessionStore.getMessages(sessionId) → message[]
sessionStore.listSessions() → sessionMetadata[]
```

That's it. Four functions. No ORM, no query builder. Raw SQL through `better-sqlite3` (Node) or `sqlite3` (Python).

### Phase 2: Config Store

Config is the simplest store. It's just files.

```
// Directory structure
~/.agent/
  config.yaml        # Main config (identity, model, defaults)
  tools.yaml         # Tool permissions and settings
  memory.yaml        # Memory backend settings
```

Load at startup:
```
const config = yaml.parse(fs.readFileSync('~/.agent/config.yaml'))
```

Expose as read-only to other services:
```
configStore.get('model.provider')    → 'anthropic'
configStore.get('tools.shell.mode')  → 'ask'
configStore.get('memory.backend')    → 'sqlite'
```

No write API during runtime. Config changes go through a CLI command or direct file edit. This eliminates an entire class of bugs where the Agent accidentally modifies its own configuration.

### Phase 3: Memory Store

Memory Store needs search, not just CRUD. Start with full-text search.

```
CREATE VIRTUAL TABLE memory_fts USING fts5(
  content,
  metadata,
  tokenize='porter unicode61'
);

CREATE TABLE memories (
  id          TEXT PRIMARY KEY,
  content     TEXT NOT NULL,
  type        TEXT NOT NULL,     -- 'fact', 'preference', 'context', 'skill_ref'
  source      TEXT,              -- where this memory came from
  created_at  TEXT DEFAULT (datetime('now')),
  accessed_at TEXT,              -- last retrieval time
  access_count INTEGER DEFAULT 0
);

-- Keep FTS in sync
CREATE TRIGGER memory_insert AFTER INSERT ON memories BEGIN
  INSERT INTO memory_fts(rowid, content, metadata)
  VALUES (NEW.rowid, NEW.content, NEW.type || ' ' || COALESCE(NEW.source, ''));
END;
```

Memory service interface:
```
memoryStore.store(content, type, source) → memoryId
memoryStore.search(query, limit = 5) → memory[]
memoryStore.delete(memoryId) → void
memoryStore.getRecent(limit = 10) → memory[]
```

The `search()` function uses FTS5's `MATCH` operator:
```
SELECT m.* FROM memories m
JOIN memory_fts ON memory_fts.rowid = m.rowid
WHERE memory_fts MATCH ?
ORDER BY rank
LIMIT ?
```

Add vector search later. You'll know when you need it — keyword searches will start returning memories that match words but not meaning.

### Phase 4: Skill Store

Skill Store is just a directory of Markdown files. No database.

```
// Directory
~/.agent/skills/
  debug-python-errors.md
  write-unit-tests.md
  optimize-sql-queries.md
```

Each file has YAML frontmatter:
```markdown
---
name: debug-python-errors
trigger: when user reports a Python error or traceback
version: 2
created: 2026-04-01
updated: 2026-04-08
---

## Steps
1. Read the full traceback
2. Identify the exception type and line number
3. Check the surrounding code for common causes
...
```

Skill Store interface:
```
skillStore.list() → skillMetadata[]
skillStore.load(name) → skillContent
skillStore.save(name, content) → void
skillStore.delete(name) → void
```

Agent Core calls `skillStore.list()` each turn and injects relevant skills into the system prompt based on trigger conditions. Tools calls `skillStore.save()` when the learning loop creates a new skill.

### Phase 5: Wire Into Agent Core

Agent Core coordinates all stores at session boundaries:

```
// Session start
const sessionId = sessionStore.createSession()
const config = configStore.getAll()
const memories = memoryStore.search(userMessage, 5)
const skills = skillStore.list()

// Build system prompt with config + memories + skills
const systemPrompt = buildPrompt(config, memories, skills)

// Conversation loop
while (budget > 0) {
  const response = await model.call(messages)
  sessionStore.appendMessage(sessionId, response)

  if (response.toolCalls) {
    const results = await tools.execute(response.toolCalls)
    for (const result of results) {
      sessionStore.appendMessage(sessionId, result)
    }
  }
}

// Session end — extract memories
const newMemories = await memory.extract(messages)
for (const mem of newMemories) {
  memoryStore.store(mem.content, mem.type, sessionId)
}
```

The key: Agent Core calls four different stores through four different interfaces. It never interacts with a generic `DataLayer`. Each store has its own API shaped by its access patterns.

---

## Common Pitfalls

**Building a generic DataLayer.** This was our most persistent architectural mistake. "Persistence is a cross-cutting concern, so let's make it a horizontal layer." It sounds reasonable. It's wrong. Session Store appends messages. Memory Store does similarity search. Config Store loads files. Skill Store reads Markdown. A generic abstraction either forces these into a lowest-common-denominator CRUD interface (losing the specific capabilities each needs) or becomes so configurable it's harder to use than the direct implementations.

**Shared mutable state between services.** If Memory can read the sessions table directly, you have an implicit coupling that no interface boundary can protect. When Agent Core changes its message format, Memory breaks silently. When Memory indexes old sessions, it competes with Core for database locks. Each service should access only its own tables. Cross-service data access goes through explicit function calls, not shared database access.

**Optimizing storage before you have users.** You spend a week benchmarking Postgres vs. SQLite vs. DynamoDB. Your Agent has three users. SQLite handles millions of rows on a single file with sub-millisecond reads. It has no server process, no connection pooling, no network latency, and backup is `cp hermes.db hermes.db.bak`. Start with SQLite. Migrate when (if) you actually hit its limits.

**Putting skills in a database.** Skills are documents. Humans need to read and edit them. Version control needs to track them. `git diff` needs to show what changed. A database row containing serialized Markdown gives you none of this. Keep skills as files. hermes gets this right.

**Forgetting that Config Store is shared-read.** If you make config a mutable runtime object that any service can write to, you get race conditions, inconsistent state, and debugging nightmares. Config should be immutable during a session. Load once, use everywhere, change only through explicit admin action.

**Not separating access_count from content storage.** Memory retrieval benefits from tracking how often each memory is accessed (for decay, prioritization, cleanup). If `access_count` is in the same table as the content, every retrieval becomes a read-then-write. Consider a separate `memory_stats` table or update access counts in batches, not per-query.

**Treating "one SQLite file" as "one table."** hermes puts sessions, memories, config, and skills in one `.db` file. That's fine — it's a deployment convenience. But each logical store still has its own tables, its own schemas, and its own access code. "One file" does not mean "one schema" or "one service."

---

## Cross-Module Contracts

Storage is vertical, but the stores still have contracts with the services that own them and the services that consume them:

| Contract | Between | What |
|----------|---------|------|
| **Message schema** | Agent Core → Session Store | Session Store persists messages in Agent Core's format. If Core adds a field (like `speculation_state` or `compression_marker`), Session Store must handle it — either storing it faithfully or explicitly ignoring it. |
| **Memory write ownership** | Memory (Module 4) → Memory Store | Only Memory service writes to Memory Store. Agent Core reads memories through Memory's search API, never by querying the store directly. If Core needs a new query pattern, Memory adds it to its API. |
| **Skill file format** | Tools (Module 3) ↔ Skill Store ↔ Agent Core (Module 1) | Tools writes skills. Core reads skills. The contract is the file format: YAML frontmatter (name, trigger, version) + Markdown body. Changing the frontmatter schema requires updating both Tools' write logic and Core's prompt builder. |
| **Config read interface** | Config Store → All services | Config Store provides a read-only interface. Services request config values by key path. The key namespace must be stable — renaming `model.provider` to `llm.service` breaks every service that reads it. |
| **Filesystem paths** | All stores ↔ Security Envelope (Module 8) | If a Security Envelope is active (Landlock, Docker volumes), all stores must write within allowed paths. Session Store, Memory Store, and Skill Store paths must be in the writable set. Config Store path must be in the readable set. Changing a store's path requires updating the security policy. |
| **Session lifecycle** | Agent Core → Session Store, Memory Store | When a session ends, Core signals completion. Memory extracts long-term knowledge from the session. Session Store marks the session as complete. The order matters: Memory extraction must happen before the session messages are compressed or archived. |

**What happens if these contracts break:**
- Change message schema without updating Session Store → old sessions can't be loaded → "session not found" errors on resume
- Memory reads Session Store directly instead of through Core → Core changes message format → Memory crashes on deserialization
- Change skill frontmatter schema → Core's prompt builder skips skills with unknown format → learning loop silently stops working
- Rename a config key → services that read the old key get `undefined` → fall back to hardcoded defaults → behavior changes without explanation
- Move the skills directory without updating the security policy → Landlock blocks writes → `skill_manage create` fails → learning loop breaks

The vertical storage principle makes these contracts simpler than they would be in a horizontal DataLayer model. Each contract is bilateral (between one service and one store), not multilateral (between every service and a shared layer). When something breaks, you know exactly which two components to look at.

---

*Next: [Module 10 — Runtime & Infrastructure](10-runtime-infrastructure.md) | Previous: [Module 8 — Security Envelope](08-security-envelope.md)*
