# Agentic Harness

Context-pressure-aware orchestrator for AI coding agents. Monitors how agents consume their context windows in real-time, intervenes when quality is at risk, and ensures work is persisted before degradation sets in. Dispatches tasks, isolates execution, captures output, evaluates results.

**MVP**: single-agent workflow. **Future**: multi-agent composition and parallel dispatch.

## Supported Agents

- **Claude Code** (`claude` CLI) — Anthropic
- **Codex CLI** (`codex` CLI) — OpenAI
- **Gemini CLI** (`gemini` CLI) — Google

## Documentation

| Document | Content |
|---|---|
| [BRIEF.md](BRIEF.md) | Project brief — what, who, why, five pillars, MVP scope, multi-agent composition |
| [ARCHITECTURE.md](ARCHITECTURE.md) | System overview, five subsystems (incl. pressure monitor), agent adapter protocol, execution flow, CLI design |
| [AGENTS.md](AGENTS.md) | Per-agent integration — invocation, JSON/NDJSON output formats, streaming events, token mappings, signal behavior |
| [TASKS.md](TASKS.md) | Task definition spec — JSON (MVP) + YAML (future), schema, examples, validation patterns |
| [TOKENS.md](TOKENS.md) | Token normalization, context pressure model (`ContextPressure`, `ZoneConfig`), budget thresholds, cache semantics |
| [REPORTS.md](REPORTS.md) | Report JSON schema, evaluation scoring, output conventions |
| [SESSIONS.md](SESSIONS.md) | Session management — lifecycle, **context pressure monitoring protocol**, zone actions, wrap-up, continuation |

## Quick Start

```bash
# Run a task against Claude Code
harness run -t tasks/example_fizzbuzz.json -a claude

# Run all tasks in a directory
harness run -t tasks/ -a claude

# Dry run
harness run -t tasks/example_fizzbuzz.json -a claude --dry-run
```

## Context Pressure

The harness monitors **context pressure** — the ratio of consumed tokens to the model's context window — as its primary metric. Configurable zones (Green 0–60%, Yellow 60–80%, Red >80%) trigger graceful wrap-up or immediate termination. See [SESSIONS.md](SESSIONS.md) for the full monitoring protocol and [TOKENS.md](TOKENS.md) for data structures.

## Token Budget

Default: **70,000 tokens** (input + output). Complementary cost/resource metric alongside context pressure. Thinking/reasoning tokens excluded. See [TOKENS.md](TOKENS.md) for the full token normalization model, budget thresholds, and analysis details.
