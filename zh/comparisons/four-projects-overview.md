# 四个项目：并排对比

四个开源 Agent 项目。四种不同的哲学。

| 项目 | 哲学 | 一句话概括 |
|---------|-----------|-----------|
| **hermes-agent** | 学习 | 一个从自身错误中进化的 Agent |
| **openclaw** | 连接 | 一个将 20+ 平台统一管理的网关 |
| **NemoClaw** | 安全 | 一个让 Agent 在生产中安全运行的沙箱 |
| **Claude Code** | 速度 | 一个为开发者吞吐量优化的 CLI Agent |

我们经过五轮分析，阅读了这四个项目约 6000 万字符的源代码。本页将我们的发现提炼为对比表格，5 分钟即可浏览完毕。

---

## 静态架构对比

每个项目如何实现 10 个核心架构模块。

从左到右逐行阅读。**加粗** 标记该模块的最强实现。

| 模块 | hermes | openclaw | NemoClaw | Claude Code |
|--------|--------|----------|----------|-------------|
| **Security** | `approval.py` 正则 + Docker 可选 | DM 配对 + 执行审批 + 速率限制 | **Landlock + 逐二进制网络策略 + seccomp** | 权限模式（ask/auto）+ 企业策略 |
| **Interface** | 16 个适配器 + CLI + ACP | **20+ 渠道 Plugin，Adapter Composition** | 仅管理 CLI | 多入口，无统一抽象 |
| **Gateway** | `gateway/run.py`（7620 行），CLI 绕过它 | **WebSocket 控制面，25+ 处理器组** | 封装 OpenShell | 不存在 |
| **Routing** | 无（单 Agent） | **9 级绑定优先级，独立模块** | 不适用 | 不存在 |
| **Agent Core** | `AIAgent` 类，9431 行上帝类 | **最干净的分离：`runEmbeddedPiAgent()`** | 使用 OpenClaw 的实现 | `query.ts` + `REPL.tsx` 双循环 |
| **Model** | 直接调用 OpenAI SDK，无抽象 | **故障转移候选链 + 冷却探测** | 配置层（`inference-config.ts`） | `services/api/` 配合干净的 DI |
| **Tools** | Registry 自注册，50+ 工具 | **Plugin SDK 注册，100+ 工具** | 仅管理命令 | `Tool<I,O,P>` 泛型，60+ 工具 |
| **Memory** | **8 种可插拔后端，基于 `MemoryProvider` ABC** | 双重注册 + "dreaming"后台清理 | 不适用 | `SessionMemory` + 后台提取 |
| **Context** | Compressor：先裁剪，再摘要 | **可插拔上下文引擎 + 主动压缩** | 不适用 | 三种策略：auto / reactive / snip |
| **Runtime/Infra** | Docker / Modal / SSH 多后端 | Docker / Fly.io / Tailscale | **OpenShell Blueprint + Landlock + seccomp** | npm install，作为本地进程运行 |

### 如何阅读此表

几点立刻可以看出来：

- **Security**：NemoClaw 独处另一个层级。Landlock 文件系统隔离 + 逐二进制网络策略 + seccomp 系统调用过滤。其他三个项目把安全当作事后补丁。
- **Gateway / Routing**：只在多用户或多 Agent 部署时才重要。Claude Code 完全跳过两者，对于单用户 CLI 来说这是正确的决策。
- **Memory**：hermes 拥有最好的抽象（8 种可插拔后端）。openclaw 拥有最好的运行时行为（后台"dreaming"清理）。Claude Code 采用极简方式。
- **Context**：每个项目的解法都不同。没有一个对自己的方案完全满意。

### 工具注册与执行模式

值得单独看一下，因为工具设计差异巨大。

