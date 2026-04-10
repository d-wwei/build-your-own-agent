# 测试涌现行为：循环测试

> 涌现行为依赖于跨模块的隐式契约。单元测试无法保护它们。你需要专门的循环测试。

---

## 问题所在

涌现行为是指没有任何单一模块被设计来做的事情，但当模块组合时系统就能做到。

hermes 没有 `SelfEvolutionEngine` 类。但 SKILLS_GUIDANCE 指令 + skill_manage 工具 + Skill Store 文件 + prompt_builder 加载 = 一个从经验中学习的系统。四个简单组件，一个复杂行为。

单独对每个组件进行单元测试？全部通过。改了技能文件格式？prompt_builder 仍然加载文件，skill_manage 仍然写文件。单元测试全绿。但学习循环已经坏了——prompt_builder 期望的格式与 skill_manage 写出的格式不匹配。涌现行为静默消亡。

你需要能够端到端测试完整循环的测试。

---

## 解决方案：`tests/loops/`

一个专门用于集成测试的目录，端到端验证涌现行为。

```
tests/loops/
├── learning-loop.test.ts
├── context-cascade.test.ts
├── model-failover.test.ts
├── delegation.test.ts
└── speculative-exec.test.ts
```

每个测试模拟一个涌现行为的完整周期。不 mock 内部实现，使用真实模块代码。只 mock 外部依赖（LLM API，某些情况下的文件系统）。

---

## 5 个循环测试

### 1. learning-loop.test

测试学习循环：经验导致技能创建，技能跨会话持久化，技能可被更新。

```
测试序列：
1. 在一次对话中执行 5 次工具调用
2. 验证 Skill Store 中创建了一个技能文件
3. 开始新会话
4. 验证 system prompt 包含了该技能内容
5. 通过 skill_patch 修改该技能
6. 开始另一个会话
7. 验证加载的技能反映了修改
```

**什么会破坏这个测试**：技能文件格式变更（frontmatter schema）、Skill Store 路径约定变更、prompt 注入顺序变更、技能 CRUD 工具参数变更。

**涉及的模块**：Agent Core、Tools、Skill Store、prompt builder。

### 2. context-cascade.test

测试压力升级：当上下文填满时，压缩策略按顺序激活。

```
测试序列：
1. 将上下文填充到 token 预算的 80%
2. 验证 L1 触发（裁剪旧的工具结果，零次 LLM 调用）
3. 继续填充上下文
4. 验证 L2 触发（LLM 驱动的中间轮次摘要）
5. 验证预算警告消息被注入到上下文中
6. 验证 LLM 看到了该警告（检查 messages 数组）
```

**什么会破坏这个测试**：压缩输入/输出格式变更（messages 数组结构）、预算警告注入位置或格式变更、token 估算接口变更、阈值变更但未更新测试。

**涉及的模块**：Agent Core、Context Manager、Model（用于 token 估算）。

### 3. model-failover.test

测试自愈循环：故障、降级、冷却、探测、恢复。

```
测试序列：
1. Mock 主模型返回 429（速率限制）
2. 验证 Agent 切换到备用模型
3. 验证 Agent 在备用模型上继续正常服务
4. Mock 冷却期已过
5. 验证向主模型发送了探测请求
6. Mock 主模型已恢复（200 OK）
7. 验证 Agent 切换回主模型
8. 验证正在进行的对话未中断
```

**什么会破坏这个测试**：错误分类枚举变更（rate_limit vs billing vs auth）、冷却状态格式变更、探测协议变更、新的错误类型未映射到分类。

**涉及的模块**：Model（内部——候选链、冷却、探测）。

### 4. delegation.test

测试子 Agent 生命周期：创建、约束、执行、总结、返回。

```
测试序列：
1. 触发一次委派（LLM 判断子任务可委派）
2. 验证子 Agent 获得了独立的 token 预算
3. 验证子 Agent 的工具集被限制（不允许递归委派）
4. 让子 Agent 完成任务
5. 验证子 Agent 返回了一条摘要消息
6. 验证父 Agent 的上下文恰好增长了 1 条消息（即摘要）
7. 验证父 Agent 继续正常运行
```

