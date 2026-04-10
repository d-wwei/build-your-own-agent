# 模块 9：存储

> 持久化不是一个水平层。它是垂直的 -- 每个服务拥有自己的数据。

---

## 它是什么

存储**不是**一个与 Agent Core 或 Tools 同等意义上的独立模块。不存在 `StorageService` 类。不存在 `DataLayer` 抽象。不存在一个所有模块都与之通信的中央数据库。

相反，Hub-and-Spoke 架构中的每个服务都有自己的存储后端 -- 或者根本没有：

```
Agent Core ───→ Session Store  (会话历史)
             ───→ Config Store   (Agent 配置)
Memory ────────→ Memory Store   (长期记忆，向量数据库 / 全文搜索)
Tools ─────────→ Skill Store    (技能文件，MD/YAML)
Model ─────────→ 无             (无状态，调用外部 API)
Context Mgr ───→ 无             (在内存中的消息数组上操作)
```

在物理层面，这些可能共享一个 SQLite 文件。hermes 就是这样做的 -- 一个 `.db` 文件包含 sessions、memories、skills 等表。但在逻辑层面，每条数据都有唯一的所有者。会话历史属于 Agent Core。长期记忆属于 Memory。技能文件属于 Tools。

**为什么这很重要：** 将持久化画成独立的水平层 -- 一个位于所有组件之下、提供通用 CRUD 的 `DataLayer` -- 在架构上是错误的。我们在最初的架构模型中犯了这个错误，花了四轮修正才纠正过来。问题在于，通用的 `DataLayer` 暗示了共享的所有权、共享的 schema 和共享的访问模式。而实际上，Agent Core 以 append-only 流的方式读写会话，Memory 做向量相似性搜索，Tools 从磁盘读取平面文件。它们之间没有共同点。强制它们通过统一抽象只会造成不必要的耦合，收获零复用。

---

## 它解决什么问题

每个存储为其所属服务解决特定的持久化问题：

**Session Store -- "这次会话中发生了什么？"**
会话历史必须在进程重启后存活。如果 Agent 在任务中途崩溃，用户期望从中断处恢复。Session Store 持久化消息数组 -- 用户消息、助手消息、工具调用、工具结果 -- 按 session ID 索引。

**Config Store -- "这个 Agent 如何配置？"**
Agent 身份、模型选择、工具权限、记忆设置、自定义指令。这些很少变更，但必须被多个服务可读。Config Store 是读多写少、共享读取的。

**Memory Store -- "这个 Agent 长期记住了什么？"**
提取的事实、用户偏好、项目上下文、学习到的模式。Memory Store 支持语义搜索（向量相似性或全文搜索），因为检索问题是"什么与这个查询相关？"而不是"给我第 47 条记录"。

**Skill Store -- "这个 Agent 学会了做什么？"**
技能文件 -- 描述 Agent 从经验中创建的流程的 Markdown 或 YAML 文档。Tools 管理它们（CRUD）。Agent Core 消费它们（加载到系统提示中）。Skill Store 是最简单的后端：磁盘上的文件。

---

## 四个项目如何实现

### hermes-agent：一个 SQLite，四个逻辑存储

hermes 是垂直存储模型最清晰的示例。

**物理层：** 单个 SQLite 文件位于 `~/.hermes/hermes.db`。一个数据库，多张表，每张表由不同的服务拥有。

**Session Store：** `conversations` 表存储序列化的消息数组。按 session ID 和时间戳索引。Agent Core 直接读写。会话期间 append-only；完整历史可用于恢复。

**Config Store：** 位于 `~/.hermes/config/` 的 YAML 文件。Agent 配置、模型设置、工具偏好。启动时加载，支持热重载。

**Memory Store：** `MemoryProvider` ABC 背后的八个可插拔后端。SQLite 是默认后端，但 Redis、文件系统和向量数据库后端也可用。Memory 服务拥有 schema 和访问模式。其他服务通过 Memory 的 API 访问，从不直接触碰表。

这 8 个后端：`sqlite`、`redis`、`filesystem`、`json`、`yaml`、`vector`（ChromaDB）、`postgres`、`custom`。实际上，大多数部署使用 `sqlite` -- 其他后端的存在是因为 hermes 的 `MemoryProvider` 抽象使添加新后端变得简单。

**Skill Store：** 位于 `~/.hermes/skills/*.md` 的平面文件。YAML frontmatter 用于元数据（名称、触发条件、版本）。Markdown 正文用于技能内容。`skill_manage` 工具处理 CRUD。`prompt_builder` 每轮扫描目录并注入相关技能。

