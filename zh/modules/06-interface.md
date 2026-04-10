# 模块 6：Interface
> 进入你的 Agent 的所有入站路径。用户和外部系统通过这一层与你的 Agent 对话——而且只能通过这一层。

## 它是什么

Interface 层是所有**入站**入口点的集合。用户在终端中输入。Telegram 消息通过 webhook 到达。IDE 通过 MCP 连接。另一个 Agent 通过 ACP 调用你的 Agent。所有这些都是 Interface。

关键区别：**方向决定了它属于哪个模块。**

- MCP **入站**（别人作为 MCP 客户端连接到你的 Agent）= Interface。
- MCP **出站**（你的 Agent 连接到外部 MCP 服务器获取工具）= Tools。
- API **入站**（外部系统调用你的 Agent 的 HTTP 端点）= Interface。
- API **出站**（你的 Agent 调用 LLM 提供商）= Model。

这不是学究式的分类。它决定了代码放在哪里，谁拥有连接生命周期，以及出问题时会发生什么。如果你把入站 Telegram Adapter 放在 Tools 目录，迟早有人会试图把它当作工具来"执行"。如果你把出站 MCP 客户端放在 Interface 目录，有人会试图把它注册为一个频道。

每个 Adapter 做三件事：**协议翻译**（将平台特定的线路格式转换为内部消息格式）、**媒体处理**（图片、文件、语音——每个平台不同）、**生命周期管理**（连接建立、重连、优雅关闭）。就这些。没有业务逻辑。没有路由决策。没有认证。那些属于 Gateway。

## 它解决什么问题

没有 Interface 层，你的 Agent 只能以一种方式与一个东西对话。一开始这没问题。然后你想要 Slack 集成。然后是 Web UI。然后另一个 Agent 需要以编程方式调用你的。

没有干净的抽象，你最终会陷入两种灾难之一：

1. **复制粘贴的 Adapter。** 每个新平台都有一个专门的集成，重复消息解析、错误处理和响应格式化。hermes 通过 `BasePlatformAdapter` 避免了这个问题。想象一下没有它时维护 16 个复制粘贴的集成。

2. **硬编码的入口点。** 你的 Agent Core 直接导入 `readline` 来处理 CLI 输入。添加 Telegram 意味着重写核心。Claude Code 在轻微程度上有这个问题——每个入口点（`cli.tsx`、`mcp.ts`、`bridge`、`daemon`）都是独立的。添加第 5 个意味着从头编写第 5 个。

Adapter 模式解决了这两个问题。定义一个通用接口。每个平台实现它。Core 依赖接口，永远不依赖特定平台。添加新频道意味着编写一个 Adapter 类。Core 零改动。

## 4 个项目是怎么做的

### hermes-agent

**Adapter 继承树。** hermes 定义了 `BasePlatformAdapter` 作为 Python ABC，有 16 个具体子类。

**文件**：`hermes/adapters/base.py` 定义了抽象基类。每个平台有自己的文件：`telegram_adapter.py`、`discord_adapter.py`、`slack_adapter.py`，以及 Mattermost、Matrix、Rocket.Chat、Line、WeChat、WhatsApp、SMS (Twilio)、Email、API Server、Web UI、CLI 和 ACP。

**BasePlatformAdapter 强制要求：**
- `connect()` / `disconnect()` — 生命周期
- `send_message()` — 向用户发送出站响应
- `handle_incoming()` — 入站消息处理
- 平台特定的媒体转换（每个重写处理自己的附件格式）

**Gateway 依赖。** `gateway/run.py`（7620 行）管理所有 Adapter 实例。它启动、停止它们，路由斜杠命令。Adapter 本身保持简单——生命周期复杂性在 Gateway 中。

**ACP Adapter** — hermes 是唯一有 Agent Communication Protocol Adapter 的项目，让其他 Agent 作为对等方调用 hermes。这与 MCP（工具级协议）不同——ACP 是 Agent 对 Agent 的。

**数量**：16 个 Adapter。所有项目中支持最多平台。如果你需要支持多个消息平台，hermes 的模式值得研究。

### openclaw

**Adapter 组合优于继承。** openclaw 采用了不同的方法。不是一个基类配 16 个子类，而是定义了一组能力接口：

