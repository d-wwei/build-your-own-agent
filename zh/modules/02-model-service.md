# 模块 2：Model Service
> Agent Core 的对等服务，而非它的下层。它的职责：从 LLM 提供商获取 token，优雅地处理故障，绝不让限流杀死用户的会话。

## 它是什么

Model Service 与 Tools、Memory 和 Context Manager 处于 Hub-and-Spoke 架构的同一层级。Agent Core 通过接口调用它，它调用外部 LLM API，返回响应。这就是整个契约。

关键洞察：Model Service 是**无状态的**。它没有自己的持久化存储，没有数据库，没有文件。它持有瞬态状态——哪个提供商在冷却中、本次请求消耗了多少 token、prompt 缓存是否命中——但这些都不会在重启后存活。这使它成为最容易替换、测试和 Mock 的模块。这也意味着所有复杂性都在运行时行为中：将请求路由到正确的提供商、在一个宕机时进行故障转移、在恢复时切回来。

可以把它想象成 LLM 的负载均衡器。你不会把 `nginx` 称为应用服务器的"下层"。它是处理流量路由的对等服务。Model Service 对模型调用做的是同样的事。

## 它解决什么问题

LLM API 是不可靠的。不是理论上的不可靠，而是实实在在地不可靠：

1. **限流总在最糟糕的时候出现。** OpenAI 在对话中途返回 429。没有故障转移，用户只能盯着错误看。有了候选链，请求会静默转移到备用提供商。用户毫无感知。
2. **提供商宕机是真实存在的。** Anthropic、OpenAI、Google——都有停机窗口。只依赖单一端点的生产 Agent 就是一个会跟着端点一起宕机的 Agent。
3. **成本差异高达 10-100 倍。** GPT-4o 大约 $2.50/百万输入 token。Claude 3.5 Sonnet 是 $3/百万。Gemini 1.5 Flash 是 $0.075/百万。没有提供商路由，你无法按任务类型优化成本。
4. **Token 预估防止意外账单。** 如果你在发送前不预估 token，你会在 API 返回 400 时才发现超出了上下文窗口。那时你已经烧掉了 prompt 缓存，浪费了这次往返。
5. **Prompt 缓存能节省 50-90% 的重复前缀开销。** Anthropic 的 prompt 缓存和 OpenAI 的缓存 token 都要求前缀在多次调用间完全一致。没有缓存感知的 Model Service，你会因为改变系统 prompt 的组装顺序而意外使缓存失效。

## 四个项目是怎么做的

### hermes-agent

**文件**：模型调用直接嵌入 `AIAgent.run_conversation()` 中。提供商元数据分散在 `model_metadata.py` 和 `auth.py` 中。

**实现**：直接调用 OpenAI SDK。没有抽象层，没有 `ModelService` 接口。模型名称和 API 密钥来自配置。有一个 `fallback_model` 字段——如果主模型失败，hermes 尝试一次备选。如果备选也失败，对话就停止了。

**Token 预估**：依赖 `tiktoken` 处理 OpenAI 模型。计数在代码中内联完成，不通过服务。

**Prompt 缓存**：无。每次请求都从头发送完整的系统 prompt。

**设计选择**：简单优先于韧性。hermes 是为开发者能在 OpenAI 宕机时重启 Agent 的单用户本地运行场景构建的。原型阶段可以用，生产环境不行。

**优点**：零开销。无间接调用。容易阅读和调试。
**缺点**：从 OpenAI 切换到 Anthropic 意味着重写核心循环。LLM 提供商单点故障。无自动恢复。元数据分散在多个文件中。

### openclaw

**文件**：`src/model/model-fallback.ts`（候选链）、`src/model/auth-profile.ts`（凭证轮换），模型配置散布在整个 Agent 配置系统中。

**实现**：四个项目中最精密的模型路由。

**候选链**：`ModelCandidate[]` —— 一个有序的（提供商、模型、auth-profile）元组列表。当当前候选在可重试错误（429、503、网络超时）上失败时，服务切换到下一个候选。不是单一的备选——而是任意长度的*链*。

**Auth profile 冷却**：每个 auth profile（API 密钥）跟踪自己的状态。如果某个密钥被限流，那个特定密钥进入冷却。同一提供商的其他密钥可以继续服务。这意味着你可以在同一模型的多个 API 密钥之间做负载均衡。

