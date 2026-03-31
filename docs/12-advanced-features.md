# 12 — 高级特性：多 Agent、Bridge 远程控制、LSP 与插件

> 文件：`src/tools/AgentTool/`，`src/bridge/`，`src/services/lsp/`，`src/plugins/`，`src/coordinator/`

---

## 1. 多 Agent 协调

### 1.1 AgentTool：启动子 Agent

```typescript
// 输入 schema（节选）
z.object({
  prompt: z.string(),                         // 子 Agent 的任务描述
  subagent_type: z.string().optional(),       // 使用哪种 Agent 定义
  description: z.string().optional(),         // 用户可见的描述
  model: z.string().optional(),              // 覆盖模型
  run_in_background: z.boolean().optional(), // 异步执行（不等待结果）
  name: z.string().optional(),               // Agent 名称（团队模式用）
  team_name: z.string().optional(),          // 团队名（多 Agent 并行）
  isolation: z.enum(['worktree', 'remote']).optional(),  // 隔离模式
  mode: permissionModeSchema().optional(),   // 权限模式覆盖
})
```

**两条执行路径：**

```
有 team_name + name
    │
    ▼
spawnTeammate()          ← 在独立 split-pane 中运行（可视化）

没有 team_name + name
    │
    ├─ run_in_background = true
    │    └─ 异步 Agent（返回 agentId + outputFile）
    │
    └─ run_in_background = false（默认）
         └─ 同步子 Agent（阻塞等待结果）
```

### 1.2 Fork 子 Agent：Prompt Cache 优化

Fork 是最常用的子 Agent 模式。关键设计目标：**所有 Fork 子 Agent 共享同一个 Prompt Cache 前缀**，最大化缓存命中率。

```typescript
// src/tools/AgentTool/forkSubagent.ts

function buildForkedMessages(directive: string, parentMessage: AssistantMessage) {
  // 策略：构造字节完全相同的前缀 + 唯一的 directive 后缀

  // 1. 克隆父 Agent 的完整 assistant message（含所有 tool_use 块）
  const fullAssistantMessage = deepClone(parentMessage)

  // 2. 为所有 tool_use 构建占位符 tool_result
  //    关键：所有 Fork 使用完全相同的占位符文本
  const FORK_PLACEHOLDER_RESULT = 'Fork started — processing in background'
  const toolResultBlocks = toolUseBlocks.map(block => ({
    type: 'tool_result',
    tool_use_id: block.id,
    content: [{ type: 'text', text: FORK_PLACEHOLDER_RESULT }],
    //                                  ↑ 对所有 Fork 完全相同 → 缓存共享！
  }))

  // 3. 最后追加子 Agent 特定的 directive（这部分不同）
  const toolResultMessage = createUserMessage({
    content: [
      ...toolResultBlocks,      // 相同的占位符
      { type: 'text', text: buildChildMessage(directive) },  // 唯一的指令
    ],
  })

  // 结果结构：
  // [...历史消息, assistant(全部工具), user(占位符 + 唯一指令)]
  //  ├─ 这部分对所有 Fork 完全相同 ──────────────────────────┤
  //                                  ├─ 这部分每个 Fork 不同 ─┘
  return [fullAssistantMessage, toolResultMessage]
}
```

**Fork 防递归检测：**

```typescript
// 防止 Fork 子 Agent 再次启动 Fork（无限嵌套）
export function isInForkChild(messages: Message[]): boolean {
  return messages.some(m =>
    m.type === 'user' &&
    m.message.content.some(b =>
      b.type === 'text' && b.text.includes('<fork-boilerplate>')
    )
  )
}
```

### 1.3 工具过滤（子 Agent 不能用所有工具）

