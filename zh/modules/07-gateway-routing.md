# 模块 7：Gateway + Routing
> 可选的门卫和调度员。Gateway 问"这个请求能进来吗？"Routing 问"谁来处理它？"你的 v1 单用户 CLI 两者都可以跳过。

## 它是什么

两个不同的关注点打包在一个模块中，因为它们处于相同的架构位置——在 Interface 和 Agent Core 之间——而且大多数需要其中一个的项目也需要另一个。

**Gateway** 是门卫。每个请求都经过它。它按顺序回答三个问题：
1. **认证（Authentication）** — 这个请求来自合法来源吗？
2. **授权（Authorization）** — 这个来源有权限做它请求的事吗？
3. **限流（Rate limiting）** — 这个来源超过配额了吗？

如果三个都通过，Gateway 执行**方法分发（method dispatch）**——它查看请求类型（`chat.send`、`config.update`、`agent.status`）并路由到正确的处理函数。这在概念上等同于 Express.js 查看 URL 路径并调用匹配的控制器。唯一的区别：大多数 Agent Gateway 使用带 `method` 字段的 WebSocket 而不是带 URL 路径的 HTTP。

**Routing** 是消息调度器。它回答一个问题：**"哪个 Agent 实例应该处理这条消息？"** 这只在你有多个 Agent 时才重要。单 Agent 系统不需要 Routing——只有一个目的地。

Routing 只对会话消息触发，不是对所有请求。配置更新、状态检查、管理命令——这些通过 Gateway 的方法分发处理，完全跳过 Routing。

**关键区别：**

| | Gateway | Routing |
|---|---|---|
| 触发条件 | 每个请求 | 仅会话消息 |
| 提出的问题 | "这个能进来吗？" | "这个去哪？" |
| 何时需要 | 多用户或多频道 | 仅多 Agent |
| 单用户 CLI | 不需要 | 不需要 |
| 多用户，单 Agent | 需要（认证 + 限流） | 不需要 |
| 多用户，多 Agent | 需要 | 需要 |

Claude Code 两者都不需要。它在本地运行，一个用户，一个 Agent。整个模块在其代码库中不存在。

## 它解决什么问题

**没有 Gateway：**
- 任何知道你 WebSocket 端点的人都能发送命令。没有认证。
- 失控的客户端可以用请求淹没你的 Agent。没有限流。
- 每个 Adapter 必须实现自己的认证检查。执行不一致。
- 方法分发逻辑分散在各 Adapter 中，更糟糕的是在 Agent Core 内部。

**没有 Routing（在多 Agent 系统中）：**
- 消息发送到硬编码的 Agent。添加第二个 Agent 意味着重写分发逻辑。
- 线程连续性断裂。Discord 线程中的回复发送到与原始消息不同的 Agent。
- 没有回退链。如果没有特定的绑定，消息无处可去。

openclaw 是在实践中学到这些教训后构建了四个项目中最完整的 Gateway + Routing 系统。hermes 通过其 7620 行的 `gateway/run.py` 部分解决了 Gateway，但因为运行的是单个 Agent 所以从不需要 Routing。

## 4 个项目是怎么做的

### openclaw

**Gateway：带有 25+ 处理器组的 WebSocket 控制面板。**

openclaw 的 Gateway 是一个 WebSocket 服务器。客户端（Web UI、CLI、外部集成）通过 WebSocket 连接并发送带 `method` 字段的 JSON 消息。Gateway 将每个方法分发到其处理器组。

处理器组包括：`chat.*`（发送消息、编辑、删除）、`config.*`（读取、更新、应用）、`agent.*`（状态、重启、列表）、`channel.*`（连接、断开、健康检查）、`memory.*`（搜索、检查）、`admin.*`（用户管理、权限），等等。共 25+ 组。

**认证模型**：基于角色，带范围限制。
- 角色：`owner`、`admin`、`user`、`viewer`、`api_key`
- 每个处理器声明其所需角色和范围
- Gateway 在分发前检查连接的认证 token 是否满足处理器的要求

**限流**：基于预算，不是简单的请求计数。
- 每个用户获得一个 token 预算（如每天 100k token）
- Gateway 在转发给 Agent 前扣除预估 token
- 响应后进行实际使用量核算
- 每个角色有不同限制（owner 无限，api_key 每天 10k）

**WebSocket 的选择很重要。** HTTP 对于请求/响应可以工作，但 Agent 对话本质上是流式的。用户发送消息，响应在几秒钟内以 token 流的形式到达。WebSocket 是自然的选择。openclaw 选择正确。

