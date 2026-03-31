# 12 — Advanced Features: Multi-Agent, Bridge Remote Control, LSP, and Plugins

> Files: `src/tools/AgentTool/`, `src/bridge/`, `src/services/lsp/`, `src/plugins/`, `src/coordinator/`

---

## 1. Multi-Agent Coordination

### 1.1 AgentTool: Launching Sub-Agents

```typescript
// Input schema (excerpted)
z.object({
  prompt: z.string(),                         // task description for the sub-agent
  subagent_type: z.string().optional(),       // which Agent definition to use
  description: z.string().optional(),         // user-visible description
  model: z.string().optional(),              // model override
  run_in_background: z.boolean().optional(), // async execution (don't wait for result)
  name: z.string().optional(),               // Agent name (used in team mode)
  team_name: z.string().optional(),          // team name (for parallel multi-agent)
  isolation: z.enum(['worktree', 'remote']).optional(),  // isolation mode
  mode: permissionModeSchema().optional(),   // permission mode override
})
```

**Two execution paths:**

```
Has team_name + name
    │
    ▼
spawnTeammate()          ← runs in an isolated split-pane (visual)

No team_name + name
    │
    ├─ run_in_background = true
    │    └─ async Agent (returns agentId + outputFile)
    │
    └─ run_in_background = false (default)
         └─ synchronous sub-agent (blocks and waits for result)
```

### 1.2 Fork Sub-Agent: Prompt Cache Optimization

Fork is the most commonly used sub-agent mode. The key design goal: **all Fork sub-agents share the same Prompt Cache prefix**, maximizing cache hit rate.

```typescript
// src/tools/AgentTool/forkSubagent.ts

function buildForkedMessages(directive: string, parentMessage: AssistantMessage) {
  // Strategy: construct a byte-identical prefix + a unique directive suffix

  // 1. Clone the parent agent's full assistant message (including all tool_use blocks)
  const fullAssistantMessage = deepClone(parentMessage)

  // 2. Build placeholder tool_results for all tool_use blocks
  //    Key: all Forks use the exact same placeholder text
  const FORK_PLACEHOLDER_RESULT = 'Fork started — processing in background'
  const toolResultBlocks = toolUseBlocks.map(block => ({
    type: 'tool_result',
    tool_use_id: block.id,
    content: [{ type: 'text', text: FORK_PLACEHOLDER_RESULT }],
    //                                  ↑ identical for all Forks → cache shared!
  }))

  // 3. Append the sub-agent-specific directive at the end (this part differs)
  const toolResultMessage = createUserMessage({
    content: [
      ...toolResultBlocks,      // identical placeholders
      { type: 'text', text: buildChildMessage(directive) },  // unique directive
    ],
  })

  // Result structure:
  // [...history messages, assistant(all tools), user(placeholders + unique directive)]
  //  ├─ this part is identical for all Forks ────────────────────────────────────────┤
  //                                            ├─ this part differs per Fork ─────────┘
  return [fullAssistantMessage, toolResultMessage]
}
```

**Fork recursion detection:**

```typescript
// Prevents Fork sub-agents from launching further Forks (infinite nesting)
export function isInForkChild(messages: Message[]): boolean {
  return messages.some(m =>
    m.type === 'user' &&
    m.message.content.some(b =>
      b.type === 'text' && b.text.includes('<fork-boilerplate>')
    )
  )
}
```

### 1.3 Tool Filtering (Sub-Agents Cannot Use All Tools)

```typescript
// Tools disallowed for all agents
const ALL_AGENT_DISALLOWED_TOOLS = new Set([
  'AskUserQuestion',  // sub-agents cannot show dialog boxes
  'SendMessage',      // cannot send messages to the user
  // ...
])

// Additional tools disallowed for custom agents
const CUSTOM_AGENT_DISALLOWED_TOOLS = new Set([
  'EnterPlanMode',    // custom agents cannot enter plan mode
  'ExitPlanMode',
  // ...
])

// Tools allowed for async agents (whitelist)
const ASYNC_AGENT_ALLOWED_TOOLS = new Set([
  'Bash', 'Read', 'Write', 'Edit', 'Glob', 'Grep',
  'WebFetch', 'WebSearch',
  'TaskCreate', 'TaskUpdate', 'TaskList', 'TaskGet', 'TaskOutput', 'TaskStop',
  // Note: interactive tools such as AskUserQuestion are excluded
])
```

