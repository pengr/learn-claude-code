# 02 — Tool System: Complete Map of All 44 Tools

> Files: `src/Tool.ts`, `src/tools/` (42 tool directories/files)

---

## 1. Core Interface: Tool\<Input, Output\>

Each tool is an object that implements the `Tool` interface and is constructed via the `buildTool()` factory function:

```typescript
// src/Tool.ts

export type Tool<
  Input extends AnyObject = AnyObject,
  Output = unknown,
  P extends ToolProgressData = ToolProgressData,
> = {
  readonly name: string
  readonly aliases?: string[]           // compatibility aliases (e.g., after a tool rename)
  readonly inputSchema: Input           // Zod schema (auto-generates JSON Schema sent to API)
  readonly inputJSONSchema?: ToolInputJSONSchema  // for MCP tools (direct JSON Schema)
  readonly shouldDefer?: boolean        // whether to lazy-load (requires ToolSearch to access)
  readonly alwaysLoad?: boolean         // never lazy-load (present in prompt from the first turn)
  readonly maxResultSizeChars: number   // if exceeded, result is persisted to disk
  readonly strict?: boolean            // strict mode (API follows schema more strictly)
  readonly isMcp?: boolean             // is an MCP tool
  readonly isLsp?: boolean             // is an LSP tool
  readonly searchHint?: string         // ToolSearch keywords (3–10 words)
  outputSchema?: z.ZodType<unknown>

  // Core methods
  call(args, context, canUseTool, parentMessage, onProgress?): Promise<ToolResult<Output>>
  description(input, options): Promise<string>

  // Permission-related
  isEnabled(): boolean
  isReadOnly(input): boolean
  isDestructive?(input): boolean
  isConcurrencySafe(input): boolean
  checkPermissions(input, context?): Promise<PermissionResult>
  toAutoClassifierInput(input): unknown     // return '' to skip the ML classifier

  // Interaction behavior
  interruptBehavior?(): 'cancel' | 'block'  // cancel or block when user types input
  requiresUserInteraction?(): boolean
  isSearchOrReadCommand?(input): { isSearch: boolean; isRead: boolean; isList?: boolean }
  isOpenWorld?(input): boolean

  // Input processing
  backfillObservableInput?(input): void     // populate legacy fields (does not modify API content)
  inputsEquivalent?(a, b): boolean          // check whether two inputs are equivalent

  userFacingName(input): string             // display name in UI
  mcpInfo?: { serverName: string; toolName: string }  // MCP metadata
}
```

---

## 2. buildTool() Factory Function

```typescript
// src/Tool.ts

// Safe defaults (fail-closed)
const TOOL_DEFAULTS = {
  isEnabled: () => true,
  isConcurrencySafe: () => false,      // default: unsafe (concurrent execution not allowed)
  isReadOnly: () => false,             // default: write operation
  isDestructive: () => false,
  checkPermissions: (input) =>
    Promise.resolve({ behavior: 'allow', updatedInput: input }),  // defer to general permission system
  toAutoClassifierInput: () => '',     // default: skip ML classifier
  userFacingName: () => '',
}

export function buildTool<D extends AnyToolDef>(def: D): BuiltTool<D> {
  return { ...TOOL_DEFAULTS, ...def }
  // BuiltTool<D> precisely represents the spread result at the type level:
  // for each field — if def provides it → use def's value; otherwise → use TOOL_DEFAULTS value
}
```

**Usage example (excerpt from FileReadTool):**

```typescript
export const FileReadTool = buildTool({
  name: FILE_READ_TOOL_NAME,  // 'Read'
  maxResultSizeChars: Infinity,    // never offload (avoids Read→file→Read cycles)
  isReadOnly: () => true,
  isConcurrencySafe: () => true,   // file reads can be concurrent
  isSearchOrReadCommand: () => ({ isSearch: false, isRead: true }),

  inputSchema: z.object({
    file_path: z.string(),
    limit: z.number().optional(),
    offset: z.number().optional(),
  }),

  async call(args, context, canUseTool, parentMessage) {
    // Read file, handle permissions, return content
    return { type: 'result', resultForAssistant: content }
  },

  async description(input) {
    return `Read ${input.file_path}`
  },
})
```

---

## 3. Complete List of All 44 Tools

### 3.1 File Operation Tools

| Tool name | Directory | Function |
|-----------|-----------|----------|
| `Read` | `FileReadTool/` | Read files; supports line ranges, PDFs, images |
| `Write` | `FileWriteTool/` | Write files (full overwrite) |
| `Edit` | `FileEditTool/` | Precise string replacement (old_string → new_string) |
| `Glob` | `GlobTool/` | File path pattern matching, sorted by modification time |
| `Grep` | `GrepTool/` | Content search powered by ripgrep |
| `NotebookEdit` | `NotebookEditTool/` | Edit Jupyter Notebook cells |

