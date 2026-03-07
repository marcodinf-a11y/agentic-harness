# Multi-Agent Orchestration Patterns: Research Report (Aug 2025 -- Mar 2026)

## 1. Executive Summary

Multi-agent orchestration has become the dominant paradigm in production AI systems. According to Deloitte, 72% of enterprise AI projects now involve multi-agent architectures, up from 23% in 2024. The period from August 2025 through March 2026 has seen the maturation of design patterns, the standardization of inter-agent protocols (Google A2A, Anthropic MCP), and the emergence of production-grade frameworks.

---

## 2. Core Design Patterns

### 2.1 Supervisor (Hierarchical)

Central orchestrator decomposes tasks, delegates to specialists, validates outputs. Anthropic's multi-agent research system uses this with Claude Opus 4 as lead + Sonnet 4 subagents, outperforming single-agent by 90.2%.

### 2.2 Swarm (Decentralized)

Peer agents with defined roles share information through memory/messages. Coordination is emergent. LangGraph Multi-Agent Swarm represents agents as nodes in a directed state graph.

### 2.3 Pipeline (Sequential)

Fixed chain where output of step A feeds step B. Easiest to test and monitor; cost predictable; rigid.

### 2.4 Debate (Multi-Agent Deliberation)

Multiple agents generate answers independently, then refine through rounds of mutual review. A-HMAD achieves 4-6% accuracy gains. However, ICLR 2025 shows majority voting alone captures most gains.

### 2.5 Ensemble (Parallel + Aggregation)

Multiple agents process same input simultaneously using different strategies. Results aggregated through voting, ranking, or meta-agent.

### 2.6 Evaluator-Optimizer

Generator + Evaluator loop. Repeats until quality thresholds met. RADAR assigns specialized roles for safety evaluation.

---

## 3. Agent-to-Agent Communication

| Mechanism | Description | Used By |
|---|---|---|
| Direct message passing | Structured messages to specific peers | OpenAI Agents SDK, CrewAI |
| Shared state / blackboard | Common state object | LangGraph, AutoGen |
| Protocol-based (A2A) | JSON-RPC over HTTP with Agent Cards | Google A2A |
| Tool-mediated | Agent A invokes Agent B as a tool | Anthropic, OpenAI |

### MCP + A2A Complementarity

- **MCP**: Agent-to-world (tools, APIs, data sources)
- **A2A**: Agent-to-agent (communication, delegation)

Both under Linux Foundation governance. Joint interoperability spec expected Q3 2026.

---

## 4. Task Decomposition

- **Static**: Predefined task graphs (pipeline pattern)
- **Dynamic**: Orchestrator LLM determines subtasks at runtime
- **Recursive**: Subtasks spawn further subtasks (tree of agents)

---

## 5. Handoff Protocols

- **Agentic handoff**: Full message history transferred (OpenAI Agents SDK)
- **Programmatic handoff**: Only structured data passed
- **Protocol-based**: A2A task objects with lifecycle tracking

---

## 6. Shared Memory

Hybrid pattern most common: orchestrator maintains "team memory" (goals, progress) while specialists record task-specific details.

---

## 7. Anthropic Multi-Agent Patterns

Official patterns: prompt chaining, routing, parallelization, orchestrator-workers, evaluator-optimizer.

Claude Code Agent Teams (Feb 2026): Lead agent + subagents with independent context windows, shared task list, peer-to-peer messaging.

---

## 8. Key Trends

1. Protocol convergence (MCP + A2A under Linux Foundation)
2. Multi-model agent teams (Opus as orchestrator, Sonnet as worker)
3. Multi-agent systems consume 10-15x more tokens than single-agent
4. Debate pattern limitations confirmed (majority voting captures most gains)

---

## Sources

- [Anthropic -- How we built our multi-agent research system](https://www.anthropic.com/engineering/multi-agent-research-system)
- [Anthropic -- Building Effective AI Agents](https://www.anthropic.com/research/building-effective-agents)
- [A2A Protocol](https://developers.googleblog.com/en/a2a-a-new-era-of-agent-interoperability/)
- [OpenAI Agents SDK](https://openai.github.io/openai-agents-python/)
- [LangGraph Multi-Agent Swarm](https://www.marktechpost.com/2025/05/15/meet-langgraph-multi-agent-swarm/)
- [Multi-LLM-Agents Debate -- ICLR 2025](https://d2jud02ci9yv69.cloudfront.net/2025-04-28-mad-159/blog/mad/)
- [Deloitte -- AI Agent Orchestration](https://www.deloitte.com/us/en/insights/industry/technology/technology-media-and-telecom-predictions/2026/ai-agent-orchestration.html)
- [AgentMesh -- arXiv:2507.19902](https://arxiv.org/html/2507.19902v1)
- [Understanding and Bridging the Planner-Coder Gap -- arXiv:2510.10460](https://arxiv.org/html/2510.10460)
