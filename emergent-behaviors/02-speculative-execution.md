# Emergent Behavior 2: Speculative Execution

> One-line: the Agent runs a tool and asks for permission at the same time, showing the completed result instantly when the user approves -- no component was designed as a "speed optimizer."

## What Is It

Speculative execution is a parallelism trick borrowed from CPU design. When the LLM returns a tool call that requires user permission, the Agent does two things simultaneously:

1. Shows the user a permission prompt ("Allow file edit?").
2. Starts executing the tool in the background.

If the user approves, the result is already done. It appears instantly. If the user rejects, the result is discarded and the system rolls back to the last `CompletionBoundary`.

Why it is "emergent": there is no `SpeedOptimizer` class. There is no `ParallelExecutionEngine`. The behavior arises from four independent mechanisms interacting: the async permission flow in the UI, the pre-execution logic in the tool runner, the parallel scheduling in Agent Core, and the `SpeculationState` tracker that knows when to commit or discard. Each was built for its own purpose. Together they produce a perceived 2-3x speedup on permission-gated operations.

Claude Code is the only project that implements this. No other Agent in the comparison set attempts it.

## How It Emerges

```
LLM returns tool_call
  │
  │  Agent Core receives the tool_call
  │  Checks: does this tool require permission?
  │
  ├──────────────────────────────────────┐
  │ (parallel)                           │ (parallel)
  │                                      │
  ▼                                      ▼
Interface/UI                          Tools
  │                                      │
  │  renders permission prompt           │  starts executing the tool
  │  "Allow: edit file X?"               │  (write to disk, run command, etc.)
  │  waits for user input                │  stores result in SpeculationState
  │                                      │  marks result as "speculative"
  │                                      │
  ▼                                      ▼
User responds                        Execution completes
  │                                      │
  ├── APPROVE ──→ commit speculative ────┘
  │               result. Show to user.
  │               Latency = 0 (already done).
  │
  └── REJECT ───→ discard speculative result.
                  Roll back to CompletionBoundary.
                  SpeculationState → cleared.
                  Tool effect is undone.
```

The key insight: the permission prompt and the tool execution have zero dependencies on each other. The permission prompt needs the tool description. The tool execution needs the tool parameters. Both are available from the original `tool_call`. So they can run in parallel.

## Which Components Participate

| Component | Role in Speculative Execution |
|-----------|-------------------------------|
| **Agent Core** | Parallel scheduler. Receives `tool_call` from LLM. Dispatches permission request and tool execution simultaneously. Reads `SpeculationState` to decide: commit or discard. |
| **Tools** | Pre-executor. Runs the tool immediately without waiting for permission. Tags the result as speculative. Must support result discarding -- the execution may be thrown away. |
| **Interface/UI** | Permission gate. Renders the approval prompt asynchronously. Returns approve/reject. Must be non-blocking -- it cannot lock the event loop while waiting for the user. |
| **SpeculationState** | Rollback tracker. Records: which tool was speculatively executed, what result it produced, where the `CompletionBoundary` is (the safe rollback point). Not a component in its own right -- it is a data structure shared between Agent Core and Tools. |

## Case Study: Claude Code

Claude Code is where this behavior was discovered. It is deeply embedded in the tool execution pipeline.

### The permission model

Claude Code has two permission modes:

- **ask mode**: every tool call that modifies state requires user approval (file writes, shell commands, etc.). Read-only tools skip permission.
- **auto mode**: pre-approved tool patterns skip permission entirely.

Speculative execution only matters in **ask mode**. In auto mode, there is no permission prompt, so no parallelism opportunity.

### Parallel dispatch in Agent Core

In `query.ts`, when the LLM returns a tool call:

1. Agent Core checks if the tool requires permission (based on tool metadata and permission config).
2. If permission is required, it creates two concurrent tasks:
   - Task A: send permission request to the UI layer (`REPL.tsx`).
   - Task B: invoke the tool runner with the tool parameters.
3. Both tasks start at the same tick. No `await` on Task A before starting Task B.

### SpeculationState tracking

The `SpeculationState` object tracks:

- `toolCallId`: which tool call is speculative.
- `result`: the tool's return value (held in memory, not yet committed to conversation).
- `boundary`: a `CompletionBoundary` marker -- the last known-good state of the conversation. If speculation fails, the conversation rewinds to this point.

### Commit path (user approves)

When the user types "y" or the approval times out to auto-approve:

1. Agent Core reads `SpeculationState.result`.
2. Appends the result to the conversation as a normal `ToolResult`.
3. Clears `SpeculationState`.
4. Continues the conversation loop.

