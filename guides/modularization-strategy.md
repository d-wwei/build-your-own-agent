# Modularization Strategy: When to Split, When Not To

> No project splits core into separate packages. All 4 keep core in one package. The lesson is clear: clean boundaries cost nothing. Physical splitting costs integration testing, version compatibility, and contract changes.

---

## The Reality Check

We read 600,000 lines of source code across 4 production agent projects. Here's how each one organizes their code:

| Project | Organization | Cost |
|---------|-------------|------|
| hermes-agent | Single repo, fully coupled | `run_agent.py` bloated to 9,431 lines -- a god class. Change one thing, break five |
| openclaw | Monorepo, core coupled, extensions independent | Adding channels/tools is frictionless. Refactoring context engine requires surgery |
| Claude Code | Single package, direct imports | Fastest startup, fastest iteration. `REPL.tsx` at 5,006 lines mixes 5 layers -- hard for newcomers |
| NemoClaw | CLI / Plugin / Blueprint loosely separated | Simplest feature set. Doesn't deal with cross-module emergent behavior complexity |

None of them split Agent Core, Model, Context Manager, or Tools into separate packages. Zero out of four.

---

## Core Insight: Boundary Does Not Equal Splitting

These are two different things:

| | Physical Splitting | Clean Boundaries |
|---|---|---|
| **Definition** | Separate package.json, independent versions | Modules communicate through interfaces, never access internals |
| **Problem solved** | Independent deployment, independent versioning, third-party extension | Change one module without breaking others. Swappable. Testable |
| **Cost** | Contract changes require cross-package updates, version compatibility, integration testing | Nearly zero -- it's just good code organization |

openclaw proves the point best. Their core (`src/`) is one package. Their extensions (`extensions/`) are separate packages. Why? Core changes frequently and modules are tightly coupled. Extensions change rarely and are independent of each other.

**Boundary is about interfaces. Splitting is about deployment.**

Put interfaces in a `contracts/` directory from day one. That's a boundary. It costs nothing. Don't create separate packages until you have a reason.

---

## Three-Phase Evolution

### Phase 1: 0 to 1 -- Monorepo single package + directory boundaries

```
agent/
├── core/           # Agent Core -- conversation loop, context building
├── model/          # Model -- provider routing, failover
├── tools/          # Tools -- registration, dispatch, execution
│   └── builtins/   # Built-in tools (shell, file, web...)
├── memory/         # Memory -- dual role, provider interface
├── context/        # Context Manager -- compression, trimming
├── interfaces/     # Interface -- CLI adapter
├── contracts/      # Cross-module interface definitions (TS interfaces / Python ABCs)
└── tests/
    ├── unit/       # Per-module unit tests
    └── loops/      # Emergent behavior loop tests <-- critical
```

**Why not split?** You don't know where the boundaries should be yet.

hermes got the Memory interface right -- a clean `MemoryProvider` ABC with 8 pluggable backends. Beautiful abstraction. But they embedded Model calls directly into the god class. Same team, same project, one boundary right, one boundary wrong.

In a single package, fixing a wrong boundary is a refactor. Cost: hours. Across packages, fixing a wrong boundary means changing contracts, updating versions, and modifying every consumer. Cost: days to weeks.

Stay in one package until you know which boundaries are stable.

### Phase 2: 1 to N -- Extract stable leaf nodes

```
agent/
├── core/               ┐
├── model/              │  Still one package (core)
├── context/            │  Changes frequently, tightly coupled, ships together
├── contracts/          ┘
│
├── packages/
│   ├── tool-shell/          <-- independent package (extracted after stabilization)
│   ├── tool-browser/        <-- independent package
│   ├── tool-mcp-client/     <-- independent package
│   ├── memory-sqlite/       <-- independent package
│   ├── memory-vector/       <-- independent package
│   ├── channel-telegram/    <-- independent package
│   └── channel-discord/     <-- independent package
```

**The rule**: 3 months without interface change = stable = can extract.

Extracting too early locks down interfaces that are still evolving. hermes's Memory interface has been stable for months -- good extraction candidate. Their Model integration is still tangled in the god class -- terrible extraction candidate.

