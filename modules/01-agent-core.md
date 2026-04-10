# Module 1: Agent Core
> The Hub that orchestrates everything. Without it, Model, Tools, Memory, and Context Manager are just libraries sitting on a shelf.

## What Is It

Agent Core is the center of the Hub-and-Spoke architecture. It does not "manage" the other modules — it *coordinates* them. Think of a project manager sitting in the middle of a room: left hand asks the designer for mockups, right hand asks the engineer for a build, front desk queries the analyst for data. Whoever finishes first delivers first. The project manager stitches results together.

Every Agent you will ever build has this exact loop at its heart: `while (budget > 0): build context -> call model -> parse response -> execute tools -> repeat`. The four projects all converge on this pattern. They diverge wildly on how clean that loop stays.

Agent Core owns two things directly: Session Store (conversation history, usually SQLite or JSONL) and Config Store (agent configuration, YAML or JSON). Everything else — model calls, tool execution, memory reads, context compression — it delegates to peer services. The moment you let Core *do* those things instead of *delegate* them, you are on the path to a 9431-line god class.

## What Problem Does It Solve

Without a central orchestrator, you get spaghetti. Model calls happen inside tool handlers. Tool results get jammed into prompts without structure. Context grows until the LLM chokes. There is no iteration budget, so a confused model loops forever.

Concrete numbers from the four projects:
- hermes-agent's `AIAgent` class reached **9431 lines** because it mixed orchestration with model calls, tool execution, memory management, and context compression. Changing one thing risks breaking four others.
- Claude Code's `REPL.tsx` hit **5006 lines** by blending five architectural layers into one file.
- openclaw kept `runEmbeddedPiAgent()` focused on pure orchestration and avoided both problems.

The core pain points:
1. **Runaway loops** — Without an iteration budget, a stuck model burns tokens indefinitely. hermes defaults to 90 iterations. That number exists because someone hit 100+ in production.
2. **Context explosion** — Without structured context construction, prompts balloon past window limits mid-conversation.
3. **Delegation chaos** — Sub-agents without isolation corrupt the parent's context, share tools they should not have, or recurse infinitely.

## How the 4 Projects Do It

### hermes-agent

**File**: `hermes/ai_agent/run_agent.py` — the infamous 9431-line `AIAgent` class.

**Loop**: Synchronous `while` loop. Call `run_conversation()`, get model response, check for tool calls, execute them sequentially (or up to 8 in parallel via path-independence detection), append results, repeat.

**Iteration budget**: `IterationBudget` — a thread-safe counter defaulting to **90 iterations**. Decrements on each loop pass. When exhausted, the agent force-stops.

**Context construction**: `prompt_builder` assembles system prompt from identity config + loaded skills + memory injections + SKILLS_GUIDANCE instructions. All done inside `AIAgent`. This is where separation breaks down — prompt assembly logic lives in the same class as loop control.

**Sub-agent delegation**: `delegate_tool` spawns a new `AIAgent` instance with `MAX_DEPTH=2`, up to 3 parallel sub-agents. Sub-agents cannot recursively delegate, cannot ask the user for clarification, and cannot write to shared Memory.

**Pros**: Dead simple to understand. One file, one class, follow the while loop. Great for learning.
**Cons**: God class. Changing model call logic means touching the same file as tool dispatch logic. Refactoring requires surgery on a 9431-line patient.

### openclaw

**File**: `src/agent/runEmbeddedPiAgent()` — wraps `pi-agent-core` (third-party library).

**Loop**: Event-driven via the pi-agent-core execution engine. openclaw adds Lane-based concurrency control: messages within the same session queue up; different sessions run in parallel.

**Hook system**: This is openclaw's signature move. Agent Core emits lifecycle hooks (`beforeModelCall`, `afterToolExec`, `onContextPressure`, etc.) and other modules subscribe. Model service, Tools, Memory, Context Engine — all register through hooks instead of being called directly. This inversion of control keeps Core lean.

**Context construction**: Handled by the pluggable Context Engine (`src/context-engine/`), registered via `registerContextEngine()`. Core triggers it; Core does not contain it.

**Sub-agent delegation**: Lane queue supports sub-agent events (Discord `subagent_spawning`). The infrastructure is there, but usage is lighter than hermes or Claude Code.

