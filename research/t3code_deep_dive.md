# T3Code Review: Patterns Applicable to Rein

## Context

T3Code (github.com/pingdotgg/t3code) is a minimal web GUI for coding agents by Ping.gg. It's a TypeScript monorepo (Bun + Turbo) with a WebSocket server wrapping agent CLIs, a React frontend, and a shared contracts package. Currently Codex-first, Claude Code planned.

This review identifies patterns from t3code that Rein should adopt, adapt, or explicitly skip.

---

## Patterns Worth Adopting

### 1. Contracts-First Type Separation

**What t3code does:** Dedicated `packages/contracts` package with zero runtime logic — only Zod schemas and TypeScript types for provider events, WebSocket protocol, models, and orchestration. No implementation leaks into the contract layer.

**Apply to Rein:** Create a `rein/contracts/` or `rein/models.py` module that is pure dataclasses/Pydantic models with no business logic. Rein's spec already plans `models.py` — ensure it stays dependency-free (no imports from runner, monitor, etc.). This makes it easy to validate task JSON, generate schemas, and keep adapters decoupled.

**Effort:** Low — already planned, just enforce the boundary.

### 2. Event Taxonomy with Discriminated Types

**What t3code does:** `ProviderRuntimeEventV2` defines **45 event types** across 9 categories (session, thread, turn, item, request, task, tool, auth, diagnostic). Each event has a discriminated `type` field enabling exhaustive pattern matching.

**Apply to Rein:** Define a `ReinEvent` union type for the monitor's stream. Categories:
- **session**: `session.started`, `session.completed`, `session.killed`
- **pressure**: `pressure.zone_changed`, `pressure.reading`
- **validation**: `validation.passed`, `validation.failed`
- **gate**: `gate.round_started`, `gate.round_completed`

This replaces ad-hoc logging with structured events that can drive a future UI, event stream file (JSONL), or webhook integration. Aligns with the JSONL event stream already on Rein's roadmap (R3/Future).

**Effort:** Medium — define ~15 event types, emit them from runner/monitor.

### 3. Causation Tracking on Events

**What t3code does:** Every event carries `commandId`, `causationEventId`, and `correlationId`. This enables replay, diffing, and full audit trails.

**Apply to Rein:** Add `session_id`, `task_id`, and `timestamp` to every `ReinEvent`. For multi-round quality gate loops, add `round_id`. This is minimal overhead but enables:
- Post-hoc analysis of why a session was killed
- Correlating pressure readings with specific agent turns
- Future replay/debugging capability

**Effort:** Low — just fields on the event dataclass.

### 4. Provider-Agnostic Options Pattern

**What t3code does:** `ProviderSessionStartInput` uses a generic `providerOptions?: Record<string, unknown>` instead of discriminated unions per provider. New providers don't require core type changes.

**Apply to Rein:** The `AgentAdapter` protocol already abstracts providers, but task-level agent config should use an open `agent_options: dict` field rather than adding top-level fields per agent. Example: `{"agent": "claude", "agent_options": {"max_turns": 50, "permission_mode": "default"}}`.

**Effort:** Low — add one optional field to TaskDefinition.

### 5. Defensive Subprocess Management

**What t3code does:** `processRunner.ts` implements:
- Platform-aware process tree kill (Windows `taskkill /T` vs Unix signals)
- Configurable timeout cascade: SIGTERM → wait → SIGKILL
- Buffer size tracking with truncation mode
- `settled` flag preventing double-resolution
- Error normalization across platforms

**Apply to Rein:** Rein's spec already defines SIGTERM → 5s → SIGKILL. Additionally adopt:
- **Settled guard**: Boolean flag to prevent double-kill race conditions in the monitor
- **Buffer overflow protection**: Track stdout buffer size, truncate if agent produces excessive output
- **Structured exit normalization**: Map all termination paths to a clean `TerminationResult` dataclass with `exit_code`, `signal`, `reason`, `partial_output_bytes`

**Effort:** Low-Medium — enhance the existing kill protocol.

### 6. Model Registry with Aliases

**What t3code does:** Three-level model mapping: full options by provider, default model per provider, and slug aliases (e.g., `"5.4"` → `"gpt-5.4"`).

**Apply to Rein:** Define a simple model registry in config that maps shorthand to full model IDs and context window sizes. This avoids hardcoding context windows in the pressure monitor. Example in `rein.toml`:
```toml
[models.claude-sonnet]
context_window = 200000
aliases = ["sonnet"]

[models.claude-opus]
context_window = 200000
aliases = ["opus"]
```

**Effort:** Low — small config-driven lookup table.

---

## Patterns to Note but Skip for Now

### Session State Machine
T3code defines explicit session states (`connecting → ready → running → error → closed`). Rein's sessions are simpler (single-shot dispatch), so a full state machine is premature. **Revisit for R2** when session resume is added.

### Event Sourcing / Replay
T3code's `replayEvents()` and `getTurnDiff()` are powerful but complex. Rein should emit structured events (pattern #2 above) which *enables* future replay, but building replay infrastructure is out of scope for R1.

### WebSocket Protocol
T3code's WS layer is for its web UI. Rein is CLI-first. Skip entirely unless a TUI/web dashboard is planned.

### Effect-Based DI
T3code uses the Effect library for dependency injection. Rein should stick with Python's simpler patterns (Protocol classes, constructor injection). Effect-style DI adds complexity without proportional benefit in a CLI tool.

---

## Summary: What Changes in Rein

| Pattern | Where in Rein | Priority | Effort |
|---------|--------------|----------|--------|
| Contracts-first models.py | `rein/models.py` | P0 (R1) | Low |
| Event taxonomy | `rein/events.py` | P1 (R1) | Medium |
| Causation fields on events | `rein/events.py` | P1 (R1) | Low |
| Provider-agnostic agent_options | `rein/models.py` TaskDefinition | P1 (R1) | Low |
| Settled guard + buffer protection | `rein/monitor.py`, `rein/runner.py` | P1 (R1) | Low-Med |
| Model registry with aliases | `rein.toml` config, `rein/models.py` | P2 (R1) | Low |
| Session state machine | — | Defer to R2 | — |
| Event replay | — | Defer to R3 | — |
