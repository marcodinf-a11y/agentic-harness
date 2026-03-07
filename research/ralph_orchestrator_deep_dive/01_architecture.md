# Ralph Orchestrator: Architecture & Design

**Deep Dive Document 01 | March 2026**

Analysis of ralph-orchestrator's architecture, process model, and design decisions. All claims cite source code from `github.com/mikeyobrien/ralph-orchestrator` (v2.7.0).

---

## 1. System Overview

Ralph-orchestrator is a Rust workspace with 9 crates that transforms the bare Ralph Wiggum bash loop into an event-driven orchestration system.

```
┌─────────────────────────────────────────────────┐
│                  ralph-cli                       │
│            (CLI entry, run_loop_impl)            │
├─────────────────────────────────────────────────┤
│                  ralph-core                      │
│  ┌──────────┐ ┌──────────┐ ┌──────────────────┐│
│  │ EventLoop│ │ HatSystem│ │ MemoryStore      ││
│  │ EventBus │ │ Topology │ │ TaskStore         ││
│  │ Hooks    │ │ Triggers │ │ ScratchpadManager ││
│  └──────────┘ └──────────┘ └──────────────────┘│
├─────────────────────────────────────────────────┤
│                ralph-adapters                    │
│  ┌────────┐ ┌───────┐ ┌──────┐ ┌─────┐ ┌────┐ │
│  │ Claude │ │ Kiro  │ │Gemini│ │Codex│ │ Pi │ │
│  │ Amp    │ │Copilot│ │O.Code│ │ ACP │ │Auto│ │
│  └────────┘ └───────┘ └──────┘ └─────┘ └────┘ │
├─────────────────────────────────────────────────┤
│  ralph-proto │ ralph-tui │ ralph-telegram       │
│  ralph-api   │ ralph-bench│ ralph-e2e           │
└─────────────────────────────────────────────────┘
         │                        │
         ▼                        ▼
   Agent subprocess          .ralph/ directory
   (PTY or pipe)          (events, scratchpad,
                           tasks, memories)
```

Source: repo Cargo.toml workspace members, crate structure.

---

## 2. How It Wraps the Agent Process

**PTY-based subprocess execution** is the primary mechanism. The `PtyExecutor` (in `ralph-core/src/pty_executor.rs`) spawns agents in a pseudo-terminal via the `portable-pty` crate. This preserves the agent's TUI features (colors, spinners). A fallback `CliExecutor` uses `tokio::process::Command` with piped stdout/stderr.

Key details:
- Non-blocking I/O via `tokio::select!` — races agent output against interrupt signals
- Interactive mode forwards user input to the PTY; observe mode is output-only
- Prompt delivery: prompts >7000 chars are written to a temp file and passed via `--input-file` (Claude); shorter prompts go as CLI args or stdin depending on backend
- Agent process exits after each iteration — a new subprocess is spawned next iteration

**Each iteration is a fresh process.** This is the core architectural difference from the Claude Code stop-hook variant, which keeps the session alive and accumulates context.

Source: `pty_executor.rs`, `cli_executor.rs`, `cli_backend.rs` in ralph-core.

---

## 3. State Management

Progress persists between iterations through the `.ralph/` directory:

| Artifact | Path | Purpose |
|----------|------|---------|
| Scratchpad | `.ralph/agent/scratchpad.md` | Agent's working notes — read and updated each iteration |
| Events | `.ralph/events-{timestamp}.jsonl` | Structured events emitted via `ralph emit` CLI |
| Tasks | `.ralph/tasks.jsonl` | Runtime work tracking (create, update, complete) |
| Memories | `.ralph/memories/` | Persistent cross-run knowledge |
| Summary | `.ralph/agent/summary.md` | Generated on loop termination |
| Handoff | `.ralph/agent/handoff.md` | Context for human or next loop |

The **scratchpad** is the primary inter-iteration memory mechanism. It is capped at ~16,000 characters (~4000 tokens × 4 chars/token). When over budget, earlier entries are trimmed but section headings are preserved as a summary. Only the tail (most recent entries) survives — a FIFO strategy.

Source: `scratchpad.rs`, `event_reader.rs`, `task_store.rs` in ralph-core.

---

## 4. Configuration Model

Configuration lives in `ralph.yml` at the project root. Key categories:

| Category | Examples | Notes |
|----------|----------|-------|
| Loop control | `max_iterations: 150`, `max_runtime_seconds: 28800`, `max_cost_usd: 20.0` | All optional with defaults |
| Backend | `backend: claude` | Or `auto` for auto-detection |
| Hats | `name`, `triggers`, `publishes`, `instructions`, `backend` | Per-hat backend override supported |
| Backpressure | `gates: [{name: test, command: "cargo test", on_fail: block}]` | Shell commands between iterations |
| Memories | `enabled: true`, `inject: auto`, `budget: 4000` | Character limit for memory injection |
| Hooks | `pre_iteration_start`, `post_loop_termination`, etc. | Lifecycle hooks with suspend/resume |
| Skills | `dirs: [".ralph/skills/"]`, `overrides: {...}` | Auto-injected per-hat instructions |
| RObot | `enabled: true`, `timeout_seconds: 300` | Telegram human-in-the-loop |

**31 presets** provide turnkey configurations for common workflows (TDD, spec-driven, debugging, etc.).

Source: `config.rs`, `ralph.yml` defaults, presets directory.

---

## 5. How It Differs from the Bare Ralph Loop

| Dimension | Bare Ralph | ralph-orchestrator |
|-----------|-----------|-------------------|
| Loop mechanism | `while :; do cat PROMPT.md \| claude ; done` | Rust `loop {}` with `tokio::select!` |
| Process isolation | Shell restart | PTY subprocess per iteration |
| Inter-iteration memory | Git commits + spec files | Scratchpad + events JSONL + task store + memories |
| Task routing | Agent reads spec, picks one | Hat system with pub/sub event routing |
| Validation | Tests pass/fail | Backpressure gates (configurable shell commands) |
| Completion | Implicit (agent exits) | Explicit `ralph emit LOOP_COMPLETE` with required-events validation |
| Stagnation detection | None | Stale loop, thrashing, consecutive failure detection |
| Cost tracking | None | Cumulative cost from backend stream metadata |
| Human interaction | `Ctrl+C` | Telegram bot, TUI guidance queue, web dashboard |
| Parallelism | None | Git worktree isolation for concurrent loops |

Source: comparison with `ghuntley.com/ralph/` reference implementation.

---

## 6. Dependency Footprint

- **Language:** Rust (edition 2024)
- **Key dependencies:** `tokio` (async runtime), `portable-pty` (PTY), `ratatui` (TUI), `serde` (serialization), `clap` (CLI)
- **Frontend:** React + Vite (web dashboard)
- **Distribution:** npm (`@ralph-orchestrator/ralph-cli`), Homebrew, Cargo, Docker, Nix
- **Platforms:** macOS (aarch64, x86_64), Linux (aarch64, x86_64). No Windows — requires Unix PTY.
- **Runtime requirements:** At least one supported agent CLI installed (claude, gemini, codex, kiro, etc.)

Source: `Cargo.toml`, `package.json`, GitHub releases.

---

## Sources

- github.com/mikeyobrien/ralph-orchestrator (v2.7.0, accessed March 2026)
- Source files: `pty_executor.rs`, `cli_executor.rs`, `cli_backend.rs`, `config.rs`, `event_loop.rs`, `scratchpad.rs`
- ghuntley.com/ralph/ (bare Ralph reference)