**Pros**: Cleanest separation of the four projects. Core is a pure orchestrator. Adding a new module means registering a hook, not editing Core.
**Cons**: Depends on `pi-agent-core` third-party library. You inherit its opinions and its bugs.

### NemoClaw

NemoClaw is not an Agent — it is a sandbox orchestrator for OpenClaw instances. It does not have its own Agent Core. It wraps OpenClaw's agent runtime inside a hardened container with Landlock, seccomp, and per-binary network policies. Mentioned here for completeness, but if you are building Agent Core, NemoClaw's contribution is in the Security Envelope, not here.

### Claude Code

**Files**: `src/core/query.ts` (inner loop) + `src/entrypoints/REPL.tsx` (outer loop).

**Loop**: Async generator pattern. `query.ts` yields events (model tokens, tool calls, tool results, errors) as they happen. `REPL.tsx` consumes the generator incrementally, updating the terminal UI in real time.

**Dual-layer orchestration**:
- **Inner loop** (`query.ts`): Handles one model call -> tool execution cycle. Yields events.
- **Outer loop** (`REPL.tsx`): Manages the multi-turn conversation session. Decides when to re-enter the inner loop, handles user interrupts, manages speculative execution state.

This split means the inner loop stays focused on the model-tool cycle while the outer loop handles session-level concerns (abort signals, permission prompts, UI updates).

**Speculative execution**: When the model returns a tool call requiring permission, Core simultaneously shows the permission prompt to the user AND starts executing the tool. If the user approves — result is already done. If denied — result is discarded via `CompletionBoundary` rollback. This is why Claude Code *feels* faster.

**Sub-agent delegation**: `AgentTool` supports worktree isolation (git worktrees for file-system isolation), fork (child process), team/swarm mode (multiple agents coordinating), and coordinator mode. The richest delegation model of the four projects.

**Pros**: Async generator is elegant. Speculative execution is a genuine UX innovation. Dual-layer keeps concerns separated better than hermes.
**Cons**: `REPL.tsx` still reached 5006 lines by mixing 5 layers. The async generator pattern is harder to debug than a simple while loop.

## Comparison Table

| Dimension | hermes | openclaw | NemoClaw | Claude Code |
|-----------|--------|----------|----------|-------------|
| Core file(s) | `run_agent.py` (9431 lines) | `runEmbeddedPiAgent()` | N/A (sandbox orchestrator) | `query.ts` + `REPL.tsx` (5006 lines) |
| Loop style | Sync while | Event-driven + Lane queue | N/A | Async generator, dual-layer |
| Iteration budget | IterationBudget, default 90 | pi-agent-core managed | N/A | Token/turn limits |
| Context construction | Inside AIAgent (mixed) | Pluggable Context Engine via hooks | N/A | Separate services, invoked from inner loop |
| Sub-agent model | MAX_DEPTH=2, 3 parallel, no recursion | Lane queue + hook events | N/A | worktree / fork / team / swarm / coordinator |
| Separation quality | God class (worst) | Hook-based inversion of control (best) | N/A | Dual-layer (good but REPL bloated) |
| Session Store | SQLite via MemoryProvider | pi-agent-core managed | N/A | JSONL + SQLite |
| Speculative execution | No | No | N/A | Yes (SpeculationState + CompletionBoundary) |

## Best Practices

**If you only do one thing: keep Core as a pure orchestrator.** It should call Model, Tools, Memory, and Context Manager — never contain their logic.

The moment you write `openai.chat.completions.create()` inside your orchestration loop instead of calling `modelService.complete(messages)`, you have coupled Core to a specific provider. The moment you write file-reading logic inside Core instead of calling `toolService.execute("read_file", path)`, you have made Core responsible for I/O. These seem harmless at first. After 6 months and 9431 lines, they are not.

Rules from the four projects:

1. **Core owns the loop, nothing else.** Build context -> call model -> dispatch tools -> check budget -> repeat. That is the entire responsibility.
2. **Delegate via interfaces, not direct calls.** openclaw's hook system is the gold standard. At minimum, define a `ModelService` interface and a `ToolDispatcher` interface. Core calls those. Implementations live elsewhere.
3. **Always have an iteration budget.** hermes's default of 90 exists for a reason. Without it, a confused model will loop until your bill is enormous or your context window explodes.
4. **Sub-agents get their own budget and a restricted tool set.** hermes bans recursive delegation, user clarification, and shared memory writes. Claude Code isolates via worktrees. Pick your mechanism, but isolation is non-negotiable.
5. **Session Store belongs to Core, not Memory.** Conversation history (the raw message array) is Core's data. Long-term extracted knowledge is Memory's data. Mixing these up leads to circular dependencies.

