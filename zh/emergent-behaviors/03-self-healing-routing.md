# 涌现行为 3：自愈式 Model Routing

> 系统能够从 LLM 提供商故障中自动恢复，并在主模型可用后自动切回——用户全程无感知，无需任何人工干预。

## 它是什么

当主 LLM 遇到速率限制时，Agent 会降级到备用模型。主模型恢复后，Agent 自动切回。用户全程无感知。没有停机。没有错误提示。没有手动重新配置。

这就是零停机降级与自动恢复。

## 它如何涌现

系统中没有 `SelfHealingRouter` 类。这个行为由四个独立机制自发组装而成：

1. **model-fallback.ts** 维护一个 `ModelCandidate[]` 数组。当调用失败时，它尝试下一个候选。这是它唯一知道的事。
2. **Auth profiles** 独立轮换。每个 profile 独立跟踪自己的失败次数和冷却时间戳。被标记为"冷却中"的 profile 会被跳过。它不知道 fallback 逻辑的存在。
3. **冷却计时器**独立重置。当计时器到期，profile 重新变为可用。计时器不知道模型选择的存在。
4. **探测逻辑**是独立的。一个轻量级测试请求被发送以检查冷却中的提供商是否恢复。探测不知道自己是自愈循环的一部分。

这四个部件互不引用对方的意图。但当它们组合在一起：

```
Main model rate_limit error
  -> mark auth profile cooldown + record failure
    -> model-fallback.ts picks ModelCandidate[1]
      -> normal service continues on backup
        -> cooldown timer expires
          -> probe request sent to main model
            -> success = auto switch back (user unaware)
            -> failure = extend cooldown, stay on backup
```

系统自我修复了。没有任何单一组件被设计来做这件事。

## 哪些组件参与

全部位于 **Model** 服务边界内：

| 机制 | 在该行为中的角色 | 对整体的感知 |
|---|---|---|
| Failover candidate chain | 选择下一个可用模型 | 对自愈一无所知 |
| Auth profile cooldown | 将提供商标记为暂时不可用 | 对 fallback 一无所知 |
| Cooldown timer | 控制提供商何时重新可用 | 对探测一无所知 |
| Transient probing | 测试提供商是否已恢复 | 对切换一无所知 |

这一点很重要：整个涌现行为完全存在于一个架构模块（Model）内部。它不跨越模块边界。这使得它比跨多个模块的行为更容易实现和测试。

## 案例研究：openclaw

openclaw 的模型层为每个 Agent 配置定义了一个 `ModelCandidate[]` 链。典型配置：

```
ModelCandidate[0]: GPT-4o via auth profile "primary-openai"
ModelCandidate[1]: Claude 3.5 via auth profile "backup-anthropic"
ModelCandidate[2]: GPT-4o-mini via auth profile "fallback-openai-mini"
```

**速率限制事件发生时：**

1. GPT-4o 返回 HTTP 429。错误被分类为 `rate_limit`（不是 `billing`，不是 `auth`）。
2. `model-fallback.ts` 捕获分类后的错误。它将"primary-openai" auth profile 标记为带冷却时间戳（例如 60 秒）。
3. 同一请求被重试到 `ModelCandidate[1]`（Claude 3.5）。用户得到响应。他们不知道模型变了。
4. 接下来 60 秒内，所有请求路由到 Claude 3.5。
5. 第 61 秒，冷却到期。下一个请求触发对 GPT-4o 的轻量级探测。
6. 探测成功。后续请求切回 GPT-4o。
7. 如果探测失败，冷却时间会延长（例如翻倍到 120 秒）。继续使用备用模型。

核心洞察：openclaw 用四个各自只做一件简单事的机制，实现了多模型零停机服务。"自愈"这个标签描述的是组合行为，而非任何单独的代码。

## 如何在你的 Agent 中让它涌现

你需要四个部件。独立构建它们。自愈行为将从它们的组合中涌现。

### 部件 1：Failover Candidate Chain

```typescript
interface ModelCandidate {
  provider: string;       // "openai" | "anthropic" | etc
  model: string;          // "gpt-4o" | "claude-3.5-sonnet"
  authProfileId: string;  // links to auth credential set
  priority: number;       // 0 = primary, 1 = first backup, etc
}

// Selection logic: pick the lowest-priority candidate
// whose auth profile is NOT in cooldown.
function selectModel(candidates: ModelCandidate[]): ModelCandidate {
  return candidates
    .sort((a, b) => a.priority - b.priority)
    .find(c => !isInCooldown(c.authProfileId));
}
```

### 部件 2：错误分类

```typescript
type ModelErrorClass = "rate_limit" | "billing" | "auth" | "server" | "unknown";

function classifyError(status: number, body: any): ModelErrorClass {
  if (status === 429) return "rate_limit";
  if (status === 402) return "billing";
  if (status === 401 || status === 403) return "auth";
  if (status >= 500) return "server";
  return "unknown";
}
```

并非所有错误都应触发冷却。`billing` 意味着账户余额不足——冷却无法修复。`auth` 意味着凭据错误。只有 `rate_limit` 和 `server` 是暂时性的，值得进入冷却。

### 部件 3：Cooldown 状态