### 3.2 Shell Execution Tools

| Tool name | Directory | Function |
|-----------|-----------|----------|
| `Bash` | `BashTool/` | Execute bash commands (120s timeout, streaming output) |
| `PowerShell` | `PowerShellTool/` | Windows PowerShell commands |
| `REPL` | `REPLTool/` | Node.js REPL (persistent VM context across calls) |

### 3.3 Agent / Coordination Tools

| Tool name | Directory | Function |
|-----------|-----------|----------|
| `Agent` | `AgentTool/` | Launch sub-agents (synchronous/async/background/team) |
| `SendMessage` | `SendMessageTool/` | Send a message to a specified agent or the user |
| `AskUserQuestion` | `AskUserQuestionTool/` | Ask the user a question (single-choice/multi-choice/preview) |
| `TeamCreate` | `TeamCreateTool/` | Create an agent team (split-pane UI) |
| `TeamDelete` | `TeamDeleteTool/` | Delete an agent team |
| `RemoteTrigger` | `RemoteTriggerTool/` | Trigger a remote agent task |

### 3.4 Task Management Tools

| Tool name | Directory | Function |
|-----------|-----------|----------|
| `TaskCreate` | `TaskCreateTool/` | Create a task (subject + description) |
| `TaskUpdate` | `TaskUpdateTool/` | Update task status/content |
| `TaskList` | `TaskListTool/` | List all tasks |
| `TaskGet` | `TaskGetTool/` | Get task details |
| `TaskOutput` | `TaskOutputTool/` | Get output from a background task |
| `TaskStop` | `TaskStopTool/` | Stop a background task |

### 3.5 Web Tools

| Tool name | Directory | Function |
|-----------|-----------|----------|
| `WebFetch` | `WebFetchTool/` | Fetch a webpage and process content with AI |
| `WebSearch` | `WebSearchTool/` | Web search (returns structured results) |

### 3.6 MCP Tools

| Tool name | Directory | Function |
|-----------|-----------|----------|
| `MCPTool` | `MCPTool/` | Dynamic MCP tool invocation (`mcp__<server>__<tool>`) |
| `McpAuth` | `McpAuthTool/` | MCP OAuth authentication handling |
| `ListMcpResources` | `ListMcpResourcesTool/` | List MCP resources |
| `ReadMcpResource` | `ReadMcpResourceTool/` | Read an MCP resource |

### 3.7 UI / Interaction Tools

| Tool name | Directory | Function |
|-----------|-----------|----------|
| `Config` | `ConfigTool/` | Read and write Claude Code configuration |
| `EnterPlanMode` | `EnterPlanModeTool/` | Enter plan mode |
| `ExitPlanMode` | `ExitPlanModeTool/` | Exit plan mode (waits for user approval) |
| `EnterWorktree` | `EnterWorktreeTool/` | Create and switch to a git worktree |
| `ExitWorktree` | `ExitWorktreeTool/` | Exit a git worktree |
| `TodoWrite` | `TodoWriteTool/` | Write to the Todo list (in memory) |
| `Skill` | `SkillTool/` | Execute a Skill (loads a Markdown instruction set) |
| `ToolSearch` | `ToolSearchTool/` | Search available tools (entry point for lazy loading) |

### 3.8 Utility Tools

| Tool name | Directory | Function |
|-----------|-----------|----------|
| `Sleep` | `SleepTool/` | Wait for a specified duration (with countdown UI) |
| `Brief` | `BriefTool/` | Generate a concise summary |
| `LSPTool` | `LSPTool/` | Language server calls via LSP |
| `ScheduleCron` | `ScheduleCronTool/` | Create scheduled tasks (CronCreate) |
| `SyntheticOutput` | `SyntheticOutputTool/` | Generate synthetic output (for testing) |

---

## 4. ToolUseContext: Execution Context

`ToolUseContext` is the "execution environment" passed into each tool call, containing all external dependencies a tool needs:

