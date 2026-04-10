# Module 10: Runtime + Infrastructure
> WHERE your Agent runs (Runtime) and WHO sets it up (Infrastructure). Two layers that look like one until something breaks.

## What Is It

Runtime is the environment your Agent executes inside. It exists for the entire lifetime of a session. A local Node.js process is a runtime. A Docker container is a runtime. An OpenShell sandbox with Landlock filesystem isolation, seccomp syscall filtering, and an L7 network proxy is a runtime.

Infrastructure is the machinery that creates, configures, and tears down that runtime. It runs intermittently — during initial deployment, when config changes, when something fails. Think of it as Terraform for your Agent: it does its job and exits. The runtime keeps running.

The confusion between these two layers is universal. Docker straddles both: `docker create` is an infrastructure operation; the running container providing process isolation is runtime. OpenShell straddles both: `sandbox create` is infrastructure; the L7 proxy filtering network traffic at runtime is runtime. Keeping the distinction clear prevents you from mixing deployment logic into your Agent's hot path.

Most Agents start as a local process and never need more. Runtime + Infrastructure becomes relevant when you need isolation (untrusted code execution), reproducibility (consistent environments across machines), or multi-tenant deployment (each user gets their own sandbox).

## What Problem Does It Solve

**Isolation**. Your Agent runs shell commands. If those commands can access the host filesystem, network, and processes without restriction, one bad tool call can compromise the machine. Runtime isolation limits the blast radius.

**Reproducibility**. "Works on my machine" kills Agent deployments. A containerized runtime guarantees the same Python version, the same system libraries, the same file layout every time.

**Lifecycle management**. Agents need inference endpoints configured, security policies applied, health checks running. Without a structured infrastructure layer, these become ad-hoc scripts that nobody maintains and everyone forgets.

**Recovery**. Containers crash. Sandboxes hit resource limits. Processes hang. Infrastructure handles detection and recovery — restart the container, rebuild the sandbox, failover to a backup. Without it, your Agent dies silently and nobody notices until a user complains.

## How the 4 Projects Do It

### Claude Code

**Runtime**: Local process. You run `npm install -g @anthropic-ai/claude-code`, then `claude`. It starts a Node.js process on your machine. No container. No sandbox. No isolation beyond OS-level user permissions.

**Infrastructure**: None. The npm package manager is the closest thing to an infrastructure layer. There is no health check, no auto-recovery, no deployment orchestration.

**Why it works**: Claude Code is a single-user developer tool. The user IS the operator. If the process dies, the user restarts it. If a tool call does something destructive, the permission system catches it before execution. The tradeoff is explicit: zero operational overhead in exchange for zero isolation.

### hermes-agent

**Runtime**: Local process by default. Docker container when deployed to servers. Also supports Modal (cloud functions) and SSH (remote execution) as alternative backends.

**Infrastructure**: Dockerfile and docker-compose for container deployment. Modal configuration for serverless. SSH connection management for remote backends. Each backend is a separate code path — there is no unified infrastructure abstraction.

**Multiple backends**: hermes is the only project that supports choosing your runtime at deployment time. Local for development, Docker for staging, Modal for burst workloads, SSH for existing servers. The flexibility comes at the cost of maintaining four separate deployment paths.

### openclaw

**Runtime**: Docker container for the Agent process. Fly.io for cloud deployment. Tailscale for secure networking between components.

**Infrastructure**: Docker Compose orchestration. Fly.io deployment configuration. No Terraform-like abstraction — deployment is imperative scripts and platform-specific config files.

**Networking**: Tailscale is notable. It creates a private mesh network between the Agent, its tools, and external services. This solves the "how does my containerized Agent reach my local database" problem without exposing ports to the internet.

### NemoClaw

**Runtime**: OpenShell sandbox. This is the most sophisticated runtime of the four projects. Each sandbox gets:
- **Landlock** filesystem isolation — config files are read-only, writable data is confined to a specific path
- **seccomp** syscall filtering — restricts which system calls the process can make, runs as non-root, removes dangerous binaries (gcc, netcat)
- **L7 network proxy** — per-binary network policies. Not just "which domains can the Agent access" but "which specific binary can access which domain." If curl is not on the whitelist, data exfiltration via curl is impossible even if the Agent has shell access
- **SSRF protection** — DNS resolution followed by IP validation to block internal network access

