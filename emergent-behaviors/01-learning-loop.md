# Emergent Behavior 1: Learning Loop (Self-Evolution)

> One-line: the Agent creates reusable skills from its own experience, loads them in future sessions, and patches them when they produce wrong results -- without any SelfEvolutionEngine class.

## What Is It

The learning loop is a self-evolution cycle. After repeated tool use, the Agent distills what worked into a skill file. Next session, that skill file gets injected into the system prompt. The LLM reads it, changes its behavior, and -- if the skill is wrong -- patches it through the same tool that created it.

Why it is "emergent": no single component was designed to do self-evolution. `SKILLS_GUIDANCE` is a prompt paragraph. `skill_manage` is a file CRUD tool. `prompt_builder` is a directory scanner. Each does one trivial thing. Combined, the system learns from experience.

hermes-agent is the only project that implements the full loop. Closeness score: 10/10. Claude Code scores 3/10 -- it extracts facts via `/remember`, but never creates reusable methodology.

## How It Emerges

```
Agent Core
  │
  │  injects SKILLS_GUIDANCE instruction into system prompt
  │  (tells LLM: "after 5+ tool calls, consider saving a skill")
  │
  ▼
LLM (Model)
  │
  │  decides: "I should save what I learned"
  │  returns tool_call: skill_manage(action=create, ...)
  │
  ▼
Tools ── skill_manage(create) ──→ Skill Store
  │                                  │
  │  writes skill file               │  ~/.hermes/skills/my-skill.md
  │  (frontmatter + markdown body)   │  (persistent file on disk)
  │                                  │
  │         ┌────────────────────────┘
  │         │
  │         ▼  next session starts
  │
  │     prompt_builder scans skills directory
  │     injects matching skills into system prompt
  │         │
  │         ▼
  │     LLM reads skill → behavior changes
  │         │
  │         ▼
  │     LLM discovers skill is wrong or outdated
  │         │
  │         ▼
  │     LLM calls skill_manage(action=patch)
  │         │
  └─────────┘  loop complete
```

No orchestrator manages this cycle. Each arrow is a different component doing its own job. The loop exists only because their contracts align.

## Which Components Participate

| Component | Role in Learning Loop |
|-----------|----------------------|
| **Agent Core** | Injects `SKILLS_GUIDANCE` into system prompt. Calls `prompt_builder` to load skills at session start. Owns the iteration counter that triggers learning guidance. |
| **Model (LLM)** | Decides *when* to save a skill and *what* to save. Decides when to patch. This is not deterministic -- the LLM is the judgment layer. |
| **Tools** | Provides `skill_manage` tool with actions: `create`, `patch`, `delete`, `list`, `search`. Pure file CRUD. No learning logic. |
| **Skill Store** | `~/.hermes/skills/*.md` files. Flat directory. Each file has YAML frontmatter (name, tags, version) and a markdown body (the skill content). |
| **Memory** | Auxiliary. LLM can recall past context via `memory_search` to decide whether a skill already exists or needs updating. Not required for the loop to function. |

## Case Study: hermes-agent

hermes is where this behavior was discovered. Here is the exact mechanism.

### Trigger: SKILLS_GUIDANCE injection

In `hermes/core/prompt_builder.py`, the system prompt is assembled from fragments. One fragment is `SKILLS_GUIDANCE` -- a block of text injected when the iteration count exceeds a threshold (typically 5 tool calls in a conversation).

The guidance says, roughly: "You have been working on this task for a while. If you learned something reusable, save it as a skill using `skill_manage`."

This is not a hardcoded rule. It is a suggestion in natural language. The LLM may ignore it.

### Creation: skill_manage tool

In `hermes/tools/skill_manage.py`, the `skill_manage` tool accepts:

- `action`: create | patch | delete | list | search
- `name`: skill identifier
- `tags`: list of strings for matching
- `content`: markdown body (the actual skill)

On `create`, it writes a file to `~/.hermes/skills/{name}.md` with YAML frontmatter:

```yaml
---
name: efficient-file-search
tags: [filesystem, search, performance]
version: 1
created: 2026-04-08
---
When searching large codebases, use ripgrep with --type flag
instead of recursive grep. Limit depth with --max-depth 3
for initial scans. Only go deeper on confirmed hits.
```

### Loading: prompt_builder scans directory

