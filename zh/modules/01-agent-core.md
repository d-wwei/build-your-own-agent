# 模块 1：Agent Core
> 协调一切的中枢。没有它，Model、Tools、Memory 和 Context Manager 不过是放在架子上的一堆库。

## 它是什么

Agent Core 是 Hub-and-Spoke 架构的中心。它不"管理"其他模块——它**协调**它们。可以把它想象成坐在会议室中央的项目经理：左手找设计师要视觉稿，右手找工程师要构建产物，前台找分析师要数据。谁先完成谁先交付，项目经理负责把结果拼到一起。

你构建的每个 Agent 在核心处都是同一个循环：`while (budget > 0): 构建上下文 -> 调用模型 -> 解析响应 -> 执行工具 -> 重复`。四个项目在这个模式上完全一致，区别在于如何让这个循环保持整洁。

Agent Core 直接拥有两样东西：Session Store（对话历史，通常用 SQLite 或 JSONL）和 Config Store（Agent 配置，YAML 或 JSON）。其余一切——模型调用、工具执行、Memory 读取、上下文压缩——都委派给对等服务。一旦你让 Core 自己*做*这些事而不是*委派*它们，你就走上了通往 9431 行 god class 的不归路。

## 它解决什么问题

没有中央协调者，你得到的是意大利面条式的混乱。模型调用发生在工具处理器内部，工具结果被毫无结构地塞进 prompt，上下文无限膨胀直到 LLM 窒息。没有迭代预算，困惑的模型就会永远循环。

来自四个项目的具体数据：
- hermes-agent 的 `AIAgent` 类膨胀到 **9431 行**，因为它把编排、模型调用、工具执行、Memory 管理和上下文压缩混在了一起。改一处就有可能搞坏其他四处。
- Claude Code 的 `REPL.tsx` 达到 **5006 行**，把五个架构层塞进了一个文件。
- openclaw 让 `runEmbeddedPiAgent()` 专注于纯粹的编排，从而避免了这两个问题。

核心痛点：
1. **失控循环** —— 没有迭代预算时，卡住的模型会无限消耗 token。hermes 默认 90 次迭代。这个数字之所以存在，是因为有人在生产环境中跑到了 100+。
2. **上下文爆炸** —— 没有结构化的上下文构建，prompt 会在对话中途膨胀到超过窗口限制。
3. **委派混乱** —— 没有隔离的子 Agent 会污染父级的上下文，共享不该拥有的工具，或者无限递归。

## 四个项目是怎么做的

### hermes-agent

**文件**：`hermes/ai_agent/run_agent.py` —— 那个臭名昭著的 9431 行 `AIAgent` 类。

**循环**：同步 `while` 循环。调用 `run_conversation()`，获取模型响应，检查是否有工具调用，按顺序执行（或通过路径独立性检测并行执行最多 8 个），追加结果，重复。

**迭代预算**：`IterationBudget` —— 一个线程安全的计数器，默认 **90 次迭代**。每次循环递减。耗尽时 Agent 强制停止。

**上下文构建**：`prompt_builder` 从身份配置 + 加载的技能 + Memory 注入 + SKILLS_GUIDANCE 指令组装系统 prompt。全部在 `AIAgent` 内部完成。这就是职责分离崩塌的地方——prompt 组装逻辑和循环控制逻辑在同一个类里。

**子 Agent 委派**：`delegate_tool` 生成一个新的 `AIAgent` 实例，`MAX_DEPTH=2`，最多 3 个并行子 Agent。子 Agent 不能递归委派、不能向用户请求澄清、不能写入共享 Memory。

**优点**：极其简单易懂。一个文件，一个类，跟着 while 循环走就行。非常适合学习。
**缺点**：God class。修改模型调用逻辑意味着要动和工具分派逻辑同一个文件。重构等于在一个 9431 行的"病人"上做手术。

### openclaw

