# Emergent Behavior 5: Sub-Agent Delegation

> One-line: A parent Agent spawns an isolated child Agent with a limited budget, restricted tools, and no shared memory — then receives a single summary back. Divide and conquer without contaminating the parent's context.

## What Is It

The parent Agent encounters a task it decides to delegate. It calls a tool. That tool creates an entirely new Agent Core instance — with its own conversation loop, its own iteration budget, its own tool set. The child runs to completion. The parent gets back one summary message. The parent's context window grew by exactly one message, not by the 15-40 messages the child produced internally.

This is not function calling. It is not spawning a thread. It is creating a second Agent that thinks independently.

## How It Emerges

No single component "does" delegation. Three modules combine:

1. **Tools** — The parent's tool registry contains a delegation tool (`delegate_task`, `AgentTool`). From the LLM's perspective, this is just another tool call. It has no idea it is spawning a second intelligence.

2. **Agent Core** — The delegation tool's implementation creates a new Agent Core instance. This instance gets its own constructor params: a fresh conversation history, an independent iteration budget, a filtered tool list. It runs the exact same `while(budget > 0)` loop as the parent.

3. **Context Manager** — When the child completes, only its final summary is injected into the parent's context. The child's full conversation history — all the model calls, tool results, errors, retries — stays isolated. The parent never sees it.

The emergent part: the parent LLM never "learned" how delegation works. It was trained on text. It sees a tool called `delegate_task` with a description like "Delegate a subtask to a specialized sub-agent." It decides to call it based on prompt patterns. The tool does all the heavy lifting. Combined = hierarchical task decomposition from a flat tool call.

## Which Components Participate

| Component | Role in Delegation |
|-----------|-------------------|
| Agent Core (parent) | Runs its normal loop, calls delegate tool like any other tool |
| Tools (delegate tool) | Creates child Agent Core instance with restricted params |
| Agent Core (child) | Independent conversation loop with own budget |
| Model (child) | Child makes its own LLM calls — parent does not see them |
| Context Manager (parent) | Receives one summary message when child returns |

**What does NOT participate:** Memory. The child cannot write to the parent's Memory store. This is a hard constraint, not a suggestion.

## Case Study: hermes-agent's `delegate_tool`

hermes-agent implements the most conservative delegation model. Here is exactly how it works:

**Trigger**: The LLM returns a tool call to `delegate_task` with parameters `{task: "...", context: "..."}`.

**Construction**: `delegate_tool` creates a new `AIAgent` instance. Constructor params:
- `MAX_DEPTH = 2` — child can delegate once more; grandchild cannot
- Parallel limit: **3** sub-agents maximum running concurrently
- Tool blacklist: `clarify_request` (cannot ask user questions), `memory_write` (cannot persist to shared memory), `delegate_task` at depth 2 (cannot recurse further)
- Budget: independent iteration counter, separate from parent

**Execution**: The child runs its own `run_conversation()` loop. It calls the LLM, executes tools, iterates. The parent is blocked waiting for the tool result, just like it would wait for a file read or web request.

**Return**: When the child exhausts its budget or completes the task, it returns a text summary. This summary becomes the tool result for `delegate_task` in the parent's context.

**Isolation accounting**:
- Child produced 23 internal messages (model calls + tool results)
- Parent's context grew by 1 message (the summary)
- Parent's token budget was not touched by the child's LLM calls
- If the child wrote to Memory, the parent would not know — but it cannot, because `memory_write` is blacklisted

## Case Study: Claude Code's `AgentTool`

Claude Code takes a different approach. Where hermes is conservative (one mode, strict limits), Claude Code offers five delegation modes:

| Mode | Isolation Mechanism | Use Case |
|------|---------------------|----------|
| **worktree** | Git worktree — child gets a separate filesystem branch | File changes that might conflict with parent's work |
| **fork** | Child process — OS-level isolation | CPU-intensive tasks, crash isolation |
| **team** | Multiple agents coordinating on related subtasks | Parallel exploration of different approaches |
| **swarm** | Many agents working on a large task simultaneously | Large-scale refactoring, mass file changes |
| **coordinator** | One agent orchestrating multiple workers | Complex multi-step workflows |

The worktree mode is notable. When the child Agent modifies files, it does so in a separate Git worktree. If the parent decides the child's work was wrong, the worktree is deleted. No files were harmed in the parent's working directory. This is filesystem-level rollback for free.

All modes share the same core contract: child gets independent budget, restricted tools, and returns a summary. The modes differ in what kind of isolation the child gets beyond context isolation.

## Case Study: openclaw's Infrastructure Support

openclaw does not have a first-class delegation command like hermes or Claude Code. What it has is infrastructure that makes delegation possible:

- **Lane queuing**: Each session runs in its own Lane. A sub-agent can be given its own Lane, which means its messages queue independently from the parent.
- **Hook system**: Discord's `subagent_spawning` event fires when a sub-agent is created. Other modules can subscribe — logging, rate limiting, quota tracking all hook in without touching the delegation logic.

This is lighter than hermes or Claude Code. The building blocks exist; the orchestration is less opinionated.

## How to Make It Emerge in Your Agent

### Step 1: Build a delegate tool

Create a tool with this signature:

```
Tool: delegate_task
Input: { task: string, context: string }
Output: { summary: string, status: "completed" | "failed" }
```

Register it in your parent Agent's tool list like any other tool.

### Step 2: The tool creates a child Agent Core

Inside the tool's execute function:

