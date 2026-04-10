# Minimum Viable Agent

> 5 core components. Everything else is optional. This is Claude Code's architecture pattern -- the same structure behind one of the most capable production agents.

---

## What You Need

```
Interface (CLI)
Agent Core + Session Store
├── Model (stateless)
├── Tools + Skill Store
├── Context Manager (stateless)
└── Memory + Memory Store (optional but recommended)
```

**What you don't need**: Gateway, Routing, Security Envelope, Runtime/Infra. Those solve multi-user, multi-channel, and deployment problems. You don't have those problems yet.

Memory is listed as optional because your agent works without it -- every session starts fresh. But add it early. Without memory, your agent forgets everything between sessions. Users notice fast.

---

## The 7 Components

| # | Component | State | Purpose |
|---|-----------|-------|---------|
| 1 | Interface (CLI) | Stateless | Read user input, display model output |
| 2 | Agent Core | Owns Session Store | The while loop. Call model, check stop reason, execute tools, repeat |
| 3 | Model | Stateless | One provider, one API call. Returns text or tool_call |
| 4 | Tools | Owns Skill Store | Register tools, match by name, execute, return results |
| 5 | Context Manager | Stateless | Count tokens, truncate when over budget |
| 6 | Session Store | Persistence | Save conversation messages to disk |
| 7 | Memory (optional) | Owns Memory Store | Inject prior knowledge into system prompt |

Model and Context Manager are stateless. They operate on data passed in and return results. No files, no databases, no side effects.

---

## Build It Step by Step

### Step 1: CLI that reads user input

A readline loop. Accept a line of text, pass it downstream, print the response. Nothing more.

```
cli/
└── index.ts          # readline loop, calls agentCore.run(userMessage)
```

This is your Interface layer. hermes, openclaw, and Claude Code all start here -- a CLI adapter that normalizes user input into a message object and feeds it to Agent Core.

**Read more**: [Interface module guide](../modules/interface.md)

### Step 2: Agent Core -- the while loop

The heart of every agent. All 4 projects we studied do the same thing:

```
while (budget > 0):
    response = model.call(messages)
    if response.stop_reason == "end_turn":
        break
    if response.stop_reason == "tool_use":
        result = tools.execute(response.tool_call)
        messages.append(result)
        budget -= 1
```

That's it. The loop calls the model, checks what came back, executes tools if requested, appends results, and loops again. Claude Code's `query.ts` does this with async generators. hermes does it in `run_conversation()`. Same pattern, different syntax.

Agent Core owns the Session Store -- it writes every message (user, assistant, tool_result) to persistence after each loop iteration.

**Read more**: [Agent Core module guide](../modules/agent-core.md)

### Step 3: Model -- one provider, one API call

Hardcode a single provider. Anthropic, OpenAI, whatever you prefer. One function: take messages in, return a response. The response contains either text content or a tool_call.

```
model/
└── provider.ts       # callModel(messages): Response
```

No failover chain. No candidate rotation. No token estimation. Those come later (see [Model module guide](../modules/model.md)). Right now you need: call API, return response.

The model is stateless. It doesn't store anything. It doesn't remember previous calls. The conversation history lives in Agent Core's messages array.

### Step 4: Tools -- register 2-3 built-in tools

Start with 2-3 tools that make your agent useful:

- **file_read**: Read a file from disk. Takes a path, returns content.
- **shell_exec**: Run a shell command. Takes a command string, returns stdout/stderr.
- **file_write** (optional): Write content to a file.

Registration is simple. hermes uses import-time self-registration -- each tool file calls `registry.register()` on load. For an MVA, a plain object map works fine:

```
tools/
├── registry.ts       # Map<string, ToolHandler>
└── builtins/
    ├── file-read.ts
    └── shell-exec.ts
```

Each tool needs: a name (string the model uses to call it), a JSON schema describing its parameters, and an execute function. The model sees the schema, decides when to use the tool, and passes parameters back. Agent Core matches the name, calls execute, returns the result.

