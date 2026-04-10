# 模块 4：Memory
> 最容易被误解的模块——它既是 LLM 调用的工具，也是 Agent Core 自动读取的基础设施。

## 它是什么

Memory 赋予你的 Agent 过去。没有它，每次对话都从零开始。用户周一告诉 Agent 自己的名字；周二 Agent 又问了一遍。

但 Memory 不仅仅是"Agent 查询的数据库"。这是大家都做对了的部分。大家都搞错的部分是**双重角色**。我们在全部 4 个项目中验证了这一点——无一例外：

**角色 1：Memory 作为工具。** LLM 决定"我应该保存这个"或"我需要回忆 X"，然后主动调用 `memory_save` 或 `memory_search`。这是 LLM 发起的。Agent Core 像处理其他任何工具调用一样处理这些——分派、执行、返回结果。

**角色 2：Memory 作为基础设施。** 在每轮对话之前，Agent Core 构建系统 prompt。其中一部分是注入的记忆——从 Memory Store 中拉取的相关上下文，无需 LLM 主动请求。这是 Core 发起的。LLM 从不看到 `memory_search` 调用；它只是发现相关的记忆已经坐在系统 prompt 里了。

在 Hub-and-Spoke 架构中，Memory 是 Agent Core 的**对等服务**，与 Model、Tools 和 Context Manager 并列。但不同于 Tools（Core 只在 LLM 请求时调用它）和 Model（Core 每轮都调用它），Memory 从**两个方向**被调用：LLM 通过工具分派调用它，Core 在 prompt 构建时读取它。

**Memory Store** 挂在 Memory 之下——向量数据库、全文搜索索引或磁盘上的纯文件。Memory 拥有读写接口；存储只是后端。

## 它解决什么问题

**跨会话连续性。** 一个每次重启都忘记你项目结构的编码 Agent，对多天任务来说毫无用处。hermes 通过可插拔后端存储 8 种不同类型的记忆。Claude Code 将记忆持久化到 `~/.claude/memory/*.md` 文件中，跨会话存活。

**减少重复。** 用户会反复说同样的话。"我偏好 TypeScript。""始终使用 Tailwind。""我的 API 密钥在 .env 里。"Memory 保存一次然后自动注入，消除了用户每次对话都重申偏好的需要。

**为 LLM 提供基础。** 没有 Memory，LLM 可以自由臆造。有了注入系统 prompt 的相关记忆，模型的回答就扎根在实际的项目上下文、用户偏好和先前决策中。

**将上下文扩展到窗口之外。** 200K token 的上下文窗口很大但不是无限的。一个 6 个月的项目会积累数百万 token 的对话历史。Memory 充当压缩索引——存储重要的部分，按需检索，而不是把所有东西都塞进上下文。

**双重角色的缺口。** 如果你只构建 Memory 作为工具，LLM 就必须记得每轮都搜索 Memory。它不会的。LLM 在自我提示进行信息检索方面是不可靠的。如果你只构建 Memory 作为基础设施，LLM 就永远无法显式保存某些东西以备后用。你需要两个角色，从第一天起就一起设计。

## 四个项目是怎么做的

### hermes-agent

**最强的抽象。** hermes 定义了一个 `MemoryProvider` ABC（Abstract Base Class），支持 8 个可插拔后端。抽象足够干净，从 SQLite 切换到 Redis 只需要改一行配置。

```python
# hermes: memory/provider.py（简化）
class MemoryProvider(ABC):
    @abstractmethod
    async def store(self, key: str, content: str, metadata: dict) -> None: ...

    @abstractmethod
    async def search(self, query: str, limit: int = 5) -> list[MemoryItem]: ...

    @abstractmethod
    async def delete(self, key: str) -> bool: ...
```

**8 个后端**：SQLite（默认）、Redis、PostgreSQL、Pinecone、Weaviate、Qdrant、ChromaDB 和纯文件系统。每个都实现 `MemoryProvider`。你在配置中选择一个。

**双重角色实现**：
- *作为工具*：`memory_save` 和 `memory_search` 是注册的工具，LLM 像调用其他工具一样调用它们。它们委派给活跃的 MemoryProvider。
- *作为基础设施*：`prompt_builder` 在每轮开始时查询 MemoryProvider，检索最近和相关的记忆，并以 `## Relevant Memories` 部分注入系统 prompt。