| 方面 | hermes | openclaw | Claude Code |
|--------|--------|----------|-------------|
| **注册** | 导入时 `registry.register()` | 通过 Plugin SDK `api.registerTool()` | 静态 import + feature flag 死代码消除 |
| **发现** | 所有已注册工具对所有 Agent 可用 | 通过配置设定每 Agent 的工具白名单 | 内置集 + MCP 工具在启动时发现 |
| **并行执行** | 路径无关性检测，最多 8 线程 | 每工具 `isConcurrencySafe` 标志 | 批次拆分，最多 10 并发 |
| **安全检查** | `approval.py` 对 shell 命令进行正则匹配 | 通过 DM 的执行审批流程 | `Tool.checkPermissions()` + 权限模式 |
| **第三方扩展** | 不支持 | 完整 Plugin SDK + NPM 分发 | 仅 MCP servers |

---

## 动态行为对比

涌现行为是指没有任何单一组件被设计来产生的能力。它们从组件间的交互中涌现。

hermes 中没有 `SelfEvolutionEngine` 类。Claude Code 中没有 `SpeculativeExecutor`。这些行为从简单组件做简单事情的组合中涌现。

| 行为 | hermes | openclaw | NemoClaw | Claude Code |
|----------|--------|----------|----------|-------------|
| **Learning Loop** | **SKILLS_GUIDANCE 指令 -> skill_manage CRUD -> prompt_builder 重新加载 -> skill_patch 修复。唯一拥有完整闭环的项目。** | 存在 Workspace skills 但无自动创建 | 不适用 | `/remember` + 自动提取。保存事实，不保存可复用方法论 |
| **Speculative Execution** | 否。严格串行执行 | 否。标准请求-响应 | 不适用 | **SpeculationState 跟踪并行的权限请求 + 工具执行。CompletionBoundary 处理回滚。** |
| **Self-Healing Model Routing** | `fallback_model` 配置，仅单次降级 | **ModelCandidate[] 链 -> auth profile 轮换 -> 冷却标记 -> 瞬态探测 -> 自动恢复** | 多 Provider 配置，手动切换 | 多 Provider 支持，无自动故障转移链 |
| **Context Pressure Cascade** | L1 裁剪工具结果 -> L2 LLM 摘要 -> budget_warning 注入 | **可插拔上下文引擎 + 主动压缩 + transcript 改写** | 不适用 | 三种并行策略：auto（80% 阈值）+ reactive（背压）+ snip（细粒度） |
| **Sub-Agent Delegation** | `delegate_tool` -> 独立预算 -> MAX_DEPTH=2 -> 最多 3 并行 | Lane 队列 + subagent hooks | 不适用 | **AgentTool：worktree 隔离 + fork + team/swarm + coordinator + async** |
| **Multi-Agent Specialization** | 否。单 Agent 架构 | **9 级绑定路由 -> 按对话分配 Agent -> 每个 Agent 拥有独立的 Tools/Memory/persona** | 不适用 | 否。单用户，单 Agent |
| **Self-Healing Runtime** | 无运行时健康管理 | 否 | **`gateway-state.ts` 健康分类 -> 检测网关死亡 -> SSH + curl 探测 -> 自动重启 -> 注册恢复** | 否 |

**加粗** = 该行为的最佳实现。

### "涌现"在实践中的含义

以 hermes 的 **Learning Loop** 为例：

1. `SKILLS_GUIDANCE` prompt 指令告诉 LLM 保存有用的模式（Agent Core）
2. LLM 判断"这值得保存"并调用 `skill_manage create`（Model + Tools）
3. 一个 markdown 技能文件被写入 `~/.hermes/skills/`（Skill Store）
4. 下次对话时，`prompt_builder` 扫描技能目录并注入相关技能（Agent Core）
5. LLM 行为因加载的技能而改变（Model）
6. 如果技能有误，LLM 调用 `skill_manage patch` 修复（Tools）

没有任何单一组件"做"学习。五个组件各做一件简单的事，产出了一个随时间进步的 Agent。

### NemoClaw 说明

NemoClaw 在大多数动态行为中显示"不适用"，因为它不是一个 Agent。它是一个封装 OpenClaw 的安全运行时。它唯一的涌现行为——Self-Healing Runtime——是所有项目中唯一在基础设施层面实现的。

---

