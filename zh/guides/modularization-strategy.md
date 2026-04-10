# 模块化策略：何时拆分，何时不拆

> 没有任何项目把核心拆成独立包。4 个项目全部把核心放在同一个包里。教训很明确：清晰的边界零成本，物理拆分的代价是集成测试、版本兼容和契约变更。

---

## 现实检视

我们阅读了 4 个生产级 Agent 项目共 60 万行源代码。以下是每个项目的代码组织方式：

| 项目 | 组织方式 | 代价 |
|---------|-------------|------|
| hermes-agent | 单仓库，完全耦合 | `run_agent.py` 膨胀到 9,431 行——一个上帝类。改一处，崩五处 |
| openclaw | Monorepo，核心耦合，扩展独立 | 添加渠道/工具毫无阻力。重构上下文引擎需要动大手术 |
| Claude Code | 单包，直接 import | 启动最快，迭代最快。`REPL.tsx` 5,006 行混合了 5 个层级——新人难以理解 |
| NemoClaw | CLI / Plugin / Blueprint 松散分离 | 功能集最简单。无需处理跨模块涌现行为的复杂性 |

它们都没有把 Agent Core、Model、Context Manager 或 Tools 拆成独立包。四个项目，零拆分。

---

## 核心洞察：边界不等于拆分

这是两件不同的事：

| | 物理拆分 | 清晰边界 |
|---|---|---|
| **定义** | 独立的 package.json，独立的版本号 | 模块通过接口通信，从不访问内部实现 |
| **解决的问题** | 独立部署、独立版本控制、第三方扩展 | 改一个模块不破坏其他模块。可替换。可测试 |
| **代价** | 契约变更需要跨包更新、版本兼容、集成测试 | 几乎为零——这只是良好的代码组织 |

openclaw 最能说明这一点。他们的核心（`src/`）是一个包。扩展（`extensions/`）是独立的包。为什么？核心变更频繁且模块紧密耦合。扩展很少变更且彼此独立。

**边界关乎接口。拆分关乎部署。**

从第一天起就把接口放在 `contracts/` 目录中。这就是边界，零成本。在有理由之前不要创建独立包。

---

## 三阶段演进

### 阶段 1：从 0 到 1 —— Monorepo 单包 + 目录边界

```
agent/
├── core/           # Agent Core —— 对话循环、上下文构建
├── model/          # Model —— Provider 路由、故障转移
├── tools/          # Tools —— 注册、分发、执行
│   └── builtins/   # 内置工具（shell、file、web...）
├── memory/         # Memory —— 双重角色、Provider 接口
├── context/        # Context Manager —— 压缩、裁剪
├── interfaces/     # Interface —— CLI 适配器
├── contracts/      # 跨模块接口定义（TS interfaces / Python ABCs）
└── tests/
    ├── unit/       # 各模块单元测试
    └── loops/      # 涌现行为循环测试 <-- 关键
```

**为什么不拆？** 因为你还不知道边界应该划在哪里。

hermes 把 Memory 接口做对了——一个干净的 `MemoryProvider` ABC，支持 8 种可插拔后端。漂亮的抽象。但他们把 Model 调用直接嵌入了上帝类。同一个团队，同一个项目，一个边界对了，一个边界错了。

在单包中，修正错误的边界是一次重构。代价：数小时。跨包修正错误的边界意味着变更契约、更新版本、修改每个消费者。代价：数天到数周。

在确定哪些边界是稳定的之前，保持在一个包里。

### 阶段 2：从 1 到 N —— 提取稳定的叶子节点

```
agent/
├── core/               ┐
├── model/              │  仍然是一个包（核心）
├── context/            │  变更频繁，紧密耦合，一起发布
├── contracts/          ┘
│
├── packages/
│   ├── tool-shell/          <-- 独立包（稳定后提取）
│   ├── tool-browser/        <-- 独立包
│   ├── tool-mcp-client/     <-- 独立包
│   ├── memory-sqlite/       <-- 独立包
│   ├── memory-vector/       <-- 独立包
│   ├── channel-telegram/    <-- 独立包
│   └── channel-discord/     <-- 独立包
```

**规则**：接口 3 个月无变更 = 稳定 = 可以提取。

过早提取会锁定仍在演进的接口。hermes 的 Memory 接口已经稳定了数月——是很好的提取候选。他们的 Model 集成仍然缠绕在上帝类中——是糟糕的提取候选。

**可提取对象**（叶子节点）：
- 单个工具（shell、browser、web search）
- 渠道适配器（telegram、discord、slack）
- Memory Provider（sqlite、vector、redis）
- Model Provider（openai、anthropic、bedrock）