**记忆类型**：hermes 区分事实记忆（"用户的名字是 Eli"）、过程记忆（"这个项目用 pnpm，不用 npm"）和情景记忆（"昨天我们调试了认证流程，发现了一个竞态条件"）。LLM 的 `memory_save` 调用包含一个 `type` 字段。

**优点**：四个项目中最好的抽象。MemoryProvider ABC 是策略模式的教科书范例。切换后端毫无痛苦。记忆的类型系统增加了有用的结构。

**缺点**：8 个后端意味着 8 个需要维护的集成点。SQLite 后端经过充分测试；一些较冷门的后端有边缘情况。另外，hermes 没有后台记忆整理——记忆会不断积累，随时间变得嘈杂。

### openclaw

**双重注册，做对了。** openclaw 将 Memory 模块分为 `memory-core`（引擎）和 `memory-lancedb`（默认向量存储）。关键洞察是**显式的双重注册**：Memory 同时将自己注册为工具提供者和 prompt 构建器 Hook。

```typescript
// openclaw: memory-core（简化）
export function onActivate(api: PluginAPI) {
  // 角色 1：注册为工具
  api.registerTool({
    name: "memory_save",
    execute: async (params) => { /* 保存到 LanceDB */ }
  });
  api.registerTool({
    name: "memory_search",
    execute: async (params) => { /* 搜索 LanceDB */ }
  });

  // 角色 2：注册为 prompt 构建器
  api.registerPromptBuilder({
    name: "memory-context",
    priority: 50,
    build: async (context) => {
      const relevant = await searchRelevant(context.lastMessage);
      return formatAsSystemPrompt(relevant);
    }
  });
}
```

**"做梦"——后台记忆整理。** openclaw 运行一个后台进程，定期审查已存储的记忆，合并重复项，解决矛盾，并按相关性重新排序。这灵感来源于人类在睡眠中的记忆巩固过程。Agent 不需要处于对话中，"做梦"也能发生。

**Memory Store**：LanceDB（嵌入式向量数据库）。选择它是为了本地优先运行——不需要外部服务器。支持向量相似度搜索和元数据过滤。

**优点**：双重注册模式在代码中使双重角色变得显式，而非依赖于约定的隐式。"做梦"解决了 hermes 没有解决的记忆腐烂问题。LanceDB 是务实的选择——快速、嵌入式、零运维。

**缺点**：做梦计算成本高（它调用 LLM 来重新评估记忆）。如果你的 API 预算有限，它会消耗 token。memory-core/memory-lancedb 的拆分增加了打包复杂性。

### NemoClaw

NemoClaw 没有自己的 Memory 系统。它是包裹 OpenClaw 的安全沙箱运行时。Memory 在这里不适用——但 NemoClaw 的凭证清理是相关的：任何存储 shell 输出或 API 响应的记忆都应该在持久化前剥离密钥。NemoClaw 在操作系统层面用正则表达式处理 API 密钥、Bearer token 和密码。Memory 模块应该在应用层面做同样的事。

### Claude Code

**两个独立子系统。** Claude Code 的 Memory 分为两个几乎不互相通信的部分：

**子系统 1：SessionMemory。** 维护会话内的对话状态。消息、工具调用、工具结果——全部在会话期间跟踪。持久化到 JSONL 文件作为历史记录。这是短期记忆。

**子系统 2：extractMemories（后台 Agent）。** 一个独立的 Fork 出的 Agent 进程，在后台运行。它读取对话记录并将突出的事实提取到 `~/.claude/memory/*.md` 文件中。这些文件在下次启动时加载到系统 prompt 中。提取异步进行——不会阻塞主对话循环。

```
主 Agent 循环                       后台提取器
     │                                     │
     │ ── 对话进行中 ──→                  │
     │                                     │ (fork)
     │                                     │ 读取记录
     │                                     │ 提取事实
     │                                     │ 写入 ~/.claude/memory/
     │                                     │
     │ ← 下次启动时，Core 加载 ─────────── │
     │    记忆文件到 prompt 中             │
```

**`/remember` 命令。** 用户可以显式告诉 Claude Code 记住某些东西。这会直接写入记忆文件，无需等待后台提取。它是角色 1（Memory 作为工具）的手动覆盖。