- `ChannelPlugin` — 每个频道必须实现的基础（connect、disconnect、handle message）
- `SupportsThreads` — 如果平台有线程（Discord、Slack）
- `SupportsReactions` — 如果平台有表情反应
- `SupportsEditing` — 如果平台允许消息编辑
- `SupportsFiles` — 如果平台支持文件上传

每个频道 Adapter 只实现**它需要的接口**。Telegram 实现 `ChannelPlugin + SupportsFiles + SupportsReactions`。一个简单的 webhook Adapter 只实现 `ChannelPlugin`。

**为什么这很重要。** 在 hermes 的继承模型中，基类必须容纳每个平台的能力。如果你在基类添加一个 `handle_thread_reply()` 方法，每个 Adapter 都必须实现它——即使那些没有线程的平台。openclaw 避免了这个问题。Discord 实现 `SupportsThreads`。SMS 不实现。两者都不需要桩方法。

**数量**：20+ 个频道 Plugin。四个项目中最具可扩展性的架构，因为第三方开发者可以使用发布的 SDK 接口编写频道 Plugin，而不需要触碰 openclaw 的核心。

**内置 WebChat。** openclaw 将基于浏览器的聊天 UI 作为其频道 Plugin 之一。这意味着 Agent 自带 Web 界面——不需要单独的前端项目。

### NemoClaw

**仅管理 CLI。** NemoClaw 是一个沙箱编排器，不是面向用户的 Agent。它唯一的接口是给运维人员的管理 CLI：

- `nemo setup` — 初始化沙箱环境
- `nemo deploy` — 将 OpenClaw 实例部署到沙箱
- `nemo status` — 检查沙箱健康状态

没有面向用户的频道。没有消息平台 Adapter。没有 Web UI。NemoClaw 将所有用户交互委托给在其沙箱内运行的 OpenClaw 实例。启示：不是每个组件都需要丰富的 Interface 层。如果你的工作是基础设施，一个给运维人员的 CLI 就够了。

### Claude Code

**多个独立的入口点，没有统一抽象。**

Claude Code 有四种不同的启动方式：

| 入口点 | 文件 | 用途 |
|-------------|------|---------|
| CLI REPL | `src/entrypoints/cli.tsx` | 交互式终端会话 |
| MCP Server | `src/entrypoints/mcp.ts` | IDE 作为 MCP 服务器连接到 Claude Code |
| HTTP Bridge | `src/entrypoints/bridge.ts` | 编程式 HTTP API 访问 |
| Background Daemon | `src/entrypoints/daemon.ts` | 长时间运行的后台操作 |

**没有共享基类。** 每个入口点都是独立模块。它们最终都调用到 `query.ts`（模块 1 中的内循环），但各自独立处理连接建立、输入解析和输出格式化。

**为什么不做抽象？** 性能。每个入口点都针对其特定场景优化。`cli.tsx` 使用 React (Ink) 进行终端渲染。`mcp.ts` 处理 MCP 协议的 JSON-RPC 帧。`bridge.ts` 处理 HTTP 请求/响应。强制通过通用抽象会增加开销，降低每个入口点为其场景优化的能力。

**代价是真实的。** 添加第 5 个入口点意味着从头编写。没有模板，没有基类，没有"实现这 3 个方法就行了"。对于由一家公司（Anthropic）维护的产品，这可以接受。对于期望社区贡献的开源项目，这就不行了。

**毫秒级优化。** Claude Code 的入口点针对启动时间进行了调优。CLI 路径避免导入不需要的模块。MCP 路径完全跳过 UI 初始化。这种级别的优化只有在每个入口点控制自己的导入图谱时才可能。

## 对比表