**关键洞察：** 尽管共享一个 SQLite 文件，每个服务的数据是清晰分离的。Memory 不读 conversations 表。Agent Core 不写 memory 表。单文件是部署上的便利，而非架构上的耦合。

### openclaw：按包分离

openclaw 的存储分布在其架构中，物理边界比 hermes 更清晰。

**Session Store：** 由 `pi-agent-core`（嵌入式 Agent 运行时）管理。会话按 lane（openclaw 的并发单元）持久化。每个 lane 有自己的会话历史，与其他 lane 隔离。

**Config Store：** 配置存在于控制平面中。Agent、通道、工具权限 -- 全部通过基于 WebSocket 的管理接口管理。配置变更通过配置漂移检测传播到运行中的 Agent（系统检测到配置变更后无需重启即可重新加载）。

**Memory Store：** 双注册模式。记忆同时注册为可搜索数据（用于检索）和工具元数据（让 LLM 知道有什么可用）。openclaw 还有一个"做梦"系统 -- Agent 空闲时在后台处理、整合和重组记忆。

**Skill Store：** 工作区级别的技能。手动管理 -- 不像 hermes 那样自动创建。技能按工作区加载，因此不同的部署可以有不同的技能集。

**架构差异：** openclaw 的存储比 hermes 更分散。没有单个 SQLite 文件。每个子系统独立管理自己的持久化。这使系统更难检查（你无法打开一个数据库就看到所有内容），但更容易扩展（每个存储可以独立部署）。

### Claude Code：文件 + SQLite 混合

Claude Code 使用三个 Agent 中最简单的存储模型，与其单用户本地部署模型相匹配。

**Session Store：** 用 JSONL 文件存储会话历史。每个会话是一个文件。便于检查（直接 `cat` 文件），便于备份（直接 `cp`）。SQLite 用于索引和快速查找，但数据源是 JSONL。

**Config Store：** 项目目录和用户主目录中的 JSON 文件。`.claude/settings.json` 用于项目配置。`~/.claude/` 用于全局配置。没有数据库，没有控制平面 -- 只有文件。

**Memory Store：** 两个子系统。`SessionMemory` 跟踪会话内状态（消息、工具调用）。长期记忆写入 `~/.claude/memory/*.md` 的 Markdown 文件。记忆提取自动运行 -- 在重要交互后，系统提取关键事实并追加到记忆文件。没有向量搜索，没有全文搜索 -- 只是将整个文件加载到上下文中。这行得通，因为 Claude Code 针对的是单用户场景，记忆文件保持在几千 token 以内。

**Skill Store：** `~/.claude/skills/` 目录。半自动 -- `/remember` 命令创建记忆条目，用户可以手动创建技能文件。不如 hermes 的自动创建那么成熟，但对其使用场景来说足够。

### NemoClaw

NemoClaw 不是一个 Agent -- 它是一个安全沙箱运行时。它没有会话历史、没有记忆、没有技能。它的存储仅限于沙箱配置（通过基础设施层，而非 Agent 存储模型）。

一个相关细节：NemoClaw 的 Landlock 文件系统策略定义了沙箱内的 Agent 可以在哪里持久化以及持久化什么。这意味着 NemoClaw 间接控制了 Session Store、Memory Store 和 Skill Store 可以写入的位置。配置始终以只读方式挂载。

---

## 对比表

| 维度 | hermes | openclaw | Claude Code | NemoClaw |
|--------|--------|----------|-------------|----------|
| **Session Store** | SQLite `conversations` 表 | pi-agent-core 按 lane 存储 | JSONL 文件 + SQLite 索引 | 不适用 |
| **Config Store** | YAML 文件 | 控制平面（WebSocket） | JSON 文件（项目 + 全局） | 只读挂载（Landlock） |
| **Memory Store** | SQLite 默认，8 个可插拔后端 | 双注册 + 做梦系统 | Markdown 文件，整体加载 | 不适用 |
| **Skill Store** | `~/.hermes/skills/*.md`（自动创建） | 工作区技能（手动） | `~/.claude/skills/`（半自动） | 不适用 |
| **物理布局** | 单个 SQLite 文件 | 按子系统分布 | 文件 + SQLite 混合 | 不适用 |
| **Schema 所有权** | 每个服务拥有自己的表 | 每个包拥有自己的数据 | 每个子系统拥有自己的文件 | 不适用 |
| **向量搜索** | 可选（ChromaDB 后端） | 可选（双注册） | 无 | 不适用 |
| **全文搜索** | SQLite FTS5 | 内置 | 无（加载整个文件） | 不适用 |
| **热重载** | 配置支持，schema 不支持 | 配置漂移检测 | 配置支持 | 不适用 |