**瞬态探测**：当候选进入冷却后，服务定期发送轻量级探测请求以检查其是否恢复。如果探测成功，候选被恢复到活跃池。如果失败，冷却继续。所有这一切透明进行——Agent 完全不知情。

**完整流程**：
```
请求到达
  -> 尝试 candidate[0]（例如 OpenAI GPT-4o, key-A）
  -> 429 限流
  -> 标记 key-A 冷却（60s）
  -> 尝试 candidate[1]（例如 OpenAI GPT-4o, key-B）
  -> 成功 -> 返回响应
  -> 同时，60s 后探测 key-A
  -> key-A 响应 200 -> 恢复到池
  -> 下一个请求回到 candidate[0]
```

**优点**：生产级韧性。零停机的提供商轮换。Auth 密钥负载均衡。自动恢复，无需人工干预。
**缺点**：正确实现较复杂。冷却计时器、探测调度和候选排序在并发模型出错时会产生微妙的 Bug。

### NemoClaw

**文件**：`src/inference-config.ts`，配置在引导向导中。

**实现**：NemoClaw 不是 Agent 运行时——它是沙箱编排器。它对 Model Service 的贡献是配置层，而非执行层。

**统一端点**：NemoClaw 将所有模型调用路由到单一本地端点（`inference.local/v1`）。在这个端点背后，它配置了对多个提供商的访问：7 个远程（OpenAI、Anthropic、Google、Mistral 等）和 3 个本地（Ollama、LM Studio、vLLM）。

**引导向导**：7 步安装向导收集提供商凭证并配置路由规则。实际的模型调用由运行在沙箱内的 OpenClaw 实例发起。

**设计选择**：配置与执行分离。NemoClaw 决定*哪些*提供商可用以及*如何*到达它们。沙箱内的 Agent 决定*何时*调用哪个模型。

**优点**：分离干净。提供商配置是基础设施关注点，不是 Agent 关注点。
**缺点**：NemoClaw 自身没有运行时故障转移逻辑——这取决于沙箱内运行的 Agent。引导完成后切换提供商需要重跑向导或手动编辑配置。

### Claude Code

**文件**：`src/services/api/client.ts`（HTTP 客户端）、`src/services/api/claude.ts`（主入口点），以及 Direct API、AWS Bedrock 和 Google Vertex 的提供商特定模块。

**实现**：干净的依赖注入模式。`callClaude()` 是唯一的入口点。具体的提供商（Direct、Bedrock 或 Vertex）在启动时根据配置注入。切换提供商意味着改一个配置值，而不是重写代码。

**多提供商支持**：Direct API（Anthropic 自己的端点）、AWS Bedrock（面向企业 AWS 客户）、Google Vertex（面向 GCP 客户）。每个提供商有自己的认证流程、端点格式和响应解析。

**Prompt 缓存**：支持 Anthropic 原生 prompt 缓存。系统 prompt 和工具定义——在各轮之间保持一致——会被缓存。Claude Code 的缓存命中率很高，因为系统 prompt 很大（仅工具定义就可以有数千个 token）且在会话内保持稳定。

**无自动故障转移**：与 openclaw 不同，Claude Code 没有候选链。如果 Anthropic API 返回 429，它会以退避策略重试，但不会自动切换到其他提供商。这对 Anthropic 自己的产品来说合理（他们同时控制客户端和服务端），但作为多提供商 Agent 的参考有局限性。

**Token 预估**：在每次请求前进行，以确保消息数组在上下文窗口内。如果超出，则触发 Context Manager。

**优点**：通过 DI 实现最干净的代码结构。容易测试（注入 Mock 提供商）。内建 prompt 缓存支持。整个调用链类型安全。
**缺点**：提供商之间无自动故障转移。锁定在 Anthropic 模型上（这是设计意图——它是 Anthropic 的产品）。

## 对比表

