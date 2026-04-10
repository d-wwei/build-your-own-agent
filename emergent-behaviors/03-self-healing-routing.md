# Emergent Behavior 3: Self-Healing Model Routing

> The system recovers from LLM provider failures and switches back to the primary model without any user awareness or manual intervention.

## What Is It

When the primary LLM hits a rate limit, the agent degrades to a backup model. When the primary recovers, the agent switches back. The user never notices. No downtime. No error messages. No manual reconfiguration.

This is zero-downtime degradation with automatic recovery.

## How It Emerges

There is no `SelfHealingRouter` class. The behavior assembles itself from four independent mechanisms:

1. **model-fallback.ts** maintains a `ModelCandidate[]` array. When a call fails, it tries the next candidate. That is all it knows how to do.
2. **Auth profiles** rotate independently. Each profile tracks its own failure count and cooldown timestamp. A profile marked "cooling down" is skipped. It does not know about fallback logic.
3. **Cooldown timers** reset independently. When a timer expires, the profile becomes eligible again. The timer does not know about model selection.
4. **Probe logic** is separate. A lightweight test request is sent to check if a cooled-down provider works. The probe does not know it is part of a healing cycle.

None of these four pieces reference each other's purpose. But when combined:

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

The system heals itself. No single component was designed to do that.

## Which Components Participate

All within the **Model** service boundary:

| Mechanism | Role in the Behavior | Awareness of the Whole |
|---|---|---|
| Failover candidate chain | Picks next available model | Knows nothing about healing |
| Auth profile cooldown | Marks providers as temporarily unavailable | Knows nothing about fallback |
| Cooldown timer | Controls when a provider becomes eligible again | Knows nothing about probing |
| Transient probing | Tests if a provider has recovered | Knows nothing about switching |

This is important: the entire emergent behavior lives inside one architectural module (Model). It does not cross module boundaries. That makes it easier to implement and test than behaviors that span multiple modules.

## Case Study: openclaw

openclaw's model layer defines a `ModelCandidate[]` chain per agent configuration. A typical setup:

```
ModelCandidate[0]: GPT-4o via auth profile "primary-openai"
ModelCandidate[1]: Claude 3.5 via auth profile "backup-anthropic"
ModelCandidate[2]: GPT-4o-mini via auth profile "fallback-openai-mini"
```

**What happens during a rate limit event:**

1. GPT-4o returns HTTP 429. The error is classified as `rate_limit` (not `billing`, not `auth`).
2. `model-fallback.ts` catches the classified error. It marks "primary-openai" auth profile with a cooldown timestamp (e.g., 60 seconds).
3. The same request is retried against `ModelCandidate[1]` (Claude 3.5). User gets a response. They do not know the model changed.
4. For the next 60 seconds, all requests route to Claude 3.5.
5. At second 61, the cooldown expires. The next request triggers a lightweight probe to GPT-4o.
6. Probe succeeds. Subsequent requests go back to GPT-4o.
7. If the probe had failed, cooldown would extend (e.g., double to 120 seconds). Backup continues.

The key insight: openclaw achieves multi-model zero-downtime service using four mechanisms that each do one simple thing. The "self-healing" label describes the combined behavior, not any individual piece of code.

## How to Make It Emerge in Your Agent

You need four pieces. Build them independently. The healing behavior will emerge from their combination.

### Piece 1: Failover Candidate Chain

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

### Piece 2: Error Classification

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

Not all errors should trigger cooldown. `billing` means the account is out of funds -- cooldown will not fix it. `auth` means credentials are wrong. Only `rate_limit` and `server` are transient and worth cooling down.

### Piece 3: Cooldown State

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

### Piece 4: Probing

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

**When to probe:** On the next real request after cooldown expires. Not on a timer. Not in a background loop. The request itself triggers the probe check. This avoids wasted API calls when the agent is idle.

### Assembly

Wire these four pieces into your model call path:

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

That is it. No `SelfHealingRouter`. No orchestrator. Four independent mechanisms. The healing emerges.

## Cross-Module Contracts

Since this behavior lives entirely within the Model module, the contracts are internal. But they must be explicit to avoid breaking the emergence during refactoring.

| Contract | Format | Between |
|---|---|---|
| Error classification enum | `"rate_limit" \| "billing" \| "auth" \| "server" \| "unknown"` | Error handler <-> Cooldown logic |
| Cooldown state | `{ profileId, cooldownUntil, failureCount, lastError }` | Cooldown logic <-> Model selector <-> Probe logic |
| Probing protocol | Lightweight request (1 token prompt, short max_tokens) returning success/failure | Probe logic <-> API client |
| Candidate ordering | `ModelCandidate[]` sorted by `priority`, filtered by cooldown | Config <-> Model selector |

If you change the error classification enum without updating the cooldown trigger conditions, the healing breaks silently. Define these as shared types in your `contracts/` directory.

## How to Test It (Loop Test)

File: `tests/loops/model-failover.test.ts`

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

**What this test catches:** Any refactoring that breaks the connection between error classification and cooldown triggering, between cooldown expiry and probe firing, or between probe success and candidate re-selection.

## Variations Across Projects

| Aspect | openclaw | hermes-agent | Claude Code | NemoClaw |
|---|---|---|---|---|
| Candidate chain | `ModelCandidate[]` with priority ordering | Single `fallback_model` config field | Multi-provider (Direct / Bedrock / Vertex) | Multi-provider config (7 remote + 3 local) |
| Degradation depth | N candidates deep (chain) | One level only (primary -> fallback) | No auto-failover chain | No auto-failover |
| Cooldown | Per-auth-profile with exponential backoff | None | None | None |
| Probing | Transient probe on cooldown expiry | None | None | None |
| Auto-recovery | Yes, fully automatic | No. Once degraded, stays degraded until restart | No. Provider selected at config time | No. Requires manual onboard reconfiguration |
| User awareness | Zero. User never knows | User may notice model quality change | User selects provider explicitly | Admin must reconfigure |

**The gap is stark.** Only openclaw achieves the full self-healing cycle. The other three projects handle the "degrade" half but not the "heal" half. hermes falls back once and stays there. Claude Code and NemoClaw do not fall back automatically at all.

If you are building an agent that needs production reliability, implement all four pieces. If you only need basic resilience, implement pieces 1-3 (failover + classification + cooldown) and skip probing. You will get degradation without auto-recovery, which is still better than crashing.
