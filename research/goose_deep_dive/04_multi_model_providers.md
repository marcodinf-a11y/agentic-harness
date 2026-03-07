# Goose Multi-Model & Provider System

**Research Date:** 2026-03-07
**Subject:** Block's open-source Goose agent framework — provider abstraction, multi-model routing, and configuration
**Primary Source:** https://github.com/block/goose (Rust codebase, `crates/goose/src/providers/`)

---

## 1. Provider Abstraction

### The `Provider` Trait

Goose defines a core `Provider` trait in `crates/goose/src/providers/base.rs` that every LLM backend must implement. The trait is `async_trait`-based and requires `Send + Sync`.

**Required methods:**

| Method | Signature (simplified) | Purpose |
|--------|----------------------|---------|
| `get_name()` | `&self -> &str` | Returns the provider's identifier string |
| `stream()` | `(&self, model_config, session_id, system, messages, tools) -> Result<MessageStream>` | **Primary method** — streams a completion given a system prompt, conversation messages, and available MCP tools |
| `get_model_config()` | `&self -> ModelConfig` | Returns the current model configuration |

**Default-implemented methods (on the trait):**

| Method | Purpose |
|--------|---------|
| `complete()` | Calls `stream()` and collects the full stream into a single `(Message, ProviderUsage)` |
| `complete_fast()` | Tries the model's "fast" variant first, falls back to the regular model on failure |
| `generate_title()` | Generates a short session title from conversation context |
| `fetch_supported_models()` | Returns an empty vec by default; providers can override to list models from the API |
| `configure_oauth()` | No-op by default; providers with OAuth (e.g., GitHub Copilot) override this |

