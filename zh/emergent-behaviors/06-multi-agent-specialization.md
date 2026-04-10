# 涌现行为 6：Multi-Agent Specialization

> 一句话：来自不同渠道的消息被路由到不同配置的 Agent 实例——每个实例拥有自己的 Tools、Memory 和人格——将一套代码库变成一个专家团队。

## 它是什么

一条消息从 Telegram 群组 12345 到达。Routing 层查询 binding 表：群组 12345 映射到 `investmentAgent`。该 Agent 拥有金融分析工具、一个充满投资组合数据的 Memory 存储，以及一个写着"你是一位资深投资分析师"的 system prompt。

与此同时，一条消息从 Slack 频道 #ops 到达。Routing 再次查询：#ops 映射到 `devopsAgent`。不同的工具。不同的 Memory。不同的人格。"你是一位专注于事件响应的 DevOps 工程师。"

两个 Agent 都是同一个 Agent Core 类的实例。相同的对话循环。相同的 `while(budget > 0)` 结构。但它们表现得像完全不同的实体，因为它们是用不同的参数构造的。

没有单个组件被设计来创建"一个专家团队"。Routing 只做模式匹配。Agent Core 只运行循环。Interface 层只投递消息。组合结果 = 一个系统，其中合适的专家自动处理每一段对话。

## 它如何涌现

三个组件交互：

1. **Routing** — 将传入消息与 binding 表进行模式匹配。"Telegram 群组 12345 → investmentAgent。"这就是 Routing 所做的全部。它不知道投资 Agent 是什么。它只是解析一个名称。

2. **Agent Core** — 一个实例工厂从配置创建 Agent Core 实例。每个配置指定：Tools、Memory 后端、人格 prompt、模型偏好、迭代预算。工厂不关心一个配置写的是"金融分析师"还是另一个写的是"DevOps 工程师"。它只是按配置描述构建。

3. **Interface** — Telegram、Slack、Discord 等的适配器。每个适配器将消息标准化为通用格式并投递到 Routing 层。适配器不知道哪个 Agent 会处理消息。

涌现行为：这些组件中没有一个"知道"专业化的存在。Routing 匹配字符串。Agent Core 遵循配置。Interface 标准化协议。但组合系统将金融问题路由给金融专家，将 DevOps 问题路由给 DevOps 专家。专业化从组合中涌现。

## 哪些组件参与

| 组件 | 在专业化中的角色 |
|-----------|----------------------|
| Interface（多渠道） | 从 Telegram、Slack、Discord 等接收消息 |
| Gateway + Routing | 将消息来源与 Agent binding 进行模式匹配 |
| Agent Core x N | 多个实例，每个拥有不同配置 |
| Tools（每 Agent） | 每个 Agent 有自己注册的工具集 |
| Memory（每 Agent） | 每个 Agent 有自己的 Memory 存储 |
| Config Store | 持有 binding 规则和每个 Agent 的配置 |

**与 Sub-Agent Delegation（行为 5）的关键区别：** 在委派中，父级为一个任务生成临时子级。在专业化中，所有 Agent 都是永久的对等关系。没有父子关系。没有摘要返回路径。每个 Agent 与自己的用户进行持续对话。

## 案例研究：openclaw 的 9 级 Binding 优先级

在四个项目中，openclaw 是唯一实现了生产级多 Agent 专业化的项目。以下是确切的 Routing 优先级：

| 优先级 | Binding 级别 | 示例 |
|----------|--------------|---------|
| 1 | 精确 chat/group 匹配 | Telegram 群组 `12345` → `investmentAgent` |
| 2 | 父消息线程继承 | 线程中的回复 → 与父消息相同的 Agent |
| 3 | 通配符匹配 | 所有匹配 `invest-*` 的 Telegram 群组 → `investmentAgent` |
| 4 | Discord server + role | Discord server `trading-guild` + role `analyst` → `marketAgent` |
| 5 | Discord server | Discord server `trading-guild` 中的任何消息 → `generalTradeAgent` |
| 6 | Slack team | Slack workspace `acme-corp` → `acmeAgent` |
| 7 | 账户级别 | 来自用户 `@ceo` 的所有消息，无论渠道 → `executiveAgent` |
| 8 | 渠道级别 | 所有 Telegram 消息 → `defaultTelegramAgent` |
| 9 | 默认兜底 | 无匹配 → `defaultAgent` |

**sessionKey 构造**：每段对话获得一个确定性的 sessionKey。对于 Telegram 群组 12345，key 为 `tg-12345-main`。对于该群组中的线程，为 `tg-12345-thread-789`。sessionKey 决定加载哪段对话历史。两条发送到同一群组的消息继续同一段对话。新线程从头开始。

**Agent 实例生命周期**：Agent 不是按消息生成的。它们是长期运行的实例，每个拥有持久化 Memory 和持续的对话状态。`investmentAgent` 记得昨天与 Telegram 群组 12345 讨论了什么。

**详细流程：**

