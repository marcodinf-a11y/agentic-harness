# Agentic Harness — Session Management

How long-running development sessions are tracked, when context degrades, and when to start fresh.

## Why Sessions Matter

AI coding agents degrade as conversations grow. More context means more tokens, which means higher cost, slower responses, and worse reasoning. The harness treats token consumption as the primary health metric for session quality.

A "session" in the harness is a single task execution: one task dispatched to one agent in one sandbox. The harness does not maintain multi-turn conversations with agents — each run is independent by default. Session continuation (multi-turn) is supported per-agent but is not the default workflow.

## Session Lifecycle

```
1. LOAD        Load task definition from JSON
                │
2. SANDBOX     Create temp directory
                Write seed files
                Run setup commands
                │
3. INVOKE      Start agent CLI as subprocess
                cwd = sandbox directory
                │
4. MONITOR     Agent runs, produces output
                (Token monitoring via JSON parsing)
                │
5. CAPTURE     Parse JSON/JSONL output
                Normalize token usage
                Capture diff and artifacts
                │
6. VALIDATE    Run validation_commands in sandbox
                Score: all pass → 1.0, any fail → 0.0
                │
7. REPORT      Generate structured JSON report
                Save to results/{task_id}_{timestamp}.json
                │
8. CLEANUP     Remove sandbox directory
                (Artifacts already captured)
```

## Token Budget Monitoring

### The 70k Default

The default budget is 70,000 tokens (input + output). This represents the point where context rot risk becomes significant for typical coding tasks.

### What Counts

- `input_tokens` + `output_tokens` = `total_tokens` (this is what the budget tracks)
- **Thinking/reasoning tokens**: excluded from budget (agent-internal, not controllable)
- **Cache read tokens**: tracked but not counted toward budget
- **Cache write tokens**: tracked but not counted toward budget

### Budget Status Thresholds

| Status | Utilization | Total Tokens | Signal |
|---|---|---|---|
| **WITHIN** | < 80% | < 56,000 | Normal operation |
| **WARNING** | 80–100% | 56,000–70,000 | Context rot risk elevated |
| **EXCEEDED** | > 100% | > 70,000 | Context rot likely |

### Budget Analysis Output

Each report includes a budget analysis block:

```json
{
    "total_tokens": 13350,
    "budget": 70000,
    "utilization_pct": 19.1,
    "status": "within",
    "remaining": 56650,
    "cache_efficiency": {
        "cache_read_tokens": 9000,
        "cache_write_tokens": 3500
    }
}
```

## Context Window Monitoring

Separate from the token budget, context window usage tracks how full the agent's context is:

| Context Used | Status | Meaning |
|---|---|---|
| 0–40% | Green | Plenty of room |
| 41–50% | Yellow | Middle range |
| 51–100% | Red | Context is running low |

This is most relevant for multi-turn sessions where the conversation accumulates.

## When to Start Fresh

The harness flags a session for restart when:

1. **Budget exceeded** — total tokens > 70k (or custom budget)
2. **Warning threshold hit** — total tokens > 80% of budget
3. **Progressive bloat** — multiple turns consuming progressively more input tokens (sign of context accumulation)

The recommended response:

- **WITHIN**: continue working
- **WARNING**: finish the current subtask, then start a new session
- **EXCEEDED**: stop, start fresh, break the task into smaller pieces

## Session Continuation (Per-Agent)

By default, each harness run is a fresh, isolated execution. But agents support session continuation for multi-turn workflows:

### Claude Code

```bash
# Get session ID from first run
session_id=$(claude -p "query" --output-format json | jq -r '.session_id')

# Continue in that session
claude -p "next query" --resume "$session_id"

# Or continue the most recent session
claude -p "next query" --continue
```

### Codex CLI

```bash
# Resume last session
codex exec resume --last "next task"

# Resume specific session
codex exec resume <SESSION_ID>
```

### Gemini CLI

No documented session resume mechanism. Each invocation is independent.

## Report Format

Each run produces a JSON report saved to `results/{task_id}_{YYYYMMDD_HHMMSS}.json`:

```json
{
    "task": {
        "id": "fizzbuzz-001",
        "name": "FizzBuzz implementation",
        "prompt": "...",
        "token_budget": 70000
    },
    "results": [
        {
            "agent_name": "claude-code",
            "task_id": "fizzbuzz-001",
            "exit_code": 0,
            "normalized_tokens": {
                "input_tokens": 12500,
                "output_tokens": 850,
                "cache_read_tokens": 9000,
                "cache_write_tokens": 3500,
                "total_tokens": 13350
            },
            "budget_status": "within",
            "budget_analysis": {
                "total_tokens": 13350,
                "budget": 70000,
                "utilization_pct": 19.1,
                "status": "within",
                "remaining": 56650
            },
            "duration_seconds": 8.2,
            "cost_usd": 0.0399,
            "result_text": "...",
            "diff": "diff --git a/fizzbuzz.py...",
            "artifacts": {
                "fizzbuzz.py": "def main():\n..."
            },
            "timestamp": "2026-03-02T22:15:00"
        }
    ],
    "evaluations": [
        {
            "agent_name": "claude-code",
            "task_id": "fizzbuzz-001",
            "validation_passed": true,
            "validation_output": "1\n2\nFizz\n4\nBuzz\n...",
            "score": 1.0,
            "notes": ""
        }
    ],
    "timestamp": "2026-03-02T22:15:15"
}
```

### Key Fields

- **`normalized_tokens`** — unified token usage, comparable across agents
- **`budget_status`** — quick health check: `within`, `warning`, or `exceeded`
- **`budget_analysis`** — detailed breakdown with utilization percentage and remaining budget
- **`cost_usd`** — dollar cost (available from Claude Code; `null` for others)
- **`diff`** — git diff of what the agent changed in the sandbox
- **`artifacts`** — map of filename → content for files the agent created/modified
- **`validation_passed`** / **`score`** — evaluation results from running validation commands

## Multi-Session Patterns

For tasks that need iteration:

1. **Break it down** — split a large task into sequential subtasks, each with its own budget
2. **Fresh context per subtask** — each subtask gets a clean sandbox and a fresh agent session
3. **Carry forward artifacts** — use `files` in the next task's JSON to seed with output from the previous task
4. **Monitor trends** — if token usage is increasing across subtasks, the problem decomposition may need rethinking
