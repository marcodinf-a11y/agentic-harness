# RLMs (Recursive Language Models) - Research Findings

Paper: https://arxiv.org/abs/2512.24601
Authors: Alex L. Zhang, Tim Kraska, Omar Khattab (MIT CSAIL)
Submitted: December 31, 2025 (v1), January 28, 2026 (v2)
Code: https://github.com/alexzhang13/rlm (`pip install rlms`)
License: MIT (code), CC BY 4.0 (paper)

---

## Phase 1: Core Paper Understanding

### Problem Statement

Modern LLMs have limited context windows and suffer from **context rot** — performance degradation as context length increases, even within the window. RLMs address this at inference time rather than through architectural changes.

### Core Thesis

RLMs expose the same external interface as standard LLMs but internally treat long prompts as **environmental objects** rather than direct neural network input. The model programmatically examines, decomposes, and recursively calls itself over context snippets. This enables processing inputs up to **100x beyond model context windows**.

### Architecture

The RLM execution flow:

1. **Initialize REPL**: Load the full prompt as a `context` Python string variable
2. **Provide `llm_query()` function**: Enables the model to make sub-LM calls from within code
3. **Model writes code**: The LM generates Python code to examine/filter/chunk the context
4. **Observe results**: Code execution outputs are fed back to the model
5. **Iterate or recurse**: Model can loop, call `llm_query()` on sub-chunks, or build up results
6. **Final output**: Model calls `FINAL()` or `FINAL_VAR()` to return its answer

Key insight: The context is never fed through the neural network all at once. Instead, it lives as a string in the REPL, and the model interacts with it programmatically — printing slices, regex-searching, chunking, etc.

### How This Differs from Standard Context Window Usage

| Aspect | Standard LLM | RLM |
|--------|-------------|-----|
| Context handling | All tokens in attention | Context as external variable |
| Scaling | Limited by window size | Arbitrarily long inputs |
| Processing strategy | Implicit (attention) | Explicit (code-driven) |
| Sub-task delegation | N/A | `llm_query()` sub-calls |
| Memory management | Attention KV cache | Python variables in REPL |
| Cost scaling | Linear with input | Proportional to task complexity |

### Key Claims and Results

| Benchmark | GPT-5 RLM | GPT-5 Base | Improvement |
|-----------|-----------|-----------|-------------|
| CodeQA | 62% ($0.11) | 24% ($0.13) | +158% |
| BrowseComp+ 1K | 91.33% ($0.99) | 0% (exceeds window) | N/A |
| OOLONG | 56.50% ($0.43) | 44% ($0.14) | +28.4% |
| OOLONG-Pairs | 58% ($0.33) | 0.04% ($0.16) | +1450x |

RLM-Qwen3-8B (fine-tuned small model): **28.3% average improvement** over base, competitive with GPT-5 on some tasks.

---

## Phase 2: Technical Deep Dive

### Recursion Strategy

The model decides when/how to decompose — there is **no hardcoded decomposition logic**. The LM itself determines strategy based on the system prompt and its own reasoning. Observed emergent patterns:

1. **Filtering via model priors**: Use regex/keyword search to narrow context before reading (e.g., search for "festival" + "La Union" instead of reading all documents)
2. **Chunking and recursive calling**: Split context by newlines, headers, or fixed character counts, then `llm_query()` each chunk
3. **Answer verification**: Use sub-LM calls to verify intermediate answers
4. **Iterative probing**: Print small slices of context to understand structure, then adapt strategy

### Memory Model

Intermediate results are stored as **Python variables in the REPL namespace**:
- Results accumulate in buffer variables (lists, dicts, strings)
- Sub-call outputs get assigned to variables and aggregated
- `FINAL_VAR()` can return a variable directly (for unbounded output length)
- The REPL namespace persists across all code execution steps within a single RLM call

### Code Generation Patterns

Typical code the LM writes in the REPL:

