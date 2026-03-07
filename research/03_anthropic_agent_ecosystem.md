# Anthropic's Agent Ecosystem (August 2025 -- March 2026)

## 1. Claude Model Releases Timeline

Anthropic released a rapid cadence of frontier models during this period:

| Model | Release Date | Notable Highlights |
|-------|-------------|-------------------|
| Claude Opus 4 | May 22, 2025 | 72.5% SWE-bench Verified; extended thinking with tool use |
| Claude Sonnet 4 | May 22, 2025 | Hybrid instant/extended thinking modes |
| Claude Opus 4.1 | August 5, 2025 | Incremental improvements |
| Claude Sonnet 4.5 | ~Q4 2025 | New default in Claude Code; OSWorld leader at 61.4%; context editing + memory tool |
| Claude Opus 4.5 | November 24, 2025 | Leads 7/8 languages on SWE-bench Multilingual; +10.6% on Aider Polyglot vs Sonnet 4.5 |
| Claude Haiku 4.5 | ~Q4 2025 | Fastest model, near-frontier performance |
| Claude Opus 4.6 | February 5, 2026 | ~81% SWE-bench Verified; highest Terminal-Bench 2.0 score |
| Claude Sonnet 4.6 | February 17, 2026 | Balanced speed/intelligence for everyday tasks |

---

## 2. Claude Code CLI

### 2.1 Architecture Overview

Claude Code is an agentic coding tool that lives in the terminal. It understands codebases and executes tasks through natural language commands. Core architectural elements:

- **CLAUDE.md files**: Markdown files in the project root that define coding standards, architecture decisions, preferred libraries, and review checklists. Claude reads these to align behavior with project conventions.
- **Agent loop**: The fundamental cycle of gathering context, taking action, and verifying work. Claude Code orchestrates tool calls (file reads/writes, shell commands, search) in a loop until the task is complete.
- **JSONL message storage**: Sessions are persisted as JSONL files, enabling resumption and inspection.

### 2.2 Session Management

- Sessions maintain full conversation context across multiple interactions.
- **Resume**: `claude --resume <session-id>` creates a new JSONL file and streams only new messages.
- **Cross-surface sessions**: Sessions are not tied to a single environment. Users can move between surfaces:
  - `/teleport` pulls a web/iOS session into the terminal.
  - `/desktop` hands off a terminal session to the Desktop app for visual diff review.
  - Remote Control enables continuing work from a phone or browser.

### 2.3 Streaming Output

Claude Code streams responses in real-time in the terminal. The `--print` and `--verbose` flags control output granularity. The SDK exposes streaming events for programmatic consumers.

### 2.4 Version 2.0 and Beyond

Key features shipped in this period:

- **Checkpoints**: Automatically save code state before each change. Rewind with Esc-Esc or `/rewind`. Restores code, conversation, or both.
- **Subagents**: Spawn multiple Claude Code agents working on different parts of a task simultaneously, with a lead agent coordinating, assigning subtasks, and merging results. Subagents use isolated context windows and only send relevant information back to the orchestrator.
- **Worktree isolation**: Git worktrees give each subagent its own working directory. Automatic cleanup when a subagent finishes without changes.
- **Hooks**: Automatically trigger actions at specific points in the agent lifecycle.
- **Background tasks**: Long-running processes stay active without blocking progress on other work.
- **Plugins**: MCP servers can be installed directly with the `/plugin` command.
- **Native VS Code extension**: First-class IDE integration.
- **Claude Code on the web**: Browser-based access.

### 2.5 Sandboxing

The sandboxing architecture isolates code execution with filesystem and network controls:

- Built on OS-level primitives: Linux bubblewrap, macOS seatbelt.
- Automatically allows safe operations, blocks malicious ones, asks permission only when needed.
- In internal usage, sandboxing reduces permission prompts by 84%.
- Every task runs in an isolated sandbox with network and filesystem restrictions.
- Git interactions use a secure proxy that ensures Claude can only access authorized repositories.

### 2.6 Autonomy Metrics

The 99.9th percentile turn duration nearly doubled between October 2025 and January 2026, growing from under 25 minutes to over 45 minutes -- indicating increasingly sophisticated and autonomous multi-agent workflows.

---

## 3. Claude Agent SDK

### 3.1 Origin and Rename

The Claude Code SDK was renamed to the **Claude Agent SDK** in September 2025, reflecting its capabilities for building AI agents across all domains, not just coding. Available in both Python (`claude-agent-sdk-python`) and TypeScript (`claude-agent-sdk-typescript`).

