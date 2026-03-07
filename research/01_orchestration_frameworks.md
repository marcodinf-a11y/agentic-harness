# AI Agent Orchestration Frameworks Research (Aug 2025 - Mar 2026)

## 1. LangChain / LangGraph

### What It Does
LangGraph is LangChain's dedicated agent runtime and low-level orchestration framework for building agents that reliably handle complex tasks. As of 2025, LangChain's team recommends LangGraph for all new agent implementations. LangGraph provides full authorship over the cognitive architecture, giving developers control over workflow and information flow.

### Architecture
- **Graph-based execution model**: Agents are defined as nodes in a stateful graph with edges representing transitions.
- **Durable execution**: Agents persist through failures and support long-running workflows.
- **Pre-built architectures**: LangGraph Pre-Builts supports common patterns like Swarm, Supervisor, and tool-calling agent with minimal configuration code.
- **LangGraph Platform**: Matured through 2025-2026 as a deployment and monitoring layer for long-running agents.

### Context / Memory Handling
- **Short-term memory**: Thread-scoped, persisted via checkpoints. Each graph invocation uses a `thread_id` for conversation isolation.
- **Long-term memory**: Cross-session, user-specific or application-level data stored in a separate memory store.
- **Checkpointers**: Pluggable persistence backends — `InMemorySaver`, `SqliteSaver`, `RedisSaver`, `AsyncRedisSaver`. State is saved at every superstep.
- **Optimization**: Message trimming (`trim_messages`), summarization nodes, and entity extraction for focused recall.

### Multi-Agent Support
- Supervisor pattern (orchestrator agent delegates to specialists).
- Swarm pattern (peer agents hand off to each other).
- Hierarchical composition of sub-graphs.
- MCP (Model Context Protocol) integration for cross-runtime tool access.

### Evaluation Capabilities
- **LangSmith**: Full trajectory evaluation capturing steps, tool calls, and reasoning. Supports human evaluation, heuristic checks, LLM-as-judge, pairwise comparisons, and custom evaluators.
- **Multi-turn Evals**: Evaluate entire interactions, not just individual traces.
- **Insights Agent**: Automatically categorizes agent usage patterns and scores conversations in production.
- **Open-source eval catalog**: Ready-made evaluators for code, extraction, RAG, and agent trajectory testing.
- **Agent-specific observability metrics**: Tool calling frequency, trajectory tracking, common path analysis.

### Latest Changes (Aug 2025 - Mar 2026)
- Open Agent Platform: no-code agent builder with MCP tools, prompt customization, and model selection.
- LangGraph Studio v2: runs locally without a desktop app; integrates traces, datasets, and prompt editing.
- LangGraph Pre-Builts for rapid prototyping.
- Polly AI assistant and `langsmith-fetch` CLI for debugging deep agents.

---

## 2. CrewAI

### What It Does
CrewAI is a standalone, high-performance multi-agent framework for orchestrating role-playing, autonomous AI agents. It assigns distinct roles to individual agents, creating specialized teams that mimic real-world organizational structures. Completely independent from LangChain.

### Architecture
- **Dual architecture — Crews and Flows**:
  - **Crews**: Autonomous teams where agents have true agency — they decide when to delegate, ask questions, and approach tasks. Includes a hierarchical process mode that auto-generates a manager agent for task delegation.
  - **Flows**: Event-driven pipelines for production workloads requiring predictability. Granular, event-driven control with native Crew support.
- **Role-based design**: Each agent operates within a defined area of expertise with unique capabilities and decision-making processes.

### Context / Memory Handling
- **Four memory types**: Short-term, long-term, entity memory, and contextual memory.
- **Scoped memory**: Context-dependent memory with tree-structured scopes for precision and performance.
- **Automatic extraction**: After each task, discrete facts are extracted from output and stored. Before each task, relevant context is recalled and injected into the prompt.
- **Shared memory**: All agents in a crew share the crew's memory unless an agent has its own override.
- **Known limitations**: Context relevance is a non-trivial RAG problem; long-term memory can grow unwieldy.

### Multi-Agent Support
- Sequential and hierarchical process modes.
- Agent delegation within crews.
- Manager agent auto-generation in hierarchical mode.
- Flows enable composing multiple crews into production pipelines.

### Evaluation Capabilities
- Built-in task output validation.
- Integration with external evaluation platforms.
- Enhanced EventListener and TraceCollectionListener for monitoring (added Jan 2026).

### Latest Changes (Aug 2025 - Mar 2026)
- January 2026: Production-ready Flows and Crews architecture, streaming tool call events, improved EventListener/TraceCollectionListener, Human-In-The-Loop for Flows.
- Community growth: 44,600+ GitHub stars, 450M monthly workflows, 100,000+ certified developers.

