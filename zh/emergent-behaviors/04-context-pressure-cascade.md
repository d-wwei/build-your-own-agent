# 涌现行为 4：Context Pressure Cascade

> 系统在上下文窗口压力下以逐级升级的压缩策略和 LLM 协同行为变化来响应，尽管没有任何组件被设计来编排优雅降级。

## 它是什么

每个 LLM Agent 都有有限的上下文窗口。填满后 Agent 就会崩溃。Context pressure cascade 是三个独立机制——多级压缩、预算警告注入和 LLM 行为适应——组合后创造出的系统行为：优雅地应对压力而非崩溃。

可以想象一根弹簧被压缩。轻度压力：轻柔响应。更大压力：更强响应。最大压力：紧急响应。系统跨级别自动升级。没有编排器协调。

每个生产 Agent 都需要这个。它不是可选的。

## 它如何涌现

三个组件独立工作：

1. **Context Manager** 监控 token 计数与阈值的关系。超过阈值时，它按升级顺序应用压缩策略。它不知道预算警告或 LLM 行为的存在。
2. **Agent Core** 在上下文紧张时，向工具结果中注入一个 `budget_warning` 字符串。它不知道压缩策略或 LLM 适应的存在。
3. **LLM** 读取预算警告并改变行为——更短的回复、更少的工具调用、更快地收尾。它没有被编程来做这件事。它只是读取警告然后适应。

没有人设计了"协同降级"。它从以下过程中涌现：

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

Context Manager 压缩。Agent Core 警告。LLM 协同配合。组合结果：系统优雅地应对压力，没有任何组件理解全貌。

## 哪些组件参与

| 组件 | 角色 | 对整体的感知 |
|---|---|---|
| Context Manager | 多级压缩（L1/L2/L3） | 知道 token 计数和阈值。不知道预算警告或 LLM 适应。 |
| Agent Core | 向工具结果注入 `budget_warning` | 知道上下文紧张。不知道当前激活了哪个压缩级别。 |
| Model (LLM) | 读取警告、改变行为 | 什么都不知道。只是遵循警告文本中的指令。 |

这个行为跨越两个模块边界：Context Manager <-> Agent Core <-> Model。这使得它比自愈 Routing（仅在一个模块内部）更难正确实现。

## 案例研究：hermes-agent

hermes 在 `context_compressor.py` 中使用 50% 阈值。以下是长编码会话中上下文填满时的具体过程：

**Turn 1-40：** 正常运行。上下文占窗口的 35%。无压缩。

**Turn 41：** 上下文达到 50%。L1 触发。
- 旧的工具结果（Turn 1-20 的 shell 输出、文件内容）被裁剪。替换为单行摘要：`[tool result pruned: 847 tokens -> 12 tokens]`。
- 零 LLM 调用。快速。低成本。
- 上下文降至 38%。

**Turn 42-65：** 上下文再次增长。L1 裁剪 Turn 20-40 的工具结果。上下文在 48%。

**Turn 66：** 上下文再次达到 50%，但已经没有更多工具结果可以裁剪。L2 触发。
- 调用 LLM 将 Turn 10-50 总结为结构化摘要。
- 原始消息被摘要替换。开销大（一次 LLM 调用）但有效。
- 上下文降至 30%。

**Turn 67：** Agent Core 检查上下文比例。仍高于 40%。向下一个工具结果注入：
```
[budget_warning: context at 82% after compression. Please wrap up current task.]
```

**Turn 68：** LLM 读取预算警告。它没有再调用 3 个工具，而是写了一份最终总结，询问用户是否要在新会话中继续。

**Turn 90+：** 如果用户继续，即使 L2 之后上下文达到 95%，L3 触发：硬截断最早的消息。这是最后手段。信息会丢失。但 Agent 不会崩溃。

协同部分——LLM 读取 `budget_warning` 并改变行为——从未被显式编程。hermes 只是注入了一个字符串。LLM 解读了它。这之所以有效，是因为 LLM 天生就是指令跟随者。

## 如何在你的 Agent 中让它涌现

你需要三个部件。分别构建它们。

### 部件 1：多级压缩（Context Manager）

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

**L1 细节——裁剪工具结果：**
- 目标：超过 N 个 turn 之前的工具结果消息。
- 用 `[pruned: {originalTokens} tokens, was: {toolName} call]` 替换内容。
- 保留消息结构（role、tool_call_id）。只替换内容。
- 成本：零 LLM 调用。仅字符串替换。

**L2 细节——LLM 总结：**
- 选择一段中间消息（不是最近的，也不是 system prompt）。
- 发送给 LLM 并附带指令："Summarize these conversation turns. Preserve all decisions, file paths, error messages, and pending tasks."
- 用一条包含摘要的 assistant 消息替换该范围。
- 成本：一次 LLM 调用。通常 500-2000 output token。

**L3 细节——截断：**
- 从前端移除消息（最旧的优先）。
- 始终保留：system prompt、最近 5 个 turn、任何包含未解决问题的用户消息。
- 成本：零。但信息从上下文中永久丢失。

### 部件 2：Budget Warning 注入（Agent Core）

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

**在哪里注入：** 追加到工具结果的 content 中，而非作为独立消息。LLM 将其视为工具输出的一部分。这比系统级指令更有效，因为它以实时上下文的形式到达，而非静态规则。

