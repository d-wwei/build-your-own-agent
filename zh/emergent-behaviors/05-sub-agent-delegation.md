# 涌现行为 5：Sub-Agent Delegation

> 一句话：父 Agent 生成一个隔离的子 Agent，赋予其有限的预算、受限的 Tools 集合和无共享 Memory——然后只接收一条摘要返回。分而治之，同时不污染父级的上下文。

## 它是什么

父 Agent 遇到一个需要委派的任务。它调用一个工具。这个工具创建了一个全新的 Agent Core 实例——拥有自己的对话循环、自己的迭代预算、自己的工具集。子 Agent 运行至完成。父 Agent 收到一条摘要消息。父级的上下文窗口只增长了一条消息，而非子 Agent 内部产生的 15-40 条消息。

这不是函数调用。不是生成线程。而是创建了一个独立思考的第二个 Agent。

## 它如何涌现

没有单个组件"执行"委派。三个模块协同组合：

1. **Tools** — 父 Agent 的工具注册表包含一个委派工具（`delegate_task`、`AgentTool`）。从 LLM 的角度看，这只是又一次工具调用。它完全不知道自己正在生成第二个智能体。

2. **Agent Core** — 委派工具的实现创建了一个新的 Agent Core 实例。这个实例获得自己的构造参数：全新的对话历史、独立的迭代预算、过滤后的工具列表。它运行与父 Agent 完全相同的 `while(budget > 0)` 循环。

3. **Context Manager** — 当子 Agent 完成时，只有其最终摘要被注入到父级的上下文中。子 Agent 的完整对话历史——所有的模型调用、工具结果、错误、重试——保持隔离。父级永远看不到它们。

涌现部分：父 LLM 从未"学会"委派是如何工作的。它是在文本上训练的。它看到一个叫 `delegate_task` 的工具，描述是"将子任务委派给一个专门的 sub-agent。"它基于 prompt 模式决定调用它。工具完成所有繁重的工作。组合结果 = 从一个扁平的工具调用中产生了层次化的任务分解。

## 哪些组件参与

| 组件 | 在委派中的角色 |
|-----------|-------------------|
| Agent Core（父级） | 运行常规循环，像调用其他工具一样调用 delegate tool |
| Tools（delegate tool） | 用受限参数创建子 Agent Core 实例 |
| Agent Core（子级） | 拥有独立预算的独立对话循环 |
| Model（子级） | 子 Agent 进行自己的 LLM 调用——父级看不到 |
| Context Manager（父级） | 子 Agent 返回时接收一条摘要消息 |

**不参与的部分：** Memory。子 Agent 不能写入父级的 Memory 存储。这是一个硬性约束，不是建议。

## 案例研究：hermes-agent 的 `delegate_tool`

hermes-agent 实现了最保守的委派模型。以下是其确切工作方式：

**触发**：LLM 返回一个对 `delegate_task` 的工具调用，参数为 `{task: "...", context: "..."}`。

**构造**：`delegate_tool` 创建一个新的 `AIAgent` 实例。构造参数：
- `MAX_DEPTH = 2` — 子 Agent 可以再委派一次；孙 Agent 不可以
- 并行限制：最多 **3** 个 sub-agent 同时运行
- 工具黑名单：`clarify_request`（不能向用户提问）、`memory_write`（不能写入共享 Memory）、在深度 2 时的 `delegate_task`（不能进一步递归）
- 预算：独立的迭代计数器，与父级分离

**执行**：子 Agent 运行自己的 `run_conversation()` 循环。它调用 LLM、执行工具、迭代。父级被阻塞等待工具结果，就像等待文件读取或网络请求一样。

**返回**：当子 Agent 耗尽预算或完成任务时，它返回一段文本摘要。这段摘要成为父级上下文中 `delegate_task` 的工具结果。

**隔离审计**：
- 子 Agent 产生了 23 条内部消息（模型调用 + 工具结果）
- 父级的上下文只增长了 1 条消息（摘要）
- 父级的 token 预算未被子 Agent 的 LLM 调用消耗
- 如果子 Agent 写入了 Memory，父级不会知道——但它不能写，因为 `memory_write` 被列入了黑名单

