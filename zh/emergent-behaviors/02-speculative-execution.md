# 涌现行为 2：推测执行

> 一句话总结：Agent 在运行工具的同时请求权限，当用户批准时立即显示已完成的结果 -- 没有任何组件被设计为"速度优化器"。

## 它是什么

推测执行是从 CPU 设计中借来的并行化技巧。当 LLM 返回一个需要用户权限的工具调用时，Agent 同时做两件事：

1. 向用户展示权限提示（"允许编辑文件？"）。
2. 在后台开始执行工具。

如果用户批准，结果已经完成。它立即出现。如果用户拒绝，结果被丢弃，系统回滚到上一个 CompletionBoundary。

为什么它是"涌现"的：不存在 `SpeedOptimizer` 类。不存在 `ParallelExecutionEngine`。这个行为源自四个独立机制的交互：UI 中的异步权限流、工具运行器中的预执行逻辑、Agent Core 中的并行调度，以及知道何时提交或丢弃的 SpeculationState 追踪器。每个机制都是为自己的目的构建的。组合在一起，它们在需要权限的操作上产生了感知上 2-3 倍的速度提升。

Claude Code 是唯一实现这一功能的项目。对比集中没有其他 Agent 尝试过。

## 它是如何涌现的

```
LLM 返回 tool_call
  │
  │  Agent Core 接收 tool_call
  │  检查：这个工具需要权限吗？
  │
  ├──────────────────────────────────────┐
  │ (并行)                               │ (并行)
  │                                      │
  ▼                                      ▼
Interface/UI                          Tools
  │                                      │
  │  渲染权限提示                         │  开始执行工具
  │  "Allow: edit file X?"               │  （写磁盘、运行命令等）
  │  等待用户输入                         │  将结果存入 SpeculationState
  │                                      │  标记结果为"推测性的"
  │                                      │
  ▼                                      ▼
用户响应                              执行完成
  │                                      │
  ├── 批准 ──→ 提交推测性结果 ────────────┘
  │             展示给用户。
  │             延迟 = 0（已经完成）。
  │
  └── 拒绝 ──→ 丢弃推测性结果。
                回滚到 CompletionBoundary。
                SpeculationState → 清除。
                工具效果被撤销。
```

关键洞察：权限提示和工具执行之间零依赖。权限提示需要工具描述。工具执行需要工具参数。两者都可以从原始 `tool_call` 获得。因此它们可以并行运行。

## 哪些组件参与

| 组件 | 在推测执行中的角色 |
|-----------|-------------------------------|
| **Agent Core** | 并行调度器。从 LLM 接收 `tool_call`。同时派发权限请求和工具执行。读取 SpeculationState 来决定：提交还是丢弃。 |
| **Tools** | 预执行器。立即运行工具而不等待权限。将结果标记为推测性的。必须支持结果丢弃 -- 执行可能被抛弃。 |
| **Interface/UI** | 权限门。异步渲染审批提示。返回 approve/reject。必须是非阻塞的 -- 在等待用户时不能锁住事件循环。 |
| **SpeculationState** | 回滚追踪器。记录：哪个工具被推测执行了、产生了什么结果、CompletionBoundary 在哪里（安全回滚点）。它本身不是一个组件 -- 它是 Agent Core 和 Tools 之间共享的数据结构。 |

## 案例研究：Claude Code

Claude Code 是发现这个行为的地方。它深度嵌入在工具执行流水线中。

### 权限模型

Claude Code 有两种权限模式：

- **ask 模式：** 每个修改状态的工具调用都需要用户审批（文件写入、shell 命令等）。只读工具跳过权限。
- **auto 模式：** 预批准的工具模式完全跳过权限。

推测执行只在 **ask 模式**中有意义。在 auto 模式中，没有权限提示，因此没有并行化机会。

### Agent Core 中的并行派发

在 `query.ts` 中，当 LLM 返回工具调用时：

1. Agent Core 检查工具是否需要权限（基于工具元数据和权限配置）。
2. 如果需要权限，它创建两个并发任务：
   - 任务 A：向 UI 层（`REPL.tsx`）发送权限请求。
   - 任务 B：用工具参数调用工具运行器。
3. 两个任务在同一个 tick 开始。任务 A 之前没有 `await` 来阻塞任务 B。

### SpeculationState 追踪

SpeculationState 对象追踪：

- `toolCallId`：哪个工具调用是推测性的。
- `result`：工具的返回值（保存在内存中，尚未提交到会话）。
- `boundary`：一个 CompletionBoundary 标记 -- 会话的上一个已知良好状态。如果推测失败，会话倒回到这个点。

### 提交路径（用户批准）

当用户输入"y"或审批超时自动批准时：