**阈值：** 75% 是一个合理的起点。hermes 在多个级别使用 context_ratio 检查。Claude Code 在 80% 时触发 auto-compact。

### 部件 3：让 LLM 协同配合（无需代码）

这个部件需要零实现。LLM 读取 `budget_warning` 文本然后自适应。它会写更短的回复。它会调用更少的工具。它会加快收尾。

这之所以有效，是因为：
- 警告使用自然语言。LLM 天生遵循自然语言指令。
- 警告被放在工具结果之后，这是 LLM 的高注意力上下文区域。
- 警告是具体的：它给出了百分比，而不仅仅是"小心点"。

你不需要微调模型。不需要特殊 token。只需要一个清晰、具体的警告字符串，放在 LLM 会读取的位置。

### 组装

将它们接入你的 Agent 循环：

```
before each LLM call:
  currentTokens = estimateTokens(messages)
  if currentTokens > threshold:
    messages = compressContext(messages, maxTokens, currentTokens)

after each tool execution:
  contextRatio = estimateTokens(messages) / maxTokens
  toolResult = maybeInjectBudgetWarning(toolResult, contextRatio)
```

级联涌现了。没有编排器。Context Manager 在需要时压缩。Agent Core 在需要时警告。LLM 在读到警告时适应。三个独立机制创造出协同降级。

## 跨模块契约

这个行为跨越两个模块边界。三个契约必须被维护。

| 契约 | 格式 | 关联方 |
|---|---|---|
| 压缩输入/输出 | `Message[]` 数组进，`Message[]` 数组出。相同 schema。压缩后的消息必须对 LLM API 有效。 | Context Manager <-> Agent Core |
| Budget warning 注入 | 追加到 `toolResult.content` 的字符串。格式：`[budget_warning: ...]`。不能破坏工具结果解析。 | Agent Core <-> Model（通过 tool result） |
| Token 估算接口 | `estimateTokens(messages: Message[]): number`。必须可被 Agent Core（用于警告阈值）和 Context Manager（用于压缩阈值）同时调用。 | 两者共享的工具函数 |

**危险区域：** 如果你更改了 `Message[]` schema（例如添加新字段、改变工具结果的结构方式），压缩可能产生无效消息。LLM 调用将失败，级联中断。schema 变更后务必运行循环测试。

## 如何测试（循环测试）

文件：`tests/loops/context-cascade.test.ts`

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

## 各项目差异

### hermes-agent

- **阈值：** 上下文窗口的 50%。
- **L1：** 裁剪较旧 turn 的工具结果。
- **L2：** LLM 结构化总结。使用迭代式摘要更新——每次新摘要基于前一次构建，而非从头重新总结。
- **Budget warning：** 基于 `context_ratio` 注入到工具结果中。
- **优势：** 迭代式摘要在长会话中保持上下文质量。
- **劣势：** 单一 50% 阈值。无法区分"快满了"和"危急状态"。

### openclaw

- **架构：** 通过 `registerContextEngine()` 实现可插拔的 context engine。
- **两种模式：** 主动压缩（在窗口填满前触发）和被动压缩（在背压时触发）。
- **特色功能：** Transcript rewrite——精确编辑特定消息，而非总结一个范围。比 hermes 基于范围的方式更精确。
- **优势：** 主动模式防止窗口进入危急状态。是前瞻性的，而非被动响应的。
- **劣势：** 插件架构增加了复杂性。你需要实现 context engine 接口。

### Claude Code

- **规模：** `services/compact/` 下 12 个文件。
- **三种策略并行运行：**
  - `auto`：80% 阈值触发。标准压缩。
  - `reactive`：背压触发（API 返回 context-too-long 错误）。紧急模式。
  - `snip`：细粒度逐消息压缩。可以压缩单个工具结果而不影响其他消息。
- **特色功能：** `microCompact` 预处理——对每条消息在进入上下文前应用轻量级 token 削减。去除空白、缩短常见模式。
- **优势：** 策略最丰富。三级粒度。microCompact 降低基线压力。
- **劣势：** 12 个文件用于上下文管理意味着显著的复杂性。三种策略之间可能产生意外交互。

### 汇总表

| 维度 | hermes | openclaw | Claude Code |
|---|---|---|---|
| 触发阈值 | 50% | 可配置（主动 + 被动） | 80%（auto）/ 背压（reactive） |
| L1（低成本） | 裁剪工具结果 | 工具结果截断 | snip（细粒度） |
| L2（高成本） | LLM 总结范围 | Transcript rewrite | auto compact（LLM 总结） |
| L3（紧急） | 截断最早消息 | 未记录 | reactive（API 错误时） |
| 预处理 | 无 | 无 | microCompact |
| 主动模式 | 否 | 是（主动压缩） | 否（全部被动） |
| Budget warning | 是（工具结果注入） | 未记录 | 未记录 |
| 协同降级 | 是 | 部分 | 部分 |
| 文件/复杂度 | 1 个文件 | 插件接口 + 实现 | 12 个文件 |

最简路径：从 hermes 的方式入手（1 个文件、3 个级别、budget warning 注入）。当你理解了 Agent 的上下文消耗模式后，再添加 openclaw 的主动模式。如果你需要在压缩启动前将更多内容塞进窗口，考虑 Claude Code 的 micro-compact 预处理。
