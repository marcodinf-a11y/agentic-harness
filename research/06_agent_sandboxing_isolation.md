# Agent Execution Isolation and Sandboxing (Aug 2025 -- Mar 2026)

## 1. Executive Summary

The proliferation of AI coding agents capable of executing arbitrary code has made execution isolation a critical infrastructure concern. Between August 2025 and March 2026, the ecosystem has converged on several layered approaches: Firecracker microVMs for maximum security, Docker containers for speed and convenience, OS-level sandboxing for lightweight local use, gVisor for syscall interception without full VM overhead, and git worktrees for source-level isolation.

---

## 2. Isolation Technologies

### 2.1 Firecracker MicroVMs

- Boot time under 125ms; creation rate up to 150 microVMs/second/host
- Less than 5 MiB RAM overhead per microVM
- Each microVM runs its own Linux kernel
- Powers E2B, Vercel Sandboxes, AWS Lambda/Fargate

### 2.2 Docker Containers

- Most widely used isolation primitive
- Three critical runC vulnerabilities disclosed November 2025
- Insufficient alone for multi-tenant or untrusted code execution
- Hardening: Seccomp, AppArmor/SELinux, read-only root, user namespaces

### 2.3 gVisor

- User-space kernel reimplementing ~70-80% of Linux syscalls
- 10-30% overhead on I/O-heavy workloads
- Best for compute-heavy agents with limited I/O

### 2.4 Kata Containers

- OCI containers + lightweight VMs via KVM
- Boot time 150-300ms
- Kubernetes-native via CRI-O/containerd

### 2.5 OS-Level Sandboxing (Bubblewrap / Seatbelt)

- Claude Code's sandbox runtime (srt)
- Linux: bubblewrap; macOS: Seatbelt
- Reduces permission prompts by 84%
- Zero container overhead

### 2.6 Git Worktrees

- Source-level isolation, near-zero overhead
- Natural branch-based isolation for parallel agents
- No process, network, or resource isolation
- Must be combined with container/VM isolation for security

---

## 3. Platform Implementations

### 3.1 E2B

Firecracker microVMs, pre-warmed snapshot pools, <200ms cold start, 24h session persistence. Partnership with Docker for 200+ MCP Catalog tools.

### 3.2 Daytona

Sub-90ms cold starts, long-lived workspaces, multiple isolation backends (Docker, Kata, Sysbox). Customer-managed compute.

### 3.3 OpenAI Codex

Isolated containers, internet disabled, repo preloaded, 12h cache. Network-isolated by default.

### 3.4 Claude Code

Local-first with OS-level sandboxing (bubblewrap/Seatbelt). Configurable directory/network allowlists. Docker Sandboxes integration for stronger isolation.

### 3.5 Docker Sandboxes

Experimental sandbox support for coding agents. Transitioning from containers to dedicated microVMs. Supports Claude Code, Codex, Gemini CLI.

---

## 4. SWE-bench Isolation

Each task in isolated Docker container. 35,000 evaluations/day. SWE-bench-Live adds 50 issues monthly. SWE-bench-Live/Windows (Feb 2026) for PowerShell environments.

---

## 5. Comparison Table

| Feature | E2B | Daytona | OpenAI Codex | Claude Code (srt) | Docker Sandboxes | Git Worktrees |
|---|---|---|---|---|---|---|
| **Isolation tech** | Firecracker microVM | Docker / Kata / Sysbox | Managed containers | bubblewrap / Seatbelt | Docker VM -> microVM | None (source only) |
| **Cold start** | <200ms | <90ms | Cached (12h) | Instant | Seconds | Instant |
| **Network isolation** | Yes | Yes | Yes (no internet) | Configurable | Yes | No |
| **Multi-tenant safe** | Yes | Yes | Yes | No (single-user) | Yes | No |
| **Best for** | Cloud agent platforms | Enterprise / on-prem | Autonomous cloud tasks | Local dev with guardrails | Multi-agent local dev | Parallel branch work |

---

## Sources

- [E2B](https://e2b.dev/) / [E2B Docs](https://e2b.dev/docs) / [E2B GitHub](https://github.com/e2b-dev/E2B)
- [Daytona](https://www.daytona.io/) / [Daytona Architecture](https://www.daytona.io/docs/en/architecture/)
- [Codex Security](https://developers.openai.com/codex/security/)
- [Claude Code Sandboxing](https://www.anthropic.com/engineering/claude-code-sandboxing)
- [Docker Sandboxes](https://www.docker.com/blog/docker-sandboxes-a-new-approach-for-coding-agent-safety/)
- [Firecracker -- Amazon Science](https://www.amazon.science/blog/how-awss-firecracker-virtual-machines-work)
- [NVIDIA Sandboxing Guidance](https://developer.nvidia.com/blog/practical-security-guidance-for-sandboxing-agentic-workflows-and-managing-execution-risk/)
- [Google Open Source -- Kubernetes Agent Execution](https://opensource.googleblog.com/2025/11/unleashing-autonomous-ai-agents-why-kubernetes-needs-a-new-standard-for-agent-execution.html)
- [Git Worktrees for AI Agents -- Upsun](https://devcenter.upsun.com/posts/git-worktrees-for-parallel-ai-coding-agents/)
- [SWE-bench Scaling -- AI21](https://www.ai21.com/blog/scaling-agentic-evaluation-swe-bench/)
