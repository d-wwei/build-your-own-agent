# Emergent Behavior 6: Multi-Agent Specialization

> One-line: Messages from different channels route to differently-configured Agent instances — each with its own tools, memory, and persona — turning one codebase into a team of specialists.

## What Is It

A message arrives from Telegram group 12345. The routing layer checks its binding table: group 12345 maps to `investmentAgent`. That Agent has financial analysis tools, a memory store full of portfolio data, and a system prompt that says "You are a senior investment analyst."

Simultaneously, a message arrives from Slack channel #ops. Routing checks again: #ops maps to `devopsAgent`. Different tools. Different memory. Different persona. "You are a DevOps engineer focused on incident response."

Both Agents are instances of the same Agent Core class. Same conversation loop. Same `while(budget > 0)` structure. But they behave like completely different entities because they were constructed with different parameters.

No single component was designed to create "a team of specialists." Routing just does pattern matching. Agent Core just runs its loop. The Interface layer just delivers messages. Combined = a system where the right specialist handles every conversation automatically.

## How It Emerges

Three components interact:

1. **Routing** — Pattern-matches incoming messages against a binding table. "Telegram group 12345 → investmentAgent." That is all Routing does. It does not know what an investment agent is. It just resolves a name.

2. **Agent Core** — An instance factory creates Agent Core instances from configuration. Each config specifies: tools, memory backend, persona prompt, model preferences, iteration budget. The factory does not care that one config says "financial analyst" and another says "DevOps engineer." It just builds what the config describes.

3. **Interface** — Adapters for Telegram, Slack, Discord, etc. Each adapter normalizes messages into a common format and delivers them to the routing layer. The adapter does not know which Agent will handle the message.

The emergent behavior: none of these components "knows" about specialization. Routing matches strings. Agent Core follows configs. Interface normalizes protocols. But the combined system routes financial questions to a financial expert and DevOps questions to a DevOps expert. Specialization emerges from composition.

## Which Components Participate

| Component | Role in Specialization |
|-----------|----------------------|
| Interface (multi-channel) | Receives messages from Telegram, Slack, Discord, etc. |
| Gateway + Routing | Pattern-matches message source to Agent binding |
| Agent Core x N | Multiple instances, each with different config |
| Tools (per Agent) | Each Agent has its own registered tool set |
| Memory (per Agent) | Each Agent has its own memory store |
| Config Store | Holds binding rules and per-Agent configurations |

**Critical distinction from Sub-Agent Delegation (Behavior 5):** In delegation, a parent spawns a temporary child for one task. In specialization, all Agents are permanent peers. No parent-child relationship. No summary return path. Each Agent has ongoing conversations with its own users.

## Case Study: openclaw's 9-Level Binding Priority

openclaw is the only project among the four that implements production-grade multi-agent specialization. Here is the exact routing priority:

| Priority | Binding Level | Example |
|----------|--------------|---------|
| 1 | Exact chat/group match | Telegram group `12345` → `investmentAgent` |
| 2 | Parent thread inheritance | Reply in thread → same Agent as parent message |
| 3 | Wildcard match | All Telegram groups matching `invest-*` → `investmentAgent` |
| 4 | Discord server + role | Discord server `trading-guild` + role `analyst` → `marketAgent` |
| 5 | Discord server | Any message in Discord server `trading-guild` → `generalTradeAgent` |
| 6 | Slack team | Slack workspace `acme-corp` → `acmeAgent` |
| 7 | Account-level | All messages from user `@ceo` regardless of channel → `executiveAgent` |
| 8 | Channel-level | All Telegram messages → `defaultTelegramAgent` |
| 9 | Default fallback | No match → `defaultAgent` |

**Session key construction**: Each conversation gets a deterministic session key. For Telegram group 12345, the key is `tg-12345-main`. For a thread within that group, `tg-12345-thread-789`. The session key determines which conversation history to load. Two messages to the same group continue the same conversation. A new thread starts fresh.

**Agent instance lifecycle**: Agents are not spawned per-message. They are long-running instances, each with persistent memory and ongoing conversation state. The `investmentAgent` remembers what it discussed with Telegram group 12345 yesterday.

**The flow in detail:**

```
1. Telegram adapter receives message from group 12345
2. Adapter normalizes: { source: "telegram", chatId: "12345", text: "What's AAPL's P/E?" }
3. Gateway passes to Routing
4. Routing checks bindings at priority 1: exact match found → "investmentAgent"
5. Routing constructs sessionKey: "tg-12345-main"
6. Agent instance factory: get or create investmentAgent
7. investmentAgent.run(message, sessionKey)
8. Agent Core loads conversation history for sessionKey
9. Agent Core builds context: persona + tools + memory + history
10. Normal conversation loop proceeds
11. Response routes back through Gateway → Telegram adapter → group 12345
```

