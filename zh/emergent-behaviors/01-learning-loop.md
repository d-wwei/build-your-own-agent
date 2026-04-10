# 涌现行为 1：学习循环（自我进化）

> 一句话总结：Agent 从自身经验中创建可复用的技能，在未来会话中加载它们，并在产生错误结果时修补 -- 不需要任何 SelfEvolutionEngine 类。

## 它是什么

学习循环是一个自我进化周期。在反复使用工具后，Agent 将有效的做法提炼成技能文件。下次会话时，该技能文件被注入系统提示。LLM 读取它，改变自身行为，并且 -- 如果技能有误 -- 通过创建该技能的同一个工具来修补它。

为什么它是"涌现"的：没有单个组件被设计用来做自我进化。`SKILLS_GUIDANCE` 是一段提示文本。`skill_manage` 是一个文件 CRUD 工具。`prompt_builder` 是一个目录扫描器。每个只做一件简单的事。组合在一起，系统从经验中学习。

hermes-agent 是唯一一个实现完整循环的项目。贴近度评分：10/10。Claude Code 得分 3/10 -- 它通过 `/remember` 提取事实，但从不创建可复用的方法论。

## 它是如何涌现的

```
Agent Core
  │
  │  将 SKILLS_GUIDANCE 指令注入系统提示
  │  (告诉 LLM: "在 5+ 次工具调用后, 考虑保存一个技能")
  │
  ▼
LLM (Model)
  │
  │  决定: "我应该保存学到的内容"
  │  返回 tool_call: skill_manage(action=create, ...)
  │
  ▼
Tools ── skill_manage(create) ──→ Skill Store
  │                                  │
  │  写入技能文件                     │  ~/.hermes/skills/my-skill.md
  │  (frontmatter + markdown 正文)   │  (磁盘上的持久化文件)
  │                                  │
  │         ┌────────────────────────┘
  │         │
  │         ▼  下一个会话开始
  │
  │     prompt_builder 扫描技能目录
  │     将匹配的技能注入系统提示
  │         │
  │         ▼
  │     LLM 读取技能 → 行为改变
  │         │
  │         ▼
  │     LLM 发现技能错误或过时
  │         │
  │         ▼
  │     LLM 调用 skill_manage(action=patch)
  │         │
  └─────────┘  循环完成
```

没有编排器管理这个循环。每个箭头是一个不同的组件在做自己的工作。循环之所以存在，仅仅因为它们的契约对齐了。

## 哪些组件参与

| 组件 | 在学习循环中的角色 |
|-----------|----------------------|
| **Agent Core** | 将 `SKILLS_GUIDANCE` 注入系统提示。调用 `prompt_builder` 在会话开始时加载技能。拥有触发学习引导的迭代计数器。 |
| **Model (LLM)** | 决定*何时*保存技能以及保存*什么*。决定何时修补。这不是确定性的 -- LLM 是判断层。 |
| **Tools** | 提供 `skill_manage` 工具，支持操作：`create`、`patch`、`delete`、`list`、`search`。纯文件 CRUD。没有学习逻辑。 |
| **Skill Store** | `~/.hermes/skills/*.md` 文件。平面目录。每个文件有 YAML frontmatter（name、tags、version）和 markdown 正文（技能内容）。 |
| **Memory** | 辅助角色。LLM 可以通过 `memory_search` 回忆过去的上下文，来决定一个技能是否已经存在或需要更新。循环的运行不依赖它。 |

## 案例研究：hermes-agent

hermes 是这个行为被发现的地方。以下是确切的机制。

### 触发：SKILLS_GUIDANCE 注入

在 `hermes/core/prompt_builder.py` 中，系统提示由片段组装而成。其中一个片段是 `SKILLS_GUIDANCE` -- 当迭代计数超过阈值（通常是一次会话中 5 次工具调用）时注入的一段文本。

这段引导大致说："你在这个任务上已经工作了一段时间。如果你学到了可复用的东西，用 `skill_manage` 保存为技能。"

这不是硬编码的规则。它是自然语言中的建议。LLM 可以忽略它。

### 创建：skill_manage 工具

在 `hermes/tools/skill_manage.py` 中，`skill_manage` 工具接受：

- `action`：create | patch | delete | list | search
- `name`：技能标识符
- `tags`：用于匹配的字符串列表
- `content`：markdown 正文（实际的技能内容）

执行 `create` 时，它将文件写入 `~/.hermes/skills/{name}.md`，带有 YAML frontmatter：

```yaml
---
name: efficient-file-search
tags: [filesystem, search, performance]
version: 1
created: 2026-04-08
---
When searching large codebases, use ripgrep with --type flag
instead of recursive grep. Limit depth with --max-depth 3
for initial scans. Only go deeper on confirmed hits.
```