**Extraction candidates** (leaf nodes):
- Individual tools (shell, browser, web search)
- Channel adapters (telegram, discord, slack)
- Memory providers (sqlite, vector, redis)
- Model providers (openai, anthropic, bedrock)

**Never extract** (core nodes):
- Agent Core
- Context Manager
- Model abstraction (failover logic)

### Phase 3: N to Platform -- Plugin SDK

```
On top of Phase 2, add:
├── sdk/                # Published contract package (like openclaw/plugin-sdk)
│   ├── tool.ts         # Tool interface
│   ├── channel.ts      # Channel interface
│   ├── memory.ts       # MemoryProvider interface
│   └── provider.ts     # ModelProvider interface
```

**Only do this if third parties need to extend your agent.** openclaw's 50+ subpath exports Plugin SDK solves a platform-level problem. If you're not building a platform, this is over-engineering.

---

## What to Split vs. What Not To

| Module | Can split? | Phase | Reason |
|--------|-----------|-------|--------|
| Individual tools (shell, browser, web) | Yes | Phase 2 | Each tool is independent. Only needs to conform to Tool interface |
| Channel adapters (telegram, discord) | Yes | Phase 2 | Each adapter is independent. Only needs to conform to Channel interface |
| Memory providers (sqlite, vector) | Yes | Phase 2 | Each provider is independent. Only needs to conform to MemoryProvider interface |
| Model providers (openai, anthropic) | Yes | Phase 2 | Each provider is independent. Only needs to conform to Provider interface |
| Gateway | Maybe | Phase 2+ | Naturally independent. Communicates with Core via standardized message format |
| Agent Core | No | Never | Tightest coupling with Context Manager. Highest change frequency |
| Context Manager | No | Never | Message format strongly coupled to Agent Core |
| Model abstraction (failover logic) | No | Never | Tightly interacts with Agent Core's conversation loop |

**One-line principle: Leaf nodes split freely, core nodes never split.**

---

## openclaw's Lesson

openclaw gets the tradeoff right: core tight, extensions loose.

Their `src/` directory is one package. Agent Core, Model routing, Context Engine, Hook system -- all tightly coupled, all ship together. Changing the context engine means touching the agent loop. That's fine. They're in the same package.

Their `extensions/` directory has dozens of independent packages. Each channel adapter, each tool plugin, each integration -- separate package, separate tests, separate release cycle. Adding a new Telegram feature doesn't touch Discord. Adding a new tool doesn't touch the shell tool.

The result: core iteration stays fast (no cross-package overhead), extension development stays safe (no accidental core breakage).

This is the model to follow.

---

## Common Mistakes

**Splitting too early.** You create `@myagent/core`, `@myagent/model`, `@myagent/tools` on day one. Three weeks later you realize Context Manager needs a new message field. Now you update `@myagent/core`, bump its version, update `@myagent/model` to depend on the new version, update `@myagent/tools` because it also reads messages, run integration tests across all three. What would have been a 5-minute refactor becomes a 2-hour dependency dance.

**Never splitting at all.** You keep everything in one package forever. Your `tools/` directory has 47 tool files, each with different dependencies, some pulling in heavy libraries. Every install downloads everything. Every test runs everything. Leaf nodes should eventually separate.

**Splitting core instead of leaves.** You extract Agent Core into its own package "for clean architecture." Now every change to the conversation loop requires a version bump and re-integration. The module that changes most often is the one with the highest coordination cost. Wrong direction.

---

## Decision Checklist

Before extracting a module into its own package, answer these:

1. Has the interface been stable for 3+ months? If no, don't split.
2. Is it a leaf node (tool, adapter, provider) or a core node (Agent Core, Context Manager)? If core, don't split.
3. Does it have independent dependencies that bloat the main package? If yes, stronger case to split.
4. Do third parties need to implement this interface? If yes, split + publish the interface as SDK.
5. Does extracting it simplify the main package's test suite? If no, splitting adds overhead without benefit.

If you answered "no" to questions 1 and 2, stop. Keep it in the main package.

---

*Based on modularization patterns from hermes-agent, openclaw, NemoClaw, and Claude Code. See the [Architecture Reference Model](../../docs/agent-architecture-reference.md) section 8 for detailed analysis.*