On the next session start, `prompt_builder` scans `~/.hermes/skills/`. For each `.md` file, it reads the frontmatter, checks tags against the current conversation context, and injects matching skills into the system prompt.

The injection happens at the same level as identity instructions. The LLM sees the skill as a first-class instruction, not as a conversation message.

### Patching: skill_manage(patch)

When the LLM follows a skill and gets a bad result, it can call `skill_manage(action=patch, name="efficient-file-search", content="...")`. The tool overwrites the file body, increments the version number in frontmatter, and the skill is updated.

On hermes, `prompt_builder` rescans every turn. The patched skill takes effect in the same session.

### Why it works in hermes but nowhere else

Three conditions are uniquely met:

1. **Core-level trigger.** The `SKILLS_GUIDANCE` injection is done by Agent Core, not by a system prompt the user writes. It fires reliably.
2. **Immediate effect.** `prompt_builder` rescans every turn. A created or patched skill changes LLM behavior within the same session.
3. **Tool is mandatory.** `skill_manage` is registered as a built-in tool. Agent Core intercepts the call. The LLM cannot "forget" to execute it once it decides to call it.

## How to Make It Emerge in Your Agent

### Step 1: Create the Skill Store

A directory. Nothing more.

```
~/.your-agent/skills/
```

Each file: YAML frontmatter + markdown body. Frontmatter schema:

```yaml
---
name: string        # unique identifier, kebab-case
tags: [string]      # for matching against conversation context
version: integer    # incremented on each patch
created: date       # ISO 8601
updated: date       # ISO 8601, optional
---
```

### Step 2: Build the skill_manage tool

A file CRUD tool with 5 actions:

| Action | Input | Behavior |
|--------|-------|----------|
| `create` | name, tags, content | Write new file. Fail if name exists. Set version=1. |
| `patch` | name, content | Overwrite body. Increment version. Update `updated` field. |
| `delete` | name | Remove file. |
| `list` | (none) | Return names + tags of all skills. |
| `search` | query | Match query against names, tags, and content. Return top 3. |

### Step 3: Wire prompt_builder to scan skills

At session start (and optionally every turn), scan the skills directory. For each file:

1. Parse frontmatter.
2. Match tags against current conversation keywords.
3. Inject matched skills into the system prompt, after identity but before conversation history.

If you inject all skills unconditionally, you will burn tokens. Tag-based matching keeps it cheap.

### Step 4: Add the learning guidance prompt

Inject a paragraph into the system prompt when the conversation exceeds N tool calls (hermes uses 5). Example:

```
You have executed several tool calls in this conversation. If you
discovered a reusable pattern, technique, or procedure, save it
as a skill using skill_manage(create). Keep skills concise and
actionable. One skill per technique.
```

This is the catalyst. Without it, the LLM will not spontaneously decide to save skills.

### Step 5: Close the loop with patch guidance

Add another paragraph that fires when a skill is loaded:

```
You are using skills from previous sessions. If any skill produces
incorrect or outdated results, update it using skill_manage(patch).
Do not delete skills unless they are fundamentally wrong.
```

This closes the feedback loop. Without it, bad skills persist forever.

## Cross-Module Contracts

These contracts must hold for the loop to function. Breaking any one of them breaks the entire behavior.

### Contract 1: Skill File Format

```
File: ~/.your-agent/skills/{name}.md
Format: YAML frontmatter (--- delimited) + markdown body
Required fields: name (string), tags (string[]), version (int), created (date)
Encoding: UTF-8
Newlines: LF
```

Both `skill_manage` (writer) and `prompt_builder` (reader) must agree on this format. If the writer outputs JSON and the reader expects YAML, the loop breaks silently.

### Contract 2: Storage Path Convention

```
Base path: configurable, default ~/.your-agent/skills/
File naming: {name}.md where name matches frontmatter name field
No subdirectories. Flat scan.
```

`skill_manage` writes here. `prompt_builder` scans here. If they disagree on the path, skills are created but never loaded.

### Contract 3: Prompt Injection Method

```
Skills are injected into the system prompt, not as user messages.
Position: after agent identity, before conversation history.
Format: each skill wrapped in a labeled block (e.g., <skill name="...">...</skill>).
```

If skills are injected as user messages, the LLM may treat them as conversation context rather than instructions. The skill loses authority.

