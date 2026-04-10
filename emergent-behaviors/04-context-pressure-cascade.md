# Emergent Behavior 4: Context Pressure Cascade

> The system responds to context window pressure with escalating compression strategies and cooperative LLM behavior change, even though no component was designed to orchestrate graceful degradation.

## What Is It

Every LLM agent has a finite context window. Fill it up and the agent breaks. The context pressure cascade is what happens when three independent mechanisms -- multi-level compression, budget warning injection, and LLM behavioral adaptation -- combine to create a system that gracefully handles pressure instead of crashing.

Think of a spring being compressed. Light pressure: gentle response. More pressure: stronger response. Maximum pressure: emergency response. The system escalates automatically across levels. No orchestrator coordinates it.

Every production agent needs this. It is not optional.

## How It Emerges

Three components act independently:

1. **Context Manager** monitors token count against thresholds. When exceeded, it applies compression strategies in escalating order. It does not know about budget warnings or LLM behavior.
2. **Agent Core** injects a `budget_warning` string into tool results when the context is running hot. It does not know about compression strategies or LLM adaptation.
3. **The LLM** reads the budget warning and changes its behavior -- shorter responses, fewer tool calls, faster wrap-up. It was not programmed to do this. It just reads the warning and adapts.

No one designed "cooperative degradation." It emerged from:

```
Context Manager detects tokens over threshold
  -> L1: prune old tool results (zero LLM calls, cheap)
    -> still over? L2: LLM summarize middle turns (expensive)
      -> still over? L3: truncate earliest messages (last resort)

Meanwhile, in parallel:
  Agent Core injects budget_warning into tool result
    -> LLM reads it
      -> LLM accelerates wrap-up (cooperative degradation)
```

Context Manager compresses. Agent Core warns. LLM cooperates. Combined: the system gracefully handles pressure without any component understanding the full picture.

## Which Components Participate

| Component | Role | Awareness of the Whole |
|---|---|---|
| Context Manager | Multi-level compression (L1/L2/L3) | Knows token counts and thresholds. Does not know about budget warnings or LLM adaptation. |
| Agent Core | Injects `budget_warning` into tool results | Knows the context is running hot. Does not know which compression level is active. |
| Model (LLM) | Reads warning, changes behavior | Knows nothing. Just follows instructions in the warning text. |

This behavior crosses two module boundaries: Context Manager <-> Agent Core <-> Model. That makes it harder to implement correctly than self-healing routing (which stays within one module).

## Case Study: hermes-agent

hermes uses a 50% threshold in `context_compressor.py`. Here is what happens when context fills up during a long coding session:

**Turn 1-40:** Normal operation. Context at 35% of window. No compression.

**Turn 41:** Context hits 50%. L1 triggers.
- Old tool results (shell output, file contents from turns 1-20) are pruned. Replaced with one-line summaries: `[tool result pruned: 847 tokens -> 12 tokens]`.
- Zero LLM calls. Fast. Cheap.
- Context drops to 38%.

**Turn 42-65:** Context grows again. L1 prunes tool results from turns 20-40. Context at 48%.

**Turn 66:** Context hits 50% again, but there are no more tool results to prune. L2 triggers.
- LLM is called to summarize turns 10-50 into a structured summary.
- Original messages replaced with the summary. Expensive (one LLM call) but effective.
- Context drops to 30%.

**Turn 67:** Agent Core checks context ratio. Still above 40%. Injects into the next tool result:
```
[budget_warning: context at 82% after compression. Please wrap up current task.]
```

**Turn 68:** LLM reads the budget warning. Instead of calling 3 more tools, it writes a final summary and asks the user if they want to continue in a new session.

**Turn 90+:** If the user keeps going and context hits 95% even after L2, L3 triggers: hard truncation of the earliest messages. This is the last resort. Information is lost. But the agent does not crash.

The cooperative part -- the LLM reading `budget_warning` and changing behavior -- was never explicitly programmed. hermes just injects a string. The LLM interprets it. This works because LLMs are instruction-followers by nature.

## How to Make It Emerge in Your Agent

You need three pieces. Build them separately.

### Piece 1: Multi-Level Compression (Context Manager)