### 1.4 Coordinator Mode

```typescript
// Enable: CLAUDE_CODE_COORDINATOR_MODE=1
// Purpose: in coordinator mode, inform the model which tools sub-Workers can use

export function getCoordinatorUserContext(mcpClients, scratchpadDir) {
  const workerTools = Array.from(ASYNC_AGENT_ALLOWED_TOOLS)
    .filter(name => !INTERNAL_WORKER_TOOLS.has(name))
    .join(', ')

  return {
    workerTools: `Workers spawned via the Agent tool have access to: ${workerTools}`,
  }
}
```

---

## 2. Bridge: Remote Control Protocol

The Bridge system lets users remotely control a locally running Claude Code instance via a **web interface or mobile device**.

### 2.1 Message Format

```typescript
// Outbound messages (Claude Code → remote client)
type SDKMessage =
  | { type: 'user';      content: string; uuid: string }  // user message (echo)
  | { type: 'assistant'; content: string; uuid: string }  // assistant message
  | { type: 'tool_use';  ... }  // tool call notification
  | { type: 'system';    ... }  // system event

// Inbound message types
type InboundMessage =
  | SDKMessage          // message from the user (triggers conversation)
  | SDKControlResponse  // response to a control request (permission decisions, etc.)
  | SDKControlRequest   // control command from the server
```

### 2.2 Server Control Requests (`control_request`)

Remote clients can send control commands to manipulate the local Claude Code instance:

```typescript
// Supported control request types
switch (request.request.subtype) {
  case 'initialize':
    // Handshake, returns current state (command list, available models, etc.)

  case 'set_model':
    // Switch model (e.g., from Sonnet to Opus)
    onSetModel(request.request.model)

  case 'set_max_thinking_tokens':
    // Adjust the thinking token limit
    onSetMaxThinkingTokens(request.request.max_thinking_tokens)

  case 'set_permission_mode':
    // Switch permission mode (default/plan/auto/bypass)
    const result = onSetPermissionMode(request.request.mode)

  case 'interrupt':
    // Interrupt the current request (equivalent to pressing Ctrl+C remotely)
    onInterrupt()
}

// All control requests have a corresponding control_response acknowledgment
// { type: 'control_response', response: { subtype: 'success' | 'error', request_id } }
```

### 2.3 UUID Deduplication (BoundedUUIDSet)

Prevents messages from being processed multiple times (network retransmission):

```typescript
class BoundedUUIDSet {
  // Ring buffer: FIFO eviction, fixed capacity
  private ring: (string | undefined)[]  // circular array
  private set = new Set<string>()       // fast lookup
  private writeIdx = 0

  add(uuid: string) {
    // Evict the oldest entry, insert the new UUID
    const evicted = this.ring[this.writeIdx]
    if (evicted) this.set.delete(evicted)
    this.ring[this.writeIdx] = uuid
    this.set.add(uuid)
    this.writeIdx = (this.writeIdx + 1) % this.capacity
  }
}

// Two independent sets:
// recentPostedUUIDs  → filter messages sent by self (echo filtering)
// recentInboundUUIDs → prevent duplicate processing of inbound messages
```

### 2.4 Outbound-Only Mode

```typescript
// outboundOnly: true → only send messages to the remote, do not accept remote control commands
// Use case: user wants to observe on mobile but does not allow remote control
if (outboundOnly && request.request.subtype !== 'initialize') {
  return error('This session is outbound-only. Enable Remote Control locally...')
}
```

---

## 3. LSP Integration

### 3.1 Architecture

```
Claude Code
    │
    ▼
LSPServerManager          manages all LSP server instances
    │
    ├─ LSPServerInstance   single server (e.g., TypeScript/Python language server)
    │    │
    │    └─ LSPClient      JSON-RPC over stdio communication layer
    │         │
    │         ▼
    │    subprocess (e.g., tsserver, pyright, rust-analyzer)
    │
    └─ LSPTool            tool interface (model can call LSP capabilities)
```

### 3.2 Supported LSP Capabilities

```typescript
// File operations
openFile(path, content)      // textDocument/didOpen
changeFile(path, content)    // textDocument/didChange
saveFile(path)               // textDocument/didSave
closeFile(path)              // textDocument/didClose

// Queries (via sendRequest)
getCompletions(path, position)    // textDocument/completion
getDiagnostics(path)              // textDocument/publishDiagnostics
getDefinition(path, position)     // textDocument/definition
getReferences(path, position)     // textDocument/references
getHover(path, position)          // textDocument/hover
```