## 案例研究：Claude Code 的 `AgentTool`

Claude Code 采用了不同的方式。hermes 是保守的（单一模式、严格限制），Claude Code 则提供五种委派模式：

| 模式 | 隔离机制 | 用途 |
|------|---------------------|----------|
| **worktree** | Git worktree — 子 Agent 获得独立的文件系统分支 | 可能与父级工作冲突的文件变更 |
| **fork** | 子进程 — 操作系统级别隔离 | CPU 密集型任务、崩溃隔离 |
| **team** | 多个 Agent 协调处理相关子任务 | 并行探索不同方案 |
| **swarm** | 大量 Agent 同时处理大型任务 | 大规模重构、批量文件变更 |
| **coordinator** | 一个 Agent 编排多个工作者 | 复杂的多步骤工作流 |

worktree 模式值得注意。当子 Agent 修改文件时，它在独立的 Git worktree 中操作。如果父级判定子 Agent 的工作有误，删除 worktree 即可。父级的工作目录中没有任何文件受到影响。这是免费的文件系统级别回滚。

所有模式共享相同的核心契约：子 Agent 获得独立预算、受限工具，并返回摘要。模式之间的区别在于子 Agent 在上下文隔离之外获得了什么样的额外隔离。

## 案例研究：openclaw 的基础设施支持

openclaw 没有像 hermes 或 Claude Code 那样的一等委派命令。它拥有的是使委派成为可能的基础设施：

- **Lane queuing**：每个会话运行在自己的 Lane 中。一个 sub-agent 可以被分配自己的 Lane，这意味着它的消息独立于父级排队。
- **Hook system**：Discord 的 `subagent_spawning` 事件在 sub-agent 创建时触发。其他模块可以订阅——日志记录、速率限制、配额跟踪都可以挂接进来而不触碰委派逻辑。

这比 hermes 或 Claude Code 更轻量。构建模块已就位；编排方面的主张更少。

## 如何在你的 Agent 中让它涌现

### 步骤 1：构建一个 delegate tool

创建一个具有以下签名的工具：

```
Tool: delegate_task
Input: { task: string, context: string }
Output: { summary: string, status: "completed" | "failed" }
```

像注册其他工具一样，将它注册到父 Agent 的工具列表中。

### 步骤 2：工具创建子 Agent Core

在工具的 execute 函数内部：

```python
def execute(params):
    child = AgentCore(
        budget=30,           # independent budget, not the parent's
        tools=CHILD_TOOLS,   # filtered tool list
        memory=None,         # no shared memory access
        depth=parent.depth + 1
    )
    result = child.run(params.task, params.context)
    return { "summary": result.summary, "status": result.status }
```

### 步骤 3：设置约束

子 Agent 的三条硬规则：
1. **不允许超过 MAX_DEPTH 的递归委派**。hermes 使用 2。从这里开始。
2. **不允许用户交互**。子 Agent 不能提出澄清性问题。它只用已有信息工作。
3. **不允许共享 Memory 写入**。子 Agent 的结论进入其摘要，而非父级的长期 Memory 存储。

### 步骤 4：接通返回路径

子 Agent 的摘要成为工具结果。父级的 Context Manager 将其当作普通工具结果处理——一条消息，长度有界。

如果子 Agent 失败，工具结果标注 `status: "failed"` 并附原因。父级决定下一步：重试、换种方式尝试、还是放弃。

### 步骤 5：添加并发（可选）

如果父级在一个 turn 中调用了三次 `delegate_task`，你需要一个并发限制。hermes 上限为 3 个并行 sub-agent。保守起步。LLM 过度委派的倾向是真实存在的。

## 跨模块契约

以下是委派能够工作之前你必须定义的接口：

### 1. 子 Agent 构造参数

```
{
  task: string,          // what to do
  context: string,       // relevant background from parent
  budget: number,        // max iterations (independent of parent)
  depth: number,         // current delegation depth
  tools: Tool[],         // restricted tool set
  memory_access: "none" | "read-only"  // never "read-write"
}
```

### 2. 工具黑名单