**Claude Code 没有做的事**：它不会从记忆中创建结构化的技能文件。提取的记忆是扁平的事实（"这个项目用 pnpm"）——不是可复用的方法论。这就是为什么 Claude Code 在学习循环完整度上得分 3/10 而 hermes 得 10/10。

**优点**：后台提取是非阻塞的。用户永远不需要等待记忆操作。Fork 出的 Agent 方式让记忆逻辑完全与主循环隔离——提取中的崩溃不会影响对话。

**缺点**：两个独立子系统意味着两套代码要维护和两组 Bug。提取质量取决于后台 Agent 的 prompt——如果它遗漏了什么，用户要到下次会话才能发现。没有向量搜索；记忆被完整加载，依赖 LLM 自行判断相关性。

## 对比表

| 维度 | hermes | openclaw | NemoClaw | Claude Code |
|-----------|--------|----------|----------|-------------|
| **架构** | MemoryProvider ABC | memory-core + memory-lancedb Plugin | N/A | SessionMemory + extractMemories |
| **后端数量** | 8 个可插拔 | 1 个（LanceDB） | N/A | 文件系统（MD 文件） |
| **双重角色** | 隐式（tools + prompt_builder 都使用 Provider） | 显式双重注册（tool + promptBuilder） | N/A | 两个独立子系统 |
| **作为工具** | `memory_save`、`memory_search` | `memory_save`、`memory_search` | N/A | `/remember` 命令 + extractMemories |
| **作为基础设施** | prompt_builder 每轮查询 Provider | registerPromptBuilder Hook，优先级 50 | N/A | 启动时加载 `~/.claude/memory/*.md` |
| **向量搜索** | 是（Pinecone、Weaviate、Qdrant、ChromaDB） | 是（LanceDB） | N/A | 否（完整文件加载） |
| **后台整理** | 否 | 是（"做梦"巩固） | N/A | 是（后台 Fork 提取） |
| **记忆类型** | 事实、过程、情景 | 无类型（元数据标签） | N/A | 仅扁平事实 |
| **凭证清理** | 否 | 否 | 操作系统级正则 | 否 |
| **抽象质量** | 最佳（策略模式 ABC） | 良好（Plugin 边界） | N/A | 弱（两个断开的系统） |

## 最佳实践

**如果你只做一件事**：从第一天起就为双重角色而设计。不要只构建 Memory 作为工具然后计划以后再补上基础设施注入。两个角色共享同一个存储和同一个数据模型。事后再拆分意味着要么复制数据，要么构建脆弱的桥接。

**来自四个项目的六条规则**：

1. **一个接口，两个调用者。** 你的 MemoryProvider（或等价物）应该暴露 `save()`、`search()` 和 `delete()`。Agent Core 在 prompt 构建时调用 `search()`。工具分派器在 LLM 请求时调用全部三个。同一个接口，两个入口点。

2. **注入要节制。** 不要把所有记忆都倒进系统 prompt。hermes 按相关性检索前 5 条。openclaw 的 prompt 构建器有优先级系统。如果你注入 50 条记忆，你会浪费 2,000+ token，而且 LLM 会忽略其中大部分。按时间和相关性排序，上限 5-10 条。

3. **存储前先清理。** Shell 输出包含 API 密钥。对话记录包含用户不小心输入的密码。Claude Code 不清理。hermes 不清理。NemoClaw 在操作系统层面清理但不在 Memory 层面。从它们都遗漏的地方学习：在 `save()` 之前运行凭证剥离步骤。

4. **后台处理值得付出额外的复杂性。** openclaw 的"做梦"和 Claude Code 的后台提取解决了同一个问题：Memory 操作不应该阻塞用户。如果你能 Fork 一个后台进程或运行定期任务，就去做。实现成本适中；用户体验的提升很显著。

5. **给你的记忆加类型。** hermes 区分事实、过程和情景。这对检索很重要：当 LLM 问"我们之前是怎么解决这个的？"时，它需要情景记忆。当它需要"这个项目用什么框架？"时，它需要事实记忆。无类型的记忆存储返回一堆混合结果，浪费上下文 token 在无关结果上。

6. **为记忆腐烂做规划。** 记忆会过时。用户从 npm 切换到了 pnpm，但旧的"用 npm"记忆仍在存储中。LLM 现在收到了互相矛盾的信号。这就是记忆腐烂，四个项目都没有完美解决。hermes 没有处理。openclaw 的做梦过程有帮助但不保证能抓住所有情况。你最好的防御：添加一个 `confidence` 字段，随时间衰减，按 `confidence * relevance` 而非单纯相关性排序结果。