| 维度 | hermes | openclaw | NemoClaw | Claude Code |
|-----------|--------|----------|----------|-------------|
| 抽象 | 无（原始 SDK） | ModelCandidate[] 链 | 仅配置层 | 基于 DI 的提供商接口 |
| 提供商数量 | 1（OpenAI） | 无限候选 | 7 远程 + 3 本地（配置） | 3（Direct/Bedrock/Vertex） |
| 故障转移 | 单一 fallback_model，一次重试 | 候选链 + auth 冷却 + 探测恢复 | 取决于内部 Agent | 退避重试，无链 |
| Auth 轮换 | N/A | 逐密钥冷却，多密钥负载均衡 | N/A | N/A |
| 自动恢复 | 否 | 是（瞬态探测） | 否 | 否 |
| Prompt 缓存 | 无 | 取决于提供商 | 取决于提供商 | Anthropic 原生缓存 |
| Token 预估 | tiktoken 内联 | 提供商 SDK | N/A | 内建请求前检查 |
| 成本追踪 | 无 | 逐请求日志 | 无 | 响应中的使用统计 |
| 代码质量 | 分散在多个文件 | 整合良好，结构清晰 | 配置分离干净 | 最佳 DI，完全类型化 |
| 测试难度 | 困难（SDK 耦合到 Core） | 中等（需要 Mock 链） | N/A | 容易（注入 Mock 提供商） |

## 最佳实践

**如果你只做一件事：构建一个带冷却和自动恢复的候选链。**

单一的 `fallback_model` 字符串是不够的。原因如下：备选也可能被限流。或者它是一个更弱的模型，无法处理你的任务。或者它更贵，你希望在主力恢复后切回来。

候选链解决了这三个问题。你定义一个有序的（提供商、模型、密钥）元组列表。失败触发下一个候选。冷却跟踪每个候选何时可以重试。探测将恢复的候选重新放回池中。

来自四个项目的规则：

1. **Core 绝不直接调用 LLM SDK。** 始终通过 `ModelService` 接口。hermes 违反了这一点，结果模型逻辑散布在 god class 中。Claude Code 遵守了，得到了干净的 DI + 轻松的测试。
2. **分类错误，不要一视同仁地处理所有失败。** 限流（429）= 临时的，冷却后重试。认证错误（401）= 永久的，不要重试。计费错误（402）= 永久的，直到用户操作。网络超时 = 临时的，立即重试。openclaw 将错误分成这些类别。hermes 没有——它对所有错误重试一次。
3. **按 auth profile 而非按提供商追踪冷却。** 如果你有两个 OpenAI API 密钥，key-A 被限流了，key-B 可能仍然可用。openclaw 做到了。其他项目将提供商视为单一单元。
4. **发送前预估 token。** 不要从 400 错误中才发现你的 prompt 超出了上下文窗口。先预估，需要的话触发压缩，然后再发送。Claude Code 和 hermes 都这么做。
5. **如果你的提供商提供 prompt 缓存，就支持它。** Anthropic 和 OpenAI 都缓存重复的 prompt 前缀。稳定的系统 prompt（在各轮之间不变）是缓存友好的前缀。如果你的系统 prompt 是 4000 token，你进行 20 轮对话，缓存大约能为每次对话节省 76,000 个输入 token。

## 分步指南：如何构建

### 第 1 步：定义接口

从 Agent Core 需要的最小契约开始：

```python
class ModelResponse:
    text: str | None
    tool_calls: list[ToolCall]
    usage: TokenUsage  # input_tokens, output_tokens, cache_hit

class ModelService(ABC):
    def complete(self, messages: list[Message], tools: list[ToolDef]) -> ModelResponse: ...
    def estimate_tokens(self, messages: list[Message]) -> int: ...
```

就这些。Core 调用 `complete()` 并获得文本和/或工具调用。Core 在发送前调用 `estimate_tokens()` 检查上下文窗口是否够用。

**决策点**：流式还是批量？如果你的界面是一个需要在 token 到达时逐个显示的 CLI，你需要流式。添加一个 `stream()` 方法来逐块产出。如果你的界面是返回完整响应的 API，批量更简单。

### 第 2 步：实现一个提供商

选择你的主力提供商并实现接口：

```python
class AnthropicProvider(ModelService):
    def __init__(self, api_key: str, model: str = "claude-sonnet-4-20250514"):
        self.client = Anthropic(api_key=api_key)
        self.model = model

    def complete(self, messages, tools):
        response = self.client.messages.create(
            model=self.model,
            messages=messages,
            tools=tools,
            max_tokens=4096,
        )
        return ModelResponse(
            text=extract_text(response),
            tool_calls=extract_tool_calls(response),
            usage=TokenUsage(response.usage),
        )
```

不要过早抽象。先把一个提供商端到端跑通，再添加第二个。

### 第 3 步：添加候选链

