# 模块 5：Context Manager

> 一个无状态服务，在对话变长时保持 Agent 的连贯性。

---

## 它是什么

Context Manager 是 Hub-and-Spoke 架构中 Agent Core 的四个对等服务之一。它的职责：最大化利用有限的上下文窗口。

它操作的是内存中的 messages 数组。它不存储任何东西。没有数据库，没有文件，没有持久状态。给它一个 messages 数组，返回一个更短的 messages 数组。这就是全部契约。

在参考架构中：

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

Context Manager 和 Model 是两个无状态的对等服务。两者都接收输入、返回输出，不拥有任何数据。

---

## 它解决什么问题

在任何长时间运行的 Agent 对话中，有三个问题会不断累积：

1. **上下文窗口是有限的。** Claude 的 200K token 听起来很多，直到你的 Agent 跑了 30 次工具调用。每次工具结果可能是 2K-10K token。光工具结果就要 60K-300K token。

2. **工具结果累积速度很快。** 文件读取返回整个文件。网页搜索返回多页结果。Shell 命令倾倒整个 stdout。十几次工具调用后，工具结果就会主导上下文窗口——把真正重要的原始对话挤出去。

3. **长对话会失去连贯性。** 当上下文填满，关键的早期消息被挤掉后，LLM 会开始自相矛盾，忘记已经尝试过什么，重复失败的方法。

没有 Context Manager，你的 Agent 会触及 token 上限，然后要么崩溃（硬故障），要么静默丢弃早期消息并产生不连贯的输出（软故障）。两者都不可接受。

---

## 4 个项目是怎么做的

### hermes-agent：`context_compressor.py`

hermes 采用了简单直接的两级方法。

**触发条件：** token 使用量超过模型上下文窗口的 50% 阈值。

**第 1 级——修剪旧的工具结果（零 LLM 调用）：**
剥离旧轮次的工具结果。原理是：工具结果体积大，大部分已经过时（一旦 LLM 已经基于它们采取行动，原始输出就很少再有用），而且移除它们是免费的——不需要摘要调用。

关键细节：hermes 保留最新的工具结果不动。只修剪较旧的轮次。这防止丢失 LLM 正在使用的数据。

**第 2 级——LLM 摘要中间轮次：**
如果修剪后仍然超过阈值，将对话的中间部分发送给 LLM，附带"摘要这些轮次"的指令。前几轮（任务上下文）和最后几轮（近期工作）保持完整。只有中间部分被压缩。

hermes 采用迭代更新现有摘要的方式，而不是每次创建全新的摘要。如果之前的压缩轮次已经存在摘要，新的轮次会将新信息合并到现有摘要中，而不是替换它。这防止了多次压缩循环中的信息漂移。

**成本概况：** 便宜。L1 是免费的。L2 每次压缩事件花费一次 LLM 调用。对于大多数对话，仅 L1 就足够了。

### openclaw：`src/context-engine/`

openclaw 采用了可扩展的方法。

**架构：** 通过 `registerContextEngine()` 实现可插拔。你可以在不触碰 Agent Core 的情况下替换整个压缩策略。这遵循了 openclaw 的理念——一切都是 Plugin。

**两种压缩模式：**

| 模式 | 触发条件 | 功能 |
|------|---------|-------------|
| 主动（proactive） | 在窗口填满之前 | 提前开始压缩，避免质量突然下降 |
| 被动（reactive） | 触及限制之后 | 主动模式不够用时的紧急压缩 |

主动模式是关键差异化特征。大多数实现会等到窗口快满了才匆忙压缩。openclaw 提前开始，逐步压缩，让质量平缓退化而不是断崖式下跌。

**对话重写：** openclaw 支持对对话历史进行精确修改。它不是做批量摘要，而是可以重写特定消息——将冗长的工具输出替换为简洁版本，将重复的来回对话浓缩为单条摘要消息。这在减少 token 数量的同时保留了对话结构。

### Claude Code：`services/compact/`（12 个文件）

Claude Code 在这个问题上投入了最多的工程资源。12 个文件，三种并行策略，加一个轻量预处理器。

**策略 1——Auto（基于阈值）：**
在上下文窗口的 80% 处触发。对较旧的轮次执行标准摘要。这是基线，始终运行。

**策略 2——Reactive（背压触发）：**
当模型 API 返回"上下文过长"错误或 token 估算检测到溢出时触发。这是紧急后备——比 auto 更激进，可能有损，但保证将对话控制在限制以内。

**策略 3——Snip（细粒度 token 级）：**
在单条消息级别操作。不是摘要整个轮次，snip 截断超大的单条消息——将 50K token 的文件读取缩减为前 2K 和后 2K token，中间放一个 "[truncated]" 标记。精确操作，保留消息边界，对大输出的信息损失最小。

**microCompact——轻量预处理：**
在主策略之前运行。剥离明显的噪声：尾部空白、冗余格式、空块。足够便宜，可以在每个轮次运行。减少 5-15% 的 token 使用量，不损失任何内容。