```typescript
interface CooldownState {
  profileId: string;
  cooldownUntil: number;    // timestamp ms
  failureCount: number;     // for exponential backoff
  lastError: ModelErrorClass;
}

function markCooldown(profileId: string, errorClass: ModelErrorClass): void {
  const baseSeconds = 60;
  const state = getCooldownState(profileId);
  const backoff = Math.min(baseSeconds * Math.pow(2, state.failureCount), 3600);
  state.cooldownUntil = Date.now() + backoff * 1000;
  state.failureCount++;
  state.lastError = errorClass;
}
```

### 部件 4：探测

```typescript
async function probeIfReady(profileId: string): Promise<boolean> {
  const state = getCooldownState(profileId);
  if (Date.now() < state.cooldownUntil) return false;

  // Lightweight request -- minimal tokens, no user-facing impact
  const success = await sendProbeRequest(profileId);

  if (success) {
    resetCooldown(profileId);
    return true;
  } else {
    // Extend cooldown
    markCooldown(profileId, state.lastError);
    return false;
  }
}
```

**何时探测：** 在冷却到期后的下一个真实请求时。不是定时器触发。不是后台循环。请求本身触发探测检查。这避免了 Agent 空闲时浪费 API 调用。

### 组装

将这四个部件接入你的模型调用路径：

```
callModel(messages)
  -> selectModel(candidates)     // picks non-cooldown candidate
  -> try apiCall(selected, messages)
  -> on error:
       classify(error)
       if transient: markCooldown(selected.authProfileId)
       retry with selectModel(candidates)  // picks next
  -> before each call: probeIfReady(primaryProfileId)
       if true: primary is back, use it
```

就是这样。没有 `SelfHealingRouter`。没有编排器。四个独立机制。自愈涌现。

## 跨模块契约

由于这个行为完全存在于 Model 模块内部，契约都是内部的。但它们必须是显式的，以避免在重构时破坏涌现。

| 契约 | 格式 | 关联方 |
|---|---|---|
| Error classification enum | `"rate_limit" \| "billing" \| "auth" \| "server" \| "unknown"` | Error handler <-> Cooldown logic |
| Cooldown state | `{ profileId, cooldownUntil, failureCount, lastError }` | Cooldown logic <-> Model selector <-> Probe logic |
| Probing protocol | Lightweight request (1 token prompt, short max_tokens) returning success/failure | Probe logic <-> API client |
| Candidate ordering | `ModelCandidate[]` sorted by `priority`, filtered by cooldown | Config <-> Model selector |

如果你更改了 error classification enum 而没有同步更新 cooldown 触发条件，自愈会静默失效。在你的 `contracts/` 目录中将这些定义为共享类型。

## 如何测试（循环测试）

文件：`tests/loops/model-failover.test.ts`

```
Test: "Self-healing model routing completes full cycle"

1. Configure: ModelCandidate[0] = main, ModelCandidate[1] = backup
2. Mock main model → return 429 (rate_limit)
3. Call agent.chat("hello")
4. Assert: request went to backup model
5. Assert: main auth profile is in cooldown state
6. Assert: user received a valid response (no error surfaced)

7. Advance time past cooldown period
8. Call agent.chat("hello again")
9. Assert: probe request was sent to main model

10. Mock main model → return 200 (recovered)
11. Assert: probe succeeded
12. Assert: main auth profile cooldown is cleared

13. Call agent.chat("third message")
14. Assert: request went to main model (switched back)

15. Bonus — probe failure path:
    Mock main model → still 429 on probe
    Assert: cooldown extended (longer than initial)
    Assert: requests continue on backup
```

**此测试捕获的内容：** 任何破坏错误分类与冷却触发之间连接、冷却到期与探测触发之间连接、或探测成功与候选重选之间连接的重构。

## 各项目差异

| 维度 | openclaw | hermes-agent | Claude Code | NemoClaw |
|---|---|---|---|---|
| Candidate chain | `ModelCandidate[]` with priority ordering | Single `fallback_model` config field | Multi-provider (Direct / Bedrock / Vertex) | Multi-provider config (7 remote + 3 local) |
| 降级深度 | N 个候选的链式降级 | 仅一层（primary -> fallback） | 无自动 failover chain | 无自动 failover |
| Cooldown | Per-auth-profile with exponential backoff | 无 | 无 | 无 |
| 探测 | Cooldown 到期时的 transient probe | 无 | 无 | 无 |
| 自动恢复 | 是，完全自动 | 否。降级后保持不变，直到重启 | 否。Provider 在配置时选定 | 否。需要手动重新配置 |
| 用户感知 | 零。用户永远不知道 | 用户可能注意到模型质量变化 | 用户显式选择 provider | 管理员必须重新配置 |

**差距非常明显。** 只有 openclaw 实现了完整的自愈循环。其他三个项目处理了"降级"的一半，但没有"恢复"的一半。hermes 降级一次后就留在那里。Claude Code 和 NemoClaw 根本不会自动降级。

如果你在构建需要生产可靠性的 Agent，请实现全部四个部件。如果你只需要基本的弹性能力，实现部件 1-3（failover + classification + cooldown）并跳过探测。你将获得无自动恢复的降级能力，这仍然比直接崩溃好得多。
