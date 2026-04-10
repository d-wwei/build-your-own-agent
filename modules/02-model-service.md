# Module 2: Model Service
> A peer service to Agent Core, not a layer beneath it. Its job: get tokens from an LLM provider, handle failures gracefully, and never let a rate limit kill the user's session.

## What Is It

Model Service sits on the same level as Tools, Memory, and Context Manager in the Hub-and-Spoke architecture. Agent Core calls it through an interface. It calls external LLM APIs. It returns responses. That is the entire contract.

The critical insight: Model Service is **stateless**. It has no persistent storage of its own. No database, no files. It holds transient state — which provider is in cooldown, how many tokens this request consumed, whether the prompt cache hit — but none of that survives a restart. This makes it the easiest module to swap, test, and mock. It also means all the complexity is in runtime behavior: routing requests to the right provider, failing over when one goes down, and recovering when it comes back.

Think of it as a load balancer for LLMs. Your application does not call `nginx` a "layer below" the app server. It is a peer service that handles traffic routing. Model Service does the same for model calls.

## What Problem Does It Solve

LLM APIs are unreliable. Not theoretically. Concretely:

1. **Rate limits hit at the worst time.** OpenAI returns 429 mid-conversation. Without failover, the user stares at an error. With a candidate chain, the request silently moves to the backup provider. The user never knows.
2. **Provider outages are real.** Anthropic, OpenAI, Google — they all have downtime windows. A production agent that depends on exactly one endpoint is a production agent that goes down with it.
3. **Cost varies by 10-100x.** GPT-4o costs roughly $2.50/M input tokens. Claude 3.5 Sonnet is $3/M. Gemini 1.5 Flash is $0.075/M. Without provider routing, you cannot optimize cost per task type.
4. **Token estimation prevents surprise bills.** If you do not estimate tokens before sending, you discover you exceeded the context window when the API returns a 400. By then you have already burned the prompt cache and wasted the round trip.
5. **Prompt caching saves 50-90% on repeated prefixes.** Anthropic's prompt cache and OpenAI's cached tokens both require the prefix to be identical across calls. Without a cache-aware Model Service, you accidentally invalidate the cache by changing system prompt assembly order.

## How the 4 Projects Do It

### hermes-agent

**Files**: Model calls are embedded directly in `AIAgent.run_conversation()`. Provider metadata is split across `model_metadata.py` and `auth.py`.

**Implementation**: Calls the OpenAI SDK directly. No abstraction layer, no `ModelService` interface. The model name and API key come from config. There is one `fallback_model` field — if the primary model fails, hermes tries the fallback once. If that also fails, the conversation stops.

**Token estimation**: Relies on `tiktoken` for OpenAI models. Counts are done inline, not through a service.

**Prompt cache**: None. Every request sends the full system prompt from scratch.

**Design choice**: Simplicity over resilience. hermes was built for single-user local operation where the developer can restart the agent if OpenAI goes down. This works for prototyping. It breaks in production.

**Pros**: Zero overhead. No indirection. Easy to read and debug.
**Cons**: Changing from OpenAI to Anthropic means rewriting the core loop. Single-point-of-failure on the LLM provider. No automatic recovery. Metadata scattered across files.

### openclaw

**Files**: `src/model/model-fallback.ts` (candidate chain), `src/model/auth-profile.ts` (credential rotation), model configuration throughout the agent config system.

**Implementation**: The most sophisticated model routing of the four projects.

**Candidate chain**: `ModelCandidate[]` — an ordered list of (provider, model, auth-profile) tuples. When the current candidate fails with a retriable error (429, 503, network timeout), the service moves to the next candidate. Not a single fallback — a *chain* of arbitrary length.

**Auth profile cooldown**: Each auth profile (API key) tracks its own state. If a key gets rate-limited, that specific key enters cooldown. Other keys for the same provider can keep serving. This means you can load-balance across multiple API keys for the same model.