**Infrastructure**: The Blueprint model. This is NemoClaw's standout contribution.

Blueprint works like Terraform: `plan` (compute what needs to change), `apply` (execute the changes), `status` (verify current state), `rollback` (revert to previous state). The onboarding process is a 7-step wizard with checkpoint resume — if step 4 fails, you restart from step 4, not step 1.

The key architectural insight: **NemoClaw itself is pure infrastructure**. It orchestrates OpenShell commands, configures Docker containers, sets up inference endpoints. Once deployment is complete, NemoClaw exits. The runtime (OpenShell sandbox + L7 proxy + openclaw instance inside the container) keeps running without NemoClaw.

**Dual-layer components**:
```
NemoClaw  -> Pure Infrastructure (orchestrates, then exits)
OpenShell -> Both layers (sandbox create = Infra, L7 proxy = Runtime)
Docker    -> Both layers (docker create = Infra, container isolation = Runtime)
Linux     -> Pure Runtime foundation (Landlock, seccomp, cgroups, namespaces)
```

## Comparison Table

| Dimension | hermes | openclaw | NemoClaw | Claude Code |
|-----------|--------|----------|----------|-------------|
| Default runtime | Local process | Docker container | OpenShell sandbox | Local process |
| Isolation level | None (local) / Basic (Docker) | Basic (Docker) | Strong (Landlock + seccomp + L7) | None |
| Alternative runtimes | Docker, Modal, SSH | Fly.io | Docker (as inner layer) | None |
| Infrastructure model | Per-backend scripts | Docker Compose + platform config | Blueprint (plan/apply/status/rollback) | npm install |
| Health checks | No | No | Yes (gateway-state.ts health classification) | No |
| Auto-recovery | No | No | Yes (detect death -> SSH probe -> restart -> registry recovery) | No |
| Checkpoint resume | No | No | Yes (7-step wizard, resume from failure point) | No |
| Networking | Direct | Tailscale mesh | Per-binary L7 proxy | Direct |

## Best Practices

**Start with a local process.** `npm install` or `pip install`, then run directly. Claude Code serves millions of users this way. Do not add Docker, sandboxing, or cloud functions until you have a concrete reason.

**Add Docker when deploying to servers.** The moment your Agent runs on a machine that is not your laptop, you need reproducible environments. A Dockerfile that pins your language version, installs dependencies, and copies your code is the minimum. hermes's approach — Dockerfile alongside the source — is the template.

**Add sandbox isolation only for untrusted code execution.** If your Agent runs arbitrary user-provided code (not just predefined tools), you need sandbox-level isolation. NemoClaw's approach — Landlock for filesystem, seccomp for syscalls, L7 proxy for network — is the gold standard. But it requires Linux, adds operational complexity, and is overkill if your Agent only calls well-defined tools.

**Separate infrastructure from runtime in your code.** Even if you start with `docker-compose up`, keep the "create and configure the environment" code separate from the "Agent runs inside this environment" code. NemoClaw's Blueprint model is the ideal. At minimum, have a `deploy.sh` that is not `start.sh`.

**Health checks are cheap insurance.** A periodic HTTP ping to your Agent's health endpoint catches crashes before users do. NemoClaw's `gateway-state.ts` classifies health into categories (healthy, degraded, dead) and triggers appropriate responses. You can start with a simple liveness check and add sophistication later.

## Step-by-Step: How to Build It

### Step 1: Start local

`python agent.py` or `node agent.js`. No Docker. No containers. This is where Claude Code stopped, and it is a valid stopping point for single-user tools.

### Step 2: Add a Dockerfile

When deploying to a server: pin your language version, `COPY` source, expose a port. Keep the Dockerfile in your repo root. Add `docker-compose.yml` if your Agent needs a database alongside it.

### Step 3: Separate deploy from run

Create two entry points: `deploy.sh` (infrastructure — build image, create container, configure endpoints) and `start.sh` (runtime — start the Agent process). The deploy script runs once. The start script runs continuously. Do not merge them.

### Step 4: Add health checks

