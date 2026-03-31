# 00 — Overview: Complete System Map of Claude Code

> A comprehensive technical analysis based on the Claude Code source code (~1332 TypeScript files, ~9.5 MB of component code)

---

## 1. Project Overview

Claude Code is a **terminal AI Coding Agent** developed by Anthropic. Unlike tools such as ChatGPT and Cursor, it is:
- **A CLI tool**: runs in the terminal, no IDE plugin required
- **Fully agentic**: autonomously invokes 44+ tools (read/write files, execute commands, search the web)
- **React-driven UI**: uses Ink (React for CLI) to render the terminal interface
- **Full TypeScript implementation**: ~1332 `.ts` files, running on Bun or Node.js

---

## 2. Core Technology Stack

```
TypeScript + React (Ink) + Bun/Node.js
    │
    ├─ Runtime: Bun (preferred) or Node.js
    ├─ UI framework: Ink (custom fork, React for terminal)
    ├─ State management: custom AppState (immutable state tree with 450+ fields)
    ├─ Schema validation: Zod v4
    ├─ Package management: npm (publishing), Bun internally
    └─ Build: bun:bundle (dead-code elimination with feature flags)
```

---

## 3. System Architecture Diagram

```
┌─────────────────────────────────────────────────────────┐
│                      UI Layer                             │
│  ┌──────────────────────────────────────────────────┐   │
│  │  REPL.tsx (React/Ink)                            │   │
│  │  • Message list • Input box • Status bar • Tool UI injection │
│  └─────────────────┬────────────────────────────────┘   │
│                    │                                      │
│  ┌─────────────────▼────────────────────────────────┐   │
│  │  Commands system (88+ slash commands)            │   │
│  │  /compact  /clear  /help  /vim-mode  ...         │   │
│  └─────────────────┬────────────────────────────────┘   │
└────────────────────┼────────────────────────────────────┘
                     │
┌────────────────────▼────────────────────────────────────┐
│                    Agent Loop Layer                       │
│  ┌──────────────────────────────────────────────────┐   │
│  │  query.ts (1729-line AsyncGenerator)             │   │
│  │  • Snip → Microcompact → Collapse → Autocompact  │   │
│  │  • callModel → streaming → tool execution → loop │   │
│  └─────────────────┬────────────────────────────────┘   │
│                    │                                      │
│  ┌─────────────────▼────────────────────────────────┐   │
│  │  Tool system (44 tools)                          │   │
│  │  Bash│Read│Edit│Write│Glob│Grep│WebFetch│Agent...│   │
│  └─────────────────┬────────────────────────────────┘   │
└────────────────────┼────────────────────────────────────┘
                     │
┌────────────────────▼────────────────────────────────────┐
│                    Infrastructure Layer                   │
│  ┌────────────┐ ┌──────────┐ ┌───────────┐             │
│  │ API layer  │ │ MCP layer│ │ LSP layer │             │
│  │ 4 backends │ │ 6 transports│ │ JSON-RPC │            │
│  └────────────┘ └──────────┘ └───────────┘             │
│  ┌────────────┐ ┌──────────┐ ┌───────────┐             │
│  │ Permission │ │ Memory   │ │ Config    │             │
│  │ 7-layer    │ │ CLAUDE.md│ │ 5-level   │             │
│  │ pipeline   │ │ system   │ │ priority  │             │
│  └────────────┘ └──────────┘ └───────────┘             │
└─────────────────────────────────────────────────────────┘
```

---

## 4. Quick Reference: 15 Core Systems

### 4.1 Agent Loop (`src/query.ts`)

The heart of Claude Code. A `while(true)` AsyncGenerator; each iteration:
1. Checks context size → compresses as needed (4 layers)
2. Calls the model API (streaming)
3. Collects `tool_use` blocks
4. Executes tools (in parallel or serially)
5. Appends tool results to messages and continues to the next iteration

**Termination conditions:** model stops calling tools, user interrupts, `maxTurns` reached, context limit exceeded, Stop Hook blocks continuation

---

### 4.2 Tool System (`src/tools/`, 44 tools)

All tools are created via the `buildTool(def)` factory function and implement the `Tool<Input, Output>` interface.