三种策略可以独立触发。Auto 处理渐进式压缩。Snip 处理超大的单条消息。Reactive 捕获其他两个遗漏的一切。它们不会冲突，因为操作粒度不同。

### NemoClaw

不适用。NemoClaw 是一个安全沙箱运行时，不是 Agent。它不管理对话或上下文窗口。

---

## 对比表

| 方面 | hermes | openclaw | Claude Code | NemoClaw |
|--------|--------|----------|-------------|----------|
| **位置** | `context_compressor.py` | `src/context-engine/` | `services/compact/`（12 个文件） | N/A |
| **触发阈值** | 50% | 可配置（主动模式提前启动） | 80%（auto），溢出（reactive） | N/A |
| **L1：免费修剪** | 剥离旧工具结果 | 工具结果截断 | Snip + microCompact | N/A |
| **L2：LLM 摘要** | 中间轮次摘要 | 可插拔策略 | Auto 策略 | N/A |
| **主动压缩** | 否（仅被动） | 是（主动模式） | 部分（auto 在 80%） | N/A |
| **对话重写** | 否 | 是（精确修改） | 否 | N/A |
| **迭代摘要更新** | 是（合并到现有） | 否 | 否 | N/A |
| **可扩展性** | 硬编码 | `registerContextEngine()` | 内部策略，不可插拔 | N/A |
| **预算警告注入** | 是（通过 Agent Core） | 是（通过 Agent Core） | 是（通过 Agent Core） | N/A |
| **保留最新结果** | 是（显式） | 可配置 | 是（snip 只修剪超大的） | N/A |

---

## 最佳实践

**从两级压缩开始。** 第 1 级：修剪旧的工具结果（零成本）。第 2 级：LLM 摘要中间轮次（一次调用）。这就是 hermes 的做法，它能处理 80% 以上的真实对话。不要在第一天就过度工程化。

**尽早添加预算警告注入。** 这是性价比最高的单一功能。当 Context Manager 检测到窗口正在填满时，Agent Core 向下一个工具结果注入一条消息，如"上下文即将耗尽——请收尾或总结你的发现"。LLM 读到后会配合地更快完成，避免开启新的话题分支。零 LLM 成本。行为改善显著。三个 Agent 项目都实现了这个功能。

**保留最近的 N 个工具结果不动。** hermes 做对了这一点。LLM 正在积极推理最近的工具输出。在它思考过程中压缩这些结果，就像在别人工作时把桌上的文件抽走。安全默认值：永远不触碰最后 3-5 个工具结果。

**将阈值设置在硬限制以下。** hermes 在 50% 触发，Claude Code 在 80%。阈值越低，在撞墙之前留给优雅压缩的空间就越多。50% 保守但安全。80% 给更多工作空间但缓冲更少。根据你的平均工具结果大小来选择。

**使压缩具有幂等性。** 对同一个 messages 数组运行两次压缩应该产生相同的结果。hermes 的迭代摘要合并很好地处理了这一点——对已经摘要过的轮次再次摘要会产生稳定的输出，而不是级联的信息损失。

**Token 估算不需要精确。** 所有四个项目都使用近似 token 计数（基于字符数的启发式，而不是实际的分词器调用）。高估 10-20% 是完全可以的，比对每条消息运行真正的分词器快得多。

---

## 分步构建指南

### 阶段 1：Token 计数 + 阈值检测

```
contextManager.estimateTokens(messages) → number
contextManager.isOverThreshold(messages, threshold) → boolean
```

使用基于字符数的启发式：英文 `tokenCount ≈ characterCount / 4`，CJK `characterCount / 2`。不要为此引入分词器库。你需要的是速度，不是精度。

将初始阈值设为模型最大上下文窗口的 60%。之后可以调整。

### 阶段 2：第 1 级——修剪工具结果

```
contextManager.pruneToolResults(messages, keepLatest = 3) → messages
```

从最旧到最新遍历 messages 数组。对于比最近 `keepLatest` 条更旧的工具结果消息，将其内容替换为简短的占位符：`"[Tool result removed — ran {toolName}]"`。保持消息结构完整，使对话流程读起来自然。

仅此一步就能处理大多数对话。在工具密集型 Agent 中，工具结果通常占总 token 使用量的 60-80%。

### 阶段 3：第 2 级——摘要中间轮次

```
contextManager.summarizeMiddle(messages, preserveFirst = 5, preserveLast = 5) → messages
```

如果修剪后仍超过阈值：
1. 确定"中间部分"——前 N 条和后 N 条消息之间的所有内容
2. 将中间部分发送给 LLM："摘要这段对话。保留所有具体的数字、日期、文件路径、错误消息和已做的决定。"
3. 用包含摘要的单条 assistant 消息替换中间的消息
4. 保持保留的头部和尾部消息不动

### 阶段 4：预算警告注入

