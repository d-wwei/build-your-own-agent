# Four Projects: Side-by-Side Comparison

Four open-source Agent projects. Four different philosophies.

| Project | Philosophy | One-liner |
|---------|-----------|-----------|
| **hermes-agent** | Learning | An Agent that evolves from its own mistakes |
| **openclaw** | Connecting | A gateway that unifies 20+ platforms under one roof |
| **NemoClaw** | Security | A sandbox that makes Agents safe to run in production |
| **Claude Code** | Speed | A CLI Agent optimized for developer throughput |

We read ~60 million characters of source code across these four projects over five rounds of analysis. This page distills what we found into comparison tables you can scan in five minutes.

---

## Static Architecture Comparison

How each project implements the 10 core architectural modules.

Read each row left to right. **Bold** marks the strongest implementation for that module.

| Module | hermes | openclaw | NemoClaw | Claude Code |
|--------|--------|----------|----------|-------------|
| **Security** | `approval.py` regex + Docker optional | DM pairing + exec approval + rate limits | **Landlock + per-binary network + seccomp** | Permission modes (ask/auto) + enterprise policy |
| **Interface** | 16 adapters + CLI + ACP | **20+ channel plugins, Adapter Composition** | Management CLI only | Multiple entrypoints, no unified abstraction |
| **Gateway** | `gateway/run.py` (7620 lines), CLI bypasses it | **WebSocket control plane, 25+ handler groups** | Wraps OpenShell | Does not exist |
| **Routing** | None (single Agent) | **9-level binding priority, independent module** | N/A | Does not exist |
| **Agent Core** | `AIAgent` class, 9431-line god class | **Cleanest separation: `runEmbeddedPiAgent()`** | Uses OpenClaw's | `query.ts` + `REPL.tsx` dual-loop |
| **Model** | Direct OpenAI SDK, no abstraction | **Failover candidate chain + cooldown probing** | Config layer (`inference-config.ts`) | `services/api/` with clean DI |
| **Tools** | Registry self-register, 50+ tools | **Plugin SDK registration, 100+ tools** | Management commands only | `Tool<I,O,P>` generics, 60+ tools |
| **Memory** | **8 pluggable backends via `MemoryProvider` ABC** | Dual registration + "dreaming" background cleanup | Not applicable | `SessionMemory` + background extraction |
| **Context** | Compressor: prune first, summarize second | **Pluggable context engine with proactive compression** | Not applicable | Three strategies: auto / reactive / snip |
| **Runtime/Infra** | Docker / Modal / SSH multi-backend | Docker / Fly.io / Tailscale | **OpenShell Blueprint with Landlock + seccomp** | npm install, runs as local process |

### Reading the Table

A few things stand out immediately:

- **Security**: NemoClaw is in a different league. Landlock filesystem isolation + per-binary network rules + seccomp syscall filtering. The other three projects bolt on security as an afterthought.
- **Gateway / Routing**: Only matters for multi-user or multi-Agent deployments. Claude Code skips both entirely and that is the right call for a single-user CLI.
- **Memory**: hermes has the best abstraction (8 pluggable backends). openclaw has the best runtime behavior (background "dreaming" cleanup). Claude Code takes a minimal approach.
- **Context**: Every project solves this differently. None of them are fully satisfied with their solution.

### Tool Registration and Execution Patterns

Worth a separate look because tool design varies dramatically.

| Aspect | hermes | openclaw | Claude Code |
|--------|--------|----------|-------------|
| **Registration** | `registry.register()` at import time | `api.registerTool()` via Plugin SDK | Static imports + feature flag dead code elimination |
| **Discovery** | All registered tools available to all Agents | Per-Agent tool whitelist via config | Built-in set, MCP tools discovered at startup |
| **Parallel Execution** | Path-independence detection, max 8 threads | `isConcurrencySafe` flag per tool | Batch splitting, max 10 concurrent |
| **Security Check** | `approval.py` regex on shell commands | Exec approval flow via DM | `Tool.checkPermissions()` + permission modes |
| **Third-party Extension** | Not supported | Full Plugin SDK with NPM distribution | MCP servers only |

---

## Dynamic Behavior Comparison

Emergent behaviors are capabilities no single component was designed to produce. They arise from interactions between components.

There is no `SelfEvolutionEngine` class in hermes. There is no `SpeculativeExecutor` in Claude Code. These behaviors emerge from simple components doing simple things in combination.

