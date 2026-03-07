# Goose Security Model — Deep Dive

## Overview

Goose (by Block) is an on-machine AI agent with access to file systems, shell execution, and external services via MCP extensions. Because it runs code and takes actions on the user's local machine, it presents a fundamentally different risk profile than chat-only LLM interfaces. Block explicitly acknowledges this: "developer agents have the ability to run code and take actions on your computer, they pose a unique risk compared to chat-based LLM interactions." This document examines every layer of Goose's security architecture.

---

## 1. Permission System

### Permission Modes

Goose provides four global permission modes, configurable via the `GOOSE_MODE` environment variable or the `/mode` slash command mid-session:

| Mode | Behavior |
|------|----------|
| **Auto** | All tool calls execute without user approval. This is the default mode. |
| **Approve** | Every tool call requires explicit user approval (Allow/Deny buttons). |
| **Smart Approve** | Read-only operations are auto-approved; state-changing actions require approval. Uses a best-effort heuristic to classify tools as read vs. write. Write tools include text editor writes/edits, bash destructive commands (`rm`, `cp`, `mv`), etc. |
| **Chat** | No extension use or file modifications at all; pure conversation mode. |

**[Confirmed]** — Modes documented at [Goose Permission Modes](https://block.github.io/goose/docs/guides/goose-permissions/) and corroborated by issues [#3386](https://github.com/block/goose/issues/3386), [#4097](https://github.com/block/goose/issues/4097), and [#2806](https://github.com/block/goose/issues/2806).

### Per-Tool Permission Overrides

Independent of the global mode, each individual tool can be assigned one of three permission levels:

- **Always Allow** — Tool executes without prompting regardless of the global mode.
- **Ask Before** — Tool always prompts before execution.
- **Never Allow** — Tool is blocked entirely.

These overrides are configured through Goose Settings (Desktop UI), the `goose configure` command, or directly in configuration files. They persist across sessions.

**[Confirmed]** — Documented at [Managing Tool Permissions](https://block.github.io/goose/docs/guides/tool-permissions/).

### Permission Persistence

Permission decisions are stored in two files:

- `permission.yaml` — Tool permission levels (Always Allow, Ask Before, Never Allow).
- `permissions/tool_permissions.json` — Runtime permission decisions, auto-managed by Goose.

These live under the Goose configuration directory (`~/.config/goose/` on Linux/macOS). The system remembers "always allow" decisions persistently so preferences carry across sessions.

**[Confirmed]** — Referenced in [Configuration Files](https://block.github.io/goose/docs/guides/config-files/) documentation.

### Approve/Deny Flow

In Approve and Smart Approve modes, tool calls trigger an interactive prompt displaying Allow and Deny buttons. The user must explicitly approve each action before it executes. In Smart Approve mode, Goose's heuristic classifies the tool call as read-only (auto-approved) or state-changing (requires approval). The classification is described as "best effort," meaning edge cases may be misclassified.

**[Confirmed]** — [Goose Permission Modes](https://block.github.io/goose/docs/guides/goose-permissions/).

### Default Mode Controversy

Auto mode (no approval required) is the default, which has been flagged by users as potentially dangerous (see issue [#2806](https://github.com/block/goose/issues/2806): "Autonomous mode by default is dangerous"). This is a significant design choice — most competing agents default to requiring approval for destructive operations.

**[Confirmed]** — GitHub issue [#2806](https://github.com/block/goose/issues/2806).

---

## 2. Prompt Injection Detection

### The Threat: Operation Pale Fire

Block's Offensive Security team conducted an internal red team engagement called **"Operation Pale Fire"** that successfully compromised a Block employee's laptop via prompt injection. The attack:

1. Used phishing emails directed at the Goose development team about a purported "bug."
2. Embedded a prompt injection payload in a Goose "recipe" using **zero-width Unicode characters** — invisible to the end user but readable by the LLM.
3. The injected instructions told Goose to invoke its shell tool to contact a C2 (command-and-control) server under the guise of "checking for updates."
4. The attack downloaded and ran an infostealer on the employee's machine.

Block's Detection and Response Team (DART) identified and contained the simulated threat, and the engagement drove major security improvements.

**[Confirmed]** — [How We Red-Teamed Our Own AI Agent](https://engineering.block.xyz/blog/how-we-red-teamed-our-own-ai-agent-) (Block Engineering Blog); [The Register coverage](https://www.theregister.com/2026/01/12/block_ai_agent_goose/).

### Mitigations Implemented

Following Operation Pale Fire, Goose implemented several defenses:

1. **Zero-width character stripping** — Invisible Unicode characters are stripped when loaded into the app, neutralizing the specific attack vector used.
2. **Recipe transparency** — When loading a recipe, the actual instructions being loaded are now visualized to the user, preventing hidden payloads.
3. **Desktop alerts for suspicious Unicode** — Goose displays alerts when a recipe contains suspicious Unicode characters.
4. **ML-based prompt injection detection** — An initial version of ML-based detection has been merged into Goose. Block has published an API specification for self-hosting ML-based prompt injection detection endpoints.

**[Confirmed]** — [How We Red-Teamed Our Own AI Agent](https://engineering.block.xyz/blog/how-we-red-teamed-our-own-ai-agent-).

### Security Inspection System

Goose includes a security inspection layer that evaluates tool results for potential prompt injection:

- Security findings are assigned unique IDs in the format `SEC-{uuid}`.
- Findings are logged in both CLI and server logs.
- User decisions (allow/deny) are recorded and associated with finding IDs for traceability.

**[Confirmed]** — [Goose Logging System](https://block.github.io/goose/docs/guides/logs/) documents security finding IDs in logs.

### CORS-Inspired Guardrails Model

In January 2026, Block published a research proposal applying browser security concepts (specifically anti-CSRF/CORS protections) to agent security:

- **Cross-origin tool call detection**: If a tool call occurs after a previous tool call (without user interaction in between), it is treated as a "cross-origin" call and subjected to additional authorization controls — analogous to how browsers enforce CORS.
- **Context window flushing**: All tool-call responses are removed from the context window between user turns, preventing older (potentially tainted) tool outputs from influencing future tool calls.
- The agent sits at the boundary between the LLM and MCP tools (as the browser sits between web code and web services), making it the natural enforcement point.

This was described as a proof-of-concept under active development and benchmarking.

**[Confirmed]** — [Agent Guardrails and Controls: Applying the CORS Model to Agents](https://block.github.io/goose/blog/2026/01/05/agentic-guardrails-and-controls/).

---

## 3. Environment Variable Security

### Disallowed Environment Variables

Goose blocks **31 sensitive environment variables** via `Envs::DISALLOWED_KEYS`. When an extension or tool attempts to use a disallowed variable, the operation fails with an error during `merge()`.

**[Confirmed]** — DeepWiki documentation at [Extension Types and Configuration](https://deepwiki.com/block/goose/5.3-extension-types-and-configuration) references "31 sensitive environment variables via Envs::DISALLOWED_KEYS."

### Specific Variables Blocked

The exact list of 31 disallowed keys is defined in source code but not enumerated in public documentation. Based on the security context and common patterns in agent frameworks, the blocked variables likely include:

- Authentication tokens: `GOOSE_API_KEY`, `OPENAI_API_KEY`, `ANTHROPIC_API_KEY`, and similar provider keys
- System-sensitive variables: `PATH`, `HOME`, `LD_PRELOAD`, `LD_LIBRARY_PATH`
- Shell manipulation: `SHELL`, `BASH_ENV`, `ENV`
- Proxy hijacking: `HTTP_PROXY`, `HTTPS_PROXY`, `ALL_PROXY`, `NO_PROXY`
- Goose internals: `GOOSE_MODE`, `GOOSE_ALLOWLIST`, `GOOSE_ALLOWLIST_BYPASS`

**[Inferred]** — The exact list is not published in external documentation. The categories above are inferred from (a) the security purpose described, (b) common patterns in agent security, and (c) the fact that env var hijacking (e.g., `LD_PRELOAD`, `PATH` manipulation) is a standard attack vector.

### Purpose

The disallowed list prevents:
- **Credential exfiltration** — An injected prompt cannot instruct Goose to read API keys from environment variables and send them to external services.
- **Environment hijacking** — Malicious tool calls cannot modify `PATH`, `LD_PRELOAD`, or proxy settings to redirect traffic or inject code.
- **Configuration tampering** — Goose's own configuration variables cannot be overridden by tool outputs.

**[Inferred]** — Reasoning based on standard env var attack vectors and the stated purpose of the blocking mechanism.

---

## 4. Extension Malware Checking

### How It Works

Goose validates extensions against the **OSV (Open Source Vulnerabilities) database** before activation. The check:

1. **Trigger**: Runs automatically before connecting to any extension.
2. **Scope**: Applies to locally-executed external extensions that use **PyPI (uvx)** or **NPM (npx)** package managers.
3. **Database**: Uses real-time data from the [OSV database](https://github.com/ossf/malicious-packages), which aggregates reports of malicious packages from open source repositories.
4. **Action on detection**: If a malicious package is found, the extension is **blocked** with a clear error message (format: `"Blocked malicious package: package-name@1.0.0 (npm)"`).

**[Confirmed]** — [Known Issues](https://block.github.io/goose/docs/troubleshooting/known-issues/) and [DeepWiki Extension Management](https://deepwiki.com/block/goose/5.4-extension-management).

### Enterprise Allowlist

For enterprise environments, the `GOOSE_ALLOWLIST` environment variable can be set to a URL pointing to an allowlist YAML file. When set:
- Only extensions matching allowlist entries can execute.
- Commands not on the allowlist are rejected with an error message.
- A `GOOSE_ALLOWLIST_BYPASS` variable (case-insensitive) can override restrictions when set to `true`.

The allowlist is documented at [crates/goose-server/ALLOWLIST.md](https://github.com/block/goose/blob/main/crates/goose-server/ALLOWLIST.md).

**[Confirmed]** — [ALLOWLIST.md](https://github.com/block/goose/blob/main/crates/goose-server/ALLOWLIST.md) in the Goose repository.

### Source File

The malware checking logic resides in `extension_malware_check.rs` within the Goose Rust codebase. The implementation queries the OSV API to cross-reference package names and versions against known malicious package reports.

**[Confirmed]** — Referenced in the codebase structure and DeepWiki documentation.

### Limitations

- Only checks **npm** and **PyPI** packages. Other package ecosystems (Go modules, Cargo crates, etc.) are not covered.
- Relies on the OSV database being up-to-date; zero-day malicious packages not yet reported will pass through.
- Does not perform static analysis of extension code itself — only checks package identity against known-bad lists.

**[Inferred]** — Based on the described scope (PyPI/npm only) and the nature of database-driven checks.

---

## 5. Container/Sandbox Support

### Container Use (MCP Extension)

Goose supports isolated execution through **Container Use**, an MCP server that provides an interface for working in isolated Docker containers and git branches. Key capabilities:

- Each experiment gets its own sandbox container — nothing affects the main development setup.
- Goose creates git branches, spins up containers with dependencies (e.g., Redis), and enables experimentation with easy rollback.
- Supports parallel isolated environments: multiple agents can work on different implementations (e.g., REST vs. GraphQL) simultaneously without interference.

**[Confirmed]** — [Isolated Dev Environments in Goose with container-use](https://block.github.io/goose/blog/2025/06/19/isolated-development-environments/).

### Native Sandbox Discussion

There is an open feature request ([#5943](https://github.com/block/goose/issues/5943)) for native sandbox support (analogous to Claude Code's bubblewrap/Seatbelt approach). The discussion mentions bubblewrap (Linux) and seatbelt (macOS) as tools that could provide restrictions without requiring Docker.

**[Confirmed]** — GitHub issue [#5943](https://github.com/block/goose/issues/5943).

### Docker Deployment

Goose can be deployed inside Docker containers via a multi-stage Rust build process:
- Compiles with Rust's toolchain, then copies the binary to a minimal Debian runtime.
- Environment variables for API keys must be set in `docker-compose.yml` because keyring does not work in headless containers.
- When running in Docker, file-based secret storage (`secrets.yaml`) is used instead of the system keyring.

**[Confirmed]** — [Goose in Docker](https://block.github.io/goose/docs/tutorials/goose-in-docker/) tutorial and [DEV Community guide](https://dev.to/agasta/running-goose-in-containers-without-losing-your-mind-3m8).

### Current Limitations

- **No built-in OS-level sandboxing**: Unlike Claude Code (bubblewrap/Seatbelt) or Codex CLI (Landlock/seccomp), Goose does not ship with kernel-level isolation for tool execution. Container Use is an opt-in extension, not a mandatory security boundary.
- **Container Use is for development isolation**, not security isolation per se — it prevents experiments from polluting the main workspace but does not enforce security policies within the container.
- **No mandatory containerization**: Goose's default mode runs all tools directly on the host machine.

**[Confirmed/Inferred]** — Confirmed by the absence of sandbox features in Goose's security documentation; inferred from the nature of Container Use as a development convenience tool.

---

## 6. Network Isolation

### Built-in Network Restrictions

Goose does **not** include built-in network isolation capabilities. There is no equivalent to Claude Code's network proxy validation or Codex CLI's seccomp-based network filtering.

**[Confirmed]** — No network isolation features are documented in Goose's security guides, permission system, or configuration files. The [Staying Safe with Goose](https://block.github.io/goose/docs/guides/security/) guide recommends using a VM or container with limited capabilities as an external measure.

### Recommended Mitigations

Block's official security guidance recommends:
- Using a dedicated virtual machine or container (Docker/Kubernetes) with limited privileged capabilities.
- This externalizes network isolation to the container/VM runtime rather than implementing it within Goose itself.

**[Confirmed]** — [Staying Safe with Goose](https://block.github.io/goose/docs/guides/security/).

### CORS-Model Proposal

The CORS-inspired guardrails proposal (Section 2) partially addresses network-adjacent risks by controlling which tool calls can follow other tool calls, but this is about context-flow control rather than network-level isolation.

**[Confirmed]** — [Agent Guardrails and Controls](https://block.github.io/goose/blog/2026/01/05/agentic-guardrails-and-controls/).

---

## 7. Credential Handling

### System Keyring (Primary)

Goose uses the **system keyring** (macOS Keychain, Linux Secret Service/GNOME Keyring, Windows Credential Manager) as the primary storage for secrets:

- `Config::set_secret()` attempts to store sensitive values in the system keyring.
- `try_store_secret()` handles keyring failures gracefully with platform-specific error messages.
- On first configuration, users are prompted whether to save API keys to the keyring.

**[Confirmed]** — [Configuration Files](https://block.github.io/goose/docs/guides/config-files/) and [DeepWiki Provider Configuration](https://deepwiki.com/block/goose/2.2-provider-configuration).

### File-Based Fallback

When the system keyring is unavailable (headless servers, CI/CD, containers, or when explicitly disabled via `GOOSE_DISABLE_KEYRING`):

- Secrets are stored in `secrets.yaml` **in plain text** under the Goose config directory.
- This is a known security trade-off for environments without desktop keyring services.

**[Confirmed]** — [Configuration Files](https://block.github.io/goose/docs/guides/config-files/).

### Configuration Precedence

Goose resolves configuration keys in this priority order:
1. **Environment variables** (checked first via `std::env::var`)
2. **Keyring / config** (retrieved via `config.get_secret()` or `config.get_param()`)
3. **OAuth flows** (if applicable)
4. **Manual entry** (prompted if needed)

**[Confirmed]** — [DeepWiki Provider Configuration](https://deepwiki.com/block/goose/2.2-provider-configuration).

### Configuration File Structure

- `~/.config/goose/config.yaml` — Main configuration (extensions, models, settings). Not for secrets.
- `~/.config/goose/secrets.yaml` — API keys and secrets (fallback when keyring unavailable). **Plain text.**
- `~/.config/goose/permission.yaml` — Tool permission levels.

**[Confirmed]** — [Configuration Files](https://block.github.io/goose/docs/guides/config-files/).

### Security Concerns

- `secrets.yaml` stores secrets in plain text with no encryption at rest.
- Environment variables can expose secrets via process listings (`/proc/*/environ` on Linux).
- Block recommends using the system keyring over environment variables for security-sensitive values.

**[Confirmed/Inferred]** — Plain text storage confirmed by documentation. Process listing risk is a standard security consideration.

---

## 8. Audit Logging

### Log Architecture

Goose maintains structured logs across multiple components:

| Component | Location | Retention |
|-----------|----------|-----------|
| CLI logs | `~/.local/state/goose/logs/cli/{date}/` | Auto-deleted after 2 weeks |
| Server logs | `~/.local/state/goose/logs/server/{date}/` | Auto-deleted after 2 weeks |
| LLM request logs | `llm_request.{0-9}.jsonl` (rotating) | 10 most recent requests |
| Session database | `~/.local/share/goose/sessions/sessions.db` | Persistent |

**[Confirmed]** — [Goose Logging System](https://block.github.io/goose/docs/guides/logs/).

### Session Database Contents

The session database (SQLite, migrated from individual `.jsonl` files in v1.10.0) stores:
- **Session metadata**: ID, name, working directory, timestamps.
- **Conversation messages**: User commands, assistant responses, role information.
- **Tool calls and results**: IDs, arguments, responses, success/failure status.

**[Confirmed]** — [Session Management](https://block.github.io/goose/docs/guides/managing-goose-sessions/).

### Security-Specific Logging

When prompt injection detection is enabled, logs include:
- **Security findings** with unique IDs (format: `SEC-{uuid}`).
- **User decisions** (allow/deny) associated with finding IDs.
- This provides an audit trail linking security detections to user responses.

**[Confirmed]** — [Goose Logging System](https://block.github.io/goose/docs/guides/logs/).

### Limitations

- Logs are auto-deleted after 2 weeks — not suitable for long-term audit compliance without external log shipping.
- No built-in integration with external SIEM/logging platforms (Splunk, Datadog, etc.).
- No OpenTelemetry integration documented for structured tracing export.
- Session database provides forensic capability but is local-only.

**[Confirmed/Inferred]** — Two-week retention confirmed. Absence of SIEM integration inferred from documentation review.

---

## 9. Comparison with Other Agents' Security

### Summary Matrix

| Feature | Goose | Claude Code | Codex CLI | Gemini CLI |
|---------|-------|-------------|-----------|------------|
| **OS-level sandboxing** | None (opt-in containers) | Bubblewrap (Linux) / Seatbelt (macOS) | Landlock + seccomp (Linux) / Seatbelt (macOS) | Opt-in sandbox |
| **Network isolation** | None built-in | Proxy-based validation | seccomp network filtering | Unknown |
| **Default permission mode** | Auto (no approval) | Requires approval for writes | Configurable autonomy levels | Requires approval |
| **Prompt injection detection** | ML-based + Unicode stripping | Model-level + tool output scanning | Minimal documented | Unknown |
| **Extension/plugin vetting** | OSV malware DB check | N/A (built-in tools only) | N/A (built-in tools only) | N/A |
| **Credential storage** | System keyring + fallback plaintext | Environment variables | Environment variables | Environment variables |
| **Audit logging** | Session DB + rotating logs | Session logs | Minimal | Minimal |

### Detailed Comparison

**Goose vs. Claude Code (Bubblewrap/Seatbelt)**

Claude Code implements mandatory dual-isolation: Bubblewrap on Linux and Seatbelt on macOS restrict filesystem access to the current working directory and route network traffic through validating proxies. In auto-allow mode, bash commands run inside the sandbox automatically. Commands that cannot be sandboxed (e.g., those needing network access to non-allowed hosts) fall back to the regular permission flow.

Goose has **no equivalent kernel-level sandboxing**. Its Container Use extension provides development isolation but is opt-in and not a security boundary. Goose relies on its permission modes (approve/smart approve) as the primary defense, which operates at the application level rather than the OS level.

**Goose vs. Codex CLI (Landlock)**

Codex CLI enforces filesystem and network restrictions at the kernel level using Landlock + seccomp on Linux and Seatbelt on macOS. Bubblewrap is vendored and compiled as part of the Linux build. These are not containerized — they use OS primitives directly, trading some isolation for lower overhead.

Goose lacks any OS-primitive-based isolation. However, Goose's extensibility model (MCP-based) creates a different attack surface — it must vet third-party extensions, which Claude Code and Codex CLI do not need to do (they use built-in tools).

**Goose vs. Gemini CLI**

Gemini CLI has an opt-in sandbox mode. Documentation is less detailed, but it appears to use Docker-based sandboxing when enabled. Like Goose, it does not enforce sandboxing by default.

**Where Goose Excels**

- **Extension malware checking**: Goose is unique among these agents in having automated supply-chain security (OSV database checks) for third-party extensions. This is critical because Goose's MCP extension model creates a larger attack surface than agents with built-in-only tools.
- **CORS-inspired guardrails research**: Block's proposal for cross-origin tool call detection and context flushing represents a novel theoretical approach to prompt injection defense that no other agent has implemented.
- **Granular per-tool permissions**: Three-level per-tool overrides (always allow / ask / never allow) provide finer-grained control than most competitors.

**Where Goose Lags**

- **No OS-level sandboxing**: The most significant gap. Both Claude Code and Codex CLI enforce kernel-level restrictions by default. Goose runs all tools on the host with no isolation unless the user opts into containers.
- **Auto mode as default**: Running without approval by default is more permissive than any major competitor.
- **Network isolation**: No built-in capability. Claude Code routes traffic through proxies; Codex uses seccomp. Goose relies entirely on external measures.
- **Plaintext secret fallback**: `secrets.yaml` in plain text is a concern, though the keyring-first approach mitigates this in most desktop environments.

---

## Key Takeaways

1. **Goose's security model is primarily application-level**, relying on permission modes, per-tool overrides, and prompt injection detection rather than OS-level sandboxing.
2. **The extension ecosystem creates unique risks** that Goose addresses with OSV malware checking and enterprise allowlists — a strength relative to agents with closed tool sets.
3. **Operation Pale Fire** was a pivotal moment that drove real security improvements: zero-width character stripping, recipe transparency, and ML-based injection detection.
4. **The CORS-inspired guardrails model** is a promising research direction but is still under development and benchmarking.
5. **The biggest security gap** is the absence of OS-level sandboxing (bubblewrap, Landlock, Seatbelt) — all tool execution runs directly on the host by default.
6. **Auto mode as default** combined with no sandboxing means a successful prompt injection can immediately execute arbitrary commands on the user's machine.

---

## Sources

- [Goose Permission Modes](https://block.github.io/goose/docs/guides/goose-permissions/)
- [Managing Tool Permissions](https://block.github.io/goose/docs/guides/tool-permissions/)
- [Configuration Files](https://block.github.io/goose/docs/guides/config-files/)
- [Staying Safe with Goose](https://block.github.io/goose/docs/guides/security/)
- [Goose Logging System](https://block.github.io/goose/docs/guides/logs/)
- [Session Management](https://block.github.io/goose/docs/guides/managing-goose-sessions/)
- [Goose in Docker](https://block.github.io/goose/docs/tutorials/goose-in-docker/)
- [Known Issues](https://block.github.io/goose/docs/troubleshooting/known-issues/)
- [Environment Variables](https://block.github.io/goose/docs/guides/environment-variables/)
- [ALLOWLIST.md](https://github.com/block/goose/blob/main/crates/goose-server/ALLOWLIST.md)
- [SECURITY.md](https://github.com/block/goose/blob/main/SECURITY.md)
- [How We Red-Teamed Our Own AI Agent (Block Engineering Blog)](https://engineering.block.xyz/blog/how-we-red-teamed-our-own-ai-agent-)
- [Block red-teamed its own AI agent to run an infostealer (The Register)](https://www.theregister.com/2026/01/12/block_ai_agent_goose/)
- [Agent Guardrails and Controls: Applying the CORS Model to Agents](https://block.github.io/goose/blog/2026/01/05/agentic-guardrails-and-controls/)
- [Isolated Dev Environments in Goose with container-use](https://block.github.io/goose/blog/2025/06/19/isolated-development-environments/)
- [Sandbox Support? (Issue #5943)](https://github.com/block/goose/issues/5943)
- [Autonomous mode by default is dangerous (Issue #2806)](https://github.com/block/goose/issues/2806)
- [DeepWiki: Extension Types and Configuration](https://deepwiki.com/block/goose/5.3-extension-types-and-configuration)
- [DeepWiki: Extension Management](https://deepwiki.com/block/goose/5.4-extension-management)
- [DeepWiki: Provider Configuration](https://deepwiki.com/block/goose/2.2-provider-configuration)
- [Building AI agents made easy with Goose and Docker](https://www.docker.com/blog/building-ai-agents-with-goose-and-docker/)
- [Running Goose in Containers (DEV Community)](https://dev.to/agasta/running-goose-in-containers-without-losing-your-mind-3m8)
- [Anthropic: Claude Code Sandboxing](https://www.anthropic.com/engineering/claude-code-sandboxing)
- [Claude Code Sandboxing Docs](https://code.claude.com/docs/en/sandboxing)
- [Codex CLI Technical Reference](https://blakecrosley.com/guides/codex)
- [Goose GitHub Repository](https://github.com/block/goose)