这里开始变得有趣。一个候选是提供商 + 模型 + 认证凭证的元组：

```python
@dataclass
class ModelCandidate:
    provider: ModelService
    priority: int
    cooldown_until: float = 0      # 时间戳
    consecutive_failures: int = 0

class CandidateChain(ModelService):
    def __init__(self, candidates: list[ModelCandidate]):
        self.candidates = sorted(candidates, key=lambda c: c.priority)

    def complete(self, messages, tools):
        for candidate in self.candidates:
            if candidate.cooldown_until > time.time():
                continue  # 仍在冷却中，跳过

            try:
                response = candidate.provider.complete(messages, tools)
                candidate.consecutive_failures = 0  # 成功时重置
                return response
            except RateLimitError:
                candidate.consecutive_failures += 1
                candidate.cooldown_until = time.time() + self._backoff(candidate)
                continue
            except AuthError:
                # 永久故障 —— 从链中移除或告警
                raise
            except NetworkError:
                # 瞬态 —— 立即尝试下一个候选
                continue

        raise AllCandidatesExhaustedError()
```

**决策点**：固定冷却还是指数退避？openclaw 使用逐候选的指数退避。先从固定 60 秒冷却开始。当你观察到 60 秒并不总是够用时，再添加指数退避。

### 第 4 步：添加自动恢复探测

没有探测，进入冷却的候选会一直待到冷却计时器过期。探测会主动检查候选是否已恢复：

```python
class CandidateChain(ModelService):
    async def _probe_loop(self):
        """后台任务：定期探测冷却中的候选。"""
        while True:
            await asyncio.sleep(30)  # 每 30 秒探测一次
            for candidate in self.candidates:
                if candidate.cooldown_until <= time.time():
                    continue  # 不在冷却中
                try:
                    # 轻量级探测：极小的消息，低 max_tokens
                    candidate.provider.complete(
                        messages=[{"role": "user", "content": "ping"}],
                        tools=[],
                    )
                    # 成功 —— 恢复候选
                    candidate.cooldown_until = 0
                    candidate.consecutive_failures = 0
                except Exception:
                    pass  # 仍然宕机，继续冷却
```

这就是 openclaw 的"瞬态探测"模式。探测很便宜（极小的请求，最少的 token）。收益很大：当主力提供商恢复时，你的 Agent 会自动切回来，而不是一直停留在昂贵的备选上。

**决策点**：在后台任务中运行探测，还是在下一次请求时惰性检查？后台探测恢复更快。惰性检查更简单，在 Agent 空闲时不会浪费探测请求。

### 第 5 步：添加 token 预估

在发送请求前，检查是否能装下：

```python
class CandidateChain(ModelService):
    def complete(self, messages, tools):
        estimated = self.estimate_tokens(messages)
        if estimated > self.context_window * 0.95:
            raise ContextWindowExceededError(estimated, self.context_window)
        # ... 继续候选链流程
```

Agent Core 捕获 `ContextWindowExceededError` 并在重试前触发 Context Manager 压缩。这是 Model Service 和 Context Manager 之间的协调点——它们不直接互相调用，而是由 Core 居中协调。

### 第 6 步（可选）：添加 prompt 缓存感知

如果使用带 prompt 缓存的 Anthropic 或 OpenAI：

```python
def complete(self, messages, tools):
    # 确保系统 prompt 是第一个元素且在各次调用间保持一致。
    # 提供商 SDK 在前缀匹配时自动处理缓存。
    # 关键：不要在各轮之间改变系统 prompt 各部分的顺序。
    system_prompt = self._build_stable_system_prompt()  # 确定性输出
    return self.client.messages.create(
        system=system_prompt,  # 缓存前缀
        messages=messages,     # 可变后缀
        tools=tools,           # 如果稳定，也是缓存前缀的一部分
    )
```

主要陷阱：如果你的系统 prompt 包含时间戳（"Current time: 2026-04-09 14:32:07"），前缀每秒都在变化，缓存永远不会命中。把易变数据放在系统 prompt 的末尾或放在用户消息中，不要放在开头。

## 常见陷阱

**1. 完全没有抽象（hermes）**
hermes 在 Agent 循环中直接调用 `openai.chat.completions.create()`。这意味着：
- 切换到 Anthropic 需要重写核心循环。
- 测试需要全局 Mock OpenAI SDK。
- 添加第二个提供商意味着复制整个调用路径。
当你只有一个提供商和一个开发者时，这看起来还行。当 OpenAI 出现 4 小时宕机而你的 Agent 也死了 4 小时时，就不行了。

