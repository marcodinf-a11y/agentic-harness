# AI Agent Evaluation and Benchmarking: August 2025 -- March 2026

## Executive Summary

The period from August 2025 to March 2026 has been transformative for AI agent evaluation. SWE-bench Verified, long considered the gold standard, was effectively retired by OpenAI in February 2026 due to pervasive training data contamination. New benchmarks like SWE-bench Pro, Terminal-Bench 2.0, FeatureBench, and SWE-EVO have emerged to address saturation and contamination. Evaluation methodology has matured significantly, with LLM-as-judge frameworks gaining academic rigor, cost-effectiveness becoming a first-class metric, and the industry recognizing that pass/fail rates are insufficient measures of code quality.

---

## 1. SWE-bench / SWE-bench Verified

### 1.1 Current Leaderboard (Feb--Mar 2026)

| Model | SWE-bench Verified (%) |
|---|---|
| Claude Opus 4.5 | 80.9 |
| Claude Opus 4.6 | 80.8 |
| Gemini 3.1 Pro | 80.6 |
| GPT-5.2 | 80.0 |
| Claude Sonnet 4.6 | 79.6 |
| DeepSeek V3.2 | 73.0 |

### 1.2 The Contamination Crisis

OpenAI retired SWE-bench Verified (Feb 2026): 59.4% of remaining tasks flawed; all frontier models confirmed training data overlap. Models scoring ~80% on Verified dropped to ~23% on SWE-bench Pro.

### 1.3 SWE-bench Pro (Scale AI / SEAL)

1,865 long-horizon tasks across Python, Go, TypeScript, JavaScript. Claude Opus 4.5 tops at 45.9%.

---

## 2. Terminal-Bench 2.0

89 tasks spanning scientific workflows, network config, cybersecurity, data pipelines. Gemini 3.1 Pro leads at 78.4%.

---

## 3. Emerging Benchmarks

| Benchmark | Top Score (%) | Task Type | Contamination Risk |
|---|---|---|---|
| SWE-bench Verified | ~81 | Single bug fixes | **High** (retired) |
| SWE-bench Pro | ~46 | Multi-file, multi-lang | Medium |
| Terminal-Bench 2.0 | ~78 | CLI/DevOps tasks | Low |
| FeatureBench | ~11 | Feature development | Low |
| SWE-EVO | ~21 | Software evolution | Low |

---

## 4. Evaluation Methods

- **Agent-as-a-Judge**: Multi-step reasoning agents as evaluators
- **Multi-Agent-as-Judge**: Panel of evaluator personas
- **Test-based**: Harness + agent wrapper + evaluation suite
- **Trace grading**: Scoring full agent execution logs (OpenAI)

---

## 5. Cost-Effectiveness

| Agent/Model | Benchmark | Score (%) | Cost |
|---|---|---|---|
| Sonar Foundation Agent | SWE-bench Verified | 79.2 | $1.26/issue |
| DeepSeek V3.2-Exp | Aider Polyglot | 74.2 | $1.30/run |

Production AI agent systems: $3,200--$13,000/month operational costs.

---

## 6. Code Quality Beyond Pass/Fail

AI-generated code introduces 1.7x more issues, 4x more duplication, 1.64x worse maintainability. The industry is shifting from "does it pass?" to "is it good?"

---

## Sources

- [SWE-bench Official Leaderboard](https://www.swebench.com/)
- [Why OpenAI No Longer Evaluates SWE-bench Verified](https://openai.com/index/why-we-no-longer-evaluate-swe-bench-verified/)
- [SWE-bench Pro -- Scale AI SEAL](https://scale.com/leaderboard/swe_bench_pro_public)
- [Terminal-Bench 2.0 -- Snorkel AI](https://snorkel.ai/blog/terminal-bench-2-0-raising-the-bar-for-ai-agent-evaluation/)
- [FeatureBench -- arXiv:2602.10975](https://arxiv.org/html/2602.10975v1)
- [SWE-EVO -- arXiv:2512.18470](https://arxiv.org/abs/2512.18470)
- [SWE-rebench -- arXiv:2505.20411](https://arxiv.org/abs/2505.20411)
- [Aider Leaderboards](https://aider.chat/docs/leaderboards/)
- [Galileo Agent Leaderboard v2](https://galileo.ai/blog/agent-leaderboard-v2)
- [Code Quality Metrics 2026 -- Qodo](https://www.qodo.ai/blog/code-quality-metrics-2026/)
- [GitClear AI Code Quality 2025](https://www.gitclear.com/ai_assistant_code_quality_2025_research)
- [Agent-as-a-Judge -- arXiv:2508.02994](https://arxiv.org/html/2508.02994v1)
- [LLMs-as-Judges Survey -- arXiv:2412.05579](https://arxiv.org/abs/2412.05579)
- [Epoch AI SWE-bench Verified](https://epoch.ai/benchmarks/swe-bench-verified)