```python
# Probe context structure
print(context[:5000])
print(f"Total length: {len(context)}")

# Chunk and process
lines = context.split('\n')
results = []
for i in range(0, len(lines), 100):
    chunk = '\n'.join(lines[i:i+100])
    answer = llm_query(f"Classify this chunk: {chunk}")
    results.append(answer)

# Aggregate
final = '\n'.join(results)
FINAL_VAR(final)
```

### Termination and Output

- `FINAL(answer_string)` — return a short answer
- `FINAL_VAR(variable_name)` — return an arbitrarily long variable
- The distinction is **brittle** — models sometimes output plans as final answers or make unexpected decisions about when to stop

### Recursion Depth

Current implementation: **maximum depth of 1**. Sub-calls are plain LMs, not recursive RLMs. The paper notes deeper recursion as future work.

### System Prompt

The GPT-5 system prompt (~6200 chars) provides:
- REPL environment description and capabilities
- Context character count and chunk specifications
- Strategy examples (chunking by section, summarize-per-chunk, aggregate)
- Usage examples for `llm_query()` and buffer management
- Instructions to reason step-by-step
- Final answer format requirements

Qwen3-Coder gets an additional warning: "Be cautious not to use too many sub-LM calls."

### Error Handling

Not explicitly discussed in the paper. The REPL executes code via Python `exec()` (in the local environment). Errors would surface as Python exceptions visible to the model in the observation step. The model can then adapt its code.

---

## Phase 3: Comparison with Related Work

### vs. ReAct (Yao et al., 2022)

ReAct interleaves reasoning traces with actions (search, lookup, finish) and observations in a flat loop. The full input context sits in the LLM prompt.

| Aspect | ReAct | RLM |
|--------|-------|-----|
| Loop structure | Thought → Action → Observation | Code → Execute → Observe (in REPL) |
| Context handling | Must fit in window | Stored as REPL variable, never ingested raw |
| Action space | Fixed text actions (search, lookup) | Turing-complete Python |
| Recursion | None | Yes, via `llm_query()` |
| Max input | Context window | Unbounded (100x+ beyond window) |
| Key limitation | Fixed context window | Cost variance |

RLM subsumes ReAct — the REPL can implement any ReAct-style tool call via code, but also enables recursive sub-calling and unbounded context.

### vs. CodeAct (Wang et al., 2024)

Closest prior work. Both use executable Python as the action mechanism. Critical differences:

| Aspect | CodeAct | RLM |
|--------|---------|-----|
| Input handling | In prompt (context window) | Stored as REPL variable |
| State persistence | Largely ephemeral per turn | Persistent REPL namespace across cells |
| Recursive self-calls | None | First-class `llm_query()` primitive |
| Purpose | Better tool-use via code format | Solve unbounded-context problem |
| CodeQA benchmark | 24% | 62% |

The performance gap (62% vs 24% on CodeQA) is attributed to RLM's persistent state and recursive decomposition.

### vs. THREAD (Schroeder et al., 2025) / Context Folding

**THREAD** frames generation as a "thread of execution" that dynamically spawns child threads. Parent offloads sub-tasks to children, which return only needed tokens.

**Context Folding** lets agents branch into sub-trajectories, handle subtasks, then "fold" back — collapsing intermediate steps into summaries. FoldGRPO trains models to fold effectively, matching ReAct with 10x smaller active context.

**Why RLM goes beyond them:**
- Both modify the *output generation* process but still require *input context* to fit in the window
- RLM decouples input from the forward pass entirely — input lives in the REPL
- THREAD's spawning is inline in the output stream; RLM's recursion is controlled by explicit Python code with full programmatic control over partitioning and aggregation

### vs. Summary Agent / Context Condensation

Summary approaches (Baleen, ReSum, OpenAI condensation) use **lossy compression** — they summarize/compact context iteratively. RLMs keep the full context intact in the REPL variable. The model decides what to examine, preserving information fidelity. RLMs are up to 3x cheaper than summarization baselines with stronger performance.

