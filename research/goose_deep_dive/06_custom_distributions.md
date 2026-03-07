# Goose: Custom Distributions & Forking

> Research document covering Block's Goose agent framework customization, forking, and distribution capabilities.
> Date: 2026-03-07

---

## 1. CUSTOM_DISTROS.md Guide -- Official Forking Guidance

**Status: Confirmed** (source: [CUSTOM_DISTROS.md](https://github.com/block/goose/blob/main/CUSTOM_DISTROS.md))

Goose maintains a comprehensive `CUSTOM_DISTROS.md` at the repo root, subtitled as a guide for "white labelling -- creating a branded or tailored version of an open source project for your organization." The document is substantial (covering 9 distinct distribution scenarios labeled A through I) and serves as the canonical reference for anyone building a custom Goose distribution.

### What the guide recommends

The guide outlines three tiers of customization strategy, in increasing order of complexity:

1. **Configuration-only** -- Modify config files (`config.yaml`, `init-config.yaml`) and environment variables. No code changes required. Covers provider selection, model defaults, and API keys.
2. **Extension-based** -- Add custom MCP servers for proprietary tools. Minimal core changes. The primary approach for adding internal tool integrations.
3. **Deep customization** -- Modify core behavior, UI components, or add new providers. Requires Rust/TypeScript development.

### Fork and clone approach

The guide explicitly recommends forking:

```bash
git clone https://github.com/YOUR_ORG/goose.git
cd goose
```

### Fork maintenance philosophy

The guide is candid about the cost of divergence:

> "While you're free to maintain private forks, contributing improvements upstream benefits everyone -- including your distribution. Private forks that diverge significantly become expensive to maintain and miss out on security updates and new features. Consider upstreaming generic improvements while keeping only organization-specific customizations private."

### Licensing requirements (Apache 2.0)

Custom distributions must:
- Include original license and copyright notices
- Clearly indicate any modifications made
- Not use "Goose" trademarks in ways that imply official endorsement

### Nine documented distribution scenarios

| Scenario | Description | Complexity |
|----------|-------------|------------|
| A. Local Model | Preconfigured Ollama, no API keys | Low |
| B. Corporate Managed Keys | Pre-provisioned API keys for frontier models | Low-Medium |
| C. Custom MCP Extensions | Internal data source connectors | Medium |
| D. Custom Branding/UI | Rebrand desktop app with org identity | Medium |
| E. New Interface | Build web/mobile UI on goose-server REST API or ACP | High |
| F. Audience-Specific | Tailored for legal, design, etc. via recipes | Low-Medium |
| G. Custom AI Provider | Add new provider via declarative JSON or Rust code | Low-Medium |
| H. Workflow Recipes | YAML-based standardized workflows | Low |
| I. Complex Workflows | Sub-recipes and subagents for multi-step orchestration | Medium |

---

## 2. Customization Surface

**Status: Confirmed** (source: [CUSTOM_DISTROS.md](https://github.com/block/goose/blob/main/CUSTOM_DISTROS.md))

### What can be customized via configuration alone (no code changes)

| Customization | Mechanism | Location |
|---------------|-----------|----------|
| AI provider & model | `init-config.yaml`, env vars (`GOOSE_PROVIDER`, `GOOSE_MODEL`) | Distribution root / shell env |
| API keys/secrets | `goose configure set-secret`, keyring, or `secrets.yaml` | `~/.config/goose/` |
| Custom AI providers | Declarative JSON files | `~/.config/goose/custom_providers/` |
| Workflow templates | Recipe YAML files | Distributable files |
| Extension bundles | JSON config in extension catalogs | Config files |
| Telemetry toggle | `GOOSE_DISABLE_TELEMETRY=1` | Environment variable |

**Declarative provider example** (no Rust code needed):
```json
{
  "name": "my_provider",
  "engine": "openai",
  "display_name": "My Custom Provider",
  "base_url": "https://llm.internal.company.com/v1/chat/completions",
  "api_key_env": "MY_PROVIDER_API_KEY",
  "models": [{"name": "company-llm-v1", "context_limit": 32768}],
  "supports_streaming": true
}
```
Supported engines: `openai`, `anthropic`, `ollama`.

### What requires code changes

| Customization | Language | Location |
|---------------|----------|----------|
| System prompts | Markdown | `crates/goose/src/prompts/system.md` |
| Desktop branding (icons, names) | Config + assets | `ui/desktop/` (icons, `forge.config.ts`, `package.json`) |
| UI components (colors, labels, features) | TypeScript/React | `ui/desktop/src/` |
| Custom provider with unique API | Rust | `crates/goose/src/providers/` (implement `Provider` trait) |
| Core agent behavior | Rust | `crates/goose/src/agents/` |
| Telemetry redirection | Rust | `crates/goose/src/posthog.rs` |
| Extension loading logic | Rust | `crates/goose/src/agents/extension_manager.rs` |

### Config precedence

Environment variables take precedence over `config.yaml`, which takes precedence over defaults. This allows distributions to set defaults that users can still override.

---

## 3. Architecture Layers for Customization

**Status: Confirmed** (source: [CUSTOM_DISTROS.md](https://github.com/block/goose/blob/main/CUSTOM_DISTROS.md), [repo structure](https://github.com/block/goose))

Goose uses a three-layer architecture explicitly designed for UI replacement:

```
+-------------------------------------------------------------------+
|                        User Interfaces                            |
|  +--------------+  +--------------+  +--------------------------+ |
|  |  CLI         |  |  Desktop     |  |  Your Custom UI          | |
|  |  (goose-cli) |  |  (Electron)  |  |  (web, mobile, etc.)    | |
|  +------+-------+  +------+-------+  +------------+-------------+ |
+---------|-----------------|-----------------------|---------------+
          |                 |                       |
          v                 v                       v
+-------------------------------------------------------------------+
|                    goose-server (goosed)                           |
|         REST API for all goose functionality                      |
+-------------------------------------------------------------------+
          |
          v
+-------------------------------------------------------------------+
|                      Core (goose crate)                           |
|  +--------------+  +--------------+  +--------------------------+ |
|  |  Providers   |  |  Extensions  |  |  Config & Recipes        | |
|  |  (AI models) |  |  (MCP tools) |  |  (behavior & defaults)  | |
|  +--------------+  +--------------+  +--------------------------+ |
+-------------------------------------------------------------------+
```

### Crate structure

The Rust workspace contains specialized crates:

| Crate | Purpose |
|-------|---------|
| `goose` | Core agent logic, providers, extensions, config, recipes |
| `goose-cli` | CLI entry point (`goose` binary) |
| `goose-server` | REST API backend (`goosed` binary) |
| `goose-acp` | Agent Client Protocol implementation |
| `goose-acp-macros` | ACP procedural macros |
| `goose-mcp` | Built-in MCP server implementations |
| `goose-test` / `goose-test-support` | Test utilities |

### How this enables UI replacement

The critical architectural decision is that **all agent functionality is accessible through `goose-server` (goosed)**, meaning any UI -- CLI, desktop Electron app, web app, mobile app, or custom integration -- can drive the same agent loop without modifying the core. The REST API exposes:

- `POST /sessions` -- Create a new session
- `POST /sessions/{id}/messages` -- Send a message (SSE streaming response)
- `GET /sessions/{id}` -- Get session state
- `GET /extensions` -- List available extensions
- `POST /extensions/{name}/enable` -- Enable an extension

An OpenAPI spec is maintained at `ui/desktop/openapi.json`.

### ACP as the emerging protocol layer

Goose is transitioning toward the Agent Client Protocol (ACP) -- a JSON-RPC bidirectional protocol over stdio or other transports. ACP provides richer capabilities than the REST API:

- Bidirectional communication (agents can request permissions, stream updates)
- Rich tool call handling with status updates
- Session management with conversation history
- Dynamic MCP server integration

Key ACP methods include `initialize`, `session/new`, `session/load`, `session/prompt`, and `session/cancel`. Notifications flow from agent to client via `session/notification` with event types like `agentMessageChunk`, `toolCall`, `toolCallUpdate`, and `requestPermission`.

**Status: Confirmed** -- There is active work (issue [#6642](https://github.com/block/goose/issues/6642)) to bring ACP over HTTP, which would eventually allow ACP to fully replace the goosed REST API.

---

## 4. Known Custom Distributions

### Stripe Minions

**Status: Confirmed** (source: [Stripe Dev Blog](https://stripe.dev/blog/minions-stripes-one-shot-end-to-end-coding-agents))

Stripe Minions is the most prominent and publicly documented custom distribution of Goose. Key details:

- **Scale**: Merging 1,000+ pull requests per week, all code written autonomously and reviewed by humans.
- **Fork basis**: Runs on a fork of Block's open-source Goose agent, customized to "interleave agent loops with deterministic code for git operations, linters, and testing."
- **Optimization target**: Unlike Goose's interactive human-supervised model, Minions is optimized for fully autonomous, unattended operation ("one-shot, end-to-end").
- **MCP integration**: Connects to "Toolshed," an internal MCP server hosting 400+ tools spanning internal systems and SaaS platforms. The orchestrator performs deterministic prefetching to curate a subset of ~15 relevant tools per task (not all 400 exposed at once).
- **Execution environment**: Isolated pre-warmed devboxes that spin up in 10 seconds with Stripe code, Bazel caches, and services pre-loaded. Treated as "cattle, not pets."
- **Agent rules**: Reads the same coding agent rule files (AGENTS.md, etc.) that human tools like Cursor and Claude Code use. Rules are conditionally applied based on subdirectories.
- **Invocation surfaces**: Slack, CLI, web interface -- all producing PRs.
- **CI integration**: Maximum 2 CI rounds with auto-fix capability.

Stripe Minions demonstrates a key pattern for custom distributions: keep the core agent loop mostly intact, customize the orchestration layer around it (devbox provisioning, tool selection, CI integration, invocation surfaces), and connect to proprietary infrastructure via MCP.

### Other known distributions

**Status: Inferred / Unknown**

- **Block internal**: Block itself almost certainly runs a customized internal distribution, given they created the framework. No public details beyond the open-source repo.
- **Public forks**: The repo has **2,984 forks** on GitHub (as of 2026-03-07) with **32,559 stars**. While many forks are personal experiments, the high fork count suggests organizational adoption. However, no other company has publicly documented a custom distribution at the level Stripe has.
- **AAIF member companies**: With Goose contributed to the Linux Foundation's Agentic AI Foundation (December 2025), founding members (AWS, Anthropic, Google, Microsoft, OpenAI, Bloomberg, Cloudflare) may be experimenting with custom distributions, but none have been publicly confirmed.

---

## 5. Fork Maintenance Considerations

**Status: Confirmed** (source: [CUSTOM_DISTROS.md](https://github.com/block/goose/blob/main/CUSTOM_DISTROS.md))

### Official guidance

The CUSTOM_DISTROS.md provides a four-point strategy for staying current:

1. **Regularly sync your fork** with the main repository
2. **Keep customizations isolated** (config files, separate extension repos) when possible
3. **Use recipes for workflow customization** rather than code changes
4. **Subscribe to release announcements** for breaking changes

### Minimizing divergence

The guide implicitly recommends a layered approach to minimize merge conflicts:

- **Prefer config over code**: Use `init-config.yaml`, environment variables, and declarative provider JSON rather than modifying Rust source.
- **Prefer extensions over core changes**: Build custom functionality as MCP servers rather than modifying the agent loop.
- **Prefer recipes over UI changes**: Use YAML recipes to define workflows rather than modifying UI components.
- **Upstream generic improvements**: Keep only organization-specific customizations in the private fork.

### Breaking change risk areas

**Status: Inferred** (based on architecture analysis)

The areas most likely to cause merge conflicts in forks:

| Risk Level | Area | Reason |
|------------|------|--------|
| High | `crates/goose/src/agents/` | Core agent loop, frequently evolving |
| High | `ui/desktop/` | Electron app UI, branding changes live here |
| Medium | `crates/goose/src/providers/` | Provider interface changes |
| Medium | `crates/goose-server/src/routes/` | API changes as ACP transition proceeds |
| Low | `crates/goose/src/prompts/` | System prompts, relatively stable |
| Low | Recipe YAML files | Standalone files, unlikely to conflict |
| Low | Declarative provider JSON | Additive, separate files |

### Release cadence

The repo has had **123+ releases**, indicating a rapid release cycle. Forks need active maintenance to stay current. The ACP-over-HTTP transition ([issue #6642](https://github.com/block/goose/issues/6642)) is a notable upcoming change that could affect custom integrations built on the current REST API.

---

## 6. White-Labeling Capabilities

**Status: Confirmed** (source: [CUSTOM_DISTROS.md](https://github.com/block/goose/blob/main/CUSTOM_DISTROS.md))

Goose provides explicit, documented support for white-labeling across multiple dimensions:

### Visual branding

- **Application icons**: Replace `icon.png`, `icon.ico`, `icon.icns` in `ui/desktop/src/images/`
- **Splash screens and logos**: Update as needed in the same directory
- **Color schemes**: Modify CSS/Tailwind configuration in `ui/desktop/src/`
- **Component text and labels**: Customize React/TypeScript components

### Application metadata

Modify `ui/desktop/forge.config.ts`:
```typescript
module.exports = {
  packagerConfig: {
    name: 'YourCompany AI Assistant',
    executableName: 'yourcompany-ai',
    icon: 'src/images/your-icon',
  },
};
```

### Build and release identity

Environment variables control distribution naming:
```bash
export GITHUB_OWNER="your-org"
export GITHUB_REPO="your-goose-fork"
export GOOSE_BUNDLE_NAME="InsightStream-goose"
```

Additional branding touchpoints:
- `ui/desktop/package.json` (`productName`, description)
- Linux desktop templates (`forge.deb.desktop`, `forge.rpm.desktop`)
- `ui/desktop/index.html`

### System prompt identity

Modify `crates/goose/src/prompts/system.md`:
```markdown
You are an AI assistant called [YourName], created by [YourCompany].
```

### Telemetry replacement

Two options:
1. **Disable entirely**: `GOOSE_DISABLE_TELEMETRY=1`
2. **Redirect to own instance**: Modify `crates/goose/src/posthog.rs` to point to your PostHog instance

### Branding consistency checklist (from the docs)

Before release, verify:
- Application metadata (`forge.config.ts`, `package.json`, `index.html`) uses your distro name
- Release artifact names and updater lookup names are consistent
- Desktop launchers (Linux `.desktop` templates) point to the correct executable name

---

## 7. Embedding Goose

### As a REST API service

**Status: Confirmed** (source: [CUSTOM_DISTROS.md](https://github.com/block/goose/blob/main/CUSTOM_DISTROS.md))

The `goosed` binary serves as a standalone REST API server that any application can integrate with:

```bash
./target/release/goosed
# API available at http://localhost:3000
```

Key API endpoints:
- `POST /sessions` -- Create session
- `POST /sessions/{id}/messages` -- Send message (SSE streaming)
- `GET /sessions/{id}` -- Get session state
- `GET /extensions` -- List extensions
- `POST /extensions/{name}/enable` -- Enable extension

Responses use Server-Sent Events (SSE) for real-time streaming. An OpenAPI spec at `ui/desktop/openapi.json` enables client generation in any language.

### As an ACP agent (stdio-based embedding)

**Status: Confirmed** (source: [CUSTOM_DISTROS.md](https://github.com/block/goose/blob/main/CUSTOM_DISTROS.md))

For tighter embedding (IDEs, desktop apps, embedded agents), Goose can run as an ACP server on stdio:

```bash
goose acp --with-builtin developer,memory
```

This enables JSON-RPC bidirectional communication with methods like `initialize`, `session/new`, `session/load`, `session/prompt`, `session/cancel`. A Python example client is provided in the docs and a reference implementation exists as `test_acp_client.py`.

ACP notifications from agent to client include:
- `session/notification` with `agentMessageChunk` -- Streaming text responses
- `session/notification` with `toolCall` -- Tool invocation started
- `session/notification` with `toolCallUpdate` -- Tool status/result updates
- `requestPermission` -- Agent requests user confirmation for sensitive operations

### As a Rust library/crate

**Status: Inferred** (based on crate structure)

The `goose` core crate contains all agent logic, providers, extensions, and configuration management. While the CUSTOM_DISTROS.md does not explicitly document using it as a library dependency (i.e., adding `goose = { git = "..." }` to your `Cargo.toml`), the crate architecture supports this in principle. The separation of `goose` (core) from `goose-cli` and `goose-server` means the core logic is already designed as a reusable library.

However, the primary documented embedding paths are the REST API and ACP protocol, not direct Rust crate dependency. This is likely because:
- API stability of internal Rust types is not guaranteed
- The REST/ACP interface provides a stable contract
- Most custom distributions will use different languages for their UI layer

### Third-party embedding tools

**Status: Confirmed** (source: [GitHub](https://github.com/coder/agentapi))

The `coder/agentapi` project provides an HTTP API wrapper for multiple agents including Goose (alongside Claude Code, Aider, Gemini, Amp, and Codex), suggesting there is demand for embedding Goose into larger systems.

---

## 8. Comparison with Other Forkable Agents

### Goose vs. other open-source agents for fork-friendliness

| Dimension | Goose (Block) | OpenHands | Aider | SWE-Agent | Codex CLI |
|-----------|--------------|-----------|-------|-----------|-----------|
| **License** | Apache 2.0 | MIT | Apache 2.0 | MIT | Apache 2.0 |
| **Custom distro docs** | Dedicated CUSTOM_DISTROS.md with 9 scenarios | No dedicated guide | No dedicated guide | No dedicated guide | No dedicated guide |
| **Architecture for embedding** | 3-layer (UI / server / core) with REST API + ACP | Event-stream SDK + Server | CLI-focused, no server layer | CLI-focused | CLI + cloud sandbox |
| **White-label support** | Explicit branding guide, env vars, checklist | Not documented | Not applicable | Not applicable | Not applicable |
| **Extension model** | MCP-native (standardized) | Custom tool system, Agent Index | LLM-agnostic, repo map | Agent-Computer Interface | Sandboxed tools |
| **Provider abstraction** | Declarative JSON (no code), or Rust trait | Python provider classes | Extensive LLM support via litellm | OpenAI-compatible | OpenAI only |
| **Custom UI path** | REST API + ACP, documented endpoints | Web UI, Docker-based | Terminal only | Terminal only | Terminal + web |
| **Recipe/workflow system** | YAML recipes with sub-recipes and subagents | No equivalent | No equivalent | No equivalent | No equivalent |
| **GitHub stars** | 32.5K | 68.6K | 39K+ | 18.6K | N/A |
| **GitHub forks** | 2,984 | High | High | Moderate | N/A |
| **Foundation governance** | Linux Foundation (AAIF) | Independent | Independent | Academic | OpenAI |

### Goose's distinctive advantages for forking

**Status: Inferred** (based on comparative analysis)

1. **Only agent with dedicated custom distribution documentation.** No other open-source agent provides a comparable guide. OpenHands has an SDK but targets building new agents, not rebranding/redistributing the existing one.

2. **Three-layer architecture enables clean UI replacement.** The goosed REST API + ACP layer means a fork can completely replace the UI without touching agent logic. OpenHands achieves something similar with its server component, but Goose's architecture is more explicitly documented for this purpose.

3. **MCP as the extension standard.** By building on MCP (now also under the Agentic AI Foundation), Goose extensions are portable and not locked to the framework. A custom distribution's MCP servers can theoretically work with any MCP-compatible agent.

4. **Declarative provider system.** Adding a new AI provider via a JSON file (no code) is unique to Goose among this group. Other agents require code changes to add providers.

5. **Recipe system for configuration-level customization.** YAML recipes with parameters, sub-recipes, and subagents allow sophisticated workflow customization without touching source code -- a significant advantage for maintaining forks.

6. **Foundation governance reduces single-vendor risk.** The contribution to the Linux Foundation's AAIF (December 2025) means the project's direction is not solely controlled by Block, reducing risk for organizations building on it.

### Goose's disadvantages for forking

1. **Rust core raises the bar.** Deep customization requires Rust expertise, which is less common than Python (OpenHands, Aider, SWE-Agent). The TypeScript UI layer is more accessible.

2. **Rapid release cadence (123+ releases).** Frequent releases mean frequent merge work for divergent forks.

3. **ACP transition in progress.** The shift from REST API to ACP-over-HTTP could be a breaking change for integrations built on the current API.

---

## Summary of Confidence Levels

| Finding | Status | Source |
|---------|--------|--------|
| CUSTOM_DISTROS.md content and recommendations | Confirmed | GitHub repo |
| Three-layer architecture (CLI/goosed/core) | Confirmed | CUSTOM_DISTROS.md |
| Customization surface (config vs code) | Confirmed | CUSTOM_DISTROS.md |
| White-labeling capabilities | Confirmed | CUSTOM_DISTROS.md |
| REST API and ACP embedding interfaces | Confirmed | CUSTOM_DISTROS.md |
| Stripe Minions as custom distribution | Confirmed | Stripe Dev Blog |
| Fork maintenance guidance | Confirmed | CUSTOM_DISTROS.md |
| Recipe and subagent system | Confirmed | CUSTOM_DISTROS.md |
| AAIF governance transition | Confirmed | Linux Foundation press release |
| ACP-over-HTTP transition plan | Confirmed | GitHub issue #6642 |
| Using goose crate as Rust library dependency | Inferred | Architecture implies it but not documented |
| Other company custom distributions beyond Stripe | Unknown | No public evidence found |
| Merge conflict risk areas for forks | Inferred | Based on architecture analysis |
| Comparative fork-friendliness advantage | Inferred | Based on feature comparison |

---

## Sources

- [CUSTOM_DISTROS.md -- block/goose](https://github.com/block/goose/blob/main/CUSTOM_DISTROS.md)
- [block/goose GitHub Repository](https://github.com/block/goose)
- [Minions: Stripe's one-shot, end-to-end coding agents (Part 1)](https://stripe.dev/blog/minions-stripes-one-shot-end-to-end-coding-agents)
- [Minions: Stripe's one-shot, end-to-end coding agents (Part 2)](https://stripe.dev/blog/minions-stripes-one-shot-end-to-end-coding-agents-part-2)
- [Block Open Source Introduces "codename goose"](https://block.xyz/inside/block-open-source-introduces-codename-goose)
- [goosed to ACP-over-HTTP -- Issue #6642](https://github.com/block/goose/issues/6642)
- [Goose OSS Roadmap (Feb-Apr 2026) -- Discussion #6973](https://github.com/block/goose/discussions/6973)
- [coder/agentapi -- HTTP API for multiple agents](https://github.com/coder/agentapi)
- [OpenHands Software Agent SDK](https://openhands.dev/)
- [OpenHands vs SWE-Agent comparison](https://localaimaster.com/blog/openhands-vs-swe-agent)
- [Top Open Source AI Agent Projects](https://www.nocobase.com/en/blog/github-open-source-ai-agent-projects)