---

## 3. AutoGen / AG2 -> Microsoft Agent Framework

### What It Does
AutoGen and Semantic Kernel have been placed into maintenance mode (bug fixes and security patches only) and consolidated into the **Microsoft Agent Framework**, which reached public preview on October 1, 2025. The new framework combines AutoGen's simple agent abstractions with Semantic Kernel's enterprise features.

### Architecture
- **Unified SDK**: Merges AutoGen's dynamic multi-agent orchestration with Semantic Kernel's production foundations.
- **Graph-based workflows**: Explicit multi-agent orchestration through graph-defined agent interactions.
- **Plugin architecture**: Semantic Kernel's function-calling plugin system for binding tools into agent policies.
- **Four pillars**: Open Standards (MCP, A2A, OpenAPI), enterprise-ready stability, multi-language support (.NET, Python), and responsible AI features.
- **Model flexibility**: Supports Azure OpenAI, OpenAI, local runtimes, and GitHub Models.

### Context / Memory Handling
- **Session-based state management** with type safety and middleware support.
- **Pluggable memory options**: Redis, Pinecone, Azure AI Search.
- **Persistent memory** for long-running workflows (requires careful architecture planning).
- **Shared memory systems**: Experimental support for agent-to-agent shared memory.

### Multi-Agent Support
- Graph-based multi-agent orchestration patterns inherited from AutoGen.
- Agent-to-Agent (A2A) messaging protocol support.
- Experimental agent-to-agent communication protocols.
- Deep Microsoft 365 integration for enterprise multi-agent scenarios.

### Evaluation Capabilities
- **Azure AI Foundry**: Comprehensive observability via Foundry Control Plane for evaluating, monitoring, and optimizing quality, performance, and safety.
- Built-in telemetry.
- **Responsible AI features** (public preview): Task adherence, prompt shields with spotlighting, PII detection.

### Latest Changes (Aug 2025 - Mar 2026)
- October 2025: Public preview of Microsoft Agent Framework.
- AutoGen and Semantic Kernel moved to maintenance mode.
- Agent Framework 1.0 GA targeted for end of Q1 2026.
- Process Framework GA planned for Q2 2026 (deterministic business workflow orchestration).
- Semantic Kernel surpassed 27,207 GitHub stars with 300+ contributors.

---

## 4. OpenAI Agents SDK

### What It Does
Released March 2025 as the production-ready evolution of OpenAI's experimental Swarm project. A lightweight, minimalist framework for building multi-agent workflows using a small set of core primitives. Provider-agnostic despite the name, supporting 100+ LLMs.

### Architecture
- **Core primitives**: Agents (LLMs with instructions and tools), Handoffs (conversation transfer between agents), Guardrails (input/output validation), Runner (execution engine), and Tracing.
- **Handoff model**: Unlike agent-as-tool, handoffs transfer the entire conversation to another agent with full history access.
- **Tool varieties**: Function tools (decorated Python functions), Hosted tools (web search, file search, code interpreter), Agent-as-tool (hierarchical invocation).
- **Runner**: Drives the agentic loop — sends messages, processes tool calls, executes handoffs, runs guardrails, iterates until final response.

### Context / Memory Handling
- Conversation history passed through handoffs.
- Automatic tracing captures full execution flow: LLM calls, tool invocations, handoffs, guardrail checks.
- Integration with OpenAI's platform dashboard for trace inspection.
- Custom trace processors for export to external systems.

### Multi-Agent Support
- **Handoffs**: Declarative agent-to-agent conversation transfer for triage, escalation, and specialist routing.
- **Agent-as-tool**: Hierarchical architecture where parent agents invoke child agents for sub-tasks.
- Supports both patterns simultaneously.

### Evaluation Capabilities
- Automatic trace generation for every agent run.
- Integration with OpenAI platform dashboard.
- Custom trace processors for external evaluation systems.

### Latest Changes (Aug 2025 - Mar 2026)
- TypeScript/JavaScript SDK (`openai-agents-js`) added.
- Voice agent support via Realtime API integration.
- Enhanced MCP integration.
- AgentKit introduced by OpenAI as a complementary tool.

---

## 5. Google Agent Development Kit (ADK)

### What It Does
Google's open-source, code-first framework for building, evaluating, and deploying AI agents. Optimized for Gemini and Google Cloud but model-agnostic and deployment-agnostic. Designed to make agent development feel like software development.

