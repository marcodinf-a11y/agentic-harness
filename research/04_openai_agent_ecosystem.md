# OpenAI Agent Ecosystem: August 2025 -- March 2026

## Table of Contents

1. [Codex CLI](#1-codex-cli)
2. [OpenAI Agents SDK](#2-openai-agents-sdk)
3. [Responses API](#3-responses-api)
4. [Operator / ChatGPT Agent](#4-operator--chatgpt-agent)
5. [Agent Evaluation](#5-agent-evaluation)
6. [Context Management](#6-context-management)
7. [Key Takeaways](#7-key-takeaways)
8. [Sources](#8-sources)

---

## 1. Codex CLI

### Architecture

The Codex platform is built around the **Codex harness** -- the agent loop and logic underlying all Codex experiences. The loop takes user input, prepares textual instructions (a prompt), queries a model for inference, then executes tool calls in a sandbox, repeating until the task is complete.

The **Codex App Server** is the critical architectural component connecting different Codex surfaces (CLI, VS Code extension, web UI). It evolved from a practical need to reuse the Codex harness across products without re-implementing the agent loop. The App Server exposes a client-friendly, bidirectional **JSON-RPC API**.

Core abstractions:
- **Thread**: A conversation between user and the Codex agent, containing turns. Threads can be created, resumed, forked, and archived.
- **Turn**: A single user request and the agent work that follows, containing items with streaming incremental updates.
- **Item**: A unit of input or output.

Thread event history is persisted so clients can reconnect mid-session.

### JSONL Protocol

The transport across all components is **JSON-RPC over stdio, framed as JSONL**. Codex uses a "JSON-RPC lite" variant that keeps the request/response/notification shape but omits the `"jsonrpc": "2.0"` header. This makes it straightforward to build client bindings in Go, Python, TypeScript, Swift, and Kotlin.

Session transcripts are saved locally under `CODEX_HOME` (e.g., `~/.codex/history.jsonl`). Thread rollback drops the last N turns from in-memory context and records a rollback marker in the thread's persisted JSONL log.

### Sandbox

Codex security is built on two layers:

1. **Sandbox mode** (what Codex can do technically)
2. **Approval policy** (when Codex must ask before acting)

Platform-specific sandbox implementations:
- **macOS**: Uses **Seatbelt** policies via `sandbox-exec` with profiles corresponding to the selected sandbox mode.
- **Linux**: Uses **Landlock** + **seccomp** by default. An alternative **bubblewrap** pipeline is available via `features.use_linux_sandbox_bwrap = true`.
- **Windows**: Experimental sandbox support added in codex-cli v0.101.0 (February 2026).

Default behavior: `codex exec` runs in a **read-only sandbox**. The `workspace-write` sandbox provides low-friction local work. In `--full-auto` mode, Codex can read files, make edits, and run commands in the working directory automatically, but asks approval for edits outside the workspace or commands requiring network access.

### Models

By mid-late 2025, **GPT-5.2-Codex** became the default for code generation, review, and repo-scale reasoning. GPT-5.3-Codex has since been introduced. The open-source Codex CLI brought agent-style coding directly into local environments, enabling developers to run Codex over real repositories with human oversight. Codex also supports AGENTS.md files and MCP servers to adapt to repositories and extend with third-party tools.

---

## 2. OpenAI Agents SDK

### Overview and Relationship to Swarm

The Agents SDK is the production-grade successor to **Swarm**, OpenAI's experimental multi-agent framework released in October 2024. Swarm was nascent and exploratory; the Agents SDK is a lightweight yet powerful framework designed for real-world multi-agent workflows.

Available in both **Python** (`pip install openai-agents`, requires Python 3.10+) and **TypeScript/JavaScript**. The SDK is **provider-agnostic**, supporting the OpenAI Responses and Chat Completions APIs, as well as 100+ other LLMs.

### Core Primitives

**Agents** are configured with instructions, tools, guardrails, and handoffs. Each agent encapsulates a specific role or capability.

**Handoffs** enable switching instructions, models, and available tools based on conversation state. A handoff transfers an active conversation from one agent to another -- the receiving agent has complete knowledge of the prior conversation. This is useful for open-ended or conversational workflows where different agents specialize in different domains.

**Agents as Tools** is an alternative pattern where one agent calls another as a tool rather than handing off the full conversation. This produces more transparent, auditable, and scalable multi-agent collaboration.

### Guardrails

Guardrails are first-class concepts in the SDK. The architecture uses **optimistic execution by default**: the primary agent proactively generates outputs while guardrails run concurrently, triggering exceptions if constraints are breached.

Guardrails can be implemented as functions or agents enforcing policies such as:
- Jailbreak prevention
- Relevance validation
- Keyword filtering
- Blocklist enforcement
- Safety classification

For organization-wide governance, the separate **openai-guardrails-python** package provides a drop-in wrapper for OpenAI's Python client, enabling automatic input/output validation and moderation.

### Tracing

Built-in tracing allows monitoring and debugging agent workflows without additional code. The `trace()` function wraps operations under a named trace, linking all spans together. After execution, developers can view the complete trace -- including every LLM call, tool execution, and handoff -- in the **OpenAI Traces Dashboard**.

**Trace grading** assigns structured scores and annotations to specific parts of a trace (decisions, tool calls, reasoning steps) to assess where the agent performed well or made mistakes.

### Additional Features

- **Human in the loop**: Mechanisms for requiring human approval at key decision points.
- **Sessions**: Automatic conversation history management.
- **Realtime agents**: Support for building voice agents.
- **MCP integration**: Support for Model Context Protocol servers as tool sources.

---

## 3. Responses API

### Overview

The Responses API, released in **March 2025**, is OpenAI's new API primitive -- an evolution of Chat Completions that brings added simplicity and powerful agentic primitives. It replaces both Chat Completions (for new agent-centric use cases) and the **Assistants API**, which was deprecated on August 26, 2025 with a sunset date of August 26, 2026.

### Built-in Tools

The Responses API includes several built-in tools:

- **Web Search**: Models search the web for up-to-date information, providing answers with sourced citations.
- **File Search**: Models search contents of uploaded files for context.
- **Computer Use**: Enables agentic workflows where a model controls a computer interface.
- **Code Interpreter**: Executes code in a sandboxed environment.
- **Remote MCP**: Gives models access to additional tools via remote Model Context Protocol servers, using the open protocol that standardizes how applications provide context to LLMs.

### Tool Use with Reasoning Models

**o3** and **o4-mini** can call tools and functions directly within their chain-of-thought in the Responses API. This produces answers that are more contextually rich and relevant. Using these models with the Responses API **preserves reasoning tokens across requests and tool calls**, improving model intelligence and reducing cost and latency.

### Migration Path

OpenAI provides a migration guide from both Chat Completions and Assistants API to the Responses API. The Responses API supports multiple inputs and outputs across different modalities, making it the foundation for agent-native development.

---

## 4. Operator / ChatGPT Agent

### Operator (January 2025)

OpenAI shipped **Operator** in January 2025 as a research preview for its **Computer Using Agent (CUA)** model. Operator is an agent that uses its own browser to perform tasks for users -- filling out forms, ordering groceries, creating memes, and other repetitive browser tasks.

CUA combines GPT-4o's vision capabilities with advanced reasoning through reinforcement learning. It interacts with graphical user interfaces (GUIs) -- buttons, menus, text fields -- just as humans do.

Benchmark performance:
- **38.1%** success rate on OSWorld (full computer use)
- **58.1%** on WebArena (web-based tasks)
- **87%** on WebVoyager (web-based tasks)

### ChatGPT Agent (July 2025)

As of **July 17, 2025**, Operator was fully integrated into ChatGPT as **ChatGPT agent**. The standalone Operator experience at operator.chatgpt.com was deprecated. ChatGPT agent is equipped with:

- A **visual browser** that interacts with the web through a GUI
- A **text-based browser** for simpler reasoning-based web queries
- A **terminal**
- **Direct API access**

Available to Pro, Plus, and Team users via the tools dropdown from the composer by selecting "agent mode."

---

## 5. Agent Evaluation

### Evals Framework

OpenAI's evaluation ecosystem matured significantly in 2025, evolving into a repeatable "measure, improve, ship" loop. Key components:

- **Evals API**: Introduced for eval-driven development, supporting programmable graders and reinforcement fine-tuning (RFT).
- **Datasets**: Allow rapid construction of agent evals from scratch, expandable over time with automated graders and human annotations.
- **Graders**: Various types of graders for scoring agent outputs.

### Trace Grading

Trace grading assigns structured scores or labels to an agent's trace -- the end-to-end log of decisions, tool calls, and reasoning steps. Trace evals use graded traces to systematically evaluate agent performance across many examples, helping benchmark changes, identify regressions, and validate improvements.

### Benchmarks

- **GDPval**: A new evaluation measuring model performance on economically valuable, real-world tasks across 44 occupations. GPT-5.2 Thinking is the first model performing at or above human expert level, beating or tying top professionals on 70.9% of comparisons.
- **Terminal-Bench 2.0**: Tests AI agents in real terminal environments with tasks including compiling code, training models, and setting up servers.
- **Safety Evaluations Hub**: Updated in August 2025 to include results for GPT-5 and gpt-oss models, featuring new production benchmarks.

### AgentKit

OpenAI introduced **AgentKit** as a framework for standardizing agent evaluation and testing.

---

## 6. Context Management

### Context Window Growth

- **GPT-5**: Up to 272k input tokens, 128k output tokens
- **GPT-5.2**: 400,000 token context window, 128k max output tokens
- **GPT-5.4**: **1M token context window** -- the first mainline model enabling analysis of entire codebases, long document collections, or extended agent trajectories in a single request

### Context Engineering Strategies

**Summarization**: Even large context windows can be overwhelmed by uncurated histories, redundant tool results, or noisy retrievals. Context summarization compresses prior messages (assistant, user, tools) into structured, shorter summaries injected into conversation history.

**Compaction**: GPT-5.4 is the first mainline model trained to support **compaction**, enabling longer agent trajectories while preserving key context. The Responses API provides a `/responses/compact` endpoint to shrink context between turns -- stateless, returning a compacted window for the next call.

**Conversations API**: Provides durable threads and replayable state for managing conversation state and persistence.

**Sessions (Agents SDK)**: Short-term memory management through the SDK's session primitives.

**Long-term Memory**: Context engineering for personalization through state management with long-term memory notes.

**Connectors and MCP Servers**: Incorporating external context from third-party sources into agent workflows.

---

## 7. Key Takeaways

1. **Agent-native API shift**: The Responses API is now the foundation, replacing both Chat Completions (for agentic use) and the Assistants API. Built-in tools (web search, file search, computer use, MCP) make it a one-stop surface for agent development.

2. **Codex as a coding surface, not just a model**: Codex evolved from a code generation model to a full agent platform with the App Server architecture, JSONL-over-stdio protocol, OS-level sandboxing, and multi-surface support (CLI, IDE, web).

3. **Multi-agent orchestration is standardized**: The Agents SDK provides production-grade primitives (handoffs, guardrails, tracing) for multi-agent workflows, available in both Python and TypeScript, and is provider-agnostic.

4. **Sandbox security is layered**: Codex implements OS-enforced sandboxing (Landlock/seatbelt/bubblewrap) combined with approval policies, giving developers control over what automated agents can do.

5. **Evaluation is now trace-centric**: Agent evaluation moved beyond simple input/output testing to trace grading -- scoring the full end-to-end log of decisions, tool calls, and reasoning steps.

6. **Context windows reached 1M tokens**: With GPT-5.4, and compaction support makes long-running agent trajectories practical without context overflow.

7. **Operator merged into ChatGPT**: The standalone browser agent became "ChatGPT agent" with visual browser, text browser, terminal, and API access integrated into the main product.

8. **Guardrails are first-class**: Both at the SDK level (optimistic execution with concurrent guardrail checks) and organization level (openai-guardrails-python), safety is built into the agent development workflow rather than bolted on.

---

## 8. Sources

- [Unlocking the Codex harness: how we built the App Server](https://openai.com/index/unlocking-the-codex-harness/)
- [Unrolling the Codex agent loop](https://openai.com/index/unrolling-the-codex-agent-loop/)
- [Harness engineering: leveraging Codex in an agent-first world](https://openai.com/index/harness-engineering/)
- [Codex CLI features](https://developers.openai.com/codex/cli/features/)
- [Codex CLI reference](https://developers.openai.com/codex/cli/reference/)
- [Codex App Server](https://developers.openai.com/codex/app-server/)
- [Codex Security](https://developers.openai.com/codex/security/)
- [Codex sandbox documentation (GitHub)](https://github.com/openai/codex/blob/main/docs/sandbox.md)
- [Codex Configuration Reference](https://developers.openai.com/codex/config-reference/)
- [Codex changelog](https://developers.openai.com/codex/changelog/)
- [OpenAI Agents SDK - Python (GitHub)](https://github.com/openai/openai-agents-python)
- [OpenAI Agents SDK - JavaScript/TypeScript (GitHub)](https://github.com/openai/openai-agents-js)
- [New tools for building agents](https://openai.com/index/new-tools-for-building-agents/)
- [Building agents](https://developers.openai.com/tracks/building-agents/)
- [Use Codex with the Agents SDK](https://developers.openai.com/codex/guides/agents-sdk)
- [Agents guide](https://platform.openai.com/docs/guides/agents)
- [Safety in building agents](https://platform.openai.com/docs/guides/agent-builder-safety)
- [OpenAI Guardrails Python (GitHub)](https://github.com/openai/openai-guardrails-python)
- [New tools and features in the Responses API](https://openai.com/index/new-tools-and-features-in-the-responses-api/)
- [Responses API Reference](https://platform.openai.com/docs/api-reference/responses)
- [Migrate to the Responses API](https://platform.openai.com/docs/guides/migrate-to-responses)
- [Using tools](https://developers.openai.com/api/docs/guides/tools/)
- [Web search tool](https://developers.openai.com/api/docs/guides/tools-web-search/)
- [Introducing Operator](https://openai.com/index/introducing-operator/)
- [Computer-Using Agent](https://openai.com/index/computer-using-agent/)
- [Introducing ChatGPT agent](https://openai.com/index/introducing-chatgpt-agent/)
- [ChatGPT agent System Card](https://openai.com/index/chatgpt-agent-system-card/)
- [Operator System Card](https://openai.com/index/operator-system-card/)
- [Agent evals](https://developers.openai.com/api/docs/guides/agent-evals/)
- [Trace grading](https://developers.openai.com/api/docs/guides/trace-grading/)
- [Working with evals](https://developers.openai.com/api/docs/guides/evals/)
- [Testing Agent Skills Systematically with Evals](https://developers.openai.com/blog/eval-skills/)
- [Introducing AgentKit](https://openai.com/index/introducing-agentkit/)
- [Measuring the performance of our models on real-world tasks (GDPval)](https://openai.com/index/gdpval/)
- [Safety evaluations hub](https://openai.com/safety/evaluations-hub/)
- [OpenAI for Developers in 2025](https://developers.openai.com/blog/openai-for-developers-2025/)
- [Introducing GPT-5.2-Codex](https://openai.com/index/introducing-gpt-5-2-codex/)
- [Introducing GPT-5.3-Codex](https://openai.com/index/introducing-gpt-5-3-codex/)
- [Context Engineering - Session Memory (Agents SDK)](https://developers.openai.com/cookbook/examples/agents_sdk/session_memory/)
- [Context Engineering for Personalization](https://developers.openai.com/cookbook/examples/agents_sdk/context_personalization/)
- [Conversation state](https://platform.openai.com/docs/guides/conversation-state)
- [GPT-5.2 Model](https://developers.openai.com/api/docs/models/gpt-5.2)
- [Using GPT-5.4](https://developers.openai.com/api/docs/guides/latest-model/)
- [Orchestrating Agents: Routines and Handoffs (Cookbook)](https://cookbook.openai.com/examples/orchestrating_agents)