**Read more**: [Tools module guide](../modules/tools.md)

### Step 5: Context Manager -- token budget enforcement

The simplest version: count tokens in the messages array. If over 80% of the model's context window, truncate oldest messages.

```
context/
└── manager.ts        # trimIfNeeded(messages, maxTokens): messages
```

Claude Code uses 80% as its auto-compression threshold. That's a good default. Token counting doesn't need to be exact -- `Math.ceil(text.length / 4)` gets you within 10% for English text. Good enough for an MVA.

Three levels to implement later (see [Context Manager module guide](../modules/context-manager.md)):
1. L1: Prune old tool results (zero LLM calls, cheap)
2. L2: LLM-powered summarization of middle turns (expensive but effective)
3. L3: Hard truncation of earliest messages (last resort)

For now, L3 alone is fine.

### Step 6: Session Store -- persist messages

Save the messages array after each loop iteration. Two options:

- **JSONL**: One JSON object per line. Append-only. Simple. Good for prototyping.
- **SQLite**: One table, columns for session_id, role, content, timestamp. Better for querying.

```
storage/
└── session-store.ts  # save(sessionId, message), load(sessionId): Message[]
```

On startup, load the previous session's messages. On each turn, append the new messages. That's session continuity.

### Step 7: Wire them together

```
agent/
├── core/
│   └── loop.ts           # Agent Core -- the while loop
├── model/
│   └── provider.ts       # Single model provider
├── tools/
│   ├── registry.ts       # Tool registration
│   └── builtins/
│       ├── file-read.ts
│       └── shell-exec.ts
├── context/
│   └── manager.ts        # Token counting + truncation
├── storage/
│   └── session-store.ts  # JSONL or SQLite persistence
├── interfaces/
│   └── cli.ts            # readline loop
├── contracts/
│   ├── tool.ts           # Tool interface definition
│   ├── model.ts          # Model interface definition
│   └── message.ts        # Message type definitions
└── tests/
    ├── unit/
    └── loops/            # Empty for now -- fill when you add emergent behaviors
```

The `contracts/` directory matters. Even in a single package, define interfaces for Tool, Model, and Message types. These cost nothing now and save everything later when you swap implementations.

---

## What This Gets You

A working agent that can:
- Accept natural language instructions
- Call an LLM to decide what to do
- Execute file and shell tools
- Manage its own context window
- Persist conversation history across sessions

Total: ~200 lines of core logic. The rest is tool implementations and boilerplate.

---

## What This Doesn't Get You

| Missing capability | Which module adds it | When to add |
|---|---|---|
| Learning from experience | Memory + Skill Store + learning loop | When you want the agent to improve over time |
| Model failover | Model candidate chain + cooldown probing | When you hit rate limits in production |
| Multi-channel support | Interface adapters + Gateway | When users need Slack/Discord/Web access |
| Security sandboxing | Security Envelope | When the agent runs shell commands in production |
| Sub-agent delegation | Agent Core delegation logic | When single-agent context gets too expensive |

Each of these is a separate guide. Build the MVA first. Add capabilities one at a time.

---

## Module Guide References

| Component | Guide |
|-----------|-------|
| Interface (CLI) | [interface.md](../modules/interface.md) |
| Agent Core | [agent-core.md](../modules/agent-core.md) |
| Model | [model.md](../modules/model.md) |
| Tools | [tools.md](../modules/tools.md) |
| Context Manager | [context-manager.md](../modules/context-manager.md) |
| Session/Config Store | [storage.md](../modules/storage.md) |
| Memory (when ready) | [memory.md](../modules/memory.md) |

---

*Based on architecture patterns from Claude Code, hermes-agent, openclaw, and NemoClaw. See the [Architecture Reference Model](../../docs/agent-architecture-reference.md) for the full 11-module breakdown.*