### 3.3 Server Lifecycle and Fault Tolerance

```
stopped → starting → running → stopping → stopped
             │                      ↑
             └──── error ───────────┘  (restart to recover)

Maximum automatic restarts: config.maxRestarts ?? 3
Error code -32801 (content modified): exponential backoff retry (3 times, 500/1000/2000ms)
```

### 3.4 Configuration Format

```jsonc
// Configure LSP servers in settings.json
{
  "lspServers": {
    "typescript": {
      "command": "typescript-language-server",
      "args": ["--stdio"],
      "extensionToLanguage": {
        ".ts": "typescript",
        ".tsx": "typescriptreact",
        ".js": "javascript"
      }
    },
    "python": {
      "command": "pyright-langserver",
      "args": ["--stdio"],
      "extensionToLanguage": {
        ".py": "python"
      }
    }
  }
}
```

---

## 4. Plugin System

### 4.1 Plugin Structure

```typescript
type LoadedPlugin = {
  name: string
  manifest: PluginManifest        // name, description, version
  path: string                    // file path ('builtin' for built-in plugins)
  source: string                  // source identifier
  isBuiltin?: boolean
  enabled?: boolean
  sha?: string                    // git commit SHA (version pinning)

  // Content a plugin can provide
  commandsPath?: string           // path to commands file
  skillsPath?: string             // path to skills file
  agentsPath?: string             // path to Agent definitions
  outputStylesPath?: string       // custom output styles
  hooksConfig?: HooksSettings     // hook configuration
  mcpServers?: Record<string, McpServerConfig>   // MCP servers
  lspServers?: Record<string, LspServerConfig>   // LSP servers
}
```

### 4.2 Built-in Plugin Registration

```typescript
// Any module can register a built-in plugin
registerBuiltinPlugin({
  name: 'my-builtin-plugin',
  description: '...',
  defaultEnabled: true,
  skills: [...],         // provides Skill commands
  mcpServers: {...},     // provides MCP servers
  hooks: {...},          // provides Hooks
  isAvailable: () => process.platform === 'darwin',  // conditionally available
})
```

### 4.3 Plugin Enabled State

```typescript
// Priority: user settings > plugin default > true (enabled by default)
const pluginId = `${name}@builtin`
const userSetting = settings?.enabledPlugins?.[pluginId]
const isEnabled = userSetting !== undefined
  ? userSetting === true
  : (definition.defaultEnabled ?? true)
```

### 4.4 Error Types (Complete)

```typescript
type PluginError =
  | { type: 'path-not-found'; ... }
  | { type: 'git-auth-failed'; gitUrl; authType: 'ssh' | 'https' }
  | { type: 'git-timeout'; operation: 'clone' | 'pull' }
  | { type: 'network-error'; url; details? }
  | { type: 'manifest-parse-error'; parseError }
  | { type: 'manifest-validation-error'; validationErrors }
  | { type: 'plugin-not-found'; pluginId; marketplace }
  | { type: 'mcp-config-invalid'; serverName; validationError }
  | { type: 'mcp-server-suppressed-duplicate'; duplicateOf }
  | { type: 'lsp-config-invalid'; serverName; validationError }
  | { type: 'hook-load-failed'; hookPath; reason }
  | { type: 'component-load-failed'; component; path; reason }
```

---

## Key File Reference

| File | Purpose |
|------|---------|
| `src/tools/AgentTool/AgentTool.tsx` | AgentTool main implementation |
| `src/tools/AgentTool/forkSubagent.ts` | Fork mechanism + buildForkedMessages |
| `src/tools/AgentTool/agentToolUtils.ts` | Tool filtering logic |
| `src/coordinator/coordinatorMode.ts` | Coordinator mode |
| `src/bridge/bridgeMain.ts` | Bridge main logic (115KB) |
| `src/bridge/bridgeMessaging.ts` | Message routing + UUID deduplication |
| `src/bridge/types.ts` | Bridge message type definitions |
| `src/services/lsp/LSPClient.ts` | JSON-RPC client |
| `src/services/lsp/LSPServerManager.ts` | Server pool management |
| `src/services/lsp/LSPServerInstance.ts` | Single-instance lifecycle |
| `src/plugins/builtinPlugins.ts` | Built-in plugin registration and loading |