### 加载：prompt_builder 扫描目录

在下一次会话开始时，`prompt_builder` 扫描 `~/.hermes/skills/`。对于每个 `.md` 文件，它读取 frontmatter，将 tags 与当前会话上下文匹配，并将匹配的技能注入系统提示。

注入发生在与身份指令相同的层级。LLM 将技能视为一等指令，而非会话消息。

### 修补：skill_manage(patch)

当 LLM 按照一个技能操作并得到坏结果时，它可以调用 `skill_manage(action=patch, name="efficient-file-search", content="...")`。工具覆写文件正文，增加 frontmatter 中的版本号，技能就更新了。

在 hermes 上，`prompt_builder` 每轮都重新扫描。修补后的技能在同一个会话中就会生效。

### 为什么它只在 hermes 中有效

三个条件被唯一满足：

1. **Core 级别的触发。** `SKILLS_GUIDANCE` 注入由 Agent Core 完成，而不是用户编写的系统提示。它可靠地触发。
2. **即时生效。** `prompt_builder` 每轮重新扫描。创建或修补的技能在同一会话中就改变 LLM 行为。
3. **工具是必须的。** `skill_manage` 被注册为内置工具。Agent Core 拦截调用。一旦 LLM 决定调用它，就不会"忘记"执行。

## 如何在你的 Agent 中使其涌现

### 步骤 1：创建 Skill Store

一个目录。仅此而已。

```
~/.your-agent/skills/
```

每个文件：YAML frontmatter + markdown 正文。Frontmatter schema：

```yaml
---
name: string        # 唯一标识符，kebab-case
tags: [string]      # 用于匹配会话上下文
version: integer    # 每次 patch 时递增
created: date       # ISO 8601
updated: date       # ISO 8601，可选
---
```

### 步骤 2：构建 skill_manage 工具

一个带 5 种操作的文件 CRUD 工具：

| 操作 | 输入 | 行为 |
|--------|-------|----------|
| `create` | name, tags, content | 写入新文件。如果 name 已存在则失败。设置 version=1。 |
| `patch` | name, content | 覆写正文。递增 version。更新 `updated` 字段。 |
| `delete` | name | 删除文件。 |
| `list` | （无） | 返回所有技能的 name + tags。 |
| `search` | query | 在 name、tags 和 content 中匹配查询。返回前 3 个结果。 |

### 步骤 3：将 prompt_builder 接入技能扫描

在会话开始时（可选地每轮都执行），扫描技能目录。对于每个文件：

1. 解析 frontmatter。
2. 将 tags 与当前会话关键词匹配。
3. 将匹配的技能注入系统提示，位置在身份之后、会话历史之前。

如果无条件注入所有技能，你会浪费 token。基于 tag 的匹配保持成本低廉。

### 步骤 4：添加学习引导提示

当会话超过 N 次工具调用时（hermes 使用 5 次），在系统提示中注入一段话。示例：

```
You have executed several tool calls in this conversation. If you
discovered a reusable pattern, technique, or procedure, save it
as a skill using skill_manage(create). Keep skills concise and
actionable. One skill per technique.
```

这是催化剂。没有它，LLM 不会自发决定保存技能。

### 步骤 5：用修补引导闭合循环

添加另一段话，在技能被加载时触发：

```
You are using skills from previous sessions. If any skill produces
incorrect or outdated results, update it using skill_manage(patch).
Do not delete skills unless they are fundamentally wrong.
```

这闭合了反馈循环。没有它，错误的技能会永远持续下去。

## 跨模块契约

这些契约必须成立，循环才能运作。打破其中任何一个都会打破整个行为。

### 契约 1：技能文件格式

```
File: ~/.your-agent/skills/{name}.md
Format: YAML frontmatter (--- delimited) + markdown body
Required fields: name (string), tags (string[]), version (int), created (date)
Encoding: UTF-8
Newlines: LF
```

`skill_manage`（写入者）和 `prompt_builder`（读取者）必须就此格式达成一致。如果写入者输出 JSON 而读取者期望 YAML，循环会静默中断。

### 契约 2：存储路径约定

```
Base path: configurable, default ~/.your-agent/skills/
File naming: {name}.md where name matches frontmatter name field
No subdirectories. Flat scan.
```

`skill_manage` 写入这里。`prompt_builder` 扫描这里。如果它们对路径不一致，技能会被创建但永远不会被加载。

### 契约 3：提示注入方式

```
Skills are injected into the system prompt, not as user messages.
Position: after agent identity, before conversation history.
Format: each skill wrapped in a labeled block (e.g., <skill name="...">...</skill>).
```

如果技能作为用户消息注入，LLM 可能将它们视为会话上下文而非指令。技能会失去权威性。

