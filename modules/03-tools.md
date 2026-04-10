# Module 3: Tools
> Your Agent's hands and feet — registration, discovery, dispatch, parallel execution, and security approval.

## What Is It

Tools are how an Agent acts on the world. The LLM thinks; tools do. When the model decides "I need to read this file" or "run this shell command," the Tools module finds the right handler, checks permissions, executes it, and returns the result.

In the Hub-and-Spoke architecture, Tools is a **peer service** to Agent Core — not below it, not inside it. Core says "execute tool X with args Y." Tools handles everything from there: looking up the handler, deciding if it's safe, running it (possibly in parallel with other tools), and handing back structured results. Core never reaches into tool internals.

Tools also owns the **Skill Store** — the directory of MD/YAML skill files that shape Agent behavior. Skills are loaded by Agent Core at prompt-build time, but their lifecycle (create, read, update, delete) is managed through tool calls. This is how the Learning Loop emergent behavior works: Tools writes the file, Core reads it next turn.

## What Problem Does It Solve

**Discovery**: The LLM needs to know what tools exist. Without a registry, you're hardcoding tool descriptions into the system prompt. hermes has 50+ tools, openclaw has 100+, Claude Code has 60+. Manual prompt management doesn't scale past 10.

**Dispatch**: The model returns a tool name and arguments as JSON. Something has to look up the handler, validate the args, and route the call. Get this wrong and your Agent either crashes on unknown tools or silently ignores valid ones.

**Parallel execution**: A single LLM response often requests 3-8 tool calls. Running them sequentially means the user stares at a spinner for 30 seconds instead of 5. But naive parallelism causes race conditions — two tools editing the same file will corrupt it.

**Security approval**: Your Agent can run `rm -rf /`. Somebody needs to stop it. Approval logic lives inside the tool execution flow, not as a separate middleware layer — because the decision depends on tool-specific context (which file? which command? what permissions?).

**MCP outbound**: Your Agent wants more tools than you built. MCP (Model Context Protocol) lets it connect to external servers for additional capabilities. This is outbound MCP — your Agent calling out. Inbound MCP (others calling your Agent) belongs in the Interface layer.

## How the 4 Projects Do It

### hermes-agent

**Registration pattern**: Import-time self-registration. Each tool file calls `registry.register()` at module load. When Python imports the file, the tool is registered. No central manifest. No configuration step.

```python
# hermes: tools/shell_tool.py (simplified)
from core.registry import registry

@registry.register(
    name="shell_exec",
    description="Execute a shell command",
    parameters={...}
)
async def shell_exec(command: str, workdir: str = "."):
    # implementation
```

**Tool count**: 50+ built-in tools across shell, file, web, browser, code execution, vision, delegation, and skill management categories.

**Parallel execution**: Path independence detection. Before running tools in parallel, hermes checks if any two tools operate on the same file path. Independent tools run concurrently; conflicting ones run sequentially. Max 8 threads.

**Security**: `approval.py` uses regex patterns to detect dangerous commands (`rm -rf`, `sudo`, `curl | bash`) and scans for prompt injection in tool arguments. Simple but effective for single-user scenarios.

**Skill Store**: `~/.hermes/skills/*.md` files with YAML frontmatter. The `skill_manage` tool handles CRUD. `prompt_builder` scans the directory each turn and injects relevant skills into the system prompt.

**Pros**: Dead simple. A new developer can add a tool in 5 minutes. Self-registration means zero configuration overhead.

**Cons**: No third-party extensibility. You want to add a tool? Fork the repo and edit the source. Also, the registry is global state — testing requires careful cleanup.

### openclaw

**Registration pattern**: Plugin SDK with `api.registerTool()`. Tools are registered by plugins during their `onActivate()` lifecycle hook. The SDK provides type-safe interfaces and validation.

```typescript
// openclaw: extensions/some-plugin/index.ts (simplified)
export function onActivate(api: PluginAPI) {
  api.registerTool({
    name: "web_search",
    description: "Search the web",
    parameters: z.object({ query: z.string() }),
    isConcurrencySafe: true,
    execute: async (params) => { /* ... */ }
  });
}
```

**Tool count**: 100+ tools across core and community plugins. The Plugin SDK exports 50+ subpath entries for third-party developers.

**Parallel execution**: Explicit `isConcurrencySafe` flag per tool. If a tool declares itself safe for concurrent execution, the runtime will parallelize it with other safe tools. No path analysis — the tool author decides.

**Security**: Multi-layer. DM pairing (the user who installed the Agent must confirm actions via direct message). Exec approval flow for dangerous commands. Rate limiting per tool category.