---

## 最佳实践

**v1 阶段一切使用 SQLite。** 一个文件。零基础设施。亚毫秒级读取。处理数万条记录毫不费力。hermes 证明这对生产环境的 Agent 工作负载有效。第一天不要评估 Postgres、Redis、Pinecone 或 ChromaDB。你不是 Google。你没有 Google 的扩展问题。

**每个服务拥有自己的表或 schema。** Agent Core 读写 `sessions` 表。Memory 读写 `memories` 表。两者不触碰对方的数据。如果 Memory 需要会话历史，它通过函数调用请求 Agent Core，而不是直接读取 sessions 表。这就是垂直存储原则。

**不要创建通用的 DataLayer 抽象。** 诱惑很大："四个存储都做 CRUD，所以让我们做一个 `DataLayer<T>`，带 `get()`、`put()`、`list()`、`delete()`。"问题是每个存储有根本不同的访问模式。Session Store 是 append-only 流。Memory Store 做相似性搜索。Config Store 读多写少。Skill Store 是平面文件。通用 CRUD 抽象要么变得太抽象而无用，要么太具体而只适合一个用例。

**配置是共享读取、很少写入的。** 多个服务需要读取配置（Agent Core 读取身份，Model 读取提供商设置，Tools 读取权限）。但只有一个东西写配置 -- 用户或管理员。在会话期间让配置不可变。启动时加载。如果需要热重载，使用文件监听器或配置漂移检测（openclaw 的方式），而不是可变的共享对象。

**将会话日志与会话状态分离。** 原始消息数组（说了什么）与会话状态（当前步骤、待处理的工具调用、累积的上下文）不是同一回事。hermes 将这些混在一张表里，造成了复杂性。更好的做法：消息放在 append-only 日志中，会话状态放在单独的记录中，跟踪当前位置、压缩状态和元数据。

**从全文搜索开始。之后再添加向量搜索。** 如果你的 Agent 在本地运行，预期记忆少于 50,000 条，SQLite FTS5 可以亚毫秒级性能处理。当关键词匹配开始返回不相关结果时再添加向量搜索 -- 对于大多数 Agent 用例来说，这比你想象的要晚得多。

**保持技能文件为纯 Markdown。** hermes 使用带 YAML frontmatter 的 `.md` 文件。这是正确的选择。技能是人类可读、可编辑、可版本控制的。不要把技能放进数据库。不要序列化为二进制。学习循环（模块 4 + 涌现行为）的全部价值取决于技能可以被检查和修改。

---

## 逐步构建：如何实现

### 阶段 1：Session Store

从这里开始。没有会话持久化，你的 Agent 在重启时会忘记一切。

```
// Schema -- 一张表
CREATE TABLE sessions (
  id          TEXT PRIMARY KEY,
  created_at  TEXT DEFAULT (datetime('now')),
  updated_at  TEXT DEFAULT (datetime('now')),
  metadata    TEXT  -- JSON: model, user, task summary
);

CREATE TABLE messages (
  id          INTEGER PRIMARY KEY AUTOINCREMENT,
  session_id  TEXT REFERENCES sessions(id),
  role        TEXT NOT NULL,  -- 'user', 'assistant', 'tool', 'system'
  content     TEXT NOT NULL,
  tool_call   TEXT,           -- JSON, null for non-tool messages
  created_at  TEXT DEFAULT (datetime('now'))
);

CREATE INDEX idx_messages_session ON messages(session_id, id);
```

Agent Core 接口：
```
sessionStore.createSession() → sessionId
sessionStore.appendMessage(sessionId, message) → void
sessionStore.getMessages(sessionId) → message[]
sessionStore.listSessions() → sessionMetadata[]
```

就这些。四个函数。不需要 ORM，不需要查询构建器。通过 `better-sqlite3`（Node）或 `sqlite3`（Python）直接使用原生 SQL。

### 阶段 2：Config Store

Config Store 是最简单的存储。就是文件。

```
// 目录结构
~/.agent/
  config.yaml        # 主配置（身份、模型、默认值）
  tools.yaml         # 工具权限和设置
  memory.yaml        # 记忆后端设置
```

启动时加载：
```
const config = yaml.parse(fs.readFileSync('~/.agent/config.yaml'))
```

以只读方式暴露给其他服务：
```
configStore.get('model.provider')    → 'anthropic'
configStore.get('tools.shell.mode')  → 'ask'
configStore.get('memory.backend')    → 'sqlite'
```