**文件**：`src/agent/runEmbeddedPiAgent()` —— 包装了 `pi-agent-core`（第三方库）。

**循环**：通过 pi-agent-core 执行引擎实现事件驱动。openclaw 加入了基于 Lane 的并发控制：同一会话内的消息排队处理，不同会话并行运行。

**Hook 系统**：这是 openclaw 的标志性设计。Agent Core 发出生命周期 Hook（`beforeModelCall`、`afterToolExec`、`onContextPressure` 等），其他模块订阅。Model Service、Tools、Memory、Context Engine —— 全部通过 Hook 注册，而非直接调用。这种控制反转让 Core 保持精简。

**上下文构建**：由可插拔的 Context Engine（`src/context-engine/`）处理，通过 `registerContextEngine()` 注册。Core 触发它，而不是包含它。

**子 Agent 委派**：Lane 队列支持子 Agent 事件（Discord `subagent_spawning`）。基础设施到位了，但使用程度不如 hermes 或 Claude Code。

**优点**：四个项目中职责分离最干净。Core 是纯粹的协调者。添加新模块意味着注册一个 Hook，而不是修改 Core。
**缺点**：依赖 `pi-agent-core` 第三方库。你继承了它的设计取向和它的 Bug。

### NemoClaw

NemoClaw 不是 Agent —— 它是 OpenClaw 实例的沙箱编排器。它没有自己的 Agent Core。它将 OpenClaw 的 Agent 运行时包裹在加固的容器中，配合 Landlock、seccomp 和逐进程网络策略。这里提及它只是为了完整性——如果你要构建 Agent Core，NemoClaw 的贡献在安全信封层，不在这里。

### Claude Code

**文件**：`src/core/query.ts`（内循环）+ `src/entrypoints/REPL.tsx`（外循环）。

**循环**：异步生成器模式。`query.ts` 在事件发生时逐个产出（模型 token、工具调用、工具结果、错误）。`REPL.tsx` 增量消费这个生成器，实时更新终端 UI。

**双层编排**：
- **内循环**（`query.ts`）：处理一次模型调用 -> 工具执行的周期。产出事件。
- **外循环**（`REPL.tsx`）：管理多轮对话会话。决定何时重新进入内循环，处理用户中断，管理推测执行状态。

这种拆分使内循环专注于模型-工具循环，而外循环负责会话级关注点（中止信号、权限提示、UI 更新）。

**推测执行**：当模型返回一个需要权限的工具调用时，Core 同时向用户显示权限提示并开始执行工具。如果用户批准——结果已经完成了。如果拒绝——通过 `CompletionBoundary` 回滚丢弃结果。这就是 Claude Code *感觉*更快的原因。

**子 Agent 委派**：`AgentTool` 支持 worktree 隔离（用 git worktree 实现文件系统隔离）、fork（子进程）、team/swarm 模式（多个 Agent 协同）和 coordinator 模式。四个项目中最丰富的委派模型。

**优点**：异步生成器很优雅。推测执行是真正的用户体验创新。双层设计比 hermes 更好地分离了关注点。
**缺点**：`REPL.tsx` 仍然达到 5006 行，因为混入了 5 层逻辑。异步生成器模式比简单的 while 循环更难调试。

## 对比表

| 维度 | hermes | openclaw | NemoClaw | Claude Code |
|-----------|--------|----------|----------|-------------|
| 核心文件 | `run_agent.py`（9431 行） | `runEmbeddedPiAgent()` | N/A（沙箱编排器） | `query.ts` + `REPL.tsx`（5006 行） |
| 循环风格 | 同步 while | 事件驱动 + Lane 队列 | N/A | 异步生成器，双层 |
| 迭代预算 | IterationBudget，默认 90 | pi-agent-core 管理 | N/A | Token/轮次限制 |
| 上下文构建 | AIAgent 内部（混合） | 可插拔 Context Engine，通过 Hook | N/A | 独立服务，从内循环调用 |
| 子 Agent 模型 | MAX_DEPTH=2，3 个并行，无递归 | Lane 队列 + Hook 事件 | N/A | worktree / fork / team / swarm / coordinator |
| 分离质量 | God class（最差） | 基于 Hook 的控制反转（最佳） | N/A | 双层（良好但 REPL 臃肿） |
| Session Store | SQLite，通过 MemoryProvider | pi-agent-core 管理 | N/A | JSONL + SQLite |
| 推测执行 | 否 | 否 | N/A | 是（SpeculationState + CompletionBoundary） |

