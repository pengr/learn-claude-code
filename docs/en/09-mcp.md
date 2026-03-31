# 09 — MCP: Model Context Protocol Integration

> Files: `src/services/mcp/`, `src/tools/MCPTool/`, `src/tools/McpAuthTool/`

---

## 1. What is MCP

MCP (Model Context Protocol) is an open protocol published by Anthropic that allows external programs (MCP Servers) to provide additional tools and resources to Claude Code.

```
Claude Code (MCP Client)
      │
      │  Tool calls / resource reads
      ▼
MCP Server (external process or remote service)
  Examples:
  - @modelcontextprotocol/server-filesystem  File system
  - @modelcontextprotocol/server-github      GitHub API
  - Your own internal tools
```

---

## 2. MCPServerConnection Types

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

## 3. Six Transport Types

```typescript
type McpServerConfig =
  | { type: 'stdio';    command: string; args?: string[]; env?: Record<string, string> }
  | { type: 'sse';      url: string; headers?: Record<string, string> }
  | { type: 'http';     url: string; headers?: Record<string, string> }  // Streamable HTTP
  | { type: 'ws';       url: string }                                    // WebSocket
  | { type: 'sse-ide';  ... }                                            // VS Code IDE extension
  | { type: 'ws-ide';   ... }                                            // VS Code WebSocket
```

| Transport | Use Case | Characteristics |
|-----------|----------|-----------------|
| **stdio** | Local subprocess (most common) | SIGINT→SIGTERM→SIGKILL graceful shutdown |
| **sse** | Remote long-lived connection | Supports OAuth auth, XAA cross-app access |
| **http** | Streamable HTTP | Conforms to the latest MCP spec |
| **ws** | WebSocket | Bun/Node.js compatible, TLS support |
| **sse-ide / ws-ide** | IDE extension communication | Used internally by VS Code |

---

## 4. settings.json Configuration Format

```jsonc
// ~/.claude/settings.json or project .claude/settings.json
{
  "mcpServers": {
    // stdio: start a local process
    "filesystem": {
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "/tmp"],
      "env": { "NODE_ENV": "production" }
    },

    // SSE: connect to a remote service
    "my-remote-server": {
      "type": "sse",
      "url": "https://my-server.example.com/mcp",
      "headers": { "Authorization": "Bearer <token>" }
    },

    // Streamable HTTP (new spec)
    "my-http-server": {
      "type": "http",
      "url": "https://api.example.com/mcp"
    }
  }
}
```

**Configuration scopes (7 types):**

| Scope | Path | Description |
|-------|------|-------------|
| `user` | `~/.claude/settings.json` | User global |
| `project` | `.claude/settings.json` | Project-level, can be committed to git |
| `local` | `.claude/settings.local.json` | Local override, not committed |
| `enterprise` | `/etc/claude-code/managed-settings.json` | Enterprise-managed |
| `dynamic` | Added at runtime | Via `/add-mcp-server` command |
| `claudeai` | Claude.ai web interface config | Cloud-synced |
| `managed` | Managed environment config | BYOC/self-hosted |

---

## 5. Dynamic Tool Registration Mechanism

MCP tools are registered in Claude Code with the format `mcp__<server>__<toolname>`:

```typescript
// src/services/mcp/client.ts (excerpt)

// Tool names are automatically normalized:
// "get-issues" → "mcp__github__get_issues" (hyphens → underscores)

async function fetchToolsForClient(
  client: MCPClient,
  serverName: string,
): Promise<Tool[]> {
  // LRU cache (caches tool lists for up to 20 servers)
  const cached = toolsCache.get(serverName)
  if (cached) return cached

  const { tools } = await client.listTools()

  const registeredTools = tools.map(mcpTool =>
    buildTool({
      name: `mcp__${serverName}__${normalizeName(mcpTool.name)}`,
      description: truncate(mcpTool.description, 2048),  // description max 2048 chars
      inputSchema: mcpTool.inputSchema ?? z.object({}),
      isMcp: true,
      mcpInfo: { serverName, toolName: mcpTool.name },

      // Infer permission attributes from MCP tool hints
      isReadOnly: (input) => mcpTool.annotations?.readOnlyHint ?? false,
      isDestructive: (input) => mcpTool.annotations?.destructiveHint ?? false,
      isOpenWorld: (input) => mcpTool.annotations?.openWorldHint ?? false,

      async call(args, context) {
        // Auto-reconnect: clear cache and retry on session expiry
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

In addition to tools, MCP Servers can also provide **resources** (read-only data such as files, database records, etc.):

```typescript
// ListMcpResourcesTool — list all available resources
// Input:  { serverName?: string }  (omit to list resources from all servers)
// Output: { resources: Array<{ uri, name, mimeType, description }> }