```python
def execute(params):
    child = AgentCore(
        budget=30,           # independent budget, not the parent's
        tools=CHILD_TOOLS,   # filtered tool list
        memory=None,         # no shared memory access
        depth=parent.depth + 1
    )
    result = child.run(params.task, params.context)
    return { "summary": result.summary, "status": result.status }
```

### Step 3: Set constraints

Three hard rules for the child:
1. **No recursive delegation** beyond MAX_DEPTH. hermes uses 2. Start there.
2. **No user interaction**. The child cannot ask clarifying questions. It works with what it has.
3. **No shared Memory writes**. The child's conclusions go into its summary, not into the parent's long-term Memory store.

### Step 4: Wire the return path

The child's summary becomes the tool result. The parent's context manager treats it like any other tool result — one message, bounded length.

If the child fails, the tool result says `status: "failed"` with a reason. The parent decides what to do next. Retry, try differently, or give up.

### Step 5: Add concurrency (optional)

If the parent calls `delegate_task` three times in one turn, you need a concurrency limit. hermes caps at 3 parallel sub-agents. Start conservative. The LLM's tendency to over-delegate is real.

## Cross-Module Contracts

These are the interfaces you must define before delegation will work:

### 1. Child Agent Constructor Params

```
{
  task: string,          // what to do
  context: string,       // relevant background from parent
  budget: number,        // max iterations (independent of parent)
  depth: number,         // current delegation depth
  tools: Tool[],         // restricted tool set
  memory_access: "none" | "read-only"  // never "read-write"
}
```

### 2. Tool Blacklist

Minimum blacklist for child agents:
- `delegate_task` (at max depth)
- `clarify_request` / `ask_user` (no user interaction)
- `memory_write` / `memory_update` (no shared persistence)

Optional blacklist based on risk:
- `shell_execute` (if parent handles all shell operations)
- `file_delete` (destructive operations stay with parent)

### 3. Summary Return Format

```
{
  summary: string,      // max 2000 tokens — enforced, not suggested
  status: "completed" | "failed" | "budget_exhausted",
  artifacts: string[],  // file paths, URLs, or other references created
  tool_calls_made: number  // for debugging and cost tracking
}
```

### 4. Depth Limit Enforcement

```
MAX_DEPTH = 2         // parent → child → grandchild. No further.
                      // Enforced in child constructor, not in the LLM prompt.
                      // Prompt-level enforcement is a suggestion.
                      // Constructor-level enforcement is a wall.
```

## How to Test It (Loop Test)

Delegation is cross-module behavior. Unit tests on the delegate tool alone miss the point. You need loop tests that exercise the full chain.

### Test 1: Child gets independent budget

```
1. Parent has budget=50
2. Parent calls delegate_task
3. Child runs with budget=30
4. Child exhausts budget after 30 iterations
5. Assert: parent's remaining budget is still 49 (one iteration spent on the delegate call)
6. Assert: child's budget is 0
```

### Test 2: Tool restriction enforced

```
1. Create child with tool blacklist = [clarify_request, memory_write]
2. Child's LLM returns a tool call to clarify_request
3. Assert: tool dispatch rejects the call
4. Assert: child receives an error message, not a crash
5. Assert: child continues its loop (handles the error gracefully)
```

### Test 3: Summary isolation

```
1. Child runs, produces 23 internal messages
2. Child completes, returns summary
3. Assert: parent's conversation history grew by exactly 2 messages
   (the delegate_task call + the tool result)
4. Assert: parent has zero access to child's 23 internal messages
```

### Test 4: Depth limit holds

```
1. Parent (depth=0) delegates to child (depth=1)
2. Child (depth=1) delegates to grandchild (depth=2)
3. Grandchild's tool list does NOT contain delegate_task
4. Assert: grandchild cannot delegate
5. Assert: this is enforced at constructor level, not prompt level
```

### Test 5: Concurrency limit

```
1. Parent calls delegate_task 5 times in parallel
2. MAX_PARALLEL = 3
3. Assert: 3 children start immediately
4. Assert: 2 children queue until a slot opens
5. Assert: all 5 eventually complete
```

## Variations Across Projects

| Dimension | hermes-agent | Claude Code | openclaw |
|-----------|-------------|-------------|----------|
| **Delegation trigger** | `delegate_tool` in tool registry | `AgentTool` with mode selection | Lane queue + hook events |
| **Max depth** | 2 | Configurable per mode | Not explicitly limited |
| **Max parallel** | 3 | Varies by mode (team can be many) | Lane-based concurrency |
| **Filesystem isolation** | None (shared filesystem) | Git worktree (full isolation) | None |
| **Process isolation** | Same process, new instance | Fork mode = child process | Same process |
| **Memory policy** | Blacklisted — no writes | Mode-dependent | Not specified |
| **User interaction** | Blacklisted — no clarify | No user-facing I/O in child | Not specified |
| **Return format** | Text summary in tool result | Structured summary with artifacts | Event-based via hooks |
| **Maturity** | Production-tested, conservative | Most feature-rich, enterprise-scale | Infrastructure-ready, lighter usage |
| **NemoClaw** | N/A — not an Agent | — | — |

**The key difference**: hermes says "delegation is one thing, do it safely." Claude Code says "delegation is five things, pick the right isolation model." openclaw says "here are the building blocks, wire it yourself."

For your first implementation, start with hermes's model. One mode, strict limits, proven safe. Add Claude Code's variations when you have production data showing which isolation modes you actually need.