**Tool categories:**
- File operations: Read, Write, Edit, Glob, Grep, NotebookEdit
- Shell: Bash, PowerShell, REPL
- Agent coordination: Agent, SendMessage, AskUserQuestion, TeamCreate
- Task management: TaskCreate/Update/List/Get/Output/Stop
- Web: WebFetch, WebSearch
- MCP: MCPTool, McpAuth, ListMcpResources, ReadMcpResource
- UI/interaction: Config, EnterPlanMode, ExitPlanMode, EnterWorktree, Skill, ToolSearch
- Utility: Sleep, Brief, LSPTool, ScheduleCron

---

### 4.3 Context Compaction (`src/services/compact/`)

4-tier progressive compaction — higher tiers are more costly but achieve greater compression:

| Tier | Mechanism | Trigger | Cost |
|------|-----------|---------|------|
| Tool result offloading | Large results written to disk | Single result exceeds limit | Lowest |
| Snip | Trims old tool content | Token count exceeds threshold | Low |
| Microcompact | Clears old tool results | Token count exceeds threshold | Low |
| Context Collapse | Progressive summary folding | Token count exceeds threshold | Medium |
| Autocompact | AI summarizes entire conversation | Approaching context window limit | High (calls Haiku) |

---

### 4.4 API Layer (`src/services/api/`)

4 backends with a unified interface:
- **Direct**: calls the Anthropic API directly (`api.anthropic.com`)
- **Bedrock**: AWS-hosted (requires AWS credentials)
- **Vertex**: Google Cloud-hosted (GCP service account)
- **Azure**: Azure AI (endpoint URL + API key)

OAuth PKCE 7-step flow for Anthropic account authentication (stored in Mac Keychain).

Prompt Cache: `cache_control` is automatically inserted at 3 positions (end of tool list, end of CLAUDE.md, end of the latest user message).

---

### 4.5 Permission System (`src/utils/permissions/`)

7-layer pipeline; decision priority is Deny > Ask > Allow:

```
validateInput → dangerous command detection → deny rules → allow rules
  → ML classifier → non-interactive auto-decision → interactive confirmation dialog
```

6 permission modes: `default`, `plan`, `acceptEdits`, `auto`, `bypassPermissions`, `dontAsk`

---

### 4.6 REPL Interface (`src/screens/REPL.tsx`, `src/ink/`)

Custom Ink fork (faster than standard Ink):
- `queueMicrotask` rendering (lower latency than `setImmediate`)
- Explicit double-buffering (diff output, minimizes stdout writes)
- Memory pool reset every 5 minutes (prevents long-session leaks)
- Alt Screen support (full-screen mode)

---

### 4.7 Commands System (`src/commands/`, 88+ commands)

3 command types:
- `prompt`: replaces input box content (`/help`, `/compact`)
- `local`: side effects, no UI (`/clear`, `/login`)
- `local-jsx`: injects a React component into the REPL (`/config`, `/theme`)

---

### 4.8 Memory System (CLAUDE.md)

7-tier priority discovery (highest to lowest): `~/.claude/CLAUDE.md` → each parent directory → `working-dir/CLAUDE.md` → subdirectories

Supports `@import path/to/file` inline imports (3-level depth limit to prevent cycles).

---

### 4.9 MCP Integration (`src/services/mcp/`)

6 transport protocols: `stdio`, `sse`, `http-sse`, `streamable-http`, `http-streaming`, `in-process`

MCP tools are registered as: `mcp__<serverName>__<toolName>` (naming conflicts handled automatically)

---

### 4.10 State Management (`src/state/AppState.ts`)

An immutable state tree with 450+ fields. All UI state, permission context, MCP connections, etc. live here.

Update pattern: `setAppState(prev => ({ ...prev, field: newValue }))`

5-level config priority: `user settings` > `project settings` > `local settings` > `CLI flags` > `policy settings`

---

### 4.11 Cost Tracking (`src/cost-tracker.ts`)

Built-in pricing table for all models. Updated after each API call, saved to `.claude/costs.json`, and restored when using `--resume`.

Conversation history is stored in `~/.claude/history.jsonl` (JSONL format, up to 100 entries; large content offloaded to `paste-cache/`).

---

### 4.12 Multi-Agent (`src/tools/AgentTool/`)

