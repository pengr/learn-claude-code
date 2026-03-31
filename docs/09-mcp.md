# 09 — MCP：Model Context Protocol 集成

> 文件：`src/services/mcp/`，`src/tools/MCPTool/`，`src/tools/McpAuthTool/`

---

## 1. MCP 是什么

MCP（Model Context Protocol）是 Anthropic 发布的开放协议，允许外部程序（MCP Server）向 Claude Code 提供额外的工具和资源。

```
Claude Code（MCP Client）
      │
      │  工具调用 / 资源读取
      ▼
MCP Server（外部进程或远程服务）
  例如：
  - @modelcontextprotocol/server-filesystem  文件系统
  - @modelcontextprotocol/server-github      GitHub API
  - 你自己写的内部工具
```

---

## 2. MCPServerConnection 类型

```typescript
// src/services/mcp/types.ts
type MCPServerConnection =
  | {
      type: 'connected'
      name: string
      client: MCPClient
      tools: Tool[]
      resources: ServerResource[]
      capabilities: ServerCapabilities
    }
  | {
      type: 'failed'
      name: string
      error: Error
      config: McpServerConfig
    }
  | {
      type: 'needs-auth'
      name: string
      authUrl: string
      config: McpServerConfig
    }
  | {
      type: 'pending'
      name: string
      config: McpServerConfig
    }
  | {
      type: 'disabled'
      name: string
      config: McpServerConfig
    }
```

---

## 3. 六种 Transport 类型

```typescript
type McpServerConfig =
  | { type: 'stdio';    command: string; args?: string[]; env?: Record<string, string> }
  | { type: 'sse';      url: string; headers?: Record<string, string> }
  | { type: 'http';     url: string; headers?: Record<string, string> }  // Streamable HTTP
  | { type: 'ws';       url: string }                                    // WebSocket
  | { type: 'sse-ide';  ... }                                            // VS Code IDE 扩展
  | { type: 'ws-ide';   ... }                                            // VS Code WebSocket
```

| Transport | 适用场景 | 特点 |
|-----------|----------|------|
| **stdio** | 本地子进程（最常用） | SIGINT→SIGTERM→SIGKILL 梯度关闭 |
| **sse** | 远程长连接 | 支持 OAuth 鉴权，XAA 跨应用访问 |
| **http** | Streamable HTTP | 符合最新 MCP 规范 |
| **ws** | WebSocket | Bun/Node.js 兼容，TLS 支持 |
| **sse-ide / ws-ide** | IDE 扩展通信 | VS Code 内部使用 |

---

## 4. settings.json 配置格式

```jsonc
// ~/.claude/settings.json 或项目 .claude/settings.json
{
  "mcpServers": {
    // stdio：启动本地进程
    "filesystem": {
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "/tmp"],
      "env": { "NODE_ENV": "production" }
    },

    // SSE：连接远程服务
    "my-remote-server": {
      "type": "sse",
      "url": "https://my-server.example.com/mcp",
      "headers": { "Authorization": "Bearer <token>" }
    },

    // Streamable HTTP（新规范）
    "my-http-server": {
      "type": "http",
      "url": "https://api.example.com/mcp"
    }
  }
}
```

**配置作用域（7 种）：**

| 作用域 | 路径 | 说明 |
|--------|------|------|
| `user` | `~/.claude/settings.json` | 用户全局 |
| `project` | `.claude/settings.json` | 项目级，可提交到 git |
| `local` | `.claude/settings.local.json` | 本地覆盖，不提交 |
| `enterprise` | `/etc/claude-code/managed-settings.json` | 企业统一管理 |
| `dynamic` | 运行时动态添加 | `/add-mcp-server` 命令 |
| `claudeai` | Claude.ai Web 界面配置 | 云端同步 |
| `managed` | 托管环境配置 | BYOC/自托管 |

---

## 5. 工具动态注册机制

MCP 工具在 Claude Code 中以 `mcp__<server>__<toolname>` 格式注册：

```typescript
// src/services/mcp/client.ts（节选）

// 工具名自动规范化：
// "get-issues" → "mcp__github__get_issues"（连字符→下划线）

async function fetchToolsForClient(
  client: MCPClient,
  serverName: string,
): Promise<Tool[]> {
  // LRU 缓存（最多缓存 20 个 server 的工具列表）
  const cached = toolsCache.get(serverName)
  if (cached) return cached

  const { tools } = await client.listTools()

  const registeredTools = tools.map(mcpTool =>
    buildTool({
      name: `mcp__${serverName}__${normalizeName(mcpTool.name)}`,
      description: truncate(mcpTool.description, 2048),  // 描述最长 2048 字符
      inputSchema: mcpTool.inputSchema ?? z.object({}),
      isMcp: true,
      mcpInfo: { serverName, toolName: mcpTool.name },

      // 从 MCP 工具的 hints 推断权限属性
      isReadOnly: (input) => mcpTool.annotations?.readOnlyHint ?? false,
      isDestructive: (input) => mcpTool.annotations?.destructiveHint ?? false,
      isOpenWorld: (input) => mcpTool.annotations?.openWorldHint ?? false,

      async call(args, context) {
        // 自动重连：会话过期时清缓存重试
        try {
          return await callMCPTool(client, mcpTool.name, args)
        } catch (err) {
          if (err instanceof McpSessionExpiredError) {
            toolsCache.delete(serverName)
            await reconnectClient(serverName)
            return await callMCPTool(client, mcpTool.name, args)
          }
          throw err
        }
      },
    })
  )

  toolsCache.set(serverName, registeredTools)
  return registeredTools
}
```