运行时没有写入 API。配置变更通过 CLI 命令或直接编辑文件完成。这消除了一整类 Agent 意外修改自身配置的 bug。

### 阶段 3：Memory Store

Memory Store 需要搜索，不仅仅是 CRUD。从全文搜索开始。

```
CREATE VIRTUAL TABLE memory_fts USING fts5(
  content,
  metadata,
  tokenize='porter unicode61'
);

CREATE TABLE memories (
  id          TEXT PRIMARY KEY,
  content     TEXT NOT NULL,
  type        TEXT NOT NULL,     -- 'fact', 'preference', 'context', 'skill_ref'
  source      TEXT,              -- where this memory came from
  created_at  TEXT DEFAULT (datetime('now')),
  accessed_at TEXT,              -- last retrieval time
  access_count INTEGER DEFAULT 0
);

-- Keep FTS in sync
CREATE TRIGGER memory_insert AFTER INSERT ON memories BEGIN
  INSERT INTO memory_fts(rowid, content, metadata)
  VALUES (NEW.rowid, NEW.content, NEW.type || ' ' || COALESCE(NEW.source, ''));
END;
```

Memory 服务接口：
```
memoryStore.store(content, type, source) → memoryId
memoryStore.search(query, limit = 5) → memory[]
memoryStore.delete(memoryId) → void
memoryStore.getRecent(limit = 10) → memory[]
```

`search()` 函数使用 FTS5 的 `MATCH` 操作符：
```
SELECT m.* FROM memories m
JOIN memory_fts ON memory_fts.rowid = m.rowid
WHERE memory_fts MATCH ?
ORDER BY rank
LIMIT ?
```

之后再添加向量搜索。你会知道什么时候需要它 -- 关键词搜索会开始返回词语匹配但含义不匹配的记忆。

### 阶段 4：Skill Store

Skill Store 就是一个 Markdown 文件目录。不需要数据库。

```
// 目录
~/.agent/skills/
  debug-python-errors.md
  write-unit-tests.md
  optimize-sql-queries.md
```

每个文件都有 YAML frontmatter：
```markdown
---
name: debug-python-errors
trigger: when user reports a Python error or traceback
version: 2
created: 2026-04-01
updated: 2026-04-08
---

## Steps
1. Read the full traceback
2. Identify the exception type and line number
3. Check the surrounding code for common causes
...
```

Skill Store 接口：
```
skillStore.list() → skillMetadata[]
skillStore.load(name) → skillContent
skillStore.save(name, content) → void
skillStore.delete(name) → void
```

Agent Core 每轮调用 `skillStore.list()`，根据触发条件将相关技能注入系统提示。Tools 在学习循环创建新技能时调用 `skillStore.save()`。

### 阶段 5：接入 Agent Core

Agent Core 在会话边界协调所有存储：

```
// 会话开始
const sessionId = sessionStore.createSession()
const config = configStore.getAll()
const memories = memoryStore.search(userMessage, 5)
const skills = skillStore.list()

// 使用 config + memories + skills 构建系统提示
const systemPrompt = buildPrompt(config, memories, skills)

// 会话循环
while (budget > 0) {
  const response = await model.call(messages)
  sessionStore.appendMessage(sessionId, response)

  if (response.toolCalls) {
    const results = await tools.execute(response.toolCalls)
    for (const result of results) {
      sessionStore.appendMessage(sessionId, result)
    }
  }
}

// 会话结束 -- 提取记忆
const newMemories = await memory.extract(messages)
for (const mem of newMemories) {
  memoryStore.store(mem.content, mem.type, sessionId)
}
```

关键点：Agent Core 通过四个不同的接口调用四个不同的存储。它从不与通用的 `DataLayer` 交互。每个存储都有自己由访问模式决定的 API。

---

## 常见陷阱

**构建通用 DataLayer。** 这是我们最顽固的架构错误。"持久化是横切关注点，所以让我们做一个水平层。"听起来合理。但是错的。Session Store 追加消息。Memory Store 做相似性搜索。Config Store 加载文件。Skill Store 读取 Markdown。通用抽象要么将这些强制压入最低公约数的 CRUD 接口（失去每个存储需要的特定能力），要么变得如此可配置以至于比直接实现更难用。

**服务间共享可变状态。** 如果 Memory 可以直接读取 sessions 表，你就有了一个任何接口边界都无法保护的隐式耦合。当 Agent Core 改变消息格式时，Memory 静默崩溃。当 Memory 索引旧会话时，它与 Core 竞争数据库锁。每个服务只应访问自己的表。跨服务数据访问通过显式函数调用，而非共享数据库访问。