**MCP outbound**: First-class support. Plugins can register MCP server connections, and their tools appear alongside native tools in the registry. The LLM doesn't know the difference.

**Skill Store**: Workspace-level skill files. Unlike hermes, skills are not auto-created by the Agent — they're manually authored and placed in the workspace.

**Pros**: Maximum extensibility. Third-party developers can publish tool plugins without touching the core. The `isConcurrencySafe` flag is elegant — it puts the safety decision where the knowledge is.

**Cons**: Complexity. The Plugin SDK is substantial infrastructure. The registration lifecycle (onActivate/onDeactivate) adds cognitive overhead. For a solo developer building a personal Agent, this is overengineered.

### NemoClaw

NemoClaw is not an Agent — it's a security sandbox runtime. It doesn't have its own tool system. It wraps OpenClaw's tools with security policies: Landlock file isolation, per-binary network restrictions, seccomp syscall filtering. Tools that try to access unauthorized paths or network endpoints get blocked at the OS level, not at the application level.

Relevant to the Tools module: NemoClaw's approach to tool security is orthogonal to application-level approval. It proves that **two layers of tool security can coexist** — application checks ("is this command dangerous?") and OS-level enforcement ("can this process access this path?").

### Claude Code

**Registration pattern**: Static imports with feature-flag dead code elimination (DCE). All tools are imported in a central file. Feature flags (evaluated at build time) determine which tools are available. The bundler strips unreachable code.

```typescript
// Claude Code: tools/index.ts (simplified)
import { ShellTool } from "./shell"
import { FileTool } from "./file"
import { BrowserTool } from "./browser"  // behind feature flag

const ALL_TOOLS: Tool<any, any, any>[] = [
  ShellTool,
  FileTool,
  ...(FEATURE_FLAGS.browser ? [BrowserTool] : []),
]
```

**Tool count**: 60+ tools. Each tool implements a generic `Tool<Input, Output, Permissions>` interface — the type system enforces that every tool declares its input schema, output shape, and required permissions.

**Parallel execution**: Batch splitting. When the LLM returns multiple tool calls, Claude Code splits them into batches of up to 10 and runs each batch concurrently. No path analysis, no safety flags — just a hard concurrency cap.

**Security**: Permission modes (ask/auto) configurable per tool category. Enterprise policy limits from MDM (Mobile Device Management). `Tool.checkPermissions()` is called before execution. For speculative execution, tools run optimistically while the permission prompt is shown — if denied, results are discarded at the CompletionBoundary.

**Skill Store**: `~/.claude/skills/` directory. Skills are loaded via the `SKILL.md` convention. The `/remember` command and `extractMemories` background agent can save facts, but don't create structured skill files the way hermes does.

**Pros**: Zero runtime overhead for disabled tools — they don't exist in the bundle. The `Tool<I,O,P>` generic enforces type safety across the entire tool surface. Permission modes give users fine-grained control.

**Cons**: Adding a tool requires rebuilding. No hot-loading, no plugin system. Feature flags are binary — a tool is either in or out, no per-user customization at runtime.

## Comparison Table

| Dimension | hermes | openclaw | NemoClaw | Claude Code |
|-----------|--------|----------|----------|-------------|
| **Registration** | `registry.register()` import-time | `api.registerTool()` Plugin SDK | N/A (not an Agent) | Static import + feature flag DCE |
| **Tool count** | 50+ | 100+ | Management CLI only | 60+ |
| **Third-party extensible** | No (fork required) | Yes (Plugin SDK) | N/A | No (rebuild required) |
| **Parallel strategy** | Path independence, max 8 | `isConcurrencySafe` flag | N/A | Batch split, max 10 |
| **Security model** | Regex pattern + injection scan | DM pairing + exec approval + rate limit | OS-level (Landlock/seccomp) | Permission modes + enterprise policy |
| **MCP outbound** | Basic | First-class plugin integration | N/A | Full support, tools appear native |
| **Skill Store** | `~/.hermes/skills/` auto-created | Workspace skills, manual | N/A | `~/.claude/skills/` semi-auto |
| **Type safety** | Python type hints | Zod schemas | N/A | `Tool<I,O,P>` generics |
| **Runtime overhead** | Minimal (import-time) | Plugin lifecycle cost | N/A | Zero (build-time DCE) |

## Best Practices

**If you only do one thing**: Start with hermes-style self-registration. Define a `Tool` interface, have each tool file register itself on import, store them in a dictionary keyed by name. You'll have a working tool system in under 100 lines. Upgrade to a Plugin SDK only if third parties need to extend your Agent.