Key optimization for forked sub-agents: **all forks share the same Prompt Cache prefix** (using identical placeholder text); only the trailing directive differs.

```
[...identical messages + same placeholder]  ← shared by all forks, cache hits
[unique directive]                           ← different per fork
```

---

### 4.13 Security Mechanisms (`src/utils/permissions/`, `src/utils/sandbox/`)

- **Glob matching**: uses the `ignore` library (gitignore format)
- **Symlink checks**: validates both the original path and `realpath`
- **Windows path bypass**: detects 7 types of NTFS special-path attacks
- **Dangerous command blocklist**: `rm -rf /`, fork bombs, `curl | bash`, etc.
- **ML classifier**: 2-second grace period; timeout falls back to manual confirmation
- **DenialTracking**: falls back to manual confirmation after 3 consecutive denials or 20 total denials

---

### 4.14 Vim/Keybindings (`src/vim/`, `src/keybindings/`)

- Vim mode: full state machine with 11 CommandStates, supports text objects, `.` repeat, registers
- Keybindings: 19 contexts, 72+ actions, supports multi-key chords (`ctrl+x ctrl+e`)
- Platform differences: Windows VT mode detection, Shift+Tab support detection

---

### 4.15 VCR Test Framework (`src/services/vcr.ts`)

Records real API responses as cassette files (JSON); playback does not call the real API.

Dehydration/rehydration: dynamic values (paths, numbers, timestamps) are replaced with placeholders to ensure portability across machines.

---

## 5. Key Metrics at a Glance

| Metric | Value |
|--------|-------|
| TypeScript file count | ~1332 |
| Component code | ~9.5 MB |
| Number of tools | 44 |
| Slash command count | 88+ |
| AppState field count | 450+ |
| Keybinding context count | 19 |
| Keybinding action count | 72+ |
| Vim CommandState count | 11 |
| Config priority levels | 5 |
| Permission pipeline layers | 7 |
| Context compaction tiers | 4 |
| API backends | 4 |
| MCP transport protocols | 6 |
| Database migration versions | 11 |
| Lines in query.ts | 1729 |
| Custom Ink fork size | 251 KB |
| Prompt Cache insertion points | 3 |
| Autocompact buffer | 13,000 tokens |
| Max conversation history entries | 100 |
| History inline threshold | 1024 bytes |

---

## 6. File System Structure

```
~/.claude/
  ├─ CLAUDE.md              # Global memory (highest priority)
  ├─ settings.json          # User config (priority level 1)
  ├─ keybindings.json       # Custom keybindings
  ├─ history.jsonl          # Conversation history (FIFO, 100 entries)
  ├─ paste-cache/           # Offloaded large paste content
  └─ projects/
       └─ <project-hash>/
            └─ ...           # Project-level config

<project>/
  ├─ CLAUDE.md              # Project memory
  ├─ .claude/
  │    ├─ settings.json     # Project config (priority level 2)
  │    ├─ settings.local.json # Local overrides (level 3, not committed to git)
  │    ├─ costs.json        # Cost tracking
  │    └─ commands/         # Custom commands directory
  └─ ...
```

---

## 7. Technical Decision Analysis

### Why use React/Ink for a CLI?

Ink maps the React virtual DOM to the terminal character grid, allowing Claude Code to build complex terminal UIs (multi-panel, rich text, dynamic updates) declaratively without manually managing ANSI escape sequences.

### Why does Prompt Cache matter so much?

For long conversations, Prompt Cache can reduce input costs from $15/MTok to $1.5/MTok (a 90% discount). Every design decision in Claude Code — immutable messages, forked sub-agents sharing a prefix, Microcompact not breaking the cache — is oriented toward maximizing cache hit rates.

### Why Zod instead of TypeScript types?

Tools' `inputSchema` is a Zod schema, which validates the LLM's tool arguments at runtime and automatically generates the JSON Schema sent to the Anthropic API. This is best practice for combining type safety with runtime validation.

### Why Bun instead of Node.js?

Bun provides faster startup times (important since users invoke it frequently from the command line). Additionally, Bun's `bun:bundle` supports conditional compilation via the `feature()` function, enabling tree-shaking of internal features (so Ant-internal features don't appear in the external release).
