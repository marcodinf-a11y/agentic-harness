# Token Management, Cost Optimization, and Context Budget Strategies for AI Agents

**Research Period: August 2025 -- March 2026**

---

## 1. Prompt Caching Across Providers

### Cross-Provider Comparison

| Feature | Anthropic | OpenAI | Google Gemini |
|---|---|---|---|
| Activation | Manual breakpoints + auto | Automatic (>= 1024 tokens) | Implicit auto + Explicit manual |
| Read discount | 90% | 50% | 75-90% |
| Write cost | 1.25x-2x | Free | Varies by TTL |
| TTL options | 5 min / 1 hour | 5-10 min (auto) | Configurable |
| Explicit control | Yes (breakpoints) | No | Yes (named caches) |

Key developments:
- Anthropic (Feb 2026): Workspace-level cache isolation; cache reads no longer count against ITPM rate limits
- OpenAI: Fully automatic, zero configuration
- Google (Gemini 2.5+): Implicit caching enabled by default, 90% discount

---

## 2. Context Optimization Strategies

### 2.1 Compaction

**Factory.ai's Anchored Iterative Summarization** (Dec 2025): Structured persistent summary with explicit sections. Scored 4.04 accuracy vs Anthropic 3.74 and OpenAI 3.43.

**Google ADK**: Sliding window compaction with configurable `compaction_interval` and `overlap_size`. 60-80% token reduction.

### 2.2 Tool Definition Optimization

- **Token-efficient tool use** (Anthropic beta): Up to 70% reduction in tool call output tokens
- **Deferred tool loading**: 85% context reduction while maintaining full tool access

### 2.3 ACON Framework (Oct 2025)

Lowers memory usage by 26-54% while maintaining task performance. Enables distillation into smaller models preserving 95% accuracy.

---

## 3. Reasoning Token Accounting

### OpenAI

o3: $10/M input, $40/M output; reasoning tokens billed as output.

### Anthropic

**Adaptive Thinking** (2026, replacing Extended Thinking): `thinking: {type: "adaptive"}` with effort parameter (low/medium/high/max). Model dynamically decides reasoning depth. `budget_tokens` deprecated on 4.6 models.

---

## 4. Streaming Formats

| Aspect | SSE | NDJSON/JSONL |
|---|---|---|
| Protocol | `text/event-stream` | `application/x-ndjson` |
| Event typing | Built-in `event:` field | Embedded in JSON |
| Primary use | Real-time streaming | Batch processing, logging |
| Browser support | Native `EventSource` | Manual parsing |

---

## 5. Production Cost Realities (2026)

- Agents make 3-10x more LLM calls than chatbots per request
- Complex agents consume 5-20x more tokens than simple chains
- Output tokens cost 3-10x more than input tokens across all providers
- Monthly operational spend: $3,200-$13,000 for typical production agents
- 40% of agentic AI projects fail before production

### Cost Optimization Playbook

1. Model routing (cheap models for simple tasks)
2. Prompt caching (maximize cache hits)
3. Batch processing (50% discount on OpenAI)
4. Adaptive reasoning (avoid over-thinking simple tasks)
5. Context compaction (structured summarization)
6. Deferred tool loading (on-demand tool definitions)

---

## 6. Model Pricing Reference (Early 2026)

| Model | Input/1M | Output/1M | Cached Input | Context |
|---|---|---|---|---|
| Claude Opus 4.6 | $15.00 | $75.00 | $1.50 | 1M |
| Claude Sonnet 4.6 | $3.00 | $15.00 | $0.30 | 1M |
| GPT-4o | $2.50 | $10.00 | $1.25 | 128K |
| o3 | $10.00 | $40.00 | $2.50 | 200K |
| Gemini 2.5 Pro | $1.25-$2.50 | $10-$15 | $0.125-$0.25 | 1M+ |

---

## Sources

- [Anthropic Prompt Caching](https://platform.claude.com/docs/en/build-with-claude/prompt-caching)
- [Anthropic Token-Saving Updates](https://www.anthropic.com/news/token-saving-updates)
- [Anthropic Adaptive Thinking](https://platform.claude.com/docs/en/build-with-claude/adaptive-thinking)
- [Anthropic Advanced Tool Use](https://www.anthropic.com/engineering/advanced-tool-use)
- [OpenAI Prompt Caching](https://developers.openai.com/api/docs/guides/prompt-caching)
- [OpenAI Reasoning Models](https://platform.openai.com/docs/guides/reasoning)
- [Google Gemini Context Caching](https://ai.google.dev/gemini-api/docs/caching)
- [Gemini Implicit Caching](https://developers.googleblog.com/en/gemini-2-5-models-now-support-implicit-caching/)
- [Google ADK Context Compression](https://google.github.io/adk-docs/context/compaction/)
- [Factory.ai Compressing Context](https://factory.ai/news/compressing-context)
- [ACON -- arXiv:2510.00615](https://arxiv.org/html/2510.00615v1)
- [Anthropic Context Engineering](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)
