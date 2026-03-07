# Goose Extension & MCP System -- Deep Dive

**Date:** 2026-03-07
**Repo:** https://github.com/block/goose (main branch)
**Scope:** Extension architecture, MCP integration, permission model, security systems

---

## 1. Extension Types

Goose supports **seven** distinct extension types, defined as variants of the `ExtensionConfig` enum in `crates/goose/src/agents/extension.rs`. Each corresponds to a different transport or execution model for MCP communication.

### 1.1 Builtin Extensions

**[Confirmed]** ([source: extension.rs](https://github.com/block/goose/blob/main/crates/goose/src/agents/extension.rs))

Builtin extensions are compiled MCP servers that ship inside the Goose binary. They run in-process using `tokio::io::DuplexStream` pairs (64KB buffers) -- no child process is spawned. They are defined in the `goose-mcp` crate (`crates/goose-mcp/src/lib.rs`) via the `BUILTIN_EXTENSIONS` map.

Four builtin extensions exist:
- **autovisualiser** -- chart/visualization rendering
- **computercontroller** -- browser automation, PDF/DOCX/XLSX processing
- **memory** -- persistent memory/recall
- **tutorial** -- interactive tutorials

Configuration example:
```yaml
extensions:
  developer:
    enabled: true
    type: builtin
    name: developer
    timeout: 300
```

When to use: Default tooling that ships with Goose. Zero configuration needed.

### 1.2 Platform Extensions

**[Confirmed]** ([source: platform_extensions/mod.rs](https://github.com/block/goose/blob/main/crates/goose/src/agents/platform_extensions/mod.rs))

Platform extensions are special in-process extensions defined in the `PLATFORM_EXTENSIONS` lazy static map. Unlike builtin extensions, they implement `McpClientTrait` directly (not through MCP transport) and have access to the `PlatformExtensionContext` which gives them direct access to the `ExtensionManager`, `SessionManager`, and current `Session`.

Nine platform extensions are registered:

| Name | Display Name | Default Enabled | Unprefixed Tools |
|------|-------------|-----------------|------------------|
| `analyze` | Analyze | Yes | Yes |
| `todo` | Todo | Yes | No |
| `apps` | Apps | Yes | No |
| `chatrecall` | Chat Recall | No | No |
| `extensionmanager` | Extension Manager | Yes | No |
| `summon` | Summon | Yes | Yes |
| `code_execution` | Code Mode | No (feature-gated) | Yes |
| `developer` | Developer | Yes | Yes |
| `tom` | Top Of Mind | Yes | No |

**Unprefixed tools** means the tool names are exposed directly (e.g., `shell`, `text_editor`) rather than with an `extension__tool` prefix. This makes them feel like first-class capabilities.

### 1.3 STDIO Extensions

**[Confirmed]** ([source: extension_manager.rs](https://github.com/block/goose/blob/main/crates/goose/src/agents/extension_manager.rs))

STDIO extensions launch an external process and communicate over stdin/stdout using the MCP protocol. This is the standard way to integrate third-party MCP servers.

Key fields:
- `cmd` -- the command to execute (resolved via `SearchPaths` which includes npm paths)
- `args` -- command-line arguments
- `envs` -- environment variables to pass (validated against disallowed list)
- `env_keys` -- secret keys resolved from Goose's config/keychain at runtime
- `timeout` -- optional, defaults to 300 seconds

Before launching, Goose runs a **malware check** via OSV for `npx` and `uvx` commands.

Configuration example:
```yaml
extensions:
  my-mcp-server:
    enabled: true
    type: stdio
    name: my-mcp-server
    cmd: npx
    args: ["-y", "@example/mcp-server"]
    envs:
      API_KEY: "sk-..."
    timeout: 300
```

When to use: Third-party MCP servers, local tools, custom scripts.

### 1.4 SSE Extensions (Deprecated)

**[Confirmed]** ([source: extensions.rs config](https://github.com/block/goose/blob/main/crates/goose/src/config/extensions.rs))

SSE (Server-Sent Events) transport is **no longer supported**. The config variant is kept only for backward compatibility. Attempting to add an SSE extension returns an error: `"SSE is unsupported, migrate to streamable_http"`. The `get_warnings()` function in the config module explicitly warns about SSE entries.

### 1.5 Streamable HTTP Extensions

**[Confirmed]** ([source: extension_manager.rs](https://github.com/block/goose/blob/main/crates/goose/src/agents/extension_manager.rs))

Streamable HTTP extensions connect to remote MCP servers over HTTP using the MCP Streamable HTTP specification. They support:

- Custom HTTP headers (with environment variable substitution via `${VAR}` syntax)
- OAuth authentication (automatic flow when `AuthRequiredError` is received)
- Environment variable injection via `envs` and `env_keys`

Configuration example:
```yaml
extensions:
  remote-tools:
    enabled: true
    type: streamable_http
    name: remote-tools
    uri: https://mcp.example.com/v1
    headers:
      Authorization: "Bearer ${API_TOKEN}"
    timeout: 300
```

When to use: Remote/cloud-hosted MCP servers, services requiring OAuth.

### 1.6 Frontend Extensions

**[Confirmed]** ([source: extension.rs](https://github.com/block/goose/blob/main/crates/goose/src/agents/extension.rs))

Frontend extensions provide tools from the UI layer (desktop app). They carry a `tools: Vec<Tool>` field with pre-defined tool definitions and optional `instructions`. They **cannot** be added as server extensions -- attempting to do so returns an error.

When to use: UI-provided capabilities (desktop app only).

### 1.7 InlinePython Extensions

**[Confirmed]** ([source: extension.rs](https://github.com/block/goose/blob/main/crates/goose/src/agents/extension.rs))

InlinePython extensions execute Python code dynamically using `uvx`. The Python source code, MCP implementation, and dependencies are provided inline in the configuration. At runtime, Goose writes the code to a temporary file and executes it via `uvx --with mcp [--with dep1 ...] python <file>`.

Configuration fields:
- `code` -- the Python source code
- `dependencies` -- optional list of pip packages
- `timeout` -- execution timeout

When to use: Quick prototyping, lightweight custom tools without a separate server process.

---

## 2. Extension Lifecycle

### 2.1 Discovery and Configuration

**[Confirmed]** ([source: config/extensions.rs](https://github.com/block/goose/blob/main/crates/goose/src/config/extensions.rs))

Extensions are stored in `~/.config/goose/config.yaml` under the `extensions` key as an `IndexMap<String, ExtensionEntry>`. Each entry has:
- `enabled: bool` -- whether the extension is active
- `config: ExtensionConfig` -- the flattened configuration (type, name, cmd, etc.)

Discovery methods:
1. **Config file** -- read from `config.yaml` at startup
2. **CLI flags** -- `--with-builtin <name>`, `--with-extension <cmd>`, `--with-streamable-http-extension <url>`
3. **`goose configure`** -- interactive CLI configuration
4. **Extension Manager tool** -- the `extensionmanager` platform extension lets the agent itself discover and enable extensions at runtime
5. **Recipes** -- recipe files can specify extension lists

### 2.2 Loading / Initialization

**[Confirmed]** ([source: extension_manager.rs](https://github.com/block/goose/blob/main/crates/goose/src/agents/extension_manager.rs))

The `ExtensionManager::add_extension()` method handles initialization for each type:

1. **Config resolution** -- `ExtensionConfig::resolve()` merges environment variables from `envs` (direct), `env_keys` (keychain/config secrets), and substitutes `${VAR}` patterns in headers
2. **Platform extensions** -- instantiated via `client_factory` function with `PlatformExtensionContext`
3. **Builtin extensions** -- spawned via duplex streams using `BUILTIN_EXTENSIONS` map or Docker `goose mcp <name>` for containerized execution
4. **STDIO extensions** -- malware check runs first, then `Command` is built with `configure_subprocess`, `SearchPaths` resolution, and environment injection; child process is spawned via `TokioChildProcess`
5. **Streamable HTTP** -- HTTP client is constructed with custom headers and User-Agent (`goose/<version>`); OAuth flow triggers automatically on `AuthRequiredError`
6. **InlinePython** -- code written to temp file, executed via `uvx`
7. **MCP handshake** -- `McpClient::connect()` performs the MCP initialize handshake with the configured timeout

If an extension with the same name but different config is added, the old one is replaced (restarted). Same config is idempotent (no-op).

### 2.3 Enabling/Disabling at Runtime

**[Confirmed]** ([source: config/extensions.rs](https://github.com/block/goose/blob/main/crates/goose/src/config/extensions.rs))

Functions available for runtime management:
- `set_extension_enabled(key, bool)` -- toggles the `enabled` flag and persists to config
- `set_extension(entry)` -- adds or updates an extension entry
- `remove_extension(key)` -- removes an extension from config
- `ExtensionManager::remove_extension(name)` -- removes from the running session

The tools cache is invalidated and version-bumped whenever extensions change, forcing re-fetching of tool lists.

### 2.4 Session Resolution

**[Confirmed]** ([source: config/extensions.rs](https://github.com/block/goose/blob/main/crates/goose/src/config/extensions.rs))

`resolve_extensions_for_new_session()` determines which extensions to load:
1. If recipe extensions are provided, use those
2. If override extensions are provided (CLI flags), use those
3. Otherwise, use all enabled extensions from config

Platform extensions are filtered by `is_extension_available()` which checks the `PLATFORM_EXTENSIONS` map.

---

## 3. Tool Registration and Invocation

### 3.1 Tool Registration / Listing

**[Confirmed]** ([source: extension_manager.rs](https://github.com/block/goose/blob/main/crates/goose/src/agents/extension_manager.rs))

Tools are fetched lazily from all extensions via `get_all_tools_cached()`:

1. For each extension, `client.list_tools()` is called (with pagination support via `next_cursor`)
2. Tools are **prefixed** with the extension name: `extensionname__toolname` (double underscore separator)
   - Exception: extensions with `unprefixed_tools: true` (developer, analyze, summon, code_execution) expose tools without prefix
3. Each tool gets metadata injected: `goose_extension` meta key stores the owning extension name
4. Tool **availability filtering**: `ExtensionConfig::is_tool_available()` checks the `available_tools` whitelist. If the list is empty, all tools are available; otherwise only listed tools are exposed
5. **Deduplication**: duplicate tool names across extensions are logged as warnings and skipped
6. Tools are cached with an `AtomicU64` version counter; cache is invalidated when extensions change

### 3.2 Tool Invocation Flow

**[Confirmed]** ([source: extension_manager.rs](https://github.com/block/goose/blob/main/crates/goose/src/agents/extension_manager.rs))

The complete flow from LLM response to tool execution:

1. **LLM generates tool call** -- the LLM responds with a `ToolRequest` containing the tool name and arguments
2. **Inspection pipeline** -- `ToolInspectionManager` runs all registered inspectors sequentially:
   - **Permission inspector** -- checks permission mode (Auto/Approve/SmartApprove) and per-tool permissions
   - **Security inspector** -- pattern matching for prompt injection and dangerous commands
3. **Permission resolution** -- inspection results are merged; security findings can override permission decisions (e.g., escalating an approved tool to require approval)
4. **User approval** (if needed) -- for tools requiring approval, the UI presents Allow/Deny options
5. **Tool resolution** -- `resolve_tool()` maps the prefixed tool name to the owning extension:
   - Tries splitting on `__` to find `(extension, actual_tool_name)`
   - Falls back to scanning all cached tools by `goose_extension` metadata
6. **Dispatch** -- `dispatch_tool_call()` calls `client.call_tool()` on the resolved extension's MCP client with the actual (unprefixed) tool name
7. **Result return** -- `ToolCallResult` contains a boxed future for the result and an optional notification stream for progress/logging updates
8. **Agent loop continues** -- results are sent back to the LLM as tool response messages

### 3.3 Resource Access

**[Confirmed]** ([source: extension_manager.rs](https://github.com/block/goose/blob/main/crates/goose/src/agents/extension_manager.rs))

Resources are accessed via `read_resource_tool()`:
- If an `extension_name` parameter is provided, reads directly from that extension
- Otherwise, searches across all resource-capable extensions (first match wins)
- UI resources (URIs starting with `ui://`) are listed separately via `get_ui_resources()`

### 3.4 Prompt Access

**[Confirmed]** ([source: extension_manager.rs](https://github.com/block/goose/blob/main/crates/goose/src/agents/extension_manager.rs))

Extensions can expose prompts via `list_prompts()` and `get_prompt()`, following the MCP prompts specification.

---

## 4. Permission Model

### 4.1 Permission Modes (GooseMode)

**[Confirmed]** ([source: permission_inspector.rs](https://github.com/block/goose/blob/main/crates/goose/src/permission/permission_inspector.rs), [docs](https://block.github.io/goose/docs/guides/goose-permissions/))

Goose has four operational modes:

| Mode | Behavior |
|------|----------|
| **Auto** | All tools are automatically approved. No user intervention. |
| **Approve** | All state-changing tools require explicit user approval. User-defined per-tool overrides respected. |
| **SmartApprove** | Like Approve, but uses tool annotations and LLM-based detection to auto-approve read-only tools. |
| **Chat** | No tool execution at all (conversation only). |

### 4.2 Per-Tool Permission Levels

**[Confirmed]** ([source: config/permission.rs](https://github.com/block/goose/blob/main/crates/goose/src/config/permission.rs))

Three permission levels per tool:

```rust
pub enum PermissionLevel {
    AlwaysAllow,  // Tool can always be used without prompt
    AskBefore,    // Tool requires permission before use
    NeverAllow,   // Tool is never allowed to be used
}
```

Permissions are stored in `~/.config/goose/permission.yaml` and managed by `PermissionManager`. Two permission categories exist:
- **user** -- explicitly set by the user
- **smart_approve** -- cached LLM read-only detection results and tool annotation-based decisions

### 4.3 Permission Check Flow (Inspection Pipeline)

**[Confirmed]** ([source: permission_inspector.rs](https://github.com/block/goose/blob/main/crates/goose/src/permission/permission_inspector.rs))

For each tool request in SmartApprove/Approve mode:

1. Check **user-defined permission** first (highest priority)
2. Check if tool has **read-only annotation** (`ToolAnnotations.read_only_hint == true`) -- auto-allow
3. Check **cached smart_approve permission** (from prior LLM detection)
4. Special case: `extensionmanager__manage_extensions` always requires approval
5. If SmartApprove mode and no cached decision: defer to **LLM-based read-only detection**
6. Default: require approval

The LLM detection (`permission_judge.rs`) sends tool names to the provider with a specialized prompt that asks it to classify each tool as read-only or write. Results are cached in `permission.yaml` for future calls.

### 4.4 User Approval Actions

**[Confirmed]** ([source: permission_confirmation.rs](https://github.com/block/goose/blob/main/crates/goose/src/permission/permission_confirmation.rs))

Five possible user responses:
```rust
pub enum Permission {
    AlwaysAllow,  // Remember this as always allowed
    AllowOnce,    // Allow just this time
    Cancel,       // Cancel the operation
    DenyOnce,     // Deny just this time
    AlwaysDeny,   // Remember this as never allowed
}
```

### 4.5 Tool Annotation Integration

**[Confirmed]** ([source: config/permission.rs](https://github.com/block/goose/blob/main/crates/goose/src/config/permission.rs))

When tools are loaded, `apply_tool_annotations()` processes MCP tool annotations:
- Tools with `read_only_hint: false` are automatically added to the `smart_approve` category as `AskBefore`
- Tools with `read_only_hint: true` are tracked in a per-session `readonly_tools` set for instant approval

---

## 5. Security Systems

### 5.1 Security Inspector (Prompt Injection Detection)

**[Confirmed]** ([source: security/security_inspector.rs](https://github.com/block/goose/blob/main/crates/goose/src/security/security_inspector.rs), [security/mod.rs](https://github.com/block/goose/blob/main/crates/goose/src/security/mod.rs))

The `SecurityInspector` implements the `ToolInspector` trait and uses pattern-based detection plus optional ML classification:

- **Enabled via config**: `SECURITY_PROMPT_ENABLED` must be `true`
- **Two detection modes**:
  - **Pattern-based** (`patterns.rs`): regex patterns for dangerous shell commands
  - **ML-based** (`classification_client.rs`): external classifier API, enabled via `SECURITY_COMMAND_CLASSIFIER_ENABLED` and `SECURITY_PROMPT_CLASSIFIER_ENABLED`
- **Threat categories**: FileSystemDestruction, RemoteCodeExecution, DataExfiltration, SystemModification, NetworkAccess, ProcessManipulation, PrivilegeEscalation, CommandInjection
- **Risk levels** with confidence scores: Critical (0.95), High (0.75), Medium (0.60), Low (0.45)
- **Action**: malicious findings with `should_ask_user: true` trigger `RequireApproval` with a security alert message containing the finding ID

Example patterns detected:
- `rm -rf /` -- filesystem destruction (High)
- `curl ... | bash` -- remote code execution (Critical)
- SSH key exfiltration, password file access, crontab modification, etc.

### 5.2 Inspection Results Merging

**[Confirmed]** ([source: tool_inspection.rs](https://github.com/block/goose/blob/main/crates/goose/src/tool_inspection.rs))

The `ToolInspectionManager` runs inspectors sequentially and merges results:
1. Permission inspector results form the baseline
2. Security inspector results act as overrides -- can escalate approved tools to require approval
3. `apply_inspection_results_to_permissions()` handles the merging logic

---

## 6. Built-in Extensions Detail

### 6.1 Platform Extensions (In-Process)

**[Confirmed]** ([source: platform_extensions/mod.rs](https://github.com/block/goose/blob/main/crates/goose/src/agents/platform_extensions/mod.rs))

| Extension | Key Tools | Purpose |
|-----------|-----------|---------|
| **Developer** | `shell`, `text_editor`, `list_directory` | File editing, shell commands, directory listing. Core developer workflow. |
| **Analyze** | Code structure analysis | Tree-sitter based code analysis: directory overviews, file details, symbol call graphs |
| **Todo** | Task tracking | Internal todo list for tracking agent progress |
| **Apps** | App management | Create/manage sandboxed HTML/CSS/JS apps in windows |
| **Chat Recall** | Session search | Search past conversations and load session summaries |
| **Extension Manager** | `manage_extensions`, `search_available_extensions` | Discover, enable, disable extensions at runtime |
| **Summon** | Knowledge loading, subagents | Load knowledge and delegate tasks to subagents |
| **Top Of Mind (TOM)** | Context injection | Injects custom context via `GOOSE_MOIM_MESSAGE_TEXT` / `GOOSE_MOIM_MESSAGE_FILE` env vars |
| **Code Execution** | Code mode (feature-gated) | Token-saving code execution mode |

### 6.2 Builtin MCP Server Extensions

**[Confirmed]** ([source: goose-mcp/src/lib.rs](https://github.com/block/goose/blob/main/crates/goose-mcp/src/lib.rs))

| Extension | Server Type | Purpose |
|-----------|------------|---------|
| **autovisualiser** | `AutoVisualiserRouter` | Chart/graph rendering (Chart.js, D3, Mermaid, Leaflet) |
| **computercontroller** | `ComputerControllerServer` | Browser automation, PDF/DOCX/XLSX processing, platform-specific (Linux/macOS/Windows) |
| **memory** | `MemoryServer` | Persistent memory and recall |
| **tutorial** | `TutorialServer` | Interactive guided tutorials |

---

## 7. Extension Configuration

### 7.1 Config File Location

**[Confirmed]** ([docs](https://block.github.io/goose/docs/guides/config-files/))

- **macOS/Linux**: `~/.config/goose/config.yaml`
- **Windows**: `%APPDATA%\Block\goose\config\config.yaml`

### 7.2 Config Structure

**[Confirmed]** ([source: config/extensions.rs](https://github.com/block/goose/blob/main/crates/goose/src/config/extensions.rs))

```yaml
extensions:
  developer:
    enabled: true
    type: builtin
    name: developer
    timeout: 300
  my-stdio-server:
    enabled: true
    type: stdio
    name: my-stdio-server
    cmd: npx
    args: ["-y", "@example/mcp-server"]
    envs:
      API_KEY: "value"
    env_keys: ["SECRET_FROM_KEYCHAIN"]
    timeout: 300
    available_tools: ["tool1", "tool2"]  # optional whitelist
  remote-server:
    enabled: true
    type: streamable_http
    name: remote-server
    uri: https://example.com/mcp
    headers:
      Authorization: "Bearer ${TOKEN}"
    timeout: 300
```

### 7.3 CLI Commands

**[Confirmed]** ([docs](https://block.github.io/goose/docs/guides/goose-cli-commands/))

- `goose configure` -- interactive configuration flow
- `goose info -v` -- validate and display configuration
- `--with-builtin <name>` -- add a builtin extension for this session
- `--with-extension <cmd>` -- add a STDIO extension
- `--with-streamable-http-extension <url>` -- add a Streamable HTTP extension

### 7.4 Environment Variable Resolution

**[Confirmed]** ([source: extension_manager.rs](https://github.com/block/goose/blob/main/crates/goose/src/agents/extension_manager.rs))

Two mechanisms for injecting secrets:
1. **`envs`** -- direct key-value pairs in config (validated against disallowed list)
2. **`env_keys`** -- keys resolved from Goose's config store (keychain/secrets) at runtime via `config.get(key, true)`

Environment variable substitution in headers supports both `${VAR}` and `$VAR` syntax.

---

## 8. Extension Validation and Malware Checking

### 8.1 OSV-Based Malware Detection

**[Confirmed]** ([source: extension_malware_check.rs](https://github.com/block/goose/blob/main/crates/goose/src/agents/extension_malware_check.rs))

Before launching STDIO extensions, Goose checks packages against the [OSV (Open Source Vulnerabilities) database](https://osv.dev/) for `MAL-*` advisories (known malicious packages).

**How it works:**
1. **Ecosystem inference** from command:
   - `uvx` commands -> PyPI ecosystem
   - `npx` commands -> npm ecosystem
   - Unknown commands -> skip check (fail open)
2. **Package parsing**:
   - npm: handles `react@18.3.1`, `@scope/pkg@1.2.3`, `eslint` (no version)
   - PyPI: handles `package==1.2.3`, `package[extra]==1.2.3`
3. **OSV API query**: POST to `https://api.osv.dev/v1/query` with package name, ecosystem, and optional version
4. **Pagination**: follows `next_page_token` to check all results
5. **Decision**: if any `MAL-*` advisory ID is found, the extension is **blocked** with a descriptive error
6. **Fail-open policy**: HTTP errors, JSON parse errors, and network failures all result in allowing the extension (logged as errors)

**Configuration**: The OSV endpoint can be overridden via the `OSV_ENDPOINT` environment variable.

**User-Agent**: `goose-osv-check/1.1 (+https://osv.dev)` with a 10-second timeout.

---

## 9. Environment Variable Security

### 9.1 The 31 Disallowed Environment Variables

**[Confirmed]** ([source: extension.rs `Envs::DISALLOWED_KEYS`](https://github.com/block/goose/blob/main/crates/goose/src/agents/extension.rs))

Extensions can set environment variables for child processes, but 31 sensitive variables are blocked to prevent security attacks. The check is case-insensitive.

| Category | Variables | Rationale |
|----------|-----------|-----------|
| **Binary path manipulation** | `PATH`, `PATHEXT`, `SystemRoot`, `windir` | Prevents command hijacking and DLL resolution attacks |
| **Linux dynamic linker hijacking** | `LD_LIBRARY_PATH`, `LD_PRELOAD`, `LD_AUDIT`, `LD_DEBUG`, `LD_BIND_NOW`, `LD_ASSUME_KERNEL` | Prevents shared library injection and preloading attacks |
| **macOS dynamic linker** | `DYLD_LIBRARY_PATH`, `DYLD_INSERT_LIBRARIES`, `DYLD_FRAMEWORK_PATH` | macOS equivalents of LD attacks |
| **Language runtime hijacking** | `PYTHONPATH`, `PYTHONHOME`, `NODE_OPTIONS`, `RUBYOPT`, `GEM_PATH`, `GEM_HOME`, `CLASSPATH`, `GO111MODULE`, `GOROOT` | Prevents module resolution hijacking across Python, Node.js, Ruby, Java, Go |
| **Windows process/DLL hijacking** | `APPINIT_DLLS`, `SESSIONNAME`, `ComSpec`, `TEMP`, `TMP`, `LOCALAPPDATA`, `USERPROFILE`, `HOMEDRIVE`, `HOMEPATH` | Prevents DLL injection, temp file redirection, and profile-based attacks |

**Behavior**: When a disallowed variable is encountered in `Envs::new()`, it is silently skipped with a warning log. The `validate()` method returns an error if disallowed vars are present (used for stricter validation paths).

---

## 10. MCP Protocol Compliance

### 10.1 Protocol Version

**[Confirmed]** ([source: mcp_client.rs](https://github.com/block/goose/blob/main/crates/goose/src/agents/mcp_client.rs))

Goose uses **MCP Protocol Version `2025-03-26`** (the latest stable spec version), configured via:
```rust
.with_protocol_version(ProtocolVersion::V_2025_03_26)
```

### 10.2 Rust SDK (rmcp)

**[Confirmed]** ([source: Cargo.toml](https://github.com/block/goose/blob/main/Cargo.toml))

Goose uses the **rmcp crate version 1.1.0** (the official Rust SDK for MCP) with features `schemars` and `auth`.

### 10.3 Supported MCP Features

**[Confirmed]** ([source: mcp_client.rs](https://github.com/block/goose/blob/main/crates/goose/src/agents/mcp_client.rs))

| MCP Feature | Supported | Details |
|-------------|-----------|---------|
| **Tools** | Yes | Full support: list, call, pagination, tool annotations |
| **Resources** | Yes | list_resources, read_resource, UI resources (`ui://` scheme) |
| **Prompts** | Yes | list_prompts, get_prompt |
| **Sampling** | Yes | `create_message` handler: servers can request LLM completions from Goose |
| **Elicitation** | Yes | `create_elicitation` handler: form-based and URL-based elicitation with 5-minute timeout |
| **Logging** | Yes | Receives `LoggingMessageNotification` from servers |
| **Progress** | Yes | Receives `ProgressNotification` from servers |
| **Cancellation** | Yes | `CancellationToken` support throughout; `CancelledNotificationParam` handling |
| **Tool Annotations** | Yes | `read_only_hint`, `destructive_hint` used for smart permission decisions |
| **Extensions (MCP)** | Yes | `ExtensionCapabilities` with `mcpui` capability for desktop apps |
| **OAuth Auth** | Yes | Automatic OAuth flow for Streamable HTTP via `AuthClient` |

**Client capabilities advertised:**
```rust
ClientCapabilities::builder()
    .enable_extensions_with(extensions)
    .enable_sampling()
    .enable_elicitation()
    .build()
```

### 10.4 Sampling Implementation

**[Confirmed]** ([source: mcp_client.rs](https://github.com/block/goose/blob/main/crates/goose/src/agents/mcp_client.rs))

When an MCP server requests a `create_message` (sampling), Goose:
1. Resolves the session ID from MCP extensions metadata or the stored fallback
2. Converts `SamplingMessage` objects to Goose's internal `Message` format
3. Calls the configured LLM provider's `complete()` method
4. Returns the result as `CreateMessageResult`

### 10.5 Elicitation Implementation

**[Confirmed]** ([source: mcp_client.rs](https://github.com/block/goose/blob/main/crates/goose/src/agents/mcp_client.rs))

Two elicitation types supported:
- **Form elicitation** -- structured schema-based input from the user
- **URL elicitation** -- URL-based input

Both use `ActionRequiredManager::global().request_and_wait()` with a 300-second timeout.

---

## 11. Architecture Summary

```
User Input
    |
    v
[Interface (CLI / Desktop)]
    |
    v
[Agent Loop] <-- system prompt with extension instructions
    |
    v
[LLM Provider] -- generates tool calls
    |
    v
[ToolInspectionManager]
    |-- PermissionInspector (mode + per-tool rules + LLM detection)
    |-- SecurityInspector (pattern matching + ML classification)
    |
    v
[User Approval] (if needed)
    |
    v
[ExtensionManager.dispatch_tool_call()]
    |-- resolve_tool() -> find owning extension
    |-- client.call_tool() -> MCP request
    |
    v
[MCP Extension]
    |-- Platform (in-process, direct trait impl)
    |-- Builtin (in-process, duplex streams)
    |-- STDIO (child process, stdin/stdout)
    |-- Streamable HTTP (remote, HTTP + OAuth)
    |-- InlinePython (temp file + uvx)
    |
    v
[Tool Result] -> back to Agent Loop -> back to LLM
```

---

## Sources

### Primary Sources (GitHub)
- [Goose Repository](https://github.com/block/goose)
- [extension.rs (ExtensionConfig enum, Envs disallowed keys)](https://github.com/block/goose/blob/main/crates/goose/src/agents/extension.rs)
- [config/extensions.rs (extension config management)](https://github.com/block/goose/blob/main/crates/goose/src/config/extensions.rs)
- [extension_manager.rs (extension lifecycle, tool registration, dispatch)](https://github.com/block/goose/blob/main/crates/goose/src/agents/extension_manager.rs)
- [extension_malware_check.rs (OSV malware scanning)](https://github.com/block/goose/blob/main/crates/goose/src/agents/extension_malware_check.rs)
- [platform_extensions/mod.rs (PLATFORM_EXTENSIONS map)](https://github.com/block/goose/blob/main/crates/goose/src/agents/platform_extensions/mod.rs)
- [goose-mcp/src/lib.rs (BUILTIN_EXTENSIONS)](https://github.com/block/goose/blob/main/crates/goose-mcp/src/lib.rs)
- [config/permission.rs (PermissionManager, PermissionLevel)](https://github.com/block/goose/blob/main/crates/goose/src/config/permission.rs)
- [permission/permission_inspector.rs (permission check flow)](https://github.com/block/goose/blob/main/crates/goose/src/permission/permission_inspector.rs)
- [permission/permission_confirmation.rs (Permission enum)](https://github.com/block/goose/blob/main/crates/goose/src/permission/permission_confirmation.rs)
- [permission/permission_judge.rs (LLM read-only detection)](https://github.com/block/goose/blob/main/crates/goose/src/permission/permission_judge.rs)
- [tool_inspection.rs (ToolInspectionManager)](https://github.com/block/goose/blob/main/crates/goose/src/tool_inspection.rs)
- [security/security_inspector.rs (SecurityInspector)](https://github.com/block/goose/blob/main/crates/goose/src/security/security_inspector.rs)
- [security/patterns.rs (ThreatPattern definitions)](https://github.com/block/goose/blob/main/crates/goose/src/security/patterns.rs)
- [security/mod.rs (SecurityManager)](https://github.com/block/goose/blob/main/crates/goose/src/security/mod.rs)
- [mcp_client.rs (McpClientTrait, MCP protocol version, sampling, elicitation)](https://github.com/block/goose/blob/main/crates/goose/src/agents/mcp_client.rs)

### Documentation
- [Goose Architecture](https://block.github.io/goose/docs/goose-architecture/)
- [Goose Permission Modes](https://block.github.io/goose/docs/guides/goose-permissions/)
- [Tool Permissions](https://block.github.io/goose/docs/guides/tool-permissions/)
- [Configuration Files](https://block.github.io/goose/docs/guides/config-files/)
- [CLI Commands](https://block.github.io/goose/docs/guides/goose-cli-commands/)
- [Building Custom Extensions](https://block.github.io/goose/docs/tutorials/custom-extensions/)

### Third-Party Analysis
- [DeepWiki: Extension Types and Configuration](https://deepwiki.com/block/goose/5.3-extension-types-and-configuration)
- [DeepWiki: Extension Management](https://deepwiki.com/block/goose/5.4-extension-management)
- [DEV Community: Deep Dive into Goose Extension System](https://dev.to/lymah/deep-dive-into-gooses-extension-system-and-model-context-protocol-mcp-3ehl)

### MCP Protocol
- [rmcp crate (Official Rust MCP SDK)](https://docs.rs/rmcp/latest/rmcp/)
- [MCP Rust SDK GitHub](https://github.com/modelcontextprotocol/rust-sdk)
- [rmcp adoption issue #3578](https://github.com/block/goose/issues/3578)
