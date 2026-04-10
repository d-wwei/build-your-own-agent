# Module 5: Context Manager

> A stateless service that keeps your Agent coherent when the conversation gets long.

---

## What Is It

Context Manager is one of Agent Core's four peer services in the Hub-and-Spoke architecture. Its job: make the most of a finite context window.

It operates on the in-memory messages array. It stores nothing. No database, no files, no persistent state. Give it a messages array, get back a smaller messages array. That's the entire contract.

In the reference architecture:

```
╔═══════════════════════════════════════════╗
║ AGENT CORE                                ║
║                                           ║
║   Model    Tools    Memory    Context Mgr ║
║   (peer)   (peer)   (peer)     (peer)     ║
║                                           ║
║   ↕ no      ↕ Skill  ↕ Memory  ↕ no      ║
║    storage    Store    Store     storage   ║
╚═══════════════════════════════════════════╝
```

Context Manager and Model are the two stateless peers. Both take input, return output, own zero data.

---

## What Problem Does It Solve

Three things compound against you in any long-running Agent conversation:

1. **Context window is finite.** Claude's 200K tokens sounds like a lot until your Agent runs 30 tool calls. Each tool result might be 2K-10K tokens. That's 60K-300K tokens from tool results alone.

2. **Tool results accumulate fast.** A file read returns the full file. A web search returns multiple pages of results. A shell command dumps its entire stdout. After a dozen tool calls, tool results dominate the context window — pushing out the original conversation that actually matters.

3. **Long conversations lose coherence.** When the context fills up and critical early messages get displaced, the LLM starts contradicting its own earlier statements, forgets what it already tried, and repeats failed approaches.

Without Context Manager, your Agent hits the token limit and either crashes (hard failure) or silently drops early messages and produces incoherent output (soft failure). Neither is acceptable.

---

## How the 4 Projects Do It

### hermes-agent: `context_compressor.py`

hermes takes the straightforward two-level approach.

**Trigger:** Token usage crosses the 50% threshold of the model's context window.

**Level 1 — Prune old tool results (zero LLM calls):**
Strip tool results from older turns. The rationale: tool results are large, mostly stale (once the LLM has acted on them, the raw output rarely matters), and removing them is free — no summarization call needed.

Critical detail: hermes preserves the latest tool results untouched. Only older turns get pruned. This prevents losing data the LLM is actively working with.

**Level 2 — LLM summarize middle turns:**
If still over threshold after pruning, send the middle portion of the conversation to the LLM with a "summarize these turns" instruction. The first few turns (task context) and last few turns (recent work) are preserved intact. Only the middle gets compressed.

Hermes iteratively updates existing summaries rather than creating fresh ones each time. If a summary already exists from a previous compression pass, the new pass merges new information into the existing summary instead of replacing it. This prevents information drift across multiple compression cycles.

**Cost profile:** Cheap. L1 is free. L2 costs one LLM call per compression event. For most conversations, L1 alone is sufficient.

### openclaw: `src/context-engine/`

openclaw takes the extensible approach.

**Architecture:** Pluggable via `registerContextEngine()`. You can swap the entire compression strategy without touching Agent Core. This follows openclaw's philosophy — everything is a plugin.

**Two compression modes:**

| Mode | Trigger | What It Does |
|------|---------|-------------|
| Active (proactive) | Before the window fills | Starts compressing early to avoid sudden quality drops |
| Reactive | After hitting limits | Emergency compression when active mode wasn't enough |

The active mode is the key differentiator. Most implementations wait until the window is nearly full, then scramble to compress. openclaw starts early, compressing gradually, so quality degrades smoothly rather than cliff-diving.

**Transcript rewrite:** openclaw supports surgical modification of the conversation history. Instead of wholesale summarization, it can rewrite specific messages — replacing verbose tool outputs with concise versions, condensing repeated back-and-forth into single summary messages. This preserves conversation structure while reducing token count.

### Claude Code: `services/compact/` (12 files)

Claude Code throws the most engineering at this problem. 12 files, three parallel strategies, plus a lightweight pre-processor.

**Strategy 1 — Auto (threshold-based):**
Triggers at 80% of the context window. Performs standard summarization of older turns. This is the baseline, always running.

**Strategy 2 — Reactive (backpressure-triggered):**
Triggers when the model API returns a "context too long" error or when token estimation detects overflow. This is the emergency fallback — more aggressive than auto, potentially lossy, but guaranteed to get the conversation under the limit.

**Strategy 3 — Snip (fine-grained token-level):**
Operates at the individual message level. Instead of summarizing entire turns, snip truncates oversized individual messages — cutting a 50K-token file read down to the first and last 2K tokens with a "[truncated]" marker in between. Surgical, preserves message boundaries, minimal information loss for large outputs.