```typescript
// src/Tool.ts (key fields, excerpted)

export type ToolUseContext = {
  options: {
    commands: Command[]
    debug: boolean
    mainLoopModel: string
    tools: Tools
    verbose: boolean
    thinkingConfig: ThinkingConfig
    mcpClients: MCPServerConnection[]
    isNonInteractiveSession: boolean
    agentDefinitions: AgentDefinitionsResult
    maxBudgetUsd?: number
    customSystemPrompt?: string       // overrides the default system prompt
    appendSystemPrompt?: string       // appended to the end of the system prompt
    refreshTools?: () => Tools        // dynamically refresh tool list (when MCP connects)
  }

  // Control and state
  abortController: AbortController   // abort signal
  readFileState: FileStateCache       // file read cache (LRU)
  getAppState(): AppState
  setAppState(f: (prev: AppState) => AppState): void

  // UI callbacks (REPL mode)
  setToolJSX?: SetToolJSXFn          // inject custom React UI
  addNotification?: (notif) => void   // add a notification
  appendSystemMessage?: (msg) => void // add a system message to the UI
  sendOSNotification?: (opts) => void // OS-level notification (iTerm2, etc.)

  // Progress tracking
  setInProgressToolUseIDs: (f) => void
  setHasInterruptibleToolInProgress?: (v: boolean) => void
  setResponseLength: (f) => void
  setStreamMode?: (mode: SpinnerMode) => void
  onCompactProgress?: (event) => void

  // Agent-related
  agentId?: AgentId                  // sub-agent identifier
  queryTracking?: QueryChainTracking  // query chain tracking
  contentReplacementState?: ContentReplacementState  // tool result offload state

  // Memory system
  nestedMemoryAttachmentTriggers?: Set<string>
  loadedNestedMemoryPaths?: Set<string>
  discoveredSkillNames?: Set<string>

  // File history
  updateFileHistoryState: (f) => void

  // Permissions
  messages: Message[]                // current context messages (readable by tools)
}
```

---

## 5. Tool Lifecycle

```
Model produces a tool_use block
    │
    ▼
1. findToolByName(tools, block.name)
   └─ locate tool (including MCP tools mcp__server__tool)
    │
    ▼
2. tool.isEnabled()
   └─ false → return error message
    │
    ▼
3. Permission check pipeline (7 layers, see Chapter 13)
   └─ deny → return error | ask → wait for user | allow → proceed
    │
    ▼
4. tool.call(args, context, canUseTool, parentMessage, onProgress)
   ├─ execute tool logic
   ├─ onProgress callback → yield ProgressMessage (streaming UI update)
   └─ return ToolResult
    │
    ▼
5. Result processing
   ├─ exceeds maxResultSizeChars → persist to disk → return preview
   ├─ ToolResult.type === 'result' → wrap as tool_result block
   └─ ToolResult.type === 'error' → mark is_error: true
    │
    ▼
6. yield UserMessage (containing tool_result)
   └─ appended to messages, sent on the next API call
```

---

## 6. Tool Result Size Limits (maxResultSizeChars)

```typescript
// When the limit is exceeded, the result is offloaded to disk
// The tool receives: "Result saved to /path/to/file (N characters total)"

// Limits by tool:
FileReadTool.maxResultSizeChars = Infinity   // never offload
GrepTool.maxResultSizeChars = 200_000        // ~200K characters
BashTool.maxResultSizeChars = 100_000
WebFetchTool.maxResultSizeChars = 50_000
// ...

// After offloading, the model can use the Read tool with the file path to retrieve the full result
```

---

## 7. Concurrency Safety (isConcurrencySafe)

```typescript
// StreamingToolExecutor executes tools while the stream is still incoming
// isConcurrencySafe = true → can be executed in parallel with other tools

FileReadTool.isConcurrencySafe = () => true    // read-only, safe to parallelize
GlobTool.isConcurrencySafe = () => true
GrepTool.isConcurrencySafe = () => true
BashTool.isConcurrencySafe = () => false        // may have side effects
FileWriteTool.isConcurrencySafe = () => false
AgentTool.isConcurrencySafe = () => false
```

---

## 8. Lazy Loading (Deferred Tools)

When the `ToolSearch` feature is enabled, some tools do not appear in the initial prompt — the model must first call `ToolSearch` to access them:

```typescript
// Deferred tools: shouldDefer = true
// In the first-turn prompt, only the tool name and a one-line description appear
// Model calls ToolSearch("keyword") → receives full schema

// Never deferred: alwaysLoad = true (e.g., Bash, Read, Write)
// MCP tools can set _meta['anthropic/alwaysLoad'] = true
```

---

## Key File Reference

| File | Purpose |
|------|---------|
| `src/Tool.ts` | Tool interface, ToolUseContext, buildTool() |
| `src/tools/BashTool/` | Bash execution, dangerous command detection, sleep detection |
| `src/tools/FileEditTool/` | Precise string replacement (Edit tool) |
| `src/tools/AgentTool/AgentTool.tsx` | Sub-agent launch logic |
| `src/tools/AgentTool/forkSubagent.ts` | Fork mechanism + Prompt Cache optimization |
| `src/tools/WebFetchTool/` | Web page fetching + AI processing |
| `src/tools/shared/` | Shared utility functions for tools |
| `src/services/tools/toolOrchestration.ts` | Serial tool execution orchestration |
| `src/services/tools/StreamingToolExecutor.ts` | Parallel streaming tool executor |
| `src/utils/toolResultStorage.ts` | Tool result persistence (offloading large results) |
