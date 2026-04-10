# 模块 3：Tools
> Agent 的手和脚——注册、发现、分派、并行执行和安全审批。

## 它是什么

Tools 是 Agent 作用于世界的方式。LLM 负责思考，Tools 负责行动。当模型决定"我需要读取这个文件"或"运行这条 shell 命令"时，Tools 模块找到正确的处理器，检查权限，执行它，然后返回结果。

在 Hub-and-Spoke 架构中，Tools 是 Agent Core 的**对等服务**——不在它之下，也不在它内部。Core 说"用参数 Y 执行工具 X"。Tools 从那里接管一切：查找处理器、判断是否安全、运行它（可能与其他工具并行），然后交回结构化的结果。Core 永远不触及工具内部。

Tools 还拥有 **Skill Store**——塑造 Agent 行为的 MD/YAML 技能文件目录。技能由 Agent Core 在 prompt 构建时加载，但它们的生命周期（创建、读取、更新、删除）通过工具调用来管理。这就是学习循环涌现行为的工作原理：Tools 写入文件，Core 在下一轮读取它。

## 它解决什么问题

**发现**：LLM 需要知道存在哪些工具。没有注册表，你就得把工具描述硬编码到系统 prompt 中。hermes 有 50+ 工具，openclaw 有 100+，Claude Code 有 60+。手动 prompt 管理在超过 10 个工具后就不可扩展了。

**分派**：模型以 JSON 形式返回工具名称和参数。需要有什么东西来查找处理器、验证参数、路由调用。搞错了你的 Agent 要么在遇到未知工具时崩溃，要么静默忽略有效的工具。

**并行执行**：单次 LLM 响应经常请求 3-8 个工具调用。串行运行意味着用户盯着加载动画看 30 秒而不是 5 秒。但天真的并行化会导致竞态条件——两个工具同时编辑同一个文件会导致损坏。

**安全审批**：你的 Agent 能运行 `rm -rf /`。得有人阻止它。审批逻辑存在于工具执行流内部，而不是作为单独的中间件层——因为决策依赖于工具特定的上下文（哪个文件？哪个命令？什么权限？）。

**MCP 出站**：你的 Agent 需要的工具比你构建的更多。MCP（Model Context Protocol）让它能连接到外部服务器获取额外能力。这是出站 MCP——你的 Agent 向外调用。入站 MCP（其他系统调用你的 Agent）属于接口层。

## 四个项目是怎么做的

### hermes-agent

**注册模式**：导入时自注册。每个工具文件在模块加载时调用 `registry.register()`。Python 导入文件时，工具就注册了。没有中心清单，没有配置步骤。

```python
# hermes: tools/shell_tool.py（简化）
from core.registry import registry

@registry.register(
    name="shell_exec",
    description="Execute a shell command",
    parameters={...}
)
async def shell_exec(command: str, workdir: str = "."):
    # 实现
```

**工具数量**：50+ 内置工具，涵盖 shell、文件、网络、浏览器、代码执行、视觉、委派和技能管理等类别。

**并行执行**：路径独立性检测。在并行运行工具之前，hermes 检查是否有两个工具操作同一文件路径。独立的工具并发运行；冲突的工具串行运行。最多 8 个线程。

**安全**：`approval.py` 使用正则表达式模式检测危险命令（`rm -rf`、`sudo`、`curl | bash`），并扫描工具参数中的 prompt 注入。简单但对单用户场景有效。

**Skill Store**：`~/.hermes/skills/*.md` 文件，带 YAML frontmatter。`skill_manage` 工具处理 CRUD。`prompt_builder` 每轮扫描该目录，将相关技能注入系统 prompt。

**优点**：极其简单。新开发者 5 分钟就能添加一个工具。自注册意味着零配置开销。

**缺点**：无第三方扩展性。想添加工具？Fork 仓库然后编辑源码。另外，注册表是全局状态——测试时需要小心清理。

### openclaw