**Transient probing**: When a candidate enters cooldown, the service periodically sends a lightweight probe request to check if it has recovered. If the probe succeeds, the candidate is restored to the active pool. If it fails, cooldown continues. All of this happens transparently — the agent never knows.

**The full flow**:
```
Request arrives
  -> Try candidate[0] (e.g., OpenAI GPT-4o, key-A)
  -> 429 rate limit
  -> Mark key-A cooldown (60s)
  -> Try candidate[1] (e.g., OpenAI GPT-4o, key-B)
  -> Success -> serve response
  -> Meanwhile, after 60s, probe key-A
  -> Key-A responds 200 -> restore to pool
  -> Next request goes back to candidate[0]
```

**Pros**: Production-grade resilience. Zero-downtime provider rotation. Auth key load balancing. Automatic recovery without manual intervention.
**Cons**: Complex to implement correctly. Cooldown timers, probe scheduling, and candidate ordering create subtle bugs if the concurrency model is wrong.

### NemoClaw

**Files**: `src/inference-config.ts`, configuration within the onboard wizard.

**Implementation**: NemoClaw is not an Agent runtime — it is a sandbox orchestrator. Its Model Service contribution is a configuration layer, not an execution layer.

**Unified endpoint**: NemoClaw routes all model calls through a single local endpoint (`inference.local/v1`). Behind this endpoint, it configures access to multiple providers: 7 remote (OpenAI, Anthropic, Google, Mistral, etc.) and 3 local (Ollama, LM Studio, vLLM).

**Onboard wizard**: The 7-step setup wizard collects provider credentials and configures routing rules. The actual model calls are made by the OpenClaw instance running inside the sandbox.

**Design choice**: Separation of configuration from execution. NemoClaw decides *which* providers are available and *how* to reach them. The agent inside the sandbox decides *when* to call which model.

**Pros**: Clean separation. Provider configuration is infrastructure concern, not agent concern.
**Cons**: No runtime failover logic in NemoClaw itself — that depends on whatever agent runs inside the sandbox. Switching providers after onboard requires re-running the wizard or manual config edits.

### Claude Code

**Files**: `src/services/api/client.ts` (HTTP client), `src/services/api/claude.ts` (main entry point), provider-specific modules for Direct API, AWS Bedrock, and Google Vertex.

**Implementation**: Clean dependency injection pattern. `callClaude()` is the single entry point. The concrete provider (Direct, Bedrock, or Vertex) is injected at startup based on configuration. Switching providers means changing one config value, not rewriting code.