## 分步指南：如何构建

### 第 1 步：定义 MemoryProvider 接口

```typescript
// contracts/memory.ts
interface MemoryItem {
  id: string;
  content: string;
  type: "factual" | "procedural" | "episodic";
  createdAt: Date;
  lastAccessed: Date;
  accessCount: number;
  metadata: Record<string, unknown>;
}

interface MemoryProvider {
  save(content: string, type: MemoryItem["type"], metadata?: Record<string, unknown>): Promise<string>;  // 返回 id
  search(query: string, options?: { limit?: number; type?: MemoryItem["type"] }): Promise<MemoryItem[]>;
  delete(id: string): Promise<boolean>;
  getRecent(limit: number): Promise<MemoryItem[]>;
}
```

**决策点**：从第一天起就用向量搜索，还是先从全文搜索开始？如果你的 Agent 在本地运行且预计记忆不超过 10,000 条，SQLite FTS5 绰绰有余。当关键词匹配开始返回无关结果时再添加向量搜索。hermes 两者都支持；Claude Code 两者都不用也能运作（只是加载完整文件）。

### 第 2 步：先构建最简单的后端

从 SQLite 开始。它是嵌入式的、零配置的、久经考验的。

```typescript
// memory/sqlite-provider.ts
class SQLiteMemoryProvider implements MemoryProvider {
  private db: Database;

  constructor(dbPath: string) {
    this.db = new Database(dbPath);
    this.db.exec(`
      CREATE TABLE IF NOT EXISTS memories (
        id TEXT PRIMARY KEY,
        content TEXT NOT NULL,
        type TEXT NOT NULL,
        created_at TEXT NOT NULL,
        last_accessed TEXT NOT NULL,
        access_count INTEGER DEFAULT 0,
        metadata TEXT DEFAULT '{}'
      );
      CREATE VIRTUAL TABLE IF NOT EXISTS memories_fts
        USING fts5(content, content=memories, content_rowid=rowid);
    `);
  }

  async search(query: string, options?: SearchOptions): Promise<MemoryItem[]> {
    const limit = options?.limit ?? 5;
    // FTS5 搜索，带相关性排序
    const rows = this.db.prepare(`
      SELECT m.*, rank
      FROM memories_fts f
      JOIN memories m ON f.rowid = m.rowid
      WHERE memories_fts MATCH ?
      ORDER BY rank
      LIMIT ?
    `).all(query, limit);

    // 更新访问时间戳
    for (const row of rows) {
      this.db.prepare(`
        UPDATE memories SET last_accessed = ?, access_count = access_count + 1 WHERE id = ?
      `).run(new Date().toISOString(), row.id);
    }

    return rows.map(toMemoryItem);
  }

  // save() 和 delete() 的实现...
}
```

### 第 3 步：将 Memory 注册为工具（角色 1）

```typescript
// memory/tools.ts
import { register } from "../tools/registry";
import { getProvider } from "./index";

register({
  name: "memory_save",
  description: "Save information to long-term memory for future reference",
  parameters: {
    type: "object",
    properties: {
      content: { type: "string", description: "What to remember" },
      type: { type: "string", enum: ["factual", "procedural", "episodic"] },
    },
    required: ["content", "type"]
  },
  isConcurrencySafe: true,
  execute: async (params) => {
    const { content, type } = params as { content: string; type: MemoryItem["type"] };
    const sanitized = stripCredentials(content);  // ← 永远不要跳过这步
    const id = await getProvider().save(sanitized, type);
    return { success: true, output: { id, message: "Saved to memory" } };
  }
});

register({
  name: "memory_search",
  description: "Search long-term memory for relevant information",
  parameters: {
    type: "object",
    properties: {
      query: { type: "string", description: "What to search for" },
      limit: { type: "number", description: "Max results", default: 5 },
    },
    required: ["query"]
  },
  isConcurrencySafe: true,
  execute: async (params) => {
    const { query, limit } = params as { query: string; limit?: number };
    const results = await getProvider().search(query, { limit });
    return { success: true, output: results };
  }
});
```

### 第 4 步：将 Memory 接入基础设施（角色 2）