**Routing：9 级绑定优先级，独立模块。**

这是四个项目中最复杂的路由系统。当消息从任何频道到达时，Routing 按以下顺序检查绑定：

```
Priority 1: Exact chat/group match     → "Telegram group 12345 → investmentAgent"
Priority 2: Parent thread inheritance   → reply in a thread goes to same agent as parent
Priority 3: Wildcard match             → "All Telegram groups → generalAgent"
Priority 4: Discord server + role      → "Server X, role Y → specialistAgent"
Priority 5: Discord server             → "Server X → serverAgent"
Priority 6: Slack team                 → "Team Z → teamAgent"
Priority 7: Account-level              → "Telegram account → accountAgent"
Priority 8: Channel-level              → "All Discord messages → discordAgent"
Priority 9: Default fallback           → the catch-all agent
```

第一个匹配胜出。这意味着你可以设置宽泛的默认值（Priority 9）并用特定绑定（Priority 1）覆盖它们，不会冲突。

**架构洞察：** 在 openclaw 中，Routing 是一个独立模块（`src/routing/`），由频道层调用，不是由 Gateway 调用。Gateway 处理认证和限流。频道处理协议翻译。Routing 位于频道和 Agent Core 之间。这种三方拆分——Gateway 守门、频道翻译协议、Routing 选择目的地——比把所有东西都放在 Gateway 里更干净。

### hermes-agent

**Gateway：`gateway/run.py` — 7620 行，管理 Adapter 生命周期 + 斜杠命令分发。**

hermes 的 Gateway 不是独立服务。它是启动所有已配置 Adapter 并分发入站消息的主进程。

**它做什么：**
- 启动/停止 Adapter 实例（模块 6 中的 16 个平台 Adapter）
- 注册和分发斜杠命令（`/help`、`/status`、`/skills`、自定义命令）
- 管理 Adapter 健康循环（重启崩溃的 Adapter）
- 将消息从 Adapter 路由到单个 `AIAgent` 实例

**它不做什么：**
- 正式认证（信任平台级认证——如果 Telegram 递送了消息，用户就是有效的）
- 基于角色的授权（没有角色系统）
- 基于 token 预算的限流（没有限流）

**为什么 7620 行？** 因为 hermes 的 Gateway 混合了关注点。Adapter 生命周期管理、斜杠命令注册、健康监控和消息分发都在一个文件中。openclaw 将这些拆分到不同的模块。

**Routing：没有。** hermes 是单 Agent 系统。每条消息都发送到唯一的 `AIAgent` 实例。没有绑定表，没有优先级链，没有 Agent 选择逻辑。

**CLI 绕过：** 在 CLI 模式下运行时，hermes 完全跳过 Gateway。CLI Adapter 直接与 `AIAgent` 对话。这是正确的选择——单个用户在终端中输入不需要认证、限流或方法分发。

### NemoClaw

**在传统意义上不适用。** NemoClaw 包装的是 OpenShell 沙箱管理，不是请求处理。它确实有一个 `gateway-state.ts` 来跟踪在沙箱*内部*运行的 OpenClaw Gateway 的健康状态，但这是基础设施监控，不是请求 Gateway。

值得注意的一点：NemoClaw 的 `gateway-state.ts` 将沙箱 Gateway 分类为健康状态（healthy、degraded、dead）并触发恢复操作（SSH 探针、curl 健康检查、进程重启）。这个模式值得在模块 10（运行时/基础设施）中注意，而不是在本模块。

### Claude Code

**不存在。** Claude Code 作为单用户 CLI Agent 在本地运行。没有 Gateway，没有认证层，没有限流器，没有方法分发，没有 Routing。

这不是缺陷——这是刻意的架构选择。当你只有一个用户在本地运行进程时，每个 Gateway 功能都是零收益的开销：
- 认证？用户启动了进程。他们凭借作为操作系统用户而被认证。
- 授权？单用户，单 Agent，完全权限。
- 限流？用户被他们的打字速度限流。
- Routing？一个 Agent。无需路由。

Claude Code 确实有权限系统（工具执行的 ask/auto 模式），但这是 Security Envelope（模块 0），不是 Gateway。权限检查发生在工具执行层，不是请求入口点。

**启示：** 不要因为觉得应该有就构建 Gateway。当你有多个用户或多个需要认证的频道时再构建。对于 v1 单用户 CLI，完全跳过它。

## 对比表