## 最佳实践

**如果你只做一件事：让 Core 保持纯粹的协调者角色。** 它应该调用 Model、Tools、Memory 和 Context Manager，而不是包含它们的逻辑。

当你在编排循环里写下 `openai.chat.completions.create()` 而不是调用 `modelService.complete(messages)` 时，你就把 Core 耦合到了特定的提供商。当你在 Core 里写文件读取逻辑而不是调用 `toolService.execute("read_file", path)` 时，你就让 Core 承担了 I/O 的职责。这些一开始看起来无伤大雅。6 个月和 9431 行之后，它们就不是了。

来自四个项目的规则：

1. **Core 只拥有循环，别无其他。** 构建上下文 -> 调用模型 -> 分派工具 -> 检查预算 -> 重复。这就是它的全部职责。
2. **通过接口委派，而非直接调用。** openclaw 的 Hook 系统是黄金标准。至少，定义一个 `ModelService` 接口和一个 `ToolDispatcher` 接口。Core 调用这些接口，具体实现放在别处。
3. **永远设置迭代预算。** hermes 的默认值 90 存在是有原因的。没有预算的情况下，困惑的模型会循环直到你的账单惊人或者上下文窗口爆炸。
4. **子 Agent 要有自己的预算和受限的工具集。** hermes 禁止递归委派、用户澄清和共享 Memory 写入。Claude Code 通过 worktree 进行隔离。选择你的机制，但隔离不可谈判。
5. **Session Store 属于 Core，不属于 Memory。** 对话历史（原始消息数组）是 Core 的数据。提取出的长期知识是 Memory 的数据。把这两者混淆会导致循环依赖。

## 分步指南：如何构建

### 第 1 步：定义对话循环

从最简单的循环开始。不要异步，不要生成器，不要 Hook。就是一个 while 循环。

```python
# 伪代码 —— 先从这里开始，后续再优化
class AgentCore:
    def __init__(self, model_service, tool_dispatcher, context_manager, config):
        self.model = model_service          # 对等服务，不是被拥有的
        self.tools = tool_dispatcher        # 对等服务，不是被拥有的
        self.context = context_manager      # 对等服务，不是被拥有的
        self.config = config                # 来自 Config Store
        self.messages = []                  # Session Store（先用内存）
        self.max_iterations = config.get("max_iterations", 90)

    def run(self, user_message: str) -> str:
        self.messages.append({"role": "user", "content": user_message})
        iteration = 0

        while iteration < self.max_iterations:
            # 1. 构建上下文（系统 prompt + Memory 注入）
            context = self.context.build(self.messages, self.config)

            # 2. 调用模型
            response = self.model.complete(context)

            # 3. 没有工具调用？完成。
            if not response.tool_calls:
                self.messages.append({"role": "assistant", "content": response.text})
                return response.text

            # 4. 执行工具
            for call in response.tool_calls:
                result = self.tools.execute(call.name, call.args)
                self.messages.append({"role": "tool", "content": result, "tool_call_id": call.id})

            # 5. 检查上下文压力
            if self.context.needs_compression(self.messages):
                self.messages = self.context.compress(self.messages)

            iteration += 1

        return "[Agent reached iteration limit]"
```

**决策点**：同步还是异步？如果你的工具是 I/O 密集型（HTTP 调用、文件读取），从一开始就走异步。如果它们主要是本地计算，同步更简单也更容易调试。

