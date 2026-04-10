# 最小可行 Agent

> 5 个核心组件，其余皆可选。这是 Claude Code 的架构模式——也是目前最强大的生产级 Agent 之一背后的同款结构。

---

## 你需要什么

```
Interface (CLI)
Agent Core + Session Store
├── Model (无状态)
├── Tools + Skill Store
├── Context Manager (无状态)
└── Memory + Memory Store (可选但推荐)
```

**你不需要的**：Gateway、Routing、Security Envelope、Runtime/Infra。它们解决的是多用户、多渠道和部署问题。你目前还没有这些问题。

Memory 标注为可选，因为没有它 Agent 也能正常工作——每次会话从零开始。但建议尽早加上。没有 Memory，Agent 会遗忘会话间的一切信息，用户很快就会发现。

---

## 7 个组件

| # | 组件 | 状态 | 用途 |
|---|-----------|-------|---------|
| 1 | Interface (CLI) | 无状态 | 读取用户输入，展示模型输出 |
| 2 | Agent Core | 拥有 Session Store | while 循环：调用 Model，检查停止原因，执行 Tools，重复 |
| 3 | Model | 无状态 | 一个 Provider，一次 API 调用。返回文本或 tool_call |
| 4 | Tools | 拥有 Skill Store | 注册工具，按名称匹配，执行并返回结果 |
| 5 | Context Manager | 无状态 | 计算 token 数量，超出预算时截断 |
| 6 | Session Store | 持久化 | 将对话消息保存到磁盘 |
| 7 | Memory (可选) | 拥有 Memory Store | 将先前知识注入到 system prompt 中 |

Model 和 Context Manager 是无状态的。它们对传入的数据进行处理并返回结果，不涉及文件、数据库和副作用。

---

## 逐步构建

### 第 1 步：读取用户输入的 CLI

一个 readline 循环。接受一行文本，向下传递，打印响应。仅此而已。

```
cli/
└── index.ts          # readline 循环，调用 agentCore.run(userMessage)
```

这就是你的 Interface 层。hermes、openclaw 和 Claude Code 都从这里起步——一个 CLI 适配器，将用户输入规范化为消息对象，然后传递给 Agent Core。

**延伸阅读**：[Interface 模块指南](../modules/interface.md)

### 第 2 步：Agent Core —— while 循环

每个 Agent 的核心。我们研究的 4 个项目做的事完全一样：

```
while (budget > 0):
    response = model.call(messages)
    if response.stop_reason == "end_turn":
        break
    if response.stop_reason == "tool_use":
        result = tools.execute(response.tool_call)
        messages.append(result)
        budget -= 1
```

就这么简单。循环调用 Model，检查返回内容，按需执行 Tools，追加结果，然后继续循环。Claude Code 的 `query.ts` 通过 async generator 实现这一逻辑，hermes 在 `run_conversation()` 中实现。模式相同，语法不同。

Agent Core 拥有 Session Store——每次循环迭代后，它会将每条消息（user、assistant、tool_result）写入持久化存储。

**延伸阅读**：[Agent Core 模块指南](../modules/agent-core.md)

### 第 3 步：Model —— 一个 Provider，一次 API 调用

硬编码一个 Provider。Anthropic、OpenAI，随你选。一个函数：传入 messages，返回 response。response 包含文本内容或 tool_call。

```
model/
└── provider.ts       # callModel(messages): Response
```

不需要故障转移链，不需要候选轮换，不需要 token 估算。这些以后再加（参见 [Model 模块指南](../modules/model.md)）。现在你只需要：调用 API，返回 response。

Model 是无状态的。它不存储任何内容，不记住之前的调用。对话历史存在于 Agent Core 的 messages 数组中。

### 第 4 步：Tools —— 注册 2-3 个内置工具

从 2-3 个让 Agent 真正有用的工具开始：

- **file_read**：从磁盘读取文件。接收路径，返回内容。
- **shell_exec**：运行 shell 命令。接收命令字符串，返回 stdout/stderr。
- **file_write**（可选）：将内容写入文件。

注册很简单。hermes 使用导入时自注册——每个工具文件在加载时调用 `registry.register()`。对于 MVA，一个普通的对象 map 就够了：

```
tools/
├── registry.ts       # Map<string, ToolHandler>
└── builtins/
    ├── file-read.ts
    └── shell-exec.ts
```