| 维度 | hermes | openclaw | NemoClaw | Claude Code |
|-----------|--------|----------|----------|-------------|
| Gateway 是否存在 | 是（混在 `gateway/run.py` 中） | 是（独立的 WebSocket 控制面板） | N/A（仅基础设施健康监控） | 否 |
| 协议 | 每个 Adapter 特定（各平台原生） | 带 JSON 方法分发的 WebSocket | N/A | N/A |
| 认证 | 仅信任平台 | 角色 + 范围 + token | N/A | N/A（操作系统用户 = 已认证） |
| 授权 | 无（无角色） | 5 个角色，每个处理器范围检查 | N/A | N/A |
| 限流 | 无 | 基于 token 预算 + 角色配额 | N/A | N/A |
| 方法分发 | 仅斜杠命令 | 25+ 处理器组 | N/A | N/A |
| Routing 是否存在 | 否（单 Agent） | 是（独立的 `src/routing/` 模块） | 否 | 否 |
| Routing 复杂度 | N/A | 9 级绑定优先级 | N/A | N/A |
| Gateway 文件大小 | 7620 行（混合关注点） | 分布在多个模块 | N/A | N/A |
| CLI 是否绕过 Gateway | 是 | 是（可用直连模式） | N/A | N/A（没有 Gateway 可绕过） |

## 最佳实践

**1. v1 单用户 CLI 完全跳过 Gateway。**
Claude Code 没有 Gateway 就发布了。hermes 的 CLI 模式绕过了它。当一个人在终端中输入时，你不需要认证、限流或方法分发。先构建 Agent。当你添加第二个用户或第二个频道时再添加 Gateway。

**2. 添加 Gateway 时，从认证 + 限流开始。**
不要在第一天就构建 25 个处理器组。从以下开始：
- 一个认证检查（API key 或 session token）
- 一个限流器（每分钟请求数，不是 token 预算——那个以后再说）
- 其他一切直接传递给 Agent Core

你可以逐步添加方法分发、基于角色的授权和 token 预算。

**3. Gateway 和 Routing 是独立模块。**
openclaw 证明了这一点。Gateway 处理认证和限流（每个请求）。Routing 处理 Agent 选择（仅消息，仅多 Agent）。合并它们意味着你的限流器知道 Agent 绑定，你的路由逻辑知道 API key。两者都不应该如此。

实际上，调用流程是：Interface -> Gateway（认证 + 限流）-> Routing（Agent 选择）-> Agent Core。如果你只有一个 Agent，去掉 Routing，流程变成：Interface -> Gateway -> Agent Core。

**4. 在你有 2+ 个 Agent 之前不要构建 Routing。**
Routing 的唯一目的是回答"哪个 Agent 处理这个？"如果你只有一个 Agent，答案永远一样。openclaw 需要 9 级优先级因为它为不同的 Discord 服务器、Slack 团队和 Telegram 群组运行不同的 Agent。如果你不需要这样做，你就不需要 Routing。

**5. 尽可能让平台处理自己的认证。**
hermes 信任平台级认证。如果 Telegram 将消息递送到你的 bot webhook，Telegram 已经验证了发送者。如果 Discord 发送了 Gateway 事件，Discord 已经认证了连接。你的 Gateway 只需要为不提供认证的频道添加认证（原始 HTTP API、来自自定义客户端的 WebSocket 连接）。

**6. 限流应该按用户，而不是按请求。**
openclaw 的 token 预算方法比简单的请求计数更好，因为：
- 一个复杂请求（"分析这份 50 页的文档"）成本是一个简单请求（"现在几点"）的 100 倍
- 请求计数惩罚频繁聊天的用户，却放过昂贵的单次请求
- Token 预算将成本与实际资源消耗对齐

v1 先用每分钟请求数。当你有使用数据后再转向 token 预算。

## 分步构建指南

### 步骤 1：决定你是否需要 Gateway

回答这些问题：
1. 你有超过一个用户吗？如果否 → 跳过 Gateway。
2. 外部系统通过 API 调用你的 Agent 吗？如果否 → 跳过 Gateway。
3. 你有多个频道（Telegram + Discord + Web）吗？如果是 → 你可能需要 Gateway 来保证一致的认证。

如果三个都回答"否"，完全跳过本模块。Interface 直接与 Agent Core 对话。当答案改变时再回来。

### 步骤 2：添加最小 Gateway（认证 + 透传）

```python
class Gateway:
    def __init__(self, agent_core, auth_provider, rate_limiter):
        self.core = agent_core
        self.auth = auth_provider
        self.rate = rate_limiter

    async def handle(self, request: InboundMessage, credentials: dict) -> AgentResponse:
        # 1. Authenticate
        user = self.auth.verify(credentials)
        if not user:
            raise AuthError("Invalid credentials")

        # 2. Rate limit
        if not self.rate.allow(user.id):
            raise RateLimitError(f"Rate limit exceeded for {user.id}")

        # 3. Pass through to Agent Core
        return await self.core.handle(request)
```