```
1. Telegram adapter receives message from group 12345
2. Adapter normalizes: { source: "telegram", chatId: "12345", text: "What's AAPL's P/E?" }
3. Gateway passes to Routing
4. Routing checks bindings at priority 1: exact match found → "investmentAgent"
5. Routing constructs sessionKey: "tg-12345-main"
6. Agent instance factory: get or create investmentAgent
7. investmentAgent.run(message, sessionKey)
8. Agent Core loads conversation history for sessionKey
9. Agent Core builds context: persona + tools + memory + history
10. Normal conversation loop proceeds
11. Response routes back through Gateway → Telegram adapter → group 12345
```

## 其他项目的对比

| 项目 | 多 Agent 支持 | 原因 |
|---------|-------------------|-----|
| **openclaw** | 完整实现。9 级 binding、每 Agent 独立配置、多渠道 | 从一开始就被设计为多租户 AI 网关 |
| **hermes-agent** | 无。单 Agent 架构。无 Routing 模块。 | 作为自进化的单 Agent 构建 |
| **Claude Code** | 无。单用户、单 Agent。 | 作为面向单个开发者的个人 CLI 工具构建 |
| **NemoClaw** | N/A。不是 Agent——它是沙箱运行时。 | 在容器内运行 openclaw 实例 |

这是四个项目之间最鲜明的分歧。专业化需要一个 Routing 层。只有 openclaw 构建了一个。其他项目选择了单 Agent 路径，优化深度（hermes：学习循环；Claude Code：推测执行）而非广度。

## 如何在你的 Agent 中让它涌现

### 步骤 1：定义 Agent 配置

每个专家 Agent 需要一个配置文件：

```yaml
# agents/investment-agent.yaml
name: investmentAgent
persona: "You are a senior investment analyst. You provide data-driven financial analysis."
model: gpt-4
budget: 60
tools:
  - stock_lookup
  - financial_report
  - portfolio_analyzer
memory_backend: sqlite
memory_namespace: "investment"
```

```yaml
# agents/devops-agent.yaml
name: devopsAgent
persona: "You are a DevOps engineer. You diagnose incidents and suggest fixes."
model: gpt-4
budget: 40
tools:
  - check_service_health
  - read_logs
  - restart_service
memory_backend: sqlite
memory_namespace: "devops"
```

### 步骤 2：构建 binding 表

```yaml
# routing/bindings.yaml
bindings:
  - match: { platform: "telegram", chatId: "12345" }
    agent: investmentAgent
    priority: 1

  - match: { platform: "slack", channel: "ops" }
    agent: devopsAgent
    priority: 1

  - match: { platform: "telegram" }
    agent: defaultAgent
    priority: 8

  - match: {}
    agent: defaultAgent
    priority: 9
```

### 步骤 3：构建路由器

路由器很简单。对于每条传入消息：

```python
def route(message):
    bindings_sorted = sorted(BINDINGS, key=lambda b: b.priority)
    for binding in bindings_sorted:
        if matches(message.source, binding.match):
            return binding.agent
    return "defaultAgent"
```

这只有 10 行代码。力量来自 binding 表，而非算法。

### 步骤 4：构建 Agent 实例工厂

```python
agents = {}

def get_agent(agent_name):
    if agent_name not in agents:
        config = load_config(f"agents/{agent_name}.yaml")
        agents[agent_name] = AgentCore(
            persona=config.persona,
            model=config.model,
            budget=config.budget,
            tools=load_tools(config.tools),
            memory=create_memory(config.memory_backend, config.memory_namespace)
        )
    return agents[agent_name]
```

Agent 在首次使用时创建并缓存。每个拥有自己的 Tools、Memory 和人格。

### 步骤 5：接通 sessionKey

```python
def build_session_key(message):
    platform = message.source.platform    # "telegram"
    chat_id = message.source.chat_id      # "12345"
    thread_id = message.source.thread_id  # None or "789"

    if thread_id:
        return f"{platform}-{chat_id}-thread-{thread_id}"
    return f"{platform}-{chat_id}-main"
```

sessionKey 决定 Agent 加载哪段对话历史。相同的 key = 继续对话。新的 key = 全新开始。

### 步骤 6：连接各部件

```python
def handle_message(message):
    agent_name = route(message)
    agent = get_agent(agent_name)
    session_key = build_session_key(message)
    response = agent.run(message.text, session_key)
    send_response(message.source, response)
```

六行代码。这就是整个编排层。复杂性存在于 binding 表和 Agent 配置中，而非胶水代码中。

## 跨模块契约

### 1. Binding 匹配规则

```
{
  platform: string,         // "telegram" | "slack" | "discord" | ...
  chatId?: string,          // exact match
  channel?: string,         // exact match
  serverId?: string,        // Discord server
  role?: string,            // Discord role
  userId?: string,          // account-level binding
  pattern?: string          // wildcard match (e.g., "invest-*")
}
```

优先级是显式的（1-9）。平级时先匹配者胜。没有基于字段特异性的隐式优先级——那会导致调试噩梦。

### 2. sessionKey 构造

格式：`{platform}-{chatId}-{threadOrMain}`