## 各模块：参考哪个项目

构建你自己的 Agent？以下是每个模块应该研究的项目。

| 模块 | 推荐参考 | 原因 |
|--------|----------------------|-----|
| Agent Core | openclaw（最干净）或 hermes（最简单） | openclaw 关注点分离好；hermes 一个文件搞定 |
| Model 抽象 | openclaw 故障转移 + Claude Code DI | 候选链 + 冷却探测在生产中至关重要；DI 让测试更容易 |
| Tool 注册 | hermes（简单）或 openclaw（平台级） | 不需要第三方扩展？用 hermes。需要 Plugin 生态？用 openclaw |
| Memory | hermes Provider ABC + openclaw 双重注册 | hermes 抽象最佳；openclaw 双重角色实现最佳 |
| Context 管理 | Claude Code 三策略或 openclaw 可插拔引擎 | 三策略 = 最灵活；可插拔引擎 = 最可扩展 |
| Gateway | openclaw | 唯一正确实现的项目 |
| Security | NemoClaw | 唯一认真对待的项目 |
| Runtime / Infra | NemoClaw | 唯一真正构建了的项目 |

---

## 项目统计

| 统计项 | hermes | openclaw | NemoClaw | Claude Code |
|------|--------|----------|----------|-------------|
| **语言** | Python 3.11+ | TypeScript ESM | JS + TS | TS + React + Bun |
| **大致行数** | ~80k | ~300k | ~20k | ~200k |
| **核心创新** | 闭环学习（从经验中学习） | Plugin 生态 + 多平台统一 | 操作系统级别隔离的安全沙箱 | Speculative Execution + 企业级性能 |
| **团队** | Nous Research | 社区 | NVIDIA | Anthropic |
| **成熟度** | 生产就绪 | 生产就绪 | Alpha | 生产级（商业） |
| **能否独立运行** | 是 | 是 | 否（需要 OpenClaw） | 是 |
| **开源** | 是 | 是 | 是 | 源码可获取（泄露构建） |
| **上帝类风险** | 高（`AIAgent` 9431 行） | 低（干净分离） | 低（范围简单） | 中（`REPL.tsx` 5006 行） |
| **最适合学习** | Learning Loop、Memory、Tool 注册 | Gateway、Routing、Model 故障转移、Plugin SDK | Security、Runtime 隔离 | 性能、子 Agent 模式、Context 策略 |

---

## 架构模式：Hub-and-Spoke

四个项目独立验证了相同的拓扑。Agent 架构**不是分层的**，而是 Hub-and-Spoke。

```
                    ┌─────────┐
                    │  Model  │
                    └────┬────┘
                         │
          ┌──────────┐   │   ┌─────────┐
          │  Memory  ├───┼───┤  Tools  │
          └──────────┘   │   └─────────┘
                         │
                  ┌──────┴──────┐
                  │ AGENT CORE  │
                  │ (hub)       │
                  └──────┬──────┘
                         │
          ┌──────────┐   │   ┌──────────────┐
          │ Context  ├───┘   │ Session Store │
          │ Manager  │       └──────────────┘
          └──────────┘
```

Agent Core 位于中心。Model、Tools、Memory 和 Context Manager 是**对等服务**——不是相互堆叠的层级。它们不互相调用，都与 hub 通信。

我们最初画了一个分层图。那是错的。经过五轮修订才得出 Hub-and-Spoke。

---

## 核心要点

1. **没有任何单一项目做到面面俱到。** 每个项目都做出了反映其哲学的权衡取舍。
2. **架构是 Hub-and-Spoke，不是分层的。** 四个项目独立验证了这一点。
3. **最有价值的能力是涌现的。** Learning Loop、Speculative Execution 和 Self-Healing Routing 不是作为功能设计的，它们从组件交互中涌现。
4. **安全是最被忽视的领域。** 只有 NemoClaw 认真对待。其他三个把它当事后补丁。
5. **上下文管理普遍困难。** 每个运行 LLM 循环的项目都发明了自己的多级压缩策略。
