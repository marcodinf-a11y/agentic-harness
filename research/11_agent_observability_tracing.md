# Agent Observability, Tracing, and Monitoring (Aug 2025 -- Mar 2026)

## 1. Executive Summary

Agent observability has matured rapidly. Key trends: OpenTelemetry as universal standard for GenAI, built-in tracing in agent SDKs (OpenAI Agents SDK), cost attribution as business-critical capability, and enterprise vendors (Datadog, Dynatrace) entering the space.

---

## 2. Platform Comparison

| Platform | Type | Open Source | OTel | Cost Tracking | Self-Host | Pricing |
|---|---|---|---|---|---|---|
| **LangSmith** | SaaS | No | Export | Auto + custom | Enterprise | $0.50-5/1k traces |
| **Arize Phoenix** | Self-hosted/Cloud | Yes (Apache 2.0) | Native OTLP | Yes | Yes (free) | Free (all features) |
| **W&B Weave** | SaaS | SDK OSS | Export | Auto | No | W&B tiers |
| **Langfuse** | SaaS + Self-host | Yes (Apache 2.0) | Native OTel | Auto + model | Yes (free) | Free self-host |
| **Braintrust** | SaaS | No | OpenLLMetry | Auto | No | Free 1M spans; $249/mo |
| **OpenAI Agents SDK** | SDK (built-in) | Yes | Export | Via exporters | N/A | Free |
| **Helicone** | Proxy/Gateway | Yes | No | Auto (proxy) | Yes | Free tier |
| **AgentOps** | SaaS | SDK OSS | No | Auto | No | Free tier |
| **Datadog** | SaaS | No | Native OTel GenAI | Infrastructure | No | Enterprise |

---

## 3. Key Platforms

### LangSmith
Full-stack tracing for agent workflows. Multi-framework support. AI-powered debugging. Automatic cost tracking.

### Arize Phoenix
Fully open-source, vendor-agnostic. Accepts OTLP traces. Supports OpenAI Agents SDK, LangGraph, CrewAI, Google ADK, and more. All features free.

### Langfuse
Open-source LLM engineering platform. Native OpenTelemetry. Prompt management with version control. LLM-as-a-judge evaluation with execution tracing.

### OpenAI Agents SDK (Built-in Tracing)
First-class built-in feature. Comprehensive events: LLM generations, tool calls, handoffs, guardrails. Export to third-party platforms.

---

## 4. OpenTelemetry GenAI Semantic Conventions

The OTel GenAI SIG is defining standardized schemas: GenAI Client Spans, Agent Spans (Development status), Events, and Metrics. Agentic systems proposal covers tasks, actions, agents, teams, artifacts, memory. Status: Development (not yet stable).

---

## 5. Key Trends

1. **OpenTelemetry convergence**: GenAI conventions becoming lingua franca
2. **Built-in tracing**: Agent frameworks shipping with observability
3. **Cost as first-class metric**: Token-level cost attribution now standard
4. **Agent-specific debugging**: Session replay and time-travel debugging
5. **Enterprise entry**: Datadog and Dynatrace expanding LLM observability
6. **Open source maturity**: Phoenix and Langfuse offer feature parity with commercial tools

---

## Sources

- [LangSmith Observability](https://www.langchain.com/langsmith/observability)
- [Arize Phoenix](https://phoenix.arize.com/) / [GitHub](https://github.com/Arize-ai/phoenix)
- [W&B Weave](https://docs.wandb.ai/weave)
- [Langfuse](https://langfuse.com/docs/observability/overview) / [GitHub](https://github.com/langfuse/langfuse)
- [Braintrust](https://www.braintrust.dev)
- [OpenAI Agents SDK Tracing](https://openai.github.io/openai-agents-python/tracing/)
- [Helicone](https://www.helicone.ai/)
- [AgentOps](https://www.agentops.ai/)
- [Datadog LLM Observability](https://www.datadoghq.com/product/llm-observability/)
- [OpenTelemetry GenAI Conventions](https://opentelemetry.io/docs/specs/semconv/gen-ai/)
- [OpenLLMetry](https://github.com/traceloop/openllmetry)
