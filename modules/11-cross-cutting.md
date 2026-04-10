# Module 11: Cross-Cutting Concerns
> Three concerns that genuinely span multiple modules — and a warning about calling things "cross-cutting" when they are not.

## What Is It

Cross-cutting concerns are things that cannot be cleanly assigned to a single module. They touch multiple layers simultaneously.

In practice, across the four projects, only three things qualify:

1. **Observability** — logging, metrics, tracing. Every module emits data. Something has to collect, structure, and route it.
2. **Plugin framework** — a system that lets third-party code register tools, channels, context engines, and model providers all at once. Not a tool registry (that is Module 3). A framework that spans multiple registries simultaneously.
3. **Hot config reload** — changing runtime configuration without restarting the Agent. Requires coordination across every module that reads config.

Most things people call "cross-cutting" are not. Command approval is just the Tools layer. Memory injection is just Core + Memory via Context Manager. Permission checks are just the Security Envelope. These feel cross-cutting because they affect user experience across features, but architecturally they live in one module with a clear interface.

## What Problem Does It Solve

**Observability solves "what just happened."** Your Agent called a tool, the tool failed, the model retried with different arguments, the retry succeeded but took 14 seconds. Without structured logging, you know the final result but not the path. Without metrics, you do not know this happens 30% of the time. Without tracing, you cannot connect the tool failure to the upstream model call that produced malformed arguments.

**Plugin framework solves "how do third parties extend this."** If you build a platform that others deploy and customize, they need to add capabilities without forking your source. A plugin system that only registers tools is not cross-cutting — it is just the Tools module with an extension point. A plugin system becomes genuinely cross-cutting when a single plugin can register tools AND channels AND context engines AND model providers in one activation call.

**Hot config reload solves "how do I change settings in production."** Your Agent serves 50 concurrent users. You need to switch the default model from GPT-4 to Claude. Without hot reload, you restart the process and drop all 50 sessions. With hot reload, you update the config file, the Agent detects the change, and new requests use the new model while existing sessions complete on the old one.

## How the 4 Projects Do It

### Observability

**hermes-agent**: Python `logging` module. Standard log levels (DEBUG, INFO, WARNING, ERROR). No structured format — plain text log lines. No metrics collection. No distributed tracing. Good enough for a single-user Agent running locally. Breaks down when you need to debug production issues across multiple sessions.

**openclaw**: Sentry integration for error tracking and performance monitoring. Structured log events via the plugin system. The most production-oriented observability of the four projects, but still focused on error cases rather than comprehensive telemetry.

**NemoClaw**: Gateway health state classification (healthy, degraded, dead) in `gateway-state.ts`. SSH + curl probes for liveness detection. This is observability in service of the infrastructure layer — focused on "is the runtime alive" rather than "what is the Agent doing." The only project with automated recovery triggered by observability signals.

**Claude Code**: Internal telemetry for Anthropic's usage analytics. Not exposed to users or operators. From an open-source perspective, effectively no observability.

### Plugin Framework

**hermes-agent**: No plugin framework. Tools are registered via `registry.register()` at import time. Want to add a tool? Edit the source code. This is a tool registry, not a plugin system. It is also perfectly adequate for any Agent that does not need third-party extensibility.

**openclaw**: The Plugin SDK with 50+ subpath exports. This is the only genuinely cross-cutting plugin system across the four projects. A single plugin's `onActivate()` method can call:
- `api.registerTool()` — add tools
- `api.registerChannel()` — add communication channels (Discord, Slack, etc.)
- `api.registerContextEngine()` — replace the context compression strategy
- `api.registerModelProvider()` — add model backends

One plugin, four registration targets across four different architectural modules. That is what makes it cross-cutting. The SDK is published as a separate package so third-party developers can build against a stable API without depending on openclaw internals.

**NemoClaw**: CLI / Plugin / Blueprint — three loosely coupled parts. The "Plugin" here refers to NemoClaw's own plugin system for extending the management CLI, not for extending the Agent runtime. It is not cross-cutting in the architectural sense.