---

## 6. MCP Resources

除了工具，MCP Server 还可以提供**资源**（只读数据，如文件、数据库记录等）：

```typescript
// ListMcpResourcesTool — 列出所有可用资源
// 输入：{ serverName?: string }  (不传则列出所有 server 的资源)
// 输出：{ resources: Array<{ uri, name, mimeType, description }> }

// ReadMcpResourceTool — 读取具体资源
// 输入：{ serverName: string, uri: string }
// 输出：资源内容（文本或 base64 编码的二进制）

// 二进制资源自动处理：
// mimeType 为图片/音频等 → 自动持久化到临时目录，返回文件路径
```

---

## 7. MCP Auth（OAuth 鉴权）

部分 MCP Server 需要 OAuth 授权（如连接 GitHub、Slack 等）：

```typescript
// McpAuthTool — 为指定 server 发起授权流程
// 触发时机：server 状态为 'needs-auth'

// 流程：
// 1. 从 server 获取授权 URL
// 2. 在用户浏览器中打开授权页
// 3. 用户完成授权后，server 回调 Claude Code
// 4. 保存授权凭证，重新连接 server

// XAA（Cross-Application Authorization）：
// 允许 MCP Server 使用 Claude Code 自身的 OAuth token
// 适合 Anthropic 自家的服务（Claude.ai 连接等）
```

---

## 8. 错误处理与重连

### 8.1 自定义错误类型

```typescript
class McpAuthError extends Error {
  constructor(public serverName: string, public authUrl: string) { ... }
}

class McpSessionExpiredError extends Error {
  // 触发条件：HTTP 404 响应 + JSON-RPC 错误码 -32001
}

class McpToolCallError extends Error {
  constructor(public serverName: string, public toolName: string, cause: Error) { ... }
}
```

### 8.2 重连策略

```
连续工具调用失败 ≥ 3 次
        │
        ▼
触发重连（disconnect → reconnect）
        │
        ▼
重连成功？
  YES → 清除工具缓存，重新拉取工具列表
  NO  → server 状态变为 'failed'，
        下次用户调用该工具时报错提示
```

### 8.3 关键超时配置

| 操作 | 超时 |
|------|------|
| 连接建立 | 30 秒 |
| 工具调用请求 | 60 秒 |
| OAuth 授权等待 | 30 秒 |
| 并发连接数（stdio） | 最多 3 个并行 |
| 并发连接数（HTTP/SSE） | 最多 20 个并行 |
| 工具结果最大大小 | 100KB |
| stderr 最大捕获 | 64MB |

---

## 9. stdio Server 生命周期

```
Claude Code 启动
    │
    ▼
spawn(command, args)          ← 启动子进程
    │
    ├─ 监听 spawn 事件，确认启动成功（捕获 ENOENT 等错误）
    │
    ▼
JSON-RPC over stdio 通信
    │
    ├─ stdin:  Claude Code → MCP Server
    └─ stdout: MCP Server → Claude Code
    (stderr: 捕获日志，不参与协议)
    │
    ▼
Claude Code 退出时优雅关闭：
    SIGINT → 等待 100ms
    → SIGTERM → 等待 200ms
    → SIGKILL（强制杀死）
    总计 ≤ 600ms
```

---

## 10. 在对话中使用 MCP 工具

模型调用 MCP 工具的方式与普通工具完全相同，对模型透明：

```typescript
// 模型看到的工具调用（tool_use block）
{
  type: 'tool_use',
  name: 'mcp__github__create_issue',  // mcp__<server>__<tool>
  input: {
    repo: 'my-org/my-repo',
    title: 'Bug: ...',
    body: '...',
  }
}

// Claude Code 路由到 MCP Server 执行
// 结果以普通 tool_result 返回给模型
```

---

## 关键文件速查

| 文件 | 功能 |
|------|------|
| `services/mcp/types.ts` | MCPServerConnection 等核心类型 |
| `services/mcp/client.ts` | 连接管理、工具注册、重连逻辑（2000+ 行） |
| `tools/MCPTool/` | MCPTool 工具实现（直接调用任意 MCP 工具） |
| `tools/McpAuthTool/` | OAuth 授权流程 |
| `tools/ListMcpResourcesTool/` | 列出 MCP 资源 |
| `tools/ReadMcpResourceTool/` | 读取 MCP 资源 |