**在有用户之前优化存储。** 你花了一周时间对比 Postgres vs. SQLite vs. DynamoDB。你的 Agent 有三个用户。SQLite 在单个文件中处理数百万行，亚毫秒级读取。没有服务进程、没有连接池、没有网络延迟，备份就是 `cp hermes.db hermes.db.bak`。从 SQLite 开始。等你真正触及其极限时再迁移。

**把技能放进数据库。** 技能是文档。人类需要阅读和编辑它们。版本控制需要追踪它们。`git diff` 需要显示变更。包含序列化 Markdown 的数据库行无法提供这些。保持技能为文件。hermes 在这点上做对了。

**忘记 Config Store 是共享读取的。** 如果你让配置成为任何服务都可以写入的可变运行时对象，你会遇到竞态条件、不一致状态和调试噩梦。配置在会话期间应该是不可变的。加载一次，到处使用，只通过显式管理员操作更改。

**没有将 access_count 从内容存储中分离。** 记忆检索受益于追踪每条记忆的访问频率（用于衰减、优先级排序、清理）。如果 `access_count` 和内容在同一张表中，每次检索都变成先读后写。考虑使用单独的 `memory_stats` 表，或者批量更新访问计数，而非每次查询都更新。

**把"一个 SQLite 文件"当作"一张表"。** hermes 将 sessions、memories、config 和 skills 放在一个 `.db` 文件中。这没问题 -- 这是部署上的便利。但每个逻辑存储仍然有自己的表、自己的 schema 和自己的访问代码。"一个文件"不等于"一个 schema"或"一个服务"。

---

## 跨模块契约

存储是垂直的，但各存储仍然与拥有它们的服务和消费它们的服务之间存在契约：

| 契约 | 涉及方 | 内容 |
|----------|---------|------|
| **消息 schema** | Agent Core → Session Store | Session Store 以 Agent Core 的格式持久化消息。如果 Core 添加字段（如 `speculation_state` 或 `compression_marker`），Session Store 必须处理 -- 要么忠实存储，要么显式忽略。 |
| **记忆写入所有权** | Memory（模块 4）→ Memory Store | 只有 Memory 服务写入 Memory Store。Agent Core 通过 Memory 的搜索 API 读取记忆，从不直接查询存储。如果 Core 需要新的查询模式，Memory 将其添加到自己的 API。 |
| **技能文件格式** | Tools（模块 3）↔ Skill Store ↔ Agent Core（模块 1） | Tools 写技能。Core 读技能。契约是文件格式：YAML frontmatter（name、trigger、version）+ Markdown 正文。更改 frontmatter schema 需要同时更新 Tools 的写逻辑和 Core 的 prompt builder。 |
| **配置读取接口** | Config Store → 所有服务 | Config Store 提供只读接口。服务通过键路径请求配置值。键命名空间必须稳定 -- 将 `model.provider` 重命名为 `llm.service` 会破坏所有读取它的服务。 |
| **文件系统路径** | 所有存储 ↔ 安全信封（模块 8） | 如果安全信封处于活动状态（Landlock、Docker 卷），所有存储必须写入允许的路径内。Session Store、Memory Store 和 Skill Store 的路径必须在可写集合中。Config Store 的路径必须在可读集合中。更改存储路径需要更新安全策略。 |
| **会话生命周期** | Agent Core → Session Store，Memory Store | 会话结束时，Core 发出完成信号。Memory 从会话中提取长期知识。Session Store 将会话标记为完成。顺序很重要：记忆提取必须在会话消息被压缩或归档之前进行。 |

**这些契约破裂会发生什么：**
- 更改消息 schema 而不更新 Session Store → 旧会话无法加载 → 恢复时出现"session not found"错误
- Memory 直接读取 Session Store 而不通过 Core → Core 更改消息格式 → Memory 在反序列化时崩溃
- 更改技能 frontmatter schema → Core 的 prompt builder 跳过格式未知的技能 → 学习循环静默停止工作
- 重命名配置键 → 读取旧键的服务得到 `undefined` → 回退到硬编码默认值 → 行为变化却没有解释
- 移动技能目录而不更新安全策略 → Landlock 阻止写入 → `skill_manage create` 失败 → 学习循环中断

垂直存储原则使这些契约比水平 DataLayer 模型更简单。每个契约是双边的（在一个服务和一个存储之间），而非多边的（在所有服务和共享层之间）。当出问题时，你准确知道该查看哪两个组件。

---

*下一篇：[模块 10 -- 运行时与基础设施](10-runtime-infrastructure.md) | 上一篇：[模块 8 -- 安全信封](08-security-envelope.md)*