**Claude Code**: No plugin framework. Tools are statically imported. Feature flags control which tools are available at build time via dead code elimination. Extension happens by modifying the source, not by loading plugins.

### Hot Config Reload

**hermes-agent**: No hot reload. Config is read at startup. Changes require restart.

**openclaw**: Config drift detection. openclaw periodically compares the running configuration against the config file on disk. When a difference is detected, affected modules are notified to reload their state. This is the only implementation of hot config across the four projects.

**NemoClaw**: No hot reload for the Agent (which is an openclaw instance inside a sandbox). NemoClaw's own Blueprint can re-apply configuration changes, but this involves redeploying the sandbox — not a live reload.

**Claude Code**: No hot reload. Single-user CLI tool. Restart is trivial.

## Comparison Table

| Dimension | hermes | openclaw | NemoClaw | Claude Code |
|-----------|--------|----------|----------|-------------|
| Logging | Python `logging`, plain text | Sentry + structured events | Health state classification | Internal telemetry only |
| Metrics | None | Sentry performance | Gateway probe results | None (user-facing) |
| Tracing | None | Sentry transactions | None | None |
| Plugin framework | None (tool registry only) | Plugin SDK, 50+ subpath exports, 4 registration targets | Management CLI plugins | None |
| Hot config reload | None | Config drift detection | Blueprint re-apply (not live) | None |
| Third-party extensibility | Fork the repo | Publish a plugin package | Not applicable | Fork the repo |

## Best Practices

**Add structured logging from day one.** This is the single cheapest investment with the highest return. Not `console.log("tool failed")`. Structured JSON with timestamp, level, module, operation, duration, and relevant IDs.

```json
{
  "ts": "2024-01-15T10:23:45Z",
  "level": "error",
  "module": "tools",
  "op": "execute",
  "tool": "shell_exec",
  "duration_ms": 14200,
  "error": "timeout",
  "session_id": "abc-123",
  "iteration": 7
}
```

You can grep this. You can pipe it to any log aggregator. You can build dashboards later. The cost of adding it on day one is near zero. The cost of retrofitting it after 6 months of `console.log` is substantial.

**Do not build a plugin system until you need third-party extensibility.** hermes has 50+ tools and no plugin system. It works. Claude Code has 60+ tools and no plugin system. It works. The trigger for building a plugin framework is a concrete third party who wants to extend your Agent without forking your code. If that third party does not exist, a simple tool registry (Module 3) is sufficient.

**Do not build hot config reload until you have a multi-user deployment.** If restarting your Agent takes 2 seconds and affects one user (you), hot reload is pure overhead. openclaw built config drift detection because it manages persistent Agent instances serving multiple users across multiple channels. The cost of a restart is dropped connections and lost session state for all connected users. That is the bar.

**If you do build a plugin system, make it cross-cutting or do not bother.** A plugin system that only registers tools is just an over-engineered tool registry. The value of openclaw's Plugin SDK is that one plugin activation touches tools, channels, context engines, and model providers simultaneously. If your "plugins" only add tools, save yourself the abstraction layer and use `registry.register()`.

## Step-by-Step: How to Build It

### Step 1: Add structured logging

Pick a structured logging library (`structlog` for Python, `pino` for Node.js). Every log line gets a JSON object with timestamp, level, module, operation, duration, and relevant IDs. Convention: `module.operation.phase` as the event name. Consistent across the codebase. Searchable.

### Step 2: Add basic metrics

Track four numbers:
1. **Tool execution count** — by tool name and success/failure
2. **Model call latency** — p50, p95, p99
3. **Iteration count per request** — how many loops before the Agent finishes
4. **Error rate** — by module and error type

Store these in memory and expose via a `/metrics` endpoint (Prometheus format) or log them periodically. Do not add a metrics database yet — that is infrastructure you do not need at this stage.

### Step 3: Add request tracing (when needed)

When your Agent handles concurrent requests or delegates to sub-agents, you need trace IDs to connect related log lines. Generate a UUID at request start, bind it to your logger, and every subsequent log line in that request includes the trace ID. OpenTelemetry is the standard for distributed tracing across services. But a simple trace ID propagated through your call chain covers 90% of debugging needs.

