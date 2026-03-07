# AI Coding Agent CLI Tools: Landscape Research (Aug 2025 -- Mar 2026)

## 1. Claude Code (Anthropic)

TypeScript/Node.js, 200K context, layered agent loop with tools + permission system + hooks. MCP integration (300+ services). Sandboxing via bubblewrap/Seatbelt. Agent Teams for multi-agent. Agent SDK for programmatic use.

## 2. Codex CLI (OpenAI)

Rust-based (v0.106.0), cloud sandbox + local CLI. JSONL output (`--json`). OS-enforced sandbox (Landlock/Seatbelt/bubblewrap). GPT-5.4 with experimental 1M context. Agents SDK integration.

## 3. Gemini CLI (Google)

Open-source TypeScript, ReAct loop, PTY-based shell execution. JSON/stream-json output formats. Session management (v0.20.0+). Seatbelt sandboxing (macOS). 1M+ token context.

## 4. Aider

Python, 100+ LLMs, git-native (auto-commits). Repository map for context. Diff/udiff/whole edit formats. No sandboxing (git rollback). ~70% of own code written by Aider.

## 5. Cursor

VS Code fork + CLI. Agent-first Cursor 2.0. Async subagents, background agents (cloud). Agent Skills for dynamic context. MCP integration.

## 6. Windsurf (Cognition AI)

VS Code fork, Cascade agent. Acquired by Cognition AI (~$250M, Dec 2025). Parallel multi-agent sessions, git worktrees. Ranked #1 LogRocket Feb 2026.

## 7. Amazon Q Developer CLI

Open-source, Bedrock-powered (Claude 3.7 Sonnet). MCP support. CLI Agent Orchestrator with tmux isolation. 66% SWE-Bench Verified.

## 8. OpenCode

112K+ GitHub stars. 75+ model providers including local. LSP integration. Agent Client Protocol (ACP) for IDE interoperability.

---

## Comparison Table

| Feature | Claude Code | Codex CLI | Gemini CLI | Aider | Cursor | Windsurf | Amazon Q | OpenCode |
|---|---|---|---|---|---|---|---|---|
| **Language** | TypeScript | Rust | TypeScript | Python | TypeScript | TypeScript | -- | Go |
| **Output** | Streaming MD | TUI + NDJSON | PTY + JSON | Diff/Udiff | IDE diffs | IDE inline | Terminal | Terminal TUI |
| **Sandbox** | bwrap/Seatbelt | Landlock+Cloud | Seatbelt | None (git) | Cloud | Dedicated term | tmux | None |
| **Multi-Agent** | Agent Teams | Agents SDK | Generalist | No | Subagents | Parallel | Orchestrator | No |
| **MCP** | Yes | Yes | Yes | No | Yes | Yes | Yes | Via ACP |
| **Local Models** | No | No | No | Yes | No | No | No | Yes |

---

## Key Trends

1. **Multi-agent convergence** (Feb 2026): Every major tool shipped multi-agent in the same two-week window
2. **MCP as standard**: 100K to 8M+ downloads in ~5 months
3. **Sandboxing maturation**: OS-enforced sandboxes becoming table stakes
4. **CLI + IDE convergence**: Boundaries blurring between terminal and IDE agents

---

## Sources

- [Claude Code Docs](https://code.claude.com/docs/en/overview)
- [Codex CLI](https://developers.openai.com/codex/cli/)
- [Gemini CLI GitHub](https://github.com/google-gemini/gemini-cli)
- [Aider](https://aider.chat/)
- [Cursor Features](https://cursor.com/features)
- [Windsurf](https://windsurf.com/cascade)
- [Amazon Q Developer](https://aws.amazon.com/q/developer/features/)
- [OpenCode](https://opencode.ai/)
