# 我们的发现 vs Claude Certified Architect

[Claude Certified Architect -- Foundations](https://github.com/paullarionov/claude-certified-architect) 认证涵盖 5 个考试领域：Agent 架构（27%）、工具设计与 MCP（18%）、Claude Code 工作流（20%）、prompt 工程（20%）和上下文管理（15%）。

我们独立分析了 hermes-agent、openclaw、NemoClaw 和 Claude Code 约 60 万行源代码。

**结论：** 我们验证了认证教授的每个核心架构概念。同时也发现了认证未涵盖的内容——涌现行为、操作系统级安全、性能技巧以及从真实代码库中学到的模块化经验。

### 如何阅读本文档

- **第 1 节**：认证教授的、我们独立验证的概念。相同结论，不同路径。
- **第 2 节**：我们发现的、认证未涵盖的内容。源代码揭示了理论无法展示的东西。
- **第 3 节**：我们有意跳过的认证主题，因为它们不属于运行时架构范畴。
- **第 4 节**：重叠关系的可视化总结。

---

## 1. 我们独立验证了什么

来自认证材料的九个概念。我们通过阅读四个项目的源代码全部验证了它们。每一次，我们都通过完全不同的路径——经验观察而非理论——得出了相同的结论。

| 概念 | 认证怎么说 | 我们在代码中发现了什么 |
|---------|-----------|----------------------|
| **Hub-and-Spoke Architecture** | 协调者 Agent 位于中心，子 Agent 在隔离的上下文中 | 我们最初画了一个分层模型。那是错的。四个项目都使用 Hub-and-Spoke——Agent Core 位于中心，Model/Tools/Memory/Context 作为对等服务。经过 5 轮修订才做对。 |
| **Agentic Loop** | Model 生成 tool_call -> 应用执行 -> 循环直到 end_turn | 所有 4 个项目：`while(budget > 0): call model -> exec tools -> repeat`。三种实现变体：同步 while（hermes）、async generator（Claude Code）、Lane 并发（openclaw）。 |
| **Sub-Agent Isolation** | 子 Agent 不能继承协调者的历史；所有上下文必须显式传递 | hermes：独立的 `AIAgent` 实例，不共享 Memory 写入，MAX_DEPTH=2。Claude Code：worktree/fork/team 模式。父 Agent 只接收摘要。 |
| **Hooks vs Prompts** | 关键业务规则使用 Hook（而非 prompt）。Hook 是确定性的；prompt 是概率性的 | openclaw Hook 系统 = 代码级拦截，总是执行。hermes SKILLS_GUIDANCE = prompt 引导，LLM 可以忽略。我们在自己的设计中也遇到了这个问题。 |
| **Context Management** | 工具结果消耗上下文窗口；渐进式摘要在数字和日期上会丢失精度 | hermes L1 裁剪 -> L2 摘要级联。Claude Code：auto/reactive/snip。openclaw：可插拔上下文引擎。我们将其抽象为"Context Pressure Cascade"涌现行为。 |
| **Task Decomposition** | 固定管道 vs 动态自适应 | hermes 同步循环 = 固定。Claude Code async generator + Speculative Execution = 动态。openclaw Lane 控制 = 混合。 |
| **MCP Protocol** | Tools、Resources、Prompts——三种能力类型 | 已验证。我们补充了认证未强调的一个关键区分：MCP 入站（他人调用你的 Agent）= Interface 层。MCP 出站（Agent 调用外部服务器）= Tools 层。搞混方向，你的架构图就是错的。 |
| **Escalation Triggers** | 基于显式条件，而非情感分析或模型自评置信度 | hermes `approval.py`：对危险命令的模式匹配。NemoClaw：操作系统级逐二进制网络策略。两者都是确定性的。 |
| **Error Classification** | 4 种类型：transient / validation / business / permission | openclaw 的 Model 故障转移建立在此基础上：rate_limit（transient）-> 触发候选链。billing/auth（non-transient）-> 不自动重试。分类直接塑造了自愈路由的设计。 |

### 验证强度

验证最强的是 Hub-and-Spoke 和 Agentic Loop。我们在多次尝试替代模型（分层、管道等）失败后才得出这些结论。最弱的是 Task Decomposition——认证的固定-vs-动态框架有用但过于简化。真实项目两者兼有。

MCP 的发现值得强调：认证教授了 MCP 的三种能力类型，但没有强调**方向**。MCP 入站（他人调用你的 Agent）属于 Interface 层。MCP 出站（Agent 调用外部服务器）属于 Tools 层。搞错方向，你的整个架构图就是错的。我们差点就搞错了。

---

## 2. 我们的分析超越之处

认证教的是"如何用好 Claude Agent"。我们的分析问的是"Agent 内部长什么样？"

以下七个领域不在认证覆盖范围内。它们只有通过阅读源代码才能发现。

| 发现 | 它是什么 | 为什么重要 |
|---------|-----------|----------------|
| **涌现行为** | 没有任何单一组件被设计来产生的能力。它们从组件间的交互中涌现。 | Agent 最有价值的能力（学习、自愈、推测）都是涌现的。你无法通过孤立地构建组件来构建它们。 |
| **Learning Loop** | hermes：SKILLS_GUIDANCE -> skill_manage create -> prompt_builder 重新加载 -> skill_manage patch。Agent 从经验中进化。 | 不存在 `SelfEvolutionEngine` 类。四个简单组件组合产生自我改进。认证没有提及这个概念。 |
| **Security Envelope 深度** | NemoClaw：Landlock 文件系统隔离、逐二进制网络策略、seccomp 进程沙箱、SSRF 防护、凭据清洗。 | 认证涵盖权限模式（ask/auto）和企业策略。NemoClaw 在根本不同的层面运作——操作系统级进程隔离，Agent 甚至感知不到。 |
| **Speculative Execution** | Claude Code：SpeculationState 跟踪并行的权限请求 + 工具执行。CompletionBoundary 在用户拒绝时处理回滚。 | 这就是 Claude Code 感觉快的原因。当你在阅读权限提示时（约 2 秒），工具已经执行完了。认证中未讨论。 |
| **Self-Healing Model Routing** | openclaw：ModelCandidate[] 链 -> auth profile 轮换 -> 冷却标记 -> 瞬态探测 -> 自动恢复。零停机。 | 认证教授错误分类。openclaw 把分类变成了自动恢复。系统在用户无感知的情况下自愈。 |
| **垂直存储** | 持久化不是水平层。每个服务有自己的存储后端：Agent Core -> Session Store，Memory -> Memory Store，Tools -> Skill Store。Model 和 Context Manager 是无状态的。 | 认证不讨论持久化架构。我们最初把它画成水平层。那是错的。经过四轮修订才修正。 |
| **模块化策略** | 三阶段演进：阶段 1 Monorepo + 目录边界 -> 阶段 2 提取叶子节点 -> 阶段 3 Plugin SDK。核心原则：叶子自由拆分，核心绝不拆分。 | 认证教组件使用，不教代码组织。我们从四个项目的实际结构及其后果中提取了这一点（hermes 9431 行上帝类 vs openclaw 干净的扩展分离）。 |

### 涌现行为差距

这是认证与我们发现之间最大的差距。认证将 Agent 能力视为组件功能："Memory 组件存储东西"、"Tool 组件执行操作"。这是对的但不完整。

最有价值的能力不是组件功能。它们是**涌现的**——当多个组件以它们各自都未被设计的方式交互时产生。

hermes 没有"学习"模块。它有一个 prompt 指令（Core）、一个工具（Tools）、一个文件存储（Skill Store）和一个加载器（Core）。它们组合在一起，产生了一个从经验中学习并修复自身错误的 Agent。没有任何单一组件"做"学习。

认证无法教授这一点，因为这需要阅读实际实现才能看到它的发生。

---

## 3. 认证涵盖但我们未覆盖的内容

我们的分析聚焦于运行时架构。以下认证主题超出该范围。它们是有效的知识——只是不在我们的研究目标内。

| 认证主题 | 我们为什么没有覆盖 |
|-----------|------------------------|
| **Batch API**（50% 成本节省，24 小时处理） | API 使用优化，不是架构 |
| **JSON Schema Structured Output** | Prompt 工程，不是架构 |
| **Few-Shot Prompting**（5 种模式） | Prompt 工程，不是架构 |
| **CLAUDE.md Hierarchy**（user / project / directory） | 平台特性，不是内部架构 |
| **CI/CD Integration**（`-p` 标志，`--output-format json`） | DevOps 实践，不是 Agent 内部 |
| **Confidence Calibration + Stratified Random Sampling** | QA 方法论，不是核心架构 |
| **Data Provenance and Attribution** | 输出质量，不是运行时架构 |

这些不是认证的缺陷，而是超出我们范围的主题。如果你想要全面了解，两者都应该学习。

---

## 4. 重叠关系图

```
              Cert Knowledge                Our Source Code Analysis
          ┌──────────────────┐          ┌──────────────────────────┐
          │                  │          │                          │
          │  Batch API       │          │  Emergent Behaviors (6)  │
          │  JSON Schema     │          │  Learning Loop           │
          │  Few-shot        │          │  Security Envelope Depth │
          │  CLAUDE.md       │          │  Speculative Execution   │
          │  CI/CD           ├────┐     │  Self-Healing Routing    │
          │  Confidence Cal. │    │     │  Vertical Storage        │
          │  Provenance      │  ┌─┴───┐ │  Modularization Strategy │
          │                  │  │     │ │                          │
          └──────────────────┘  │BOTH │ └──────────────────────────┘
                                │     │
                           ┌────┴─────┴────┐
                           │ Hub-and-Spoke  │
                           │ Agentic Loop   │
                           │ Sub-Agent Iso. │
                           │ Hooks/Prompts  │
                           │ Context Mgmt   │
                           │ Task Decomp.   │
                           │ MCP Protocol   │
                           │ Escalation     │
                           │ Error Classes  │
                           └───────────────┘
```

### 数据统计

| | 仅认证 | 两者共有 | 仅我们的分析 |
|---|---|---|---|
| **数量** | 7 个主题 | 9 个概念 | 7 项发现 |
| **性质** | API 使用、prompting、DevOps、QA | 核心架构、模式、协议 | 涌现行为、安全深度、代码组织 |
| **来源** | 理论 + Anthropic 文档 | 理论与经验观察一致 | 仅源代码 |

---

## 总结

认证是一个理论端的知识体系："如何用 Claude Agent 构建。"

我们的分析是一项经验研究："Agent 内部到底长什么样？"

理论与证据在核心架构上达成一致。九个概念被独立验证。但源代码揭示了认证无法教授的东西：

- **涌现行为**——最有价值的能力不是设计出来的，它们涌现出来
- **安全深度**——操作系统级隔离 vs 应用级权限检查
- **性能技巧**——Speculative Execution、prompt 缓存、并行工具分发
- **代码组织经验**——过早拆分或过晚拆分的后果

两者互补。学认证获得广度，读代码获得深度。