子 Agent 的最小黑名单：
- `delegate_task`（在最大深度时）
- `clarify_request` / `ask_user`（不允许用户交互）
- `memory_write` / `memory_update`（不允许共享持久化）

基于风险的可选黑名单：
- `shell_execute`（如果由父级处理所有 shell 操作）
- `file_delete`（破坏性操作保留给父级）

### 3. 摘要返回格式

```
{
  summary: string,      // max 2000 tokens — enforced, not suggested
  status: "completed" | "failed" | "budget_exhausted",
  artifacts: string[],  // file paths, URLs, or other references created
  tool_calls_made: number  // for debugging and cost tracking
}
```

### 4. 深度限制强制

```
MAX_DEPTH = 2         // parent → child → grandchild. No further.
                      // Enforced in child constructor, not in the LLM prompt.
                      // Prompt-level enforcement is a suggestion.
                      // Constructor-level enforcement is a wall.
```

## 如何测试（循环测试）

委派是跨模块行为。仅对 delegate tool 做单元测试是不够的。你需要循环测试来演练完整链路。

### 测试 1：子 Agent 获得独立预算

```
1. Parent has budget=50
2. Parent calls delegate_task
3. Child runs with budget=30
4. Child exhausts budget after 30 iterations
5. Assert: parent's remaining budget is still 49 (one iteration spent on the delegate call)
6. Assert: child's budget is 0
```

### 测试 2：工具限制被强制执行

```
1. Create child with tool blacklist = [clarify_request, memory_write]
2. Child's LLM returns a tool call to clarify_request
3. Assert: tool dispatch rejects the call
4. Assert: child receives an error message, not a crash
5. Assert: child continues its loop (handles the error gracefully)
```

### 测试 3：摘要隔离

```
1. Child runs, produces 23 internal messages
2. Child completes, returns summary
3. Assert: parent's conversation history grew by exactly 2 messages
   (the delegate_task call + the tool result)
4. Assert: parent has zero access to child's 23 internal messages
```

### 测试 4：深度限制守住

```
1. Parent (depth=0) delegates to child (depth=1)
2. Child (depth=1) delegates to grandchild (depth=2)
3. Grandchild's tool list does NOT contain delegate_task
4. Assert: grandchild cannot delegate
5. Assert: this is enforced at constructor level, not prompt level
```

### 测试 5：并发限制

```
1. Parent calls delegate_task 5 times in parallel
2. MAX_PARALLEL = 3
3. Assert: 3 children start immediately
4. Assert: 2 children queue until a slot opens
5. Assert: all 5 eventually complete
```

## 各项目差异

| 维度 | hermes-agent | Claude Code | openclaw |
|-----------|-------------|-------------|----------|
| **委派触发** | 工具注册表中的 `delegate_tool` | 带模式选择的 `AgentTool` | Lane queue + Hook events |
| **最大深度** | 2 | 按模式可配置 | 未显式限制 |
| **最大并行** | 3 | 按模式变化（team 模式可以很多） | 基于 Lane 的并发 |
| **文件系统隔离** | 无（共享文件系统） | Git worktree（完全隔离） | 无 |
| **进程隔离** | 同进程，新实例 | Fork 模式 = 子进程 | 同进程 |
| **Memory 策略** | 黑名单——不允许写入 | 按模式而定 | 未明确规定 |
| **用户交互** | 黑名单——不允许 clarify | 子 Agent 无面向用户的 I/O | 未明确规定 |
| **返回格式** | 工具结果中的文本摘要 | 带 artifacts 的结构化摘要 | 通过 Hook 的事件驱动 |
| **成熟度** | 生产验证，保守路线 | 功能最丰富，企业级规模 | 基础设施就绪，更轻量的使用方式 |
| **NemoClaw** | N/A — 不是 Agent | — | — |

**关键区别**：hermes 说"委派是一件事，安全地做。"Claude Code 说"委派是五件事，选择正确的隔离模型。"openclaw 说"这里是构建模块，自己组装。"

对于你的第一个实现，从 hermes 的模型开始。单一模式、严格限制、已被证明安全。当你有了生产数据表明你实际需要哪些隔离模式时，再添加 Claude Code 的变体。