```typescript
// 所有 Agent 禁用的工具
const ALL_AGENT_DISALLOWED_TOOLS = new Set([
  'AskUserQuestion',  // 子 Agent 不能弹对话框
  'SendMessage',      // 不能发消息给用户
  // ...
])

// 自定义 Agent 额外禁用
const CUSTOM_AGENT_DISALLOWED_TOOLS = new Set([
  'EnterPlanMode',    // 自定义 Agent 不能进入计划模式
  'ExitPlanMode',
  // ...
])

// 异步 Agent 允许的工具（白名单）
const ASYNC_AGENT_ALLOWED_TOOLS = new Set([
  'Bash', 'Read', 'Write', 'Edit', 'Glob', 'Grep',
  'WebFetch', 'WebSearch',
  'TaskCreate', 'TaskUpdate', 'TaskList', 'TaskGet', 'TaskOutput', 'TaskStop',
  // 注意：不包含 AskUserQuestion 等交互工具
])
```

### 1.4 Coordinator 模式

```typescript
// 启用：CLAUDE_CODE_COORDINATOR_MODE=1
// 作用：协调器模式下，告知模型子 Worker 可以使用哪些工具

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

## 2. Bridge：远程控制协议

Bridge 系统让用户可以通过 **Web 界面或移动端**远程控制本地运行的 Claude Code 实例。

### 2.1 消息格式

```typescript
// 出站消息（Claude Code → 远程客户端）
type SDKMessage =
  | { type: 'user';      content: string; uuid: string }  // 用户消息（echo）
  | { type: 'assistant'; content: string; uuid: string }  // 助手消息
  | { type: 'tool_use';  ... }  // 工具调用通知
  | { type: 'system';    ... }  // 系统事件

// 入站消息类型
type InboundMessage =
  | SDKMessage          // 用户发来的消息（触发对话）
  | SDKControlResponse  // 对控制请求的响应（权限决定等）
  | SDKControlRequest   // 服务器发来的控制命令
```

### 2.2 服务器控制请求（`control_request`）

远程客户端可以发送控制命令来操控本地 Claude Code：

```typescript
// 支持的控制请求类型
switch (request.request.subtype) {
  case 'initialize':
    // 握手，返回当前状态（命令列表、可用模型等）

  case 'set_model':
    // 切换模型（e.g., 从 Sonnet 切换到 Opus）
    onSetModel(request.request.model)

  case 'set_max_thinking_tokens':
    // 调整思考 token 上限
    onSetMaxThinkingTokens(request.request.max_thinking_tokens)

  case 'set_permission_mode':
    // 切换权限模式（default/plan/auto/bypass）
    const result = onSetPermissionMode(request.request.mode)

  case 'interrupt':
    // 中断当前请求（相当于远程按 Ctrl+C）
    onInterrupt()
}

// 所有控制请求都有对应的 control_response 回包
// { type: 'control_response', response: { subtype: 'success' | 'error', request_id } }
```

### 2.3 UUID 去重（BoundedUUIDSet）

防止消息被重复处理（网络重传）：

```typescript
class BoundedUUIDSet {
  // 环形缓冲区：FIFO 淘汰，固定容量
  private ring: (string | undefined)[]  // 环形数组
  private set = new Set<string>()       // 快速查找
  private writeIdx = 0

  add(uuid: string) {
    // 驱逐最旧的条目，插入新 UUID
    const evicted = this.ring[this.writeIdx]
    if (evicted) this.set.delete(evicted)
    this.ring[this.writeIdx] = uuid
    this.set.add(uuid)
    this.writeIdx = (this.writeIdx + 1) % this.capacity
  }
}

// 两个独立的集合：
// recentPostedUUIDs  → 过滤自己发出的消息（echo 过滤）
// recentInboundUUIDs → 防止重复处理入站消息
```

### 2.4 仅出站模式

```typescript
// outboundOnly: true → 只发消息给远程，不接受远程控制命令
// 适用场景：用户希望在手机上观察，但不允许远程操控
if (outboundOnly && request.request.subtype !== 'initialize') {
  return error('This session is outbound-only. Enable Remote Control locally...')
}
```

---

## 3. LSP 集成

### 3.1 架构

```
Claude Code
    │
    ▼
