# Module 6: Interface
> Every inbound path into your agent. Users and external systems talk to your agent through this layer — and only through this layer.

## What Is It

The Interface layer is the collection of all **inbound** entry points. A user typing in a terminal. A Telegram message arriving via webhook. An IDE connecting over MCP. Another agent calling yours via ACP. All of these are Interface.

The critical distinction: **direction determines which module it belongs to.**

- MCP **inbound** (someone connects to your agent as an MCP server) = Interface.
- MCP **outbound** (your agent connects to an external MCP server to get tools) = Tools.
- API **inbound** (external system calls your agent's HTTP endpoint) = Interface.
- API **outbound** (your agent calls an LLM provider) = Model.

This is not a pedantic taxonomy. It determines where code lives, who owns the connection lifecycle, and what happens when something breaks. If you put your inbound Telegram adapter in the Tools directory, someone will eventually try to "execute" it as a tool. If you put your outbound MCP client in the Interface directory, someone will try to register it as a channel.

Each adapter does three things: **protocol translation** (convert platform-specific wire format to internal message format), **media handling** (images, files, voice — different per platform), and **lifecycle management** (connection setup, reconnection, graceful shutdown). That is all. No business logic. No routing decisions. No authentication. Those belong in Gateway.

## What Problem Does It Solve

Without an Interface layer, your agent talks to exactly one thing in exactly one way. That is fine at first. Then you want Slack integration. Then a web UI. Then another agent needs to call yours programmatically.

Without a clean abstraction, you end up with one of two disasters:

1. **Copy-paste adapters.** Each new platform gets a bespoke integration that duplicates message parsing, error handling, and response formatting. hermes avoided this with `BasePlatformAdapter`. Imagine maintaining 16 copy-pasted integrations without it.

2. **Hardcoded entry point.** Your agent core directly imports `readline` for CLI input. Adding Telegram means rewriting core. Claude Code has this problem in mild form — each entry point (`cli.tsx`, `mcp.ts`, `bridge`, `daemon`) is independent. Adding a 5th means writing a 5th from scratch.

The adapter pattern solves both. Define a common interface. Each platform implements it. Core depends on the interface, never on a specific platform. Adding a new channel means writing one adapter class. Zero changes to Core.

## How the 4 Projects Do It

### hermes-agent

**The adapter inheritance tree.** hermes defines `BasePlatformAdapter` as a Python ABC with 16 concrete subclasses.

**File**: `hermes/adapters/base.py` defines the abstract base. Each platform gets its own file: `telegram_adapter.py`, `discord_adapter.py`, `slack_adapter.py`, and so on through Mattermost, Matrix, Rocket.Chat, Line, WeChat, WhatsApp, SMS (Twilio), Email, API Server, Web UI, CLI, and ACP.

**What BasePlatformAdapter enforces:**
- `connect()` / `disconnect()` — lifecycle
- `send_message()` — outbound response to the user
- `handle_incoming()` — inbound message processing
- Platform-specific media conversion (each override handles its own attachment format)

**The gateway dependency.** `gateway/run.py` (7620 lines) manages all adapter instances. It starts them, stops them, routes slash commands. The adapters themselves stay simple — the lifecycle complexity lives in the gateway.

**ACP adapter** — hermes is the only project with an Agent Communication Protocol adapter, letting other agents call hermes as a peer. This is distinct from MCP (tool-level protocol) — ACP is agent-to-agent.

**Count**: 16 adapters. The most platforms of any project. If you need to support many messaging channels, hermes's pattern is the one to study.

### openclaw

**Adapter Composition over inheritance.** openclaw takes a different approach. Instead of one base class with 16 subclasses, it defines a set of capability interfaces:

- `ChannelPlugin` — the base that every channel must implement (connect, disconnect, handle message)
- `SupportsThreads` — if the platform has threads (Discord, Slack)
- `SupportsReactions` — if the platform has emoji reactions
- `SupportsEditing` — if the platform allows message editing
- `SupportsFiles` — if the platform supports file uploads

Each channel adapter implements **only the interfaces it needs**. Telegram implements `ChannelPlugin + SupportsFiles + SupportsReactions`. A bare-bones webhook adapter implements only `ChannelPlugin`.

**Why this matters.** In hermes's inheritance model, the base class must accommodate every platform's capabilities. If you add a `handle_thread_reply()` method to the base, every adapter now has to implement it — even platforms that do not have threads. openclaw avoids this. Discord implements `SupportsThreads`. SMS does not. Neither needs stub methods.

**Count**: 20+ channel plugins. The most extensible architecture of the four projects, because third-party developers can write channel plugins using the published SDK interfaces without touching openclaw's core.

**WebChat built-in.** openclaw includes a browser-based chat UI as one of its channel plugins. This means the agent ships with a web interface out of the box — no separate frontend project needed.

### NemoClaw

**Management CLI only.** NemoClaw is a sandbox orchestrator, not a user-facing agent. Its only interface is a management CLI for operators:

- `nemo setup` — initialize sandbox environment
- `nemo deploy` — deploy an OpenClaw instance into a sandbox
- `nemo status` — check sandbox health

No user-facing channels. No messaging platform adapters. No web UI. NemoClaw delegates all user interaction to the OpenClaw instance running inside its sandbox. The lesson: not every component needs a rich Interface layer. If your job is infrastructure, a CLI for operators is sufficient.

### Claude Code

**Multiple independent entry points, no unified abstraction.**

Claude Code has four distinct ways to start:

| Entry point | File | Purpose |
|-------------|------|---------|
| CLI REPL | `src/entrypoints/cli.tsx` | Interactive terminal session |
| MCP Server | `src/entrypoints/mcp.ts` | IDE connects to Claude Code as an MCP server |
| HTTP Bridge | `src/entrypoints/bridge.ts` | Programmatic HTTP API access |
| Background Daemon | `src/entrypoints/daemon.ts` | Long-running background operations |

**No shared base class.** Each entry point is a standalone module. They all eventually call into `query.ts` (the inner loop from Module 1), but they handle their own connection setup, input parsing, and output formatting independently.

**Why no abstraction?** Performance. Each entry point is optimized for its specific use case. `cli.tsx` uses React (Ink) for terminal rendering. `mcp.ts` handles the MCP protocol's JSON-RPC framing. `bridge.ts` handles HTTP request/response. Forcing them through a common abstraction would add overhead and reduce each entry point's ability to optimize for its scenario.

**The tradeoff is real.** Adding a 5th entry point means writing it from scratch. There is no template, no base class, no "implement these 3 methods and you are done." For a product maintained by one company (Anthropic), this is acceptable. For an open-source project expecting community contributions, it would not be.

**Millisecond-level optimization.** Claude Code's entry points are tuned for startup time. The CLI path avoids importing modules it does not need. The MCP path skips UI initialization entirely. This level of optimization is only possible because each entry point controls its own import graph.

## Comparison Table

| Dimension | hermes | openclaw | NemoClaw | Claude Code |
|-----------|--------|----------|----------|-------------|
| Pattern | Inheritance (ABC + 16 subclasses) | Composition (capability interfaces) | Management CLI only | Independent entry points, no abstraction |
| Channel count | 16 | 20+ | 1 (operator CLI) | 4 entry points |
| Adding a new channel | Subclass `BasePlatformAdapter`, implement required methods | Implement `ChannelPlugin` + desired capability interfaces | N/A | Write from scratch |
| Lifecycle management | `gateway/run.py` manages all adapters | Plugin lifecycle hooks | Manual CLI invocation | Each entry point manages itself |
| Third-party extensibility | No (internal adapters only) | Yes (published Plugin SDK) | No | No |
| Media handling | Per-adapter, inherited helpers | Per-adapter, interface-driven | N/A | Per-entry-point |
| Agent-to-agent protocol | ACP adapter | Channel can be another agent | N/A | MCP server mode |
| Startup optimization | Standard Python import | Standard ESM import | N/A | Per-entry-point tree-shaking |

## Best Practices

**1. Start with CLI. Always.**
Every project starts with a CLI. hermes has one. openclaw has one. Claude Code's primary mode is CLI. Even NemoClaw is just a CLI. The reason: CLI is the fastest feedback loop during development. No webhook setup, no OAuth tokens, no platform-specific message format. `stdin` in, `stdout` out. Add messaging platforms later, when you have a working agent.

**2. Use the adapter pattern from day one — but keep it simple.**
You do not need openclaw's full composition system or hermes's 16 adapters. You need one interface with three methods:

```python
class ChannelAdapter(ABC):
    async def start(self) -> None: ...           # connect / listen
    async def stop(self) -> None: ...            # graceful shutdown
    async def send(self, response: AgentResponse) -> None: ...  # send reply
    # Inbound messages arrive via a callback registered during start()
```

Your CLI adapter implements this. When you add Telegram, it also implements this. Core never knows or cares which platform it is talking to.

**3. Keep adapters thin.**
An adapter does protocol translation and nothing else. No authentication logic (that is Gateway). No message routing (that is Routing). No business rules. If your Telegram adapter has an `if user.is_admin:` check, that logic belongs elsewhere.

hermes's individual adapters stay thin because the gateway handles complexity. openclaw's adapters stay thin because capability interfaces limit scope. Claude Code's entry points are NOT thin — they each contain their own setup logic, which is the tradeoff of skipping a shared abstraction.

**4. Distinguish inbound from outbound early in your project.**
When you add MCP support, decide immediately: is this MCP-server mode (inbound — someone else connects to your agent) or MCP-client mode (outbound — your agent connects to external tool servers)? The answer determines which module owns the code. Mixing directions in one module creates confusion that compounds over time.

Claude Code does this clearly: `entrypoints/mcp.ts` is inbound (Interface), while MCP client connections to external servers are outbound (Tools). hermes's ACP adapter is inbound; its tool-calling capabilities are outbound.

**5. Plan for platform-specific capabilities, but do not over-abstract.**
Discord has threads, reactions, and slash commands. Telegram has inline keyboards and message editing. SMS has none of these. openclaw's composition model handles this elegantly — each adapter declares what it supports. hermes's inheritance model handles it with optional method overrides. Both work. What does NOT work: a base class that tries to enumerate every possible platform capability upfront. You will never finish that list.

## Step-by-Step: How to Build It

### Step 1: Define the message contract

Before writing any adapter, define what a message looks like inside your agent.

```python
@dataclass
class InboundMessage:
    text: str
    sender_id: str
    channel: str              # "cli", "telegram", "discord", etc.
    thread_id: str | None     # if the platform supports threads
    attachments: list[Attachment]  # images, files
    raw: dict                 # platform-specific data (for edge cases)

@dataclass
class AgentResponse:
    text: str
    attachments: list[Attachment]
    metadata: dict            # e.g., {"reply_to": message_id}
```

This is the contract between Interface and Agent Core (or Gateway, if you have one). Every adapter converts from its platform's format into `InboundMessage`, and converts `AgentResponse` back into its platform's format.

**Decision point**: Include `raw` or not? Including it lets adapters pass platform-specific data through without the contract knowing every platform's quirks. hermes and openclaw both do this. It prevents the contract from growing every time you add a platform.

### Step 2: Build the CLI adapter

```python
class CLIAdapter(ChannelAdapter):
    def __init__(self, on_message: Callable[[InboundMessage], Awaitable[None]]):
        self.on_message = on_message

    async def start(self):
        while True:
            user_input = await asyncio.to_thread(input, "> ")
            if user_input.strip().lower() in ("exit", "quit"):
                break
            msg = InboundMessage(
                text=user_input,
                sender_id="local_user",
                channel="cli",
                thread_id=None,
                attachments=[],
                raw={},
            )
            await self.on_message(msg)

    async def stop(self):
        pass  # nothing to clean up

    async def send(self, response: AgentResponse):
        print(response.text)
```

This is your entire Interface layer for v1. Ship it. Use it. Build the rest of the agent. Come back here when you need Telegram.

### Step 3: Add a second adapter (when needed)

Pick the platform your users actually use. Implement the same `ChannelAdapter` interface.

```python
class TelegramAdapter(ChannelAdapter):
    def __init__(self, bot_token: str, on_message: Callable):
        self.bot_token = bot_token
        self.on_message = on_message
        self.bot = None

    async def start(self):
        self.bot = TelegramBot(self.bot_token)
        self.bot.on("message", self._handle)
        await self.bot.start_polling()

    async def stop(self):
        await self.bot.stop()

    async def send(self, response: AgentResponse):
        await self.bot.send_message(
            chat_id=response.metadata["chat_id"],
            text=response.text,
        )

    async def _handle(self, update):
        msg = InboundMessage(
            text=update.message.text,
            sender_id=str(update.message.from_user.id),
            channel="telegram",
            thread_id=str(update.message.chat.id),
            attachments=self._parse_attachments(update),
            raw=update.to_dict(),
        )
        await self.on_message(msg)
```

**Decision point**: Who manages adapter lifecycle — a gateway, or the main process? If you only have 2-3 adapters, the main process can start/stop them directly. If you have 10+, you need hermes's gateway pattern: a central manager that starts, stops, health-checks, and restarts adapters.

### Step 4: Add adapter registration (optional)

Once you have 3+ adapters, you want a registry so configuration drives which adapters are active.

```python
class AdapterRegistry:
    def __init__(self):
        self._factories: dict[str, Callable] = {}

    def register(self, channel: str, factory: Callable):
        self._factories[channel] = factory

    def create(self, channel: str, config: dict, on_message: Callable) -> ChannelAdapter:
        if channel not in self._factories:
            raise ValueError(f"Unknown channel: {channel}")
        return self._factories[channel](config=config, on_message=on_message)

# Usage:
registry = AdapterRegistry()
registry.register("cli", CLIAdapter)
registry.register("telegram", TelegramAdapter)
registry.register("discord", DiscordAdapter)

# Start only configured adapters
for channel_config in config["channels"]:
    adapter = registry.create(channel_config["type"], channel_config, handle_message)
    await adapter.start()
```

This is essentially what hermes's gateway does for its 16 adapters, minus the slash command dispatch and health monitoring.

### Step 5 (optional): Add capability interfaces

If your platforms have divergent features and you want type-safe capability checking:

```python
class SupportsThreads(ABC):
    async def reply_in_thread(self, thread_id: str, response: AgentResponse): ...

class SupportsReactions(ABC):
    async def add_reaction(self, message_id: str, emoji: str): ...

class SupportsEditing(ABC):
    async def edit_message(self, message_id: str, new_text: str): ...

# Discord adapter:
class DiscordAdapter(ChannelAdapter, SupportsThreads, SupportsReactions, SupportsEditing):
    ...

# Check at runtime:
if isinstance(adapter, SupportsThreads):
    await adapter.reply_in_thread(thread_id, response)
else:
    await adapter.send(response)  # fallback
```

This is openclaw's model. Do not add this until you have 3+ platforms with genuinely different capabilities. For CLI + one messaging platform, the base `ChannelAdapter` is enough.

## Common Pitfalls

**1. Putting auth logic in adapters.**
Your Telegram adapter should NOT validate API keys or check user permissions. It receives a message, converts it to `InboundMessage`, and passes it through. Authentication belongs in Gateway (Module 7). Mixing auth into adapters means every new adapter must re-implement your auth logic — and they will do it inconsistently.

**2. Building adapters for platforms you do not have users on.**
hermes has 16 adapters. Most projects need 2-3. Build the CLI first. Add one messaging platform when you have real users requesting it. Each adapter is a maintenance commitment: API changes, library updates, edge cases in message formatting. Do not maintain adapters nobody uses.

**3. Skipping the abstraction because "we only need CLI."**
You only need CLI today. The adapter interface costs almost nothing to define upfront (one ABC with three methods). Retrofitting it later means rewriting how Core receives messages. Define the interface now. Implement only CLI. Thank yourself in 3 months.

**4. Fat adapters that contain business logic.**
If your Discord adapter has 500 lines of code, something is wrong. Adapters should be 50-150 lines. Protocol translation, media conversion, lifecycle management. That is it. If you are doing message routing, user preference lookups, or response formatting inside an adapter, extract it.

**5. Treating MCP-inbound and MCP-outbound as the same module.**
Claude Code gets this right: `entrypoints/mcp.ts` (inbound, Interface layer) is completely separate from MCP client connections (outbound, Tools layer). They share the MCP protocol but have opposite data flow directions, different error handling, and different lifecycle concerns. Keep them in separate modules.

## Cross-Module Contracts

### What Interface expects from downstream modules

| Downstream module | Contract | Format |
|-------------------|----------|--------|
| **Gateway** (if exists) | Accept `InboundMessage`, return routing decision | Gateway receives the message; Interface does not care what happens next |
| **Agent Core** (if no Gateway) | Accept `InboundMessage`, return `AgentResponse` | Direct handoff when Gateway is skipped (single-user CLI) |

### What Interface provides to downstream modules

| Consumer | What Interface provides | Format |
|----------|------------------------|--------|
| **Gateway / Agent Core** | Normalized inbound message | `InboundMessage{text, sender_id, channel, thread_id, attachments, raw}` |
| **Gateway / Agent Core** | Response delivery callback | `adapter.send(AgentResponse)` — Core or Gateway calls this to send the reply back to the user |

### What Interface does NOT do

- **Authentication.** Interface does not validate tokens or check permissions. It passes `sender_id` through. Gateway decides if that sender is allowed.
- **Routing.** Interface does not decide which agent handles the message. It passes `channel` and `thread_id` through. Routing (Module 7) decides the destination.
- **Response formatting.** Interface converts `AgentResponse.text` to platform format (e.g., Markdown to Telegram HTML). It does not decide *what* the response says.
- **Message storage.** Interface does not persist messages. Session Store (owned by Agent Core) handles that.

### Invariants

- Every inbound path produces the same `InboundMessage` format. Core and Gateway never need to know which platform a message came from (unless they explicitly check the `channel` field for platform-specific behavior).
- Adapters never call Agent Core directly when a Gateway exists. The flow is always Interface -> Gateway -> Core (or Interface -> Core if no Gateway).
- Adapter lifecycle (start/stop) is managed by either the main process or the Gateway, never by Agent Core.
- Adding a new adapter requires zero changes to Agent Core, Gateway, or any other module. Only the adapter registry configuration changes.