规则：
- 确定性的。相同输入始终产生相同的 key。
- 线程消息继承父 binding 但获得独立的 sessionKey。
- 一个 Agent 可以服务多个 sessionKey（例如 `investmentAgent` 同时处理 `tg-12345-main` 和 `tg-67890-main`）。
- sessionKey 永远不会跨 Agent。如果 `tg-12345-main` 绑定到 `investmentAgent`，将其重新分配给 `devopsAgent` 会开始一段新对话，而非继续之前的。

### 3. Agent 实例工厂

```
Input:  agent_name (string)
Output: AgentCore instance

Contract:
- First call creates the instance from config
- Subsequent calls return the cached instance
- Each instance has isolated: tools, memory namespace, persona, budget
- Shared: model provider connection (pooled, not duplicated)
```

### 4. Memory 命名空间隔离

每个 Agent 写入自己的 Memory 命名空间。`investmentAgent` 不能读取 `devopsAgent` 的 Memory。这在 Memory 后端级别被强制执行：

```
investmentAgent → memory.sqlite, table: investment_*
devopsAgent     → memory.sqlite, table: devops_*
```

共用同一个数据库没问题。独立的表是强制要求。

## 如何测试（循环测试）

### 测试 1：正确的 Routing

```
1. Send message from Telegram group 12345
2. Assert: routed to investmentAgent
3. Send message from Slack #ops
4. Assert: routed to devopsAgent
5. Send message from unknown source
6. Assert: routed to defaultAgent
```

### 测试 2：独立的 Memory

```
1. Send "Remember: AAPL target is $200" to investmentAgent via Telegram 12345
2. Send "What is AAPL's target?" to devopsAgent via Slack #ops
3. Assert: devopsAgent does NOT know about the $200 target
4. Send "What is AAPL's target?" to investmentAgent via Telegram 12345
5. Assert: investmentAgent recalls $200
```

### 测试 3：独立的工具集

```
1. investmentAgent has tool: stock_lookup
2. devopsAgent has tool: restart_service
3. Assert: investmentAgent cannot call restart_service
4. Assert: devopsAgent cannot call stock_lookup
```

### 测试 4：会话连续性

```
1. Send message A to Telegram 12345 → investmentAgent
2. Send message B to Telegram 12345 → investmentAgent
3. Assert: message B sees message A in conversation history (same session key)
4. Send message C to Telegram 67890 → investmentAgent
5. Assert: message C does NOT see message A or B (different session key)
```

### 测试 5：优先级解析

```
1. Create binding: { platform: "telegram", chatId: "12345" } → investmentAgent (priority 1)
2. Create binding: { platform: "telegram" } → defaultTelegramAgent (priority 8)
3. Send message from Telegram 12345
4. Assert: routed to investmentAgent, NOT defaultTelegramAgent
5. Send message from Telegram 99999
6. Assert: routed to defaultTelegramAgent (priority 1 did not match, fell through to 8)
```

### 测试 6：线程继承

```
1. Send message to Telegram group 12345 → investmentAgent
2. Reply in a thread within group 12345
3. Assert: thread message routes to investmentAgent (inherited from parent)
4. Assert: thread message gets a separate session key (tg-12345-thread-789, not tg-12345-main)
```

## 各项目差异

只有一种差异：**openclaw 有这个能力；其他项目都没有。**

| 维度 | openclaw | hermes / Claude Code / NemoClaw |
|-----------|----------|-------------------------------|
| **Routing 层** | 9 级优先级 binding | 无 |
| **Agent 实例** | 多个，每个有独立配置 | 单实例 |
| **Memory 隔离** | 每 Agent 独立命名空间 | 单命名空间 |
| **Tools 隔离** | 每 Agent 独立工具集 | 单工具注册表 |
| **多渠道** | 20+ 平台适配器 | 仅 CLI（hermes 有适配器但只有单 Agent） |
| **会话管理** | 按 binding 的 sessionKey | 单会话或按用户 |

这使得专业化成为 Agent 领域中最清晰的架构分叉。每个项目都能做委派（行为 5）——它只是一个创建子 Agent 的工具。但专业化需要单 Agent 项目不会去构建的基础设施：一个 Routing 模块、一个 binding 表、一个实例工厂、命名空间隔离。

**演进路径：**

```
Stage 1: Single Agent, single channel
         (hermes, Claude Code — already powerful)

Stage 2: Single Agent, multiple channels
         (hermes with adapters — same Agent, more entry points)

Stage 3: Multiple Agents, multiple channels
         (openclaw — different specialists for different contexts)
```

如果你在构建个人工具，停在 Stage 1。如果你在构建团队工具，考虑 Stage 2。如果你在构建平台——不同用户群体需要根本不同的 Agent 行为——你需要 Stage 3。

大多数项目永远不需要 Stage 3。但如果你需要，上面展示的 Routing-binding-factory 模式是经过验证的方法。从 2 个 Agent、2 个 binding 开始。验证 Memory 隔离有效。然后扩展到 N 个。
