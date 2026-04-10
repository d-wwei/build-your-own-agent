# Module 7: Gateway + Routing
> The optional bouncer and dispatcher. Gateway asks "can this request come in?" Routing asks "who should handle it?" Skip both for your v1 single-user CLI.

## What Is It

Two distinct concerns bundled into one module because they sit at the same architectural position — between Interface and Agent Core — and because most projects that need one also need the other.

**Gateway** is the door guard. Every request passes through it. It answers three questions in sequence:
1. **Authentication** — Is this request from a legitimate source?
2. **Authorization** — Does this source have permission to do what it is asking?
3. **Rate limiting** — Has this source exceeded its quota?

If all three pass, Gateway performs **method dispatch** — it looks at the request type (`chat.send`, `config.update`, `agent.status`) and routes it to the correct handler function. This is conceptually identical to Express.js looking at a URL path and calling the matching controller. The only difference: most agent gateways use WebSocket with a `method` field instead of HTTP with URL paths.

**Routing** is the message dispatcher. It answers one question: **"Which agent instance should handle this message?"** This only matters when you have multiple agents. A single-agent system does not need routing — there is only one destination.

Routing is triggered only by conversation messages, not by all requests. Config updates, status checks, admin commands — those go through Gateway's method dispatch and skip Routing entirely.

**The key distinction:**

| | Gateway | Routing |
|---|---|---|
| Triggers on | Every request | Only conversation messages |
| Question asked | "Can this come in?" | "Where does this go?" |
| Required when | Multiple users OR multiple channels | Multiple agents ONLY |
| Single-user CLI | Not needed | Not needed |
| Multi-user, single agent | Needed (auth + rate limit) | Not needed |
| Multi-user, multi agent | Needed | Needed |

Claude Code needs neither. It runs locally, for one user, with one agent. This entire module does not exist in its codebase.

## What Problem Does It Solve

**Without Gateway:**
- Anyone who knows your WebSocket endpoint can send commands. No authentication.
- A runaway client can flood your agent with requests. No rate limiting.
- Every adapter must implement its own auth checks. Inconsistent enforcement.
- Method dispatch logic scatters across adapters or, worse, inside Agent Core.

**Without Routing (in a multi-agent system):**
- Messages go to a hardcoded agent. Adding a second agent means rewriting the dispatch logic.
- Thread continuity breaks. A reply in a Discord thread goes to a different agent than the original message.
- No fallback chain. If no specific binding exists, the message has nowhere to go.

openclaw learned these lessons the hard way and built the most complete Gateway + Routing system of the four projects. hermes partially solved Gateway through its 7620-line `gateway/run.py` but never needed Routing because it runs a single agent.

## How the 4 Projects Do It

### openclaw

**Gateway: WebSocket control plane with 25+ handler groups.**

openclaw's gateway is a WebSocket server. Clients (the web UI, CLI, external integrations) connect via WebSocket and send JSON messages with a `method` field. The gateway dispatches each method to its handler group.

Handler groups include: `chat.*` (send message, edit, delete), `config.*` (read, update, apply), `agent.*` (status, restart, list), `channel.*` (connect, disconnect, health), `memory.*` (search, inspect), `admin.*` (user management, permissions), and more. 25+ groups total.

**Authentication model**: Role-based with scope restrictions.
- Roles: `owner`, `admin`, `user`, `viewer`, `api_key`
- Each handler declares its required role and scopes
- Gateway checks the connection's auth token against the handler's requirements before dispatch

**Rate limiting**: Budget-based, not simple request counting.
- Each user gets a token budget (e.g., 100k tokens/day)
- Gateway deducts estimated tokens before forwarding to the agent
- Actual usage is reconciled after the response
- Separate limits per role (owner gets unlimited, api_key gets 10k/day)

**The WebSocket choice matters.** HTTP would work for request/response, but agent conversations are inherently streaming. The user sends a message, and the response arrives as a stream of tokens over seconds. WebSocket is the natural fit. openclaw chose correctly.

**Routing: 9-level binding priority, independent module.**