**[Confirmed]** — Source: [`base.rs` on GitHub](https://github.com/block/goose/blob/main/crates/goose/src/providers/base.rs)

### The `ProviderDef` Trait

Separate from `Provider`, the `ProviderDef` trait handles provider construction and metadata:

```rust
pub trait ProviderDef: Send + Sync {
    type Provider: Provider + 'static;
    fn metadata() -> ProviderMetadata where Self: Sized;
    fn from_env(model: ModelConfig, extensions: Vec<ExtensionConfig>) -> BoxFuture<'static, Result<Self::Provider>>;
}
```

This separation means metadata (name, default model, config keys) is decoupled from provider instances. The registry calls `ProviderDef::metadata()` without needing to instantiate anything.

**[Confirmed]** — Source: [`base.rs`](https://github.com/block/goose/blob/main/crates/goose/src/providers/base.rs)

### ProviderMetadata

Each provider publishes rich metadata:

- `name` — unique identifier (e.g., `"anthropic"`, `"openai"`, `"ollama"`)
- `display_name` — human-readable name for UIs
- `description` — capability description
- `default_model` — the recommended model
- `known_models` — `Vec<ModelInfo>` with name, context limit, optional cost data, and cache control support
- `model_doc_link` — URL to the provider's model documentation
- `config_keys` — `Vec<ConfigKey>` describing required secrets/params (API keys, hosts, etc.)

**[Confirmed]** — Source: [`base.rs`](https://github.com/block/goose/blob/main/crates/goose/src/providers/base.rs)

### Tool Call Mapping

Tool calls are represented internally using `rmcp::model::Tool` (from the MCP SDK). Each provider's format module converts between the internal representation and the provider's native API format:

- **OpenAI format** (`formats/openai.rs`): Converts tools to OpenAI's `functions` array with JSON Schema. Handles streaming `tool_calls` deltas with index-based accumulation.
- **Anthropic format** (`formats/anthropic.rs`): Converts to Anthropic's `tool_use` / `tool_result` content blocks. Handles extended thinking (`thinking` / `redacted_thinking` blocks) and cache control.
- **Google format** (`formats/google.rs`): Converts to Gemini's `functionDeclarations` format.
- **Ollama format** (`formats/ollama.rs`): Reuses OpenAI format but adds XML tool call fallback parsing for models like `qwen3-coder` that emit `<function=name><parameter=key>value</parameter></function>` instead of native JSON tool calls.

**[Confirmed]** — Source: [`formats/` directory](https://github.com/block/goose/tree/main/crates/goose/src/providers/formats)

---

## 2. Supported Providers

### Built-in (Native) Providers

These are registered at startup in `init.rs` via `registry.register::<ProviderType>(preferred)`:

| Provider | Rust Type | Preferred | Default Model | Auth Method |
|----------|-----------|-----------|---------------|-------------|
| **Anthropic** | `AnthropicProvider` | Yes | `claude-sonnet-4-5` | `ANTHROPIC_API_KEY` (x-api-key header) |
| **OpenAI** | `OpenAiProvider` | Yes | `gpt-4o` | `OPENAI_API_KEY` (Bearer token) |
| **Google** | `GoogleProvider` | Yes | `gemini-2.5-pro` | `GOOGLE_API_KEY` |
| **Ollama** | `OllamaProvider` | Yes | `qwen3` | No auth (local) |
| **OpenRouter** | `OpenRouterProvider` | Yes | — | `OPENROUTER_API_KEY` |
| **Azure OpenAI** | `AzureProvider` | No | — | Azure AD / API key |
| **AWS Bedrock** | `BedrockProvider` | No | — | AWS credentials |
| **GCP Vertex AI** | `GcpVertexAIProvider` | No | — | GCP service account |
| **Databricks** | `DatabricksProvider` | Yes | — | Databricks token |
| **Snowflake** | `SnowflakeProvider` | No | — | Snowflake credentials |
| **xAI (Grok)** | `XaiProvider` | No | — | `XAI_API_KEY` |
| **Venice** | `VeniceProvider` | No | — | Venice API key |
| **LiteLLM** | `LiteLLMProvider` | No | — | LiteLLM proxy |
| **Tetrate** | `TetrateProvider` | Yes | — | Tetrate Agent Router |
| **SageMaker TGI** | `SageMakerTgiProvider` | No | — | AWS credentials |
| **GitHub Copilot** | `GithubCopilotProvider` | No | — | OAuth device flow |
| **Local Inference** | `LocalInferenceProvider` | No | `bartowski/Llama-3.2-1B-Instruct-GGUF:Q4_K_M` | No auth |
| **ChatGPT Codex** | `ChatGptCodexProvider` | Yes | — | OpenAI key |
| **Claude Code** | `ClaudeCodeProvider` | Yes | — | Anthropic key |
| **Codex** | `CodexProvider` | Yes | — | OpenAI key |
| **Cursor Agent** | `CursorAgentProvider` | No | — | Cursor credentials |
| **Gemini CLI** | `GeminiCliProvider` | No | — | Google key |

The "Preferred" flag controls whether the provider appears prominently in onboarding/settings UIs.

**[Confirmed]** — Source: [`init.rs`](https://github.com/block/goose/blob/main/crates/goose/src/providers/init.rs)

### Declarative (JSON-Configured) Providers

These are loaded from JSON files in `crates/goose/src/providers/declarative/`:

| Provider | Engine | File |
|----------|--------|------|
| **Groq** | OpenAI-compatible | `groq.json` |
| **DeepSeek** | OpenAI-compatible | `deepseek.json` |
| **Cerebras** | OpenAI-compatible | `cerebras.json` |
| **Inception** | OpenAI-compatible | `inception.json` |
| **Kimi** | OpenAI-compatible | `kimi.json` |
| **LM Studio** | OpenAI-compatible | `lmstudio.json` |
| **Mistral** | OpenAI-compatible | `mistral.json` |
| **Moonshot** | OpenAI-compatible | `moonshot.json` |
| **OVHcloud** | OpenAI-compatible | `ovhcloud.json` |

**[Confirmed]** — Source: [`declarative/` directory listing](https://github.com/block/goose/tree/main/crates/goose/src/providers/declarative)

### Provider Catalog System

Beyond the built-in and declarative providers, Goose includes a **provider catalog** (`catalog.rs`) that reads from `canonical/data/provider_metadata.json`. This catalog lists 100+ providers from the broader ecosystem (any OpenAI/Anthropic/Ollama-compatible endpoint). The catalog system:

1. Reads a bundled JSON metadata file with provider IDs, display names, npm package references, API URLs, and environment variable names
2. Detects the API format (OpenAI, Anthropic, or Ollama) from the npm package name
3. Provides `ProviderCatalogEntry` and `ProviderTemplate` data for the UI to present when adding custom providers
4. Cross-references with the `CanonicalModelRegistry` to populate model templates with context limits and capabilities (tool call, reasoning, attachment, temperature support)

**[Confirmed]** — Source: [`catalog.rs`](https://github.com/block/goose/blob/main/crates/goose/src/providers/catalog.rs)

---

## 3. Lead-Worker Pattern

### Overview

The `LeadWorkerProvider` (`lead_worker.rs`) implements a dual-model strategy where a powerful "lead" model handles the first few turns of a conversation, then a cheaper "worker" model takes over. If the worker struggles, the system falls back to the lead.

### Architecture

```
LeadWorkerProvider
  |-- lead_provider: Arc<dyn Provider>     // Frontier model (e.g., Claude Opus)
  |-- worker_provider: Arc<dyn Provider>   // Cheaper model (e.g., Claude Haiku)
  |-- lead_turns: usize                    // Initial turns with lead (default: 3)
  |-- turn_count: Arc<Mutex<usize>>        // Current turn counter
  |-- failure_count: Arc<Mutex<usize>>     // Consecutive failure counter
  |-- max_failures_before_fallback: usize  // Threshold (default: 2)
  |-- fallback_turns: usize               // Turns to use lead in fallback (default: 2)
  |-- in_fallback_mode: Arc<Mutex<bool>>   // Whether currently in fallback
  |-- fallback_remaining: Arc<Mutex<usize>>// Remaining fallback turns
```

### Default Configuration Constants

Defined in `init.rs`:

```rust
const DEFAULT_LEAD_TURNS: usize = 3;
const DEFAULT_FAILURE_THRESHOLD: usize = 2;
const DEFAULT_FALLBACK_TURNS: usize = 2;
```

### How Switching Works

1. **Initial phase (turns 0 to `lead_turns - 1`):** The lead (frontier) model handles all requests.
2. **Worker phase (turns >= `lead_turns`):** The worker (cheaper) model takes over.
3. **Failure detection:** After each completion, `handle_completion_result()` inspects the response for task-level failures.
4. **Fallback trigger:** If consecutive failures reach `max_failures_before_fallback` (default: 2) during the worker phase, the system enters fallback mode and switches back to the lead model for `fallback_turns` (default: 2) turns.
5. **Recovery:** After successful fallback turns, the worker resumes. The failure counter resets on any successful (non-failure) completion.

### Failure Detection Logic

The `detect_task_failures()` method inspects the response message for:

- **Failed tool requests** — malformed tool calls (`tool_call.is_err()`)
- **Failed tool executions** — tool responses containing errors
- **Error indicators in tool output** — strings like `"error:"`, `"syntax error"`, `"compilation failed"`, `"test failed"`, `"file not found"`, `"permission denied"`, etc.
- **User correction patterns** — phrases like `"that's wrong"`, `"try again"`, `"fix this"`, `"this is broken"`, `"actually, "`, etc.

A single failure indicator (`failure_indicators >= 1`) counts as a task failure.

**Important distinction:** Technical failures (API errors, network issues) do NOT increment the failure counter or turn count. Only task-level failures (bad code, wrong answers) trigger the fallback logic.

### Environment Variable Configuration

| Variable | Purpose | Default |
|----------|---------|---------|
| `GOOSE_LEAD_MODEL` | Model name for the lead (triggers lead/worker mode when set) | *required to enable* |
| `GOOSE_LEAD_PROVIDER` | Provider for the lead model (can differ from worker) | Same as default provider |
| `GOOSE_LEAD_TURNS` | Number of initial turns using the lead | `3` |
| `GOOSE_LEAD_FAILURE_THRESHOLD` | Consecutive failures before fallback | `2` |
| `GOOSE_LEAD_FALLBACK_TURNS` | Turns to use lead model during fallback | `2` |
| `GOOSE_LEAD_CONTEXT_LIMIT` | Context limit override for the lead model | Auto-detected |

The lead/worker mode is **activated by setting `GOOSE_LEAD_MODEL`**. The worker model is always the default `GOOSE_MODEL` configured for the session.

### Cross-Provider Lead/Worker

The lead and worker can use **different providers**. For example:
- Lead: `anthropic` / `claude-opus-4-5`
- Worker: `openai` / `gpt-4o-mini`

The `create_lead_worker_from_env()` function in `init.rs` constructs both providers independently from the registry.

**[Confirmed]** — Source: [`lead_worker.rs`](https://github.com/block/goose/blob/main/crates/goose/src/providers/lead_worker.rs), [`init.rs`](https://github.com/block/goose/blob/main/crates/goose/src/providers/init.rs)

---

## 4. Model Selection and Configuration

### Configuration Hierarchy (Priority Order)

1. **Environment variables** (highest priority) — `GOOSE_PROVIDER`, `GOOSE_MODEL`
2. **Config file** — `~/.config/block/goose/config.yaml` (Linux/macOS) or `%APPDATA%\Block\goose\config\config.yaml` (Windows)
3. **Interactive CLI** — `goose configure` walks through provider selection
4. **Provider defaults** — each provider specifies a `default_model`

**[Inferred]** — Based on code analysis of `Config::global()` and `get_param()` / `get_secret()` which check env vars first, then stored config. The exact priority chain is visible in the `crate::config` module.

### Key Environment Variables

| Variable | Purpose |
|----------|---------|
| `GOOSE_PROVIDER` | Provider identifier (e.g., `"anthropic"`, `"openai"`, `"ollama"`) |
| `GOOSE_MODEL` | Model name (e.g., `"claude-sonnet-4-5"`, `"gpt-4o"`) |
| `GOOSE_TEMPERATURE` | Sampling temperature |
| `GOOSE_MAX_TOKENS` | Maximum output tokens |
| `GOOSE_INPUT_LIMIT` | Context window override |
| `GOOSE_CONTEXT_LIMIT` | Context limit for the provider |
| Provider-specific keys | `ANTHROPIC_API_KEY`, `OPENAI_API_KEY`, `GOOGLE_API_KEY`, `OLLAMA_HOST`, etc. |
| `ANTHROPIC_HOST` | Custom Anthropic API endpoint |
| `OPENAI_HOST` | Custom OpenAI API endpoint |

**[Confirmed]** — Source: provider source files in [`providers/`](https://github.com/block/goose/tree/main/crates/goose/src/providers)

### Fast Model Support

Several providers define a "fast" model variant for cheaper/quicker operations (e.g., title generation):

- Anthropic: `claude-haiku-4-5` (fast variant of `claude-sonnet-4-5`)
- OpenAI: `gpt-4o-mini` (fast variant of `gpt-4o`)
- Google: `gemini-2.5-flash` (fast variant of `gemini-2.5-pro`)

The `complete_fast()` method on the `Provider` trait tries the fast model first and falls back to the regular model on failure.

**[Confirmed]** — Source: [`anthropic.rs`](https://github.com/block/goose/blob/main/crates/goose/src/providers/anthropic.rs), [`openai.rs`](https://github.com/block/goose/blob/main/crates/goose/src/providers/openai.rs), [`google.rs`](https://github.com/block/goose/blob/main/crates/goose/src/providers/google.rs)

### Auto-Detection

`auto_detect.rs` implements `detect_provider_from_api_key()` which takes an API key and tests it against multiple providers (Anthropic, OpenAI, Google, Groq, xAI) in parallel, returning the first provider that successfully lists models. This enables "paste your API key and Goose figures out which provider it belongs to."

**[Confirmed]** — Source: [`auto_detect.rs`](https://github.com/block/goose/blob/main/crates/goose/src/providers/auto_detect.rs)

---

## 5. Provider-Specific Adaptations

### Format Modules

Each provider family has a dedicated format module in `crates/goose/src/providers/formats/`:

| Format Module | Used By | Key Differences |
|--------------|---------|-----------------|
| `openai.rs` | OpenAI, Azure, xAI, OpenRouter, LiteLLM, all OpenAI-compatible declarative providers | Standard chat completions API. Tool calls via `functions` array. Streaming uses `delta.tool_calls` with index-based accumulation. |
| `anthropic.rs` | Anthropic, Bedrock (Anthropic models), Vertex AI (Claude) | Content blocks (`tool_use`, `tool_result`). Extended thinking support (`thinking`, `redacted_thinking` blocks). Cache control via `cache_control` field. Adaptive thinking for Claude 4.6 models. |
| `google.rs` | Google Gemini | `functionDeclarations` format. Different streaming chunk structure. |
| `ollama.rs` | Ollama | Reuses OpenAI format. Adds XML tool call fallback for Qwen3-coder models. Custom `num_ctx` option for context window. |
| `openai_responses.rs` | OpenAI (Responses API variant) | OpenAI's newer Responses API format alongside the traditional Chat Completions API. |
| `bedrock.rs` | AWS Bedrock | AWS SigV4 signing, Bedrock-specific request wrapping. |
| `gcpvertexai.rs` | GCP Vertex AI | GCP OAuth, Vertex AI endpoint construction. |
| `databricks.rs` | Databricks | Databricks-specific auth and endpoint handling. |
| `snowflake.rs` | Snowflake Cortex | Snowflake-specific auth and API format. |
| `openrouter.rs` | OpenRouter | OpenRouter-specific headers and model routing. |

**[Confirmed]** — Source: [`formats/` directory](https://github.com/block/goose/tree/main/crates/goose/src/providers/formats)

### Streaming

All providers default to streaming (`supports_streaming: true`). The `stream()` method returns a `MessageStream` (a pinned `Stream<Item = Result<StreamEvent>>`) that yields content deltas, tool call fragments, and usage information progressively. Providers that cannot stream (some custom endpoints) can set `supports_streaming: false` in their declarative config, which causes `complete()` to be used instead (though Anthropic explicitly rejects this configuration with an error).

**[Confirmed]** — Source: [`anthropic.rs`](https://github.com/block/goose/blob/main/crates/goose/src/providers/anthropic.rs)

### Extended Thinking (Anthropic-Specific)

The Anthropic format module supports three thinking modes:
- `Disabled` — no thinking blocks (default for non-Claude models)
- `Enabled` — always include thinking blocks
- `Adaptive` — server-side adaptive thinking (only for `claude-opus-4-6` and `claude-sonnet-4-6`)

Controlled by `CLAUDE_THINKING_TYPE` env var or model config param. The older `CLAUDE_THINKING_ENABLED` env var is still supported but emits a deprecation warning.

**[Confirmed]** — Source: [`formats/anthropic.rs`](https://github.com/block/goose/blob/main/crates/goose/src/providers/formats/anthropic.rs)

### Image Handling

Different providers expect images in different formats:
- **OpenAI**: `ImageFormat::OpenAi` — base64 data URLs in `image_url` content parts
- **Anthropic**: Anthropic-native image content blocks
- **Google**: Google-specific inline data format

The `utils.rs` module provides `detect_image_path()`, `load_image_file()`, and `convert_image()` helpers that handle format conversion.

**[Confirmed]** — Source: [`formats/openai.rs`](https://github.com/block/goose/blob/main/crates/goose/src/providers/formats/openai.rs)

### Function Name Sanitization

Some models produce invalid function names in tool calls. The `utils.rs` module provides `is_valid_function_name()` (checks against `[a-zA-Z0-9_-]+`) and `sanitize_function_name()` to handle this across all providers.

**[Confirmed]** — Source: [`utils.rs`](https://github.com/block/goose/blob/main/crates/goose/src/providers/utils.rs)

### Context Length Detection

The `check_context_length_exceeded()` function in `openai_compatible.rs` scans error messages for phrases like `"too long"`, `"context_length_exceeded"`, `"max_tokens"`, etc., to detect context limit errors across different provider error message formats.

**[Confirmed]** — Source: [`openai_compatible.rs`](https://github.com/block/goose/blob/main/crates/goose/src/providers/openai_compatible.rs)

### Retry Logic

A `ProviderRetry` trait and `RetryConfig` system provides configurable retry behavior. Providers use `self.with_retry(|| async { ... })` to wrap API calls with automatic retries on transient failures (rate limits, 5xx errors, network timeouts).

**[Confirmed]** — Source: [`retry.rs`](https://github.com/block/goose/blob/main/crates/goose/src/providers/retry.rs)

---

## 6. Local/Offline Model Support

### Ollama Integration

The `OllamaProvider` connects to a local Ollama instance:

- **Default host:** `localhost:11434` (configurable via `OLLAMA_HOST`)
- **Default model:** `qwen3` (previously was `qwen2.5-coder`)
- **Known models:** `qwen3`, `qwen3-vl`, `qwen3-coder:30b`, `qwen3-coder:480b-cloud`
- **Timeout:** 600 seconds (configurable via `OLLAMA_TIMEOUT`)
- **No authentication** by default
- **Context window:** Configurable via `GOOSE_INPUT_LIMIT` or model config's `context_limit`, passed as Ollama's `num_ctx` option

The Ollama provider handles XML tool call fallback for models (notably Qwen3-coder) that emit XML-style tool calls (`<function=name><parameter=key>value</parameter></function>`) instead of native JSON format when given 6+ tools. This is documented as affecting `qwen3-coder` and `qwen3-coder-32b` specifically.

**[Confirmed]** — Source: [`ollama.rs`](https://github.com/block/goose/blob/main/crates/goose/src/providers/ollama.rs), [`formats/ollama.rs`](https://github.com/block/goose/blob/main/crates/goose/src/providers/formats/ollama.rs)

### Local Inference (llama.cpp)

The `LocalInferenceProvider` provides fully offline inference using `llama-cpp-2` (Rust bindings to llama.cpp):

- **Default model:** `bartowski/Llama-3.2-1B-Instruct-GGUF:Q4_K_M`
- **GPU acceleration:** Detects GPU/accelerator devices and uses them when available. Checks for GPU, integrated GPU, and accelerator device types.
- **Memory-aware model selection:** `recommend_local_model()` checks available GPU/CPU memory and picks the largest "featured" model that fits. Falls back to the smallest featured model if nothing fits.
- **Model caching:** Uses an `InferenceRuntime` singleton with a global `Weak<InferenceRuntime>` pattern. Models are cached in memory slots (`HashMap<String, ModelSlot>`). The field order in `InferenceRuntime` is explicitly designed so models drop before the backend to avoid GPU resource assertions on shutdown.
- **Tool call support:** Two modes:
  - **Native tools** (`inference_native_tools`): For models with built-in tool call support via chat templates
  - **Emulated tools** (`inference_emulated_tools`): Uses a tool description prompt and text parsing to extract tool calls from models without native support
- **Model registry:** A `local_model_registry` tracks downloaded GGUF models with their file paths, sizes, context limits, and settings.

**[Confirmed]** — Source: [`local_inference.rs`](https://github.com/block/goose/blob/main/crates/goose/src/providers/local_inference.rs)

### Tool Shim for Non-Tool-Calling Models

The `toolshim.rs` module provides a `ToolInterpreter` trait that can send model output to a separate interpreter model to extract tool call intentions:

- **Purpose:** Enables tool calling with models that do not natively support it
- **Default interpreter model:** `mistral-nemo` via Ollama
- **Activation:** Set `GOOSE_TOOLSHIM=true` or `GOOSE_TOOLSHIM=1`
- **Configurable interpreter model:** `GOOSE_TOOLSHIM_OLLAMA_MODEL` env var
- **Mechanism:** Takes text output from any LLM, sends it to the interpreter model, uses structured output to extract tool call intentions, and augments the original message with proper tool call structs

**[Confirmed]** — Source: [`toolshim.rs`](https://github.com/block/goose/blob/main/crates/goose/src/providers/toolshim.rs)

### How Well Does Goose Work with Local Models?

**[Inferred from code analysis]:**
- Goose is designed to work with any LLM, but the quality of the agentic experience degrades significantly with smaller models that have weak tool-calling capabilities.
- The XML tool call fallback in the Ollama format module, the ToolShim system, and the emulated tools mode in local inference are all evidence that local models frequently fail at native tool calling, requiring multiple layers of workarounds.
- The `LocalInferenceProvider` defaults to a 1B parameter model (`Llama-3.2-1B-Instruct`), which is extremely small for complex agentic tasks. This suggests local inference is positioned as an experimental or lightweight feature, not the primary usage mode.
- The featured model system and memory-aware selection show Goose actively tries to optimize the local experience within hardware constraints.
- The Ollama provider's default model (`qwen3`) and long timeout (600s) indicate awareness that local inference is slower and requires patience.

---

## 7. Cost Tracking

### Token Counting

Goose tracks token usage through the `Usage` and `ProviderUsage` structs in `base.rs`:

```rust
pub struct Usage {
    pub input_tokens: Option<i32>,
    pub output_tokens: Option<i32>,
    pub total_tokens: Option<i32>,
}

pub struct ProviderUsage {
    pub model: String,
    pub usage: Usage,
}
```

All three token fields are `Option<i32>`, reflecting that not all providers return usage data. `total_tokens` is auto-calculated as `input + output` when not explicitly provided.

**[Confirmed]** — Source: [`base.rs`](https://github.com/block/goose/blob/main/crates/goose/src/providers/base.rs)

### Usage Estimation Fallback

When providers do not return token counts (common with local/smaller providers), the `usage_estimator.rs` module fills in estimates:

1. If both `input_tokens` and `output_tokens` are already present, it does nothing.
2. Otherwise, it creates a `TokenCounter` (via `create_token_counter()`) and estimates:
   - **Input tokens:** counted from the system prompt, request messages, and tools
   - **Output tokens:** counted from the response message content
3. `total_tokens` is always calculated as `input + output` if not provided.

This ensures every `ProviderUsage` has token counts, even if estimated. The `ensure_tokens()` method on `ProviderUsage` wraps this logic.

**[Confirmed]** — Source: [`usage_estimator.rs`](https://github.com/block/goose/blob/main/crates/goose/src/providers/usage_estimator.rs)

### Cost Information in ModelInfo

The `ModelInfo` struct supports optional per-token cost data:

```rust
pub struct ModelInfo {
    pub name: String,
    pub context_limit: usize,
    pub input_token_cost: Option<f64>,   // Cost per token for input in USD
    pub output_token_cost: Option<f64>,  // Cost per token for output in USD
    pub currency: Option<String>,        // Default: "$"
    pub supports_cache_control: Option<bool>,
}
```

The `ModelInfo::with_cost()` constructor creates entries with pricing data pre-populated.

**[Confirmed]** — Source: [`base.rs`](https://github.com/block/goose/blob/main/crates/goose/src/providers/base.rs)

### Session Cost Display

- **Desktop app:** Shows a colored circle indicator next to the model name that changes from green (normal) to orange (80% of context consumed). Hovering reveals a detailed cost breakdown by model.
- **CLI:** Estimated cost shown when `GOOSE_CLI_SHOW_COST` is set.
- **Pricing data:** Regularly fetched from the OpenRouter API and cached locally.
- **Important caveat:** Costs shown are **estimates only**, not connected to actual provider billing.

**[Confirmed]** — Source: [Smart Context Management docs](https://block.github.io/goose/docs/guides/sessions/smart-context-management/)

### Budget Management

Goose does not have explicit budget caps or spending limits. Cost management is indirect:
- **Context compaction** triggers at approximately 80% of the token limit, summarizing older conversation history to stay within bounds.
- The lead/worker pattern is itself a cost optimization strategy — using cheaper models for routine tasks after the initial complex setup.
- There was a [feature request (Issue #1703)](https://github.com/block/goose/issues/1703) for token usage tracking and warnings, suggesting this was a community-identified gap.

**[Unknown]:** Whether a hard budget cap feature (e.g., "stop after $X spent") has been implemented. Based on code analysis of the provider layer, no such feature exists there. It may exist in the session/agent layer.

---

## 8. Declarative Provider System

### How It Works

Goose supports adding new providers through JSON configuration files without modifying Rust code. There are two tiers:

#### Tier 1: Built-in Declarative Providers

JSON files in `crates/goose/src/providers/declarative/` are compiled into the binary via `include_dir!()` from the `include_dir` crate. These are shipped with Goose and loaded at startup alongside native providers.

Example (`groq.json`):

```json
{
  "name": "groq",
  "engine": "openai",
  "display_name": "Groq (d)",
  "description": "Fast inference with Groq hardware",
  "api_key_env": "GROQ_API_KEY",
  "base_url": "https://api.groq.com/openai/v1/chat/completions",
  "models": [
    {
      "name": "llama-3.3-70b-versatile",
      "context_limit": 131072,
      "max_tokens": 32768
    }
  ],
  "supports_streaming": true
}
```

Example (`deepseek.json`):

```json
{
  "name": "custom_deepseek",
  "engine": "openai",
  "display_name": "DeepSeek",
  "api_key_env": "DEEPSEEK_API_KEY",
  "base_url": "https://api.deepseek.com",
  "models": [
    { "name": "deepseek-chat", "context_limit": 128000 },
    { "name": "deepseek-reasoner", "context_limit": 128000 }
  ],
  "supports_streaming": true
}
```

#### Tier 2: User Custom Providers

Users can add providers at runtime via:
1. **Desktop app settings** — Add custom provider UI
2. **Manual file creation** — Place JSON files in `~/.config/block/goose/custom_providers/`
3. **CLI** — `goose configure`

Custom provider IDs are auto-generated with a `custom_` prefix (e.g., `custom_my_endpoint`). An ID generation function prevents collisions by appending numeric suffixes if needed.

**[Confirmed]** — Source: [`declarative_providers.rs`](https://github.com/block/goose/blob/main/crates/goose/src/config/declarative_providers.rs)

### DeclarativeProviderConfig Schema

```rust
pub struct DeclarativeProviderConfig {
    pub name: String,                              // Unique provider ID
    pub engine: ProviderEngine,                    // "openai", "ollama", or "anthropic"
    pub display_name: String,                      // Human-readable name
    pub description: Option<String>,               // Provider description
    pub api_key_env: String,                       // Env var name for the API key
    pub base_url: String,                          // API endpoint URL
    pub models: Vec<ModelInfo>,                    // Known models with context limits
    pub headers: Option<HashMap<String, String>>,  // Custom HTTP headers
    pub timeout_seconds: Option<u64>,              // Request timeout override
    pub supports_streaming: Option<bool>,          // Whether streaming is supported
    pub requires_auth: bool,                       // Whether auth is needed (default: true)
    pub catalog_provider_id: Option<String>,       // Link to catalog entry
    pub base_path: Option<String>,                 // Custom path prefix for API endpoint
}
```

### Three Engine Types

The `ProviderEngine` enum determines which format module handles the API communication:

| Engine | Maps To | Used For |
|--------|---------|----------|
| `openai` | `OpenAiProvider::from_custom_config()` | Most cloud APIs (Groq, DeepSeek, Cerebras, Mistral, etc.) |
| `anthropic` | `AnthropicProvider::from_custom_config()` | Anthropic-compatible endpoints |
| `ollama` | `OllamaProvider::from_custom_config()` | Ollama-compatible local endpoints (e.g., LM Studio) |

Each of the three native providers (`OpenAiProvider`, `AnthropicProvider`, `OllamaProvider`) implements a `from_custom_config()` constructor that accepts a `DeclarativeProviderConfig` and configures itself accordingly — setting custom base URLs, headers, auth, timeouts, and streaming support.

**[Confirmed]** — Source: [`declarative_providers.rs`](https://github.com/block/goose/blob/main/crates/goose/src/config/declarative_providers.rs), [`anthropic.rs`](https://github.com/block/goose/blob/main/crates/goose/src/providers/anthropic.rs), [`ollama.rs`](https://github.com/block/goose/blob/main/crates/goose/src/providers/ollama.rs)

### Provider Type Classification

The registry tracks four provider types:

```rust
pub enum ProviderType {
    Preferred,    // Built-in, shown prominently in UI
    Builtin,      // Built-in, but not featured in UI
    Declarative,  // Shipped JSON-configured providers
    Custom,       // User-created providers
}
```

**[Confirmed]** — Source: [`base.rs`](https://github.com/block/goose/blob/main/crates/goose/src/providers/base.rs)

### Dynamic Reloading

The `refresh_custom_providers()` function:
1. Removes all entries from the registry whose names start with `custom_`
2. Re-reads JSON files from the custom providers directory
3. Re-registers them with the `Custom` provider type

This allows adding or modifying providers without restarting Goose.

**[Confirmed]** — Source: [`init.rs`](https://github.com/block/goose/blob/main/crates/goose/src/providers/init.rs)

### Provider Catalog Integration

The catalog system (`catalog.rs`) enables a streamlined "add from catalog" flow:
1. `get_providers_by_format(ProviderFormat::OpenAI)` returns all OpenAI-compatible providers from the bundled catalog that are not already registered as native providers
2. `get_provider_template(provider_id)` returns a `ProviderTemplate` with pre-populated models, context limits, capabilities (tool_call, reasoning, attachment, temperature), and API URLs
3. The UI pre-fills the custom provider form from this template, reducing configuration to just entering an API key

The catalog reads from a bundled `provider_metadata.json` containing hundreds of providers with their npm package references (used for format detection), API URLs, env var names, and model counts.

**[Confirmed]** — Source: [`catalog.rs`](https://github.com/block/goose/blob/main/crates/goose/src/providers/catalog.rs)

---

## Summary of Key Architectural Decisions

1. **Streaming-first:** The `Provider` trait's primary method is `stream()`, not `complete()`. All providers are expected to stream by default.
2. **Format layer separation:** Provider-specific API format conversion is isolated into `formats/` modules, keeping provider implementations focused on auth and HTTP mechanics.
3. **Three API format families:** All providers ultimately map to OpenAI, Anthropic, or Ollama wire formats. No provider needs a completely novel format implementation.
4. **Declarative extensibility:** New OpenAI/Anthropic/Ollama-compatible providers can be added via JSON without code changes, at both build time (compiled-in) and runtime (user custom).
5. **Canonical model registry:** A shared registry provides context limits and capabilities across all providers, enabling consistent behavior regardless of which provider is used.
6. **Cost as estimates:** Token counting and cost tracking are best-effort, with fallback estimation for providers that do not report usage.
7. **Failure-aware routing:** The lead/worker pattern is not just turn-based. It actively detects task failures (bad tool calls, error outputs, user corrections) and adapts by switching models.
8. **Multiple local model strategies:** Three approaches (Ollama, llama.cpp local inference, ToolShim) address different local model scenarios with increasing levels of workaround for tool-calling deficiencies.

---

## Sources

- [Goose GitHub Repository](https://github.com/block/goose)
- [Goose Architecture Documentation](https://block.github.io/goose/docs/goose-architecture/)
- [Configure LLM Provider](https://block.github.io/goose/docs/getting-started/providers/)
- [Lead/Worker Multi-Model Setup Tutorial](https://block.github.io/goose/docs/tutorials/lead-worker/)
- [Smart Context Management](https://block.github.io/goose/docs/guides/sessions/smart-context-management/)
- [Provider Configuration — DeepWiki](https://deepwiki.com/block/goose/2.2-provider-configuration)
- [Block Introduces Codename Goose](https://block.xyz/inside/block-open-source-introduces-codename-goose)
- [Token Usage Tracking Feature Request #1703](https://github.com/block/goose/issues/1703)
- [Docker + Goose Blog Post](https://www.docker.com/blog/building-ai-agents-with-goose-and-docker/)
- [Tetrate Agent Router + Goose](https://tetrate.io/blog/frictionless-setup-of-goose-with-tetrate-agent-router-service/)
- [InfoQ: Block Launches Open-Source AI Framework Codename Goose](https://www.infoq.com/news/2025/02/codename-goose/)
- Source code files: `base.rs`, `init.rs`, `lead_worker.rs`, `provider_registry.rs`, `catalog.rs`, `declarative_providers.rs`, `usage_estimator.rs`, `auto_detect.rs`, `toolshim.rs`, `local_inference.rs`, `ollama.rs`, `anthropic.rs`, `openai.rs`, `google.rs`, `openai_compatible.rs`, and all `formats/*.rs` modules
