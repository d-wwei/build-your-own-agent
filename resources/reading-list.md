# Reading List: Source Repos, Papers, and Tutorials

Everything we studied, plus resources we recommend for building your own Agent.

---

## The 4 Source Projects

These are the projects we analyzed in depth.

| Project | Repo | Language | Focus |
|---------|------|----------|-------|
| **hermes-agent** | [github.com/NousResearch/hermes-agent](https://github.com/NousResearch/hermes-agent) | Python 3.11+ | Self-evolving Agent with closed-loop learning. Best reference for Memory and Learning Engine. |
| **openclaw** | [github.com/open-claw/open-claw](https://github.com/open-claw/open-claw) | TypeScript ESM | Multi-channel AI gateway with plugin ecosystem. Best reference for Gateway, Routing, Model failover, and Context Engine. |
| **NemoClaw** | NVIDIA (companion to openclaw) | JS + TS | Security sandbox runtime. Best reference for Security Envelope and Runtime/Infrastructure. |
| **Claude Code** | Anthropic CLI Agent (source via leaked build) | TS + React + Bun | Performance-optimized developer Agent. Best reference for Speculative Execution, Sub-Agent delegation, and Context strategies. |

---

## Claude Certified Architect

Study guide and course list for Anthropic's official certification.

- **Study Guide**: [github.com/paullarionov/claude-certified-architect](https://github.com/paullarionov/claude-certified-architect)

### Anthropic Academy Free Courses (13)

1. [Claude 101](https://anthropic.skilljar.com/claude-101)
2. [AI Fluency: Framework & Foundations](https://anthropic.skilljar.com/ai-fluency-framework-foundations)
3. [Introduction to Agent Skills](https://anthropic.skilljar.com/introduction-to-agent-skills)
4. [Building with the Claude API](https://anthropic.skilljar.com/claude-with-the-anthropic-api)
5. [Claude Code in Action](https://anthropic.skilljar.com/claude-code-in-action)
6. [Intro to Model Context Protocol](https://anthropic.skilljar.com/introduction-to-model-context-protocol)
7. [MCP: Advanced Topics](https://anthropic.skilljar.com/model-context-protocol-advanced-topics)
8. [AI Fluency for Students](https://anthropic.skilljar.com/ai-fluency-for-students)
9. [AI Fluency for Educators](https://anthropic.skilljar.com/ai-fluency-for-educators)
10. [Teaching AI Fluency](https://anthropic.skilljar.com/teaching-ai-fluency)
11. [AI Fluency for Nonprofits](https://anthropic.skilljar.com/ai-fluency-for-nonprofits)
12. [Claude with Amazon Bedrock](https://anthropic.skilljar.com/claude-in-amazon-bedrock)
13. [Claude with Google Cloud's Vertex AI](https://anthropic.skilljar.com/claude-with-google-vertex)

Courses 1-7 are the most relevant for Agent architecture.

---

## Agent Architecture Resources

Guides and specifications directly relevant to building Agents.

- **Building Effective Agents** -- Anthropic's guide to agentic patterns. Covers orchestration, tool use, and evaluation: [anthropic.com/research/building-effective-agents](https://www.anthropic.com/research/building-effective-agents)
- **Model Context Protocol (MCP)** -- The open standard for tool integration. Defines Tools, Resources, and Prompts: [modelcontextprotocol.io](https://modelcontextprotocol.io)
- **Agent Protocol** -- Standardized Agent communication interface: [agentprotocol.ai](https://agentprotocol.ai)
- **Agent Communication Protocol (ACP)** -- IBM's Agent-to-Agent protocol. hermes-agent implements this for inter-Agent communication.
- **OpenAI Function Calling** -- The original tool_call specification that all four projects implement in some form: [platform.openai.com/docs/guides/function-calling](https://platform.openai.com/docs/guides/function-calling)

---

## Related Concepts

These are patterns and ideas referenced throughout our analysis.

- **Hub-and-Spoke Architecture** -- The core Agent topology. Agent Core at center, services as peer spokes. Not layered. All four projects confirm this independently.
- **Emergent Behavior in Complex Systems** -- Capabilities that no single component was designed to produce. The most valuable Agent abilities (learning loops, self-healing routing) are emergent.
- **Context Window Management** -- Every Agent that runs an LLM loop must solve this. Multi-level compression strategies are universal. See our [Context Pressure Cascade](../emergent-behaviors/context-cascade.md) guide.
- **Speculative Execution** -- Borrowed from CPU design. Execute work before you know if it will be needed. Roll back if not. Claude Code applies this to tool execution during permission prompts.
- **Landlock / seccomp** -- Linux kernel security modules used by NemoClaw for filesystem and syscall isolation. The gold standard for Agent sandboxing.

---

## Key Files to Study in Each Project

If you want to go straight to the source, these are the most important files.

| Project | File | What You Will Learn |
|---------|------|---------------------|
| hermes | `core/run_agent.py` | The agentic loop in its simplest form |
| hermes | `tools/skill_manage.py` | How skill CRUD powers the learning loop |
| hermes | `memory/memory_manager.py` | Provider ABC pattern for pluggable memory backends |
| openclaw | `src/core/runEmbeddedPiAgent.ts` | The cleanest Agent Core separation we found |
| openclaw | `src/routing/` | 9-level binding priority for multi-Agent routing |
| openclaw | `src/model-fallback.ts` | Self-healing model routing with cooldown probing |
| NemoClaw | `sandbox/` | Landlock + seccomp security configuration |
| Claude Code | `query.ts` | Async generator agentic loop + speculative execution |
| Claude Code | `services/compact/` | Three-strategy context compression |

---

## Suggested Reading Order

If you are starting from scratch:

1. Read Anthropic's "Building Effective Agents" guide (30 min)
2. Take the Claude 101 + Claude Code in Action courses (2 hours)
3. Read our [four-projects-overview](../comparisons/four-projects-overview.md) for the landscape (10 min)
4. Pick a module from the [modules](../modules/) directory and read the corresponding source project
5. Study the key files listed above for your module of interest
6. Build something small. Then read more.