1. Agent Core 读取 `SpeculationState.result`。
2. 将结果作为正常的 `ToolResult` 追加到会话。
3. 清除 SpeculationState。
4. 继续会话循环。

从用户角度看：他们按"y"，结果立即出现。无需等待执行。

### 丢弃路径（用户拒绝）

当用户输入"n"或按 Escape 时：

1. Agent Core 读取 `SpeculationState.boundary`。
2. 将会话倒回到那个 CompletionBoundary。
3. 丢弃 `SpeculationState.result`。
4. 在会话中插入拒绝消息："User denied permission for [tool]."
5. 继续循环。LLM 看到拒绝并调整。

对于文件写入：文件在推测期间已经被写入。拒绝时，文件恢复到执行前的状态（Claude Code 追踪原始内容）。

对于 shell 命令：输出被捕获但从未显示。拒绝时被丢弃。副作用（例如命令启动的进程）必须被处理 -- 这是困难的部分。

### 为什么它感觉很快

在没有推测的 ask 模式下，时间线是：

```
LLM 返回 tool_call ─→ 显示权限 ─→ 用户批准 ─→ 执行工具 ─→ 显示结果
                        [用户思考]                [执行时间]
总等待 = 用户时间 + 执行时间
```

有了推测：

```
LLM 返回 tool_call ─→ 显示权限 ─────→ 用户批准 ─→ 显示结果
                   └──→ 执行工具 (并行) ─→ 完成 (等待中)
总等待 = max(用户时间, 执行时间)
                       ≈ 用户时间（执行通常更快）
```

对于一个需要 200ms 执行的文件编辑，用户需要 1.5 秒按"y"：没有推测，总计 = 1.7s。有推测，总计 = 1.5s。那 200ms 被隐藏了。

对于一个需要 5 秒的 shell 命令：没有推测，总计 = 6.5s。有推测，总计 = 5s。用户的思考时间被隐藏在执行时间内。

## 如何在你的 Agent 中使其涌现

### 步骤 1：让权限流异步

权限请求必须是非阻塞的。它发送请求给 UI 并返回 Promise/Future。它不阻塞主循环。

```
function requestPermission(toolCall): Promise<"approve" | "reject">
```

如果你的权限流是同步的（阻塞直到用户响应），推测就不可能。这是第一个前提条件。

### 步骤 2：让工具执行返回可丢弃的结果

工具运行器必须支持推测模式：

```
function executeSpeculative(toolCall): { result: ToolResult, rollback: () => void }
```

`rollback` 函数撤销工具的副作用。对于文件写入：恢复原始内容。对于 shell 命令：杀死进程并丢弃输出。对于只读工具：无操作（没有什么要回滚的）。

不是所有工具都能被推测执行。发送邮件的工具无法回滚。在工具元数据中将这些标记为 `speculationSafe: false`。

### 步骤 3：向 Agent Core 添加 SpeculationState

```
interface SpeculationState {
  toolCallId: string
  result: ToolResult | null    // null = 仍在执行
  rollback: (() => void) | null
  boundary: CompletionBoundary // 推测前的会话状态
}
```

Agent Core 在派发并行任务时创建此状态。在权限解决时读取它。

### 步骤 4：接入并行派发

在你的主循环中，将：

```
// 顺序执行（慢）
const approved = await requestPermission(toolCall)
if (approved) {
  const result = await executeTool(toolCall)
  appendToConversation(result)
}
```

替换为：

```
// 并行执行（快）
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

### 步骤 5：处理边缘情况

- **工具执行失败。** 正常显示权限提示。如果批准，显示错误。如果拒绝，同时丢弃错误和拒绝。
- **用户在执行完成前就响应了。** 如果批准，等待执行完成，然后显示结果。如果拒绝，取消执行（如果可能）并回滚。
- **一次 LLM 响应中有多个工具调用。** 只对第一个需要权限的调用进行推测。其余的在它解决后顺序执行。同时对多个调用进行推测会创建复杂的回滚链。

## 跨模块契约

### 契约 1：ToolResult 支持回滚状态

```
interface ToolResult {
  toolCallId: string
  output: string | object
  isSpeculative: boolean          // true = 尚未提交
  rollback: (() => void) | null   // 对于只读工具为 null
}
```

Tools 必须产生可丢弃的结果。如果工具直接写入会话流（例如流式输出），推测就会中断 -- 你无法取消流式输出。

### 契约 2：权限请求是异步可取消的

```
interface PermissionRequest {
  prompt(): Promise<"approve" | "reject">
  cancel(): void    // 当推测需要中止时调用
}
```

权限提示必须支持取消。如果工具执行在用户响应前灾难性失败，Agent Core 取消权限请求并显示错误。

### 契约 3：CompletionBoundary 格式

```
interface CompletionBoundary {
  messageIndex: number           // 会话历史中的索引
  fileSnapshots: Map<string, string>  // 路径 → 推测前的内容
  processIds: number[]           // 推测期间生成的 PID
}
```

这定义了回滚范围。`messageIndex` 告诉会话回滚到哪里。`fileSnapshots` 告诉文件系统恢复什么。`processIds` 告诉操作系统杀死什么。

边界必须在推测开始*之前*捕获。如果在之后捕获，它包含了推测性变更，回滚就会损坏。

## 如何测试（循环测试）

```
Test: speculative-execution-full-cycle
Precondition: agent is in "ask" permission mode. A test file exists.