## How the Other Projects Compare

| Project | Multi-Agent Support | Why |
|---------|-------------------|-----|
| **openclaw** | Full implementation. 9-level binding, per-Agent config, multi-channel | Designed as a multi-tenant AI gateway from the start |
| **hermes-agent** | None. Single Agent architecture. No routing module. | Built as a self-evolving single Agent |
| **Claude Code** | None. Single user, single Agent. | Built as a personal CLI tool for one developer |
| **NemoClaw** | N/A. Not an Agent — it is a sandbox runtime. | Runs openclaw instances inside containers |

This is the sharpest divergence across the four projects. Specialization requires a routing layer. Only openclaw built one. The other projects chose the single-Agent path and optimized for depth (hermes: learning loop; Claude Code: speculative execution) rather than breadth.

## How to Make It Emerge in Your Agent

### Step 1: Define Agent configurations

Each specialist Agent needs a config file:

```yaml
# agents/investment-agent.yaml
name: investmentAgent
persona: "You are a senior investment analyst. You provide data-driven financial analysis."
model: gpt-4
budget: 60
tools:
  - stock_lookup
  - financial_report
  - portfolio_analyzer
memory_backend: sqlite
memory_namespace: "investment"
```

```yaml
# agents/devops-agent.yaml
name: devopsAgent
persona: "You are a DevOps engineer. You diagnose incidents and suggest fixes."
model: gpt-4
budget: 40
tools:
  - check_service_health
  - read_logs
  - restart_service
memory_backend: sqlite
memory_namespace: "devops"
```

### Step 2: Build a binding table

```yaml
# routing/bindings.yaml
bindings:
  - match: { platform: "telegram", chatId: "12345" }
    agent: investmentAgent
    priority: 1

  - match: { platform: "slack", channel: "ops" }
    agent: devopsAgent
    priority: 1

  - match: { platform: "telegram" }
    agent: defaultAgent
    priority: 8

  - match: {}
    agent: defaultAgent
    priority: 9
```

### Step 3: Build the router

The router is simple. For each incoming message:

```python
def route(message):
    bindings_sorted = sorted(BINDINGS, key=lambda b: b.priority)
    for binding in bindings_sorted:
        if matches(message.source, binding.match):
            return binding.agent
    return "defaultAgent"
```

This is 10 lines of code. The power comes from the binding table, not the algorithm.

### Step 4: Build the Agent instance factory

```python
agents = {}

def get_agent(agent_name):
    if agent_name not in agents:
        config = load_config(f"agents/{agent_name}.yaml")
        agents[agent_name] = AgentCore(
            persona=config.persona,
            model=config.model,
            budget=config.budget,
            tools=load_tools(config.tools),
            memory=create_memory(config.memory_backend, config.memory_namespace)
        )
    return agents[agent_name]
```

Agents are created on first use and cached. Each has its own tools, memory, and persona.

### Step 5: Wire session keys

```python
def build_session_key(message):
    platform = message.source.platform    # "telegram"
    chat_id = message.source.chat_id      # "12345"
    thread_id = message.source.thread_id  # None or "789"

    if thread_id:
        return f"{platform}-{chat_id}-thread-{thread_id}"
    return f"{platform}-{chat_id}-main"
```

The session key determines which conversation history the Agent loads. Same key = continued conversation. New key = fresh start.

### Step 6: Connect the pieces

```python
def handle_message(message):
    agent_name = route(message)
    agent = get_agent(agent_name)
    session_key = build_session_key(message)
    response = agent.run(message.text, session_key)
    send_response(message.source, response)
```

Six lines. That is the entire orchestration layer. The complexity lives in the binding table and the Agent configs, not in the glue code.

## Cross-Module Contracts

### 1. Binding Match Rules

```
{
  platform: string,         // "telegram" | "slack" | "discord" | ...
  chatId?: string,          // exact match
  channel?: string,         // exact match
  serverId?: string,        // Discord server
  role?: string,            // Discord role
  userId?: string,          // account-level binding
  pattern?: string          // wildcard match (e.g., "invest-*")
}
```

Priority is explicit (1-9). On tie, first match wins. No implicit priority from field specificity — that causes debugging nightmares.

### 2. Session Key Construction

Format: `{platform}-{chatId}-{threadOrMain}`