## Step-by-Step: How to Build It

### Step 1: Define the conversation loop

Start with the simplest possible loop. No async, no generators, no hooks. Just a while loop.

```python
# pseudocode — start here, refine later
class AgentCore:
    def __init__(self, model_service, tool_dispatcher, context_manager, config):
        self.model = model_service          # peer service, not owned
        self.tools = tool_dispatcher        # peer service, not owned
        self.context = context_manager      # peer service, not owned
        self.config = config                # from Config Store
        self.messages = []                  # Session Store (in-memory first)
        self.max_iterations = config.get("max_iterations", 90)

    def run(self, user_message: str) -> str:
        self.messages.append({"role": "user", "content": user_message})
        iteration = 0

        while iteration < self.max_iterations:
            # 1. Build context (system prompt + memory injections)
            context = self.context.build(self.messages, self.config)

            # 2. Call model
            response = self.model.complete(context)

            # 3. No tool calls? Done.
            if not response.tool_calls:
                self.messages.append({"role": "assistant", "content": response.text})
                return response.text

            # 4. Execute tools
            for call in response.tool_calls:
                result = self.tools.execute(call.name, call.args)
                self.messages.append({"role": "tool", "content": result, "tool_call_id": call.id})

            # 5. Check context pressure
            if self.context.needs_compression(self.messages):
                self.messages = self.context.compress(self.messages)

            iteration += 1

        return "[Agent reached iteration limit]"
```

**Decision point**: Sync or async? If your tools are I/O-bound (HTTP calls, file reads), go async from the start. If they are mostly local computation, sync is simpler and debuggable.

### Step 2: Add session persistence

Replace the in-memory `self.messages` list with a Session Store. SQLite is the pragmatic default.

```python
class SessionStore:
    def load(self, session_id: str) -> list[Message]
    def append(self, session_id: str, message: Message) -> None
    def replace_all(self, session_id: str, messages: list[Message]) -> None  # for compression
```

**Decision point**: One SQLite file per session, or one database with a sessions table? Single database is simpler to manage. hermes uses one SQLite file for everything.

### Step 3: Define peer service interfaces

These are the contracts between Core and its spokes:

```python
class ModelService(ABC):
    def complete(self, messages: list[Message]) -> ModelResponse: ...

class ToolDispatcher(ABC):
    def execute(self, tool_name: str, args: dict) -> ToolResult: ...
    def list_tools(self) -> list[ToolDefinition]: ...

class ContextManager(ABC):
    def build(self, messages: list[Message], config: AgentConfig) -> list[Message]: ...
    def needs_compression(self, messages: list[Message]) -> bool: ...
    def compress(self, messages: list[Message]) -> list[Message]: ...
```

Core depends on these interfaces, never on concrete implementations. This is the line hermes crossed (embedding OpenAI SDK calls directly in `AIAgent`) and openclaw kept (hook-based indirection).

### Step 4: Add sub-agent delegation

The key insight from hermes and Claude Code: a sub-agent is just another `AgentCore` instance with constraints.

```python
def delegate(self, task: str, allowed_tools: list[str], max_depth: int = 2) -> str:
    if max_depth <= 0:
        return "[Cannot delegate further]"

    sub_tools = self.tools.create_restricted(allowed_tools)  # subset of parent's tools
    sub_core = AgentCore(
        model_service=self.model,       # share the model service
        tool_dispatcher=sub_tools,      # restricted tools
        context_manager=self.context,   # share compression logic
        config=self.config.child_config(max_iterations=30, max_depth=max_depth - 1)
    )
    result = sub_core.run(task)
    return summarize(result)  # parent only sees a summary, not the full conversation
```

**Decision point**: Share model service instance or create a new one? Sharing is simpler and avoids duplicate connections. But if you need separate token budgets per sub-agent, create separate instances.

### Step 5: Add interrupt handling

Users will press Ctrl+C. The model will hang. You need a way to break the loop.

```python
# Add to the while loop:
if self.abort_signal.is_set():
    self.messages.append({"role": "system", "content": "[Interrupted by user]"})
    break
```

