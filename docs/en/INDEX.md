# Learn Claude Code — Complete Documentation Index

> A complete technical analysis document set based on Claude Code source code
>
> Target audience: developers who want to understand Claude Code's internals, engineers who want to build similar tools

---

## Quick Navigation

### 🗺️ Overview & Architecture

| Document | Summary |
|----------|---------|
| **[00 — Overview](./00-overview.md)** | System architecture diagram, quick reference for 15 core systems, key numbers, technical decision analysis |
| **[README (original)](./README.md)** | Introductory overview of Agent Loop, tool system, permissions, compaction, and API (749 lines) |

### 🔧 Core Agent Mechanisms

| Document | Summary |
|----------|---------|
| **[01 — Agent Loop](./01-agent-loop.md)** | Deep dive into query.ts (1729 lines): 8 phases, State machine, transition reasons, termination conditions |
| **[02 — Tool System](./02-tool-system.md)** | Complete list of 44 tools, buildTool() factory, ToolUseContext fields, tool lifecycle |
| **[03 — Permission System](./03-permission-system.md)** | 7-layer pipeline, 6 modes, Glob matching, dangerous command detection, Windows bypass defenses |
| **[04 — Context Compaction](./04-compaction.md)** | 4-layer compaction architecture: Snip → Microcompact → Collapse → Autocompact, threshold calculation |

### 🌐 Infrastructure Layer

| Document | Summary |
|----------|---------|
| **[05 — API Layer](./05-api-layer.md)** | 4 backends, OAuth PKCE 7-step flow, Prompt Cache insertion rules, withRetry algorithm |
| **[06 — REPL Interface](./06-repl-and-ink.md)** | Custom Ink fork, double-buffered rendering, REPL state machine, setToolJSX injection mechanism |
| **[07 — Commands System](./07-commands.md)** | 88+ slash commands, 3 types, dispatch flow, Skill format |
| **[08 — Memory System](./08-memory-system.md)** | CLAUDE.md 7-level priority, @import syntax, memdir, Session Memory threshold |
| **[09 — MCP Integration](./09-mcp.md)** | 6 transport protocols, MCPServerConnection state machine, tool registration, configuration format |

### ⚙️ Configuration, State & Data

| Document | Summary |
|----------|---------|
| **[10 — State & Config](./10-state-and-config.md)** | AppState 450+ fields, immutable updates, 5-level config priority, 11 database migrations |
| **[11 — Cost & History](./11-cost-and-history.md)** | Model pricing table, cost formula, history.jsonl, paste-cache, Esc undo mechanism |

### 🚀 Advanced Features

| Document | Summary |
|----------|---------|
| **[12 — Advanced Features](./12-advanced-features.md)** | Multi-agent Fork Prompt Cache optimization, Bridge remote control, LSP integration, plugin system |
| **[13 — Security](./13-security.md)** | Permission rules, dangerous command detection, Auto mode ML classifier, DenialTracking loop prevention |
| **[14 — Miscellaneous Features](./14-misc-features.md)** | Vim 11-state machine, Keybindings 19 contexts, Voice 3-layer gate, Teleport, VCR |

---

## Suggested Learning Paths

### Path A: Quick Overview (30 minutes)

1. [00 — Overview](./00-overview.md): build the big picture
2. [README](./README.md): core data flow

### Path B: Technical Depth (2–3 hours)

1. [00 — Overview](./00-overview.md)
2. [01 — Agent Loop](./01-agent-loop.md): understand the core loop
3. [02 — Tool System](./02-tool-system.md): understand the tool mechanism
4. [04 — Context Compaction](./04-compaction.md): understand the most complex optimization
5. [05 — API Layer](./05-api-layer.md): understand Prompt Cache

### Path C: Build a Similar System (all documents)

Read all 15 documents in the order listed above; each takes approximately 20–40 minutes.

---

## Topic Keyword Quick Reference

| Keyword | Related Documents |
|---------|------------------|
| AsyncGenerator, yield, streaming | 01, 05, 06 |
| Tool call, tool_use, tool_result | 01, 02, 03 |
| Prompt Cache, cache hit | 01, 04, 05, 12 |
| Zod schema, input validation | 02, 03 |
| Permission dialog, ML classifier | 03, 06 |
| Context compaction, context window | 01, 04 |
| Anthropic API, Bedrock, Vertex | 05 |
| OAuth, PKCE, Keychain | 05 |
| React, Ink, terminal UI | 06 |
| Slash commands, /compact, /help | 07 |
| CLAUDE.md, project memory | 08 |
| MCP, Model Context Protocol | 09 |
| settings.json, config priority | 10 |
| Cost tracking, token pricing | 11 |
| Sub-agent, Fork, Coordinator | 12 |
| Bridge, remote control | 12 |
| LSP, language server | 12 |
| Plugin | 12 |
| Dangerous commands, security | 03, 13 |
| Vim mode, state machine | 14 |
| Keybindings, shortcuts | 14 |
| Voice, speech input | 14 |
| Teleport, session migration | 14 |
| VCR, test fixtures | 14 |

---

## Source File Quick Index

| Source File | Document |
|-------------|----------|
| `src/query.ts` | 01 |
| `src/Tool.ts` | 02 |
| `src/tools/` | 02, 13 |
| `src/utils/permissions/` | 03, 13 |
| `src/utils/sandbox/` | 03, 13 |
| `src/services/compact/` | 04 |
| `src/services/contextCollapse/` | 04 |
| `src/services/api/` | 05 |
| `src/screens/REPL.tsx` | 06 |
| `src/ink/ink.tsx` | 06 |
| `src/hooks/` | 06, 03 |
| `src/commands/` | 07 |
| `src/utils/memory/` | 08 |
| `src/services/mcp/` | 09 |
| `src/state/AppState.ts` | 10 |
| `src/utils/config.ts` | 10 |
| `src/cost-tracker.ts` | 11 |
| `src/history.ts` | 11 |
| `src/tools/AgentTool/` | 12 |
| `src/bridge/` | 12 |
| `src/services/lsp/` | 12 |
| `src/plugins/` | 12 |
| `src/vim/` | 14 |
| `src/keybindings/` | 14 |
| `src/voice/` | 14 |
| `src/utils/teleport/` | 14 |
| `src/services/vcr.ts` | 14 |

---

*This document set was written based on Claude Code source code analysis and covers approximately 95% of the core technical systems.*