**Five rules from the 4 projects**:

1. **Tools declare themselves.** The tool knows its name, description, parameter schema, and safety properties. Don't put this in a central config file — put it next to the implementation. All 4 projects do this.

2. **Validate inputs before execution.** hermes uses parameter schemas. openclaw uses Zod. Claude Code uses TypeScript generics. Pick your poison, but validate. An LLM will send malformed arguments roughly 2-5% of the time.

3. **Return structured results.** Not strings. A tool result should include: success/failure status, the actual output, any side effects (files created, processes started), and resource usage. Claude Code's `ToolResult` is the gold standard here.

4. **Security is per-tool, not per-layer.** The decision "is this safe?" depends on what the tool does and what arguments it received. A blanket "all shell commands need approval" is too coarse. A blanket "auto-approve everything" is too dangerous. Claude Code's per-category permission modes hit the sweet spot.

5. **Skill files are tools' downstream output, Core's upstream input.** Tools write them (via skill CRUD operations). Core reads them (via prompt builder). This separation is what enables the Learning Loop.

## Step-by-Step: How to Build It

### Step 1: Define the Tool interface

Every tool needs: a name (unique string), a description (for the LLM), a parameter schema (for validation), and an execute function.

```typescript
// contracts/tool.ts
interface Tool {
  name: string;
  description: string;
  parameters: JSONSchema;          // or Zod schema
  isConcurrencySafe?: boolean;     // default false
  requiresApproval?: boolean;      // default false
  execute(params: unknown): Promise<ToolResult>;
}

interface ToolResult {
  success: boolean;
  output: unknown;
  sideEffects?: string[];          // "created file: /tmp/foo.txt"
}
```

**Decision point**: Zod vs JSON Schema? If your codebase is TypeScript, use Zod — it gives you runtime validation and type inference in one shot. If you're in Python, use Pydantic models or plain JSON Schema.

### Step 2: Build the registry

A dictionary. That's it. Don't overcomplicate this.

```typescript
// tools/registry.ts
const registry = new Map<string, Tool>();

function register(tool: Tool): void {
  if (registry.has(tool.name)) {
    throw new Error(`Tool "${tool.name}" already registered`);
  }
  registry.set(tool.name, tool);
}

function getTool(name: string): Tool | undefined {
  return registry.get(name);
}

function getAllTools(): Tool[] {
  return Array.from(registry.values());
}
```

### Step 3: Create your first tool

```typescript
// tools/builtins/shell.ts
import { register } from "../registry";

register({
  name: "shell_exec",
  description: "Execute a shell command and return stdout/stderr",
  parameters: {
    type: "object",
    properties: {
      command: { type: "string", description: "The command to run" },
      workdir: { type: "string", description: "Working directory" },
      timeout: { type: "number", description: "Timeout in ms", default: 30000 }
    },
    required: ["command"]
  },
  isConcurrencySafe: false,  // shell commands can conflict
  requiresApproval: true,     // always ask for shell
  execute: async (params) => {
    const { command, workdir, timeout } = params as ShellParams;
    // ... actual execution
  }
});
```

### Step 4: Build the dispatcher

The dispatcher sits between Agent Core and tools. It receives tool calls from the LLM response, validates, checks approval, and executes.

```typescript
// tools/dispatcher.ts
async function dispatch(toolCalls: ToolCall[]): Promise<ToolResult[]> {
  // 1. Look up all tools
  const resolved = toolCalls.map(call => {
    const tool = getTool(call.name);
    if (!tool) return { call, tool: null, error: `Unknown tool: ${call.name}` };
    return { call, tool, error: null };
  });

  // 2. Check approval for tools that need it
  for (const item of resolved) {
    if (item.tool?.requiresApproval) {
      const approved = await requestApproval(item.call);  // ← your UI hook
      if (!approved) {
        item.error = "User denied";
      }
    }
  }

  // 3. Split into concurrent-safe and sequential
  const safe = resolved.filter(r => r.tool?.isConcurrencySafe && !r.error);
  const sequential = resolved.filter(r => !r.tool?.isConcurrencySafe && !r.error);

  // 4. Run safe tools in parallel, sequential ones in order
  const results = await Promise.all([
    Promise.all(safe.map(r => r.tool!.execute(r.call.arguments))),
    runSequential(sequential)
  ]);

  return flatten(results);
}
```

**Decision point**: How many parallel tools? hermes caps at 8, Claude Code at 10. Start with 5 and increase based on your workload. The bottleneck is usually I/O (network, disk), not CPU.

### Step 5: Add MCP outbound support