这是大多数构建者止步的地方——也是双重角色变为现实的地方。

```typescript
// core/prompt-builder.ts（在 Agent Core 内部）
async function buildSystemPrompt(lastUserMessage: string): Promise<string> {
  const parts: string[] = [];

  // 1. 身份和指令
  parts.push(loadIdentityPrompt());

  // 2. 可用工具
  parts.push(formatToolDescriptions(getAllTools()));

  // 3. ← MEMORY 注入（角色 2）
  const memories = await memoryProvider.search(lastUserMessage, { limit: 5 });
  if (memories.length > 0) {
    parts.push("## Relevant Context from Memory");
    for (const mem of memories) {
      parts.push(`- [${mem.type}] ${mem.content}`);
    }
  }

  // 4. 加载的技能（来自 Skill Store）
  parts.push(loadRelevantSkills());

  return parts.join("\n\n");
}
```

**决策点**：按最近一条用户消息搜索，还是按对话摘要搜索？按最新消息搜索适用于直接提问（"这个项目用什么框架？"）。对于复杂的多轮对话，你可能想按最近 3-5 轮的摘要搜索。先从简单的开始——按最近消息搜索。当检索质量下降时再添加基于摘要的搜索。

### 第 5 步：添加凭证清理

```typescript
// memory/sanitize.ts
const PATTERNS = [
  /(?:api[_-]?key|token|secret|password|bearer)\s*[:=]\s*["']?[\w\-\.]{20,}/gi,
  /(?:sk|pk|ak|rk)[-_][a-zA-Z0-9]{20,}/g,    // 常见 API 密钥前缀
  /Bearer\s+[a-zA-Z0-9\-._~+\/]+=*/g,          // Bearer token
  /-----BEGIN (?:RSA |EC )?PRIVATE KEY-----/g,   // PEM 密钥
];

function stripCredentials(content: string): string {
  let sanitized = content;
  for (const pattern of PATTERNS) {
    sanitized = sanitized.replace(pattern, "[REDACTED]");
  }
  return sanitized;
}
```

在每次 `save()` 调用时都运行这个。没有例外。这就是 NemoClaw 在操作系统层面所做的事——你应该在应用层面同样做到。

### 第 6 步：添加后台提取（可选但推荐）

不要仅仅依赖 LLM 调用 `memory_save`，定期从对话记录中提取记忆。

```typescript
// memory/extractor.ts
async function extractMemories(transcript: Message[]): Promise<void> {
  // 在后台运行——不要阻塞主循环
  const extraction = await callLLM({
    system: "Extract important facts, preferences, and decisions from this conversation. Return as JSON array.",
    messages: [{ role: "user", content: formatTranscript(transcript) }],
    model: "fast-cheap-model"  // 提取不用贵的模型
  });

  const items = JSON.parse(extraction);
  for (const item of items) {
    const existing = await provider.search(item.content, { limit: 1 });
    if (existing.length === 0 || existing[0].content !== item.content) {
      await provider.save(stripCredentials(item.content), item.type);
    }
  }
}
```

**决策点**：每次对话后提取，还是按时间表提取？Claude Code 在每次会话后进行。openclaw 的做梦定期运行。对大多数 Agent 来说，会话后提取更简单也足够用。基于时间表的提取在你每天有很多短对话时更有意义。

## 常见陷阱

**1. 只构建了工具角色的 Memory。** 你注册了 `memory_save` 和 `memory_search` 作为工具，拍拍手觉得大功告成，然后上线了。接着你发现 LLM 只在大约 30% 该搜索的时候调用了 `memory_search`。为什么？因为模型不知道自己不知道什么。如果用户 3 个会话前提到了他们的偏好，LLM 没有信号表明存在一条记忆——它不会想到去搜索。基础设施注入（角色 2）通过主动浮现相关记忆来解决这个问题。从第一天起就构建两个角色。

**2. 注入太多。** 你搜索记忆并将前 20 条结果注入系统 prompt。那是 1,500 token 的上下文，LLM 在每一轮都要费力穿越。其中大部分无关。hermes 上限 5 条。从那里开始。如果 LLM 持续遗漏相关上下文，提升到 10 条。永远不要超过 15——到那个程度你弊大于利。