### Architecture
- **Hierarchical agent composition**: Specialized agents composed in a hierarchy.
- **Workflow agents**: Sequential, Parallel, and Loop agents for predictable pipelines.
- **LLM-driven routing**: Dynamic `LlmAgent` transfer for adaptive behavior.
- **Multi-language**: Python, TypeScript/JavaScript, and Go SDKs.
- **Tool ecosystem**: Pre-built tools (Search, Code Exec), MCP tools, third-party library integration (LangChain, LlamaIndex), and agent-as-tool (LangGraph, CrewAI).
- **Multimodal**: Built-in bidirectional audio and video streaming.
- **A2A protocol**: Agent-to-Agent delegation across local and remote deployments.

### Context / Memory Handling
- **Session**: Every interaction gets a session managed by `SessionService`, containing session ID, user ID, event history, and state.
- **State with magic prefixes**: Default state is session-scoped; `user:` prefix persists across all user sessions; `app:` prefix persists globally.
- **Storage backends**: `InMemorySessionService` (dev), `DatabaseSessionService` (SQL — SQLite, MySQL, PostgreSQL) for production.
- **Memory services**: `InMemoryMemoryService` (keyword matching, prototyping) and production-grade options.

### Multi-Agent Support
- Hierarchical agent composition.
- Workflow agents for deterministic orchestration.
- A2A protocol for cross-agent delegation (local and remote).
- Agent-as-tool pattern.

### Evaluation Capabilities
- Built-in development UI (`ADK web`) for inspecting events, traces, and artifacts.
- Integration with Vertex AI Agent Builder for evaluation.
- Event and trace inspection in the runtime.

### Latest Changes (Aug 2025 - Mar 2026)
- TypeScript SDK release.
- Go SDK release (November 2025).
- MCP Toolbox for Databases with native ADK TypeScript integration.
- A2A protocol support for secure, opaque agent interactions.

---

## 6. Other Notable Frameworks

| Framework | Description |
|---|---|
| **Pydantic AI** | Type-safe Python agent framework from the Pydantic team. V1 released September 2025. Durable execution, MCP/A2A support, graph-based workflows. Version jumped to 1.66.0 by March 2026. |
| **LlamaIndex** | Introduced Agentic Document Workflows in 2025, combining document processing, retrieval, structured outputs, and agentic orchestration for knowledge work automation. |
| **Langflow** | Low-code AI builder for agentic and RAG applications; visual interface approach. |
| **Mem0 / Zep / LangMem** | Emerging memory-focused platforms (2025) with hybrid architectures addressing long-term agent memory challenges. |

---

## Comparison Table

| Dimension | LangGraph | CrewAI | Microsoft Agent Framework | OpenAI Agents SDK | Google ADK |
|---|---|---|---|---|---|
| **Release / Status** | Production (GA) | Production (GA) | Public Preview (Oct 2025); 1.0 GA target Q1 2026 | Production (Mar 2025+) | Production (GA) |
| **Primary Language** | Python, JS/TS | Python | Python, .NET | Python, JS/TS | Python, TS/JS, Go |
| **Architecture Model** | Stateful graph | Crews + Flows (role-based) | Graph-based + Plugin architecture | Primitives (Agent, Handoff, Guardrail, Runner) | Hierarchical agents + Workflow agents |
| **Multi-Agent Pattern** | Supervisor, Swarm, sub-graphs | Sequential, Hierarchical, Flows | Graph orchestration, A2A protocol | Handoffs, Agent-as-tool | Hierarchical, Sequential/Parallel/Loop, A2A |
| **Memory / State** | Checkpointers (thread + long-term), pluggable backends | 4 memory types (short/long/entity/contextual), scoped | Pluggable (Redis, Pinecone, Azure AI Search), session-based | Conversation history via handoffs, tracing | Session + State (magic prefixes), SQL backends |
| **Evaluation** | LangSmith (trajectory, multi-turn, LLM-as-judge, open-source evals) | EventListener, TraceCollectionListener | Azure AI Foundry observability, Responsible AI features | Auto-tracing, platform dashboard | Built-in dev UI, Vertex AI integration |
| **Human-in-the-Loop** | Yes (native) | Yes (added Jan 2026 for Flows) | Yes | Yes (Guardrails) | Yes |
| **MCP Support** | Yes | Via tools | Yes | Yes | Yes |
| **Model Agnostic** | Yes | Yes | Yes (Azure OpenAI, OpenAI, local, GitHub Models) | Yes (100+ LLMs) | Yes (optimized for Gemini) |
| **Complexity** | Medium-High (flexible, low-level) | Low-Medium (opinionated, role-based) | Medium-High (enterprise, plugin-based) | Low (minimalist, 4 primitives) | Medium (code-first, multi-pattern) |
| **Best For** | Complex custom workflows, production agents | Team-based task automation, rapid prototyping | Enterprise .NET/Azure ecosystems | Simple-to-moderate agent systems, OpenAI-centric stacks | Google Cloud / Gemini ecosystems, polyglot teams |

