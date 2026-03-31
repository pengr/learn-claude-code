# 02 — 工具系统：44 个工具的完整图谱

> 文件：`src/Tool.ts`，`src/tools/`（42 个工具目录/文件）

---

## 1. 核心接口：Tool\<Input, Output\>

每个工具是一个实现了 `Tool` 接口的对象，通过 `buildTool()` 工厂函数构造：

```typescript
// src/Tool.ts

export type Tool<
  Input extends AnyObject = AnyObject,
  Output = unknown,
  P extends ToolProgressData = ToolProgressData,
> = {
  readonly name: string
  readonly aliases?: string[]           // 兼容旧名称（如工具重命名时）
  readonly inputSchema: Input           // Zod schema（自动生成 JSON Schema 传给 API）
  readonly inputJSONSchema?: ToolInputJSONSchema  // MCP 工具用（直接 JSON Schema）
  readonly shouldDefer?: boolean        // 是否延迟加载（需要 ToolSearch 才能访问）
  readonly alwaysLoad?: boolean         // 永不延迟（首轮就出现在 prompt 中）
  readonly maxResultSizeChars: number   // 超出则持久化到磁盘
  readonly strict?: boolean            // 严格模式（API 更严格遵循 schema）
  readonly isMcp?: boolean             // 是 MCP 工具
  readonly isLsp?: boolean             // 是 LSP 工具
  readonly searchHint?: string         // ToolSearch 关键词（3-10 词）
  outputSchema?: z.ZodType<unknown>

  // 核心方法
  call(args, context, canUseTool, parentMessage, onProgress?): Promise<ToolResult<Output>>
  description(input, options): Promise<string>

  // 权限相关
  isEnabled(): boolean
  isReadOnly(input): boolean
  isDestructive?(input): boolean
  isConcurrencySafe(input): boolean
  checkPermissions(input, context?): Promise<PermissionResult>
  toAutoClassifierInput(input): unknown     // 返回 '' 则跳过 ML 分类器

  // 交互行为
  interruptBehavior?(): 'cancel' | 'block'  // 用户输入时是取消还是等待
  requiresUserInteraction?(): boolean
  isSearchOrReadCommand?(input): { isSearch: boolean; isRead: boolean; isList?: boolean }
  isOpenWorld?(input): boolean

  // 输入处理
  backfillObservableInput?(input): void     // 填充 legacy 字段（不修改 API 内容）
  inputsEquivalent?(a, b): boolean          // 判断两个输入是否等价

  userFacingName(input): string             // UI 显示名称
  mcpInfo?: { serverName: string; toolName: string }  // MCP 元数据
}
```

---

## 2. buildTool() 工厂函数

```typescript
// src/Tool.ts

// 安全默认值（fail-closed）
const TOOL_DEFAULTS = {
  isEnabled: () => true,
  isConcurrencySafe: () => false,      // 默认：不安全（不允许并发）
  isReadOnly: () => false,             // 默认：写操作
  isDestructive: () => false,
  checkPermissions: (input) =>
    Promise.resolve({ behavior: 'allow', updatedInput: input }),  // 交给通用权限系统
  toAutoClassifierInput: () => '',     // 默认：跳过 ML 分类器
  userFacingName: () => '',
}

export function buildTool<D extends AnyToolDef>(def: D): BuiltTool<D> {
  return { ...TOOL_DEFAULTS, ...def }
  // BuiltTool<D> 在类型层面精确表示 spread 的结果
  // 每个字段：如果 def 提供了 → 用 def 的；否则 → 用 TOOL_DEFAULTS 的
}
```

**使用示例（FileReadTool 节选）：**

```typescript
export const FileReadTool = buildTool({
  name: FILE_READ_TOOL_NAME,  // 'Read'
  maxResultSizeChars: Infinity,    // 不外置（避免 Read→file→Read 循环）
  isReadOnly: () => true,
  isConcurrencySafe: () => true,   // 文件读取可并发
  isSearchOrReadCommand: () => ({ isSearch: false, isRead: true }),

  inputSchema: z.object({
    file_path: z.string(),
    limit: z.number().optional(),
    offset: z.number().optional(),
  }),

  async call(args, context, canUseTool, parentMessage) {
    // 读取文件，处理权限，返回内容
    return { type: 'result', resultForAssistant: content }
  },

  async description(input) {
    return `Read ${input.file_path}`
  },
})
```

---

## 3. 44 个工具完整清单

### 3.1 文件操作工具

| 工具名 | 目录 | 功能 |
|--------|------|------|
| `Read` | `FileReadTool/` | 读取文件，支持行范围、PDF、图片 |
| `Write` | `FileWriteTool/` | 写入文件（全量覆盖）|
| `Edit` | `FileEditTool/` | 精确字符串替换（old_string → new_string）|
| `Glob` | `GlobTool/` | 文件路径模式匹配，按修改时间排序 |
| `Grep` | `GrepTool/` | 基于 ripgrep 的内容搜索 |
| `NotebookEdit` | `NotebookEditTool/` | 编辑 Jupyter Notebook 单元格 |