MCP outbound means your Agent connects to external MCP servers to discover more tools. These external tools get registered in the same registry as built-in tools.

```typescript
// tools/mcp-client.ts
async function connectMCPServer(config: MCPServerConfig): Promise<void> {
  const client = new MCPClient(config.uri);
  const remoteTools = await client.listTools();

  for (const remote of remoteTools) {
    register({
      name: `mcp_${config.name}_${remote.name}`,  // namespace to avoid collision
      description: remote.description,
      parameters: remote.inputSchema,
      isConcurrencySafe: true,  // MCP calls are usually independent
      execute: async (params) => {
        return await client.callTool(remote.name, params);
      }
    });
  }
}
```

**Decision point**: Namespace MCP tools or not? hermes doesn't (collision risk). Claude Code prefixes with server name (verbose but safe). Namespace them. The LLM can handle longer names.

### Step 6: Set up the Skill Store

```
~/.your-agent/skills/
├── code-review.md         # YAML frontmatter + markdown body
├── debugging-strategy.md
└── api-design.md
```

Each skill file:

```markdown
---
name: code-review
version: 1
created: 2026-04-01
tags: [development, review]
---

# Code Review Methodology

When reviewing code, follow these steps:
1. Check for security issues first...
2. ...
```

The skill CRUD tool manages this directory. The prompt builder in Agent Core scans it and injects relevant skills into the system prompt.

## Common Pitfalls

**1. The god-class registry.** hermes's `AIAgent` class (9431 lines) mixes tool registration, dispatch, execution, and approval into a single object. By the time you have 30 tools, this class is unmaintainable. Keep the registry, dispatcher, and individual tools as separate concerns from the start.

**2. Trusting LLM-generated arguments.** The model will send `{"file": "/etc/passwd"}` to your file-read tool. It will send `{"command": "sudo rm -rf /"}` to your shell tool. Validate everything. hermes's regex-based detection catches the obvious cases but misses encoding tricks. Claude Code's per-tool permission system is more robust.

**3. Ignoring tool result size.** A `shell_exec` call that runs `find /` will return megabytes. This blows up your context window. Claude Code truncates tool results to configurable limits. hermes's Context Manager prunes old tool results as a first compression step. Build result size limits into ToolResult from day one.

**4. Over-parallelizing.** Running 8 file-edit operations concurrently will corrupt files. openclaw's `isConcurrencySafe` flag is the right idea — each tool declares its own safety. If you don't want per-tool flags, at least distinguish between read-only tools (safe to parallelize) and write tools (serialize by default).

**5. Hardcoding tool descriptions.** When you have 60 tools, the combined descriptions can eat 3,000+ tokens of your system prompt. Claude Code generates tool descriptions dynamically based on context. hermes includes all tools always. Start by including everything; optimize when you hit context limits.

**6. Building a Plugin SDK too early.** openclaw's Plugin SDK (50+ subpath exports, lifecycle hooks, typed APIs) is impressive engineering. It also took months to build and stabilize. If you don't have third-party developers, you don't need it. hermes's self-registration pattern serves solo and small-team projects just fine.

## Cross-Module Contracts

### What Tools expects from other modules

| Source Module | What Tools Needs | Contract |
|--------------|-----------------|----------|
| **Agent Core** | Tool calls extracted from LLM response | `ToolCall { name: string, arguments: object, id: string }` |
| **Agent Core** | Approval decision (approve/deny) for tools requiring it | `requestApproval(call: ToolCall) → Promise<boolean>` |
| **Model** | Nothing directly — Tools and Model are peer services | (no direct dependency) |

### What Tools provides to other modules

| Target Module | What Tools Provides | Contract |
|--------------|-------------------|----------|
| **Agent Core** | Tool execution results | `ToolResult { success: boolean, output: unknown, sideEffects?: string[] }` |
| **Agent Core** | List of available tools (for system prompt) | `getAllTools() → Tool[]` with name, description, parameters |
| **Agent Core** | Skill files for prompt injection | Files in `~/.agent/skills/` with YAML frontmatter |
| **Context Manager** | Tool result content (for potential truncation) | ToolResult.output is the truncation target |
| **Memory** | Nothing directly — Memory and Tools are peer services | (no direct dependency) |

### Direction confusion to avoid

- **MCP outbound** (Agent calling external MCP servers for tools) = Tools module.
- **MCP inbound** (external systems calling your Agent) = Interface module.
- **Security approval** (should this tool run?) = Inside the tool dispatch flow, not a separate middleware.
- **Skill Store** lifecycle = Tools writes it. Agent Core reads it. Memory doesn't touch it.