| 维度 | hermes | openclaw | NemoClaw | Claude Code |
|-----------|--------|----------|----------|-------------|
| 模式 | 继承（ABC + 16 子类） | 组合（能力接口） | 仅管理 CLI | 独立入口点，无抽象 |
| 频道数量 | 16 | 20+ | 1（运维 CLI） | 4 个入口点 |
| 添加新频道 | 子类化 `BasePlatformAdapter`，实现必需方法 | 实现 `ChannelPlugin` + 所需能力接口 | N/A | 从头编写 |
| 生命周期管理 | `gateway/run.py` 管理所有 Adapter | Plugin 生命周期 Hook | 手动 CLI 调用 | 每个入口点自行管理 |
| 第三方可扩展性 | 否（仅内部 Adapter） | 是（发布的 Plugin SDK） | 否 | 否 |
| 媒体处理 | 每个 Adapter，继承的辅助方法 | 每个 Adapter，接口驱动 | N/A | 每个入口点独立 |
| Agent 对 Agent 协议 | ACP Adapter | 频道可以是另一个 Agent | N/A | MCP server 模式 |
| 启动优化 | 标准 Python import | 标准 ESM import | N/A | 每个入口点 tree-shaking |

## 最佳实践

**1. 从 CLI 开始。永远如此。**
每个项目都从 CLI 开始。hermes 有一个。openclaw 有一个。Claude Code 的主要模式是 CLI。即使 NemoClaw 也只是一个 CLI。原因：CLI 是开发期间最快的反馈循环。不需要 webhook 设置、OAuth token、平台特定的消息格式。`stdin` 进，`stdout` 出。之后再添加消息平台，当你有了一个能工作的 Agent 之后。

**2. 从第一天就使用 Adapter 模式——但保持简单。**
你不需要 openclaw 的完整组合系统或 hermes 的 16 个 Adapter。你需要一个接口加三个方法：

```python
class ChannelAdapter(ABC):
    async def start(self) -> None: ...           # connect / listen
    async def stop(self) -> None: ...            # graceful shutdown
    async def send(self, response: AgentResponse) -> None: ...  # send reply
    # Inbound messages arrive via a callback registered during start()
```

你的 CLI Adapter 实现这个。当你添加 Telegram 时，它也实现这个。Core 永远不知道也不关心它在和哪个平台对话。

**3. 保持 Adapter 轻薄。**
Adapter 做协议翻译，别的什么都不做。没有认证逻辑（那是 Gateway 的事）。没有消息路由（那是 Routing 的事）。没有业务规则。如果你的 Telegram Adapter 有 `if user.is_admin:` 检查，那个逻辑应该放在别处。

hermes 的各个 Adapter 保持轻薄，因为 Gateway 处理复杂性。openclaw 的 Adapter 保持轻薄，因为能力接口限制了范围。Claude Code 的入口点不轻薄——它们各自包含自己的设置逻辑，这是跳过共享抽象的代价。

**4. 在项目早期就区分入站和出站。**
当你添加 MCP 支持时，立即决定：这是 MCP-server 模式（入站——别人连接到你的 Agent）还是 MCP-client 模式（出站——你的 Agent 连接到外部工具服务器）？答案决定哪个模块拥有代码。在一个模块中混合方向会造成随时间累积的混乱。

Claude Code 做得很清楚：`entrypoints/mcp.ts` 是入站（Interface），而连接到外部服务器的 MCP 客户端连接是出站（Tools）。hermes 的 ACP Adapter 是入站；其工具调用能力是出站。

**5. 为平台特定能力做规划，但不要过度抽象。**
Discord 有线程、反应和斜杠命令。Telegram 有内联键盘和消息编辑。SMS 没有这些。openclaw 的组合模型优雅地处理了这个——每个 Adapter 声明它支持什么。hermes 的继承模型通过可选的方法重写来处理。两者都有效。不有效的是：一个试图预先枚举所有可能的平台能力的基类。你永远列不完那个清单。

## 分步构建指南

### 步骤 1：定义消息契约

在编写任何 Adapter 之前，定义你的 Agent 内部消息是什么样的。

```python
@dataclass
class InboundMessage:
    text: str
    sender_id: str
    channel: str              # "cli", "telegram", "discord", etc.
    thread_id: str | None     # if the platform supports threads
    attachments: list[Attachment]  # images, files
    raw: dict                 # platform-specific data (for edge cases)

@dataclass
class AgentResponse:
    text: str
    attachments: list[Attachment]
    metadata: dict            # e.g., {"reply_to": message_id}
```

这是 Interface 和 Agent Core（或 Gateway，如果有的话）之间的契约。每个 Adapter 从其平台格式转换为 `InboundMessage`，并将 `AgentResponse` 转换回其平台格式。