### 契约 4：技能 CRUD 参数

```
skill_manage tool interface:
  action: enum("create", "patch", "delete", "list", "search")
  name: string (required for create, patch, delete)
  tags: string[] (required for create, optional for patch)
  content: string (required for create and patch)
  query: string (required for search)
```

LLM 生成这些参数。如果工具 schema 变了但系统提示仍然描述旧 schema，LLM 会产生无效的调用。

## 如何测试（循环测试）

这个测试验证完整周期：创建、加载、使用、修补、重新加载。

```
Test: learning-loop-full-cycle
Precondition: skills directory is empty.

Step 1 — 触发创建
  模拟一个包含 5+ 次工具调用的会话（例如 5 次文件读取）。
  验证：SKILLS_GUIDANCE 提示已注入（检查系统提示）。
  发送消息："That search pattern was useful. Save it."
  验证：LLM 返回 skill_manage(create) 的 tool_call。
  执行工具调用。
  验证：技能目录中存在文件。
  验证：frontmatter 有 version=1、有效的 tags、有效的 name。

Step 2 — 验证加载
  开始新会话。
  验证：prompt_builder 扫描了技能目录。
  验证：系统提示包含技能内容。
  （通过在发送给 LLM 之前检查组装好的提示来验证。）

Step 3 — 验证行为变化
  在新会话中，提出一个匹配技能领域的任务。
  验证：LLM 响应引用或遵循了技能的指导。
  （这是非确定性的。使用评分标准：响应是否匹配
   技能的技术？是/否。）

Step 4 — 触发修补
  提出一个技能给出错误建议的场景。
  发送："That technique doesn't work for binary files. Update the skill."
  验证：LLM 返回 skill_manage(patch) 的 tool_call。
  执行工具调用。
  验证：文件版本增加到 2。
  验证：文件内容反映了修正。

Step 5 — 验证修补后版本加载
  开始另一个新会话。
  验证：系统提示包含修补后的技能内容（version 2）。
  验证：旧内容已消失。

Pass criteria: all 5 steps pass. Any failure means the loop is broken.
```

运行时间：使用本地 LLM 约 3 分钟，使用模拟 LLM 响应约 30 秒。

## 跨项目变体

| 项目 | 学习机制 | 贴近完整循环的程度 | 评分 |
|---------|-------------------|----------------------|-------|
| **hermes-agent** | `SKILLS_GUIDANCE` 提示注入 + `skill_manage` 内置工具 + `prompt_builder` 目录扫描 + 同会话重载 + `skill_manage(patch)` 修正。 | 完整循环。每个环节都已接通。 | **10/10** |
| **openclaw** | 工作区技能存在于 `extensions/skills/`。技能可以加载到会话中。但没有自动创建机制 -- 必须由人类编写技能文件。没有修补流程。 | 存储和加载存在。创建和修补是手动的。 | **4/10** |
| **Claude Code** | `/remember` 命令和 `auto-extract` 后台 agent 将事实提取到 `~/.claude/memory/*.md`。事实在下次会话时注入。但这些是原始事实（"用户偏好 TypeScript"），不是可复用的方法论（"做 X 时，使用技术 Y"）。没有修补机制。 | 提取事实，不是技能。不创建可复用的流程。没有反馈修正。 | **3/10** |
| **NemoClaw** | 不适用。NemoClaw 是安全运行时，不是 Agent。没有学习机制。 | 不适用 | **0/10** |

### 是什么让 hermes 独特

两件事将 hermes 与其他项目分开：

1. **方法论，不是事实。** hermes 的技能包含流程（"先做 X，再做 Y，然后做 Z"）。Claude Code 的记忆包含观察（"用户喜欢 Z"）。流程改变行为。观察增加上下文。

2. **自我修正。** hermes 的技能可以被创建它们的同一个 Agent 修补。Claude Code 的记忆无法被修正 -- Agent 只能添加新记忆，可能与旧记忆产生矛盾。

### 如果你想要学习循环但基于 Claude Code

你需要添加 Claude Code 没有的三样东西：

1. 一个 `skill_manage` MCP 工具（create、patch、delete、list、search）。
2. 一个技能加载步骤，将匹配的技能注入系统提示（通过 CLAUDE.md 或 Agent 启动时读取的自定义技能文件）。
3. 一个在 N 次工具调用后触发的学习引导提示。

最薄弱的环节将是步骤 2。Claude Code 在会话开始时加载 CLAUDE.md 和技能文件，而非每轮都加载。会话中途创建的技能要到下次会话才会生效。hermes 每轮都重新加载。

添加这些后的预期贴近度评分：6/10。差距在于重载时机以及 MCP 工具调用是建议而非保证这一事实。