### 3.2 Shell 执行工具

| 工具名 | 目录 | 功能 |
|--------|------|------|
| `Bash` | `BashTool/` | 执行 bash 命令（120s 超时，流式输出）|
| `PowerShell` | `PowerShellTool/` | Windows PowerShell 命令 |
| `REPL` | `REPLTool/` | Node.js REPL（跨调用持久 VM 上下文）|

### 3.3 Agent/协调工具

| 工具名 | 目录 | 功能 |
|--------|------|------|
| `Agent` | `AgentTool/` | 启动子 Agent（同步/异步/后台/团队）|
| `SendMessage` | `SendMessageTool/` | 向指定 Agent 或用户发消息 |
| `AskUserQuestion` | `AskUserQuestionTool/` | 向用户提问（单选/多选/预览）|
| `TeamCreate` | `TeamCreateTool/` | 创建 Agent 团队（split-pane UI）|
| `TeamDelete` | `TeamDeleteTool/` | 删除 Agent 团队 |
| `RemoteTrigger` | `RemoteTriggerTool/` | 触发远程 Agent 任务 |

### 3.4 任务管理工具

| 工具名 | 目录 | 功能 |
|--------|------|------|
| `TaskCreate` | `TaskCreateTool/` | 创建任务（subject + description）|
| `TaskUpdate` | `TaskUpdateTool/` | 更新任务状态/内容 |
| `TaskList` | `TaskListTool/` | 列出所有任务 |
| `TaskGet` | `TaskGetTool/` | 获取任务详情 |
| `TaskOutput` | `TaskOutputTool/` | 获取后台任务输出 |
| `TaskStop` | `TaskStopTool/` | 停止后台任务 |

### 3.5 Web 工具

| 工具名 | 目录 | 功能 |
|--------|------|------|
| `WebFetch` | `WebFetchTool/` | 抓取网页并 AI 处理内容 |
| `WebSearch` | `WebSearchTool/` | 网络搜索（返回结构化结果）|

### 3.6 MCP 工具

| 工具名 | 目录 | 功能 |
|--------|------|------|
| `MCPTool` | `MCPTool/` | 动态 MCP 工具调用（`mcp__<server>__<tool>`）|
| `McpAuth` | `McpAuthTool/` | MCP OAuth 认证处理 |
| `ListMcpResources` | `ListMcpResourcesTool/` | 列出 MCP 资源 |
| `ReadMcpResource` | `ReadMcpResourceTool/` | 读取 MCP 资源 |

### 3.7 UI/交互工具

| 工具名 | 目录 | 功能 |
|--------|------|------|
| `Config` | `ConfigTool/` | 读写 Claude Code 配置 |
| `EnterPlanMode` | `EnterPlanModeTool/` | 进入计划模式 |
| `ExitPlanMode` | `ExitPlanModeTool/` | 退出计划模式（等待用户审批）|
| `EnterWorktree` | `EnterWorktreeTool/` | 创建并切换 git worktree |
| `ExitWorktree` | `ExitWorktreeTool/` | 退出 git worktree |
| `TodoWrite` | `TodoWriteTool/` | 写入 Todo 列表（内存中）|
| `Skill` | `SkillTool/` | 执行 Skill（加载 Markdown 指令集）|
| `ToolSearch` | `ToolSearchTool/` | 搜索可用工具（延迟加载入口）|

### 3.8 辅助工具

| 工具名 | 目录 | 功能 |
|--------|------|------|
| `Sleep` | `SleepTool/` | 等待指定时间（带倒计时 UI）|
| `Brief` | `BriefTool/` | 生成简洁摘要 |
| `LSPTool` | `LSPTool/` | LSP 语言服务器调用 |
| `ScheduleCron` | `ScheduleCronTool/` | 创建定时任务（CronCreate）|
| `SyntheticOutput` | `SyntheticOutputTool/` | 生成合成输出（测试用）|

---

## 4. ToolUseContext：执行上下文

`ToolUseContext` 是每次工具调用时传入的"执行环境"，包含了工具需要的所有外部依赖：