**决策点**：是否包含 `raw`？包含它可以让 Adapter 透传平台特定数据，而契约不需要知道每个平台的特殊情况。hermes 和 openclaw 都这样做。这防止了每次添加平台时契约不断膨胀。

### 步骤 2：构建 CLI Adapter

```python
class CLIAdapter(ChannelAdapter):
    def __init__(self, on_message: Callable[[InboundMessage], Awaitable[None]]):
        self.on_message = on_message

    async def start(self):
        while True:
            user_input = await asyncio.to_thread(input, "> ")
            if user_input.strip().lower() in ("exit", "quit"):
                break
            msg = InboundMessage(
                text=user_input,
                sender_id="local_user",
                channel="cli",
                thread_id=None,
                attachments=[],
                raw={},
            )
            await self.on_message(msg)

    async def stop(self):
        pass  # nothing to clean up

    async def send(self, response: AgentResponse):
        print(response.text)
```

这就是你 v1 的全部 Interface 层。发布它。使用它。构建 Agent 的其余部分。当你需要 Telegram 时再回来。

### 步骤 3：添加第二个 Adapter（需要时）

选择你的用户实际使用的平台。实现同样的 `ChannelAdapter` 接口。

```python
class TelegramAdapter(ChannelAdapter):
    def __init__(self, bot_token: str, on_message: Callable):
        self.bot_token = bot_token
        self.on_message = on_message
        self.bot = None

    async def start(self):
        self.bot = TelegramBot(self.bot_token)
        self.bot.on("message", self._handle)
        await self.bot.start_polling()

    async def stop(self):
        await self.bot.stop()

    async def send(self, response: AgentResponse):
        await self.bot.send_message(
            chat_id=response.metadata["chat_id"],
            text=response.text,
        )

    async def _handle(self, update):
        msg = InboundMessage(
            text=update.message.text,
            sender_id=str(update.message.from_user.id),
            channel="telegram",
            thread_id=str(update.message.chat.id),
            attachments=self._parse_attachments(update),
            raw=update.to_dict(),
        )
        await self.on_message(msg)
```

**决策点**：谁管理 Adapter 生命周期——Gateway 还是主进程？如果你只有 2-3 个 Adapter，主进程可以直接启动/停止它们。如果有 10+，你需要 hermes 的 Gateway 模式：一个中央管理器来启动、停止、健康检查和重启 Adapter。

### 步骤 4：添加 Adapter 注册（可选）

当你有 3+ 个 Adapter 时，你需要一个注册表，让配置驱动哪些 Adapter 是活跃的。

```python
class AdapterRegistry:
    def __init__(self):
        self._factories: dict[str, Callable] = {}

    def register(self, channel: str, factory: Callable):
        self._factories[channel] = factory

    def create(self, channel: str, config: dict, on_message: Callable) -> ChannelAdapter:
        if channel not in self._factories:
            raise ValueError(f"Unknown channel: {channel}")
        return self._factories[channel](config=config, on_message=on_message)

# Usage:
registry = AdapterRegistry()
registry.register("cli", CLIAdapter)
registry.register("telegram", TelegramAdapter)
registry.register("discord", DiscordAdapter)

# Start only configured adapters
for channel_config in config["channels"]:
    adapter = registry.create(channel_config["type"], channel_config, handle_message)
    await adapter.start()
```

这本质上就是 hermes 的 Gateway 为其 16 个 Adapter 所做的事，减去斜杠命令分发和健康监控。

### 步骤 5（可选）：添加能力接口

如果你的平台有不同的功能，而你想要类型安全的能力检查：

```python
class SupportsThreads(ABC):
    async def reply_in_thread(self, thread_id: str, response: AgentResponse): ...

class SupportsReactions(ABC):
    async def add_reaction(self, message_id: str, emoji: str): ...

class SupportsEditing(ABC):
    async def edit_message(self, message_id: str, new_text: str): ...

# Discord adapter:
class DiscordAdapter(ChannelAdapter, SupportsThreads, SupportsReactions, SupportsEditing):
    ...

# Check at runtime:
if isinstance(adapter, SupportsThreads):
    await adapter.reply_in_thread(thread_id, response)
else:
    await adapter.send(response)  # fallback
```