**2. 有抽象但无故障转移（Claude Code）**
Claude Code 有干净的 DI 和正式的 `ModelService` 接口。但它没有候选链。如果 Anthropic API 宕机，Claude Code 以退避策略重试直到成功或放弃。对 Anthropic 自己的产品来说可以接受（他们宁愿修复 API 也不会路由到竞争对手）。对你的 Agent 来说，这不是值得复制的模式。

**3. 一视同仁地对待所有错误**
429（限流）意味着"60 秒后再试"。401（API 密钥无效）意味着"在密钥修复前永远不会成功"。500（服务器错误）意味着"立即重试或换另一个提供商"。如果你对 401 进行指数退避重试，你是在浪费时间。如果你对 429 立即放弃，你会错过一个简单的恢复机会。

错误分类表：

| 错误 | 类型 | 操作 |
|-------|------|--------|
| 429 Rate Limit | 瞬态 | 冷却候选，尝试下一个 |
| 503 Overloaded | 瞬态 | 冷却候选，尝试下一个 |
| 网络超时 | 瞬态 | 立即尝试下一个候选 |
| 401 Unauthorized | 永久 | 从链中移除候选，告警 |
| 402 Billing | 永久 | 移除候选，通知用户 |
| 400 Bad Request | Bug | 不重试，修复请求 |

**4. 忽略 prompt 缓存失效**
你的系统 prompt 有 4000 token。你进行 20 轮对话。有缓存的话，第 2-20 轮在那 4000 token 上命中缓存 = 约 72,000 token 的节省。但如果你在第 7 轮往系统 prompt 中间插入了 "Current iteration: 7"，所有后续轮次的缓存都会失效。把易变数据放在稳定前缀*之后*，或者放在用户消息中。

**5. 缺失 token 预估**
你组装了一个 180,000 token 的消息数组，发送给一个上下文窗口为 200,000 token 的模型。模型还剩 20,000 token 用于输出——可能够，可能不够。没有发送前预估，你只能从截断的响应或 400 错误中发现这个问题。先预估，需要时压缩，然后再发送。

## 跨模块契约

### Model Service 对其他模块的期望

| 模块 | 契约 | 说明 |
|--------|----------|-------|
| **Agent Core** | 调用 `complete(messages, tools)` | 消息使用标准聊天格式。Tools 使用 JSON schema 定义。 |
| **Agent Core** | 调用 `estimate_tokens(messages)` | 发送前检查上下文窗口是否够用。 |
| **Context Manager** | 需要时提供压缩后的消息 | Core 居中协调：如果 `estimate_tokens` 超过阈值，Core 调用 Context Manager，然后重试 Model。Model 和 Context Manager 之间不直接通信。 |
| **Config** | 提供商凭证、模型名称、候选顺序 | 启动时读取。Model Service 不监视配置变更（无状态）。 |

### Model Service 向其他模块提供什么

| 消费者 | 提供什么 | 格式 |
|----------|-----------------|--------|
| **Agent Core** | `ModelResponse` | `.text`（助手消息或 null）、`.tool_calls[]`（工具调用）、`.usage`（输入/输出/缓存 token 计数） |
| **Agent Core** | Token 预估 | 整数：消息数组的预估 token 数 |
| **Context Manager**（间接） | 使用量数据 | Core 从 ModelResponse 中读取 `.usage` 并传给 Context Manager 用于预算追踪。Model Service 不调用 Context Manager。 |

### 不变量

- Model Service 在重启后是无状态的。冷却计时器、探测状态和候选健康度都是瞬态运行时状态。
- Model Service 永远不修改 `messages` 数组。它接收消息，发送给提供商，返回响应。修改是 Context Manager 的职责。
- Model Service 永远不决定*发什么*。Agent Core 构建消息数组。Model Service 决定*发到哪里*（哪个提供商/候选）。
- 错误分类是 Model Service 的职责。Core 收到的要么是 `ModelResponse`，要么是类型化的异常（`AllCandidatesExhaustedError`、`PermanentAuthError`）。Core 不解析 HTTP 状态码。
- Token 预估是近似值。不同的提供商有不同的分词方式。预估应该保守（宁多勿少），以避免上下文窗口溢出。
