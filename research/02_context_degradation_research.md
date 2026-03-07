# Context Window Degradation, Context Rot, and Long-Context Performance in LLMs

**Research Survey: August 2025 -- March 2026**

This document surveys academic papers, official vendor research, and professional research team findings on context window degradation in large language models. All sources are from August 2025 to March 2026 unless noted as foundational prior work.

---

## Table of Contents

1. [Chroma Research: Context Rot](#1-chroma-research-context-rot)
2. [Context Length Alone Hurts LLM Performance Despite Perfect Retrieval](#2-context-length-alone-hurts-llm-performance-despite-perfect-retrieval)
3. [Context Is What You Need: Maximum Effective Context Window](#3-context-is-what-you-need-maximum-effective-context-window)
4. [Context Discipline and Performance Correlation](#4-context-discipline-and-performance-correlation)
5. [Lost in the Middle: Emergent Property from Information Retrieval Demands](#5-lost-in-the-middle-emergent-property-from-information-retrieval-demands)
6. [Precipitous Long-Context Collapse in Sequence Models](#6-precipitous-long-context-collapse-in-sequence-models)
7. [Long-Context LLMs Meet RAG (ICLR 2025)](#7-long-context-llms-meet-rag-iclr-2025)
8. [The Complexity Trap: Observation Masking vs. LLM Summarization (NeurIPS 2025)](#8-the-complexity-trap-observation-masking-vs-llm-summarization-neurips-2025)
9. [Anthropic: Effective Context Engineering for AI Agents](#9-anthropic-effective-context-engineering-for-ai-agents)
10. [Epoch AI: Context Window Usage Trends](#10-epoch-ai-context-window-usage-trends)
11. [Synthesis: Key Takeaways for Agent Orchestration](#11-synthesis-key-takeaways-for-agent-orchestration)
12. [Sources](#12-sources)

---

## 1. Chroma Research: Context Rot

**Source:** Chroma Technical Report, July 2025
**Authors:** Kelly Hong, Anton Troynikov, Jeff Huber
**Credibility:** Professional research team (Chroma); code and data publicly available on GitHub

### Key Findings

- Evaluated **18 LLMs** including GPT-4.1, Claude 4, Gemini 2.5, and Qwen3.
- Models do **not** process context uniformly. Performance grows increasingly unreliable as input length grows -- a phenomenon the authors term "context rot."
- Two evaluation tasks were used: (1) conversational Q&A using LongMemEval, and (2) a synthetic repeated-words replication task deliberately controlled to isolate input length effects.
- No single model ranked first across all experiments. Claude Sonnet 4 performed best on the repeated words task; GPT-4.1 topped Needle in a Haystack.

### Quantitative Results

- Models scoring **>95% on short prompts** fell to approximately **60--70%** on longer contexts containing semantically relevant distractors.
- As needle-question similarity decreases, performance degrades more sharply with increasing input length -- reflecting realistic scenarios where exact question-answer matches are rare.
- **Claude models** decay the slowest overall, though they sometimes refuse long tasks for safety reasons.
- **Gemini models** showed the earliest onset of degradation, with random word artifacts appearing around 500--750 words in some variants.
- **GPT-4 Turbo** had the highest output variability, with tendencies to generate extra words or truncate output.
- GPT-3.5 hit a **60% refusal rate** on longer tasks.

### Practical Implications for Agent Orchestration

- Do not assume that filling a context window is cost-free. Every token added carries a degradation risk.
- Model selection should be task-specific: the best model for retrieval is not necessarily the best for generation under long context.
- Context rot is highly task-dependent and non-uniform, making it unpredictable without empirical testing for each use case.

---

## 2. Context Length Alone Hurts LLM Performance Despite Perfect Retrieval

**Source:** arXiv:2510.05381, October 2025; published at EMNLP 2025 Findings
**Authors:** Amazon Science research team
**Credibility:** Peer-reviewed venue (EMNLP 2025 Findings)

### Key Findings

- Across **5 open- and closed-source LLMs** on math, QA, and coding tasks, performance degrades **13.9%--85%** as input length increases, even when the model can perfectly retrieve all relevant information.
- The degradation occurs even when irrelevant tokens are replaced with **minimally distracting whitespace**.
- Most surprisingly, degradation persists even when irrelevant tokens are **fully masked** and the model is forced to attend only to the relevant tokens.
- This proves that **sheer input length alone** -- independent of retrieval quality, distraction, or attention dispersion -- is sufficient to degrade reasoning.

### Proposed Mitigation

- A simple, model-agnostic strategy: prompt the model to **recite the retrieved evidence** before attempting to solve the problem.
- On RULER benchmark, this approach improved GPT-4o performance by up to **4%** on an already strong baseline.

### Practical Implications for Agent Orchestration

- This is the strongest evidence that context length is a **first-order concern**, not merely an engineering detail.
- Even "perfect RAG" cannot overcome the reasoning tax imposed by long context. Shorter contexts are fundamentally better for reasoning quality.
- The recitation strategy is cheap and applicable: have the model restate key facts before answering.

---

## 3. Context Is What You Need: Maximum Effective Context Window

**Source:** arXiv:2509.21361, September 2025; published in Advances in Artificial Intelligence and Machine Learning, January 2026
**Author:** Norman Paulsen
**Credibility:** Peer-reviewed journal publication

### Key Findings

- Introduces the concept of **Maximum Effective Context Window (MECW)** vs. the advertised **Maximum Context Window (MCW)**.
- Collected **hundreds of thousands of data points** across several models.
- MECW is drastically smaller than MCW and **shifts based on problem type**.
- Some top-line models **failed with as few as 100 tokens** in context.
- Most models showed **severe accuracy degradation by 1,000 tokens**.
- All models fell short of their MCW by **as much as 99%**.

### Quantitative Results

- The gap between MCW and MECW is not a minor discrepancy; it is often **orders of magnitude**.
- Problem type significantly modulates the effective window. The same model may have vastly different MECW values for retrieval vs. reasoning vs. summarization.

### Practical Implications for Agent Orchestration

- Never trust advertised context window sizes. Empirically measure MECW for each model-task pair.
- Design agent systems with context budgets far below the nominal MCW.
- Task-specific MECW testing should be part of model evaluation pipelines.

---

## 4. Context Discipline and Performance Correlation

**Source:** arXiv:2601.11564, December 2025
**Credibility:** Preprint; detailed empirical study

### Key Findings

- Investigated Llama-3.1-70B and Qwen1.5-14B under up to **15,000 words of irrelevant distracting context**.
- Identified **non-linear performance degradation** tied to the growth of the Key-Value (KV) cache.
- Dense transformer models maintained accuracy between **97.5% and 98.5%** even with 15,000 words of distractors -- but at enormous computational cost.
- Latency increased by **719.64%** at the 15,000-word regime.
- Analysis of **Mixture-of-Experts (MoE) architecture** revealed unique behavioral anomalies at varying context scales, suggesting that MoE architectural benefits may be masked by infrastructure bottlenecks at high token volumes.

### Practical Implications for Agent Orchestration

- There is a distinction between **quality degradation** and **performance degradation**. Dense transformers may maintain accuracy but at extreme latency cost.
- For latency-sensitive agent systems, context discipline (keeping context lean) has massive throughput benefits even when accuracy holds.
- MoE models may not scale context as expected due to infrastructure interactions.

---

## 5. Lost in the Middle: Emergent Property from Information Retrieval Demands

**Source:** arXiv:2510.10276, October 2025
**Authors:** Nikolaus Salvatore, Hao Wang, Qiong Zhang
**Credibility:** Academic research; follow-up to the seminal "Lost in the Middle" (Liu et al., 2023)

### Key Findings

- Proposes that the U-shaped "lost in the middle" performance curve is **not a flaw** but an **emergent adaptation** to competing information retrieval demands during pre-training.
- Two competing demands: **long-term memory** (uniform recall across input) vs. **short-term memory** (prioritizing recent information).
- Trained GPT-2 and Llama variants from scratch on simulated long-term and short-term memory paradigms; the U-shaped curve **emerged naturally**.
- The **recency effect** aligns with short-term memory demand in training data.
- The **primacy effect** is induced by uniform long-term memory demand and reinforced by autoregressive properties and **attention sink formation**.

### Additional Context: MIT Follow-Up (2025)

- MIT researchers explained the architectural roots: **causal masking** in transformer attention means each token can only attend to preceding tokens, creating inherent position bias.
- **Positional encodings** further contribute to the bias.
- As of early 2026, **no production model has fully eliminated position bias**.

### Practical Implications for Agent Orchestration

- Place the most critical information at the **beginning and end** of context, not the middle.
- For retrieval-augmented systems, **reorder retrieved passages** to place the most relevant at the start.
- Position bias is architectural and training-data-driven -- it cannot be fully eliminated by prompt engineering alone.

---

## 6. Precipitous Long-Context Collapse in Sequence Models

**Source:** Synthesis from ICLR 2025 and related 2025 publications
**Credibility:** Conference-level peer review (ICLR 2025)

### Key Findings

- "Precipitous long-context collapse" denotes the **abrupt, non-linear** performance degradation when inputs approach or exceed the advertised context window length.
- Measured as dramatic declines in recall, F1, and reasoning ability on tasks trivial at short lengths.

#### Root Cause Analysis

- **Transformers:** Dense softmax-based self-attention disperses attention weights as context grows. "Critical attention scaling" theory shows that only **polylogarithmic scaling** of softmax temperature avoids rank-collapse.
- **RNNs (e.g., Mamba):** Insufficient long-context exposure during training relative to state size causes "state collapse" -- blowup in hidden-state activations and inability to forget. Follows a **strictly linear scaling law** for training length vs. state dimension.

#### Collapse Threshold

- Improvements through prompt design provide measurable benefits but **do not overcome architectural collapse** past **32k--128k token** inputs.
- LLM-based web agents saw success rates drop from **40--50% baseline** to **<10%** in long-context scenarios.

### Mitigations

- **Sparse and adaptive attention** mechanisms (alpha-entmax, ASEntmax, sparse-global attention) with critical polylogarithmic scaling achieve high accuracy over extended contexts.
- **Training on long-dependency-rich data** (e.g., via ProLong framework) sustains performance and mitigates abrupt collapse.

### Practical Implications for Agent Orchestration

- Expect a **hard ceiling** around 32k--128k tokens for reliable reasoning, regardless of advertised window size.
- Plan agent workflows to stay well below this threshold.
- For tasks requiring reasoning over large codebases or document sets, decompose into sub-tasks rather than cramming everything into one context.

---

## 7. Long-Context LLMs Meet RAG (ICLR 2025)

**Source:** ICLR 2025 Conference Paper
**Credibility:** Top-tier peer-reviewed venue

### Key Findings

- For many long-context LLMs, generation quality **initially improves then subsequently declines** as the number of retrieved passages increases.
- **Hard negatives** (passages that are topically related but not actually answering the question) are a key contributor to degradation.
- Current long-context benchmarks may not adequately capture the challenge of hard negatives in real-world RAG.
- LLMs struggle more with hard negatives from **stronger retrievers** (counterintuitively, better retrieval can surface harder-to-distinguish distractors).

### Proposed Solutions

- **Retrieval reordering:** A simple, training-free optimization that provides significant gains.
- **RAG-specific fine-tuning:** Outperforms direct fine-tuning, enabling the LLM to extract relevant information from retrieved context more effectively while maintaining generalization.

### Practical Implications for Agent Orchestration

- More retrieved context is not always better. There is an optimal number of passages beyond which quality drops.
- Invest in retrieval quality (precision over recall) rather than maximizing context fill.
- Consider RAG-specific fine-tuning for production systems.

---

## 8. The Complexity Trap: Observation Masking vs. LLM Summarization (NeurIPS 2025)

**Source:** NeurIPS 2025 DL4Code Workshop; Master's thesis at TUM
**Authors:** JetBrains Research (Tobias Lindenbauer et al.)
**Credibility:** NeurIPS workshop paper; code available on GitHub

### Key Findings

- Compared two context management strategies for SE (software engineering) agents:
  1. **Observation Masking:** Reduces only environment observations while preserving full action and reasoning history.
  2. **LLM Summarization:** Compresses entire history into compact summaries.
- Surprising result: LLM summarization was **unable to consistently or significantly outperform** simple observation masking.
- Both approaches reduce cost by approximately **50%** without significantly degrading downstream task performance.
- A **hybrid** of both approaches yielded **7% and 11% cost reductions** compared to either method alone.

### Practical Implications for Agent Orchestration

- Simple observation masking is a cost-effective, low-complexity strategy for managing agent context.
- LLM-based summarization adds cost and latency without proportional quality gains.
- For SE agents specifically, observations (tool outputs, file contents) dominate context -- masking these is high-leverage.
- Hybrid approaches offer marginal additional gains.

---

## 9. Anthropic: Effective Context Engineering for AI Agents

**Source:** Anthropic Engineering Blog, October 2025
**Credibility:** Official vendor guidance from the Claude model developer

### Key Recommendations

Anthropic frames context management as the central challenge: "What configuration of context is most likely to generate our model's desired behavior?"

#### Three Core Techniques for Extended Agent Sessions

1. **Compaction:** When approaching the context window limit, summarize the conversation and reinitiate with the summary in a new context window.
2. **Structured note-taking:** Have agents maintain persistent structured notes across turns, rather than relying on raw conversation history.
3. **Multi-agent architectures:** Split work across multiple agents, each with a focused, lean context.

#### System Prompt Design

- Use simple, direct language at the right "altitude" -- specific enough to guide behavior, flexible enough to provide strong heuristics.
- Provide diverse, canonical few-shot examples.

#### Tool Design

- Build **small, distinct, and efficient tools** -- avoid bloated or overlapping functionality.
- Fewer, well-designed tools lead to more reliable agent behavior.

#### Context Rot Acknowledgment

Anthropic explicitly acknowledges context rot: "LLMs have limited attention -- the more information they're given, the harder it becomes for them to stay focused and recall details accurately. Simply increasing the context window doesn't guarantee better performance."

### Practical Implications for Agent Orchestration

- Use compaction as a first-class mechanism, not an afterthought.
- Design multi-agent systems where each agent sees only what it needs.
- Structured notes are more durable than raw conversation history.
- Tool proliferation worsens context pollution.

---

## 10. Epoch AI: Context Window Usage Trends

**Source:** Epoch AI Data Insights, 2025
**Credibility:** Independent AI research organization; data-driven analysis

### Key Findings

- Since mid-2023, the longest LLM context windows have grown by approximately **30x per year**.
- Models' ability to effectively use input is improving even faster: on two long-context benchmarks, the input length where top models reach 80% accuracy has risen by **over 250x in 9 months**.
- Real-world average sequence length has more than **tripled** over the past 20 months (from ~2,000 tokens in late 2023 to over **5,400 by late 2025**).
- Average prompt tokens per request have increased roughly **4x** (from ~1.5K to over 6K).
- Usage is shifting from open-ended generation toward **reasoning over substantial user-provided material** (codebases, documents, transcripts).

### Practical Implications for Agent Orchestration

- Users are already pushing toward longer contexts in practice, making degradation management increasingly urgent.
- The gap between "accepted" and "effectively used" context is narrowing for top models, but degradation beyond effective limits remains severe.
- Agent systems should track effective context utilization, not just token counts.

---

## 11. Synthesis: Key Takeaways for Agent Orchestration

### The Degradation Landscape

| Factor | Evidence | Severity |
|--------|----------|----------|
| Sheer input length | 13.9--85% reasoning degradation even with perfect retrieval | Critical |
| Position bias (lost in the middle) | Architectural; no production model eliminates it | Moderate-High |
| Hard negatives from retrieval | Quality initially improves then declines with more passages | Moderate |
| KV-cache growth / latency | Up to 719% latency increase at 15K words | High (operational) |
| Advertised vs. effective window | MECW can be 99% smaller than MCW | Critical |
| Architectural collapse threshold | 32K--128K tokens regardless of advertised size | Hard ceiling |

### Decision Framework: When to Split Context

1. **Always split when context exceeds 32K tokens** for reasoning-heavy tasks. Evidence from multiple studies shows architectural collapse beyond this range.
2. **Split when adding retrieved passages beyond the optimal count.** The ICLR 2025 RAG paper shows quality inverts after a peak.
3. **Split when latency matters.** Even if accuracy holds, the 719% latency tax at 15K words makes long context impractical for interactive agents.
4. **Split by information type.** JetBrains research shows that observation/output data should be masked or summarized while preserving reasoning history.

### Recommended Context Management Stack

1. **Context budgeting:** Set hard limits per agent turn based on empirically measured MECW, not advertised MCW.
2. **Information placement:** Critical information at the beginning and end of context; never in the middle.
3. **Compaction:** Summarize and reinitiate when approaching limits (Anthropic's recommended approach).
4. **Retrieval precision over recall:** Fewer, higher-quality passages outperform more passages.
5. **Evidence recitation:** Have the model restate key facts before reasoning (4% improvement demonstrated).
6. **Multi-agent decomposition:** Each agent gets a focused, lean context rather than one agent seeing everything.
7. **Observation masking:** For tool-heavy agents, mask prior tool outputs while preserving reasoning traces (50% cost reduction, negligible quality loss).

### Token-to-Quality Degradation Curve (Composite)

Based on the aggregated research, the approximate degradation profile is:

```
Quality
  |
  |============================
  |                            \
  |                             \
  |                              \====
  |                                   \
  |                                    \
  |                                     \========
  |                                              \
  +--+-----+--------+----------+---------+--------> Tokens
   0  1K   4K      16K        32K       128K     MCW
     (stable)  (mild decay)  (significant)  (collapse)
```

- **0--4K tokens:** Generally stable performance across models.
- **4K--16K tokens:** Mild, task-dependent decay. Position effects become measurable.
- **16K--32K tokens:** Significant degradation for reasoning tasks. Retrieval tasks may still hold.
- **32K--128K tokens:** Architectural collapse zone. Prompt engineering cannot compensate.
- **Beyond 128K:** Only viable for simple retrieval or summarization; reasoning is severely impaired.

---

## 12. Sources

### Academic Papers

1. Hong, K., Troynikov, A., & Huber, J. (2025). "Context Rot: How Increasing Input Tokens Impacts LLM Performance." Chroma Technical Report.
   - https://research.trychroma.com/context-rot
   - https://github.com/chroma-core/context-rot

2. Du et al. (2025). "Context Length Alone Hurts LLM Performance Despite Perfect Retrieval." EMNLP 2025 Findings. arXiv:2510.05381.
   - https://arxiv.org/abs/2510.05381

3. Paulsen, N. (2025/2026). "Context Is What You Need: The Maximum Effective Context Window for Real World Limits of LLMs." AAIML. arXiv:2509.21361.
   - https://arxiv.org/abs/2509.21361

4. (2025). "Context Discipline and Performance Correlation: Analyzing LLM Performance and Quality Degradation Under Varying Context Lengths." arXiv:2601.11564.
   - https://arxiv.org/html/2601.11564v1

5. Salvatore, N., Wang, H., & Zhang, Q. (2025). "Lost in the Middle: An Emergent Property from Information Retrieval Demands in LLMs." arXiv:2510.10276.
   - https://arxiv.org/abs/2510.10276

6. (2025). "Long-Context LLMs Meet RAG: Overcoming Challenges for Long Inputs in RAG." ICLR 2025.
   - https://proceedings.iclr.cc/paper_files/paper/2025/file/11440c427f0f76f191ac06b50d7a2517-Paper-Conference.pdf

7. Lindenbauer, T. et al. (2025). "The Complexity Trap: Simple Observation Masking Is as Efficient as LLM Summarization for Agent Context Management." NeurIPS 2025 DL4Code Workshop.
   - https://blog.jetbrains.com/research/2025/12/efficient-context-management/

### Vendor and Professional Research

8. Anthropic (2025). "Effective Context Engineering for AI Agents."
   - https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents

9. Epoch AI (2025). "LLMs now accept longer inputs, and the best models can use them more effectively."
   - https://epoch.ai/data-insights/context-windows

### Foundational Prior Work (pre-August 2025, referenced for context)

10. Liu, N. F. et al. (2023). "Lost in the Middle: How Language Models Use Long Contexts." TACL 2024. arXiv:2307.03172.
    - https://arxiv.org/abs/2307.03172