This is the most sophisticated routing system in the four projects. When a message arrives from any channel, Routing checks bindings in this order:

```
Priority 1: Exact chat/group match     → "Telegram group 12345 → investmentAgent"
Priority 2: Parent thread inheritance   → reply in a thread goes to same agent as parent
Priority 3: Wildcard match             → "All Telegram groups → generalAgent"
Priority 4: Discord server + role      → "Server X, role Y → specialistAgent"
Priority 5: Discord server             → "Server X → serverAgent"
Priority 6: Slack team                 → "Team Z → teamAgent"
Priority 7: Account-level              → "Telegram account → accountAgent"
Priority 8: Channel-level              → "All Discord messages → discordAgent"
Priority 9: Default fallback           → the catch-all agent
```

First match wins. This means you can set broad defaults (Priority 9) and override them with specific bindings (Priority 1) without conflict.

**Architectural insight:** In openclaw, Routing is a separate module (`src/routing/`) called by the channel layer, NOT by Gateway. Gateway handles auth and rate limiting. Channels handle protocol translation. Routing sits between channels and Agent Core. This three-way split — Gateway guards the door, channels translate the protocol, Routing picks the destination — is cleaner than putting everything in Gateway.

### hermes-agent

**Gateway: `gateway/run.py` — 7620 lines managing adapter lifecycle + slash command dispatch.**

hermes's gateway is not a standalone service. It is the main process that boots up, starts all configured adapters, and dispatches incoming messages.

**What it does:**
- Starts/stops adapter instances (the 16 platform adapters from Module 6)
- Registers and dispatches slash commands (`/help`, `/status`, `/skills`, custom commands)
- Manages the adapter health loop (restart crashed adapters)
- Routes messages from adapters to the single `AIAgent` instance

**What it does NOT do:**
- Formal authentication (trusts platform-level auth — if Telegram delivered the message, the user is valid)
- Role-based authorization (no roles system)
- Token-budget rate limiting (no rate limits)

**Why 7620 lines?** Because hermes's gateway mixes concerns. Adapter lifecycle management, slash command registration, health monitoring, and message dispatch are all in one file. openclaw splits these into separate modules.

**Routing: None.** hermes is a single-agent system. Every message goes to the one `AIAgent` instance. There is no binding table, no priority chain, no agent selection logic.

**CLI bypass:** When running in CLI mode, hermes skips the gateway entirely. The CLI adapter talks directly to `AIAgent`. This is the right call — a single user typing in a terminal does not need auth, rate limiting, or method dispatch.

### NemoClaw

**Not applicable in the traditional sense.** NemoClaw wraps OpenShell sandbox management, not request handling. It does have a `gateway-state.ts` that tracks the health of the OpenClaw gateway running *inside* the sandbox, but this is infrastructure monitoring, not a request gateway.

The one relevant piece: NemoClaw's `gateway-state.ts` classifies the sandbox gateway into health states (healthy, degraded, dead) and triggers recovery actions (SSH probe, curl health check, process restart). This is a pattern worth noting for Module 10 (Runtime/Infrastructure), not for this module.

### Claude Code

**Does not exist.** Claude Code runs locally as a single-user CLI agent. There is no gateway, no authentication layer, no rate limiter, no method dispatch, no routing.

This is not a gap — it is a deliberate architectural choice. When you have exactly one user running the process locally, every gateway feature is overhead with zero benefit:
- Authentication? The user launched the process. They are authenticated by virtue of being the OS user.
- Authorization? Single user, single agent, full permissions.
- Rate limiting? The user is rate-limited by how fast they type.
- Routing? One agent. Nothing to route.

Claude Code does have a permission system (ask/auto mode for tool execution), but this is Security Envelope (Module 0), not Gateway. The permission check happens at the tool execution layer, not at the request entry point.

**The lesson:** Do not build a gateway because you think you should. Build it when you have multiple users or multiple channels that need auth. For a v1 single-user CLI, skip it entirely.

## Comparison Table