这个功能位于 Agent Core 而非 Context Manager 中，但两者需要协调：

```
// In Agent Core's tool execution loop:
if (contextManager.usageRatio(messages) > 0.7) {
  toolResult.content += "\n\n[System: Context window is 70% full. Please wrap up your current task or summarize findings before starting new investigations.]"
}
```

警告的阈值应低于压缩的阈值。在 70% 警告，在 80% 压缩。这给 LLM 一个在机械压缩启动之前自我调节的机会。

### 阶段 5：接入 Agent Core

Context Manager 插入对话循环的两个点：

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

### 阶段 6（可选）：细粒度 Snip

添加 Claude Code 的 snip 策略来处理单条超大消息：

```
contextManager.snipOversized(messages, maxPerMessage = 8000) → messages
```

对于任何超过 `maxPerMessage` token 的单条消息，保留前 2K 和后 2K token，中间替换为 `[...truncated {N} tokens...]`。这处理了"Agent 读取了一个 200KB 文件"的场景，同时不丢失开头（通常包含关键信息）和结尾（通常包含结论）。

---

## 常见陷阱

**过于激进的摘要会丢失关键数据。** 数字、日期、文件路径、精确的错误消息和具体决定——这些是摘要最容易丢弃的东西。"函数返回了一个错误"是对 "TypeError: Cannot read property 'map' of undefined at line 47 of UserList.tsx" 的无用摘要。hermes 通过只摘要较旧轮次、完整保留最近轮次来缓解这个问题。如果你必须摘要，明确指示 LLM 保留具体信息。

**在 LLM 处理结果之前就压缩。** 如果 LLM 请求了一次文件读取，而你在它推理之前就压缩了结果，那你浪费了这次工具调用，LLM 很可能会再次请求同一个文件。始终保留最近的工具结果。

**完全不压缩直到触及限制。** 从零压缩直接跳到紧急截断会产生最差的结果。LLM 突然失去大量上下文且无法恢复。渐进式压缩（openclaw 的主动模式，或者简单地设置较低的阈值）可以防止断崖式下跌。

**对每条消息运行完整的分词器。** 有些实现对每次 token 计数估算都调用 tiktoken 或模型的实际分词器。这在每个轮次都增加延迟。`chars / 4` 的启发式加 20% 安全余量既快速又足够准确。

**忘记 system prompt 也消耗 token。** 你的 system prompt——身份、指令、Memory 注入、技能定义——很容易占到 5K-15K token。如果你的阈值计算只统计 user/assistant 消息，你会持续低估使用量，触发压缩过晚。

**将压缩视为纯粹的机械操作。** 预算警告注入不是一个机械的压缩步骤——它是一个协作信号。LLM 是一个能在被告知资源受限时调整自身行为的推理 Agent。利用这一点。在 70% 使用量时的一行警告往往比在 90% 时进行激进的 L2 摘要更有效。

---

## 跨模块契约

Context Manager 是无状态的，但与 Agent Core 的消息格式紧密耦合。以下是各模块之间的依赖关系：

| 契约 | 模块之间 | 内容 |
|----------|---------|------|
| **Messages 数组格式** | Agent Core → Context Manager | Context Manager 读取并返回 `messages[]`。如果你改变了消息的 schema（添加元数据字段、更改角色名称），Context Manager 必须能处理。 |
| **Token 估算接口** | Context Manager → Agent Core | `estimateTokens(messages) → number`。Agent Core 在模型调用前调用此方法来决定是否需要压缩。 |
| **预算警告注入点** | Agent Core ← Context Manager | Agent Core 负责注入警告。Context Manager 提供使用率；Agent Core 决定措辞和放置位置。 |
| **压缩输入/输出** | Agent Core ↔ Context Manager | 输入：`messages[]`。输出：`messages[]`（更短的）。契约是压缩后的消息是合法的模型输入——相同的 schema，更少的条目或更小的内容。 |
| **模型上下文窗口大小** | Model → Context Manager | Context Manager 需要知道最大 token 数来计算阈值。这来自 Model 服务的配置。 |
| **工具结果格式** | Tools → Context Manager | Context Manager 必须能识别哪些消息是工具结果（以便选择性修剪）。这需要稳定的 `role: "tool"` 或等效标记。 |

**这些契约被破坏时会发生什么：**
- 改变消息格式但不更新 Context Manager → 压缩会静默跳过它不认识的消息 → 上下文溢出
- 改变工具结果的角色标记 → L1 修剪停止工作 → 直接跳到昂贵的 L2 摘要
- 模型改变最大上下文而不更新 Context Manager → 阈值百分比不正确 → 压缩过早或过晚

将这些契约保存在你的 `contracts/` 目录中。当任何参与模块改变其接口时，契约文件会告诉你还需要更新什么。

---

*下一篇：[模块 6 — Interface](06-interface.md) | 上一篇：[模块 4 — Memory](04-memory.md)*