Step 1 — 验证并行启动
  发送一条触发文件写入工具调用的消息。
  检测：权限提示出现的时间戳（T_perm）。
  检测：工具执行开始的时间戳（T_exec）。
  断言：|T_perm - T_exec| < 50ms。
  （两者必须在同一个事件循环 tick 内启动。）

Step 2 — 验证批准路径
  模拟用户延迟 500ms 后批准。
  检测：结果展示给用户的时间戳（T_show）。
  检测：工具执行完成的时间戳（T_done）。
  断言：T_show - T_done < 100ms。
  （批准到来时结果已经准备好了。）
  断言：磁盘上的文件包含预期的编辑。

Step 3 — 验证拒绝路径
  发送另一条触发文件写入的消息。
  模拟用户延迟 500ms 后拒绝。
  断言：磁盘上的文件未更改（回滚到原始状态）。
  断言：会话不包含推测性工具结果。
  断言：会话包含拒绝消息。
  断言：SpeculationState 已清除。

Step 4 — 验证边界完整性
  发送一条触发两个顺序工具调用的消息。
  第一个调用：批准（推测性的）。
  第二个调用：拒绝。
  断言：第一个调用的结果已提交到会话。
  断言：第二个调用的结果已丢弃。
  断言：文件系统只反映第一个调用的变更。

Pass criteria: all 4 steps pass.
```

运行时间：使用模拟计时器约 5 秒。使用真实用户模拟约 15 秒。

## 跨项目变体

| 项目 | 推测执行支持 | 原因 |
|---------|-------------------|---------------|
| **Claude Code** | 完整实现。SpeculationState 追踪、CompletionBoundary 回滚、并行权限 + 执行。仅在"ask"模式下。 | Anthropic 为 CLI 中的感知速度做了优化。没有推测的 ask 模式体验会感觉迟钝。对这种模式的投入很高。 |
| **hermes-agent** | 无。严格顺序执行。LLM 返回工具调用，Agent Core 等待审批（如果需要），然后执行。 | hermes 运行在多个平台上（Telegram、Discord 等），"权限提示"是聊天线程中的一条消息。跨网络往返到聊天平台的并行化不实际。 |
| **openclaw** | 无。标准请求-响应。工具调用只在 LLM 响应完全处理后执行。 | openclaw 的架构是基于插件的。工具执行经过多个层（Plugin SDK、Hook 系统、lane 队列）。添加推测需要跨所有层的变更。 |
| **NemoClaw** | 不适用。NemoClaw 是包装另一个 Agent 的安全运行时。它不直接执行工具。 | NemoClaw 的贡献是安全信封，而非执行流水线。 |

### 为什么只有 Claude Code 这样做

三个条件使推测执行在 Claude Code 中可行，在其他地方不实际：

1. **单用户 CLI。** 权限提示是本地的 -- 终端中的一次按键。往返时间是毫秒级。在 Telegram 上，往返是秒级。当网络占主导时，推测的节省可以忽略不计。

2. **以文件为中心的工具。** 大多数 Claude Code 工具是文件读/写。文件操作快（< 200ms）且可逆（保存原始内容，回滚时恢复）。Shell 命令更难但仍可管理（捕获输出，杀死进程）。发送 HTTP 请求或修改数据库的工具无法安全推测。

3. **对执行流水线的紧密控制。** Claude Code 是单体应用。Agent Core、Tools 和 UI 在同一个进程中。没有插件边界、没有 RPC、没有序列化。并行派发两个任务就是一个 `Promise.all()`。在 openclaw 中，同样的操作需要跨越插件边界、Hook 链和潜在的网络调用。

### 如果你想要推测执行但你的 Agent 不是 CLI

专注于推测安全的工具子集：

- 文件读取：始终安全（无副作用）。
- 文件写入：如果在写入前做快照则安全。
- 代码格式化/linting：安全（确定性的，可逆的）。
- 只产生输出的 shell 命令：基本安全（除了进程本身无副作用）。

将其他所有东西标记为 `speculationSafe: false` 并顺序执行。即使部分推测（50% 的工具调用）也会产生明显的加速。