**microCompact — Lightweight pre-processing:**
Runs before the main strategies. Strips obvious noise: trailing whitespace, redundant formatting, empty blocks. Cheap enough to run on every turn. Shaves off 5-15% of token usage without any content loss.

The three strategies can fire independently. Auto handles gradual compression. Snip handles oversized individual messages. Reactive catches anything the other two miss. They don't conflict because they operate at different granularities.

### NemoClaw

Does not apply. NemoClaw is a security sandbox runtime, not an Agent. It doesn't manage conversations or context windows.

---

## Comparison Table

| Aspect | hermes | openclaw | Claude Code | NemoClaw |
|--------|--------|----------|-------------|----------|
| **Location** | `context_compressor.py` | `src/context-engine/` | `services/compact/` (12 files) | N/A |
| **Trigger threshold** | 50% | Configurable (active starts early) | 80% (auto), overflow (reactive) | N/A |
| **L1: Free pruning** | Strip old tool results | Tool result truncation | Snip + microCompact | N/A |
| **L2: LLM summarization** | Middle-turn summarization | Pluggable strategy | Auto strategy | N/A |
| **Proactive compression** | No (reactive only) | Yes (active mode) | Partially (auto at 80%) | N/A |
| **Transcript rewrite** | No | Yes (surgical modification) | No | N/A |
| **Iterative summary updates** | Yes (merge into existing) | No | No | N/A |
| **Extensibility** | Hardcoded | `registerContextEngine()` | Internal strategies, not pluggable | N/A |
| **Budget warning injection** | Yes (via Agent Core) | Yes (via Agent Core) | Yes (via Agent Core) | N/A |
| **Preserves latest results** | Yes (explicitly) | Configurable | Yes (snip only trims oversized) | N/A |

---

## Best Practices

**Start with two-level compression.** Level 1: prune old tool results (zero cost). Level 2: LLM summarize middle turns (one call). This is what hermes does, and it handles 80%+ of real-world conversations. Don't over-engineer on day one.

**Add budget warning injection early.** This is the single cheapest high-value addition. When Context Manager detects the window is filling up, Agent Core injects a message like "Context is running low — please wrap up or summarize your findings" into the next tool result. The LLM reads it, cooperatively finishes faster, avoids starting new tangents. Zero LLM cost. Dramatic behavior improvement. All three Agent projects implement this.

**Preserve the latest N tool results untouched.** hermes gets this right. The LLM is actively reasoning about recent tool outputs. Compressing them mid-thought is like pulling the papers off someone's desk while they're working. A safe default: never touch the last 3-5 tool results.

**Set your threshold below the hard limit.** hermes triggers at 50%, Claude Code at 80%. The lower the threshold, the more room for graceful compression before hitting the wall. 50% is conservative but safe. 80% gives more working room but less buffer. Pick based on your average tool result size.

**Make compression idempotent.** Running compression twice on the same messages array should produce the same result. hermes's iterative summary merge handles this well — re-summarizing an already-summarized turn produces a stable output rather than cascading information loss.

**Token estimation doesn't need to be exact.** All four projects use approximate token counting (character-based heuristics, not actual tokenizer calls). Overestimating by 10-20% is fine and much faster than running a real tokenizer on every message.

---

## Step-by-Step: How to Build It

### Phase 1: Token Counting + Threshold Detection

```
contextManager.estimateTokens(messages) → number
contextManager.isOverThreshold(messages, threshold) → boolean
```

Use a character-based heuristic: `tokenCount ≈ characterCount / 4` for English, `characterCount / 2` for CJK. Don't import a tokenizer library for this. You need speed, not precision.

Set your initial threshold at 60% of the model's max context window. You can tune later.

### Phase 2: Level 1 — Prune Tool Results

```
contextManager.pruneToolResults(messages, keepLatest = 3) → messages
```

Walk the messages array from oldest to newest. For any tool result message older than the `keepLatest` most recent ones, replace its content with a short stub: `"[Tool result removed — ran {toolName}]"`. Keep the message structure intact so the conversation flow reads naturally.

This alone will handle most conversations. Tool results are typically 60-80% of total token usage in a tool-heavy Agent.

### Phase 3: Level 2 — Summarize Middle Turns

```
contextManager.summarizeMiddle(messages, preserveFirst = 5, preserveLast = 5) → messages
```

If still over threshold after pruning:
1. Identify the "middle" — everything between the first N and last N messages
2. Send the middle to the LLM: "Summarize this conversation segment. Preserve all specific numbers, dates, file paths, error messages, and decisions made."
3. Replace the middle messages with a single assistant message containing the summary
4. Keep the preserved head and tail messages untouched

### Phase 4: Budget Warning Injection

This lives in Agent Core, not Context Manager, but they coordinate:

```
// In Agent Core's tool execution loop:
if (contextManager.usageRatio(messages) > 0.7) {
  toolResult.content += "\n\n[System: Context window is 70% full. Please wrap up your current task or summarize findings before starting new investigations.]"
}
```