A `/health` endpoint returning `{"status": "healthy", "uptime_seconds": N}`. Infrastructure polls this. Non-200 for N consecutive checks triggers recovery (restart process, rebuild container).

### Step 5 (optional): Add Blueprint-style infrastructure

Only if you manage multiple Agent instances or need reproducible multi-step deployments.

The interface: `plan(desired_state) -> steps`, `apply(steps) -> result`, `status() -> current_state`, `rollback(checkpoint) -> result`. Four methods, like Terraform.

NemoClaw's 7-step onboarding wizard is the reference. Each step (create sandbox, configure network, deploy Agent, set up inference, apply security policies, verify health, register in gateway) checkpoints on success. Failure at step 5 means resume from step 5, not step 1.

### Step 6 (optional): Add sandbox isolation

Only if your Agent executes untrusted code:

```bash
# Filesystem isolation (Linux only)
landlock --ro /etc --ro /usr --rw /app/data

# Syscall filtering
seccomp --default deny --allow read,write,open,close,mmap,...

# Network isolation
l7-proxy --allow "node:api.openai.com:443" --deny-all
```

This is NemoClaw territory. The per-binary network policy is the key innovation: even if an attacker gets shell access inside the sandbox, they cannot exfiltrate data because only the Agent binary (not curl, not wget, not netcat) is allowed to make network requests.

## Common Pitfalls

**1. Premature containerization.** Your Agent has 3 users and runs on your laptop. You spend two days debugging Docker networking instead of improving your Agent. Local process is fine until you actually need isolation or multi-machine deployment.

**2. Infrastructure in the hot path.** Your Agent checks "is my container healthy?" on every tool call. Health checks belong in a separate process or cron job, not in the request path. NemoClaw's `gateway-state.ts` runs independently of the Agent's main loop.

**3. No checkpoint resume.** Your 7-step deployment fails at step 6. You re-run from step 1, which takes 10 minutes. NemoClaw solved this with checkpoint-based resume. If you have multi-step deployment, checkpoint after each step.

**4. Confusing Docker-the-infrastructure with Docker-the-runtime.** `docker build` and `docker create` are infrastructure operations. The process isolation and resource limits the running container provides are runtime. When people say "just use Docker," they usually mean both layers without realizing it. Keep your mental model clean.

**5. Sandbox without threat model.** You add Landlock, seccomp, and an L7 proxy because "security." But your Agent only calls a weather API and reads local files. The operational overhead of sandbox management exceeds the security benefit. Write down what you are protecting against before adding isolation layers.

## Cross-Module Contracts

### What Runtime + Infrastructure expects from other modules

| Module | Contract | Why |
|--------|----------|-----|
| **Agent Core** | Health endpoint or heartbeat signal | Infrastructure needs to know if the Agent is alive |
| **Model Service** | Inference endpoint configuration | Infrastructure configures which model endpoint the Agent connects to |
| **Security Envelope** | Security policy definition | Infrastructure applies filesystem, network, and syscall policies at deployment time |
| **Tools** | Resource requirements declaration | Infrastructure needs to know what system resources tools require (network access, filesystem paths, GPU) |

### What Runtime + Infrastructure provides to other modules

| Consumer | What it provides | Format |
|----------|-----------------|--------|
| **Agent Core** | Isolated execution environment | Process/container/sandbox with configured permissions |
| **Model Service** | Configured inference connectivity | Network route to model API, credentials injected as env vars or mounted secrets |
| **Tools** | Filesystem and network boundaries | Writable paths, allowed network destinations, available system binaries |
| **Security Envelope** | Enforcement of security policies | Landlock rules, seccomp filters, L7 proxy rules — enforced at OS level, not application level |

### Invariants

- Infrastructure operations never run inside the Agent's main loop. They run before the Agent starts, or in a separate management process.
- Runtime isolation is transparent to the Agent. The Agent does not know (and should not care) whether it runs in a local process or an OpenShell sandbox. The same code runs in both.
- Health checks are pull-based (infrastructure polls the Agent), not push-based (Agent reports to infrastructure). This way, a crashed Agent that cannot push is still detected.
- Checkpoint state is stored outside the runtime it manages. If the sandbox crashes, checkpoint data survives for recovery.
