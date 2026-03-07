# Goose Context Management — Deep Dive

## Overview

Goose (by Block) implements a multi-layered context management system designed to handle long-running agent sessions without manual intervention. Written in Rust, the system uses a two-tiered approach: **auto-compaction** (proactive summarization as context fills) and **context limit strategies** (fallback mechanisms when compaction is insufficient). This document covers the architecture, algorithms, configuration, and comparisons with other agent systems.

---

## 1. Conversation Compaction: `check_if_compaction_needed` and `compact_messages`

### When Compaction Triggers

**[Confirmed]** Auto-compaction triggers by default when token usage reaches **80% of the model's context limit** in both Goose Desktop and the CLI.
- Source: [Smart Context Management docs](https://block.github.io/goose/docs/guides/sessions/smart-context-management/)

**[Confirmed]** The threshold is configurable via the `GOOSE_AUTO_COMPACT_THRESHOLD` environment variable. Setting it to `0.6` triggers compaction at 60% usage; setting it to `0.0` disables auto-compaction entirely.
- Source: [Smart Context Management docs](https://block.github.io/goose/docs/guides/sessions/smart-context-management/)

### The Compaction Algorithm

**[Confirmed]** Compaction works by making a **synchronous, non-streaming `complete()` call** to the LLM, requesting a summary of the conversation. This synchronous approach ensures usage data is returned in the primary response object, bypassing streaming token-count issues.
- Source: [Issue #6588](https://github.com/block/goose/issues/6588)

**[Confirmed]** The compaction prompt is defined in a `compaction.md` template file that users can customize. This template controls how the LLM summarizes conversations during compaction.
- Source: [Smart Context Management docs](https://block.github.io/goose/docs/guides/sessions/smart-context-management/)

**[Confirmed]** After compaction, Goose displays a confirmation: `"Auto-compacted context: X -> Y tokens (Z% reduction)"`.
- Source: [Smart Context Management docs](https://block.github.io/goose/docs/guides/sessions/smart-context-management/)

### What Is Preserved vs. Summarized

**[Confirmed]** Goose preserves **recent tool calls in full detail** while summarizing older ones. By default, the most recent **10 tool calls** are kept intact; older tool call outputs are replaced with a short placeholder like `"[Old tool result content cleared]"`.
- Source: [Smart Context Management docs](https://block.github.io/goose/docs/guides/sessions/smart-context-management/)

**[Inferred]** The system walks backward through tool calls, estimating token cost, and marks older outputs as "compacted." This preserves traceability (the fact that a tool ran) without carrying the full output payload. The cutoff is configurable via `GOOSE_TOOL_CALL_CUTOFF`.
- Reasoning: Documented behavior of tool output summarization, confirmed env var name from documentation.

**[Inferred]** The compaction summary is designed to capture: conversation state, next steps, key decisions, and learnings — following a pattern similar to other agent compaction systems.
- Reasoning: The customizable `compaction.md` template and the general approach described in documentation.

---

## 2. Autocompact + One-Shot Summarization

### The Algorithm

**[Confirmed]** Goose uses a one-shot summarization approach: when the auto-compact threshold is reached, the entire conversation (minus recent tool call outputs) is sent to the LLM in a single synchronous call with the compaction prompt. The LLM returns a summary that replaces the older conversation history.
- Source: [Smart Context Management docs](https://block.github.io/goose/docs/guides/sessions/smart-context-management/)

**[Confirmed]** Users can also proactively trigger summarization using the `/summarize` slash command before reaching context limits.
- Source: [Smart Context Management docs](https://block.github.io/goose/docs/guides/sessions/smart-context-management/)

### What Problem It Solves

**[Confirmed]** Before auto-compaction, Goose did not manage context limits gracefully — sessions would fail or behave unpredictably when context was exhausted. Issue #3485 ("Improve Goose's Context Window Management") describes the motivation: making the flow trigger more automatically and proactively, while being transparent and minimally disruptive to the user.
- Source: [Issue #3485](https://github.com/block/goose/issues/3485)

### Tool Call Summarization (Background)

**[Confirmed]** Goose summarizes older tool call outputs **in the background** while keeping recent calls in full detail. This is a separate mechanism from full conversation compaction — it runs continuously as tool calls accumulate.
- Source: [Smart Context Management docs](https://block.github.io/goose/docs/guides/sessions/smart-context-management/)

**[Confirmed]** The default cutoff is 10 tool calls; calls beyond the 10 most recent have their outputs cleared/summarized. The `GOOSE_TOOL_CALL_CUTOFF` environment variable allows advanced tuning.
- Source: [Smart Context Management docs](https://block.github.io/goose/docs/guides/sessions/smart-context-management/)

---

## 3. Token Counting

### Implementation

**[Confirmed]** Token counting is implemented in `crates/goose/src/token_counter.rs`. The file is part of the core `goose` crate.
- Source: [GitHub file listing](https://github.com/block/goose/blob/main/crates/goose/src/token_counter.rs)

**[Inferred]** Goose likely uses a tiktoken-based or character-estimation approach for token counting, consistent with the Rust ecosystem. The `tiktoken-rs` crate is a common dependency for Rust projects needing OpenAI-compatible tokenization. For non-OpenAI models, a character-based estimation (e.g., ~4 characters per token) is likely used as a fallback.
- Reasoning: The Rust ecosystem has `tiktoken-rs` available on crates.io; Goose needs cross-provider token counting.

### Per-Provider Differences

**[Confirmed]** Token counting has known inconsistencies across providers:
  - Google Gemini models and models via OpenRouter **consistently report 0% token usage**, indicating the usage data from these providers is not properly consumed.
  - There are inconsistencies between the UI-displayed token count and the actual error message when context is exceeded.
- Source: [Issue #4986](https://github.com/block/goose/issues/4986)

**[Confirmed]** Auto-compaction relies on the synchronous `complete()` call's response for accurate usage data. This works because the response object includes usage metadata. Streaming responses do not reliably provide this data.
- Source: [Issue #6588](https://github.com/block/goose/issues/6588)

### Token Display

**[Confirmed]** After the first message, both Desktop and CLI display token usage. Desktop shows a colored circle indicator:
  - **Green**: Normal usage, plenty of space
  - **Orange**: Warning state (approaching 80% capacity)
- Source: [Smart Context Management docs](https://block.github.io/goose/docs/guides/sessions/smart-context-management/)

---

## 4. Context Window Awareness

### How Goose Knows Model Limits

**[Confirmed]** Goose maintains a `MODEL_SPECIFIC_LIMITS` constant in `crates/goose/src/model.rs` that defines context window sizes for known model families using pattern matching. Examples:
  - `gpt-5*` -> 272,000 tokens
  - `gpt-4.1*` -> 1,000,000 tokens
  - Default fallback for unknown models: **32,000 tokens** (32k)
- Source: [Issue #6185](https://github.com/block/goose/issues/6185), [Issue #2925](https://github.com/block/goose/issues/2925), [DeepWiki provider config](https://deepwiki.com/block/goose/2.2-provider-configuration)

**[Confirmed]** The `ModelConfig` struct in `crates/goose/src/model.rs` manages model-specific settings including context limits, temperature, and toolshim configuration.
- Source: [DeepWiki](https://deepwiki.com/block/goose/2.2-provider-configuration)

### User Overrides

**[Confirmed]** Users can override context limits with the `GOOSE_CONTEXT_LIMIT` environment variable (e.g., `GOOSE_CONTEXT_LIMIT=32600`). This is important for:
  - Self-hosted models via Ollama (which default to 2048 tokens)
  - Providers like GitHub Copilot that impose lower limits than the raw model supports
- Source: [Issue #3259](https://github.com/block/goose/issues/3259), [Issue #1253](https://github.com/block/goose/issues/1253)

### Adaptive Behavior

**[Confirmed]** As context fills, Goose follows this layered approach:
  1. **Tool call output summarization** (continuous, background, after 10+ tool calls)
  2. **Auto-compaction** (triggered at threshold, default 80%)
  3. **Context limit strategy** (fallback if still exceeded after compaction)
- Source: [Smart Context Management docs](https://block.github.io/goose/docs/guides/sessions/smart-context-management/)

---

## 5. Message Types and Structure

### Internal Message Format

**[Confirmed]** Messages use a `Message` struct with the following fields:
  - `role`: `Role` enum (`Role::User`, `Role::Assistant`)
  - `created`: Unix timestamp (integer)
  - `content`: `Vec<MessageContent>` — a vector of content blocks
- Source: [GitHub source references](https://github.com/block/goose/blob/main/crates/goose/src/token_counter.rs)

**[Confirmed]** Serialized JSON format:
```json
{
  "role": "user",
  "created": 1741285622,
  "content": [{"type": "text", "text": "hi"}]
}
```
- Source: Issue discussions and token_counter.rs references

### MessageContent Variants

**[Confirmed]** `MessageContent` is an enum with at least these variants:
  - `MessageContent::Text` — plain text content
  - `MessageContent::ToolUse` — a tool call request (with tool name, arguments, call ID)
  - `MessageContent::ToolResult` — the result of a tool call (with matching call ID)
  - `MessageContent::Image` — image content (supported in user messages, at least for Google provider)
- Source: [Issue #1224](https://github.com/block/goose/issues/1224), [Issue #1102](https://github.com/block/goose/issues/1102)

### Tool Call Ordering

**[Confirmed]** Goose enforces strict ordering of `ToolUse` and `ToolResult` blocks, mirroring the Anthropic API convention:
  - Messages containing `ToolUse` blocks must be followed by a user message with matching `ToolResult` blocks
  - The number of `ToolResult` blocks must match the number of `ToolUse` blocks from the previous turn
  - Violations result in 400 Bad Request errors from the API
- Source: [Issue #1102](https://github.com/block/goose/issues/1102), [Issue #1224](https://github.com/block/goose/issues/1224)

### Session Persistence

**[Confirmed]** Sessions are persisted as JSONL files with automatic backup and recovery mechanisms. The system maintains conversation history, tool execution results, and user preferences across restarts.
- Source: [DeepWiki](https://deepwiki.com/block/goose)

---

## 6. System Prompt Management

### System Prompt Source

**[Confirmed]** Goose's system prompt is stored in `crates/goose/src/prompts/system.md`. It defines Goose as "a general-purpose AI agent called goose, created by Block (parent company of Square, CashApp, and Tidal)."
- Source: [GitHub file listing](https://github.com/block/goose/blob/main/crates/goose/src/prompts/system.md)

**[Confirmed]** Model-specific system prompts exist (e.g., `system_gpt_4.1.md`) and are loaded from `crates/goose/src/prompts/`. A prompt manager component handles selection based on the active model.
- Source: [Issue #3348](https://github.com/block/goose/issues/3348)

### Prompt Construction

**[Confirmed]** The system prompt is composed of multiple layers:
  1. **Base system prompt** — from `system.md` or model-specific variant
  2. **Extension tool descriptions** — dynamically injected based on enabled MCP extensions
  3. **`.goosehints` file** — project-specific instructions loaded once at session start (requires developer extension)
  4. **Persistent instructions** — re-read and injected fresh on every turn, ideal for behavioral guardrails
- Source: [Issue #1800](https://github.com/block/goose/issues/1800), [Discussion #2182](https://github.com/block/goose/discussions/2182)

### Customization Limitations

**[Confirmed]** Users cannot directly customize the core system prompt. The available extension points are:
  - `.goosehints` file (per-project, loaded once at session start)
  - Persistent instructions (per-turn injection)
  - `compaction.md` template (controls summarization behavior)
- Source: [Issue #1800](https://github.com/block/goose/issues/1800)

**[Inferred]** The system prompt is sent with every LLM call and is not subject to compaction — it remains constant throughout the session while conversation messages are summarized around it.
- Reasoning: Standard LLM agent architecture; the system prompt occupies a fixed portion of the context window.

---

## 7. Compaction Thresholds and Configuration

### Environment Variables

| Variable | Default | Description |
|---|---|---|
| `GOOSE_AUTO_COMPACT_THRESHOLD` | `0.8` (80%) | Fraction of context limit that triggers auto-compaction. Set to `0.0` to disable. |
| `GOOSE_CONTEXT_LIMIT` | Model-specific (see `MODEL_SPECIFIC_LIMITS`) | Override the total context window size in tokens. |
| `GOOSE_TOOL_CALL_CUTOFF` | `10` | Number of recent tool calls to keep in full detail; older outputs are summarized. |
| `GOOSE_CONTEXT_STRATEGY` | `summarize` (Desktop) | Fallback strategy when context limit is exceeded after compaction. CLI supports: `summarize`, `truncate`, `clear`, `prompt`. |

**[Confirmed]** Sources: [Smart Context Management docs](https://block.github.io/goose/docs/guides/sessions/smart-context-management/), [Issue #3259](https://github.com/block/goose/issues/3259)

### Context Limit Strategies (Fallback)

When auto-compaction is disabled or insufficient, the following strategies apply:

1. **Summarize** (Desktop default): Goose compacts the conversation, preserving key information while reducing size. Notification: `"Context maxed out - automatically summarized messages."`
2. **Truncate** (CLI option): Older messages are dropped entirely. Notification: `"Context maxed out - automatically truncated messages."`
3. **Clear** (CLI option): Entire session history is cleared. Notification: `"Context maxed out - automatically cleared session."`
4. **Prompt** (CLI option): User is prompted to choose an action.

**[Confirmed]** Goose Desktop exclusively uses the `summarize` strategy. The CLI supports all four.
- Source: [Smart Context Management docs](https://block.github.io/goose/docs/guides/sessions/smart-context-management/)

### Known Issues

**[Confirmed]** Manual `/compact` command does not update the displayed token count in the UI, while auto-compaction does. This is because auto-compaction uses the synchronous `complete()` response for metric synchronization, while the slash command response is not wired to the same logic.
- Source: [Issue #6588](https://github.com/block/goose/issues/6588)

**[Confirmed]** A bug was reported where Goose Desktop would immediately trigger compaction on brand-new sessions (before any user input) after removing a custom MCP extension.
- Source: [Issue #6146](https://github.com/block/goose/issues/6146)

---

## 8. Comparison with Other Agents

### Claude Code

| Aspect | Goose | Claude Code |
|---|---|---|
| **Compaction trigger** | 80% of context (configurable) | ~95-98% of context (configurable via `CLAUDE_AUTOCOMPACT_PCT_OVERRIDE`) |
| **Algorithm** | One-shot LLM summarization via `compaction.md` prompt | One-shot LLM summarization with structured compaction blocks |
| **History preservation** | Session JSONL persists full history; compacted summary replaces in-memory context | Full transcript preserved in storage; compaction block serves as checkpoint |
| **Manual trigger** | `/summarize` or `/compact` command | `/compact` command with optional custom instructions |
| **Tool output handling** | Background summarization of older tool outputs (>10 calls) | Not explicitly documented as a separate mechanism |
| **Fallback strategies** | Summarize, truncate, clear, prompt | Primarily summarization |

**[Confirmed]** Claude Code triggers auto-compaction much later (95-98%) compared to Goose's 80%, reflecting different risk tolerance. Claude Code also supports compaction boundaries that prevent recursive compaction failures.
- Source: [Claude Code docs](https://platform.claude.com/docs/en/build-with-claude/compaction), [DeepWiki Claude Code](https://deepwiki.com/anthropics/claude-code/3.3-session-and-conversation-management)

### Cursor

| Aspect | Goose | Cursor |
|---|---|---|
| **Context retrieval** | Full conversation history with compaction | RAG-based: semantic search via embeddings in Turbopuffer vector DB |
| **Codebase awareness** | Via tool calls (file read/search tools from extensions) | Automatic indexing + `@codebase` semantic retrieval |
| **Context selection** | All conversation is always in context (until compacted) | Selective: only relevant chunks retrieved per query |
| **Compaction** | LLM-based summarization | Not needed (context is assembled per-query, not accumulated) |

**[Confirmed]** Cursor uses a fundamentally different approach — RAG retrieval means it never faces the compaction problem in the same way. Each query gets freshly assembled context from the vector DB rather than carrying cumulative conversation history.
- Source: [How Cursor Indexes Your Codebase](https://towardsdatascience.com/how-cursor-actually-indexes-your-codebase/)

### Aider

| Aspect | Goose | Aider |
|---|---|---|
| **Context strategy** | Full conversation + compaction | Repo-map (treesitter AST) + selective file inclusion |
| **Codebase awareness** | Extension-provided tools | Structural map of entire codebase via treesitter |
| **Token efficiency** | Compaction reduces accumulated context | Repo-map provides compressed structural overview; only edited files are fully loaded |
| **Compaction** | LLM summarization | Not applicable (context is assembled per-turn from repo-map + selected files) |

**[Confirmed]** Aider uses treesitter and ripgrep for context fetching, building a structural map of the codebase. This is fundamentally different from Goose's approach of accumulating conversation history and compacting it.
- Source: [CLI tools comparison](https://sanj.dev/post/comparing-ai-cli-coding-assistants)

### OpenCode

| Aspect | Goose | OpenCode |
|---|---|---|
| **History model** | Session JSONL with full history | Full session stored; model receives curated slice |
| **Compaction** | Replaces older messages with summary in active context | Summary becomes new anchor; older messages remain stored |
| **Trigger** | 80% threshold | When context usage exceeds model limit |

**[Confirmed]** OpenCode separates stored history from model context, allowing compaction and pruning without deleting history. The summary becomes the new anchor for subsequent interactions.
- Source: [Context management gist](https://gist.github.com/migom6/70ccd3485ea4db9dd8039245cd9dde4a)

### Summary of Approaches

| Agent | Strategy | Context Assembly | Compaction |
|---|---|---|---|
| **Goose** | Accumulate + compact | Full conversation in context | LLM summarization at 80% threshold |
| **Claude Code** | Accumulate + compact | Full conversation in context | LLM summarization at ~95% threshold |
| **Cursor** | Retrieve per-query | RAG from vector DB | Not needed |
| **Aider** | Assemble per-turn | Repo-map + selected files | Not needed |
| **OpenCode** | Accumulate + compact | Curated slice of stored history | LLM summarization at limit |

---

## Key Architectural Insights

1. **Two-tier defense**: Goose's layered approach (background tool summarization -> auto-compaction -> fallback strategy) is more robust than single-mechanism systems.

2. **Early trigger, conservative approach**: The 80% default threshold is notably more conservative than Claude Code's ~95%. This reduces the risk of context overflow but means more frequent compaction events, which can cause information loss.

3. **Tool output as the main bloat vector**: Goose's dedicated tool output summarization (separate from full conversation compaction) reflects the reality that tool results (file contents, command output) are typically the largest context consumers.

4. **Provider-dependent reliability**: Token counting accuracy varies significantly by provider, with Gemini and OpenRouter reporting 0% usage. This is a fundamental limitation since compaction timing depends on accurate token counts.

5. **Customizable compaction prompt**: The `compaction.md` template allows users to control what information is preserved during summarization, which is a differentiating feature compared to systems with hardcoded compaction prompts.

---

## Areas of Uncertainty

- **Exact implementation of `check_if_compaction_needed`**: The function name appears in architectural references but the exact implementation logic (beyond threshold checking) is not publicly documented in detail. **[Unknown]**
- **Compaction prompt content**: The default `compaction.md` template content is not publicly documented outside the source code. **[Unknown]**
- **Token estimation for non-OpenAI models**: Whether Goose uses tiktoken, character estimation, or provider-specific tokenizers for local/non-OpenAI models. **[Unknown]**
- **Interaction between compaction and persistent instructions**: Whether persistent instructions are included in the compaction summary or re-injected independently. **[Inferred: re-injected independently, since they are described as "re-read and injected fresh with every interaction"]**

---

## Sources

- [Smart Context Management | Goose Docs](https://block.github.io/goose/docs/guides/sessions/smart-context-management/)
- [Goose GitHub Repository](https://github.com/block/goose)
- [token_counter.rs source](https://github.com/block/goose/blob/main/crates/goose/src/token_counter.rs)
- [system.md prompt](https://github.com/block/goose/blob/main/crates/goose/src/prompts/system.md)
- [Issue #3485: Improve Context Window Management](https://github.com/block/goose/issues/3485)
- [Issue #4986: Context Length Exceeded](https://github.com/block/goose/issues/4986)
- [Issue #6588: Manual compact does not update tokens](https://github.com/block/goose/issues/6588)
- [Issue #6146: Auto-compacts new sessions](https://github.com/block/goose/issues/6146)
- [Issue #3259: Env variable for context window limits](https://github.com/block/goose/issues/3259)
- [Issue #1253: Ollama context size configuration](https://github.com/block/goose/issues/1253)
- [Issue #6185: Devstral incorrect context length](https://github.com/block/goose/issues/6185)
- [Issue #2925: GitHub Copilot GPT 4.1 token limits](https://github.com/block/goose/issues/2925)
- [Issue #1800: Allow user to customize Goose prompt](https://github.com/block/goose/issues/1800)
- [Issue #3348: GPT 4.1 system prompt not used](https://github.com/block/goose/issues/3348)
- [Discussion #3319: Goose Roadmap (July 2025)](https://github.com/block/goose/discussions/3319)
- [DeepWiki: block/goose](https://deepwiki.com/block/goose)
- [DeepWiki: Provider Configuration](https://deepwiki.com/block/goose/2.2-provider-configuration)
- [Context Compaction Research Gist (badlogic)](https://gist.github.com/badlogic/cd2ef65b0697c4dbe2d13fbecb0a0a5f)
- [OpenCode Context Management Gist](https://gist.github.com/migom6/70ccd3485ea4db9dd8039245cd9dde4a)
- [Claude Code Compaction Docs](https://platform.claude.com/docs/en/build-with-claude/compaction)
- [How Cursor Indexes Your Codebase](https://towardsdatascience.com/how-cursor-actually-indexes-your-codebase/)
- [Claude Code vs Goose vs Aider Comparison](https://sanj.dev/post/comparing-ai-cli-coding-assistants)