Rules:
- Deterministic. Same input always produces same key.
- Thread messages inherit the parent binding but get a separate session key.
- One Agent can serve multiple session keys (e.g., `investmentAgent` handles both `tg-12345-main` and `tg-67890-main`).
- Session keys never cross Agents. If `tg-12345-main` is bound to `investmentAgent`, reassigning it to `devopsAgent` starts a new conversation, not a continuation.

### 3. Agent Instance Factory

```
Input:  agent_name (string)
Output: AgentCore instance

Contract:
- First call creates the instance from config
- Subsequent calls return the cached instance
- Each instance has isolated: tools, memory namespace, persona, budget
- Shared: model provider connection (pooled, not duplicated)
```

### 4. Memory Namespace Isolation

Each Agent writes to its own memory namespace. `investmentAgent` cannot read `devopsAgent`'s memory. This is enforced at the memory backend level:

```
investmentAgent → memory.sqlite, table: investment_*
devopsAgent     → memory.sqlite, table: devops_*
```

Same database is fine. Separate tables are mandatory.

## How to Test It (Loop Test)

### Test 1: Correct routing

```
1. Send message from Telegram group 12345
2. Assert: routed to investmentAgent
3. Send message from Slack #ops
4. Assert: routed to devopsAgent
5. Send message from unknown source
6. Assert: routed to defaultAgent
```

### Test 2: Independent memory

```
1. Send "Remember: AAPL target is $200" to investmentAgent via Telegram 12345
2. Send "What is AAPL's target?" to devopsAgent via Slack #ops
3. Assert: devopsAgent does NOT know about the $200 target
4. Send "What is AAPL's target?" to investmentAgent via Telegram 12345
5. Assert: investmentAgent recalls $200
```

### Test 3: Independent tool sets

```
1. investmentAgent has tool: stock_lookup
2. devopsAgent has tool: restart_service
3. Assert: investmentAgent cannot call restart_service
4. Assert: devopsAgent cannot call stock_lookup
```

### Test 4: Session continuity

```
1. Send message A to Telegram 12345 → investmentAgent
2. Send message B to Telegram 12345 → investmentAgent
3. Assert: message B sees message A in conversation history (same session key)
4. Send message C to Telegram 67890 → investmentAgent
5. Assert: message C does NOT see message A or B (different session key)
```

### Test 5: Priority resolution

```
1. Create binding: { platform: "telegram", chatId: "12345" } → investmentAgent (priority 1)
2. Create binding: { platform: "telegram" } → defaultTelegramAgent (priority 8)
3. Send message from Telegram 12345
4. Assert: routed to investmentAgent, NOT defaultTelegramAgent
5. Send message from Telegram 99999
6. Assert: routed to defaultTelegramAgent (priority 1 did not match, fell through to 8)
```

### Test 6: Thread inheritance

```
1. Send message to Telegram group 12345 → investmentAgent
2. Reply in a thread within group 12345
3. Assert: thread message routes to investmentAgent (inherited from parent)
4. Assert: thread message gets a separate session key (tg-12345-thread-789, not tg-12345-main)
```

## Variations Across Projects

There is only one variation: **openclaw has it; nobody else does.**

| Dimension | openclaw | hermes / Claude Code / NemoClaw |
|-----------|----------|-------------------------------|
| **Routing layer** | 9-level priority binding | None |
| **Agent instances** | Multiple, each with own config | Single instance |
| **Memory isolation** | Per-Agent namespace | Single namespace |
| **Tool isolation** | Per-Agent tool set | Single tool registry |
| **Multi-channel** | 20+ platform adapters | CLI only (hermes has adapters but single Agent) |
| **Session management** | Per-binding session keys | Single session or per-user |

This makes specialization the clearest architectural fork in the Agent landscape. Every project can do delegation (Behavior 5) — it is just a tool that creates a child. But specialization requires infrastructure that single-Agent projects do not build: a routing module, a binding table, an instance factory, namespace isolation.

**The progression path:**

```
Stage 1: Single Agent, single channel
         (hermes, Claude Code — already powerful)

Stage 2: Single Agent, multiple channels
         (hermes with adapters — same Agent, more entry points)

Stage 3: Multiple Agents, multiple channels
         (openclaw — different specialists for different contexts)
```

If you are building a personal tool, stop at Stage 1. If you are building a team tool, consider Stage 2. If you are building a platform — where different user groups need fundamentally different Agent behavior — you need Stage 3.

Most projects never need Stage 3. But if you do, the routing-binding-factory pattern shown above is the proven approach. Start with 2 Agents, 2 bindings. Validate that memory isolation works. Then scale to N.