```typescript
interface CompressionResult {
  messages: Message[];
  tokensBefore: number;
  tokensAfter: number;
  levelApplied: 1 | 2 | 3;
}

async function compressContext(
  messages: Message[],
  maxTokens: number,
  currentTokens: number
): Promise<CompressionResult> {

  // L1: Prune old tool results (cheap, no LLM calls)
  if (currentTokens > maxTokens * 0.80) {
    const pruned = pruneToolResults(messages);
    if (estimateTokens(pruned) <= maxTokens * 0.80) {
      return { messages: pruned, levelApplied: 1, ... };
    }
  }

  // L2: LLM summarize middle turns (expensive, one LLM call)
  if (currentTokens > maxTokens * 0.80) {
    const summarized = await llmSummarize(messages);
    if (estimateTokens(summarized) <= maxTokens * 0.80) {
      return { messages: summarized, levelApplied: 2, ... };
    }
  }

  // L3: Truncate earliest messages (last resort, information loss)
  const truncated = truncateEarliest(messages, maxTokens * 0.70);
  return { messages: truncated, levelApplied: 3, ... };
}
```

**L1 details -- pruning tool results:**
- Target: tool result messages older than N turns.
- Replace content with: `[pruned: {originalTokens} tokens, was: {toolName} call]`.
- Preserve the message structure (role, tool_call_id). Only replace content.
- Cost: zero LLM calls. Just string replacement.

**L2 details -- LLM summarization:**
- Select a range of middle messages (not the most recent, not the system prompt).
- Send them to the LLM with: "Summarize these conversation turns. Preserve all decisions, file paths, error messages, and pending tasks."
- Replace the range with one assistant message containing the summary.
- Cost: one LLM call. Typically 500-2000 output tokens.

**L3 details -- truncation:**
- Remove messages from the front (oldest first).
- Always preserve: system prompt, most recent 5 turns, any user message with an unresolved question.
- Cost: zero. But information is permanently lost from the context.

### Piece 2: Budget Warning Injection (Agent Core)

```typescript
function maybeInjectBudgetWarning(
  toolResult: ToolResult,
  contextRatio: number
): ToolResult {
  if (contextRatio > 0.75) {
    const warning = `[budget_warning: context window is ${Math.round(contextRatio * 100)}% full. `
      + `Finish the current task concisely. Avoid unnecessary tool calls.]`;

    return {
      ...toolResult,
      content: toolResult.content + "\n\n" + warning
    };
  }
  return toolResult;
}
```

**Where to inject:** Append to the tool result content, not as a separate message. The LLM sees it as part of the tool output. This is more effective than a system-level instruction because it arrives in real-time context, not as a static rule.

**Threshold:** 75% is a reasonable starting point. hermes uses context_ratio checks at multiple levels. Claude Code uses 80% for auto-compact.

### Piece 3: Let the LLM Cooperate (No Code Needed)

This piece requires zero implementation. The LLM reads the `budget_warning` text and adapts. It writes shorter responses. It calls fewer tools. It wraps up.

This works because:
- The warning is in natural language. LLMs follow natural language instructions.
- The warning is positioned after a tool result, which is high-attention context for the LLM.
- The warning is specific: it says the percentage, not just "be careful."

You do not need to fine-tune the model. You do not need special tokens. You just need a clear, specific warning string placed where the LLM will read it.

### Assembly

Wire these into your agent loop:

```
before each LLM call:
  currentTokens = estimateTokens(messages)
  if currentTokens > threshold:
    messages = compressContext(messages, maxTokens, currentTokens)

after each tool execution:
  contextRatio = estimateTokens(messages) / maxTokens
  toolResult = maybeInjectBudgetWarning(toolResult, contextRatio)
```

The cascade emerges. No orchestrator. Context Manager compresses when needed. Agent Core warns when needed. LLM adapts when it reads the warning. Three independent mechanisms creating cooperative degradation.

## Cross-Module Contracts

This behavior crosses two module boundaries. Three contracts must be maintained.

| Contract | Format | Between |
|---|---|---|
| Compression input/output | `Message[]` array in, `Message[]` array out. Same schema. Compressed messages must be valid for the LLM API. | Context Manager <-> Agent Core |
| Budget warning injection | String appended to `toolResult.content`. Format: `[budget_warning: ...]`. Must not break tool result parsing. | Agent Core <-> Model (via tool result) |
| Token estimation interface | `estimateTokens(messages: Message[]): number`. Must be callable by both Agent Core (for warning threshold) and Context Manager (for compression threshold). | Shared utility consumed by both |

**Danger zone:** If you change the `Message[]` schema (e.g., add a new field, change how tool results are structured), compression may produce invalid messages. The LLM call will fail, and the cascade breaks. Always run the loop test after schema changes.

## How to Test It (Loop Test)

File: `tests/loops/context-cascade.test.ts`

