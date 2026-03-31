# Learn Claude Code — 源码深度解析

> 基于 [instructkr/claude-code](https://github.com/instructkr/claude-code) 泄露源码的完整技术文档
>
> 目标读者：想理解 Claude Code 内部原理的开发者、想自己实现类似工具的工程师

---

## 目录

1. [整体架构概览](#1-整体架构概览)
2. [核心：Agent Loop（query.ts）](#2-核心agent-loopqueryts)
3. [工具系统（Tool System）](#3-工具系统tool-system)
4. [权限系统（Permission System）](#4-权限系统permission-system)
5. [Context 压缩（Compaction）](#5-context-压缩compaction)
6. [API 层](#6-api-层)
7. [REPL 界面层](#7-repl-界面层)
8. [启动流程](#8-启动流程)

---

## 1. 整体架构概览

Claude Code 是一个 **TypeScript + React（Ink）** 构建的终端 AI Agent，代码量约 1332 个 TS 文件。核心数据流如下：

```
用户输入（REPL）
      │
      ▼
  query()          ← 核心 Agent Loop（query.ts, 1729行）
      │
      ├─ 构造 messages + system prompt
      ├─ [上下文压缩] microcompact / autocompact / snip
      │
      ▼
  callModel()      ← 调用 Anthropic API（流式）
      │
      ▼
  收集 stream      ← AssistantMessage + tool_use blocks
      │
      ├─ 有 tool_use？
      │     │
      │     ├─ YES → StreamingToolExecutor 并行执行工具
      │     │         → 追加 tool_result → 继续循环
      │     │
      │     └─ NO  → 返回 Terminal，循环结束
      │
      ▼
  REPL 渲染输出
```

### 技术栈

| 层次 | 技术 |
|------|------|
| 语言 | TypeScript (Bun 运行时) |
| CLI 渲染 | React + [Ink](https://github.com/vadimdemedes/ink)（终端版 React） |
| API | `@anthropic-ai/sdk`，支持直连 / Bedrock / Azure / Vertex |
| Schema 验证 | Zod v4 |
| 构建 | Bun bundle（`feature()` 做 dead-code elimination） |
| 测试 | Vitest |

---

## 2. 核心：Agent Loop（query.ts）

> 文件：`src/query.ts`（1729 行）

这是整个项目最重要的文件。它是一个 **AsyncGenerator**，驱动"调用模型→执行工具→再调用模型"的循环。

### 2.1 入口函数签名

```typescript
export async function* query(
  params: QueryParams,
): AsyncGenerator<
  StreamEvent | RequestStartEvent | Message | TombstoneMessage | ToolUseSummaryMessage,
  Terminal   // 返回值：循环终止原因
>
```

**QueryParams** 关键字段：

```typescript
type QueryParams = {
  messages: Message[]                // 当前对话历史
  systemPrompt: SystemPrompt         // 系统提示词
  userContext: { [k: string]: string }
  systemContext: { [k: string]: string }
  canUseTool: CanUseToolFn           // 权限检查函数
  toolUseContext: ToolUseContext      // 工具执行上下文（含 tools 列表、appState 等）
  fallbackModel?: string
  querySource: QuerySource
  maxTurns?: number
  taskBudget?: { total: number }     // Token 预算（Swarm 模式用）
}
```

### 2.2 内部状态

每一轮循环携带一个 `State` 对象：

```typescript
type State = {
  messages: Message[]
  toolUseContext: ToolUseContext
  autoCompactTracking: AutoCompactTrackingState | undefined
  maxOutputTokensRecoveryCount: number  // 最多重试3次
  hasAttemptedReactiveCompact: boolean
  maxOutputTokensOverride: number | undefined
  pendingToolUseSummary: Promise<...> | undefined
  stopHookActive: boolean | undefined
  turnCount: number
  transition: Continue | undefined   // 上一轮为何继续
}
```

### 2.3 单次循环的完整流程

```
while (true) {
  1. yield { type: 'stream_request_start' }          // 通知 UI 开始请求

  2. 更新 queryTracking（chainId + depth，用于分析）

  3. messagesForQuery = getMessagesAfterCompactBoundary(messages)
                                                     // 只取压缩边界之后的消息

  4. applyToolResultBudget(...)                      // 对超大 tool_result 截断

  5. [HISTORY_SNIP] snipCompactIfNeeded(...)         // 历史裁剪

  6. microcompact(...)                               // 微型压缩（清理旧工具结果）

  7. [CONTEXT_COLLAPSE] applyCollapsesIfNeeded(...)  // 折叠旧对话

  8. autocompact(...)                                // 自动完整压缩（阈值触发）

  9. 检查 isAtBlockingLimit → 超限则返回 'blocking_limit'

  10. for await (message of deps.callModel(...)) {   // 流式 API 调用
        if message.type === 'assistant':
          - 收集 assistantMessages[]
          - 提取 toolUseBlocks[]
          - 喂给 streamingToolExecutor（边流边执行工具）

        yield message                                // 实时推给 UI
      }

  11. 汇总所有 tool results（来自 streamingToolExecutor）

  12. 有 tool_use？
       YES → messages.push(assistant + tool_results)，continue
       NO  → return { reason: 'stop_sequence' | 'end_turn' | ... }
}
```

### 2.4 错误恢复机制

| 错误类型 | 处理方式 |
|----------|----------|
| `max_output_tokens` | 提升 maxOutputTokensOverride，最多重试 3 次 |
| `prompt_too_long` | 触发 reactive compact，再重试 |
| 模型切换（fallback） | 用 Tombstone 标记废弃消息，清空重来 |
| 图片过大 | 触发 media recovery compact |

**Tombstone 机制**：当流式中途发生 fallback 时，已 yield 出去的 AssistantMessage 无法撤回，于是额外 yield 一条 `{ type: 'tombstone', message }` 通知 UI 抹掉它：

```typescript
for (const msg of assistantMessages) {
  yield { type: 'tombstone' as const, message: msg }
}
assistantMessages.length = 0
```

### 2.5 并行工具执行：StreamingToolExecutor

这是一个关键优化。工具执行**不等待 API 流结束**，而是在模型还在生成文字时就开始并行执行：

```
模型流：[text block] → [tool_use: Bash "ls"] → [text] → [tool_use: Read "foo.ts"] → [stop]
                               ↓ 立刻开始执行                        ↓ 立刻开始执行
工具执行：         Bash 在后台运行 ─────────────────┐   Read 在后台运行 ──┐
                                                   ↓                    ↓
收集结果：                              等流结束后统一收集两个 tool_result
```

```typescript
// 每收到一个 tool_use block 就立刻提交
streamingToolExecutor.addTool(toolBlock, message)

// 同时轮询已完成的结果并 yield
for (const result of streamingToolExecutor.getCompletedResults()) {
  yield result.message
}
```

---

## 3. 工具系统（Tool System）

> 文件：`src/Tool.ts`，`src/tools/*/`

### 3.1 Tool 接口

所有工具都实现这个类型（节选核心字段）：

```typescript
type Tool<Input, Output, P extends ToolProgressData> = {
  // 基本信息
  name: string
  aliases?: string[]

  // 核心执行
  call(
    args: z.infer<Input>,
    context: ToolUseContext,
    canUseTool: CanUseToolFn,
    parentMessage: AssistantMessage,
    onProgress?: ToolCallProgress<P>,
  ): Promise<ToolResult<Output>>

  // 给模型看的描述（动态生成，会根据权限模式变化）
  prompt(options): Promise<string>
  description(input, options): Promise<string>

  // Schema（Zod 优先，也支持直接 JSON Schema）
  inputSchema: Input               // Zod schema → 自动转 JSON Schema 给 API
  inputJSONSchema?: ToolInputJSONSchema

  // 权限相关
  checkPermissions(input, context): Promise<PermissionResult>
  isReadOnly(input): boolean
  isDestructive?(input): boolean
  isConcurrencySafe(input): boolean   // 是否可以并行

  // UI 渲染（React/Ink）
  renderToolUseMessage(input, options): React.ReactNode
  renderToolResultMessage(output, progress, options): React.ReactNode
  renderToolUseProgressMessage(progress, options): React.ReactNode

  // 分类器输入（auto 模式安全检查用）
  toAutoClassifierInput(input): unknown

  // 工具结果 → API 格式
  mapToolResultToToolResultBlockParam(content, toolUseID): ToolResultBlockParam
}
```

### 3.2 buildTool 工厂函数

不需要手写全部字段，用 `buildTool()` 填充安全默认值：

```typescript
const TOOL_DEFAULTS = {
  isEnabled:          () => true,
  isConcurrencySafe:  () => false,   // fail-closed：默认不并行
  isReadOnly:         () => false,   // fail-closed：默认认为有写操作
  isDestructive:      () => false,
  checkPermissions:   (input) => Promise.resolve({ behavior: 'allow', updatedInput: input }),
  toAutoClassifierInput: () => '',   // 空字符串 = 跳过分类器
  userFacingName:     () => def.name,
}

export function buildTool<D extends AnyToolDef>(def: D): BuiltTool<D> {
  return { ...TOOL_DEFAULTS, ...def } as BuiltTool<D>
}
```

### 3.3 工具完整列表（44+）

**执行类**
| 工具 | 功能 |
|------|------|
| `BashTool` | Shell 命令执行，带超时、沙箱检测 |
| `PowerShellTool` | Windows PowerShell（Windows only） |
| `REPLTool` | Python/Node REPL 持久会话 |
| `MCPTool` | 动态 MCP 工具调用 |
| `SkillTool` | 斜杠命令（/skill）执行 |
| `AgentTool` | 启动子 Agent（Swarm 模式） |

**文件 I/O 类**
| 工具 | 功能 |
|------|------|
| `FileReadTool` | 读文件（支持图片、PDF、Notebook） |
| `FileWriteTool` | 写文件 |
| `FileEditTool` | 精确字符串替换编辑 |
| `GlobTool` | 文件模式匹配 |
| `GrepTool` | 内容搜索（基于 ripgrep） |
| `NotebookEditTool` | Jupyter Notebook 编辑 |

**信息检索类**
| 工具 | 功能 |
|------|------|
| `WebSearchTool` | 网络搜索 |
| `WebFetchTool` | 抓取 URL 内容 |
| `ListMcpResourcesTool` | 枚举 MCP 资源 |
| `ReadMcpResourceTool` | 读取 MCP 资源 |

**任务管理类**（`TaskCreateTool`、`TaskListTool`、`TaskGetTool`、`TaskUpdateTool`、`TaskOutputTool`、`TaskStopTool`）

**交互类**（`AskUserQuestionTool`、`SendMessageTool`、`TodoWriteTool`）

**工作区类**（`EnterWorktreeTool`、`ExitWorktreeTool`、`EnterPlanModeTool`、`ExitPlanModeTool`）

**其他**（`SleepTool`、`ConfigTool`、`ScheduleCronTool`、`ToolSearchTool`、`McpAuthTool`、`TeamCreateTool`、`TeamDeleteTool`…）

### 3.4 以 BashTool 为例

```typescript
// src/tools/BashTool/BashTool.tsx（节选）
export const BashTool = buildTool({
  name: 'Bash',

  inputSchema: z.object({
    command: z.string(),
    timeout: z.number().optional(),      // 默认 120s，最大 600s
    description: z.string().optional(),  // 给用户看的说明
  }),

  isReadOnly: (input) => isSearchOrReadBashCommand(input.command),
  isConcurrencySafe: (input) => isSearchOrReadBashCommand(input.command),

  checkPermissions: async (input, context) => {
    // 检查是否命中 alwaysDeny 规则
    // 检查是否命中 alwaysAllow 规则
    // 否则返回 'ask'
  },

  async call(args, context, canUseTool, parentMessage, onProgress) {
    // 1. 检查是否 abort
    // 2. 阻止危险命令（如 fork bomb、rm -rf /）
    // 3. 执行 runShellCommand()（async generator，边执行边 yield 进度）
    // 4. 返回 ToolResult<string>
  },

  renderToolUseMessage: (input) => (
    <BashToolUseMessage command={input.command} />
  ),
})
```

### 3.5 ToolUseContext：工具的"运行时环境"

每次工具调用都携带这个大对象：

```typescript
type ToolUseContext = {
  options: {
    commands: Command[]
    tools: Tools
    mainLoopModel: string
    mcpClients: MCPServerConnection[]
    agentDefinitions: AgentDefinitionsResult
    // ...
  }
  abortController: AbortController   // 用于取消
  getAppState(): AppState
  setAppState(f): void
  setToolJSX?: SetToolJSXFn          // 工具可以注入自定义 UI
  addNotification?: (n) => void
  messages: Message[]                // 当前对话（工具可以读取历史）
  queryTracking?: QueryChainTracking // 链式调用追踪
  agentId?: AgentId                  // Swarm 中的 Agent 标识
  // ...
}
```

---

## 4. 权限系统（Permission System）

> 文件：`src/hooks/toolPermission/`，`src/types/permissions.ts`

### 4.1 三种权限模式

```typescript
type PermissionMode =
  | 'default'    // 每次询问用户（标准模式）
  | 'bypass'     // 全部自动通过（--dangerously-skip-permissions）
  | 'plan'       // 计划模式：只读操作自动通过
  | 'auto'       // ML 分类器决定
```

### 4.2 决策链（Pipeline）

```
tool.call() 发起请求
      │
      ▼
canUseTool()    ← 门控函数，由 query.ts 传入
      │
      ├─ 1. validateInput()          工具自己的输入校验
      │
      ├─ 2. tool.checkPermissions()  工具层规则（e.g. Bash 的 always-deny 模式）
      │
      ├─ 3. alwaysDenyRules          全局规则：永远拒绝（e.g. "rm -rf *"）
      │
      ├─ 4. alwaysAllowRules         全局规则：永远允许（e.g. "git status"）
      │
      ├─ 5. [auto mode] ML 分类器    根据 toAutoClassifierInput() 判断风险
      │
      ├─ 6. shouldAvoidPrompts？      非交互模式 → 自动决策，不弹框
      │
      └─ 7. 弹出用户确认框            等待用户 Allow / Deny / Always Allow
```

### 4.3 PermissionResult 类型

```typescript
type PermissionResult =
  | { behavior: 'allow'; updatedInput?: Record<string, unknown> }
  | { behavior: 'deny';  message: string }
  | { behavior: 'ask';   message?: string }
```

### 4.4 规则格式

权限规则存储在 `settings.json` 的 `permissions` 字段：

```json
{
  "permissions": {
    "allow": ["Bash(git *)", "Read(*)", "Bash(npm run *)"],
    "deny":  ["Bash(rm -rf *)", "Bash(curl * | bash)"]
  }
}
```

规则语法：`ToolName(pattern)`，支持 glob 通配符。

### 4.5 preparePermissionMatcher

每个工具可以定义自己的模式匹配逻辑：

```typescript
// BashTool 示例
preparePermissionMatcher: async (input) => {
  return (pattern: string) => {
    // pattern = "git *"，input.command = "git status"
    return micromatch.isMatch(input.command, pattern)
  }
}
```

---

## 5. Context 压缩（Compaction）

> 文件：`src/services/compact/`

这是防止对话超出 token 上限的多层防御系统。

### 5.1 四层压缩策略（按触发顺序）

```
每次 API 调用前执行：

  Layer 1: History Snip（HISTORY_SNIP feature flag）
           裁剪历史，释放 tokens，同时 yield 一个 boundary marker
           ↓
  Layer 2: Microcompact（每轮都运行）
           清除最近几轮中"已不重要"的工具结果
           （只留 summary，不删除对话逻辑）
           ↓
  Layer 3: Context Collapse（CONTEXT_COLLAPSE feature flag）
           把较早的对话轮次折叠成摘要
           ↓
  Layer 4: Autocompact（阈值触发）
           tokens 超限时，用模型重写整个对话为一份详细摘要
```

### 5.2 Token 阈值计算

```typescript
// 有效上下文窗口 = 总窗口 - 为输出预留的 token
function getEffectiveContextWindowSize(model: string): number {
  const reservedForOutput = Math.min(
    getMaxOutputTokensForModel(model),
    20_000,   // 最多预留 2 万 token 给输出
  )
  return getContextWindowForModel(model) - reservedForOutput
}

// 各阈值
const AUTOCOMPACT_THRESHOLD  = effectiveWindow - 13_000  // 触发自动压缩
const WARNING_THRESHOLD      = effectiveWindow - 20_000  // 显示警告
const BLOCKING_LIMIT         = effectiveWindow - 3_000   // 硬性阻断（手动压缩才用）
```

支持环境变量覆盖：
- `CLAUDE_CODE_AUTO_COMPACT_WINDOW` — 覆盖上下文窗口大小
- `CLAUDE_AUTOCOMPACT_PCT_OVERRIDE` — 用百分比设置压缩阈值
- `DISABLE_COMPACT` — 完全禁用压缩
- `DISABLE_AUTO_COMPACT` — 只禁用自动压缩

### 5.3 Microcompact 可压缩的工具

```typescript
const COMPACTABLE_TOOLS = new Set([
  'Read',         // 文件读取结果（往往很大）
  'Bash',         // Shell 输出
  'Grep',         // 搜索结果
  'Glob',         // 文件列表
  'WebSearch',    // 搜索结果
  'WebFetch',     // 网页内容
  'Write',        // 写文件确认
  'Edit',         // 编辑确认
])
```

逻辑：如果这些工具的结果在"近期对话"中没有被明确引用，就把结果替换成简短摘要。

### 5.4 Autocompact：对话重写

触发条件：`tokenCount(messages) > AUTOCOMPACT_THRESHOLD`

执行步骤：
1. 把当前完整对话发给另一个模型调用（或同一模型）
2. 使用固定的 prompt 模板（见下）让模型生成详细摘要
3. 用摘要替换整个对话历史
4. 在消息列表里插入 `CompactBoundaryMarker`
5. 后续的 `getMessagesAfterCompactBoundary()` 只看 boundary 之后的内容

**压缩 prompt 模板（精髓）：**

```
你的任务是为到目前为止的对话创建一份详细摘要，格式包含：

1. Primary Request and Intent：用户的所有明确请求
2. Key Technical Concepts：讨论过的技术概念、框架
3. Files and Code Sections：检查/修改/创建的文件，包含完整代码片段
4. Errors and fixes：遇到的错误和修复方式
5. Problem Solving：已解决的问题和正在进行的排查
6. All user messages：所有非工具结果的用户消息（关键！保留意图）
7. Pending Tasks：被明确要求但尚未完成的任务
8. Current Work：最近正在做的事（最重要）
9. Optional Next Step：下一步（必须与用户最近的请求一致）
```

### 5.5 熔断器（Circuit Breaker）

```typescript
const MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES = 3

// 连续失败 3 次后，停止自动压缩（防止无限 API 调用）
if (tracking?.consecutiveFailures >= 3) {
  return { wasCompacted: false }
}
```

---

## 6. API 层

> 文件：`src/services/api/`，`src/services/claude.ts`

### 6.1 支持的后端

```typescript
// 根据环境变量自动选择客户端
// 1. 直接 API：ANTHROPIC_API_KEY
// 2. AWS Bedrock：AWS credentials + AWS_REGION
// 3. Azure Foundry：ANTHROPIC_FOUNDRY_RESOURCE + ANTHROPIC_FOUNDRY_API_KEY
// 4. Vertex AI：ANTHROPIC_VERTEX_PROJECT_ID + GCP credentials
const client = await getAnthropicClient({ apiKey, model, source })
```

### 6.2 Prompt Caching

Claude Code 大量使用 prompt caching 降低成本：

```typescript
function getCacheControl({ scope, querySource } = {}) {
  return {
    type: 'ephemeral',
    ...(should1hCacheTTL(querySource) && { ttl: '1h' }),  // 某些场景用 1 小时 TTL
    ...(scope === 'global' && { scope }),
  }
}
```

缓存控制位置：
- `system prompt` → 加 cache_control（每次请求不变的部分）
- 较早的 `user messages` → 加 cache_control
- 最近 2 轮消息 → 不缓存（频繁变化）

### 6.3 流式处理

```typescript
// deps.callModel() 是一个 AsyncGenerator
for await (const message of deps.callModel({ messages, systemPrompt, tools, ... })) {
  // message 类型：
  // - AssistantMessage（文本 + tool_use blocks）
  // - 错误消息（api_error、max_output_tokens 等）
}
```

### 6.4 工具定义格式（发给 API 的）

```typescript
// 每个 Tool 的 inputSchema（Zod）会被转成：
{
  name: "Bash",
  description: "Execute a shell command...",
  input_schema: {
    type: "object",
    properties: {
      command: { type: "string", description: "..." },
      timeout: { type: "number", description: "..." },
    },
    required: ["command"]
  }
}
```

---

## 7. REPL 界面层

> 文件：`src/screens/REPL.tsx`（~900 行）

### 7.1 技术：React + Ink

Ink 让你用 React 写终端 UI：

```typescript
// 不是 <div>，而是终端组件
import { Box, Text, useInput } from 'ink'

function REPL() {
  return (
    <Box flexDirection="column">
      <MessageList messages={messages} />
      <ToolUseDisplay toolUse={currentToolUse} />
      <InputBox onSubmit={handleSubmit} />
    </Box>
  )
}
```

### 7.2 流式渲染

REPL 消费 `query()` 的 AsyncGenerator yield 出来的事件：

```typescript
for await (const event of query(params)) {
  switch (event.type) {
    case 'stream_request_start':
      showSpinner()
      break
    case 'assistant':
      appendText(event.message)  // 实时显示生成文字
      break
    case 'user':  // tool_result
      showToolResult(event)
      break
    case 'tombstone':
      removeMessage(event.message)  // 抹掉失败的消息
      break
  }
}
```

### 7.3 setToolJSX 机制

工具可以向 UI 注入自定义 React 组件：

```typescript
// 工具执行时
context.setToolJSX?.({
  jsx: <InteractiveFileTree files={files} />,
  shouldHidePromptInput: true,  // 暂时隐藏输入框
})

// 工具完成后
context.setToolJSX?.(null)  // 清除
```

这样 `AskUserQuestionTool` 才能弹出交互式选择框，`EnterPlanModeTool` 才能展示计划视图。

---

## 8. 启动流程

> 文件：`src/entrypoints/cli.tsx` → `src/entrypoints/init.ts` → `src/main.tsx`

### 8.1 快速路径（Fast Paths）

`cli.tsx` 首先检查特殊参数，走对应的轻量路径（避免加载全部模块）：

```
--version              → 直接打印版本，退出
--dump-system-prompt   → 渲染 system prompt，退出
--daemon-worker=<kind> → 启动 daemon worker（不带 analytics）
remote-control / rc    → 启动远程控制 bridge
daemon                 → 启动长期运行的 supervisor
ps/logs/attach/kill    → Session 管理命令
new/list/reply         → Template jobs
                       ↓（都不是）
动态 import main.tsx   → 加载完整应用
```

### 8.2 并行预加载

`main.tsx` 在模块加载时就立刻启动一些慢操作（并行，不阻塞主流程）：

```typescript
// 并行启动（副作用，不等结果）
startMdmRawRead()         // MDM 企业策略读取（macOS: plutil，Windows: reg）
startKeychainPrefetch()   // 预加载 OAuth token 和 API key（避免后面串行等待 65ms+）
```

### 8.3 初始化顺序（init.ts）

```
1. 解析 CLI 参数
2. 读取 config（~/.claude/settings.json + 项目 .claude/settings.json）
3. 鉴权（OAuth token / API key / Bedrock / Vertex）
4. 网络检查
5. Telemetry 初始化（GrowthBook feature flags）
6. 加载 MCP servers
7. 加载工具列表（tools.ts）
8. 启动 REPL
```

---

## 附录：关键文件速查

| 文件 | 作用 |
|------|------|
| `src/query.ts` | ⭐ 核心 Agent Loop |
| `src/Tool.ts` | Tool 接口定义 + buildTool 工厂 |
| `src/tools/BashTool/` | 最复杂的工具实现，可作为学习范本 |
| `src/hooks/toolPermission/` | 权限系统 |
| `src/services/compact/autoCompact.ts` | 自动压缩阈值计算 |
| `src/services/compact/compact.ts` | 压缩执行 + prompt 模板 |
| `src/services/compact/microCompact.ts` | 微型压缩 |
| `src/services/api/claude.ts` | API 调用层 |
| `src/screens/REPL.tsx` | 终端 UI |
| `src/entrypoints/cli.tsx` | CLI 入口，快速路径分发 |
| `src/main.tsx` | 应用初始化 |
| `src/types/message.ts` | 消息类型定义（关键数据结构） |
| `src/types/permissions.ts` | 权限类型定义 |

---

*文档基于 instructkr/claude-code 源码分析，仅供学习研究。*