```typescript
// src/Tool.ts（节选关键字段）

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
    customSystemPrompt?: string       // 覆盖默认系统提示
    appendSystemPrompt?: string       // 追加到系统提示末尾
    refreshTools?: () => Tools        // 动态刷新工具列表（MCP 连接时）
  }

  // 控制与状态
  abortController: AbortController   // 中断信号
  readFileState: FileStateCache       // 文件读取缓存（LRU）
  getAppState(): AppState
  setAppState(f: (prev: AppState) => AppState): void

  // UI 回调（REPL 模式）
  setToolJSX?: SetToolJSXFn          // 注入自定义 React UI
  addNotification?: (notif) => void   // 添加通知
  appendSystemMessage?: (msg) => void // 添加系统消息到 UI
  sendOSNotification?: (opts) => void // 系统级通知（iTerm2 等）

  // 进度追踪
  setInProgressToolUseIDs: (f) => void
  setHasInterruptibleToolInProgress?: (v: boolean) => void
  setResponseLength: (f) => void
  setStreamMode?: (mode: SpinnerMode) => void
  onCompactProgress?: (event) => void

  // Agent 相关
  agentId?: AgentId                  // 子 Agent 标识
  queryTracking?: QueryChainTracking  // 查询链追踪
  contentReplacementState?: ContentReplacementState  // 工具结果外置状态

  // 内存系统
  nestedMemoryAttachmentTriggers?: Set<string>
  loadedNestedMemoryPaths?: Set<string>
  discoveredSkillNames?: Set<string>

  // 文件历史
  updateFileHistoryState: (f) => void

  // 权限
  messages: Message[]                // 当前上下文消息（工具可读取）
}
```

---

## 5. 工具生命周期

```
模型产出 tool_use block
    │
    ▼
1. findToolByName(tools, block.name)
   └─ 找到工具（含 MCP 工具 mcp__server__tool）
    │
    ▼
2. tool.isEnabled()
   └─ false → 返回错误消息
    │
    ▼
3. 权限检查流水线（7 层，见第 13 章）
   └─ deny → 返回错误 | ask → 等待用户 | allow → 继续
    │
    ▼
4. tool.call(args, context, canUseTool, parentMessage, onProgress)
   ├─ 执行工具逻辑
   ├─ onProgress 回调 → yield ProgressMessage（streaming UI 更新）
   └─ 返回 ToolResult
    │
    ▼
5. 结果处理
   ├─ 超出 maxResultSizeChars → 持久化到磁盘 → 返回预览
   ├─ ToolResult.type === 'result' → 包装为 tool_result block
   └─ ToolResult.type === 'error' → 标记 is_error: true
    │
    ▼
6. yield UserMessage（含 tool_result）
   └─ 追加到 messages，下一轮 API 调用时发送
```

---

## 6. 工具结果大小限制（maxResultSizeChars）

```typescript
// 超出限制时，结果外置到磁盘
// 工具收到的是："结果已保存到 /path/to/file（共 N 字符）"

// 不同工具的限制：
FileReadTool.maxResultSizeChars = Infinity   // 永不外置
GrepTool.maxResultSizeChars = 200_000        // ~200K 字符
BashTool.maxResultSizeChars = 100_000
WebFetchTool.maxResultSizeChars = 50_000
// ...

// 外置后，模型可以用 Read 工具读取文件路径来获取完整结果
```

---

## 7. 并发安全（isConcurrencySafe）

```typescript
// StreamingToolExecutor 边流式接收边执行工具
// isConcurrencySafe = true → 可以和其他工具并行执行

FileReadTool.isConcurrencySafe = () => true    // 只读，安全并发
GlobTool.isConcurrencySafe = () => true
GrepTool.isConcurrencySafe = () => true
BashTool.isConcurrencySafe = () => false        // 可能有副作用
FileWriteTool.isConcurrencySafe = () => false
AgentTool.isConcurrencySafe = () => false
```

---

## 8. 延迟加载（Deferred Tools）

当启用 `ToolSearch` 特性时，部分工具不在初始 prompt 中出现，模型需要先调用 `ToolSearch` 才能访问：

```typescript
// 延迟工具：shouldDefer = true
// 首轮 prompt 中只有工具名和一行描述
// 模型调用 ToolSearch("关键词") → 获取完整 schema

// 永不延迟：alwaysLoad = true（如 Bash、Read、Write）
// MCP 工具可以通过 _meta['anthropic/alwaysLoad'] = true 设置
```

---

## 关键文件速查

| 文件 | 功能 |
|------|------|
| `src/Tool.ts` | Tool 接口、ToolUseContext、buildTool() |
| `src/tools/BashTool/` | Bash 执行、危险命令检测、sleep 检测 |
| `src/tools/FileEditTool/` | 精确字符串替换（Edit 工具）|
| `src/tools/AgentTool/AgentTool.tsx` | 子 Agent 启动逻辑 |
| `src/tools/AgentTool/forkSubagent.ts` | Fork 机制 + Prompt Cache 优化 |
| `src/tools/WebFetchTool/` | 网页抓取 + AI 处理 |
| `src/tools/shared/` | 工具共享工具函数 |
| `src/services/tools/toolOrchestration.ts` | 串行工具执行协调 |
| `src/services/tools/StreamingToolExecutor.ts` | 并行流式工具执行器 |
| `src/utils/toolResultStorage.ts` | 工具结果持久化（大结果外置）|