**Multi-provider support**: Direct API (Anthropic's own endpoint), AWS Bedrock (for enterprise AWS customers), Google Vertex (for GCP customers). Each provider has its own auth flow, endpoint format, and response parsing.

**Prompt cache**: Anthropic's native prompt cache is supported. The system prompt and tool definitions — which are identical across turns — get cached. Claude Code's cache hit rates are high because the system prompt is large (tool definitions alone can be thousands of tokens) and stable within a session.

**No auto-failover**: Unlike openclaw, Claude Code does not have a candidate chain. If the Anthropic API returns 429, it retries with backoff but does not automatically switch to a different provider. This makes sense for Anthropic's own product (they control both client and server), but limits it as a reference for multi-provider agents.

**Token estimation**: Done before each request to ensure the message array fits within the context window. If it does not, Context Manager is triggered.

**Pros**: Cleanest code structure via DI. Easy to test (inject mock provider). Prompt cache support is built-in. Type-safe across the entire call chain.
**Cons**: No automatic failover between providers. Locked to Anthropic models (by design — it is Anthropic's product).

## Comparison Table

| Dimension | hermes | openclaw | NemoClaw | Claude Code |
|-----------|--------|----------|----------|-------------|
| Abstraction | None (raw SDK) | ModelCandidate[] chain | Config layer only | DI-based provider interface |
| Provider count | 1 (OpenAI) | Unlimited candidates | 7 remote + 3 local (config) | 3 (Direct/Bedrock/Vertex) |
| Failover | Single fallback_model, one retry | Candidate chain + auth cooldown + probe recovery | Depends on inner agent | Retry with backoff, no chain |
| Auth rotation | N/A | Per-key cooldown, multi-key load balance | N/A | N/A |
| Auto-recovery | No | Yes (transient probing) | No | No |
| Prompt cache | None | Provider-dependent | Provider-dependent | Anthropic native cache |
| Token estimation | tiktoken inline | Provider SDK | N/A | Built-in pre-request check |
| Cost tracking | None | Per-request logging | None | Usage stats in response |
| Code quality | Scattered across files | Consolidated, well-structured | Clean config separation | Best DI, fully typed |
| Testing ease | Hard (SDK coupled to Core) | Medium (need to mock chain) | N/A | Easy (inject mock provider) |

## Best Practices

**If you only do one thing: build a candidate chain with cooldown and auto-recovery.**

A single `fallback_model` string is not enough. Here is why: the fallback might also be rate-limited. Or it might be a weaker model that cannot handle your task. Or it might be more expensive and you want to switch back when the primary recovers.

A candidate chain solves all three. You define an ordered list of (provider, model, key) tuples. Failure triggers the next candidate. Cooldown tracks when each candidate can be retried. Probing restores recovered candidates to the pool.

Rules from the four projects:

1. **Core never calls the LLM SDK directly.** Always through a `ModelService` interface. hermes violated this and ended up with model logic scattered across the god class. Claude Code enforced it and got clean DI + easy testing.
2. **Classify errors, do not treat all failures the same.** Rate limit (429) = temporary, retry after cooldown. Auth error (401) = permanent, do not retry. Billing error (402) = permanent until user action. Network timeout = temporary, retry immediately. openclaw classifies errors into these buckets. hermes does not — it retries everything once.
3. **Track cooldown per auth profile, not per provider.** If you have two API keys for OpenAI, and key-A gets rate-limited, key-B might still work. openclaw does this. Everyone else treats the provider as a single unit.
4. **Estimate tokens before sending.** Do not discover that your prompt exceeds the context window from a 400 error. Estimate first, trigger compression if needed, then send. Claude Code and hermes both do this.
5. **Support prompt caching if your provider offers it.** Anthropic and OpenAI both cache repeated prompt prefixes. A stable system prompt (which does not change between turns) is a cache-friendly prefix. If your system prompt is 4000 tokens and you make 20 turns, caching saves roughly 76,000 input tokens per conversation.

## Step-by-Step: How to Build It

### Step 1: Define the interface

Start with the minimal contract that Agent Core needs:

```python
class ModelResponse:
    text: str | None
    tool_calls: list[ToolCall]
    usage: TokenUsage  # input_tokens, output_tokens, cache_hit

class ModelService(ABC):
    def complete(self, messages: list[Message], tools: list[ToolDef]) -> ModelResponse: ...
    def estimate_tokens(self, messages: list[Message]) -> int: ...
```

That is it. Core calls `complete()` and gets back text and/or tool calls. Core calls `estimate_tokens()` before sending to check context window fit.

**Decision point**: Streaming or batch? If your interface is a CLI that shows tokens as they arrive, you need streaming. Add a `stream()` method that yields chunks. If your interface is an API that returns complete responses, batch is simpler.

### Step 2: Implement one provider

Pick your primary provider and implement the interface:

```python
class AnthropicProvider(ModelService):
    def __init__(self, api_key: str, model: str = "claude-sonnet-4-20250514"):
        self.client = Anthropic(api_key=api_key)
        self.model = model

    def complete(self, messages, tools):
        response = self.client.messages.create(
            model=self.model,
            messages=messages,
            tools=tools,
            max_tokens=4096,
        )
        return ModelResponse(
            text=extract_text(response),
            tool_calls=extract_tool_calls(response),
            usage=TokenUsage(response.usage),
        )
```

Do not abstract too early. Get one provider working end-to-end before adding the second.

### Step 3: Add a candidate chain

This is where it gets interesting. A candidate is a provider + model + auth credential tuple:

```python
@dataclass
class ModelCandidate:
    provider: ModelService
    priority: int
    cooldown_until: float = 0      # timestamp
    consecutive_failures: int = 0

class CandidateChain(ModelService):
    def __init__(self, candidates: list[ModelCandidate]):
        self.candidates = sorted(candidates, key=lambda c: c.priority)

    def complete(self, messages, tools):
        for candidate in self.candidates:
            if candidate.cooldown_until > time.time():
                continue  # still in cooldown, skip

            try:
                response = candidate.provider.complete(messages, tools)
                candidate.consecutive_failures = 0  # reset on success
                return response
            except RateLimitError:
                candidate.consecutive_failures += 1
                candidate.cooldown_until = time.time() + self._backoff(candidate)
                continue
            except AuthError:
                # permanent failure — remove from chain or alert
                raise
            except NetworkError:
                # transient — try next candidate immediately
                continue

        raise AllCandidatesExhaustedError()
```

**Decision point**: Fixed cooldown or exponential backoff? openclaw uses exponential backoff per candidate. Start with a fixed 60-second cooldown. Add exponential backoff when you observe that 60 seconds is not always enough.

### Step 4: Add auto-recovery probing

Without probing, a candidate that entered cooldown stays there until the cooldown timer expires. Probing actively checks whether the candidate has recovered:

```python
class CandidateChain(ModelService):
    async def _probe_loop(self):
        """Background task: periodically probe cooled-down candidates."""
        while True:
            await asyncio.sleep(30)  # probe every 30 seconds
            for candidate in self.candidates:
                if candidate.cooldown_until <= time.time():
                    continue  # not in cooldown
                try:
                    # lightweight probe: tiny message, low max_tokens
                    candidate.provider.complete(
                        messages=[{"role": "user", "content": "ping"}],
                        tools=[],
                    )
                    # success — restore candidate
                    candidate.cooldown_until = 0
                    candidate.consecutive_failures = 0
                except Exception:
                    pass  # still down, continue cooldown
```

This is openclaw's "transient probing" pattern. The probe is cheap (tiny request, minimal tokens). The benefit is large: when the primary provider recovers, your agent switches back automatically instead of staying on the expensive fallback.

**Decision point**: Run probes in a background task or check lazily on next request? Background probing gives faster recovery. Lazy checking is simpler and avoids wasted probe requests when the agent is idle.

### Step 5: Add token estimation

Before sending a request, check that it fits:

```python
class CandidateChain(ModelService):
    def complete(self, messages, tools):
        estimated = self.estimate_tokens(messages)
        if estimated > self.context_window * 0.95:
            raise ContextWindowExceededError(estimated, self.context_window)
        # ... proceed with candidate chain
```

Agent Core catches `ContextWindowExceededError` and triggers Context Manager compression before retrying. This is the coordination point between Model Service and Context Manager — they do not call each other directly, but Core mediates.

### Step 6 (optional): Add prompt cache awareness

If using Anthropic or OpenAI with prompt caching:

```python
def complete(self, messages, tools):
    # Ensure system prompt is the first element and identical across calls.
    # Provider SDKs handle caching automatically if the prefix matches.
    # Key: do NOT change the order of system prompt components between turns.
    system_prompt = self._build_stable_system_prompt()  # deterministic output
    return self.client.messages.create(
        system=system_prompt,  # cached prefix
        messages=messages,     # variable suffix
        tools=tools,           # also part of cached prefix if stable
    )
```

The main pitfall: if your system prompt includes a timestamp ("Current time: 2026-04-09 14:32:07"), the prefix changes every second and the cache never hits. Put volatile data at the end of the system prompt or in a user message, not at the beginning.

## Common Pitfalls

**1. No Abstraction at All (hermes)**
hermes calls `openai.chat.completions.create()` directly in the agent loop. This means:
- Switching to Anthropic requires rewriting the core loop.
- Testing requires mocking the OpenAI SDK globally.
- Adding a second provider means duplicating the entire call path.
This seems fine when you have one provider and one developer. It stops being fine the day OpenAI has a 4-hour outage and your agent is dead for 4 hours.

**2. Abstraction Without Failover (Claude Code)**
Claude Code has clean DI and a proper `ModelService` interface. But it does not have a candidate chain. If the Anthropic API goes down, Claude Code retries with backoff until it either succeeds or gives up. For Anthropic's own product this is acceptable (they would rather fix their API than route to a competitor). For your agent, this is not a model to copy.

**3. Treating All Errors the Same**
A 429 (rate limit) means "try again in 60 seconds." A 401 (bad API key) means "this will never work until the key is fixed." A 500 (server error) means "try again immediately or try another provider." If you retry a 401 with exponential backoff, you waste time. If you give up on a 429 immediately, you miss an easy recovery.

Error classification table:

| Error | Type | Action |
|-------|------|--------|
| 429 Rate Limit | Transient | Cooldown candidate, try next |
| 503 Overloaded | Transient | Cooldown candidate, try next |
| Network timeout | Transient | Try next candidate immediately |
| 401 Unauthorized | Permanent | Remove candidate from chain, alert |
| 402 Billing | Permanent | Remove candidate, alert user |
| 400 Bad Request | Bug | Do not retry, fix the request |

**4. Ignoring Prompt Cache Invalidation**
Your system prompt has 4000 tokens. You make 20 turns. With caching, turns 2-20 get a cache hit on those 4000 tokens = ~72,000 tokens saved. But if you insert "Current iteration: 7" into the middle of the system prompt on turn 7, the cache is invalidated for all subsequent turns. Put variable data *after* the stable prefix, or in user messages.

**5. Missing Token Estimation**
You assemble a 180,000-token message array and send it to a model with a 200,000-token context window. The model has 20,000 tokens left for output — maybe enough, maybe not. Without pre-send estimation, you discover this from a truncated response or a 400 error. Estimate first, compress if needed, then send.

## Cross-Module Contracts

### What Model Service expects from other modules

| Module | Contract | Notes |
|--------|----------|-------|
| **Agent Core** | Calls `complete(messages, tools)` | Messages in standard chat format. Tools as JSON schema definitions. |
| **Agent Core** | Calls `estimate_tokens(messages)` | Before sending, to check context window fit. |
| **Context Manager** | Provides compressed messages when needed | Core mediates: if `estimate_tokens` exceeds threshold, Core calls Context Manager, then retries Model. Model and Context Manager never talk directly. |
| **Config** | Provider credentials, model names, candidate order | Read at startup. Model Service does not watch for config changes (stateless). |

### What Model Service provides to other modules

| Consumer | What it provides | Format |
|----------|-----------------|--------|
| **Agent Core** | `ModelResponse` | `.text` (assistant message or null), `.tool_calls[]` (tool invocations), `.usage` (input/output/cache token counts) |
| **Agent Core** | Token estimate | Integer: estimated token count for a message array |
| **Context Manager** (indirect) | Usage data | Core reads `.usage` from ModelResponse and passes it to Context Manager for budget tracking. Model Service does not call Context Manager. |

### Invariants

- Model Service is stateless across restarts. Cooldown timers, probe states, and candidate health are transient runtime state.
- Model Service never modifies the `messages` array. It receives messages, sends them to the provider, and returns the response. Modification is Context Manager's job.
- Model Service never decides *what* to send. Agent Core constructs the message array. Model Service decides *where* to send it (which provider/candidate).
- Error classification is Model Service's responsibility. Core receives either a `ModelResponse` or a typed exception (`AllCandidatesExhaustedError`, `PermanentAuthError`). Core does not parse HTTP status codes.
- Token estimation is an approximation. Different providers tokenize differently. The estimate should be conservative (overcount rather than undercount) to avoid context window overflow.