The threshold for warnings should be lower than the compression threshold. Warn at 70%, compress at 80%. This gives the LLM a chance to self-regulate before mechanical compression kicks in.

### Phase 5: Wire Into Agent Core

Context Manager plugs into two points of the conversation loop:

```
while (budget > 0) {
  // POINT 1: Before calling model
  if (contextManager.isOverThreshold(messages)) {
    messages = contextManager.pruneToolResults(messages)
    if (contextManager.isOverThreshold(messages)) {
      messages = contextManager.summarizeMiddle(messages)
    }
  }

  response = model.call(messages)
  toolResults = tools.execute(response.toolCalls)

  // POINT 2: After tool execution, before appending results
  for (result of toolResults) {
    if (contextManager.usageRatio(messages) > 0.7) {
      result.content += budgetWarning
    }
    messages.push(result)
  }
}
```

### Phase 6 (Optional): Fine-Grained Snip

Add Claude Code's snip strategy for individual oversized messages:

```
contextManager.snipOversized(messages, maxPerMessage = 8000) → messages
```

For any single message exceeding `maxPerMessage` tokens, keep the first 2K and last 2K tokens, replace the middle with `[...truncated {N} tokens...]`. This handles the "Agent read a 200KB file" scenario without losing the beginning (usually contains the key information) or the end (usually contains the conclusion).

---

## Common Pitfalls

**Summarizing too aggressively loses critical data.** Numbers, dates, file paths, exact error messages, and specific decisions — these are the things summaries tend to drop. "The function returned an error" is a useless summary of "TypeError: Cannot read property 'map' of undefined at line 47 of UserList.tsx". hermes mitigates this by only summarizing older turns and preserving recent ones intact. If you must summarize, explicitly instruct the LLM to preserve specifics.

**Compressing before the LLM has acted on results.** If the LLM requested a file read and you compress the result before it gets to reason about it, you've wasted the tool call and the LLM will likely request the same file again. Always preserve the most recent tool results.

**No compression at all until you hit the limit.** Going from zero compression to emergency truncation produces the worst outcomes. The LLM suddenly loses massive context and can't recover. Gradual compression (openclaw's active mode, or simply a lower threshold) prevents the cliff-edge.

**Running the full tokenizer on every message.** Some implementations call tiktoken or the model's actual tokenizer for every token count estimate. This adds latency on every turn. A `chars / 4` heuristic with a 20% safety margin is fast and good enough.

**Forgetting that system prompt also consumes tokens.** Your system prompt — identity, instructions, memory injections, skill definitions — can easily be 5K-15K tokens. If your threshold calculation only counts user/assistant messages, you'll consistently underestimate usage and trigger compression too late.

**Treating compression as purely mechanical.** Budget warning injection is not a mechanical compression step — it's a cooperative signal. The LLM is a reasoning agent that can adapt its behavior when told resources are constrained. Leverage that. A one-line warning at 70% usage is often more effective than aggressive L2 summarization at 90%.

---

## Cross-Module Contracts

Context Manager is stateless but tightly coupled to Agent Core's message format. Here's what each module needs from the others:

| Contract | Between | What |
|----------|---------|------|
| **Messages array format** | Agent Core → Context Manager | Context Manager reads and returns `messages[]`. If you change the message schema (add metadata fields, change role names), Context Manager must handle it. |
| **Token estimation interface** | Context Manager → Agent Core | `estimateTokens(messages) → number`. Agent Core calls this before model invocations to decide if compression is needed. |
| **Budget warning injection point** | Agent Core ← Context Manager | Agent Core is responsible for injecting warnings. Context Manager provides the usage ratio; Agent Core decides the wording and placement. |
| **Compression input/output** | Agent Core ↔ Context Manager | Input: `messages[]`. Output: `messages[]` (shorter). The contract is that compressed messages are valid model input — same schema, fewer entries or smaller content. |
| **Model context window size** | Model → Context Manager | Context Manager needs to know the max tokens for threshold calculation. This comes from Model service's configuration. |
| **Tool result format** | Tools → Context Manager | Context Manager must identify which messages are tool results (to prune them selectively). This requires a stable `role: "tool"` or equivalent marker. |

**What happens if these contracts break:**
- Change message format without updating Context Manager → compression silently skips messages it doesn't recognize → context overflow
- Change tool result role marker → L1 pruning stops working → jumps straight to expensive L2 summarization
- Model changes max context without updating Context Manager → threshold percentages are wrong → compresses too early or too late

Keep these contracts in your `contracts/` directory. When any participating module changes its interface, the contracts file tells you exactly what else needs updating.

---

*Next: [Module 6 — Interface](06-interface.md) | Previous: [Module 4 — Memory](04-memory.md)*