**注册模式**：Plugin SDK，使用 `api.registerTool()`。工具在 Plugin 的 `onActivate()` 生命周期 Hook 中注册。SDK 提供类型安全的接口和验证。

```typescript
// openclaw: extensions/some-plugin/index.ts（简化）
export function onActivate(api: PluginAPI) {
  api.registerTool({
    name: "web_search",
    description: "Search the web",
    parameters: z.object({ query: z.string() }),
    isConcurrencySafe: true,
    execute: async (params) => { /* ... */ }
  });
}
```

**工具数量**：100+ 工具，分布在核心和社区 Plugin 中。Plugin SDK 导出 50+ 子路径入口供第三方开发者使用。

**并行执行**：每个工具上的显式 `isConcurrencySafe` 标志。如果工具声明自己可以安全并发执行，运行时会把它与其他安全工具并行化。不做路径分析——由工具作者自行决定。

**安全**：多层。DM 配对（安装 Agent 的用户必须通过私信确认操作）。危险命令的执行审批流程。按工具类别的限流。

**MCP 出站**：一等支持。Plugin 可以注册 MCP 服务器连接，它们的工具与注册表中的原生工具并列。LLM 看不出区别。

**Skill Store**：工作区级别的技能文件。与 hermes 不同，技能不是由 Agent 自动创建的——它们由人手动编写并放在工作区中。

**优点**：最大的扩展性。第三方开发者可以发布工具 Plugin 而无需接触核心代码。`isConcurrencySafe` 标志设计优雅——它把安全决策放在了知识所在之处。

**缺点**：复杂性。Plugin SDK 是相当大的基础设施。注册生命周期（onActivate/onDeactivate）增加了认知负担。对于构建个人 Agent 的独立开发者来说，这是过度设计。

### NemoClaw

NemoClaw 不是 Agent——它是安全沙箱运行时。它没有自己的工具系统。它用安全策略包裹 OpenClaw 的工具：Landlock 文件隔离、逐进程网络限制、seccomp 系统调用过滤。尝试访问未授权路径或网络端点的工具在操作系统层面被阻止，而不是在应用层面。

与 Tools 模块相关的是：NemoClaw 的工具安全方式与应用级审批是正交的。它证明了**两层工具安全可以共存**——应用检查（"这个命令危险吗？"）和操作系统级强制（"这个进程能访问这个路径吗？"）。

### Claude Code

**注册模式**：静态导入加上特性标志的死代码消除（DCE）。所有工具在中心文件中导入。特性标志（构建时评估）决定哪些工具可用。打包器剥离不可达的代码。

```typescript
// Claude Code: tools/index.ts（简化）
import { ShellTool } from "./shell"
import { FileTool } from "./file"
import { BrowserTool } from "./browser"  // 在特性标志后面

const ALL_TOOLS: Tool<any, any, any>[] = [
  ShellTool,
  FileTool,
  ...(FEATURE_FLAGS.browser ? [BrowserTool] : []),
]
```

**工具数量**：60+ 工具。每个工具实现一个泛型 `Tool<Input, Output, Permissions>` 接口——类型系统强制每个工具声明其输入 schema、输出形状和所需权限。

**并行执行**：批次拆分。当 LLM 返回多个工具调用时，Claude Code 将它们拆分为最多 10 个一批并发运行。不做路径分析，不用安全标志——只是一个硬性并发上限。

**安全**：权限模式（ask/auto）可按工具类别配置。来自 MDM（移动设备管理）的企业策略限制。执行前调用 `Tool.checkPermissions()`。对于推测执行，工具在显示权限提示的同时乐观地运行——如果被拒绝，结果在 CompletionBoundary 处被丢弃。

**Skill Store**：`~/.claude/skills/` 目录。技能通过 `SKILL.md` 约定加载。`/remember` 命令和 `extractMemories` 后台 Agent 可以保存事实，但不会像 hermes 那样创建结构化的技能文件。

**优点**：禁用的工具零运行时开销——它们在打包中根本不存在。`Tool<I,O,P>` 泛型在整个工具表面强制类型安全。权限模式给用户细粒度的控制。