### 3.2 Core Capabilities

The SDK provides the same tools, agent loop, and context management that power Claude Code, programmable for custom workflows:

- **Context compaction**: Automatically summarizes previous messages when the context limit approaches (built on Claude Code's `/compact` command). Recent updates improved memory usage in long sessions with subagents by stripping heavy progress message payloads during compaction.
- **Subagent support**: SDK-level support for spawning and coordinating subagents.
- **Hooks**: Lifecycle event hooks for custom automation.
- **Structured outputs**: Agents can return validated JSON matching a schema.
- **Beta features**: `betas` option in `ClaudeAgentOptions` for enabling API betas (e.g., extended context windows).
- **Claude Code bundled**: Claude Code is now included by default in the SDK package.

### 3.3 Long-Running Agent Harnesses

Anthropic published guidance on effective harnesses for long-running agents that span hours or days across multiple context windows:

- **Two-agent architecture**: An initializer agent sets up the environment on the first run; a coding agent makes incremental progress in every subsequent session.
- **State artifacts**: A `claude-progress.txt` file alongside the git history lets agents quickly understand work state when starting with a fresh context window.
- **Different prompts per stage**: The first context window gets a different prompt than continuation windows.

---

## 4. Model Context Protocol (MCP)

### 4.1 Scale and Adoption

By the one-year anniversary (November 2025), MCP had become one of the fastest-growing open-source projects in AI:

- Over 97 million monthly SDK downloads.
- 10,000+ active servers.
- First-class client support across ChatGPT, Claude, Cursor, Gemini, Microsoft Copilot, and VS Code.

### 4.2 Major Milestones

- **MCP Registry** (September 2025): Launched in preview; grew to nearly 2,000 entries (407% growth from initial onboarding). Progressing toward general availability.
- **MCP Apps** (January 2026): Official extension allowing tools to return interactive UI components (dashboards, forms, visualizations, multi-step workflows) that render directly in the conversation.
- **Donation to Agentic AI Foundation** (December 2025): Anthropic donated MCP to the Agentic AI Foundation (AAIF), a directed fund under the Linux Foundation. Co-founded by Anthropic, Block, and OpenAI, with support from Google, Microsoft, AWS, Cloudflare, and Bloomberg.

### 4.3 MCP in Claude Code

- Claude Code accesses both tools and resources exposed by MCP servers.
- MCP servers can run as local processes, connect over HTTP, or execute within an SDK application.
- Configuration scopes: local (project-specific, private), user-level, and global.
- Example integrations: Sentry (error debugging), Linear (project management).
- Desktop Extensions: One-click install from a curated directory in Claude Desktop (zip archive with local MCP server + manifest.json).

### 4.4 MCP on the Anthropic API

- **MCP Connector**: Connect Claude to any remote MCP server without writing client code. The API handles connection management, tool discovery, and error handling automatically.
- **Tool Search**: Access thousands of tools without consuming the context window.
- **Programmatic Tool Calling**: Claude writes code that calls multiple tools, processes outputs, and controls what enters its context window -- reducing round-trips.

### 4.5 2026 Roadmap

Active SEPs (Specification Enhancement Proposals):
- DPoP extension for authentication.
- Multi-turn SSE for transport.
- Server Cards for discovery.
- Focus on enterprise hardening, security/auth patterns, and scaling guidance.

---

## 5. Extended Thinking

### 5.1 Core Capability

Extended thinking allows Claude to reason more deeply before responding. First introduced with Claude 3.7 Sonnet, it has been refined across the Claude 4.x family.

- **Thinking budget**: Developers can set how much computation Claude allocates to reasoning via a budget parameter.
- **Hybrid modes**: Claude Opus 4 and Sonnet 4 offer both near-instant responses and extended thinking modes.
- **Interleaved thinking**: Claude 4 models support thinking between tool calls -- Claude can reason after receiving tool results before making the next call.
- **Tool use during thinking**: Claude can invoke tools (like web search) during extended thinking, alternating between reasoning and information gathering.

### 5.2 Practical Impact

Extended thinking with tool use enables agents that work autonomously for hours on complex tasks (versus minutes), combining deep reasoning with parallel tool execution and MCP connectivity.

---

## 6. Agent Evaluation Approaches

### 6.1 SWE-bench Verified Results

SWE-bench evaluates end-to-end agent systems (model + scaffolding) on real GitHub issues from popular open-source Python repositories. SWE-bench Verified is a 500-problem human-reviewed subset.

| Model | SWE-bench Verified Score |
|-------|------------------------|
| Claude 3.5 Sonnet (updated) | 49.0% |
| Claude Opus 4 | 72.5% |
| Claude Opus 4.6 | ~81% (81.42% with prompt modification) |

Additional coding benchmarks:
- **OSWorld** (real-world computer tasks): Sonnet 4.5 leads at 61.4%.
- **Aider Polyglot**: Opus 4.5 +10.6% over Sonnet 4.5.
- **SWE-bench Multilingual**: Opus 4.5 leads across 7/8 programming languages.
- **Terminal-Bench 2.0**: Opus 4.6 holds the highest score.

### 6.2 Bloom: Automated Behavioral Evaluations

Bloom is an open-source agentic framework for generating behavioral evaluations of frontier AI models without ground-truth labels. Key characteristics:

- **Pipeline**: Understanding agent analyzes behavior descriptions and generates detailed context; ideation agent creates evaluation scenarios; scenarios are rolled out in parallel with simulated users and tool responses.
- **Benchmark behaviors**: Delusional sycophancy, instructed long-horizon sabotage, self-preservation, self-preferential bias -- measured across 16 frontier models.
- **Validation**: Evaluations correlate strongly with hand-labeled judgments and reliably separate baseline models from intentionally misaligned ones.

### 6.3 Petri: Automated Behavioral Auditing

As of January 2026, Anthropic's Petri auditing tool has been upgraded with:
- Improved realism mitigations to counter eval-awareness.
- Expanded seed library with 70 new scenarios.
- Evaluation results for more recent frontier models.

### 6.4 Context Engineering Evaluations

Anthropic validated their context management tools with rigorous evaluations:
- In a 100-turn web search evaluation, context editing enabled agents to complete workflows that would otherwise fail due to context exhaustion, while reducing token consumption by 84%.

---

## 7. Additional Agent Infrastructure

### 7.1 Agent Skills (October 2025)

Agent Skills are organized folders of instructions, scripts, and resources that Claude loads dynamically to perform specialized tasks. Supported across Claude.ai, Claude Code, the Claude Agent SDK, and the Claude Developer Platform.

### 7.2 Context Management on the Developer Platform

Two new capabilities for managing long-running agents:

- **Context editing**: Automatically clears stale tool calls and results from within the context window when approaching token limits, preserving conversation flow.
- **Memory tool**: File-based system allowing Claude to store and consult information outside the context window, persisting across conversations. Agents can build knowledge bases over time and maintain project state across sessions.

### 7.3 Files API and Prompt Caching

Released as part of the Agent Capabilities API:
- **Files API**: Store and access files efficiently across sessions.
- **Extended prompt caching**: Cache prompts for up to one hour, enabling cost-effective context maintenance.

---

## 8. Key Takeaways

1. **SWE-bench performance nearly doubled** in under a year: from 49% (Claude 3.5 Sonnet) to ~81% (Claude Opus 4.6), demonstrating rapid gains in autonomous coding ability.

2. **MCP has become an industry standard**, with 97M+ monthly SDK downloads and governance transferred to a Linux Foundation entity co-founded with OpenAI, Google, Microsoft, and others.

3. **The Agent SDK generalizes Claude Code's infrastructure** beyond coding. The rename from "Code SDK" to "Agent SDK" signals Anthropic's bet that the agentic loop (context gathering, action, verification) applies across all domains.

4. **Context engineering has emerged as the central discipline** for building effective agents, superseding prompt engineering. Anthropic's tools (compaction, context editing, memory tool) address the fundamental challenge of maintaining coherent state across long-running, multi-context-window tasks.

5. **Multi-agent architectures are production-ready**: Subagents with worktree isolation, hooks, and background tasks enable 45+ minute autonomous sessions at the 99.9th percentile.

6. **Extended thinking with interleaved tool use** is a differentiating capability -- Claude can reason between tool calls, enabling deeper analysis during complex agentic workflows.

7. **Evaluation methodology is maturing**: Bloom and Petri provide automated behavioral auditing without ground-truth labels, enabling rapid assessment of alignment-relevant properties across frontier models.

---

## Sources

- [Claude Code overview](https://docs.anthropic.com/en/docs/agents-and-tools/claude-code/overview)
- [Claude Code GitHub repository](https://github.com/anthropics/claude-code)
- [Making Claude Code more secure and autonomous (Sandboxing)](https://www.anthropic.com/engineering/claude-code-sandboxing)
- [Enabling Claude Code to work more autonomously](https://www.anthropic.com/news/enabling-claude-code-to-work-more-autonomously)
- [Building agents with the Claude Agent SDK](https://www.anthropic.com/engineering/building-agents-with-the-claude-agent-sdk)
- [Claude Agent SDK Python (GitHub)](https://github.com/anthropics/claude-agent-sdk-python)
- [Claude Agent SDK TypeScript (GitHub)](https://github.com/anthropics/claude-agent-sdk-typescript)
- [Agent SDK overview (Docs)](https://docs.anthropic.com/en/docs/claude-code/sdk)
- [Effective harnesses for long-running agents](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents)
- [Effective context engineering for AI agents](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)
- [One Year of MCP: November 2025 Spec Release](http://blog.modelcontextprotocol.io/posts/2025-11-25-first-mcp-anniversary/)
- [MCP joins the Agentic AI Foundation](http://blog.modelcontextprotocol.io/posts/2025-12-09-mcp-joins-agentic-ai-foundation/)
- [Donating MCP and establishing the Agentic AI Foundation](https://www.anthropic.com/news/donating-the-model-context-protocol-and-establishing-of-the-agentic-ai-foundation)
- [MCP Registry preview](http://blog.modelcontextprotocol.io/posts/2025-09-08-mcp-registry-preview/)
- [MCP Apps](http://blog.modelcontextprotocol.io/posts/2026-01-26-mcp-apps/)
- [January 2026 MCP Core Maintainer Update](https://blog.modelcontextprotocol.io/posts/2026-01-22-core-maintainer-update/)
- [MCP Roadmap](https://modelcontextprotocol.io/development/roadmap)
- [Connect Claude Code to tools via MCP](https://docs.anthropic.com/en/docs/claude-code/mcp)
- [Remote MCP support in Claude Code](https://www.anthropic.com/news/claude-code-remote-mcp)
- [Code execution with MCP](https://www.anthropic.com/engineering/code-execution-with-mcp)
- [Claude's extended thinking](https://www.anthropic.com/news/visible-extended-thinking)
- [Building with extended thinking (Docs)](https://docs.anthropic.com/en/docs/build-with-claude/extended-thinking)
- [Introducing Claude 4](https://www.anthropic.com/news/claude-4)
- [Introducing Claude Opus 4.5](https://www.anthropic.com/news/claude-opus-4-5)
- [Introducing Claude Sonnet 4.5](https://www.anthropic.com/news/claude-sonnet-4-5)
- [Introducing Claude Sonnet 4.6](https://www.anthropic.com/news/claude-sonnet-4-6)
- [Introducing Claude Opus 4.6](https://www.anthropic.com/news/claude-opus-4-6)
- [Claude Opus 4.1](https://www.anthropic.com/news/claude-opus-4-1)
- [Claude SWE-Bench Performance](https://www.anthropic.com/research/swe-bench-sonnet)
- [Bloom: automated behavioral evaluations](https://alignment.anthropic.com/2025/bloom-auto-evals/)
- [Building and evaluating alignment auditing agents](https://alignment.anthropic.com/2025/automated-auditing/)
- [New capabilities for building agents on the Anthropic API](https://www.anthropic.com/news/agent-capabilities-api)
- [Introducing Agent Skills](https://www.anthropic.com/news/skills)
- [Equipping agents for the real world with Agent Skills](https://www.anthropic.com/engineering/equipping-agents-for-the-real-world-with-agent-skills)
- [Managing context on the Claude Developer Platform](https://www.anthropic.com/news/context-management)
- [Claude Sonnet 4 now supports 1M tokens of context](https://www.anthropic.com/news/1m-context)
- [Create custom subagents (Docs)](https://docs.anthropic.com/en/docs/claude-code/sub-agents)
- [Session Management (SDK Docs)](https://docs.anthropic.com/en/docs/claude-code/sdk/sdk-sessions)
- [Claude Code Releases (GitHub)](https://github.com/anthropics/claude-code/releases)
- [Claude Sonnet 4.6 System Card](https://anthropic.com/claude-sonnet-4-6-system-card)
- [2026 Agentic Coding Trends Report (PDF)](https://resources.anthropic.com/hubfs/2026%20Agentic%20Coding%20Trends%20Report.pdf)
- [Claude Code plugins](https://www.anthropic.com/news/claude-code-plugins)
- [Desktop Extensions](https://www.anthropic.com/engineering/desktop-extensions)
- [Migration guide to Claude 4](https://docs.anthropic.com/en/docs/about-claude/models/migrating-to-claude-4)