| Behavior | hermes | openclaw | NemoClaw | Claude Code |
|----------|--------|----------|----------|-------------|
| **Learning Loop** | **SKILLS_GUIDANCE instruction -> skill_manage CRUD -> prompt_builder reload -> skill_patch fix. Only project with a complete closed loop.** | Workspace skills exist but no auto-creation | N/A | `/remember` + auto-extract. Saves facts, not reusable methodologies |
| **Speculative Execution** | No. Strict serial execution | No. Standard request-response | N/A | **SpeculationState tracks parallel permission-request + tool-execution. CompletionBoundary handles rollback.** |
| **Self-Healing Model Routing** | `fallback_model` config, single downgrade only | **ModelCandidate[] chain -> auth profile rotation -> cooldown marking -> transient probe -> auto-recovery** | Multi-provider config, manual switch | Multi-provider support, no automatic failover chain |
| **Context Pressure Cascade** | L1 prune tool results -> L2 LLM summary -> budget_warning injection | **Pluggable context engine + proactive compression + transcript rewrite** | N/A | Three parallel strategies: auto (80% threshold) + reactive (backpressure) + snip (fine-grained) |
| **Sub-Agent Delegation** | `delegate_tool` -> independent budget -> MAX_DEPTH=2 -> 3 parallel max | Lane queuing + subagent hooks | N/A | **AgentTool: worktree isolation + fork + team/swarm + coordinator + async** |
| **Multi-Agent Specialization** | No. Single-Agent architecture | **9-level binding routing -> per-conversation Agent assignment -> each Agent has own tools/memory/persona** | N/A | No. Single-user, single-Agent |
| **Self-Healing Runtime** | No runtime health management | No | **`gateway-state.ts` health classification -> detect gateway death -> SSH + curl probe -> auto-restart -> registry recovery** | No |

**Bold** = best-in-class for that behavior.

### What "Emergent" Means in Practice

Take the **Learning Loop** in hermes as an example:

1. `SKILLS_GUIDANCE` prompt instruction tells the LLM to save useful patterns (Agent Core)
2. LLM decides "this is worth saving" and calls `skill_manage create` (Model + Tools)
3. A markdown skill file gets written to `~/.hermes/skills/` (Skill Store)
4. Next conversation, `prompt_builder` scans the skills directory and injects relevant skills (Agent Core)
5. LLM behavior changes based on the loaded skill (Model)
6. If the skill is wrong, LLM calls `skill_manage patch` to fix it (Tools)

No single component "does" learning. Five components, each doing one simple thing, produce an Agent that gets better over time.

### NemoClaw Note

NemoClaw shows N/A for most dynamic behaviors because it is not an Agent. It is a security runtime that wraps OpenClaw. Its one emergent behavior -- self-healing runtime -- is the only one any project implements at the infrastructure level.

---

## Each Module: Who to Reference

Building your own Agent? Here is which project to study for each module.

| Module | Recommended Reference | Why |
|--------|----------------------|-----|
| Agent Core | openclaw (cleanest) or hermes (simplest) | openclaw separates concerns well; hermes fits in one file |
| Model Abstraction | openclaw failover + Claude Code DI | Candidate chain + cooldown probing is production-critical; DI makes testing easy |
| Tool Registration | hermes (simple) or openclaw (platform) | No third-party extensions needed? Use hermes. Need a plugin ecosystem? Use openclaw |
| Memory | hermes Provider ABC + openclaw dual-registration | hermes has the best abstraction; openclaw has the best dual-role implementation |
| Context Management | Claude Code three-strategy or openclaw pluggable engine | Three strategies = most flexible; pluggable engine = most extensible |
| Gateway | openclaw | Only project that did it properly |
| Security | NemoClaw | Only project that took it seriously |
| Runtime / Infra | NemoClaw | Only project that built it |

---

## Project Stats

| Stat | hermes | openclaw | NemoClaw | Claude Code |
|------|--------|----------|----------|-------------|
| **Language** | Python 3.11+ | TypeScript ESM | JS + TS | TS + React + Bun |
| **Approx. Lines** | ~80k | ~300k | ~20k | ~200k |
| **Core Innovation** | Closed-loop learning from experience | Plugin ecosystem + multi-platform unification | Security sandbox with OS-level isolation | Speculative execution + enterprise-grade performance |
| **Team** | Nous Research | Community | NVIDIA | Anthropic |
| **Maturity** | Production-ready | Production-ready | Alpha | Production (commercial) |
| **Can Run Standalone** | Yes | Yes | No (needs OpenClaw) | Yes |
| **Open Source** | Yes | Yes | Yes | Source available (leaked build) |
| **God Class Risk** | High (`AIAgent` 9431 lines) | Low (clean separation) | Low (simple scope) | Medium (`REPL.tsx` 5006 lines) |
| **Best For Studying** | Learning, Memory, Tool registration | Gateway, Routing, Model failover, Plugin SDK | Security, Runtime isolation | Performance, Sub-agent patterns, Context strategies |

---

## Architecture Pattern: Hub-and-Spoke

All four projects independently confirm the same topology. Agent architecture is **not layered**. It is Hub-and-Spoke.

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

Agent Core sits at the center. Model, Tools, Memory, and Context Manager are **peer services** -- not layers stacked on top of each other. They do not call each other. They all talk to the hub.

We initially drew a layered diagram. It was wrong. Took five revision rounds to arrive at Hub-and-Spoke.

---

## Key Takeaways

1. **No single project does everything well.** Each one made trade-offs that reflect its philosophy.
2. **The architecture is Hub-and-Spoke, not layered.** All four projects confirm this independently.
3. **The most valuable capabilities are emergent.** Learning loops, speculative execution, and self-healing routing are not designed as features. They arise from component interactions.
4. **Security is the most neglected area.** Only NemoClaw took it seriously. The other three treat it as an afterthought.
5. **Context management is universally hard.** Every project that runs an LLM loop invented its own multi-level compression strategy.
