# Goose Lineage: From Block's Open-Source Agent to Stripe's Minions

## 1. What Is Goose?

### Confirmed (source: GitHub repo, CUSTOM_DISTROS.md, source code)

Goose is an open-source, on-machine AI agent created by Block (parent company of Square, CashApp, and Tidal). It is designed for automating engineering tasks autonomously -- not just code suggestions, but building projects, writing and executing code, debugging, orchestrating workflows, and interacting with external APIs.

**Key facts:**
- **Language:** Rust (workspace of 8 crates: `goose`, `goose-cli`, `goose-server`, `goose-mcp`, `goose-acp`, `goose-acp-macros`, `goose-test`, `goose-test-support`)
- **License:** Apache 2.0
- **Repository:** https://github.com/block/goose
- **Stars:** ~32,500 (as of March 2026)
- **Forks:** ~2,979
- **Created:** August 23, 2024
- **Current version:** v1.27.2 (March 6, 2026)
- **Total releases:** 30 (earliest available: v1.14.0, November 12, 2025)

### Architecture

Goose follows a layered architecture (from CUSTOM_DISTROS.md):

```
User Interfaces (CLI / Desktop Electron / Custom UI)
        |
   goose-server (goosed) -- REST API
        |
   Core (goose crate)
     - Providers (AI models)
     - Extensions (MCP tools)
     - Config & Recipes
```

**Agent Loop:** The core agent loop lives in `crates/goose/src/agents/agent.rs`. The main entry point is the `reply()` method, which delegates to `reply_internal()`. The loop follows a standard ReAct pattern:
1. Prepare context (conversation history, tools, system prompt)
2. Send to LLM provider for completion
3. Parse tool calls from response
4. Categorize tools (frontend tools vs backend tools)
5. Check permissions (approve/deny/always-allow)
6. Execute approved tool calls via MCP
7. Feed results back to the LLM
8. Repeat until the LLM produces a final response or hits `DEFAULT_MAX_TURNS` (1,000)

**Context management:** Goose has conversation compaction built in (`check_if_compaction_needed`, `compact_messages`, `DEFAULT_COMPACTION_THRESHOLD`). An "Autocompact + One Shot Summarization algorithm" was added in July 2025 (commit from July 31, 2025).

**Tool/Extension System:** Goose uses MCP (Model Context Protocol) as its tool integration layer. Extensions are MCP servers that provide tools. The system supports:
- Dynamic extension loading/unloading at runtime
- Extension validation and malware checking (`extension_malware_check.rs`)
- Platform-specific built-in extensions
- A permission system (approve once, always allow, deny) with security inspection for prompt injection detection
- Environment variable security (31 disallowed environment variables to prevent hijacking)

**Subagent System:** Goose has a built-in subagent capability (`subagent_handler.rs`, `subagent_execution_tool/`, `subagent_task_config.rs`). Subagents are:
- Spawned by the main agent for specific tasks
- Independent but bounded (max turns, timeout)
- Cannot spawn additional subagents (security constraint)
- Have access to a subset of tools specified at spawn time
- Can be configured to return only the last message or full conversation

**Multi-Model Support (Lead-Worker Pattern):** Goose has a `LeadWorkerProvider` that switches between a frontier "lead" model for the first N turns (default: 3) and a cheaper "worker" model for subsequent turns. It includes automatic fallback to the lead model after consecutive worker failures.

**LLM Providers:** Goose supports an extensive list of providers:
- Anthropic, OpenAI, Google (Gemini), Azure, AWS Bedrock, GCP Vertex AI
- Ollama (local), local inference
- OpenRouter, LiteLLM, Databricks, Snowflake, SageMaker
- ChatGPT Codex, Claude Code, Gemini CLI, Cursor Agent, Codex
- Venice, xAI, GitHub Copilot, Tetrate
- Declarative provider system for easy addition of new providers

**Recipes:** YAML-based task definitions for guided workflows, including sub-recipes and subagent orchestration.

**Container Support:** Basic Docker container support (`container.rs`) for isolated execution.

**Custom Distributions:** Goose is explicitly designed for forking ("white labeling"). The CUSTOM_DISTROS.md document provides detailed guidance on creating organization-specific versions with preconfigured providers, bundled extensions, custom branding, and modified system prompts. This is a first-class architectural consideration, not an afterthought.

### Project History

- **August 2024:** Repository created. Initially a Python project (early commits reference `pytest`, Python-style tooling, `exchange` module).
- **September 2024:** v0.9.0 released. Early features include safety rails, goosehints (Jinja-templated), file tracking.
- **October-December 2024:** Rapid feature development. Cost calculation, synopsis mode, prompt rendering improvements.
- **January 2025:** Rust rewrite underway. v1.0.4 released January 31, 2025, indicating the 1.0 milestone with the Rust rewrite. The codebase shifted from Python to a Rust workspace with Cargo.toml.
- **November 2025:** v1.14.0 (earliest release still visible in the releases API).
- **March 2026:** v1.27.2, highly active with daily releases.