### vs. MemWalker (Chen et al., 2023) / MemGPT (Packer et al., 2023)

**MemWalker** builds a fixed tree of summary nodes from long text at construction time (query-agnostic). At query time, navigates root-to-leaf to find relevant segments. Quality depends on whether summarization preserved the right information.

**MemGPT** draws from OS virtual memory: main context (fast, limited = RAM) + external context (large, slow = disk). Model explicitly pages information in/out via memory management function calls.

| Aspect | MemWalker | MemGPT | RLM |
|--------|-----------|--------|-----|
| Memory structure | Pre-built summary tree | Explicit tiered memory API | Implicit via Python variables |
| Query-adaptive | No (fixed at construction) | Partial | Fully adaptive |
| Information loss | Yes (summarization) | Yes (compression) | No (raw text via code) |
| Exploration | Tree navigation | Page in/out | Lazy (grep, slice, filter) |
| Learning curve | Low | Must learn memory API | Must write Python |

RLM's memory management is implicit — any Python variable persists in the REPL. The model writes normal code and gets memory for free through standard programming constructs.

### Positioning Summary

| Dimension | ReAct | CodeAct | THREAD | MemWalker/MemGPT | RLM |
|-----------|-------|---------|--------|-------------------|-----|
| Action space | Fixed text | Python code | Thread spawning | Memory API | Python REPL + recursion |
| Input handling | In prompt | In prompt | In prompt | Summarized/paged | REPL variable |
| Max input length | Window | Window | Window | Beyond (with info loss) | Unbounded |
| Recursion | None | None | Yes (output-side) | Tree nav / paging | Yes (input-side, code-controlled) |
| Memory model | Conv. history | Ephemeral | Summary folding | Explicit management | Implicit via variables |

RLMs sit at the intersection of code-as-action (CodeAct), recursive decomposition (THREAD), and external memory (MemGPT). The unique contribution is combining all three: code-driven interaction with external context + recursive sub-calling.

### Key Related Work Sources