| Dimension | hermes | openclaw | NemoClaw | Claude Code |
|-----------|--------|----------|----------|-------------|
| Gateway exists | Yes (mixed into `gateway/run.py`) | Yes (standalone WebSocket control plane) | N/A (infra health monitor only) | No |
| Protocol | Adapter-specific (each platform's native) | WebSocket with JSON method dispatch | N/A | N/A |
| Authentication | Platform-trust only | Role + scope + token-based | N/A | N/A (OS user = authenticated) |
| Authorization | None (no roles) | 5 roles, per-handler scope checks | N/A | N/A |
| Rate limiting | None | Token-budget with role-based quotas | N/A | N/A |
| Method dispatch | Slash commands only | 25+ handler groups | N/A | N/A |
| Routing exists | No (single agent) | Yes (independent `src/routing/` module) | No | No |
| Routing complexity | N/A | 9-level binding priority | N/A | N/A |
| Gateway file size | 7620 lines (mixed concerns) | Distributed across modules | N/A | N/A |
| CLI bypasses gateway | Yes | Yes (direct mode available) | N/A | N/A (no gateway to bypass) |

## Best Practices

**1. Skip Gateway entirely for your v1 single-user CLI.**
Claude Code ships without one. hermes's CLI mode bypasses it. You do not need auth, rate limiting, or method dispatch when one human is typing into a terminal. Build the agent first. Add the Gateway when you add your second user or your second channel.

**2. When you add Gateway, start with auth + rate limiting only.**
Do not build 25 handler groups on day one. Start with:
- One auth check (API key or session token)
- One rate limiter (requests per minute, not token budget — that comes later)
- Pass-through to Agent Core for everything else

You can add method dispatch, role-based authorization, and token budgets incrementally.

**3. Gateway and Routing are separate modules.**
openclaw proves this. Gateway handles auth and rate limiting (every request). Routing handles agent selection (only messages, only multi-agent). Merging them means your rate limiter knows about agent bindings and your routing logic knows about API keys. Neither should.

In practice, the call flow is: Interface -> Gateway (auth + rate limit) -> Routing (agent selection) -> Agent Core. If you only have one agent, drop Routing and the flow becomes: Interface -> Gateway -> Agent Core.

**4. Do not build Routing until you have 2+ agents.**
Routing exists solely to answer "which agent handles this?" If you have one agent, the answer is always the same. openclaw needs 9-level priority because it runs different agents for different Discord servers, Slack teams, and Telegram groups. If you are not doing that, you do not need routing.

**5. Let platforms handle their own auth when possible.**
hermes trusts platform-level authentication. If Telegram delivered a message to your bot's webhook, Telegram has already verified the sender. If Discord sent a gateway event, Discord has already authenticated the connection. Your Gateway only needs to add auth for channels that do not provide it (raw HTTP API, WebSocket connections from custom clients).

**6. Rate limiting should be per-user, not per-request.**
openclaw's token-budget approach is better than simple request counting because:
- One complex request ("analyze this 50-page document") costs 100x more than one simple request ("what time is it")
- Request counting penalizes chatty users while letting expensive single requests through
- Token budgets align costs with actual resource consumption

For v1, start with requests-per-minute. Move to token budgets when you have usage data.

## Step-by-Step: How to Build It

### Step 1: Decide if you need a Gateway at all

Answer these questions:
1. Do you have more than one user? If no → skip Gateway.
2. Do external systems call your agent via API? If no → skip Gateway.
3. Do you have multiple channels (Telegram + Discord + Web)? If yes → you probably need Gateway for consistent auth.

If you answered "no" to all three, skip this module entirely. Interface talks directly to Agent Core. Come back when the answer changes.

### Step 2: Add a minimal Gateway (auth + pass-through)

```python
class Gateway:
    def __init__(self, agent_core, auth_provider, rate_limiter):
        self.core = agent_core
        self.auth = auth_provider
        self.rate = rate_limiter

    async def handle(self, request: InboundMessage, credentials: dict) -> AgentResponse:
        # 1. Authenticate
        user = self.auth.verify(credentials)
        if not user:
            raise AuthError("Invalid credentials")

        # 2. Rate limit
        if not self.rate.allow(user.id):
            raise RateLimitError(f"Rate limit exceeded for {user.id}")

        # 3. Pass through to Agent Core
        return await self.core.handle(request)
```

Three checks, one pass-through. That is your entire Gateway for v1. No method dispatch. No roles. No token budgets.

### Step 3: Add method dispatch (when you need admin commands)

When you want to support commands beyond chat (status checks, config updates, agent restart), add method dispatch.

```python
class Gateway:
    def __init__(self, ...):
        self.handlers: dict[str, Callable] = {
            "chat.send": self._handle_chat,
            "agent.status": self._handle_status,
            "config.get": self._handle_config_get,
            "config.update": self._handle_config_update,
        }

    async def handle(self, method: str, payload: dict, credentials: dict) -> Any:
        user = self.auth.verify(credentials)
        if not user:
            raise AuthError("Invalid credentials")

        handler = self.handlers.get(method)
        if not handler:
            raise MethodNotFoundError(f"Unknown method: {method}")

        if not self.rate.allow(user.id, method):
            raise RateLimitError("Rate limit exceeded")

        return await handler(payload, user)
```

**Decision point**: HTTP or WebSocket? If your agent only does request/response (user sends message, agent replies once), HTTP is simpler. If your agent streams responses token-by-token, WebSocket is necessary. Most production agents stream, so WebSocket is the pragmatic default.

### Step 4: Add role-based authorization (when you need it)

```python
# Define roles and their allowed methods
ROLE_PERMISSIONS = {
    "owner": {"*"},                          # everything
    "admin": {"chat.*", "config.get", "agent.status"},
    "user": {"chat.send", "agent.status"},
    "viewer": {"agent.status"},
}

class Gateway:
    async def handle(self, method: str, payload: dict, credentials: dict) -> Any:
        user = self.auth.verify(credentials)
        if not user:
            raise AuthError("Invalid credentials")

        # Check role permissions
        if not self._has_permission(user.role, method):
            raise ForbiddenError(f"Role '{user.role}' cannot call '{method}'")

        # ... rest of dispatch
```

openclaw has 5 roles with per-handler scope checks. Start with 3 roles (`owner`, `user`, `viewer`) and add granularity when you hit real permission conflicts.

### Step 5: Add Routing (only for multi-agent)

If and only if you have multiple agent instances:

```python
class Router:
    def __init__(self):
        self.bindings: list[Binding] = []  # ordered by priority

    def resolve(self, message: InboundMessage) -> str:
        """Returns the agent_id that should handle this message."""
        for binding in self.bindings:
            if binding.matches(message):
                return binding.agent_id
        return self.default_agent_id

@dataclass
class Binding:
    priority: int
    agent_id: str
    channel: str | None       # match specific channel
    chat_id: str | None       # match specific chat/group
    thread_parent: str | None # inherit from parent thread
    role: str | None          # match platform role (Discord)

    def matches(self, message: InboundMessage) -> bool:
        if self.chat_id and message.thread_id == self.chat_id:
            return True
        if self.channel and message.channel == self.channel:
            return True
        # ... more match conditions
        return False
```

**Decision point**: How many priority levels? openclaw has 9. Start with 3:
1. Exact chat/group match
2. Channel-level match
3. Default fallback

Add levels when you encounter real routing conflicts. Do not pre-build 9 levels because openclaw has them.

### Step 6: Wire the full flow

```python
# Full flow with Gateway + Routing:
async def on_message(message: InboundMessage, credentials: dict):
    # Gateway: auth + rate limit
    user = gateway.authenticate(credentials)
    gateway.rate_limit(user)

    # Routing: which agent?
    agent_id = router.resolve(message)
    agent = agent_registry.get(agent_id)

    # Agent Core: handle the message
    response = await agent.handle(message)

    # Interface: send response back
    adapter = adapter_registry.get(message.channel)
    await adapter.send(response)

# Without Gateway or Routing (single-user CLI):
async def on_message(message: InboundMessage):
    response = await agent_core.handle(message)
    await cli_adapter.send(response)
```

The second version is where you start. The first version is where you end up — if you need it.

## Common Pitfalls

**1. Building a Gateway for a single-user CLI.**
The most common over-engineering mistake. You read about openclaw's 25+ handler groups and think you need that. You do not. Claude Code — a production agent used by thousands — does not have a gateway. Start without one. Add it when you have the problem it solves.

**2. Putting Routing inside Gateway.**
Gateway asks "can this come in?" (auth, rate limit, method dispatch). Routing asks "where does it go?" (agent selection). These are different concerns with different trigger conditions (every request vs only messages). openclaw keeps them in separate modules (`src/gateway/` vs `src/routing/`). If you merge them, your auth middleware starts knowing about agent bindings, and refactoring either requires touching both.

**3. Reimplementing platform auth.**
Discord verifies user identity before delivering events to your bot. Telegram verifies webhook signatures. Slack sends signed requests. If you re-verify users that the platform already authenticated, you are doing double work and maintaining duplicate auth logic. Trust platform auth for platform channels. Add your own auth only for raw API/WebSocket channels.

**4. Flat routing without priority.**
If you store bindings as a flat map (`chat_id -> agent_id`), you cannot have fallback chains. A message from an unmapped chat has nowhere to go. openclaw's prioritized binding list means every message eventually matches *something* — even if it is just the Priority 9 default. Always have a default fallback.

**5. Rate limiting by request count instead of cost.**
"100 requests per hour" treats a one-word greeting the same as a request to analyze a 200-page PDF. The user who sends 100 greetings gets blocked. The user who sends 1 expensive request costs you 100x more and stays under the limit. openclaw's token-budget model avoids this. Even a rough estimate (short messages = 100 tokens, long messages = 1000 tokens) is better than flat counting.

**6. Adding Routing before you have 2 agents.**
Routing is a zero-value module when you have one agent. It adds code, configuration, and a failure mode (what if the binding is misconfigured?) with no benefit. The message is going to the only agent that exists. Wait until you actually deploy a second agent instance.

## Cross-Module Contracts

### What Gateway expects from other modules

| Module | Contract | Format |
|--------|----------|--------|
| **Interface** | Deliver inbound request with credentials | `InboundMessage` + auth token/credentials |
| **Auth Provider** | Verify credentials, return user info | `verify(credentials) -> User{id, role, scopes}` |
| **Rate Limiter** | Check and decrement quota | `allow(user_id, method?) -> bool` |

### What Gateway provides to other modules

| Consumer | What Gateway provides | Format |
|----------|----------------------|--------|
| **Routing** (if exists) | Authenticated, rate-limited message | `InboundMessage` with verified `user_id` |
| **Agent Core** (if no Routing) | Authenticated, rate-limited message | Same `InboundMessage` format from Module 6 |
| **Method handlers** | Dispatched requests for non-chat methods | Handler-specific payload |

### What Routing expects from other modules

| Module | Contract | Format |
|--------|----------|--------|
| **Gateway or Interface** | Authenticated inbound message | `InboundMessage` with `channel`, `thread_id`, `sender_id` |
| **Binding Store** | Ordered list of bindings | `Binding{priority, agent_id, match_conditions}` |

### What Routing provides to other modules

| Consumer | What Routing provides | Format |
|----------|----------------------|--------|
| **Agent Core** | Resolved destination + original message | `(agent_id, InboundMessage)` |

### Invariants

- Gateway runs on every request. No exceptions. If Gateway exists, nothing bypasses it (except possibly CLI in development mode, as hermes does).
- Routing runs only on conversation messages. Config updates, status checks, and admin commands are handled by Gateway's method dispatch, not by Routing.
- Routing never modifies the message. It reads `channel`, `thread_id`, and platform metadata to select an agent. The message arrives at Agent Core unchanged.
- Gateway never selects an agent. Its job ends after auth + rate limit + method dispatch. Agent selection is exclusively Routing's responsibility.
- A system with one agent has no Routing module. A system with one local user has no Gateway module. Both are additive — their absence does not require any changes to Interface or Agent Core.
- The binding list always has a default fallback at the lowest priority. No message should ever fail routing with "no matching binding."