### Governance

Goose follows an open governance model with:
- **Contributors:** Community members (issues, PRs, discussions)
- **Maintainers:** Trusted members with write access to branches
- **Core Maintainers:** Admin access, set technical direction
- **Tiebreaker:** Bradley Axen (Goose creator) resolves deadlocks

Top contributors are all Block employees (Square email domains): zanesq, alexhancock, angiejones, michaelneale, dianed-square, blackgirlbytes, DOsinga, jamadeo.

---

## 2. When Did Stripe Fork Goose?

### Confirmed

The Stripe Minions blog posts (Part 1 and Part 2, published on stripe.dev) confirm that Minions is "built on Block's Goose agent (forked)." The exact fork date is not publicly disclosed.

### Inferred (with reasoning)

**Estimated fork period: Q2-Q3 2025 (April-September 2025)**

Reasoning:
1. The Stripe Minions blog posts were published in late 2025/early 2026 and describe a system already at scale (1,000+ PRs/week). Building to that scale requires months of development and rollout.
2. Goose hit v1.0 in January 2025 (Rust rewrite), making it a viable fork target from that point forward.
3. The Goose project was adding key features throughout Q1-Q2 2025 that would be relevant to Stripe's use case: MCP integration, provider abstraction, extension system maturation.
4. By July 2025, Goose was adding "Autocompact + One Shot Summarization" -- features that suggest the core agent loop was stable enough for production forking.
5. Stripe would likely fork after the Rust rewrite stabilized but before the project diverged too far from their needs.

**State of Goose at likely fork time:** The Rust rewrite was complete, MCP was the tool integration standard, the extension system was functional, multi-provider support existed, and the agent loop was stable. The subagent system and lead-worker provider may or may not have existed at fork time (these appear in more recent code).

### Unknown

- Exact fork date
- Exact Goose version/commit that was forked
- Whether Stripe forked incrementally or took a single snapshot

---

## 3. What Did Stripe Change in the Fork?

### Confirmed (source: existing research in 12_production_agent_deployments.md, Minions blog references)

Stripe's modifications to Goose include:
1. **Blueprint Pattern:** A workflow orchestration layer that combines "deterministic nodes" with "open-ended agent loops." This is not present in upstream Goose (Goose has recipes, which are simpler YAML-based task definitions).
2. **Toolshed MCP Server:** A custom MCP server providing 400+ tools specific to Stripe's internal infrastructure. Goose's extension system made this a natural integration point.
3. **Pre-warmed Devboxes:** Custom isolated execution environments with 10-second spin-up. Goose has basic Docker container support, but Stripe's devbox system is far more sophisticated.
4. **Slack Integration:** Invocation via Slack. Goose supports CLI and desktop app; Stripe replaced the user interface layer entirely.
5. **Production/Internet Isolation:** Network-level isolation for agent execution. Goose has no built-in network isolation.
6. **CI Integration:** Max 2 CI rounds per task, automated PR creation and management. Goose's core loop has no CI awareness.
7. **One-Shot Architecture:** Minions are described as "one-shot end-to-end coding agents," implying the agent loop was modified to optimize for single-pass task completion rather than interactive conversation.

### Inferred (with reasoning)