From the user's perspective: they press "y" and the result appears instantly. No waiting for execution.

### Discard path (user rejects)

When the user types "n" or presses Escape:

1. Agent Core reads `SpeculationState.boundary`.
2. Rewinds the conversation to that `CompletionBoundary`.
3. Discards `SpeculationState.result`.
4. Inserts a rejection message into the conversation: "User denied permission for [tool]."
5. Continues the loop. The LLM sees the rejection and adjusts.

For file writes: the file was already written during speculation. On rejection, the file is restored to its pre-execution state (Claude Code tracks the original content).

For shell commands: output was captured but never displayed. On rejection, it is dropped. Side effects (e.g., a process started by the command) must be handled -- this is the hard part.

### Why it feels fast

In ask mode without speculation, the timeline is:

```
LLM returns tool_call ─→ show permission ─→ user approves ─→ execute tool ─→ show result
                          [user thinking]                     [execution time]
Total wait = user_time + execution_time
```

With speculation:

```
LLM returns tool_call ─→ show permission ─────→ user approves ─→ show result
                     └──→ execute tool (parallel) ─→ done (waiting)
Total wait = max(user_time, execution_time)
                         ≈ user_time (execution is usually faster)
```

For a file edit that takes 200ms to execute, and a user who takes 1.5 seconds to press "y": without speculation, total = 1.7s. With speculation, total = 1.5s. The 200ms is hidden.

For a shell command that takes 5 seconds: without speculation, total = 6.5s. With speculation, total = 5s. The user's thinking time is hidden inside the execution time.

## How to Make It Emerge in Your Agent

### Step 1: Make the permission flow async

The permission request must be non-blocking. It sends a request to the UI and returns a Promise/Future. It does not block the main loop.

```
function requestPermission(toolCall): Promise<"approve" | "reject">
```

If your permission flow is synchronous (blocking until the user responds), speculation is impossible. This is the first prerequisite.

### Step 2: Make tool execution return a discardable result

The tool runner must support a speculative mode:

```
function executeSpeculative(toolCall): { result: ToolResult, rollback: () => void }
```

The `rollback` function undoes the tool's side effects. For file writes: restore original content. For shell commands: kill the process and discard output. For read-only tools: no-op (nothing to roll back).

Not all tools can be speculated. A tool that sends an email cannot be rolled back. Mark these as `speculationSafe: false` in tool metadata.

### Step 3: Add SpeculationState to Agent Core

```
interface SpeculationState {
  toolCallId: string
  result: ToolResult | null    // null = still executing
  rollback: (() => void) | null
  boundary: CompletionBoundary // conversation state before speculation
}
```

Agent Core creates this state when it dispatches parallel tasks. It reads it when the permission resolves.

### Step 4: Wire the parallel dispatch

In your main loop, replace:

```
// Sequential (slow)
const approved = await requestPermission(toolCall)
if (approved) {
  const result = await executeTool(toolCall)
  appendToConversation(result)
}
```

With:

```
// Parallel (fast)
const boundary = captureConversationState()
const [permissionResult, specResult] = await Promise.all([
  requestPermission(toolCall),
  executeSpeculative(toolCall)
])

if (permissionResult === "approve") {
  appendToConversation(specResult.result)
} else {
  specResult.rollback()
  rewindConversation(boundary)
  appendRejectionMessage(toolCall)
}
```

### Step 5: Handle edge cases

- **Tool execution fails.** Show the permission prompt normally. If approved, show the error. If rejected, discard both the error and the rejection.
- **User responds before execution finishes.** If approved, wait for execution to complete, then show result. If rejected, cancel execution (if possible) and roll back.
- **Multiple tool calls in one LLM response.** Speculate only on the first permission-gated call. Execute the rest sequentially after it resolves. Speculating on multiple calls simultaneously creates complex rollback chains.

## Cross-Module Contracts

### Contract 1: ToolResult Supports Rollback State

```
interface ToolResult {
  toolCallId: string
  output: string | object
  isSpeculative: boolean          // true = not yet committed
  rollback: (() => void) | null   // null for read-only tools
}
```

Tools must produce results that can be discarded. If a tool writes directly to the conversation stream (e.g., streaming output), speculation breaks -- you cannot un-stream output.

### Contract 2: Permission Request Is Async-Cancelable

```
interface PermissionRequest {
  prompt(): Promise<"approve" | "reject">
  cancel(): void    // called when speculation needs to abort
}
```