每个工具需要：一个名称（Model 用来调用它的字符串）、一个描述参数的 JSON schema、一个 execute 函数。Model 看到 schema，决定何时使用工具，并传回参数。Agent Core 匹配名称，调用 execute，返回结果。

**延伸阅读**：[Tools 模块指南](../modules/tools.md)

### 第 5 步：Context Manager —— token 预算执行

最简版本：统计 messages 数组中的 token 数。如果超过 Model 上下文窗口的 80%，截断最旧的消息。

```
context/
└── manager.ts        # trimIfNeeded(messages, maxTokens): messages
```

Claude Code 使用 80% 作为自动压缩阈值，这是一个不错的默认值。token 计数不需要精确——`Math.ceil(text.length / 4)` 对英文文本的误差在 10% 以内。对于 MVA 来说足够了。

后续可实现三个层级（参见 [Context Manager 模块指南](../modules/context-manager.md)）：
1. L1：裁剪旧的工具结果（零次 LLM 调用，低成本）
2. L2：LLM 驱动的中间轮次摘要（成本高但效果好）
3. L3：硬截断最早的消息（最后手段）

目前只用 L3 就够了。

### 第 6 步：Session Store —— 持久化消息

每次循环迭代后保存 messages 数组。两种方案：

- **JSONL**：每行一个 JSON 对象。仅追加。简单。适合原型阶段。
- **SQLite**：一张表，包含 session_id、role、content、timestamp 列。更方便查询。

```
storage/
└── session-store.ts  # save(sessionId, message), load(sessionId): Message[]
```

启动时加载上一次会话的消息。每轮追加新消息。这就是会话连续性。

### 第 7 步：串联起来

```
agent/
├── core/
│   └── loop.ts           # Agent Core —— while 循环
├── model/
│   └── provider.ts       # 单个 Model Provider
├── tools/
│   ├── registry.ts       # 工具注册
│   └── builtins/
│       ├── file-read.ts
│       └── shell-exec.ts
├── context/
│   └── manager.ts        # token 计数 + 截断
├── storage/
│   └── session-store.ts  # JSONL 或 SQLite 持久化
├── interfaces/
│   └── cli.ts            # readline 循环
├── contracts/
│   ├── tool.ts           # Tool 接口定义
│   ├── model.ts          # Model 接口定义
│   └── message.ts        # Message 类型定义
└── tests/
    ├── unit/
    └── loops/            # 暂时为空——添加涌现行为时再填充
```

`contracts/` 目录很重要。即使在单个包中，也要为 Tool、Model 和 Message 类型定义接口。现在零成本，以后当你替换实现时，它们会帮你省下一切。

---

## 这能给你什么

一个可以工作的 Agent，能够：
- 接受自然语言指令
- 调用 LLM 决定下一步行动
- 执行文件和 shell 工具
- 管理自身的上下文窗口
- 跨会话持久化对话历史

总计：约 200 行核心逻辑。其余是工具实现和样板代码。

---

## 这不能给你什么

| 缺失能力 | 由哪个模块补充 | 何时添加 |
|---|---|---|
| 从经验中学习 | Memory + Skill Store + 学习循环 | 当你希望 Agent 能随时间进步时 |
| Model 故障转移 | Model 候选链 + 冷却探测 | 当你在生产环境中遇到速率限制时 |
| 多渠道支持 | Interface 适配器 + Gateway | 当用户需要通过 Slack/Discord/Web 访问时 |
| 安全沙箱 | Security Envelope | 当 Agent 在生产环境中运行 shell 命令时 |
| 子 Agent 委派 | Agent Core 委派逻辑 | 当单 Agent 上下文成本过高时 |

每项能力都有单独的指南。先构建 MVA，然后逐一添加能力。

---

## 模块指南索引

| 组件 | 指南 |
|-----------|-------|
| Interface (CLI) | [interface.md](../modules/interface.md) |
| Agent Core | [agent-core.md](../modules/agent-core.md) |
| Model | [model.md](../modules/model.md) |
| Tools | [tools.md](../modules/tools.md) |
| Context Manager | [context-manager.md](../modules/context-manager.md) |
| Session/Config Store | [storage.md](../modules/storage.md) |
| Memory（准备好时） | [memory.md](../modules/memory.md) |

---

*基于 Claude Code、hermes-agent、openclaw 和 NemoClaw 的架构模式。完整的 11 模块分解请参见 [架构参考模型](../../docs/agent-architecture-reference.md)。*