三个检查，一个透传。这就是你 v1 的全部 Gateway。没有方法分发。没有角色。没有 token 预算。

### 步骤 3：添加方法分发（当你需要管理命令时）

当你想支持聊天之外的命令（状态检查、配置更新、Agent 重启）时，添加方法分发。

```python
class Gateway:
    def __init__(self, ...):
        self.handlers: dict[str, Callable] = {
            "chat.send": self._handle_chat,
            "agent.status": self._handle_status,
            "config.get": self._handle_config_get,
            "config.update": self._handle_config_update,
        }

    async def handle(self, method: str, payload: dict, credentials: dict) -> Any:
        user = self.auth.verify(credentials)
        if not user:
            raise AuthError("Invalid credentials")

        handler = self.handlers.get(method)
        if not handler:
            raise MethodNotFoundError(f"Unknown method: {method}")

        if not self.rate.allow(user.id, method):
            raise RateLimitError("Rate limit exceeded")

        return await handler(payload, user)
```

**决策点**：HTTP 还是 WebSocket？如果你的 Agent 只做请求/响应（用户发消息，Agent 回复一次），HTTP 更简单。如果你的 Agent 逐 token 流式返回响应，WebSocket 是必要的。大多数生产 Agent 是流式的，所以 WebSocket 是务实的默认选择。

### 步骤 4：添加基于角色的授权（当你需要时）

```python
# Define roles and their allowed methods
ROLE_PERMISSIONS = {
    "owner": {"*"},                          # everything
    "admin": {"chat.*", "config.get", "agent.status"},
    "user": {"chat.send", "agent.status"},
    "viewer": {"agent.status"},
}

class Gateway:
    async def handle(self, method: str, payload: dict, credentials: dict) -> Any:
        user = self.auth.verify(credentials)
        if not user:
            raise AuthError("Invalid credentials")

        # Check role permissions
        if not self._has_permission(user.role, method):
            raise ForbiddenError(f"Role '{user.role}' cannot call '{method}'")

        # ... rest of dispatch
```

openclaw 有 5 个角色，带每个处理器的范围检查。从 3 个角色开始（`owner`、`user`、`viewer`），当你遇到真正的权限冲突时再添加粒度。

### 步骤 5：添加 Routing（仅用于多 Agent）

当且仅当你有多个 Agent 实例时：

```python
class Router:
    def __init__(self):
        self.bindings: list[Binding] = []  # ordered by priority

    def resolve(self, message: InboundMessage) -> str:
        """Returns the agent_id that should handle this message."""
        for binding in self.bindings:
            if binding.matches(message):
                return binding.agent_id
        return self.default_agent_id

@dataclass
class Binding:
    priority: int
    agent_id: str
    channel: str | None       # match specific channel
    chat_id: str | None       # match specific chat/group
    thread_parent: str | None # inherit from parent thread
    role: str | None          # match platform role (Discord)

    def matches(self, message: InboundMessage) -> bool:
        if self.chat_id and message.thread_id == self.chat_id:
            return True
        if self.channel and message.channel == self.channel:
            return True
        # ... more match conditions
        return False
```

**决策点**：多少优先级层级？openclaw 有 9 个。从 3 个开始：
1. 精确 chat/group 匹配
2. 频道级匹配
3. 默认回退

当你遇到真正的路由冲突时再添加层级。不要因为 openclaw 有 9 级就预先构建 9 级。

### 步骤 6：串联完整流程

```python
# Full flow with Gateway + Routing:
async def on_message(message: InboundMessage, credentials: dict):
    # Gateway: auth + rate limit
    user = gateway.authenticate(credentials)
    gateway.rate_limit(user)

    # Routing: which agent?
    agent_id = router.resolve(message)
    agent = agent_registry.get(agent_id)

    # Agent Core: handle the message
    response = await agent.handle(message)

    # Interface: send response back
    adapter = adapter_registry.get(message.channel)
    await adapter.send(response)

# Without Gateway or Routing (single-user CLI):
async def on_message(message: InboundMessage):
    response = await agent_core.handle(message)
    await cli_adapter.send(response)
```

第二个版本是你的起点。第一个版本是你的终点——如果你需要的话。

## 常见陷阱

**1. 为单用户 CLI 构建 Gateway。**
最常见的过度工程错误。你读了 openclaw 的 25+ 处理器组然后觉得你也需要。你不需要。Claude Code——一个被数千人使用的生产 Agent——没有 Gateway。从没有开始。当你有它解决的问题时再添加。