**什么会破坏这个测试**：子 Agent 构造函数参数变更、工具黑名单格式变更、摘要返回格式变更、深度限制执行方式变更。

**涉及的模块**：Agent Core（父 + 子实例）、Tools（限制逻辑）。

### 5. speculative-exec.test

测试并行权限 + 执行：Agent 在用户批准之前就开始工作。

```
测试序列：
1. LLM 返回一个需要权限的 tool_call
2. 验证权限请求和工具执行并行启动
3. Mock 用户批准
4. 验证结果立即显示（无执行延迟）
5. 重置。LLM 返回另一个 tool_call
6. Mock 用户拒绝
7. 验证结果被丢弃
8. 验证状态回滚到 CompletionBoundary
```

**什么会破坏这个测试**：ToolResult 回滚状态格式变更、权限请求异步取消机制变更、CompletionBoundary 格式变更、并行执行路径被移除。

**涉及的模块**：Agent Core、Tools、Interface（权限 UI）。

---

## 为什么循环测试有效

关键洞察：涌现行为完全由核心模块组成。核心模块永远不会被拆分成独立包（参见 [模块化策略](modularization-strategy.md)）。因此，循环测试始终与被测代码在同一个包中运行。

当你将叶子节点（某个具体工具、某个渠道适配器）提取为独立包时，没有循环测试会被破坏。循环测试使用 mock 的工具和 mock 的渠道。真实的工具包有自己的单元测试。

```
阶段 1（单包）：          循环测试直接运行。零配置。
阶段 2（叶子提取后）：    循环测试仍在核心包中运行。
                          所有循环参与者仍然在这里。
阶段 3（Plugin SDK 添加后）：循环测试不变。
                          SDK 消费者编写自己的测试。
```

这是有意为之，而非偶然。模块化策略保护了循环测试的有效性。

---

## 跨模块契约表

每个涌现行为都依赖于模块间的隐式契约。升级任何模块时，查阅此表以了解必须维护的契约。

| 涌现行为 | 关键契约 | 涉及模块 |
|---|---|---|
| Learning Loop | 技能文件格式（frontmatter schema）-- 存储路径约定 -- Prompt 注入方式 -- 技能 CRUD 参数 | Agent Core <-> Tools <-> Skill Store |
| Speculative Execution | ToolResult 支持回滚状态 -- 权限请求可异步取消 -- CompletionBoundary 格式 | Agent Core <-> Tools <-> Interface |
| Self-Healing Model Routing | 错误分类枚举（rate_limit / billing / auth）-- 冷却状态格式 -- 探测协议 | Model（内部） |
| Context Pressure Cascade | 压缩 I/O 格式（messages 数组）-- 预算警告注入位置和格式 -- Token 估算接口 | Agent Core <-> Context Manager <-> Model |
| Sub-Agent Delegation | 子 Agent 构造函数参数 -- 工具黑名单 -- 摘要返回格式 -- 深度限制 | Agent Core <-> Tools |

在变更任何这些契约之前，找到对应的循环测试并运行它。如果你的变更后测试通过，涌现行为存活。如果测试失败，你破坏了一个跨模块假设。

---

## 实践建议

**每个 PR 都运行循环测试。** 它们是集成测试但运行很快——没有真实 LLM 调用，没有真实文件系统（除非测试持久化）。Mock LLM 返回可预测的 tool_call。完整的循环测试套件在 10 秒内运行完毕。

**以行为命名测试，而不是模块。** 用 `learning-loop.test`，而不是 `skill-store-integration.test`。重点是测试涌现行为，而不是任何单个模块。

**当循环测试失败时，bug 在边界上。** 单个模块是正常的（单元测试证明了这一点）。失败出在模块间的契约上。检查消息格式、参数结构和注入点。

**添加新的涌现行为时就添加循环测试。** 如果你实现了 Model 故障转移，在发布前就添加 `model-failover.test`。如果该行为没有循环测试，第一次重构就会静默破坏它。

---

*基于 hermes-agent、openclaw、NemoClaw 和 Claude Code 的测试模式。原始分析请参见 [架构参考模型](../../docs/agent-architecture-reference.md) 第 8.5 和 8.6 节。*