**绝不提取**（核心节点）：
- Agent Core
- Context Manager
- Model 抽象层（故障转移逻辑）

### 阶段 3：从 N 到平台 —— Plugin SDK

```
在阶段 2 基础上添加：
├── sdk/                # 发布的契约包（类似 openclaw/plugin-sdk）
│   ├── tool.ts         # Tool 接口
│   ├── channel.ts      # Channel 接口
│   ├── memory.ts       # MemoryProvider 接口
│   └── provider.ts     # ModelProvider 接口
```

**只有在第三方需要扩展你的 Agent 时才这样做。** openclaw 的 50+ 子路径导出的 Plugin SDK 解决的是平台级问题。如果你不是在构建平台，这就是过度工程。

---

## 什么该拆 vs. 什么不该拆

| 模块 | 能拆？ | 阶段 | 原因 |
|--------|-----------|-------|--------|
| 单个工具（shell、browser、web） | 能 | 阶段 2 | 每个工具是独立的，只需符合 Tool 接口 |
| 渠道适配器（telegram、discord） | 能 | 阶段 2 | 每个适配器是独立的，只需符合 Channel 接口 |
| Memory Provider（sqlite、vector） | 能 | 阶段 2 | 每个 Provider 是独立的，只需符合 MemoryProvider 接口 |
| Model Provider（openai、anthropic） | 能 | 阶段 2 | 每个 Provider 是独立的，只需符合 Provider 接口 |
| Gateway | 也许 | 阶段 2+ | 天然独立，通过标准化消息格式与 Core 通信 |
| Agent Core | 不能 | 永不 | 与 Context Manager 耦合最紧。变更频率最高 |
| Context Manager | 不能 | 永不 | 消息格式与 Agent Core 强耦合 |
| Model 抽象层（故障转移逻辑） | 不能 | 永不 | 与 Agent Core 的对话循环紧密交互 |

**一句话原则：叶子节点自由拆分，核心节点绝不拆分。**

---

## openclaw 的经验

openclaw 把握住了正确的权衡：核心紧耦合，扩展松耦合。

他们的 `src/` 目录是一个包。Agent Core、Model 路由、Context Engine、Hook 系统——全部紧密耦合，一起发布。改上下文引擎意味着要碰 Agent 循环。没问题，它们在同一个包里。

他们的 `extensions/` 目录有数十个独立包。每个渠道适配器、每个工具 Plugin、每个集成——独立的包、独立的测试、独立的发布周期。添加新的 Telegram 功能不会碰到 Discord。添加新工具不会碰到 shell 工具。

结果：核心迭代保持快速（无跨包开销），扩展开发保持安全（不会意外破坏核心）。

这就是值得遵循的模型。

---

## 常见错误

**过早拆分。** 你在第一天就创建了 `@myagent/core`、`@myagent/model`、`@myagent/tools`。三周后你意识到 Context Manager 需要一个新的消息字段。现在你需要更新 `@myagent/core`，升版本号，更新 `@myagent/model` 以依赖新版本，更新 `@myagent/tools` 因为它也读消息，跑三个包的集成测试。本来 5 分钟的重构变成了 2 小时的依赖之舞。

**从不拆分。** 你把所有东西永远放在一个包里。你的 `tools/` 目录有 47 个工具文件，每个有不同的依赖，有些引入了沉重的库。每次安装都下载所有东西，每次测试都运行所有东西。叶子节点最终应该分离。

**拆分核心而非叶子。** 你把 Agent Core 提取成独立包"追求干净架构"。现在对话循环的每次改动都需要升版本和重新集成。变更最频繁的模块反而有了最高的协调成本。方向错了。

---

## 决策清单

将一个模块提取为独立包之前，回答以下问题：

1. 接口是否已稳定 3 个月以上？如果没有，不拆。
2. 它是叶子节点（工具、适配器、Provider）还是核心节点（Agent Core、Context Manager）？如果是核心，不拆。
3. 它是否有独立的依赖导致主包臃肿？如果是，拆分的理由更充分。
4. 第三方是否需要实现这个接口？如果是，拆分 + 将接口作为 SDK 发布。
5. 提取后是否能简化主包的测试套件？如果不能，拆分只会增加开销而没有收益。

如果你对第 1 和第 2 题的回答都是"否"，停下来。把它留在主包里。

---

*基于 hermes-agent、openclaw、NemoClaw 和 Claude Code 的模块化模式。详细分析请参见 [架构参考模型](../../docs/agent-architecture-reference.md) 第 8 节。*