```
Test: "Context pressure cascade escalates through all levels"

Setup:
  - Configure context window: 4096 tokens (small, for fast testing)
  - L1 threshold: 80% (3277 tokens)
  - Budget warning threshold: 75% (3072 tokens)

Phase 1 — L1 Prune:
  1. Fill context with 20 turns containing large tool results (total ~3500 tokens)
  2. Trigger compression check
  3. Assert: L1 applied (tool results pruned)
  4. Assert: token count dropped below 3277
  5. Assert: message structure still valid (roles, tool_call_ids preserved)
  6. Assert: zero LLM calls made during compression

Phase 2 — L2 Summarize:
  7. Continue filling context (20 more turns, minimal tool results)
  8. Trigger compression check
  9. Assert: L1 attempted but insufficient (few tool results left to prune)
  10. Assert: L2 applied (LLM summarization called)
  11. Assert: summary message contains key facts from original turns
  12. Assert: token count dropped below 3277

Phase 3 — Budget Warning:
  13. After L2, check context ratio
  14. Execute a tool call
  15. Assert: budget_warning string present in tool result content
  16. Assert: warning contains current percentage number
  17. Assert: tool result is still valid (warning appended, not replacing)

Phase 4 — L3 Truncation:
  18. Force context to 95% (bypass L1 and L2)
  19. Trigger compression check
  20. Assert: L3 applied (earliest messages removed)
  21. Assert: system prompt preserved
  22. Assert: most recent 5 turns preserved
  23. Assert: token count below threshold

Phase 5 — Cooperative degradation (optional, harder to test):
  24. Mock LLM to return tool_call responses
  25. Inject budget_warning at 85%
  26. Assert: LLM's next response is shorter / uses fewer tool calls
  (This tests LLM behavior, which is non-deterministic.
   Use statistical assertion: over 10 runs, average response
   length with warning < average without warning.)
```

## Variations Across Projects

### hermes-agent

- **Threshold:** 50% of context window.
- **L1:** Prune tool results from older turns.
- **L2:** LLM structured summarization. Uses iterative summary updates -- each new summary builds on the previous one rather than re-summarizing from scratch.
- **Budget warning:** Injected into tool results based on `context_ratio`.
- **Strength:** Iterative summaries preserve context quality over long sessions.
- **Weakness:** Single 50% threshold. No distinction between "getting full" and "critically full."

### openclaw

- **Architecture:** Pluggable context engine via `registerContextEngine()`.
- **Two modes:** Active compression (triggers before window fills) and reactive compression (triggers on backpressure).
- **Special feature:** Transcript rewrite -- surgically edits specific messages rather than summarizing a range. More precise than hermes's range-based approach.
- **Strength:** Active mode prevents the window from ever getting critically full. Proactive, not reactive.
- **Weakness:** Plugin architecture adds complexity. You need to implement the context engine interface.

### Claude Code

- **Scale:** 12 files in `services/compact/`.
- **Three strategies running in parallel:**
  - `auto`: Triggers at 80% threshold. Standard compression.
  - `reactive`: Triggers on backpressure (API returns context-too-long error). Emergency mode.
  - `snip`: Fine-grained per-message compression. Can compress individual tool results without touching others.
- **Special feature:** `microCompact` preprocessing -- lightweight token reduction applied to every message before it enters the context. Strips whitespace, shortens common patterns.
- **Strength:** Most diverse strategy set. Three levels of granularity. microCompact reduces baseline pressure.
- **Weakness:** 12 files for context management is significant complexity. The three strategies can interact in unexpected ways.

### Summary Table

| Aspect | hermes | openclaw | Claude Code |
|---|---|---|---|
| Trigger threshold | 50% | Configurable (active + reactive) | 80% (auto) / backpressure (reactive) |
| L1 (cheap) | Prune tool results | Tool result truncation | snip (fine-grained) |
| L2 (expensive) | LLM summarize range | Transcript rewrite | auto compact (LLM summarize) |
| L3 (emergency) | Truncate earliest | Not documented | reactive (on API error) |
| Preprocessing | None | None | microCompact |
| Proactive mode | No | Yes (active compression) | No (all reactive) |
| Budget warning | Yes (tool result injection) | Not documented | Not documented |
| Cooperative degradation | Yes | Partial | Partial |
| Files/complexity | 1 file | Plugin interface + implementations | 12 files |

The simplest path: start with hermes's approach (1 file, 3 levels, budget warning injection). Add openclaw's proactive mode later when you understand your agent's context consumption patterns. Consider Claude Code's micro-compact preprocessing if you need to squeeze more into the window before compression kicks in.