**3. 没有去重。** 用户在第 1、4、7、12 次会话中说了"我偏好 TypeScript"。你现在有 4 份同样的事实。注入记忆时，你浪费了 3 倍的 token。openclaw 的做梦过程会合并重复项。至少在保存前检查近似重复——简单的字符串相似度阈值（Jaccard > 0.8）就能抓到明显的情况。

**4. 过时的记忆与现实矛盾。** "项目用 npm"是 3 个月前保存的。团队上个月切换到了 pnpm。LLM 现在收到了矛盾的信号。这就是记忆腐烂，四个项目都没有完美解决。hermes 没有处理。openclaw 的做梦有帮助但不保证能抓住所有情况。你最好的防御：添加一个 `confidence` 字段，随时间衰减，按 `confidence * relevance` 而非单纯相关性排序结果。

**5. 把 Memory Store 的选择当成第一天就要做的决定。** 你花了一周评估 Pinecone vs Weaviate vs ChromaDB，然后才写了一行 Agent 代码。别这样。hermes 的 MemoryProvider ABC 证明了你可以以后再换后端。从 SQLite FTS5 开始。它能以亚毫秒级搜索处理大约 50,000 条记忆。当（如果）你超出它时再添加向量搜索。

**6. 忘记了清理。** 用户不小心把 API 密钥粘贴到了聊天中。Agent 把它保存到了 Memory。下次会话，它被注入系统 prompt 并可能通过工具输出泄露。hermes 和 Claude Code 都不清理记忆内容。NemoClaw 清理 shell 输出但不清理 Memory。做得比它们都好——在写入时清理。

**7. 两个断开的系统。** Claude Code 的 SessionMemory 和 extractMemories 几乎不通信。会话系统不知道提取器保存了什么。提取器不知道会话系统认为什么重要。这能用是因为 Claude Code 的 Memory 需求很简单（扁平事实）。对于更复杂的 Agent，你想要一个单一的 MemoryProvider 服务两个角色——这正是 hermes 和 openclaw 所做的。

## 跨模块契约

### Memory 对其他模块的期望

| 来源模块 | Memory 需要什么 | 契约 |
|--------------|------------------|----------|
| **Agent Core** | 当前用户消息（用于 prompt 构建时的相关性搜索） | `string` —— 最新的用户消息或对话摘要 |
| **Agent Core** | 对 memory_save/memory_search 调用的工具分派 | 标准工具分派（与其他任何工具相同） |
| **Tools** | Memory 工具的注册机制 | `register(tool: Tool)` —— Memory 通过工具注册表注册自己 |
| **Context Manager** | 知晓注入的记忆消耗 token 预算 | Memory 注入发生在 Context Manager 的预算计算之前 |

### Memory 向其他模块提供什么

| 目标模块 | Memory 提供什么 | 契约 |
|--------------|---------------------|----------|
| **Agent Core** | 用于系统 prompt 的格式化记忆注入 | `search(query, options) → MemoryItem[]` —— Core 在 prompt 构建时调用 |
| **Agent Core** | memory_save/memory_search 的工具执行结果 | 标准 `ToolResult` 格式 |
| **Tools** | 无直接依赖——Memory 和 Tools 是对等服务 | （无直接依赖，但 Memory 工具注册在工具注册表中） |
| **Context Manager** | 注入记忆的 token 数（用于预算追踪） | 隐式——Core 在 prompt 中包含记忆内容，Context Manager 测量总量 |

### 双重角色接线图

```
                          ┌──────────────────┐
                          │   Agent Core     │
                          │  (prompt builder) │
                          └───────┬──────────┘
                                  │
                    ┌─────────────┼─────────────┐
                    │             │              │
              角色 2（读取）      │        角色 1（工具调用）
              "将相关记忆        │        "LLM 调用了 memory_save"
               注入系统          │        "LLM 调用了 memory_search"
               prompt"          │
                    │             │              │
                    ▼             │              ▼
              ┌──────────────────┴────────────────┐
              │         MemoryProvider             │
              │    save() / search() / delete()    │
              └──────────────────┬─────────────────┘
                                 │
                                 ▼
                          ┌────────────┐
                          │ Memory     │
                          │ Store      │
                          │ (SQLite /  │
                          │  Vector DB │
                          │  / Files)  │
                          └────────────┘
```

两个角色都命中同一个 MemoryProvider。两个角色都读写同一个 Memory Store。这是关键的架构洞察。两个调用者，一个接口，一个存储。
