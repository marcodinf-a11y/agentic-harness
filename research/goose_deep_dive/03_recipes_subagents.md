# Goose Deep Dive: Recipes & Subagent System

**Research Date:** 2026-03-07
**Researcher:** Automated research agent
**Primary Sources:** Goose GitHub repository (block/goose), official documentation, Stripe engineering blog

---

## Table of Contents

1. [Recipe System](#1-recipe-system)
2. [Subrecipes](#2-subrecipes)
3. [Subagent System](#3-subagent-system)
4. [Subagent Constraints](#4-subagent-constraints)
5. [Orchestration Patterns](#5-orchestration-patterns)
6. [Recipe Discovery and Sharing](#6-recipe-discovery-and-sharing)
7. [Comparison with Stripe Blueprint Pattern](#7-comparison-with-stripe-blueprint-pattern)
8. [Sources](#sources)

---

## 1. Recipe System

### What Recipes Are

**[Confirmed]** Recipes are reusable YAML (or JSON) configuration files that package a specific agent setup -- instructions, extensions, parameters, and prompt -- so it can be easily shared and launched by others. They function as templates for common development workflows such as code reviews, security audits, documentation generation, and data analysis.

Source: [Recipe Reference Guide](https://block.github.io/goose/docs/guides/recipes/recipe-reference/), `crates/goose/src/recipe/mod.rs`

### YAML Schema / Struct Definition

**[Confirmed]** The canonical `Recipe` struct is defined in `crates/goose/src/recipe/mod.rs`. The full schema:

```rust
pub struct Recipe {
    pub version: String,           // Required. Semver, defaults to "1.0.0"
    pub title: String,             // Required. Short title
    pub description: String,       // Required. Longer description
    pub instructions: Option<String>,  // System instructions for the model
    pub prompt: Option<String>,        // Initial user prompt to start the session
    pub extensions: Option<Vec<ExtensionConfig>>,  // Extensions to enable
    pub settings: Option<Settings>,    // Provider/model/temperature/max_turns
    pub activities: Option<Vec<String>>,  // Activity pills shown in UI
    pub author: Option<Author>,        // Contact and metadata
    pub parameters: Option<Vec<RecipeParameter>>,  // Parameterization
    pub response: Option<Response>,    // JSON schema for structured output
    pub sub_recipes: Option<Vec<SubRecipe>>,  // Sub-recipe definitions
    pub retry: Option<RetryConfig>,    // Retry configuration
}
```

**Validation rule:** At least one of `instructions` or `prompt` must be set. Both `title` and `description` are required.

Source: `crates/goose/src/recipe/mod.rs` (full struct with serde attributes)

### Settings Sub-Schema

**[Confirmed]** The `Settings` struct allows overriding the LLM configuration per-recipe:

```rust
pub struct Settings {
    pub goose_provider: Option<String>,  // e.g. "openai", "anthropic"
    pub goose_model: Option<String>,     // e.g. "gpt-4", "claude-sonnet"
    pub temperature: Option<f32>,
    pub max_turns: Option<usize>,
}
```

### Parameter System

**[Confirmed]** Recipes support typed parameters with validation:

```rust
pub struct RecipeParameter {
    pub key: String,
    pub input_type: RecipeParameterInputType,  // String, Number, Boolean, Date, File, Select
    pub requirement: RecipeParameterRequirement,  // Required, Optional, UserPrompt
    pub description: String,
    pub default: Option<String>,
    pub options: Option<Vec<String>>,  // For Select type
}
```

Parameters are referenced in instructions and prompts using Jinja2 template syntax: `{{ parameter_name }}`. The `File` input type imports content from a file path and explicitly cannot have default values to prevent importing sensitive user files.

Source: `crates/goose/src/recipe/mod.rs`

### Extension Configuration

**[Confirmed]** Extensions can be configured in several formats within a recipe:

- **Builtin:** `{ type: builtin, name: developer }` -- Enables built-in extensions like `developer`, `computercontroller`
- **Stdio:** `{ type: stdio, name: ..., cmd: ..., args: [...], timeout: ... }` -- External MCP servers launched via stdio
- **Platform:** `{ type: platform, name: ... }` -- Platform-provided extensions like `analyze`, `summon`
- **Inline Python:** `{ type: inline_python, name: ..., code: ..., dependencies: [...] }` -- Python code executed as an extension

Source: Unit tests in `crates/goose/src/recipe/mod.rs`

### Real Recipe Example

**[Confirmed]** The repo root contains `recipe.yaml`, a "404Portfolio" recipe that creates personalized 404 error pages using public profile data. Another example is the `workflow_recipes/release_risk_check/recipe.yaml`, a multi-step release risk assessment that:

1. Runs a Python script to generate heuristic risk scores
2. Feeds MEDIUM/HIGH risk PRs through AI review
3. Combines outputs into a final report

This recipe uses `{{ recipe_dir }}` and `{{ version }}` template parameters, demonstrating the template system in action.

Source: `recipe.yaml`, `workflow_recipes/release_risk_check/recipe.yaml`

### Nested Recipe Format

**[Confirmed]** Goose supports a nested format where the recipe is wrapped under a `recipe:` key alongside metadata fields like `name` and `isGlobal`:

```yaml
name: test_recipe
recipe:
  title: Nested Recipe Test
  description: A test recipe with nested structure
  instructions: ...
isGlobal: true
```

The parser (`Recipe::from_content`) auto-detects this format and extracts the inner recipe.

Source: Unit test `test_from_content_with_nested_recipe_yaml` in `crates/goose/src/recipe/mod.rs`

### Security

**[Confirmed]** Recipes are scanned for Unicode tag characters (e.g., `\u{E0041}`) in instructions, prompt, and activities fields via `check_for_security_warnings()`. This prevents prompt injection attacks through invisible Unicode sequences.

Source: `crates/goose/src/recipe/mod.rs`

---

## 2. Subrecipes

### Definition and Schema

**[Confirmed]** Subrecipes are defined in the parent recipe's `sub_recipes` array. Each entry has the following schema:

```rust
pub struct SubRecipe {
    pub name: String,                          // Identifier for the subrecipe
    pub path: String,                          // File path to the subrecipe YAML
    pub values: Option<HashMap<String, String>>, // Pre-set parameter values
    pub sequential_when_repeated: bool,         // Default: false
    pub description: Option<String>,            // Optional description override
}
```

Example YAML:

```yaml
sub_recipes:
  - name: "security_scan"
    path: "./subrecipes/security-analysis.yaml"
    values:
      scan_level: "comprehensive"
      include_dependencies: "true"
  - name: "quality_check"
    path: "./subrecipes/quality-analysis.yaml"
    description: "Performs code quality analysis"
    sequential_when_repeated: true
```

Source: `crates/goose/src/recipe/mod.rs` (SubRecipe struct)

### How Subrecipes Work

**[Confirmed]** When a recipe with `sub_recipes` is loaded, Goose automatically injects the `summon` platform extension (via `ensure_summon_for_subrecipes()`). This extension provides two tools:

1. **`load`** -- Injects subrecipe knowledge into current context, or lists available sources
2. **`delegate`** -- Runs the subrecipe as an isolated subagent

The main agent sees subrecipes as available "sources" and can invoke them via the `delegate` tool. The subrecipe file is loaded, its template parameters are merged (pre-set `values` from the parent + runtime `parameters` from the delegate call), and the result is built into a full `Recipe` via `build_recipe_from_template()`.

Source: `crates/goose/src/agents/platform_extensions/summon.rs`

### Subrecipe Nesting Rules

**[Confirmed]** Subrecipes cannot define their own sub-recipes. This is enforced at the subagent level: when a session has `SessionType::SubAgent`, the `delegate` tool is not exposed in the tool list. The `list_tools` implementation explicitly filters it out:

```rust
if !is_subagent {
    tools.push(self.create_delegate_tool());
}
```

Additionally, the `handle_delegate` method checks:

```rust
if session.session_type == SessionType::SubAgent {
    return Err("Delegated tasks cannot spawn further delegations".to_string());
}
```

Source: `crates/goose/src/agents/platform_extensions/summon.rs` (McpClientTrait impl)

### Parameter Passing

**[Confirmed]** Subrecipes receive parameters in two ways:

1. **Pre-set values:** Fixed values defined in the parent recipe's `values` field, automatically provided and not overridable at runtime
2. **Runtime parameters:** Passed by the main agent via the `delegate` tool's `parameters` argument, which can extract values from conversation context or prior subrecipe results

When both are present, they are merged, with runtime parameters taking precedence over pre-set values:

```rust
let mut merged: HashMap<String, String> = HashMap::new();
if let Some(values) = &sr.values {
    for (k, v) in values { merged.insert(k.clone(), v.clone()); }
}
if let Some(provided_params) = &params.parameters {
    for (k, v) in provided_params { merged.insert(k.clone(), value_str); }
}
```

Source: `crates/goose/src/agents/platform_extensions/summon.rs` (`build_source_recipe`)

### Sequential vs Parallel Execution

**[Confirmed]** By default, when the same subrecipe is invoked multiple times with different parameters, instances run concurrently (in parallel). The `sequential_when_repeated: true` flag on a SubRecipe forces sequential execution, overriding any user request for parallelism.

Parallel execution is achieved through the `async: true` parameter on the `delegate` tool, which spawns background tasks. The main agent can then:
1. Continue with other work
2. Dispatch additional async delegates
3. Use `load(source: "<task_id>")` to wait for and retrieve results

Real-time progress tracking is available during parallel execution, showing task IDs, parameter sets, timing, output previews, and errors.

**[Confirmed]** Parallel execution is described as an experimental feature in active development.

Source: [Running Subrecipes In Parallel](https://block.github.io/goose/docs/tutorials/subrecipes-in-parallel/), `crates/goose/src/agents/platform_extensions/summon.rs`

---

## 3. Subagent System

### Architecture Overview

**[Confirmed]** The subagent system is implemented across several key files:

| File | Purpose |
|------|---------|
| `crates/goose/src/agents/subagent_handler.rs` | Core subagent execution logic |
| `crates/goose/src/agents/subagent_task_config.rs` | Task configuration (provider, extensions, max turns) |
| `crates/goose/src/agents/subagent_execution_tool/mod.rs` | Module entry point |
| `crates/goose/src/agents/subagent_execution_tool/notification_events.rs` | Task status tracking and MCP notifications |
| `crates/goose/src/agents/platform_extensions/summon.rs` | Unified tooling (load/delegate) -- 2,184 lines |
| `crates/goose/src/prompts/subagent_system.md` | System prompt template for subagents |

### How Subagents Are Spawned

**[Confirmed]** When the `delegate` tool is called, the following sequence occurs:

1. **Recipe construction:** An ad-hoc recipe is built from instructions, or a source recipe/subrecipe/skill/agent is loaded and parameterized
2. **Task config creation:** Provider, extensions, and max turns are resolved (with inheritance from parent session)
3. **Agent config creation:** A new `AgentConfig` is created with session manager, permission manager, and `GooseMode::Auto`
4. **Session creation:** A new session of type `SessionType::SubAgent` is created via the session manager
5. **Notification channel:** An unbounded MPSC channel is set up for streaming tool-call notifications back to the parent
6. **Subagent execution:** `run_subagent_task()` is called with `SubagentRunParams`, which:
   - Creates a new `Agent` instance via `Agent::with_config()`
   - Sets the provider on the subagent
   - Adds each extension from the task config
   - Applies response schema (if any) for structured output
   - Builds a specialized system prompt via `build_subagent_prompt()`
   - Sends the user message and streams replies

Source: `crates/goose/src/agents/subagent_handler.rs`, `crates/goose/src/agents/platform_extensions/summon.rs`

### The `SubagentRunParams` Struct

**[Confirmed]** All subagent execution parameters are bundled into a single struct:

```rust
pub struct SubagentRunParams {
    pub config: AgentConfig,
    pub recipe: Recipe,
    pub task_config: TaskConfig,
    pub return_last_only: bool,
    pub session_id: String,
    pub cancellation_token: Option<CancellationToken>,
    pub on_message: Option<OnMessageCallback>,
    pub notification_tx: Option<tokio::sync::mpsc::UnboundedSender<ServerNotification>>,
}
```

Source: `crates/goose/src/agents/subagent_handler.rs`

### Subagent System Prompt

**[Confirmed]** Subagents receive a specialized system prompt (from `crates/goose/src/prompts/subagent_system.md`) that defines:

- **Role:** "Specialized subagent within the goose AI framework, created by Block"
- **Independence:** Makes decisions and executes tools within scope
- **Bounded operation:** Operates within defined turn limits
- **Security:** Cannot spawn additional subagents
- **Tool efficiency:** Must use minimum tools needed, avoid exploratory usage
- **Communication:** Progress updates, clear completion indication, Markdown formatting

The prompt is templated with Jinja2, receiving context via `SubagentPromptContext`:

```rust
pub struct SubagentPromptContext {
    pub max_turns: usize,
    pub subagent_id: String,
    pub task_instructions: String,
    pub tool_count: usize,
    pub available_tools: String,
}
```

Full prompt text includes directives like: "Use the minimum number of tools needed to complete your task", "Avoid exploratory tool usage unless explicitly required", and "You are part of a larger system. Your specialized focus helps the main agent handle multiple concerns efficiently."

Source: `crates/goose/src/prompts/subagent_system.md`, `crates/goose/src/agents/subagent_handler.rs`

### Synchronous vs Asynchronous Execution

**[Confirmed]** The `delegate` tool supports two modes:

1. **Synchronous (default, `async: false`):** The parent agent blocks until the subagent completes, then receives the result inline
2. **Asynchronous (`async: true`):** The subagent runs as a background Tokio task. The parent receives a task ID immediately and can retrieve results later via `load(source: "<task_id>")`

Background tasks are tracked with:
- Atomic turn counter (`AtomicU32`)
- Last activity timestamp (`AtomicU64`)
- Cancellation token (`CancellationToken`)
- `JoinHandle` for the spawned future
- Notification buffer for streaming events

Source: `crates/goose/src/agents/platform_extensions/summon.rs` (`BackgroundTask` struct, `handle_async_delegate`)

### Result Communication

**[Confirmed]** Subagent results are communicated back in two ways:

1. **Text extraction:** By default, `extract_response_text()` collects all text content from the conversation. If `return_last_only` is true (the default for delegate calls), only the last message's text is returned.
2. **Structured output:** If the recipe defines a `response.json_schema`, a `final_output_tool` is used to capture structured JSON output from the subagent.

Tool-call notifications from subagents are streamed to the parent via MCP `LoggingMessageNotification` events, tagged with `type: "subagent_tool_request"` and the subagent's session ID.

Source: `crates/goose/src/agents/subagent_handler.rs`

### Task Status Tracking

**[Confirmed]** The notification system tracks tasks through four states:

```rust
pub enum TaskStatus { Pending, Running, Completed, Failed }
```

Events include:
- `LineOutput` -- streaming output from a specific task
- `TasksUpdate` -- progress stats (total, pending, running, completed, failed counts)
- `TasksComplete` -- final stats with success rate and list of failed tasks

Source: `crates/goose/src/agents/subagent_execution_tool/notification_events.rs`

---

## 4. Subagent Constraints

### Max Turns

**[Confirmed]** The default maximum number of turns for subagent execution is **25** (`DEFAULT_SUBAGENT_MAX_TURNS`). This can be overridden through a priority chain:

1. Environment variable `GOOSE_SUBAGENT_MAX_TURNS` (highest priority)
2. Recipe `settings.max_turns`
3. Global config parameter `GOOSE_SUBAGENT_MAX_TURNS`
4. Default of 25 (lowest priority)

The max turns value is injected into the subagent's system prompt so the LLM is aware of its bounds: "The maximum number of turns to respond is {{max_turns}}."

Source: `crates/goose/src/agents/subagent_task_config.rs`, `crates/goose/src/agents/platform_extensions/summon.rs` (`resolve_max_turns`)

### No Recursive Spawning

**[Confirmed]** Subagents are explicitly prevented from spawning further subagents through two mechanisms:

1. **Tool-level hiding:** The `delegate` tool is not listed when `session_type == SessionType::SubAgent`
2. **Runtime check (defense in depth):** An explicit error is returned: `"Delegated tasks cannot spawn further delegations"`

Source: `crates/goose/src/agents/platform_extensions/summon.rs`

### Tool Subset Restrictions

**[Confirmed]** Subagents can operate with restricted tool access via the `extensions` parameter on the `delegate` tool:

- **Omit `extensions`:** Inherits all extensions from the parent session
- **Empty array `[]`:** No extensions (bare subagent with only platform tools)
- **Specific names `["developer", "analyze"]`:** Only named extensions are enabled

The filtering is applied in `build_task_config()`:
```rust
if let Some(filter) = &params.extensions {
    if filter.is_empty() { extensions = Vec::new(); }
    else { extensions.retain(|ext| filter.contains(&ext.name())); }
}
```

Source: `crates/goose/src/agents/platform_extensions/summon.rs`

### Background Task Limits

**[Confirmed]** The maximum number of concurrent background (async) tasks defaults to **5**, configurable via `GOOSE_MAX_BACKGROUND_TASKS` environment variable. Attempting to exceed this returns: "Maximum N background tasks already running. Wait for completion or use sync mode."

Source: `crates/goose/src/agents/platform_extensions/summon.rs` (`max_background_tasks()`)

### Timeout Behavior

**[Confirmed]** When waiting for a background task via `load(source: "<task_id>")`, there is a **5-minute (300-second)** timeout. If the task is still running after 5 minutes, the tool returns an error suggesting the user try again or cancel.

Cancellation is supported via `load(source: "<task_id>", cancel: true)`, which:
1. Triggers the `CancellationToken`
2. Waits up to 5 seconds for graceful shutdown
3. Aborts the task handle if it does not stop in time

Source: `crates/goose/src/agents/platform_extensions/summon.rs` (`handle_load_task_result`)

### Provider Override for Subagents

**[Confirmed]** Subagents can use a different LLM provider/model than the parent. Provider resolution follows this priority:

1. Explicit `provider`/`model` parameters on the delegate call (highest)
2. Recipe `settings.goose_provider`/`goose_model`
3. Global config `GOOSE_SUBAGENT_PROVIDER`/`GOOSE_SUBAGENT_MODEL`
4. Parent session's provider/model (lowest)

Temperature can also be overridden at each level.

Source: `crates/goose/src/agents/platform_extensions/summon.rs` (`resolve_provider`)

---

## 5. Orchestration Patterns

### What's Possible

**[Confirmed]** The recipe + subagent system enables several orchestration patterns:

1. **Sequential pipeline:** A recipe's instructions describe multi-step workflows. The main agent executes steps in order, potentially delegating specific steps to subagents. Example: the release risk check recipe runs a Python script, feeds output to LLM analysis, then combines results.

2. **Fan-out/fan-in (parallel):** Multiple async delegates are spawned for independent tasks (e.g., analyzing multiple repos, processing documents), then results are collected via `load()` and synthesized by the main agent. The delegate tool documentation explicitly describes: "Decompose -> async delegates -> load(taskId) for each -> synthesize."

3. **Specialized workers:** Different subrecipes handle different concerns (security scan, quality check, documentation) with their own instructions, extensions, and even different LLM providers/models.

4. **Mixed read/write with guidance:** The delegate tool documentation recommends: "Research (read-only): parallelize freely -- delegates explore and report back. Work (writes): partition files strictly -- no two delegates touch the same file."

5. **Multi-source delegation:** Beyond subrecipes, the `delegate` tool can run:
   - **Skills** (SKILL.md files with frontmatter defining name/description)
   - **Agents** (.md files with frontmatter including model preference)
   - **Ad-hoc tasks** (raw instructions with no source)
   - **Combined** (source + additional instructions)

6. **Scheduled execution:** Recipes can be scheduled via the `platform__manage_schedule` tool with cron expressions (5- or 6-field) for recurring automated workflows. Supports list, create, run_now, pause, unpause, delete, kill, inspect, sessions, and session_content actions.

7. **Background task monitoring (MOIM):** The `get_moim` method provides a status overview of all background tasks, showing running tasks (with elapsed time, turns, idle time) and completed tasks awaiting result retrieval.

### Source Discovery Hierarchy

**[Confirmed]** The `summon` extension discovers sources from five categories, searched in priority order:

| Kind | Discovery Locations |
|------|-------------------|
| Subrecipe | Current recipe's `sub_recipes` array |
| Recipe | Working dir, `.goose/recipes/`, `GOOSE_RECIPE_PATH`, `~/.config/goose/recipes/` |
| Skill | `.goose/skills/`, `.claude/skills/`, `.agents/skills/`, `~/.config/goose/skills/`, `~/.claude/skills/` |
| Agent | `.goose/agents/`, `.claude/agents/`, `~/.config/goose/agents/`, `~/.claude/agents/` |
| Builtin Skill | Compiled-in skill files via `builtin_skills::get_all()` |

First-found wins (deduplication by name). Sources are cached for 60 seconds. Notable: Goose also scans `.claude/` directories, suggesting cross-compatibility with Claude Code's skill/agent conventions.

Source: `crates/goose/src/agents/platform_extensions/summon.rs` (`discover_filesystem_sources`)

### What's Limited

**[Inferred]** Based on the architecture:

1. **Single-level delegation only:** No recursive subagent spawning. Complex multi-layer orchestration requires the main agent to mediate all subagent interactions.

2. **No inter-subagent communication:** Subagents are fully isolated. They cannot share state, coordinate, or communicate with each other. The main agent must serve as the sole coordination point.

3. **No conditional branching in recipes:** The recipe schema has no built-in control flow (if/then/else, loops). All branching logic depends on the LLM's interpretation of instructions.

4. **No persistent subagent state:** Each subagent runs in a fresh session. There is no mechanism for subagent memory across invocations.

5. **Concurrent write conflicts:** There is no file-locking mechanism. Parallel subagents writing to the same files will conflict. This is mitigated only by prompt guidance ("partition files strictly").

6. **5-task concurrency ceiling:** The default limit of 5 background tasks may be insufficient for large fan-out operations. This is configurable via environment variable but still represents a hard limit.

---

## 6. Recipe Discovery and Sharing

### Local Discovery

**[Confirmed]** Recipes are discovered from multiple local paths, searched in order:

1. Current working directory (`.`)
2. `GOOSE_RECIPE_PATH` environment variable (colon-separated on Unix, semicolon on Windows)
3. `~/.config/goose/recipes/` (global recipe library)
4. `.goose/recipes/` (project-local recipe library)

Files must have `.yaml` or `.json` extensions. The `list_local_recipes()` function scans all directories. Paths are canonicalized and deduplicated before scanning.

Source: `crates/goose/src/recipe/local_recipes.rs`

### GitHub Recipe Loading

**[Confirmed]** Recipes can be loaded from GitHub. The CLI includes `crates/goose-cli/src/recipes/github_recipe.rs` for fetching recipes from remote repositories.

**[Inferred]** Based on file naming and the deep-link system, this likely supports loading recipes by GitHub URL or repository reference, enabling sharing via links like `https://block.github.io/goose/docs/recipes?recipe=<encoded>`.

Source: `crates/goose-cli/src/recipes/github_recipe.rs` (file existence confirmed)

### Deep Links

**[Confirmed]** Goose supports creating shareable deep-link URLs that encode a recipe configuration. Others can use these URLs to launch a session with predefined settings. The CLI provides a `deeplink` command for generating these URLs.

Source: [Goose Recipes Guide](https://block.github.io/goose/docs/guides/recipes/)

### Community Cookbook

**[Confirmed]** Block maintains a community recipe cookbook. Contributed recipes are stored in `documentation/src/pages/recipes/data/recipes/`. The contribution process is documented in `CONTRIBUTING_RECIPES.md`:

1. Fork the repository
2. Add a recipe YAML file to the recipes directory
3. Submit a PR (auto-validated for YAML syntax, required fields, proper structure)

Block offered OpenRouter LLM credits ($10) to the first 50 approved recipe contributors as an incentive.

Source: [CONTRIBUTING_RECIPES.md](https://github.com/block/goose/blob/main/CONTRIBUTING_RECIPES.md)

### Recipe CLI Commands

**[Confirmed]** The CLI provides recipe management commands:
- `validate` -- Check recipe syntax and structure
- `deeplink` -- Generate shareable URLs
- Recipe search functionality (`search_recipe.rs`)
- Secret discovery in recipes (`secret_discovery.rs`)
- Recipe extraction from CLI sessions (`extract_from_cli.rs`)
- Recipe printing/display (`print_recipe.rs`)

**[Unknown]** Whether a `goose recipe list` command exists -- this was an open feature request (GitHub issue #2814).

Source: `crates/goose-cli/src/recipes/` directory listing

### Recipe Scanner

**[Confirmed]** The repository contains a `recipe-scanner/` top-level directory, suggesting batch tooling for scanning and validating recipe collections.

Source: Repository file listing

---

## 7. Comparison with Stripe Blueprint Pattern

### Stripe's Minions Architecture

**[Confirmed]** Stripe's "Minions" coding agents are built on a fork of Block's open-source Goose agent, customized with deep internal tool integrations. Minions are responsible for 1,000+ pull requests merged per week, where humans review the code but minions write it from start to finish.

Source: [Minions Part 2 - Stripe Dev Blog](https://stripe.dev/blog/minions-stripes-one-shot-end-to-end-coding-agents-part-2)

### What Blueprints Are

**[Confirmed]** Blueprints are Stripe's core orchestration pattern: flows that alternate between **fixed, deterministic code nodes** and **open-ended agent loops**. A typical blueprint:

1. Agent writes code (creative/agentic step)
2. Hardcoded script runs the linter (deterministic gate -- cannot be skipped)
3. If linter fails, agent fixes it (creative step)
4. Hardcoded script handles git commit (deterministic gate)

The key insight: "Putting LLMs into contained boxes compounds into system-wide reliability upside." Execution is deterministic in structure even though model output is probabilistic.

Source: [Stripe's coding agents: the walls matter more than the model](https://www.anup.io/stripes-coding-agents-the-walls-matter-more-than-the-model/)

### Structural Comparison

| Dimension | Goose Recipes (Native) | Stripe Blueprints (Built on Goose) |
|-----------|----------------------|-----------------------------------|
| **Control flow** | Fully agentic -- LLM interprets instructions | Hybrid -- deterministic nodes interleaved with agent loops |
| **Deterministic gates** | None built-in; all steps are LLM-driven | Hardcoded steps (linting, git, CI) that always execute |
| **Skip prevention** | Agent might skip steps based on judgment | Deterministic nodes cannot be bypassed |
| **Execution model** | Interactive session or one-shot recipe | One-shot "minion" execution in isolated devbox |
| **Environment** | Local machine or user's environment | Pre-warmed isolated "devboxes" (spin up in 10s, no internet) |
| **Validation** | Recipe YAML schema + Unicode security scan | Strict input/output contracts at every boundary |
| **Error handling** | LLM-driven retry via `RetryConfig` | Deterministic error detection + LLM-driven fix cycles |
| **Scope** | General-purpose agent workflows | Purpose-built for code migration/generation tasks |
| **Delegation** | `delegate` tool with sync/async modes | Blueprint nodes with fixed orchestration |
| **Parallelism** | Up to 5 async background tasks | Managed externally by Stripe's infrastructure |

### What Stripe Added on Top of Goose

**[Confirmed]** Stripe extended Goose in several important ways:

1. **Blueprint framework:** A coding framework that combines deterministic workflows with agent-like flexibility, going beyond the pure-agentic recipe model
2. **Isolated execution environments:** Pre-warmed devboxes identical to human engineer environments but isolated from production and internet
3. **Deep internal tooling:** Integration with Stripe's internal systems (CI, code review, deployment)
4. **Deterministic orchestration structure:** Wraps LLM calls in strict input/output contracts, catching drift at boundaries rather than mid-conversation

### What Goose Could Learn from Blueprints

**[Inferred]** Based on the architectural comparison:

1. **Deterministic nodes in recipes:** Goose recipes currently have no mechanism for mandatory, non-skippable steps. Adding deterministic gate support would let recipe authors enforce that certain validations always run.

2. **Input/output contracts between steps:** Blueprints validate at every boundary. Goose's `response.json_schema` provides structured output only at the final output level. There is no mechanism for inter-step validation or typed handoffs between subrecipes.

3. **Isolated execution:** Goose subagents share the host filesystem. Stripe's devbox isolation provides stronger safety guarantees for unattended execution.

4. **Failure modes:** Goose relies on LLM judgment for error recovery. Blueprints use deterministic detection (e.g., linter exit code) to trigger fix cycles, making failures more predictable and recoverable.

### The Summon Extension as Unification Layer

**[Confirmed]** The `summon` extension unifies the previously separate concepts of recipes, skills, agents, and subrecipes under a single discovery and delegation framework. Its module docstring reads: "Summon Extension - Unified tooling for recipes, skills, and subagents." It provides:

- **`load` tool:** Injects knowledge from any source type into the current context (no subagent spawning)
- **`delegate` tool:** Runs any source type as an isolated subagent (spawns a new agent)

This is architecturally distinct from Stripe's approach: Goose provides a flexible, general-purpose delegation mechanism where the LLM decides what to delegate and when, while Stripe built a rigid, purpose-specific orchestration layer where the control flow is predetermined and the LLM operates only within defined agentic nodes.

Source: `crates/goose/src/agents/platform_extensions/summon.rs`

---

## Summary of Confidence Levels

| Finding | Confidence | Basis |
|---------|-----------|-------|
| Recipe YAML schema and all fields | Confirmed | Source code (`recipe/mod.rs`) |
| SubRecipe struct with `sequential_when_repeated` | Confirmed | Source code |
| Subagent spawning via `run_subagent_task()` | Confirmed | Source code (`subagent_handler.rs`) |
| No recursive subagent spawning (two-layer check) | Confirmed | Source code (`summon.rs`) |
| Default 25 max turns | Confirmed | Source code (`subagent_task_config.rs`) |
| Max 5 concurrent background tasks | Confirmed | Source code (`summon.rs`) |
| 5-minute wait timeout for background tasks | Confirmed | Source code (`summon.rs`) |
| Subagent system prompt full content | Confirmed | Source code (`prompts/subagent_system.md`) |
| Summon extension dual-tool architecture | Confirmed | Source code (`summon.rs`) |
| Provider override priority chain | Confirmed | Source code (`summon.rs`) |
| Stripe Minions built on Goose fork | Confirmed | Stripe engineering blog |
| Blueprint = deterministic + agentic nodes | Confirmed | Stripe engineering blog |
| No inter-subagent communication | Inferred | Architecture (isolated sessions, no shared channels) |
| No conditional branching in recipe schema | Inferred | Schema analysis (no control flow fields in struct) |
| GitHub recipe loading details | Inferred | File existence only |

---

## Sources

### Primary (Source Code)
- [Goose GitHub Repository](https://github.com/block/goose)
  - `crates/goose/src/recipe/mod.rs` -- Recipe struct, SubRecipe struct, parsing, validation
  - `crates/goose/src/agents/subagent_handler.rs` -- Core subagent execution, prompt building
  - `crates/goose/src/agents/subagent_task_config.rs` -- TaskConfig, DEFAULT_SUBAGENT_MAX_TURNS
  - `crates/goose/src/agents/platform_extensions/summon.rs` -- Unified load/delegate tooling (2,184 lines)
  - `crates/goose/src/agents/subagent_execution_tool/notification_events.rs` -- Task status tracking
  - `crates/goose/src/prompts/subagent_system.md` -- Subagent system prompt template
  - `crates/goose/src/recipe/local_recipes.rs` -- Local recipe discovery
  - `crates/goose/src/prompt_template.rs` -- Template rendering with Jinja2
  - `crates/goose/src/agents/platform_tools.rs` -- Schedule management tool
- [recipe.yaml (repo root)](https://github.com/block/goose/blob/main/recipe.yaml)
- [workflow_recipes/release_risk_check/recipe.yaml](https://github.com/block/goose/blob/main/workflow_recipes/release_risk_check/recipe.yaml)
- [CONTRIBUTING_RECIPES.md](https://github.com/block/goose/blob/main/CONTRIBUTING_RECIPES.md)

### Documentation
- [Recipe Reference Guide](https://block.github.io/goose/docs/guides/recipes/recipe-reference/)
- [Sub-Recipes For Specialized Tasks](https://block.github.io/goose/docs/guides/recipes/sub-recipes/)
- [Running Subrecipes In Parallel](https://block.github.io/goose/docs/tutorials/subrecipes-in-parallel/)
- [Subagents Guide](https://block.github.io/goose/docs/guides/subagents/)
- [Recipes Overview](https://block.github.io/goose/docs/guides/recipes/)

### Community and Blog
- [How to Choose Between Subagents and Subrecipes](https://dev.to/blockopensource/how-to-choose-between-subagents-and-subrecipes-in-goose-553h)
- [Automate Your Complex Workflows with Sub-Recipes](https://dev.to/blockopensource/automate-your-complex-workflows-with-sub-recipes-in-goose-23bd)
- [A Recipe for Success: Cooking Up Repeatable Agentic Workflows](https://dev.to/goose_oss/a-recipe-for-success-cooking-up-repeatable-agentic-workflows-48m2)
- [Advent of AI - Day 11: Goose Subagents](https://www.nickyt.co/blog/advent-of-ai-day-11-goose-subagents-2n2/)
- [Advent of AI - Day 15: Goose Sub-Recipes](https://dev.to/nickytonline/advent-of-ai-2025-day-15-goose-sub-recipes-3mnd)
- [Goose Subagents Recipe Collection (Gist)](https://gist.github.com/mootrichard/8a2e4bf200e750f54bebfc78bbe4601f)
- [Creating and Sharing Effective Goose Recipes](https://medium.com/@shreyanshrewa/creating-and-sharing-effective-goose-recipes-abf9767d5128)
- [PulseMCP: A Human, A Goose, and Some Agents - Part IV](https://www.pulsemcp.com/building-agents-with-goose/part-4-configure-your-agent-with-goose-recipes)
- [Build Your Own Recipe Cookbook Generator](https://block.github.io/goose/blog/2025/10/08/recipe-cookbook-generator/)

### Stripe Blueprint Comparison
- [Minions: Stripe's one-shot, end-to-end coding agents (Part 1)](https://stripe.dev/blog/minions-stripes-one-shot-end-to-end-coding-agents)
- [Minions Part 2](https://stripe.dev/blog/minions-stripes-one-shot-end-to-end-coding-agents-part-2)
- [Stripe's coding agents: the walls matter more than the model](https://www.anup.io/stripes-coding-agents-the-walls-matter-more-than-the-model/)
- [Deconstructing Stripe's 'Minions': One-Shot Agents at Scale](https://www.sitepoint.com/stripe-minions-architecture-explained/)
- [DeepWiki: Retry and Validation / Subagents and Tasks](https://deepwiki.com/block/goose/4.1.4-subagents-and-tasks)