// ReadMcpResourceTool — read a specific resource
// Input:  { serverName: string, uri: string }
// Output: resource content (text or base64-encoded binary)

// Binary resource auto-handling:
// mimeType is image/audio/etc. → automatically persisted to a temp directory, returns file path
```

---

## 7. MCP Auth (OAuth Authorization)

Some MCP Servers require OAuth authorization (e.g. connecting to GitHub, Slack, etc.):

```typescript
// McpAuthTool — initiate the authorization flow for a specified server
// Triggered when: server status is 'needs-auth'

// Flow:
// 1. Fetch authorization URL from the server
// 2. Open the authorization page in the user's browser
// 3. After the user completes authorization, the server calls back to Claude Code
// 4. Save the authorization credentials and reconnect to the server

// XAA (Cross-Application Authorization):
// Allows an MCP Server to use Claude Code's own OAuth token
// Suitable for Anthropic's own services (Claude.ai connections, etc.)
```

---

## 8. Error Handling & Reconnection

### 8.1 Custom Error Types

```typescript
class McpAuthError extends Error {
  constructor(public serverName: string, public authUrl: string) { ... }
}

class McpSessionExpiredError extends Error {
  // Triggered by: HTTP 404 response + JSON-RPC error code -32001
}

class McpToolCallError extends Error {
  constructor(public serverName: string, public toolName: string, cause: Error) { ... }
}
```

### 8.2 Reconnection Strategy

```
Consecutive tool call failures ≥ 3
        │
        ▼
Trigger reconnect (disconnect → reconnect)
        │
        ▼
Reconnect successful?
  YES → Clear tool cache, re-fetch tool list
  NO  → Server status becomes 'failed';
        next time the user calls a tool on this server, an error is shown
```

### 8.3 Key Timeout Configuration

| Operation | Timeout |
|-----------|---------|
| Connection establishment | 30 seconds |
| Tool call request | 60 seconds |
| OAuth authorization wait | 30 seconds |
| Concurrent connections (stdio) | Max 3 in parallel |
| Concurrent connections (HTTP/SSE) | Max 20 in parallel |
| Maximum tool result size | 100 KB |
| Maximum stderr captured | 64 MB |

---

## 9. stdio Server Lifecycle

```
Claude Code starts
    │
    ▼
spawn(command, args)          ← Start subprocess
    │
    ├─ Listen for spawn event to confirm startup (capture ENOENT errors, etc.)
    │
    ▼
JSON-RPC over stdio communication
    │
    ├─ stdin:  Claude Code → MCP Server
    └─ stdout: MCP Server → Claude Code
    (stderr: captured for logs, not part of the protocol)
    │
    ▼
Graceful shutdown when Claude Code exits:
    SIGINT → wait 100ms
    → SIGTERM → wait 200ms
    → SIGKILL (forced kill)
    Total ≤ 600ms
```

---

## 10. Using MCP Tools in Conversation

The model invokes MCP tools in exactly the same way as regular tools — completely transparent to the model:

```typescript
// Tool call seen by the model (tool_use block)
{
  type: 'tool_use',
  name: 'mcp__github__create_issue',  // mcp__<server>__<tool>
  input: {
    repo: 'my-org/my-repo',
    title: 'Bug: ...',
    body: '...',
  }
}

// Claude Code routes execution to the MCP Server
// Results are returned to the model as a normal tool_result
```

---

## Key File Reference

| File | Purpose |
|------|---------|
| `services/mcp/types.ts` | MCPServerConnection and other core types |
| `services/mcp/client.ts` | Connection management, tool registration, reconnection logic (2000+ lines) |
| `tools/MCPTool/` | MCPTool implementation (directly calls any MCP tool) |
| `tools/McpAuthTool/` | OAuth authorization flow |
| `tools/ListMcpResourcesTool/` | List MCP resources |
| `tools/ReadMcpResourceTool/` | Read MCP resources |