**2. 把 Routing 放在 Gateway 里。**
Gateway 问"这个能进来吗？"（认证、限流、方法分发）。Routing 问"它去哪？"（Agent 选择）。这是不同的关注点，有不同的触发条件（每个请求 vs 仅消息）。openclaw 将它们保持在独立模块（`src/gateway/` vs `src/routing/`）。如果你合并它们，你的认证中间件会开始知道 Agent 绑定，重构任何一个都需要碰另一个。

**3. 重新实现平台认证。**
Discord 在将事件递送给你的 bot 之前验证了用户身份。Telegram 验证 webhook 签名。Slack 发送签名请求。如果你重新验证平台已经认证的用户，你是在做双重工作并维护重复的认证逻辑。对平台频道信任平台认证。只对不提供认证的频道（原始 HTTP API、来自自定义客户端的 WebSocket 连接）添加你自己的认证。

**4. 没有优先级的扁平路由。**
如果你将绑定存储为扁平映射（`chat_id -> agent_id`），你就无法有回退链。来自未映射 chat 的消息无处可去。openclaw 的优先级绑定列表意味着每条消息最终都会匹配*某些东西*——即使只是 Priority 9 的默认值。始终有一个默认回退。

**5. 按请求数而不是按成本限流。**
"每小时 100 个请求"将一个一词问候和分析 200 页 PDF 的请求同等对待。发送 100 个问候的用户被阻止。发送 1 个昂贵请求的用户花费你 100 倍却在限制以内。openclaw 的 token 预算模型避免了这个问题。即使是粗略的估计（短消息 = 100 token，长消息 = 1000 token）也比扁平计数好。

**6. 在有 2 个 Agent 之前就添加 Routing。**
当你只有一个 Agent 时，Routing 是零价值模块。它添加了代码、配置和一个故障模式（如果绑定配置错误怎么办？）却没有收益。消息就是发送到唯一存在的 Agent。等到你真正部署第二个 Agent 实例时再说。

## 跨模块契约

### Gateway 对其他模块的期望

| 模块 | 契约 | 格式 |
|--------|----------|--------|
| **Interface** | 递送带凭据的入站请求 | `InboundMessage` + 认证 token/凭据 |
| **Auth Provider** | 验证凭据，返回用户信息 | `verify(credentials) -> User{id, role, scopes}` |
| **Rate Limiter** | 检查并扣减配额 | `allow(user_id, method?) -> bool` |

### Gateway 提供给其他模块的

| 消费者 | Gateway 提供什么 | 格式 |
|----------|----------------------|--------|
| **Routing**（如果存在） | 已认证、已限流的消息 | 带已验证 `user_id` 的 `InboundMessage` |
| **Agent Core**（如果没有 Routing） | 已认证、已限流的消息 | 与模块 6 相同的 `InboundMessage` 格式 |
| **方法处理器** | 非聊天方法的已分发请求 | 处理器特定的载荷 |

### Routing 对其他模块的期望

| 模块 | 契约 | 格式 |
|--------|----------|--------|
| **Gateway 或 Interface** | 已认证的入站消息 | 带 `channel`、`thread_id`、`sender_id` 的 `InboundMessage` |
| **Binding Store** | 有序的绑定列表 | `Binding{priority, agent_id, match_conditions}` |

### Routing 提供给其他模块的

| 消费者 | Routing 提供什么 | 格式 |
|----------|----------------------|--------|
| **Agent Core** | 已解析的目的地 + 原始消息 | `(agent_id, InboundMessage)` |

### 不变量

- Gateway 在每个请求上运行。没有例外。如果 Gateway 存在，没有东西可以绕过它（除了开发模式下的 CLI，如 hermes 所做的那样）。
- Routing 只在会话消息上运行。配置更新、状态检查和管理命令由 Gateway 的方法分发处理，不由 Routing 处理。
- Routing 永远不修改消息。它读取 `channel`、`thread_id` 和平台元数据来选择 Agent。消息到达 Agent Core 时保持不变。
- Gateway 永远不选择 Agent。它的工作在认证 + 限流 + 方法分发之后结束。Agent 选择完全是 Routing 的职责。
- 单 Agent 系统没有 Routing 模块。单本地用户系统没有 Gateway 模块。两者都是增量的——它们的缺席不需要对 Interface 或 Agent Core 做任何改动。
- 绑定列表始终在最低优先级有一个默认回退。没有消息应该因为"没有匹配的绑定"而路由失败。