**缺点**：添加工具需要重新构建。没有热加载，没有 Plugin 系统。特性标志是二元的——工具要么在要么不在，运行时无法按用户自定义。

## 对比表

| 维度 | hermes | openclaw | NemoClaw | Claude Code |
|-----------|--------|----------|----------|-------------|
| **注册** | `registry.register()` 导入时 | `api.registerTool()` Plugin SDK | N/A（不是 Agent） | 静态导入 + 特性标志 DCE |
| **工具数量** | 50+ | 100+ | 仅管理 CLI | 60+ |
| **第三方可扩展** | 否（需要 Fork） | 是（Plugin SDK） | N/A | 否（需要重新构建） |
| **并行策略** | 路径独立性，最多 8 | `isConcurrencySafe` 标志 | N/A | 批次拆分，最多 10 |
| **安全模型** | 正则模式 + 注入扫描 | DM 配对 + 执行审批 + 限流 | 操作系统级（Landlock/seccomp） | 权限模式 + 企业策略 |
| **MCP 出站** | 基础 | 一等 Plugin 集成 | N/A | 完整支持，工具呈现为原生 |
| **Skill Store** | `~/.hermes/skills/` 自动创建 | 工作区技能，手动 | N/A | `~/.claude/skills/` 半自动 |
| **类型安全** | Python 类型提示 | Zod schema | N/A | `Tool<I,O,P>` 泛型 |
| **运行时开销** | 最小（导入时） | Plugin 生命周期成本 | N/A | 零（构建时 DCE） |

## 最佳实践

**如果你只做一件事**：从 hermes 风格的自注册开始。定义一个 `Tool` 接口，让每个工具文件在导入时自行注册，存储在一个以名称为键的字典中。不到 100 行代码你就有了一个可用的工具系统。只有在第三方需要扩展你的 Agent 时才升级到 Plugin SDK。

**来自四个项目的五条规则**：

1. **工具声明自己。** 工具知道自己的名称、描述、参数 schema 和安全属性。不要把这些放在中心配置文件里——放在实现旁边。四个项目都这么做。

2. **执行前验证输入。** hermes 用参数 schema。openclaw 用 Zod。Claude Code 用 TypeScript 泛型。选择你的方式，但一定要验证。LLM 发送格式错误的参数的概率大约是 2-5%。

3. **返回结构化结果。** 不是字符串。工具结果应该包含：成功/失败状态、实际输出、所有副作用（创建的文件、启动的进程）以及资源使用情况。Claude Code 的 `ToolResult` 是这里的黄金标准。

4. **安全是按工具的，不是按层的。** "这安全吗？"的决策取决于工具做什么以及收到了什么参数。笼统的"所有 shell 命令都需要审批"太粗糙。笼统的"自动批准一切"太危险。Claude Code 的按类别权限模式恰到好处。

5. **技能文件是 Tools 的下游输出，Core 的上游输入。** Tools 写入它们（通过技能 CRUD 操作）。Core 读取它们（通过 prompt 构建器）。这种分离是使学习循环成为可能的关键。

## 分步指南：如何构建

### 第 1 步：定义 Tool 接口

每个工具需要：名称（唯一字符串）、描述（给 LLM 看的）、参数 schema（用于验证）和执行函数。

```typescript
// contracts/tool.ts
interface Tool {
  name: string;
  description: string;
  parameters: JSONSchema;          // 或 Zod schema
  isConcurrencySafe?: boolean;     // 默认 false
  requiresApproval?: boolean;      // 默认 false
  execute(params: unknown): Promise<ToolResult>;
}

interface ToolResult {
  success: boolean;
  output: unknown;
  sideEffects?: string[];          // "created file: /tmp/foo.txt"
}
```

**决策点**：Zod 还是 JSON Schema？如果你的代码库是 TypeScript，用 Zod——它一次性给你运行时验证和类型推断。如果你用 Python，用 Pydantic 模型或纯 JSON Schema。