这是 openclaw 的模型。在你有 3+ 个具有真正不同能力的平台之前，不要添加这个。对于 CLI + 一个消息平台，基础 `ChannelAdapter` 就够了。

## 常见陷阱

**1. 在 Adapter 中放认证逻辑。**
你的 Telegram Adapter 不应该验证 API key 或检查用户权限。它接收消息，转换为 `InboundMessage`，然后传递出去。认证属于 Gateway（模块 7）。在 Adapter 中混入认证意味着每个新 Adapter 都必须重新实现你的认证逻辑——而且它们会做得不一致。

**2. 为没有用户的平台构建 Adapter。**
hermes 有 16 个 Adapter。大多数项目只需要 2-3 个。先构建 CLI。当你有真实用户请求时再添加一个消息平台。每个 Adapter 都是一个维护承诺：API 变更、库更新、消息格式化的边缘情况。不要维护没人用的 Adapter。

**3. 因为"我们只需要 CLI"就跳过抽象。**
你今天只需要 CLI。Adapter 接口的定义几乎没有成本（一个带三个方法的 ABC）。后来改造意味着重写 Core 接收消息的方式。现在定义接口。只实现 CLI。3 个月后感谢自己。

**4. 包含业务逻辑的臃肿 Adapter。**
如果你的 Discord Adapter 有 500 行代码，说明有问题。Adapter 应该是 50-150 行。协议翻译、媒体转换、生命周期管理。就这些。如果你在 Adapter 里做消息路由、用户偏好查询或响应格式化，把它们提取出来。

**5. 把 MCP 入站和 MCP 出站当作同一个模块。**
Claude Code 做对了：`entrypoints/mcp.ts`（入站，Interface 层）与 MCP 客户端连接（出站，Tools 层）完全分离。它们共享 MCP 协议，但数据流方向相反、错误处理不同、生命周期关注点不同。把它们放在不同的模块中。

## 跨模块契约

### Interface 对下游模块的期望

| 下游模块 | 契约 | 格式 |
|-------------------|----------|--------|
| **Gateway**（如果存在） | 接受 `InboundMessage`，返回路由决策 | Gateway 接收消息；Interface 不关心之后发生什么 |
| **Agent Core**（如果没有 Gateway） | 接受 `InboundMessage`，返回 `AgentResponse` | 单用户 CLI 场景下跳过 Gateway 的直接交接 |

### Interface 提供给下游模块的

| 消费者 | Interface 提供什么 | 格式 |
|----------|------------------------|--------|
| **Gateway / Agent Core** | 标准化的入站消息 | `InboundMessage{text, sender_id, channel, thread_id, attachments, raw}` |
| **Gateway / Agent Core** | 响应交付回调 | `adapter.send(AgentResponse)` — Core 或 Gateway 调用此方法将回复发回给用户 |

### Interface 不做什么

- **认证。** Interface 不验证 token 或检查权限。它透传 `sender_id`。Gateway 决定该发送者是否被允许。
- **路由。** Interface 不决定哪个 Agent 处理消息。它透传 `channel` 和 `thread_id`。Routing（模块 7）决定目的地。
- **响应格式化。** Interface 将 `AgentResponse.text` 转换为平台格式（如 Markdown 到 Telegram HTML）。它不决定响应*说什么*。
- **消息存储。** Interface 不持久化消息。Session Store（由 Agent Core 拥有）处理这个。

### 不变量

- 每个入站路径产生相同的 `InboundMessage` 格式。Core 和 Gateway 永远不需要知道消息来自哪个平台（除非它们显式检查 `channel` 字段来处理平台特定行为）。
- 当 Gateway 存在时，Adapter 永远不直接调用 Agent Core。流程始终是 Interface -> Gateway -> Core（或如果没有 Gateway 则 Interface -> Core）。
- Adapter 的生命周期（start/stop）由主进程或 Gateway 管理，永远不由 Agent Core 管理。
- 添加新 Adapter 不需要对 Agent Core、Gateway 或任何其他模块做任何改动。只有 Adapter 注册表的配置需要改变。