### Step 4 (optional): Add a plugin system

Only when a third party needs to extend your Agent.

Define a `PluginAPI` interface with registration methods for each module: `registerTool()`, `registerChannel()`, `registerContextEngine()`, `registerModelProvider()`. A `Plugin` has `name`, `version`, `onActivate(api)`, and `onDeactivate()`. The key design decision: the `PluginAPI` interface determines how cross-cutting your plugin system is. If it only has `registerTool()`, it is a tool registry with extra steps. openclaw's 4-target registration is the reference.

### Step 5 (optional): Add hot config reload

Only for multi-user deployments where restart has a user-visible cost.

The pattern: a `ConfigWatcher` hashes the config file every N seconds. When the hash changes, it loads the new config and notifies subscribers. Each module that reads config registers a callback. Model Service might switch providers. Tools might update rate limits. Context Manager might adjust compression thresholds. The interval (30 seconds is typical) balances responsiveness against file I/O overhead.

## Common Pitfalls

**1. Calling everything "cross-cutting."** Command approval is Tools. Memory injection is Context Manager. Auth is Security Envelope. The word "cross-cutting" gets used to justify putting code in a shared location when it actually belongs in a specific module. Ask: "Does this concern require simultaneous changes in 3+ modules to implement?" If no, it is not cross-cutting.

**2. Unstructured logging.** `console.log("Error in tool: " + toolName)` is useless at scale. You cannot grep for all tool errors. You cannot calculate error rates. You cannot build alerts. Structured logging costs 5 minutes more per log statement and saves hours of debugging later.

**3. Building the plugin system before the tool registry is stable.** openclaw's Plugin SDK exports 50+ subpaths. That means 50+ interfaces that third-party code depends on. If those interfaces change, every plugin breaks. Build and stabilize your tool registry, context engine, and model service interfaces first. The plugin system is a contract layer on top of already-stable abstractions.

**4. Hot reload without versioning.** You change the model from GPT-4 to Claude mid-session. Existing conversations started with GPT-4's system prompt format. Claude expects a different format. The session breaks. Hot reload needs per-session config pinning: new sessions get the new config, existing sessions keep the old one until they end.

**5. Observability as an afterthought.** "We will add logging later" means you will debug production issues with `print()` statements for 6 months. Structured logging is the one thing in this module that has zero reasons to postpone.

## Cross-Module Contracts

### What Cross-Cutting Concerns expect from other modules

| Module | Contract | Why |
|--------|----------|-----|
| **All modules** | Emit structured log events | Observability collects and routes them |
| **All modules** | Expose a config reload callback (if hot reload is implemented) | Config watcher notifies subscribers on change |
| **Tools, Model, Context, Channels** | Provide registration interfaces | Plugin framework wraps them into a unified PluginAPI |

### What Cross-Cutting Concerns provide to other modules

| Consumer | What it provides | Format |
|----------|-----------------|--------|
| **Operators** | Aggregated logs, metrics, traces | JSON log stream, Prometheus metrics, OpenTelemetry spans |
| **Infrastructure** | Health signals derived from observability | Alert triggers based on error rates or latency thresholds |
| **Third-party developers** | Stable extension API | PluginAPI interface with versioned registration methods |
| **All modules** | Updated configuration values | Config reload notifications with new config object |

### Invariants

- Observability never blocks the Agent's main loop. Logging is fire-and-forget. Metrics are accumulated in memory and flushed asynchronously.
- The plugin system does not create new interfaces. It wraps existing registration methods (from Tools, Model, Context, Channels) into a single activation point. If a module does not have a registration interface, the plugin system cannot extend it.
- Hot config reload is eventual, not instant. Modules process config changes on their next natural boundary (next request, next health check cycle), not mid-operation.
- No module depends on the plugin system. Plugins depend on modules. If you remove the plugin system, every module continues to work. This is the test for whether your plugin abstraction is correctly layered.