LSPServerManager          管理所有 LSP 服务器实例
    │
    ├─ LSPServerInstance   单个服务器（如 TypeScript/Python 语言服务器）
    │    │
    │    └─ LSPClient      JSON-RPC over stdio 通信层
    │         │
    │         ▼
    │    子进程（如 tsserver, pyright, rust-analyzer）
    │
    └─ LSPTool            工具接口（模型可以调用 LSP 功能）
```

### 3.2 支持的 LSP 能力

```typescript
// 文件操作
openFile(path, content)      // textDocument/didOpen
changeFile(path, content)    // textDocument/didChange
saveFile(path)               // textDocument/didSave
closeFile(path)              // textDocument/didClose

// 查询（通过 sendRequest）
getCompletions(path, position)    // textDocument/completion
getDiagnostics(path)              // textDocument/publishDiagnostics
getDefinition(path, position)     // textDocument/definition
getReferences(path, position)     // textDocument/references
getHover(path, position)          // textDocument/hover
```

### 3.3 服务器生命周期与容错

```
stopped → starting → running → stopping → stopped
             │                      ↑
             └──── error ───────────┘（restart 恢复）

最大自动重启次数：config.maxRestarts ?? 3
错误码 -32801（内容已修改）：指数退避重试（3 次，500/1000/2000ms）
```

### 3.4 配置格式

```jsonc
// settings.json 中配置 LSP 服务器
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

## 4. 插件系统

### 4.1 插件结构

```typescript
type LoadedPlugin = {
  name: string
  manifest: PluginManifest        // name, description, version
  path: string                    // 文件路径（内置为 'builtin'）
  source: string                  // 来源标识
  isBuiltin?: boolean
  enabled?: boolean
  sha?: string                    // git commit SHA（版本锁定）

  // 插件可以提供的内容
  commandsPath?: string           // 命令文件路径
  skillsPath?: string             // 技能文件路径
  agentsPath?: string             // Agent 定义路径
  outputStylesPath?: string       // 自定义输出样式
  hooksConfig?: HooksSettings     // 钩子配置
  mcpServers?: Record<string, McpServerConfig>   // MCP 服务器
  lspServers?: Record<string, LspServerConfig>   // LSP 服务器
}
```

### 4.2 内置插件注册

```typescript
// 任何模块都可以注册内置插件
registerBuiltinPlugin({
  name: 'my-builtin-plugin',
  description: '...',
  defaultEnabled: true,
  skills: [...],         // 提供 Skill 命令
  mcpServers: {...},     // 提供 MCP 服务器
  hooks: {...},          // 提供 Hooks
  isAvailable: () => process.platform === 'darwin',  // 条件可用
})
```

### 4.3 插件启用状态

```typescript
// 优先级：用户设置 > 插件默认值 > true（默认启用）
const pluginId = `${name}@builtin`
const userSetting = settings?.enabledPlugins?.[pluginId]
const isEnabled = userSetting !== undefined
  ? userSetting === true
  : (definition.defaultEnabled ?? true)
```

### 4.4 错误类型（完整）

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

## 关键文件速查

| 文件 | 功能 |
|------|------|
| `src/tools/AgentTool/AgentTool.tsx` | AgentTool 主实现 |
| `src/tools/AgentTool/forkSubagent.ts` | Fork 机制 + buildForkedMessages |
| `src/tools/AgentTool/agentToolUtils.ts` | 工具过滤逻辑 |
| `src/coordinator/coordinatorMode.ts` | Coordinator 模式 |
| `src/bridge/bridgeMain.ts` | Bridge 主逻辑（115KB） |
| `src/bridge/bridgeMessaging.ts` | 消息路由 + UUID 去重 |
| `src/bridge/types.ts` | Bridge 消息类型定义 |
| `src/services/lsp/LSPClient.ts` | JSON-RPC 客户端 |
| `src/services/lsp/LSPServerManager.ts` | 服务器池管理 |
| `src/services/lsp/LSPServerInstance.ts` | 单实例生命周期 |
| `src/plugins/builtinPlugins.ts` | 内置插件注册和加载 |