- **System prompt overhaul:** Stripe almost certainly replaced Goose's system prompt entirely. Goose's default prompt identifies itself as "goose, created by Block" and focuses on general-purpose engineering tasks. Stripe would need task-specific, Blueprint-aware prompts.
- **Provider lock-in:** Stripe likely locked the provider to a specific frontier model (probably Claude, given Stripe's relationship with Anthropic) rather than using Goose's multi-provider flexibility.
- **Telemetry replacement:** Goose uses PostHog for telemetry. Stripe would replace this with internal observability (the blog mentions monitoring and metrics).
- **Permission system modification:** Goose's interactive permission system (approve/deny per tool call) would not work for automated, one-shot execution. Stripe likely pre-authorized all tools or replaced the permission model entirely.
- **Subagent adaptation:** If the fork predates Goose's subagent system, Stripe may have built their own. If post-dates, they may have adapted it for the Blueprint pattern.
- **Extension manager simplification:** Goose's dynamic extension loading (add/remove at runtime, malware checking) would be unnecessary in a controlled CI environment. Stripe likely simplified this to a fixed set of Toolshed extensions.

### Unknown

- Exact diff between Stripe's fork and upstream Goose
- Whether Stripe contributes anything back upstream
- Whether Stripe tracks upstream Goose changes and merges them
- The extent of Rust-level code changes vs configuration-only changes

---

## 4. Is the Fork Public or Internal?

### Confirmed

**The fork is internal/private.** Evidence:
1. No `stripe/goose` repository exists on GitHub (404 response from GitHub API).
2. No Stripe fork appears in the public forks list of block/goose.
3. The Minions blog posts describe internal infrastructure (Toolshed, devboxes, Slack integration) that would not be open-sourced.
4. Goose's CUSTOM_DISTROS.md explicitly acknowledges that organizations may "maintain private forks" while noting the maintenance cost.

### Public Traces

The only public traces of Stripe's modifications are:
1. The two Stripe engineering blog posts on stripe.dev
2. The reference to "Block's Goose agent (forked)" in the existing research (12_production_agent_deployments.md)
3. Any conference talks by Stripe engineers (not yet identified)

### Unknown

- Whether Stripe has contributed any patches back to upstream Goose
- Whether any Stripe engineers are among the Goose contributors (none of the top 10 have obvious Stripe affiliations)
- Whether Stripe has published any patents related to their Goose modifications

---

## 5. Why Goose Specifically?

### Confirmed (source: Goose architecture, CUSTOM_DISTROS.md)

Goose has several properties that make it uniquely suitable as a base for enterprise forking:

1. **Apache 2.0 License:** Permits private forks, modification, and internal distribution without open-sourcing changes. No copyleft obligations.
2. **Explicit Fork-Friendliness:** Goose's CUSTOM_DISTROS.md is a detailed guide for creating organizational forks. This is not a feature of Claude Code, Codex CLI, or Gemini CLI. Goose was *designed* to be forked.
3. **MCP as Tool Protocol:** Goose adopted MCP early as its tool integration layer, making it straightforward to connect to internal tool ecosystems via custom MCP servers (Toolshed).
4. **Provider Agnosticism:** Goose supports 20+ LLM providers. Stripe could swap in any model without modifying the core agent loop.
5. **Rust Performance:** For CI/CD workloads processing 1,000+ tasks/week, Rust's performance characteristics (low latency, efficient concurrency) are advantageous over Python or TypeScript alternatives.
6. **Modular Architecture:** Clean separation between UI (CLI/Desktop), server (goosed), and core (goose crate) means Stripe could replace the UI layer (with Slack) without touching the agent loop.
7. **Extensible Agent Loop:** The agent loop supports configurable max turns, tool permission overrides, context compaction, and retry logic -- all knobs Stripe would need to tune for CI execution.

### Inferred (with reasoning)

- **Block-Stripe Relationship:** Both Block and Stripe are major payment infrastructure companies. While competitors, they operate in overlapping ecosystems and may have informal engineering relationships that facilitated awareness of Goose.
- **Timing:** Goose reached Rust v1.0 in January 2025, providing a stable base just as enterprise interest in agentic CI/CD was accelerating.
- **Alternatives were unsuitable:**
  - Claude Code: Anthropic-proprietary, not designed for forking, tied to Claude models
  - Codex CLI: OpenAI-proprietary, cloud sandbox model incompatible with Stripe's isolation requirements
  - Gemini CLI: Google-specific, less mature at the time
  - Aider: Python, no MCP support, no sandboxing
  - Building from scratch: Higher cost and time than forking a mature project

### Unknown

- Whether Stripe evaluated other agent frameworks before choosing Goose
- Whether there was any direct communication between Block and Stripe about the fork
- Whether Stripe's choice was influenced by specific Goose features or simply by availability and license

---

## 6. Goose vs Claude Code, Codex CLI, and Gemini CLI

### Architectural Comparison

| Dimension | Goose (Block) | Claude Code (Anthropic) | Codex CLI (OpenAI) | Gemini CLI (Google) |
|---|---|---|---|---|
| **Language** | Rust | TypeScript | Rust | TypeScript |
| **License** | Apache 2.0 | Proprietary (source-available) | Open source | Apache 2.0 |
| **Forkability** | Explicit support, CUSTOM_DISTROS guide | Not designed for forking | Forkable but no guidance | Forkable, no guidance |
| **Tool Protocol** | MCP (native) | MCP (native) | MCP (supported) | MCP (supported) |
| **Agent Loop** | ReAct with compaction, retry, subagents | Layered ReAct with hooks | ReAct with cloud sandbox | ReAct loop |
| **Multi-Model** | 20+ providers, lead-worker pattern | Claude only | OpenAI models only | Gemini only |
| **Sandboxing** | Docker containers (basic) | bubblewrap/Seatbelt (OS-level) | Landlock + cloud sandbox | Seatbelt (macOS) |
| **Multi-Agent** | Subagent system (built-in) | Agent Teams | Agents SDK | Single agent |
| **UI** | CLI + Desktop (Electron) | CLI (terminal) | CLI (terminal) | CLI (terminal) |
| **Context Management** | Autocompact + summarization | Compaction built-in | Cloud handles context | Session management |
| **Permissions** | Interactive approve/deny/always-allow + security inspection | Permission system with hooks | Sandbox-based (no interactive) | Basic approval |
| **Server Mode** | REST API (goosed) | SDK for programmatic use | JSONL output | JSON/stream output |

### Key Differentiators for Enterprise Forking

Goose is the only agent in this comparison that:
1. Has explicit documentation for creating custom distributions
2. Supports arbitrary LLM providers (critical for enterprise model flexibility)
3. Provides a REST API server mode (goosed) that decouples the agent from the UI
4. Is written in Rust with a clean crate architecture designed for embedding
5. Has an Apache 2.0 license with no copyleft restrictions

Claude Code, Codex CLI, and Gemini CLI are all designed as end-user tools for individual developers. Goose is designed as both an end-user tool and an embeddable agent platform.

---

## 7. Block's Current Relationship with Goose

### Confirmed (source: GitHub activity, governance doc)

**Goose is actively maintained and heavily invested in by Block.**

Evidence:
- **Release cadence:** v1.27.2 released March 6, 2026. Multiple releases per week.
- **Commit activity:** Active daily commits from Block employees.
- **Version velocity:** From v1.0 (January 2025) to v1.27 (March 2026) = 27 minor versions in 14 months.
- **Community:** 32,500+ stars, 2,979 forks, active Discord community.
- **Top contributors:** All 10 top contributors appear to be Block employees (Square email domains in commits).
- **Governance:** Formal governance structure with Core Maintainers, Maintainers, and Contributors. Tiebreaker is Bradley Axen (creator).
- **Features in active development:** TUI improvements, extension management, subagent system, new provider integrations, tree-sitter code analysis (Go, Java, JavaScript, Kotlin, Python, Ruby, Rust parsers in dependencies).
- **Topics:** Repository tagged with "mcp" topic, indicating alignment with the MCP ecosystem.

### Inferred

Block appears to be positioning Goose as a platform play rather than just an internal tool. The CUSTOM_DISTROS guide, the governance model, the community engagement (Discord, YouTube, LinkedIn, Twitter, Bluesky), and the Apache 2.0 license all suggest Block wants Goose to become the standard open-source agent platform that organizations fork and customize -- similar to how Kubernetes became the standard orchestration platform.

Stripe's use of Goose as the base for Minions validates this strategy and may be the most high-profile evidence that Goose's "fork-friendly" design is working as intended.

---

## Summary of Confidence Levels

| Question | Confidence | Key Gap |
|---|---|---|
| What is Goose? | **High** -- direct source code analysis | None |
| When did Stripe fork? | **Medium** -- inferred Q2-Q3 2025 | No public fork date |
| What did Stripe change? | **Medium** -- inferred from blog posts + architecture analysis | No access to Stripe's fork |
| Is the fork public? | **High** -- confirmed private | N/A |
| Why Goose? | **High** -- architectural analysis strongly supports | No direct Stripe statement on selection criteria |
| Goose vs competitors? | **High** -- direct comparison from source | Complete |
| Block's relationship? | **High** -- directly observable | None |

---

## Sources

1. **Goose GitHub Repository:** https://github.com/block/goose (README, source code, CUSTOM_DISTROS.md, GOVERNANCE.md, Cargo.toml)
2. **Goose Source Code:** `crates/goose/src/agents/agent.rs` (agent loop), `crates/goose/src/agents/extension.rs` (extension system), `crates/goose/src/providers/` (LLM providers), `crates/goose/src/prompts/system.md` (system prompt), `crates/goose/src/agents/subagent_handler.rs` (subagent system), `crates/goose/src/providers/lead_worker.rs` (multi-model pattern)
3. **Stripe Minions Part 1:** https://stripe.dev/blog/minions-stripes-one-shot-end-to-end-coding-agents
4. **Stripe Minions Part 2:** https://stripe.dev/blog/minions-stripes-one-shot-end-to-end-coding-agents-part-2
5. **Existing Research:** `/home/marco/repos/agentic_harness/research/12_production_agent_deployments.md` (Stripe Minions summary)
6. **Existing Research:** `/home/marco/repos/agentic_harness/research/10_agent_cli_tools_landscape.md` (agent CLI comparison)
7. **GitHub API:** Repository metadata, commit history, release history, contributor data, fork analysis
8. **Goose Documentation:** https://block.github.io/goose/