### Contract 4: Skill CRUD Parameters

```
skill_manage tool interface:
  action: enum("create", "patch", "delete", "list", "search")
  name: string (required for create, patch, delete)
  tags: string[] (required for create, optional for patch)
  content: string (required for create and patch)
  query: string (required for search)
```

The LLM generates these parameters. If the tool schema changes but the system prompt still describes the old schema, the LLM will produce invalid calls.

## How to Test It (Loop Test)

This test verifies the complete cycle: create, load, use, patch, reload.

```
Test: learning-loop-full-cycle
Precondition: skills directory is empty.

Step 1 — Trigger creation
  Simulate a conversation with 5+ tool calls (e.g., 5 file reads).
  Verify: SKILLS_GUIDANCE prompt was injected (check system prompt).
  Send a message: "That search pattern was useful. Save it."
  Verify: LLM returns tool_call for skill_manage(create).
  Execute the tool call.
  Verify: file exists in skills directory.
  Verify: frontmatter has version=1, valid tags, valid name.

Step 2 — Verify loading
  Start a new session.
  Verify: prompt_builder scanned skills directory.
  Verify: system prompt contains the skill content.
  (Check by inspecting the assembled prompt before sending to LLM.)

Step 3 — Verify behavior change
  In the new session, present a task that matches the skill's domain.
  Verify: LLM response references or follows the skill's guidance.
  (This is non-deterministic. Use a rubric: does the response match
   the skill's technique? Yes/No.)

Step 4 — Trigger patch
  Present a scenario where the skill gives wrong advice.
  Send: "That technique doesn't work for binary files. Update the skill."
  Verify: LLM returns tool_call for skill_manage(patch).
  Execute the tool call.
  Verify: file version incremented to 2.
  Verify: file content reflects the correction.

Step 5 — Verify patched version loads
  Start another new session.
  Verify: system prompt contains the patched skill content (version 2).
  Verify: old content is gone.

Pass criteria: all 5 steps pass. Any failure means the loop is broken.
```

Runtime: ~3 minutes with a local LLM, ~30 seconds with mocked LLM responses.

## Variations Across Projects

| Project | Learning Mechanism | Closeness to Full Loop | Score |
|---------|-------------------|----------------------|-------|
| **hermes-agent** | `SKILLS_GUIDANCE` prompt injection + `skill_manage` built-in tool + `prompt_builder` directory scan + same-session reload + `skill_manage(patch)` for corrections. | Full loop. Every link is wired. | **10/10** |
| **openclaw** | Workspace skills exist in `extensions/skills/`. Skills can be loaded into conversations. But no automatic creation mechanism -- a human must write the skill file. No patch flow. | Storage and loading exist. Creation and patching are manual. | **4/10** |
| **Claude Code** | `/remember` command and `auto-extract` background agent extract facts into `~/.claude/memory/*.md`. Facts are injected on next session. But these are raw facts ("user prefers TypeScript"), not reusable methodology ("when doing X, use technique Y"). No patch mechanism. | Extracts facts, not skills. No creation of reusable procedures. No feedback correction. | **3/10** |
| **NemoClaw** | Not applicable. NemoClaw is a security runtime, not an agent. It has no learning mechanism. | N/A | **0/10** |

### What makes hermes unique

Two things separate hermes from the rest:

1. **Methodology, not facts.** hermes skills contain procedures ("do X, then Y, then Z"). Claude Code memories contain observations ("user likes Z"). Procedures change behavior. Observations add context.

2. **Self-correction.** hermes skills can be patched by the same agent that created them. Claude Code memories cannot be corrected -- the agent can only add new memories, creating potential contradictions with old ones.

### If you want the loop but use Claude Code as your base

You need to add three things Claude Code does not have:

1. A `skill_manage` MCP tool (create, patch, delete, list, search).
2. A skill loading step that injects matched skills into the system prompt (via CLAUDE.md or a custom skill file that the agent reads on startup).
3. A learning guidance prompt that triggers after N tool calls.

The weakest link will be step 2. Claude Code loads CLAUDE.md and skill files at session start, not every turn. A skill created mid-session will not take effect until the next session. hermes reloads every turn.

Expected closeness score with these additions: 6/10. The gap is reload timing and the fact that MCP tool calls are suggestions, not guarantees.
