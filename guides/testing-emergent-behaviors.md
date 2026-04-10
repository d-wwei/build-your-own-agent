# Testing Emergent Behaviors: Loop Tests

> Emergent behaviors depend on cross-module implicit contracts. Unit tests can't protect them. You need dedicated loop tests.

---

## The Problem

An emergent behavior is something no single module is designed to do, but the system does anyway when modules combine.

hermes has no `SelfEvolutionEngine` class. But SKILLS_GUIDANCE instructions + skill_manage tool + Skill Store files + prompt_builder loading = a system that learns from experience. Four simple components, one complex behavior.

Unit-test each component in isolation? They all pass. Change the skill file format? prompt_builder still loads files, skill_manage still writes files. Unit tests green. But the learning loop is broken -- the format prompt_builder expects doesn't match what skill_manage writes. The emergent behavior silently dies.

You need tests that exercise the full loop.

---

## The Solution: `tests/loops/`

A dedicated directory for integration tests that verify emergent behaviors end-to-end.

```
tests/loops/
├── learning-loop.test.ts
├── context-cascade.test.ts
├── model-failover.test.ts
├── delegation.test.ts
└── speculative-exec.test.ts
```

Each test simulates the full cycle of one emergent behavior. Not mocking internals. Using real module code. Only mocking external dependencies (LLM API, filesystem in some cases).

---

## The 5 Loop Tests

### 1. learning-loop.test

Tests the learning cycle: experience leads to skill creation, skill persists across sessions, skill can be updated.

```
Test sequence:
1. Execute 5 tool calls in a conversation
2. Verify a skill file is created in Skill Store
3. Start a new session
4. Verify the system prompt contains the skill content
5. Modify the skill via skill_patch
6. Start another session
7. Verify the loaded skill reflects the modification
```

**What breaks this**: Changes to skill file format (frontmatter schema), changes to Skill Store path conventions, changes to prompt injection order, changes to skill CRUD tool parameters.

**Modules exercised**: Agent Core, Tools, Skill Store, prompt builder.

### 2. context-cascade.test

Tests the pressure escalation: as context fills up, compression strategies activate in sequence.

```
Test sequence:
1. Fill context to 80% of token budget
2. Verify L1 triggers (prune old tool results, zero LLM calls)
3. Keep filling context
4. Verify L2 triggers (LLM-powered summarization of middle turns)
5. Verify budget warning message is injected into context
6. Verify the LLM sees the warning (check messages array)
```

**What breaks this**: Changes to compression input/output format (messages array structure), changes to budget warning injection position or format, changes to token estimation interface, threshold changes without test updates.

**Modules exercised**: Agent Core, Context Manager, Model (for token estimation).

### 3. model-failover.test

Tests the self-healing cycle: failure, fallback, cooldown, probe, recovery.

```
Test sequence:
1. Mock main model returning 429 (rate limit)
2. Verify agent switches to backup model
3. Verify agent continues serving normally on backup
4. Mock cooldown period elapsed
5. Verify probe request sent to main model
6. Mock main model recovered (200 OK)
7. Verify agent switches back to main model
8. Verify ongoing conversation is uninterrupted
```

**What breaks this**: Changes to error classification enum (rate_limit vs billing vs auth), changes to cooldown state format, changes to probe protocol, new error types not mapped to the classification.

**Modules exercised**: Model (internal -- candidate chain, cooldown, probing).

### 4. delegation.test

Tests sub-agent lifecycle: spawn, constrain, execute, summarize, return.

```
Test sequence:
1. Trigger a delegation (LLM decides subtask is delegatable)
2. Verify child agent receives independent token budget
3. Verify child agent's tool set is restricted (no recursive delegation)
4. Let child agent complete its task
5. Verify child returns a summary message
6. Verify parent agent's context grew by exactly 1 message (the summary)
7. Verify parent agent continues normally
```

**What breaks this**: Changes to child agent constructor parameters, changes to tool blacklist format, changes to summary return format, changes to depth limit enforcement.

**Modules exercised**: Agent Core (parent + child instances), Tools (restriction logic).

### 5. speculative-exec.test

Tests parallel permission + execution: the agent starts work before the user approves.

```
Test sequence:
1. LLM returns a tool_call requiring permission
2. Verify permission request AND tool execution start in parallel
3. Mock user approves
4. Verify result is displayed immediately (no execution delay)
5. Reset. LLM returns another tool_call
6. Mock user rejects
7. Verify result is discarded
8. Verify state rolls back to CompletionBoundary
```

**What breaks this**: Changes to ToolResult rollback state format, changes to permission request async cancellation, changes to CompletionBoundary format, removal of parallel execution path.

**Modules exercised**: Agent Core, Tools, Interface (permission UI).

---

## Why Loop Tests Work

The key insight: emergent behaviors are composed entirely of core modules. Core modules never split into separate packages (see [Modularization Strategy](modularization-strategy.md)). Therefore, loop tests always run in the same package as the code they test.

When you extract a leaf node (a specific tool, a channel adapter) into its own package, no loop test breaks. The loop tests use mock tools and mock channels. The real tool packages have their own unit tests.

```
Phase 1 (single package):  Loop tests run directly. Zero config.
Phase 2 (leaves extracted): Loop tests still run in core package.
                             All loop participants are still here.
Phase 3 (plugin SDK added): Loop tests unchanged.
                             SDK consumers write their own tests.
```

This is by design, not by accident. The modularization strategy protects loop test validity.

---

## Cross-Module Contract Table

Every emergent behavior depends on implicit contracts between modules. When upgrading any module, check this table to know which contracts you must maintain.

| Emergent Behavior | Key Contracts | Modules Involved |
|---|---|---|
| Learning Loop | Skill file format (frontmatter schema) -- Storage path conventions -- Prompt injection method -- Skill CRUD parameters | Agent Core <-> Tools <-> Skill Store |
| Speculative Execution | ToolResult supports rollback state -- Permission requests are async-cancellable -- CompletionBoundary format | Agent Core <-> Tools <-> Interface |
| Self-Healing Model Routing | Error classification enum (rate_limit / billing / auth) -- Cooldown state format -- Probe protocol | Model (internal) |
| Context Pressure Cascade | Compression I/O format (messages array) -- Budget warning injection position and format -- Token estimation interface | Agent Core <-> Context Manager <-> Model |
| Sub-Agent Delegation | Child agent constructor params -- Tool blacklist -- Summary return format -- Depth limit | Agent Core <-> Tools |

Before changing any of these contracts, find the corresponding loop test. Run it. If it passes after your change, the emergent behavior survives. If it fails, you broke a cross-module assumption.

---

## Practical Tips

**Run loop tests on every PR.** They're integration tests but they're fast -- no real LLM calls, no real filesystem (unless testing persistence). Mock the LLM to return predictable tool_calls. A full loop test suite runs in under 10 seconds.

**Name tests after behaviors, not modules.** `learning-loop.test`, not `skill-store-integration.test`. The point is testing the emergent behavior, not any single module.

**When a loop test fails, the bug is at a boundary.** The individual modules work (unit tests prove that). The failure is in the contract between them. Check message formats, parameter shapes, and injection points.

**Add a new loop test when you add a new emergent behavior.** If you implement model failover, add `model-failover.test` before shipping. If the behavior doesn't have a loop test, the first refactor will silently break it.

---

*Based on testing patterns from hermes-agent, openclaw, NemoClaw, and Claude Code. See the [Architecture Reference Model](../../docs/agent-architecture-reference.md) sections 8.5 and 8.6 for the original analysis.*