### 第 2 步：构建注册表

一个字典。就这样。不要过度复杂化。

```typescript
// tools/registry.ts
const registry = new Map<string, Tool>();

function register(tool: Tool): void {
  if (registry.has(tool.name)) {
    throw new Error(`Tool "${tool.name}" already registered`);
  }
  registry.set(tool.name, tool);
}

function getTool(name: string): Tool | undefined {
  return registry.get(name);
}

function getAllTools(): Tool[] {
  return Array.from(registry.values());
}
```

### 第 3 步：创建你的第一个工具

```typescript
// tools/builtins/shell.ts
import { register } from "../registry";

register({
  name: "shell_exec",
  description: "Execute a shell command and return stdout/stderr",
  parameters: {
    type: "object",
    properties: {
      command: { type: "string", description: "The command to run" },
      workdir: { type: "string", description: "Working directory" },
      timeout: { type: "number", description: "Timeout in ms", default: 30000 }
    },
    required: ["command"]
  },
  isConcurrencySafe: false,  // shell 命令可能冲突
  requiresApproval: true,     // shell 始终需要审批
  execute: async (params) => {
    const { command, workdir, timeout } = params as ShellParams;
    // ... 实际执行
  }
});
```

### 第 4 步：构建分派器

分派器位于 Agent Core 和工具之间。它接收来自 LLM 响应的工具调用，进行验证、检查审批并执行。

```typescript
// tools/dispatcher.ts
async function dispatch(toolCalls: ToolCall[]): Promise<ToolResult[]> {
  // 1. 查找所有工具
  const resolved = toolCalls.map(call => {
    const tool = getTool(call.name);
    if (!tool) return { call, tool: null, error: `Unknown tool: ${call.name}` };
    return { call, tool, error: null };
  });

  // 2. 对需要审批的工具检查审批
  for (const item of resolved) {
    if (item.tool?.requiresApproval) {
      const approved = await requestApproval(item.call);  // ← 你的 UI Hook
      if (!approved) {
        item.error = "User denied";
      }
    }
  }

  // 3. 拆分为可并发的和需要串行的
  const safe = resolved.filter(r => r.tool?.isConcurrencySafe && !r.error);
  const sequential = resolved.filter(r => !r.tool?.isConcurrencySafe && !r.error);

  // 4. 安全工具并行运行，串行工具按顺序运行
  const results = await Promise.all([
    Promise.all(safe.map(r => r.tool!.execute(r.call.arguments))),
    runSequential(sequential)
  ]);

  return flatten(results);
}
```

**决策点**：并行多少个工具？hermes 上限 8 个，Claude Code 上限 10 个。从 5 开始，根据你的工作负载增加。瓶颈通常是 I/O（网络、磁盘），不是 CPU。

### 第 5 步：添加 MCP 出站支持

MCP 出站意味着你的 Agent 连接到外部 MCP 服务器以发现更多工具。这些外部工具与内置工具注册在同一个注册表中。

```typescript
// tools/mcp-client.ts
async function connectMCPServer(config: MCPServerConfig): Promise<void> {
  const client = new MCPClient(config.uri);
  const remoteTools = await client.listTools();

  for (const remote of remoteTools) {
    register({
      name: `mcp_${config.name}_${remote.name}`,  // 命名空间以避免冲突
      description: remote.description,
      parameters: remote.inputSchema,
      isConcurrencySafe: true,  // MCP 调用通常是独立的
      execute: async (params) => {
        return await client.callTool(remote.name, params);
      }
    });
  }
}
```

**决策点**：MCP 工具要不要加命名空间？hermes 不加（有冲突风险）。Claude Code 加上服务器名称前缀（冗长但安全）。加上命名空间。LLM 能处理更长的名称。

### 第 6 步：设置 Skill Store

```
~/.your-agent/skills/
├── code-review.md         # YAML frontmatter + markdown 正文
├── debugging-strategy.md
└── api-design.md
```

每个技能文件：