### 第 2 步：添加会话持久化

用 Session Store 替换内存中的 `self.messages` 列表。SQLite 是务实的默认选择。

```python
class SessionStore:
    def load(self, session_id: str) -> list[Message]
    def append(self, session_id: str, message: Message) -> None
    def replace_all(self, session_id: str, messages: list[Message]) -> None  # 用于压缩
```

**决策点**：每个会话一个 SQLite 文件，还是一个数据库加一张会话表？单数据库更容易管理。hermes 使用一个 SQLite 文件存储所有数据。

### 第 3 步：定义对等服务接口

这些是 Core 与各个辐条之间的契约：

```python
class ModelService(ABC):
    def complete(self, messages: list[Message]) -> ModelResponse: ...

class ToolDispatcher(ABC):
    def execute(self, tool_name: str, args: dict) -> ToolResult: ...
    def list_tools(self) -> list[ToolDefinition]: ...

class ContextManager(ABC):
    def build(self, messages: list[Message], config: AgentConfig) -> list[Message]: ...
    def needs_compression(self, messages: list[Message]) -> bool: ...
    def compress(self, messages: list[Message]) -> list[Message]: ...
```

Core 依赖这些接口，永远不依赖具体实现。这就是 hermes 越过的那条线（直接在 `AIAgent` 中嵌入 OpenAI SDK 调用），也是 openclaw 守住的那条线（基于 Hook 的间接调用）。

### 第 4 步：添加子 Agent 委派

来自 hermes 和 Claude Code 的关键洞察：子 Agent 就是另一个带约束的 `AgentCore` 实例。

```python
def delegate(self, task: str, allowed_tools: list[str], max_depth: int = 2) -> str:
    if max_depth <= 0:
        return "[Cannot delegate further]"

    sub_tools = self.tools.create_restricted(allowed_tools)  # 父级工具的子集
    sub_core = AgentCore(
        model_service=self.model,       # 共享模型服务
        tool_dispatcher=sub_tools,      # 受限工具
        context_manager=self.context,   # 共享压缩逻辑
        config=self.config.child_config(max_iterations=30, max_depth=max_depth - 1)
    )
    result = sub_core.run(task)
    return summarize(result)  # 父级只看到摘要，不是完整对话
```

**决策点**：共享模型服务实例还是创建新的？共享更简单，避免重复连接。但如果你需要为每个子 Agent 设置独立的 token 预算，就创建独立实例。

### 第 5 步：添加中断处理

用户会按 Ctrl+C。模型会卡住。你需要一种打断循环的方式。

```python
# 添加到 while 循环中：
if self.abort_signal.is_set():
    self.messages.append({"role": "system", "content": "[Interrupted by user]"})
    break
```

Claude Code 的做法：外循环（`REPL.tsx`）管理中止信号，可以在内循环的异步生成器产出过程中取消它。对于更简单的实现，一个线程 Event 或 asyncio Event 就够了。

### 第 6 步（可选）：添加推测执行

这是 Claude Code 的创新。只有当你的 Agent 有权限系统时才值得做。

```python
# 不再是：
#   请求权限 -> 执行工具
# 而是：
#   后台启动工具执行
#   前台请求权限
#   如果批准：await 已经在运行的执行
#   如果拒绝：取消并回滚

async def speculative_execute(self, tool_call):
    execution_task = asyncio.create_task(self.tools.execute(tool_call))
    approved = await self.permission_prompt(tool_call)
    if approved:
        return await execution_task
    else:
        execution_task.cancel()
        return ToolResult(status="denied", rollback=True)
```

## 常见陷阱

**1. God Class 陷阱（hermes）**
hermes 的 `AIAgent` 一开始很干净。然后有人"临时"加了 prompt 构建。然后"因为更方便"加了模型调用逻辑。然后"暂时"加了上下文压缩。9431 行之后，没人能改一处而不冒破坏其他一切的风险。解决方案不是靠纪律——而是靠接口。如果 `ModelService` 是一个接口，你**物理上就无法**在 Core 中嵌入模型逻辑，因为 Core 不会导入 SDK。

