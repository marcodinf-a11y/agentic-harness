# Goose Core Architecture — Deep Dive Research

**Date:** 2026-03-07
**Repository:** https://github.com/block/goose
**Primary Language:** Rust (57.2%), TypeScript (34.3%)
**License:** Apache-2.0

---

## Table of Contents

1. [Crate/Module Structure](#1-cratemodule-structure)
2. [Agent Loop](#2-agent-loop)
3. [State Management](#3-state-management)
4. [Error Handling and Retry Logic](#4-error-handling-and-retry-logic)
5. [Configuration System](#5-configuration-system)
6. [Server Mode (goosed)](#6-server-mode-goosed)
7. [Platform Support](#7-platform-support)
8. [Summary of Unknowns](#summary-of-unknowns)
9. [Sources](#sources)

---

## 1. Crate/Module Structure

### Workspace Layout

The workspace is defined in the root `Cargo.toml` with `members = ["crates/*"]` and `resolver = "2"`. All crates live under `crates/`. The workspace uses edition 2021 and centralizes dependency versions at the workspace level (50+ shared dependencies).

**[Confirmed — root Cargo.toml on GitHub]**

### Crate Inventory

| Crate | Role | Key Dependencies | Binary |
|-------|------|------------------|--------|
| **`goose`** | Core agent logic: agent loop, provider system, extension management, session management, prompt construction, configuration | tokio, axum, reqwest, serde, sqlx, tree-sitter (9 languages), candle-core/nn/transformers, llama-cpp-2, rmcp, aws-sdk-*, minijinja, keyring | `analyze_cli`, `build_canonical_models`, `agent`, `databricks_oauth` (examples/bins) |
| **`goose-cli`** | CLI entry point (`goose` binary). Interactive terminal UI, command parsing, desktop launch | goose, goose-acp, goose-mcp, rmcp, clap, cliclack, console, bat, rustyline, indicatif | `goose`, `generate_manpages` |
| **`goose-server`** | HTTP/SSE backend server. REST API for desktop app and programmatic access | goose, goose-mcp, axum, utoipa (OpenAPI), tokio-tungstenite (WebSocket), reqwest | `goosed`, `generate_schema` |
| **`goose-acp`** | Agent Communication Protocol implementation. Bridges goose agent to ACP-over-HTTP clients | goose, goose-mcp, sacp v10.1, agent-client-protocol-schema v0.10, rmcp, axum (WebSocket), futures | `goose-acp-server`, `generate-acp-schema` |
| **`goose-acp-macros`** | Procedural macros for ACP. Code generation for protocol boilerplate | proc-macro2, quote, syn (full features) | — (proc-macro crate) |
| **`goose-mcp`** | MCP extension implementations. Built-in tools: file ops, code analysis, screen capture, document processing | rmcp, tree-sitter (8 languages), lopdf, docx-rs, umya-spreadsheet, image, xcap, rayon | — |
| **`goose-test`** | Test capture utility | clap, serde_json | `capture` |
| **`goose-test-support`** | Test helpers and MCP fixture servers | axum 0.7, rmcp (server, macros, http-streaming), tokio | — (has `mcp_fixture_server` example) |

**[Confirmed — individual Cargo.toml files fetched from GitHub]**

### Dependency Graph (Simplified)

```
goose-cli ──────┐
goose-server ───┤
goose-acp ──────┼──> goose (core) ──> providers (30+ modules)
                │         │
                │         ├──> agents/ (agent loop, extension_manager, prompt_manager, retry, ...)
                │         ├──> session/ (SessionManager, SessionStorage, SQLite)
                │         └──> config/ (configuration system)
                │
                └──> goose-mcp (built-in extensions)

goose-acp-macros ──> (proc-macro, used by goose-acp)
goose-test-support ──> rmcp (test MCP servers)
```

**[Inferred — from dependency lists in each crate's Cargo.toml]**

### Internal Module Structure of `goose` Crate

The `agents/mod.rs` file reveals a rich internal structure:

**Private modules:** `agent`, `builtin_skills`, `large_response_handler`, `reply_parts`, `schedule_tool`, `subagent_handler`, `subagent_task_config`, `tool_execution`

**Public modules:** `container`, `execute_commands`, `extension`, `extension_malware_check`, `extension_manager`, `final_output_tool`, `mcp_client`, `moim`, `platform_extensions`, `platform_tools`, `prompt_manager`, `retry`, `subagent_execution_tool`, `types`, `validate_extensions`

**Key public exports:** `Agent`, `AgentConfig`, `AgentEvent`, `ExtensionLoadResult`, `GoosePlatform`, `Container`, `ExtensionConfig`, `ExtensionError`, `ExtensionManager`, `PromptManager`, `TaskConfig`, `FrontendTool`, `RetryConfig`, `SessionConfig`, `SuccessCheck`

**[Confirmed — agents/mod.rs fetched from GitHub]**

---

## 2. Agent Loop

### Overview

The agent loop follows a **ReAct-style** (Reason + Act) cycle. The LLM reasons about the task, emits tool call requests, goose executes them, feeds results back, and the loop continues until the task is complete or limits are hit.

### The `Agent` Struct

```rust
pub struct Agent {
    pub provider: SharedProvider,              // Arc<Mutex<Option<Arc<dyn Provider>>>>
    pub config: AgentConfig,
    pub extension_manager: Arc<ExtensionManager>,
    pub final_output_tool: Arc<Mutex<Option<FinalOutputTool>>>,
    pub frontend_tools: Mutex<HashMap<String, FrontendTool>>,
    pub prompt_manager: Mutex<PromptManager>,
    pub confirmation_tx/rx: mpsc channels,     // permission confirmation channels
    pub tool_result_tx/rx: mpsc channels,      // frontend tool result channels
    pub retry_manager: RetryManager,
    pub tool_inspection_manager: ToolInspectionManager,
}
```

**[Confirmed — agent.rs fetched from GitHub]**

### `reply()` — Outer Entry Point

The `reply()` method is the public entry point. It implements a **streaming async generator pattern** that yields `AgentEvent` items. Its flow:

1. **Slash command execution** — Processes any `/` commands via `execute_command()`
2. **Auto-compaction check** — Calls `check_if_compaction_needed()` to determine if context window is too full
3. **Context compaction** — If token count exceeds configured limits, compacts the conversation (summarization) before proceeding
4. **Delegation** — Hands off to `reply_internal()` for the core agent loop

**[Confirmed — agent.rs source]**

### `reply_internal()` — The Core ReAct Loop

This is the heart of the agent. It iterates through conversation turns, capped at `max_turns` (default: **1000**, configurable via `DEFAULT_MAX_TURNS`).

Each iteration:

1. **Stream LLM response** — Calls `stream_response_from_provider()`, which sends the conversation to the configured LLM provider and yields streaming chunks as `AgentEvent` items.

2. **Tool categorization** — The response may contain tool call requests. These are split by `categorize_tools()` into:
   - **Extension tools** — Tools provided by MCP extensions
   - **Frontend tools** — Tools that require UI interaction (delegated to the frontend)
   - **Platform/schedule tools** — Handled locally by the agent

3. **Tool inspection chain** — Before execution, each tool call passes through an inspection pipeline:
   ```
   SecurityInspector (highest priority)
     -> PermissionInspector (with provider-specific permission routing)
       -> RepetitionInspector
   ```
   Results are categorized as: **approved** (execute immediately), **needs_approval** (await user confirmation), or **denied** (reject).

4. **Tool dispatch** — `dispatch_tool_call()` routes to the appropriate handler:
   - Platform schedule management -> handled locally
   - Final output tool -> stored in agent state
   - Extension/frontend tools -> delegated to `ExtensionManager` or frontend channels

5. **Result aggregation** — Tool outputs are collected, converted to tool response messages, and appended to the conversation.

6. **Exit condition check** — The loop breaks if:
   - A **final output tool** has been called (structured output mode)
   - **Max turns** reached
   - **Retry logic** triggers a restart
   - Provider returns no tool calls (natural completion)

**[Confirmed — agent.rs source]**

### Tool Dispatch in Detail (`ExtensionManager`)

The `ExtensionManager` resolves and dispatches tool calls through:

1. **Name parsing** — Tool names use a prefix convention: `extension__toolname` for regular extensions, unprefixed for first-class platform extensions. Controlled via `TOOL_EXTENSION_META_KEY` metadata.

2. **Cache lookup** — Tools are looked up in a versioned cache with atomic invalidation to prevent stale listings.

3. **Client resolution** — The owning extension's MCP client is retrieved.

4. **Allowlist check** — Tool is validated against the extension's allowlist.

5. **Execution** — The MCP `call_tool` method is invoked on the appropriate client.

Extension types supported:
- **Platform/Builtin** — In-process clients or subprocess launches
- **StreamableHttp** — Remote MCP servers with OAuth + header templating
- **Stdio** — Child processes with environment variable injection (`${VAR}` and `$VAR` syntax)
- **InlinePython** — Temporary Python files executed via `uvx`

**[Confirmed — extension_manager.rs fetched from GitHub]**

### Prompt Construction (`PromptManager`)

The `PromptManager` uses a builder pattern (`SystemPromptBuilder`) to assemble system prompts:

- `with_extension()` / `with_extensions()` — Registers available tools
- `with_extension_and_tool_counts()` — Tracks resource limits
- `with_hints()` — Loads contextual files (e.g., `.goosehints`) from the working directory
- `with_code_execution_mode()` — Enables/disables execution capabilities

Hard limits are enforced: **MAX_EXTENSIONS = 5**, **MAX_TOOLS = 50**. Tools are sorted alphabetically for stable prompt caching across sessions.

All user-provided content undergoes **Unicode tag sanitization** to protect against prompt injection attacks (e.g., `\u{E0041}` characters are stripped).

**[Confirmed — prompt_manager.rs fetched from GitHub]**

### Constants

| Constant | Value | Notes |
|----------|-------|-------|
| `DEFAULT_MAX_TURNS` | 1000 | Maximum agent loop iterations |
| `MAX_EXTENSIONS` | 5 | In prompt construction |
| `MAX_TOOLS` | 50 | In prompt construction |
| `GOOSE_TOOL_CALL_CUTOFF` | 10 (default) | Configurable via env var |
| `COMPACTION_THINKING_TEXT` | `"goose is compacting..."` | Displayed during compaction |

**[Confirmed — agent.rs, prompt_manager.rs sources]**

---

## 3. State Management

### Session Model

Sessions are the primary unit of state. Each session represents a continuous interaction between user and agent.

**Session ID format:** `YYYYMMDD_<COUNT>` (e.g., `20260213_9`)

**Storage:** SQLite database at `~/.local/share/goose/sessions/sessions.db` (migrated from individual `.jsonl` files in v1.10.0).

**[Confirmed — Goose documentation and SessionManager source]**

### Database Schema

The `SessionManager` delegates to `SessionStorage`, which manages two primary tables:

**Sessions table:**
- UUID-based identity
- User-provided and system-generated names
- Creation and update timestamps
- Current and accumulated token counts
- Recipe values, provider info, model settings
- Working directory and session type

**Messages table:**
- `message_id TEXT` — unique identifier
- `session_id` — foreign key to sessions
- Role classification (user/assistant)
- JSON-serialized content
- Metadata JSON (runtime annotations)
- Timestamps and token usage

**Indexes:** `idx_messages_session ON messages(session_id)` for efficient retrieval.

**Ordering:** `ORDER BY created_timestamp, id` maintains causality for same-second events.

**Database configuration:**
- WAL (Write-Ahead Logging) mode for concurrent read/write
- 30-second busy timeout
- `BEGIN IMMEDIATE` transactions to prevent lock-upgrade deadlocks

**[Confirmed — session_manager.rs fetched from GitHub]**

### Singleton Pattern

The `SessionStorage` uses a lazy-loaded static instance:
```rust
static SESSION_STORAGE: LazyLock<Arc<SessionStorage>>
```
This ensures a single database connection pool is shared across the application.

**[Confirmed — session_manager.rs source]**

### Conversation State Flow

1. **Message persistence** — Messages are saved after each turn via `SessionManager`
2. **Atomic replacement** — `replace_conversation_inner()` deletes and reinserts all messages (used during compaction)
3. **Metadata updates** — Runtime annotations added via `UPDATE messages SET metadata_json = ?`
4. **Cross-platform resumption** — Sessions created in Desktop can be resumed in CLI and vice versa (shared SQLite DB)

### Message Types and Content

Messages use role-based classification (user/assistant) with JSON-serialized content. The streaming system emits typed `AgentEvent` variants:

- `Message` — Standard content
- `Error` — Error notifications
- `Finish` — Completion signal
- `ModelChange` — Provider/model switch notification
- `Notification` — System notifications
- `UpdateConversation` — Conversation state updates
- `Ping` — Heartbeat (500ms interval)

**Content preservation rules:**
- Thinking content (Gemini) is echoed back in a separate message
- Reasoning content (Kimi/DeepSeek) is attached to all split tool request messages
- Tool responses are paired with original requests for context

**Visibility control:**
- User messages after commands: `visible_to_user=true, visible_to_model=false`
- Command responses: `visible_to_user=true, visible_to_model=false`
- System compaction messages use `SystemNotificationType` enum

**[Confirmed — agent.rs, reply.rs sources]**

### Extension State

Extension configurations are serialized via `EnabledExtensionsState::to_extension_data()` and persisted alongside session data. When extensions change mid-conversation, tools and prompts are re-prepared via `prepare_tools_and_prompt()`.

Extensions are loaded per-session via `load_extensions_from_session()`, which skips already-enabled extensions and persists state changes via `persist_extension_state()`.

**[Confirmed — agent.rs source]**

---

## 4. Error Handling and Retry Logic

### Provider Error Taxonomy

The `ProviderError` enum defines 10 error variants:

| Variant | Description | Agent Behavior |
|---------|-------------|----------------|
| `Authentication` | Invalid credentials or permissions | Notify user, break loop |
| `ContextLengthExceeded` | Input exceeds model context window | Compact conversation, retry (max 2 attempts) |
| `RateLimitExceeded` | Request throttled | Contains optional `retry_delay` Duration |
| `ServerError` | Provider-side failure | Log + notify user |
| `NetworkError` | Connection issues | Yield user-facing message, break loop |
| `RequestFailed` | HTTP request problems | Log + notify user |
| `ExecutionError` | Runtime failures | Log + notify user |
| `UsageError` | Data-related issues | Log + notify user |
| `NotImplemented` | Unsupported operations | Return error |
| `CreditsExhausted` | Account balance depleted | Emit `SystemNotification` with top-up URL |

Each error maps to a telemetry classification via `telemetry_type()` for monitoring/analytics (PostHog integration via `crate::posthog`).

**[Confirmed — errors.rs fetched from GitHub]**

### Error Conversion Chain

```
reqwest::Error -> ProviderError
  - Connection check -> NetworkError (with host/port details)
  - Timeout -> NetworkError
  - Other HTTP -> RequestFailed (with status code)

anyhow::Error -> ProviderError
  - Unwraps nested reqwest::Error first
  - Falls back to ExecutionError
```

**[Confirmed — errors.rs source]**

### Tool Call Error Handling

When tool calls fail, errors are **not terminal**. Instead:
1. The error is converted to a tool response message
2. The response is sent back to the LLM with error details
3. The LLM receives the error context and can adjust its approach
4. This enables self-correction: invalid JSON, missing tools, and execution errors are all recoverable

Specific handling:
- **Declined tools** — When a user denies permission, the message "The user has declined to run this tool" is sent back to the model
- **Invalid tool calls** — Malformed JSON or unknown tool names are returned as error responses, giving the LLM information to correct itself

**[Confirmed — agent.rs, tool_execution.rs sources]**

### Provider-Level Retry

Rate limit handling uses a **caller-side retry pattern**:
1. The provider response handler detects `429 Too Many Requests`
2. For Google APIs, it parses `RetryInfo` from error details to extract `retryDelay`
3. Returns `ProviderError::RateLimitExceeded` with the optional delay
4. The caller implements the actual backoff strategy

The workspace exports `retry_operation` and `RetryConfig` as public interfaces for configurable retry across provider calls.

**[Confirmed — utils.rs, errors.rs sources]**

### Agent-Level Retry (Task Retry)

The `RetryManager` provides task-level retry for automated/scheduled runs:

```rust
pub struct RetryConfig {
    pub max_retries: u32,
    pub checks: Vec<SuccessCheck>,        // e.g., SuccessCheck::Shell { command: String }
    pub on_failure: Option<String>,        // cleanup command
    pub timeout_seconds: Option<u64>,
    pub on_failure_timeout_seconds: Option<u64>,
}
```

Retry flow:
1. After the agent produces output, **success checks** are run (shell commands that validate task completion)
2. If checks fail and attempts remain below `max_retries`, the system:
   - Executes optional `on_failure` cleanup command
   - Resets message history to initial state
   - Clears final output tool state
   - Increments attempt counter
   - Re-enters the agent loop
3. If the final output tool hasn't been called, a `FINAL_OUTPUT_CONTINUATION_MESSAGE` is injected

**[Confirmed — retry.rs fetched from GitHub]**

### Context Length Recovery

When `ContextLengthExceeded` is thrown:
1. The agent compacts the conversation (summarization)
2. Retries with the compacted context
3. Maximum of 2 compaction attempts before giving up

**[Confirmed — agent.rs source]**

---

## 5. Configuration System

### Configuration Files

All configuration lives under `~/.config/goose/`:

| File | Purpose | Format |
|------|---------|--------|
| `config.yaml` | Main configuration: provider, model, extensions, general settings | YAML |
| `permission.yaml` | Tool permission levels | YAML |
| `secrets.yaml` | API keys and secrets (plain text, file-based storage mode) | YAML |
| `init-config.yaml` | Bootstrap/default configuration (rarely modified) | YAML |
| `tool_permissions.json` | Runtime permission decisions (auto-managed) | JSON |

**[Confirmed — Goose documentation site]**

### Key Configuration Parameters

| Key | Env Var | Description | Default |
|-----|---------|-------------|---------|
| `GOOSE_PROVIDER` | `GOOSE_PROVIDER` | LLM provider name (e.g., `"anthropic"`, `"openai"`) | — |
| `GOOSE_MODEL` | `GOOSE_MODEL` | Model identifier (e.g., `"claude-4.5-sonnet"`) | — |
| `GOOSE_TEMPERATURE` | `GOOSE_TEMPERATURE` | Sampling temperature | Provider default |
| `GOOSE_PLANNER_PROVIDER` | `GOOSE_PLANNER_PROVIDER` | Separate provider for planning phase | Same as main |
| `GOOSE_PLANNER_MODEL` | `GOOSE_PLANNER_MODEL` | Separate model for planning | Same as main |
| `GOOSE_MODE` | `GOOSE_MODE` | Operating mode (e.g., `"auto"`) | — |
| `GOOSE_TOOL_CALL_CUTOFF` | `GOOSE_TOOL_CALL_CUTOFF` | Max tool calls per turn | 10 |

**[Confirmed — Goose documentation, search results]**

### Configuration Precedence

Priority order (highest to lowest):
1. **Environment variables** — Checked first via `std::env::var(&key.name)`
2. **Existing config** — Retrieved from `config.get_secret()` or `config.get_param()`
3. **OAuth flow** — If `key.oauth_flow == true`, initiates device code authentication
4. **Manual entry** — User prompted via CLI

**[Confirmed — DeepWiki provider configuration page]**

### Extension Configuration

Extensions are configured under the `extensions` key in `config.yaml`:

```yaml
extensions:
  - name: my-extension
    type: stdio           # stdio | streamable_http | inline_python
    enabled: true
    bundled: false
    timeout: 30
    cmd: my-tool-binary
    args: ["--flag"]
    description: "My custom extension"
    env:
      MY_VAR: "value"
```

Extension types:
- **stdio** — Child process communicating via stdin/stdout MCP protocol
- **streamable_http** — Remote MCP server via HTTP with SSE streaming
- **inline_python** — Python code executed via `uvx`

Environment variable substitution is supported in headers and arguments using `${VAR}` and `$VAR` syntax.

**[Confirmed — extension_manager.rs, Goose documentation]**

### CLI Configuration Command

`goose configure` provides an interactive setup wizard that writes to `config.yaml`. The CLI also supports `--provider` and `--model` flags for one-off overrides.

**[Confirmed — search results, GitHub issues]**

### Feature Flags (Compile-Time)

| Feature | Description |
|---------|-------------|
| `code-mode` | Default. Enables code execution capabilities |
| `cuda` | GPU acceleration via CUDA for candle and llama-cpp-2 |
| `disable-update` | Disables auto-update checks (goose-cli) |

**[Confirmed — Cargo.toml files]**

---

## 6. Server Mode (goosed)

### Overview

The `goose-server` crate produces the `goosed` binary — an HTTP server that enables programmatic access to the agent. It is consumed by the Electron desktop app and can be used for CI/CD integration.

The server uses **Axum** as its web framework with **utoipa** for OpenAPI schema generation.

**[Confirmed — goose-server Cargo.toml, main.rs]**

### CLI Commands

The `goosed` binary supports three subcommands:

1. **`agent`** — Run the agent HTTP server (primary mode)
2. **`mcp`** — Run as an MCP server (4 server types: AutoVisualiserRouter, ComputerControllerServer, MemoryServer, TutorialServer)
3. **`validate-extensions`** — Validate a bundled-extensions JSON file

**[Confirmed — main.rs fetched from GitHub]**

### Application State

```rust
pub struct AppState {
    pub agent_manager: Arc<AgentManager>,
    pub recipe_file_hash_map: Arc<Mutex<HashMap<String, PathBuf>>>,
    recipe_session_tracker: Arc<Mutex<HashSet<String>>>,
    pub tunnel_manager: Arc<TunnelManager>,
    pub gateway_manager: Arc<GatewayManager>,
    pub extension_loading_tasks: ExtensionLoadingTasks,
    pub inference_runtime: Arc<InferenceRuntime>,
}
```

Key design points:
- **AgentManager** handles per-session agent creation and retrieval (`get_or_create_agent()`)
- **Recipe tracking** monitors active recipe sessions
- **TunnelManager/GatewayManager** manage network infrastructure
- **InferenceRuntime** — Shared local model inference engine
- Errors surface as HTTP 500 responses

**[Confirmed — state.rs fetched from GitHub]**

### API Endpoints

The server merges 16+ route modules. Key endpoints:

#### Reply (Core Agent Interaction)

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/reply` | Send a message and receive streaming SSE response |

Request body (`ChatRequest`):
- `user_message` — The user's input
- `session_id` — Session identifier
- `override_conversation` — Optional full conversation override (admin)
- `recipe_name`, `recipe_version` — Optional recipe metadata

Body size limit: **50 MB**

**[Confirmed — reply.rs fetched from GitHub]**

#### Session Management

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/sessions` | List all sessions |
| `GET` | `/sessions/{session_id}` | Get specific session |
| `GET` | `/sessions/insights` | Aggregated session insights |
| `GET` | `/sessions/{session_id}/export` | Export session data |
| `GET` | `/sessions/{session_id}/extensions` | Get session's extensions |
| `GET` | `/sessions/search` | Search sessions (max 50 results) |
| `PUT` | `/sessions/{session_id}/name` | Update session name (max 200 chars) |
| `PUT` | `/sessions/{session_id}/user_recipe_values` | Modify recipe parameters |
| `POST` | `/sessions/import` | Import session from JSON (max 25 MB) |
| `POST` | `/sessions/{session_id}/fork` | Fork/duplicate session |
| `DELETE` | `/sessions/{session_id}` | Delete session |

**[Confirmed — session.rs fetched from GitHub]**

#### Other Route Modules

| Module | Purpose |
|--------|---------|
| `status` | Health check / server status |
| `action_required` | User confirmation handling |
| `agent` | Agent lifecycle management |
| `config_management` | Runtime configuration |
| `dictation` | Voice input support |
| `local_inference` | Local model inference |
| `prompts` | Prompt management |
| `recipe` | Recipe execution |
| `schedule` | Scheduled task management |
| `setup` | Initial setup flows |
| `telemetry` | Usage telemetry |
| `tunnel` | Network tunneling |
| `gateway` | Gateway proxy |
| `mcp_ui_proxy` | MCP UI proxy (uses secret key) |
| `mcp_app_proxy` | MCP app proxy (uses secret key) |
| `sampling` | LLM sampling configuration |

**[Confirmed — routes/mod.rs fetched from GitHub]**

### SSE Streaming Implementation

The `SseResponse` struct wraps a `ReceiverStream<String>` and implements Axum's `IntoResponse`:

- HTTP 200 with headers: `Content-Type: text/event-stream`, `Cache-Control: no-cache`, `Connection: keep-alive`
- Messages formatted as `data: {json}\n\n`
- Heartbeat `Ping` events every **500ms**

Stream event types: `Message`, `Error`, `Finish`, `ModelChange`, `Notification`, `UpdateConversation`, `Ping`

**[Confirmed — reply.rs source]**

### Authentication

API routes are protected with a secret key passed via the `X-Secret-Key` header.

**[Confirmed — routes/mod.rs, search results]**

### ACP Protocol (Transition in Progress)

The project is actively transitioning from the custom REST+SSE API to a standards-based **ACP (Agent Communication Protocol) over HTTP** interface (tracked in [issue #6642](https://github.com/block/goose/issues/6642)):

- `POST /acp` with `Acp-Session-Id` header
- Methods: `"initialize"`, `"session/new"`, `"session/prompt"`
- Each request opens a new SSE stream for the response

**[Confirmed — GitHub issue #6642]**

### CI/CD Integration Pattern

For CI/CD use, `goosed` can be started as a background process, and the REST API can be used to:
1. Create sessions programmatically
2. Send prompts via `POST /reply`
3. Stream responses via SSE
4. Use retry configuration with success checks for validation
5. Fork/export sessions for audit trails

The per-session agent isolation means concurrent sessions do not interfere with each other — enabling/disabling extensions in one session never affects another.

**[Inferred — from API endpoints, session isolation design, and retry configuration with shell-based success checks]**

---

## 7. Platform Support

### Supported Operating Systems

| Platform | CLI | Desktop (Electron) | Installation Methods |
|----------|-----|---------------------|---------------------|
| **macOS** (Silicon + Intel) | Yes | Yes | Homebrew (`brew install --cask block-goose`), direct download, shell script |
| **Linux** | Yes | Yes | DEB (Ubuntu/Debian), RPM (RHEL/Fedora), Flatpak, shell script |
| **Windows** | Yes | Yes (native) | Direct download, WSL also supported |

**[Confirmed — Goose installation documentation]**

### System Requirements

- No external runtime required (Rust-compiled binaries for CLI, bundled Electron for Desktop)
- Internet access needed for LLM provider APIs
- Minimum ~100 MB storage for application and session data

**[Confirmed — DeepWiki installation page]**

### Platform-Specific Code Paths

**macOS:**
- Metal GPU acceleration for `candle` and `llama-cpp-2` (local inference)
- Conditional compilation via `#[cfg(target_os = "macos")]`

**Windows:**
- `winreg` dependency in `goose-server` for Windows registry operations
- WinAPI credential management in the `goose` crate (via `keyring` crate)
- `goose-server/Cargo.toml` includes `[target.'cfg(windows)'.dependencies]` for `winreg`

**Linux:**
- Standard POSIX paths, no special platform code identified beyond packaging

**GPU Support:**
- `cuda` feature flag enables NVIDIA GPU acceleration for local inference
- Metal (macOS) enabled automatically in relevant crates

**[Confirmed — Cargo.toml platform-specific dependency sections]**

### Patched Dependencies

The workspace patches the `v8` crate to use a local vendor directory (`vendor/v8`), likely for deterministic builds or custom modifications.

**[Confirmed — root Cargo.toml]**

### Cross-Platform Session Sharing

Sessions created in Desktop can be resumed in CLI and vice versa because they share the same SQLite database. This works across the same machine only (not across platforms).

**[Confirmed — Goose session management documentation]**

---

## Summary of Unknowns

| Area | What is Unknown | Why |
|------|-----------------|-----|
| Provider trait definition | Exact trait signature for `Provider` | The `providers/mod.rs` source showed the registry pattern but not the trait body |
| `AgentConfig` struct | Full field list | Not included in the `types.rs` excerpt retrieved |
| `AgentEvent` enum | Complete variant list | Only partially revealed through SSE event types in `reply.rs` |
| Compaction algorithm | How conversation summarization works internally | Not in any fetched source file |
| Elicitation system | Full `ActionRequiredManager` implementation | Referenced in agent.rs but not fetched |
| Container system | How `Container` isolation works | Module exists but not investigated |
| Subagent system | How subagent execution and task decomposition works | Modules exist (`subagent_handler`, `subagent_execution_tool`) but not fetched |
| `moim` module | Purpose unclear from name alone | Listed in public modules but not investigated |

---

## Sources

### Primary Sources (GitHub)
- [block/goose repository](https://github.com/block/goose)
- [Root Cargo.toml](https://github.com/block/goose/blob/main/Cargo.toml) — workspace configuration
- [AGENTS.md](https://github.com/block/goose/blob/main/AGENTS.md) — development guidelines and architecture overview
- [crates/ directory](https://github.com/block/goose/tree/main/crates) — all 8 workspace crates
- [goosed to ACP-over-HTTP issue #6642](https://github.com/block/goose/issues/6642) — ACP transition plans
- [Unify Agent Execution discussion #4389](https://github.com/block/goose/discussions/4389) — per-session agent design

### Documentation
- [Goose Architecture](https://block.github.io/goose/docs/goose-architecture/) — official architecture overview
- [Configuration Files](https://block.github.io/goose/docs/guides/config-files/) — config system documentation
- [Session Management](https://block.github.io/goose/docs/guides/sessions/session-management/) — session documentation

### Secondary Sources
- [DeepWiki — block/goose](https://deepwiki.com/block/goose) — AI-generated architecture analysis
- [DeepWiki — Installation and Setup](https://deepwiki.com/block/goose/2-installation-and-setup)
- [DeepWiki — Provider Configuration](https://deepwiki.com/block/goose/2.2-provider-configuration)
- [Configuring goose for Team Environments (DEV Community)](https://dev.to/lymah/configuring-goose-for-team-environments-and-shared-workflows-5ehn)
- [Setting Up Codename Goose in WSL (Scott Spence)](https://scottspence.com/posts/setting-up-codename-goose-in-wsl)

### Source Files Fetched (via raw.githubusercontent.com)
- `crates/goose/src/agents/agent.rs` — Agent struct and reply loop
- `crates/goose/src/agents/mod.rs` — module declarations and public API
- `crates/goose/src/agents/extension_manager.rs` — tool dispatch and extension lifecycle
- `crates/goose/src/agents/prompt_manager.rs` — system prompt construction
- `crates/goose/src/agents/tool_execution.rs` — tool dispatch and permission handling
- `crates/goose/src/agents/retry.rs` — RetryManager implementation
- `crates/goose/src/providers/errors.rs` — ProviderError enum
- `crates/goose/src/session/session_manager.rs` — SQLite session storage
- `crates/goose-server/src/main.rs` — server entry point
- `crates/goose-server/src/state.rs` — AppState struct
- `crates/goose-server/src/routes/mod.rs` — route configuration
- `crates/goose-server/src/routes/reply.rs` — SSE streaming reply endpoint
- `crates/goose-server/src/routes/session.rs` — session CRUD endpoints
- `crates/goose-mcp/Cargo.toml` — MCP extensions crate config
- `crates/goose-test-support/Cargo.toml` — test support crate config