```markdown
---
name: code-review
version: 1
created: 2026-04-01
tags: [development, review]
---

# Code Review Methodology

When reviewing code, follow these steps:
1. Check for security issues first...
2. ...
```

技能 CRUD 工具管理这个目录。Agent Core 中的 prompt 构建器扫描它，将相关技能注入系统 prompt。

## 常见陷阱

**1. God class 注册表。** hermes 的 `AIAgent` 类（9431 行）把工具注册、分派、执行和审批混在了一个对象里。当你有 30 个工具时，这个类就无法维护了。从一开始就把注册表、分派器和各个工具作为独立的关注点。

**2. 信任 LLM 生成的参数。** 模型会向你的文件读取工具发送 `{"file": "/etc/passwd"}`。它会向你的 shell 工具发送 `{"command": "sudo rm -rf /"}`。验证一切。hermes 基于正则表达式的检测能抓到明显的情况，但会遗漏编码技巧。Claude Code 的按工具权限系统更加健壮。

**3. 忽视工具结果大小。** 运行 `find /` 的 `shell_exec` 调用会返回几兆字节。这会撑爆你的上下文窗口。Claude Code 将工具结果截断到可配置的限制。hermes 的 Context Manager 将修剪旧工具结果作为第一步压缩措施。从第一天起就在 ToolResult 中内建结果大小限制。

**4. 过度并行化。** 并发运行 8 个文件编辑操作会损坏文件。openclaw 的 `isConcurrencySafe` 标志是正确的思路——每个工具声明自己的安全性。如果你不想要每工具标志，至少区分只读工具（可安全并行）和写入工具（默认串行）。

**5. 硬编码工具描述。** 当你有 60 个工具时，合并的描述可能占用系统 prompt 的 3,000+ token。Claude Code 根据上下文动态生成工具描述。hermes 始终包含所有工具。先全部包含；当你遇到上下文限制时再优化。

**6. 过早构建 Plugin SDK。** openclaw 的 Plugin SDK（50+ 子路径导出、生命周期 Hook、类型化 API）是令人印象深刻的工程。它也花了数月来构建和稳定。如果你没有第三方开发者，你就不需要它。hermes 的自注册模式完全能满足个人和小团队项目的需求。

## 跨模块契约

### Tools 对其他模块的期望

| 来源模块 | Tools 需要什么 | 契约 |
|--------------|-----------------|----------|
| **Agent Core** | 从 LLM 响应中提取的工具调用 | `ToolCall { name: string, arguments: object, id: string }` |
| **Agent Core** | 对需要审批的工具的审批决策（批准/拒绝） | `requestApproval(call: ToolCall) → Promise<boolean>` |
| **Model** | 无直接依赖——Tools 和 Model 是对等服务 | （无直接依赖） |

### Tools 向其他模块提供什么

| 目标模块 | Tools 提供什么 | 契约 |
|--------------|-------------------|----------|
| **Agent Core** | 工具执行结果 | `ToolResult { success: boolean, output: unknown, sideEffects?: string[] }` |
| **Agent Core** | 可用工具列表（用于系统 prompt） | `getAllTools() → Tool[]` 包含 name、description、parameters |
| **Agent Core** | 用于 prompt 注入的技能文件 | `~/.agent/skills/` 中带 YAML frontmatter 的文件 |
| **Context Manager** | 工具结果内容（可能需要截断） | ToolResult.output 是截断目标 |
| **Memory** | 无直接依赖——Memory 和 Tools 是对等服务 | （无直接依赖，但 Memory 工具注册在工具注册表中） |

### 避免方向混淆

- **MCP 出站**（Agent 调用外部 MCP 服务器获取工具）= Tools 模块。
- **MCP 入站**（外部系统调用你的 Agent）= 接口模块。
- **安全审批**（这个工具该不该运行？）= 在工具分派流程内部，不是单独的中间件。
- **Skill Store** 生命周期 = Tools 写入它。Agent Core 读取它。Memory 不碰它。