---

## Key Industry Trends (Aug 2025 - Mar 2026)

1. **Framework consolidation**: Microsoft merged AutoGen + Semantic Kernel into a single Agent Framework. The industry is moving from experimentation frameworks to production-grade platforms.
2. **Protocol standardization**: MCP (Model Context Protocol) and A2A (Agent-to-Agent) are becoming standard interoperability layers across all major frameworks.
3. **Multi-language support**: All major frameworks expanded beyond Python — TypeScript/JS is now universally supported; Go is emerging (Google ADK).
4. **Human-in-the-loop as standard**: Every major framework now supports HITL workflows natively.
5. **Evaluation maturity**: LangSmith leads with trajectory and multi-turn evals; other frameworks are catching up with observability and tracing integrations.
6. **Memory specialization**: Dedicated memory platforms (Mem0, Zep, LangMem) are emerging alongside framework-native memory systems.

---

## Sources

- [LangGraph: Agent Orchestration Framework](https://www.langchain.com/langgraph)
- [LangGraph Overview — LangChain Docs](https://docs.langchain.com/oss/python/langgraph/overview)
- [Recap of Interrupt 2025 — LangChain Blog](https://blog.langchain.com/interrupt-2025-recap/)
- [LangGraph Memory Overview — LangChain Docs](https://docs.langchain.com/oss/python/langgraph/memory)
- [LangSmith Evaluation Platform](https://www.langchain.com/langsmith/evaluation)
- [Improve Agent Quality with Insights Agent and Multi-turn Evals — LangChain Blog](https://www.blog.langchain.com/insights-agent-multiturn-evals-langsmith/)
- [Agent Evals — GitHub](https://github.com/langchain-ai/agentevals)
- [CrewAI Changelog](https://docs.crewai.com/en/changelog)
- [CrewAI GitHub](https://github.com/crewAIInc/crewAI)
- [CrewAI Memory — Docs](https://docs.crewai.com/en/concepts/memory)
- [CrewAI — AWS Prescriptive Guidance](https://docs.aws.amazon.com/prescriptive-guidance/latest/agentic-ai-frameworks/crewai.html)
- [Microsoft Agent Framework Overview — Microsoft Learn](https://learn.microsoft.com/en-us/agent-framework/overview/)
- [Introducing Microsoft Agent Framework — Azure Blog](https://azure.microsoft.com/en-us/blog/introducing-microsoft-agent-framework/)
- [AutoGen to Microsoft Agent Framework Migration Guide](https://learn.microsoft.com/en-us/agent-framework/migration-guide/from-autogen/)
- [Semantic Kernel + AutoGen = Microsoft Agent Framework — Visual Studio Magazine](https://visualstudiomagazine.com/articles/2025/10/01/semantic-kernel-autogen--open-source-microsoft-agent-framework.aspx)
- [Semantic Kernel Agent Orchestration — Microsoft Learn](https://learn.microsoft.com/en-us/semantic-kernel/frameworks/agent/agent-orchestration/)
- [OpenAI Agents SDK — Docs](https://openai.github.io/openai-agents-python/)
- [New Tools for Building Agents — OpenAI](https://openai.com/index/new-tools-for-building-agents/)
- [OpenAI Agents SDK — GitHub](https://github.com/openai/openai-agents-python)
- [Introducing AgentKit — OpenAI](https://openai.com/index/introducing-agentkit/)
- [Google ADK — Docs](https://google.github.io/adk-docs/)
- [ADK Agent State and Memory — Google Cloud Blog](https://cloud.google.com/blog/topics/developers-practitioners/remember-this-agent-state-and-memory-with-adk)
- [ADK Sessions Introduction](https://google.github.io/adk-docs/sessions/)
- [Google ADK for TypeScript — Google Developers Blog](https://developers.googleblog.com/introducing-agent-development-kit-for-typescript-build-ai-agents-with-the-power-of-a-code-first-approach/)
- [Google ADK for Go — Google Developers Blog](https://developers.googleblog.com/announcing-the-agent-development-kit-for-go-build-powerful-ai-agents-with-your-favorite-languages/)
- [ADK Python — GitHub](https://github.com/google/adk-python)
- [Pydantic AI Documentation](https://ai.pydantic.dev/)
- [AI Agent Orchestration — Deloitte](https://www.deloitte.com/us/en/insights/industry/technology/technology-media-and-telecom-predictions/2026/ai-agent-orchestration.html)
