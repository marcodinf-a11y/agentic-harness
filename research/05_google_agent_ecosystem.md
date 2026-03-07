# Google's Agent Ecosystem: Comprehensive Research (Aug 2025 -- Mar 2026)

## 1. Gemini CLI

### 1.1 Architecture

Gemini CLI is an open-source AI agent that brings Gemini directly into the terminal. It is hosted at [github.com/google-gemini/gemini-cli](https://github.com/google-gemini/gemini-cli).

**Monorepo structure using npm workspaces:**
- `packages/cli` -- User-facing terminal UI, input processing, display rendering.
- `packages/core` -- Backend logic, Gemini API orchestration, prompt construction, tool execution.

**Core loop:** Gemini CLI operates on a **ReAct (Reason and Act) loop**. The model reasons about the task, selects a tool, executes it, observes the result, and repeats until the task is complete.

**Built-in tools:** `read_many_files` / `list_directory`, `run_shell_command`, `edit_file`, `WebFetchTool`, `WebSearchTool`, `MemoryTool`.

**MCP integration:** Gemini CLI acts as an MCP client, connecting to local or remote MCP servers.

**Safety model:** Tools that modify files or execute shell commands require manual approval. Optionally, tool execution can run in secure, containerized sandboxes.

### 1.2 Output Format

- **Interactive (default)** -- Rich terminal rendering with Markdown.
- `--output-format json` -- Full JSON output for programmatic consumption.
- `--output-format stream-json` -- Streaming JSON for real-time CI/CD integration.

### 1.3 Session Management

Session management was introduced in **December 2025** (v0.20.0+) and is enabled by default. Sessions are stored at `~/.gemini/tmp/<project_hash>/chats/`, scoped per project directory. Resume via `/resume` command or `gemini --resume`.

---

## 2. Agent Development Kit (ADK)

ADK was announced at **Google Cloud NEXT 2025** as an open-source, code-first framework for building, evaluating, and deploying AI agents and multi-agent systems.

### 2.1 Language Support

Python (initial), Java (v0.1.0), Go (November 2025), TypeScript (2025).

### 2.2 Multi-Agent Orchestration Patterns

| Pattern | Description |
|---|---|
| **SequentialAgent** | Assembly line -- sub-agents run one after another in a predefined order. |
| **ParallelAgent** | Manager pattern -- all sub-agents run concurrently for independent tasks. |
| **LoopAgent** | Iterative refinement with `max_iterations` and early exit via `escalate=True`. |

Plus **LLM-driven dynamic routing** for adaptive behavior.

### 2.3 Evaluation Framework

Built-in evaluation with two dimensions: final response quality and trajectory evaluation. **User Simulation** generates dynamic user-side conversations to evaluate agent goal achievement.

---

## 3. Vertex AI Agent Builder

### 3.1 Core Components

| Component | Function |
|---|---|
| **Agent Development Kit (ADK)** | Open-source framework for building multi-agent systems. |
| **Agent Engine Runtime** | Fully managed runtime for deploying and scaling agents. |
| **Agent Engine Sessions** | Stores individual user-agent interactions. |
| **Memory Bank** | Stores and retrieves information across sessions. Public preview. |
| **Code Execution** | Managed code execution environment. |
| **Agent Designer** | Visual design interface for building agents. |

---

## 4. Agent2Agent Protocol (A2A)

### 4.1 Overview

A2A is an open protocol enabling AI agents to communicate across platforms. Announced **April 2025** with 50+ partners. Built on HTTP, SSE, JSON-RPC.

### 4.2 Core Concepts

- **AgentCard:** Metadata document describing agent capabilities.
- **Tasks:** Unit of work with lifecycle tracking.
- **Streaming:** SSE-based for long-running tasks.

### 4.3 Versions

v0.1 (initial), v0.2 (stateless interactions), v0.3 (gRPC, signed security cards).

### 4.4 Ecosystem

Donated to Linux Foundation (June 2025). Founding members: AWS, Cisco, Google, Microsoft, Salesforce, SAP, ServiceNow. Companion protocols: A2UI, AP2 (Agent Payments Protocol).

---

## 5. Gemini Context Window Management

### 5.1 Context Windows

| Model | Context Window |
|---|---|
| Gemini 1.5 Pro | 2M tokens |
| Gemini 2.5 Pro | 1M tokens (2M coming) |
| Gemini 3 Pro | 1M tokens |
| Gemini 3.1 Pro | 1M tokens |

### 5.2 Context Caching

- **Implicit caching (Gemini 2.5+):** Automatic, 90% discount on cached reads.
- **Explicit caching:** Named cache objects with configurable TTL.

Gemini models achieve **>99% retrieval accuracy** across the full context window.

---

## Key Takeaways

1. **Gemini CLI** is a mature, open-source coding agent with ReAct loop, MCP, structured output, and session persistence.
2. **ADK** is Google's code-first, multi-language, model-agnostic agent framework with built-in evaluation.
3. **Vertex AI Agent Builder** is the production platform tying ADK to enterprise deployment.
4. **A2A** is rapidly becoming an industry standard under Linux Foundation governance.
5. **Context caching** at 90% discount makes Gemini uniquely cost-effective for large-context workflows.

---

## Sources

- [Gemini CLI GitHub Repository](https://github.com/google-gemini/gemini-cli)
- [Gemini CLI Architecture Documentation](https://github.com/google-gemini/gemini-cli/blob/main/docs/architecture.md)
- [Gemini CLI Session Management](https://github.com/google-gemini/gemini-cli/blob/main/docs/cli/session-management.md)
- [Session Management in Gemini CLI -- Google Developers Blog](https://developers.googleblog.com/pick-up-exactly-where-you-left-off-with-session-management-in-gemini-cli/)
- [Introducing Gemini CLI -- Google Blog](https://blog.google/technology/developers/introducing-gemini-cli-open-source-ai-agent/)
- [ADK Overview -- Google Cloud Documentation](https://docs.cloud.google.com/agent-builder/agent-development-kit/overview)
- [ADK: Making it Easy to Build Multi-Agent Applications](https://developers.googleblog.com/en/agent-development-kit-easy-to-build-multi-agent-applications/)
- [ADK for TypeScript](https://developers.googleblog.com/introducing-agent-development-kit-for-typescript-build-ai-agents-with-the-power-of-a-code-first-approach/)
- [ADK for Go](https://developers.googleblog.com/announcing-the-agent-development-kit-for-go-build-powerful-ai-agents-with-your-favorite-languages/)
- [User Simulation in ADK Evaluation](https://developers.googleblog.com/announcing-user-simulation-in-adk-evaluation/)
- [Vertex AI Agent Builder Overview](https://docs.cloud.google.com/agent-builder/overview)
- [Vertex AI Memory Bank](https://docs.cloud.google.com/agent-builder/agent-engine/memory-bank/overview)
- [Announcing A2A Protocol](https://developers.googleblog.com/en/a2a-a-new-era-of-agent-interoperability/)
- [A2A Protocol Upgrade](https://cloud.google.com/blog/products/ai-machine-learning/agent2agent-protocol-is-getting-an-upgrade)
- [A2A Donated to Linux Foundation](https://developers.googleblog.com/en/google-cloud-donates-a2a-to-linux-foundation/)
- [Gemini 2.5 Implicit Caching](https://developers.googleblog.com/en/gemini-2-5-models-now-support-implicit-caching/)
- [Context Caching Overview](https://docs.cloud.google.com/vertex-ai/generative-ai/docs/context-cache/context-cache-overview)
- [Gemini 3 Pro](https://docs.cloud.google.com/vertex-ai/generative-ai/docs/models/gemini/3-pro)
- [Agent Evaluation -- Google Cloud Blog](https://cloud.google.com/blog/topics/developers-practitioners/a-methodical-approach-to-agent-evaluation)
- [AI Agent Trends 2026 Report](https://cloud.google.com/resources/content/ai-agent-trends-2026)