- [ReAct](https://arxiv.org/abs/2210.03629) — Yao et al., 2022
- [CodeAct](https://arxiv.org/abs/2402.01030) — Wang et al., 2024
- [THREAD](https://arxiv.org/abs/2405.17402) — Schroeder et al., 2025
- [Context Folding](https://arxiv.org/abs/2510.11967) — 2025
- [MemWalker](https://arxiv.org/abs/2310.05029) — Chen et al., 2023
- [MemGPT](https://arxiv.org/abs/2310.08560) — Packer et al., 2023

---

## Phase 4: Practical Implications

### Where RLMs Excel

- **Tasks exceeding context windows**: BrowseComp+ (6-11M tokens) — base models literally can't process it
- **Information-dense tasks**: OOLONG-Pairs (quadratic complexity) — 58% F1 vs 0.04% base
- **Code understanding**: CodeQA on large repos (900K tokens) — 62% vs 24%
- **Multi-hop reasoning**: BrowseComp+ requires piecing together info from 1000 documents

### Where RLMs Struggle

- **Short, simple contexts**: Overhead of REPL setup not worth it for tasks that fit in the window
- **S-NIAH at small scales**: Base GPT-5 matches or beats RLM for contexts < 2^14 tokens
- **Model-dependent quality**: Qwen3-Coder solves only 44.66% of BrowseComp+ vs GPT-5's 91.33%
- **Consistency**: High cost variance — 95th percentile runs can be significantly more expensive

### Token / Cost Efficiency

- **Median runs are cheaper** than base models on long-context tasks
- Cost scales with **task complexity**, not input length:
  - Constant-complexity tasks (S-NIAH): ~constant cost regardless of input
  - Linear tasks (OOLONG): linear cost growth
  - Quadratic tasks (OOLONG-Pairs): quadratic cost growth
- **High variance**: Outlier runs (redundant verification, excessive sub-calls) can be 3-5x more expensive
- RLMs are **up to 3x cheaper** than summary agent baselines

### Limitations and Failure Modes

1. **Output tag brittleness**: Models sometimes output plans or intermediate thoughts as final answers via `FINAL()`
2. **Redundant verification**: Models may verify answers 5+ times, wasting tokens (Example B.3)
3. **Model-specific behavior**: Identical prompts produce very different strategies across models
4. **Smaller models struggle**: Models without strong coding abilities can't effectively use the REPL
5. **Sequential execution**: Sub-calls are synchronous — no parallelism in current implementation
6. **Max depth = 1**: Sub-calls can't themselves recurse (future work)
7. **Thinking model limits**: Models with insufficient output token limits fail

---

## Phase 5: Implementation Considerations

### Reference Implementation

GitHub: https://github.com/alexzhang13/rlm (2.9k stars, 547 forks)
Minimal version: https://github.com/alexzhang13/rlm-minimal

**Install**: `pip install rlms`

### Key Abstractions

```
RLM class
├── Environment (REPL backend)
│   ├── LocalREPL — exec() in same process
│   ├── DockerREPL — containerized execution
│   ├── ModalSandbox — cloud isolation
│   ├── DaytonaSandbox
│   └── E2BSandbox
├── Client (LLM backend)
│   ├── OpenAI
│   ├── Anthropic
│   ├── OpenRouter
│   ├── LiteLLM
│   └── vLLM
├── Logger (RLMLogger)
│   ├── In-memory mode
│   └── JSONL disk output
└── Visualizer (Node.js frontend at localhost:3001)
```

### API Surface

```python
from rlm import RLM, RLMLogger

logger = RLMLogger(log_dir="./logs")
rlm = RLM(
    environment="local",      # or "docker", "modal", etc.
    backend="openai",
    backend_kwargs={"model": "gpt-5-nano"},
    verbose=True,
)
result = rlm.completion(prompt="...", logger=logger)
print(result.response)
```

### Safety Considerations

- **LocalREPL uses `exec()`** — arbitrary code execution in the host process (dangerous)
- **DockerREPL** — containerized but still requires Docker daemon access
- **Cloud sandboxes** (Modal, E2B, Daytona) — true isolation but add latency and cost
- Sub-calls to `llm_query()` happen within the sandbox — the LM generates the prompts for sub-calls, which could be adversarial if the input context is untrusted

### Model Requirements

- Strong code generation ability (Python)
- Ability to follow complex system prompts
- Sufficient output token limit (thinking models need especially high limits)
- Smaller models (8B) can work with fine-tuning (RLM-Qwen3-8B)

---

## Key Questions Answered

### 1. How does RLM handle unbounded context differently from chunking or summarization?

The context is stored as a Python string variable in the REPL. The model writes code to interact with it — printing slices, regex searching, chunking programmatically. Unlike summarization, **no information is lost**. Unlike fixed chunking, the **model chooses its own strategy** based on the task and what it observes.

### 2. What is the recursion depth in practice?

**Depth = 1** in current implementation. Sub-calls are plain LMs, not RLMs. The root LM calls `llm_query()` which invokes a (potentially different, cheaper) sub-LM. Deeper recursion is identified as future work.

### 3. Is the approach model-agnostic?

Partially. The framework works with any LM, but **performance is highly model-dependent**. GPT-5 significantly outperforms Qwen3-Coder on BrowseComp+ (91% vs 45%). Models need strong code generation skills. An 8B model can work with fine-tuning (RLM-Qwen3-8B). Different models require different prompt tuning (Qwen needs warnings about excessive sub-calls).

### 4. How does performance scale with task complexity?

- Constant-complexity tasks: RLM matches base models, slight overhead
- Linear-complexity tasks: RLM outperforms by ~28-33%
- Quadratic-complexity tasks: RLM dramatically outperforms (58% vs 0.04%)
- Cost scales proportionally to task complexity, not input length

### 5. What are the failure modes?

- Brittle output tags (premature/incorrect `FINAL()`)
- Redundant verification loops (wasting tokens)
- Model-dependent strategy quality
- Smaller models can't code well enough
- Sequential sub-calls create latency
- No error recovery strategy beyond Python exceptions

---

## Phase 6: Community Reception and External Validation

### Reproduction Studies

**"Think, But Don't Overthink: Reproducing Recursive Language Models"** (arXiv:2603.02615, March 3, 2026)
- By Daren Wang. Tested RLMs with DeepSeek v3.2 and Kimi K2 on S-NIAH and OOLONG
- **Confirmed** RLM gains on complex tasks
- **Critical finding**: Depth-1 RLMs *degrade performance on simple retrieval tasks* vs vanilla LLMs — unnecessary overhead
- **Deeper recursion backfires**: Depth-2 RLMs performed significantly worse than depth-1 (DeepSeek v3.2 accuracy fell from 42.1% to 33.7%) due to compounding formatting errors and redundant loops
- Conclusion: "large-scale industrial deployment remains highly challenging"

**"Recursive Models for Long-Horizon Reasoning"** (arXiv:2603.02112, March 2, 2026)
- By Yang, Srebro, and Li. Theoretical paper proving that any computable problem admits recursive decomposition requiring exponentially smaller active context
- Trained a 3B model on Boolean satisfiability as proof of concept

### Community Implementations (5+ independent repos)

- `avbiswas/fast-rlm` — performance-focused reimplementation
- `fullstackwebdev/rlm_repl` — standalone REPL-based version
- `codecrack3/Recursive-Language-Models-RLM-with-DSpy` — DSPy integration
- `ysz/recursive-llm` — simplified "context as variables" approach
- `rileyseaburg` (GitHub Gist) — fine-tuning Qwen3-4B for RLM behavior

### Industry Adoption

**DSPy Integration**: RLM is now an official DSPy module (`dspy.ai/api/modules/RLM/`). Notable since Omar Khattab co-authored both DSPy and the RLM paper.

**Prime Intellect**: Built **RLMEnv**, integrated into their verifiers stack. Called RLMs "the paradigm of 2026." Their implementation adds `llm_batch` for parallel sub-query fan-out. Training INTELLECT-3 with native RLM scaffolding.

### Community Critiques (Beyond Paper's Own Limitations)

1. **Simple tasks hurt**: RLMs add overhead that degrades performance on straightforward retrieval (reproduction paper)
2. **Reliability variance**: Scores ranged 0/6 to 6/6 across 30 identical runs on aggregation tasks
3. **Information loss risk**: While RLMs avoid summarization, the model still must correctly identify which chunks matter — itself error-prone
4. **Cost spikes**: Token usage increases by "orders of magnitude" once RLM architecture engages vs. single forward pass
5. **Hacker News reception**: Mixed — enthusiasm about paradigm shift, skepticism about practical reliability (3+ threads)

### RLM-Qwen3-8B Status

- 28.3% improvement over base, approaches vanilla GPT-5 on 3/4 tasks
- On OOLONG: Qwen3-8B went from 0.0% to 24.0% with REPL
- Trained on only **1,000 samples** — remarkably data-efficient
- No independent benchmarks published yet (reproduction paper used DeepSeek/Kimi instead)

### Key Sources

- [Reproduction paper](https://arxiv.org/abs/2603.02615)
- [Theoretical foundations paper](https://arxiv.org/abs/2603.02112)
- [Prime Intellect blog](https://www.primeintellect.ai/blog/rlm)
- [DSPy RLM module](https://dspy.ai/api/modules/RLM/)
- [Author's blog post](https://alexzhang13.github.io/blog/2025/rlm/)
- [HN discussion](https://news.ycombinator.com/item?id=45596059)