**2. 臃肿的外循环（Claude Code）**
`REPL.tsx` 达到 5006 行，因为在一个组件中处理了 UI 渲染、会话管理、中止信号、推测执行状态和权限提示。内循环（`query.ts`）保持了专注。教训：如果你有双层编排，对外层的守护要和对内层一样严格。

**3. 缺失的迭代预算**
如果你不设置最大迭代次数，误解任务的模型会循环直到上下文窗口满了或者你的 API 账单令人惊恐。hermes 的默认值 90 已经很宽裕了——大多数任务在 20 次迭代以内完成。从 30 开始。有证据需要更多时再提高。

**4. 子 Agent 污染父级上下文**
没有隔离时，子 Agent 的完整对话历史会被追加到父级的上下文中。10 轮子 Agent 对话变成父级中的 10 条消息，摧毁了父级的专注度。hermes 和 Claude Code 都强制要求父级只接收子 Agent 工作的*摘要*。

**5. Session Store 与 Memory Store 的混淆**
Session Store 保存原始对话（消息数组）。Memory Store 保存提取出的知识（事实、偏好、技能）。如果你把提取的知识放回消息数组，你会使上下文大小翻倍。如果你把原始消息放进 Memory Store，你会用短暂的聊天记录污染长期知识。把它们分开。Core 拥有 Session Store。Memory 模块拥有 Memory Store。

## 跨模块契约

### Agent Core 对其他模块的期望

| 模块 | 契约 | 格式 |
|--------|----------|--------|
| **Model Service** | `complete(messages) -> ModelResponse` | `ModelResponse` 包含 `.text`、`.tool_calls[]`、`.usage`（token 计数） |
| **Tool Dispatcher** | `execute(name, args) -> ToolResult` | `ToolResult` 包含 `.content`（字符串）、`.is_error`（布尔值） |
| **Tool Dispatcher** | `list_tools() -> ToolDefinition[]` | 在上下文构建阶段用于为模型构建工具 schema |
| **Context Manager** | `build(messages, config) -> messages` | 返回包含系统 prompt 和 Memory 注入的增强消息数组 |
| **Context Manager** | `needs_compression(messages) -> bool` | 当 token 数超过阈值时返回 True（例如窗口的 80%） |
| **Context Manager** | `compress(messages) -> messages` | 返回缩短后的消息数组 |
| **Memory**（作为基础设施） | `get_injections(context) -> string[]` | 在 Context Manager 的 `build()` 中调用，不由 Core 直接调用 |

### Agent Core 向其他模块提供什么

| 消费者 | Core 提供什么 | 格式 |
|----------|-------------------|--------|
| **Model Service** | 组装好的消息数组 | 标准聊天格式：`{role, content, tool_calls?, tool_call_id?}` |
| **Tool Dispatcher** | 工具调用指令 | 来自模型响应的 `{name: string, args: dict, id: string}` |
| **Context Manager** | 原始消息历史 | 来自 Session Store 的 `messages` 数组 |
| **子 Agent** | 受限配置 | 包含更低 `max_iterations`、更小 `max_depth`、工具白名单的 `AgentConfig` |
| **接口层** | 对话事件 | 最终的助手响应文本，或流式 token/事件（异步生成器模式） |

### 不变量

- Core 永远不直接调用 LLM API。始终通过 `ModelService`。
- Core 永远不直接执行系统命令。始终通过 `ToolDispatcher`。
- Core 永远不直接读写 Memory Store。Memory 注入通过 `ContextManager.build()` 传入。
- 子 Agent 永远不共享父级的 Session Store。它们有自己的。
- 迭代预算由 Core 强制执行，不由 Model 或 Tools 执行。其他模块不能覆盖它。