The permission prompt must support cancellation. If the tool execution fails catastrophically before the user responds, Agent Core cancels the permission request and shows an error instead.

### Contract 3: CompletionBoundary Format

```
interface CompletionBoundary {
  messageIndex: number           // index in conversation history
  fileSnapshots: Map<string, string>  // path → content before speculation
  processIds: number[]           // PIDs spawned during speculation
}
```

This defines the rollback scope. `messageIndex` tells the conversation where to rewind. `fileSnapshots` tells the file system what to restore. `processIds` tells the OS what to kill.

The boundary must be captured *before* speculation starts. If captured after, it includes speculative changes and the rollback is corrupt.

## How to Test It (Loop Test)

```
Test: speculative-execution-full-cycle
Precondition: agent is in "ask" permission mode. A test file exists.

Step 1 — Verify parallel start
  Send a message that triggers a file write tool call.
  Instrument: timestamp when permission prompt appears (T_perm).
  Instrument: timestamp when tool execution starts (T_exec).
  Assert: |T_perm - T_exec| < 50ms.
  (Both must start within the same event loop tick.)

Step 2 — Verify approve path
  Mock user approval after 500ms delay.
  Instrument: timestamp when result is shown to user (T_show).
  Instrument: timestamp when tool execution completed (T_done).
  Assert: T_show - T_done < 100ms.
  (Result was already ready when approval came.)
  Assert: file on disk contains the expected edit.

Step 3 — Verify reject path
  Send another message that triggers a file write.
  Mock user rejection after 500ms delay.
  Assert: file on disk is unchanged (rolled back to original).
  Assert: conversation does not contain the speculative tool result.
  Assert: conversation contains a rejection message.
  Assert: SpeculationState is cleared.

Step 4 — Verify boundary integrity
  Send a message that triggers two sequential tool calls.
  First call: approve (speculative).
  Second call: reject.
  Assert: first call's result is committed in conversation.
  Assert: second call's result is discarded.
  Assert: file system reflects only the first call's changes.

Pass criteria: all 4 steps pass.
```

Runtime: ~5 seconds with mocked timers. ~15 seconds with real user simulation.

## Variations Across Projects

| Project | Speculation Support | Why / Why Not |
|---------|-------------------|---------------|
| **Claude Code** | Full implementation. SpeculationState tracking, CompletionBoundary rollback, parallel permission + execution. Only in "ask" mode. | Anthropic optimized for perceived speed in the CLI. The ask-mode UX would feel sluggish without it. Investment in this pattern is high. |
| **hermes-agent** | None. Strict sequential execution. LLM returns tool call, Agent Core waits for approval (if required), then executes. | hermes runs on many platforms (Telegram, Discord, etc.) where the "permission prompt" is a message in a chat thread. Parallelism across a network round-trip to a chat platform is impractical. |
| **openclaw** | None. Standard request-response. Tool calls execute only after full processing of the LLM response. | openclaw's architecture is plugin-based. Tool execution goes through multiple layers (plugin SDK, hook system, lane queuing). Adding speculation would require changes across all layers. |
| **NemoClaw** | Not applicable. NemoClaw is a security runtime that wraps another agent. It does not execute tools directly. | NemoClaw's contribution is the security envelope, not the execution pipeline. |

### Why only Claude Code does this

Three conditions make speculation viable in Claude Code and impractical elsewhere:

1. **Single-user CLI.** The permission prompt is local -- a keystroke in the terminal. Round-trip time is milliseconds. On Telegram, the round-trip is seconds. Speculation savings are negligible when the network dominates.

2. **File-centric tools.** Most Claude Code tools are file reads/writes. File operations are fast (< 200ms) and reversible (save original, restore on rollback). Shell commands are harder but still manageable (capture output, kill process). Tools that send HTTP requests or modify databases are not safely speculatable.

3. **Tight control of the execution pipeline.** Claude Code is a monolith. Agent Core, Tools, and UI are in the same process. There is no plugin boundary, no RPC, no serialization. Dispatching two tasks in parallel is a `Promise.all()`. In openclaw, the same operation would cross plugin boundaries, hook chains, and potentially network calls.

### If you want speculation but your agent is not a CLI

Focus on the subset of tools where speculation is safe:

- File reads: always safe (no side effects).
- File writes: safe if you snapshot before writing.
- Code formatting/linting: safe (deterministic, reversible).
- Shell commands that only produce output: safe-ish (no side effects beyond the process).

Mark everything else as `speculationSafe: false` and execute it sequentially. Even partial speculation (50% of tool calls) produces a noticeable speedup.