Claude Code's approach: the outer loop (`REPL.tsx`) manages abort signals and can cancel the inner loop's async generator mid-yield. For a simpler implementation, a threading Event or asyncio Event works.

### Step 6 (optional): Add speculative execution

This is Claude Code's innovation. Only worth doing if your agent has a permission system.

```python
# Instead of:
#   ask permission -> execute tool
# Do:
#   start tool execution in background
#   ask permission in foreground
#   if approved: await the already-running execution
#   if denied: cancel and rollback

async def speculative_execute(self, tool_call):
    execution_task = asyncio.create_task(self.tools.execute(tool_call))
    approved = await self.permission_prompt(tool_call)
    if approved:
        return await execution_task
    else:
        execution_task.cancel()
        return ToolResult(status="denied", rollback=True)
```

## Common Pitfalls

**1. The God Class Trap (hermes)**
hermes's `AIAgent` started clean. Then someone added prompt building "just for now." Then model call logic "because it's easier." Then context compression "temporarily." 9431 lines later, nobody can change one thing without risking everything else. The fix is not discipline — it is interfaces. If `ModelService` is an interface, you physically cannot embed model logic in Core because Core does not import the SDK.

**2. The Bloated Outer Loop (Claude Code)**
`REPL.tsx` reached 5006 lines by handling UI rendering, session management, abort signals, speculative execution state, and permission prompts in one component. The inner loop (`query.ts`) stayed focused. The lesson: if you have dual-layer orchestration, guard the outer layer as strictly as the inner one.

**3. Missing Iteration Budget**
If you do not set a max iteration count, a model that misunderstands a task will loop until your context window is full or your API bill is alarming. hermes's default of 90 is generous — most tasks complete in under 20 iterations. Start with 30. Raise it when you have evidence you need more.

**4. Sub-agents Contaminating Parent Context**
Without isolation, a sub-agent's full conversation history gets appended to the parent's context. 10 sub-agent turns become 10 messages in the parent, destroying the parent's focus. hermes and Claude Code both enforce that the parent only receives a *summary* of the sub-agent's work.

**5. Session Store vs Memory Store Confusion**
Session Store holds the raw conversation (messages array). Memory Store holds extracted knowledge (facts, preferences, skills). If you put extracted knowledge back into the messages array, you double your context size. If you put raw messages into Memory Store, you pollute long-term knowledge with ephemeral chatter. Keep them separate. Core owns Session Store. Memory module owns Memory Store.

## Cross-Module Contracts

### What Agent Core expects from other modules

| Module | Contract | Format |
|--------|----------|--------|
| **Model Service** | `complete(messages) -> ModelResponse` | `ModelResponse` contains `.text`, `.tool_calls[]`, `.usage` (token counts) |
| **Tool Dispatcher** | `execute(name, args) -> ToolResult` | `ToolResult` contains `.content` (string), `.is_error` (bool) |
| **Tool Dispatcher** | `list_tools() -> ToolDefinition[]` | Used during context construction to build tool schemas for the model |
| **Context Manager** | `build(messages, config) -> messages` | Returns enriched message array with system prompt, memory injections |
| **Context Manager** | `needs_compression(messages) -> bool` | True when token count exceeds threshold (e.g., 80% of window) |
| **Context Manager** | `compress(messages) -> messages` | Returns shortened message array |
| **Memory** (as infra) | `get_injections(context) -> string[]` | Called by Context Manager during `build()`, not by Core directly |

### What Agent Core provides to other modules

| Consumer | What Core provides | Format |
|----------|-------------------|--------|
| **Model Service** | Assembled message array | Standard chat format: `{role, content, tool_calls?, tool_call_id?}` |
| **Tool Dispatcher** | Tool call instructions | `{name: string, args: dict, id: string}` from model response |
| **Context Manager** | Raw message history | The `messages` array from Session Store |
| **Sub-agents** | Restricted config | `AgentConfig` with lower `max_iterations`, reduced `max_depth`, tool whitelist |
| **Interface layer** | Conversation events | Final assistant response text, or streaming tokens/events (async generator pattern) |

### Invariants

- Core never calls LLM APIs directly. Always through `ModelService`.
- Core never executes system commands directly. Always through `ToolDispatcher`.
- Core never reads/writes Memory Store directly. Memory injections come through `ContextManager.build()`.
- Sub-agents never share their parent's Session Store. They get their own.
- The iteration budget is enforced by Core, not by Model or Tools. No other module can override it.
